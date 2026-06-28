---
title: Persona — Platform Operator
personaId: P4
layer: 01-requirements
status: current
lastUpdated: 2026-06-28
---

# Persona — Platform Operator (P4)

| Field | Value |
|---|---|
| **Persona ID** | P4 |
| **Name** | Platform Operator |
| **Role** | SaaS Infrastructure Operator |
| **Technical Level** | High |

---

## Who Is This Person?

Morgan is a senior Site Reliability Engineer on the AnjiSchedulo internal team. They have spent six years working across distributed systems and have a deep instinct for where silent failures hide: the event pipelines that look healthy on a status page until a booking at 2 AM leaves a customer uncharged and a slot unreleased, the refund that failed after a Stripe timeout and then sat in a dead letter queue for eight hours before anyone noticed, the tenant whose dashboard stopped updating after a deploy because the materialized view replay job was never triggered. Morgan's job is to make sure none of these events go undetected for long — and when they do occur, to resolve them cleanly, with a full audit trail and no cross-tenant data leakage.

Morgan starts every shift by checking the Grafana dashboard for DLQ depth across all consumer queues, outbox backlog, and replay job status. They are the person who receives a PagerDuty alert at 3 AM when a DLQ queue depth crosses zero, opens the ops console, reads the failed event payload, determines whether it can be re-queued immediately or requires payload correction, and resolves the incident within the 4-hour SLA. They are also the person who handles tenant offboarding: when a business requests to leave AnjiSchedulo, Morgan coordinates the data export, confirms the 30-day window has elapsed, initiates deprovisioning, and ensures that 90 days later, hard deletion runs correctly and no residual PII remains. Morgan does not write code to fix platform bugs during on-call — they work within the ops console and ops-service tooling — but they understand the architecture well enough to read an event payload, identify a malformed field, and correct it before re-queuing.

---

## Goals

**Know instantly when any service degrades.** Morgan cannot afford to discover a problem via a tenant support ticket. The Grafana alerting stack must surface DLQ depth, outbox backlog, notification failures, and booking saga errors within minutes of occurrence — with enough context in the alert to understand the scope without reading log files first. Silence in a monitoring system is not a sign of health; it is a sign that something is wrong with the monitoring.

**Resolve stuck events without redeploying.** The most common operational task Morgan faces is a failed event sitting in a DLQ. They need to inspect the full payload, understand exactly why it failed, apply a correction if the payload is malformed, and re-queue the event through the ops console — all without deploying new code or escalating to the engineering team. Re-queuing must be idempotent: if the event has already been processed by the time they re-queue, the system must handle it gracefully rather than creating a duplicate.

**Rebuild corrupted views without data loss.** When a bug in the dashboard-service projection logic is discovered and fixed, Morgan needs to trigger a replay of historical AppointmentEvents so the TenantDashboardView can be rebuilt from the correct source of truth. The replay must be scoped (to a tenant, a date range, or specific event IDs), must not disrupt live booking operations, and must be trackable via a ReplayJob record so Morgan knows when it has completed and how many events were published.

**Offboard tenants cleanly and compliantly.** When a tenant leaves the platform, the offboarding process has strict regulatory constraints: a 30-day read-only window for data export, a 90-day retention period before hard deletion, and permanent retention of audit logs and payment records. Morgan needs tooling that enforces these constraints automatically rather than relying on manual calendar reminders — and produces an audit trail that would satisfy a GDPR audit.

---

## Frustrations

**Silent failures in event pipelines.** Morgan's deepest frustration is a failure that produces no alert. A message that is malformed at the envelope level, rejected by all consumers, and routed to a DLQ — but triggers no Grafana alert because the DLQ monitoring threshold was set to the wrong queue — can sit for hours. Every hour of undetected failure is an hour during which a customer may have been uncharged, a notification unsent, or a dashboard showing stale data. Comprehensive DLQ monitoring across every named queue is non-negotiable.

**No way to inspect failed events.** In previous systems Morgan has worked on, failed messages in a DLQ were opaque — the only information available was the message body, which was often compressed or encoded. Without full payload inspection, Morgan could not determine whether a failure was caused by a malformed field, a schema mismatch, or a downstream service timeout. The AnjiSchedulo ops console must surface the complete event payload, the failure reason, the attempt count, and the timestamp of each delivery failure.

**Tenant data leaking across boundaries.** Morgan has experienced incidents at other companies where a cross-tenant read — caused by a missing tenant_id filter in an application query — exposed one customer's data to another. In a multi-tenant scheduling platform handling PII (customer names, phone numbers, appointment histories), such a leak would be a GDPR breach. Morgan relies on PostgreSQL Row-Level Security as the safety net beneath the application layer, but they also need tooling that makes cross-tenant reads impossible from the ops console without explicit scoping to a specific tenant_id.

---

## Primary Actions in the Platform

| Action | Functional Requirements |
|---|---|
| Monitor DLQ depth across all consumer queues and triage failed events | FR010 |
| Trigger tenant event replay to rebuild derived views | FR011 |
| Initiate tenant suspension and deprovisioning in the controlled sequence | FR013 |
| Respond to Grafana/PagerDuty alerts for DLQ depth, outbox backlog, booking errors | FR010, FR011 |

---

## User Journeys

| Journey ID | Name | P4's Role |
|---|---|---|
| UJ006 | Platform Operator DLQ Triage | Primary actor — receives alert, inspects event payload, corrects and re-queues or discards, confirms resolution |
| UJ007 | Tenant Data Export and Offboarding | Primary actor (deprovisioning side) — confirms 30-day window, initiates deprovisioning, schedules hard deletion |
| UJ001 | Tenant Onboarding | Observer — monitors provisioning events for failures; receives alert if billing-service DLQ fires during subscription activation |

---

## Needs from the Platform

In priority order:

1. **Comprehensive, named DLQ monitoring** — every consumer queue has a corresponding DLQ with an alert that fires when depth > 0, identifying the queue name and the first failed event's type.
2. **Full payload inspection in the ops console** — complete event body, failure reason, attempt count, and delivery timestamps for every DLQ entry.
3. **Safe idempotent re-queue** — re-queuing a DLQ event must be safe to do even if the event was already processed; InboxRecord deduplication on the consumer side handles duplicates.
4. **Scoped event replay** — ability to replay events for a single tenant, a date range, a specific event type, or a set of event IDs, with a ReplayJob record tracking progress.
5. **Cross-tenant read access without data leakage** — the platform_operator role grants read access across tenants for operational purposes, but every query must be explicitly scoped to a tenant_id; no wildcard tenant reads.
6. **Automated deprovisioning lifecycle** — 30-day and 90-day timers enforced by the platform, not by calendar reminders; status machine transitions that reject out-of-sequence operations.

---

## Success Looks Like

A successful on-call shift for Morgan is one where the Grafana dashboard shows all DLQ queues at zero depth, outbox backlog under five records, and no open P1 or P2 incidents. When an alert does fire — a payment.refund_failed event in the payment-service DLQ — Morgan opens the ops console within ten minutes, reads the full Stripe error code in the event payload, determines the refund failed due to an expired PaymentIntent, and escalates to the finance team for a manual Stripe Dashboard refund. They record the resolution in the audit log, re-queue the event with a corrected status, and close the incident within the 4-hour SLA. The customer whose refund failed has already been notified of the cancellation — that part worked. Only the refund step needs manual resolution. By the end of the shift, every queue is clear, every replay job has a `completed` status, and the audit log shows a clean chain of operator actions from alert to resolution.

---

## Main Interaction Flow

```
  PLATFORM OPERATOR — MAIN INTERACTION FLOW
  ──────────────────────────────────────────────────────────
  1. MONITOR                                         [FR010]
     Grafana/PagerDuty alert fires: DLQ depth > 0
       → Alert identifies queue name, event type, tenant_id
     Check Grafana dashboards:
       - DLQ depth per queue
       - Outbox backlog (records unpublished > 10s)
       - Replay job status

  2. INSPECT DLQ EVENT                               [FR010]
     Open ops console → DLQ inspector
     Filter by tenant_id, event_type, failure_reason
       → FR010: Full payload, attempt count, failure timestamps visible
     Identify root cause:
       - Malformed envelope (missing correlation_id, tenant_id)
       - Schema mismatch (consumer rejected payload)
       - Downstream timeout (transient, safe to re-queue)
       - Systemic failure (multiple events from same root cause)

  3. TRIAGE AND RESOLVE                              [FR010]
     Option A — Re-queue (transient failure):
       Correct payload if malformed
       Write to source SQS queue via ops-service
         → InboxRecord deduplication ensures safe idempotency
       Audit log entry: operator_id, action=re-queue, event_id
     Option B — Discard (unprocessable):
       Mark event as discarded with reason
       Audit log entry: operator_id, action=discard, event_id
     Confirm event is processed successfully (queue depth returns to 0)

  4. TRIGGER REPLAY (if needed)                     [FR011]
     Identify corrupted or missing derived view
     Create ReplayJob: scope = tenant | date_range | event_type
       → FR011: ops-service re-publishes events to SNS
     Monitor ReplayJob.status: queued → running → completed
     Verify view rebuilt correctly (dashboard or query)

  5. TENANT LIFECYCLE                                [UJ007]
     Suspension: halt new bookings, preserve read access (BR011)
     Confirm 30-day window elapsed
     Initiate deprovisioning:
       → tenant.deprovisioned event published
       → 90-day hard deletion timer scheduled (BR012)
     PII erasure: pseudonymise on customer request (FR013)
  ──────────────────────────────────────────────────────────
```
