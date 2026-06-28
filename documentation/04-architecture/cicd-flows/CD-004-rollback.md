# CD-004 · Rollback Flow

**Project:** AnjiSchedulo  
**Version:** 1.0.0  
**Last Updated:** 2026-06-28

---

## 1. Overview

This document describes rollback procedures for production. There are two types of rollback:

1. **Automatic rollback** — triggered by the deployment pipeline when smoke tests fail or the ECS circuit breaker fires. Requires no operator action.
2. **Manual rollback** — triggered by the on-call engineer when post-deploy monitoring detects regression, or when a production incident is caused by the deployment.

Rollback does **not** include database schema rollback. Migrations are forward-only — see section 4 for data rollback via PITR.

---

## 2. Automatic Rollback (Pipeline-Triggered)

Fires when:
- Any smoke test (`GET /health` or key endpoint check) returns non-200
- ECS deployment circuit breaker detects a task failing to reach RUNNING state

### Steps (automated, no operator action)

```
Smoke test failure detected
          │
          ▼
Pipeline retrieves previous ECS task definition revision
    (stored as pipeline artifact from the deploy step)
          │
          ▼
aws ecs update-service \
  --cluster anji-schedulo-production \
  --service <service-name> \
  --task-definition <service-name>:<N-1>
          │
          ▼
ECS rolling update reverts to previous task definition
    (same 50%/200% parameters — zero downtime)
          │
          ▼
Smoke tests re-run against rolled-back services
          │
          ▼
Slack #deployments notification:
  "🚨 Auto-rollback complete for {sha}. Previous version restored."
          │
          ▼
Pipeline marked FAILED — investigate before re-deploying
```

---

## 3. Manual Rollback (Operator-Triggered)

Used when a production regression is detected during or after the 10-minute monitoring window, or at any time a deployment is suspected to be the root cause of an incident.

### Decision Criteria

| Symptom | Action |
|---|---|
| Booking saga error rate > 5% | Rollback immediately |
| p95 saga latency > 3s for > 5 minutes | Rollback if no fix available within 15 minutes |
| DLQ growing after deploy | Investigate first; rollback if DLQ is caused by new code |
| Notification failure rate > 10% | Assess impact; rollback if booking flow is affected |
| Cross-tenant isolation violation (ALT005) | Rollback immediately (P1 incident) |

### Rollback Procedure

```
On-call engineer makes rollback decision
          │
          ▼
Notify Slack #incidents:
  "@here Manual rollback initiated for {sha}. Reason: {reason}"
          │
          ▼
Identify previous task definition revision for each service:

  aws ecs describe-services \
    --cluster anji-schedulo-production \
    --services api-gateway booking-command-service ... \
    --query 'services[*].{name:serviceName,td:taskDefinition}'
          │
          ▼
Roll back services in reverse dependency order:
  (api-gateway first, outbox-relay last)

  for SERVICE in api-gateway ops-service tenant-service \
    analytics-service dashboard-service notification-service \
    booking-command-service payment-service availability-service \
    outbox-relay; do

    aws ecs update-service \
      --cluster anji-schedulo-production \
      --service $SERVICE \
      --task-definition $SERVICE:<N-1>
  done
          │
          ▼
Wait for all services to reach RUNNING state with previous revision
  aws ecs wait services-stable --cluster anji-schedulo-production
          │
          ▼
Run smoke tests against rolled-back services
          │
          ▼
Confirm metrics return to baseline on Grafana
  - Booking success rate ≥ 99.9%
  - p95 latency < 3s
  - DLQ depth stable
          │
          ▼
Notify Slack #incidents:
  "✅ Rollback complete. Production stable at {prev-sha}."
          │
          ▼
Write incident report (O5-incident-response.md template)
Open tracking issue with root cause + fix plan
```

---

## 4. Database Schema Rollback

Flyway migrations are **forward-only**. A migration cannot be automatically reversed. Options when a migration causes production issues:

### Option A — Write a forward-fix migration

If the schema change is backward-compatible (e.g. added an optional column that needs to be removed):

1. Write a new Flyway migration that undoes the schema change (e.g. `ALTER TABLE ... DROP COLUMN IF EXISTS ...`).
2. Deploy via the normal hotfix pipeline (HOT-001).
3. Application code rollback (Option 3 above) handles the service binary; schema fix handles the data.

### Option B — Point-In-Time Recovery (PITR)

If the migration caused data corruption (rare — reserved for serious incidents):

1. Stop all write traffic: suspend tenant via `ops-service` or scale down ECS services.
2. Identify the PITR target time (last known-good timestamp, before migration ran).
3. Initiate restore:
   ```bash
   aws rds restore-db-instance-to-point-in-time \
     --source-db-instance-identifier anji-schedulo-production \
     --target-db-instance-identifier anji-schedulo-production-restored \
     --restore-time <ISO-8601-timestamp>
   ```
4. Verify data integrity on restored instance.
5. Update PgBouncer connection string to point to restored instance.
6. Resume traffic.
7. Replay any events that arrived between the PITR target time and the outage (FR011).

**PITR RPO:** 5 minutes. **RTO:** 30 minutes. See A12-disaster-recovery.md.

---

## 5. Rollback Runbook Summary

| Scenario | Procedure | Time to Execute |
|---|---|---|
| Smoke test failure (auto) | Automatic — no operator action | 5–8 minutes |
| Post-deploy regression (manual) | Reverse ECS task definitions | 10–15 minutes |
| Migration data issue | Forward-fix migration | 30–60 minutes |
| Data corruption requiring PITR | PITR + event replay | 60–120 minutes |

---

## 6. Prevention

To reduce rollback frequency:

- All schema migrations tested in staging before production (CD-002)
- Pre-migration RDS snapshot taken every production deploy
- Manual approval gate prevents rushed deploys
- Preferred deployment windows: Tuesday–Thursday 10:00–14:00 UTC
- Post-deploy 10-minute Grafana monitoring window is mandatory
