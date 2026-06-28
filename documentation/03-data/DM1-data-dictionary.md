---
title: Data Dictionary
layer: 03-data
status: current
lastUpdated: 2026-06-28
---

# DM1 · Data Dictionary

## Overview

This dictionary provides a field-level reference for every entity persisted by AnjiSchedulo. Use it as the authoritative source for field names, types, nullability, PII classification, and constraints when implementing services, writing migrations, or authoring integration tests.

**Conventions used throughout:**

| Convention | Detail |
|---|---|
| **snake_case** | All table and column names are lowercase snake_case |
| **UUID** | All ID fields are UUID v4, stored as `uuid` in PostgreSQL |
| **Timestamps** | All timestamps are UTC, stored as `timestamptz`, serialised as ISO 8601 |
| **PII** | Fields marked `Yes` contain personal data subject to GDPR erasure and pseudonymisation obligations |
| **Encrypted at rest** | All data inherits AES-256 encryption from RDS KMS key; column-level encryption noted where applied |
| **RLS** | Every table is protected by PostgreSQL Row-Level Security; `tenant_id` is present on every tenant-scoped table |

---

## Data Dictionary

### `tenants`

The root aggregate for a business registered on AnjiSchedulo.

| Field | Type | Nullable | PII | Description | Constraints |
|---|---|---|---|---|---|
| `tenant_id` | `uuid` | No | No | Primary key | PK |
| `name` | `varchar(255)` | No | No | Business name | NOT NULL |
| `owner_email` | `varchar(320)` | No | **Yes** | Owner's email address — used for welcome email and data export delivery | UNIQUE per active/provisioning tenant |
| `owner_name` | `varchar(255)` | No | **Yes** | Owner's display name | NOT NULL |
| `plan` | `enum` | No | No | Pricing tier: `starter`, `pro`, `enterprise` | NOT NULL |
| `status` | `enum` | No | No | Lifecycle state: `pending`, `provisioning`, `active`, `suspended`, `deprovisioned` | NOT NULL |
| `stripe_customer_id` | `varchar(100)` | Yes | No | Stripe Customer ID created at registration | NULL until Stripe call succeeds |
| `created_at` | `timestamptz` | No | No | Record creation timestamp | NOT NULL, DEFAULT NOW() |
| `suspended_at` | `timestamptz` | Yes | No | Timestamp of suspension; NULL if not suspended | Set on suspension |
| `deprovisioned_at` | `timestamptz` | Yes | No | Timestamp of deprovisioning | Set on deprovisioning |

---

### `tenant_configs`

Operational configuration parameters for a tenant. Owned by the Tenant aggregate; read by booking-command-service and availability-service.

| Field | Type | Nullable | PII | Description | Constraints |
|---|---|---|---|---|---|
| `tenant_id` | `uuid` | No | No | FK → tenants.tenant_id; also serves as PK | PK, FK |
| `cancellation_hours` | `integer` | No | No | Minimum hours before appointment start for permitted cancellation (BR002) | NOT NULL, DEFAULT 24 |
| `booking_window_days` | `integer` | No | No | Maximum days in advance a slot can be booked (BR010) | NOT NULL, DEFAULT 60 |
| `max_advance_bookings` | `integer` | No | No | Max active bookings per customer at one time (BR008) | NOT NULL, DEFAULT 5 |
| `timezone` | `varchar(64)` | No | No | IANA timezone string, e.g. `Europe/London` | NOT NULL, DEFAULT 'UTC' |
| `currency` | `char(3)` | No | No | ISO 4217 currency code, e.g. `GBP` | NOT NULL, DEFAULT 'GBP' |
| `updated_at` | `timestamptz` | No | No | Last configuration update timestamp | NOT NULL |

---

### `services`

A bookable offering defined by a tenant.

| Field | Type | Nullable | PII | Description | Constraints |
|---|---|---|---|---|---|
| `service_id` | `uuid` | No | No | Primary key | PK |
| `tenant_id` | `uuid` | No | No | FK → tenants.tenant_id | NOT NULL, FK, indexed |
| `name` | `varchar(255)` | No | No | Service name, e.g. "Haircut" | NOT NULL |
| `duration_minutes` | `integer` | No | No | Duration of the service in minutes (minimum 15) | NOT NULL, CHECK >= 15 |
| `price_amount` | `bigint` | No | No | Price in smallest currency unit (pence/cents) to avoid floating-point errors | NOT NULL, CHECK >= 0 |
| `price_currency` | `char(3)` | No | No | ISO 4217 currency code | NOT NULL |
| `active` | `boolean` | No | No | Whether this service is currently bookable | NOT NULL, DEFAULT true |
| `created_at` | `timestamptz` | No | No | Creation timestamp | NOT NULL |
| `updated_at` | `timestamptz` | No | No | Last modification timestamp | NOT NULL |

---

### `staff_members`

An employee or contractor whose availability is managed on the platform (P2).

| Field | Type | Nullable | PII | Description | Constraints |
|---|---|---|---|---|---|
| `staff_id` | `uuid` | No | No | Primary key | PK |
| `tenant_id` | `uuid` | No | No | FK → tenants.tenant_id | NOT NULL, FK, indexed |
| `name` | `varchar(255)` | No | **Yes** | Staff member's full name | NOT NULL |
| `email` | `varchar(320)` | No | **Yes** | Email address for booking notifications | NOT NULL |
| `active` | `boolean` | No | No | Whether this staff member accepts new bookings | NOT NULL, DEFAULT true |
| `created_at` | `timestamptz` | No | No | Creation timestamp | NOT NULL |

---

### `working_hours`

Recurring weekly availability schedule for a staff member.

| Field | Type | Nullable | PII | Description | Constraints |
|---|---|---|---|---|---|
| `working_hours_id` | `uuid` | No | No | Primary key | PK |
| `staff_id` | `uuid` | No | No | FK → staff_members.staff_id | NOT NULL, FK, indexed |
| `tenant_id` | `uuid` | No | No | Denormalised tenant scope for RLS | NOT NULL |
| `day_of_week` | `enum` | No | No | `monday`, `tuesday`, `wednesday`, `thursday`, `friday`, `saturday`, `sunday` | NOT NULL |
| `start_time` | `time` | No | No | Working hours start time (local to tenant timezone) | NOT NULL |
| `end_time` | `time` | No | No | Working hours end time | NOT NULL, CHECK > start_time |
| `effective_from` | `date` | No | No | Date from which this schedule applies | NOT NULL |
| `effective_to` | `date` | Yes | No | Date on which this schedule expires; NULL means indefinite | NULL = no expiry |

---

### `staff_blocks`

A manual period of unavailability set by a staff member or admin.

| Field | Type | Nullable | PII | Description | Constraints |
|---|---|---|---|---|---|
| `block_id` | `uuid` | No | No | Primary key | PK |
| `staff_id` | `uuid` | No | No | FK → staff_members.staff_id | NOT NULL, FK, indexed |
| `tenant_id` | `uuid` | No | No | Denormalised tenant scope for RLS | NOT NULL |
| `block_start` | `timestamptz` | No | No | Start of unavailability period | NOT NULL |
| `block_end` | `timestamptz` | No | No | End of unavailability period | NOT NULL, CHECK > block_start |
| `reason` | `varchar(500)` | Yes | No | Optional reason for the block | NULL if not provided |
| `created_by_role` | `enum` | No | No | Who created the block: `staff`, `admin` | NOT NULL |
| `created_at` | `timestamptz` | No | No | Creation timestamp | NOT NULL |

---

### `customers`

The end customer who books appointments within a tenant's scope. Tenant-scoped — no cross-tenant customer identity in v1.

| Field | Type | Nullable | PII | Description | Constraints |
|---|---|---|---|---|---|
| `customer_id` | `uuid` | No | No | Primary key | PK |
| `tenant_id` | `uuid` | No | No | FK → tenants.tenant_id | NOT NULL, FK, indexed |
| `name` | `varchar(255)` | No | **Yes** | Customer's full name | NOT NULL |
| `email` | `varchar(320)` | No | **Yes** | Customer's email address | NOT NULL, UNIQUE per tenant |
| `phone` | `varchar(30)` | Yes | **Yes** | Customer's phone number (E.164 format) | NULL if not provided |
| `active_booking_count` | `integer` | No | No | Denormalised count of current confirmed bookings; enforces BR008 | NOT NULL, DEFAULT 0, CHECK >= 0 |
| `created_at` | `timestamptz` | No | No | Creation timestamp | NOT NULL |
| `pseudonymised_at` | `timestamptz` | Yes | No | Timestamp of GDPR erasure; NULL if not erased | NULL until erasure |

---

### `appointments`

The core aggregate. Represents a confirmed reservation. State is derived from `appointment_events`, not a status column.

| Field | Type | Nullable | PII | Description | Constraints |
|---|---|---|---|---|---|
| `appointment_id` | `uuid` | No | No | Primary key | PK |
| `tenant_id` | `uuid` | No | No | FK → tenants.tenant_id | NOT NULL, FK, indexed |
| `customer_id` | `uuid` | No | No | FK → customers.customer_id | NOT NULL, FK |
| `staff_id` | `uuid` | No | No | FK → staff_members.staff_id | NOT NULL, FK |
| `service_id` | `uuid` | No | No | FK → services.service_id | NOT NULL, FK |
| `slot_date` | `date` | No | No | Date of the appointment | NOT NULL |
| `slot_start` | `time` | No | No | Start time of the appointment | NOT NULL |
| `slot_end` | `time` | No | No | End time of the appointment | NOT NULL, CHECK > slot_start |
| `idempotency_key` | `uuid` | No | No | Client-provided key mapping to BookingIdempotencyKey (BR006) | UNIQUE |
| `correlation_id` | `uuid` | No | No | Distributed trace correlation ID propagated from API Gateway (NFR014) | NOT NULL |
| `created_at` | `timestamptz` | No | No | Creation timestamp | NOT NULL |

---

### `appointment_events`

Immutable, ordered event records for the Appointment aggregate. The source of truth for appointment state (Event Sourcing).

| Field | Type | Nullable | PII | Description | Constraints |
|---|---|---|---|---|---|
| `event_id` | `uuid` | No | No | Primary key | PK |
| `appointment_id` | `uuid` | No | No | FK → appointments.appointment_id | NOT NULL, FK, indexed |
| `tenant_id` | `uuid` | No | No | Denormalised tenant scope | NOT NULL, indexed |
| `event_type` | `enum` | No | No | `appointment.requested`, `appointment.confirmed`, `appointment.cancelled`, `appointment.rescheduled`, `appointment.completed`, `appointment.failed` | NOT NULL |
| `occurred_at` | `timestamptz` | No | No | Business time the event occurred | NOT NULL |
| `correlation_id` | `uuid` | No | No | Distributed trace correlation ID (NFR014) | NOT NULL |
| `causation_id` | `uuid` | Yes | No | event_id of the event that triggered this event | NULL for originating events |
| `actor_id` | `uuid` | No | No | ID of the actor causing the event (customer_id, staff_id, or system UUID) | NOT NULL |
| `actor_role` | `enum` | No | No | `customer`, `staff`, `admin`, `system` | NOT NULL |
| `payload` | `jsonb` | No | No | Full appointment snapshot at event time (Event-Carried State Transfer) | NOT NULL |

> **INSERT-only.** No UPDATE or DELETE on this table. `booking_app_user` has INSERT and SELECT privileges only.

---

### `slot_locks`

Database-level exclusive reservation on a time slot. Enforces BR001 (no double-booking).

| Field | Type | Nullable | PII | Description | Constraints |
|---|---|---|---|---|---|
| `lock_id` | `uuid` | No | No | Primary key | PK |
| `tenant_id` | `uuid` | No | No | Tenant scope | NOT NULL |
| `staff_id` | `uuid` | No | No | FK → staff_members.staff_id | NOT NULL, FK |
| `slot_date` | `date` | No | No | Date of the locked slot | NOT NULL |
| `slot_start` | `time` | No | No | Start time of the locked slot | NOT NULL |
| `appointment_id` | `uuid` | No | No | FK → appointments.appointment_id | NOT NULL, FK |
| `locked_at` | `timestamptz` | No | No | Timestamp when lock was acquired | NOT NULL |
| `released_at` | `timestamptz` | Yes | No | Timestamp when lock was released; NULL = currently locked | NULL = active lock |

> **Key constraint:** `UNIQUE (tenant_id, staff_id, slot_date, slot_start) WHERE released_at IS NULL` — enforces BR001.

---

### `payments`

Payment record associated with a confirmed appointment.

| Field | Type | Nullable | PII | Description | Constraints |
|---|---|---|---|---|---|
| `payment_id` | `uuid` | No | No | Primary key | PK |
| `appointment_id` | `uuid` | No | No | FK → appointments.appointment_id | NOT NULL, FK |
| `tenant_id` | `uuid` | No | No | Tenant scope for RLS | NOT NULL |
| `stripe_payment_intent_id` | `varchar(100)` | No | No | Stripe PaymentIntent ID | NOT NULL, UNIQUE |
| `amount` | `bigint` | No | No | Charged amount in smallest currency unit | NOT NULL, CHECK > 0 |
| `currency` | `char(3)` | No | No | ISO 4217 currency code | NOT NULL |
| `status` | `enum` | No | No | `pending`, `captured`, `failed`, `refunded`, `refund_failed` | NOT NULL |
| `captured_at` | `timestamptz` | Yes | No | Timestamp of successful capture | NULL until captured |
| `refunded_at` | `timestamptz` | Yes | No | Timestamp of successful refund | NULL until refunded |
| `stripe_refund_id` | `varchar(100)` | Yes | No | Stripe Refund ID | NULL until refunded |
| `created_at` | `timestamptz` | No | No | Creation timestamp | NOT NULL |

> Payment records are retained for a minimum of 7 years (BR012).

---

### `notification_records`

Record of every outbound notification attempt by notification-service.

| Field | Type | Nullable | PII | Description | Constraints |
|---|---|---|---|---|---|
| `notification_id` | `uuid` | No | No | Primary key | PK |
| `tenant_id` | `uuid` | No | No | Tenant scope for RLS | NOT NULL |
| `appointment_id` | `uuid` | Yes | No | FK → appointments.appointment_id | NULL for tenant-level notifications (e.g. welcome email) |
| `recipient_type` | `enum` | No | No | `customer`, `staff`, `admin` | NOT NULL |
| `recipient_contact` | `varchar(320)` | No | **Yes** | Destination email address or phone number | NOT NULL; never written to logs |
| `channel` | `enum` | No | No | `email`, `sms`, `whatsapp` | NOT NULL |
| `template_name` | `varchar(100)` | No | No | Notification template identifier | NOT NULL |
| `status` | `enum` | No | No | `pending`, `sent`, `failed`, `dlq` | NOT NULL |
| `attempt_count` | `integer` | No | No | Number of delivery attempts made | NOT NULL, DEFAULT 0 |
| `last_attempted_at` | `timestamptz` | Yes | No | Timestamp of the most recent attempt | NULL before first attempt |
| `sent_at` | `timestamptz` | Yes | No | Timestamp of successful delivery | NULL until sent |
| `failure_reason` | `varchar(500)` | Yes | No | Error detail from provider on last failure | NULL on success |
| `trigger_event_id` | `uuid` | No | No | event_id of the AppointmentEvent that triggered this notification | NOT NULL |

---

### `tenant_dashboard_views`

CQRS read model — pre-aggregated dashboard data per tenant per day. Served by dashboard-service.

| Field | Type | Nullable | PII | Description | Constraints |
|---|---|---|---|---|---|
| `tenant_id` | `uuid` | No | No | Part of composite PK | PK component |
| `view_date` | `date` | No | No | Date this row represents | PK component |
| `confirmed_count` | `integer` | No | No | Confirmed appointments on this date | NOT NULL, DEFAULT 0 |
| `cancelled_count` | `integer` | No | No | Cancelled appointments on this date | NOT NULL, DEFAULT 0 |
| `rescheduled_count` | `integer` | No | No | Rescheduled appointments on this date | NOT NULL, DEFAULT 0 |
| `net_revenue` | `bigint` | No | No | Net revenue in smallest currency unit (confirmed minus refunded) | NOT NULL, DEFAULT 0 |
| `cancellation_rate` | `numeric(5,4)` | No | No | Ratio of cancellations to total bookings (0.0–1.0) | NOT NULL, DEFAULT 0 |
| `peak_hour` | `smallint` | Yes | No | Hour of day (0–23) with highest booking volume | NULL if no data |
| `last_updated_at` | `timestamptz` | No | No | Timestamp of last incremental update | NOT NULL |
| `last_event_id` | `uuid` | Yes | No | Watermark — event_id of the last processed AppointmentEvent; enables idempotent updates (NFR009) | NULL until first event |

> Composite PK: `(tenant_id, view_date)`. RLS enforced: queries scoped to `tenant_id` from JWT.

---

### `analytics_summaries`

Read model maintained by analytics-service via Kinesis stream processing.

| Field | Type | Nullable | PII | Description | Constraints |
|---|---|---|---|---|---|
| `tenant_id` | `uuid` | No | No | Tenant scope | PK component |
| `window_start` | `timestamptz` | No | No | Start of the aggregation window | PK component |
| `window_end` | `timestamptz` | No | No | End of the aggregation window | NOT NULL |
| `booking_count` | `integer` | No | No | Bookings in this window | NOT NULL |
| `cancellation_count` | `integer` | No | No | Cancellations in this window | NOT NULL |
| `velocity_ratio` | `numeric(8,4)` | No | No | Current 5-min rate vs historical average; alert if > 3.0 | NOT NULL |
| `alert_triggered` | `boolean` | No | No | Whether an anomaly alert was fired for this window | NOT NULL, DEFAULT false |
| `computed_at` | `timestamptz` | No | No | Timestamp when this row was computed | NOT NULL |

---

### `outbox_records`

Infrastructure entity. Written in the same DB transaction as the domain event it carries; read and published by outbox-relay.

| Field | Type | Nullable | PII | Description | Constraints |
|---|---|---|---|---|---|
| `outbox_id` | `uuid` | No | No | Primary key | PK |
| `tenant_id` | `uuid` | No | No | Tenant scope for relay filtering | NOT NULL |
| `topic` | `varchar(200)` | No | No | SNS topic name to publish to | NOT NULL |
| `event_type` | `varchar(100)` | No | No | Event type string, e.g. `appointment.confirmed` | NOT NULL |
| `payload` | `jsonb` | No | No | Full event envelope JSON | NOT NULL |
| `created_at` | `timestamptz` | No | No | Creation timestamp | NOT NULL |
| `published_at` | `timestamptz` | Yes | No | Timestamp when relay successfully published to SNS; NULL = not yet published | NULL = pending |

> Indexed on `(published_at IS NULL)` for efficient relay polling.

---

### `inbox_records`

Infrastructure entity for consumer-side deduplication under at-least-once SQS delivery.

| Field | Type | Nullable | PII | Description | Constraints |
|---|---|---|---|---|---|
| `event_id` | `uuid` | No | No | event_id from the event envelope | PK component |
| `consumer_name` | `varchar(100)` | No | No | Name of the consuming service | PK component |
| `processed_at` | `timestamptz` | No | No | Timestamp of first successful processing | NOT NULL |
| `result` | `varchar(500)` | Yes | No | Optional processing outcome summary | NULL if not recorded |

> Composite PK: `(event_id, consumer_name)`. Unique constraint means duplicate event_id for the same consumer fails the INSERT → event is skipped (NFR009).

---

### `booking_idempotency_keys`

Infrastructure entity that prevents duplicate booking operations from network retries (BR006).

| Field | Type | Nullable | PII | Description | Constraints |
|---|---|---|---|---|---|
| `idempotency_key` | `uuid` | No | No | Client-provided key; primary key | PK |
| `tenant_id` | `uuid` | No | No | Tenant scope | NOT NULL |
| `appointment_id` | `uuid` | Yes | No | Set on successful booking | NULL on failure |
| `result_status` | `enum` | No | No | `confirmed` or `failed` | NOT NULL |
| `result_payload` | `jsonb` | No | No | Full HTTP response returned to client on first request | NOT NULL |
| `created_at` | `timestamptz` | No | No | Creation timestamp | NOT NULL |
| `expires_at` | `timestamptz` | No | No | TTL — 24 hours after creation; expired keys purged by scheduled job | NOT NULL |

---

### `audit_log`

Immutable record of every write operation on tenant-scoped data (BR014).

| Field | Type | Nullable | PII | Description | Constraints |
|---|---|---|---|---|---|
| `audit_id` | `uuid` | No | No | Primary key | PK |
| `tenant_id` | `uuid` | No | No | Tenant scope | NOT NULL |
| `entity_type` | `varchar(100)` | No | No | Name of the modified entity, e.g. `Appointment`, `Tenant` | NOT NULL |
| `entity_id` | `uuid` | No | No | ID of the modified entity | NOT NULL |
| `operation` | `enum` | No | No | `create`, `update`, `cancel`, `reschedule`, `suspend`, `deprovision`, `export`, `replay` | NOT NULL |
| `actor_id` | `uuid` | No | No | ID of the actor performing the operation | NOT NULL |
| `actor_role` | `enum` | No | No | `customer`, `staff`, `admin`, `operator`, `system` | NOT NULL |
| `occurred_at` | `timestamptz` | No | No | Business time of the operation | NOT NULL |
| `before_snapshot` | `jsonb` | Yes | No | Entity state before the change | NULL for create operations |
| `after_snapshot` | `jsonb` | No | No | Entity state after the change | NOT NULL |

> **INSERT-only.** `booking_app_user` has INSERT and SELECT privileges only — no UPDATE or DELETE. Retained permanently (BR012, BR014).

---

### `replay_jobs`

Tracks the lifecycle of a tenant event replay operation triggered by a Platform Operator (FR011).

| Field | Type | Nullable | PII | Description | Constraints |
|---|---|---|---|---|---|
| `job_id` | `uuid` | No | No | Primary key | PK |
| `tenant_id` | `uuid` | No | No | Tenant whose events are being replayed | NOT NULL |
| `initiated_by` | `uuid` | No | No | operator user_id of the person who triggered the replay | NOT NULL |
| `scope_type` | `enum` | No | No | `full_tenant`, `date_range`, `event_type`, `specific_ids` | NOT NULL |
| `scope_params` | `jsonb` | No | No | Scope parameters, e.g. `{"from": "2026-01-01", "to": "2026-06-01"}` | NOT NULL |
| `status` | `enum` | No | No | `queued`, `running`, `completed`, `failed`, `cancelled` | NOT NULL |
| `events_published` | `integer` | No | No | Running count of events published during replay | NOT NULL, DEFAULT 0 |
| `started_at` | `timestamptz` | Yes | No | Timestamp when replay began processing | NULL until started |
| `completed_at` | `timestamptz` | Yes | No | Timestamp when replay finished | NULL until completed |
| `error_message` | `varchar(2000)` | Yes | No | Error detail if status = failed | NULL on success |

---

## PII Field Registry

| Entity | Field | PII Category | Handling Requirement |
|---|---|---|---|
| `tenants` | `owner_email` | Contact data | Pseudonymised when tenant is deprovisioned |
| `tenants` | `owner_name` | Contact data | Pseudonymised when tenant is deprovisioned |
| `staff_members` | `name` | Contact data | Pseudonymised on GDPR erasure request |
| `staff_members` | `email` | Contact data | Pseudonymised on GDPR erasure request |
| `customers` | `name` | Contact data | Replaced with `"Anonymised Customer"` on erasure; `pseudonymised_at` set |
| `customers` | `email` | Contact data | Replaced with `{uuid}@erasure.internal` on erasure |
| `customers` | `phone` | Contact data | Set to NULL on erasure |
| `notification_records` | `recipient_contact` | Contact data | Never written to logs (Fluent Bit redaction rule); retained in DB for delivery tracking only |

> Audit log entries created before a GDPR erasure retain the `customer_id` FK. The PII is removed from the `customers` table but the appointment history structure is preserved.

---

## Enum Reference

| Enum | Entity / Field | Allowed Values |
|---|---|---|
| `tenant.status` | `tenants.status` | `pending`, `provisioning`, `active`, `suspended`, `deprovisioned` |
| `tenant.plan` | `tenants.plan` | `starter`, `pro`, `enterprise` |
| `appointment.event_type` | `appointment_events.event_type` | `appointment.requested`, `appointment.confirmed`, `appointment.cancelled`, `appointment.rescheduled`, `appointment.completed`, `appointment.failed` |
| `actor.role` | `appointment_events.actor_role`, `audit_log.actor_role` | `customer`, `staff`, `admin`, `operator`, `system` |
| `payment.status` | `payments.status` | `pending`, `captured`, `failed`, `refunded`, `refund_failed` |
| `notification.status` | `notification_records.status` | `pending`, `sent`, `failed`, `dlq` |
| `notification.recipient_type` | `notification_records.recipient_type` | `customer`, `staff`, `admin` |
| `notification.channel` | `notification_records.channel` | `email`, `sms`, `whatsapp` |
| `day_of_week` | `working_hours.day_of_week` | `monday`, `tuesday`, `wednesday`, `thursday`, `friday`, `saturday`, `sunday` |
| `staff_block.created_by_role` | `staff_blocks.created_by_role` | `staff`, `admin` |
| `replay_job.status` | `replay_jobs.status` | `queued`, `running`, `completed`, `failed`, `cancelled` |
| `audit.operation` | `audit_log.operation` | `create`, `update`, `cancel`, `reschedule`, `suspend`, `deprovision`, `export`, `replay` |

---

## Naming Conventions

| Convention | Example | Rule |
|---|---|---|
| Table names | `appointment_events`, `slot_locks` | Plural snake_case |
| Primary keys | `tenant_id`, `appointment_id` | `{entity_singular}_id` |
| Foreign keys | `tenant_id`, `staff_id` | Match the referenced table's PK exactly |
| Timestamps | `created_at`, `published_at`, `released_at` | Always end with `_at` suffix; always `timestamptz` |
| Booleans | `active`, `alert_triggered` | Positive adjective without `is_` prefix |
| JSONB payload fields | `payload`, `scope_params`, `before_snapshot` | Descriptive noun; never `data` or `json` |
| Monetary amounts | `price_amount`, `amount`, `net_revenue` | Always stored in smallest currency unit (pence/cents) as `bigint` |
| Soft-delete indicator | `pseudonymised_at` | Nullable timestamp; NULL = not erased; timestamp = erased at that time |
