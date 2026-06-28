---
title: User Stories — Tenant Dashboard Review
journeyId: UJ005
layer: 02-design
status: current
lastUpdated: 2026-06-28
---

# D12 · User Stories — Tenant Dashboard Review (UJ005)

## Epic Header

| Field | Value |
|---|---|
| Epic Name | Tenant Dashboard Review |
| Journey ID | UJ005 |
| Primary Persona | P1 — Tenant Admin (Business Owner) |
| Related FRs | FR008, FR009 |
| Related BRs | BR005 |
| Priority | High |
| Architecture Pattern | CQRS + Materialized View, Event Aggregator, Idempotent Consumer |

## Epic Description

As a **Tenant Admin**, I want to open a dashboard at the start of my business day and see a real-time summary of today's confirmed bookings, net revenue, cancellation rate, and anomaly alerts — so that I can make informed staffing and operational decisions without running manual reports.

---

## User Stories

---

### US-UJ005-01 · View today's booking summary

**Title:** Load a complete operational summary for today in under 200 ms

**User Story:**
As a **Tenant Admin**, I want to open the dashboard and immediately see today's booking counts, net revenue, and cancellation rate, so that I have a full operational picture before my first customer arrives.

**Acceptance Criteria:**

- **Given** I am authenticated as a tenant admin and I navigate to my dashboard,  
  **When** the page loads,  
  **Then** the response contains `confirmed_count`, `cancelled_count`, `rescheduled_count`, `net_revenue`, `cancellation_rate`, and `peak_hour` for today — all returned within 200ms (FR008, NFR006).

- **Given** a new appointment is confirmed during the day,  
  **When** the `appointment.confirmed` event is processed by dashboard-service,  
  **Then** the dashboard reflects the incremented `confirmed_count` and updated `net_revenue` within 5 seconds of the event being published (NFR008).

- **Given** I have no bookings today,  
  **When** the dashboard loads,  
  **Then** `confirmed_count = 0`, `net_revenue = 0.00`, `cancellation_rate = 0.0` — the page renders without errors in empty-state.

- **Given** I am authenticated as a tenant admin,  
  **When** the dashboard query executes,  
  **Then** only my own tenant's data is returned — the `TenantDashboardView` is read with RLS enforcing `tenant_id` scope (BR005).

**Size:** M  
**Priority:** High  
**Related:** FR008, BR005

**Technical Notes:**
- Dashboard data is served directly from `TenantDashboardView` — a pre-aggregated materialised view updated incrementally by dashboard-service on every `AppointmentEvent`.
- No joins to write-side tables at read time; the < 200ms response target is achievable because the view is indexed on `(tenant_id, view_date)`.
- `last_event_id` watermark on `TenantDashboardView` enables idempotent incremental updates: dashboard-service checks the watermark before applying each event to avoid double-counting on SQS redelivery.
- PostgreSQL RLS is the safety net: even if application code omits a tenant filter, RLS returns zero rows for an incorrect `tenant_id` context (BR005).

---

### US-UJ005-02 · Identify peak booking hour for staffing decisions

**Title:** See which hour of the day has the highest confirmed booking volume

**User Story:**
As a **Tenant Admin**, I want to see the peak booking hour on my dashboard, so that I can make informed decisions about staff scheduling and break times.

**Acceptance Criteria:**

- **Given** today's confirmed bookings span multiple hours,  
  **When** I view the dashboard,  
  **Then** `peak_hour` shows the hour of day (0–23) with the highest number of confirmed bookings for today.

- **Given** a new booking is confirmed at a time that creates a new peak hour,  
  **When** dashboard-service processes the `appointment.confirmed` event,  
  **Then** `peak_hour` is recalculated and updated in `TenantDashboardView` within 5 seconds.

- **Given** today has no confirmed bookings,  
  **When** the dashboard loads,  
  **Then** `peak_hour` is `null` — it is not shown or defaults to a misleading value.

**Size:** S  
**Priority:** Medium  
**Related:** FR008

**Technical Notes:**
- `peak_hour` is recomputed by dashboard-service on each `appointment.confirmed` and `appointment.completed` event by scanning the `slot_start` hour distribution within the current view_date window.
- The `appointment.completed` event also triggers a `peak_hour` recalculation to reflect appointments that ran to completion vs. were cancelled.
- `peak_hour` is stored as an integer (0–23) in `TenantDashboardView`; the frontend formats it as a human-readable time range.

---

### US-UJ005-03 · Receive real-time anomaly alerts on the dashboard

**Title:** Surface booking velocity spikes and high cancellation rate alerts within 60 seconds

**User Story:**
As a **Tenant Admin**, I want to see anomaly alerts on my dashboard as soon as they are detected, so that I can investigate and respond to unusual booking patterns before they affect my business operations.

**Acceptance Criteria:**

- **Given** analytics-service detects a booking velocity spike (> 3× the tenant's historical 5-minute average),  
  **When** the alert is generated,  
  **Then** it appears on my dashboard within 60 seconds of detection (FR009).

- **Given** a cancellation rate alert fires (> 30% over the last 50 bookings),  
  **When** the alert is surfaced,  
  **Then** the dashboard shows the specific cancellation rate percentage and the sample size of 50 bookings (FR009).

- **Given** an alert is displayed on my dashboard,  
  **When** I view it,  
  **Then** the alert includes the detection window (start and end timestamps) so I can correlate it with specific bookings.

- **Given** the anomaly condition resolves itself (e.g. velocity returns to normal),  
  **When** the next analytics window completes,  
  **Then** the alert is automatically cleared from the dashboard — no manual dismissal is required.

**Size:** M  
**Priority:** High  
**Related:** FR009

**Technical Notes:**
- analytics-service operates on a sliding 5-minute Kinesis window for velocity and a rolling 50-booking window for cancellation rate — both per tenant_id.
- `analytics.velocity_spike_detected` and `analytics.cancellation_rate_alert` events fan out to notification-service (for email alert) and ops-service (for operator visibility).
- Dashboard polling interval: the frontend polls the dashboard API every 10 seconds so alert display lag is at most 10 seconds after the event is processed.
- Analytics alerts are introduced in v1.0 per roadmap.json — not in MVP scope but the dashboard API response schema includes an `alerts[]` field from day one.

---

### US-UJ005-04 · Rebuild dashboard from event history after data corruption

**Title:** Restore an accurate dashboard view by replaying the full AppointmentEvent stream

**User Story:**
As a **Platform Operator**, I want to trigger a dashboard rebuild for a tenant by replaying their AppointmentEvent history, so that a corrupted or stale TenantDashboardView can be restored to a correct state without data loss.

**Acceptance Criteria:**

- **Given** `TenantDashboardView` for a tenant is corrupted or shows stale data,  
  **When** a Platform Operator triggers a full-tenant replay for that tenant via the ops console,  
  **Then** dashboard-service processes the replayed events idempotently and rebuilds `TenantDashboardView` with accurate counts (FR011).

- **Given** a replay is in progress,  
  **When** live booking events continue to arrive,  
  **Then** live booking operations are not blocked or slowed — replay events and live events are processed from the same SQS queue and the InboxRecord watermark prevents double-counting.

- **Given** the replay completes,  
  **When** I open the dashboard as the tenant admin,  
  **Then** `confirmed_count`, `cancelled_count`, `rescheduled_count`, `net_revenue`, `cancellation_rate`, and `peak_hour` all reflect the true state derived from the complete event history.

- **Given** replayed events arrive,  
  **When** dashboard-service applies them,  
  **Then** the `last_event_id` watermark is updated per event so that any event already applied (from a previous replay or normal processing) is skipped without side effects.

**Size:** M  
**Priority:** Medium  
**Related:** FR008, FR011

**Technical Notes:**
- ops-service reads `AppointmentEvent[]` from PostgreSQL and re-publishes them to the `appointment-*` SNS topics tagged with `replay_job_id`.
- dashboard-service identifies replay traffic by the `replay_job_id` field in the event envelope and logs it accordingly — live traffic has no `replay_job_id`.
- `TenantDashboardView` is reset to zero before a full-tenant replay begins; partial (date-range) replays increment/decrement from the current state.
- `last_event_id` watermark prevents double-counting: if an event_id has already been applied, the incremental update is skipped.

---

## Definition of Ready

Before any story in this epic enters a sprint, confirm all items below are checked:

- [ ] Acceptance criteria reviewed and agreed by Product Owner
- [ ] `TenantDashboardView` schema and `last_event_id` watermark logic finalised in `specs/ai/domain.json`
- [ ] Dashboard API response schema agreed — including `alerts[]` array shape for future analytics integration
- [ ] 200ms response time target confirmed achievable with `TenantDashboardView` index on `(tenant_id, view_date)` (NFR006)
- [ ] 5-second event lag target confirmed with dashboard-service team (NFR008)
- [ ] InboxRecord deduplication for all `appointment.*` events in dashboard-service verified
- [ ] RLS policy on `TenantDashboardView` confirmed — `tenant_id` scope enforced at the database layer (BR005)
- [ ] Replay rebuild path confirmed: `TenantDashboardView` reset + idempotent incremental replay (FR011)
- [ ] Analytics alert event schemas confirmed from analytics-service team (FR009)
- [ ] Definition of Done agreed for this layer (see `documentation/00-governance/G5-definition-of-done.md`)
