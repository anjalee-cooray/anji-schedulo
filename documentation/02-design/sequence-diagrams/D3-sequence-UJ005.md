---
title: Sequence Diagram — Tenant Dashboard Review
journeyId: UJ005
layer: 02-design
status: current
lastUpdated: 2026-06-28
---

# D3 · Sequence Diagram — UJ005: Tenant Dashboard Review

## Overview

This diagram shows two concurrent flows that make the tenant dashboard work:

1. **Write side (background):** `appointment.*` events arriving from SNS → SQS → dashboard-service, which applies idempotent incremental updates to the `tenant_dashboard_views` materialised view.
2. **Read side (foreground):** The tenant admin opens the dashboard and receives a < 200ms response served directly from the pre-aggregated view — no joins, no real-time computation.

The CQRS pattern separates these two concerns entirely. The read path is always fast; the write path is eventually consistent with a < 5s lag target.

---

```mermaid
sequenceDiagram
    participant BCS as booking-command-service
    participant OR as outbox-relay
    participant SNS as SNS
    participant SQS as SQS (dashboard queue)
    participant DS as dashboard-service
    participant DB as PostgreSQL<br/>(tenant_dashboard_views)
    participant GW as api-gateway
    participant Admin as Tenant Admin (Browser)

    Note over BCS,DB: Write Side — Background Event Processing (CQRS write path)

    Note over BCS: When appointment saga completes...
    BCS->>BCS: INSERT outbox_records (appointment.confirmed / cancelled / rescheduled / completed)

    loop Every 500ms
        OR->>BCS: Poll outbox_records WHERE published_at IS NULL
    end

    OR->>+SNS: Publish appointment.confirmed<br/>(or .cancelled / .rescheduled / .completed)
    SNS-->>-OR: MessageId
    OR->>BCS: UPDATE outbox_records SET published_at=NOW()

    SNS->>SQS: Deliver to dashboard-service-appointment-confirmed-queue<br/>(FIFO, MessageGroupId=tenant_id)

    SQS->>+DS: Receive appointment.confirmed message

    Note over DS: Idempotency check (NFR009)
    DS->>DS: INSERT inbox_records (event_id, consumer=dashboard-service)<br/>— if unique constraint violation: skip (already processed)

    DS->>+DB: SELECT FROM tenant_dashboard_views<br/>WHERE tenant_id=? AND view_date=?
    DB-->>-DS: Current view state (confirmed_count, net_revenue, etc.)

    Note over DS: Check: event_id > last_event_id (watermark — prevents out-of-order updates)

    alt Event already reflected (last_event_id ≥ this event)
        DS->>SQS: Delete message (idempotent — no re-processing)
    end

    DS->>+DB: UPDATE tenant_dashboard_views SET<br/>confirmed_count = confirmed_count + 1,<br/>net_revenue = net_revenue + amount_charged,<br/>last_event_id = event_id,<br/>last_updated_at = NOW()<br/>WHERE tenant_id=? AND view_date=?
    DB-->>-DS: Row updated

    DS->>-SQS: Delete message (ack)

    Note over DS,DB: Same pattern for .cancelled (increment cancelled_count, recalculate cancellation_rate),<br/>.rescheduled (increment rescheduled_count),<br/>.completed (update peak_hour)

    Note over Admin,DB: ─────────────────────────────────────────
    Note over Admin,DB: Read Side — Dashboard Request (CQRS read path)

    Admin->>+GW: GET /api/v1/tenants/{id}/dashboard?date=today<br/>Bearer JWT (role=tenant_admin)
    GW->>GW: Validate JWT signature (RS256)<br/>Verify role=tenant_admin<br/>Extract tenant_id from tid claim

    GW->>+DS: Forward dashboard request (tenant_id, date)

    DS->>+DB: SELECT confirmed_count, cancelled_count,<br/>rescheduled_count, net_revenue,<br/>cancellation_rate, peak_hour,<br/>last_updated_at<br/>FROM tenant_dashboard_views<br/>WHERE tenant_id=? AND view_date=?<br/>(O(1) — single indexed row, RLS enforced)
    DB-->>-DS: View row

    DS->>DS: Format response (derive cancellation_rate = cancelled/confirmed if not stored)
    DS-->>-GW: 200 OK {dashboard data}
    GW-->>-Admin: 200 OK {<br/>  confirmed_count,<br/>  cancelled_count,<br/>  rescheduled_count,<br/>  net_revenue,<br/>  cancellation_rate,<br/>  peak_hour,<br/>  last_updated_at<br/>}<br/>(< 200ms — NFR006)

    Note over Admin,DB: Dashboard lag: time between appointment event and view update < 5s (NFR008)

    Note over Admin,DB: Rebuild path — if view is corrupted:
    Note over Admin,DB: ops-service triggers full AppointmentEvent replay for tenant<br/>dashboard-service reprocesses all events idempotently<br/>via InboxRecord watermark — same write-side flow above
```

---

## Key Design Points

### Why CQRS + Materialised View?

The dashboard serves a radically different access pattern from the booking write side:
- **Write side:** High-volume, concurrent, saga-orchestrated, strongly consistent.
- **Read side:** Low-latency (< 200ms), single-tenant, pre-aggregated, never blocking write operations.

Joining live tables at dashboard query time would not meet the 200ms target under load. A pre-aggregated materialised view, updated incrementally by the event stream, delivers O(1) read performance regardless of appointment volume.

### Idempotency (NFR009)

The `last_event_id` watermark on `tenant_dashboard_views` ensures that:
- If SQS delivers the same event twice, the second update is a no-op (event_id ≤ last_event_id).
- Replay events are safe: they reapply each event once in order, rebuilding the view correctly.

### Lag target

The dashboard reflects booking events within **5 seconds** under normal load (NFR008). This is the sum of: outbox-relay poll interval (≤ 500ms) + SNS → SQS delivery (< 1s) + dashboard-service processing (< 100ms) + DB write (< 100ms).

---

## Key NFRs Satisfied

| NFR | Target | How |
|---|---|---|
| NFR005 | Dashboard < 200ms p95 | O(1) read from pre-aggregated view, no joins |
| NFR006 | Dashboard served from materialized view | tenant_dashboard_views — no write-side join |
| NFR008 | Dashboard lag < 5 seconds | Outbox relay + FIFO SQS + incremental update |
| NFR009 | Idempotent consumer | InboxRecord + last_event_id watermark |
