---
title: Sequence Diagram — Appointment Booking
journeyId: UJ002
layer: 02-design
status: current
lastUpdated: 2026-06-28
---

# D3 · Sequence Diagram — UJ002: Appointment Booking

## Overview

This diagram shows the full technical sequence for booking an appointment. It covers the synchronous availability query, the orchestrated booking saga (idempotency check → slot lock → payment capture → confirmation), compensation paths on slot unavailability or payment failure, and the asynchronous fan-out of `appointment.confirmed` to downstream consumers (notification, dashboard, availability cache, analytics).

---

```mermaid
sequenceDiagram
    participant Client as Client (Browser)
    participant GW as api-gateway
    participant AS as availability-service
    participant Redis as Redis (ElastiCache)
    participant BCS as booking-command-service
    participant PS as payment-service
    participant Stripe as Stripe
    participant OR as outbox-relay
    participant SNS as SNS
    participant NS as notification-service
    participant DS as dashboard-service
    participant AnS as analytics-service
    participant SG as SendGrid

    Note over Client,SG: Phase A — Browse Available Slots

    Client->>+GW: GET /tenants/{slug}/availability<br/>?service_id=&staff_id=&date=<br/>(no auth required — public endpoint)
    GW->>+AS: Forward availability query
    AS->>+Redis: GET availability:{tenant_id}:{staff_id}:{date}
    Redis-->>-AS: Cache hit / miss

    alt Cache miss
        AS->>AS: Query working_hours, staff_blocks, slot_locks (PostgreSQL RLS)
        AS->>AS: Compute open slots (within booking_window_days — BR010)
        AS->>Redis: SET availability:{tenant_id}:{staff_id}:{date} TTL=30s
    end

    AS-->>-GW: 200 OK {available_slots[]}
    GW-->>-Client: 200 OK — slots returned (< 500ms p95 — NFR005)

    Note over Client,SG: Phase B — Booking Saga (Synchronous Orchestration)

    Client->>+GW: POST /tenants/{slug}/bookings<br/>{service_id, staff_id, slot_date, slot_start,<br/>customer_name, customer_email,<br/>idempotency_key: UUID}<br/>(no auth required — BR006)
    GW->>GW: Inject X-Correlation-ID
    GW->>+BCS: Forward booking request

    Note over BCS: Step 1 — Idempotency check (BR006)
    BCS->>BCS: SELECT FROM booking_idempotency_keys<br/>WHERE idempotency_key = ?

    alt Idempotency key already exists (duplicate request)
        BCS-->>GW: Return stored result (same status as original)
        GW-->>-Client: Cached response (no re-processing)
    end

    Note over BCS: Step 2 — Business rule checks
    BCS->>BCS: Check customer active_booking_count < max_advance_bookings (BR008)
    BCS->>BCS: Check slot is within booking_window_days (BR010)

    alt BR008 exceeded
        BCS-->>GW: 422 Unprocessable Entity (booking limit exceeded)
        GW-->>-Client: 422 — max bookings reached
    end

    Note over BCS: Step 3 — Slot reservation (BR001)
    Note over BCS: BEGIN TRANSACTION
    BCS->>BCS: INSERT slot_locks<br/>(tenant_id, staff_id, slot_date, slot_start, appointment_id)<br/>UNIQUE constraint on (tenant_id, staff_id, slot_date, slot_start) WHERE released_at IS NULL

    alt Unique constraint violation — slot already locked (BR001)
        Note over BCS: ROLLBACK
        BCS-->>GW: 409 Conflict (slot unavailable — no charge)
        GW-->>-Client: 409 Conflict
    end

    BCS->>BCS: INSERT appointments row
    BCS->>BCS: INSERT appointment_events (type=requested)
    BCS->>BCS: INSERT outbox_records (appointment.requested)
    BCS->>BCS: INSERT audit_log
    Note over BCS: COMMIT (slot is now locked)

    Note over BCS: Step 4 — Payment capture (synchronous call to payment-service)
    BCS->>+PS: POST /payments/capture<br/>{appointment_id, amount, currency, idempotency_key}
    PS->>+Stripe: POST /v1/payment_intents/{id}/confirm
    Stripe-->>-PS: Payment confirmed / declined

    alt Payment fails (Stripe decline or timeout)
        PS-->>BCS: payment.failed {failure_reason}
        Note over BCS: BEGIN TRANSACTION
        BCS->>BCS: UPDATE slot_locks SET released_at=NOW() (BR004 compensation)
        BCS->>BCS: INSERT appointment_events (type=failed)
        BCS->>BCS: INSERT outbox_records (appointment.failed)
        Note over BCS: COMMIT
        BCS-->>GW: 402 Payment Required
        GW-->>-Client: 402 — slot released, no charge
    end

    PS->>PS: INSERT payments (status=captured)
    PS->>PS: INSERT outbox_records (payment.captured)
    PS-->>-BCS: 200 OK {payment_id}

    Note over BCS: Step 5 — Confirm appointment
    Note over BCS: BEGIN TRANSACTION
    BCS->>BCS: INSERT appointment_events (type=confirmed)
    BCS->>BCS: INSERT outbox_records (appointment.confirmed)
    BCS->>BCS: UPDATE booking_idempotency_keys (result=confirmed)
    BCS->>BCS: INCREMENT customer.active_booking_count
    BCS->>BCS: INSERT audit_log
    Note over BCS: COMMIT

    BCS-->>GW: 201 Created {appointment_id, slot_date, slot_start}
    GW-->>-Client: 201 Created — booking confirmed

    Note over Client,SG: Phase C — Async Fan-out (appointment.confirmed)

    loop Every 500ms
        OR->>BCS: Poll outbox_records WHERE published_at IS NULL
    end

    OR->>+SNS: Publish appointment.confirmed<br/>(MessageGroupId=tenant_id)
    SNS-->>-OR: MessageId

    SNS->>NS: SQS: notification-service-appointment-confirmed-queue
    SNS->>DS: SQS: dashboard-service-appointment-confirmed-queue
    SNS->>AnS: SQS: analytics-service-appointment-confirmed-queue
    SNS->>AS: SQS: availability-service-appointment-confirmed-queue

    Note over NS: InboxRecord dedup → read payload (ECST)
    NS->>+SG: Send confirmation email to customer + staff
    SG-->>-NS: 202 Accepted (within 30s of booking — FR004)
    NS->>NS: UPDATE notification_records (status=sent)

    Note over DS: InboxRecord dedup → UPDATE tenant_dashboard_views
    DS->>DS: INCREMENT confirmed_count, ADD net_revenue<br/>SET last_event_id (watermark — NFR009)

    Note over AnS: UPDATE booking velocity window (5-min sliding window)

    Note over AS: Invalidate Redis cache for the confirmed slot
    AS->>Redis: DEL availability:{tenant_id}:{staff_id}:{slot_date}
```

---

## Key Business Rules Enforced

| Step | Rule | Enforcement |
|---|---|---|
| Idempotency check | BR006 — duplicate requests return same result | booking_idempotency_keys unique constraint |
| Customer booking limit | BR008 — max active bookings per customer | BCS checks active_booking_count before saga |
| Booking window | BR010 — no bookings beyond configured window | availability-service excludes future slots |
| Slot lock | BR001 — no double-booking | Unique DB constraint on slot_locks |
| Saga atomicity | BR004 — all steps succeed or all compensate | BCS orchestrates with full compensation |
| Notification isolation | BR007 — notification failure never blocks booking | notification-service is async subscriber |
| Event envelope | BR013 — tenant_id, event_id, correlation_id, occurred_at on all events | EventEnvelope validator |

## Compensation Summary

| Failure Point | Compensation |
|---|---|
| Slot unavailable (BR001) | Transaction rolled back; 409 returned; no charge |
| Payment declined | SlotLock released, appointment.failed published; 402 returned |
| Notification failure | Retry × 4, then DLQ; booking unaffected (BR007) |
