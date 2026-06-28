---
title: Journey — Appointment Booking
journeyId: UJ002
persona: P3
priority: critical
layer: 01-requirements
status: current
lastUpdated: 2026-06-27
---

# R6 · Journey UJ002 — Appointment Booking

## Metadata

| Field | Value |
|---|---|
| Journey ID | UJ002 |
| Name | Appointment Booking |
| Primary Persona | [P3 — End Customer](../personas/R4a-persona-P3.md) |
| Priority | Critical |
| Related Functional Requirements | FR003, FR004, FR007 |
| Related Business Rules | BR001, BR003, BR004, BR006, BR007, BR008, BR010 |
| Architecture Pattern | Saga Orchestration, Transactional Outbox, Idempotent Consumer, Inbox Pattern, Event-Carried State Transfer |
| Services Involved | booking-command-service, availability-service, payment-service, notification-service |

---

## Goal

A customer finds an available appointment slot with a business on AnjiSchedulo, provides their contact details, completes payment where required, and receives a confirmed appointment — all in under 2 minutes.

---

## Preconditions

- The tenant has completed onboarding (UJ001): business hours, at least one service, and at least one staff member are configured.
- The tenant's booking page is live and publicly accessible.
- The requested date falls within the tenant's configured booking advance window (BR010).
- The customer has not exceeded the tenant's maximum active bookings limit (BR008).
- The customer has a valid payment method if the chosen service requires payment at booking.

---

## Step-by-Step Journey

### Step 1 — Visit the Booking Page

The customer navigates to the tenant's public booking page, either via a direct link shared by the business or via a booking widget embedded on the business's website.

**Outcome:** The customer sees the business name, available services, and any introductory information configured by the tenant admin.

---

### Step 2 — Select a Service

The customer browses the service catalogue and selects the service they want to book (e.g. "30-Minute Consultation", "Standard Haircut").

**Outcome:** The booking flow advances to staff selection, filtered to staff who deliver the chosen service.

---

### Step 3 — Select a Staff Member

The customer chooses a specific staff member from the list, or selects "Any available" to let the platform assign the first available provider.

**Outcome:** The booking flow advances to date and time selection, showing only slots that fall within that staff member's configured working hours (BR003).

---

### Step 4 — Choose a Date and Time Slot

The customer sees a calendar view of available slots. Slots are shown only within the tenant's booking advance window (BR010). Already-booked slots and periods outside working hours are hidden.

The customer selects their preferred date and time.

**Outcome:** The customer's desired slot is selected. The platform displays the service, staff member, date, and time for review before commitment.

---

### Step 5 — Enter Contact Details

The customer provides:
- Full name
- Email address
- Phone number (if required by the tenant)

**Outcome:** Contact details are captured. The booking flow advances to payment if the service has a price, or directly to confirmation if no payment is required.

---

### Step 6 — Complete Payment (if applicable)

If the selected service has a price set by the tenant, the customer is shown a payment screen and enters their card details. The platform handles payment via a secure hosted payment form.

If no payment is required, this step is skipped.

**Outcome:** Payment is submitted. The platform processes the booking request atomically: it checks slot availability, captures payment, and confirms the appointment in sequence. The customer sees a "Booking in progress" indicator while this completes.

---

### Step 7 — Receive Confirmation

Within 30 seconds of the booking completing, the customer receives a confirmation notification containing:
- Booking reference number
- Service name
- Staff member name
- Date and time of the appointment
- Business address or meeting link
- Cancellation and rescheduling instructions

The booking page also displays an on-screen confirmation summary.

**Outcome:** The customer has a confirmed appointment and a confirmation in their inbox. The booking is complete.

---

## Success Outcome

The customer holds a confirmed appointment. The confirmation notification is delivered within 30 seconds. The end-to-end booking process completes in under 2 minutes from the moment the customer lands on the booking page. The slot is locked and unavailable to other customers. The staff member has received their own booking notification.

---

## Failure Paths

### Slot No Longer Available (Double-Booking Attempt)

If the slot the customer selected is taken by another customer between Step 4 and Step 6 (a race condition under concurrent load), the booking saga detects the conflict when attempting to acquire the slot lock. The booking fails immediately. The customer receives a clear message — "This slot is no longer available. Please select another time." — and is returned to the slot selection screen. No payment is taken (BR001, BR004).

### Payment Failed

If the customer's payment is declined or the payment provider returns an error, the slot reservation is released immediately and the booking fails with a payment error message. The customer is invited to try a different card or contact the business. No appointment is created and the slot becomes available for other customers (BR004).

### Payment Provider Unavailable

If the payment service is unreachable (circuit breaker is open), the booking saga returns a 503 response. The customer sees a "Booking temporarily unavailable — please try again in a moment" message. No slot is held and no charge is attempted. The customer can retry when the service recovers.

### Customer Exceeds Active Booking Limit

If the customer already holds the maximum number of active bookings permitted by the tenant configuration (BR008), the booking is rejected before any payment attempt. The customer sees a message indicating the limit has been reached and is advised to cancel an existing booking before making a new one.

### Duplicate Booking Request (Network Retry)

If the customer's browser retries the booking request due to a network issue, the platform detects the duplicate via the idempotency key (BR006). The original booking result is returned — either the confirmed appointment or the original failure reason — without creating a second appointment or charging a second payment.

### Notification Delivery Failure

If the confirmation email or SMS fails to deliver, the notification pipeline retries up to 4 times with exponential backoff. After 4 failures, the notification is routed to the DLQ and a platform operator alert is raised. The booking itself remains confirmed — notification failure never cancels or delays the appointment (BR007).

---

## Related Business Rules

| Rule ID | Rule Summary | Application in This Journey |
|---|---|---|
| BR001 | A slot cannot be booked by more than one customer simultaneously. | The booking saga acquires a unique slot lock in PostgreSQL before confirming. A second request for the same slot is rejected with a 409 conflict at the database constraint level. |
| BR003 | Staff cannot be booked outside their configured working hours. | Available slot computation in Step 4 only surfaces slots within `tenant_working_hours` for the selected staff member. |
| BR004 | The booking saga must be atomic — all steps succeed or all compensations apply. | If availability check succeeds but payment fails, the slot lock is released. If payment succeeds but confirmation write fails, a void is issued. The customer never ends up in a partially booked state. |
| BR006 | Duplicate booking requests with the same idempotency key must produce exactly one confirmed appointment. | `booking-command-service` inserts an idempotency record before processing. Duplicate keys return the cached result without reprocessing. |
| BR007 | A notification delivery failure must never block, delay, or roll back the booking operation. | The notification service is a fully decoupled async subscriber. The booking saga completes independently of notification delivery. |
| BR008 | A customer may not have more active bookings than the tenant's `max_advance_bookings` limit. | `booking-command-service` validates the customer's active booking count before initiating the saga. |
| BR010 | Appointments may only be booked within the tenant's configured booking window. | The availability service excludes slots beyond `tenant_config.booking_window_days` from the slots shown in Step 4. |

---

## Implementation Note

The booking saga is orchestrated synchronously by `booking-command-service`, which calls `availability-service` to acquire a slot lock and `payment-service` to capture payment in sequence. On any failure the orchestrator applies compensation inline — releasing the slot lock or voiding the payment — and returns a clear HTTP status code (409 for slot conflict, 402 for payment failure, 503 for service unavailability) within the same request. On success, `booking-command-service` writes an `AppointmentEvent` and an `OutboxRecord` for `appointment.confirmed` in a single database transaction. The `outbox-relay` publishes the event to SNS, which fans out via SQS to `notification-service` (customer and staff confirmations), `dashboard-service` (materialised view update), and `analytics-service` (booking velocity window update). All downstream consumers use the Idempotent Consumer pattern via `inbox_records` to guard against duplicate delivery under at-least-once SQS semantics.
