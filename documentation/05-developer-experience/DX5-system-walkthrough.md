# DX5 · System Walkthrough

**Project:** AnjiSchedulo  
**Version:** 1.0.0  
**Last Updated:** 2026-06-28

---

## 1. Overview

This document walks through the AnjiSchedulo codebase from the perspective of a developer who has completed local setup (DX1) and wants to understand how the system fits together. It traces the two most important flows — booking creation and cancellation — through every layer of the architecture.

---

## 2. Repository Structure

```
anji-schedulo/                    # Gradle 8 multi-module monorepo root
  services/
    api-gateway/                  # Spring Boot — JWT auth, routing, rate limiting
    booking-command-service/      # Spring Boot — booking saga orchestrator
    availability-service/         # Spring Boot — slot availability + slot lock
    payment-service/              # Spring Boot — Stripe integration
    tenant-service/               # Spring Boot — tenant provisioning + config
    notification-service/         # Spring Boot — email + SMS via SendGrid + Twilio
    dashboard-service/            # Spring Boot — tenant dashboard view projections
    analytics-service/            # Spring Boot — analytics summary projections
    ops-service/                  # Spring Boot — platform operator tools (DLQ, replay)
    outbox-relay/                 # Spring Boot — reads outbox_records, publishes to SNS
  web/                            # Next.js 14 — customer + tenant admin frontend
  packages/
    shared-types/                 # Event envelope records, domain types (Java)
    shared-telemetry/             # OpenTelemetry Java agent configuration
    shared-db/                    # Base repository with TenantContext SET LOCAL
    shared-auth/                  # JWT filter, TenantContextFilter
    database/
      migrations/                 # Flyway SQL migration files
      seeds/                      # Seed data scripts
  infra/
    modules/                      # Reusable Terraform modules
    environments/                 # dev / staging / production tfvars
  documentation/                  # This folder
  specs/                          # User specs and AI-generated specs
```

---

## 3. Booking Creation Flow (Saga Orchestration)

This is the most important flow in the system. Follow it to understand how CQRS, the Outbox pattern, and saga orchestration work together.

### Step 1 — HTTP Request enters api-gateway

**File:** `services/api-gateway/src/main/java/com/anjischedulo/gateway/BookingController.java`

```
POST /tenants/{slug}/bookings
Authorization: Bearer {jwt}
{
  "staffId": "...",
  "serviceId": "...",
  "slotStart": "2026-07-01T10:00:00Z",
  "stripePaymentIntentId": "pi_...",
  "idempotencyKey": "client-generated-uuid"
}
```

`api-gateway` validates the JWT (RS256, checks `tid` claim matches `{slug}`), rate-limits the request, enriches it with `X-Tenant-Id` and `X-Correlation-Id` headers, and forwards to `booking-command-service`.

### Step 2 — Saga orchestrated in booking-command-service

**File:** `services/booking-command-service/src/main/java/com/anjischedulo/booking/BookingService.java`

```
BookingService.createBooking(command)
  │
  ├── [Idempotency check]
  │     SELECT * FROM booking_idempotency_keys
  │     WHERE tenant_id=$1 AND idempotency_key=$2
  │     → if found: return cached result (BR006)
  │
  ├── [Saga Step 1: Slot availability check]
  │     HTTP POST availability-service/internal/slots/lock
  │     { tenantId, staffId, slotStart, appointmentId }
  │     → availability-service: SET NX in Redis (slot_lock:{appointmentId})
  │     → TTL: 300s (5-minute hold)
  │     → if lock fails: throw SlotUnavailableException (BR001)
  │
  ├── [Saga Step 2: Payment capture]
  │     HTTP POST payment-service/internal/capture
  │     { tenantId, stripePaymentIntentId, amount }
  │     → payment-service: POST /v1/payment_intents/{id}/confirm to Stripe
  │     → if payment fails: compensate → release slot lock → throw
  │
  └── [Saga Step 3: Persist + emit event]
        BEGIN TRANSACTION
          INSERT INTO appointments (...)   → appointment_id
          INSERT INTO appointment_events (event_type='appointment.confirmed', payload={...})
          INSERT INTO outbox_records (event_type='appointment.confirmed', payload={...})
          INSERT INTO booking_idempotency_keys (...)
        COMMIT
        → return { appointmentId, status: 'confirmed' }
```

**Key design points:**
- Slot lock (Redis) and payment happen BEFORE the DB write. If either fails, the DB is never written to — no compensation needed for the DB.
- The outbox record is written in the same transaction as the appointment. Either both succeed or neither does (atomicity).
- The saga never calls back to check state — it flows forward, compensating explicitly on each failure.

### Step 3 — outbox-relay publishes the event

**File:** `services/outbox-relay/src/main/java/com/anjischedulo/outbox/OutboxRelayService.java`

```
outbox-relay polls outbox_records (every 100ms):
  SELECT * FROM outbox_records WHERE published_at IS NULL ORDER BY created_at LIMIT 100
  
  For each record:
    sns.publish({
      TopicArn: SNS_TOPIC_ARN[record.event_type],
      Message: JSON.stringify(record.payload),
      MessageGroupId: record.tenant_id,     ← FIFO ordering per tenant
    })
    UPDATE outbox_records SET published_at = NOW() WHERE id = $1
```

### Step 4 — Consumers process the event

SNS fans out to SQS queues. Each consumer processes independently:

**notification-service** (`services/notification-service/src/main/java/com/anjischedulo/notification/NotificationConsumer.java`):
- Reads customer name + email from `event.payload` (no back-call to booking service)
- Selects template based on `event_type`
- Calls SendGrid API → sends confirmation email to customer and staff
- Writes `notifications` record

**dashboard-service** (`services/dashboard-service/src/main/java/com/anjischedulo/dashboard/DashboardConsumer.java`):
- Increments `tenant_dashboard_views.confirmed_bookings_today`
- Updates `last_event_id` watermark

**analytics-service** (`services/analytics-service/src/main/java/com/anjischedulo/analytics/AnalyticsConsumer.java`):
- Updates `analytics_summaries` aggregate counts

---

## 4. Cancellation Flow (Saga Choreography)

Cancellation uses choreography (not orchestration) — each participant reacts to the event independently.

```
DELETE /tenants/{slug}/appointments/{id}
          │
          ▼
booking-command-service:
  1. UPDATE appointments SET status='cancelled' WHERE appointment_id=$1
  2. INSERT INTO appointment_events (event_type='appointment.cancelled', ...)
  3. INSERT INTO outbox_records (event_type='appointment.cancelled', ...)
  4. COMMIT → return 200
          │
          ▼
outbox-relay publishes appointment.cancelled to SNS
          │
          ├── availability-service receives event
          │     Releases slot (DELETE Redis key OR marks slot available in DB)
          │
          ├── payment-service receives event
          │     POST /v1/refunds to Stripe
          │     If refund fails → writes to DLQ (BR009: slot still released)
          │     If refund succeeds → updates payment record
          │
          └── notification-service receives event
                Sends cancellation email/SMS to customer and staff
```

**Why choreography here?** Cancellation participants can fail independently. A failed refund doesn't block slot release or notification. Each participant processes at their own pace, with their own retry policy.

---

## 5. Key Files to Know

| File | Why it matters |
|---|---|
| `services/api-gateway/src/main/java/com/anjischedulo/gateway/JwtAuthFilter.java` | JWT validation, `tid` claim extraction, TenantContext setup |
| `services/booking-command-service/src/main/java/com/anjischedulo/booking/BookingService.java` | The saga orchestrator — most critical business logic |
| `packages/shared-db/src/main/java/com/anjischedulo/db/BaseRepository.java` | Sets `SET LOCAL app.tenant_id` on every transaction (RLS enforcement) |
| `packages/shared-types/src/main/java/com/anjischedulo/types/EventEnvelope.java` | The event envelope record — must be used for all events |
| `database/migrations/V001__initial_schema.sql` | Full schema with RLS policies |
| `services/outbox-relay/src/main/java/com/anjischedulo/outbox/OutboxRelayService.java` | Transactional outbox polling loop |
| `services/ops-service/src/main/java/com/anjischedulo/ops/ReplayService.java` | Event replay implementation (FR011) |

---

## 6. How Tenant Isolation Works in Code

Every database transaction goes through the base repository:

```java
// packages/shared-db/src/main/java/com/anjischedulo/db/BaseRepository.java
@Transactional
public <T> T withTenant(UUID tenantId, Supplier<T> fn) {
    // Executed at the start of each transaction via TransactionSynchronizationManager
    jdbcTemplate.execute(
        "SET LOCAL app.tenant_id = '" + tenantId + "'");  // activates RLS
    return fn.get();
}
```

In practice this is wired via a Spring `TransactionSynchronizationAdapter` that fires `SET LOCAL app.tenant_id` at the start of each `@Transactional` method, using the `tenantId` stored in the `TenantContextFilter`'s `ThreadLocal`.

PostgreSQL RLS then enforces `tenant_id = current_setting('app.tenant_id')` on every read and write. If `app.tenant_id` is null, RLS returns zero rows — never all rows (BR005 safe default).

---

## 7. Adding a New Feature — Checklist

When adding a new capability to AnjiSchedulo:

1. **Does it cross a bounded context?** If yes, communicate via events (not direct DB access).
2. **Does it touch tenant data?** Add RLS policy and `tenant_id` index to any new table.
3. **Does it need durability?** Use the outbox for event publishing, not direct SNS calls.
4. **Does it need idempotency?** Add an idempotency key check before any write.
5. **Does it need an audit trail?** Write to `audit_log` in the same transaction (BR014).
6. **Does it emit an event?** Use `EventEnvelope` type with all mandatory fields.
7. **Add tests:** unit test for the service, integration test for the saga step, isolation test if new table.
