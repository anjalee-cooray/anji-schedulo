---
title: API Design
layer: 02-design
status: current
lastUpdated: 2026-06-28
---

# D5 · API Design

## Overview

AnjiSchedulo exposes a REST API following OpenAPI 3.1 conventions. The API is versioned at `/api/v1/`. All requests are routed through the `api-gateway` service, which handles authentication, rate limiting, correlation ID injection, and tenant context extraction before forwarding to the appropriate downstream service.

**Base URL:** `https://api.anji-schedulo.io/api/v1`

**Auth:** JWT RS256. All endpoints require a valid `Authorization: Bearer <token>` header except those marked `[PUBLIC]`. The `tid` claim in the JWT identifies the tenant — api-gateway extracts it and injects it into the downstream request context. Downstream services never accept `tenant_id` from request bodies; they read it from the validated JWT context only (BR005).

**Idempotency:** Booking mutations (`POST /appointments`) require an `Idempotency-Key` header (UUID). See BookingIdempotencyKey in D1.

**Error format (all error responses):**
```json
{
  "error": {
    "code": "SLOT_UNAVAILABLE",
    "message": "The requested slot is no longer available.",
    "correlation_id": "9f41a3bb-..."
  }
}
```

---

## Error Codes Reference

| HTTP Status | Error Code | Meaning |
|---|---|---|
| 400 | `VALIDATION_ERROR` | Request body or query param failed validation |
| 401 | `UNAUTHENTICATED` | JWT missing, expired, or invalid |
| 403 | `FORBIDDEN` | Valid JWT but insufficient role or wrong tenant |
| 404 | `NOT_FOUND` | Resource does not exist within tenant scope |
| 409 | `SLOT_UNAVAILABLE` | Double-booking attempt (BR001) |
| 409 | `DUPLICATE_REGISTRATION` | Email already registered (BR on tenant creation) |
| 409 | `IDEMPOTENT_CONFLICT` | Idempotency key already used with different params |
| 422 | `CANCELLATION_WINDOW_CLOSED` | Cancellation requested inside minimum window (BR002) |
| 422 | `BOOKING_LIMIT_EXCEEDED` | Customer already has max_advance_bookings (BR008) |
| 422 | `BOOKING_WINDOW_EXCEEDED` | Slot date beyond booking_window_days (BR010) |
| 429 | `RATE_LIMITED` | Request rate exceeds tenant tier limit |
| 503 | `SERVICE_UNAVAILABLE` | Upstream dependency unavailable; retry with Retry-After |

---

## 1. Tenant Management

### POST /tenants
**[PUBLIC — no auth required]**  
Create a new tenant account. Initiates the provisioning saga (UJ001, FR001).

**Request Body:**
```json
{
  "name": "string",
  "owner_email": "string (email)",
  "owner_name": "string",
  "plan": "starter | pro | enterprise"
}
```

**Responses:**

`202 Accepted` — Tenant provisioning initiated.
```json
{
  "tenant_id": "uuid",
  "status": "provisioning",
  "correlation_id": "uuid"
}
```

`409 Conflict` — `DUPLICATE_REGISTRATION` — email already associated with an active or provisioning tenant.

`422 Unprocessable Entity` — `VALIDATION_ERROR` — field-level errors returned.

`503 Service Unavailable` — Stripe Customer creation failed; `Retry-After` header included.

**Notes:** The endpoint returns `202 Accepted` because provisioning is asynchronous. Poll `GET /tenants/{tenant_id}/status` to track completion. The `tenant.provisioned` event is published via Transactional Outbox (ADR005).

---

### GET /tenants/{tenant_id}/status
**[AUTH: tenant_admin, platform_operator]**  
Poll tenant provisioning or suspension status.

**Path Parameter:** `tenant_id` (UUID)

**Responses:**

`200 OK`
```json
{
  "tenant_id": "uuid",
  "status": "provisioning | active | suspended | deprovisioned",
  "plan": "starter | pro | enterprise"
}
```

`404 Not Found` — Tenant not found within requesting context.

---

### PUT /tenants/{tenant_id}/config
**[AUTH: tenant_admin]**  
Update tenant operational configuration (cancellation hours, booking window, etc.).

**Path Parameter:** `tenant_id` (UUID)

**Request Body:**
```json
{
  "cancellation_hours": "integer (≥ 0)",
  "booking_window_days": "integer (1–365)",
  "max_advance_bookings": "integer (1–50)",
  "timezone": "string (IANA, e.g. Europe/London)",
  "currency": "string (ISO 4217, e.g. GBP)"
}
```

**Responses:**

`200 OK` — Config updated. `tenant.configured` event published to downstream services.

`403 Forbidden` — Not a tenant_admin for this tenant.

`422 Unprocessable Entity` — Validation error with field-level detail.

---

### POST /tenants/{tenant_id}/export
**[AUTH: tenant_admin]**  
Request a full data export for the tenant (UJ007, FR012). Asynchronous — returns a job ID.

**Request Body:** Empty

**Responses:**

`202 Accepted`
```json
{
  "export_job_id": "uuid",
  "estimated_completion_seconds": 120
}
```

`409 Conflict` — An export is already in progress for this tenant.

Poll `GET /tenants/{tenant_id}/export/{export_job_id}` for status. Completed exports provide a signed S3 URL valid for 24 hours.

---

## 2. Services (Catalogue)

### GET /tenants/{tenant_id}/services
**[PUBLIC — no auth; or AUTH for admin view with inactive services]**  
List all active services for a tenant. Auth users with `tenant_admin` role see inactive services too.

**Query Parameters:**
- `include_inactive=true` — Admin only; returns both active and inactive services.

**Responses:**

`200 OK`
```json
{
  "services": [
    {
      "service_id": "uuid",
      "name": "string",
      "duration_minutes": "integer",
      "price_amount": "integer (smallest unit)",
      "price_currency": "string",
      "active": "boolean"
    }
  ]
}
```

---

### POST /tenants/{tenant_id}/services
**[AUTH: tenant_admin]**  
Create a new service.

**Request Body:**
```json
{
  "name": "string",
  "duration_minutes": "integer (≥ 15)",
  "price_amount": "integer (≥ 0)",
  "price_currency": "string (ISO 4217)"
}
```

**Responses:**

`201 Created`
```json
{
  "service_id": "uuid",
  "name": "string",
  "duration_minutes": "integer",
  "price_amount": "integer",
  "price_currency": "string",
  "active": true
}
```

`422 Unprocessable Entity` — Validation error or plan tier limit reached (max 5 for Starter).

---

### PATCH /tenants/{tenant_id}/services/{service_id}
**[AUTH: tenant_admin]**  
Update a service or toggle its active state.

**Request Body (all fields optional):**
```json
{
  "name": "string",
  "duration_minutes": "integer (≥ 15)",
  "price_amount": "integer (≥ 0)",
  "price_currency": "string",
  "active": "boolean"
}
```

**Responses:**

`200 OK` — Updated service returned.

`404 Not Found` — Service not found within tenant scope.

---

## 3. Staff Members

### GET /tenants/{tenant_id}/staff
**[AUTH: tenant_admin, staff]**  
List all staff members for the tenant.

**Responses:**

`200 OK`
```json
{
  "staff": [
    {
      "staff_id": "uuid",
      "name": "string",
      "email": "string",
      "active": "boolean"
    }
  ]
}
```

---

### POST /tenants/{tenant_id}/staff
**[AUTH: tenant_admin]**  
Add a new staff member.

**Request Body:**
```json
{
  "name": "string",
  "email": "string (email)"
}
```

**Responses:**

`201 Created`
```json
{
  "staff_id": "uuid",
  "name": "string",
  "email": "string",
  "active": true
}
```

`422 Unprocessable Entity` — Plan tier limit reached (max 3 Starter, max 20 Pro).

---

### PUT /tenants/{tenant_id}/staff/{staff_id}/working-hours
**[AUTH: tenant_admin, staff (own record only)]**  
Set or replace the weekly working hours for a staff member.

**Request Body:**
```json
{
  "schedules": [
    {
      "day_of_week": "monday | tuesday | wednesday | thursday | friday | saturday | sunday",
      "start_time": "HH:MM",
      "end_time": "HH:MM",
      "effective_from": "YYYY-MM-DD"
    }
  ]
}
```

**Responses:**

`200 OK` — Working hours updated. `tenant.configured` published.

`403 Forbidden` — Staff member attempted to modify another staff member's hours.

---

### POST /tenants/{tenant_id}/staff/{staff_id}/blocks
**[AUTH: tenant_admin, staff (own record only)]**  
Add a manual unavailability block for a staff member.

**Request Body:**
```json
{
  "block_start": "ISO 8601 datetime",
  "block_end": "ISO 8601 datetime",
  "reason": "string (optional)"
}
```

**Responses:**

`201 Created`
```json
{
  "block_id": "uuid",
  "staff_id": "uuid",
  "block_start": "ISO 8601 datetime",
  "block_end": "ISO 8601 datetime"
}
```

`422 Unprocessable Entity` — `block_end` before `block_start`.

---

## 4. Availability

### GET /tenants/{tenant_id}/availability
**[PUBLIC — no auth required]**  
Query available appointment slots for a given service, staff member, and date range. Consumed by the customer booking page (UJ002, FR003).

**Query Parameters:**
- `service_id` (UUID, required)
- `staff_id` (UUID, optional — omit to see slots across all qualified staff)
- `date_from` (YYYY-MM-DD, required)
- `date_to` (YYYY-MM-DD, required; max 30-day window)

**Responses:**

`200 OK`
```json
{
  "tenant_id": "uuid",
  "service_id": "uuid",
  "slots": [
    {
      "staff_id": "uuid",
      "staff_name": "string",
      "date": "YYYY-MM-DD",
      "start_time": "HH:MM",
      "end_time": "HH:MM",
      "available": true
    }
  ]
}
```

`400 Bad Request` — `date_to` before `date_from`, or range exceeds 30 days.

**Cache:** Availability responses are cached in Redis per `(tenant_id, service_id, staff_id, date)` with TTL 30 seconds. Cache is invalidated on `appointment.confirmed`, `appointment.cancelled`, and `tenant.configured` events.

**Performance:** p95 response time target < 200ms (NFR006). Cache hit expected for > 80% of requests.

---

## 5. Appointments (Booking)

### POST /appointments
**[AUTH: customer (JWT from booking page — may be anonymous token with tenant_id claim)]**  
Submit a booking request (UJ002, FR004). Idempotent via `Idempotency-Key` header (BR006).

**Headers:**
- `Idempotency-Key: <UUID>` (required)

**Request Body:**
```json
{
  "tenant_id": "uuid",
  "service_id": "uuid",
  "staff_id": "uuid",
  "slot_date": "YYYY-MM-DD",
  "slot_start": "HH:MM",
  "customer": {
    "name": "string",
    "email": "string (email)",
    "phone": "string (optional)"
  },
  "payment_method_id": "string (Stripe PaymentMethod ID; required if service price > 0)"
}
```

**Responses:**

`202 Accepted` — Booking saga initiated. Poll `GET /appointments/{appointment_id}` for confirmation.
```json
{
  "appointment_id": "uuid",
  "status": "requested",
  "correlation_id": "uuid"
}
```

`409 Conflict` — `SLOT_UNAVAILABLE` — slot already booked (BR001).

`409 Conflict` — `IDEMPOTENT_CONFLICT` — idempotency key already used with different parameters.

`422 Unprocessable Entity` — `BOOKING_LIMIT_EXCEEDED` — customer already has `max_advance_bookings` (BR008).

`422 Unprocessable Entity` — `BOOKING_WINDOW_EXCEEDED` — slot date beyond `booking_window_days` (BR010).

**Note:** Response is `202` because payment capture is asynchronous via Stripe webhook. The booking saga completes in < 5 seconds under normal conditions (NFR006).

---

### GET /appointments/{appointment_id}
**[AUTH: customer (own appointments), staff, tenant_admin]**  
Get the current derived state of an appointment.

**Path Parameter:** `appointment_id` (UUID)

**Responses:**

`200 OK`
```json
{
  "appointment_id": "uuid",
  "tenant_id": "uuid",
  "customer": {
    "customer_id": "uuid",
    "name": "string",
    "email": "string"
  },
  "staff_id": "uuid",
  "service": {
    "service_id": "uuid",
    "name": "string",
    "duration_minutes": "integer",
    "price_amount": "integer",
    "price_currency": "string"
  },
  "slot_date": "YYYY-MM-DD",
  "slot_start": "HH:MM",
  "slot_end": "HH:MM",
  "status": "requested | confirmed | rescheduled | cancelled | completed | failed",
  "payment_status": "pending | captured | failed | refunded | null",
  "created_at": "ISO 8601 datetime"
}
```

`403 Forbidden` — Customer attempting to read another customer's appointment.

`404 Not Found` — Appointment not found within tenant scope.

---

### POST /appointments/{appointment_id}/cancel
**[AUTH: customer (own), staff, tenant_admin]**  
Cancel a confirmed appointment (UJ004, FR006, BR002, BR009).

**Path Parameter:** `appointment_id` (UUID)

**Request Body:**
```json
{
  "reason": "string (optional)"
}
```

**Responses:**

`200 OK` — Appointment cancelled. Slot released. Refund triggered if applicable.
```json
{
  "appointment_id": "uuid",
  "status": "cancelled",
  "refund_status": "triggered | not_applicable"
}
```

`422 Unprocessable Entity` — `CANCELLATION_WINDOW_CLOSED` — within `cancellation_hours` of appointment (BR002). Response includes `cancellation_deadline` field.

`409 Conflict` — Appointment is already cancelled or completed.

---

### POST /appointments/{appointment_id}/reschedule
**[AUTH: customer (own), staff, tenant_admin]**  
Reschedule a confirmed appointment to a new slot (UJ003, FR005).

**Path Parameter:** `appointment_id` (UUID)

**Request Body:**
```json
{
  "new_slot_date": "YYYY-MM-DD",
  "new_slot_start": "HH:MM"
}
```

**Responses:**

`200 OK` — Appointment rescheduled. Old slot released. New slot locked.
```json
{
  "appointment_id": "uuid",
  "status": "rescheduled",
  "slot_date": "YYYY-MM-DD",
  "slot_start": "HH:MM",
  "slot_end": "HH:MM"
}
```

`409 Conflict` — `SLOT_UNAVAILABLE` — new slot already booked (BR001). Original appointment remains `confirmed`.

`422 Unprocessable Entity` — New slot date beyond `booking_window_days` (BR010).

---

### GET /tenants/{tenant_id}/appointments
**[AUTH: tenant_admin, staff]**  
List appointments for the tenant with filtering.

**Query Parameters:**
- `date_from` (YYYY-MM-DD, optional)
- `date_to` (YYYY-MM-DD, optional)
- `staff_id` (UUID, optional)
- `status` (optional — `confirmed | cancelled | completed | all`)
- `page` (integer, default 1)
- `per_page` (integer, default 50, max 200)

**Responses:**

`200 OK`
```json
{
  "appointments": [ "...appointment objects..." ],
  "pagination": {
    "page": 1,
    "per_page": 50,
    "total": 342,
    "total_pages": 7
  }
}
```

---

## 6. Dashboard

### GET /tenants/{tenant_id}/dashboard
**[AUTH: tenant_admin]**  
Retrieve the pre-aggregated dashboard view for today (UJ005, FR008).

**Query Parameters:**
- `date` (YYYY-MM-DD, optional — defaults to today in tenant's configured timezone)

**Responses:**

`200 OK` — Served from `TenantDashboardView` read model. p95 < 200ms (NFR006).
```json
{
  "tenant_id": "uuid",
  "view_date": "YYYY-MM-DD",
  "confirmed_count": "integer",
  "cancelled_count": "integer",
  "rescheduled_count": "integer",
  "net_revenue": "decimal (e.g. 1250.00)",
  "cancellation_rate": "decimal (0.0–1.0)",
  "peak_hour": "integer | null",
  "last_updated_at": "ISO 8601 datetime"
}
```

---

### GET /tenants/{tenant_id}/analytics/anomalies
**[AUTH: tenant_admin]**  
Get the most recent anomaly summary for the tenant (FR009).

**Responses:**

`200 OK`
```json
{
  "tenant_id": "uuid",
  "window_start": "ISO 8601 datetime",
  "window_end": "ISO 8601 datetime",
  "booking_count": "integer",
  "cancellation_count": "integer",
  "velocity_ratio": "decimal",
  "alert_triggered": "boolean",
  "computed_at": "ISO 8601 datetime"
}
```

---

## 7. GDPR / Data Subject Rights

### DELETE /tenants/{tenant_id}/customers/{customer_id}/personal-data
**[AUTH: tenant_admin, platform_operator]**  
Pseudonymise a customer's PII data (FR013). The Customer record is retained (for appointment history continuity), but `name`, `email`, and `phone` are replaced with pseudonymous tokens. The `appointment_events` records retain the `customer_id` FK, but the PII has been removed from the source.

**Request Body:** Empty

**Responses:**

`200 OK`
```json
{
  "customer_id": "uuid",
  "pseudonymised_at": "ISO 8601 datetime",
  "fields_anonymised": ["name", "email", "phone"]
}
```

`409 Conflict` — Customer has active confirmed appointments that have not yet been completed or cancelled.

---

## 8. Platform Operator Endpoints

These endpoints require `platform_operator` role (P4). Accessible only to AnjiSchedulo staff.

### POST /ops/tenants/{tenant_id}/suspend
Suspend a tenant account.

**Request Body:**
```json
{
  "reason": "string"
}
```

**Responses:**

`200 OK` — Tenant status set to `suspended`. `tenant.suspended` event published.

---

### POST /ops/tenants/{tenant_id}/reinstate
Reinstate a suspended tenant.

**Responses:**

`200 OK` — Tenant status set to `active`.

---

### GET /ops/dlq
List all DLQ entries across all queues (FR010).

**Query Parameters:**
- `queue_name` (string, optional)
- `tenant_id` (UUID, optional)
- `event_type` (string, optional)

**Responses:**

`200 OK`
```json
{
  "entries": [
    {
      "dlq_entry_id": "uuid",
      "queue_name": "string",
      "event_type": "string",
      "tenant_id": "uuid",
      "failure_reason": "string",
      "received_at": "ISO 8601 datetime",
      "payload": {}
    }
  ]
}
```

---

### POST /ops/dlq/{dlq_entry_id}/requeue
Re-queue a DLQ entry to its source queue after correction (FR010).

**Request Body:**
```json
{
  "corrected_payload": {} 
}
```

**Responses:**

`200 OK` — Event re-queued. AuditLog entry created (BR014).

---

### POST /ops/replay
Initiate an event replay job (FR011, UJ006).

**Request Body:**
```json
{
  "tenant_id": "uuid",
  "scope_type": "full_tenant | date_range | event_type | specific_ids",
  "scope_params": {
    "from": "YYYY-MM-DD (for date_range)",
    "to": "YYYY-MM-DD (for date_range)",
    "event_type": "string (for event_type scope)",
    "event_ids": ["uuid", "..."] 
  }
}
```

**Responses:**

`202 Accepted`
```json
{
  "job_id": "uuid",
  "status": "queued"
}
```

---

### GET /ops/replay/{job_id}
Get replay job status.

**Responses:**

`200 OK`
```json
{
  "job_id": "uuid",
  "tenant_id": "uuid",
  "status": "queued | running | completed | failed | cancelled",
  "events_published": "integer",
  "started_at": "ISO 8601 datetime | null",
  "completed_at": "ISO 8601 datetime | null",
  "error_message": "string | null"
}
```

---

## Authentication Details

**Mechanism:** JWT RS256 (NFR010)

**Token Claims (required):**

| Claim | Description |
|---|---|
| `sub` | Subject — user ID (UUID) |
| `tid` | Tenant ID (UUID) — extracted by api-gateway and injected into all downstream requests |
| `role` | Role — `tenant_admin` \| `staff` \| `customer` \| `platform_operator` |
| `iss` | Issuer — `https://auth.anji-schedulo.io` |
| `aud` | Audience — `https://api.anji-schedulo.io` |
| `exp` | Expiry timestamp (1-hour TTL for access tokens) |
| `iat` | Issued-at timestamp |

**Access Token TTL:** 1 hour  
**Refresh Token TTL:** 30 days  
**Public Endpoints (no JWT required):** `POST /tenants`, `GET /tenants/{tenant_id}/availability`, `GET /tenants/{tenant_id}/services`

**Enterprise SSO:** Enterprise tenants may federate identity via OIDC (Google Workspace, Azure AD). The IdP token is exchanged at the `/auth/sso/exchange` endpoint for a normalised AnjiSchedulo JWT containing the same claim structure. All downstream services are agnostic to the identity source.

---

## Rate Limits

| Tier | Requests/minute (API) | Notes |
|---|---|---|
| Starter | 60 | Per tenant |
| Pro | 300 | Per tenant |
| Enterprise | 1,000 | Per tenant, configurable |
| Public (unauthenticated) | 20 | Per IP address — availability/services reads only |

Rate limit exceeded → `429 Too Many Requests` with `X-RateLimit-Limit`, `X-RateLimit-Remaining`, and `Retry-After` headers.

---

## Versioning

The API is versioned at the URL path level (`/api/v1/`). Backwards-compatible changes (new optional fields, new endpoints) do not require a version bump. Breaking changes (removed fields, changed semantics) require a new version prefix (`/api/v2/`) with a minimum 6-month deprecation window for `v1`. API deprecation notices are communicated via the `Deprecation` and `Sunset` response headers on affected endpoints.
