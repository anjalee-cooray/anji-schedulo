---
title: Persona — Staff Member
personaId: P2
layer: 01-requirements
status: current
lastUpdated: 2026-06-27
---

# Persona — Staff Member (P2)

| Field | Value |
|---|---|
| **Persona ID** | P2 |
| **Name** | Staff Member |
| **Role** | Service Provider |
| **Technical Level** | Low |

---

## Who Is This Person?

Marcus is a physiotherapist employed at a sports rehabilitation clinic that runs on AnjiSchedulo. He sees eight to ten patients a day across a combination of initial assessments, follow-up sessions, and discharge appointments. Marcus does not manage the clinic's scheduling software — that is his practice manager's domain — but the platform touches his working day at every turn. He checks his upcoming appointments first thing each morning before the first patient arrives, blocks out his diary for a continuing education course next Thursday through his staff profile, and relies on the platform to alert him the moment a patient cancels so he can decide whether to use the gap for admin work or take an early break. What Marcus cares about is simple: knowing his schedule with certainty, finding out about changes the moment they happen, and being able to manage his own availability without routing everything through the front desk. He is not technical and has no interest in configuration panels or reports. His interaction with AnjiSchedulo is almost entirely read-only and notification-driven.

---

## Goals

- **Know the schedule without calling the front desk.** Marcus starts every morning by reviewing his confirmed appointment list for the day. Before AnjiSchedulo, this meant a phone call or a walk to the reception desk, both of which interrupted the morning flow. He needs a reliable, always-current view of his upcoming appointments — who is coming, for what service, and at what time — accessible from his phone at any moment.

- **Get notified immediately when a booking is made or cancelled.** A cancellation mid-morning is either an opportunity (a longer lunch, catching up on notes) or a problem to plan around (rearranging a room setup). Marcus cannot act on information he does not have. Every booking and cancellation event must generate a notification that reaches him within seconds, not discovered on his next check of the reception system.

- **Block personal time without admin involvement.** When Marcus books a week of annual leave or a half-day medical appointment, he needs to mark himself as unavailable immediately — without submitting a form, waiting for admin confirmation, or hoping the front desk remembers to update the calendar before a patient is booked into a blocked slot. Self-service availability management is a basic professional expectation.

---

## Frustrations

- **Finding out about bookings late.** Under previous systems, Marcus would sometimes arrive at his treatment room to find a patient already waiting — booked into a slot he had not been told about because the front desk forgot to pass on the message or the group WhatsApp notification was buried. Every late-discovery creates a rushed appointment and erodes trust.

- **No way to block unavailability without calling admin.** When Marcus needed to mark personal time, the previous process required him to call reception, wait for the calendar to be updated, and then verify the update had been made before booking a patient could be declined. This created dependency, friction, and occasional errors when the admin forgot or made the change on the wrong date.

- **Schedule changes communicated via WhatsApp groups.** Clinic-wide group messages meant that every staff member received every piece of scheduling information, relevant or not. Changes to Marcus's own bookings were lost in a stream of notifications for colleagues' appointments, rescheduling requests from other patients, and administrative reminders. The signal-to-noise ratio was too low to act reliably.

---

## Primary Actions in the Platform

| Action | Functional Requirements |
|---|---|
| View upcoming appointments (own schedule, filtered by day/week) | FR002 |
| Block personal unavailability (holiday, personal time, training) | FR002 |
| Receive booking confirmation notification when assigned to a new appointment | FR007 |
| Receive cancellation notification when a patient cancels | FR005, FR007 |
| Receive rescheduling notification when a patient changes their slot | FR007 |

---

## User Journeys

| Journey ID | Name | P2's Role |
|---|---|---|
| UJ003 | Appointment Cancellation | Downstream recipient — receives cancellation notification when a patient cancels; freed slot reflects immediately in own schedule |
| UJ004 | Appointment Rescheduling | Downstream recipient — receives rescheduling notification when a patient moves to a new slot; original slot freed in own schedule |

---

## Needs from the Platform

In priority order:

1. **Real-time notifications** for every booking event that affects their schedule — confirmation, cancellation, and rescheduling — delivered within 30 seconds of the triggering event.
2. **An accurate, always-current view** of their own upcoming appointments, filterable by day and week, requiring no manual refresh or check-in with reception.
3. **Self-service availability blocking** that takes effect immediately and is reflected in the public booking calendar without admin involvement.
4. **Notification delivery that does not depend on the booking operation** — a failed reminder must never delay or block a booking confirmation that the patient needs to receive.
5. **No exposure to other staff members' schedules or patient data** — the staff view must be strictly scoped to the authenticated staff member's own appointments and availability.

---

## Success Looks Like

A successful day for Marcus begins at 8:15 AM when he opens the AnjiSchedulo staff view on his phone and sees his eight confirmed appointments laid out chronologically, with patient first names, service types, and duration. At 10:40 AM his 11:00 session is cancelled by the patient. Marcus receives a push notification at 10:40:18 AM — before the clock on the wall has moved — and decides to use the gap to complete his session notes. He has not spoken to reception once about his schedule today. At 4 PM he opens the app again, marks next Thursday and Friday as blocked for his CPD course, and closes the tab. The booking calendar for those days is immediately closed to patients. He receives no calls from admin asking him to confirm the dates. His schedule is accurate, his time is respected, and his working day is uninterrupted by scheduling coordination tasks that the platform handles automatically.

---

## Main Interaction Flow

```
  STAFF MEMBER — MAIN INTERACTION FLOW
  ──────────────────────────────────────────────────────────
  1. START OF DAY — SCHEDULE CHECK
     Open staff view (mobile or desktop)
       → View own upcoming appointments (name, service, time)
       → FR002: Schedule derived from confirmed bookings + staff config
     Review any overnight changes (new bookings, cancellations)
       → FR007: Notifications delivered within 30s of each event

  2. INBOUND BOOKING NOTIFICATION
     New appointment booked by a customer
       → FR007: Staff receives immediate booking confirmation notification
     Notification includes: patient name, service, date, time
       → No action required — awareness only

  3. INBOUND CANCELLATION NOTIFICATION              [UJ003]
     Patient cancels confirmed appointment
       → FR005: Slot released; staff notified of cancellation
       → FR007: Notification delivered within 30s
     Staff chooses how to use freed slot (no action in platform required)

  4. INBOUND RESCHEDULING NOTIFICATION             [UJ004]
     Patient moves appointment to new slot
       → FR007: Staff receives rescheduling confirmation notification
     Original slot freed; new slot shows in upcoming schedule

  5. BLOCK PERSONAL AVAILABILITY
     Open availability panel
       → FR002: Select dates and times to mark as unavailable
     Block saved immediately
       → Booking calendar updated in real time
       → No admin approval required
  ──────────────────────────────────────────────────────────
```
