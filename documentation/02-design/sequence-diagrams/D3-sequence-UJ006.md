---
title: Sequence Diagram — Platform Operator DLQ Triage
journeyId: UJ006
layer: 02-design
status: current
lastUpdated: 2026-06-28
---

# D3 · Sequence Diagram — UJ006: Platform Operator DLQ Triage

## Overview

This diagram shows the full lifecycle of a dead letter queue (DLQ) event: from initial consumer failure through automated Grafana alerting, to the Platform Operator inspecting the failure payload, and resolving it via either re-queue (with idempotent reprocessing) or discard (with audit trail). All operator actions produce immutable audit log entries (BR014).

---

```mermaid
sequenceDiagram
    participant Consumer as Consumer Service<br/>(e.g. payment-service)
    participant SQS as SQS (source queue)
    participant DLQ as SQS DLQ
    participant Grafana as Grafana
    participant Operator as Platform Operator (P4)
    participant GW as api-gateway
    participant OPS as ops-service
    participant DB as PostgreSQL<br/>(audit_log)

    Note over Consumer,DB: Phase A — Consumer Failure → DLQ Routing

    SQS->>+Consumer: Deliver message (appointment.cancelled)
    Consumer->>Consumer: Attempt processing (attempt 1)
    Consumer-->>SQS: Negative acknowledgement (processing failed)
    Note over SQS,Consumer: SQS makes message invisible for backoff period

    loop Retry × 4 (exponential backoff: 1s, 2s, 4s, 8s)
        SQS->>Consumer: Re-deliver message
        Consumer->>Consumer: Attempt processing
        Consumer-->>SQS: Negative acknowledgement
    end

    Note over Consumer: 4 retries exhausted — maxReceiveCount reached
    SQS->>DLQ: Route message to payment-service-appointment-cancelled-queue-dlq<br/>(full message payload preserved)
    SQS-->>-Consumer: Message moved to DLQ

    Note over DLQ,Grafana: Phase B — Automated Alerting

    Grafana->>DLQ: Poll DLQ depth metric (CloudWatch)<br/>every 60 seconds
    DLQ-->>Grafana: ApproximateNumberOfMessages > 0

    Grafana->>Operator: ALERT: P2 — DLQ depth > 0<br/>Queue: payment-service-appointment-cancelled-queue-dlq<br/>Runbook: /docs/06-operations/O4-runbook.md<br/>(within 5 minutes of message arrival — FR010)

    Note over Operator,DB: Phase C — Operator Inspection

    Operator->>+GW: GET /ops/dlq<br/>?queue=payment-service-appointment-cancelled-queue-dlq<br/>Bearer JWT (role=platform_operator)
    GW->>GW: Validate JWT<br/>Verify role=platform_operator (cross-tenant read permitted)
    GW->>+OPS: Forward DLQ inspection request

    OPS->>+DLQ: ReceiveMessage (peek — visibility timeout applied)
    DLQ-->>-OPS: Message payload {<br/>  event_id, tenant_id, appointment_id,<br/>  failure_reason, attempt_count,<br/>  original_payload,<br/>  first_failed_at, last_failed_at<br/>}
    OPS-->>-GW: 200 OK {dlq_entries[]}
    GW-->>-Operator: DLQ entries with full payload

    Operator->>Operator: Inspect failure_reason<br/>(e.g. "stripe_api_timeout", "malformed_payload", "dependency_unavailable")

    Note over Operator,DB: Phase D-A — Resolution: Re-queue

    alt Operator decides to re-queue (root cause resolved)
        Operator->>+GW: POST /ops/dlq/{message_id}/requeue<br/>Bearer JWT (role=platform_operator)<br/>{corrected_payload (optional)}
        GW->>+OPS: Forward re-queue request

        OPS->>+SQS: SendMessage to source queue<br/>(payment-service-appointment-cancelled-queue)<br/>with corrected payload, MessageGroupId=tenant_id
        SQS-->>-OPS: MessageId

        OPS->>+DLQ: DeleteMessage (remove from DLQ)
        DLQ-->>-OPS: 200 OK

        OPS->>+DB: INSERT audit_log {<br/>  entity_type: DLQEntry,<br/>  entity_id: message_id,<br/>  operation: requeue,<br/>  actor_id: operator_user_id,<br/>  actor_role: operator,<br/>  occurred_at,<br/>  after_snapshot: {corrected_payload, target_queue}<br/>}
        DB-->>-OPS: Inserted

        OPS-->>-GW: 200 OK {message_id, status: requeued}
        GW-->>-Operator: 200 OK — message re-queued

        Note over SQS,Consumer: Consumer receives re-queued message
        SQS->>+Consumer: Deliver corrected message
        Consumer->>Consumer: INSERT inbox_records (event_id, consumer=payment-service)<br/>— if duplicate (already processed): skip idempotently
        Consumer->>Consumer: Process message successfully
        Consumer-->>-SQS: Positive acknowledgement (delete)
    end

    Note over Operator,DB: Phase D-B — Resolution: Discard

    alt Operator decides to discard (message is unrecoverable / obsolete)
        Operator->>+GW: POST /ops/dlq/{message_id}/discard<br/>Bearer JWT (role=platform_operator)<br/>{reason: "appointment already refunded manually via Stripe Dashboard"}
        GW->>+OPS: Forward discard request

        OPS->>+DLQ: DeleteMessage
        DLQ-->>-OPS: 200 OK

        OPS->>+DB: INSERT audit_log {<br/>  entity_type: DLQEntry,<br/>  entity_id: message_id,<br/>  operation: discard,<br/>  actor_id: operator_user_id,<br/>  actor_role: operator,<br/>  occurred_at,<br/>  after_snapshot: {discard_reason}<br/>}
        DB-->>-OPS: Inserted

        OPS-->>-GW: 200 OK {message_id, status: discarded}
        GW-->>-Operator: 200 OK — message discarded with audit trail
    end

    Note over Operator,DB: Phase E — Verify Resolution
    Operator->>Grafana: Confirm DLQ depth = 0 (queue cleared)
    Grafana->>DLQ: Poll CloudWatch metric
    DLQ-->>Grafana: ApproximateNumberOfMessages = 0
    Grafana->>Operator: Alert resolved — DLQ clear
```

---

## Idempotency Safety on Re-queue

When a message is re-queued, the consumer may have already processed the original message partially (which is why it ended up in the DLQ). The InboxRecord pattern guarantees safety:

1. The consumer attempts `INSERT INTO inbox_records (event_id, consumer_name)`.
2. If the insert fails (unique constraint) — the event was already processed successfully before the failure. No re-processing occurs; the message is deleted.
3. If the insert succeeds — the event was never fully processed. Processing proceeds normally.

This means re-queuing is always safe, regardless of which retry number caused the DLQ routing.

---

## Key Business Rules and NFRs

| Rule | Enforcement Point |
|---|---|
| BR014 — immutable audit log on all operator actions | audit_log INSERT for every re-queue or discard action |
| BR013 — event envelope validation | OPS validates event_id, tenant_id, correlation_id before re-queue |
| FR010 — DLQ alert within 5 minutes | Grafana CloudWatch alarm on ApproximateNumberOfMessages > 0 |
| NFR016 — resolution SLA 4 hours | Incident SLA tracked via Grafana alert open time |
| Cross-tenant access | Gated on platform_operator role — the only role with cross-tenant read (security.json) |
