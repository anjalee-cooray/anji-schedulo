---
title: Journey — Platform Operator DLQ Triage
journeyId: UJ006
persona: P4
priority: high
layer: 01-requirements
status: current
lastUpdated: 2026-06-28
---

# R6 · Journey UJ006 — Platform Operator DLQ Triage

## Metadata

| Field | Value |
|---|---|
| Journey ID | UJ006 |
| Name | Platform Operator DLQ Triage |
| Primary Persona | [P4 — Platform Operator](../personas/R4a-persona-P4.md) |
| Priority | High |
| Related Functional Requirements | FR010, FR011 |
| Related Business Rules | BR013, BR014 |
| Architecture Pattern | Dead Letter Queue + Ops Service |
| Services Involved | ops-service |

---

## Goal

A Platform Operator detects a failed event in a dead letter queue, diagnoses the root cause, and resolves the incident — either by re-queuing the corrected event or discarding it with justification — within the 4-hour resolution SLA, with a complete audit trail preserved.

---

## Preconditions

- At least one event has been routed to a named DLQ after exhausting 4 retry attempts.
- The Grafana alert for DLQ depth > 0 has fired and paged the on-call operator.
- The Platform Operator has authenticated into the ops console with `role = platform_operator`.
- The ops-service has read access to all SQS DLQ queues.

---

## Step-by-Step Journey

### Step 1 — Receive Grafana Alert

The Platform Operator receives a PagerDuty notification sourced from a Grafana alert: `DLQ depth > 0 on [queue-name]`. The alert includes:
- Queue name (e.g. `payment-service-appointment-cancelled-queue-dlq`)
- Event type of the first failed message (e.g. `appointment.cancelled`)
- Tenant ID of the affected tenant
- Timestamp of the first failure

**Outcome:** The operator has enough context to open the correct queue in the ops console immediately, without triage guesswork.

---

### Step 2 — Open Ops Console and Inspect Failed Event Payload

The operator opens the ops console and navigates to the DLQ inspector. They filter by the queue name from the alert and open the failed event record. The ops console displays:
- Full event payload (deserialized JSON)
- Failure reason (e.g. schema validation error, downstream timeout, missing envelope field)
- Attempt count (always 4 when a message reaches the DLQ)
- Timestamp of each delivery failure with the error returned by the consumer

**Outcome:** The operator has a complete picture of what the event contains, what failed, and how many times it was attempted.

---

### Step 3 — Identify the Failure Reason

The operator analyses the payload and failure reason to classify the incident:

- **Transient failure** (e.g. consumer was unavailable during all 4 attempts, or a Stripe API timeout occurred): payload is correct, the event can be re-queued directly.
- **Malformed payload** (e.g. missing `correlation_id` or `tenant_id` field, violating BR013): payload must be corrected in the ops console before re-queuing.
- **Systemic failure** (e.g. multiple events of the same type are in the DLQ, all with the same error): re-queuing individual events will not help — the root cause (schema change, downstream service bug, misconfigured consumer) must be addressed first.

**Outcome:** The operator has a classified diagnosis and a clear next action.

---

### Step 4 — Correct the Payload or Acknowledge Root Cause

For transient failures: no payload change is needed — proceed to Step 5.

For malformed payloads: the operator uses the ops console payload editor to add or correct the missing or invalid field, ensuring the corrected event envelope satisfies BR013 (`tenant_id`, `event_id`, `correlation_id`, `occurred_at` all present and valid). The original and corrected payloads are logged in the audit record.

For systemic failures: the operator opens a P1 incident, pages the engineering team, and does not re-queue events until the root cause is confirmed fixed. Events remain in the DLQ during investigation.

**Outcome:** The event payload is correct and safe to re-queue, or the operator has acknowledged the systemic root cause and escalated.

---

### Step 5 — Re-queue the Event for Reprocessing

The operator clicks "Re-queue" in the ops console. ops-service writes the event back to the source SQS queue (not the DLQ). The consumer will process the event on its next poll cycle.

Each re-queue action writes an `audit_log` entry (BR014) recording:
- `operator_id` (the platform operator's user_id)
- `action = re-queue`
- `event_id` of the re-queued event
- `occurred_at`
- `before_snapshot` (original payload)
- `after_snapshot` (corrected payload, if modified)

**Outcome:** The event is back in the consumer's source queue. The audit trail records the operator's action.

---

### Step 6 — Confirm the Event is Processed Successfully

The operator watches the DLQ depth in Grafana. Within one to two minutes, the consumer should process the re-queued event and the DLQ depth should return to zero. The operator verifies by checking:
- DLQ depth returns to 0 for the affected queue
- The downstream effect of the event is visible (e.g. if a `notification.failed` event was re-queued, the NotificationRecord status transitions from `dlq` to `sent`)

If the event fails again after re-queuing, it will re-enter the DLQ and the alert will fire again. The operator escalates to engineering at this point.

**Outcome:** The incident is resolved within SLA. The audit trail is complete. The Grafana alert clears.

---

## Success Outcome

The failed event is resolved within 4 hours of the DLQ alert firing. The event is either successfully reprocessed — with the downstream effect confirmed — or discarded with a documented justification in the audit log. The Grafana DLQ depth metric returns to zero. The operator has a complete audit record of their actions from alert receipt to resolution.

---

## Failure Paths

### Event Cannot Be Re-queued Automatically

If the malformed payload cannot be corrected through the ops console (e.g. the event references an entity that no longer exists, or the failure is in the consumer's business logic, not the payload), re-queuing will cause the event to fail again immediately. The operator must discard the event and coordinate with engineering to apply the fix through a compensating action (e.g. manual data correction in the database).

**Platform behaviour:** Operator selects "Discard" in the ops console with a required justification string. Audit log records the discard. The incident is escalated to a P1 if business data is affected.

### Root Cause Is Systemic

If many events of the same type have accumulated in the DLQ with the same failure reason, re-queuing individual events will not resolve the incident. The operator creates a P1 incident and does not re-queue until the engineering team confirms the consumer has been fixed and redeployed. After the fix, the operator can re-queue all DLQ events in bulk.

**Platform behaviour:** Bulk re-queue via ops-service: reads all events from the DLQ and writes them to the source queue. Each re-queue generates an individual audit log entry. InboxRecord deduplication on the consumer side ensures events already processed before the incident are not processed twice.

### DLQ Alert Does Not Fire

If the Grafana alert for a DLQ is misconfigured or the queue is unnamed and not included in the alert group, a failed event may sit in the DLQ indefinitely without the operator being paged. This represents a monitoring gap — not a failure in the event pipeline itself.

**Resolution:** The Platform Operator reviews DLQ alert coverage quarterly. Every named DLQ in operations.json must have a corresponding Grafana alert in observability.json.

---

## Related Business Rules

| Rule ID | Rule Summary | Application in This Journey |
|---|---|---|
| BR013 | Every event must carry `tenant_id`, `event_id`, `correlation_id`, and `occurred_at`. | Before re-queuing a corrected event, the operator must verify all four envelope fields are present and valid. The EventEnvelope validator in the consumer will reject the event again if any field is missing. |
| BR014 | All write operations that modify tenant-scoped data must produce an immutable audit log entry. | Every re-queue and discard action by the Platform Operator writes an `audit_log` record. The audit table has INSERT-only privileges — the operator cannot delete their own audit entries. |

---

## Implementation Note

ops-service polls all named DLQ queues and surfaces entries in the ops console with full payload deserialization. The console provides a payload editor for correcting malformed events, restricted to fields that do not alter the event's identity (the `event_id` is immutable). Re-queue writes to the source SQS queue (not the DLQ) using the SQS `SendMessage` API with the same `MessageGroupId = tenant_id` as the original message to preserve FIFO ordering per tenant. Consumer-side InboxRecord deduplication on `(event_id, consumer_name)` ensures that if the event was already processed before the operator's re-queue action, the consumer skips it silently — making re-queue always safe to attempt.
