# RES-004 · Backup and Restore Flow

**Project:** AnjiSchedulo  
**Version:** 1.0.0  
**Last Updated:** 2026-06-28

---

## 1. Overview

This document describes the automated backup schedules, retention policies, and restore procedures for all AnjiSchedulo data stores. Backups are fully automated — no operator action is required for routine backup execution. Operator action is only required for restore operations.

---

## 2. Backup Inventory

| Data Store | Backup Method | Schedule | Retention | RPO |
|---|---|---|---|---|
| RDS PostgreSQL | Automated daily snapshot + WAL archiving | Daily 03:00 UTC | 7 days snapshots; WAL continuous | 5 minutes |
| RDS (cross-region) | Snapshot copy to eu-central-1 | Nightly | 30 days | 24 hours |
| RDS (pre-migration) | Manual snapshot before every deploy | Per deploy | Indefinite | At deploy time |
| ElastiCache Redis | Automated snapshot | Daily 02:00 UTC | 1 day | 24 hours |
| S3 tenant-exports | S3 versioning (deletion recovery) | Continuous | 90 days (lifecycle) | Near real-time |
| S3 event-archives | S3 versioning + Glacier after 12 months | Continuous | Permanent | Near real-time |
| S3 audit-logs | S3 Object Lock (WORM) | Continuous | Permanent | Near real-time |
| appointment_events | Monthly NDJSON export to S3 | 1st of month 02:00 UTC | Permanent | 1 month |

---

## 3. RDS Automated Backup Flow

```
Daily at 03:00 UTC
          │
          ▼
RDS automated backup window opens
    Duration: 03:00–04:00 UTC
    I/O not affected for Multi-AZ (snapshot taken from standby)
          │
          ▼
RDS creates automated snapshot:
    Name: rds:anji-schedulo-{env}-{YYYY-MM-DD}
    Storage: S3 (managed by RDS — not visible in S3 console)
    Encrypted: SSE-KMS (env CMK)
    Retention: 7 days (older snapshots auto-deleted by RDS)
          │
          ▼
WAL archiving runs continuously:
    PostgreSQL WAL segments → RDS managed S3 (every 5 minutes)
    Enables PITR to any second within the 7-day window
          │
          ▼
Cross-region snapshot copy (nightly, after backup completes):
    Source: latest automated snapshot in eu-west-1
    Destination: eu-central-1
    Encrypted: copied with destination region KMS key
    Retention: 30 days
```

---

## 4. RDS Restore Procedures

### 4.1 Point-In-Time Recovery (PITR)

Use when: data corruption, accidental deletion, or need to recover to a specific second.

```bash
# Determine restore time (ISO-8601)
RESTORE_TIME="2026-06-28T09:30:00Z"

# Initiate restore (creates a NEW instance)
aws rds restore-db-instance-to-point-in-time \
  --source-db-instance-identifier anji-schedulo-production \
  --target-db-instance-identifier anji-schedulo-production-restored \
  --restore-time $RESTORE_TIME \
  --db-instance-class db.t4g.medium \
  --multi-az \
  --db-subnet-group-name anji-schedulo-production-subnet-group \
  --vpc-security-group-ids sg-{rds-sg-id} \
  --region eu-west-1

# Wait for instance to be available
aws rds wait db-instance-available \
  --db-instance-identifier anji-schedulo-production-restored

# Verify data integrity before switching
psql -h {restored-endpoint} -U readonly_user -d anjischedulo \
  -c "SELECT COUNT(*), MAX(created_at) FROM appointments;"
```

### 4.2 Restore from Automated Snapshot

Use when: need to restore to a daily snapshot rather than a specific time.

```bash
# List available snapshots
aws rds describe-db-snapshots \
  --db-instance-identifier anji-schedulo-production \
  --snapshot-type automated \
  --query 'DBSnapshots[*].{ID:DBSnapshotIdentifier,Time:SnapshotCreateTime}' \
  --output table

# Restore from snapshot
aws rds restore-db-instance-from-db-snapshot \
  --db-instance-identifier anji-schedulo-production-restored \
  --db-snapshot-identifier rds:anji-schedulo-production-2026-06-28 \
  --db-instance-class db.t4g.medium \
  --multi-az
```

### 4.3 Restore from Pre-Migration Snapshot

Use when: a database migration caused issues.

```bash
# List pre-migration snapshots
aws rds describe-db-snapshots \
  --filters Name=snapshot-type,Values=manual \
  --query 'DBSnapshots[?contains(DBSnapshotIdentifier, `prod-pre`)].{ID:DBSnapshotIdentifier,Time:SnapshotCreateTime}' \
  --output table

# Restore
aws rds restore-db-instance-from-db-snapshot \
  --db-instance-identifier anji-schedulo-production-restored \
  --db-snapshot-identifier prod-pre-{git-sha}
```

---

## 5. ElastiCache Redis Restore

Redis is a cache — not a source of truth. Cache misses fall through to PostgreSQL (RLS-enforced). Redis does not need disaster recovery.

The daily Redis snapshot is kept for debugging purposes only. After a Redis failure, services restart and the cache warms up naturally within 5–10 minutes as requests flow through.

**Restore procedure:** none needed. If ElastiCache fails, ECS tasks reconnect automatically when the cluster recovers. No data is lost because Redis holds only cached reads.

---

## 6. S3 Object Restore

### 6.1 Tenant Export Archives (90-day versioning)

```bash
# Restore accidentally deleted object from version history
aws s3api list-object-versions \
  --bucket anji-schedulo-tenant-exports-production \
  --prefix {tenant_id}/{date}.ndjson.gz

# Copy specific version back to current
aws s3api copy-object \
  --bucket anji-schedulo-tenant-exports-production \
  --copy-source anji-schedulo-tenant-exports-production/{key}?versionId={version-id} \
  --key {key}
```

### 6.2 Event Archive (Glacier Retrieval)

Objects moved to Glacier Instant Retrieval after 12 months are available within milliseconds (Instant Retrieval tier).

```bash
# Objects in Glacier are retrieved transparently via standard S3 GetObject
# No restore initiation needed for Glacier Instant Retrieval

aws s3 cp \
  s3://anji-schedulo-event-archives-production/2025/06/appointment_events.ndjson.gz \
  ./local-restore.ndjson.gz
```

### 6.3 Audit Logs (WORM — cannot be deleted)

Audit logs under S3 Object Lock COMPLIANCE mode cannot be deleted by any principal. They are always accessible via standard S3 GetObject. No restore procedure is needed — deletion is structurally prevented.

---

## 7. Monthly Event Archive Export Flow

```
EventBridge: 1st of month, 02:00 UTC
          │
          ▼
outbox-exporter Lambda executes:
    SELECT * FROM appointment_events
    WHERE occurred_at >= date_trunc('month', NOW() - INTERVAL '1 month')
      AND occurred_at <  date_trunc('month', NOW())
    Stream → NDJSON → gzip
          │
          ▼
PUT to s3://anji-schedulo-event-archives-production/{year}/{month}/
    appointment_events.ndjson.gz
          │
          ▼
CloudWatch Logs: archive job completion + row count
    If Lambda errors: CloudWatch alarm → Slack #incidents
```

---

## 8. Backup Verification

Backup integrity is not assumed — it is verified on a schedule:

| Backup | Verification | Frequency |
|---|---|---|
| RDS PITR | Restore to non-prod instance; run integrity SQL | Quarterly |
| Cross-region snapshot | Restore to eu-central-1 test instance | Semi-annually |
| Event archive (S3) | Download latest archive; verify NDJSON count matches DB count | Monthly |
| Audit log (S3 WORM) | Verify Object Lock retention policy still active | Quarterly |

---

## 9. Traceability

| Backup Component | NFR | BR | Doc |
|---|---|---|---|
| RDS WAL archiving (RPO 5 min) | NFR001 | — | A12-disaster-recovery.md |
| Cross-region snapshot | NFR001 | — | A12-disaster-recovery.md |
| Pre-migration manual snapshot | NFR007 | — | CD-003-production-deploy.md |
| S3 WORM audit logs | — | BR014 | A6-data-privacy-arch.md |
| Event archive (monthly) | NFR009 | — | INF-004-serverless-provisioning.md |
| S3 versioning (exports) | — | BR012 | FR012 |
