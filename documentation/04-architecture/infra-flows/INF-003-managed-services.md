# INF-003 · Managed Services Setup Flow

**Project:** AnjiSchedulo  
**Version:** 1.0.0  
**Last Updated:** 2026-06-28

---

## 1. Overview

This document covers the setup and configuration of AWS managed data services: RDS PostgreSQL, ElastiCache Redis, SNS topics, SQS FIFO queues, and S3 buckets. These services are provisioned by Terraform (INF-001 Phases 2–3) and require post-provisioning steps for schema initialisation and seed data.

---

## 2. RDS PostgreSQL 15

### 2.1 Provisioning

```
Terraform creates:
  aws_db_subnet_group  → private subnets in 2 AZs
  aws_security_group   → inbound 5432 from ECS private subnet CIDR only
  aws_db_parameter_group → custom params:
      shared_preload_libraries = 'pg_stat_statements,pgcrypto'
      log_min_duration_statement = 1000  (ms)
      rds.force_ssl = 1
  aws_db_instance:
      engine: postgres 15.x
      instance_class:
          Starter/Pro: db.t4g.medium
          Enterprise:  db.r7g.large (dedicated instance)
      multi_az: true
      storage_encrypted: true
      kms_key_id: {env CMK ARN}
      backup_retention_period: 7
      backup_window: "03:00-04:00"
      maintenance_window: "sun:04:00-sun:05:00"
      deletion_protection: true  (production)
      performance_insights_enabled: true
```

### 2.2 Post-Provisioning Schema Setup

After RDS is available, run Flyway migrations as an ECS one-off task:

```
ECS run-task → flyway-migration-{env}
    Command: flyway migrate
    DB: {rds-endpoint}:5432/anjischedulo
    
Creates all tables with:
  - RLS policies on all tenant-scoped tables
  - Required indexes (including CONCURRENTLY where needed)
  - Privilege grants (booking_app_user, booking_readonly_user, flyway_user)
  - Extensions: pgcrypto, pg_stat_statements
```

### 2.3 PgBouncer Sidecar

PgBouncer runs as a sidecar container on each DB-connected ECS task:

```
Mode: transaction
max_client_conn: unlimited
max_server_connections: 200 (per service)
server_tls_sslmode: require
```

---

## 3. ElastiCache Redis 7

### 3.1 Provisioning

```
Terraform creates:
  aws_elasticache_subnet_group → private subnets
  aws_security_group → inbound 6379 from ECS private subnet only
  aws_elasticache_replication_group:
      engine: redis 7.x
      node_type: cache.t4g.medium
      num_cache_clusters: 2 (primary + replica)
      at_rest_encryption_enabled: true
      transit_encryption_enabled: true
      auth_token: from Secrets Manager
      snapshot_retention_limit: 1 day
      snapshot_window: "02:00-03:00"
```

### 3.2 Key Namespaces

All keys are prefixed with `{tenant_id}:` to enforce tenant isolation in cache:

| Prefix | TTL | Purpose |
|---|---|---|
| `{tid}:availability:{staff_id}:{date}` | 30s | Slot availability cache |
| `{tid}:tenant_config:{tid}` | 300s | Tenant configuration |
| `{tid}:idempotency:{key}` | 86400s | Booking idempotency key |
| `slot_lock:{appointment_id}` | 300s | Slot lock during saga (SET NX) |

---

## 4. SNS Topics

One SNS topic per domain event type, all encrypted with the environment CMK:

| Topic Name | Event |
|---|---|
| `anji-schedulo-{env}-tenant-provisioned` | `tenant.provisioned` |
| `anji-schedulo-{env}-appointment-confirmed` | `appointment.confirmed` |
| `anji-schedulo-{env}-appointment-cancelled` | `appointment.cancelled` |
| `anji-schedulo-{env}-appointment-rescheduled` | `appointment.rescheduled` |
| `anji-schedulo-{env}-appointment-reminder-due` | `appointment.reminder_due` |
| `anji-schedulo-{env}-payment-captured` | `payment.captured` |
| `anji-schedulo-{env}-payment-refunded` | `payment.refunded` |
| `anji-schedulo-{env}-customer-erased` | `customer.erased` |

**Access policy:** only `outbox-relay` task IAM role may call `sns:Publish` on these topics.

---

## 5. SQS FIFO Queues

One SQS FIFO queue (+ one DLQ) per `(consumer service × SNS topic)` subscription:

```
Naming: anji-schedulo-{env}-{consumer}-{event}.fifo
DLQ:    anji-schedulo-{env}-{consumer}-{event}-dlq.fifo

Parameters:
  ContentBasedDeduplication: false (explicit MessageDeduplicationId required)
  MessageRetentionPeriod: 14 days
  VisibilityTimeout: 30s
  ReceiveMessageWaitTimeSeconds: 20 (long poll)
  RedrivePolicy: maxReceiveCount=4, deadLetterTargetArn={dlq-arn}
  SSE-KMS: environment CMK
```

Subscription filter: none — consumers filter by `event_type` in the envelope.

---

## 6. S3 Buckets

### Bucket: `anji-schedulo-tenant-exports-{env}`

```
Versioning: enabled
Encryption: SSE-KMS (env CMK)
Public access: blocked
Lifecycle:
  - Expire objects after 90 days
  - Delete incomplete multipart uploads after 1 day
CORS: none (access via presigned URLs only)
```

### Bucket: `anji-schedulo-event-archives-{env}`

```
Versioning: enabled
Encryption: SSE-KMS (env CMK)
Public access: blocked
Lifecycle:
  - Transition to Glacier Instant Retrieval after 12 months
  - Retain permanently (legal obligation for payment-related events)
```

### Bucket: `anji-schedulo-audit-logs-{env}`

```
Versioning: enabled
Encryption: SSE-KMS (env CMK)
Public access: blocked
Object Lock: COMPLIANCE mode, retention: PERMANENT
  (Objects cannot be deleted by any user including root)
Lifecycle: none — permanent retention
```

---

## 7. Post-Provisioning Verification

After all managed services are provisioned, run this checklist:

| Check | Command / Verification |
|---|---|
| RDS reachable | `psql -h {endpoint} -U flyway_user -d anjischedulo -c '\l'` |
| Flyway migrations applied | `flyway info` — all migrations `Success` |
| RLS active | `SELECT * FROM appointments;` as `booking_app_user` without `SET app.tenant_id` → 0 rows |
| Redis auth works | `redis-cli -h {endpoint} -a {token} PING` → PONG |
| SQS queues created | `aws sqs list-queues --queue-name-prefix anji-schedulo-{env}` |
| SNS topics created | `aws sns list-topics` — all 8 present |
| S3 Object Lock on audit bucket | `aws s3api get-object-lock-configuration --bucket anji-schedulo-audit-logs-{env}` |

---

## 8. Traceability

| Service | NFR | BR | Doc |
|---|---|---|---|
| RDS PostgreSQL + RLS | NFR001, NFR007 | BR005 | A7-infrastructure.md, D8-db-schema.md |
| ElastiCache Redis | NFR004, NFR005 | — | A8-scaling-strategy.md |
| SNS/SQS | NFR003, NFR015 | BR013 | A10-integrations.md |
| S3 audit bucket (WORM) | — | BR014 | A6-data-privacy-arch.md |
| S3 export bucket | — | BR012 | FR012 |
