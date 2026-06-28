---
title: Flow Spec — Appointment Rescheduling
journeyId: UJ004
layer: 02-design
status: current
lastUpdated: 2026-06-28
---

# D2 · Flow Spec — UJ004: Appointment Rescheduling

## Header

| Field | Value |
|---|---|
| Journey ID | UJ004 |
| Journey Name | Appointment Rescheduling |
| Persona | P3 — End Customer |
| Priority | High |
| Related Functional Requirements | FR006 |
| Related Business Rules | BR001, BR003, BR004, BR010, BR013, BR014 |

---

## 1. Technical Flow Overview

**Architecture Pattern: Sequential Saga Orchestration — new slot first, then release old slot**

Rescheduling is the most constraint-sensitive flow in the booking domain. The customer must never lose both slots. The critical invariant is: **the old slot is never released until the new slot is successfully locked.** This is enforced at the database transaction level, not through saga compensation.

The flow is deliberately synchronous at the slot-lock step. Rather than a two-phase saga with compensation, booking-command-service acquires the new `SlotLock` and releases the old one in a single PostgreSQL transaction. If the new slot is unavailable, the INSERT fails and the old booking is untouched — no compensation is required.

The `appointment.rescheduled` event is published via the Transactional Outbox pattern (NFR003). Downstream consumers — notification-service (confirmation to customer), dashboard-service (rescheduled_count increment), and availability-service (cache invalidation for both old and new slots) — are fully decoupled and eventually consistent.

---

## 2. Services Involved

| Service | Role in This Flow |
|---|---|
| api-gateway | Validates JWT (must own the appointment being rescheduled), injects `correlation_id`, routes to booking-command-service |
| booking-command-service | Owns the rescheduling saga. Validates state and availability, acquires new SlotLock, releases old SlotLock, writes AppointmentEvent(rescheduled) and OutboxRecord atomically. |
| availability-service | Queried to confirm the new slot is within working hours and not blocked before the lock attempt. |
| outbox-relay | Polls `outbox_records` every 500 ms. Publishes `appointment.rescheduled` to SNS. |
| notification-service | Consumes `appointment.rescheduled`. Sends rescheduling confirmation to customer via email/SMS. |
| dashboard-service | Consumes `appointment.rescheduled`. Increments `rescheduled_count` on `TenantDashboardView`. |
| availability-service (consumer) | Consumes `appointment.rescheduled`. Invalidates Redis slot cache for both the old slot and the new slot. |

---

## 3. Step-by-Step Technical Flow

### Phase A — Reschedule Request (Synchronous)

**Step 1 — Customer submits reschedule request**

The customer (P3) calls:
```
POST /api/v1/appointments/{appointment_id}/reschedule
Authorization: Bearer <JWT>

{
  "new_slot_date": "2026-07-15",
  "new_slot_start": "10:00",
  "idempotency_key": "<client-generated UUID>"
}
```

api-gateway validates the JWT:
- `role = customer`
- `tid` matches the appointment's `tenant_id`
- Injects `X-Correlation-ID` and forwards to booking-command-service.

**Step 2 — booking-command-service validates preconditions**

booking-command-service performs the following checks in order. Any failure returns immediately without acquiring any lock:

1. **Appointment ownership:** `SELECT * FROM appointments WHERE appointment_id = $1 AND tenant_id = $tid AND customer_id = $sub`. If not found → `404 Not Found`.
2. **Appointment state:** Derived by reading the latest `AppointmentEvent` for this appointment. Must be `confirmed`. If `cancelled` or `completed` → `409 Conflict` with reason `appointment_not_reschedulable`.
3. **New slot availability (pre-check):** Calls availability-service internally to confirm the new slot falls within the staff member's working hours (BR003) and does not already have a confirmed booking. This is a read-only pre-check — it does not guarantee the slot will still be available when the lock is attempted, but it surfaces obvious failures cheaply before touching the database.
4. **Booking window:** Confirms `new_slot_date` is within `tenant_config.booking_window_days` of today (BR010). If outside window → `422 Unprocessable Entity`.
5. **Idempotency:** Checks `booking_idempotency_keys WHERE idempotency_key = $key`. If found and result is `confirmed`, returns the stored response without reprocessing (BR006).

**Step 3 — booking-command-service acquires new SlotLock and releases old SlotLock atomically**

Within a single PostgreSQL transaction (RLS context: `SET LOCAL app.tenant_id = $tid`):

1. **INSERT new SlotLock:**
   ```sql
   INSERT INTO slot_locks
     (lock_id, tenant_id, staff_id, slot_date, slot_start, appointment_id, locked_at)
   VALUES
     (gen_random_uuid(), $tenant_id, $staff_id, $new_slot_date, $new_slot_start,
      $appointment_id, NOW())
   ```
   The unique partial index on `(tenant_id, staff_id, slot_date, slot_start) WHERE released_at IS NULL` enforces BR001. If the slot is already taken, the INSERT raises a unique constraint violation — the transaction is rolled back and booking-command-service returns **`409 Conflict`** with `failure_reason: slot_unavailable`. The original booking is completely unchanged.

2. **UPDATE old SlotLock to released:**
   ```sql
   UPDATE slot_locks
   SET released_at = NOW()
   WHERE tenant_id = $tenant_id
     AND staff_id = $staff_id
     AND slot_date = $old_slot_date
     AND slot_start = $old_slot_start
     AND appointment_id = $appointment_id
     AND released_at IS NULL
   ```

3. **INSERT AppointmentEvent (rescheduled):**
   ```sql
   INSERT INTO appointment_events
     (event_id, appointment_id, tenant_id, event_type, occurred_at,
      correlation_id, causation_id, actor_id, actor_role, payload)
   VALUES
     (gen_random_uuid(), $appointment_id, $tenant_id,
      'appointment.rescheduled', NOW(),
      $correlation_id, $reschedule_request_event_id,
      $customer_id, 'customer',
      { old_slot_date, old_slot_start, new_slot_date, new_slot_start,
        customer_snapshot, staff_snapshot, service_snapshot })
   ```

4. **INSERT OutboxRecord:**
   ```
   outbox_id, tenant_id, topic = 'appointment-rescheduled-topic',
   event_type = 'appointment.rescheduled',
   payload = { full appointment.rescheduled envelope (see Section 5) },
   created_at = NOW(), published_at = NULL
   ```

5. **INSERT AuditLog:**
   ```
   entity_type = 'Appointment', entity_id = $appointment_id,
   operation = 'reschedule', actor_id = $customer_id, actor_role = 'customer',
   occurred_at = NOW(), before_snapshot = { old slot }, after_snapshot = { new slot }
   ```

Transaction commits. All five writes succeed or none do.

**Step 4 — booking-command-service returns HTTP 200 OK**

Response:
```json
{
  "appointment_id": "<uuid>",
  "status": "confirmed",
  "new_slot_date": "2026-07-15",
  "new_slot_start": "10:00",
  "new_slot_end": "11:00",
  "staff_name": "Alex Johnson",
  "service_name": "Haircut",
  "correlation_id": "<uuid>"
}
```

The old slot is now released and immediately available for new bookings. The new slot is locked. The `appointment.rescheduled` event is in the outbox and will be published within 500 ms.

### Phase B — Event Fan-out (Asynchronous)

**Step 5 — outbox-relay publishes appointment.rescheduled to SNS**

outbox-relay finds the `appointment.rescheduled` record in `outbox_records WHERE published_at IS NULL`, publishes to `appointment-rescheduled-topic` on SNS with `MessageGroupId = tenant_id`, and marks `published_at = NOW()` only on SNS publish success.

**Step 6 — SNS fans out to subscriber queues**

SNS delivers to three SQS FIFO queues concurrently:
- `notification-service-appointment-rescheduled-queue`
- `dashboard-service-appointment-rescheduled-queue`
- `availability-service-appointment-rescheduled-queue`

Each queue has a paired DLQ with `maxReceiveCount = 4`.

**Step 7 — notification-service sends rescheduling confirmation**

1. InboxRecord deduplication: `INSERT INTO inbox_records (event_id, consumer_name='notification-service')`. If duplicate, skip.
2. Reads `old_slot_date`, `old_slot_start`, `new_slot_date`, `new_slot_start`, `customer_snapshot`, `staff_snapshot`, `service_snapshot` from event payload (Event-Carried State Transfer — no back-call).
3. Selects the rescheduling confirmation template.
4. Sends email and/or SMS to `customer_snapshot.email` and `customer_snapshot.phone` via SendGrid/Twilio.
5. Writes `notification_records` row.
6. On failure after 4 retries: `notification_records.status = dlq`, raises `notification.failed` to ops-service.

**Step 8 — dashboard-service increments rescheduled_count**

1. InboxRecord deduplication check.
2. `UPDATE tenant_dashboard_views SET rescheduled_count = rescheduled_count + 1, last_event_id = $event_id WHERE tenant_id = $tenant_id AND view_date = $slot_date`. Uses `last_event_id` watermark for idempotency (NFR009).

**Step 9 — availability-service invalidates cache**

1. InboxRecord deduplication check.
2. Deletes Redis keys for the **old slot** (`availability:{tenant_id}:{staff_id}:{old_slot_date}`) — the old slot is now free.
3. Deletes Redis key for the **new slot** (`availability:{tenant_id}:{staff_id}:{new_slot_date}`) — the new slot is now locked.
4. Next customer availability query rebuilds the cache from the database (Cache-Aside pattern, NFR005).

---

## 4. Compensation / Failure Flows

### Failure Point 1 — New slot already taken (Step 3, INSERT fails)

**Impact:** The unique constraint violation on `slot_locks` is caught within the transaction. The transaction rolls back. No SlotLock, AppointmentEvent, OutboxRecord, or AuditLog entry is written. The original booking is completely unchanged.

**Compensation:** None required — the atomicity of the transaction prevents any partial state.

**Client response:** `409 Conflict`
```json
{
  "error": "slot_unavailable",
  "message": "The requested slot is no longer available. Please select a different time.",
  "appointment_id": "<original appointment_id>",
  "original_slot_date": "2026-07-10",
  "original_slot_start": "09:00"
}
```

The client can immediately query availability and present the customer with alternative slots without any cleanup.

### Failure Point 2 — Database error mid-transaction (Step 3)

**Impact:** PostgreSQL rolls back the entire transaction. No partial writes exist. The original booking remains confirmed at the original slot.

**Compensation:** None required (transaction atomicity).

**Client response:** `503 Service Unavailable` with `Retry-After: 5` header. The customer can retry safely — idempotency key prevents double-processing if the first transaction actually committed before the error was surfaced.

### Failure Point 3 — outbox-relay SNS publish fails (Step 5)

**Impact:** The `appointment.rescheduled` event is not delivered to consumers. The slot swap has already committed — the new slot is locked and the old slot is released. The appointment state is correct in the write model. Downstream read models (dashboard, notification) lag until the event is delivered.

**Compensation:** outbox-relay retries on every subsequent poll cycle (every 500 ms). An alert fires if the outbox backlog exceeds 100 records or relay lag exceeds 10 seconds (observability.json). The customer has already received an HTTP 200 response confirming the reschedule.

### Failure Point 4 — notification-service fails to send rescheduling confirmation (Step 7)

**Impact:** The customer does not receive a rescheduling confirmation email/SMS. The slot swap is fully committed.

**Compensation:** Retry up to 4 times with exponential backoff. After 4 failures, routes to DLQ and ops alert is raised. The rescheduling is complete regardless of notification delivery (BR007). The customer can view the updated appointment in the platform.

---

## 5. Events Produced

| Event Name | Producer | Consumers | SNS Topic |
|---|---|---|---|
| `appointment.rescheduled` | booking-command-service | notification-service, dashboard-service, availability-service | `appointment-rescheduled-topic` |

**Key payload fields for `appointment.rescheduled`:**

| Field | Type | Notes |
|---|---|---|
| `event_id` | UUID | Unique event identifier for InboxRecord deduplication |
| `appointment_id` | UUID | The appointment being rescheduled |
| `tenant_id` | UUID | BR013 mandatory envelope field |
| `correlation_id` | UUID | Propagated from api-gateway (NFR014) |
| `causation_id` | UUID | The reschedule request event_id |
| `occurred_at` | ISO 8601 timestamp | BR013 mandatory envelope field |
| `customer_snapshot` | object | `customer_id`, `name`, `email`, `phone` — for notification routing |
| `staff_snapshot` | object | `staff_id`, `name`, `email` |
| `service_snapshot` | object | `service_id`, `name`, `duration_minutes` |
| `old_slot_date` | date | The original appointment date |
| `old_slot_start` | time | The original appointment start time |
| `new_slot_date` | date | The confirmed new appointment date |
| `new_slot_start` | time | The confirmed new appointment start time |
| `new_slot_end` | time | Derived from new_slot_start + service.duration_minutes |

---

## 6. Business Rules Enforced

| BR ID | Rule Summary | Enforcement Point |
|---|---|---|
| BR001 | A slot cannot be booked by more than one customer simultaneously | booking-command-service: unique partial index on `slot_locks` prevents concurrent new slot acquisition |
| BR003 | Staff cannot be booked outside their configured working hours | availability-service: pre-check validates new slot falls within working hours before lock attempt |
| BR004 | A booking saga must be atomic — all steps succeed or all compensations applied | booking-command-service: new SlotLock INSERT and old SlotLock UPDATE are in a single transaction — commit or full rollback, no partial state |
| BR010 | Appointments may only be booked within the tenant's booking window | booking-command-service: validates new_slot_date ≤ today + tenant_config.booking_window_days |
| BR013 | Every event must carry `tenant_id`, `event_id`, `correlation_id`, `occurred_at` | booking-command-service: EventEnvelope validator applied before writing OutboxRecord; outbox-relay validates before SNS publish |
| BR014 | All write operations on tenant-scoped data must produce an immutable audit log entry | booking-command-service: `AuditLog` INSERT in the same transaction as the slot swap |

---

## 7. Consistency Guarantees

| Concern | Consistency Model | Detail |
|---|---|---|
| New SlotLock acquisition | Strong | INSERT within PostgreSQL transaction; unique constraint prevents race conditions |
| Old SlotLock release | Strong | UPDATE in the same transaction as new SlotLock INSERT — both succeed or neither does |
| AppointmentEvent write | Strong | Written in the same transaction as slot operations |
| Rescheduling confirmation email | Eventually consistent | notification-service is a decoupled async subscriber. Delivery lag < 30 seconds under normal load |
| Dashboard rescheduled_count | Eventually consistent | dashboard-service consumes appointment.rescheduled from SQS. Lag target < 5 seconds (NFR008) |
| Availability cache invalidation | Eventually consistent | availability-service invalidates Redis for both slots on event receipt. Lag target < 5 seconds |

---

## 8. Idempotency

### Reschedule request idempotency

The reschedule endpoint accepts an `idempotency_key` from the client (BR006). booking-command-service checks `booking_idempotency_keys WHERE idempotency_key = $key` before executing any saga logic:

- If found with `result_status = confirmed`: the stored response payload is returned immediately without reprocessing.
- If not found: the reschedule proceeds. On completion, a new `BookingIdempotencyKey` row is inserted with the result payload.

This handles the case where the client receives a network timeout but the reschedule actually committed.

### Event idempotency (downstream consumers)

notification-service, dashboard-service, and availability-service use the InboxRecord pattern for deduplication:

- Before processing `appointment.rescheduled`, the consumer inserts `(event_id, consumer_name)` into `inbox_records`.
- If the insert fails due to a unique constraint, the event has already been processed — the SQS message is deleted without side effects.
- This guarantees that even if SQS delivers the same `appointment.rescheduled` message twice (at-least-once delivery), the customer receives exactly one rescheduling confirmation and the dashboard count is incremented exactly once.

### Old slot release idempotency

If booking-command-service crashes after the transaction commits but before returning the HTTP response, the client will retry with the same `idempotency_key`. The idempotency check returns the stored confirmed response. The `appointment.rescheduled` OutboxRecord is still in the database and will be published by outbox-relay on its next poll cycle.
