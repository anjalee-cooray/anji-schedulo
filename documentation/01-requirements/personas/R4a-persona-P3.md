---
title: Persona — End Customer
personaId: P3
layer: 01-requirements
status: current
lastUpdated: 2026-06-28
---

# Persona — End Customer (P3)

| Field | Value |
|---|---|
| **Persona ID** | P3 |
| **Name** | End Customer |
| **Role** | Appointment Booker |
| **Technical Level** | Low |

---

## Who Is This Person?

Jordan is a 34-year-old project manager at a consultancy firm in a mid-sized city. They work long hours, attend back-to-back meetings, and generally handle personal admin during lunch or on the commute home. When their usual physio clinic sends a text saying the receptionist line is closed until Monday, Jordan searches on their phone and finds the clinic's booking page — built on AnjiSchedulo. Within ninety seconds, without creating an account or navigating a confusing menu, they have selected a 45-minute sports massage with their preferred therapist, picked a Thursday at 1 PM that fits between two meetings, and received a confirmation email. Jordan doesn't know or care about the platform powering the booking. They notice only that it worked, it was fast, and they didn't have to make a phone call.

When Jordan's schedule later shifts — a client deadline moves to Thursday — they open the confirmation email, click the rescheduling link, pick a new slot on Friday, and confirm in under a minute. When a family event forces a last-minute change, they cancel online rather than leaving a voicemail nobody will hear until the morning. Jordan expects the refund to appear automatically. They never want to explain who they are, what they booked, or when they called — the platform should already know.

---

## Goals

**Book an appointment in under 2 minutes.** Jordan's time is genuinely constrained. If the booking flow takes longer than a typical coffee order, they will call instead — and calling means waiting on hold or leaving a message. Speed and minimal friction are not nice-to-haves; they are the threshold between using the platform and abandoning it.

**Receive instant confirmation.** Jordan needs to trust that the booking exists. A confirmation email within thirty seconds, containing the date, time, service, and staff member name, converts a moment of uncertainty into a calendar entry they can act on. Without that confirmation, they worry whether the booking registered and will check back, or assume something went wrong.

**Cancel or reschedule without calling.** Life changes. Jordan doesn't expect appointments to be immovable, but they do expect to manage changes the same way they made the booking — online, quickly, without navigating a phone menu or waiting for a human to acknowledge a voicemail. If the only way to reschedule is to call during business hours, the platform has failed.

**Know exactly who they are seeing and when.** Jordan has multiple service providers across several businesses. Receiving a notification that says "Appointment confirmed at 1:00 PM Thursday" is not sufficient — they need the staff member's name, the service, the address, and the duration in every communication. Ambiguity leads to no-shows, which wastes everyone's time.

**No account required for a single booking.** Jordan won't create an account for a one-time appointment. The booking must be completable with just a name, email, and payment details. Account creation, if it exists, must be optional and clearly skippable — anything else adds friction that drives abandonment.

---

## Frustrations

**Calling during business hours only to find no availability.** Before platforms like AnjiSchedulo, Jordan wasted lunch breaks on hold, only to discover that the available slots didn't fit their schedule. The entire call then achieved nothing. Online self-serve availability browsing eliminates this problem entirely — but only if the availability shown is accurate and up to date.

**No confirmation received after booking.** More than once, Jordan submitted a booking form online and heard nothing. No email, no SMS, no acknowledgement. They then had to call to verify the booking existed, which eroded any benefit the online form provided. Instant confirmation, delivered reliably, is the minimum viable expectation.

**Hard to cancel without calling.** Jordan once tried to cancel a dentist appointment outside business hours. The only mechanism was a voicemail, which went unanswered. The appointment remained on the books, Jordan didn't show, and the practice sent an invoice. Self-service cancellation, available at any hour, resolves this completely.

**Showing up to find the appointment was never booked.** Jordan attended a hair appointment after booking through a business's custom form, only to find no record of their booking at the salon. The form had never actually connected to the salon's calendar. This is the worst possible outcome — wasted time, embarrassment, and zero trust in the business or the platform. Real-time confirmation that the slot is locked is the only acceptable guarantee.

---

## Primary Actions in the Platform

| Action | Functional Requirements |
|---|---|
| Browse available appointment slots by service, staff member, and date | FR003 |
| Book an appointment with contact details and optional payment | FR004 |
| Cancel a confirmed appointment within the policy window | FR005 |
| Reschedule an appointment to a new available slot | FR006 |

---

## User Journeys

| Journey ID | Name | P3's Role |
|---|---|---|
| UJ002 | Appointment Booking | Primary actor — browses slots, selects service and staff, completes contact and payment, receives confirmation |
| UJ003 | Appointment Cancellation | Primary actor — requests cancellation, receives confirmation, expects refund if applicable |
| UJ004 | Appointment Rescheduling | Primary actor — selects a new slot for an existing confirmed appointment, receives rescheduling confirmation |

---

## Needs from the Platform

In priority order:

1. **Sub-2-minute booking flow** from slot selection to confirmed appointment, with no account creation required.
2. **Instant confirmation** — confirmation email or SMS within 30 seconds of booking, containing all appointment details.
3. **Self-service cancellation and rescheduling** available at any hour without calling, within the tenant's cancellation policy window.
4. **Accurate, real-time availability** — slots shown must be genuinely available and locked the moment the booking completes.
5. **Clear, unambiguous notifications** — every communication must include service name, staff member name, date, time, and duration.
6. **Refund without intervention** — if a cancellation qualifies for a refund, it must be initiated automatically without Jordan contacting anyone.

---

## Success Looks Like

A successful interaction for Jordan is one they barely remember. They opened the booking page during a two-minute break, selected a slot, typed their email and phone number, tapped confirm, and within fifteen seconds had a calendar-ready confirmation in their inbox. The appointment is correct. The staff member's name matches who they expected. Three days before the appointment, a reminder arrives without Jordan having to request it. When their schedule changes, they click the reschedule link in the confirmation email, pick an alternative slot, and receive a new confirmation immediately. The refund from a later cancellation appears in their account within a few days, initiated automatically. Jordan never had to call anyone, explain their booking twice, or wonder whether something had worked.

---

## Main Interaction Flow

```
  END CUSTOMER — MAIN INTERACTION FLOW
  ──────────────────────────────────────────────────────────
  1. BROWSE AVAILABILITY                             [UJ002]
     Visit tenant's public booking page (no login required)
     Select a service
       → FR003: Available slots filtered by service + staff + date
     Select a staff member (or "any available")
     Choose a date and time slot
       → Availability enforced in real-time (BR001, BR003)

  2. BOOK AN APPOINTMENT                             [UJ002]
     Enter name, email, and phone number
     Complete payment if required by tenant
       → FR004: Saga: slot lock → payment → confirmation
     Receive confirmation email/SMS within 30 seconds
       → FR007: Notification with full appointment details

  3. RECEIVE REMINDERS                               [FR007]
     Automated reminder sent before appointment
       → Notification contains staff name, date, time, address

  4. MANAGE — RESCHEDULE                             [UJ004]
     Click reschedule link in confirmation email
     Select a new available slot
       → FR006: Old slot released only after new slot confirmed
     Receive rescheduling confirmation

  5. MANAGE — CANCEL                                 [UJ003]
     Request cancellation within policy window
       → FR005: Validation against tenant cancellation_hours (BR002)
     Slot released immediately
     Refund initiated automatically if applicable (BR009)
     Cancellation confirmation received within 30 seconds
  ──────────────────────────────────────────────────────────
```
