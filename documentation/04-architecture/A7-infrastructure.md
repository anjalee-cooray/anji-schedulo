---
title: Infrastructure
layer: 04-architecture
status: current
lastUpdated: 2026-06-28
---

# A7 · Infrastructure

## 1. Cloud & Region

| Field | Value |
|---|---|
| Cloud Provider | AWS |
| Primary Region | eu-west-1 (Ireland) |
| Backup Region | eu-central-1 (Frankfurt) |

**Region rationale:** eu-west-1 is selected to satisfy EU GDPR data residency requirements, minimise latency for the UK and mainland Europe customer base, and remain within the same AWS regional group as eu-central-1 (Frankfurt) for cross-region backup replication. All data processing and storage remains within the EU.

---

## 2. Compute — AWS ECS Fargate

All services run on ECS Fargate (serverless containers; no EC2 node management).

| Property | Value |
|---|---|
| Orchestration | AWS ECS Fargate |
| Service Discovery | AWS Cloud Map (private DNS) + AWS ALB (path-based routing) |
| Deployment Model | One ECS service per bounded context service |
| Auto-scaling Triggers | CPU > 70% OR SQS queue depth > 50 messages |
| Min Tasks | 1 per service (production) |
| Max Tasks | 10 per service (configurable per environment) |

**Services:** api-gateway, tenant-service, booking-command-service, availability-service, payment-service, notification-service, dashboard-service, analytics-service, ops-service, outbox-relay — each runs as an independent ECS Fargate service with its own task definition, IAM task role, and auto-scaling policy.

---

## 3. Database — PostgreSQL 15 (AWS RDS)

### Starter and Pro Tiers (Shared Cluster)

| Property | Value |
|---|---|
| Instance Type | db.t4g.medium |
| Deployment | Multi-AZ (synchronous standby in alternate AZ) |
| Storage | gp3 SSD, 100 GB initial with autoscaling to 1 TB |
| Connection Pooling | PgBouncer — transaction mode, max 200 connections per service |
| Encryption at Rest | AES-256 via AWS KMS (aws/rds managed key) |
| Encryption in Transit | TLS 1.3 enforced on all connection strings |

### Enterprise Tier (Dedicated Cluster)

| Property | Value |
|---|---|
| Instance Type | db.r7g.large per Enterprise tenant |
| Deployment | Multi-AZ |
| Storage | io2 SSD, 200 GB initial |
| Encryption | Customer-managed KMS key |

### Extensions

| Extension | Purpose |
|---|---|
| pgcrypto | UUID generation, column-level encryption |
| pg_stat_statements | Query performance monitoring |
| pg_trgm | Fuzzy name search on services and staff |

### Row-Level Security

All tables carry a `tenant_id` column. PostgreSQL RLS policies enforce tenant isolation at the database layer:
- `SET LOCAL app.tenant_id` at transaction start
- Safe default: missing or null `tenant_id` context returns **zero rows**, never all rows (BR005, ADR002)
- RLS policies applied to all SELECT, INSERT, UPDATE, DELETE operations

### Migrations

Flyway (versioned, forward-only migrations). Run as an ECS one-off task before any service rollout. Backwards-compatible expand-contract pattern enforced.

### Backup Strategy

| Property | Value |
|---|---|
| Automated Snapshots | Daily at 03:00 UTC; 7-day retention |
| WAL Archiving | Continuous WAL to S3 every 5 minutes; RPO: 5 minutes |
| Point-in-Time Recovery | Any timestamp within last 7 days |
| Pre-Migration Snapshot | Manual snapshot before every schema migration |
| Cross-Region Copy | Nightly to eu-central-1 via AWS Backup; 30-day retention |
| RTO | 30 minutes (restore + DNS failover + service reconnection) |

---

## 4. Cache — Redis 7 (AWS ElastiCache)

| Property | Value |
|---|---|
| Instance Type | cache.t4g.medium |
| Deployment | Single shard; read replica in alternate AZ |
| Encryption at Rest | AES-256 via AWS KMS |
| Encryption in Transit | TLS 1.3 enforced |
| Eviction Policy | allkeys-lru |

### Use Cases

| Purpose | Cache Key Pattern | TTL |
|---|---|---|
| Slot availability | `availability:{tenant_id}:{staff_id}:{date}` | 30 s |
| TenantConfig hot cache | `tenant_config:{tenant_id}` | 300 s; invalidated on `tenant.configured` |
| BookingIdempotencyKey | `idempotency:{tenant_id}:{idempotency_key}` | 86 400 s (24 h) |
| Rate limit counters | `rate_limit:{tenant_id}:{endpoint}` | 60 s |

All cache keys are namespaced by `tenant_id` (NFR012).

---

## 5. Message Bus — AWS SNS + SQS FIFO

**Topology:** one SNS topic per event type; one SQS FIFO queue per consumer per topic.

- `MessageGroupId = tenant_id` — per-tenant ordering without cross-tenant head-of-line blocking (NFR011, ADR003)
- `maxReceiveCount = 4` — routes to DLQ after 4 failures
- DLQ retention: 14 days; source queue retention: 14 days
- Encryption: AWS KMS on all SNS topics and SQS queues (NFR013)

**Queues** (each has a paired `-dlq`):

| Queue | Event Type | Consumer |
|---|---|---|
| notification-service-tenant-provisioned-queue | tenant.provisioned | notification-service |
| billing-service-tenant-provisioned-queue | tenant.provisioned | billing-service |
| booking-command-service-tenant-configured-queue | tenant.configured | booking-command-service |
| availability-service-tenant-configured-queue | tenant.configured | availability-service |
| booking-command-service-tenant-suspended-queue | tenant.suspended | booking-command-service |
| notification-service-appointment-confirmed-queue | appointment.confirmed | notification-service |
| dashboard-service-appointment-confirmed-queue | appointment.confirmed | dashboard-service |
| analytics-service-appointment-confirmed-queue | appointment.confirmed | analytics-service |
| availability-service-appointment-confirmed-queue | appointment.confirmed | availability-service |
| payment-service-appointment-cancelled-queue | appointment.cancelled | payment-service |
| notification-service-appointment-cancelled-queue | appointment.cancelled | notification-service |
| dashboard-service-appointment-cancelled-queue | appointment.cancelled | dashboard-service |
| availability-service-appointment-cancelled-queue | appointment.cancelled | availability-service |
| analytics-service-appointment-cancelled-queue | appointment.cancelled | analytics-service |
| notification-service-appointment-rescheduled-queue | appointment.rescheduled | notification-service |
| dashboard-service-appointment-rescheduled-queue | appointment.rescheduled | dashboard-service |
| availability-service-appointment-rescheduled-queue | appointment.rescheduled | availability-service |
| dashboard-service-appointment-completed-queue | appointment.completed | dashboard-service |
| booking-command-service-payment-captured-queue | payment.captured | booking-command-service |
| booking-command-service-payment-failed-queue | payment.failed | booking-command-service |
| notification-service-payment-refunded-queue | payment.refunded | notification-service |
| ops-service-notification-failed-queue | notification.failed | ops-service |

---

## 6. Object Storage — AWS S3

### anji-schedulo-tenant-exports-{env}

| Property | Value |
|---|---|
| Purpose | Tenant data export archives (FR012) |
| Access | Private; pre-signed URLs from ops-service (24 h expiry) |
| Lifecycle | Objects expire 90 days after creation (BR012) |
| Encryption | SSE-KMS (customer-managed key) |

### anji-schedulo-event-archives-{env}

| Property | Value |
|---|---|
| Purpose | Monthly NDJSON exports of `appointment_events` for DR |
| Lifecycle | Standard 12 months → Glacier Flexible Retrieval; payment events 7 years |
| Encryption | SSE-KMS |

### anji-schedulo-audit-logs-{env}

| Property | Value |
|---|---|
| Purpose | Nightly exports of `audit_log` table — compliance retention (BR012, BR014) |
| Access | S3 Object Lock — Compliance WORM mode; no deletion permitted |
| Encryption | SSE-KMS |

---

## 7. Secrets Manager — AWS Secrets Manager

All secrets injected into ECS tasks via ARN references — no application-level decryption.

| Secret | Contents | Rotation |
|---|---|---|
| `anji-schedulo/{env}/db/password` | PostgreSQL master and app user passwords | 30 days (automatic Lambda rotation) |
| `anji-schedulo/{env}/stripe/secret-key` | Stripe secret API key | 90 days (manual) |
| `anji-schedulo/{env}/jwt/private-key` | RS256 signing key for JWT | 90 days (rolling key, 15 min grace period) |
| `anji-schedulo/{env}/jwt/public-key` | RS256 validation key | Rotated in sync with private key |
| `anji-schedulo/{env}/sendgrid/api-key` | SendGrid API key | 180 days (manual) |
| `anji-schedulo/{env}/twilio/auth-token` | Twilio Account SID + Auth Token | 90 days (manual) |
| `anji-schedulo/{env}/redis/auth-token` | ElastiCache Redis AUTH token | 30 days (automatic) |

---

## 8. Stream Processing — AWS Kinesis (v1.0)

| Property | Value |
|---|---|
| Shards | 2 |
| Partition Key | `tenant_id` |
| Retention | 24 hours |
| Consumer | analytics-service (Enhanced Fan-Out) |

**Use cases:** booking velocity spike detection (sliding 5-minute window); cancellation rate monitoring (rolling 50-booking window). Introduced in v1.0 — MVP reads from SQS directly.

---

## 9. Networking

| Component | Technology | Notes |
|---|---|---|
| VPC | AWS VPC | Private subnets for all services and databases; public subnet for ALB only |
| Load Balancer | AWS ALB | Path-based routing; HTTPS only |
| CDN | Amazon CloudFront | Static assets + booking page HTML; tenant-aware cache keys |
| DNS | AWS Route 53 | Public zones; private zones for service discovery |
| SSL/TLS | AWS Certificate Manager | Wildcard cert `*.anji-schedulo.com`; custom CNAME for Enterprise |
| Tenant Routing | Subdomain | `{tenant_slug}.anji-schedulo.com` for Starter/Pro; custom domain for Enterprise |
| Service Discovery | AWS Cloud Map | Private DNS namespaces per environment |
