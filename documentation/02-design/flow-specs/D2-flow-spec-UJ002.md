---
title: Flow Spec — Appointment Booking
journeyId: UJ002
layer: 02-design
status: current
lastUpdated: 2026-06-28
---

# D2 · Flow Spec — UJ002: Appointment Booking

## Header

| Field | Value |
|---|---|
| Journey ID | UJ002 |
| Journey Name | Appointment Booking |
| Persona | P3 — End Customer |
| Priority | Critical |
| Related Functional Requirements | FR003, FR004 |
| Related Business Rules | BR001, BR004, BR006, BR007, BR008, BR010, BR013, BR014 |

---

## 1. Technical Flow Overview

**Architecture Pattern: Saga Orchestration with Transactional Outbox**

Appointment booking is the most critical user journey in the system. It comprises two phases: a synchronous read phase (slot browsing) and a synchronous orchestrated saga (booking). The saga serialises three steps — slot reservation, payment capture, and appointment confirmation — in a strict sequence. Any step failure triggers immediate compensation, ensuring the system never produces a partial state: no slot is held without a confirmed appointment, and no charge is made without a confirmed booking.

The Transactional Outbox pattern (ADR005, NFR003) decouples the saga's write operations from downstream notification and analytics side effects. The `appointment.confirmed` event is written to `outbox_records` in the same PostgreSQL transaction as the SlotLock and AppointmentEvent rows. The outbox-relay publishes to SNS independently, ensuring no event is lost between the DB commit and the message bus.

Slot availability is served from a Redis cache (TTL 30 seconds) to meet the 500ms p95 response time target (NFR005). Cache invalidation on `appointment.confirmed` and `appointment.cancelled` events ensures customers never see stale availability.

The booking endpoint is public — no AnjiSchedulo account is required. A customer is identified by their email address in the request body. An idempotency key (UUID, client-generated) is mandatory to prevent duplicate bookings from network retries (BR006).

---

## 2. Services Involved

| Service | Role in This Flow |
|---|---|
| api-gateway | Validates request, injects `X-Correlation-ID`, enforces rate limits, routes to appropriate service. No auth required for public booking endpoint. |
| availability-service | Serves available slot queries. Checks Redis cache; on miss computes from PostgreSQL (working_hours, staff_blocks, slot_locks). |
| booking-command-service | Orchestrates the booking saga. Owns SlotLock, Appointment, AppointmentEvent, BookingIdempotencyKey. Enforces BR001, BR006, BR008, BR010. |
| payment-service | Called synchronously by booking-command-service during saga step 2. Captures payment via Stripe PaymentIntents API. Publishes payment events via Outbox. |
| outbox-relay | Polls `outbox_records` every 500 ms. Publishes `appointment.requested` and `appointment.confirmed` to SNS. Marks records as published only after SNS success. |
| notification-service | Consumes `appointment.confirmed` to send booking confirmation to customer and staff (email/SMS). |
| dashboard-service | Consumes `appointment.confirmed` to increment confirmed_count and net_revenue on TenantDashboardView. |
| analytics-service | Consumes `appointment.confirmed` to update the per-tenant booking velocity window via Kinesis. |

---

## 3. Step-by-Step Technical Flow

### Phase A — Slot Browsing (Synchronous)

**Step 1 — Customer queries available slots**

The End Customer (P3) navigates to a tenant's public booking page and requests available time slots:

```
GET /tenants/{tenantSlug}/availability?service_id=<uuid>&staff_id=<uuid>&date=2026-07-15
```

This endpoint is public — no JWT is required. api-gateway resolves `tenantSlug` to `tenant_id` via the `tenants` table and injects `X-Correlation-ID`. The request is routed to availability-service.

**Step 2 — availability-service checks the Redis cache**

availability-service constructs the cache key:
```
availability:{tenant_id}:{staff_id}:{date}
```

If the key exists in ElastiCache Redis (TTL 30 seconds), the cached slot array is returned immediately. Response time < 50ms on cache hit. If the key does not exist, proceed to Step 3.

**Step 3 — availability-service computes open slots from PostgreSQL**

On a cache miss, availability-service queries PostgreSQL (via RLS-scoped read replica):

1. Reads `working_hours` for the given `staff_id` and `date` to determine the operating window (enforces BR003).
2. Reads `staff_blocks` where `block_start ≤ date ≤ block_end` for the same staff member.
3. Reads `slot_locks WHERE staff_id = ? AND slot_date = ? AND released_at IS NULL` to identify currently occupied slots.
4. Subtracts occupied and blocked intervals from working hours, aligned to the service duration, to produce the list of open slot start times.
5. Excludes slots beyond the tenant's `booking_window_days` (BR010).

The computed slot array is written to Redis with TTL 30 seconds and returned to the customer. Response time target < 500ms p95 (NFR005, FR003).

**Step 4 — Customer selects a slot and proceeds to booking**

The customer selects a date, time, and (optionally) a specific staff member. They enter their contact details (name, email, optional phone). If the service requires payment at booking, payment details are collected client-side and a Stripe PaymentMethod ID is included in the booking request.

---

### Phase B — Booking Saga (Synchronous Orchestration)

**Step 5 — Customer submits booking request**

The customer submits a `POST /tenants/{tenantSlug}/bookings` request:
```json
{
  "idempotency_key": "<client-generated UUID>",
  "service_id": "<uuid>",
  "staff_id": "<uuid>",
  "slot_date": "2026-07-15",
  "slot_start": "10:00",
  "customer_name": "Jane Smith",
  "customer_email": "jane@example.com",
  "customer_phone": "+44 7700 900000",
  "payment_method_id": "<stripe-pm-id>"
}
```

api-gateway injects `X-Correlation-ID` and routes to booking-command-service. No JWT is required; the customer's identity is established by their email address in the request body.

**Step 6 — booking-command-service: idempotency check**

booking-command-service performs the first guard check against `booking_idempotency_keys`:

```sql
SELECT result_status, result_payload FROM booking_idempotency_keys
WHERE idempotency_key = ? AND expires_at > NOW();
```

If a row is found, the stored `result_payload` is returned immediately with the original HTTP status code. No processing occurs. This handles network retries and duplicate submissions from the client (BR006).

**Step 7 — booking-command-service: business rule validation**

If no idempotency key match is found, booking-command-service validates:

- **BR008**: The customer's `active_booking_count` (denormalised on the Customer row) does not exceed `tenant_config.max_advance_bookings`. If exceeded → `422 Unprocessable Entity` with message "Booking limit reached for this customer."
- **BR010**: `slot_date` is within `NOW() + tenant_config.booking_window_days`. If exceeded → `422` with message "Slot is outside the booking window."
- **BR003**: `slot_start` and derived `slot_end` fall within the staff member's configured `working_hours` for `slot_date`. If not → `422`.

If any validation fails, an idempotency key record is inserted with `result_status = failed` and the 422 payload, so retry returns the same error without re-evaluating.

**Step 8 — Saga Step 1: Reserve the slot (atomic lock)**

booking-command-service opens a PostgreSQL transaction and attempts to insert a SlotLock:

```sql
INSERT INTO slot_locks (lock_id, tenant_id, staff_id, slot_date, slot_start, appointment_id, locked_at)
VALUES (gen_random_uuid(), ?, ?, ?, ?, ?, NOW());
```

A partial unique index enforces BR001:
```sql
UNIQUE (tenant_id, staff_id, slot_date, slot_start) WHERE released_at IS NULL
```

If the INSERT raises a unique constraint violation, the slot is taken. The transaction is rolled back. booking-command-service stores `result_status = failed` (409) in `booking_idempotency_keys` and returns `409 Conflict`:
```json
{ "error": "slot_unavailable", "message": "This time slot is no longer available." }
```

No charge has occurred. The customer is invited to select an alternative slot.

If the INSERT succeeds, proceed to Step 9 within the same transaction.

**Step 9 — Saga Step 1 continued: Write Appointment and AppointmentEvent(requested)**

Within the same transaction as the SlotLock INSERT:

1. Upsert `Customer` row (insert if email not seen before for this tenant; update `active_booking_count` atomically).
2. Insert `Appointment` row:
   ```
   appointment_id, tenant_id, customer_id, staff_id, service_id,
   slot_date, slot_start, slot_end (= slot_start + service.duration_minutes),
   idempotency_key, correlation_id, created_at = NOW()
   ```
3. Insert `AppointmentEvent` row:
   ```
   event_type = 'appointment.requested', actor_role = 'customer',
   occurred_at = NOW(), correlation_id, payload = full appointment snapshot
   ```
4. Insert `OutboxRecord` for `appointment.requested` topic (notifies customer of booking in-progress).
5. Insert `AuditLog` entry for appointment creation (BR014).

Transaction commits. If any step fails, the full transaction rolls back — the SlotLock is removed and the slot returns to available (BR004 atomicity).

**Step 10 — Saga Step 2: Capture payment (synchronous call to payment-service)**

If the service requires payment (`service.price_amount > 0` and `tenant_config.payment_mode = 'at_booking'`), booking-command-service calls payment-service synchronously:

```
POST /internal/payments
{ appointment_id, tenant_id, payment_method_id, amount, currency, idempotency_key }
```

payment-service calls the Stripe PaymentIntents API to capture the charge. Stripe's idempotency key is set to `appointment_id` to prevent duplicate charges on network retries.

- **Payment succeeds**: payment-service writes a `Payment` row with `status = captured` and an `OutboxRecord` for `payment.captured`. Returns 200 to booking-command-service with `payment_id`.
- **Payment declined (Stripe returns card error)**: payment-service returns `402 Payment Required`. booking-command-service compensation path executes (Step 11a).
- **Stripe unreachable (circuit breaker open)**: After 5 consecutive failures, the circuit breaker opens. booking-command-service receives `503` and executes compensation (Step 11b).

**Step 11 — Compensation paths (failure only)**

**Step 11a — Payment declined:**

booking-command-service opens a new transaction:
1. `UPDATE slot_locks SET released_at = NOW() WHERE lock_id = ? AND released_at IS NULL` — releases the slot.
2. Inserts `AppointmentEvent(type = 'appointment.failed', failure_reason = 'payment_failed')`.
3. Inserts `OutboxRecord` for `appointment.failed`.
4. Inserts `booking_idempotency_keys` with `result_status = failed`, `http_status = 402`.

Returns `402 Payment Required` to client:
```json
{ "error": "payment_failed", "message": "Your card was declined. The slot has been released." }
```

**Step 11b — Payment service unavailable:**

Same compensation sequence as 11a, with `failure_reason = 'payment_service_unavailable'` and `http_status = 503`. Returns `503 Service Unavailable` with `Retry-After: 30` header.

**Step 12 — Saga Step 3: Write AppointmentEvent(confirmed) and finalize**

On payment success (or if no payment is required), booking-command-service opens a final transaction:

1. Inserts `AppointmentEvent(type = 'appointment.confirmed', causation_id = appointment.requested event_id)` with full appointment snapshot in payload (Event-Carried State Transfer).
2. Inserts `OutboxRecord` for `appointment.confirmed` topic.
3. Updates `Customer.active_booking_count` (atomically increments by 1 — enforces BR008 on subsequent bookings).
4. Inserts `booking_idempotency_keys` with `result_status = confirmed`, `appointment_id`, and the full 201 response payload (so retries return the same confirmation).

Transaction commits. booking-command-service returns `201 Created`:
```json
{
  "appointment_id": "<uuid>",
  "status": "confirmed",
  "slot_date": "2026-07-15",
  "slot_start": "10:00",
  "slot_end": "11:00",
  "staff_name": "Alice",
  "service_name": "Haircut",
  "correlation_id": "<uuid>"
}
```

Response time target: < 2 seconds end-to-end (FR004).

---

### Phase C — Event Fan-out (Asynchronous)

**Step 13 — outbox-relay publishes appointment.confirmed to SNS**

outbox-relay polls `outbox_records WHERE published_at IS NULL` every 500 ms. It finds the `appointment.confirmed` record and publishes it to `appointment-confirmed-topic` on SNS with `MessageGroupId = tenant_id`. It marks `published_at = NOW()` only after the SNS publish returns success.

**Step 14 — SNS fans out to four subscriber queues**

SNS delivers the `appointment.confirmed` message to four SQS FIFO queues concurrently:
- `notification-service-appointment-confirmed-queue`
- `dashboard-service-appointment-confirmed-queue`
- `analytics-service-appointment-confirmed-queue`
- `availability-service-appointment-confirmed-queue`

Each queue has `MessageGroupId = tenant_id`, ensuring per-tenant ordering. Each has a paired DLQ with `maxReceiveCount = 4`.

**Step 15 — notification-service sends confirmation**

notification-service dequeues from `notification-service-appointment-confirmed-queue`:

1. InboxRecord deduplication: `INSERT INTO inbox_records (event_id, consumer_name='notification-service')`. If unique constraint fails → event already processed, message deleted.
2. Reads `customer_snapshot` and `staff_snapshot` from the event payload — no back-call to booking-command-service (Event-Carried State Transfer, BR007).
3. Selects confirmation email template.
4. Calls SendGrid API to send confirmation email to `customer_snapshot.email`.
5. (If Pro/Enterprise plan) Calls Twilio to send confirmation SMS to `staff_snapshot.email` or `customer_snapshot.phone`.
6. Writes `NotificationRecord` with `status = sent`.
7. Retry policy: up to 4 retries with exponential backoff (1s, 2s, 4s, 8s). After 4 failures: `status = dlq`, publishes `notification.failed` event to ops-service queue, raises alert. Booking remains confirmed (BR007).

Customer receives confirmation within 30 seconds of booking (FR004).

**Step 16 — dashboard-service updates TenantDashboardView**

1. InboxRecord deduplication.
2. Checks `TenantDashboardView.last_event_id` — skips if this event_id has already been applied.
3. Increments `confirmed_count` by 1; adds `amount_charged` to `net_revenue`; recalculates `peak_hour` based on `slot_start` hour.
4. Updates `last_event_id` to this event's `event_id`.

Dashboard reflects new booking within 5 seconds (NFR008, UJ005).

**Step 17 — availability-service invalidates cache**

1. InboxRecord deduplication.
2. Deletes the Redis cache key `availability:{tenant_id}:{staff_id}:{slot_date}`.
3. The next availability query for this slot will see `released_at IS NULL` for the new SlotLock and exclude the confirmed slot.

**Step 18 — analytics-service updates velocity window**

1. InboxRecord deduplication.
2. Publishes the event to Kinesis with `partition_key = tenant_id`.
3. Kinesis consumer increments the 5-minute sliding window booking count for the tenant.
4. If `velocity_ratio > 3.0` (current rate vs. historical 5-minute average): publishes `analytics.velocity_spike_detected` event (FR009, in scope from v1.0).

---

## 4. Compensation / Failure Flows

### Failure Point 1 — Slot unavailable (Step 8)

**Impact:** SlotLock INSERT fails due to unique constraint violation. No Appointment row or OutboxRecord has been written.

**Compensation:** None required — the transaction contains only the failed INSERT, which is automatically rolled back. The slot remains locked by whoever holds it.

**Client experience:** `409 Conflict` with `"error": "slot_unavailable"`. The booking page refreshes availability and prompts the customer to select an alternative slot.

---

### Failure Point 2 — Payment declined (Step 10)

**Impact:** Stripe returns a card error. The SlotLock is held; the Appointment row exists with status `requested` (as derived from AppointmentEvent). No charge has been made.

**Compensation:** booking-command-service releases the SlotLock atomically (`UPDATE slot_locks SET released_at = NOW()`) and writes `AppointmentEvent(type='appointment.failed', failure_reason='payment_failed')` in a new transaction. The idempotency key is stored with `result_status = failed` (402). Slot immediately becomes available for new bookings.

**Client experience:** `402 Payment Required` with a human-readable explanation. No charge appears on the customer's card.

---

### Failure Point 3 — payment-service unreachable (Step 10)

**Impact:** The circuit breaker has opened after 5 consecutive Stripe API failures. The SlotLock is held.

**Compensation:** Identical to Failure Point 2. SlotLock released, `appointment.failed` written with `failure_reason = 'payment_service_unavailable'`, idempotency key stored as failed (503).

**Client experience:** `503 Service Unavailable` with `Retry-After: 30` header. The customer is advised to try again.

**Operator action:** The Stripe status page is the primary diagnostic. If the circuit breaker remains open, the Platform Operator (P4) receives a Grafana alert (NFR015).

---

### Failure Point 4 — outbox-relay fails to publish to SNS (Step 13)

**Impact:** `appointment.confirmed` is written in PostgreSQL but not yet delivered to consumers. The booking saga has already committed — the booking is confirmed. Notification, dashboard, and analytics updates are delayed.

**Compensation:** outbox-relay retries the SNS publish on every subsequent 500 ms poll cycle until it succeeds. No data loss: the OutboxRecord remains unpublished. An alert fires if outbox backlog exceeds 100 records or relay lag exceeds 10 seconds (observability.json).

---

### Failure Point 5 — notification-service fails to deliver confirmation (Step 15)

**Impact:** The customer does not receive the confirmation email or SMS. The booking is fully confirmed in the system.

**Compensation:** notification-service retries up to 4 times with exponential backoff. After exhaustion: `NotificationRecord.status = dlq`, `notification.failed` event published to ops-service queue. Platform Operator can re-trigger the notification from the ops console.

**No booking impact:** Notification failure is fully isolated from booking correctness (BR007). The appointment remains confirmed and visible on the tenant dashboard.

---

## 5. Events Produced

| Event Name | Producer | Consumers | SNS Topic |
|---|---|---|---|
| `appointment.requested` | booking-command-service | notification-service (optional in-progress acknowledgement) | `appointment-requested-topic` |
| `appointment.confirmed` | booking-command-service | notification-service, dashboard-service, analytics-service, availability-service | `appointment-confirmed-topic` |
| `appointment.failed` | booking-command-service | notification-service (optional — not in MVP) | `appointment-failed-topic` |
| `payment.captured` | payment-service | booking-command-service (advances saga) | `payment-captured-topic` |
| `payment.failed` | payment-service | booking-command-service (triggers compensation) | `payment-failed-topic` |
| `notification.sent` | notification-service | (none in MVP) | `notification-sent-topic` |
| `notification.failed` | notification-service | ops-service | `notification-failed-topic` |

---

## 6. Business Rules Enforced

| BR ID | Rule Summary | Enforcement Point |
|---|---|---|
| BR001 | A slot cannot be booked by more than one customer simultaneously | booking-command-service: partial unique index on `(tenant_id, staff_id, slot_date, slot_start) WHERE released_at IS NULL`; INSERT fails atomically on conflict |
| BR004 | A booking saga must be atomic — all steps succeed or all compensations are applied | booking-command-service: compensation path releases SlotLock and writes `appointment.failed` on any step failure; no partial state is persisted |
| BR006 | Duplicate requests with the same idempotency key return the original result without re-processing | booking-command-service: `booking_idempotency_keys` table checked before any processing; unique constraint on `idempotency_key` |
| BR007 | Notification delivery failure must never block, delay, or roll back the booking that triggered it | notification-service is a fully decoupled async subscriber; booking-command-service does not wait for or check notification delivery |
| BR008 | A customer may not exceed `max_advance_bookings` active bookings | booking-command-service: validates `Customer.active_booking_count < tenant_config.max_advance_bookings` before initiating saga |
| BR010 | Appointments may only be booked within the tenant's configured booking window | availability-service: excludes slots beyond `booking_window_days`; booking-command-service: re-validates `slot_date` before saga |
| BR013 | Every event must carry `tenant_id`, `event_id`, `correlation_id`, `occurred_at` | booking-command-service: EventEnvelope validator applied before writing OutboxRecord |
| BR014 | All write operations on tenant-scoped data must produce an immutable audit log entry | booking-command-service: `AuditLog` INSERT in same transaction as every Appointment state change |

---

## 7. Consistency Guarantees

| Concern | Consistency Model | Detail |
|---|---|---|
| Slot reservation (SlotLock) | Strong | Single PostgreSQL transaction with unique index constraint. Commit or full rollback. Two concurrent requests for the same slot cannot both succeed. |
| Payment capture | Synchronous within saga | payment-service is called synchronously. The saga does not advance until payment returns success or failure. |
| AppointmentEvent(confirmed) write | Strong | Written in the same PostgreSQL transaction as the final saga step. Commit or full rollback. |
| Confirmation event fan-out | Eventually consistent | outbox-relay publishes within 500 ms of transaction commit under normal load. Consumer lag target < 5 seconds (NFR008). |
| Customer notification delivery | Eventually consistent | notification-service is a decoupled async subscriber. Delivery target: within 30 seconds of booking (FR004). |
| Dashboard update | Eventually consistent | dashboard-service processes `appointment.confirmed` from its SQS queue. Lag target < 5 seconds (NFR008). |
| Availability cache invalidation | Eventually consistent | availability-service invalidates Redis cache on `appointment.confirmed`. Lag < 5 seconds; stale cache slots are protected by the SlotLock unique constraint even if cache is slow to invalidate. |

---

## 8. Idempotency

### Client-side: idempotency_key (BR006)

The client (booking page) generates a UUID v4 `idempotency_key` when the customer clicks the booking button. This key is included in the `POST /tenants/{tenantSlug}/bookings` request body.

booking-command-service checks `booking_idempotency_keys` before processing any request:
- **Key found, not expired**: Return the stored `result_payload` and `http_status` immediately. No saga execution, no DB writes, no Stripe call. This handles network retries, page refreshes mid-submit, and double-clicks.
- **Key found, result_status = confirmed**: The booking succeeded on a prior attempt. Return `201 Created` with the stored appointment details.
- **Key found, result_status = failed**: The booking failed on a prior attempt. Return the stored error code without retrying.
- **Key not found**: Proceed with full saga execution.

Idempotency keys expire after 24 hours and are purged by a scheduled cleanup job.

### Downstream consumers: InboxRecord pattern (NFR009)

All four downstream consumers of `appointment.confirmed` (notification-service, dashboard-service, analytics-service, availability-service) implement the InboxRecord pattern:

Before processing any event, the consumer inserts:
```sql
INSERT INTO inbox_records (event_id, consumer_name, processed_at)
VALUES (?, '<service-name>', NOW());
```

If the INSERT fails (unique constraint on `(event_id, consumer_name)`), the event has already been processed. The SQS message is deleted without any side effects. This guarantees that at-least-once SQS delivery does not produce duplicate notifications, double dashboard increments, or redundant cache invalidations.

### Stripe payment idempotency

payment-service passes `appointment_id` as the Stripe PaymentIntent idempotency key. If payment-service retries a Stripe call for the same appointment (due to network timeout), Stripe returns the result of the original request without charging twice.
