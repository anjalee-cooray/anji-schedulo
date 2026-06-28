# O3 · Rollback Plan

**Project:** AnjiSchedulo  
**Version:** 1.0.0  
**Last Updated:** 2026-06-28

---

## 1. Overview

This document defines when to roll back, how to decide between a rollback and a kill switch, and the rollback procedures by scenario. It is the operator-facing companion to [CD-004 · Rollback Flow](../04-architecture/cicd-flows/CD-004-rollback.md), which contains the technical commands. This document focuses on decision-making and authorisation.

---

## 2. Rollback vs. Kill Switch

Before initiating a rollback, consider whether a feature flag kill switch is faster and lower risk:

| Situation | Preferred action |
|---|---|
| Specific new feature is causing errors, rest of deploy is fine | Kill switch (FLAG-003) — faster, no re-deploy of other services |
| Multiple services behaving incorrectly after deploy | Full rollback (CD-004) |
| Payment flow broken | Full rollback — do not leave partially functioning booking |
| Cross-tenant isolation violation (ALT005) | Full rollback immediately — P1, no discussion |
| Database migration caused schema issues | Forward-fix migration, not rollback (migrations are forward-only) |
| Performance degradation but no errors | Monitor for 15 min; rollback if SLO breach persists |

See HOT-002 for the full decision tree.

---

## 3. Rollback Authorisation

| Scenario | Who can authorise |
|---|---|
| Automatic rollback (smoke test failure) | CI pipeline — no human approval needed |
| Manual rollback during monitoring window | On-call engineer — self-authorised |
| Manual rollback > 1 hour post-deploy | On-call engineer + Engineering Lead notification |
| Database PITR (data recovery) | Engineering Lead required |
| Cross-region failover | Platform Owner required |

---

## 4. Rollback Procedures Summary

| Type | Steps | Time |
|---|---|---|
| **Automatic** (smoke test) | Pipeline reverts ECS task definition automatically | 5–8 min |
| **Manual ECS rollback** | `aws ecs update-service --task-definition {service}:<N-1>` for all services in reverse order | 10–15 min |
| **Kill switch** | Update Terraform flag to `false`, `terraform apply`, ECS redeploys | 10–15 min |
| **Migration forward-fix** | Write corrective Flyway migration, deploy via hotfix pipeline | 30–60 min |
| **Database PITR** | `aws rds restore-db-instance-to-point-in-time` + service reconnection | 30–60 min |

Full commands: [CD-004 · Rollback Flow](../04-architecture/cicd-flows/CD-004-rollback.md).

---

## 5. Rollback Decision Criteria

Initiate a manual rollback when any of the following are true for more than 5 minutes after deploy:

| Metric | Threshold |
|---|---|
| Booking saga error rate | > 5% (excluding slot_unavailable, booking_limit_exceeded) |
| Booking saga p95 latency | > 3 seconds |
| Platform availability | < 99.9% |
| Cross-tenant isolation violation | Any — immediate |
| DLQ depth growing post-deploy | Growing at > 10 messages/minute |
| Notification failure rate | > 10% AND payment flow is affected |

If the metric is degraded but below these thresholds: continue monitoring. Do not rollback preemptively.

---

## 6. Post-Rollback Actions

After a rollback completes and production is stable:

1. Post to Slack #incidents: rollback complete, metrics restored.
2. Open a tracking issue with the failure root cause and reproduction steps.
3. Keep the failed `{git-sha}` out of production until the issue is fixed and re-tested in staging.
4. Write a post-incident review (PIR) within 24 hours (template in O5).
5. Do not re-deploy the same SHA — fix forward and create a new commit.

---

## 7. Traceability

| Rollback Type | Doc |
|---|---|
| Automatic pipeline rollback | CD-004-rollback.md |
| Manual ECS rollback | CD-004-rollback.md |
| Kill switch | FLAG-003-kill-switch.md, HOT-002-kill-switch-vs-hotfix.md |
| Database PITR | RES-003-disaster-recovery.md, A12-disaster-recovery.md |
| Hotfix (roll forward) | HOT-001-hotfix-release.md |
