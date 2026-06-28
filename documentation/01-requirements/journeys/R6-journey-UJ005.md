---
title: Journey — Tenant Dashboard Review
journeyId: UJ005
persona: P1
priority: high
layer: 01-requirements
status: current
lastUpdated: 2026-06-28
---

# R6 · Journey UJ005 — Tenant Dashboard Review

## Metadata

| Field | Value |
|---|---|
| Journey ID | UJ005 |
| Name | Tenant Dashboard Review |
| Primary Persona | [P1 — Tenant Admin](../personas/R4a-persona-P1.md) |
| Priority | High |
| Related Functional Requirements | FR008, FR009 |
| Related Business Rules | BR005 |
| Architecture Pattern | CQRS + Materialized View |
| Services Involved | dashboard-service, booking-query-service |

---

## Goal

A tenant admin opens the operational dashboard at the start of their business day and gets an accurate, complete picture of their booking operation — today's confirmed bookings, net revenue, cancellation rate, and peak hour — within one second, without running reports or navigating multiple screens.

---

## Preconditions

- The tenant's account is in `active` status.
- The admin is authenticated with a valid JWT containing `role = tenant_admin` and a `tid` claim scoped to their tenant.
- At least one appointment exists for the tenant (otherwise the dashboard is empty but functional).
- The dashboard-service has been consuming `AppointmentEvent` messages without a gap; if a gap occurred, a replay job may be required before data is current.

---

## Step-by-Step Journey

### Step 1 — Open the Dashboard

The tenant admin logs into the AnjiSchedulo web application and navigates to the Dashboard. The client sends a single authenticated `GET /api/v1/dashboard` request. The api-gateway validates the JWT, extracts `tenant_id` from the `tid` claim, and routes the request to dashboard-service.

**Outcome:** A dashboard request is in flight. The admin sees a loading indicator for a sub-second period.

---

### Step 2 — View Today's Confirmed Bookings and Upcoming Schedule

dashboard-service queries the `TenantDashboardView` table using `WHERE tenant_id = :tid AND view_date = CURRENT_DATE`. This is a direct primary-key lookup on a pre-aggregated row — no joins, no aggregation at query time. The response returns within 200 milliseconds under normal load (NFR006).

The dashboard surface shows:
- Today's confirmed booking count
- Rescheduled and cancelled counts for today
- The upcoming appointments list (fetched from booking-query-service) in chronological order

**Outcome:** The admin has an immediate, accurate view of today's schedule. No phone calls, no whiteboard checks.

---

### Step 3 — Check Cancellation Rate and Revenue

The same `TenantDashboardView` row carries `net_revenue` (sum of captured payments for today's confirmed appointments), `cancellation_rate` (cancelled / total bookings for today), and `peak_hour` (the hour with the most confirmed bookings).

The admin reviews these figures:
- A high cancellation rate (e.g. > 15%) may prompt them to review their cancellation policy or contact no-shows.
- Net revenue provides a quick sanity check before reconciling with their payment provider at end of day.

**Outcome:** Financial and operational health indicators are visible at a glance without running a report.

---

### Step 4 — Identify Peak Hours for Staffing Decisions

The `peak_hour` field in the dashboard view tells the admin which hour of the day has historically attracted the most bookings. The admin uses this information to plan staffing — for example, ensuring a senior stylist is available at 11 AM on Saturdays when demand peaks.

**Outcome:** The admin has actionable staffing intelligence derived automatically from their booking history, with no manual analysis required.

---

### Step 5 — Act on Any Alerts

If the analytics-service has detected an anomaly — a booking velocity spike or an elevated cancellation rate over the last 50 bookings — an alert banner appears on the dashboard. The admin can click through to a detail view showing the affected metric, the detection window, and any recommended action.

Alert types surfaced:
- Booking velocity > 3× the tenant's 5-minute historical average (potential unusual demand — may indicate a promotion or a data entry error)
- Cancellation rate > 30% over last 50 bookings (potential policy issue or service quality problem)

**Outcome:** The admin is not surprised by trends that have been building for hours. They can act on emerging patterns before they become operational problems.

---

## Success Outcome

The admin opens the dashboard and within one second sees today's seven confirmed bookings laid out in chronological order, a net revenue figure, a healthy 5% cancellation rate, and a peak hour of 11 AM. The data reflects the last booking event within 5 seconds of it occurring (NFR008). No anomaly alerts are active. The admin closes the tab and begins their day having taken no manual action.

---

## Failure Paths

### Dashboard Response Exceeds 200 Milliseconds

If the `TenantDashboardView` row does not exist for today (e.g. first day of operation) or the query plan degrades due to missing index, the response may exceed the 200ms target. The platform should create an empty `TenantDashboardView` row for the current date when the first event of the day arrives, ensuring the row always exists by the time the admin opens the dashboard.

**Platform behaviour:** Monitoring alert fires if dashboard p95 latency exceeds 200ms (observability.json). The admin receives a slightly slower response but correct data.

### Dashboard Data Is Stale (Lag > 5 Seconds)

If the dashboard-service consumer falls behind — e.g. due to an SQS processing delay or a service crash — the `TenantDashboardView` may reflect a state from more than 5 seconds ago. The dashboard surface should display a staleness indicator when `last_updated_at` is more than 5 seconds behind current time.

**Platform behaviour:** Grafana alert fires on `dashboard_event_lag_seconds > 5` (observability.json). Platform Operator investigates and restarts dashboard-service if needed. The admin's data is temporarily stale but eventually consistent.

### Corrupted or Incorrect Materialized View

If a bug in the dashboard-service projection logic incorrectly computed aggregates (e.g. double-counted a rescheduled event), the data shown may be wrong. The resolution is for the Platform Operator to trigger an event replay (UJ006, FR011) — the `TenantDashboardView` is rebuilt from the immutable `AppointmentEvent` history using correct projection logic.

**Platform behaviour:** Platform Operator triggers ReplayJob scoped to the affected tenant. Dashboard data is temporarily cleared and rebuilt. The `last_event_id` watermark in the view ensures idempotent rebuild.

### Analytics Alert Not Visible

If the analytics-service is not publishing alerts (e.g. Kinesis stream processing lag), the admin may not see a cancellation rate alert that should have been surfaced. This is an eventual consistency gap — alerts are near-real-time but not guaranteed to arrive within any specific sub-second window.

**Platform behaviour:** Grafana alert fires on analytics processing lag. Platform Operator investigates Kinesis consumer.

---

## Related Business Rules

| Rule ID | Rule Summary | Application in This Journey |
|---|---|---|
| BR005 | A tenant's data must never be readable by another tenant. | The `TenantDashboardView` query is always scoped to `WHERE tenant_id = :tid`, derived from the JWT's `tid` claim. PostgreSQL RLS enforces the same constraint as a safety net — a missing tenant_id filter returns zero rows, never all rows. |

---

## Implementation Note

The `TenantDashboardView` is a pre-aggregated CQRS read-side projection maintained by dashboard-service. Every time an `AppointmentEvent` is processed — confirmed, cancelled, rescheduled, or completed — dashboard-service increments or decrements the relevant counters atomically using the `last_event_id` watermark to skip already-processed events (idempotent incremental update). Reads are O(1) primary-key lookups on `(tenant_id, view_date)` with no joins, enabling the sub-200ms response target. The view can be fully rebuilt from the `appointment_events` table at any time by issuing a replay job — the `last_event_id` watermark is reset to NULL and all events for the tenant are replayed in `occurred_at` order.
