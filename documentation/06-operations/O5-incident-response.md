# O5 · Incident Response

**Project:** AnjiSchedulo  
**Version:** 1.0.0  
**Last Updated:** 2026-06-28

---

## 1. Severity Levels

| Severity | Definition | Response Target | Examples |
|---|---|---|---|
| **P1 — Critical** | Complete booking unavailability; cross-tenant data breach; active data loss | Acknowledge ≤ 15 min; resolve ≤ 60 min | ALT001 saga error rate > 5%, ALT002 availability < 99.9%, ALT005 isolation violation |
| **P2 — High** | Degraded performance; DLQ growing; single-tenant outage | Acknowledge ≤ 1 hour; resolve ≤ 4 hours | ALT003 DLQ, ALT004 saga latency, ALT006 outbox lag, ALT007 RDS CPU |
| **P3 — Low** | Non-critical feature degraded; notification failures for one tenant | Acknowledge next business day | ALT008 notification failure rate |

---

## 2. Alert-to-Resolution Flow

```
Alert fires in Grafana
    │
    ├── P1/P2 → PagerDuty pages on-call engineer
    └── P3    → Slack #incidents notification only
          │
          ▼
On-call engineer acknowledges (within SLA)
    │
    ▼
Open Slack #incidents thread:
    "🚨 P{N} INCIDENT — {alert name}
     Current value: {metric}
     Runbook: {link}
     Responder: @{engineer}"
    │
    ▼
Triage (0–15 min for P1):
    1. Open Grafana Platform Overview
    2. Is this related to a recent deploy? (last 30 min)
       YES → Initiate rollback (O3, CD-004)
       NO  → Continue investigation
    3. Is a feature flag the cause?
       YES → Kill switch (FLAG-003)
       NO  → Continue investigation
    │
    ▼
Apply fix (rollback / scale / config / code)
    │
    ▼
Verify resolution:
    - Booking success rate ≥ 99.9%
    - p95 latency < 3s
    - DLQ depth not growing
    - No new P1/P2 alerts
    │
    ▼
Post to Slack #incidents:
    "✅ RESOLVED — {alert name}
     Duration: {X} minutes
     Root cause: {brief}
     PIR: {link or 'TBD'}"
    │
    ▼
P1/P2: Open PIR ticket within 24 hours
P3: Log in tracking issue
```

---

## 3. Escalation Chain

| Level | Role | Contact |
|---|---|---|
| 1st | On-call engineer | PagerDuty rotation |
| 2nd (auto at 30 min P1) | Engineering Lead | PagerDuty escalation policy |
| 3rd (auto at 60 min P1) | Platform Owner | PagerDuty escalation policy |
| External | AWS Support | Business support tier |
| External | Stripe support | Stripe dashboard → Support |

---

## 4. P1 Playbooks

### P1-A: Booking Saga Error Rate > 5% (ALT001)

```
1. Check booking-command-service error logs:
   Loki: {service="booking-command-service", level="error"} | json

2. Identify failure_reason:
   slot_unavailable       → normal business condition, not an incident
   payment_failed         → check Stripe status: status.stripe.com
   db_connection_error    → check RDS CPU and PgBouncer connections
   timeout                → check downstream service health

3. If Stripe outage:
   - Enable booking hold mode: PATCH /ops/tenants/*/hold
   - Notify tenants via status page
   - Await Stripe recovery; resume normal operations

4. If DB or application error:
   - If post-deploy: rollback (CD-004)
   - If infrastructure: scale ECS or investigate RDS
```

### P1-B: Platform Availability < 99.9% (ALT002)

```
1. Check ALB 5xx rate: Grafana Infrastructure Health
2. Check ECS task health: aws ecs describe-services ...
3. Check RDS connections: Grafana → db connection count
4. If tasks crashing: check ECS task stopped reason in AWS console
5. If post-deploy: rollback immediately
6. If RDS overloaded: check pg_stat_activity for blocking queries
```

### P1-C: Cross-Tenant Isolation Violation (ALT005)

```
STOP ALL WRITE TRAFFIC IMMEDIATELY:
  Scale booking-command-service, tenant-service to 0 tasks

1. Do not attempt self-remediation — escalate to Engineering Lead immediately
2. Preserve evidence: export relevant Loki logs + Tempo traces
3. Identify the affected tenants from isolation_monitor logs
4. Assess data exposure scope
5. Engineering Lead coordinates fix + communication plan
6. Platform Owner approves any tenant notification
7. After fix: full re-run of cross-tenant isolation test suite before resuming
```

---

## 5. Post-Incident Review (PIR) Template

Required for all P1 incidents and P2 incidents lasting > 4 hours. Complete within 5 business days.

```markdown
## PIR — {Incident Title}

**Date:** YYYY-MM-DD  
**Duration:** {X} hours {Y} minutes  
**Severity:** P{N}  
**Author:** {name}  
**Reviewers:** {Engineering Lead}

### Timeline

| Time (UTC) | Event |
|---|---|
| HH:MM | Alert fired |
| HH:MM | On-call acknowledged |
| HH:MM | Root cause identified |
| HH:MM | Fix applied |
| HH:MM | Resolved |

### Root Cause

{Describe the technical root cause. Not "human error" — what made the human error possible?}

### Impact

- Tenants affected: {N or "all"}
- Bookings failed: {N (estimate)}
- Revenue impact: {estimate if applicable}
- Data exposure: {yes/no — details if yes}

### What Went Well

- {List things that worked — fast detection, good runbooks, etc.}

### What Could Be Improved

- {List gaps — slow detection, missing runbook, unclear escalation, etc.}

### Action Items

| Action | Owner | Due |
|---|---|---|
| {Specific preventive action} | {name} | YYYY-MM-DD |
```

---

## 6. Contacts and Resources

| Resource | Link / Contact |
|---|---|
| Grafana dashboards | http://grafana.internal |
| PagerDuty | https://anji-schedulo.pagerduty.com |
| AWS Console | https://console.aws.amazon.com |
| Stripe status | https://status.stripe.com |
| SendGrid status | https://status.sendgrid.com |
| Twilio status | https://status.twilio.com |
| Slack #incidents | Internal Slack workspace |
| Runbook | O4-runbook.md |

---

## 7. Traceability

| Incident Type | Alert | NFR | Runbook |
|---|---|---|---|
| Booking unavailability | ALT001, ALT002 | NFR001, NFR004 | O4-runbook.md |
| Isolation violation | ALT005 | NFR012 | A5-threat-model.md (T-I01) |
| DLQ backlog | ALT003 | NFR016 | O4-runbook.md#dlq-triage |
| Saga latency | ALT004 | NFR004 | OBS-002-alerting-escalation.md |
| Outbox relay lag | ALT006 | NFR003 | O4-runbook.md |
