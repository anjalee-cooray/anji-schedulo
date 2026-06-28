---
title: User Stories — Platform Operator DLQ Triage
journeyId: UJ006
layer: 02-design
status: current
lastUpdated: 2026-06-28
---

# D12 · User Stories — Platform Operator DLQ Triage (UJ006)

## Epic Header

| Field | Value |
|---|---|
| Epic Name | Platform Operator DLQ Triage |
| Journey ID | UJ006 |
| Primary Persona | P4 — Platform Operator |
| Related FRs | FR010, FR011 |
| Related BRs | BR013, BR014 |
| Priority | High |
| Architecture Pattern | Dead Letter Queue, Replay Pattern, Idempotent Consumer |

## Epic Description

As a **Platform Operator**, I want to be alerted immediately when events fail after all retries, inspect the failed payloads in an ops console, and resolve them by re-queuing or discarding — so that stuck events are resolved within the 4-hour SLA with a full audit trail preserved.

---

## User Stories

---

### US-UJ006-01 · Receive a DLQ alert within 5 minutes of failure

**Title:** Get an automated alert within 5 minutes when any consumer exhausts its retries

**User Story:**
As a **Platform Operator**, I want to receive an automated alert within 5 minutes when a consumer routes a message to the Dead Letter Queue, so that I can begin triage before the issue affects tenants or cascades into other systems.

**Acceptance Criteria:**

- **Given** a consumer service exhausts all 4 retry attempts on a message (`maxReceiveCount = 4`),  
  **When** the message is moved to the DLQ,  
  **Then** a Grafana alert fires within 5 minutes of the DLQ message arriving (FR010).

- **Given** the alert fires,  
  **When** I receive the PagerDuty notification,  
  **Then** it includes the DLQ name, the approximate message count, and a link to the ops console filtered to that queue.

- **Given** the alert fires,  
  **When** I open the ops console,  
  **Then** I can see all DLQ entries filterable by `tenant_id`, `event_type`, and `failure_reason` without requiring a direct AWS Console login.

- **Given** a DLQ entry is resolved (re-queued or discarded),  
  **When** the DLQ depth returns to zero,  
  **Then** the Grafana alert resolves automatically — no manual alert dismissal is required.

**Size:** M  
**Priority:** High  
**Related:** FR010

**Technical Notes:**
- Grafana alert condition: `aws_sqs_approximate_number_of_messages_not_visible{queue=~".*-dlq"} > 0` for 5 minutes → P2 alert.
- ops-service polls all named DLQ queues every 60 seconds and syncs entries to its internal `dlq_entries` table for the ops console UI.
- The ops console filters are server-side: `GET /api/ops/dlq?tenant_id=&event_type=&failure_reason=` — protected by `platform_operator` role JWT.
- Named DLQs (from `specs/ai/operations.json`) are listed explicitly: one DLQ per consumer service per event type.

---

### US-UJ006-02 · Inspect the full failed event payload

**Title:** View the complete event payload and failure metadata for a DLQ entry

**User Story:**
As a **Platform Operator**, I want to see the full event payload and failure details for any DLQ entry, so that I can diagnose whether the failure was caused by a malformed payload, a missing envelope field, a downstream timeout, or an application bug.

**Acceptance Criteria:**

- **Given** a DLQ entry exists in the ops console,  
  **When** I open the entry detail view,  
  **Then** I see: `event_id`, `tenant_id`, `event_type`, `failure_reason`, `attempt_count`, `first_failed_at`, `last_failed_at`, and the full JSON event body formatted for readability.

- **Given** the `failure_reason` is `invalid_envelope`,  
  **When** I inspect the payload,  
  **Then** the ops console highlights which required envelope field (`tenant_id`, `event_id`, `correlation_id`, or `occurred_at`) is missing — corresponding to the BR013 validation failure.

- **Given** I search for DLQ entries by `tenant_id`,  
  **When** results are returned,  
  **Then** only entries for that specific tenant are shown — the ops console enforces tenant scoping on search results even for operators (defence in depth).

- **Given** the entry contains a `payment_id`,  
  **When** I view it,  
  **Then** the ops console renders a direct link to the Stripe Dashboard for that PaymentIntent ID — enabling fast manual resolution if a refund is stuck.

**Size:** M  
**Priority:** High  
**Related:** FR010, BR013

**Technical Notes:**
- ops-service stores DLQ entry metadata (not the full payload) in its `dlq_entries` table; the full payload is fetched from SQS on-demand when the operator opens the detail view.
- BR013 envelope validation occurs in each consumer on message receipt; if the envelope is invalid, the consumer immediately sends the message to DLQ without retrying (retry would always fail the same check).
- The ops console requires a `platform_operator` role JWT — all operator actions are gated at the api-gateway level before reaching ops-service.
- PII fields in the event payload (`customer_name`, `customer_email`) are masked in the ops console UI unless the operator has explicit PII access rights configured in the ops role definition.

---

### US-UJ006-03 · Re-queue a corrected event for reprocessing

**Title:** Post a corrected event back to the source SQS queue for reprocessing

**User Story:**
As a **Platform Operator**, I want to edit and re-queue a failed event so that the consumer can process it correctly, without having to redeploy any services or manually intervene in the SQS console.

**Acceptance Criteria:**

- **Given** I have diagnosed the root cause of a DLQ failure,  
  **When** I click Re-queue (with or without payload edits) in the ops console,  
  **Then** the event is sent to the source SQS queue and deleted from the DLQ in a single atomic operation.

- **Given** the re-queued event is consumed by the target service,  
  **When** the consumer processes it,  
  **Then** the InboxRecord deduplication check ensures the event is processed at most once — even if it was partially processed before landing in the DLQ (BR006-style idempotency).

- **Given** every re-queue action,  
  **When** it is executed by the operator,  
  **Then** an immutable `AuditLog` entry is written with: `operator_id`, `occurred_at`, `dlq_entry_id`, `original_event_id`, `action = re_queue`, and any payload changes made (BR014).

- **Given** a re-queued event causes the consumer to fail again,  
  **When** retries are exhausted,  
  **Then** the event routes back to the DLQ and a new alert fires — the original DLQ entry is marked as `requeued` in the ops console for traceability.

**Size:** M  
**Priority:** High  
**Related:** FR010, BR014

**Technical Notes:**
- ops-service calls `sqs:SendMessage` on the source queue and `sqs:DeleteMessage` on the DLQ in sequence; if the DLQ delete fails, the entry remains in the DLQ (safe — consumer InboxRecord prevents double processing).
- The re-queue endpoint: `POST /api/ops/dlq/{entry_id}/requeue` with an optional `payload_override` body — requires `platform_operator` JWT.
- `AuditLog` entry is written by ops-service in its own database transaction before calling SQS, so the audit record exists even if the SQS call fails.
- ops-service stamps the re-queued message with `X-Requeue-Operator-Id` and `X-Requeue-Job-Id` message attributes for traceability in the consumer's logs.

---

### US-UJ006-04 · Discard an unresolvable DLQ entry

**Title:** Remove a DLQ entry that cannot be reprocessed, with a full audit trail

**User Story:**
As a **Platform Operator**, I want to discard a DLQ entry that references a deleted tenant or a permanently invalid payload, so that the DLQ is cleared and alerts are resolved — with a full record of the decision preserved.

**Acceptance Criteria:**

- **Given** an event cannot be reprocessed (e.g. the referenced tenant has been deprovisioned),  
  **When** I click Discard with a mandatory reason field,  
  **Then** the event is deleted from the DLQ and the DLQ depth decrements immediately.

- **Given** the discard action is confirmed,  
  **When** it is executed,  
  **Then** an immutable `AuditLog` entry is written with: `operator_id`, `occurred_at`, `dlq_entry_id`, `action = discard`, and `discard_reason` (BR014).

- **Given** the discarded event had a `payment_id` referencing a captured payment,  
  **When** I fill in the reason field,  
  **Then** the ops console prompts me to confirm that I have reviewed the payment status in Stripe before discarding — ensuring no money is silently abandoned.

- **Given** the DLQ entry is discarded,  
  **When** the Grafana alert condition is re-evaluated,  
  **Then** if the DLQ depth is now zero, the alert resolves automatically.

**Size:** S  
**Priority:** Medium  
**Related:** FR010, BR014

**Technical Notes:**
- Discard endpoint: `POST /api/ops/dlq/{entry_id}/discard` with required `reason` body field — requires `platform_operator` JWT.
- ops-service calls `sqs:DeleteMessage` on the DLQ only after writing the `AuditLog` entry; if the audit write fails, the SQS message is not deleted.
- The ops console maintains a `discarded` state on the `dlq_entries` table row (not deleted) so historical ops decisions are searchable and auditable indefinitely.
- Payment-check prompt is enforced in the UI when `payment_id != null` in the event payload — it is a soft guard (operator can override) but the decision is logged.

---

### US-UJ006-05 · Trigger a tenant event replay from the ops console

**Title:** Re-publish historical events for a tenant to rebuild corrupted downstream views

**User Story:**
As a **Platform Operator**, I want to trigger a scoped event replay for a tenant from the ops console, so that corrupted or missing read-model data can be rebuilt from the authoritative AppointmentEvent history without affecting live bookings.

**Acceptance Criteria:**

- **Given** I initiate a replay for a tenant from the ops console,  
  **When** I select the replay scope (full tenant, date range, event type, or specific IDs),  
  **Then** ops-service creates a `ReplayJob` record and begins re-publishing matching `AppointmentEvent` records to SNS within 10 seconds (FR011).

- **Given** a replay is running,  
  **When** live booking events continue to arrive at consumers,  
  **Then** live operations are not interrupted — replay events are processed through the same consumer queues and InboxRecord deduplication prevents any double-processing (FR011).

- **Given** the replay is running,  
  **When** I check the ops console,  
  **Then** I can see the `ReplayJob` status (`queued | running | completed | failed`), `events_published` count, and elapsed time — updated every 5 seconds.

- **Given** the replay completes,  
  **When** I verify the target service (e.g. dashboard-service) state,  
  **Then** the downstream view reflects data consistent with the full replayed event history, and the `ReplayJob.status = completed`.

**Size:** M  
**Priority:** Medium  
**Related:** FR011, BR014

**Technical Notes:**
- ops-service reads `AppointmentEvent[]` from PostgreSQL directly (no SQS involvement) and re-publishes each to the appropriate `appointment-*` SNS topic with `replay_job_id` in the message attributes.
- Replay events are tagged so consumers can log replay vs. live traffic separately — they are functionally identical to live events for processing purposes.
- `ReplayJob` lifecycle states: `queued → running → completed | failed | cancelled`.
- Replay does not bypass InboxRecord deduplication: if an event was already processed by a consumer, it is silently skipped via unique constraint on `(event_id, consumer_name)`.

---

## Definition of Ready

Before any story in this epic enters a sprint, confirm all items below are checked:

- [ ] Acceptance criteria reviewed and agreed by Product Owner
- [ ] All named DLQ queue names confirmed in `specs/ai/operations.json` — one per consumer per event type
- [ ] Grafana alert query for DLQ depth confirmed (`aws_sqs_approximate_number_of_messages_not_visible{queue=~".*-dlq"} > 0`)
- [ ] ops-service DLQ polling mechanism confirmed — 60-second interval, explicit queue name list, not a wildcard scan
- [ ] Re-queue SQS atomicity confirmed: `SendMessage` to source queue + `DeleteMessage` from DLQ in sequence
- [ ] AuditLog write-before-SQS-call ordering confirmed for both re-queue and discard paths (BR014)
- [ ] InboxRecord deduplication confirmed in all consumer services — prevents double-processing on re-queue
- [ ] Replay scope options confirmed: full_tenant, date_range, event_type, specific_ids (FR011)
- [ ] `ReplayJob` schema and status transitions confirmed in `specs/ai/domain.json`
- [ ] PII masking rules in ops console UI agreed with data protection officer (BR005)
- [ ] Definition of Done agreed for this layer (see `documentation/00-governance/G5-definition-of-done.md`)
