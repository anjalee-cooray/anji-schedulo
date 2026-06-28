# OBS-002 · Alerting and Escalation Flow

**Project:** AnjiSchedulo  
**Version:** 1.0.0  
**Last Updated:** 2026-06-28

---

## 1. Overview

Alerts are defined in Grafana and evaluate PromQL queries against Grafana Mimir. When an alert fires, it routes through Grafana Alerting to PagerDuty (P1/P2) and Slack (all severities). This document describes the alert evaluation cycle, routing logic, escalation tiers, and runbook locations.

---

## 2. Alert Evaluation Flow

```
Grafana Mimir (metrics store)
          │ Grafana scrapes every 15s
          │ Metrics: bookings_saga_duration_seconds,
          │          dlq_depth_total, isolation_monitor_violations_total, etc.
          ▼
Grafana Alerting (evaluation every 1 minute)
          │ Evaluates PromQL against each alert rule
          │ Alert fires when condition is true for > {pending_period}
          ▼
┌─────────────────────────────────────┐
│  Alert State Machine                │
│                                     │
│  Normal → Pending → Firing          │
│           │ condition true           │
│           │ for pending_period       │
│           ▼                         │
│         Firing → Resolved           │
│                  │ condition false   │
│                  │ for 5 minutes    │
└─────────────────────────────────────┘
          │ Alert fires
          ▼
Grafana Contact Points
    ├── PagerDuty (P1, P2 only)
    │     - Integration key from Secrets Manager
    │     - Payload: alert name, severity, runbook link, current metric value
    │
    └── Slack webhook → #incidents
          - All severities (P1, P2, P3)
          - Message includes: alert name, severity, current value, runbook link
```

---

## 3. Alert Definitions

| ID | Name | PromQL Condition | Pending | Severity |
|---|---|---|---|---|
| ALT001 | Booking Saga Error Rate Critical | `sum(rate(booking_saga_errors_total{failure_reason!~"slot_unavailable\|booking_limit_exceeded"}[5m])) > 0.05` | 2m | P1 |
| ALT002 | Platform Availability Below SLO | `sum(rate(bookings_saga_duration_seconds_count{status="confirmed"}[15m])) / sum(rate(bookings_saga_duration_seconds_count[15m])) < 0.999` | 5m | P1 |
| ALT003 | DLQ Depth Non-Zero | `sum(dlq_depth_total) > 0` | 5m | P2 |
| ALT004 | Booking Saga p95 > 3s | `histogram_quantile(0.95, sum(rate(bookings_saga_duration_seconds_bucket[10m])) by (le)) > 3` | 10m | P2 |
| ALT005 | Cross-Tenant Isolation Violation | `sum(isolation_monitor_violations_total) > 0` | 0m | P1 |
| ALT006 | Outbox Relay Lag | `sum(outbox_backlog_total) > 100` | 5m | P2 |
| ALT007 | RDS CPU High | `avg(aws_rds_cpu_utilization_average) > 80` | 10m | P2 |
| ALT008 | Notification Failure Rate High | `sum(rate(notification_delivery_failures_total[5m])) / sum(rate(notification_attempts_total[5m])) > 0.10` | 5m | P3 |

**ALT005 has 0 minute pending period** — a single isolation violation fires immediately, without waiting. This is a critical security event.

---

## 4. Escalation Tiers

### P1 — Critical (ALT001, ALT002, ALT005)

| Step | Action | Time |
|---|---|---|
| 0 | Alert fires — PagerDuty pages on-call engineer | Immediate |
| 1 | On-call engineer acknowledges | ≤ 15 minutes |
| 2 | On-call engineer assesses severity and applies first fix (rollback/scaling/toggle) | ≤ 30 minutes |
| 3 | If not resolved in 30 min: escalate to Engineering Lead (PagerDuty escalation policy) | +30 minutes |
| 4 | If not resolved in 60 min: escalate to platform owner | +60 minutes |

### P2 — High (ALT003, ALT004, ALT006, ALT007)

| Step | Action | Time |
|---|---|---|
| 0 | Alert fires — PagerDuty pages on-call engineer | Immediate |
| 1 | On-call engineer acknowledges | ≤ 1 hour |
| 2 | On-call engineer investigates and resolves | ≤ 4 hours |
| 3 | If not resolved in 4 hours: escalate to Engineering Lead | +4 hours |

### P3 — Low (ALT008)

| Step | Action | Time |
|---|---|---|
| 0 | Alert fires — Slack #incidents notification | Immediate |
| 1 | Engineer picks up from Slack during business hours | ≤ 1 business day |
| 2 | Resolve at next business hour | — |

---

## 5. Runbook Locations

| Alert | Runbook |
|---|---|
| ALT001 — Booking saga errors | `ops-service/docs/runbooks/booking-saga-errors.md` |
| ALT002 — Availability below SLO | `ops-service/docs/runbooks/availability-slo-breach.md` |
| ALT003 — DLQ depth | `documentation/06-operations/O4-runbook.md#dlq-triage` |
| ALT004 — Saga latency | `ops-service/docs/runbooks/saga-latency.md` |
| ALT005 — Isolation violation | `ops-service/docs/runbooks/isolation-violation.md` (P1 emergency) |
| ALT006 — Outbox lag | `ops-service/docs/runbooks/outbox-relay-lag.md` |
| ALT007 — RDS CPU | `ops-service/docs/runbooks/rds-cpu.md` |
| ALT008 — Notification failure | `ops-service/docs/runbooks/notification-failure.md` |

---

## 6. On-Call Rotation

- On-call rotation managed in PagerDuty.
- 1 engineer on-call at all times (7×24).
- Rotation: weekly.
- Escalation chain: on-call engineer → Engineering Lead → Platform Owner.
- PagerDuty escalation policy configured to auto-escalate after 30 minutes of no acknowledgement.

---

## 7. Alert Suppression

Alerts are suppressed during:
- Planned maintenance windows (configured as Grafana Silences in advance)
- Known deployments (suppression applied for 15 minutes after deploy completes — except ALT005, which is never suppressed)

ALT005 (isolation violation) has no suppression, no pending period, and is never silenced.

---

## 8. SLO Burn Rate Alerts

In addition to the per-metric alerts above, Grafana SLO burn rate alerts fire when the error budget is being consumed too quickly:

| SLO | Fast Burn (1h) | Slow Burn (6h) | Severity |
|---|---|---|---|
| SLO001 (booking saga latency) | > 14× burn rate | > 2× burn rate | P1 / P2 |
| SLO002 (availability) | > 14× burn rate | > 2× burn rate | P1 / P2 |
| SLO003 (availability query latency) | > 14× burn rate | > 2× burn rate | P1 / P2 |
| SLO004 (dashboard latency) | > 14× burn rate | > 2× burn rate | P1 / P2 |

---

## 9. Traceability

| Alert | NFR | BR | Doc |
|---|---|---|---|
| ALT001, ALT002, ALT004 | NFR001, NFR004 | — | A11-observability.md |
| ALT003 | NFR016 | — | O4-runbook.md |
| ALT005 | NFR012 | BR005 | A5-threat-model.md |
| ALT006 | NFR003 | BR013 | A10-integrations.md |
| ALT007 | NFR001 | — | A7-infrastructure.md |
| ALT008 | NFR002 | BR007 | A10-integrations.md |
