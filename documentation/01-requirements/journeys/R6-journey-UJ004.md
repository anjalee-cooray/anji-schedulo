---
title: Journey — Appointment Rescheduling
journeyId: UJ004
persona: P3
priority: high
layer: 01-requirements
status: current
lastUpdated: 2026-06-28
---

# R6 · Journey UJ004 — Appointment Rescheduling

## Metadata

| Field | Value |
|---|---|
| Journey ID | UJ004 |
| Name | Appointment Rescheduling |
| Primary Persona | [P3 — End Customer](../personas/R4a-persona-P3.md) |
| Priority | High |
| Related Functional Requirements | FR006 |
| Related Business Rules | BR001, BR002, BR004 |
| Architecture Pattern | Saga Orchestration (sequential: new slot saga → release old slot) |
| Services Involved | booking-command-service, availability-service, payment-service, notification-service |

---

## Goal

A customer moves an existing confirmed appointment to a new available time slot — without losing their original booking if the desired new slot is taken, and without the platform allowing even a brief moment of double-booking on either the old or the new slot.

---

## Preconditions

- The customer holds a confirmed appointment with status `confirmed`.
- The desired new slot is within the tenant's booking window (BR010) and falls within the staff member's working hours (BR003).
- The current appointment's original slot time has not yet passed.
- If a cancellation policy applies, the original appointment must be within the policy window for the old slot to be released (BR002).

---

## Step-by-Step Journey

### Step 1 — Request a New Time

The customer navigates to their confirmed appointment — typically via a link in the original confirmation email — and selects the option to reschedule. They are presented with the same availability browsing interface used for a new booking, pre-filtered to the original service and (optionally) the same staff member.

**Outcome:** The customer sees available slots for the same service on future dates.

---

### Step 2 — Select a New Slot

The customer browses available dates and times and selects their preferred new slot.

**Outcome:** The customer has nominated a new slot. No reservation has been made yet — the original booking remains fully intact.

---

### Step 3 — System Confirms New Slot Availability

Before releasing the original slot, the platform verifies that the chosen new slot is genuinely available at the moment of request. The booking-command-service initiates the new-slot saga: it attempts to place a SlotLock on the new slot under the unique constraint on `(tenant_id, staff_id, slot_date, slot_start)`.

If the new slot is taken by a concurrent booking between the customer selecting it and the platform attempting to lock it, the lock attempt fails and the rescheduling request is rejected with a clear error — the original booking remains unchanged.

**Outcome:** The new slot is confirmed available and temporarily locked, or the customer is informed the slot is no longer available.

---

### Step 4 — Release the Original Slot

Only after the new slot lock is successfully acquired does the platform release the original slot. The booking-command-service writes an `AppointmentEvent(rescheduled)` and atomically releases the original `SlotLock` in the same database transaction.

**Outcome:** The original slot is freed and immediately available for new bookings. The appointment record reflects the new date and time.

---

### Step 5 — Confirm the Rescheduled Appointment

The platform publishes an `appointment.rescheduled` event via the Transactional Outbox. This event carries both the original slot details and the new slot details, so all downstream consumers can update their views without a back-call.

**Outcome:** The appointment is confirmed at the new time. The rescheduled event is in the outbox awaiting publication.

---

### Step 6 — Customer Receives Rescheduling Confirmation

The notification-service consumes the `appointment.rescheduled` event and sends a confirmation notification to the customer containing:
- Service name
- New date and time
- Staff member name
- Duration

**Outcome:** The customer has a written record of the new appointment. They do not need to verify or follow up.

---

## Success Outcome

The customer's appointment is confirmed at the new time. The original slot is freed for new bookings. The customer has received a rescheduling confirmation containing the new date, time, staff member, and service. No double-booking occurred on either the original or new slot at any point during the process.

---

## Failure Paths

### New Slot Unavailable at Confirmation Time

If the chosen new slot is taken between the customer selecting it and the platform attempting the lock, the lock fails with a unique constraint violation. The platform returns a clear error to the customer explaining that the slot was just taken and they should choose another. The original booking is completely unchanged.

**Platform behaviour:** 409 Conflict response. No state change. Customer remains confirmed on original slot.

### Rescheduling Attempted Outside Cancellation Window

If the tenant has configured a cancellation policy that would block release of the original slot — for example, the appointment is within 24 hours and the policy requires 24 hours' notice — the rescheduling request is rejected before any slot lock is attempted on the new slot.

**Platform behaviour:** 422 Unprocessable Entity. Clear error indicating the cancellation window restriction. No state change.

### New Slot Becomes Unavailable Between Lock and Commit

If the database transaction fails after the new slot lock is acquired but before the old SlotLock is released (e.g. a database write error), the transaction rolls back atomically. The new SlotLock is never committed, and the original booking remains intact.

**Platform behaviour:** 503 Service Unavailable with retry guidance. No partial state.

### Rescheduling Confirmation Notification Fails

If the notification-service fails to deliver the rescheduling confirmation after 4 retry attempts, the failure is routed to the DLQ and an ops alert is raised. The rescheduling itself is already complete — the appointment is correctly recorded at the new time. The notification failure is fully isolated from the booking operation (BR007).

**Platform behaviour:** Appointment is rescheduled correctly. Operator resolves notification DLQ. Customer may need to be contacted manually in extreme cases.

---

## Related Business Rules

| Rule ID | Rule Summary | Application in This Journey |
|---|---|---|
| BR001 | A slot cannot be booked by more than one customer simultaneously. | The new slot lock uses the same unique constraint on `(tenant_id, staff_id, slot_date, slot_start)` as a new booking. If two customers attempt to reschedule to the same slot concurrently, only one succeeds. |
| BR002 | Cancellation is only permitted within the tenant's policy window. | The release of the original slot is gated on the same cancellation window check applied to a direct cancellation. Rescheduling close to the appointment time may be rejected if outside the policy window. |
| BR004 | A booking saga must be atomic — either all steps succeed or all compensations are applied. | The rescheduling saga follows the same atomicity guarantee: new slot lock → release old SlotLock → write AppointmentEvent(rescheduled) must all succeed in a single transaction, or none commit. |

---

## Implementation Note

Rescheduling is implemented as a sequential saga: the new-slot saga runs first and must succeed before the original slot is released. This prevents the customer from ever being left without a confirmed booking. Inside a single PostgreSQL transaction, booking-command-service acquires the SlotLock on the new slot (unique constraint enforces BR001), writes `AppointmentEvent(rescheduled)` carrying `old_slot_date`, `old_slot_start`, `new_slot_date`, `new_slot_start`, and releases `SlotLock` on the original slot by setting `released_at = NOW()`. The `appointment.rescheduled` event published via the Transactional Outbox carries both old and new slot details in its payload, enabling downstream consumers (notification-service, dashboard-service, availability-service) to derive the correct state without back-calls.
