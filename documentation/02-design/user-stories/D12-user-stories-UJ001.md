---
title: User Stories — Tenant Onboarding
journeyId: UJ001
layer: 02-design
status: current
lastUpdated: 2026-06-27
---

# D12 · User Stories — Tenant Onboarding (UJ001)

## Epic Header

| Field | Value |
|---|---|
| Epic Name | Tenant Onboarding |
| Journey ID | UJ001 |
| Primary Persona | P1 — Tenant Admin (Business Owner / Administrator) |
| Related FRs | FR001, FR002 |
| Related BRs | BR005, BR014 |
| Priority | Critical |
| Architecture Pattern | Pub/Sub fan-out on `tenant.provisioned` event |

## Epic Description

As a **Tenant Admin (business owner)**, I want to register my business on AnjiSchedulo, configure my services, hours, and staff, and go live with a booking page — so that my customers can book appointments online within 30 minutes of sign-up, without needing technical support.

---

## User Stories

---

### US-UJ001-01 · Submit tenant registration form

**Title:** Submit business registration to create a new tenant account

**User Story:**
As a **Tenant Admin**, I want to submit a registration form with my business name, owner email address, and chosen pricing plan, so that the platform provisions my tenant account and I can begin configuration immediately.

**Acceptance Criteria:**

- **Given** I am on the AnjiSchedulo registration page,  
  **When** I submit a form with a valid business name, a unique owner email address, and a selected plan (Starter, Growth, or Enterprise),  
  **Then** the platform creates a tenant record with status `provisioning` and returns a 201 response within 2 seconds.

- **Given** I submit the registration form,  
  **When** provisioning begins,  
  **Then** a billing subscription is activated automatically for the selected plan without any further action from me.

- **Given** I attempt to register with an email address already associated with an existing tenant,  
  **When** I submit the form,  
  **Then** the platform returns a 409 Conflict error with a clear message indicating the email is already in use.

- **Given** I submit the registration form with any required field missing or malformed,  
  **When** the request is validated,  
  **Then** the platform returns a 422 Unprocessable Entity with field-level error messages before any provisioning begins.

- **Given** my registration is submitted successfully,  
  **When** the `tenant.provisioned` event is published,  
  **Then** the event envelope contains `tenant_id`, `event_id`, `correlation_id`, and `occurred_at` as required by BR013.

**Size:** M  
**Priority:** Critical  
**Related:** FR001, BR005, BR013, BR014

**Technical Notes:**
- Tenant isolation (BR005) is established at creation time: a `tenant_id` UUID is generated and applied to all provisioned records. PostgreSQL Row-Level Security is active immediately.
- Registration must produce an immutable audit log entry (BR014) recording the `tenant_id`, owner email, plan, and timestamp.
- The `tenant.provisioned` event is published via the Transactional Outbox pattern to guarantee at-least-once delivery to downstream subscribers (billing-service, notification-service).

---

### US-UJ001-02 · Receive welcome email with setup link

**Title:** Receive welcome email with a personalised setup link

**User Story:**
As a **Tenant Admin**, I want to receive a welcome email at my registered address immediately after sign-up, so that I can access my setup dashboard and complete configuration without searching for the login URL.

**Acceptance Criteria:**

- **Given** the `tenant.provisioned` event has been published,  
  **When** the notification-service processes the event,  
  **Then** a welcome email is delivered to the registered owner email within 60 seconds of successful registration.

- **Given** the welcome email is sent,  
  **When** I open the email,  
  **Then** it contains the business name, a personalised setup link that authenticates me directly into my tenant dashboard, and the selected plan name.

- **Given** the notification-service fails to deliver the welcome email after 4 retry attempts with exponential backoff,  
  **When** delivery is exhausted,  
  **Then** the event is routed to the DLQ and an ops alert is raised — the tenant account remains active and accessible.

- **Given** the welcome email delivery fails,  
  **When** the failure is recorded,  
  **Then** the tenant provisioning status is not affected; the failure is isolated to the notification pipeline (BR007).

**Size:** S  
**Priority:** High  
**Related:** FR001, BR007, BR013

**Technical Notes:**
- The notification-service is a fully decoupled async subscriber of the `tenant.provisioned` event. Delivery failure must not block or roll back provisioning (BR007).
- Email content is derived entirely from the event payload — no back-call to tenant-service at send time.

---

### US-UJ001-03 · Configure business hours

**Title:** Set operating hours for each day of the week

**User Story:**
As a **Tenant Admin**, I want to configure the days and hours my business is open, so that the availability engine only shows customers slots within my actual operating schedule.

**Acceptance Criteria:**

- **Given** I am logged into my tenant dashboard,  
  **When** I navigate to Business Hours and save a schedule with open/closed toggle and start/end times per day,  
  **Then** the configuration is persisted and availability queries immediately exclude times outside my configured hours.

- **Given** I save business hours,  
  **When** I set a day as closed,  
  **Then** no appointment slots are available to customers on that day (BR003).

- **Given** I configure different hours per staff member,  
  **When** customers browse availability,  
  **Then** each staff member's available slots reflect their individual schedule, not the global business hours.

- **Given** I update business hours,  
  **When** the change is saved,  
  **Then** an immutable audit log entry is produced recording the previous and new configuration values (BR014).

**Size:** M  
**Priority:** Critical  
**Related:** FR002, BR003, BR014

**Technical Notes:**
- Business hours are stored per staff member per day in `tenant_working_hours`. The availability-service reads this table to compute open slots.
- Updates to business hours do not affect already-confirmed bookings; they affect only future availability queries.

---

### US-UJ001-04 · Add services to the catalogue

**Title:** Create services with name, duration, and price

**User Story:**
As a **Tenant Admin**, I want to create services specifying a name, duration in minutes, and price, so that customers can see exactly what they are booking and how long each appointment will take.

**Acceptance Criteria:**

- **Given** I am on the Services configuration page,  
  **When** I submit a new service with a name, duration (in minutes, minimum 15), and price (in pence/cents),  
  **Then** the service is created with status `active` and appears in the public booking page catalogue.

- **Given** I have created a service,  
  **When** I update its name, duration, or price,  
  **Then** the changes take effect for new bookings only; existing confirmed bookings retain their original service snapshot.

- **Given** I deactivate a service,  
  **When** customers browse the booking page,  
  **Then** the deactivated service no longer appears in the service catalogue and cannot be booked.

- **Given** I attempt to save a service without a name or with a duration less than 15 minutes,  
  **When** the request is validated,  
  **Then** the platform returns field-level validation errors and the service is not created.

- **Given** any service write operation,  
  **When** the record is modified,  
  **Then** an immutable audit log entry is produced (BR014).

**Size:** M  
**Priority:** Critical  
**Related:** FR002, BR014

**Technical Notes:**
- Service records are tenant-scoped with `tenant_id` on every row; RLS enforces that no cross-tenant reads are possible (BR005).
- Service pricing is stored in the smallest currency unit (pence/cents) to avoid floating-point errors.

---

### US-UJ001-05 · Add staff members

**Title:** Add staff members and assign them to services

**User Story:**
As a **Tenant Admin**, I want to add staff members to my account and assign them to specific services, so that customers can choose which staff member they want for their appointment and availability is calculated per person.

**Acceptance Criteria:**

- **Given** I am on the Staff configuration page,  
  **When** I submit a staff record with a name and email address,  
  **Then** the staff member is created with status `active` and is selectable on the booking page.

- **Given** I assign a staff member to one or more services,  
  **When** a customer selects one of those services,  
  **Then** only staff assigned to that service appear as options on the booking page.

- **Given** I remove a staff member,  
  **When** the record is deactivated,  
  **Then** the staff member is hidden from the booking page and no future slots are generated for them; existing confirmed bookings are not affected.

- **Given** I add a staff member,  
  **When** the record is saved,  
  **Then** an immutable audit log entry is produced (BR014).

**Size:** M  
**Priority:** Critical  
**Related:** FR002, BR003, BR014

**Technical Notes:**
- Staff records carry `tenant_id` and are protected by RLS (BR005).
- Staff deactivation must not cancel or alter existing confirmed appointments — those retain the staff member's snapshot data.

---

### US-UJ001-06 · Go live — accept customer bookings

**Title:** Verify booking page is live and accept the first customer appointment

**User Story:**
As a **Tenant Admin**, I want to confirm that my booking page is publicly accessible and correctly shows my services and available slots, so that I can confidently share the booking link with customers and begin receiving appointments.

**Acceptance Criteria:**

- **Given** I have configured business hours, at least one service, and at least one staff member,  
  **When** I navigate to my public booking page URL,  
  **Then** the page loads without authentication, displays my services, and shows available time slots.

- **Given** my booking page is live,  
  **When** a customer completes a booking,  
  **Then** the appointment appears in my tenant dashboard within 5 seconds of confirmation.

- **Given** tenant provisioning is complete,  
  **When** the tenant status transitions to `active`,  
  **Then** new bookings are accepted; the platform did not require any manual operator action to activate the account (FR001).

- **Given** my tenant account has not completed all required configuration (hours, services, staff),  
  **When** I attempt to view my booking page,  
  **Then** the page indicates that the business is not yet accepting bookings and prompts me to complete setup.

**Size:** S  
**Priority:** High  
**Related:** FR001, FR002, BR005

**Technical Notes:**
- The public booking page is unauthenticated — it queries the availability-service with only the `tenant_id` (derived from the URL slug), enforcing read-only tenant scope.
- Tenant status state machine: `provisioning` → `active`. The transition fires only after all provisioning subscribers (billing, notification) have confirmed.

---

## Definition of Ready

Before any story in this epic enters a sprint, confirm all items below are checked:

- [ ] Acceptance criteria reviewed and agreed by Product Owner
- [ ] `tenant.provisioned` event schema finalised and documented in `specs/ai/events.json`
- [ ] PostgreSQL RLS policies for tenant isolation verified in local environment (BR005)
- [ ] Transactional Outbox table schema confirmed in `specs/ai/infrastructure.json`
- [ ] Billing subscription activation flow confirmed with billing-service team (FR001)
- [ ] Welcome email template content approved
- [ ] Audit log schema confirmed — INSERT-only privilege for `booking_app_user` (BR014)
- [ ] Test data and seed scripts available for local development
- [ ] Definition of Done agreed for this layer (see `documentation/00-governance/G5-definition-of-done.md`)
