---
title: Data Flow Diagram
layer: 03-data
status: current
lastUpdated: 2026-06-28
---

# DM2 · Data Flow Diagram

## Overview

This document describes how data moves through AnjiSchedulo at multiple levels of abstraction. It covers:

- **L0 — System Context**: external actors and boundary crossings
- **L1 — Service-level**: internal services and their data stores
- **Event bus topology**: SNS topics, SQS queues, and DLQ pairings
- **Cache flows**: Redis read/write paths and invalidation triggers
- **Data export flow**: how tenant data leaves the system
- **Audit and observability flows**: how operational data is captured

All data flows enforce the constraint that no data ever crosses a tenant boundary at the application level (BR005, NFR005).

---

## L0 — System Context Diagram

```
  ┌─────────────────────────────────────────────────────────────────────┐
  │                         External Actors                             │
  │                                                                     │
  │  ┌──────────────┐  ┌──────────────┐  ┌───────────────────────────┐ │
  │  │ Tenant Admin │  │    Staff     │  │    End Customer (P3)      │ │
  │  │    (P1)      │  │   Member     │  │ (unauthenticated booking)  │ │
  │  │              │  │    (P2)      │  │                           │ │
  │  └──────┬───────┘  └──────┬───────┘  └─────────────┬─────────────┘ │
  │         │                 │                        │               │
  │         └────────────────┬┴────────────────────────┘               │
  │                          │ HTTPS / JWT                              │
  └──────────────────────────┼──────────────────────────────────────────┘
                             │
                    ┌────────▼────────┐
                    │   API Gateway   │ ← injects correlation_id
                    │  (CloudFront +  │   validates JWT / enforces
                    │   ALB routing)  │   rate limits
                    └────────┬────────┘
                             │
            ┌────────────────┼────────────────┐
            │                │                │
     ┌──────▼──────┐  ┌──────▼──────┐  ┌─────▼──────┐
     │  tenant-    │  │  booking-   │  │availability│
     │  service    │  │  command-   │  │  -service  │
     │             │  │  service    │  │            │
     └──────┬──────┘  └──────┬──────┘  └─────┬──────┘
            │                │               │
     ┌──────▼──────┐  ┌──────▼──────┐  ┌─────▼──────┐
     │ PostgreSQL  │  │ PostgreSQL  │  │   Redis 7  │
     │  (tenants,  │  │(appointments│  │ (slot avail│
     │  configs,   │  │ slot_locks, │  │  cache,    │
     │   staff,    │  │ idempotency │  │ TenantConf │
     │  services)  │  │    keys)    │  │  cache)    │
     └─────────────┘  └─────────────┘  └────────────┘

  ┌────────────────────────────────────────────────────────────────────┐
  │                     External Service Calls                         │
  │  ┌────────────┐  ┌────────────┐  ┌────────────┐  ┌─────────────┐  │
  │  │  Stripe    │  │  SendGrid  │  │   Twilio   │  │  AWS KMS /  │  │
  │  │(payments,  │  │  (email    │  │   (SMS)    │  │ SecretsMan  │  │
  │  │   refunds) │  │  delivery) │  │            │  │             │  │
  │  └────────────┘  └────────────┘  └────────────┘  └─────────────┘  │
  └────────────────────────────────────────────────────────────────────┘

  ┌────────────────────────────────────────────────────────────────────┐
  │                    Platform Operator (P4)                          │
  │  ┌─────────────────────────────────────────────────────────────┐   │
  │  │                       ops-service                           │   │
  │  │  (DLQ management, replay jobs, tenant suspend/deprovision,  │   │
  │  │   data export, billing overrides, anomaly investigation)    │   │
  │  └─────────────────────────────────────────────────────────────┘   │
  └────────────────────────────────────────────────────────────────────┘
```

---

## L1 — Service-Level Data Flow

### Tenant Provisioning Flow (UJ001)

```
  P1 (Tenant Admin)
       │
       │ POST /api/v1/tenants
       ▼
  api-gateway ──────────────────────► tenant-service
                                           │
                              ┌────────────┤ (single transaction)
                              │            │
                              ▼            ▼
                         Stripe API    PostgreSQL
                         (create       ┌────────────────────┐
                          customer)    │ INSERT tenants      │
                              │        │ INSERT tenant_config│
                              │        │ INSERT outbox_record│
                              │        │ INSERT audit_log    │
                              └───────►│ (commit)           │
                                       └────────────────────┘
                                                │
                                                │ (outbox_relay polls every 500ms)
                                                ▼
                                         outbox-relay
                                                │
                                                │ publish to SNS
                                                ▼
                                    ┌───────────────────────┐
                                    │  tenant-provisioned   │
                                    │      SNS topic        │
                                    └───────────┬───────────┘
                                                │ fan-out
                               ┌────────────────┴────────────────┐
                               ▼                                  ▼
                  notification-service                    billing-service
                  (SQS FIFO queue)                        (SQS FIFO queue)
                        │                                        │
                        │                                        │
                   SendGrid API                            Stripe API
                  (welcome email)                    (create subscription)
                        │                                        │
                  notification_records                  billing.subscription
                      INSERT                              _activated event
                                                               │
                                                               ▼
                                                         tenant-service
                                                     (UPDATE tenants.status
                                                      provisioning → active)
```

---

### Booking Request Flow (UJ002)

```
  P3 (End Customer)
       │
       │ POST /api/v1/bookings
       │   { customer_id, staff_id, service_id,
       │     slot_date, slot_start, idempotency_key }
       ▼
  api-gateway ──► booking-command-service
                        │
              ┌─────────┤ (saga orchestration)
              │         │
              ▼         ▼
          availability  PostgreSQL
          -service     ┌──────────────────────────┐
          (Redis       │ 1. CHECK idempotency_key  │
           cache       │ 2. INSERT slot_lock       │◄── UNIQUE constraint
           lookup)     │    (serialisable txn)     │    enforces BR001
                       │ 3. Call Stripe            │
                       └──────────────────────────┘
                                    │
                              ┌─────┴─────┐
                              │  payment  │
                              │  -service │
                              │  (Stripe  │
                              │  capture) │
                              └─────┬─────┘
                                    │ payment.captured
                                    ▼
                       booking-command-service
                            │
                            │ (single transaction)
                            ▼
                       PostgreSQL
                  ┌───────────────────────────┐
                  │ INSERT appointments        │
                  │ INSERT appointment_events  │ ← event_type: confirmed
                  │ UPDATE slot_locks          │ ← mark confirmed
                  │ INSERT outbox_record       │
                  │ INSERT audit_log           │
                  │ DELETE idempotency_key     │ ← mark used
                  └───────────────────────────┘
                                    │
                              outbox-relay
                                    │
                          appointment-confirmed
                               SNS topic
                                    │ fan-out
            ┌───────────────────────┼───────────────────────┐
            ▼                       ▼                        ▼
   notification-service      dashboard-service       analytics-service
   (booking confirmation     (UPDATE               (Kinesis stream
    email + staff alert)      TenantDashboardView)  processing)
```

---

### Cancellation Flow (UJ003)

```
  P3 (End Customer) or P1 (Tenant Admin)
       │
       │ DELETE /api/v1/bookings/{appointment_id}
       ▼
  api-gateway ──► booking-command-service
                        │
                        │ (validate cancellation window BR002)
                        │ (check within cancellation_hours config)
                        ▼
                   PostgreSQL
              ┌──────────────────────────┐
              │ INSERT appointment_event  │ ← event_type: cancelled
              │ UPDATE slot_locks        │ ← set released_at
              │ DECREMENT customer       │
              │   active_booking_count   │
              │ INSERT outbox_record     │
              │ INSERT audit_log         │
              └──────────────────────────┘
                        │
                  outbox-relay
                        │
              appointment-cancelled
                   SNS topic
                        │ fan-out (saga choreography)
       ┌────────────────┼─────────────────┐
       ▼                ▼                 ▼
payment-service  notification-service  dashboard-service
(Stripe refund   (cancellation email   (UPDATE view)
 if applicable)   to customer + staff)
       │
  (refund.failed → DLQ)
  (slot release NOT blocked — BR009)
```

---

## Event Bus Topology

### SNS Topics → SQS FIFO Queues

```
  SNS Topic                          SQS FIFO Queues (per consumer)
  ─────────────────────────────────────────────────────────────────────
  tenant-provisioned-topic     →  notification-service-tenant-provisioned-queue
                               →  billing-service-tenant-provisioned-queue

  tenant-configured-topic      →  booking-command-service-tenant-configured-queue
                               →  availability-service-tenant-configured-queue

  tenant-suspended-topic       →  booking-command-service-tenant-suspended-queue

  appointment-confirmed-topic  →  notification-service-appointment-confirmed-queue
                               →  dashboard-service-appointment-confirmed-queue
                               →  analytics-service-appointment-confirmed-queue
                               →  availability-service-appointment-confirmed-queue

  appointment-cancelled-topic  →  payment-service-appointment-cancelled-queue
                               →  notification-service-appointment-cancelled-queue
                               →  dashboard-service-appointment-cancelled-queue

  appointment-rescheduled-topic→  notification-service-appointment-rescheduled-queue
                               →  dashboard-service-appointment-rescheduled-queue

  payment-captured-topic       →  booking-command-service-payment-captured-queue

  payment-failed-topic         →  booking-command-service-payment-failed-queue

  notification-failed-topic    →  ops-service-notification-failed-queue
```

**FIFO ordering:** Every SQS queue uses `MessageGroupId = tenant_id` to maintain per-tenant ordering while allowing parallel processing across tenants.

**DLQ pairing:** Every consumer queue has a paired DLQ with `maxReceiveCount = 4`. After 4 failed attempts, the message is moved to the DLQ and ALT003 fires (see operations.json).

---

## Cache Flows (Redis)

### Slot Availability Cache

```
  Customer requests available slots
       │
       ▼
  availability-service
       │
       ├─► Redis HGET slot_cache:{tenant_id}:{staff_id}:{date}
       │        │
       │   ┌────┴────┐
       │   │  HIT?   │
       │   └────┬────┘
       │        │ YES → return cached slots (TTL: 30s)
       │        │ NO  → query PostgreSQL (working_hours, staff_blocks, slot_locks)
       │             → populate Redis cache
       │             → return slots
       │
       │ Cache invalidation triggers:
       │   appointment.confirmed  → invalidate slot
       │   appointment.cancelled  → invalidate slot (slot becomes available)
       │   tenant.configured      → invalidate all slots for tenant
       │   staff_block created    → invalidate affected date range
       └─────────────────────────────────────────────────────────
```

### TenantConfig Cache

```
  booking-command-service receives booking request
       │
       ▼
  Read TenantConfig from Redis:
    GET tenant_config:{tenant_id}  (TTL: 300s)
       │
       ├─ HIT → use cached config for rule enforcement
       │         (cancellation_hours, booking_window_days,
       │          max_advance_bookings, plan limits)
       │
       └─ MISS → query PostgreSQL tenant_configs
                → populate Redis cache
                → apply rules

  Invalidation:
    tenant.configured event → booking-command-service
    → DEL tenant_config:{tenant_id}
    → next request repopulates from PostgreSQL
```

---

## Outbox → SNS Data Flow

```
  Service writes domain event:
  ┌──────────────────────────────────────────────────────────────┐
  │ BEGIN TRANSACTION                                            │
  │   INSERT INTO domain_table ...     (e.g. appointment)       │
  │   INSERT INTO appointment_events ...                         │
  │   INSERT INTO outbox_records                                 │
  │     (outbox_id, tenant_id, topic, event_type, payload,      │
  │      created_at, published_at = NULL)                        │
  │   INSERT INTO audit_log ...                                  │
  │ COMMIT                                                       │
  └──────────────────────────────────────────────────────────────┘
                         │
                         │ (every 500ms)
                         ▼
                   outbox-relay
                   SELECT * FROM outbox_records
                   WHERE published_at IS NULL
                         │
                         │ for each record:
                         ▼
                     SNS.publish(topic, payload)
                     ← success?
                         │
                         ▼
                   UPDATE outbox_records
                   SET published_at = NOW()
                   WHERE outbox_id = ?
```

---

## Data Export Flow (UJ007)

```
  P1 (Tenant Admin) requests export
       │
       │ POST /api/v1/tenants/{tenant_id}/export
       ▼
  ops-service
       │
       │ (validates tenant_id matches JWT claim — BR005)
       ▼
  PostgreSQL
  ┌─────────────────────────────────────────────────────────────────┐
  │ SELECT * FROM appointments WHERE tenant_id = $1                 │
  │ SELECT * FROM appointment_events WHERE tenant_id = $1           │
  │ SELECT * FROM payments WHERE tenant_id = $1                     │
  │ SELECT * FROM customers WHERE tenant_id = $1                    │
  │ SELECT * FROM notification_records WHERE tenant_id = $1         │
  │ SELECT * FROM audit_log WHERE tenant_id = $1                    │
  └─────────────────────────────────────────────────────────────────┘
       │
       ▼
  Stream to S3 (anji-schedulo-tenant-exports)
  /{tenant_id}/{export_timestamp}/
    ├── appointments.csv
    ├── events.ndjson
    ├── payments.csv
    ├── customers.csv       ← PII included (scoped to requesting tenant only)
    ├── notifications.csv
    └── audit.csv
       │
       ▼
  Generate pre-signed S3 URL (expires 24 hours)
       │
       ▼
  Send URL to tenant's owner_email via SendGrid
  INSERT notification_records (channel: email, template: export_ready)
  INSERT audit_log (operation: export, actor: tenant admin)
```

> Export is scoped strictly to `tenant_id` from the JWT claim. The query includes an explicit `WHERE tenant_id = $1` on every table. Cross-tenant queries are structurally impossible through the API — additionally, PostgreSQL RLS denies any row not matching the session's `app.tenant_id` (BR005).

---

## Audit Log Data Flow

```
  Every write operation in any service:
       │
       ├─► INSERT audit_log (same transaction)
       │     { audit_id, tenant_id, entity_type, entity_id,
       │       operation, actor_id, actor_role, occurred_at,
       │       before_snapshot, after_snapshot }
       │
       └─► Stream to S3 (anji-schedulo-audit-logs bucket)
               via Kinesis Firehose or monthly batch export
               S3 Object Lock (Compliance mode — WORM)
               Retained permanently (BR012, BR014)
```

---

## Observability Data Flow

```
  All services emit:
  │
  ├─► Structured logs → Fluent Bit sidecar
  │     → Loki (3-day hot, 90-day warm, 1-year cold)
  │     Redaction rules: email, phone, name, recipient_contact
  │
  ├─► Metrics → OpenTelemetry Collector
  │     → Mimir (13-month retention)
  │     Key metrics: booking_latency_p95, outbox_lag_seconds,
  │       dlq_depth_total, slot_lock_conflicts_total
  │
  └─► Traces → OpenTelemetry Collector
        → Tempo (7-day retention)
        Sampling: 100% for booking/cancellation/payment paths
                  10% for read-only availability queries
        span attributes: tenant_id, correlation_id, appointment_id
```

---

## Data Residency Summary

| Data Class | Primary Store | Backup | Region |
|---|---|---|---|
| Operational data (tenants, appointments, events) | PostgreSQL 15, RDS eu-west-1 | RDS automated snapshot + cross-region copy to eu-central-1 | EU |
| Slot availability cache | Redis 7, ElastiCache eu-west-1 | No backup (derived, rebuilds from PostgreSQL) | EU |
| TenantConfig cache | Redis 7, ElastiCache eu-west-1 | No backup (derived) | EU |
| Event envelopes (in-flight) | SQS FIFO, eu-west-1 | 4-day retention on DLQs | EU |
| Tenant data exports | S3 eu-west-1 | S3 versioning | EU |
| Event archives | S3 eu-west-1 → Glacier after 12 months | S3 versioning | EU |
| Audit logs | S3 eu-west-1 (WORM) | S3 Object Lock — permanent | EU |
| Metrics | Mimir, eu-west-1 | 13-month in-cluster retention | EU |
| Logs | Loki, eu-west-1 | 90-day warm tier | EU |
| Traces | Tempo, eu-west-1 | 7-day retention | EU |

All data is stored within the European Union. No data transits outside EU/EEA regions except for third-party API calls to Stripe (processes card data under PCI DSS Level 1), SendGrid (processes email addresses), and Twilio (processes phone numbers) — each under Data Processing Agreements compliant with GDPR Chapter V.
