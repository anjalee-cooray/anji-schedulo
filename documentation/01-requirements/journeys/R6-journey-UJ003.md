---
title: Journey — Appointment Cancellation
journeyId: UJ003
persona: P3
priority: critical
layer: 01-requirements
status: current
lastUpdated: 2026-06-27
---

# R6 · Journey UJ003 — Appointment Cancellation

## Metadata

| Field | Value |
|---|---|
| Journey ID | UJ003 |
| Name | Appointment Cancellation |
| Primary Persona | [P3 — End Customer](../personas/R4a-persona-P3.md) |
| Priority | Critical |
| Related Functional Requirements | FR005, FR007 |
| Related Business Rules | BR002, BR007, BR009, BR013, BR014 |
| Architecture Pattern | Saga Choreography, Dead Letter Queue, Retry with Exponential Backoff, Transactional Outbox |
| Services Involved | booking-command-service, payment-service, availability-service, notification-service |

---

## Goal

A customer cancels a confirmed appointment without calling the business, receives a cancellation confirmation, and — where applicable — has their refund initiated, all within the tenant's cancellation policy window.

---

## Preconditions

- The customer holds a confirmed appointment with status `confirmed`.
- The appointment has not already been cancelled or completed.
- The customer is making the cancellation request more than the number of hours ahead of the appointment defined in the tenant's cancellation policy (`tenant_config.cancellation_hours`) (BR002).
- The platform is available and able to accept cancellation requests.

---

## Step-by-Step Journey

### Step 1 — Initiate Cancellation

The customer opens the cancellation link included in their original confirmation email, or navigates to the "My Bookings" section of the tenant's booking page and selects the appointment they wish to cancel.

They tap or click the "Cancel Appointment" button.

**Outcome:** The platform displays a cancellation review screen showing the appointment details — service, staff member, date and time — and the tenant's cancellation policy (e.g. "Free cancellation up to 24 hours before your appointment"). If a refund applies, the refund amount and method are shown.

---

### Step 2 — Confirm Cancellation

The customer confirms they want to proceed with the cancellation.

**Outcome:** The cancellation request is submitted to `booking-command-service`. The platform displays a "Cancellation in progress" indicator while the request is validated.

---

### Step 3 — Receive Cancellation Confirmation

The platform validates that the cancellation falls within the tenant's policy window (BR002) and writes an `appointment.cancelled` event to the appointment event stream and the Transactional Outbox. The customer sees an on-screen confirmation that their appointment has been cancelled.

Within 30 seconds, the customer receives a cancellation confirmation notification (email and/or SMS depending on their contact details and the tenant's plan) containing:
- The cancelled appointment details
- Confirmation of the cancellation reference
- Refund details and estimated timeline (if applicable)

**Outcome:** The customer has on-screen confirmation and a notification confirming the cancellation.

---

### Step 4 — Slot Released for Rebooking

The `availability-service` consumes the `appointment.cancelled` event and invalidates the Redis cache for the affected slot. The slot immediately becomes available for other customers to book.

This step is independent of refund processing and notification delivery — it completes as soon as the event is consumed (BR009).

**Outcome:** The freed slot is visible to other customers browsing availability within 10 seconds of the cancellation being confirmed.

---

### Step 5 — Refund Initiated (if applicable)

If a payment was captured for the appointment, `payment-service` consumes the `appointment.cancelled` event and initiates a refund via the Stripe Refunds API. The customer is not required to take any action.

The estimated refund timeline (typically 5–10 business days depending on the card issuer) is communicated in the cancellation confirmation notification.

**Outcome:** A refund has been initiated. The customer will see the credit in their account within the card issuer's standard processing window.

---

### Step 6 — Staff Member Notified

`notification-service` consumes the `appointment.cancelled` event and sends a cancellation notice to the assigned staff member, informing them that the appointment has been removed from their schedule.

**Outcome:** The staff member is informed of the gap in their schedule without any manual intervention from the tenant admin.

---

## Success Outcome

The appointment is marked cancelled. The freed slot is available for new bookings within 10 seconds. The customer receives a cancellation confirmation notification within 30 seconds. Where a payment was taken, a refund is initiated within 60 seconds of the cancellation event. The staff member receives their own cancellation notice. An immutable audit log entry records the cancellation (BR014).

---

## Failure Paths

### Cancellation Outside Policy Window

If the customer attempts to cancel within fewer hours of the appointment than the tenant's policy permits (e.g. cancelling 2 hours before when the policy requires 24 hours notice), `booking-command-service` rejects the request before writing any event. The customer sees a clear error message: "Cancellations must be made at least [N] hours before your appointment. Please contact the business directly." No slot is released, no refund is initiated, and no events are published.

### Refund Failure

If the Stripe refund call fails (e.g. the original payment method is no longer valid, or Stripe returns an error), `payment-service` publishes a `payment.refund_failed` event. This event is routed to the Dead Letter Queue for manual operator review. Critically, the slot has already been released and the customer has already received their cancellation confirmation — the refund failure does not reverse these steps (BR009). The platform operator is alerted via the DLQ monitoring alert and resolves the refund manually via the ops console within the 4-hour resolution SLA.

### Cancellation Notification Delivery Failure

If the cancellation confirmation email or SMS fails to deliver, `notification-service` retries delivery up to 4 times with exponential backoff. After 4 failures, the `NotificationRecord` is marked as `dlq` and a platform operator alert is raised. The cancellation itself remains in effect — notification failure never reverses or delays the cancellation (BR007). The customer's appointment is cancelled regardless of whether the confirmation notification reaches their inbox.

### Appointment Already Cancelled or Completed

If the customer attempts to cancel an appointment that is already in a `cancelled` or `completed` state, `booking-command-service` returns a clear error response. The customer sees a message indicating the appointment cannot be cancelled because it is no longer active. No duplicate events are published and no refund is processed.

---

## Related Business Rules

| Rule ID | Rule Summary | Application in This Journey |
|---|---|---|
| BR002 | A customer may only cancel within the tenant's configured cancellation policy window. | `booking-command-service` validates the cancellation request against `tenant_config.cancellation_hours` before processing. Requests outside the window are rejected with a clear error; no event is written. |
| BR007 | A notification delivery failure must never block, delay, or roll back the booking operation. | The `notification-service` is a fully decoupled async subscriber to `appointment.cancelled`. Notification failures are isolated to the notification retry pipeline and do not affect slot release or refund initiation. |
| BR009 | When a cancellation triggers a refund, the refund failure must not prevent slot release or customer notification. | The cancellation saga uses Choreography — slot release, refund, and notification are independent subscribers to the same `appointment.cancelled` event. `payment.refund_failed` routes to DLQ without blocking the other subscribers. |
| BR013 | Every event must carry `tenant_id`, `event_id`, `correlation_id`, and `occurred_at`. | The `appointment.cancelled` event envelope is validated by all consuming services before processing. Any event missing required fields is routed to the DLQ rather than processed. |
| BR014 | All write operations that modify tenant-scoped data must produce an immutable audit log entry. | `booking-command-service` writes an `audit_log` record when the `appointment.cancelled` event is persisted. This record is INSERT-only and cannot be modified. |

---

## Implementation Note

When `booking-command-service` validates and accepts the cancellation request, it writes an `appointment.cancelled` AppointmentEvent and an `OutboxRecord` in a single database transaction (Transactional Outbox pattern). The `outbox-relay` publishes the event to the `appointment-cancelled` SNS topic, which fans out via separate SQS FIFO queues to three independent subscribers: `availability-service` (invalidates the Redis slot cache, freeing the slot for new bookings), `payment-service` (initiates the Stripe refund if a payment was captured), and `notification-service` (sends cancellation confirmations to the customer and the staff member). Each subscriber operates independently using the Idempotent Consumer pattern via `inbox_records`, ensuring that at-least-once delivery from SQS does not result in duplicate refunds or duplicate notifications. A refund failure in `payment-service` produces a `payment.refund_failed` event that routes to the DLQ without affecting the other two subscribers, satisfying the independence guarantee of BR009.
