# R8 · Use Case Document

**Project:** AnjiSchedulo  
**Version:** 1.0.0  
**Owner:** Pubudu Anjalee Cooray  
**Last Updated:** 2026-06-28

---

## 1. Overview

This document describes the primary use cases for AnjiSchedulo, derived from the functional requirements and user journeys. Each use case defines the actor, preconditions, main flow, alternative flows, and postconditions.

---

## 2. Actors

| Actor | Description | Persona |
|---|---|---|
| **Tenant Admin** | Business owner / operations manager managing their tenant on the platform | P1 |
| **Staff Member** | Employee or contractor whose schedule is managed on the platform | P2 |
| **Customer** | End user booking an appointment with a tenant's business | P3 |
| **Platform Operator** | Internal AnjiSchedulo team member managing infrastructure and tenant lifecycle | P4 |
| **System** | AnjiSchedulo platform acting autonomously (e.g. sending reminders, detecting anomalies) | — |

---

## 3. Use Cases

---

### UC001 — Tenant Self-Service Onboarding
**Actor:** Tenant Admin (P1)  
**Related FR:** FR001  
**Related Journey:** UJ001  
**Priority:** Critical

**Preconditions:**
- Actor has a valid email address and business name.
- No existing tenant account exists for the email.

**Main Flow:**
1. Tenant Admin submits registration form with business name, owner email, and selected pricing plan.
2. System activates billing subscription automatically.
3. System sends a welcome email with setup link to the owner email.
4. Tenant Admin configures business hours for each staff member and day.
5. Tenant Admin creates services with name, duration, and optional price.
6. Tenant Admin adds staff members with their working hours.
7. System transitions tenant status to `active`.
8. Booking page is live and accepting customer appointments.

**Alternative Flows:**
- *A1 — Billing activation fails:* System marks tenant as `pending_billing`, notifies the admin, and blocks the booking page until resolved.
- *A2 — Duplicate email:* System returns an error and prompts the admin to log in instead.

**Postconditions:**
- Tenant is active and accepting bookings.
- Billing subscription is active.
- Welcome email has been delivered.

---

### UC002 — Configure Business Parameters
**Actor:** Tenant Admin (P1)  
**Related FR:** FR002  
**Priority:** High

**Preconditions:**
- Tenant is active.
- Actor is authenticated with the `tenant_admin` role.

**Main Flow:**
1. Tenant Admin navigates to configuration settings.
2. Admin sets business hours per staff member per day of the week.
3. Admin creates or updates services (name, duration, price, active status).
4. Admin adds or deactivates staff members.
5. Admin sets the cancellation policy window (hours before appointment).
6. Admin sets the booking advance window (maximum days ahead customers can book).
7. Admin sets the maximum active bookings per customer.
8. System persists all configuration changes and applies them to future availability calculations immediately.

**Alternative Flows:**
- *A1 — Invalid hours configuration (end before start):* System rejects with a validation error.

**Postconditions:**
- Configuration is persisted and effective for all subsequent availability queries.

---

### UC003 — Browse Available Slots
**Actor:** Customer (P3)  
**Related FR:** FR003  
**Related Journey:** UJ002 (steps 1–4)  
**Priority:** Critical

**Preconditions:**
- Tenant is active.
- At least one service and one staff member are configured.

**Main Flow:**
1. Customer visits the tenant's booking page.
2. Customer selects a service.
3. Customer optionally selects a preferred staff member (or selects "any available").
4. Customer selects a date.
5. System returns available time slots, excluding confirmed bookings, staff blocks, and periods outside working hours.
6. Slots beyond the tenant's configured booking window are not returned.

**Alternative Flows:**
- *A1 — No slots available:* System returns an empty result with a clear message.
- *A2 — Service is inactive:* System excludes the service from the selection list.

**Postconditions:**
- Customer sees accurate available slots in under 500ms (NFR005).

---

### UC004 — Book an Appointment
**Actor:** Customer (P3)  
**Related FR:** FR004  
**Related Journey:** UJ002  
**Priority:** Critical

**Preconditions:**
- Customer has selected a service, staff member, and available slot.
- Tenant is active.

**Main Flow:**
1. Customer enters contact details (name, email, phone).
2. Customer completes payment if the service has a price (via Stripe).
3. System initiates booking saga: checks slot availability, processes payment, confirms appointment.
4. System emits `appointment.confirmed` event.
5. System sends confirmation notification to customer within 30 seconds.
6. System notifies the assigned staff member.

**Alternative Flows:**
- *A1 — Slot no longer available:* System returns 409 Conflict; no charge is made; customer is shown a prompt to select another slot.
- *A2 — Payment fails:* System releases the slot reservation; returns 402 Payment Required; customer is prompted to retry.
- *A3 — Duplicate request (same idempotency key):* System returns the original result without re-processing.
- *A4 — Customer exceeds max active bookings:* System returns 422 with clear error before initiating the saga.

**Postconditions:**
- Appointment is confirmed and persisted.
- Slot is locked to prevent double-booking (BR001).
- Customer and staff notifications are dispatched.

---

### UC005 — Cancel an Appointment
**Actor:** Customer (P3) or Staff Member (P2)  
**Related FR:** FR005  
**Related Journey:** UJ003  
**Priority:** Critical

**Preconditions:**
- Appointment is in `confirmed` status.
- Actor is authenticated and is either the booking customer or an assigned staff member (or tenant admin).

**Main Flow:**
1. Actor requests cancellation of a confirmed appointment.
2. System validates that the cancellation falls within the tenant's cancellation policy window (BR002).
3. System emits `appointment.cancelled` event.
4. System releases the slot for new bookings.
5. System initiates refund if payment was taken (via Stripe).
6. System notifies the customer of cancellation.
7. System notifies the assigned staff member of cancellation.

**Alternative Flows:**
- *A1 — Outside cancellation window:* System rejects with a 422 error and the policy window details.
- *A2 — Refund fails:* Refund failure routes to DLQ for manual operator review. Slot is released and customer is notified regardless (BR009).

**Postconditions:**
- Appointment status is `cancelled`.
- Slot is available for new bookings.
- Full appointment history including cancellation is preserved.

---

### UC006 — Reschedule an Appointment
**Actor:** Customer (P3)  
**Related FR:** FR006  
**Related Journey:** UJ004  
**Priority:** High

**Preconditions:**
- Appointment is in `confirmed` status.
- Customer is authenticated or identified by email + booking reference.

**Main Flow:**
1. Customer requests a new time for their existing appointment.
2. Customer selects a new available slot.
3. System checks availability of the new slot.
4. If available, system releases original slot and confirms new slot atomically.
5. System emits `appointment.rescheduled` event.
6. System sends rescheduling confirmation to customer.
7. System notifies the assigned staff member.

**Alternative Flows:**
- *A1 — New slot unavailable:* System returns 409; original booking is unchanged.
- *A2 — New slot is with a different staff member:* System handles as a normal reschedule, updating the staff assignment.

**Postconditions:**
- Appointment is confirmed at the new slot.
- Original slot is released and available for new bookings.
- Rescheduling event is recorded in appointment history.

---

### UC007 — View Operational Dashboard
**Actor:** Tenant Admin (P1)  
**Related FR:** FR008  
**Related Journey:** UJ005  
**Priority:** High

**Preconditions:**
- Tenant Admin is authenticated with the `tenant_admin` role.
- At least one booking event exists for the tenant.

**Main Flow:**
1. Admin opens the operational dashboard.
2. System returns the pre-aggregated materialized view for the tenant in under 200ms.
3. Dashboard displays: today's bookings, confirmed/cancelled/rescheduled counts, net revenue, cancellation rate, peak hour.
4. Dashboard data reflects all booking events within the last 5 seconds.

**Alternative Flows:**
- *A1 — Materialized view is corrupted or stale:* Platform Operator triggers an event replay to rebuild the view (UC010).

**Postconditions:**
- Admin has an accurate, real-time operational picture.

---

### UC008 — Receive Booking Anomaly Alert
**Actor:** Tenant Admin (P1), Platform Operator (P4)  
**Related FR:** FR009  
**Priority:** High

**Preconditions:**
- Stream processing is active for the tenant.
- Booking history exists (at least 50 bookings for cancellation rate alerting).

**Main Flow:**
1. System continuously monitors booking velocity and cancellation rate per tenant.
2. If booking velocity exceeds 3× historical average in a 5-minute window, System emits an anomaly alert.
3. If cancellation rate exceeds 30% over the last 50 bookings, System emits an anomaly alert.
4. Alert is delivered to the Tenant Admin dashboard and Platform Operator Grafana within 60 seconds.

**Postconditions:**
- Tenant Admin and Platform Operator are aware of the anomaly within 60 seconds.

---

### UC009 — DLQ Triage and Event Reprocessing
**Actor:** Platform Operator (P4)  
**Related FR:** FR010  
**Related Journey:** UJ006  
**Priority:** High

**Preconditions:**
- One or more events have failed after retry exhaustion and are present in the DLQ.
- Grafana alert has been fired (DLQ depth > 0).

**Main Flow:**
1. Platform Operator receives Grafana alert: DLQ depth > 0 for a specific queue.
2. Operator opens Ops console and inspects the failed event payload, including tenant_id, event_type, and failure_reason.
3. Operator identifies the root cause (e.g. malformed payload, downstream timeout).
4. Operator corrects the payload (if required) or acknowledges the root cause.
5. Operator re-queues the event for reprocessing.
6. System confirms the event is processed successfully.
7. DLQ depth returns to zero.

**Alternative Flows:**
- *A1 — Event is unrecoverable:* Operator discards the event with a documented reason; an immutable audit log entry is created.

**Postconditions:**
- DLQ is cleared.
- Audit trail records all operator actions.
- Resolution completed within 4-hour SLA (NFR016).

---

### UC010 — Tenant Event Replay
**Actor:** Platform Operator (P4)  
**Related FR:** FR011  
**Priority:** High

**Preconditions:**
- A derived view (e.g. dashboard materialized view) is corrupted or requires rebuild.
- Event log is intact.

**Main Flow:**
1. Platform Operator identifies the need for replay (e.g. corrupted materialized view, data migration).
2. Operator selects replay scope: full tenant, date range, specific event IDs.
3. System re-publishes historical events to the target consumer queues.
4. Target services rebuild state idempotently from replayed events (NFR009).
5. Replay job status is tracked and visible in ops console.
6. Replay completes without disrupting live booking operations.

**Postconditions:**
- Derived view is rebuilt and consistent with the event log.
- Live bookings are unaffected.

---

### UC011 — Tenant Data Export
**Actor:** Tenant Admin (P1)  
**Related FR:** FR012  
**Related Journey:** UJ007 (steps 1–4)  
**Priority:** Medium

**Preconditions:**
- Tenant Admin is authenticated with the `tenant_admin` role.
- Tenant is active or suspended.

**Main Flow:**
1. Tenant Admin requests a full data export from the settings page.
2. System generates a structured archive of all tenant-scoped data: config, staff, services, appointments, payments, notifications, analytics.
3. System uploads the archive to S3 and generates a signed URL.
4. Signed URL is delivered to the verified owner email.
5. URL expires after 24 hours.

**Alternative Flows:**
- *A1 — Export generation fails:* System retries and notifies the admin. Export never delivers a partial archive.

**Postconditions:**
- Tenant has a complete, scoped data export.
- Download link has been sent to verified owner email only.

---

### UC012 — Tenant Suspension and Deprovisioning
**Actor:** Platform Operator (P4)  
**Related FR:** FR013  
**Related Journey:** UJ007 (steps 5–6)  
**Priority:** Medium

**Preconditions:**
- Tenant has requested cancellation or violated terms of service.
- Operator has appropriate access level.

**Main Flow:**
1. Platform Operator suspends the tenant: new bookings are halted; read access is preserved.
2. Tenant Admin retains read access for 30 days to export their data (BR011).
3. After 30 days, Operator initiates deprovisioning.
4. All tenant data is retained for 90 days (BR012).
5. After 90 days, hard deletion runs; payment records are retained for 7 years.
6. PII is pseudonymised on individual customer erasure requests.

**Alternative Flows:**
- *A1 — Tenant requests reactivation during suspension:* Operator can reactivate before deprovisioning.

**Postconditions:**
- Tenant is fully deprovisioned; no residual data after retention period.
- Audit log entry records all lifecycle actions.

---

## 4. Use Case Summary

| Use Case | Actor | Priority | Related FR |
|---|---|---|---|
| UC001 — Tenant Onboarding | P1 | Critical | FR001 |
| UC002 — Configure Business Parameters | P1 | High | FR002 |
| UC003 — Browse Available Slots | P3 | Critical | FR003 |
| UC004 — Book an Appointment | P3 | Critical | FR004 |
| UC005 — Cancel an Appointment | P2, P3 | Critical | FR005 |
| UC006 — Reschedule an Appointment | P3 | High | FR006 |
| UC007 — View Operational Dashboard | P1 | High | FR008 |
| UC008 — Receive Anomaly Alert | P1, P4 | High | FR009 |
| UC009 — DLQ Triage and Event Reprocessing | P4 | High | FR010 |
| UC010 — Tenant Event Replay | P4 | High | FR011 |
| UC011 — Tenant Data Export | P1 | Medium | FR012 |
| UC012 — Tenant Suspension and Deprovisioning | P4 | Medium | FR013 |
