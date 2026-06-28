# RES-002 · Incident Response Flow

**Project:** AnjiSchedulo  
**Version:** 1.0.0  
**Last Updated:** 2026-06-28

---

## 1. Overview

This document describes the step-by-step incident response flow for AnjiSchedulo from alert firing to resolution and post-incident review. Incidents are classified into three severity levels based on business impact. Response time targets and escalation paths differ per level.

---

## 2. Incident Severity Definitions

| Severity | Definition | Examples |
|---|---|---|
| **P1 — Critical** | Complete booking unavailability; cross-tenant data breach; active data loss | ALT001 (saga error > 5%), ALT002 (availability < 99.9%), ALT005 (isolation violation) |
| **P2 — High** | Degraded performance; DLQ growing; single-tenant outage; infrastructure warning | ALT003 (DLQ), ALT004 (saga latency > 3s), ALT006 (outbox lag), ALT007 (RDS CPU) |
| **P3 — Low** | Individual feature degraded; non-critical notification failure | ALT008 (notification failure rate) |

---

## 3. P1 Incident Response Flow

```
ALT001 / ALT002 / ALT005 fires in Grafana
          │
          ▼
PagerDuty pages on-call engineer (phone + SMS)
    SLA: Engineer acknowledges within 15 minutes
          │
          ▼
┌────────────────────────────────────────┐
│  Step 1: Triage (0–15 minutes)         │
│                                        │
│  Open Grafana Platform Overview        │
│    - Which service has errors?         │
│    - When did it start? (recent deploy?)│
│    - What is the error rate / metric?  │
│                                        │
│  Check: did a deployment happen        │
│    in the last 30 minutes?             │
│    YES → initiate rollback (CD-004)    │
│    NO  → continue investigation        │
└────────────────┬───────────────────────┘
                 │
                 ▼
┌────────────────────────────────────────┐
│  Step 2: First Response (15–30 min)    │
│                                        │
│  For ALT001 (saga errors):             │
│    - Check booking-command-service logs│
│    - Check payment-service logs        │
│    - Check DLQ depth (ALT003 related?) │
│    - If Stripe outage: enable booking  │
│      hold mode via ops-service         │
│                                        │
│  For ALT002 (availability SLO):        │
│    - Check ECS task health             │
│    - Check RDS connections             │
│    - Check ALB 5xx rate               │
│    - Scale out if CPU-bound            │
│                                        │
│  For ALT005 (isolation violation):     │
│    - STOP all write traffic immediately│
│      (scale ECS to 0 for affected svc) │
│    - Escalate to Engineering Lead NOW  │
│    - Do not attempt self-remediation   │
│    - Preserve all logs and traces      │
└────────────────┬───────────────────────┘
                 │
                 ▼
┌────────────────────────────────────────┐
│  Step 3: Resolution                    │
│                                        │
│  Apply fix (rollback / scale / config) │
│  Verify: Grafana metrics return to SLO │
│  Confirm: booking saga success ≥ 99.9% │
│  Confirm: DLQ depth not growing        │
│                                        │
│  If not resolved within 30 min:        │
│    Escalate to Engineering Lead        │
│  If not resolved within 60 min:        │
│    Escalate to Platform Owner          │
└────────────────┬───────────────────────┘
                 │
                 ▼
┌────────────────────────────────────────┐
│  Step 4: Post-Incident                 │
│                                        │
│  - Resolve PagerDuty alert             │
│  - Post resolution note to             │
│    Slack #incidents                    │
│  - Open post-incident review (PIR)     │
│    ticket within 24h                   │
│  - PIR completed within 5 business     │
│    days (template: O5-incident-        │
│    response.md)                        │
└────────────────────────────────────────┘
```

---

## 4. P2 Incident Response Flow

```
ALT003 / ALT004 / ALT006 / ALT007 fires
          │
          ▼
PagerDuty pages on-call engineer
    SLA: Acknowledge within 1 hour
          │
          ▼
Triage using Grafana DLQ Triage or
Infrastructure Health dashboard
          │
          ▼
┌────────────────────────────────────────────┐
│  ALT003 — DLQ Depth Non-Zero               │
│    1. Identify which queue(s) have depth   │
│    2. Inspect oldest DLQ message (body +   │
│       attributes)                          │
│    3. Identify failure_reason from message │
│    4. If transient: re-queue via ops-svc   │
│    5. If poison pill: quarantine message   │
│    6. Verify queue drains after re-queue   │
│    See: O4-runbook.md#dlq-triage           │
│                                            │
│  ALT004 — Saga Latency > 3s               │
│    1. Check Stripe API status page         │
│    2. Check RDS query latency (pg_stat)    │
│    3. Check Redis hit rate (cache miss?)   │
│    4. Scale out booking-command-service    │
│       if CPU-bound                         │
│                                            │
│  ALT006 — Outbox Lag > 100                 │
│    1. Check outbox-relay logs              │
│    2. Check SNS topic error rate           │
│    3. Restart outbox-relay if stuck        │
│    4. Verify backlog draining              │
│                                            │
│  ALT007 — RDS CPU > 80%                   │
│    1. Check pg_stat_statements for         │
│       expensive queries                    │
│    2. Check if scale-out is needed         │
│    3. Check for lock contention            │
└────────────────────────────────────────────┘
          │
          ▼
Resolve within 4 hours or escalate
Open issue to track root cause fix
```

---

## 5. P3 Incident Response Flow

```
ALT008 fires → Slack #incidents notification only
          │
          ▼
Engineer acknowledges during business hours
          │
          ▼
Check notification-service logs
    - Which channel? (email / SMS)
    - Which tenants affected?
    - Is it a provider outage? (SendGrid / Twilio status page)
          │
          ▼
If provider outage: await recovery; DLQ messages will be replayed
If application error: investigate + deploy fix
          │
          ▼
Resolve within 1 business day
```

---

## 6. Incident Communication Templates

### Slack #incidents — P1 Open

```
🚨 P1 INCIDENT OPEN
Alert: {ALT name}
Service: {service name}
Started: {time}
Current value: {metric value}
Runbook: {link}
Responder: @{on-call engineer}
```

### Slack #incidents — P1 Update

```
🔄 P1 UPDATE
Root cause identified: {description}
Action taken: {rollback/scale/fix}
ETA to resolution: {time}
```

### Slack #incidents — Resolved

```
✅ P1 RESOLVED
Duration: {X} minutes
Root cause: {brief description}
PIR ticket: {link}
```

---

## 7. Contacts and Runbooks

| Role | Contact Method |
|---|---|
| On-call engineer | PagerDuty rotation |
| Engineering Lead | PagerDuty escalation tier 2 |
| Platform Owner (Pubudu Anjalee Cooray) | PagerDuty escalation tier 3 |
| AWS Support | AWS Support console (Business tier) |
| Stripe status | https://status.stripe.com |
| SendGrid status | https://status.sendgrid.com |
| Twilio status | https://status.twilio.com |

---

## 8. Traceability

| Incident Type | NFR | Alert | Runbook |
|---|---|---|---|
| Booking unavailability | NFR001 | ALT001, ALT002 | O5-incident-response.md |
| Tenant isolation violation | NFR012 | ALT005 | isolation-violation runbook |
| DLQ backlog | NFR016 | ALT003 | O4-runbook.md |
| Saga performance | NFR004 | ALT004 | saga-latency runbook |
| Outbox relay lag | NFR003 | ALT006 | outbox-relay-lag runbook |
