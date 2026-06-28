---
title: Flow Spec — Tenant Dashboard Review
journeyId: UJ005
layer: 02-design
status: current
lastUpdated: 2026-06-28
---

# D2 · Flow Spec — UJ005: Tenant Dashboard Review

## Header

| Field | Value |
|---|---|
| Journey ID | UJ005 |
| Journey Name | Tenant Dashboard Review |
| Persona | P1 — Tenant Admin |
| Priority | High |
| Related Functional Requirements | FR008, FR009 |
| Related NFRs | NFR006 (< 200 ms dashboard response), NFR008 (< 5 s event lag) |
| Related Business Rules | BR005 |

---

## 1. Technical Flow Overview

**Architecture Pattern: CQRS + Materialised View + Stream Processing (Analytics)**

The tenant dashboard is a read-heavy CQRS pattern. The write side is driven by appointment lifecycle events arriving from the booking-command-service event stream; the read side serves a single pre-aggregated row per tenant per day with O(1) latency.

The key design decision is that **no joins occur at read time**. The `TenantDashboardView` is pre-computed and maintained incrementally by dashboard-service on each incoming `AppointmentEvent`. A single `SELECT` on this materialised row returns in under 200 ms regardless of booking volume (NFR006). The dashboard view can be rebuilt from scratch at any time by replaying the full `AppointmentEvent` history for a tenant (FR011).

Analytics alerts (booking velocity spikes, high cancellation rates) are a separate concern, computed by analytics-service via Kinesis stream processing and surfaced on the dashboard as a secondary panel.

---

## 2. Services Involved

| Service | Role in This Flow |
|---|---|
| api-gateway | Validates JWT (role = tenant_admin), enforces tid scoping, routes GET /dashboard to dashboard-service |
| dashboard-service | Owns `TenantDashboardView`. Maintains the materialised view by consuming AppointmentEvents. Serves the read API. |
| booking-command-service | Write-side producer of all appointment.* events that drive the view |
| outbox-relay | Publishes appointment.* events from booking-command-service to SNS |
| analytics-service | Maintains `AnalyticsSummary` via Kinesis stream processing; surfaces alerts to ops-service alert queue |
| ops-service | Collects analytics alerts; surfaced to dashboard as a secondary panel |

---

## 3. Step-by-Step Technical Flow

### The Write Side — Incremental View Maintenance (Continuous Background)

The write side runs continuously in the background, independent of any admin request. It is what makes the read side fast.

**Step 1 — AppointmentEvents arrive at dashboard-service**

dashboard-service maintains four dedicated SQS FIFO consumer queues:
- `dashboard-service-appointment-confirmed-queue`
- `dashboard-service-appointment-cancelled-queue`
- `dashboard-service-appointment-rescheduled-queue`
- `dashboard-service-appointment-completed-queue`

Each queue has `MessageGroupId = tenant_id` (ordered per tenant) and a paired DLQ with `maxReceiveCount = 4`.

**Step 2 — InboxRecord deduplication**

Before processing any event, dashboard-service attempts:
```sql
INSERT INTO inbox_records (event_id, consumer_name)
VALUES ($event_id, 'dashboard-service')
```
If the INSERT fails due to a unique constraint, the event has already been processed. The SQS message is deleted without reprocessing. This ensures safe at-least-once delivery from SQS (NFR009).

**Step 3 — Incremental view update**

For each event type, dashboard-service applies a targeted update to `tenant_dashboard_views`. All updates use the `last_event_id` watermark to ensure idempotency:

**On `appointment.confirmed`:**
```sql
INSERT INTO tenant_dashboard_views
  (tenant_id, view_date, confirmed_count, net_revenue, last_event_id, last_updated_at)
VALUES
  ($tenant_id, $slot_date, 1, $amount_charged, $event_id, NOW())
ON CONFLICT (tenant_id, view_date) DO UPDATE
SET confirmed_count   = tenant_dashboard_views.confirmed_count + 1,
    net_revenue       = tenant_dashboard_views.net_revenue + EXCLUDED.net_revenue,
    last_event_id     = EXCLUDED.last_event_id,
    last_updated_at   = NOW()
WHERE tenant_dashboard_views.last_event_id != EXCLUDED.last_event_id
```

**On `appointment.cancelled`:**
```sql
UPDATE tenant_dashboard_views
SET cancelled_count    = cancelled_count + 1,
    cancellation_rate  = CAST(cancelled_count + 1 AS decimal)
                         / NULLIF(confirmed_count + cancelled_count + 1, 0),
    last_event_id      = $event_id,
    last_updated_at    = NOW()
WHERE tenant_id = $tenant_id
  AND view_date = $slot_date
  AND last_event_id != $event_id
```

**On `appointment.rescheduled`:**
```sql
UPDATE tenant_dashboard_views
SET rescheduled_count = rescheduled_count + 1,
    last_event_id     = $event_id,
    last_updated_at   = NOW()
WHERE tenant_id = $tenant_id
  AND view_date = $slot_date
  AND last_event_id != $event_id
```

**On `appointment.completed`:**
```sql
UPDATE tenant_dashboard_views
SET peak_hour       = (
      SELECT EXTRACT(HOUR FROM slot_start::time)
      FROM appointment_events
      WHERE tenant_id = $tenant_id
        AND event_type = 'appointment.confirmed'
        AND occurred_at::date = $slot_date
      GROUP BY EXTRACT(HOUR FROM slot_start::time)
      ORDER BY COUNT(*) DESC
      LIMIT 1
    ),
    last_event_id   = $event_id,
    last_updated_at = NOW()
WHERE tenant_id = $tenant_id
  AND view_date = $slot_date
  AND last_event_id != $event_id
```

The `last_event_id != $event_id` guard in the WHERE clause makes each update idempotent — a re-delivered event produces the same result as if it were not re-delivered.

**Step 4 — SQS message deleted**

After a successful UPDATE, dashboard-service deletes the SQS message from the queue. If the UPDATE fails, the message is not deleted — SQS will redeliver after the visibility timeout, triggering retry.

---

### The Read Side — Dashboard Request (On Admin Load)

**Step 5 — Tenant admin opens the dashboard**

The tenant admin (P1) navigates to the AnjiSchedulo dashboard. The frontend calls:
```
GET /api/v1/dashboard
Authorization: Bearer <JWT>
```

api-gateway validates:
- JWT signature (RS256, public key from key registry)
- `role = tenant_admin`
- `exp` not expired
- Injects `X-Correlation-ID` and forwards to dashboard-service.

**Step 6 — dashboard-service enforces tenant scope**

dashboard-service sets the PostgreSQL RLS context before querying:
```sql
SET LOCAL app.tenant_id = $tid  -- extracted from JWT tid claim
```

This activates the Row-Level Security policy on `tenant_dashboard_views`, which enforces `tenant_id = current_setting('app.tenant_id')`. Even if a bug in application code omitted the WHERE clause, RLS returns zero rows for any other tenant's data (BR005, NFR012).

**Step 7 — Single-row read from TenantDashboardView**

```sql
SELECT
  confirmed_count,
  cancelled_count,
  rescheduled_count,
  net_revenue,
  cancellation_rate,
  peak_hour,
  last_updated_at
FROM tenant_dashboard_views
WHERE tenant_id = $tenant_id
  AND view_date = CURRENT_DATE
```

This is a primary key lookup — O(1), no joins, no aggregation at read time. Response time target: under 200 ms (NFR006).

If no row exists for today (first day of business, or new tenant), an empty row with zeroed counters is returned rather than a 404.

**Step 8 — Fetch upcoming bookings list**

To populate the day's schedule panel, dashboard-service queries the booking read model:
```sql
SELECT
  appointment_id, slot_start, slot_end,
  customer_name, staff_name, service_name, status
FROM appointment_read_view
WHERE tenant_id = $tenant_id
  AND slot_date = CURRENT_DATE
ORDER BY slot_start ASC
LIMIT 50
```

This view is maintained by booking-command-service's read projection. RLS is active. Returned in the same API response.

**Step 9 — Fetch analytics alerts (if any)**

dashboard-service checks for active analytics alerts:
```sql
SELECT alert_type, detected_at, metric_value
FROM analytics_alerts
WHERE tenant_id = $tenant_id
  AND resolved_at IS NULL
ORDER BY detected_at DESC
LIMIT 5
```

Active alerts (velocity spike, high cancellation rate) are returned as a `alerts[]` array in the response. Empty array if no active alerts.

**Step 10 — dashboard-service returns HTTP 200**

Response payload:
```json
{
  "date": "2026-06-28",
  "confirmed_count": 7,
  "cancelled_count": 1,
  "rescheduled_count": 0,
  "net_revenue": 34500,
  "currency": "GBP",
  "cancellation_rate": 0.125,
  "peak_hour": 10,
  "last_updated_at": "2026-06-28T08:47:12Z",
  "upcoming_bookings": [
    {
      "appointment_id": "...",
      "slot_start": "09:00",
      "slot_end": "10:00",
      "customer_name": "Jane Smith",
      "staff_name": "Alex Johnson",
      "service_name": "Haircut",
      "status": "confirmed"
    }
  ],
  "alerts": []
}
```

---

### The Analytics Path — Stream Processing (Continuous Background)

**Step 11 — analytics-service computes sliding windows via Kinesis**

analytics-service is a Kinesis consumer. Appointment events are written to a Kinesis Data Stream (`appointment-events-stream`, partition key = `tenant_id`) in parallel with the SQS fan-out.

analytics-service maintains two rolling windows per tenant:
- **Velocity window (5-minute sliding):** Counts `appointment.confirmed` events in the last 5 minutes. If count > 3× the tenant's historical 5-minute average → trigger `analytics.velocity_spike_detected` alert.
- **Cancellation rate window (last 50 bookings):** Tracks the ratio of `appointment.cancelled` to total appointments in the last 50 events. If ratio > 0.30 → trigger `analytics.cancellation_rate_alert`.

Alert conditions: `velocity_ratio > 3.0` OR `cancellation_rate > 0.30` (FR009).

Alert delivery to tenant admin and operator: within 60 seconds of detection (FR009).

**Step 12 — Alert surfaced on dashboard**

When an alert fires, analytics-service publishes to `analytics-velocity-spike-topic` or `analytics-cancellation-alert-topic`. ops-service receives the alert, stores it in `analytics_alerts`, and it is returned in Step 9 the next time the dashboard is polled. The frontend polls the dashboard API every 30 seconds to surface new alerts without requiring a page reload.

---

### Event Replay — Rebuilding a Corrupted View (Operator-Triggered)

**Step 13 — Operator triggers view rebuild**

If `TenantDashboardView` becomes corrupted (e.g. after a bug fix or data migration), a Platform Operator (P4) triggers a replay from the ops console (FR011):
```
POST /ops/replay
{ "tenant_id": "<uuid>", "scope_type": "full_tenant", "target_consumer": "dashboard-service" }
```

ops-service reads all `AppointmentEvent` records for the tenant from PostgreSQL in `occurred_at` order and re-publishes them to SNS.

**Step 14 — dashboard-service rebuilds view from replay**

dashboard-service receives the replayed events through its standard SQS queues. The `last_event_id` watermark on `TenantDashboardView` ensures each event is applied exactly once even if it was previously applied. Events tagged with `replay_job_id` are processed identically to live events — the view update logic is the same.

Before replay begins, the operator may optionally truncate `TenantDashboardView` for the affected tenant_id to rebuild from zero. ops-service can do this via the ops console: `DELETE FROM tenant_dashboard_views WHERE tenant_id = $tenant_id`.

Live booking operations continue unaffected during replay. The dashboard may show stale data until the replay completes, which takes seconds to minutes depending on event history volume.

---

## 4. Compensation / Failure Flows

### Failure Point 1 — Dashboard lag > 5 seconds (SQS queue backlog)

**Impact:** `TenantDashboardView.last_updated_at` falls more than 5 seconds behind the current booking activity. Admin sees slightly stale data.

**Detection:** Grafana alert fires when SQS queue depth for any `dashboard-service-appointment-*-queue` exceeds 100 messages or when `last_updated_at` age exceeds 10 seconds.

**Resolution:** Ops investigates dashboard-service consumer throughput. If the service is healthy, the backlog typically clears in seconds. If dashboard-service is down, restart it — SQS retains messages and delivery resumes automatically.

### Failure Point 2 — TenantDashboardView row corrupted or missing

**Impact:** Admin receives zeroed-out or incorrect dashboard data.

**Resolution:** Platform Operator triggers a full event replay scoped to the affected tenant (Step 13–14). View is rebuilt from the canonical `AppointmentEvent` history within minutes. No booking operations are affected.

### Failure Point 3 — dashboard-service is down during a dashboard request

**Impact:** Admin receives a `503 Service Unavailable`. Booking operations continue unaffected — dashboard-service is a fully decoupled read-side service (BR007 principle applied to reads).

**Resolution:** Service restarts automatically via ECS health checks. SQS buffers all incoming events during the outage; events are processed in order when the service recovers. The dashboard data will be slightly stale for the duration of the outage, then reconcile automatically.

### Failure Point 4 — analytics-service Kinesis consumer lag

**Impact:** Velocity spike or cancellation rate alert is delayed beyond 60 seconds.

**Detection:** Kinesis consumer lag metric monitored in Grafana. Alert fires if lag exceeds 60 seconds.

**Resolution:** Ops investigates analytics-service throughput. In degraded mode, alerts are delayed but the dashboard itself continues to serve accurate booking counts and revenue data from `TenantDashboardView`.

---

## 5. Events Consumed

| Event Name | Source Queue | Triggers |
|---|---|---|
| `appointment.confirmed` | `dashboard-service-appointment-confirmed-queue` | confirmed_count++, net_revenue += amount |
| `appointment.cancelled` | `dashboard-service-appointment-cancelled-queue` | cancelled_count++, recalculate cancellation_rate |
| `appointment.rescheduled` | `dashboard-service-appointment-rescheduled-queue` | rescheduled_count++ |
| `appointment.completed` | `dashboard-service-appointment-completed-queue` | Recalculate peak_hour |

dashboard-service produces **no domain events**. It is a pure read-side service.

---

## 6. Business Rules Enforced

| BR ID | Rule Summary | Enforcement Point |
|---|---|---|
| BR005 | Tenant data must never be readable by another tenant | dashboard-service: `SET LOCAL app.tenant_id` before every query; PostgreSQL RLS enforces tenant scoping at DB layer; safe default returns zero rows when context is missing (NFR012) |

NFR006 (< 200 ms response) and NFR008 (< 5 s event lag) are performance targets, not business rules, but are enforced through the materialised view design and SQS consumer throughput monitoring.

---

## 7. Consistency Guarantees

| Concern | Consistency Model | Detail |
|---|---|---|
| Dashboard view updates | Eventually consistent | dashboard-service is an async SQS consumer. Target lag < 5 seconds under normal load (NFR008) |
| Dashboard read response | Strong (for own tenant) | PostgreSQL RLS enforces strict tenant scoping; no cross-tenant reads possible |
| Analytics alert detection | Eventually consistent | analytics-service Kinesis window. Alert delivery within 60 seconds of condition trigger (FR009) |
| View rebuild from replay | Eventually consistent | Replay re-processes events in occurred_at order; view reaches correct state after all events processed |

---

## 8. Idempotency

### Incremental view updates

Each UPDATE to `TenantDashboardView` includes the guard:
```sql
WHERE last_event_id != $current_event_id
```

This ensures that re-delivered SQS messages produce no double-counting. The `last_event_id` field acts as the watermark for idempotent incremental updates (NFR009).

### InboxRecord deduplication

Before processing any event, dashboard-service inserts a row into `inbox_records (event_id, consumer_name = 'dashboard-service')`. If the insert fails due to a unique constraint, the event has already been processed and the SQS message is deleted immediately. This is the primary deduplication layer; the `last_event_id` watermark is a secondary defence against the unlikely case where an InboxRecord is written but the view UPDATE fails and is retried.

### Replay safety

Replayed events carry the same `event_id` as the original events. The InboxRecord and `last_event_id` watermark together ensure that replaying an event that was already applied is a safe no-op. If the operator truncates `TenantDashboardView` before replay, `inbox_records` for `dashboard-service` should also be cleared for the affected tenant to allow full re-processing.
