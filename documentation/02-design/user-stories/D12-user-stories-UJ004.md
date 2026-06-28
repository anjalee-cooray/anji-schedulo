---
title: User Stories — Appointment Rescheduling
journeyId: UJ004
layer: 02-design
status: current
lastUpdated: 2026-06-28
---

# D12 · User Stories — Appointment Rescheduling (UJ004)

## Epic Header

| Field | Value |
|---|---|
| Epic Name | Appointment Rescheduling |
| Journey ID | UJ004 |
| Primary Persona | P3 — End Customer |
| Related FRs | FR006 |
| Related BRs | BR001, BR004, BR013, BR014 |
| Priority | High |
| Architecture Pattern | Saga Orchestration, Transactional Outbox |

## Epic Description

As an **End Customer**, I want to move my confirmed appointment to a different time slot — so that my original booking is preserved if the new slot is unavailable, and I receive confirmation of the new time if the swap succeeds.

---

## User Stories

---

### US-UJ004-01 · Select a new slot for an existing appointment

**Title:** Submit a reschedule request that atomically swaps to a new slot

**User Story:**
As an **End Customer**, I want to select a new time slot for my existing confirmed appointment, so that the platform checks new-slot availability before releasing my current slot and completes the swap atomically.

**Acceptance Criteria:**

- **Given** I have a confirmed appointment,  
  **When** I submit a reschedule request specifying a new `slot_date` and `slot_start`,  
  **Then** the platform first checks and locks the new slot before releasing the original — if the new slot is unavailable, my original appointment remains unchanged (FR006, BR001).

- **Given** the new slot is available and the SlotLock INSERT succeeds,  
  **When** rescheduling completes,  
  **Then** I receive HTTP 200 with the updated appointment details including the new date and time.

- **Given** the new slot is already booked by another customer,  
  **When** the SlotLock INSERT for the new slot fails due to the unique partial index,  
  **Then** I receive HTTP 409 Conflict with `slot_unavailable` and my original appointment is not modified (BR001).

- **Given** rescheduling succeeds,  
  **When** the `appointment.rescheduled` event is published,  
  **Then** the event payload includes `old_slot_date`, `old_slot_start`, `new_slot_date`, `new_slot_start`, and `new_slot_end` — and all required envelope fields (`tenant_id`, `event_id`, `correlation_id`, `occurred_at`) are present (BR013).

**Size:** M  
**Priority:** High  
**Related:** FR006, BR001, BR004, BR013

**Technical Notes:**
- Saga sequence: (1) new SlotLock INSERT → (2) old SlotLock `released_at = NOW()` → (3) `AppointmentEvent(rescheduled)` INSERT → (4) `OutboxRecord` INSERT — all in one PostgreSQL transaction.
- The atomic transaction guarantees there is never a moment when neither slot is locked: if the new SlotLock fails, the old SlotLock is never touched.
- booking-command-service orchestrates the reschedule; no separate saga service is needed as both slot operations are within the same service boundary.
- The public reschedule endpoint (`PATCH /tenants/{tenantSlug}/bookings/{appointment_id}`) requires the customer's JWT or an OTP-verified token.

---

### US-UJ004-02 · Receive rescheduling confirmation notification

**Title:** Receive an email confirming the new appointment details after a successful reschedule

**User Story:**
As an **End Customer**, I want to receive a confirmation email immediately after my appointment is rescheduled, so that I have the new date and time in writing and can update my calendar.

**Acceptance Criteria:**

- **Given** my appointment is rescheduled and `appointment.rescheduled` is published,  
  **When** notification-service processes the event,  
  **Then** I receive a confirmation email containing the new date, new start time, staff member name, and service name within 30 seconds (FR007).

- **Given** the confirmation email content,  
  **When** it is assembled by notification-service,  
  **Then** all details are derived from the event payload — no back-call to booking-command-service is made (Event-Carried State Transfer).

- **Given** notification delivery fails after 4 retry attempts,  
  **When** the failure is routed to DLQ,  
  **Then** the rescheduled appointment is not rolled back — the slot swap is complete and permanent (BR007).

**Size:** S  
**Priority:** High  
**Related:** FR006, FR007, BR007

**Technical Notes:**
- `appointment.rescheduled` payload includes `customer_snapshot`, `staff_snapshot`, `service_snapshot`, `old_slot_date`, `old_slot_start`, `new_slot_date`, `new_slot_start`, `new_slot_end`.
- InboxRecord deduplication in notification-service prevents duplicate confirmation emails on SQS redelivery.
- availability-service also consumes `appointment.rescheduled` to invalidate Redis cache for both the old slot (now free) and the new slot (now locked).

---

### US-UJ004-03 · Guarantee atomic slot swap — no orphaned locks

**Title:** Ensure the slot swap is all-or-nothing with no intermediate broken state

**User Story:**
As a **Platform** guaranteeing booking correctness, I want the new-slot lock and old-slot release to occur in the same database transaction, so that there is never a point where a customer has neither slot confirmed nor any gap in the booking history.

**Acceptance Criteria:**

- **Given** a reschedule request is being processed,  
  **When** the database transaction commits,  
  **Then** exactly one of the following is true: (a) both the new SlotLock exists and the old SlotLock is released, or (b) neither has changed — partial state is impossible (BR004).

- **Given** a database error occurs mid-transaction,  
  **When** PostgreSQL rolls back,  
  **Then** the old SlotLock remains locked and no new SlotLock is created — the customer's original appointment is unchanged.

- **Given** a reschedule succeeds,  
  **When** the availability-service consumes `appointment.rescheduled`,  
  **Then** the old slot appears as available and the new slot appears as booked in subsequent availability queries within 5 seconds.

**Size:** S  
**Priority:** High  
**Related:** FR006, BR004

**Technical Notes:**
- All four writes (new SlotLock INSERT, old SlotLock UPDATE `released_at`, `AppointmentEvent(rescheduled)` INSERT, `OutboxRecord` INSERT) are wrapped in a single `BEGIN/COMMIT` block.
- PostgreSQL's ACID guarantees make partial commit impossible; no distributed transaction coordinator is needed.
- availability-service cache invalidation for both old and new slots is driven by the `appointment.rescheduled` event, not a direct cache write from booking-command-service.

---

### US-UJ004-04 · Preserve full appointment history through rescheduling

**Title:** Record rescheduling as an immutable event in the appointment's event stream

**User Story:**
As a **Platform** meeting audit requirements, I want every reschedule to be recorded as an immutable event in the appointment's `AppointmentEvent` stream, so that the full lifecycle of any appointment — including all rescheduling operations — can be reconstructed at any time.

**Acceptance Criteria:**

- **Given** a reschedule succeeds,  
  **When** the `AppointmentEvent(rescheduled)` is written,  
  **Then** the event is INSERT-only, contains `old_slot_date`, `old_slot_start`, `new_slot_date`, `new_slot_start`, `actor_id`, `actor_role`, and `occurred_at` (BR014).

- **Given** the full `AppointmentEvent` stream for a rescheduled appointment,  
  **When** it is replayed in `occurred_at` order,  
  **Then** the derived state correctly reflects all transitions: `requested → confirmed → rescheduled` with the updated slot details.

- **Given** an audit log request for the appointment,  
  **When** the `AuditLog` is queried,  
  **Then** an `AuditLog` entry exists for the reschedule operation with `before_snapshot` (original slot) and `after_snapshot` (new slot).

**Size:** S  
**Priority:** Medium  
**Related:** BR013, BR014

**Technical Notes:**
- `AppointmentEvent` is an INSERT-only table; no UPDATE or DELETE operations are permitted on existing rows.
- `AuditLog` entry is written in the same transaction as the `AppointmentEvent(rescheduled)` record.
- Event Sourcing: the Appointment's current state is derived by replaying its `AppointmentEvent[]` in `occurred_at` order; the rescheduled state is always recoverable even if the appointment row itself is corrupted.

---

## Definition of Ready

Before any story in this epic enters a sprint, confirm all items below are checked:

- [ ] Acceptance criteria reviewed and agreed by Product Owner
- [ ] `appointment.rescheduled` event schema finalised in `specs/ai/events.json` — confirm all payload fields including `old_slot_*` and `new_slot_*`
- [ ] Atomic swap transaction boundary confirmed — new SlotLock INSERT + old SlotLock release + AppointmentEvent + OutboxRecord in one `BEGIN/COMMIT`
- [ ] Conflict behaviour confirmed: HTTP 409 returns with original appointment unchanged (BR001)
- [ ] availability-service confirmed to consume `appointment.rescheduled` and invalidate cache for both old and new slots
- [ ] Notification template for rescheduling confirmation content approved
- [ ] AuditLog before_snapshot and after_snapshot fields confirmed for reschedule operation (BR014)
- [ ] Test scenarios cover: unavailable new slot (original unchanged), concurrent reschedule attempts (race condition with SlotLock), database rollback path (BR004)
- [ ] Definition of Done agreed for this layer (see `documentation/00-governance/G5-definition-of-done.md`)
