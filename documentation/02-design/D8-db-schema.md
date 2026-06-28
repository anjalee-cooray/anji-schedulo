# D8 · Database Schema

**Project:** AnjiSchedulo  
**Version:** 1.0.0  
**Owner:** Pubudu Anjalee Cooray  
**Last Updated:** 2026-06-28

---

## 1. Overview

AnjiSchedulo uses **PostgreSQL 15** as its primary data store. The schema is multi-tenant, with Row-Level Security (RLS) enforced on every table. All tenant-scoped tables carry a `tenant_id` column; RLS policies restrict every query to the current session's tenant context. A null tenant context returns zero rows — never all rows (BR005).

The schema is organised into five logical groups:

| Group | Tables | Purpose |
|---|---|---|
| **Tenant** | `tenants`, `tenant_configs` | Tenant identity, plan, status, and operational config |
| **Catalogue** | `services`, `staff_members`, `working_hours`, `staff_blocks` | Bookable services and staff availability |
| **Booking** | `customers`, `appointments`, `appointment_events`, `slot_locks`, `payments` | Core booking lifecycle and payment |
| **Operations** | `notifications`, `tenant_dashboard_views`, `analytics_summaries`, `audit_log`, `replay_jobs` | Observability, dashboarding, and ops tooling |
| **Infrastructure** | `outbox_records`, `inbox_records`, `booking_idempotency_keys` | Event durability, deduplication, and idempotency |

---

## 2. Database Configuration

| Parameter | Value |
|---|---|
| Engine | PostgreSQL 15 |
| Starter / Pro instance | `db.t4g.medium` — shared cluster, Multi-AZ |
| Enterprise instance | `db.r7g.large` — dedicated cluster per tenant, Multi-AZ |
| Storage (Starter/Pro) | gp3 SSD, 100 GB initial, autoscaling to 1 TB |
| Storage (Enterprise) | io2 SSD, 200 GB initial |
| Connection pooler | PgBouncer — transaction mode, max 200 connections per service |
| Encryption at rest | AES-256 via AWS KMS |
| Encryption in transit | TLS 1.3 (`ssl=require` on all connections) |
| Extensions | `pgcrypto`, `pg_stat_statements`, `pg_trgm` |
| Backups | Automated daily snapshots (7-day retention) + WAL archiving (RPO: 5 min) |
| Cross-region backup | Nightly snapshot copy to `eu-central-1` |

### RLS Activation Pattern

Every connection sets the tenant context before executing any query:

```sql
SET LOCAL app.tenant_id = '<tenant_uuid>';
```

All RLS policies reference `current_setting('app.tenant_id', true)`. A missing or empty setting returns an empty string, which matches no `tenant_id` — ensuring zero rows are returned when context is absent.

---

## 3. Schema

### 3.1 Tenant Group

#### `tenants`

Aggregate root for a registered business.

```sql
CREATE TABLE tenants (
    tenant_id       UUID            PRIMARY KEY DEFAULT gen_random_uuid(),
    name            TEXT            NOT NULL,
    owner_email     TEXT            NOT NULL,                    -- PII
    plan            TEXT            NOT NULL CHECK (plan IN ('starter', 'pro', 'enterprise')),
    status          TEXT            NOT NULL DEFAULT 'pending'
                                    CHECK (status IN ('pending', 'provisioning', 'active', 'suspended', 'deprovisioned')),
    created_at      TIMESTAMPTZ     NOT NULL DEFAULT now(),
    deprovisioned_at TIMESTAMPTZ    NULL
);

CREATE UNIQUE INDEX idx_tenants_owner_email ON tenants (owner_email);
```

**Lifecycle:** `pending` → `provisioning` → `active` → `suspended` → `deprovisioned`  
**Business Rules:** BR005, BR011, BR012  
**Notes:** `owner_email` is PII. Subject to pseudonymisation on GDPR erasure request for the tenant owner (FR013). `deprovisioned_at` is set when deprovisioning begins; hard deletion is scheduled 90 days later (BR012).

---

#### `tenant_configs`

Tenant operational parameters governing booking rules.

```sql
CREATE TABLE tenant_configs (
    tenant_id               UUID        PRIMARY KEY REFERENCES tenants (tenant_id) ON DELETE CASCADE,
    cancellation_hours      INTEGER     NOT NULL DEFAULT 24 CHECK (cancellation_hours >= 0),
    booking_window_days     INTEGER     NOT NULL DEFAULT 30 CHECK (booking_window_days > 0),
    max_advance_bookings    INTEGER     NOT NULL DEFAULT 5  CHECK (max_advance_bookings > 0),
    timezone                TEXT        NOT NULL DEFAULT 'UTC',
    currency                CHAR(3)     NOT NULL DEFAULT 'USD'
);

ALTER TABLE tenant_configs ENABLE ROW LEVEL SECURITY;

CREATE POLICY tenant_configs_isolation ON tenant_configs
    USING (tenant_id::TEXT = current_setting('app.tenant_id', true));
```

**Business Rules:** BR002 (`cancellation_hours`), BR008 (`max_advance_bookings`), BR010 (`booking_window_days`)  
**Notes:** One row per tenant. Updated by `tenant-service` on admin configuration changes (FR002).

---

### 3.2 Catalogue Group

#### `services`

Bookable offerings defined by a tenant.

```sql
CREATE TABLE services (
    service_id      UUID            PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID            NOT NULL REFERENCES tenants (tenant_id),
    name            TEXT            NOT NULL,
    duration_minutes INTEGER        NOT NULL CHECK (duration_minutes > 0),
    price_amount    NUMERIC(10, 2)  NOT NULL DEFAULT 0.00 CHECK (price_amount >= 0),
    price_currency  CHAR(3)         NOT NULL DEFAULT 'USD',
    active          BOOLEAN         NOT NULL DEFAULT TRUE,
    created_at      TIMESTAMPTZ     NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ     NOT NULL DEFAULT now()
);

CREATE INDEX idx_services_tenant ON services (tenant_id);
CREATE INDEX idx_services_tenant_active ON services (tenant_id) WHERE active = TRUE;

ALTER TABLE services ENABLE ROW LEVEL SECURITY;

CREATE POLICY services_isolation ON services
    USING (tenant_id::TEXT = current_setting('app.tenant_id', true));
```

**Tier Limits:** Starter: 5 active services; Pro and Enterprise: unlimited  
**Business Rules:** BR003  
**Notes:** Deactivating a service (`active = FALSE`) does not cancel existing confirmed appointments referencing it. Historical appointments retain the `service_id` foreign key.

---

#### `staff_members`

Employees or contractors whose availability is managed on the platform (P2).

```sql
CREATE TABLE staff_members (
    staff_id        UUID            PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID            NOT NULL REFERENCES tenants (tenant_id),
    name            TEXT            NOT NULL,                    -- PII
    email           TEXT            NOT NULL,                    -- PII
    active          BOOLEAN         NOT NULL DEFAULT TRUE,
    created_at      TIMESTAMPTZ     NOT NULL DEFAULT now()
);

CREATE UNIQUE INDEX idx_staff_tenant_email ON staff_members (tenant_id, email);
CREATE INDEX idx_staff_tenant ON staff_members (tenant_id);

ALTER TABLE staff_members ENABLE ROW LEVEL SECURITY;

CREATE POLICY staff_members_isolation ON staff_members
    USING (tenant_id::TEXT = current_setting('app.tenant_id', true));
```

**Tier Limits:** Starter: 3; Pro: 20; Enterprise: unlimited  
**Notes:** `name` and `email` are PII. Subject to pseudonymisation on GDPR erasure. Deactivating a staff member does not cancel their existing confirmed appointments.

---

#### `working_hours`

Recurring weekly schedule for a staff member. Defines time windows within which slots are generated.

```sql
CREATE TABLE working_hours (
    working_hours_id    UUID        PRIMARY KEY DEFAULT gen_random_uuid(),
    staff_id            UUID        NOT NULL REFERENCES staff_members (staff_id) ON DELETE CASCADE,
    tenant_id           UUID        NOT NULL,
    day_of_week         TEXT        NOT NULL CHECK (day_of_week IN ('monday','tuesday','wednesday','thursday','friday','saturday','sunday')),
    start_time          TIME        NOT NULL,
    end_time            TIME        NOT NULL,
    effective_from      DATE        NOT NULL,
    effective_to        DATE        NULL,
    CONSTRAINT wh_valid_range CHECK (start_time < end_time)
);

CREATE INDEX idx_working_hours_staff ON working_hours (staff_id, day_of_week);
CREATE INDEX idx_working_hours_tenant ON working_hours (tenant_id);

ALTER TABLE working_hours ENABLE ROW LEVEL SECURITY;

CREATE POLICY working_hours_isolation ON working_hours
    USING (tenant_id::TEXT = current_setting('app.tenant_id', true));
```

**Business Rules:** BR003  
**Notes:** `effective_from`/`effective_to` allows future schedule changes without breaking historic appointment data. Availability Service queries active records (`effective_to IS NULL OR effective_to >= CURRENT_DATE`).

---

#### `staff_blocks`

Manual unavailability periods set by staff or admin, preventing bookings even within working hours.

```sql
CREATE TABLE staff_blocks (
    block_id            UUID        PRIMARY KEY DEFAULT gen_random_uuid(),
    staff_id            UUID        NOT NULL REFERENCES staff_members (staff_id) ON DELETE CASCADE,
    tenant_id           UUID        NOT NULL,
    block_start         TIMESTAMPTZ NOT NULL,
    block_end           TIMESTAMPTZ NOT NULL,
    reason              TEXT        NULL,
    created_by_role     TEXT        NOT NULL CHECK (created_by_role IN ('staff', 'admin')),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    CONSTRAINT sb_valid_range CHECK (block_start < block_end)
);

CREATE INDEX idx_staff_blocks_staff_time ON staff_blocks (staff_id, block_start, block_end);
CREATE INDEX idx_staff_blocks_tenant ON staff_blocks (tenant_id);

ALTER TABLE staff_blocks ENABLE ROW LEVEL SECURITY;

CREATE POLICY staff_blocks_isolation ON staff_blocks
    USING (tenant_id::TEXT = current_setting('app.tenant_id', true));
```

**Notes:** Consumed by Availability Service when computing open slots. Blocks are immutable once created; to modify, delete and recreate.

---

### 3.3 Booking Group

#### `customers`

End customers (P3) who book appointments. Tenant-scoped — no cross-tenant customer identity in v1.

```sql
CREATE TABLE customers (
    customer_id         UUID        PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id           UUID        NOT NULL REFERENCES tenants (tenant_id),
    name                TEXT        NOT NULL,                    -- PII
    email               TEXT        NOT NULL,                    -- PII
    phone               TEXT        NULL,                        -- PII
    active_booking_count INTEGER    NOT NULL DEFAULT 0 CHECK (active_booking_count >= 0),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    pseudonymised_at    TIMESTAMPTZ NULL
);

CREATE UNIQUE INDEX idx_customers_tenant_email ON customers (tenant_id, email);
CREATE INDEX idx_customers_tenant ON customers (tenant_id);

ALTER TABLE customers ENABLE ROW LEVEL SECURITY;

CREATE POLICY customers_isolation ON customers
    USING (tenant_id::TEXT = current_setting('app.tenant_id', true));
```

**Business Rules:** BR008 (`active_booking_count` enforces max active bookings per customer)  
**Notes:** `name`, `email`, `phone` are PII. On GDPR erasure, these fields are replaced with pseudonymous tokens and `pseudonymised_at` is set. `active_booking_count` is incremented/decremented atomically by `BookingCommandService` within the booking/cancellation saga.

---

#### `appointments`

Core aggregate. Represents a confirmed reservation. State is derived by replaying `appointment_events`.

```sql
CREATE TABLE appointments (
    appointment_id      UUID        PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id           UUID        NOT NULL REFERENCES tenants (tenant_id),
    customer_id         UUID        NOT NULL REFERENCES customers (customer_id),
    staff_id            UUID        NOT NULL REFERENCES staff_members (staff_id),
    service_id          UUID        NOT NULL REFERENCES services (service_id),
    slot_date           DATE        NOT NULL,
    slot_start          TIME        NOT NULL,
    slot_end            TIME        NOT NULL,
    idempotency_key     UUID        NOT NULL,
    correlation_id      UUID        NOT NULL,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE UNIQUE INDEX idx_appointments_idempotency ON appointments (tenant_id, idempotency_key);
CREATE INDEX idx_appointments_tenant ON appointments (tenant_id);
CREATE INDEX idx_appointments_customer ON appointments (tenant_id, customer_id);
CREATE INDEX idx_appointments_staff_date ON appointments (tenant_id, staff_id, slot_date);

ALTER TABLE appointments ENABLE ROW LEVEL SECURITY;

CREATE POLICY appointments_isolation ON appointments
    USING (tenant_id::TEXT = current_setting('app.tenant_id', true));
```

**Business Rules:** BR001, BR004, BR006  
**Notes:** No `status` column — current state is derived by replaying `appointment_events` in `occurred_at` order. `idempotency_key` maps to `booking_idempotency_keys` (BR006). `correlation_id` propagated from API Gateway (NFR014).

---

#### `appointment_events`

Immutable event stream for an appointment. Source of truth for appointment state.

```sql
CREATE TABLE appointment_events (
    event_id        UUID        PRIMARY KEY DEFAULT gen_random_uuid(),
    appointment_id  UUID        NOT NULL REFERENCES appointments (appointment_id),
    tenant_id       UUID        NOT NULL,
    event_type      TEXT        NOT NULL CHECK (event_type IN (
                        'appointment.requested',
                        'appointment.confirmed',
                        'appointment.cancelled',
                        'appointment.rescheduled',
                        'appointment.completed',
                        'appointment.failed'
                    )),
    occurred_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    correlation_id  UUID        NOT NULL,
    causation_id    UUID        NULL,
    actor_id        UUID        NOT NULL,
    actor_role      TEXT        NOT NULL CHECK (actor_role IN ('customer', 'staff', 'admin', 'system')),
    payload         JSONB       NOT NULL
);

CREATE INDEX idx_appt_events_appointment ON appointment_events (appointment_id, occurred_at);
CREATE INDEX idx_appt_events_tenant ON appointment_events (tenant_id, occurred_at);
CREATE INDEX idx_appt_events_type ON appointment_events (tenant_id, event_type);

ALTER TABLE appointment_events ENABLE ROW LEVEL SECURITY;

CREATE POLICY appointment_events_isolation ON appointment_events
    USING (tenant_id::TEXT = current_setting('app.tenant_id', true));
```

**Business Rules:** BR013, BR014  
**Notes:** INSERT-only — no UPDATE or DELETE permitted. `payload` contains the full appointment snapshot at event time (Event-Carried State Transfer), enabling downstream consumers to render notifications without calling back to the booking service. Required fields per BR013: `tenant_id`, `event_id` (= `event_id`), `correlation_id`, `occurred_at`.

---

#### `slot_locks`

Database-level exclusive reservation for a time slot. Primary mechanism for preventing double-bookings (BR001).

```sql
CREATE TABLE slot_locks (
    lock_id         UUID        PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID        NOT NULL,
    staff_id        UUID        NOT NULL REFERENCES staff_members (staff_id),
    slot_date       DATE        NOT NULL,
    slot_start      TIME        NOT NULL,
    appointment_id  UUID        NOT NULL REFERENCES appointments (appointment_id),
    locked_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
    released_at     TIMESTAMPTZ NULL
);

-- The core double-booking prevention constraint:
-- Only one active lock per (tenant, staff, date, slot) is permitted.
CREATE UNIQUE INDEX idx_slot_locks_unique_active
    ON slot_locks (tenant_id, staff_id, slot_date, slot_start)
    WHERE released_at IS NULL;

CREATE INDEX idx_slot_locks_appointment ON slot_locks (appointment_id);
CREATE INDEX idx_slot_locks_staff_date ON slot_locks (tenant_id, staff_id, slot_date);

ALTER TABLE slot_locks ENABLE ROW LEVEL SECURITY;

CREATE POLICY slot_locks_isolation ON slot_locks
    USING (tenant_id::TEXT = current_setting('app.tenant_id', true));
```

**Business Rules:** BR001, BR004, BR009  
**Notes:** `released_at IS NULL` means the slot is currently locked. The partial unique index is the enforcement mechanism for BR001 — concurrent insert attempts for the same slot fail with a unique constraint violation, triggering saga compensation. `released_at` is set atomically when a booking is cancelled or a saga is compensated.

---

#### `payments`

Payment record associated with a confirmed appointment.

```sql
CREATE TABLE payments (
    payment_id              UUID        PRIMARY KEY DEFAULT gen_random_uuid(),
    appointment_id          UUID        NOT NULL REFERENCES appointments (appointment_id),
    tenant_id               UUID        NOT NULL,
    stripe_payment_intent_id TEXT       NOT NULL,
    amount                  NUMERIC(10, 2) NOT NULL CHECK (amount >= 0),
    currency                CHAR(3)     NOT NULL,
    status                  TEXT        NOT NULL DEFAULT 'pending'
                                        CHECK (status IN ('pending', 'captured', 'failed', 'refunded', 'refund_failed')),
    captured_at             TIMESTAMPTZ NULL,
    refunded_at             TIMESTAMPTZ NULL,
    stripe_refund_id        TEXT        NULL,
    created_at              TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE UNIQUE INDEX idx_payments_stripe_intent ON payments (stripe_payment_intent_id);
CREATE INDEX idx_payments_appointment ON payments (appointment_id);
CREATE INDEX idx_payments_tenant ON payments (tenant_id);

ALTER TABLE payments ENABLE ROW LEVEL SECURITY;

CREATE POLICY payments_isolation ON payments
    USING (tenant_id::TEXT = current_setting('app.tenant_id', true));
```

**Business Rules:** BR004, BR009, BR012  
**Notes:** Payment is optional per tenant. `refund_failed` status routes to DLQ for manual ops resolution (BR009). Retained for 7 years per BR012 — excluded from all automated deletion jobs.

---

### 3.4 Operations Group

#### `notifications`

Outbound notification attempt records for all appointment lifecycle events.

```sql
CREATE TABLE notifications (
    notification_id     UUID        PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id           UUID        NOT NULL,
    appointment_id      UUID        NOT NULL REFERENCES appointments (appointment_id),
    recipient_type      TEXT        NOT NULL CHECK (recipient_type IN ('customer', 'staff', 'admin')),
    recipient_contact   TEXT        NOT NULL,                    -- PII (email or phone)
    channel             TEXT        NOT NULL CHECK (channel IN ('email', 'sms', 'whatsapp')),
    template_name       TEXT        NOT NULL,
    status              TEXT        NOT NULL DEFAULT 'pending'
                                    CHECK (status IN ('pending', 'sent', 'failed', 'dlq')),
    attempt_count       INTEGER     NOT NULL DEFAULT 0,
    last_attempted_at   TIMESTAMPTZ NULL,
    sent_at             TIMESTAMPTZ NULL,
    failure_reason      TEXT        NULL,
    trigger_event_id    UUID        NOT NULL,                    -- FK to appointment_events.event_id
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_notifications_appointment ON notifications (appointment_id);
CREATE INDEX idx_notifications_tenant_status ON notifications (tenant_id, status);

ALTER TABLE notifications ENABLE ROW LEVEL SECURITY;

CREATE POLICY notifications_isolation ON notifications
    USING (tenant_id::TEXT = current_setting('app.tenant_id', true));
```

**Business Rules:** BR007, BR013  
**Notes:** `recipient_contact` is PII. Retried up to 4 times with exponential backoff (FR007). After 4 failures, `status = 'dlq'` and ops alert fires. Notification failures never block booking operations (BR007).

---

#### `tenant_dashboard_views`

CQRS read-side materialised view providing O(1) dashboard data per tenant per day (FR008).

```sql
CREATE TABLE tenant_dashboard_views (
    tenant_id               UUID        NOT NULL,
    view_date               DATE        NOT NULL,
    confirmed_count         INTEGER     NOT NULL DEFAULT 0,
    cancelled_count         INTEGER     NOT NULL DEFAULT 0,
    rescheduled_count       INTEGER     NOT NULL DEFAULT 0,
    net_revenue             NUMERIC(12, 2) NOT NULL DEFAULT 0.00,
    cancellation_rate       NUMERIC(5, 4)  NOT NULL DEFAULT 0.0000,
    peak_hour               SMALLINT    NULL CHECK (peak_hour BETWEEN 0 AND 23),
    last_updated_at         TIMESTAMPTZ NOT NULL DEFAULT now(),
    last_event_id           UUID        NULL,                    -- watermark for idempotent updates
    PRIMARY KEY (tenant_id, view_date)
);

ALTER TABLE tenant_dashboard_views ENABLE ROW LEVEL SECURITY;

CREATE POLICY dashboard_views_isolation ON tenant_dashboard_views
    USING (tenant_id::TEXT = current_setting('app.tenant_id', true));
```

**NFRs:** NFR006 (< 200ms response), NFR008 (< 5s lag)  
**Notes:** Updated incrementally by `dashboard-service` on each `AppointmentEvent`. `last_event_id` watermark enables idempotent incremental updates and safe replay (FR011). Fully rebuilt from event replay after data corruption.

---

#### `analytics_summaries`

Near-real-time stream processing output for anomaly detection (FR009).

```sql
CREATE TABLE analytics_summaries (
    tenant_id           UUID        NOT NULL,
    window_start        TIMESTAMPTZ NOT NULL,
    window_end          TIMESTAMPTZ NOT NULL,
    booking_count       INTEGER     NOT NULL DEFAULT 0,
    cancellation_count  INTEGER     NOT NULL DEFAULT 0,
    velocity_ratio      NUMERIC(8, 4) NOT NULL DEFAULT 0.0000,
    alert_triggered     BOOLEAN     NOT NULL DEFAULT FALSE,
    computed_at         TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (tenant_id, window_start)
);

CREATE INDEX idx_analytics_tenant_time ON analytics_summaries (tenant_id, window_start DESC);

ALTER TABLE analytics_summaries ENABLE ROW LEVEL SECURITY;

CREATE POLICY analytics_summaries_isolation ON analytics_summaries
    USING (tenant_id::TEXT = current_setting('app.tenant_id', true));
```

**Notes:** Written by `analytics-service` from Kinesis stream processing. Alert fires when `velocity_ratio > 3.0` or cancellation rate over last 50 bookings > 30% (FR009).

---

#### `audit_log`

Immutable record of every write operation modifying tenant-scoped data (BR014).

```sql
CREATE TABLE audit_log (
    audit_id        UUID        PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID        NOT NULL,
    entity_type     TEXT        NOT NULL,
    entity_id       UUID        NOT NULL,
    operation       TEXT        NOT NULL CHECK (operation IN (
                        'create', 'update', 'cancel', 'reschedule',
                        'suspend', 'deprovision', 'export', 'replay', 'pseudonymise'
                    )),
    actor_id        UUID        NOT NULL,
    actor_role      TEXT        NOT NULL CHECK (actor_role IN ('customer', 'staff', 'admin', 'operator', 'system')),
    occurred_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    before_snapshot JSONB       NULL,
    after_snapshot  JSONB       NOT NULL,
    correlation_id  UUID        NULL
);

CREATE INDEX idx_audit_log_tenant ON audit_log (tenant_id, occurred_at DESC);
CREATE INDEX idx_audit_log_entity ON audit_log (entity_type, entity_id, occurred_at DESC);
```

**Business Rules:** BR014  
**Notes:** No RLS on `audit_log` — it is queried by platform operators across tenants for compliance. `booking_app_user` has INSERT-only privilege; no UPDATE or DELETE. Retained permanently — excluded from all tenant deprovisioning deletion jobs. Post-deprovisioning, records are retained under a system-scoped archive partition.

---

#### `replay_jobs`

Tracks lifecycle of tenant event replay operations (FR011).

```sql
CREATE TABLE replay_jobs (
    job_id          UUID        PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID        NOT NULL,
    initiated_by    UUID        NOT NULL,                        -- operator user_id
    scope_type      TEXT        NOT NULL CHECK (scope_type IN ('full_tenant', 'date_range', 'event_type', 'specific_ids')),
    scope_params    JSONB       NOT NULL DEFAULT '{}',
    status          TEXT        NOT NULL DEFAULT 'queued'
                                CHECK (status IN ('queued', 'running', 'completed', 'failed', 'cancelled')),
    events_published INTEGER    NOT NULL DEFAULT 0,
    started_at      TIMESTAMPTZ NULL,
    completed_at    TIMESTAMPTZ NULL,
    error_message   TEXT        NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_replay_jobs_tenant ON replay_jobs (tenant_id, created_at DESC);
CREATE INDEX idx_replay_jobs_status ON replay_jobs (status) WHERE status IN ('queued', 'running');
```

**Notes:** Created and updated by `ops-service`. Replay events are tagged with `job_id` so consumers can differentiate live vs replay traffic. Live booking operations are never blocked by replay (FR011).

---

### 3.5 Infrastructure Group

#### `outbox_records`

Transactional Outbox entries — written in the same DB transaction as the domain event (NFR003).

```sql
CREATE TABLE outbox_records (
    outbox_id       UUID        PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID        NOT NULL,
    topic           TEXT        NOT NULL,                        -- SNS topic ARN or name
    event_type      TEXT        NOT NULL,
    payload         JSONB       NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    published_at    TIMESTAMPTZ NULL                             -- NULL = pending publish
);

CREATE INDEX idx_outbox_pending ON outbox_records (created_at ASC)
    WHERE published_at IS NULL;
```

**Business Rules:** BR013  
**NFRs:** NFR003  
**Notes:** `outbox-relay` polls for `published_at IS NULL`, publishes to SNS, then sets `published_at`. At-least-once delivery guaranteed. Consumers handle duplicates via `inbox_records`.

---

#### `inbox_records`

Consumer-side deduplication table. One per consumer service (NFR009).

```sql
CREATE TABLE inbox_records (
    event_id        UUID        NOT NULL,
    consumer_name   TEXT        NOT NULL,
    processed_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    result          TEXT        NULL,
    PRIMARY KEY (event_id, consumer_name)
);
```

**Business Rules:** BR006  
**NFRs:** NFR009  
**Notes:** Before processing an event, the consumer attempts to INSERT a row. A unique constraint violation means the event was already processed — skip. Safe for replay (FR011). Each consumer service maintains its own partition of this table (e.g. `notification-service`, `dashboard-service`, `analytics-service`).

---

#### `booking_idempotency_keys`

Prevents duplicate booking operations from network retries (BR006).

```sql
CREATE TABLE booking_idempotency_keys (
    idempotency_key UUID        PRIMARY KEY,                     -- client-provided
    tenant_id       UUID        NOT NULL,
    appointment_id  UUID        NULL,                            -- set on success
    result_status   TEXT        NOT NULL CHECK (result_status IN ('confirmed', 'failed')),
    result_payload  JSONB       NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    expires_at      TIMESTAMPTZ NOT NULL DEFAULT (now() + INTERVAL '24 hours')
);

CREATE INDEX idx_idempotency_tenant ON booking_idempotency_keys (tenant_id);
CREATE INDEX idx_idempotency_expiry ON booking_idempotency_keys (expires_at)
    WHERE expires_at < now();
```

**Business Rules:** BR006  
**Notes:** If `idempotency_key` already exists, `BookingCommandService` returns `result_payload` without reprocessing. Expired keys are purged by a scheduled cleanup job. TTL: 24 hours.

---

## 4. RLS Policy Summary

All tenant-scoped tables follow this pattern:

```sql
ALTER TABLE <table> ENABLE ROW LEVEL SECURITY;

CREATE POLICY <table>_isolation ON <table>
    USING (tenant_id::TEXT = current_setting('app.tenant_id', true));
```

The `audit_log` table is exempt from RLS — it is platform-scoped and queried by operators across tenants for compliance purposes. Access is controlled via role-based DB privileges instead.

| Table | RLS | Safe Default |
|---|---|---|
| `tenants` | No (platform-scoped, accessed by tenant-service) | N/A |
| `tenant_configs` | Yes | Zero rows |
| `services` | Yes | Zero rows |
| `staff_members` | Yes | Zero rows |
| `working_hours` | Yes | Zero rows |
| `staff_blocks` | Yes | Zero rows |
| `customers` | Yes | Zero rows |
| `appointments` | Yes | Zero rows |
| `appointment_events` | Yes | Zero rows |
| `slot_locks` | Yes | Zero rows |
| `payments` | Yes | Zero rows |
| `notifications` | Yes | Zero rows |
| `tenant_dashboard_views` | Yes | Zero rows |
| `analytics_summaries` | Yes | Zero rows |
| `audit_log` | No (platform-scoped) | N/A |
| `replay_jobs` | No (platform-scoped) | N/A |
| `outbox_records` | No (relay service scoped) | N/A |
| `inbox_records` | No (service-scoped) | N/A |
| `booking_idempotency_keys` | No (service-scoped) | N/A |

---

## 5. Key Indexes Summary

| Index | Table | Columns | Purpose |
|---|---|---|---|
| `idx_slot_locks_unique_active` | `slot_locks` | `(tenant_id, staff_id, slot_date, slot_start) WHERE released_at IS NULL` | Double-booking prevention (BR001) |
| `idx_appointments_idempotency` | `appointments` | `(tenant_id, idempotency_key)` | Idempotency enforcement (BR006) |
| `idx_customers_tenant_email` | `customers` | `(tenant_id, email)` | Customer lookup and uniqueness |
| `idx_staff_tenant_email` | `staff_members` | `(tenant_id, email)` | Staff uniqueness per tenant |
| `idx_outbox_pending` | `outbox_records` | `(created_at ASC) WHERE published_at IS NULL` | Outbox relay polling efficiency |
| `idx_idempotency_expiry` | `booking_idempotency_keys` | `(expires_at) WHERE expires_at < now()` | Efficient TTL cleanup |
| `idx_appt_events_appointment` | `appointment_events` | `(appointment_id, occurred_at)` | Event sourcing replay |
| `idx_appt_events_tenant` | `appointment_events` | `(tenant_id, occurred_at)` | Tenant-scoped replay (FR011) |

---

## 6. Database Privileges

```sql
-- Application service user — standard operations
CREATE ROLE booking_app_user LOGIN;
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO booking_app_user;

-- Audit log is INSERT-only (BR014)
REVOKE UPDATE, DELETE ON audit_log FROM booking_app_user;

-- Outbox relay user — polling and marking published
CREATE ROLE outbox_relay_user LOGIN;
GRANT SELECT, UPDATE ON outbox_records TO outbox_relay_user;

-- Read-only reporting user
CREATE ROLE reporting_user LOGIN;
GRANT SELECT ON tenant_dashboard_views, analytics_summaries, audit_log TO reporting_user;
```

---

## 7. Data Retention and Deletion Policy

| Table | Retention | Deletion Mechanism |
|---|---|---|
| Tenant data (all tenant-scoped tables) | 90 days post-deprovisioning | Scheduled batch job triggered 90 days after `tenants.deprovisioned_at` |
| `payments` | 7 years | Excluded from all deletion jobs (BR012) |
| `audit_log` | Permanent | Never deleted; INSERT-only |
| `booking_idempotency_keys` | 24 hours | Scheduled cleanup on `expires_at` |
| `outbox_records` (published) | 7 days | Scheduled cleanup on `published_at` |
| `inbox_records` | 90 days | Scheduled cleanup |
| Customer PII fields | Pseudonymised on erasure request | `UPDATE customers SET name='[REDACTED]', email='[REDACTED-<uuid>]', phone=NULL, pseudonymised_at=now()` |

---

## 8. Traceability

| Entity / Table | FR | BR | NFR |
|---|---|---|---|
| `tenants`, `tenant_configs` | FR001, FR002 | BR005, BR011, BR012 | NFR012 |
| `services`, `staff_members`, `working_hours`, `staff_blocks` | FR002, FR003 | BR003 | NFR005 |
| `customers` | FR004 | BR008 | — |
| `appointments`, `appointment_events` | FR004–FR006 | BR001, BR004, BR006, BR013, BR014 | NFR007, NFR014 |
| `slot_locks` | FR004, FR005 | BR001, BR004, BR009 | NFR007 |
| `payments` | FR004, FR005 | BR004, BR009, BR012 | — |
| `notifications` | FR007 | BR007, BR013 | NFR002 |
| `tenant_dashboard_views` | FR008, FR011 | — | NFR006, NFR008 |
| `analytics_summaries` | FR009 | — | — |
| `audit_log` | FR013 | BR014 | — |
| `replay_jobs` | FR011 | — | NFR009 |
| `outbox_records` | FR004–FR007 | BR013 | NFR003 |
| `inbox_records` | FR004–FR007 | BR006 | NFR009 |
| `booking_idempotency_keys` | FR004 | BR006 | — |
