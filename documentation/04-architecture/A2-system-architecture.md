---
title: System Architecture
layer: 04-architecture
status: current
lastUpdated: 2026-06-28
---

# A2 · System Architecture

## Architectural Style

**Event-Driven Microservices with CQRS, Event Sourcing, and Saga Orchestration/Choreography.**

AnjiSchedulo combines synchronous orchestration for strong-consistency operations with asynchronous choreography for decoupled side effects. The booking outcome is always deterministic for the client while notification outages, analytics delays, or dashboard lag never affect booking availability or correctness.

---

## Hybrid Synchronous / Asynchronous Split

**Synchronous (strong consistency):**
- The booking saga (UJ002, UJ004) — booking-command-service orchestrates slot-lock → payment-capture → appointment.confirmed in one coherent sequence. The client receives a definitive 201 (confirmed), 409 (slot unavailable), or 402 (payment failed) within 3 seconds (NFR004). Compensation is applied inline on any step failure.
- Slot availability queries (UJ002) — availability-service responds synchronously within 500ms p95, backed by Redis cache (NFR005).
- Admin configuration writes (UJ001) — tenant-service writes synchronously and returns success to the admin.

**Asynchronous (eventually consistent):**
- All downstream side effects of a booking confirmation: dashboard updates, notification delivery, analytics ingestion, cache invalidation. These are independent SQS consumers of the `appointment.confirmed` SNS event. Failure in any subscriber does not roll back or delay the booking (BR007, NFR002).
- The cancellation saga (UJ003) uses Saga Choreography — slot release, refund initiation, and notification delivery are independent subscribers of `appointment.cancelled`. Refund failure routes to DLQ without blocking slot release or notification (BR009).

---

## Service Topology

| Service | Bounded Context | Responsibility | Deployment |
|---|---|---|---|
| api-gateway | — | JWT validation, correlation_id injection, rate limiting, routing | ECS Fargate |
| tenant-service | Tenant Context | Tenant lifecycle, config, staff, services, status machine | ECS Fargate |
| booking-command-service | Booking Context | Appointment write operations, booking saga orchestration, event store | ECS Fargate |
| availability-service | Availability Context | Slot computation from WorkingHours + StaffBlocks + SlotLocks; Redis cache | ECS Fargate |
| payment-service | Payment Context | Stripe PaymentIntent capture and refund; Payment record lifecycle | ECS Fargate |
| notification-service | Notification Context | Email/SMS/WhatsApp delivery; retry and DLQ; NotificationRecord | ECS Fargate |
| dashboard-service | Dashboard Context | TenantDashboardView materialised read model; dashboard API | ECS Fargate |
| analytics-service | Analytics Context | Kinesis stream processing; velocity and cancellation rate anomaly detection | ECS Fargate |
| ops-service | Operations Context | DLQ triage, event replay, data export, deprovisioning orchestration | ECS Fargate |
| outbox-relay | Outbox Relay Context | Polls outbox_records; publishes to SNS; marks published_at | ECS Fargate |

---

## Communication Patterns

| From | To | Pattern | Protocol |
|---|---|---|---|
| Client / Browser | api-gateway | REST request | HTTPS (TLS 1.3) |
| api-gateway | Domain services | Synchronous HTTP proxy | HTTPS within VPC |
| Domain services | PostgreSQL | Query / transaction | TLS over TCP (PgBouncer pool) |
| Domain services | outbox_records table | Transactional write | Same PostgreSQL transaction |
| outbox-relay | SNS topics | Publish | HTTPS to AWS SNS |
| SNS topics | SQS FIFO queues | Fan-out | AWS managed |
| Consumer services | SQS FIFO queues | Long-poll consume | AWS SDK |
| booking-command-service | availability-service | Synchronous HTTP | HTTPS within VPC |
| booking-command-service | payment-service | Synchronous HTTP | HTTPS within VPC |
| analytics-service | Kinesis Data Streams | Produce / consume | AWS SDK |

---

## Core Patterns

### Transactional Outbox
Every domain event is written to `outbox_records` in the same PostgreSQL transaction as the state change. outbox-relay polls and publishes to SNS every 500ms. This eliminates dual-write risk: if the service crashes after the DB commit but before the SNS call, the relay will pick up and publish on the next poll. Satisfies NFR003 (no event lost between write and consumption). Used in: booking-command-service, tenant-service, payment-service.

### Saga Orchestration
booking-command-service orchestrates the booking sequence: reserve-slot → capture-payment → write-appointment.confirmed. Each step is synchronous. On any failure, the orchestrator applies compensation inline: release slot, void payment. Returns a deterministic result to the client. Satisfies BR004 (atomic booking) and FR004 (clear 409/402 responses). Used in: booking flow (UJ002), rescheduling flow (UJ004).

### Saga Choreography
The cancellation flow (UJ003) uses choreography. `appointment.cancelled` triggers independent subscribers: payment-service (refund), availability-service (slot release), notification-service (customer and staff notifications). Each has its own DLQ path. Refund failure routes to DLQ without blocking slot release (BR009). Satisfies the requirement that partial step failures are isolated.

### CQRS
Read and write models are fully separated. Write side: AppointmentEvent table owned by booking-command-service. Read side: TenantDashboardView in dashboard-service, updated incrementally from events. Enables < 200ms dashboard reads (NFR006) without impacting booking write throughput. Used in: dashboard-service (FR008, UJ005).

### Event Sourcing
Appointment state is derived by replaying `AppointmentEvent` records in `occurred_at` order — there is no mutable status column on the Appointment row. This enables full event replay for view rebuilding (FR011) and provides an immutable audit trail of every appointment lifecycle transition (BR014). Used in: booking-command-service, Appointment aggregate.

### Event-Carried State Transfer
All published events include a full entity snapshot at event time. Downstream consumers derive all content from the payload without calling back to the originating service. Reduces coupling and eliminates stale-read risk across service boundaries. Applied to all appointment.*, tenant.*, and payment.* events.

### Idempotent Consumer
All SQS consumers maintain an `inbox_records` table. Before processing an event, the consumer inserts `(event_id, consumer_name)`. A unique constraint violation means the event was already processed — skip without side effects. Enables safe at-least-once delivery and safe replay (NFR009, FR011). Used in: all consumer services.

### Dead Letter Queue
Every SQS consumer queue has a paired DLQ. After 4 processing attempts, unprocessable messages route to the DLQ automatically (maxReceiveCount = 4). ops-service monitors all DLQs and surfaces entries for triage (FR010, NFR016). DLQ retention: 14 days.

### Materialised View
`TenantDashboardView` is updated incrementally per AppointmentEvent. Dashboard API reads a single indexed row — O(1) regardless of booking history depth. Satisfies NFR006 (< 200ms p95) and NFR008 (< 5s event lag). The `last_event_id` watermark ensures idempotent incremental updates safe for replay. Used in: dashboard-service.

### Pool with Row-Level Security
All tenants share a single PostgreSQL cluster (Starter/Pro). Every table has `tenant_id`. RLS policies restrict all operations to `current_setting('app.tenant_id')`. Safe default: null context returns zero rows, never all rows. Enterprise tenants receive a dedicated cluster but the same RLS policies apply for defence in depth. Satisfies BR005, NFR012. Governed by ADR002.

### Circuit Breaker
Applied to booking-command-service → payment-service synchronous call during the booking saga. Opens after 5 consecutive failures; half-open probe after 30 seconds. On open circuit, the saga returns 503 with no charge and no slot hold. Prevents the booking endpoint from hanging indefinitely on a payment-service outage (NFR002).

---

## ADR Summary

| ADR | Decision | Status |
|---|---|---|
| [ADR001](adrs/ADR-001-event-driven-architecture.md) | Strong consistency at the slot level; eventual consistency for all derived views | Accepted |
| [ADR002](adrs/ADR-002-shared-database-rls.md) | Pool multi-tenancy with PostgreSQL Row-Level Security over schema-per-tenant | Accepted |
| [ADR003](adrs/ADR-003-sns-sqs-over-kafka.md) | AWS SNS fan-out to SQS FIFO queues as the message bus | Accepted |
| [ADR004](adrs/ADR-004-saga-pattern-selection.md) | Saga Orchestration for booking; Saga Choreography for cancellation | Accepted |
| [ADR005](adrs/ADR-005-java-virtual-threads.md) | PostgreSQL as the appointment event store over a dedicated event store | Accepted |

---

## Service Dependency Graph

```
                    ┌──────────────────────────────────────────────┐
  Client ──HTTPS──▶ │               api-gateway                    │
                    └────────────────┬─────────────────────────────┘
                                     │ routes by path + role
          ┌──────────────────────────┼────────────────────────────────┐
          │                          │                                 │
          ▼                          ▼                                 ▼
  tenant-service           booking-command-service          availability-service
  (tenant lifecycle)       (appointment write + saga)       (slot computation)
          │                    │          │                           │
          │                    │ (sync)   │ (sync)              Redis cache
          │                    ▼          ▼
          │              availability-  payment-
          │              service        service ──▶ Stripe API
          │
          │  All services write outbox_records in the same DB transaction
          │
          ▼
  outbox-relay ──SNS──▶ SNS Topics
                                │
          ┌─────────────────────┼──────────────────────────────┐
          ▼                     ▼                ▼             ▼
  notification-service  dashboard-service analytics-service ops-service
  (email/SMS/WhatsApp) (materialised view) (anomaly detect) (DLQ/replay)
```

Synchronous calls: booking-command-service ↔ availability-service and payment-service during the booking saga only. All other cross-service communication is asynchronous via events.
