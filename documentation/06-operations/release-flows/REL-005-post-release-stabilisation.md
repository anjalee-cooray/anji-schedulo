# REL-005 · Post-Release Stabilisation

**Project:** AnjiSchedulo  
**Last Updated:** 2026-06-28

---

## 1. Purpose

The stabilisation period after every production release is the window during which the on-call engineer maintains heightened vigilance and P1 rollback is available without Engineering Lead approval. After the period ends with no P1/P2 incidents, the release is declared stable.

---

## 2. Stabilisation Period

| Release type | Period |
|---|---|
| PATCH (bug fix only) | 24 hours |
| MINOR (new features) | 48 hours |
| MAJOR (breaking changes) | 72 hours |

Start time: immediately after deployment smoke tests pass (REL-004).

---

## 3. Heightened Monitoring Checklist

On-call checks these metrics every 4 hours:

| Metric | Expected | Action if breached |
|---|---|---|
| Booking saga success rate | ≥ 99.9% | Page Engineering Lead; assess rollback |
| Booking saga p95 latency | < 3 seconds | Investigate; rollback if breach > 5 min |
| DLQ depth | 0 | Triage immediately (O4) |
| Outbox backlog | < 10 records | Investigate relay health |
| `isolation_monitor_violations_total` | 0 | P1 immediately — stop writes |
| Notification failure rate | < 2% | Triage DLQ; re-queue if transient |

---

## 4. Rollback Window

Rollback is self-authorised by on-call during the stabilisation period. After the period ends, rollback requires Engineering Lead approval and a hotfix forward-fix is preferred.

---

## 5. End of Stabilisation

```
# Post to Slack #deployments:
"🟢 v{N} STABLE — 48-hour stabilisation period complete, no P1/P2 incidents.
 Release branch: release/v{N} archived.
 Retrospective: COM-002 scheduled for {RETRO_DATE}."

# Archive release branch
git push origin --delete release/v{N}
# The git tag v{N} remains as the permanent reference point
```

---

## 6. Early Exit (P1 During Stabilisation)

If a P1 incident occurs:
1. Assess rollback vs. hotfix (HOT-002).
2. If rollback: initiate immediately (O3, CD-004).
3. Reset the stabilisation period start time after recovery.

The release is not declared stable until a full stabilisation period completes without incident.

---

## 7. Post-Stabilisation Actions

| Action | Owner | When |
|---|---|---|
| Archive release branch | Engineer | End of stabilisation |
| Schedule release retrospective | Engineering Lead | Within 5 business days |
| Close release tracking issue | Engineer | Same day |
| Update roadmap for shipped features | Engineering Lead | Within 1 business day |

---

## 8. Traceability

| Concern | Doc |
|---|---|
| Production deploy | REL-004-production-release.md |
| Rollback | O3-rollback-plan.md |
| Hotfix | HOT-001-hotfix-release.md |
| Retrospective | COM-002-release-retrospective.md |
