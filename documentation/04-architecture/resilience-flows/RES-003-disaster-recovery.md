# RES-003 · Disaster Recovery Flow

**Project:** AnjiSchedulo  
**Version:** 1.0.0  
**Last Updated:** 2026-06-28

---

## 1. Overview

This document describes the step-by-step disaster recovery flows for AnjiSchedulo. It complements A12-disaster-recovery.md (which defines RPO/RTO targets and failure scenarios) by providing operator runbook-style procedures for each recovery path.

**Recovery Targets:**

| Metric | Target |
|---|---|
| RPO (Recovery Point Objective) | 5 minutes (WAL archiving) |
| RTO (Recovery Time Objective) | 30 minutes (PITR + service reconnection) |
| Cross-region RPO | 24 hours (nightly snapshot) |

---

## 2. DR Scenario 1 — RDS Multi-AZ Automatic Failover

**Trigger:** Primary RDS instance failure in one AZ. RDS automatically promotes the standby.

```
RDS primary fails (hardware / AZ outage)
          │
          ▼
RDS triggers automatic Multi-AZ failover:
    - Promotes standby to primary
    - Updates DNS endpoint (typically 60–120 seconds)
    │
    ▼
PgBouncer reconnects automatically:
    - PgBouncer's server_idle_timeout forces reconnection
    - Within 60–90 seconds of DNS update, pool reconnected
    │
    ▼
ECS tasks resume DB operations
    │
    ▼
Operator action: NONE required
    Monitor: Grafana Infrastructure Health dashboard
    Verify: RDS event in AWS console shows failover complete
    Verify: booking_success_rate ≥ 99.9% restores within 2 minutes
```

**Expected impact:** 60–120 seconds of elevated saga errors during failover. ALT001 may fire; alert resolves automatically when reconnection completes.

---

## 3. DR Scenario 2 — Data Corruption Requiring PITR

**Trigger:** Bad migration or application bug corrupts production data.

```
Data corruption detected (wrong data in API responses
  or DBA notifies after query investigation)
          │
          ▼
Step 1: Stop write traffic
    Scale all ECS write services to 0 tasks:
      aws ecs update-service --cluster anji-schedulo-production \
        --service booking-command-service --desired-count 0
      aws ecs update-service ... --service payment-service --desired-count 0
      (outbox-relay, tenant-service, ops-service also scaled to 0)
    Scale read-only services to 0 (to prevent dirty reads):
      api-gateway --desired-count 0
    │
    ▼
Step 2: Identify PITR target time
    Determine last known-good timestamp:
      - Check git commit for migration time
      - Check audit_log for last clean write
      - Target: {ISO-8601 timestamp} just before corruption
    │
    ▼
Step 3: Initiate PITR restore
    aws rds restore-db-instance-to-point-in-time \
      --source-db-instance-identifier anji-schedulo-production \
      --target-db-instance-identifier anji-schedulo-production-restored \
      --restore-time {ISO-8601-timestamp} \
      --db-instance-class db.t4g.medium \
      --multi-az \
      --region eu-west-1
    Wait: RDS console shows instance AVAILABLE (15–20 minutes)
    │
    ▼
Step 4: Verify restored data
    Connect to restored instance:
      psql -h {restored-endpoint} -U readonly_user -d anjischedulo
    Run data integrity checks:
      SELECT COUNT(*) FROM appointments WHERE status = 'confirmed';
      SELECT MAX(occurred_at) FROM appointment_events;
    Confirm data looks correct up to the target restore time
    │
    ▼
Step 5: Update PgBouncer connection string
    Update Secrets Manager secret:
      anji-schedulo/production/db/password → same password, new host
    Update ECS task definitions with new DB_HOST:
      Terraform apply with new RDS endpoint
    │
    ▼
Step 6: Resume traffic (staged)
    Scale up in order:
      1. outbox-relay (desiredCount=1)
      2. booking-command-service (desiredCount=1)
      3. api-gateway (desiredCount=2)
      4. All other services
    │
    ▼
Step 7: Event replay for lost window
    If events were written between PITR target and corruption detection:
      Use FR011 replay via ops-service to re-derive dashboard views
      from the restored appointment_events table
    │
    ▼
Step 8: Post-incident
    Rename restored instance to production (swap DNS or rename)
    Terminate original corrupted instance (after full verification)
    Write PIR ticket with root cause and prevention plan
```

---

## 4. DR Scenario 3 — eu-west-1 Regional Outage

**Trigger:** AWS declares an eu-west-1 service disruption affecting ECS, RDS, or both.

```
eu-west-1 outage confirmed
    (external uptime monitor + PagerDuty alert)
          │
          ▼
Step 1: Confirm scope
    - Is it ECS only? RDS only? Both?
    - Is this a full AZ or full region outage?
    - Check: https://health.aws.amazon.com
    │
    ▼
Step 2: Restore RDS in eu-central-1 (Frankfurt)
    aws rds restore-db-instance-from-db-snapshot \
      --db-instance-identifier anji-schedulo-dr \
      --db-snapshot-identifier {latest-cross-region-snapshot} \
      --db-instance-class db.t4g.medium \
      --region eu-central-1
    Wait for AVAILABLE status
    │
    ▼
Step 3: Deploy ECS services in eu-central-1
    Terraform workspace: dr
    terraform apply -var-file=environments/dr/terraform.tfvars
    (ECR images must be cross-region replicated — enable ECR replication
     in advance as a preventive measure)
    │
    ▼
Step 4: Update Route 53 records
    Change api.anji-schedulo.com ALIAS to eu-central-1 ALB
    Change app.anji-schedulo.com ALIAS to eu-central-1 CloudFront
    TTL was already set to 60s for fast failover
    │
    ▼
Step 5: Validate in eu-central-1
    Run smoke tests against DR endpoints
    Verify booking flow works against DR DB
    │
    ▼
Step 6: Communicate to tenants
    Status page update: "We are experiencing a regional issue.
    Service has been restored in an alternate region.
    Data loss window: up to 24 hours."
    │
    ▼
Step 7: Failback (when eu-west-1 recovers)
    Re-provision eu-west-1 environment
    Sync any new data from eu-central-1 DR instance
    Switch Route 53 back to eu-west-1
    Decommission eu-central-1 DR environment
```

**Data loss window:** Up to 24 hours (last nightly cross-region snapshot). This is the accepted risk for a full regional outage.

---

## 5. DR Scenario 4 — Event Store Rebuild

**Trigger:** Derived read models (dashboard views, analytics) show incorrect data. `appointment_events` (source of truth) is intact.

```
Incorrect dashboard or analytics data detected
          │
          ▼
Identify affected scope:
    - Which tenants? (all or specific)
    - Which time range? (all-time or date range)
    │
    ▼
Trigger event replay via ops-service (FR011):
    POST /ops/replay
    {
      "scope": "tenant",           // or "date_range" / "specific_ids"
      "tenant_id": "{tid}",        // if scope = tenant
      "from_date": "...",          // if scope = date_range
      "to_date": "...",
      "target_consumers": ["dashboard-service", "analytics-service"]
    }
    │
    ▼
ops-service creates ReplayJob record (state: pending → running)
    │
    ▼
outbox-relay re-publishes appointment_events to SNS
    → dashboard-service rebuilds tenant_dashboard_views
    → analytics-service rebuilds analytics_summaries
    │
    ▼
Monitor ReplayJob via ops console
    GET /ops/replay/{job_id}
    Wait for state: completed
    │
    ▼
Verify dashboard data is correct
```

---

## 6. DR Testing Schedule

| Test | Frequency | Procedure |
|---|---|---|
| RDS PITR restore to non-production | Quarterly | Restore to `anji-schedulo-dr-test`; verify integrity; destroy |
| Cross-region snapshot restore | Semi-annually | Restore to eu-central-1; run smoke tests; destroy |
| Event replay drill | Quarterly | Ops team triggers replay on staging; verify dashboard rebuild |
| Regional failover simulation | Annually | Table-top exercise with full team |

---

## 7. Traceability

| DR Scenario | NFR | Doc |
|---|---|---|
| Multi-AZ failover | NFR001 | A12-disaster-recovery.md |
| PITR data recovery | NFR001 | A12-disaster-recovery.md |
| Cross-region failover | NFR001 | A12-disaster-recovery.md |
| Event replay | NFR009 | FR011, O4-runbook.md |
