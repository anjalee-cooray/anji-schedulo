# A8 · Scaling Strategy

**Project:** AnjiSchedulo  
**Version:** 1.0.0  
**Owner:** Pubudu Anjalee Cooray  
**Last Updated:** 2026-06-28

---

## 1. Overview

AnjiSchedulo scales horizontally without architectural changes as tenant count grows (NFR010). Every service is stateless and independently scalable. The event-driven architecture decouples scaling concerns — a burst of bookings increases ECS task count for `booking-command-service` without affecting `dashboard-service`. Tenant noisy-neighbour protection is enforced at the API, message bus, and database layers (NFR011).

---

## 2. Scaling Dimensions

| Dimension | Strategy |
|---|---|
| Tenant count growth | Pool tenancy with RLS — no new infrastructure per Starter/Pro tenant |
| Request volume per tenant | ECS HPA on CPU; per-tenant API rate limits at API Gateway |
| Event throughput | SQS FIFO `MessageGroupId = tenant_id`; horizontal consumer scaling on queue depth |
| Database connections | PgBouncer transaction-mode pooling — connection count decoupled from ECS task count |
| Read-heavy endpoints | ElastiCache Redis (availability cache, config cache); Materialised View for dashboard |
| Enterprise tenant isolation | Dedicated `db.r7g.large` RDS cluster — physically separated from Starter/Pro |

---

## 3. Application Layer — ECS Fargate Auto-Scaling

| Service | Scale-Out Trigger | Min Tasks | Max Tasks | Scale-In Cooldown |
|---|---|---|---|---|
| `api-gateway` | CPU > 60% (5 min avg) | 2 | 20 | 5 min |
| `booking-command-service` | CPU > 60% or SQS depth > 50 | 2 | 10 | 5 min |
| `availability-service` | CPU > 70% | 2 | 10 | 5 min |
| `tenant-service` | CPU > 60% | 1 | 5 | 10 min |
| `notification-service` | SQS queue depth > 100 | 1 | 10 | 5 min |
| `dashboard-service` | SQS queue depth > 200 | 1 | 5 | 10 min |
| `analytics-service` | SQS queue depth > 500 | 1 | 5 | 10 min |
| `payment-service` | CPU > 60% | 1 | 5 | 5 min |
| `outbox-relay` | outbox backlog > 100 records | 1 | 3 | 5 min |
| `ops-service` | CPU > 60% | 1 | 3 | 10 min |

All services:
- Stateless — no local state between requests
- Hard memory/CPU limits in task definitions — no container-level noisy-neighbour
- Rolling deployment: 50% minimum healthy, 200% maximum — zero-downtime deploys

---

## 4. Message Bus — SQS FIFO Noisy-Neighbour Protection

SQS FIFO `MessageGroupId = tenant_id` is the primary mechanism for per-tenant isolation on the event bus (NFR011).

- A burst of events from tenant_A queues messages under tenant_A's group — other tenants' groups continue to be processed concurrently and are unaffected.
- No single tenant causes head-of-line blocking for other tenants.
- Consumer services scale ECS task count proportionally to SQS queue depth — each task handles multiple `MessageGroupId` partitions concurrently.

---

## 5. Database Layer — Connection Pooling and Read Scaling

### 5.1 PgBouncer Connection Pooling

| Parameter | Value |
|---|---|
| Mode | Transaction-mode (connection returned to pool after each transaction) |
| Max server connections | 200 per service |
| Max client connections | Unlimited (ECS task count × threads) |

At 10 ECS tasks × 10 threads = 100 application threads → ≤ 200 real DB connections. Adding more ECS tasks does not add DB connections.

### 5.2 Materialised View (Dashboard Read Scaling)

`tenant_dashboard_views` is a pre-aggregated single row per `(tenant_id, view_date)`. Dashboard reads are O(1) — no joins, no aggregations. Decouples dashboard read scaling from `appointment_events` table size (NFR006).

### 5.3 ElastiCache Read Scaling

| Cache | Key Pattern | TTL | Target Hit Rate |
|---|---|---|---|
| Slot availability | `availability:{tenant_id}:{staff_id}:{date}` | 30 seconds | > 80% |
| Tenant config | `tenant_config:{tenant_id}` | 300 seconds | > 95% |
| Idempotency key lookup | `idempotency:{tenant_id}:{key}` | 24 hours | > 70% |

Cache misses always fall through to the RLS-enforced PostgreSQL query — no correctness risk.

### 5.4 Enterprise Dedicated Clusters

Enterprise tenants are provisioned on a dedicated `db.r7g.large` Multi-AZ RDS instance. This physically isolates their write throughput from the shared Starter/Pro cluster and eliminates noisy-neighbour risk at the database layer.

---

## 6. API Layer — Rate Limiting and Throttling

Per-tenant throughput quotas enforced at API Gateway (NFR011):

| Endpoint Class | Rate Limit |
|---|---|
| Public booking endpoints | 100 req/min per `tenant_id` |
| Admin configuration endpoints | 30 req/min per `tenant_id` |
| Operator endpoints | 60 req/min per `operator_id` |
| Auth endpoints | 10 req/min per source IP |

AWS WAF on CloudFront blocks source IPs exceeding 1,000 requests/minute globally.

---

## 7. Frontend and CDN Scaling

- CloudFront serves static assets and caches tenant booking page HTML at edge nodes globally.
- AWS Amplify scales Lambda@Edge horizontally for server-side rendering — no capacity planning.
- Cache keys include `{tenant_slug}` to prevent cross-tenant cache pollution.
- Cache invalidated on `tenant.configured` events (service/staff/hours changes).

---

## 8. Scalability Targets

| Metric | Current Target | Architecture Ceiling |
|---|---|---|
| Concurrent tenants (Starter/Pro) | 1,000 | ~10,000 (shared cluster + RLS overhead) |
| Bookings per minute (platform-wide) | 500 | ~5,000 (ECS + PgBouncer) |
| Slot availability queries per second | 1,000 | ~10,000 (Redis cache hit path) |
| Dashboard reads per second | 5,000 | ~50,000 (Materialised View single-row lookup) |
| Events per second (message bus) | 2,000 | ~20,000 (SQS FIFO ~3,000 msg/s per queue) |
| Enterprise dedicated tenants | 10 | Unlimited (one cluster per Enterprise tenant) |

---

## 9. Scaling Failure Modes

| Failure Mode | Detection | Mitigation |
|---|---|---|
| ECS task count at max, CPU still high | ALT004 (saga latency > 3s) | Increase max task count via Terraform; investigate root cause |
| PgBouncer connection saturation | RDS `connection_count` alert | Increase `max_server_connections`; add PgBouncer sidecar instance |
| Redis eviction under memory pressure | ElastiCache `evictions` metric | Upgrade instance type; tune TTLs |
| SQS backlog growing faster than consumers scale | ALT006 (outbox backlog > 100) | Increase consumer max task count; check for DLQ growth |
| Single tenant causing RLS overhead at scale | Slow query dashboard (Grafana) | Move tenant to dedicated Enterprise cluster |

---

## 10. Traceability

| Scaling Concern | NFR | BR | Pattern |
|---|---|---|---|
| Horizontal tenant growth | NFR010 | BR005 | Pool tenancy + RLS |
| Tenant noisy-neighbour protection | NFR011 | — | SQS FIFO MessageGroupId + per-tenant rate limits |
| Booking latency under load | NFR004 | — | ECS HPA + PgBouncer + ElastiCache |
| Dashboard read performance | NFR006 | — | Materialised View + Redis |
| Event durability during consumer downtime | NFR003 | — | SQS 14-day retention + Outbox Pattern |
