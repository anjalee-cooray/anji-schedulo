# A12 · Disaster Recovery

**Project:** AnjiSchedulo  
**Version:** 1.0.0  
**Owner:** Pubudu Anjalee Cooray  
**Last Updated:** 2026-06-28

---

## 1. Recovery Objectives

| Metric | Target | Mechanism |
|---|---|---|
| **RPO** (Recovery Point Objective) | 5 minutes | WAL archiving to S3 every 5 minutes |
| **RTO** (Recovery Time Objective) | 30 minutes | RDS PITR to new instance + Route 53 DNS failover + PgBouncer reconnection |
| **Event log RPO** | 5 minutes | Same as database RPO (event store is in RDS) |
| **Cross-region RPO** | 24 hours | Nightly cross-region RDS snapshot to eu-central-1 (Frankfurt) |

---

## 2. Failure Scenarios and Recovery Procedures

### 2.1 Single ECS Task Failure

**Detection:** ALB health check fails 3 consecutive times → task marked unhealthy.  
**Recovery:** ECS replaces the task automatically (ECS service scheduler). No operator action required.  
**Impact:** < 30 seconds of reduced capacity during task replacement. ALB routes traffic to healthy tasks.

---

### 2.2 ECS Service Complete Outage (All Tasks Fail)

**Detection:** ALT001 or ALT002 fires — error rate or availability SLO breach.  

**Recovery steps:**
1. On-call engineer acknowledges PagerDuty alert (< 15 minutes).
2. Check ECS service events in AWS Console for root cause (bad task definition, image pull failure, secrets injection failure).
3. If bad deployment: auto-rollback should have fired. If not, manually revert to previous task definition:
   ```bash
   aws ecs update-service --cluster anji-schedulo-production \
     --service <service-name> --task-definition <service-name>:<prev-revision>
   ```
4. If infrastructure issue (VPC, ALB): check CloudTrail and VPC Flow Logs.
5. Confirm service recovery via Grafana Platform Overview dashboard.

**Impact:** Duration depends on root cause. RTO: 15–30 minutes for rollback scenarios.

---

### 2.3 PostgreSQL RDS Primary Instance Failure

**Detection:** RDS CloudWatch `DatabaseConnections` drops to zero; ALT001 fires (booking errors); application logs show DB connection failures.

**Recovery (Multi-AZ automatic failover):**
- RDS Multi-AZ is enabled on all clusters. AWS automatically promotes the standby to primary.
- DNS endpoint (`{cluster}.rds.amazonaws.com`) is updated by AWS — typically within 60–120 seconds.
- PgBouncer reconnects automatically when the DNS endpoint resolves to the new primary.
- **Operator action required:** None for Multi-AZ failover — automated. Monitor Grafana for reconnection.

**Recovery (Point-In-Time Recovery to new instance — if data corruption):**
1. Determine the target restore time (last known-good timestamp).
2. Initiate PITR:
   ```bash
   aws rds restore-db-instance-to-point-in-time \
     --source-db-instance-identifier anji-schedulo-production \
     --target-db-instance-identifier anji-schedulo-production-restored \
     --restore-time 2026-06-28T09:00:00Z
   ```
3. Once instance is available, update PgBouncer connection string to point to restored instance.
4. Run `flyway repair` if any migration metadata is inconsistent.
5. Verify data integrity by checking `appointment_events` for the last known event.
6. Update Route 53 CNAME if using custom DNS alias.

**RPO:** 5 minutes (WAL archiving frequency).  
**RTO:** 30 minutes (RDS PITR restore time + DNS propagation + service reconnection).

---

### 2.4 eu-west-1 (Primary Region) Complete Outage

**Detection:** CloudWatch cross-region health check fails; PagerDuty alert from external uptime monitor.

**Recovery (cross-region failover to eu-central-1):**
1. Confirm eu-west-1 is unavailable (not a false alarm).
2. Restore RDS from the nightly cross-region snapshot in eu-central-1:
   ```bash
   aws rds restore-db-instance-from-db-snapshot \
     --db-instance-identifier anji-schedulo-dr \
     --db-snapshot-identifier <latest-cross-region-snapshot> \
     --region eu-central-1
   ```
3. Deploy ECS services to eu-central-1 using the same ECR images (ECR is per-region — images need to be replicated or rebuilt; ECR cross-region replication recommended to pre-provision).
4. Update Route 53 records to point to eu-central-1 ALB.
5. Update Secrets Manager ARNs in ECS task definitions if secrets are not replicated to eu-central-1.
6. Validate all services via smoke tests.

**Data loss window:** Up to 24 hours (last nightly cross-region snapshot). This exceeds the primary RPO (5 min) — acceptable for a full regional outage scenario given the rarity and the multi-AZ protection against single-instance failures.

**RTO:** 60–120 minutes for cross-region failover.

---

### 2.5 Event Store Corruption

**Detection:** `dashboard-service` shows incorrect aggregates; `booking-query-service` returns inconsistent history; manual audit by operator.

**Recovery:**
1. Platform Operator identifies the affected tenant(s) and the corruption scope (date range or specific event IDs).
2. Trigger event replay via ops console (FR011):
   - Scope: `date_range` or `specific_ids` targeting the corrupted window
   - Target: `dashboard-service` and/or other affected consumers
3. Monitor replay job status in ops console.
4. Verify `tenant_dashboard_views.last_event_id` matches the latest `appointment_events.event_id` after replay.
5. Confirm dashboard data is correct.

If the PostgreSQL `appointment_events` table itself is corrupted (not just the derived views):
1. Restore from RDS PITR (section 2.3).
2. Identify uncorrupted event IDs using the S3 monthly event archive (`anji-schedulo-event-archives`).
3. Re-insert any missing events from the S3 archive using the `ops-service` import endpoint.

---

### 2.6 SQS Dead Letter Queue Backlog (Operational Incident)

**Detection:** ALT003 fires — DLQ depth > 0.

**Recovery:** See FR010 DLQ triage procedure and O4-runbook.md. Operational incident — not a disaster recovery scenario. RTO: 4 hours (NFR016).

---

### 2.7 S3 Bucket Data Loss (Tenant Export Archives)

**Detection:** Tenant admin reports download link returns 404.

**Recovery:**
- S3 versioning is enabled on `anji-schedulo-tenant-exports` — accidental deletions can be recovered from version history.
- If the export archive is genuinely lost, trigger a new data export via ops console (`POST /ops/tenants/{id}/export`).

---

## 3. Backup Summary

### 3.1 Database Backups

| Backup Type | Schedule | Retention | Recovery Use |
|---|---|---|---|
| RDS automated snapshot | Daily at 03:00 UTC | 7 days | PITR within last 7 days |
| WAL archiving (PITR) | Continuous (every 5 min) | 7 days | Restore to any second within window |
| Manual pre-migration snapshot | Before every migration | Indefinite | Migration rollback |
| Cross-region snapshot (eu-central-1) | Nightly | 30 days | Regional outage recovery |

### 3.2 Object Storage

| Bucket | Versioning | Protection |
|---|---|---|
| `anji-schedulo-tenant-exports` | Enabled | Version history recovery |
| `anji-schedulo-event-archives` | Enabled | S3 Glacier after 12 months; 7-year retention for payment events |
| `anji-schedulo-audit-logs` | Enabled | S3 Object Lock (WORM, Compliance mode) — no deletion ever |

### 3.3 Event Store Protection

The `appointment_events` table is the source of truth for all appointment history. It is protected by:
1. RDS automated backups + WAL archiving (primary protection)
2. Monthly NDJSON export to `anji-schedulo-event-archives` S3 (secondary)
3. S3 versioning + Glacier tiering on event archives

---

## 4. DR Testing Schedule

| Test | Frequency | Procedure |
|---|---|---|
| RDS PITR restore to non-production instance | Quarterly | Restore to `anji-schedulo-dr-test` instance; verify data integrity; destroy instance |
| Cross-region snapshot restore | Semi-annually | Restore to eu-central-1; run smoke tests; destroy |
| ECS rollback | Every production deployment | Smoke test failure → auto-rollback verified by pipeline |
| Event replay drill | Quarterly | Ops team triggers replay on staging for a known tenant; verify dashboard rebuild |
| Secrets rotation | Per rotation schedule | Automated (DB, Redis); manual (Stripe, JWT, SendGrid, Twilio) |

---

## 5. Incident Severity and Response Time

| Severity | Definition | Response Target |
|---|---|---|
| P1 | Complete booking unavailability; cross-tenant data breach; data loss | 15 minutes to active engagement |
| P2 | Degraded booking performance; DLQ growth; single-tenant outage | 1 hour to active investigation |
| P3 | Individual tenant config issue; notification failure for one tenant | 4 hours (business hours) |

Runbook locations: `ops-service/docs/runbooks/` and O5-incident-response.md.

---

## 6. Traceability

| DR Scenario | NFR | Operations Doc |
|---|---|---|
| ECS task failure | NFR001 | A9-deployment.md |
| RDS PITR recovery | NFR001 | A7-infrastructure.md |
| Cross-region failover | NFR001 | A7-infrastructure.md |
| Event store rebuild via replay | NFR009 | FR011, O4-runbook.md |
| DLQ resolution | NFR016 | FR010, O4-runbook.md |
