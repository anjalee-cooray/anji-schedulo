---
title: Flow Spec — Platform Operator DLQ Triage
journeyId: UJ006
layer: 02-design
status: current
lastUpdated: 2026-06-28
---

# D2 · Flow Spec — UJ006: Platform Operator DLQ Triage

## Header

| Field | Value |
|---|---|
| Journey ID | UJ006 |
| Journey Name | Platform Operator DLQ Triage |
| Persona | P4 — Platform Operator |
| Priority | High |
| Related Functional Requirements | FR010, FR011 |
| Related Business Rules | BR013, BR014 |

---

## 1. Technical Flow Overview

**Architecture Pattern: Dead Letter Queue + Ops Service + Idempotent Re-queue**

Dead Letter Queues (DLQs) are the system's backstop for events that cannot be processed after exhausting all retry attempts. Every consumer service has a dedicated DLQ, and the Platform Operator (P4) is the human-in-the-loop for resolving events that reach them.

The key design principle is that **DLQ events are a recoverable state, not data loss.** The original event payload is fully preserved in the SQS DLQ message. The ops-service provides a structured triage interface that allows the operator to:

1. Inspect the full event payload and failure history.
2. Re-queue a corrected message to the source consumer queue (not the DLQ) for reprocessing.
3. Discard an event that is invalid, stale, or already manually resolved.

**Idempotency is the key safety guarantee for re-queuing.** Every consumer uses the InboxRecord pattern — if an event was already successfully processed before reaching the DLQ, re-queuing it is safe because the consumer will detect the duplicate `event_id` in `inbox_records` and skip processing without side effects.

---

## 2. Services Involved

| Service | Role in This Flow |
|---|---|
| ops-service | Central DLQ inspection, re-queue, and discard interface. Writes AuditLog for all operator actions. |
| All consumer services | Indirectly involved — their DLQs contain the stranded events. Not directly called during triage. |
| notification-service | Produces `notification.failed` events routed to `ops-service-notification-failed-queue` — a specialised DLQ for notification failures. |
| api-gateway | Validates operator JWT (role = platform_operator), routes to ops-service |
| Amazon SQS | The DLQ queues and source queues managed during triage |

---

## 3. DLQ Inventory

The following DLQs are monitored. Each DLQ is named `{source-queue}-dlq` and has `maxReceiveCount = 4` on its paired source queue:

| DLQ Name | Source Queue | Consumer Service |
|---|---|---|
| `notification-service-tenant-provisioned-queue-dlq` | `notification-service-tenant-provisioned-queue` | notification-service |
| `billing-service-tenant-provisioned-queue-dlq` | `billing-service-tenant-provisioned-queue` | billing-service |
| `notification-service-appointment-confirmed-queue-dlq` | `notification-service-appointment-confirmed-queue` | notification-service |
| `payment-service-appointment-cancelled-queue-dlq` | `payment-service-appointment-cancelled-queue` | payment-service |
| `notification-service-appointment-cancelled-queue-dlq` | `notification-service-appointment-cancelled-queue` | notification-service |
| `booking-command-service-payment-captured-queue-dlq` | `booking-command-service-payment-captured-queue` | booking-command-service |
| `booking-command-service-payment-failed-queue-dlq` | `booking-command-service-payment-failed-queue` | booking-command-service |
| `ops-service-notification-failed-queue` | (feeds from notification.failed SNS topic) | ops-service |
| `dashboard-service-appointment-confirmed-queue-dlq` | `dashboard-service-appointment-confirmed-queue` | dashboard-service |

---

## 4. Step-by-Step Technical Flow

### Phase A — Continuous DLQ Monitoring (Background)

**Step 1 — ops-service polls DLQ depth**

ops-service runs a scheduled background job every 60 seconds:
```
for each dlq_name in DLQ_INVENTORY:
    depth = sqs.get_queue_attributes(QueueUrl=dlq_url, AttributeNames=['ApproximateNumberOfMessages'])
    if depth > 0:
        emit metric: dlq.depth{queue=dlq_name} = depth
```

The metric is emitted to CloudWatch and scraped by Prometheus (Grafana LGTM stack).

**Step 2 — Grafana alert fires**

A Grafana alert rule triggers within 5 minutes of a DLQ receiving its first message (FR010, NFR016):

```
ALERT: "DLQ depth > 0"
Condition: max(dlq_depth{queue=~".*-dlq"}) > 0
For: 5 minutes
Severity: P2
Channels: PagerDuty, Slack #ops-incidents
Runbook: https://internal.docs/runbooks/dlq-triage
```

The alert fires with the queue name and the current depth. If multiple DLQs have messages, one alert fires per DLQ.

---

### Phase B — Alert Receipt and Triage (Operator-Driven)

**Step 3 — Operator receives alert**

The Platform Operator (P4) receives a PagerDuty page and a Slack notification containing:
- DLQ name
- Current message depth
- Tenant IDs affected (from message group IDs, if immediately readable)
- Runbook link

Resolution SLA: **4 hours** from alert receipt (NFR016).

**Step 4 — Operator opens the ops console and inspects DLQ**

The operator calls:
```
GET /api/v1/ops/dlq/{queue_name}
Authorization: Bearer <JWT>   (role = platform_operator)
```

ops-service calls SQS `ReceiveMessage` with `MaxNumberOfMessages = 10, VisibilityTimeout = 300s` (5-minute window for inspection before the message becomes available to other consumers).

For each message received, ops-service returns:
```json
{
  "dlq_entry_id": "<SQS message receipt handle>",
  "queue_name": "payment-service-appointment-cancelled-queue-dlq",
  "approximate_receive_count": 4,
  "first_enqueued_at": "2026-06-28T07:32:00Z",
  "message_body": {
    "event_id": "...",
    "event_type": "appointment.cancelled",
    "tenant_id": "...",
    "correlation_id": "...",
    "occurred_at": "...",
    "appointment_id": "...",
    "failure_reason": "Stripe API timeout — HTTP 524 after 30s",
    "attempt_count": 4
  },
  "inbox_status": "not_processed"  // ops-service checks inbox_records for this event_id
}
```

**`inbox_status`** is a critical field: ops-service queries `inbox_records WHERE event_id = $event_id AND consumer_name = $consumer_service`. If `processed` is returned, re-queuing is safe but will be a no-op (InboxRecord will reject the duplicate). If `not_processed`, re-queuing will trigger actual reprocessing.

**Step 5 — Operator identifies failure reason**

The operator reviews `failure_reason` and the event payload. Common failure patterns:

| Pattern | Failure Reason | Recommended Action |
|---|---|---|
| Downstream service timeout | `Stripe API timeout`, `SendGrid 503` | Re-queue after confirming downstream is healthy |
| Invalid event envelope | Missing `tenant_id` or `correlation_id` | Fix payload, re-queue — or discard if the event source had a bug |
| Business logic rejection | `tenant_suspended`, `customer_not_found` | Investigate business state, then discard or re-queue |
| Poison pill | Repeatedly fails with no clear cause | Discard with documented reason, file bug ticket |
| Already processed | `inbox_status = processed` | Discard safely — no reprocessing needed |

---

### Phase C — Resolution: Re-queue (Option A)

**Step 6A — Operator re-queues the event**

The operator clicks "Re-queue" in the ops console with an optional corrected payload (e.g. if the original payload had a malformed field):
```
POST /api/v1/ops/dlq/{queue_name}/{dlq_entry_id}/requeue
Authorization: Bearer <JWT>
{
  "corrected_payload": { ... },  // optional — omit to re-queue original
  "operator_note": "Stripe was down at 07:30 UTC; now recovered. Re-queuing for retry."
}
```

ops-service executes:
1. Validates the corrected payload (or uses the original) against the EventEnvelope schema. If `tenant_id`, `event_id`, `correlation_id`, or `occurred_at` are missing → rejects with `422 Unprocessable Entity` (BR013).
2. Posts the message to the **source queue** (not the DLQ):
   ```
   sqs.send_message(
       QueueUrl = source_queue_url,
       MessageBody = corrected_payload,
       MessageGroupId = tenant_id,
       MessageDeduplicationId = event_id + "-requeued"
   )
   ```
   Using the original `event_id` in `MessageDeduplicationId` prevents SQS FIFO from deduplicating the re-queued message against itself, while the InboxRecord check in the consumer handles end-to-end deduplication.
3. Deletes the original message from the DLQ:
   ```
   sqs.delete_message(QueueUrl = dlq_url, ReceiptHandle = $dlq_entry_id)
   ```
4. Writes AuditLog entry:
   ```
   entity_type = 'DLQEvent', entity_id = event_id,
   operation = 'replay',
   actor_id = $operator_user_id, actor_role = 'operator',
   occurred_at = NOW(),
   before_snapshot = { original_payload, failure_reason, attempt_count },
   after_snapshot  = { corrected_payload, operator_note, requeued_to = source_queue_name }
   ```

**Step 7A — Consumer service processes the re-queued event**

The consumer service dequeues the message from its source queue:
1. InboxRecord deduplication check: attempts `INSERT INTO inbox_records (event_id, consumer_name)`.
   - If `event_id` already in `inbox_records` (was previously processed before the DLQ): INSERT fails → **message deleted, no side effects.** The DLQ entry was a false alarm — the event was already processed.
   - If not in `inbox_records`: INSERT succeeds → **event processed normally.**
2. Business logic executes (e.g. payment-service initiates Stripe refund).
3. On success: SQS message deleted, result logged.
4. On failure: message returns to source queue. After `maxReceiveCount = 4` attempts, it routes back to the DLQ. Operator receives a new alert.

---

### Phase C — Resolution: Discard (Option B)

**Step 6B — Operator discards the event**

When the event is invalid, stale, or the situation has been manually resolved:
```
POST /api/v1/ops/dlq/{queue_name}/{dlq_entry_id}/discard
Authorization: Bearer <JWT>
{
  "operator_note": "Tenant was deprovisioned manually. Refund processed directly in Stripe Dashboard. Event no longer relevant."
}
```

ops-service executes:
1. Deletes the message from the DLQ:
   ```
   sqs.delete_message(QueueUrl = dlq_url, ReceiptHandle = $dlq_entry_id)
   ```
2. Writes AuditLog entry:
   ```
   operation = 'discard',
   actor_id = $operator_user_id, actor_role = 'operator',
   after_snapshot = { event_id, queue_name, operator_note }
   ```

No event is re-published. The DLQ entry is gone. The audit trail preserves why it was discarded.

---

## 5. Compensation / Failure Flows

### Failure Point 1 — Re-queued event fails again and returns to DLQ

**Impact:** The consumer service still cannot process the event after re-queuing. The event routes back to the DLQ after 4 more failures. The attempt_count will now reflect the additional attempts.

**Resolution:** A new Grafana alert fires. Operator investigates whether the root cause has actually been resolved. If the issue is systemic (e.g. the payment-service has a code bug), a service fix and deployment may be required before re-queuing is useful.

### Failure Point 2 — Operator cannot reach ops console

**Impact:** DLQ triage is blocked. Alert SLA clock is running.

**Resolution:** Escalate to infrastructure team. The DLQ messages are safely retained in SQS (default retention: 14 days). No data is at risk. The operator can use the AWS Console or AWS CLI directly to inspect and re-queue messages as a manual fallback:
```bash
aws sqs receive-message --queue-url $DLQ_URL
aws sqs send-message --queue-url $SOURCE_QUEUE_URL --message-body "$PAYLOAD"
aws sqs delete-message --queue-url $DLQ_URL --receipt-handle "$HANDLE"
```

All manual actions must be followed up with an AuditLog entry via `POST /ops/audit` once the ops console is available.

### Failure Point 3 — Systemic failure (many events in the same DLQ)

**Impact:** A large backlog indicates a systemic issue — a downstream service is down, a schema change broke message parsing, or a business rule changed without backward compatibility.

**Resolution:**
1. Do not re-queue in bulk until root cause is identified. Re-queuing into a broken consumer will loop messages back to the DLQ immediately and increase noise.
2. Fix the underlying issue (service deploy, schema migration, config change).
3. Use the replay scope tools (FR011) to re-publish events to the source queue in controlled batches after the fix.
4. Monitor DLQ depth as the backlog drains.

### Failure Point 4 — SQS VisibilityTimeout expires during inspection

**Impact:** The message becomes visible to other ops-service poll cycles before the operator takes action. Another ops-service instance may read the same message.

**Resolution:** The visibility timeout is set to 300 seconds (5 minutes). If an operator is actively triaging a message and the timeout expires, ops-service will receive a `MessageNotInFlight` error when attempting to delete the message. The operator should reload the DLQ console — the message will reappear and can be re-triaged.

---

## 6. Events Produced

ops-service produces **no domain events** during DLQ triage. All operator actions are captured in `AuditLog` entries only. The `replay.started` and `replay.completed` events are produced only when a full tenant event replay (FR011) is triggered — not for individual DLQ re-queues.

---

## 7. Business Rules Enforced

| BR ID | Rule Summary | Enforcement Point |
|---|---|---|
| BR013 | Every event must carry `tenant_id`, `event_id`, `correlation_id`, `occurred_at` | ops-service: EventEnvelope validator applied to any corrected payload before re-queuing. Invalid envelope rejected with 422. |
| BR014 | All write operations on tenant-scoped data must produce an immutable audit log entry | ops-service: AuditLog INSERT for every re-queue and discard action, recording the operator identity, timestamp, original payload, and operator note. |

---

## 8. Consistency Guarantees

| Concern | Consistency Model | Detail |
|---|---|---|
| DLQ message deletion | Strong (SQS) | SQS delete is idempotent; receipt handle ensures only the held message is deleted |
| Re-queue to source queue | At-least-once | SQS guarantees delivery; consumer InboxRecord handles any duplicate |
| AuditLog for operator actions | Strong | AuditLog is written synchronously in the same service call as the DLQ operation |
| DLQ depth metric | Eventually consistent | ops-service polls every 60 seconds; metric reflects state at last poll |

---

## 9. Idempotency

### Re-queue idempotency

If the operator re-queues the same DLQ entry twice (e.g. due to a double-click or a session timeout), the second SQS delete will fail with `ReceiptHandleIsInvalid` (the handle is no longer valid after the first delete succeeds). The second re-queue message sent to the source queue will be processed by the consumer, which checks InboxRecord. Since `event_id` is already in `inbox_records` from the first re-queuing (assuming it succeeded), the consumer will skip processing.

### Discard idempotency

A discard operation deletes the SQS message by receipt handle. A second discard attempt on the same handle returns `ReceiptHandleIsInvalid` from SQS — this is safe and expected. ops-service handles this error gracefully and confirms the discard as already complete.

### AuditLog idempotency

AuditLog entries use the operator's correlation_id from their JWT request. If an operator's request is retried, the AuditLog may contain two entries for the same action. This is acceptable — AuditLog is designed to be append-only (BR014) and a duplicate entry causes no operational harm. The `correlation_id` allows the duplicate to be identified if needed.
