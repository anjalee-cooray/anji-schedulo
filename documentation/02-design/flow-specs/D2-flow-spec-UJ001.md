---
title: Flow Spec — Tenant Onboarding
journeyId: UJ001
layer: 02-design
status: current
lastUpdated: 2026-06-27
---

# D2 · Flow Spec — UJ001: Tenant Onboarding

## Header

| Field | Value |
|---|---|
| Journey ID | UJ001 |
| Journey Name | Tenant Onboarding |
| Persona | P1 — Tenant Admin (Business Owner) |
| Priority | Critical |
| Related Functional Requirements | FR001, FR002 |
| Related Business Rules | BR005, BR013, BR014 |

---

## 1. Technical Flow Overview

**Architecture Pattern: Pub/Sub Fan-out via Transactional Outbox**

Tenant onboarding is a multi-step provisioning process that must fan-out to independent downstream systems (billing and notification) without coupling the registration response to those systems' availability. The core registration is synchronous — the tenant-service writes the Tenant row and returns a response to the client — but all provisioning side effects are fully asynchronous.

The Transactional Outbox pattern (ADR005, NFR003) eliminates dual-write risk: the `tenant.provisioned` event is written to `outbox_records` in the same PostgreSQL transaction as the Tenant row. The outbox-relay polls for unpublished records every 500 ms and publishes to SNS. From SNS, the event fans out to dedicated SQS FIFO queues for each consumer. Each consumer maintains an InboxRecord for deduplication under at-least-once delivery.

This pattern ensures:
- Registration is never blocked by downstream system availability (billing, notification provider).
- Provisioning steps (billing activation, welcome email) happen independently and can each retry on failure.
- No event can be lost between the DB write and the message bus publish (NFR003).

---

## 2. Services Involved

| Service | Role in This Flow |
|---|---|
| api-gateway | Validates client request, extracts JWT (if present), injects `correlation_id`, routes to tenant-service |
| tenant-service | Owns Tenant aggregate creation. Writes Tenant row and OutboxRecord atomically. Enforces tier limits. Produces `tenant.provisioned`. |
| outbox-relay | Polls `outbox_records` every 500 ms. Publishes `tenant.provisioned` payload to SNS. Marks record as published. |
| notification-service | Consumes `tenant.provisioned`. Sends welcome email to `owner_email` via SendGrid. |
| billing-service | Consumes `tenant.provisioned`. Activates Stripe subscription for the chosen plan. Publishes `billing.subscription_activated`. |

---

## 3. Step-by-Step Technical Flow

### Phase A — Registration (Synchronous)

**Step 1 — Client submits registration request**

The Business Owner (P1) submits a `POST /api/v1/tenants` request with:
```
{ business_name, owner_email, owner_name, plan: "starter|pro|enterprise" }
```

api-gateway receives the request, injects `X-Correlation-ID` (generated UUID if not supplied), and forwards to tenant-service.

**Step 2 — tenant-service validates the request**

tenant-service validates:
- `owner_email` format is valid.
- `plan` is one of `starter | pro | enterprise`.
- No existing Tenant with the same `owner_email` is in `active` or `provisioning` status (prevents accidental duplicate registration).

If validation fails, tenant-service returns `400 Bad Request` with error detail. No DB writes occur.

**Step 3 — tenant-service opens a database transaction**

Within a single PostgreSQL transaction (RLS context: system-level for this INSERT, since the tenant does not yet have a `tenant_id`), tenant-service:

1. Generates a new `tenant_id` (UUID v4).
2. Creates a Stripe Customer via the Stripe API (`POST /v1/customers`). The Stripe `customer_id` is returned synchronously before the transaction commits — this is a pre-condition for the `tenant.provisioned` event payload.
3. Inserts the `tenants` row:
   ```
   tenant_id, name, owner_email, owner_name, plan,
   status = 'provisioning', created_at = NOW()
   ```
4. Inserts `tenant_config` row with plan defaults (cancellation_hours, booking_window_days, max_advance_bookings, timezone, currency).
5. Inserts `outbox_records` row:
   ```
   outbox_id, tenant_id, topic = 'tenant-provisioned-topic',
   event_type = 'tenant.provisioned',
   payload = { event_id, tenant_id, correlation_id, occurred_at,
               name, owner_email, owner_name, plan, stripe_customer_id },
   created_at = NOW(), published_at = NULL
   ```
6. Inserts `audit_log` row:
   ```
   entity_type = 'Tenant', entity_id = tenant_id,
   operation = 'create', actor_role = 'system',
   occurred_at = NOW(), after_snapshot = { tenant row }
   ```

Transaction commits. If any step within the transaction fails (e.g. Stripe API call fails), the entire transaction rolls back and no Tenant row or OutboxRecord is created. The client receives `503 Service Unavailable` with retry guidance.

**Step 4 — tenant-service returns HTTP 202 Accepted**

Response:
```json
{
  "tenant_id": "<uuid>",
  "status": "provisioning",
  "correlation_id": "<uuid>"
}
```

The `202 Accepted` communicates that registration is accepted and provisioning is underway asynchronously. The client can poll `GET /api/v1/tenants/{tenant_id}/status` to track completion.

### Phase B — Event Fan-out (Asynchronous)

**Step 5 — outbox-relay publishes tenant.provisioned to SNS**

outbox-relay polls `outbox_records WHERE published_at IS NULL` across all services every 500 ms. It finds the `tenant.provisioned` record, publishes it to `tenant-provisioned-topic` on SNS with `MessageGroupId = tenant_id`, and only marks `published_at = NOW()` after the SNS publish call succeeds. If the SNS call fails, the record remains unpublished and will be retried on the next poll cycle.

**Step 6 — SNS fans out to subscriber queues**

SNS delivers the `tenant.provisioned` message to two SQS FIFO queues concurrently:
- `notification-service-tenant-provisioned-queue`
- `billing-service-tenant-provisioned-queue`

Each queue is FIFO with `MessageGroupId = tenant_id`, ensuring ordered delivery per tenant. Each queue has a paired DLQ with `maxReceiveCount = 4`.

**Step 7 — notification-service processes the event**

notification-service dequeues the message from `notification-service-tenant-provisioned-queue`:

1. InboxRecord deduplication: attempts `INSERT INTO inbox_records (event_id, consumer_name='notification-service')`. If the insert fails (unique constraint), the event has already been processed — message is deleted from SQS without reprocessing.
2. Reads `owner_email` and `owner_name` from the event payload (Event-Carried State Transfer — no back-call needed).
3. Selects the welcome email template for the plan tier.
4. Calls SendGrid API to send the welcome email with the setup link.
5. Writes `notification_records` row with `status = sent` (or `status = failed` if SendGrid returns an error).
6. If delivery fails: increments `attempt_count`. After 4 failures, sets `status = dlq` and raises a `notification.failed` event to the ops alert queue.
7. Deletes the SQS message on success.

**Step 8 — billing-service processes the event**

billing-service dequeues the message from `billing-service-tenant-provisioned-queue`:

1. InboxRecord deduplication check (as above).
2. Reads `stripe_customer_id` and `plan` from the event payload.
3. Creates a Stripe Subscription (`POST /v1/subscriptions`) for the plan's price ID.
4. On success: publishes `billing.subscription_activated` event (via its own Outbox). This event is consumed by tenant-service.

**Step 9 — tenant-service receives billing.subscription_activated**

tenant-service subscribes to `billing.subscription_activated`. On receipt:

1. InboxRecord deduplication check.
2. Updates `tenants.status` from `provisioning` → `active`.
3. Writes `audit_log` entry for the status transition.
4. Publishes `tenant.configured` event via Outbox (with default config snapshot) so downstream services can cache the initial TenantConfig.

### Phase C — Tenant Configuration (Synchronous)

**Step 10 — Admin configures business hours**

The tenant admin (now authenticated with a JWT containing `tenant_id`, `role = admin`) calls:
```
PUT /api/v1/tenants/{tenant_id}/working-hours
```
tenant-service validates and writes `working_hours` rows per staff member per day. Publishes `tenant.configured` event via Outbox with updated `config_snapshot`.

**Step 11 — Admin adds services**

```
POST /api/v1/tenants/{tenant_id}/services
```
tenant-service enforces the plan's service limit (5 for Starter, unlimited for Pro/Enterprise). Writes `services` row. Publishes `tenant.configured` event.

**Step 12 — Admin adds staff members**

```
POST /api/v1/tenants/{tenant_id}/staff
```
tenant-service enforces the plan's staff limit (3 for Starter, 20 for Pro, unlimited for Enterprise). Writes `staff_members` row. Publishes `tenant.configured` event.

**Step 13 — booking-command-service and availability-service refresh caches**

Both services consume `tenant.configured` from their dedicated SQS FIFO queues:
- booking-command-service: refreshes its local `TenantConfig` cache for booking rule enforcement.
- availability-service: invalidates the Redis slot cache for the tenant so next slot query uses fresh data.

**Booking page is now live.** The tenant's public booking URL (`/{tenant_slug}/book`) begins serving available slots to customers.

---

## 4. Compensation / Failure Flows

### Failure Point 1 — Stripe Customer creation fails during Step 3

**Impact:** The PostgreSQL transaction never commits. No Tenant row, OutboxRecord, or AuditLog entry is written.

**Compensation:** None required (transaction is atomic). The tenant-service circuit breaker on the Stripe API call opens after 5 consecutive failures. The client receives `503 Service Unavailable` with a `Retry-After` header.

**Operator action:** No DLQ involvement — this is a synchronous error. If Stripe is down persistently, the Stripe status page provides SLA guidance.

### Failure Point 2 — outbox-relay SNS publish fails (Step 5)

**Impact:** The `tenant.provisioned` event is not delivered to consumers. The Tenant row is already committed.

**Compensation:** outbox-relay retries the SNS publish on every subsequent poll cycle (every 500 ms) until it succeeds. The OutboxRecord `published_at` remains NULL. An alert fires if outbox backlog exceeds 100 records for more than 10 seconds (observability.json).

### Failure Point 3 — notification-service fails to send welcome email (Step 7)

**Impact:** The tenant admin does not receive the welcome email. The Tenant row is already active.

**Compensation:** notification-service retries the SendGrid call up to 4 times with exponential backoff (delays: 1s, 2s, 4s, 8s). After 4 failures, the `NotificationRecord.status` transitions to `dlq` and a `notification.failed` event is published to `ops-service-notification-failed-queue`. The Platform Operator can re-trigger the welcome email from the ops console. This failure does not affect the tenant's ability to configure or accept bookings (BR007).

### Failure Point 4 — billing-service fails to activate subscription (Step 8)

**Impact:** The tenant's Stripe subscription is not activated. The Tenant may remain in `provisioning` status.

**Compensation:** billing-service retries up to 4 times with exponential backoff on the SQS queue before routing to DLQ (`billing-service-tenant-provisioned-queue-dlq`). The Platform Operator receives a DLQ alert within 5 minutes (FR010). Manual Stripe subscription activation or re-queuing resolves the issue.

---

## 5. Events Produced

| Event Name | Producer | Consumers | SNS Topic |
|---|---|---|---|
| `tenant.provisioned` | tenant-service | notification-service, billing-service | `tenant-provisioned-topic` |
| `tenant.configured` | tenant-service | booking-command-service, availability-service | `tenant-configured-topic` |
| `billing.subscription_activated` | billing-service | tenant-service | `billing-subscription-activated-topic` |
| `notification.sent` | notification-service | (none in MVP) | `notification-sent-topic` |
| `notification.failed` | notification-service | ops-service | `notification-failed-topic` |

---

## 6. Business Rules Enforced

| BR ID | Rule Summary | Enforcement Point |
|---|---|---|
| BR005 | Tenant data must never be readable by another tenant | tenant-service: RLS policy on all tables; `SET LOCAL app.tenant_id` at transaction start; safe default returns zero rows when context is missing |
| BR013 | Every event must carry `tenant_id`, `event_id`, `correlation_id`, `occurred_at` | tenant-service: EventEnvelope validator applied before writing OutboxRecord; outbox-relay: validates envelope before SNS publish |
| BR014 | All write operations on tenant-scoped data must produce an immutable audit log entry | tenant-service: `AuditLog` INSERT in the same transaction as every Tenant state change; INSERT-only privilege for `booking_app_user` on `audit_log` |

---

## 7. Consistency Guarantees

| Concern | Consistency Model | Detail |
|---|---|---|
| Tenant row creation | Strong | Single PostgreSQL transaction — commit or full rollback. No partial state. |
| Stripe Customer creation | Synchronous within transaction | Stripe API called before transaction commits. Failure rolls back the entire registration. |
| Welcome email delivery | Eventually consistent | notification-service is a decoupled async subscriber. Delivery lag < 30 seconds under normal load. |
| Billing subscription activation | Eventually consistent | billing-service is a decoupled async subscriber. Activation lag < 30 seconds under normal load. |
| Tenant status `provisioning` → `active` | Eventually consistent | Depends on `billing.subscription_activated` event processing. Lag < 60 seconds under normal load. |
| TenantConfig cache in booking-command-service | Eventually consistent | Updated on `tenant.configured` event. Lag target < 5 seconds (NFR008). |
| Slot availability cache invalidation | Eventually consistent | availability-service invalidates Redis cache on `tenant.configured` event. Lag target < 5 seconds. |

---

## 8. Idempotency

### Registration request idempotency

The tenant registration endpoint (`POST /api/v1/tenants`) is **not idempotent by design** — a second registration with the same `owner_email` is rejected with `409 Conflict` because the `owner_email` has a unique index on the `tenants` table (for active and provisioning tenants). A client receiving a `503` or network timeout should retry — if the first attempt committed, the retry will receive `409 Conflict` and can treat this as the successful outcome.

### Event idempotency (downstream consumers)

Both notification-service and billing-service use the InboxRecord pattern for deduplication:

- Before processing `tenant.provisioned`, the consumer inserts `(event_id, consumer_name)` into `inbox_records`.
- If the insert fails due to a unique constraint, the event has already been processed — the SQS message is deleted without any side effects.
- This guarantees that even if SQS delivers the same message twice (at-least-once delivery), the welcome email is sent exactly once and the Stripe subscription is created exactly once.

### Configuration events idempotency

`tenant.configured` events include a full `config_snapshot` in the payload. Consumers that receive the same event twice apply the same snapshot twice — the result is identical (idempotent by nature of snapshot delivery). The `occurred_at` field allows consumers to discard out-of-order events by comparing timestamps before applying a snapshot.
