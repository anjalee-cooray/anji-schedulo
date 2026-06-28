---
title: User Stories — Tenant Data Export and Offboarding
journeyId: UJ007
layer: 02-design
status: current
lastUpdated: 2026-06-28
---

# D12 · User Stories — Tenant Data Export and Offboarding (UJ007)

## Epic Header

| Field | Value |
|---|---|
| Epic Name | Tenant Data Export and Offboarding |
| Journey ID | UJ007 |
| Primary Persona | P1 — Tenant Admin (export); P4 — Platform Operator (deprovisioning) |
| Related FRs | FR012, FR013 |
| Related BRs | BR005, BR011, BR012, BR014 |
| Priority | Medium |
| Architecture Pattern | Pub/Sub (tenant.deprovisioned), Status Machine Deprovisioning |

## Epic Description

As a **Tenant Admin** departing from AnjiSchedulo, I want to export all my business data and exit the platform cleanly — so that I leave with a complete data archive, the platform retains no residual data beyond the legal minimum, and my customers' PII is properly managed throughout the offboarding lifecycle.

---

## User Stories

---

### US-UJ007-01 · Request a full data export

**Title:** Generate and receive a complete archive of all tenant data via a signed download link

**User Story:**
As a **Tenant Admin**, I want to request a full export of all my business data, so that I have a portable archive of my bookings, payments, and configuration before I leave the platform.

**Acceptance Criteria:**

- **Given** I am authenticated as the tenant admin,  
  **When** I submit a data export request via the dashboard,  
  **Then** the platform acknowledges with HTTP 202 and begins archive generation without blocking my session.

- **Given** archive generation completes,  
  **When** the export is ready,  
  **Then** a signed S3 download URL is emailed to my verified `owner_email` address only — no other email address can receive it (FR012).

- **Given** the signed download URL is emailed,  
  **When** I attempt to access it more than 24 hours after it was generated,  
  **Then** the URL returns HTTP 403 Expired — I must request a new export if needed (FR012).

- **Given** I download the archive,  
  **When** I inspect its contents,  
  **Then** it includes: tenant configuration, services, staff members, all appointment events, payment records, notification records, and the audit log — all records scoped strictly to my `tenant_id` (FR012, BR005).

- **Given** the export is generated,  
  **When** the `tenant.data_exported` event is published,  
  **Then** the event includes the `signed_url`, `owner_email`, `tenant_id`, `event_id`, `correlation_id`, and `occurred_at` in the envelope (BR013).

**Size:** M  
**Priority:** Medium  
**Related:** FR012, BR005, BR013, BR014

**Technical Notes:**
- ops-service generates the archive by querying all tenant-scoped tables via RLS-enforced connections; the archive is written to the `tenant-exports` S3 bucket under `exports/{tenant_id}/{job_id}.zip`.
- The S3 pre-signed URL is generated with a 24-hour TTL and sent in the `tenant.data_exported` event payload consumed by notification-service.
- The export request is recorded in `AuditLog` with `operation = export`, `actor_id = admin_user_id`, `actor_role = admin` (BR014).
- Archive format: ZIP containing one JSON file per entity type (appointments.json, payments.json, etc.), each an array of records sorted by `created_at`.

---

### US-UJ007-02 · Ensure export is scoped strictly to own tenant

**Title:** Guarantee that the exported archive contains only the requesting tenant's data

**User Story:**
As a **Platform** upholding multi-tenancy guarantees, I want every data export to be cryptographically and logically scoped to the requesting tenant, so that cross-tenant data leakage is impossible even through the export mechanism.

**Acceptance Criteria:**

- **Given** the export archive is assembled,  
  **When** every record is selected,  
  **Then** only records with `tenant_id` matching the requesting tenant are included — PostgreSQL RLS enforces this at the database query level (BR005).

- **Given** a `platform_operator` user issues an export request for tenant A,  
  **When** the request is validated,  
  **Then** the operator cannot override the tenant scope to export data belonging to tenant B through the standard export endpoint (BR005).

- **Given** the archive download link is emailed,  
  **When** it is sent,  
  **Then** it is sent only to the `owner_email` address registered for the tenant — the ops console cannot override the delivery address for a tenant export request.

- **Given** the S3 object is created,  
  **When** its bucket policy is evaluated,  
  **Then** the object is private by default and accessible only via the pre-signed URL with the tenant-scoped key prefix `exports/{tenant_id}/`.

**Size:** S  
**Priority:** High  
**Related:** BR005

**Technical Notes:**
- All export queries run under `SET LOCAL app.tenant_id = {tenant_id}` — PostgreSQL RLS blocks any row whose `tenant_id` does not match.
- ops-service validates the `tenant_id` in the export request matches the `tid` claim in the requester's JWT; a mismatch returns HTTP 403 before any query runs.
- S3 bucket policy restricts object access to IAM roles held by ops-service; pre-signed URLs are generated with a scoped IAM session token per export job.

---

### US-UJ007-03 · Suspend a tenant — halt bookings while preserving read access

**Title:** Immediately block new bookings for a suspended tenant while keeping data readable for 30 days

**User Story:**
As a **Platform Operator**, I want to suspend a tenant's account and immediately halt new bookings, while preserving the tenant admin's read access for 30 days so they can export their data before deprovisioning begins.

**Acceptance Criteria:**

- **Given** a Platform Operator initiates tenant suspension via the ops console,  
  **When** the suspension request is processed,  
  **Then** `Tenant.status` transitions to `suspended` and all new booking requests for that tenant are rejected with HTTP 503 within 5 seconds.

- **Given** the tenant is suspended,  
  **When** the tenant admin attempts to log in,  
  **Then** they can access the dashboard in read-only mode — no configuration changes, no staff modifications, and no new bookings can be initiated (BR011).

- **Given** the tenant is suspended,  
  **When** the `tenant.suspended` event is published,  
  **Then** the event includes `data_export_deadline = occurred_at + 30 days` so the tenant admin knows exactly how long they have to export their data (BR011).

- **Given** the tenant is suspended,  
  **When** an existing customer attempts to cancel or reschedule a confirmed booking,  
  **Then** the cancellation and rescheduling operations are permitted — suspension only blocks new bookings.

**Size:** M  
**Priority:** Medium  
**Related:** FR013, BR011

**Technical Notes:**
- booking-command-service consumes `tenant.suspended` from its SQS queue and sets a local cache flag that causes immediate rejection of new booking requests for the tenant.
- Read-only enforcement is applied at the api-gateway route level: POST/PUT/DELETE routes for the suspended tenant return HTTP 503 with `TENANT_SUSPENDED`; GET routes are unaffected.
- The suspension event is published via the Transactional Outbox: `Tenant` status update and `OutboxRecord` INSERT occur in the same PostgreSQL transaction.
- `AuditLog` entry is written for the suspension with `operator_id`, `occurred_at`, and `before_snapshot = {status: active}`, `after_snapshot = {status: suspended}` (BR014).

---

### US-UJ007-04 · Deprovision a tenant after the 30-day window

**Title:** Permanently deprovision a tenant account only after the mandatory data export window has elapsed

**User Story:**
As a **Platform Operator**, I want to deprovision a suspended tenant only after their 30-day data access window has elapsed, so that no tenant is permanently cut off from their data before they have had a chance to export it.

**Acceptance Criteria:**

- **Given** a tenant has been suspended for fewer than 30 days,  
  **When** a Platform Operator attempts to initiate deprovisioning,  
  **Then** the request is rejected with HTTP 422 and the response includes `days_remaining` until the 30-day window closes (BR011).

- **Given** 30 days have elapsed since the tenant was suspended,  
  **When** the operator initiates deprovisioning,  
  **Then** `Tenant.status` transitions to `deprovisioned`, `deprovisioned_at` is set to `NOW()`, and `hard_deletion_at` is set to `NOW() + 90 days` (BR012).

- **Given** the tenant is deprovisioned,  
  **When** the `tenant.deprovisioned` event is published,  
  **Then** the event includes `hard_deletion_at` so downstream systems can schedule data deletion tasks.

- **Given** `hard_deletion_at` arrives,  
  **When** the data deletion job runs,  
  **Then** all tenant data is hard-deleted — except payment records (retained 7 years), and the `AuditLog` (retained permanently) (BR012).

**Size:** M  
**Priority:** Medium  
**Related:** FR013, BR011, BR012

**Technical Notes:**
- Deprovisioning guard: `NOW() < (Tenant.suspended_at + 30 days)` returns 422; enforcement is in ops-service before writing any state change.
- The hard-deletion scheduled job runs daily; it selects all `tenants WHERE status = deprovisioned AND hard_deletion_at <= NOW()` and executes cascading deletes in tenant-scoped tables.
- Payment records are moved to a compliance-retained schema partition before tenant data is deleted — retained for 7 years under a separate data governance policy.
- `AuditLog` rows are excluded from cascading deletes by their schema partition; they survive indefinitely even after tenant data is gone.

---

### US-UJ007-05 · Pseudonymise customer PII on erasure request

**Title:** Replace customer PII with pseudonymous tokens while preserving booking history integrity

**User Story:**
As a **Platform** meeting GDPR obligations, I want to pseudonymise a customer's personally identifiable information on erasure request while preserving the structural integrity of historical booking records, so that compliance is met without corrupting the event history.

**Acceptance Criteria:**

- **Given** a customer submits a GDPR erasure request for their data,  
  **When** the Platform Operator processes it via the ops console,  
  **Then** `Customer.name`, `Customer.email`, and `Customer.phone` are replaced with pseudonymous tokens and `Customer.pseudonymised_at` is set to `NOW()`.

- **Given** the customer record is pseudonymised,  
  **When** historical `AppointmentEvent` records referencing that `customer_id` are queried,  
  **Then** the events are retained intact — the `customer_id` foreign key is preserved but the PII it pointed to is gone.

- **Given** pseudonymisation is complete,  
  **When** notification-service or any downstream service queries for the customer's contact details,  
  **Then** it receives the pseudonymous token — no real PII is returned.

- **Given** the pseudonymisation operation,  
  **When** it is executed,  
  **Then** an `AuditLog` entry is written with `operation = pseudonymise`, `entity_type = Customer`, `entity_id = customer_id`, `actor_role = operator`, and `occurred_at` (BR014).

**Size:** M  
**Priority:** Medium  
**Related:** FR013, BR014

**Technical Notes:**
- Pseudonymous token format: `REDACTED-{sha256(customer_id + secret_salt)}` — deterministic per customer but not reversible without the salt.
- The erasure operation updates only `Customer.name`, `Customer.email`, `Customer.phone`, and `Customer.pseudonymised_at` — no `AppointmentEvent` records are modified (they are INSERT-only).
- Notification content derived from `AppointmentEvent` payloads that already contain the PII snapshot will need to be handled separately — future-dated events for this customer are cancelled at the booking level before pseudonymisation is confirmed.
- `AuditLog` for pseudonymisation itself does not contain the pre-pseudonymisation PII values in `before_snapshot` — it logs the fact of erasure and the timestamp only, to avoid storing the erased PII in the audit log indefinitely.

---

## Definition of Ready

Before any story in this epic enters a sprint, confirm all items below are checked:

- [ ] Acceptance criteria reviewed and agreed by Product Owner and Data Protection Officer
- [ ] `tenant.suspended`, `tenant.deprovisioned`, and `tenant.data_exported` event schemas finalised in `specs/ai/events.json`
- [ ] S3 export bucket naming, key prefix, and IAM policy confirmed with infrastructure team
- [ ] Pre-signed URL TTL (24 hours) confirmed against S3 pre-signed URL maximum duration limits
- [ ] 30-day suspension guard logic confirmed — `suspended_at` timestamp precision and comparison formula agreed (BR011)
- [ ] Hard-deletion job schedule confirmed — daily scan of `deprovisioned` tenants with elapsed `hard_deletion_at` (BR012)
- [ ] Payment record 7-year retention schema partition confirmed with compliance team (BR012)
- [ ] Pseudonymisation token format and salt rotation strategy approved by Data Protection Officer
- [ ] AuditLog retention confirmed: survives hard-deletion job; stored in separate schema partition (BR014)
- [ ] RLS policy validated: export queries cannot return cross-tenant records even for platform_operator role (BR005)
- [ ] Definition of Done agreed for this layer (see `documentation/00-governance/G5-definition-of-done.md`)
