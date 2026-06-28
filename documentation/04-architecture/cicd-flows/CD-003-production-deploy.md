# CD-003 · Production Deployment Flow

**Project:** AnjiSchedulo  
**Version:** 1.0.0  
**Last Updated:** 2026-06-28

---

## 1. Overview

Production deployments run when a `release/*` branch is merged to `main`. They require a manual approval gate from an Engineering Lead before proceeding. The flow is identical to staging (CD-002) but with additional safeguards: manual gate, pre-flight checks, and 10-minute post-deploy Grafana monitoring window.

**Trigger:** Push to `main` branch (from `release/*` merge)  
**Environment:** `production` (eu-west-1)  
**Duration:** ~25–35 minutes including manual gate  
**Auto-deploy:** No — requires manual approval from Engineering Lead

---

## 2. Pre-Flight Requirements

Before the manual approval gate can be granted, the following must all be true:

- [ ] The same `{git-sha}` was successfully deployed to staging (CD-002 green)
- [ ] All staging smoke tests and extended validation passed
- [ ] No active P1 or P2 Grafana alert in production
- [ ] Maintenance window is active (if migration requires > 5 seconds of table lock)
- [ ] Rollback plan documented in the deploy PR
- [ ] On-call engineer is available (not in a meeting, reachable on PagerDuty)

---

## 3. Pipeline Stages

```
Merge to main (release/* → main)
          │
          ▼
┌────────────────────────────────────┐
│  Pre-Flight Checks (automated)     │
│  - Verify staging deploy succeeded │
│    with same {git-sha}             │
│  - Check no active P1/P2 alerts    │
│  - Verify ECR image exists for     │
│    this {git-sha}                  │
└────────────────┬───────────────────┘
                 │ pass
                 ▼
┌────────────────────────────────────┐
│  MANUAL APPROVAL GATE              │
│  GitHub Actions environment:       │
│    "production"                    │
│  Required approver: Engineering    │
│    Lead or Principal Engineer      │
│  Timeout: 4 hours (then expires)   │
└────────────────┬───────────────────┘
                 │ approved
                 ▼
┌────────────────────────────────────┐
│  Stage 1: RDS Manual Snapshot      │
│  - aws rds create-db-snapshot      │
│    --identifier prod-pre-{sha}     │
│  - Wait for AVAILABLE status       │
│  - Retained indefinitely           │
└────────────────┬───────────────────┘
                 │
                 ▼
┌────────────────────────────────────────┐
│  Stage 2: Database Migration           │
│  - ECS one-off task: flyway migrate    │
│  - Target: production DB               │
│  - Wait for EXIT_CODE=0                │
│  - Pipeline halts on failure           │
│    (snapshot available for recovery)   │
│  - Notify Slack #deployments on start  │
└────────────────┬───────────────────────┘
                 │ success
                 ▼
┌────────────────────────────────────────┐
│  Stage 3: ECS Rolling Update           │
│  Deploy in dependency order:           │
│    1. outbox-relay                     │
│    2. availability-service             │
│    3. payment-service                  │
│    4. booking-command-service          │
│    5. notification-service             │
│    6. dashboard-service                │
│    7. analytics-service                │
│    8. tenant-service                   │
│    9. ops-service                      │
│   10. api-gateway                      │
│  Parameters:                           │
│    minHealthyPercent: 50%              │
│    maxPercent: 200%                    │
│    circuitBreaker: enabled             │
└────────────────┬───────────────────────┘
                 │
                 ▼
┌────────────────────────────────────────┐
│  Stage 4: Smoke Tests                  │
│  - POST /auth/token                    │
│  - GET /tenants/{slug}/availability    │
│  - GET /health (all 10 services)       │
│  Auto-rollback on any failure          │
└────────────────┬───────────────────────┘
                 │ pass
                 ▼
┌────────────────────────────────────────┐
│  Stage 5: Post-Deploy Monitoring       │
│  - On-call engineer watches Grafana    │
│    for 10 minutes:                     │
│    - Booking saga success rate ≥ 99.9% │
│    - p95 latency < 3s                  │
│    - No new P1/P2 alerts               │
│    - DLQ depth = 0                     │
│  - If degradation detected: manual     │
│    rollback (CD-004)                   │
└────────────────┬───────────────────────┘
                 │ stable
                 ▼
┌────────────────────────────────────────┐
│  Stage 6: Post-Deploy Actions          │
│  - Update Slack #deployments with      │
│    deploy summary                      │
│  - Update status page                  │
│  - Create Git tag: v{version}          │
│  - Close staging release branch PR     │
└────────────────────────────────────────┘
```

---

## 4. Slack Deployment Notifications

| Event | Message |
|---|---|
| Approval requested | `🔔 Production deploy awaiting approval for {sha}. Approvers: @engineering-lead` |
| Approved | `✅ Production deploy approved by {approver}. Starting deployment.` |
| Migration started | `🗄️ Flyway migration running against production DB.` |
| ECS update started | `🚀 ECS rolling update started. Services deploying in order.` |
| Smoke tests passed | `✅ Smoke tests passed. Monitoring for 10 minutes.` |
| Deploy complete | `✅ Production deploy {sha} complete. v{version} is live.` |
| Any failure | `🚨 Production deploy FAILED at stage {N}. Auto-rollback initiated.` |

---

## 5. Automatic Rollback Conditions

The pipeline triggers automatic rollback (CD-004) if:

- Any smoke test returns non-200
- ECS deployment circuit breaker fires (task fails to reach RUNNING state)
- Flyway migration fails (service update does not proceed)

The on-call engineer can also trigger manual rollback at any point during the 10-minute monitoring window.

---

## 6. Deployment Windows

Preferred deployment windows (lowest risk):

| Window | Day | Time (UTC) |
|---|---|---|
| Primary | Tuesday–Thursday | 10:00–14:00 |
| Secondary | Monday | 10:00–12:00 |
| Avoid | Friday–Sunday | Any time |
| Avoid | Any day | After 16:00 UTC |

Emergency hotfix deployments (outside window): follow CD-004 rollback plan first, then HOT-001-hotfix-release.md.

---

## 7. Traceability

| Step | Purpose | NFR |
|---|---|---|
| Manual approval gate | Prevents accidental production deploy | NFR001 |
| Pre-flight: staging deploy check | Ensures staging validation was completed | NFR001 |
| Pre-migration RDS snapshot | Rollback point for data corruption | NFR001 |
| Dependency-ordered ECS rollout | No version mismatch during deploy | NFR001 |
| Post-deploy Grafana monitoring | Early detection of regressions | NFR004, NFR001 |
