# REL-003 · Go/No-Go Decision

**Project:** AnjiSchedulo  
**Last Updated:** 2026-06-28

---

## 1. Purpose

The Go/No-Go meeting is the final gate before a production deploy. It is a structured, time-boxed decision meeting (target: 15 minutes) that confirms the release is safe to ship or documents the reason for deferral.

---

## 2. Meeting Details

| Field | Value |
|---|---|
| Time | Release day at 09:00 UTC |
| Duration | 15 minutes (time-boxed) |
| Attendees | Engineering Lead (decision maker), On-call engineer |
| Channel | Slack #deployments (async acceptable for PATCH releases) |

---

## 3. Go/No-Go Criteria

**GO** if all of the following are true:

| Criterion | Source |
|---|---|
| All blocking validation checks passed (REL-002) | REL-002 checklist |
| No active P1 or P2 incidents on production | Grafana Alerting |
| Booking saga error rate stable (< 1%) on staging over last 12 hours | Grafana → Booking Operations |
| No DLQ messages in staging | Grafana → DLQ Triage |
| Cross-tenant isolation metric = 0 throughout validation | Grafana → Security |
| Rollback plan confirmed (O3, CD-004) | Engineering Lead |
| On-call engineer available to monitor for 2 hours post-deploy | Confirmed |

**NO-GO** if any of the following are true:

| Condition | Required action |
|---|---|
| Any blocking REL-002 check is ❌ | Fix and re-run validation. Set new Go/No-Go date. |
| P1 or P2 incident active on production | Wait for resolution. Do not release during an incident. |
| Booking saga error rate > 1% on staging | Investigate root cause. Fix and re-validate. |
| DLQ non-empty on staging | Triage DLQ before release. |
| Cross-tenant isolation violation detected | Blocking — escalate to Engineering Lead. No release. |
| On-call not available for monitoring window | Reschedule. |

---

## 4. Decision Record

Complete in Slack #deployments:

```
## Go/No-Go Record — v{N}

Date: YYYY-MM-DD
Decision: GO / NO-GO
Decision maker: {Engineering Lead name}

| Criterion | Status | Notes |
|---|---|---|
| Validation checks (REL-002) | ✓ / ❌ | |
| No active incidents | ✓ / ❌ | |
| Staging saga error rate | ✓ / ❌ | {value}% |
| DLQ empty | ✓ / ❌ | |
| Isolation metric = 0 | ✓ / ❌ | |
| Rollback plan confirmed | ✓ / ❌ | |
| On-call available | ✓ / ❌ | {name} |

Next step:
GO:    Production deploy begins at 10:00 UTC (REL-004)
NO-GO: New target date — {YYYY-MM-DD} at {HH:MM} UTC
```

---

## 5. Next Step

**GO:** Proceed to [REL-004 · Production Release](REL-004-production-release.md) at 10:00 UTC.

**NO-GO:** Post the deferral reason to Slack #deployments. Leave the RC branch intact. Fix the blocking issue, re-validate (REL-002), and schedule a new Go/No-Go.

---

## 6. Traceability

| Concern | Doc |
|---|---|
| RC validation results | REL-002-rc-validation.md |
| Production deploy | REL-004-production-release.md |
| Rollback plan | O3-rollback-plan.md, CD-004-rollback.md |
