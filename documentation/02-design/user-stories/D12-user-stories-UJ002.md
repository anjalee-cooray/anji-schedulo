---
title: User Stories — Appointment Booking
journeyId: UJ002
layer: 02-design
status: current
lastUpdated: 2026-06-28
---

# D12 · User Stories — Appointment Booking (UJ002)

## Epic Header

| Field | Value |
|---|---|
| Epic Name | Appointment Booking |
| Journey ID | UJ002 |
| Primary Persona | P3 — End Customer (Appointment Booker) |
| Related FRs | FR003, FR004 |
| Related BRs | BR001, BR004, BR006, BR007, BR008, BR010, BR013 |
| Priority | Critical |
| Architecture Pattern | Saga Orchestration, Transactional Outbox, Idempotent Consumer, Inbox Pattern, Event-Carried State Transfer |

## Epic Description

As an **End Customer**, I want to browse a business's available slots, book an appointment with a single form submission, and receive an instant confirmation — so that I can secure my appointment in under 2 minutes without making a phone call.

---

## User Stories

---

### US-UJ002-01 · Browse available appointment slots

**Title:** Query available slots by service, staff member, and date

**User Story:**
As an **End Customer**, I want to browse available time slots for a specific service and staff member on a given date, so that I can choose an appointment time that fits my schedule before committing to a booking.

**Acceptance Criteria:**

- **Given** I am on the tenant's public booking page,  
  **When** I select a service and a date,  
  **Then** the platform returns available time slots within 500ms (FR003).

- **Given** I browse available slots,  
  **When** a slot already has a confirmed booking,  
  **Then** that slot does not appear in the results — it is never shown as available (BR001).

- **Given** I browse slots for a date outside the tenant's booking window,  
  **When** the request is evaluated,  
  **Then** no slots are returned and the response indicates the date is outside the bookable range (BR010).

- **Given** a staff member has a StaffBlock covering the requested date,  
  **When** I browse slots for that staff member on that date,  
  **Then** no slots are shown for them on that day (BR003).

- **Given** I browse slots without a registered account,  
  **When** the request reaches the availability-service,  
  **Then** availability is returned without requiring authentication — the endpoint is public (FR003).

**Size:** S  
**Priority:** Critical  
**Related:** FR003, BR001, BR003, BR010

**Technical Notes:**
- availability-service caches computed slot results in Redis (ElastiCache) with a 30-second TTL; cache key format: `availability:{tenant_id}:{staff_id}:{date}`.
- Cache is invalidated on `appointment.confirmed` (slot no longer available) and `appointment.cancelled` (slot re-surfaced) events via SQS subscription.
- Slot computation: WorkingHours MINUS StaffBlocks MINUS existing SlotLocks WHERE released_at IS NULL.
- No JWT required on `GET /tenants/{tenantSlug}/availability`; tenant_id is derived from the URL slug.

---

### US-UJ002-02 · Book an appointment with an idempotency key

**Title:** Submit a booking request that is atomic and safe to retry

**User Story:**
As an **End Customer**, I want to submit a booking request with my chosen service, staff member, time slot, and contact details — so that my appointment is confirmed, my payment is captured, and I receive proof of booking, all as a single atomic operation.

**Acceptance Criteria:**

- **Given** I submit a `POST /tenants/{tenantSlug}/bookings` request with a unique `idempotency_key`, a valid slot, and valid contact details,  
  **When** the slot is available and payment captures successfully,  
  **Then** the platform returns HTTP 201 with the confirmed appointment details within 2 seconds (FR004).

- **Given** I submit a booking request with the same `idempotency_key` a second time (e.g. due to a network retry),  
  **When** the second request arrives at booking-command-service,  
  **Then** the stored result from the first request is returned without creating a duplicate appointment or charging the card again (BR006).

- **Given** I select a slot that becomes unavailable between browsing and booking,  
  **When** the SlotLock INSERT fails due to the unique partial index,  
  **Then** I receive HTTP 409 Conflict with a clear `slot_unavailable` message, and my card is not charged (BR001, BR004).

- **Given** payment is required for the service and my card is declined,  
  **When** the payment capture step in the saga fails,  
  **Then** the SlotLock is released atomically, I receive HTTP 402 with `payment_failed` and no appointment is created (BR004).

- **Given** the booking saga completes all steps successfully,  
  **When** the `appointment.confirmed` event is published to the Outbox,  
  **Then** the event envelope contains `tenant_id`, `event_id`, `correlation_id`, and `occurred_at` (BR013).

- **Given** I already have the maximum number of active bookings for this tenant,  
  **When** I submit a new booking request,  
  **Then** the platform returns HTTP 422 with a clear message indicating the booking limit has been reached (BR008).

**Size:** L  
**Priority:** Critical  
**Related:** FR004, BR001, BR004, BR006, BR008, BR013

**Technical Notes:**
- SlotLock INSERT uses a unique partial index on `(tenant_id, staff_id, slot_date, slot_start) WHERE released_at IS NULL` to enforce BR001 at the database level.
- BookingIdempotencyKey record is written before SlotLock INSERT; stores `result_status` and `result_payload` so any subsequent identical request returns the cached outcome within 24 hours of the original.
- Saga orchestrated by booking-command-service: (1) idempotency check → (2) BR008 check → (3) SlotLock INSERT → (4) payment capture via payment-service → (5) AppointmentEvent(confirmed) + OutboxRecord in one transaction.
- `Customer.active_booking_count` is incremented atomically on saga completion and decremented on cancellation confirmation.

---

### US-UJ002-03 · Receive booking confirmation notification

**Title:** Receive an email confirmation within 30 seconds of a successful booking

**User Story:**
As an **End Customer**, I want to receive a confirmation email immediately after my booking is confirmed, so that I have a record of my appointment details and can be confident the booking was successful.

**Acceptance Criteria:**

- **Given** my booking is confirmed and `appointment.confirmed` is published,  
  **When** notification-service processes the event from its SQS queue,  
  **Then** a confirmation email is delivered to my registered email address within 30 seconds (FR004, FR007).

- **Given** the confirmation email is delivered,  
  **When** I open it,  
  **Then** it contains the staff member's name, service name, appointment date, start time, and end time — all derived directly from the event payload without a back-call to booking-command-service.

- **Given** the notification delivery fails on the first attempt,  
  **When** notification-service retries with exponential backoff (delays: 1s, 2s, 4s, 8s),  
  **Then** the confirmed appointment and the slot reservation remain intact throughout all retry attempts (BR007).

- **Given** all 4 delivery retry attempts are exhausted,  
  **When** notification-service routes the `NotificationRecord` to DLQ status,  
  **Then** the confirmed appointment is unaffected — booking correctness is fully independent of notification delivery (BR007).

**Size:** S  
**Priority:** High  
**Related:** FR004, FR007, BR007, BR013

**Technical Notes:**
- Event-Carried State Transfer: the `appointment.confirmed` payload includes `customer_snapshot`, `staff_snapshot`, `service_snapshot`, `slot_date`, `slot_start`, `slot_end` — notification-service never calls back to booking-command-service.
- InboxRecord deduplication in notification-service: `INSERT INTO inbox_records (event_id, consumer_name='notification-service')` before processing; duplicate events are discarded silently.
- Notification delivery channel (email/SMS/WhatsApp) is gated by the tenant's plan tier as encoded in the event payload.
- NotificationRecord is written with `status = sent` or `status = failed` after each attempt; `attempt_count` is incremented on each failure.

---

### US-UJ002-04 · Enforce per-customer booking limit

**Title:** Prevent a customer from holding more than the tenant-configured maximum of active bookings

**User Story:**
As a **Platform** enforcing tenant configuration, I want to reject booking requests from customers who already hold the maximum number of active bookings, so that no single customer can monopolise staff availability for other customers.

**Acceptance Criteria:**

- **Given** a tenant has configured `max_advance_bookings = 3`,  
  **When** a customer with 3 active confirmed bookings submits a new booking request,  
  **Then** the platform returns HTTP 422 with `booking_limit_exceeded` and a human-readable explanation of the limit (BR008).

- **Given** a customer cancels one of their active bookings,  
  **When** the cancellation saga completes and `Customer.active_booking_count` is decremented,  
  **Then** the customer can immediately submit a new booking request without hitting the limit (BR008).

- **Given** `max_advance_bookings` is updated by the tenant admin,  
  **When** the `tenant.configured` event is consumed by booking-command-service,  
  **Then** subsequent booking requests evaluate against the new limit — in-flight bookings are not retroactively cancelled.

**Size:** S  
**Priority:** High  
**Related:** FR004, BR008

**Technical Notes:**
- `Customer.active_booking_count` is a denormalised integer column incremented and decremented atomically by booking-command-service in the same transaction as the saga outcome.
- The booking limit check occurs before SlotLock INSERT, so a rejected request never holds a slot or triggers a payment attempt.
- The `tenant.configured` event triggers a local cache refresh in booking-command-service; the updated `max_advance_bookings` is applied immediately to new requests.

---

## Definition of Ready

Before any story in this epic enters a sprint, confirm all items below are checked:

- [ ] Acceptance criteria reviewed and agreed by Product Owner
- [ ] `appointment.confirmed`, `appointment.requested`, and `appointment.failed` event schemas finalised in `specs/ai/events.json`
- [ ] SlotLock unique partial index verified in local PostgreSQL environment (BR001)
- [ ] BookingIdempotencyKey table schema confirmed — 24-hour TTL and cleanup job defined (BR006)
- [ ] Payment saga step confirmed with payment-service team — circuit breaker behaviour documented
- [ ] Availability cache key format and invalidation triggers confirmed with availability-service team
- [ ] Booking limit enforcement logic confirmed — `Customer.active_booking_count` increment/decrement transaction boundaries agreed (BR008)
- [ ] Notification delivery SLA of 30 seconds confirmed achievable with SendGrid plan tier
- [ ] Test data and seed scripts available for local development (tenants with various plan configs)
- [ ] Definition of Done agreed for this layer (see `documentation/00-governance/G5-definition-of-done.md`)
