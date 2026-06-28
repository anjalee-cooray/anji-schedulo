---
title: Persona — Tenant Admin
personaId: P1
layer: 01-requirements
status: current
lastUpdated: 2026-06-27
---

# Persona — Tenant Admin (P1)

| Field | Value |
|---|---|
| **Persona ID** | P1 |
| **Name** | Tenant Admin |
| **Role** | Business Owner / Administrator |
| **Technical Level** | Low to medium |

---

## Who Is This Person?

Sarah runs a four-chair hair salon in the city centre with two full-time stylists and one part-time colourist. For years she managed appointments in a paper diary and a WhatsApp group — a system that regularly produced double-bookings, no-shows with no recourse, and late-night messages from customers wanting to change their 9 AM slot. She discovered AnjiSchedulo through a trade association newsletter and signed up during a quiet Tuesday. Within 25 minutes she had configured her business hours, added her three service types (cut, colour, blow-dry), and listed her staff. By lunchtime, her first customer had booked online without Sarah touching her phone. AnjiSchedulo is now the operational spine of her salon: she opens the dashboard every morning to see the day's confirmed bookings, watches the cancellation rate so she can spot emerging trends, and trusts that her team is notified automatically of every booking change. She is not technical — she can follow a guided setup flow but will not debug a configuration error. Everything must be self-evident.

---

## Goals

- **Reduce no-shows through automated reminders.** Every missed appointment costs Sarah the full slot revenue with no compensation. She needs the platform to send timely reminders automatically without her having to remember or manually trigger anything. If reminders require action from her, they will not be sent.

- **Eliminate double-bookings entirely.** Two customers arriving for the same stylist at the same time creates an immediate operational crisis and damages trust. Availability enforcement must be handled at the system level — Sarah cannot rely on staff self-coordination or manual calendar checks.

- **See today's schedule and revenue at a glance.** Sarah checks the dashboard before the first customer arrives each morning. She needs confirmed bookings, net revenue, and cancellation rate visible on a single screen without navigating multiple tabs or running reports.

- **Manage staff availability without back-and-forth messages.** When a stylist blocks a holiday week, that change must propagate to the public booking calendar immediately. Sarah cannot be the human router for every availability change — each message she has to relay is an opportunity for error.

- **Onboard without needing technical help.** Sarah does not have an IT department or a developer on call. The entire setup experience must be self-serve, guided, and completable by a non-technical business owner in under 30 minutes from registration to accepting live bookings.

---

## Frustrations

- **Manual scheduling via phone or spreadsheet.** Before AnjiSchedulo, Sarah lost hours each week taking bookings by phone, transcribing them to a spreadsheet, and cross-referencing availability by hand. Every hour spent on administrative scheduling is an hour not available for customers or business development.

- **No-shows with no penalty.** When customers failed to show, Sarah had no system record, no way to detect repeat offenders, and no mechanism to enforce her stated cancellation policy. Revenue evaporated silently and the slot could not be recovered.

- **Hard to see which services are most popular.** Without analytics, Sarah could not tell at a glance whether haircuts or colour treatments drove more revenue. Staffing decisions and promotional planning were made on intuition rather than data.

- **Staff calling in sick with no easy rebooking flow.** When a stylist was unexpectedly absent, Sarah had to manually identify every affected appointment, contact each customer individually to reschedule, and find alternative slots. This process consumed an entire morning and produced errors.

---

## Primary Actions in the Platform

| Action | Functional Requirements |
|---|---|
| Configure business hours and services (per staff member, per day) | FR002 |
| Add, update, and remove staff members | FR002 |
| View and manage the daily booking dashboard (bookings, revenue, cancellation rate, peak hour) | FR008, FR009 |
| Respond to anomaly alerts (cancellation rate spike, booking velocity spike) | FR009 |
| Export full booking history for compliance, auditing, or offboarding | FR012 |

---

## User Journeys

| Journey ID | Name | P1's Role |
|---|---|---|
| UJ001 | Tenant Onboarding | Primary actor — registers business, configures hours, services, and staff, makes booking page live |
| UJ005 | Tenant Dashboard Review | Primary actor — opens dashboard at start of day, reviews operational metrics, acts on alerts |
| UJ007 | Tenant Data Export and Offboarding | Primary actor — requests data export, downloads archive; Platform Operator (P4) executes deprovisioning |

---

## Needs from the Platform

In priority order:

1. **Double-booking prevention** enforced at the system level — no booking must ever be confirmed into an already-occupied slot.
2. **A fast, accurate dashboard** that reflects booking events within 5 seconds and loads in under 200 ms.
3. **Automated notifications** for customers and staff that require zero manual effort from the admin after initial configuration.
4. **Self-service staff availability management** — staff members must be able to block their own time without admin involvement.
5. **Self-serve onboarding** with no technical prerequisites — from registration to accepting live bookings in under 30 minutes.
6. **Portable data export** that includes all booking history, payment records, and analytics, scoped strictly to the tenant's own data.

---

## Success Looks Like

A successful morning for Sarah is opening the AnjiSchedulo dashboard at 8:30 AM and seeing the day's seven confirmed bookings laid out in chronological order, alongside net revenue for the day and a healthy 5% cancellation rate. She has not made a single phone call to confirm appointments — all reminders were sent automatically the evening before. One customer cancelled at 7 AM: the refund was triggered automatically, the slot was released immediately, and her stylist received a push notification before Sarah even saw the dashboard update. By 8:45 AM that slot had already been taken by a new booking. Sarah closes the tab and starts the day.

---

## Main Interaction Flow

```
  TENANT ADMIN — MAIN INTERACTION FLOW
  ──────────────────────────────────────────────────────────
  1. ONBOARD                                         [UJ001]
     Register (business name, email, plan)
       → FR001: Platform provisions tenant account
     Configure business hours (per staff, per day)
       → FR002: Business hours saved
     Add services (name, duration, price)
       → FR002: Service catalogue created
     Add staff members
       → FR002: Staff roster created
     Booking page goes live
       → First customer booking now possible

  2. DAILY OPERATIONS                                [UJ005]
     Open dashboard
       → FR008: Today's bookings, revenue, cancellation rate
     Review cancellation rate and booking velocity
       → FR009: Anomaly alerts surfaced if thresholds breached
     Act on alert (e.g. high cancellation rate)
       → Investigate, adjust policy if needed

  3. ONGOING STAFF MANAGEMENT                        [FR002]
     Add / update / remove staff
     Set per-staff business hours
     Staff member blocks own unavailability
       → Booking calendar updates immediately

  4. DATA EXPORT / OFFBOARD (if needed)             [UJ007]
     Request data export
       → FR012: Archive generated, signed URL to owner email
     Download archive
     After 30-day window: Operator initiates deprovisioning
       → FR013: Data retained 90 days then deleted
  ──────────────────────────────────────────────────────────
```
