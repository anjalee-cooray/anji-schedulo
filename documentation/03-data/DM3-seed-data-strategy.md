---
title: Seed Data Strategy
layer: 03-data
status: current
lastUpdated: 2026-06-28
---

# DM3 · Seed Data Strategy

## Overview

This document defines the seed data strategy for AnjiSchedulo across three environments:

| Environment | Purpose | Seed Type |
|---|---|---|
| **Local development** | Developer onboarding and feature work | Full fixture set: one demo tenant with all entities populated |
| **CI / integration tests** | Deterministic test execution | Minimal, test-scoped fixture sets per test suite |
| **Staging** | End-to-end acceptance testing | Serenity Salon demo tenant (shared, reset nightly) |

Seed data never enters the production database. Production data is always real — it is created by users through the application.

---

## 1. Seed Data Principles

1. **Never seed PII into CI.** CI fixtures use synthetic, clearly fake names and emails that cannot be confused with real users (`test-customer-001@example.invalid`).
2. **Seed data is version-controlled alongside migrations.** Every change to seed fixtures must be committed to the repository and reviewed in the same pull request as any related migration.
3. **Seeds are idempotent.** Running the seed script twice must produce identical database state. Use `INSERT ... ON CONFLICT DO NOTHING` or `ON CONFLICT DO UPDATE` for all seed inserts.
4. **Seeds respect RLS.** Seed scripts run as `booking_app_user` with `SET LOCAL app.tenant_id` to simulate the runtime context and verify RLS policies are not bypassed by seed data.
5. **Every seeded entity satisfies all business rules.** Seeds are not a bypass — they must produce state that would be valid if produced through the application.

---

## 2. Flyway Migration Seed Pattern

AnjiSchedulo uses Flyway for database migrations. Seed data is loaded via repeatable migrations:

```
db/
  migrations/
    V001__create_tenants.sql
    V002__create_services.sql
    ...
    R__seed_dev.sql         ← repeatable (R prefix), runs in local/staging only
    R__seed_ci.sql          ← repeatable, runs in CI only
```

Repeatable migrations (R prefix) re-run whenever their checksum changes. They are excluded from production deployment by the Flyway configuration:

```properties
# flyway.conf (production)
flyway.locations=classpath:db/migrations
flyway.skipDefaultCallbacks=false
# No seed files included in production Flyway locations
```

```properties
# flyway.conf (local / staging)
flyway.locations=classpath:db/migrations,classpath:db/seeds
```

---

## 3. Local Development Seed

The local development seed creates a fully operational demo tenant ("Serenity Salon") in the developer's local database. After running `npm run db:seed`, the developer can immediately:

- Log in as Tenant Admin and view the populated dashboard
- Browse the public booking page and make a test booking
- View staff availability and working hours
- Inspect appointment events in the event store

### 3a. Demo Tenant: Serenity Salon

```sql
-- R__seed_dev.sql (excerpt)
-- Tenant
INSERT INTO tenants (
  tenant_id, name, owner_email, owner_name,
  plan, status, stripe_customer_id, created_at
) VALUES (
  '00000000-0000-0000-0000-000000000001',
  'Serenity Salon',
  'sarah@serenity-salon.dev',
  'Sarah Bloom',
  'pro',
  'active',
  'cus_dev_serenity_salon_001',
  NOW() - INTERVAL '30 days'
) ON CONFLICT (tenant_id) DO NOTHING;

-- Tenant Config
INSERT INTO tenant_configs (
  tenant_id, cancellation_hours, booking_window_days,
  max_advance_bookings, timezone, currency, updated_at
) VALUES (
  '00000000-0000-0000-0000-000000000001',
  24, 60, 5, 'Europe/London', 'GBP', NOW()
) ON CONFLICT (tenant_id) DO NOTHING;
```

### 3b. Staff Members

| staff_id | name | email | active |
|---|---|---|---|
| `...0000-0101` | Alex Chen | `alex@serenity-salon.dev` | true |
| `...0000-0102` | Jordan Reeves | `jordan@serenity-salon.dev` | true |
| `...0000-0103` | Sam Torres | `sam@serenity-salon.dev` | true |

All staff members have working hours set Monday–Friday 09:00–18:00.

### 3c. Services

| service_id | name | duration_minutes | price_amount | price_currency |
|---|---|---|---|---|
| `...0000-0201` | Haircut | 45 | 4500 | GBP |
| `...0000-0202` | Colour Treatment | 120 | 9500 | GBP |
| `...0000-0203` | Blow-Dry | 30 | 2500 | GBP |
| `...0000-0204` | Consultation | 20 | 0 | GBP |

> Prices in pence (e.g. 4500 = £45.00).

### 3d. Customers

| customer_id | name | email | phone |
|---|---|---|---|
| `...0000-0301` | Alice Green | `alice@example.dev` | `+447700900001` |
| `...0000-0302` | Ben Harper | `ben@example.dev` | NULL |
| `...0000-0303` | Clara Kim | `clara@example.dev` | `+447700900003` |

### 3e. Appointments

The seed creates appointments in three states to populate the event store and dashboard view:

**Confirmed appointments (today and next 7 days):**

```
Alice Green  — Haircut         — Alex Chen    — Today 10:00
Ben Harper   — Colour Treatment— Jordan Reeves — Today 11:00
Clara Kim    — Blow-Dry        — Sam Torres   — Today 14:30
Alice Green  — Consultation    — Alex Chen    — Tomorrow 09:00
Ben Harper   — Haircut         — Sam Torres   — Tomorrow 11:00
```

**Cancelled appointment (yesterday):**

```
Clara Kim    — Haircut  — Alex Chen — Yesterday 10:00
  ↑ cancelled 25 hours before start (satisfies BR002)
  ↑ refund issued (payment_records seeded accordingly)
```

**Completed appointments (last 30 days):**

30 completed appointments distributed across all staff and services, distributed across the past month, to populate `tenant_dashboard_views` with realistic data.

### 3f. Appointment Events

Each appointment in the seed has the corresponding `appointment_events` records. For example, the confirmed Alice Green haircut:

```sql
-- appointment_events for confirmed appointment
INSERT INTO appointment_events VALUES (
  gen_random_uuid(), '{appointment_id}', '{tenant_id}',
  'appointment.requested', NOW() - INTERVAL '2 hours',
  '{correlation_id}', NULL,
  '{customer_id}', 'customer', '{"snapshot": ...}'::jsonb
), (
  gen_random_uuid(), '{appointment_id}', '{tenant_id}',
  'appointment.confirmed', NOW() - INTERVAL '2 hours' + INTERVAL '500ms',
  '{correlation_id}', '{request_event_id}',
  '{system_uuid}', 'system', '{"snapshot": ...}'::jsonb
) ON CONFLICT DO NOTHING;
```

### 3g. Dashboard View

The seed populates `tenant_dashboard_views` for the past 30 days:

```sql
INSERT INTO tenant_dashboard_views (
  tenant_id, view_date, confirmed_count, cancelled_count,
  rescheduled_count, net_revenue, cancellation_rate, peak_hour,
  last_updated_at, last_event_id
)
SELECT
  '00000000-0000-0000-0000-000000000001',
  generate_series(CURRENT_DATE - 29, CURRENT_DATE, '1 day'::interval)::date,
  floor(random() * 6 + 2)::int,  -- 2-7 bookings/day
  floor(random() * 2)::int,       -- 0-1 cancellations/day
  0,
  (floor(random() * 6 + 2) * 4500)::bigint,
  0.05 + random() * 0.10,         -- 5-15% cancellation rate
  10 + floor(random() * 5)::int,  -- peak hour 10:00-14:00
  NOW(),
  NULL
ON CONFLICT (tenant_id, view_date) DO NOTHING;
```

### 3h. Infrastructure Entities

The local seed also creates:

- One completed `outbox_record` (already published) to verify relay logic
- Three `inbox_records` (for the confirmed appointments) to verify deduplication
- Five `audit_log` entries covering tenant creation, config update, and the completed appointments

---

## 4. CI / Integration Test Fixtures

CI fixtures are loaded per test suite, not globally. Each test suite is self-contained and manages its own setup and teardown. The shared seed pattern is:

```typescript
// test/fixtures/tenant.fixture.ts
export const TEST_TENANT_ID = '10000000-0000-0000-0000-000000000001';

export async function seedTestTenant(db: Pool): Promise<void> {
  await db.query(`
    INSERT INTO tenants (tenant_id, name, owner_email, owner_name, plan, status, created_at)
    VALUES ($1, 'Test Tenant', 'test@example.invalid', 'Test Owner', 'pro', 'active', NOW())
    ON CONFLICT (tenant_id) DO NOTHING
  `, [TEST_TENANT_ID]);

  await db.query(`SET LOCAL app.tenant_id = $1`, [TEST_TENANT_ID]);
}

export async function teardownTestTenant(db: Pool): Promise<void> {
  // Delete in FK-safe order
  await db.query(`DELETE FROM audit_log WHERE tenant_id = $1`, [TEST_TENANT_ID]);
  await db.query(`DELETE FROM appointment_events WHERE tenant_id = $1`, [TEST_TENANT_ID]);
  await db.query(`DELETE FROM slot_locks WHERE tenant_id = $1`, [TEST_TENANT_ID]);
  await db.query(`DELETE FROM appointments WHERE tenant_id = $1`, [TEST_TENANT_ID]);
  await db.query(`DELETE FROM customers WHERE tenant_id = $1`, [TEST_TENANT_ID]);
  await db.query(`DELETE FROM services WHERE tenant_id = $1`, [TEST_TENANT_ID]);
  await db.query(`DELETE FROM working_hours WHERE tenant_id = $1`, [TEST_TENANT_ID]);
  await db.query(`DELETE FROM staff_members WHERE tenant_id = $1`, [TEST_TENANT_ID]);
  await db.query(`DELETE FROM tenant_configs WHERE tenant_id = $1`, [TEST_TENANT_ID]);
  await db.query(`DELETE FROM tenants WHERE tenant_id = $1`, [TEST_TENANT_ID]);
}
```

### 4a. CI Fixture Datasets

| Dataset Name | Purpose | Entities |
|---|---|---|
| `minimal-tenant` | Tenant-level tests (provisioning, suspension, config) | 1 tenant, 1 config |
| `booking-ready-tenant` | Booking flow tests (request, confirm, cancel, reschedule) | 1 tenant + 2 staff + 3 services + working hours |
| `customer-with-bookings` | Customer journey tests, max-bookings BR008 enforcement | Above + 1 customer + 3 confirmed appointments |
| `dlq-scenario` | DLQ triage tests (UJ006) | 1 tenant + corrupted outbox record + failed notification |
| `export-ready-tenant` | Data export tests (UJ007) | Full tenant with 10 appointments (3 cancelled, 1 rescheduled, 6 confirmed) |
| `rls-isolation` | Cross-tenant isolation tests | 2 tenants with identical entity structures |

### 4b. Cross-Tenant Isolation Test Fixture

The `rls-isolation` fixture is specifically designed to verify BR005. It seeds two identical tenants:

```sql
-- Tenant A
INSERT INTO tenants VALUES ('A0000000-...', 'Tenant A', ...)
INSERT INTO customers VALUES ('A0000001-...', 'A0000000-...', 'Customer A', 'a@example.invalid', ...)

-- Tenant B
INSERT INTO tenants VALUES ('B0000000-...', 'Tenant B', ...)
INSERT INTO customers VALUES ('B0000001-...', 'B0000000-...', 'Customer B', 'b@example.invalid', ...)
```

The test asserts:

```typescript
it('should never return Tenant B data when authenticated as Tenant A', async () => {
  // Set RLS context to Tenant A
  await db.query(`SET LOCAL app.tenant_id = 'A0000000-...'`);

  const result = await db.query(`SELECT * FROM customers`);

  // Must return only Tenant A's customer — never Tenant B's
  expect(result.rows).toHaveLength(1);
  expect(result.rows[0].tenant_id).toBe('A0000000-...');
  expect(result.rows.every(r => r.tenant_id === 'A0000000-...')).toBe(true);
});

it('should return zero rows when tenant context is missing', async () => {
  // No SET LOCAL app.tenant_id — simulates missing context
  const result = await db.query(`SELECT * FROM customers`);

  // RLS safe default: zero rows (never all rows)
  expect(result.rows).toHaveLength(0);
});
```

---

## 5. Staging Environment Seed

The staging environment uses a nightly reset to the canonical "Serenity Salon" demo state:

```bash
# Runs nightly at 01:00 UTC (GitHub Actions cron)
- name: Reset staging seed
  run: |
    npm run db:reset:staging   # Truncates all tables, re-runs migrations
    npm run db:seed:staging    # Runs R__seed_dev.sql against staging DB
```

Staging uses separate Stripe test-mode keys. All payment operations in staging use Stripe's test card numbers (`4242 4242 4242 4242`). No real money moves.

The staging SendGrid integration sends to a single catch-all inbox (`staging-inbox@anjiSchedulo.dev`) rather than real recipients — no notification reaches real email addresses from staging.

---

## 6. Seed Script Commands

| Command | Environment | Effect |
|---|---|---|
| `npm run db:seed` | Local | Runs `R__seed_dev.sql` — idempotent |
| `npm run db:reset` | Local | Drops and recreates DB, runs all migrations, then seeds |
| `npm run db:seed:ci` | CI | Runs `R__seed_ci.sql` — minimal fixtures only |
| `npm run db:reset:staging` | Staging | Truncates staging DB, re-runs migrations |
| `npm run db:seed:staging` | Staging | Runs `R__seed_dev.sql` against staging |
| `npm run db:seed:verify` | Any | Runs assertions against expected seed state (used in CI smoke test) |

---

## 7. Seed Data IDs Reference

All deterministic seed UUIDs follow the pattern `{prefix}-0000-0000-0000-{suffix}` for easy identification in logs and test output.

| Entity | Seed ID | Display Name |
|---|---|---|
| Tenant (Serenity Salon) | `00000000-0000-0000-0000-000000000001` | Serenity Salon |
| Staff — Alex Chen | `00000000-0000-0000-0000-000000000101` | Alex Chen |
| Staff — Jordan Reeves | `00000000-0000-0000-0000-000000000102` | Jordan Reeves |
| Staff — Sam Torres | `00000000-0000-0000-0000-000000000103` | Sam Torres |
| Service — Haircut | `00000000-0000-0000-0000-000000000201` | Haircut (45 min, £45) |
| Service — Colour | `00000000-0000-0000-0000-000000000202` | Colour Treatment (120 min, £95) |
| Service — Blow-Dry | `00000000-0000-0000-0000-000000000203` | Blow-Dry (30 min, £25) |
| Service — Consult | `00000000-0000-0000-0000-000000000204` | Consultation (20 min, free) |
| Customer — Alice | `00000000-0000-0000-0000-000000000301` | Alice Green |
| Customer — Ben | `00000000-0000-0000-0000-000000000302` | Ben Harper |
| Customer — Clara | `00000000-0000-0000-0000-000000000303` | Clara Kim |
| CI Tenant A | `10000000-0000-0000-0000-000000000001` | Test Tenant (CI) |
| CI Tenant B | `20000000-0000-0000-0000-000000000001` | Isolation Tenant (CI) |

---

## 8. Seed Data Governance

- Seed files are reviewed in pull requests alongside their associated feature or migration.
- Adding a new entity type requires updating this document with the seed pattern for that entity.
- Seed IDs are reserved namespaces: production data must never use IDs starting with `00000000-`, `10000000-`, or `20000000-` (enforced by a runtime guard that rejects these prefixes in non-development environments).
- PII in seed data is entirely synthetic. No real customer names, email addresses, or phone numbers appear in any seed file. CI pipelines run `npm run db:seed:verify` to assert no real-looking PII patterns exist in seed data.
