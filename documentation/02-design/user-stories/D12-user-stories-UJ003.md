---
title: User Stories — Appointment Cancellation
journeyId: UJ003
layer: 02-design
status: current
lastUpdated: 2026-06-28
---

# D12 · User Stories — Appointment Cancellation (UJ003)

## Epic Header

| Field | Value |
|---|---|
| Epic Name | Appointment Cancellation |
| Journey ID | UJ003 |
| Primary Persona | P3 — End Customer; P2 — Staff Member (secondary actor) |
| Related FRs | FR005 |
| Related BRs | BR002, BR007, BR009, BR013, BR014 |
| Priority | Critical |
| Architecture Pattern | Saga Choreography, Dead Letter Queue, Retry + Backoff |

## Epic Description

As an **End Customer or Staff Member**, I want to cancel a confirmed appointment within the policy window — so that the slot is released immediately for new bookings, any payment is refunded automatically, and all affected parties are notified without requiring manual intervention.

---

## User Stories

---

### US-UJ003-01 · Cancel an appointment within the policy window

**Title:** Successfully cancel a confirmed appointment and release the slot immediately

**User Story:**
As an **End Customer**, I want to cancel my confirmed appointment before the cancellation deadline, so that the slot is freed for other customers and I receive confirmation that my cancellation was accepted.

**Acceptance Criteria:**

- **Given** I have a confirmed appointment and the current time is within the tenant's cancellation window,  
  **When** I submit a cancellation request,  
  **Then** the platform returns HTTP 200 within 2 seconds and the appointment status reflects cancelled.

- **Given** my cancellation is accepted,  
  **When** the `appointment.cancelled` event is published and the SlotLock is released,  
  **Then** the slot becomes available for new bookings within 10 seconds.

- **Given** my cancellation is confirmed,  
  **When** I view my appointment history,  
  **Then** the appointment record shows status `cancelled` with the `cancelled_at` timestamp preserved — the full event history is retained (BR014).

- **Given** the cancellation request is valid,  
  **When** booking-command-service writes `AppointmentEvent(cancelled)`, releases the SlotLock, and inserts the OutboxRecord,  
  **Then** all three writes occur in a single PostgreSQL transaction — there is no state where the event is published but the slot is not yet released.

**Size:** M  
**Priority:** Critical  
**Related:** FR005, BR002, BR013, BR014

**Technical Notes:**
- booking-command-service validates the cancellation window using `NOW() + cancellation_hours <= slot_start` before processing.
- SlotLock `released_at` is set to `NOW()` in the same transaction as `AppointmentEvent(cancelled)` INSERT and `OutboxRecord` INSERT.
- The `appointment.cancelled` event fans out to five consumers via SNS: payment-service, notification-service, dashboard-service, availability-service, analytics-service.
- `Customer.active_booking_count` is decremented atomically in the same transaction.

---

### US-UJ003-02 · Reject a cancellation outside the policy window

**Title:** Enforce the tenant's cancellation policy by rejecting late cancellation requests

**User Story:**
As a **Platform** enforcing tenant business rules, I want to reject cancellation requests submitted after the policy deadline has passed, so that tenants are protected from last-minute revenue loss.

**Acceptance Criteria:**

- **Given** my appointment starts in fewer hours than the tenant's `cancellation_hours` setting,  
  **When** I submit a cancellation request,  
  **Then** the platform returns HTTP 422 with a `cancellation_window_expired` error and the exact policy window in the response body (BR002).

- **Given** the rejection is returned,  
  **When** no processing occurs,  
  **Then** the slot remains confirmed, no refund is initiated, and no `appointment.cancelled` event is published — the appointment state is unchanged.

- **Given** the tenant admin updates their `cancellation_hours` policy,  
  **When** the `tenant.configured` event is consumed by booking-command-service,  
  **Then** subsequent cancellation requests evaluate against the updated policy.

**Size:** S  
**Priority:** Critical  
**Related:** FR005, BR002

**Technical Notes:**
- Policy check: `appointment.slot_start - NOW() < TenantConfig.cancellation_hours` results in 422 rejection.
- No DB writes occur on a rejected cancellation — the check is purely in-memory using the cached TenantConfig.
- The `cancellation_hours` value is refreshed in booking-command-service on every `tenant.configured` event via InboxRecord deduplication.

---

### US-UJ003-03 · Trigger a refund automatically on cancellation

**Title:** Initiate and confirm a payment refund as part of the cancellation choreography

**User Story:**
As an **End Customer**, I want a refund to be initiated automatically when I cancel within the policy window and a payment was captured, so that I do not have to contact the business to recover my money.

**Acceptance Criteria:**

- **Given** my appointment had a captured payment and I cancel within the policy window,  
  **When** payment-service consumes the `appointment.cancelled` event,  
  **Then** a refund is initiated via Stripe within 60 seconds of the cancellation event being published (FR005).

- **Given** a refund is initiated and Stripe confirms it,  
  **When** payment-service writes `Payment.status = refunded`,  
  **Then** a `payment.refunded` event is published and a refund confirmation notification is sent to me.

- **Given** the Stripe refund API returns an error after 4 retry attempts,  
  **When** payment-service routes the failure to the DLQ,  
  **Then** my cancellation is still confirmed, the slot is still released, and I have still been notified of the cancellation — only the refund is pending manual resolution (BR009).

- **Given** the refund fails and routes to DLQ,  
  **When** the Platform Operator inspects the DLQ entry,  
  **Then** the entry contains the `payment_id`, `stripe_payment_intent_id`, `refund_amount`, and `failure_reason` needed for manual Stripe resolution.

**Size:** M  
**Priority:** Critical  
**Related:** FR005, BR009

**Technical Notes:**
- payment-service subscribes to `appointment.cancelled`; before attempting a refund it checks `Payment.status = captured` to avoid double-refund attempts (idempotency safety).
- Refund choreography is fully independent: slot release and customer notification proceed regardless of refund outcome (BR009).
- `payment.refund_failed` is published after 4 SQS retries exhaust; it routes to `payment-service-appointment-cancelled-queue-dlq` for ops triage.
- Stripe Refunds API call uses `idempotency_key = appointment_id` to prevent duplicate refunds on retry.

---

### US-UJ003-04 · Notify staff member of cancellation

**Title:** Automatically inform the assigned staff member when their appointment is cancelled

**User Story:**
As a **Staff Member**, I want to receive an immediate notification when an appointment on my calendar is cancelled, so that I know my schedule has changed without checking the system manually.

**Acceptance Criteria:**

- **Given** a customer cancels their appointment,  
  **When** notification-service processes the `appointment.cancelled` event,  
  **Then** the assigned staff member receives a cancellation notification within 30 seconds (FR007).

- **Given** the staff notification is delivered,  
  **When** I open it,  
  **Then** it contains the customer's name, service name, original appointment date and time — all from the event payload with no back-call.

- **Given** the notification delivery to the staff member fails,  
  **When** notification-service retries up to 4 times,  
  **Then** the cancellation confirmation, slot release, and refund processing are not affected (BR007).

**Size:** S  
**Priority:** High  
**Related:** FR005, FR007, BR007

**Technical Notes:**
- notification-service derives staff contact details from `staff_snapshot` in the `appointment.cancelled` event payload (Event-Carried State Transfer).
- InboxRecord deduplication ensures notification-service does not send duplicate staff notifications if the event is delivered twice by SQS.
- Staff notification channel (email vs SMS) is gated by tenant plan tier as encoded in the event payload.

---

### US-UJ003-05 · Preserve immutable audit trail for every cancellation

**Title:** Record an immutable audit log entry on every cancellation write operation

**User Story:**
As a **Platform** meeting compliance requirements, I want every cancellation operation to produce an immutable audit record, so that any cancellation can be forensically traced by its actor, timestamp, and full appointment snapshot.

**Acceptance Criteria:**

- **Given** any cancellation request that results in a state change,  
  **When** the `AppointmentEvent(cancelled)` is written,  
  **Then** the event record is INSERT-only and includes `cancelled_by_role`, `cancelled_by_id`, `occurred_at`, and the full appointment snapshot in the payload (BR014).

- **Given** the `AuditLog` entry is written,  
  **When** any application or operator attempts to UPDATE or DELETE it,  
  **Then** the database rejects the operation — the `booking_app_user` role has INSERT-only privileges on `audit_log` (BR014).

- **Given** the tenant is later deprovisioned,  
  **When** tenant data is hard-deleted after the 90-day retention window,  
  **Then** the `AuditLog` entries for all cancellations are retained permanently — they are not deleted with tenant data (BR012).

**Size:** S  
**Priority:** High  
**Related:** BR014, BR012

**Technical Notes:**
- `AuditLog` INSERT occurs in the same database transaction as `AppointmentEvent(cancelled)` and SlotLock update.
- `before_snapshot` contains the full Appointment row at the time of the request; `after_snapshot` contains the cancelled state.
- `audit_log` table uses a separate PostgreSQL schema partition that survives tenant deprovisioning.

---

## Definition of Ready

Before any story in this epic enters a sprint, confirm all items below are checked:

- [ ] Acceptance criteria reviewed and agreed by Product Owner
- [ ] `appointment.cancelled` and `payment.refund_failed` event schemas finalised in `specs/ai/events.json`
- [ ] Cancellation policy window validation logic confirmed — exact timestamp comparison formula agreed (BR002)
- [ ] Choreography sequence confirmed: which consumer does what in which order, and what fails independently (BR009)
- [ ] Stripe Refunds API idempotency key strategy agreed with payment-service team
- [ ] DLQ retry configuration confirmed: `maxReceiveCount = 4` on `payment-service-appointment-cancelled-queue`
- [ ] InboxRecord deduplication table schemas verified for all five consumers of `appointment.cancelled`
- [ ] AuditLog INSERT-only privilege confirmed for `booking_app_user` in PostgreSQL role setup (BR014)
- [ ] Test scenarios cover: policy window rejection, refund DLQ path, slot-release-before-refund ordering (BR009)
- [ ] Definition of Done agreed for this layer (see `documentation/00-governance/G5-definition-of-done.md`)
