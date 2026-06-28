---
title: Journey — Tenant Data Export and Offboarding
journeyId: UJ007
persona: P1
priority: medium
layer: 01-requirements
status: current
lastUpdated: 2026-06-28
---

# R6 · Journey UJ007 — Tenant Data Export and Offboarding

## Metadata

| Field | Value |
|---|---|
| Journey ID | UJ007 |
| Name | Tenant Data Export and Offboarding |
| Primary Persona | [P1 — Tenant Admin](../personas/R4a-persona-P1.md) (export); [P4 — Platform Operator](../personas/R4a-persona-P4.md) (deprovisioning) |
| Priority | Medium |
| Related Functional Requirements | FR012, FR013 |
| Related Business Rules | BR005, BR011, BR012, BR014 |
| Architecture Pattern | Ops Service scoped export + status machine deprovisioning |
| Services Involved | ops-service, tenant-service, notification-service |

---

## Goal

A departing tenant admin receives a complete, scoped export of all their data and is then cleanly offboarded from the platform — with a 30-day read-only window before deprovisioning, data retained for 90 days after deprovisioning, and no residual PII or tenant data remaining after the retention period expires.

---

## Preconditions

- The tenant is in `active` or `suspended` status.
- The tenant admin has authenticated with a valid JWT containing `role = tenant_admin`.
- The Platform Operator (P4) is available to initiate deprovisioning after the 30-day window.
- The ops-service has write access to the S3 `tenant-exports` bucket.

---

## Step-by-Step Journey

### Step 1 — Tenant Admin Requests a Full Data Export

The tenant admin navigates to the Account Settings section of the dashboard and selects "Export my data". They confirm the request. The admin receives a message that the export is being prepared and they will receive a download link by email when it is ready.

The request reaches ops-service, which begins generating the archive.

**Outcome:** An export job is initiated. The admin does not need to wait on screen — they will be notified by email.

---

### Step 2 — Platform Generates Structured Archive

ops-service generates a structured data archive scoped strictly to the requesting tenant's `tenant_id` (BR005). The archive includes:
- Tenant configuration (business hours, services, staff)
- All appointment events (the complete AppointmentEvent history)
- Payment records (within scope for data portability; retained separately for 7 years per BR012)
- Notification delivery records
- Audit log entries scoped to the tenant

The archive is written to the S3 `tenant-exports` bucket in the tenant's designated prefix (`exports/{tenant_id}/{job_id}/`). Server-side encryption (AES-256) is applied at rest.

**Outcome:** A complete, tenant-scoped archive is stored securely in S3. No other tenant's data is included.

---

### Step 3 — Signed Download Link Sent to Verified Owner Email

ops-service generates an S3 pre-signed URL for the archive object. The URL expires in 24 hours. ops-service publishes a `tenant.data_exported` event via the Transactional Outbox.

notification-service consumes the event and sends an email to the verified owner email address on record — not the requester's session email, but the `owner_email` field on the Tenant record — containing the download link and an expiry notice.

**Outcome:** The download link is in the admin's inbox. The link is valid for 24 hours. No other party can access the link.

---

### Step 4 — Tenant Admin Downloads Their Data

The admin clicks the link in the email and downloads the archive. If the link has expired (24-hour window missed), they can request a new export from the same settings page — each export generates a fresh pre-signed URL.

**Outcome:** The tenant has a local copy of all their data in a structured format. They are now ready to complete offboarding.

---

### Step 5 — Platform Operator Initiates Suspension

After the tenant confirms they wish to proceed with offboarding (or after a payment failure triggers suspension), the Platform Operator initiates suspension via the ops console. tenant-service transitions the tenant status from `active` → `suspended` (BR011).

From this point:
- New bookings are rejected — the booking-command-service receives the `tenant.suspended` event and halts saga initiation for this tenant.
- The tenant admin retains read-only access to their dashboard and data for 30 days.
- Existing confirmed appointments are not cancelled; the admin must handle any outstanding appointments manually.

**Outcome:** The tenant cannot accept new bookings. Their data remains readable and exportable for 30 days.

---

### Step 6 — After the 30-Day Window, Operator Initiates Deprovisioning

After the 30-day suspension window has elapsed, the Platform Operator confirms in the ops console that the window has passed and initiates deprovisioning. The platform enforces this — an attempt to deprovision before 30 days is rejected with a clear error.

tenant-service transitions the tenant status from `suspended` → `deprovisioned`. A `tenant.deprovisioned` event is published via the Transactional Outbox. ops-service receives the event and schedules a hard deletion job for 90 days from `deprovisioned_at` (BR012).

**Outcome:** The tenant is deprovisioned. Their booking page returns 404. Admin access is revoked. A hard deletion is scheduled.

---

### Step 7 — Data Retained for 90 Days, Then Permanently Deleted

After 90 days from `deprovisioned_at`, the scheduled hard deletion job runs:
- All tenant-scoped rows are deleted from all tables (except audit_log, which is retained permanently, and payment_records, which are retained for 7 years — BR012).
- The S3 export objects are deleted.
- Customer PII (name, email, phone) is pseudonymised or deleted per GDPR erasure requirements.

**Outcome:** The platform holds no residual PII or operational data for the deprovisioned tenant, beyond the audit log and payment records required for legal compliance.

---

## Success Outcome

The tenant admin receives their full data export by email within a reasonable window, downloads the archive, and departs. After 90 days from deprovisioning, the platform holds no residual data for the tenant except the permanent audit log and legally required payment records. The entire process required no engineering involvement — the Platform Operator executed it entirely through the ops console.

---

## Failure Paths

### Export Generation Fails

If ops-service fails to generate the archive (e.g. a database query timeout on a large tenant dataset), the export job fails. The admin does not receive a download link. The admin can request a new export from the settings page — the previous failed job is logged in the audit record with the failure reason.

**Platform behaviour:** Export job status is recorded as `failed`. Operator receives an alert. Retry the export after investigating the root cause (e.g. increase query timeout, add pagination).

### Signed URL Expires Before Download

If the admin does not download their archive within 24 hours of the email, the pre-signed URL expires. The admin can request a new export from the same settings page at any time before deprovisioning. A new archive is generated and a new 24-hour link is issued.

**Platform behaviour:** No data loss. The S3 object remains in the `tenant-exports` bucket until the tenant is deprovisioned and the retention period expires.

### Deprovisioning Attempted Before 30-Day Window

If the Platform Operator attempts to initiate deprovisioning before 30 days have elapsed since suspension, the ops-service rejects the action with a clear error message and the number of days remaining in the window.

**Platform behaviour:** No state change. Operator is informed of the remaining wait period. BR011 is enforced by the status machine — no exceptions.

### PII Erasure Request During Retention Period

If a customer submits a GDPR erasure request during the 90-day post-deprovisioning retention period, ops-service pseudonymises the customer's PII fields (name, email, phone) immediately, even though the tenant's data has not yet been hard deleted. The `pseudonymised_at` timestamp is recorded on the Customer record. Historical appointment event records retain the `customer_id` foreign key but the PII is gone.

**Platform behaviour:** Pseudonymisation is immediate. Appointment history integrity is preserved via foreign key. Audit log records the erasure action.

---

## Related Business Rules

| Rule ID | Rule Summary | Application in This Journey |
|---|---|---|
| BR005 | A tenant's data must never be readable by another tenant. | The export archive is generated using queries scoped to `tenant_id = :tid`. RLS policies enforce this at the database layer — the ops-service query cannot accidentally include another tenant's rows even if a filter is missing at the application level. |
| BR011 | A suspended tenant's data must remain read-accessible for 30 days before deprovisioning. | Status machine enforces: `suspended` → `deprovisioned` transition is rejected before 30 days. The ops console shows the exact date the window elapses. |
| BR012 | All tenant data must be retained for a minimum of 90 days after deprovisioning before hard deletion. | Hard deletion is scheduled, not immediate. Payment records have a 7-year minimum retention. Audit logs are retained permanently. |
| BR014 | All write operations that modify tenant-scoped data must produce an immutable audit log entry. | Every ops action — suspension, deprovisioning, export generation, hard deletion, PII pseudonymisation — produces an `audit_log` entry with `actor_role = operator` and full before/after snapshots where applicable. |

---

## Implementation Note

ops-service generates the export archive by reading directly from PostgreSQL with RLS active, using a service account scoped to the requesting `tenant_id`. The archive is written to `s3://tenant-exports/{tenant_id}/{job_id}/export.zip` with server-side AES-256 encryption. A pre-signed URL is generated using the S3 `GeneratePresignedUrl` API with a 24-hour expiry and `Content-Disposition: attachment` to force download. The `tenant.data_exported` event is published via the Transactional Outbox to ensure the notification email is sent at-least-once regardless of transient failures. Deprovisioning is a status machine transition enforced in tenant-service: the `suspended` → `deprovisioned` transition validates `suspended_at + 30 days ≤ NOW()` before committing. The hard deletion job is a scheduled task in ops-service with a `run_after = deprovisioned_at + 90 days` field on the job record, polled hourly by a background worker.
