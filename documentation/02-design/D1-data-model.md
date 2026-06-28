---
title: Data Model
layer: 02-design
status: current
lastUpdated: 2026-06-28
---

# D1 · Data Model

## Overview

AnjiSchedulo uses an **Event-Driven Microservices** architecture with **Event Sourcing** at the Appointment aggregate level. The data model reflects three categories of entities:

- **Domain Entities** (13): Entities with business meaning, owned by bounded contexts.
- **Read Models** (3): CQRS read-side projections updated incrementally from domain events: `TenantDashboardView`, `AnalyticsSummary`, and incremental appointment state.
- **Infrastructure Entities** (5): Plumbing that enables reliability patterns: `OutboxRecord`, `InboxRecord`, `BookingIdempotencyKey`, `AuditLog`, `ReplayJob`.

**Multi-tenancy:** All entities carry `tenant_id`. PostgreSQL Row-Level Security (RLS) enforces isolation at the database layer — a missing or null tenant context returns zero rows, never all rows (BR005, ADR002).

**Core Aggregate:** `Appointment` is the central aggregate. Its state is derived by replaying its ordered `AppointmentEvent` stream (Event Sourcing, ADR001). There is no mutable status column on the Appointment row.

---

## Entity Relationship Overview

```
  TENANT AGGREGATE ROOT
  ┌──────────────────────────────────────┐
  │  Tenant                              │
  │    ├── TenantConfig (1:1)            │
  │    ├── Service (1:N)                 │
  │    └── StaffMember (1:N)             │
  │          ├── WorkingHours (1:N)      │
  │          └── StaffBlock (1:N)        │
  └──────────────────────────────────────┘

  CUSTOMER (tenant-scoped, not an aggregate root)
  ┌──────────────────────────────────────────────┐
  │  Customer (tenant_id FK → Tenant)            │
  └──────────────────────────────────────────────┘

  APPOINTMENT AGGREGATE ROOT
  ┌──────────────────────────────────────────────────────┐
  │  Appointment (tenant_id, customer_id, staff_id,      │
  │               service_id)                            │
  │    ├── AppointmentEvent[] (event stream, 1:N)        │
  │    ├── SlotLock (1:1, lifecycle-scoped)              │
  │    └── Payment (0:1)                                 │
  └──────────────────────────────────────────────────────┘

  NOTIFICATIONS
  ┌──────────────────────────────────────┐
  │  NotificationRecord (appointment_id) │
  └──────────────────────────────────────┘

  READ MODELS (CQRS — derived, never written by commands)
  ┌──────────────────────────────────────────────────────┐
  │  TenantDashboardView (tenant_id, view_date)          │
  │  AnalyticsSummary (tenant_id, window_start/end)      │
  └──────────────────────────────────────────────────────┘

  INFRASTRUCTURE ENTITIES
  ┌──────────────────────────────────────────────────────┐
  │  OutboxRecord          (per service)                 │
  │  InboxRecord           (per consumer service)        │
  │  BookingIdempotencyKey (booking-command-service)     │
  │  AuditLog              (all services)                │
  │  ReplayJob             (ops-service)                 │
  └──────────────────────────────────────────────────────┘
```

---

## Entity Catalogue

---

### Tenant

**Type:** Aggregate Root  
**Owned by:** tenant-service  
**Source:** FR001, BR005, BR011, BR014

**Description:** A business registered on AnjiSchedulo. Owns all configuration, staff, and service definitions for one isolated business unit. The Tenant aggregate is the entry point for onboarding (UJ001) and offboarding (UJ007). Its provisioning is a multi-step pub/sub fan-out driven by the `tenant.provisioned` event.

**Attributes:**

| Attribute | Type | Constraints | Notes |
|---|---|---|---|
| `tenant_id` | UUID | PK | Generated at registration |
| `name` | string | NOT NULL | Business name |
| `owner_email` | string (PII) | NOT NULL, UNIQUE (active+provisioning) | Owner email address |
| `owner_name` | string (PII) | NOT NULL | Owner full name |
| `plan` | enum | NOT NULL | `starter` \| `pro` \| `enterprise` |
| `status` | enum | NOT NULL | `pending` \| `provisioning` \| `active` \| `suspended` \| `deprovisioned` |
| `stripe_customer_id` | string | NOT NULL | Stripe Customer ID created at registration |
| `created_at` | timestamp | NOT NULL | Registration timestamp |
| `deprovisioned_at` | timestamp | NULLABLE | Set on final deprovisioning |

**Lifecycle:** `pending` → `provisioning` → `active` → `suspended` → `deprovisioned`

**Children:** TenantConfig (1:1), Service (1:N), StaffMember (1:N)

**Pattern Notes:** Transactional Outbox ensures `tenant.provisioned` event delivery without dual-write risk (ADR005). Pool+RLS — `tenant_id` on all child tables. Immutable audit log on every state transition (BR014). Suspension permits read-only access for 30 days before deprovisioning (BR011).

---

### TenantConfig

**Type:** Child Entity (Value Object embedded in Tenant aggregate)  
**Owned by:** tenant-service  
**Source:** FR002, BR002, BR008, BR010

**Description:** Tenant-level operational parameters governing booking rules. Owned by Tenant. Read by availability-service and booking-command-service to enforce business rules.

**Attributes:**

| Attribute | Type | Constraints | Notes |
|---|---|---|---|
| `tenant_id` | UUID | PK, FK → Tenant | Part of PK |
| `cancellation_hours` | integer | NOT NULL | Min hours before appt for allowed cancellation (BR002) |
| `booking_window_days` | integer | NOT NULL | Max days in advance a slot can be booked (BR010) |
| `max_advance_bookings` | integer | NOT NULL | Max active bookings per customer (BR008) |
| `timezone` | string | NOT NULL | IANA timezone identifier |
| `currency` | string | NOT NULL | ISO 4217 currency code |

**Lifecycle:** None (stateless value object)

**Pattern Notes:** Validated by booking-command-service (BR002, BR008, BR010) and availability-service (BR003, BR010) at request time. Broadcast via `tenant.configured` event on every change so consumers maintain a local cache (NFR008).

---

### Service

**Type:** Child Entity  
**Owned by:** tenant-service  
**Source:** FR002, FR003

**Description:** A bookable offering defined by a tenant. Customers select a Service when browsing availability (FR003). Deactivation hides the service from the booking page but does not modify historical appointment records.

**Attributes:**

| Attribute | Type | Constraints | Notes |
|---|---|---|---|
| `service_id` | UUID | PK | |
| `tenant_id` | UUID | NOT NULL, FK, RLS | |
| `name` | string | NOT NULL | |
| `duration_minutes` | integer | NOT NULL, ≥ 15 | |
| `price_amount` | decimal | NOT NULL, ≥ 0 | Stored in smallest currency unit (pence/cents) |
| `price_currency` | string | NOT NULL | ISO 4217 |
| `active` | boolean | NOT NULL, default true | |
| `created_at` | timestamp | NOT NULL | |
| `updated_at` | timestamp | NOT NULL | |

**Lifecycle:** `active` → `inactive`

**Pattern Notes:** Starter tier: max 5 services. Pro/Enterprise: unlimited. Deactivation is soft-delete — historical appointment records retain service snapshot data via Event-Carried State Transfer in AppointmentEvent payload.

---

### StaffMember

**Type:** Child Entity  
**Owned by:** tenant-service  
**Source:** FR002, FR003

**Description:** An employee or contractor whose availability is managed on the platform (P2). Their computed open slots form the availability read model consumed by customers browsing slots (FR003).

**Attributes:**

| Attribute | Type | Constraints | Notes |
|---|---|---|---|
| `staff_id` | UUID | PK | |
| `tenant_id` | UUID | NOT NULL, FK, RLS | |
| `name` | string (PII) | NOT NULL | |
| `email` | string (PII) | NOT NULL | Used for booking notifications |
| `active` | boolean | NOT NULL, default true | |
| `created_at` | timestamp | NOT NULL | |

**Lifecycle:** `active` → `inactive`

**Children:** WorkingHours (1:N), StaffBlock (1:N)

**Pattern Notes:** Tier limits: 3 (Starter), 20 (Pro), unlimited (Enterprise). Deactivating a StaffMember must not cancel existing confirmed appointments.

---

### WorkingHours

**Type:** Child Entity  
**Owned by:** tenant-service  
**Source:** FR002, BR003

**Description:** The recurring weekly schedule for a StaffMember. Defines the time windows within which slots are generated by availability-service. Enforces BR003 — outside these windows, no slots are shown to customers.

**Attributes:**

| Attribute | Type | Constraints | Notes |
|---|---|---|---|
| `working_hours_id` | UUID | PK | |
| `staff_id` | UUID | NOT NULL, FK | |
| `tenant_id` | UUID | NOT NULL, RLS | Denormalised for RLS |
| `day_of_week` | enum | NOT NULL | `monday`…`sunday` |
| `start_time` | time | NOT NULL | |
| `end_time` | time | NOT NULL | |
| `effective_from` | date | NOT NULL | When this schedule becomes active |
| `effective_to` | date | NULLABLE | When this schedule ends |

**Pattern Notes:** `effective_from`/`to` allows future schedule changes without breaking historic data. Availability-service joins WorkingHours with StaffBlock and existing confirmed appointments to compute open slots.

---

### StaffBlock

**Type:** Child Entity  
**Owned by:** tenant-service  
**Source:** FR002, P2

**Description:** A manual unavailability period set by a StaffMember or Tenant Admin. Prevents bookings during that window even if WorkingHours would otherwise allow it.

**Attributes:**

| Attribute | Type | Constraints | Notes |
|---|---|---|---|
| `block_id` | UUID | PK | |
| `staff_id` | UUID | NOT NULL, FK | |
| `tenant_id` | UUID | NOT NULL, RLS | |
| `block_start` | timestamp | NOT NULL | |
| `block_end` | timestamp | NOT NULL | |
| `reason` | string | NULLABLE | Optional operator note |
| `created_by_role` | enum | NOT NULL | `staff` \| `admin` |

**Pattern Notes:** Immutable once created. To reschedule, create a new block and delete the old one. Consumed by availability-service on every slot computation.

---

### Customer

**Type:** Child Entity (tenant-scoped)  
**Owned by:** booking-command-service  
**Source:** FR004, FR013, BR008

**Description:** The end customer who books appointments (P3). Tenant-scoped — no cross-tenant customer identity in v1. PII subject to GDPR erasure (FR013).

**Attributes:**

| Attribute | Type | Constraints | Notes |
|---|---|---|---|
| `customer_id` | UUID | PK | |
| `tenant_id` | UUID | NOT NULL, FK, RLS | |
| `name` | string (PII) | NOT NULL | |
| `email` | string (PII) | NOT NULL | |
| `phone` | string (PII) | NULLABLE | |
| `active_booking_count` | integer | NOT NULL, default 0 | Denormalised for BR008 enforcement |
| `created_at` | timestamp | NOT NULL | |
| `pseudonymised_at` | timestamp | NULLABLE | Set on GDPR erasure |

**Lifecycle:** `active` → `pseudonymised`

**Pattern Notes:** `active_booking_count` incremented/decremented atomically by booking-command-service to enforce BR008. On GDPR erasure, name/email/phone are replaced with pseudonymous tokens; historical AppointmentEvent records retain `customer_id` FK but the PII is removed from the Customer row.

---

### Appointment

**Type:** Aggregate Root  
**Owned by:** booking-command-service  
**Source:** FR004, BR001, BR004, BR006, ADR001

**Description:** The core aggregate. Represents a confirmed reservation between a Customer and a StaffMember for a Service at a specific time. State is derived by replaying AppointmentEvent[] in `occurred_at` order — there is no mutable status column. The booking saga orchestrated by booking-command-service is the sole writer.

**Attributes:**

| Attribute | Type | Constraints | Notes |
|---|---|---|---|
| `appointment_id` | UUID | PK | |
| `tenant_id` | UUID | NOT NULL, FK, RLS | |
| `customer_id` | UUID | NOT NULL, FK | |
| `staff_id` | UUID | NOT NULL, FK | |
| `service_id` | UUID | NOT NULL, FK | |
| `slot_date` | date | NOT NULL | |
| `slot_start` | time | NOT NULL | |
| `slot_end` | time | NOT NULL | |
| `idempotency_key` | UUID | NOT NULL, UNIQUE | Client-provided dedup key (BR006) |
| `correlation_id` | UUID | NOT NULL | Distributed tracing ID from api-gateway (NFR014) |
| `created_at` | timestamp | NOT NULL | |

**Lifecycle:** `requested` → `confirmed` → `rescheduled` → `cancelled` → `completed` → `failed`

**Children:** AppointmentEvent (1:N)

**Pattern Notes:** Event Sourcing (ADR001) — current state derived by replaying AppointmentEvent[] in `occurred_at` order. Saga Orchestration (BR004): booking-command-service coordinates slot reservation → payment → confirmation. SlotLock unique partial index on `(tenant_id, staff_id, slot_date, slot_start) WHERE released_at IS NULL` is the double-booking prevention mechanism (BR001).

---

### AppointmentEvent

**Type:** Child Entity (Event Store Record)  
**Owned by:** booking-command-service  
**Source:** ADR001, BR013, FR011

**Description:** An immutable, ordered event record in the Appointment event stream. The ordered sequence of AppointmentEvents is the source of truth for Appointment state. Also the source for event replay (FR011). Full snapshot in payload enables Event-Carried State Transfer — downstream consumers never need to call back.

**Attributes:**

| Attribute | Type | Constraints | Notes |
|---|---|---|---|
| `event_id` | UUID | PK | |
| `appointment_id` | UUID | NOT NULL, FK | |
| `tenant_id` | UUID | NOT NULL, RLS | |
| `event_type` | enum | NOT NULL | `appointment.requested` \| `appointment.confirmed` \| `appointment.cancelled` \| `appointment.rescheduled` \| `appointment.completed` \| `appointment.failed` |
| `occurred_at` | timestamp | NOT NULL | Ordering field |
| `correlation_id` | UUID | NOT NULL | Distributed tracing ID (BR013) |
| `causation_id` | UUID | NULLABLE | ID of the triggering event |
| `actor_id` | UUID | NOT NULL | customer_id, staff_id, or system |
| `actor_role` | enum | NOT NULL | `customer` \| `staff` \| `admin` \| `system` |
| `payload` | JSONB | NOT NULL | Full appointment + service + customer snapshot |

**Lifecycle:** None (INSERT-only, immutable)

**Pattern Notes:** INSERT-only table — no UPDATE or DELETE. Full snapshot in `payload` enables Event-Carried State Transfer — downstream consumers (dashboard-service, analytics-service, notification-service) need no back-calls to booking-command-service. Required by BR013: every event carries `tenant_id`, `event_id`, `correlation_id`, `occurred_at`. Source for replay operations (FR011).

---

### SlotLock

**Type:** Infrastructure Entity (domain-visible)  
**Owned by:** booking-command-service  
**Source:** BR001, BR004, BR009

**Description:** A database-level exclusive reservation placed on a time slot when a booking saga begins. The unique partial index on the active lock is the primary enforcement mechanism for the double-booking invariant (BR001).

**Attributes:**

| Attribute | Type | Constraints | Notes |
|---|---|---|---|
| `lock_id` | UUID | PK | |
| `tenant_id` | UUID | NOT NULL, RLS | |
| `staff_id` | UUID | NOT NULL, FK | |
| `slot_date` | date | NOT NULL | |
| `slot_start` | time | NOT NULL | |
| `appointment_id` | UUID | NOT NULL, FK | |
| `locked_at` | timestamp | NOT NULL | |
| `released_at` | timestamp | NULLABLE | NULL = lock is held |

**Lifecycle:** `locked` → `released`

**Pattern Notes:** Unique partial index on `(tenant_id, staff_id, slot_date, slot_start) WHERE released_at IS NULL` enforces BR001. Released atomically by the compensation path if the booking saga fails (BR004). Released by the cancellation saga when an appointment is cancelled (BR009).

---

### Payment

**Type:** Child Entity  
**Owned by:** payment-service  
**Source:** FR005, BR009, BR012

**Description:** Payment record associated with a confirmed appointment. Stores the Stripe PaymentIntent ID and current status. Created by payment-service during the booking saga. Refunded by the cancellation saga (BR009).

**Attributes:**

| Attribute | Type | Constraints | Notes |
|---|---|---|---|
| `payment_id` | UUID | PK | |
| `appointment_id` | UUID | NOT NULL, FK | |
| `tenant_id` | UUID | NOT NULL, RLS | |
| `stripe_payment_intent_id` | string | NOT NULL | |
| `amount` | decimal | NOT NULL | Smallest currency unit |
| `currency` | string | NOT NULL | ISO 4217 |
| `status` | enum | NOT NULL | `pending` \| `captured` \| `failed` \| `refunded` \| `refund_failed` |
| `captured_at` | timestamp | NULLABLE | |
| `refunded_at` | timestamp | NULLABLE | |
| `stripe_refund_id` | string | NULLABLE | |

**Lifecycle:** `pending` → `captured` / `failed` / `refunded` / `refund_failed`

**Pattern Notes:** `refund_failed` transitions route to the DLQ for manual resolution (BR009). Records retained for 7 years per BR012.

---

### NotificationRecord

**Type:** Child Entity  
**Owned by:** notification-service  
**Source:** FR007, BR007

**Description:** A record of every outbound notification attempt. Used for delivery tracking, retry management, and DLQ escalation (FR007). Failure of a notification never affects booking operations (BR007).

**Attributes:**

| Attribute | Type | Constraints | Notes |
|---|---|---|---|
| `notification_id` | UUID | PK | |
| `tenant_id` | UUID | NOT NULL, RLS | |
| `appointment_id` | UUID | NOT NULL, FK | |
| `recipient_type` | enum | NOT NULL | `customer` \| `staff` \| `admin` |
| `recipient_contact` | string (PII) | NOT NULL | Email or phone |
| `channel` | enum | NOT NULL | `email` \| `sms` \| `whatsapp` |
| `template_name` | string | NOT NULL | |
| `status` | enum | NOT NULL | `pending` \| `sent` \| `failed` \| `dlq` |
| `attempt_count` | integer | NOT NULL, default 0 | |
| `last_attempted_at` | timestamp | NULLABLE | |
| `sent_at` | timestamp | NULLABLE | |
| `failure_reason` | string | NULLABLE | |
| `trigger_event_id` | UUID | NOT NULL | AppointmentEvent that triggered this |

**Lifecycle:** `pending` → `sent` / `failed` → `dlq`

**Pattern Notes:** Retry up to 4 times with exponential backoff (delays: 1s, 2s, 4s, 8s). After 4 failures, routes to DLQ and ops alert fires. Notification failures are fully isolated from booking operations (BR007).

---

### TenantDashboardView

**Type:** Read Model (CQRS Read Side)  
**Owned by:** dashboard-service  
**Source:** FR008, NFR006, NFR008

**Description:** Pre-aggregated per-tenant per-day dashboard data for tenant admins (UJ005). Updated incrementally when AppointmentEvents arrive. Rebuilt from full event history by replay (FR011). Designed for O(1) dashboard load.

**Attributes:**

| Attribute | Type | Constraints | Notes |
|---|---|---|---|
| `tenant_id` | UUID | PK (part), RLS | |
| `view_date` | date | PK (part) | Calendar date this view covers |
| `confirmed_count` | integer | NOT NULL, default 0 | |
| `cancelled_count` | integer | NOT NULL, default 0 | |
| `rescheduled_count` | integer | NOT NULL, default 0 | |
| `net_revenue` | decimal | NOT NULL, default 0.00 | Confirmed minus refunded |
| `cancellation_rate` | decimal | NOT NULL, default 0.0 | Ratio 0.0–1.0 |
| `peak_hour` | integer | NULLABLE | Hour 0–23 with most confirmed bookings |
| `last_updated_at` | timestamp | NOT NULL | |
| `last_event_id` | UUID | NULLABLE | Watermark for idempotent incremental updates |

**Pattern Notes:** `last_event_id` watermark enables idempotent incremental updates and safe replay. Response time < 200ms (NFR006) served directly from this view. CQRS read model — never written by commands, only by events.

---

### AnalyticsSummary

**Type:** Read Model (Stream Processing)  
**Owned by:** analytics-service  
**Source:** FR009, NFR005

**Description:** Per-tenant booking velocity and cancellation rate summary maintained via Kinesis stream processing. Used to detect booking anomalies (FR009).

**Attributes:**

| Attribute | Type | Constraints | Notes |
|---|---|---|---|
| `tenant_id` | UUID | NOT NULL, RLS | |
| `window_start` | timestamp | NOT NULL | |
| `window_end` | timestamp | NOT NULL | |
| `booking_count` | integer | NOT NULL | |
| `cancellation_count` | integer | NOT NULL | |
| `velocity_ratio` | decimal | NOT NULL | Current rate vs. historical average |
| `alert_triggered` | boolean | NOT NULL, default false | |
| `computed_at` | timestamp | NOT NULL | |

**Pattern Notes:** Derived from real-time event stream. Anomaly alerts when `velocity_ratio > 3.0` or `cancellation_rate > 0.30` (FR009). Not rebuilt by replay in v1.

---

### OutboxRecord

**Type:** Infrastructure Entity  
**Owned by:** All services (each owns its own outbox table)  
**Source:** ADR005, NFR003, BR013

**Description:** Written in the same database transaction as the domain event it carries. Polled by outbox-relay, which publishes to SNS and marks published. Eliminates dual-write risk (NFR003).

**Attributes:**

| Attribute | Type | Constraints | Notes |
|---|---|---|---|
| `outbox_id` | UUID | PK | |
| `tenant_id` | UUID | NOT NULL | |
| `topic` | string | NOT NULL | SNS topic name |
| `event_type` | string | NOT NULL | |
| `payload` | JSONB | NOT NULL | Full event envelope |
| `created_at` | timestamp | NOT NULL | |
| `published_at` | timestamp | NULLABLE | NULL = unpublished |

**Lifecycle:** `pending` (published_at IS NULL) → `published`

**Pattern Notes:** Transactional Outbox (ADR005, NFR003). outbox-relay polls every 500ms. At-least-once delivery. Consumers handle duplicates via InboxRecord deduplication.

---

### InboxRecord

**Type:** Infrastructure Entity  
**Owned by:** All consumer services (each owns its own inbox table)  
**Source:** BR006, NFR009

**Description:** Consumer-side deduplication table. Before processing any event, the consumer inserts `(event_id, consumer_name)`. If the insert fails on the unique constraint, the event is a duplicate and is skipped. Safe for replay.

**Attributes:**

| Attribute | Type | Constraints | Notes |
|---|---|---|---|
| `event_id` | UUID | PK (part) | From event envelope |
| `consumer_name` | string | PK (part) | Service name |
| `processed_at` | timestamp | NOT NULL | |
| `result` | string | NULLABLE | |

**Pattern Notes:** Idempotent Consumer (BR006, NFR009). Unique constraint on `(event_id, consumer_name)`. INSERT-only. Enables safe replay (FR011) — already-processed events are skipped without side effects.

---

### BookingIdempotencyKey

**Type:** Infrastructure Entity  
**Owned by:** booking-command-service  
**Source:** BR006, FR004

**Description:** Prevents duplicate booking operations from network retries (BR006). Client provides an `idempotency_key` with each booking request. If the key has been seen, the stored result is returned without re-processing.

**Attributes:**

| Attribute | Type | Constraints | Notes |
|---|---|---|---|
| `idempotency_key` | UUID | PK | Client-provided |
| `tenant_id` | UUID | NOT NULL | |
| `appointment_id` | UUID | NULLABLE | Set on successful booking |
| `result_status` | enum | NOT NULL | `confirmed` \| `failed` |
| `result_payload` | JSONB | NOT NULL | Full response to return |
| `created_at` | timestamp | NOT NULL | |
| `expires_at` | timestamp | NOT NULL | 24-hour TTL |

**Pattern Notes:** Idempotent Consumer pattern at the API entry point (BR006). TTL-expired keys purged by scheduled cleanup job.

---

### AuditLog

**Type:** Infrastructure Entity  
**Owned by:** All services (each writes to a shared audit_log table)  
**Source:** BR014, FR013, BR012

**Description:** Immutable audit record for every write operation that modifies tenant-scoped data (BR014). Supports GDPR compliance, dispute resolution, and incident forensics (FR013). Retained permanently — never deleted.

**Attributes:**

| Attribute | Type | Constraints | Notes |
|---|---|---|---|
| `audit_id` | UUID | PK | |
| `tenant_id` | UUID | NOT NULL | |
| `entity_type` | string | NOT NULL | e.g. `Appointment`, `Tenant`, `StaffMember` |
| `entity_id` | UUID | NOT NULL | |
| `operation` | enum | NOT NULL | `create` \| `update` \| `cancel` \| `reschedule` \| `suspend` \| `deprovision` \| `export` \| `replay` |
| `actor_id` | UUID | NOT NULL | |
| `actor_role` | enum | NOT NULL | `customer` \| `staff` \| `admin` \| `operator` \| `system` |
| `occurred_at` | timestamp | NOT NULL | |
| `before_snapshot` | JSONB | NULLABLE | Entity state before change |
| `after_snapshot` | JSONB | NOT NULL | Entity state after change |

**Pattern Notes:** INSERT-only. Database role `booking_app_user` has no UPDATE or DELETE privilege on `audit_log` (BR014). Retained permanently even after tenant deprovisioning. Mirrored to S3 Object Lock (WORM) bucket.

---

### ReplayJob

**Type:** Infrastructure Entity  
**Owned by:** ops-service  
**Source:** FR011, UJ006

**Description:** Tracks the lifecycle of a tenant event replay operation (FR011, UJ006). Provides operator visibility into replay progress.

**Attributes:**

| Attribute | Type | Constraints | Notes |
|---|---|---|---|
| `job_id` | UUID | PK | |
| `tenant_id` | UUID | NOT NULL | Target tenant |
| `initiated_by` | UUID | NOT NULL | Operator user_id |
| `scope_type` | enum | NOT NULL | `full_tenant` \| `date_range` \| `event_type` \| `specific_ids` |
| `scope_params` | JSONB | NOT NULL | e.g. `{"from":"2026-01-01","to":"2026-06-01"}` |
| `status` | enum | NOT NULL | `queued` \| `running` \| `completed` \| `failed` \| `cancelled` |
| `events_published` | integer | NOT NULL, default 0 | |
| `started_at` | timestamp | NULLABLE | |
| `completed_at` | timestamp | NULLABLE | |
| `error_message` | string | NULLABLE | |

**Lifecycle:** `queued` → `running` → `completed` / `failed` / `cancelled`

**Pattern Notes:** ops-service creates and updates ReplayJob records. Replay events include `replay_job_id` in their envelope. Live booking operations are not affected by replay (FR011).

---

## Aggregate Boundaries

AnjiSchedulo defines two aggregate roots, each owning a distinct consistency boundary.

### Tenant Aggregate
**Root:** `Tenant`  
**Owns:** TenantConfig, Service, StaffMember, WorkingHours, StaffBlock

All configuration for a single business. Changes within this aggregate require no coordination with the Appointment aggregate. Consistency boundary enforced by tenant-service: all writes go through it and produce `tenant.*` events via Transactional Outbox.

### Appointment Aggregate
**Root:** `Appointment`  
**Owns:** AppointmentEvent[]

The most critical consistency boundary. The invariant that a slot can never be double-booked (BR001) is enforced by a unique partial index on `slot_locks`. The Appointment row is INSERT-only; all state change is expressed through AppointmentEvents. booking-command-service is the sole orchestrator that may write to this aggregate.

**Why two roots?** Tenant configuration and appointment booking operate at different timescales with independent consistency requirements. A tenant updating business hours should not acquire any lock on the Appointment stream, and a booking surge should not contend with configuration writes.

---

## Multi-Tenancy Model

Pool multi-tenancy with PostgreSQL Row-Level Security (ADR002). Every entity carries `tenant_id` without exception.

**Three isolation layers:**
1. **JWT validation (api-gateway):** `tid` claim extracted and validated. Requests without a valid `tid` are rejected before reaching any service.
2. **Service layer:** Every service reads `tenant_id` from request context and applies it as a filter on all queries.
3. **PostgreSQL RLS:** `SET LOCAL app.tenant_id = $1` at the start of every transaction. Safe default: null or missing `app.tenant_id` causes RLS to evaluate to `false` — returns zero rows, never all rows (BR005).

**Enterprise isolation:** Dedicated PostgreSQL cluster (db.r7g.large) in addition to RLS policies applied for defence in depth.

**Cache isolation:** Redis cache keys namespaced by `tenant_id` (e.g. `availability:{tenant_id}:{staff_id}:{date}`).

**Event isolation:** All events carry `tenant_id` in their envelope (BR013). SQS FIFO `MessageGroupId = tenant_id` ensures per-tenant ordering without cross-tenant interference.
