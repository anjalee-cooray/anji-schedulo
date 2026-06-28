---
title: Functional Specification
layer: 02-design
status: current
lastUpdated: 2026-06-28
sources:
  - specs/user/functional-requirements.json
  - specs/user/business-rules.json
  - specs/user/personas.json
  - specs/user/user-journeys.json
  - specs/ai/bounded-contexts.json
  - specs/ai/events.json
---

# D6 · Functional Specification

## 1. Overview

AnjiSchedulo is a multi-tenant SaaS appointment scheduling platform. This document specifies the exact behaviour of each functional capability, the rules the system enforces, and the expected inputs, outputs, and error conditions. It bridges the business requirements in `R3-brd.md` with the technical design in the architecture layer.

**System Boundary:** AnjiSchedulo manages the full lifecycle of appointments between a tenant's customers and staff members — from tenant self-service onboarding through booking, rescheduling, cancellation, and deprovisioning. Out of scope for v1: POS systems, inventory management, video conferencing, native mobile applications, and public API access for third-party integrations.

**Actors:**

| Actor | Persona ID | Description |
|---|---|---|
| Tenant Admin | P1 | Business owner who registers the tenant and manages configuration |
| Staff Member | P2 | Staff who view their schedules and block unavailability |
| End Customer | P3 | Customer who browses slots and books/cancels/reschedules appointments |
| Platform Operator | P4 | Infrastructure operator who manages DLQ events, replays, and tenant lifecycle |
| System | — | Automated processes: outbox-relay, scheduled jobs, Kinesis stream processor |

---

## 2. Feature Specifications

---

### FR001 — Tenant Self-Service Onboarding

**Related Personas:** P1  
**Related BRs:** BR005, BR013, BR014  
**Related UJ:** UJ001

**Capability Description:**

A new business owner registers their organisation on AnjiSchedulo by submitting a form with their business name, owner email, and chosen pricing plan. The platform provisions the tenant account automatically without operator intervention and delivers a welcome email to the registered address. Provisioning is asynchronous: registration returns `202 Accepted` immediately with the tenant in `provisioning` status. Billing activation and welcome email proceed independently as async subscribers to `tenant.provisioned`. The tenant transitions to `active` only after billing confirms. Full provisioning — from registration submission to accepting live bookings — targets completion within 30 minutes.

**Functional Rules:**

1. The registration form requires: `business_name` (required, ≤ 200 chars), `owner_email` (valid RFC 5322 format), `owner_name` (required), `plan` (one of `starter | pro | enterprise`).
2. If an active or provisioning tenant already exists with the same `owner_email`, the registration is rejected with `409 Conflict` — error code `DUPLICATE_REGISTRATION`.
3. A new `tenant_id` (UUID v4) is generated at registration time and applied to all provisioned records.
4. PostgreSQL Row-Level Security is established from the moment the tenant row is created (BR005).
5. The `tenant.provisioned` event must be emitted containing: `tenant_id`, `event_id`, `correlation_id`, `occurred_at` (BR013).
6. An immutable `AuditLog` entry is created for the registration in the same database transaction (BR014).
7. Tenant status machine: `pending → provisioning → active`. The transition to `active` occurs only after `billing.subscription_activated` is received.
8. Welcome email delivery failure must not affect tenant account activation or the admin's ability to access the platform (BR007).

**Input / Output:**

- **Input:** `POST /api/v1/tenants` — `{ business_name, owner_email, owner_name, plan }`
- **Output (success):** `202 Accepted` — `{ tenant_id, status: "provisioning", correlation_id }`
- **Output (duplicate email):** `409 Conflict` — `{ error: { code: "DUPLICATE_REGISTRATION", ... } }`
- **Output (validation error):** `422 Unprocessable Entity` — `{ error: { code: "VALIDATION_ERROR", fields: [...] } }`

**Error Conditions:**

| Condition | HTTP Status | Error Code |
|---|---|---|
| Email already registered (active or provisioning) | 409 | DUPLICATE_REGISTRATION |
| Missing required field | 422 | VALIDATION_ERROR |
| Invalid plan value | 422 | VALIDATION_ERROR |
| Stripe unavailable during registration | 503 | SERVICE_UNAVAILABLE |

**Integration Points:** tenant-service (owner), notification-service (welcome email subscriber), billing-service (subscription activation subscriber), outbox-relay (event publication).  
**Events Emitted:** `tenant.provisioned`, `tenant.configured` (after billing confirms and admin completes setup).

---

### FR002 — Tenant Configuration Management

**Related Personas:** P1  
**Related BRs:** BR002, BR003, BR008, BR010, BR014  
**Related UJ:** UJ001

**Capability Description:**

After registration, a tenant admin configures all operational parameters for their business: business hours (per staff member per day), the service catalogue, staff roster, and booking policy settings. Configuration changes take effect immediately for new bookings and do not affect already-confirmed appointments.

**Functional Rules:**

1. Business hours are set per staff member per day of the week (open/closed + start_time + end_time).
2. A staff member marked closed on a given day has no available slots that day (BR003).
3. Services require: name (required), duration_minutes (integer, minimum 15), price_amount (integer, smallest currency unit), price_currency.
4. Plan tier service limits: Starter — max 5 services, max 3 staff; Pro — unlimited services, max 20 staff; Enterprise — unlimited services and staff.
5. Deactivating a service or staff member removes them from the booking page but does not cancel or modify existing confirmed appointments.
6. Cancellation policy is stored in `tenant_config.cancellation_hours` (integer hours ≥ 0) and enforced on every cancellation (BR002).
7. Booking advance window is stored in `tenant_config.booking_window_days` and enforced by availability-service (BR010).
8. Maximum advance bookings is stored in `tenant_config.max_advance_bookings` and validated per customer on each booking attempt (BR008).
9. Every configuration change produces an immutable `AuditLog` entry with before/after snapshot (BR014).
10. A `tenant.configured` event is published after each change so booking-command-service and availability-service can refresh their caches.

**Input / Output:**

- **Business hours:** `PUT /api/v1/tenants/{tenant_id}/working-hours` — `{ staff_id, day_of_week, start_time, end_time }`
- **Services:** `POST /api/v1/tenants/{tenant_id}/services` — `{ name, duration_minutes, price_amount, price_currency }`
- **Staff:** `POST /api/v1/tenants/{tenant_id}/staff` — `{ name, email, service_ids[] }`
- **Policy:** `PUT /api/v1/tenants/{tenant_id}/config` — `{ cancellation_hours, booking_window_days, max_advance_bookings }`

**Error Conditions:**

| Condition | HTTP Status | Error Code |
|---|---|---|
| Service limit exceeded for plan | 422 | PLAN_LIMIT_EXCEEDED |
| Staff limit exceeded for plan | 422 | PLAN_LIMIT_EXCEEDED |
| Duration below 15 minutes | 422 | VALIDATION_ERROR |
| Tenant suspended | 403 | TENANT_SUSPENDED |
| Tenant not found | 404 | TENANT_NOT_FOUND |

**Events Emitted:** `tenant.configured`.

---

### FR003 — Customer Slot Browsing

**Related Personas:** P3  
**Related BRs:** BR003, BR010  
**Related UJ:** UJ002

**Capability Description:**

A customer browses available appointment slots for a tenant's staff members and services without requiring authentication. The availability-service computes open slots by combining staff working hours with existing confirmed slot locks and manual blocks. Results are cached in Redis (TTL: 30 seconds) for performance.

**Functional Rules:**

1. The endpoint is public (unauthenticated). Tenant context is derived from the URL slug.
2. Available slots exclude: confirmed bookings (active SlotLocks), staff blocks, and times outside working hours (BR003).
3. Slots beyond `tenant_config.booking_window_days` in the future are not returned (BR010).
4. Response time p95 < 500ms (NFR005).
5. Cache key: `availability:{tenant_id}:{staff_id}:{date}`. TTL: 30 seconds.
6. Cache is invalidated on `appointment.confirmed` and `appointment.cancelled` events.
7. If `staff_id` is omitted, the response returns the union of available slots across all active staff for the service.

**Input / Output:**

- **Input:** `GET /tenants/{tenantSlug}/availability?service_id=&date=&staff_id=` (staff_id optional)
- **Output (success):** `200 OK` — `{ slots: [{ staff_id, staff_name, date, start_time, end_time }] }`
- **Output (no slots available):** `200 OK` — `{ slots: [] }`

**Error Conditions:**

| Condition | HTTP Status | Error Code |
|---|---|---|
| Tenant not found | 404 | TENANT_NOT_FOUND |
| Tenant suspended | 403 | TENANT_SUSPENDED |
| Invalid date format | 422 | VALIDATION_ERROR |
| Service not found | 404 | SERVICE_NOT_FOUND |

---

### FR004 — Appointment Booking

**Related Personas:** P3  
**Related BRs:** BR001, BR004, BR006, BR007, BR008  
**Related UJ:** UJ002

**Capability Description:**

A customer books an appointment by submitting a request with a client-provided idempotency key, service, staff member, slot details, and contact information. No account is required — the customer is identified by email within the tenant scope. The booking flows through a synchronous saga orchestrated by booking-command-service: slot reservation → payment capture (if required) → confirmation. Each step compensates if the next fails.

**Functional Rules:**

1. The request must include a client-provided `idempotency_key` (UUID v4). If the same key has been seen before, the stored result is returned without reprocessing (BR006).
2. A unique constraint on `slot_locks (tenant_id, staff_id, slot_date, slot_start) WHERE released_at IS NULL` prevents double-booking (BR001). Constraint violation returns 409.
3. If the customer's active booking count is at or above `tenant_config.max_advance_bookings`, the booking is rejected with 422 — error code `BOOKING_LIMIT_EXCEEDED` (BR008).
4. If payment is required and the charge fails, the slot lock is released and the booking fails with 402 — error code `PAYMENT_FAILED` (BR004).
5. If payment-service circuit breaker is open, the saga returns 503 with `Retry-After: 30` and releases any slot lock.
6. The booking saga targets completion within 10 seconds. After 10 seconds without resolution, a saga timeout alert fires.
7. Customer receives a confirmation notification within 30 seconds of booking (FR007). Notification failure does not affect confirmation (BR007).
8. Every booking attempt (success and failure) produces an immutable `AuditLog` entry (BR014).

**Input / Output:**

- **Input:** `POST /tenants/{tenantSlug}/bookings` — `{ idempotency_key, service_id, staff_id, slot_date, slot_start, customer_name, customer_email, customer_phone? }`
- **Output (success):** `201 Created` — `{ appointment_id, status: "confirmed", slot_date, slot_start, slot_end, staff_name, service_name, correlation_id }`
- **Output (slot unavailable):** `409 Conflict` — `{ error: { code: "SLOT_UNAVAILABLE", ... } }`
- **Output (payment failed):** `402 Payment Required` — `{ error: { code: "PAYMENT_FAILED", ... } }`

**Error Conditions:**

| Condition | HTTP Status | Error Code |
|---|---|---|
| Slot already taken (concurrent booking) | 409 | SLOT_UNAVAILABLE |
| Booking limit reached | 422 | BOOKING_LIMIT_EXCEEDED |
| Booking window exceeded | 422 | BOOKING_WINDOW_EXCEEDED |
| Payment declined | 402 | PAYMENT_FAILED |
| Payment service unavailable | 503 | PAYMENT_SERVICE_UNAVAILABLE |
| Tenant suspended | 403 | TENANT_SUSPENDED |
| Invalid idempotency key format | 422 | VALIDATION_ERROR |

**Integration Points:** booking-command-service (orchestrator), availability-service (slot check), payment-service (charge), notification-service (confirmation), outbox-relay.  
**Events Emitted:** `appointment.requested`, `appointment.confirmed`, `appointment.failed`.

---

### FR005 — Appointment Cancellation

**Related Personas:** P2, P3  
**Related BRs:** BR002, BR009  
**Related UJ:** UJ003

**Capability Description:**

A customer or staff member cancels a confirmed appointment if the cancellation falls within the tenant's configured policy window. The cancellation releases the slot immediately, initiates a refund if applicable, and notifies all parties. Refund failure never blocks the slot from being re-booked (BR009).

**Functional Rules:**

1. Cancellation is only permitted if `NOW() < slot_datetime - tenant_config.cancellation_hours` (BR002). Out-of-window cancellations are rejected with 422.
2. The slot lock is released in the same database transaction as the `AppointmentEvent(cancelled)` write. The slot becomes available for new bookings immediately on commit.
3. If a payment was captured, a refund is initiated asynchronously as a subscriber to `appointment.cancelled`. Refund failure routes to DLQ without blocking the slot release (BR009).
4. Customer and staff are notified of the cancellation via notification-service (async subscriber). Notification failure does not block slot release (BR007).
5. All cancellation history is preserved in the immutable `AppointmentEvent` stream (BR014).

**Timing Targets:**
- Slot released: within 10 seconds
- Customer notified: within 30 seconds
- Refund initiated: within 60 seconds

**Input / Output:**

- **Input:** `POST /appointments/{appointment_id}/cancel` (JWT required: customer who owns, assigned staff, or tenant_admin)
- **Output (success):** `200 OK` — `{ appointment_id, status: "cancelled", cancelled_at, refund_status: "initiated" | "not_applicable" }`
- **Output (outside window):** `422 Unprocessable Entity` — `{ error: { code: "CANCELLATION_WINDOW_EXCEEDED", ... } }`

**Error Conditions:**

| Condition | HTTP Status | Error Code |
|---|---|---|
| Cancellation outside policy window | 422 | CANCELLATION_WINDOW_EXCEEDED |
| Appointment not in cancellable state | 409 | APPOINTMENT_NOT_CANCELLABLE |
| Insufficient permissions | 403 | FORBIDDEN |
| Appointment not found | 404 | APPOINTMENT_NOT_FOUND |

**Events Emitted:** `appointment.cancelled`.

---

### FR006 — Appointment Rescheduling

**Related Personas:** P3  
**Related BRs:** BR001, BR004  
**Related UJ:** UJ003

**Capability Description:**

A customer moves their confirmed appointment to a different available slot. The system uses a sequential saga: the new slot is locked first; if that succeeds, the original slot is released. This guarantees the customer never loses both slots — if the new slot is unavailable, the original booking is completely unchanged.

**Functional Rules:**

1. The new slot must be available (unique constraint on slot_locks — BR001). If unavailable, 409 is returned and the original booking is unchanged.
2. The new slot must be within the booking window (BR010).
3. The new slot must be within the staff member's working hours (BR003).
4. The new slot lock and old slot lock release occur in the same database transaction — atomic swap.
5. Customer receives a rescheduling confirmation notification (notification-service, async).
6. Rescheduling history is recorded in the AppointmentEvent stream.

**Input / Output:**

- **Input:** `POST /appointments/{appointment_id}/reschedule` — `{ new_slot_date, new_slot_start }`
- **Output (success):** `200 OK` — `{ appointment_id, status: "confirmed", old_slot, new_slot }`
- **Output (new slot unavailable):** `409 Conflict` — `{ error: { code: "SLOT_UNAVAILABLE", ... } }`

**Error Conditions:**

| Condition | HTTP Status | Error Code |
|---|---|---|
| New slot unavailable | 409 | SLOT_UNAVAILABLE |
| New slot outside booking window | 422 | BOOKING_WINDOW_EXCEEDED |
| New slot outside working hours | 422 | SLOT_OUTSIDE_WORKING_HOURS |
| Appointment not in reschedulable state | 409 | APPOINTMENT_NOT_RESCHEDULABLE |

**Events Emitted:** `appointment.rescheduled`.

---

### FR007 — Lifecycle Notifications

**Related Personas:** P2, P3  
**Related BRs:** BR007, BR013  
**Related UJ:** UJ002, UJ003

**Capability Description:**

Customers and staff receive automated notifications at every significant appointment lifecycle event. All notification content is derived from the event payload snapshot — notification-service never makes back-calls to other services (Event-Carried State Transfer). Delivery failures are retried with exponential backoff and routed to DLQ after exhaustion, isolated from booking operations.

**Functional Rules:**

1. Notifications trigger at: booking confirmed, booking failed (optional), booking cancelled, booking rescheduled.
2. Content is derived entirely from the event payload — no back-call to booking-command-service.
3. Failed deliveries are retried up to 4 times with exponential backoff (delays: 1s, 2s, 4s, 8s).
4. After 4 failures, `NotificationRecord.status = dlq` and `notification.failed` event is published to ops-service.
5. Notification failures never delay, block, or roll back the triggering booking operation (BR007).
6. Channel selection by plan tier: Starter — email only; Pro — email + SMS; Enterprise — email + SMS + WhatsApp.
7. Events consumed by notification-service are deduplicated via InboxRecord (idempotent consumer pattern).

**Events Consumed:** `appointment.confirmed`, `appointment.cancelled`, `appointment.rescheduled`, `tenant.provisioned`.  
**Events Emitted:** `notification.sent`, `notification.failed`.

---

### FR008 — Tenant Operational Dashboard

**Related Personas:** P1  
**Related UJ:** UJ005

**Capability Description:**

A tenant admin views a real-time operational summary of their business: today's confirmed bookings, cancellation rate, net revenue, peak hour, and upcoming schedule. The dashboard is served from a pre-aggregated `TenantDashboardView` materialised view, enabling O(1) reads with p95 latency < 200ms.

**Functional Rules:**

1. Dashboard response time p95 < 200ms — served from a single pre-aggregated row in `tenant_dashboard_views` (NFR006).
2. Dashboard data must reflect events within 5 seconds under normal load (NFR008).
3. If the view is corrupted or stale, a Platform Operator can trigger a replay to rebuild from the event log (FR011).
4. The dashboard shows: `confirmed_count`, `cancelled_count`, `rescheduled_count`, `net_revenue`, `cancellation_rate`, `peak_hour`, `last_updated_at`.
5. All dashboard data is scoped strictly to the requesting tenant via RLS and JWT `tid` claim.

**Input / Output:**

- **Input:** `GET /dashboard` (JWT required: role = tenant_admin)
- **Output:** `200 OK` — `{ confirmed_count, cancelled_count, rescheduled_count, net_revenue, cancellation_rate, peak_hour, last_updated_at }`

**Events Consumed:** `appointment.confirmed`, `appointment.cancelled`, `appointment.rescheduled`, `appointment.completed`.

---

### FR009 — Stream Processing Alerts

**Related Personas:** P1, P4  
**Related UJ:** UJ005

**Capability Description:**

The platform continuously monitors per-tenant booking patterns via Kinesis stream processing (v1.0). Two anomaly types trigger alerts: booking velocity spikes (> 3× historical average in a 5-minute window) and elevated cancellation rates (> 30% over the last 50 bookings). Alerts are delivered within 60 seconds of detection.

**Functional Rules:**

1. Velocity spike: detected when `current_5min_bookings / historical_avg_5min_bookings > 3.0`.
2. Cancellation rate alert: detected when `cancelled_in_last_50 / 50 > 0.30`.
3. Alerts delivered within 60 seconds of threshold breach.
4. Alert events — `analytics.velocity_spike_detected`, `analytics.cancellation_rate_alert` — consumed by notification-service and ops-service.

**Events Emitted:** `analytics.velocity_spike_detected`, `analytics.cancellation_rate_alert`.

---

### FR010 — Dead Letter Queue Triage

**Related Personas:** P4

**Capability Description:**

Platform Operators inspect, correct, and reprocess failed events via the ops console. Events are routed to DLQ after 4 failed processing attempts. An alert fires within 5 minutes of any DLQ message arriving.

**Functional Rules:**

1. DLQ alert fires within 5 minutes of any message arriving in any DLQ (NFR016).
2. Ops console shows DLQ entries with full payload, filterable by `tenant_id`, `event_type`, `failure_reason`.
3. Operator can re-queue with optional payload correction, or discard with documented reason.
4. Every operator action (re-queue or discard) produces an immutable `AuditLog` entry (BR014).
5. Re-queued events are safe because all consumers implement InboxRecord deduplication (NFR009).

---

### FR011 — Tenant Event Replay

**Related Personas:** P4

**Capability Description:**

Platform Operators replay historical events for a tenant from the `appointment_events` table to rebuild derived views after a bug fix, data migration, or consumer outage. Live booking operations continue unaffected during replay.

**Functional Rules:**

1. Replay scope options: `full_tenant`, `date_range`, `event_type`, `specific_ids`.
2. Replayed events carry an additional `replay_job_id` envelope field for consumer tracing.
3. All consumers process replayed events idempotently via InboxRecord deduplication (NFR009).
4. Replay rate limited to 500 events/second to protect consumer throughput.
5. ReplayJob status lifecycle: `queued → running → completed | failed | cancelled`.

---

### FR012 — Tenant Data Export

**Related Personas:** P1  
**Related BRs:** BR005, BR014  
**Related UJ:** UJ007

**Capability Description:**

A tenant admin requests a full structured export of all their data. The export includes all configuration, staff, services, appointments, events, payments, notifications, and audit log, scoped strictly to the requesting tenant. A signed S3 URL is delivered to the verified owner email only.

**Functional Rules:**

1. Export is scoped by `tenant_id` from the authenticated JWT — no cross-tenant access (BR005).
2. Signed URL expires after 24 hours (FR012).
3. Archive delivered to `owner_email` only.
4. Export request returns `202 Accepted` immediately; archive generation and email delivery are async.
5. Export generates an `AuditLog` entry (BR014).

**Input / Output:**

- **Input:** `POST /api/v1/tenants/{tenant_id}/export` (JWT required: role = tenant_admin)
- **Output:** `202 Accepted` — `{ export_id, status: "queued", estimated_completion_minutes: 5 }`

---

### FR013 — Tenant Suspension and Deprovisioning

**Related Personas:** P4  
**Related BRs:** BR011, BR012  
**Related UJ:** UJ007

**Capability Description:**

Platform Operators suspend a tenant (halting new bookings while preserving read access for 30 days) and deprovision them after the 30-day window. Data is retained for 90 days post-deprovisioning before hard deletion (with exceptions for payment records and audit log).

**Functional Rules:**

1. Suspension immediately halts new booking acceptance by publishing `tenant.suspended`.
2. Suspended tenants retain read-only access for exactly 30 days (BR011).
3. Deprovisioning requires: (a) tenant status is `suspended`, and (b) `NOW() >= suspended_at + 30 days`. Otherwise 422 is returned with `days_remaining`.
4. After deprovisioning: `hard_deletion_at = deprovisioned_at + 90 days` (BR012).
5. Payment records retained 7 years; audit log retained permanently (BR012 exceptions).
6. Customer PII pseudonymised on erasure: name → `"Anonymised Customer"`, email → `{uuid}@erasure.internal`, phone → `null`, `pseudonymised_at` set.

**Events Emitted:** `tenant.suspended`, `tenant.deprovisioned`.

---

## 3. Cross-Cutting Functional Requirements

### 3.1 Tenant Context Enforcement

Every authenticated API request carries a JWT with a `tid` (tenant_id) claim. The api-gateway validates the JWT and injects `tenant_id` into downstream request context. All service-layer queries execute `SET LOCAL app.tenant_id = $tenant_id` at transaction start. PostgreSQL RLS enforces isolation at the database layer as a safety net (BR005, NFR012).

### 3.2 Idempotency (BR006)

All booking creation endpoints require a client-provided `idempotency_key` (UUID v4). booking-command-service checks `booking_idempotency_keys` before any processing: if found, the stored result is returned immediately without reprocessing. Keys expire after 24 hours.

### 3.3 Audit Trail (BR014)

Every write operation modifying tenant-scoped data produces an immutable `AuditLog` entry in the same database transaction. The `booking_app_user` database role has INSERT-only privileges on `audit_log` — UPDATE and DELETE are denied at the database level.

### 3.4 Event Envelope Validation (BR013)

Every event published to SNS must include: `tenant_id`, `event_id`, `correlation_id`, `occurred_at`. The outbox-relay validates the envelope before publishing. Consumer services validate the envelope before processing and route to DLQ if any required field is missing.

### 3.5 Notification Isolation (BR007)

Notification delivery is handled by a fully decoupled async subscriber (notification-service) via SQS. Delivery failure retries are isolated to the notification pipeline. A notification failure never blocks, delays, or rolls back the booking operation that triggered it.

---

## 4. Data Validation Rules

| Field | Rule | Error Code |
|---|---|---|
| `owner_email` | Valid RFC 5322 format | VALIDATION_ERROR |
| `plan` | One of: `starter`, `pro`, `enterprise` | VALIDATION_ERROR |
| `business_name` | Required, ≤ 200 characters | VALIDATION_ERROR |
| `service.duration_minutes` | Integer ≥ 15 | VALIDATION_ERROR |
| `service.price_amount` | Integer ≥ 0 (smallest currency unit) | VALIDATION_ERROR |
| `slot_date` | ISO 8601 date, not in the past | VALIDATION_ERROR |
| `slot_start` | ISO 8601 time, within working hours | SLOT_OUTSIDE_WORKING_HOURS |
| `idempotency_key` | UUID v4 format | VALIDATION_ERROR |
| `customer_email` | Valid RFC 5322 format | VALIDATION_ERROR |
| `cancellation_hours` | Integer ≥ 0 | VALIDATION_ERROR |
| `booking_window_days` | Integer ≥ 1, ≤ 365 | VALIDATION_ERROR |
| `max_advance_bookings` | Integer ≥ 1 | VALIDATION_ERROR |
| `new_slot_date` (reschedule) | ISO 8601 date, within booking window | BOOKING_WINDOW_EXCEEDED |
| `day_of_week` | One of: `monday`–`sunday` | VALIDATION_ERROR |
| `start_time` | HH:MM format, before `end_time` | VALIDATION_ERROR |

---

## 5. Business Rules Summary

| BR ID | Rule | Gated Feature |
|---|---|---|
| BR001 | No slot can be double-booked | FR004 (slot lock unique constraint) |
| BR002 | Cancellation only within policy window | FR005 |
| BR003 | Staff cannot be booked outside working hours | FR003, FR002 |
| BR004 | Booking saga is atomic — all succeed or all compensate | FR004, FR006 |
| BR005 | Tenant data never readable by another tenant | All features |
| BR006 | Duplicate idempotency key returns stored result | FR004 |
| BR007 | Notification failure never affects booking availability | FR007 |
| BR008 | Customer cannot exceed max_advance_bookings | FR004 |
| BR009 | Refund failure never blocks slot release | FR005 |
| BR010 | Bookings only within configured advance window | FR003, FR004 |
| BR011 | Suspended tenant has 30-day read-only window before deprovisioning | FR013 |
| BR012 | Data retained 90 days post-deprovisioning; payments 7 years; audit log permanent | FR012, FR013 |
| BR013 | Every event must carry tenant_id, event_id, correlation_id, occurred_at | All events |
| BR014 | All write operations produce an immutable audit log entry | All write features |
