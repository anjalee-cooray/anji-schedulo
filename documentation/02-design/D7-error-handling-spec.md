---
title: Error Handling Specification
layer: 02-design
status: current
lastUpdated: 2026-06-28
sources:
  - specs/user/functional-requirements.json
  - specs/user/business-rules.json
  - specs/ai/architecture.json
  - specs/ai/events.json
  - specs/ai/operations.json
---

# D7 · Error Handling Specification

## 1. Overview

This document defines how AnjiSchedulo handles errors at every layer of the system: HTTP API responses, saga compensation flows, event consumer failures, DLQ escalation, and downstream provider errors. Every error the system can emit is catalogued with a machine-readable code, human-readable message, HTTP status, and the action the caller or operator should take.

**Design Principles:**

1. **Never leak internal state.** Error responses to external clients never include stack traces, internal service names, or database error details.
2. **Machine-readable codes.** Every error has a unique `code` field so clients can handle errors programmatically.
3. **Idempotency safety.** Errors from idempotent endpoints return the same response on retry for the same idempotency key.
4. **Saga atomicity.** Booking sagas compensate fully on failure — no partial state is visible to the customer.
5. **Notification isolation.** Notification failures never propagate to the booking response (BR007).

---

## 2. HTTP Error Response Format

All API errors return a consistent JSON envelope:

```json
{
  "error": {
    "code": "SLOT_UNAVAILABLE",
    "message": "The requested appointment slot is no longer available.",
    "details": {},
    "correlation_id": "550e8400-e29b-41d4-a716-446655440000",
    "occurred_at": "2026-06-28T09:00:00Z"
  }
}
```

| Field | Type | Description |
|---|---|---|
| `code` | string | Machine-readable error code (see catalogue below) |
| `message` | string | Human-readable description safe for display to end users |
| `details` | object | Optional structured context (e.g. field-level validation errors) |
| `correlation_id` | UUID | Trace ID for log correlation. Always present in authenticated requests. |
| `occurred_at` | ISO 8601 | Server timestamp of the error |

**Validation error `details` format:**

```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "One or more fields failed validation.",
    "details": {
      "fields": [
        { "field": "owner_email", "message": "Must be a valid email address." },
        { "field": "plan", "message": "Must be one of: starter, pro, enterprise." }
      ]
    }
  }
}
```

---

## 3. HTTP Status Code Usage

| Status | Meaning | When Used |
|---|---|---|
| 200 OK | Success | GET requests, cancellation, rescheduling |
| 201 Created | Resource created | POST that creates a booking |
| 202 Accepted | Accepted for async processing | Tenant registration, data export |
| 400 Bad Request | Malformed request syntax | Unparseable JSON body, invalid Content-Type |
| 401 Unauthorized | Authentication missing or invalid | Missing/expired/malformed JWT |
| 403 Forbidden | Authenticated but insufficient permissions | Wrong role, cross-tenant access attempt, suspended tenant |
| 404 Not Found | Resource does not exist | Tenant, appointment, service, or staff not found |
| 409 Conflict | State conflict | Duplicate registration, slot unavailable, appointment in wrong state |
| 410 Gone | Resource permanently deleted | Accessing a deprovisioned tenant |
| 422 Unprocessable Entity | Validation or business rule failure | Field validation, booking limit, cancellation window, plan limit |
| 402 Payment Required | Payment failed | Booking blocked by payment failure |
| 429 Too Many Requests | Rate limit exceeded | > 60 requests/minute per IP on public endpoints |
| 500 Internal Server Error | Unexpected server error | Unhandled exceptions — never in normal operation |
| 503 Service Unavailable | Downstream dependency unavailable | Stripe unavailable, circuit breaker open |

---

## 4. Error Code Catalogue

### 4.1 Authentication and Authorisation Errors

| Code | HTTP | Description | Caller Action |
|---|---|---|---|
| `AUTHENTICATION_REQUIRED` | 401 | No JWT provided or JWT is missing the `Authorization` header | Include a valid Bearer token |
| `TOKEN_EXPIRED` | 401 | JWT has passed its expiry time (`exp` claim) | Refresh the access token and retry |
| `TOKEN_INVALID` | 401 | JWT signature verification failed | Re-authenticate and obtain a new token |
| `TOKEN_MISSING_CLAIM` | 401 | Required JWT claim (`tid`, `sub`, `role`) is absent | Re-authenticate — token is malformed |
| `FORBIDDEN` | 403 | Authenticated user does not have the required role for this action | Check role assignment; contact tenant admin |
| `TENANT_SUSPENDED` | 403 | The tenant account is suspended; write operations are blocked | Read-only access is still permitted; contact support |
| `CROSS_TENANT_ACCESS` | 403 | Requested resource belongs to a different tenant | Verify the tenant_id in the JWT matches the resource |

### 4.2 Resource Not Found Errors

| Code | HTTP | Description | Caller Action |
|---|---|---|---|
| `TENANT_NOT_FOUND` | 404 | No tenant exists for the given slug or ID | Verify the tenant identifier |
| `APPOINTMENT_NOT_FOUND` | 404 | Appointment ID does not exist in this tenant | Verify the appointment ID |
| `SERVICE_NOT_FOUND` | 404 | Service ID does not exist or has been deactivated | Refresh the service catalogue |
| `STAFF_NOT_FOUND` | 404 | Staff member does not exist or has been deactivated | Refresh the staff list |
| `EXPORT_NOT_FOUND` | 404 | Export job ID not found | Check the export ID returned at creation time |
| `REPLAY_JOB_NOT_FOUND` | 404 | Replay job ID not found | Check the job ID |

### 4.3 Conflict Errors

| Code | HTTP | Description | Caller Action |
|---|---|---|---|
| `DUPLICATE_REGISTRATION` | 409 | An active or provisioning tenant already exists for this email | Log in with the existing account or use a different email |
| `SLOT_UNAVAILABLE` | 409 | The requested slot is already booked or no longer available | Refresh the slot list and choose a different time |
| `APPOINTMENT_NOT_CANCELLABLE` | 409 | Appointment is not in a cancellable state (e.g. already cancelled or completed) | No action needed; the appointment is already in a terminal state |
| `APPOINTMENT_NOT_RESCHEDULABLE` | 409 | Appointment is not in a reschedulable state | Check the current appointment status |

### 4.4 Validation and Business Rule Errors

| Code | HTTP | Description | Caller Action |
|---|---|---|---|
| `VALIDATION_ERROR` | 422 | One or more request fields failed format or type validation | Inspect `details.fields` for field-level messages and correct the request |
| `CANCELLATION_WINDOW_EXCEEDED` | 422 | Cancellation request is outside the tenant's configured cancellation window | The cancellation cannot be processed; contact the tenant admin |
| `BOOKING_LIMIT_EXCEEDED` | 422 | Customer has reached the maximum number of advance bookings | Cancel an existing booking before making a new one |
| `BOOKING_WINDOW_EXCEEDED` | 422 | Requested slot is beyond the tenant's configured booking advance window | Select a slot within the permitted booking window |
| `SLOT_OUTSIDE_WORKING_HOURS` | 422 | Requested slot falls outside the staff member's configured working hours | Select a slot within business hours |
| `PLAN_LIMIT_EXCEEDED` | 422 | Creating this resource would exceed the current plan's limits | Upgrade the plan or remove unused resources |
| `SERVICE_DURATION_TOO_SHORT` | 422 | Service duration is below the 15-minute minimum | Set a duration of at least 15 minutes |
| `DEPROVISIONING_NOT_PERMITTED` | 422 | Tenant cannot be deprovisioned yet (30-day suspension period not elapsed) | Retry after the indicated `days_remaining` |
| `IDEMPOTENCY_KEY_CONFLICT` | 409 | The idempotency key was used for a different payload | Generate a new idempotency key for this request |

### 4.5 Payment Errors

| Code | HTTP | Description | Caller Action |
|---|---|---|---|
| `PAYMENT_FAILED` | 402 | The payment charge was declined by the payment provider | The slot is not booked. Retry with a different payment method. |
| `PAYMENT_SERVICE_UNAVAILABLE` | 503 | The payment provider is unreachable (circuit breaker open) | Retry after the `Retry-After` interval indicated in the response header |
| `REFUND_FAILED` | — | Internal only — refund failure goes to DLQ and does not surface to the customer | Platform Operator resolves via DLQ triage (FR010) |

### 4.6 Server and Infrastructure Errors

| Code | HTTP | Description | Caller Action |
|---|---|---|---|
| `SERVICE_UNAVAILABLE` | 503 | An upstream service is temporarily unavailable | Retry with exponential backoff; check `Retry-After` header |
| `RATE_LIMIT_EXCEEDED` | 429 | Too many requests from this IP or client | Reduce request rate; observe `Retry-After` header |
| `INTERNAL_ERROR` | 500 | Unexpected internal error (circuit breaker, unhandled exception) | Log the `correlation_id` and contact support; do not retry immediately |

---

## 5. Saga Failure and Compensation

The booking saga in booking-command-service orchestrates three steps. When any step fails, all preceding steps are compensated.

### 5.1 Booking Saga Steps and Compensation

```
Step 1: Reserve slot (insert into slot_locks)
Step 2: Capture payment (payment-service)
Step 3: Persist AppointmentEvent(confirmed) + publish appointment.confirmed
```

**Failure at Step 1 — Slot already taken:**

- Cause: Unique constraint violation on `slot_locks`.
- Compensation: None required (no writes succeeded).
- Response to client: `409 SLOT_UNAVAILABLE`.

**Failure at Step 2 — Payment declined:**

- Cause: Stripe returns a card decline or insufficient funds error.
- Compensation: DELETE from `slot_locks` to release the reserved slot.
- Response to client: `402 PAYMENT_FAILED`.

**Failure at Step 2 — Payment service unavailable (circuit breaker open):**

- Cause: payment-service ECS task is unreachable; circuit breaker has tripped after 5 consecutive failures.
- Compensation: DELETE from `slot_locks` to release the reserved slot.
- Response to client: `503 PAYMENT_SERVICE_UNAVAILABLE` with `Retry-After: 30`.

**Failure at Step 3 — AppointmentEvent write fails:**

- Cause: Unexpected PostgreSQL error during event persistence.
- Compensation: DELETE from `slot_locks`; issue a Stripe refund if payment was already captured.
- Response to client: `500 INTERNAL_ERROR`.

**Saga timeout (> 10 seconds):**

- Cause: Any step takes longer than the saga timeout budget.
- Compensation: Any slot lock acquired is released; any payment captured is refunded.
- Response to client: `503 SERVICE_UNAVAILABLE`.
- Alert: `ALT006 — Booking Saga Timeout` fires; P2 severity.

### 5.2 Rescheduling Saga Compensation

```
Step 1: Lock new slot
Step 2: Release old slot (atomic swap in same transaction)
```

**Failure at Step 1 — New slot unavailable:**

- Compensation: None (no writes committed).
- Response: `409 SLOT_UNAVAILABLE`. Original booking is unchanged.

### 5.3 Cancellation Compensation

Cancellation is a single atomic transaction (release slot lock + write AppointmentEvent). If the transaction fails:

- Compensation: None (transaction rollback).
- Response: `500 INTERNAL_ERROR`.
- Refund is initiated asynchronously via `appointment.cancelled` event — refund failure is isolated to the notification pipeline (BR009).

---

## 6. Event Consumer Error Handling

### 6.1 SQS Dead Letter Queue Pattern

All SQS FIFO consumer queues have a paired DLQ with `maxReceiveCount = 4`. The SQS `visibilityTimeout` is set to the maximum expected processing time per consumer:

| Consumer | visibilityTimeout | DLQ Name |
|---|---|---|
| notification-service (appointment.confirmed) | 30s | notification-service-appointment-confirmed-queue-dlq |
| notification-service (appointment.cancelled) | 30s | notification-service-appointment-cancelled-queue-dlq |
| notification-service (appointment.rescheduled) | 30s | notification-service-appointment-rescheduled-queue-dlq |
| notification-service (tenant.provisioned) | 60s | notification-service-tenant-provisioned-queue-dlq |
| payment-service (appointment.cancelled) | 60s | payment-service-appointment-cancelled-queue-dlq |
| booking-command-service (payment.captured) | 30s | booking-command-service-payment-captured-queue-dlq |
| booking-command-service (payment.failed) | 30s | booking-command-service-payment-failed-queue-dlq |
| booking-command-service (tenant.configured) | 30s | booking-command-service-tenant-configured-queue-dlq |
| booking-command-service (tenant.suspended) | 30s | booking-command-service-tenant-suspended-queue-dlq |
| dashboard-service (appointment.confirmed) | 30s | dashboard-service-appointment-confirmed-queue-dlq |
| dashboard-service (appointment.cancelled) | 30s | dashboard-service-appointment-cancelled-queue-dlq |
| dashboard-service (appointment.rescheduled) | 30s | dashboard-service-appointment-rescheduled-queue-dlq |
| analytics-service (appointment.confirmed) | 60s | analytics-service-appointment-confirmed-queue-dlq |
| availability-service (tenant.configured) | 30s | availability-service-tenant-configured-queue-dlq |
| availability-service (appointment.confirmed) | 30s | availability-service-appointment-confirmed-queue-dlq |
| billing-service (tenant.provisioned) | 60s | billing-service-tenant-provisioned-queue-dlq |
| ops-service (notification.failed) | 30s | ops-service-notification-failed-queue-dlq |

### 6.2 Consumer Retry Logic

Each consumer implements the following retry strategy before a message reaches the DLQ:

1. **Attempt 1** — Immediate (SQS delivers the message).
2. **Attempt 2** — After `visibilityTimeout` expiry (SQS re-delivers).
3. **Attempt 3** — After second `visibilityTimeout` expiry.
4. **Attempt 4** — After third `visibilityTimeout` expiry.
5. **DLQ** — After 4 failures, SQS routes to the DLQ automatically. `notification.failed` or equivalent event is published.

### 6.3 Envelope Validation Errors

If a consumed message is missing any required envelope field (`tenant_id`, `event_id`, `correlation_id`, `occurred_at`), the consumer immediately routes the message to its DLQ without retry — the message is structurally invalid and retrying will not fix it. The consumer logs the failure with the raw payload for operator investigation.

### 6.4 InboxRecord Deduplication

Before processing any event, the consumer attempts:

```sql
INSERT INTO inbox_records (event_id, consumer_name, processed_at)
VALUES ($1, $2, NOW())
ON CONFLICT (event_id, consumer_name) DO NOTHING
RETURNING event_id;
```

If the INSERT returns 0 rows (duplicate detected), the consumer deletes the SQS message and terminates without any side effects. This makes all consumers idempotent under at-least-once delivery (NFR009).

---

## 7. Outbox-Relay Error Handling

The outbox-relay publishes events from `outbox_records` to SNS. If the SNS publish call fails:

1. The `published_at` column remains NULL.
2. The relay retries on the next polling cycle (every 500ms).
3. An alert fires if the outbox backlog exceeds 100 unpublished records for more than 10 seconds (`ALT004 — Outbox Relay Lag`).

The relay never marks a record as published until SNS confirms receipt. This guarantees at-least-once delivery — the cost is possible duplicate delivery to consumers, which is handled by InboxRecord deduplication.

---

## 8. Circuit Breaker Configuration

Circuit breakers protect booking-command-service from downstream failures. Configuration:

| Dependency | Failure Threshold | Recovery Timeout | Half-Open Probes |
|---|---|---|---|
| payment-service | 5 consecutive failures | 30 seconds | 1 probe request |
| notification-service | Not applicable (async via SQS) | — | — |
| availability-service | 3 consecutive failures | 15 seconds | 1 probe request |

**Open state behaviour:**

- payment-service open: Booking requests return `503 PAYMENT_SERVICE_UNAVAILABLE` with `Retry-After: 30`. Slot locks are not acquired.
- availability-service open: Slot browsing requests return `503 SERVICE_UNAVAILABLE`. Booking requests return `503 SERVICE_UNAVAILABLE`.

**Monitoring:** Circuit breaker state changes emit logs and are tracked via Grafana dashboard `AnjiSchedulo — Circuit Breaker Status`. `ALT007 — Payment Circuit Breaker Open` fires when the payment-service breaker opens (P2 severity).

---

## 9. PostgreSQL Error Handling

All service database interactions use connection pooling via PgBouncer. Application-layer PostgreSQL error handling:

| PostgreSQL Error Code | Situation | Application Response |
|---|---|---|
| `23505` (unique_violation) | Slot lock conflict (concurrent booking) | `409 SLOT_UNAVAILABLE` |
| `23505` on `inbox_records` | Duplicate event delivery | Silent skip — delete SQS message, no error |
| `23505` on `idempotency_keys` | Duplicate booking request | Return stored result — `200 OK` with original response |
| `28000` (invalid_authorization) | RLS rejection — wrong tenant context | `403 FORBIDDEN` — alert triggered (possible isolation breach) |
| `57014` (query_canceled) | Query timeout | `503 SERVICE_UNAVAILABLE` |
| `08*` connection errors | Database connection lost | Retry 3 times with exponential backoff; `503` if all fail |

**RLS rejection (28000) is a P1 alert** — it indicates a possible tenant isolation violation. The Platform Operator is paged immediately (ALT002).

---

## 10. Notification Delivery Errors

Notification-service error handling is fully isolated from the booking pipeline (BR007):

| Failure Scenario | Behaviour |
|---|---|
| SendGrid API returns 4xx | Log error; mark `NotificationRecord.status = failed`; increment `attempt_count`; retry with backoff |
| SendGrid API returns 5xx | Log error; retry with exponential backoff |
| Twilio SMS failure | Log error; retry with exponential backoff; fallback to email if SMS fails |
| All retries exhausted (4 attempts) | Set `status = dlq`; publish `notification.failed` to ops-service queue |
| `notification.failed` consumed by ops-service | Create ops alert; Platform Operator can trigger manual resend |

Notification failures have **no effect** on:
- The booking confirmation status
- The slot availability for other customers
- The tenant admin dashboard

---

## 11. Rate Limiting

Public endpoints (unauthenticated slot browsing) are rate limited per source IP:

| Endpoint Group | Limit | Window | Error Response |
|---|---|---|---|
| `GET /tenants/{slug}/availability` | 60 requests | 60 seconds | 429 with `Retry-After` header |
| `POST /tenants/{slug}/bookings` | 10 requests | 60 seconds | 429 with `Retry-After` header |
| `POST /api/v1/tenants` (registration) | 5 requests | 60 seconds per IP | 429 with `Retry-After` header |
| Authenticated admin endpoints | 300 requests | 60 seconds per tenant | 429 with `Retry-After` header |

Rate limit responses include:

```http
HTTP/1.1 429 Too Many Requests
Retry-After: 30
X-RateLimit-Limit: 60
X-RateLimit-Remaining: 0
X-RateLimit-Reset: 1751100060

{ "error": { "code": "RATE_LIMIT_EXCEEDED", "message": "Too many requests. Please retry after 30 seconds.", ... } }
```

---

## 12. Security Error Handling

Security errors must never reveal internal information:

- **Cross-tenant access attempt:** The service returns `404 NOT_FOUND` for cross-tenant resource requests (not 403) to avoid revealing whether the resource exists in a different tenant. The access attempt is logged and monitored for anomaly detection.
- **JWT tampering:** A JWT with an invalid signature returns `401 TOKEN_INVALID`. No detail about which claim failed is returned.
- **Missing `tid` claim:** If a JWT lacks the `tenant_id` claim, the request is rejected with `401 TOKEN_MISSING_CLAIM`. The PostgreSQL RLS safe default (zero rows) ensures no data is accessible even if the application layer fails to check.

---

## 13. Error Monitoring and Alerting

All errors are logged to Loki with structured fields. Key Grafana alert rules for error conditions:

| Alert ID | Condition | Severity |
|---|---|---|
| ALT001 | `error_rate_5xx > 1%` over 5 minutes | P1 |
| ALT002 | Any `28000` (RLS rejection) event | P1 |
| ALT003 | Any DLQ depth > 0 | P2 |
| ALT004 | Outbox backlog > 100 for > 10 seconds | P2 |
| ALT005 | Booking saga success rate < 95% over 5 minutes | P1 |
| ALT006 | Booking saga p95 latency > 10s | P2 |
| ALT007 | Payment circuit breaker opens | P2 |
| ALT008 | Availability p95 > 500ms | P3 |

Operators receive P1 alerts via PagerDuty and Slack `#incidents`. P2 and P3 alerts go to Slack `#incidents` only.
