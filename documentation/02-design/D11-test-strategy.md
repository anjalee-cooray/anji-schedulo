---
title: Test Strategy
layer: 02-design
status: current
lastUpdated: 2026-06-28
---

# D11 · Test Strategy

## 1. Objectives

This document defines the testing approach for AnjiSchedulo. Every testing decision is traceable to a specific NFR, BR, or functional requirement.

| Objective | Traceability |
|---|---|
| Prevent double-bookings under concurrent load | BR001 — slot locking; NFR001 — p95 booking latency ≤ 1s |
| Prove cross-tenant isolation at every layer | BR005 — RLS; NFR006 — zero cross-tenant reads |
| Guarantee idempotent event processing | NFR009 — InboxRecord deduplication |
| Verify saga compensation paths | BR008 (cancellation policy), BR009 (refund within 24h) |
| Validate DLQ resolution flows | FR010; operations.json — DLQ operator actions |
| Enforce WCAG 2.1 AA accessibility | NFR013 |
| Meet availability and latency SLOs | NFR002 (99.9% uptime), NFR001 (p95 ≤ 1s) |

---

## 2. Test Pyramid

```
                      ┌──────────────────┐
                      │  E2E / Browser   │  (few, high-value flows)
                      │   Playwright     │
                     ─┤──────────────────├─
                    / │  Integration     │ \
                   /  │  (service layer, │  \
                  /   │  DB, message bus)│   \
                ──────┤──────────────────├────
               /      │  Unit            │    \
              /       │  (domain logic,  │     \
             /        │  validators,     │      \
            /         │  pure functions) │       \
           ───────────┴──────────────────┴────────
```

| Layer | Tool | Coverage Target | Run Frequency |
|---|---|---|---|
| Unit | JUnit 5 + Mockito | 90% branch coverage on domain logic | Every commit (CI) |
| Integration | JUnit 5 + Testcontainers | All service → DB and service → SQS paths | Every PR (CI) |
| Contract | Pact | All producer → consumer event contracts | Every PR (CI) |
| E2E | Playwright | Critical booking and cancellation flows | Every merge to main |
| Performance | k6 | Booking API p95 < 1s at 100 concurrent | Nightly (CI scheduled) |
| Security | OWASP ZAP + Gradle dependency-check | OWASP Top 10 on all public endpoints | Weekly + pre-release |
| Accessibility | axe-core + Playwright | WCAG 2.1 AA on all screens | Every PR (CI) |
| Chaos | AWS Fault Injection Simulator | Outbox relay recovery, DLQ handling | Monthly |

---

## 3. Unit Tests

### Scope

Unit tests cover pure domain logic where the correctness guarantee is most valuable and the test is cheapest to maintain.

| Module | Tests Required |
|---|---|
| `booking-command-service` — slot lock acquisition logic | Lock acquired when slot is free; rejected when already locked; lock TTL expires correctly |
| `booking-command-service` — cancellation eligibility check | Within window: refund eligible; outside window: no refund (BR008) |
| `booking-command-service` — plan tier limit enforcement | Starter ≤ 3 staff; Pro ≤ 20 staff; Enterprise unlimited (BR002) |
| `availability-service` — slot generation | Slots generated within business hours; slots excluded on closed days (BR003); slots excluded when staff is blocked |
| `notification-service` — channel selection logic | Starter: email only; Pro: email + SMS; Enterprise: email + SMS + WhatsApp |
| `tenant-service` — TenantConfig validation | Cancellation hours must be ≥ 0; booking window ≤ 365 days |
| `payment-service` — refund eligibility | Refund if cancelled within policy window; no refund outside window |
| Domain event envelope validator | Events missing `tenant_id`, `event_id`, `correlation_id`, or `occurred_at` are rejected (BR013) |
| Idempotency key generation | UUIDs are deterministically unique per request context |

### Tooling

- **JUnit 5** + **Mockito** — standard Java testing stack
- Tests live in `src/test/java/` within each service module
- No database, no network, no file I/O — pure method inputs and outputs
- Mocking: `Mockito.mock()` / `@MockBean` for external collaborators only. Domain logic has no external calls — no mocks needed there.

---

## 4. Integration Tests

Integration tests verify the interaction between a service and its external dependencies (PostgreSQL, Redis, SQS). They run against real infrastructure spun up via Testcontainers.

### Testcontainers Setup

Each service's integration test suite starts:
- PostgreSQL 15 container (with Flyway migrations applied via `flyway/V*.sql`)
- Redis 7 container
- LocalStack (for SQS FIFO queue simulation)

All containers are started once per test suite (not per test) and share a fresh database schema. RLS policies are applied as part of Flyway migrations.

### Critical Integration Test Cases

#### Tenant Isolation (BR005 — highest priority)

```
Test: tenant A cannot read tenant B's appointments
Setup:
  - Create tenant A and tenant B
  - Create 3 appointments under tenant A
  - Set app.tenant_id = tenant_B_id in the PostgreSQL session
  - Query SELECT * FROM appointments
Assert: zero rows returned (not tenant A's rows)

Test: missing tenant context returns zero rows, not all rows
Setup:
  - Create 3 appointments under tenant A
  - Do NOT set app.tenant_id in the PostgreSQL session
  - Query SELECT * FROM appointments
Assert: zero rows returned (safe default — BR005)
```

This test must run in CI on every PR. Failure is a P1 blocker — no merge permitted.

#### Slot Lock Acquisition Under Concurrency (BR001)

```
Test: two concurrent booking requests for the same slot — exactly one succeeds
Setup:
  - Create a slot S at time T for staff M
  - Issue 2 concurrent POST /bookings requests for slot S from different customer IDs
  - Both requests reach the database at the same time
Assert:
  - Exactly one of the two requests returns 200 OK with status='requested'
  - The other returns 409 Conflict
  - Exactly one SlotLock row exists for slot S
  - Exactly one appointment_events row with event_type='appointment.requested'
```

Uses a database-level advisory lock or `SELECT ... FOR UPDATE SKIP LOCKED` pattern. Verified via two concurrent test threads.

#### Transactional Outbox — No Lost Events (NFR003)

```
Test: outbox record is written in the same transaction as the appointment row
Setup:
  - Begin a booking transaction
  - Write appointment row + outbox_records row
  - Simulate a crash by rolling back (no commit)
Assert:
  - No appointment row in the database
  - No outbox_records row in the database
  - The outbox-relay has not published any message to LocalStack SQS

Test: outbox record published exactly once even under relay restart
Setup:
  - Write appointment + outbox_records row, commit
  - Start outbox-relay, begin publishing
  - Kill and restart outbox-relay mid-publish
  - Restart outbox-relay
Assert:
  - Exactly one message in the target SQS queue
  - outbox_records.published_at is set
```

#### InboxRecord Deduplication (NFR009)

```
Test: consumer processes the same event_id twice — side effect occurs exactly once
Setup:
  - Publish event E with event_id = UUID_1 to notification-service SQS queue
  - notification-service processes E — sends welcome email, inserts inbox_records row
  - Publish the same event E (same event_id UUID_1) again to the queue
  - notification-service processes E again
Assert:
  - inbox_records has exactly ONE row for UUID_1
  - SendGrid API was called exactly ONCE (verified via LocalStack mock or Vitest spy)
  - NotificationRecord has exactly ONE row for UUID_1
```

#### Payment Capture and Saga Completion (UJ002)

```
Test: booking saga completes successfully — appointment transitions from requested to confirmed
Setup:
  - POST /bookings creates appointment in 'requested' state
  - payment-service publishes payment.captured event
  - booking-command-service receives payment.captured
Assert:
  - Appointment status transitions to 'confirmed'
  - appointment_events has: appointment.requested + appointment.confirmed
  - SlotLock is released
  - outbox_records has an appointment.confirmed event pending publish
```

#### Cancellation with Refund (UJ003, BR009)

```
Test: cancellation within policy window triggers refund and slot release
Setup:
  - Confirmed appointment at time T+3h (policy window = 2h)
  - POST /bookings/{id}/cancel at time T
Assert:
  - Appointment status = cancelled
  - payment.refund_initiated event published
  - Slot released (SlotLock deleted)
  - Cancellation within window: amount_eligible_for_refund > 0

Test: cancellation outside policy window — no refund
Setup:
  - Confirmed appointment at time T+1h (policy window = 2h)
  - POST /bookings/{id}/cancel at time T
Assert:
  - Appointment status = cancelled
  - No payment.refund_initiated event (BR008)
  - amount_eligible_for_refund = 0
```

---

## 5. Contract Tests (Pact)

Contract tests verify that event producers emit schemas that consumers expect. This prevents event schema drift from breaking downstream consumers silently.

### Producer–Consumer Pairs

| Producer | Event | Consumer | Pact Test Location |
|---|---|---|---|
| booking-command-service | `appointment.confirmed` | notification-service | `pact/appointment-confirmed.pact.ts` |
| booking-command-service | `appointment.confirmed` | dashboard-service | `pact/appointment-confirmed-dashboard.pact.ts` |
| booking-command-service | `appointment.cancelled` | payment-service | `pact/appointment-cancelled-payment.pact.ts` |
| booking-command-service | `appointment.cancelled` | notification-service | `pact/appointment-cancelled-notification.pact.ts` |
| tenant-service | `tenant.provisioned` | notification-service | `pact/tenant-provisioned.pact.ts` |
| tenant-service | `tenant.configured` | booking-command-service | `pact/tenant-configured.pact.ts` |
| payment-service | `payment.captured` | booking-command-service | `pact/payment-captured.pact.ts` |

### Contract Verification Process

1. Consumer team writes a Pact consumer test (using `pact-jvm`) that defines the expected event structure.
2. The generated Pact file is published to Pact Broker (hosted in CI).
3. Producer team runs `./gradlew pactVerify` which reads all Pact files from the Broker and verifies that the producer's actual events satisfy every consumer's expectations.
4. CI blocks merge if any Pact verification fails.

### Required Fields in All Events (BR013)

Every consumer Pact test must assert the presence of:
- `tenant_id` (string UUID format)
- `event_id` (string UUID format)
- `correlation_id` (string UUID format)
- `occurred_at` (string ISO 8601 datetime)

---

## 6. End-to-End Tests

E2E tests run in a staging environment against real AWS services (RDS, SQS, SNS, SendGrid sandbox). Playwright drives the Next.js frontend.

### E2E Test Suite — Critical Paths

#### UJ001 — Tenant Onboarding

```
Test: complete onboarding from registration to first booking
Steps:
  1. Navigate to /register
  2. Fill in business name, email, password — select Starter plan
  3. Submit form — verify "Setting up your account" state
  4. Wait for redirect to /dashboard/onboarding (poll for tenant status = active)
  5. Configure business hours (Mon–Fri, 09:00–18:00)
  6. Add a service: "Consultation", 30 minutes, £50
  7. Add a staff member: "Alice Smith", alice@example.com
  8. Navigate to /{tenant-slug}/book — verify booking page loads
  9. Verify welcome email received (SendGrid event API, sandbox)
```

#### UJ002 — Appointment Booking

```
Test: customer books and pays for an appointment
Steps:
  1. Navigate to /{tenant-slug}/book
  2. Select "Consultation" service
  3. Select "Any available" staff
  4. Select a date 3 days from today
  5. Select the first available slot
  6. Fill customer details (test name, test email)
  7. Enter Stripe test card (4242424242424242, any future expiry, any CVC)
  8. Submit — verify booking confirmation screen appears
  9. Verify appointment appears in Tenant Admin dashboard within 5 seconds
  10. Verify confirmation email received (SendGrid event API)
Assert: appointment status = confirmed in database
```

#### UJ003 — Cancellation

```
Test: customer cancels appointment within policy window — receives refund notice
Steps:
  1. Pre-condition: confirmed appointment at T+4h (policy window = 2h)
  2. Navigate to cancel link from confirmation email
  3. Verify cancellation policy and refund amount displayed
  4. Click "Cancel my appointment"
  5. Verify cancellation confirmation page
  6. Verify appointment status = cancelled in database
  7. Verify refund initiated (Stripe dashboard sandbox)
  8. Verify cancellation email received
```

#### UJ006 — DLQ Triage

```
Test: operator re-queues a failed notification DLQ event
Steps:
  1. Trigger a notification failure (configure SendGrid with invalid API key in test env)
  2. Complete a booking — notification-service fails 4 times → DLQ
  3. Navigate to /ops/dlq as P4 (Platform Operator)
  4. Verify DLQ entry appears with correct queue name and failure reason
  5. Fix SendGrid API key (update to valid sandbox key)
  6. Click "Re-queue" on the DLQ entry
  7. Verify notification is delivered (SendGrid event API)
  8. Verify DLQ entry is cleared
```

---

## 7. Performance Tests (k6)

Target: booking API p95 < 1s at 100 concurrent users (NFR001).

### Load Test — Booking Flow

```javascript
// k6/booking-load-test.js
export let options = {
  vus: 100,
  duration: '5m',
  thresholds: {
    http_req_duration: ['p(95)<1000'],  // p95 < 1s
    http_req_failed: ['rate<0.01'],     // error rate < 1%
  },
};

export default function () {
  const res = http.post(`${BASE_URL}/api/v1/bookings`, JSON.stringify({
    tenant_id: PERF_TENANT_ID,
    service_id: PERF_SERVICE_ID,
    staff_id: PERF_STAFF_ID,
    slot_date: TOMORROW,
    slot_start: nextSlot(),
    customer_name: 'Load Test User',
    customer_email: `load+${Date.now()}@example.com`,
  }), { headers: { 'Content-Type': 'application/json', 'Authorization': `Bearer ${JWT}` } });

  check(res, {
    'status is 202': (r) => r.status === 202,
    'response time < 1s': (r) => r.timings.duration < 1000,
  });
}
```

**Slot deduplication:** Performance tests use a dedicated `perf-tenant` with 200 pre-generated slots. Each VU picks a different slot to avoid slot contention (which would generate 409s, not a load error). A separate concurrency test deliberately targets the same slot from 10 VUs to verify the 409 conflict path.

### Load Test — Availability Endpoint

```
VUs: 200 (availability is read-heavy)
Duration: 10m
Target: p95 < 200ms (cached response from Redis)
Threshold: cache hit rate > 95% measured via custom k6 metric from response headers
```

### Nightly Schedule

Performance tests run as a nightly GitHub Actions workflow against a dedicated staging environment with production-equivalent data volumes. Results published to Grafana dashboard "Performance — Nightly Runs". Failures trigger a Slack #incidents alert.

---

## 8. Security Tests

### OWASP ZAP (Automated Scanning)

Runs weekly against staging and pre-release against production. Scans all public endpoints:
- `/{tenant-slug}/book` — booking page XSS, CSRF
- `/api/v1/tenants` — registration endpoint injection
- `/api/v1/bookings` — booking API injection
- `/ops/dlq` — ops console (authenticated, P4 role required)

ZAP passive scan runs in CI as part of every PR pipeline. ZAP active scan runs in the weekly scheduled job only (to avoid side effects against shared environments).

### JWT Validation Tests

```
Test: request with no Authorization header is rejected
Test: request with expired JWT is rejected (401)
Test: request with JWT signed by wrong key is rejected (401)
Test: request with JWT missing tenant_id claim is rejected (403)
Test: request with JWT tenant_id != target resource tenant_id is rejected (403)
Test: P2 (staff) cannot call admin-only endpoints — rejected (403)
Test: P4 (operator) can call ops endpoints — accepted (200)
```

### Cross-Tenant Isolation Security Tests

These tests are tagged `@security` and run in CI on every PR:

```
Test: authenticated as tenant A, request resource owned by tenant B
  GET /api/v1/bookings?tenant_id=TENANT_B_ID with JWT for tenant A
  Assert: 403 Forbidden or empty result set — never tenant B's data
```

### Dependency Audit

`./gradlew dependencyCheckAnalyze` (OWASP Dependency-Check) runs on every PR. High and critical severity vulnerabilities block merge. Moderate vulnerabilities are tracked and resolved within 30 days.

---

## 9. Accessibility Tests

Axe-core integration runs via Playwright on every PR for all screens that have E2E tests. A separate accessibility-only Playwright suite covers all public-facing screens.

### Automated Checks (axe-core)

Checks run after page load and after each user interaction that changes the DOM:
- Colour contrast (AA 4.5:1 minimum)
- Missing alt text on images
- Missing labels on form inputs
- Invalid ARIA roles
- Tab order anomalies

All axe violations block CI. Warnings are logged but do not block.

### Manual Checks (pre-release)

Before each major release, a manual accessibility audit covers:
- Keyboard-only navigation through the complete booking flow (UJ002)
- Screen reader walkthrough (VoiceOver on macOS, NVDA on Windows) through the dashboard (UJ005)
- Mobile touch target sizing verification on a physical iOS and Android device
- Focus management after modal open/close on all modals in the booking and cancellation flows

---

## 10. Chaos Engineering

Monthly chaos experiments via AWS Fault Injection Simulator against a dedicated chaos environment (isolated from staging).

| Experiment | Target | Expected Behaviour |
|---|---|---|
| Kill outbox-relay | outbox-relay ECS task | OutboxRecords remain unpublished; no events lost; relay recovers on restart and resumes publishing from `published_at IS NULL` |
| Kill notification-service | notification-service ECS task | Bookings continue confirming; SQS messages queue up; on service restart, messages processed in order; notifications delivered late but not lost |
| Kill booking-command-service | booking-command-service ECS task | Booking requests return 503; slot locks TTL and expire; on restart, in-flight sagas resume from outbox |
| Inject RDS latency (500ms) | RDS read traffic | Booking API p95 degrades; circuit breaker opens on availability-service reads; cached data served from Redis |
| SQS queue unavailable (60s) | `appointment-confirmed` SNS→SQS | OutboxRecords backlog; alerts fire (ALT004); on queue recovery, messages delivered in order |

All experiments have a defined hypothesis, inject duration, and rollback procedure. Results are written to `ops-service/docs/chaos-reports/`.

---

## 11. Test Data Management

### Isolated Tenant Per Test Suite

Each integration test suite creates a fresh tenant with a unique `tenant_id` and tears it down after the suite completes. This prevents test pollution across suites running in parallel.

### Seed Data

Test fixtures are defined in `packages/test-fixtures/` and include:
- `createTestTenant(plan: 'starter' | 'pro' | 'enterprise')` — creates tenant + config + 3 services + 3 staff
- `createTestAppointment(tenantId, status)` — creates a confirmed appointment at a future slot
- `createTestSlotLock(tenantId, slotDate, slotStart, staffId)` — creates a soft slot lock

### PII in Tests

Test data uses synthetic names and email addresses only (`{uuid}@example.com`). No production customer data is used in any test environment. SendGrid sandbox mode ensures test emails are not delivered to real recipients.

---

## 12. CI/CD Integration

### GitHub Actions Pipeline (per PR)

```yaml
steps:
  - unit-tests          # JUnit 5 + Mockito — all services in parallel
  - integration-tests   # JUnit 5 + Testcontainers — all services in parallel
  - contract-verify     # Pact (pact-jvm) — verify all producer contracts
  - e2e-tests           # Playwright — critical paths on staging
  - accessibility       # axe-core via Playwright
  - security-audit      # Gradle dependency-check + ZAP passive scan
  - build-check         # Gradle bootJar — compile and package all services
```

All steps must pass before merge to main. Unit, integration, contract, and accessibility tests run in parallel. E2E tests run after deployment to staging (sequential dependency).

### Nightly

```yaml
steps:
  - performance-tests   # k6 — booking and availability load tests
  - security-scan       # ZAP active scan
  - chaos-smoke         # Lightweight outbox relay restart test
```

### Pre-Release (Manual Trigger)

```yaml
steps:
  - full-e2e-suite      # All UJ001–UJ007 E2E flows
  - manual-accessibility-audit (checklist — human verification)
  - performance-baseline # k6 with production data volume snapshot
  - security-zap-active  # Full OWASP ZAP active scan
```
