# R10 · Acceptance Criteria

**Project:** AnjiSchedulo  
**Version:** 1.0.0  
**Owner:** Pubudu Anjalee Cooray  
**Last Updated:** 2026-06-28

---

## 1. Overview

This document consolidates the acceptance criteria for every functional requirement in AnjiSchedulo. Each criterion is stated as a verifiable, testable condition. Criteria are organised by feature area and linked to their source FR, related business rules, and test type.

---

## 2. Tenant Management

### FR001 — Tenant Self-Service Onboarding

| # | Criterion | Test Type |
|---|---|---|
| AC001.1 | A tenant can submit a registration form with business name, owner email, and plan selection. The system accepts the request and returns a success response. | Integration |
| AC001.2 | A billing subscription is activated automatically on successful registration. | Integration (Stripe sandbox) |
| AC001.3 | A welcome email is sent to the owner email within 60 seconds of successful registration. | Integration |
| AC001.4 | The tenant status transitions from `provisioning` to `active` once all provisioning steps complete successfully. | Integration |
| AC001.5 | A tenant can configure business hours, services, and staff before the first customer booking. | E2E |
| AC001.6 | Submitting a registration with a duplicate email returns an appropriate error; no duplicate tenant is created. | Integration |

**Business Rules:** BR005  
**NFRs:** —

---

### FR002 — Tenant Configuration Management

| # | Criterion | Test Type |
|---|---|---|
| AC002.1 | Admin can set distinct business hours for each staff member for each day of the week. | Integration |
| AC002.2 | Admin can create a service with name, duration, and optional price; the service is immediately visible in slot availability. | Integration |
| AC002.3 | Admin can deactivate a service; deactivated services no longer appear in availability queries. | Integration |
| AC002.4 | Admin can add a staff member with their working hours; the staff member's slots appear in availability. | Integration |
| AC002.5 | Admin can remove a staff member; their future slots are no longer bookable. | Integration |
| AC002.6 | Admin can set the cancellation policy window (in hours); cancellations outside this window are rejected for all future bookings. | Integration |
| AC002.7 | Admin can set the booking advance window (in days); slots beyond this window do not appear in availability. | Integration |
| AC002.8 | Admin can set the maximum active bookings per customer; requests exceeding this limit are rejected. | Integration |

**Business Rules:** BR002, BR003, BR008, BR010  
**NFRs:** —

---

## 3. Booking

### FR003 — Customer Slot Browsing

| # | Criterion | Test Type |
|---|---|---|
| AC003.1 | A customer can query available slots by service, date, and staff member. | Integration |
| AC003.2 | Confirmed booking slots are not returned in availability results. | Integration |
| AC003.3 | Staff block periods are excluded from available slots. | Integration |
| AC003.4 | Slots outside the staff member's configured working hours are not returned. | Integration |
| AC003.5 | Slots beyond the tenant's booking window (in days) are not returned. | Integration |
| AC003.6 | Availability queries respond within 500ms at p95 under normal load. | Performance |

**Business Rules:** BR003, BR010  
**NFRs:** NFR005

---

### FR004 — Appointment Booking

| # | Criterion | Test Type |
|---|---|---|
| AC004.1 | A customer can complete a booking for a valid slot; the appointment is confirmed and the slot is locked. | E2E |
| AC004.2 | If the selected slot is no longer available at booking time, the system returns 409 Conflict and no payment is charged. | Integration |
| AC004.3 | If payment fails, the slot reservation is released and the system returns 402 Payment Required. | Integration (Stripe sandbox) |
| AC004.4 | Two concurrent booking requests for the same slot result in exactly one confirmed booking; the second request receives 409 Conflict. | Concurrency |
| AC004.5 | Duplicate booking requests carrying the same idempotency key return the original result without creating a duplicate appointment or charge. | Integration |
| AC004.6 | A customer who exceeds the tenant's max active bookings limit receives a 422 error before the saga is initiated. | Integration |
| AC004.7 | Customer receives a booking confirmation notification within 30 seconds of confirmation. | Integration |
| AC004.8 | The assigned staff member receives a booking notification within 30 seconds of confirmation. | Integration |
| AC004.9 | End-to-end booking confirmation latency is under 3 seconds at p95. | Performance |

**Business Rules:** BR001, BR004, BR006, BR007, BR008  
**NFRs:** NFR004, NFR007

---

### FR005 — Appointment Cancellation

| # | Criterion | Test Type |
|---|---|---|
| AC005.1 | A customer can cancel a confirmed appointment that is within the tenant's cancellation policy window. | Integration |
| AC005.2 | A cancellation request outside the policy window is rejected with a clear error message. | Integration |
| AC005.3 | On successful cancellation, the slot is released and available for new bookings. | Integration |
| AC005.4 | On successful cancellation with payment, a refund is initiated via Stripe. | Integration (Stripe sandbox) |
| AC005.5 | If the refund fails, the slot is still released and customer is still notified. The refund failure routes to DLQ for operator review. | Integration |
| AC005.6 | Customer receives a cancellation notification within 30 seconds. | Integration |
| AC005.7 | Assigned staff member receives a cancellation notification within 30 seconds. | Integration |
| AC005.8 | The full appointment history, including the cancellation event, is preserved in the event log. | Integration |

**Business Rules:** BR002, BR009  
**NFRs:** —

---

### FR006 — Appointment Rescheduling

| # | Criterion | Test Type |
|---|---|---|
| AC006.1 | A customer can reschedule a confirmed appointment to a new available slot. | E2E |
| AC006.2 | If the new slot is unavailable, the original booking remains unchanged and the customer receives a 409 error. | Integration |
| AC006.3 | The original slot is not released until the new slot is confirmed. | Integration |
| AC006.4 | Customer receives a rescheduling confirmation notification within 30 seconds. | Integration |
| AC006.5 | The reschedule event is recorded in the appointment event history. | Integration |

**Business Rules:** BR001, BR004  
**NFRs:** —

---

## 4. Notifications

### FR007 — Lifecycle Notifications

| # | Criterion | Test Type |
|---|---|---|
| AC007.1 | Notifications are dispatched for the following events: booking requested, confirmed, cancelled, rescheduled, upcoming reminder. | Integration |
| AC007.2 | Notification content is derived entirely from the event snapshot; the notification service makes no back-calls to booking services. | Unit |
| AC007.3 | Failed notification delivery is retried up to 4 times with exponential backoff. | Integration |
| AC007.4 | After 4 retries, the event routes to DLQ and an ops alert is raised. | Integration |
| AC007.5 | A notification delivery failure does not delay or roll back the triggering booking operation. | Integration |

**Business Rules:** BR007, BR013  
**NFRs:** NFR002

---

## 5. Dashboard and Analytics

### FR008 — Tenant Operational Dashboard

| # | Criterion | Test Type |
|---|---|---|
| AC008.1 | Dashboard displays: today's booking count, confirmed/cancelled/rescheduled counts, net revenue, cancellation rate, and peak hour. | Integration |
| AC008.2 | Dashboard data reflects booking events within 5 seconds under normal load. | Performance |
| AC008.3 | Dashboard API responds in under 200ms at p95. | Performance |
| AC008.4 | Dashboard can be fully rebuilt from event history after an event replay without data loss. | Integration |

**Business Rules:** —  
**NFRs:** NFR006, NFR008

---

### FR009 — Stream Processing Alerts

| # | Criterion | Test Type |
|---|---|---|
| AC009.1 | A booking velocity spike (> 3× historical average in a 5-minute window) triggers an alert. | Integration |
| AC009.2 | A cancellation rate exceeding 30% over the last 50 bookings triggers an alert. | Integration |
| AC009.3 | Peak hour data is updated continuously and available on the tenant admin dashboard. | Integration |
| AC009.4 | Alerts are delivered to tenant admin and platform operator within 60 seconds of detection. | Integration |

**Business Rules:** —  
**NFRs:** —

---

## 6. Platform Operations

### FR010 — Dead Letter Queue Triage

| # | Criterion | Test Type |
|---|---|---|
| AC010.1 | Failed events are routed to DLQ after retry exhaustion with full payload preserved. | Integration |
| AC010.2 | Ops console lists DLQ entries filterable by tenant_id, event_type, and failure_reason. | Manual (UI) |
| AC010.3 | Operator can re-queue a corrected event from the ops console. | Integration |
| AC010.4 | Operator can discard a DLQ event with a documented reason; an audit log entry is created. | Integration |
| AC010.5 | A DLQ entry triggers an automated ops alert within 5 minutes of appearing. | Integration |

**Business Rules:** BR013  
**NFRs:** NFR016

---

### FR011 — Tenant Event Replay

| # | Criterion | Test Type |
|---|---|---|
| AC011.1 | Operator can trigger an event replay scoped to a specific tenant. | Integration |
| AC011.2 | Operator can scope replay to a date range. | Integration |
| AC011.3 | Operator can scope replay to specific event IDs. | Integration |
| AC011.4 | Replayed events are processed idempotently; no duplicate state is created. | Integration |
| AC011.5 | Replay does not block or interfere with live booking operations. | Integration |
| AC011.6 | Replay job status (pending, running, completed, failed) is visible in ops console. | Manual (UI) |

**Business Rules:** —  
**NFRs:** NFR009

---

## 7. Tenant Lifecycle

### FR012 — Tenant Data Export

| # | Criterion | Test Type |
|---|---|---|
| AC012.1 | Tenant admin can request a full data export from the settings page. | E2E |
| AC012.2 | Export includes: configuration, staff, services, appointment events, payments, notifications, and analytics. | Integration |
| AC012.3 | Export is scoped strictly to the requesting tenant's data; no other tenant data is included. | Security |
| AC012.4 | Signed download link is delivered to the verified owner email only. | Integration |
| AC012.5 | Signed URL expires after 24 hours; requests after expiry return 403. | Integration |

**Business Rules:** BR005, BR014  
**NFRs:** —

---

### FR013 — Tenant Suspension and Deprovisioning

| # | Criterion | Test Type |
|---|---|---|
| AC013.1 | Suspending a tenant immediately halts new bookings (booking requests return an appropriate error). | Integration |
| AC013.2 | A suspended tenant admin retains read access to their data for 30 days. | Integration |
| AC013.3 | Deprovisioning requires explicit operator action and cannot be triggered before 30 days post-suspension. | Integration |
| AC013.4 | All tenant data is retained for 90 days post-deprovisioning before any deletion. | Integration |
| AC013.5 | Payment records are retained for 7 years and are excluded from hard deletion. | Integration |
| AC013.6 | PII fields are pseudonymised on individual customer erasure requests. | Integration |

**Business Rules:** BR011, BR012  
**NFRs:** —

---

## 8. Cross-Cutting Acceptance Criteria

### Tenant Isolation

| # | Criterion | Test Type |
|---|---|---|
| AC-ISO.1 | A request authenticated with tenant A's token cannot return, modify, or delete data belonging to tenant B. | Security |
| AC-ISO.2 | A missing or null tenant context in a database query returns zero rows (not all rows). | Security |
| AC-ISO.3 | Isolation monitor cross-checks run every 5 minutes in production; any violation generates a P1 alert. | Monitoring |

**Business Rules:** BR005  
**NFRs:** NFR012

---

### Audit Log

| # | Criterion | Test Type |
|---|---|---|
| AC-AUD.1 | Every write operation modifying tenant-scoped data produces an immutable audit log entry. | Integration |
| AC-AUD.2 | The audit log is INSERT-only; no UPDATE or DELETE is permitted on audit records. | Integration |
| AC-AUD.3 | Audit log entries include: timestamp, actor identity, operation type, entity affected, and before/after state. | Integration |

**Business Rules:** BR014  
**NFRs:** —

---

### Event Envelope

| # | Criterion | Test Type |
|---|---|---|
| AC-ENV.1 | Every event emitted by the platform carries: tenant_id, event_id, correlation_id, causation_id, and occurred_at. | Unit |
| AC-ENV.2 | An event missing any required envelope field is rejected and routed to DLQ; it is never silently dropped. | Integration |

**Business Rules:** BR013  
**NFRs:** NFR014
