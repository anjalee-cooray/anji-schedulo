---
title: Sequence Diagram — Appointment Rescheduling
journeyId: UJ004
layer: 02-design
status: current
lastUpdated: 2026-06-28
---

# D3 · Sequence Diagram — UJ004: Appointment Rescheduling

## Overview

This diagram shows the rescheduling flow. The strategy is "new slot first, then release old" to ensure atomicity: the new slot is locked before the old slot is released. If the new slot is unavailable, the original booking is unchanged. Both operations (new lock + old release + event write) are committed in a single transaction, preventing any window where neither slot is held.

---

```mermaid
sequenceDiagram
    participant Client as Client (Browser)
    participant GW as api-gateway
    participant BCS as booking-command-service
    participant OR as outbox-relay
    participant SNS as SNS
    participant NS as notification-service
    participant AS as availability-service
    participant DS as dashboard-service
    participant Redis as Redis (ElastiCache)
    participant SG as SendGrid

    Note over Client,SG: Phase A — Reschedule Request (Synchronous)

    Client->>+GW: POST /appointments/{appointment_id}/reschedule<br/>Bearer JWT (role=customer)<br/>{new_slot_date, new_slot_start, staff_id}
    GW->>GW: Validate JWT, extract tenant_id + customer_id
    GW->>+BCS: Forward reschedule request

    Note over BCS: Step 1 — Load and validate existing appointment
    BCS->>BCS: SELECT appointment WHERE appointment_id AND tenant_id (RLS)
    BCS->>BCS: Verify appointment status = confirmed
    BCS->>BCS: Verify caller owns appointment

    alt Appointment not found or already cancelled
        BCS-->>GW: 404 Not Found / 422 Unprocessable Entity
        GW-->>-Client: Error response
    end

    Note over BCS: Step 2 — Validate new slot is within booking window (BR010)
    BCS->>BCS: Check new_slot_date ≤ NOW() + booking_window_days

    alt New slot beyond booking window (BR010)
        BCS-->>GW: 422 Unprocessable Entity (slot beyond booking window)
        GW-->>-Client: 422 — slot not available in that window
    end

    Note over BCS: Step 3 — Attempt new slot lock (BR001)
    Note over BCS: BEGIN TRANSACTION
    BCS->>BCS: INSERT slot_locks (tenant_id, staff_id, new_slot_date, new_slot_start)<br/>UNIQUE constraint — prevents double-booking (BR001)

    alt New slot unavailable — unique constraint violation
        Note over BCS: ROLLBACK — original booking UNCHANGED
        BCS-->>GW: 409 Conflict {error: "slot_unavailable"}
        GW-->>-Client: 409 — original booking retained, no change
    end

    Note over BCS: Step 4 — Atomic: release old slot + confirm reschedule
    BCS->>BCS: UPDATE slot_locks SET released_at=NOW()<br/>WHERE appointment_id=existing AND released_at IS NULL
    BCS->>BCS: INSERT appointment_events (type=rescheduled,<br/>old_slot_date, old_slot_start,<br/>new_slot_date, new_slot_start)
    BCS->>BCS: INSERT outbox_records (appointment.rescheduled, full payload)
    BCS->>BCS: INSERT audit_log (before_snapshot, after_snapshot)
    Note over BCS: COMMIT — new slot locked, old slot freed atomically

    BCS-->>-GW: 200 OK {appointment_id,<br/>new_slot_date, new_slot_start, new_slot_end}
    GW-->>-Client: 200 OK — rescheduled confirmed

    Note over Client,SG: Phase B — Async Fan-out (appointment.rescheduled)

    loop Every 500ms
        OR->>BCS: Poll outbox_records WHERE published_at IS NULL
    end

    OR->>+SNS: Publish appointment.rescheduled<br/>(MessageGroupId=tenant_id)
    SNS-->>-OR: MessageId
    OR->>BCS: UPDATE outbox_records SET published_at=NOW()

    SNS->>NS: SQS: notification-service-appointment-rescheduled-queue
    SNS->>DS: SQS: dashboard-service-appointment-rescheduled-queue
    SNS->>AS: SQS: availability-service-appointment-rescheduled-queue

    Note over NS: Send rescheduling confirmation (FR007)
    NS->>NS: INSERT inbox_records (event_id, consumer=notification-service)
    NS->>NS: Read customer_snapshot, old_slot, new_slot from payload (ECST)
    NS->>+SG: Send rescheduling confirmation to customer<br/>(new date/time, staff member, service)
    SG-->>-NS: 202 Accepted
    NS->>NS: INSERT notification_records (status=sent)

    Note over DS: Update dashboard (increment rescheduled_count)
    DS->>DS: INSERT inbox_records (event_id, consumer=dashboard-service)
    DS->>DS: UPDATE tenant_dashboard_views<br/>INCREMENT rescheduled_count<br/>SET last_event_id (idempotency watermark)

    Note over AS: Invalidate cache for BOTH affected slots
    AS->>AS: INSERT inbox_records (event_id, consumer=availability-service)
    AS->>Redis: DEL availability:{tenant_id}:{staff_id}:{old_slot_date}
    AS->>Redis: DEL availability:{tenant_id}:{staff_id}:{new_slot_date}
    Note over AS: Old slot now visible for new bookings<br/>New slot removed from available slots
```

---

## Key Design Decisions

### "New slot first, then release old" strategy

The new slot lock is acquired before the old slot is released, both in the same transaction. This prevents:
- A window where the old slot is freed but the new slot is not yet locked (another booking could take either).
- A partial state where neither slot is held (atomicity violated).

If the new slot lock fails, the transaction rolls back and the original booking is completely unchanged.

### No payment re-processing on reschedule

Rescheduling does not trigger a new payment or refund in v1. The original payment remains captured. If the service price changed between booking and rescheduling, the tenant handles any price difference out-of-band. This is a deliberate simplification for the MVP.

---

## Key Business Rules Enforced

| Step | Rule | Enforcement |
|---|---|---|
| New slot availability | BR001 — no double-booking on new slot | Unique constraint on slot_locks |
| Original booking preserved on failure | BR004 — atomicity | Transaction rollback on constraint violation |
| Booking window | BR010 — new slot must be within window | BCS validates before saga begins |
| Audit trail | BR014 — immutable record of reschedule | audit_log INSERT in transaction |
| Event envelope | BR013 — full envelope on appointment.rescheduled | EventEnvelope validator |
