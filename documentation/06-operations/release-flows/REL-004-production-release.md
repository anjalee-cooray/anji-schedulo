# REL-004 · Production Release

**Project:** AnjiSchedulo  
**Last Updated:** 2026-06-28

---

## 1. Purpose

This flow covers the mechanical steps of a production deployment after a Go decision (REL-003). Executed by the Engineering Lead with the on-call engineer monitoring.

---

## 2. Pre-Deploy Checks (10 min before deploy)

```bash
# 1. Confirm production is healthy
# Open Grafana Platform Overview: booking saga ≥ 99.9%, DLQ = 0, no active alerts

# 2. Confirm correct RC branch
git status
# Expected: On branch release/v{N}

# 3. Confirm ECR image exists
aws ecr describe-images \
  --repository-name anji-schedulo \
  --image-ids imageTag=v{N}-rc.{date} \
  --region eu-west-1
```

---

## 3. Deploy Flow

```
GitHub Actions: CD-003 production deploy workflow
    │
    Trigger: manual workflow_dispatch by Engineering Lead
    Input: image_tag = v{N}-rc.{date}
    │
    ├── Step 1: Run migration dry-run (Flyway validate)
    │           If validate fails → abort
    │
    ├── Step 2: Engineering Lead approves production gate in GitHub Actions
    │
    ├── Step 3: Apply Flyway migrations to production DB
    │
    ├── Step 4: Update ECS task definitions (dependency order)
    │           tenant-service → booking-command-service → payment-service →
    │           notification-service → dashboard-service → analytics-service →
    │           availability-service → customer-service → ops-service → api-gateway
    │
    ├── Step 5: ECS rolling deployment per service (min healthy = 50%)
    │
    ├── Step 6: Automated smoke tests (booking creation + cancellation via API)
    │           If smoke test fails → automatic rollback (CD-004)
    │
    └── Step 7: Create git tag and merge RC → main
                git tag v{N}
                git push origin v{N}
                git checkout main && git merge release/v{N}
                git push origin main
```

---

## 4. Post-Deploy Monitoring (10-minute window)

| Metric | Where | Pass threshold |
|---|---|---|
| Booking saga success rate | Grafana → Booking Operations | ≥ 99.9% |
| Booking saga p95 latency | Grafana → Booking Operations | < 3 seconds |
| DLQ depth | Grafana → DLQ Triage | 0 |
| ECS task count per service | AWS Console → ECS | All at configured count |
| ALB 5xx rate | Grafana → Infrastructure Health | < 0.1% |
| `isolation_monitor_violations_total` | Grafana → Security | 0 |

If any metric is outside threshold at the 5-minute mark, assess rollback immediately (O3).

---

## 5. Post-Deploy Actions

```
# Post to Slack #deployments
"✅ DEPLOYED — v{N} is live in production
 SHA: {git-sha}
 Monitoring window: 10 minutes
 On-call: @{engineer}"

# After monitoring window passes:
"🟢 v{N} STABLE — monitoring window passed. Release complete."
```

Then:
1. Update changelog (VER-002).
2. Send internal release comms (COM-001).
3. Start 48-hour stabilisation period (REL-005).

---

## 6. Rollback

If smoke tests fail (automated) or metrics degrade:
- Automated: pipeline reverts to previous task definition.
- Manual: see [O3 · Rollback Plan](../O3-rollback-plan.md) and [CD-004 · Rollback Flow](../../04-architecture/cicd-flows/CD-004-rollback.md).

---

## 7. Traceability

| Concern | Doc |
|---|---|
| Go/No-Go decision | REL-003-go-no-go.md |
| CD pipeline | CD-003-production-deploy.md |
| Rollback | CD-004-rollback.md, O3-rollback-plan.md |
| Post-release stabilisation | REL-005-post-release-stabilisation.md |
| Changelog | VER-002-changelog-release-notes.md |
