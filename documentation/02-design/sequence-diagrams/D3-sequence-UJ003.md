---
title: Sequence Diagram — Appointment Cancellation
journeyId: UJ003
layer: 02-design
status: current
lastUpdated: 2026-06-28
---

# D3 · Sequence Diagram — UJ003: Appointment Cancellation

## Overview

This diagram shows the cancellation flow using Saga Choreography. The booking-command-service handles the synchronous cancellation validation and slot release atomically. It then publishes `appointment.cancelled`, which fans out to independent async subscribers — payment-service (refund), notification-service (cancellation notices), availability-service (cache invalidation), dashboard-service and analytics-service (metrics). Critically, refund failure (BR009) does not block slot release or notifications — it routes to DLQ for manual ops resolution.

---

```mermaid
sequenceDiagram
    participant Client as Client (Browser)
    participant GW as api-gateway
    participant BCS as booking-command-service
    participant OR as outbox-relay
    participant SNS as SNS
    participant PS as payment-service
    participant Stripe as Stripe
    participant NS as notification-service
    participant AS as availability-service
    participant DS as dashboard-service
    participant AnS as analytics-service
    participant OPS as ops-service
    participant SG as SendGrid
    participant Redis as Redis (ElastiCache)
    participant DLQPS as SQS DLQ (payment)

    Note over Client,Redis: Phase A — Cancellation Request (Synchronous)

    Client->>+GW: POST /appointments/{appointment_id}/cancel<br/>Bearer JWT (role=customer or tenant_admin)
    GW->>GW: Validate JWT, extract tenant_id + customer_id
    GW->>+BCS: Forward cancellation request

    Note over BCS: Step 1 — Load appointment and validate ownership
    BCS->>BCS: SELECT appointment WHERE appointment_id AND tenant_id (RLS — BR005)
    BCS->>BCS: Verify caller owns appointment OR role=tenant_admin

    Note over BCS: Step 2 — Cancellation policy check (BR002)
    BCS->>BCS: Compute: slot_start - NOW() vs cancellation_hours (TenantConfig)

    alt Outside cancellation window (BR002 violated)
        BCS-->>GW: 422 Unprocessable Entity<br/>{error: "cancellation_window_expired",<br/>cancellation_deadline: ISO8601}
        GW-->>-Client: 422 — cancellation not permitted
    end

    Note over BCS: Step 3 — Atomic cancellation (slot release + event write)
    Note over BCS: BEGIN TRANSACTION
    BCS->>BCS: UPDATE slot_locks SET released_at=NOW()<br/>WHERE appointment_id AND released_at IS NULL
    BCS->>BCS: INSERT appointment_events (type=cancelled,<br/>cancelled_by_role, cancelled_by_id, occurred_at)
    BCS->>BCS: DECREMENT customer.active_booking_count
    BCS->>BCS: INSERT outbox_records (appointment.cancelled,<br/>full payload including payment_id, amount_eligible_for_refund)
    BCS->>BCS: INSERT audit_log (op=cancel, before/after snapshots)
    Note over BCS: COMMIT — slot is now free (< 10s target)

    BCS-->>-GW: 200 OK {appointment_id, status: cancelled}
    GW-->>-Client: 200 OK — cancellation confirmed

    Note over Client,Redis: Phase B — Async Choreography Fan-out (appointment.cancelled)

    loop Every 500ms
        OR->>BCS: Poll outbox_records WHERE published_at IS NULL
    end

    OR->>+SNS: Publish appointment.cancelled<br/>(MessageGroupId=tenant_id)
    SNS-->>-OR: MessageId
    OR->>BCS: UPDATE outbox_records SET published_at=NOW()

    SNS->>PS: SQS: payment-service-appointment-cancelled-queue
    SNS->>NS: SQS: notification-service-appointment-cancelled-queue
    SNS->>AS: SQS: availability-service-appointment-cancelled-queue
    SNS->>DS: SQS: dashboard-service-appointment-cancelled-queue
    SNS->>AnS: SQS: analytics-service-appointment-cancelled-queue

    Note over PS,DLQPS: Refund path (independent of notification / slot release — BR009)
    PS->>PS: INSERT inbox_records (event_id, consumer=payment-service)
    PS->>PS: SELECT payment WHERE appointment_id AND status=captured
    PS->>PS: Check: amount_eligible_for_refund > 0

    alt No payment captured (in-person payment or no charge)
        PS->>PS: No refund required — process complete
    end

    PS->>+Stripe: POST /v1/refunds<br/>{payment_intent_id, amount, idempotency_key=appointment_id}
    Stripe-->>-PS: Refund response

    alt Refund succeeds
        PS->>PS: UPDATE payments SET status=refunded, refunded_at=NOW()
        PS->>PS: INSERT outbox_records (payment.refunded)
        OR->>SNS: Publish payment.refunded
        SNS->>NS: SQS: notification-service-payment-refunded-queue
        NS->>SG: Send refund confirmation email to customer
    end

    alt Refund fails (Stripe error or timeout)
        Note over PS,DLQPS: BR009 — refund failure MUST NOT block slot release or notification
        PS->>PS: UPDATE payments SET status=refund_failed
        PS->>PS: Retry × 4 with exponential backoff (1s, 2s, 4s, 8s)
        PS->>DLQPS: Route to payment-service-appointment-cancelled-queue-dlq
        DLQPS->>OPS: ops-service monitors DLQ depth
        OPS->>OPS: Alert raised (Grafana — DLQ depth > 0)
        Note over OPS: Operator inspects via /ops/dlq<br/>Manual Stripe refund or re-queue
    end

    Note over NS: Cancellation notification (runs independently of refund — BR007)
    NS->>NS: INSERT inbox_records (event_id, consumer=notification-service)
    NS->>NS: Read customer_snapshot, staff_snapshot from payload (ECST)
    NS->>+SG: Send cancellation notice to customer
    SG-->>-NS: 202 Accepted
    NS->>SG: Send cancellation notice to staff member
    NS->>NS: INSERT notification_records (status=sent)

    Note over AS: Slot re-opened (< 10s target — FR005)
    AS->>AS: INSERT inbox_records (event_id, consumer=availability-service)
    AS->>Redis: DEL availability:{tenant_id}:{staff_id}:{slot_date}
    Note over AS: Next availability query will re-compute including freed slot

    Note over DS: Update dashboard metrics
    DS->>DS: INSERT inbox_records (event_id, consumer=dashboard-service)
    DS->>DS: UPDATE tenant_dashboard_views<br/>INCREMENT cancelled_count<br/>RECALCULATE cancellation_rate<br/>SET last_event_id (idempotency watermark)

    Note over AnS: Update cancellation rate window
    AnS->>AnS: INSERT inbox_records (event_id, consumer=analytics-service)
    AnS->>AnS: UPDATE cancellation rate (rolling 50-booking window)
```

---

## Key Business Rules Enforced

| Step | Rule | Enforcement |
|---|---|---|
| Cancellation policy | BR002 — cancellation only within configured window | BCS validates slot_start - NOW() vs cancellation_hours |
| Slot release | BR009 — refund failure must not block slot release | Slot released in same transaction as appointment.cancelled; refund is async and independent |
| Refund failure | BR009 — routes to DLQ, never blocks booking operations | payment-service DLQ + ops alert; slot and notifications proceed regardless |
| Tenant isolation | BR005 — appointment must belong to caller's tenant | RLS on all queries; tenant_id enforced at api-gateway (JWT) and DB level |
| Audit log | BR014 — immutable record of cancellation | audit_log INSERT in same transaction as slot release |
| Event envelope | BR013 — tenant_id, event_id, correlation_id, occurred_at | EventEnvelope validator |
| Notification isolation | BR007 — notification failure never blocks business operations | notification-service is fully decoupled async subscriber |

## Success Targets (from FR005)

| Outcome | Target |
|---|---|
| Slot re-opened | Within 10 seconds of cancellation |
| Customer notified | Within 30 seconds |
| Refund initiated | Within 60 seconds (if applicable) |
