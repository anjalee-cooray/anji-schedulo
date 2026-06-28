---
title: Flow Spec — Appointment Cancellation
journeyId: UJ003
layer: 02-design
status: current
lastUpdated: 2026-06-28
---

# D2 · Flow Spec — UJ003: Appointment Cancellation

## Header

| Field | Value |
|---|---|
| Journey ID | UJ003 |
| Journey Name | Appointment Cancellation |
| Persona | P3 — End Customer (P2 — Staff Member may also cancel) |
| Priority | Critical |
| Related Functional Requirements | FR005 |
| Related Business Rules | BR002, BR009, BR013, BR014 |

---

## 1. Technical Flow Overview

**Architecture Pattern: Saga Choreography via Transactional Outbox**

Appointment cancellation is implemented as a choreography saga. Unlike the orchestrated booking saga (UJ002), cancellation side effects are fully decoupled: refund, slot release, notification, dashboard update, and analytics update are independent subscribers to the same `appointment.cancelled` event. No orchestrator waits for each step. Compensation is handled within each step's own failure path.

This design reflects the business priority of UJ003: **the slot must be released immediately and unconditionally**. A Stripe refund failure must never delay or block slot availability or customer notification (BR009). By making each side effect a choreographed subscriber, the failure of any one step is contained within that step's retry and DLQ handling and cannot cascade.

The synchronous phase is minimal: validate the cancellation window (BR002), release the SlotLock, write the AppointmentEvent, and return a confirmation response. The entire synchronous phase is a single PostgreSQL transaction — if any part of the write fails, the full transaction rolls back and the appointment remains in its prior state. Once the transaction commits and the `appointment.cancelled` event is in `outbox_records`, the choreography begins.

The Transactional Outbox pattern (ADR005, NFR003) guarantees that `appointment.cancelled` reaches all five subscribers even if the application restarts after the DB commit.

---

## 2. Services Involved

| Service | Role in This Flow |
|---|---|
| api-gateway | Validates JWT, enforces ownership check (customer must own this appointment; staff must be assigned), injects `X-Correlation-ID`, routes to booking-command-service. |
| booking-command-service | Owns the synchronous cancellation phase: validates BR002 policy window, writes SlotLock release, AppointmentEvent(cancelled), OutboxRecord, and AuditLog in a single transaction. Returns 200. |
| outbox-relay | Polls `outbox_records` every 500 ms. Publishes `appointment.cancelled` to SNS. Marks record as published only after SNS success. |
| payment-service | Subscribes to `appointment.cancelled`. Initiates Stripe refund if a captured payment exists. Publishes `payment.refunded` or `payment.refund_failed`. |
| notification-service | Subscribes to `appointment.cancelled`. Sends cancellation confirmation to customer and staff. Publishes `notification.sent` or `notification.failed`. |
| availability-service | Subscribes to `appointment.cancelled`. Invalidates Redis slot cache for the freed slot so it immediately re-surfaces for new bookings. |
| dashboard-service | Subscribes to `appointment.cancelled`. Increments `cancelled_count` and recalculates `cancellation_rate` on TenantDashboardView. |
| analytics-service | Subscribes to `appointment.cancelled`. Updates the per-tenant cancellation rate rolling window in Kinesis. |
| ops-service | Receives `payment.refund_failed` and `notification.failed` events when their respective DLQs are exhausted. Raises alerts. |

---

## 3. Step-by-Step Technical Flow

### Phase A — Cancellation Request (Synchronous)

**Step 1 — Customer or staff submits cancellation request**

The End Customer (P3) or Staff Member (P2) submits a cancellation request:

```
DELETE /appointments/{appointment_id}
```

Or via a named action endpoint:
```
POST /appointments/{appointment_id}/cancel
```

api-gateway validates the JWT:
- **Customer token**: The `sub` claim (customer_id) must match `Appointment.customer_id` for the given `appointment_id`. A customer may not cancel another customer's appointment.
- **Staff token**: The `sub` claim (staff_id) must match `Appointment.staff_id` for the given `appointment_id`. A staff member may cancel only appointments assigned to them.
- **Tenant admin token**: May cancel any appointment within their tenant (no ownership restriction).

If the JWT does not authorise the action → `403 Forbidden`. If the appointment is not found → `404 Not Found`.

**Step 2 — booking-command-service validates the cancellation window (BR002)**

booking-command-service reads the appointment's `slot_date` and `slot_start`, and the tenant's `cancellation_hours` from the local `TenantConfig` cache:

```
Deadline = slot_date + slot_start - cancellation_hours
If NOW() > Deadline → cancellation not permitted
```

If the current time is past the cancellation deadline:

```json
HTTP 422 Unprocessable Entity
{
  "error": "cancellation_window_expired",
  "message": "Cancellations must be made at least 24 hours before the appointment.",
  "deadline": "2026-07-14T10:00:00Z"
}
```

No database writes occur on this path. The appointment remains unchanged.

**Step 3 — booking-command-service writes the cancellation in a single transaction**

booking-command-service opens a PostgreSQL transaction and executes the following writes:

1. **Release the SlotLock**:
   ```sql
   UPDATE slot_locks
   SET released_at = NOW()
   WHERE appointment_id = ? AND released_at IS NULL;
   ```
   This makes the slot immediately available for new bookings. The update is idempotent: if `released_at` is already set (duplicate request), it is a no-op.

2. **Insert AppointmentEvent(cancelled)**:
   ```
   event_id = gen_random_uuid(),
   appointment_id,
   tenant_id,
   event_type = 'appointment.cancelled',
   occurred_at = NOW(),
   correlation_id = (from X-Correlation-ID header),
   causation_id = (event_id of the most recent appointment.confirmed event),
   actor_id = (customer_id or staff_id from JWT sub),
   actor_role = (customer | staff | admin),
   payload = {
     customer_snapshot: { customer_id, name, email, phone },
     staff_snapshot: { staff_id, name, email },
     service_snapshot: { service_id, name },
     slot_date, slot_start,
     payment_id (nullable),
     amount_eligible_for_refund (nullable)
   }
   ```
   The payload carries the full event snapshot so all five downstream consumers can operate without back-calling booking-command-service (Event-Carried State Transfer).

3. **Insert OutboxRecord for appointment.cancelled**:
   ```
   topic = 'appointment-cancelled-topic',
   event_type = 'appointment.cancelled',
   payload = full event envelope (same as AppointmentEvent.payload + envelope fields),
   published_at = NULL
   ```

4. **Insert AuditLog entry** (BR014):
   ```
   entity_type = 'Appointment',
   entity_id = appointment_id,
   operation = 'cancel',
   actor_id, actor_role, occurred_at,
   before_snapshot = { prior appointment state },
   after_snapshot = { appointment state with cancellation applied }
   ```

5. **Decrement Customer.active_booking_count**:
   ```sql
   UPDATE customers
   SET active_booking_count = active_booking_count - 1
   WHERE customer_id = ? AND tenant_id = ?;
   ```

Transaction commits. If any write fails, the entire transaction rolls back. The SlotLock remains held, the AppointmentEvent is not written, and no OutboxRecord exists — the appointment remains confirmed.

**Step 4 — booking-command-service returns 200 OK**

```json
HTTP 200 OK
{
  "appointment_id": "<uuid>",
  "status": "cancelled",
  "slot_date": "2026-07-15",
  "slot_start": "10:00",
  "message": "Your appointment has been successfully cancelled.",
  "correlation_id": "<uuid>"
}
```

The customer receives immediate confirmation. The slot is already free. Refund and notification processing continue asynchronously.

---

### Phase B — Choreography (Asynchronous, Independent Steps)

**Step 5 — outbox-relay publishes appointment.cancelled to SNS**

outbox-relay polls `outbox_records WHERE published_at IS NULL` every 500 ms. It finds the `appointment.cancelled` record and publishes it to `appointment-cancelled-topic` on SNS with `MessageGroupId = tenant_id`. It marks `published_at = NOW()` only after SNS publish succeeds. If SNS is temporarily unavailable, the record remains unpublished and retried on the next cycle. An alert fires if backlog exceeds 100 records for more than 10 seconds.

**Step 6 — SNS fans out to five subscriber queues**

SNS delivers `appointment.cancelled` to five SQS FIFO queues concurrently:
- `payment-service-appointment-cancelled-queue`
- `notification-service-appointment-cancelled-queue`
- `availability-service-appointment-cancelled-queue`
- `dashboard-service-appointment-cancelled-queue`
- `analytics-service-appointment-cancelled-queue`

Each queue is FIFO with `MessageGroupId = tenant_id`. Each queue has a paired DLQ with `maxReceiveCount = 4`.

**Step 7 — payment-service initiates refund**

payment-service dequeues from `payment-service-appointment-cancelled-queue`:

1. **InboxRecord deduplication**: `INSERT INTO inbox_records (event_id, consumer_name='payment-service')`. If unique constraint fails → already processed, message deleted.
2. **Determine refund eligibility**: reads `payment_id` and `amount_eligible_for_refund` from event payload. If `payment_id` is null or `Payment.status ≠ captured` → no refund action. Message deleted.
3. **Initiate Stripe refund**: calls `POST /v1/refunds` with `payment_intent = stripe_payment_intent_id` and `amount = amount_eligible_for_refund`. Stripe idempotency key is set to `payment_id` to prevent duplicate refunds on retry.
4. **On success**: Updates `Payment.status = refunded`, sets `Payment.refunded_at` and `stripe_refund_id`. Inserts `OutboxRecord` for `payment.refunded`. Message deleted from SQS.
5. **On Stripe failure**: Marks attempt in `NotificationRecord` equivalent (retry count on SQS). After 4 retries (SQS `maxReceiveCount`), message is moved to `payment-service-appointment-cancelled-queue-dlq`. payment-service updates `Payment.status = refund_failed` and publishes `payment.refund_failed` event (BR009). ops-service receives the event and raises a P2 alert.

Slot availability is already restored (Phase A, Step 3). Refund failure has zero impact on slot release or customer notification (BR009).

Target: refund initiated within 60 seconds of cancellation (UJ003 success outcome).

**Step 8 — notification-service sends cancellation confirmation**

notification-service dequeues from `notification-service-appointment-cancelled-queue`:

1. **InboxRecord deduplication** check.
2. Reads `customer_snapshot` (name, email, phone) and `staff_snapshot` (name, email) from the event payload — no back-call needed (Event-Carried State Transfer).
3. Selects the cancellation email template.
4. Sends cancellation email to `customer_snapshot.email` via SendGrid: "Your appointment has been cancelled. Refund is being processed." (if `payment_id` is present and `amount_eligible_for_refund > 0`).
5. Sends cancellation notification to `staff_snapshot.email` (or SMS if Pro/Enterprise): "Your appointment at 10:00 on 15 July has been cancelled."
6. Writes `NotificationRecord` rows for each recipient with `status = sent`.
7. **On delivery failure**: Retries up to 4 times with exponential backoff (1s, 2s, 4s, 8s). After 4 failures: `NotificationRecord.status = dlq`, publishes `notification.failed` to ops-service queue.

Notification failure does not affect cancellation confirmation or slot release (BR007).

Target: customer notified within 30 seconds of cancellation (UJ003 success outcome).

**Step 9 — availability-service re-surfaces the freed slot**

availability-service dequeues from `availability-service-appointment-cancelled-queue`:

1. **InboxRecord deduplication** check.
2. Constructs the affected cache key: `availability:{tenant_id}:{staff_id}:{slot_date}`.
3. Deletes the key from ElastiCache Redis.
4. The next availability query for this date and staff member will recompute from PostgreSQL. The `slot_locks` row now has `released_at IS NOT NULL`, so the slot is excluded from occupied intervals and included in the computed open slot list.

Target: slot re-opened within 10 seconds of cancellation (UJ003 success outcome). Cache invalidation typically completes within 1 second.

**Step 10 — dashboard-service updates TenantDashboardView**

dashboard-service dequeues from `dashboard-service-appointment-cancelled-queue`:

1. **InboxRecord deduplication** via `last_event_id` watermark: if this `event_id` matches or precedes `TenantDashboardView.last_event_id` → skip.
2. Increments `TenantDashboardView.cancelled_count` by 1.
3. Recalculates `cancellation_rate = cancelled_count / (confirmed_count + cancelled_count)` for the view date.
4. Updates `last_event_id` to this event's `event_id`.

Dashboard reflects updated cancellation rate within 5 seconds (NFR008).

**Step 11 — analytics-service updates cancellation rate window**

analytics-service dequeues from `analytics-service-appointment-cancelled-queue`:

1. **InboxRecord deduplication** check.
2. Publishes to Kinesis with `partition_key = tenant_id`.
3. Kinesis consumer increments the cancellation count in the rolling 50-booking window for the tenant.
4. If `cancellation_rate > 0.30` (30 cancellations in the last 50 bookings): publishes `analytics.cancellation_rate_alert` event (FR009, in scope from v1.0).

---

## 4. Compensation / Failure Flows

### Failure Point 1 — Cancellation outside policy window (Step 2)

**Impact:** The cancellation is rejected before any database write. The appointment remains confirmed.

**Compensation:** None required — no state has changed.

**Client experience:** `422 Unprocessable Entity` with the cancellation deadline in the response. The customer is advised to contact the business directly for late cancellations.

---

### Failure Point 2 — Database transaction fails during write (Step 3)

**Impact:** The PostgreSQL transaction rolls back. SlotLock is still held, AppointmentEvent is not written, OutboxRecord does not exist. The appointment remains confirmed.

**Compensation:** None required — the transaction is atomic. The client receives `500 Internal Server Error` and can retry the cancellation.

**Operator action:** The error is logged with `correlation_id`. Standard PostgreSQL monitoring alerts on transaction failure rates.

---

### Failure Point 3 — outbox-relay fails to publish appointment.cancelled (Step 5)

**Impact:** The `appointment.cancelled` event is written to PostgreSQL (the transaction committed) but not yet delivered to consumers. The cancellation confirmation has already been returned to the client (`200 OK`). The slot is already released (SlotLock updated in Phase A). However, refund, notification, dashboard, and analytics updates are pending.

**Compensation:** outbox-relay retries the SNS publish on every subsequent 500 ms poll cycle until it succeeds. The OutboxRecord remains unpublished. An alert fires if backlog exceeds 100 records for more than 10 seconds. Once SNS publish succeeds, all five consumers receive the event and process normally. There is no data loss.

---

### Failure Point 4 — Refund fails after 4 retries (Step 7, BR009)

**Impact:** Stripe is unable to process the refund. The Payment record is in `status = refund_failed`. The slot is already released. The customer has already been notified of their cancellation.

**Compensation:** payment-service routes the message to DLQ after 4 SQS retries. `payment.refund_failed` event published to ops-service. P2 alert raised in Grafana. Platform Operator (P4) inspects the full event payload (including `stripe_payment_intent_id`) in the ops console and can manually issue the refund via the Stripe Dashboard. An AuditLog entry is written when the operator resolves the event.

**No impact on**: slot release (already done), cancellation confirmation (already sent), customer notification (already sent), dashboard update (already applied). Only the refund is pending (BR009).

---

### Failure Point 5 — Notification fails after 4 retries (Step 8)

**Impact:** Customer and/or staff do not receive a cancellation notification. The cancellation is confirmed, slot is released, and refund is in progress.

**Compensation:** notification-service routes message to DLQ after 4 retries. `notification.failed` event published. Platform Operator receives alert and can re-trigger the notification from the ops console (FR010). Booking correctness is unaffected (BR007).

---

## 5. Events Produced

| Event Name | Producer | Consumers | SNS Topic |
|---|---|---|---|
| `appointment.cancelled` | booking-command-service | payment-service, notification-service, availability-service, dashboard-service, analytics-service | `appointment-cancelled-topic` |
| `payment.refunded` | payment-service | notification-service (optional refund confirmation in v1.0) | `payment-refunded-topic` |
| `payment.refund_failed` | payment-service | ops-service (DLQ escalation, manual resolution) | `payment-refund-failed-topic` |
| `notification.sent` | notification-service | (none in MVP) | `notification-sent-topic` |
| `notification.failed` | notification-service | ops-service | `notification-failed-topic` |

---

## 6. Business Rules Enforced

| BR ID | Rule Summary | Enforcement Point |
|---|---|---|
| BR002 | A customer may only cancel within the tenant's configured cancellation window | booking-command-service: validates `NOW() < (slot_start - cancellation_hours)` before any DB write; returns 422 if outside window |
| BR007 | Notification delivery failure must never block, delay, or roll back the triggering operation | notification-service is a fully decoupled async subscriber; cancellation is confirmed before notification processing begins |
| BR009 | Refund failure must not prevent slot release or customer notification | Cancellation saga is choreographed — slot release happens in Phase A before any refund attempt; refund failure routes to DLQ independently of slot and notification steps |
| BR013 | Every event must carry `tenant_id`, `event_id`, `correlation_id`, `occurred_at` | booking-command-service: EventEnvelope validator applied before writing OutboxRecord; outbox-relay: validates envelope before SNS publish |
| BR014 | All write operations on tenant-scoped data must produce an immutable audit log entry | booking-command-service: AuditLog INSERT in same transaction as AppointmentEvent(cancelled) and SlotLock release |

---

## 7. Consistency Guarantees

| Concern | Consistency Model | Detail |
|---|---|---|
| Slot release | Strong | SlotLock `UPDATE released_at = NOW()` is in the same transaction as AppointmentEvent(cancelled). The slot is freed atomically on transaction commit. No timing gap between cancellation and slot availability. |
| AppointmentEvent(cancelled) | Strong | Written in the same PostgreSQL transaction as the SlotLock release. The event log is the source of truth for appointment state. |
| Refund initiation | Eventually consistent | payment-service is a decoupled async subscriber. Refund initiated within 60 seconds under normal load (UJ003 success outcome). |
| Customer notification | Eventually consistent | notification-service is a decoupled async subscriber. Target: within 30 seconds (UJ003 success outcome). |
| Slot cache invalidation | Eventually consistent | availability-service invalidates Redis cache on consuming `appointment.cancelled`. Target: within 10 seconds. The SlotLock is released immediately at DB level — cache lag only affects pre-computed slot lists, not the authoritative constraint. |
| Dashboard cancellation rate | Eventually consistent | dashboard-service updates TenantDashboardView. Lag target < 5 seconds (NFR008). |
| Analytics rate window | Eventually consistent | analytics-service updates Kinesis window. Alert detection lag < 60 seconds (FR009). |

---

## 8. Idempotency

### Cancellation request idempotency

The cancellation endpoint (`DELETE /appointments/{appointment_id}`) is idempotent by design:

- **SlotLock update**: `UPDATE slot_locks SET released_at = NOW() WHERE appointment_id = ? AND released_at IS NULL`. If `released_at` is already set (second request), the WHERE clause matches zero rows — a no-op. The appointment is already cancelled.
- **AppointmentEvent insertion**: A second cancellation attempt will find no SlotLock to release and will fail the pre-condition check in booking-command-service (appointment is already in `cancelled` state as derived from event history). booking-command-service returns `409 Conflict` with `"error": "appointment_already_cancelled"`.

The combination ensures that a customer refreshing the page or retrying due to a network timeout receives a safe response without creating a duplicate cancellation event.

### Event idempotency (downstream consumers, InboxRecord pattern)

All five downstream consumers implement the InboxRecord pattern before processing any event:

```sql
INSERT INTO inbox_records (event_id, consumer_name, processed_at)
VALUES (?, '<service-name>', NOW());
```

If the insert fails (unique constraint on `(event_id, consumer_name)`), the event was already processed. The SQS message is deleted without side effects. This guarantees:

- **payment-service**: A duplicate `appointment.cancelled` delivery does not trigger a second Stripe refund.
- **notification-service**: A duplicate delivery does not send a second cancellation email or SMS.
- **availability-service**: A duplicate delivery does not cause cache corruption (delete is idempotent).
- **dashboard-service**: A duplicate delivery does not double-increment the cancelled count; the `last_event_id` watermark provides a second layer of protection.
- **analytics-service**: A duplicate delivery does not skew the rolling window count.

### payment-service double-refund prevention

In addition to InboxRecord deduplication, payment-service checks `Payment.status` before calling Stripe:

```python
if payment.status != 'captured':
    # already refunded or no payment — skip
    return
```

This prevents a duplicate refund in the scenario where:
1. payment-service processes `appointment.cancelled`, calls Stripe, marks `Payment.status = refunded`, but crashes before deleting the SQS message.
2. SQS redelivers the same message.
3. InboxRecord INSERT fails (already processed) — message deleted safely.

If the InboxRecord insert were to succeed (edge case on consumer restart without committed inbox write), the `Payment.status` check provides a final safety net.
