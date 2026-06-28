# R9 · Non-Functional Requirements

**Project:** AnjiSchedulo  
**Version:** 1.0.0  
**Owner:** Pubudu Anjalee Cooray  
**Last Updated:** 2026-06-28

---

## 1. Overview

This document defines the non-functional requirements (NFRs) for AnjiSchedulo. These requirements constrain how the system must behave — covering availability, performance, consistency, scalability, security, observability, and recoverability. Each NFR is traceable to enforcement mechanisms and, where applicable, SLOs and alerts.

---

## 2. Availability

### NFR001 — Core Booking Availability
**Requirement:** Core booking operations (create, confirm, cancel) must achieve 99.9% monthly availability.  
**Measurement:** Uptime monitoring via Grafana SLO dashboard.  
**Enforcement:** Multi-AZ deployment on EKS; health checks via AWS ALB; automated pod recovery.  
**Related SLO:** SLO002  
**Error Budget:** 43.8 minutes downtime per month.

---

### NFR002 — Non-Core Service Isolation
**Requirement:** Failure of any non-core service (notifications, analytics, reporting) must not degrade booking availability.  
**Enforcement:** Notification and Analytics services are fully decoupled async subscribers of the event bus. Circuit breakers are applied on synchronous calls to Availability and Payment services. Booking completion does not wait for notification delivery.

---

### NFR003 — Event Durability
**Requirement:** No event is lost between the time it is written and the time it is consumed, even if a consumer is temporarily unavailable.  
**Enforcement:** Transactional Outbox Pattern — events are durably persisted in PostgreSQL in the same transaction as the command. Outbox Relay publishes to SNS. SQS retains messages for up to 14 days.

---

## 3. Performance

### NFR004 — Booking Confirmation Latency
**Requirement:** Booking confirmation end-to-end latency p95 must be under 3 seconds.  
**Measurement:** `bookings_saga_duration_seconds` histogram p95 in Prometheus.  
**Enforcement:** Saga orchestration with synchronous availability check and payment; slot lock acquired via DB constraint; async notification dispatch.  
**Related SLO:** SLO001

---

### NFR005 — Slot Availability Query Latency
**Requirement:** Slot availability query p95 must be under 500ms.  
**Measurement:** `http_server_requests_seconds` histogram p95 for `/availability` endpoints.  
**Enforcement:** Availability Service computes open slots from indexed PostgreSQL queries on `slot_locks` and `tenant_working_hours`.  
**Related SLO:** SLO003

---

### NFR006 — Dashboard Response Latency
**Requirement:** Dashboard view response p95 must be under 200ms.  
**Enforcement:** Served from pre-aggregated Materialized View (`tenant_dashboard_view`) — O(1) read regardless of booking history depth. No joins or aggregation at query time.  
**Related SLO:** SLO004

---

## 4. Consistency

### NFR007 — No Double-Booking
**Requirement:** A slot can never be double-booked, regardless of concurrent requests.  
**Enforcement:** Unique constraint on `slot_locks(tenant_id, staff_id, slot_date, slot_start) WHERE released_at IS NULL`. Any concurrent attempt to insert for the same slot will fail with a unique constraint violation, triggering saga compensation.  
**Related Business Rule:** BR001

---

### NFR008 — Dashboard Event Lag
**Requirement:** All derived views (including the tenant dashboard) must be eventually consistent with the event log, with target lag under 5 seconds under normal load.  
**Enforcement:** Dashboard Service subscribes to all booking events via SQS and applies projections in near-real-time. Lag monitored via `dashboard_projection_lag_seconds`.  
**Related SLO:** SLO004

---

### NFR009 — Replay Idempotency
**Requirement:** Replaying events for a tenant must produce identical derived state to live processing, regardless of how many times events are replayed.  
**Enforcement:** All event consumers implement the Idempotent Consumer pattern using `event_id` watermarks and inbox deduplication (`processed_events` table). Re-processing an already-processed event produces no state change.

---

## 5. Scalability

### NFR010 — Horizontal Tenant Growth
**Requirement:** The platform must support horizontal growth in tenant count without architectural changes.  
**Enforcement:** Pool tenancy model — all tenants share infrastructure with RLS isolation. Event bus partitioned by `tenant_id`. All services are stateless and scale horizontally via EKS HPA.

---

### NFR011 — Tenant Noisy-Neighbour Protection
**Requirement:** A single high-volume tenant must not degrade performance for other tenants.  
**Enforcement:** SQS FIFO queues use per-tenant `MessageGroupId` to prevent a single tenant's event storm from blocking other tenant processing. Dedicated per-tenant partitions for Enterprise high-volume tenants. Per-tenant throughput quotas enforced at the API Gateway.

---

## 6. Security

### NFR012 — Tenant Data Isolation
**Requirement:** Tenant data isolation violations must be zero. A tenant's data must never be readable by another tenant under any condition, including infrastructure failure or application bug.  
**Enforcement:** Three-layer enforcement:
1. JWT `tid` claim validated at API Gateway.
2. Application service filters all queries by `tenant_id` from the JWT.
3. PostgreSQL Row-Level Security enforced at DB layer — safe default returns zero rows when tenant context is null.

**Measurement:** Isolation monitor runs every 5 minutes in production, cross-checking query results against expected tenant scope.

---

### NFR013 — Encryption
**Requirement:** All data in transit and at rest must be encrypted.  
**Enforcement:**
- **In transit:** TLS 1.3 enforced at ALB and between all internal services.
- **At rest:** AES-256 encryption on RDS, S3, SQS, and ElastiCache. Keys managed via AWS KMS.

---

## 7. Observability

### NFR014 — Distributed Tracing
**Requirement:** Every event and request must carry a `correlation_id` traceable across all services.  
**Enforcement:** API Gateway generates `correlation_id` on every inbound request. All services propagate it via HTTP headers (`X-Correlation-ID`), MDC logging context, and event envelopes. Every span in distributed traces includes `correlation_id` as a span attribute.

---

## 8. Extensibility

### NFR015 — Open Event Subscription
**Requirement:** A new downstream consumer of booking events can be added without modifying the booking service.  
**Enforcement:** Booking Service publishes to SNS topics only. New consumers subscribe via a new SQS queue + SNS subscription. No producer code changes required.

---

## 9. Recoverability

### NFR016 — DLQ Resolution SLA
**Requirement:** Dead Letter Queue entries must be resolved within 4 hours of appearing.  
**Measurement:** `dlq.depth > 0` alert triggers within 5 minutes of a DLQ entry appearing; resolution tracked in ops console.  
**Enforcement:** Automated Grafana alert fires to PagerDuty on-call channel. Ops console provides full event payload inspection and re-queue tooling.  
**Related Alert:** ALT003

---

## 10. NFR Summary Table

| ID | Category | Requirement | Target | Enforcement |
|---|---|---|---|---|
| NFR001 | Availability | Core booking uptime | 99.9% monthly | Multi-AZ EKS, ALB health checks |
| NFR002 | Availability | Non-core service isolation | No booking degradation | Async pub/sub, circuit breakers |
| NFR003 | Availability | Event durability | Zero event loss | Transactional Outbox, SQS 14-day retention |
| NFR004 | Performance | Booking confirmation p95 | < 3 seconds | Saga orchestration, async notifications |
| NFR005 | Performance | Slot availability query p95 | < 500ms | Indexed PostgreSQL queries |
| NFR006 | Performance | Dashboard response p95 | < 200ms | Pre-aggregated materialized view |
| NFR007 | Consistency | No double-booking | Zero violations | DB unique constraint on slot_locks |
| NFR008 | Consistency | Dashboard event lag | < 5 seconds | Near-real-time event projection |
| NFR009 | Consistency | Replay idempotency | Identical state | event_id watermarks + inbox dedup |
| NFR010 | Scalability | Horizontal tenant growth | No arch changes | Pool tenancy + stateless services |
| NFR011 | Scalability | Tenant noisy-neighbour | No cross-tenant impact | FIFO MessageGroupId + per-tenant quotas |
| NFR012 | Security | Tenant isolation violations | Zero | JWT + app filter + RLS |
| NFR013 | Security | Encryption | All data in transit + at rest | TLS 1.3 + AES-256 + KMS |
| NFR014 | Observability | Distributed tracing | 100% correlation_id coverage | API Gateway + MDC + event envelope |
| NFR015 | Extensibility | Open event subscription | No producer changes | SNS pub/sub decoupling |
| NFR016 | Recoverability | DLQ resolution SLA | Within 4 hours | Grafana alert + PagerDuty + ops console |
