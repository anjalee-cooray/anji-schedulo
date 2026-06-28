---
title: Flow Spec — Tenant Data Export and Offboarding
journeyId: UJ007
layer: 02-design
status: current
lastUpdated: 2026-06-28
---

# D2 · Flow Spec — UJ007: Tenant Data Export and Offboarding

## Header

| Field | Value |
|---|---|
| Journey ID | UJ007 |
| Journey Name | Tenant Data Export and Offboarding |
| Personas | P1 — Tenant Admin (export), P4 — Platform Operator (deprovisioning) |
| Priority | Medium |
| Related Functional Requirements | FR012, FR013 |
| Related Business Rules | BR005, BR011, BR012, BR014 |

---

## 1. Technical Flow Overview

**Architecture Pattern: Ops Service Scoped Export + Tenant Status Machine Deprovisioning**

Offboarding is a multi-phase, operator-gated process with strict data residency obligations. It spans four distinct sub-flows:

1. **Data Export (P1-initiated):** The tenant admin requests a full export of their data at any time. The export is scoped strictly to their `tenant_id` (BR005). A signed S3 URL is delivered to their verified owner email only (FR012).

2. **Suspension (P4-initiated):** The Platform Operator suspends the tenant — halting new bookings while preserving read-only access for 30 days so the admin can export their data (BR011).

3. **Deprovisioning (P4-initiated, after 30-day window):** The operator confirms deprovisioning, triggering the retention countdown. Data is retained for 90 days before hard deletion (BR012). Payment records and the audit log are exempt from deletion.

4. **GDPR Customer Erasure (Customer-initiated, separate from offboarding):** Any customer can request erasure of their PII at any time. This pseudonymises their data without deleting historical appointment records.

The Transactional Outbox pattern (NFR003) is used for all status change events. Every operator and admin action produces an immutable AuditLog entry (BR014).

---

## 2. Services Involved

| Service | Role in This Flow |
|---|---|
| api-gateway | Validates JWT for both tenant_admin (export) and platform_operator (suspend/deprovision) requests |
| ops-service | Orchestrates export generation, S3 upload, signed URL creation, suspension, and deprovisioning |
| tenant-service | Owns the Tenant status machine. Executes status transitions (provisioning → active → suspended → deprovisioned) |
| outbox-relay | Publishes tenant.data_exported, tenant.suspended, tenant.deprovisioned events from outbox to SNS |
| notification-service | Consumes tenant.data_exported → sends signed URL download link email to owner |
| booking-command-service | Consumes tenant.suspended → halts new booking acceptance for the tenant |
| Amazon S3 | tenant-exports bucket for export archive storage and pre-signed URL generation |

---

## 3. Step-by-Step Technical Flow

### Phase A — Data Export (P1-Initiated)

**Step 1 — Tenant admin requests data export**

The tenant admin (P1) calls:
```
POST /api/v1/tenants/{tenant_id}/export
Authorization: Bearer <JWT>
```

api-gateway validates:
- `role = tenant_admin`
- `tid = tenant_id` in the path — the JWT `tid` claim must match exactly. Cross-tenant export requests are rejected with `403 Forbidden` (BR005).
- Injects `X-Correlation-ID` and forwards to ops-service.

**Step 2 — ops-service generates the export archive**

ops-service queries PostgreSQL with RLS context `SET LOCAL app.tenant_id = $tenant_id` to enforce strict data scoping (BR005). The export is built by joining all tenant-scoped tables:

```sql
-- Each query is executed separately; combined into one structured JSON export
SELECT * FROM tenants           WHERE tenant_id = $tenant_id;
SELECT * FROM tenant_config     WHERE tenant_id = $tenant_id;
SELECT * FROM services          WHERE tenant_id = $tenant_id;
SELECT * FROM staff_members     WHERE tenant_id = $tenant_id;
SELECT * FROM working_hours     WHERE tenant_id = $tenant_id;
SELECT * FROM staff_blocks      WHERE tenant_id = $tenant_id;
SELECT * FROM appointments      WHERE tenant_id = $tenant_id;
SELECT * FROM appointment_events WHERE tenant_id = $tenant_id;
SELECT * FROM payments          WHERE tenant_id = $tenant_id;
SELECT * FROM notification_records WHERE tenant_id = $tenant_id;
SELECT * FROM audit_log         WHERE tenant_id = $tenant_id;
-- NOT included: inbox_records (infrastructure), outbox_records (infrastructure)
```

The result is serialised as a structured JSON document with a top-level schema version field and one key per table.

**Step 3 — Archive uploaded to S3**

ops-service uploads the archive to the `tenant-exports` S3 bucket:
```
s3://tenant-exports/{tenant_id}/{export_timestamp}/export.json
```

S3 configuration:
- Server-side encryption: AWS KMS (`alias/tenant-exports-key`)
- Lifecycle policy: Object expires after 90 days (FR013 — 90-day retention post-deprovisioning alignment)
- Bucket policy: `tenant-exports` is private; no public access; signed URL is the only access mechanism

**Step 4 — Pre-signed URL generated**

ops-service generates an S3 pre-signed GET URL valid for 24 hours (FR012):
```python
presigned_url = s3.generate_presigned_url(
    'get_object',
    Params={'Bucket': 'tenant-exports', 'Key': object_key},
    ExpiresIn=86400  # 24 hours
)
```

**Step 5 — ops-service writes OutboxRecord and returns HTTP 202**

Within a single PostgreSQL transaction (RLS context: operator scope — system-level):
1. Inserts `outbox_records` row:
   ```
   topic = 'tenant-data-exported-topic',
   event_type = 'tenant.data_exported',
   payload = {
     event_id, tenant_id, correlation_id, occurred_at,
     owner_email, signed_url, export_includes: [...table list...]
   }
   ```
2. Inserts `audit_log` row:
   ```
   entity_type = 'Tenant', entity_id = $tenant_id,
   operation = 'export',
   actor_id = $admin_user_id, actor_role = 'admin',
   occurred_at = NOW(),
   after_snapshot = { export_timestamp, s3_object_key, tables_included }
   ```

Returns HTTP `202 Accepted`:
```json
{
  "message": "Export generating — a download link will be sent to your registered email address within 60 seconds.",
  "export_timestamp": "2026-06-28T09:00:00Z",
  "link_expires_in_hours": 24
}
```

**Step 6 — notification-service delivers the download link**

outbox-relay publishes `tenant.data_exported` to `tenant-data-exported-topic` → `notification-service-tenant-data-exported-queue`.

notification-service:
1. InboxRecord deduplication check.
2. Reads `owner_email` and `signed_url` from event payload (Event-Carried State Transfer).
3. Sends a transactional email via SendGrid containing the download link with clear expiry warning (24 hours).
4. Email is sent **only to `owner_email` from the tenant record** — no forwarding, no sharing (FR012).
5. Writes `notification_records` row.

---

### Phase B — Suspension (P4-Initiated)

**Step 7 — Platform Operator initiates suspension**

The Platform Operator (P4) calls:
```
POST /api/v1/ops/tenants/{tenant_id}/suspend
Authorization: Bearer <JWT>   (role = platform_operator)
{
  "reason": "Non-payment of subscription",
  "operator_note": "Stripe subscription lapsed. Grace period expired."
}
```

api-gateway validates `role = platform_operator`. Platform operator tokens have no `tid` claim — they have cross-tenant write scope scoped only by explicit tenant_id in the path.

ops-service validates: `tenant.status = active`. If already suspended or deprovisioned → `422 Unprocessable Entity` with current status.

**Step 8 — tenant-service executes the suspension**

Within a single PostgreSQL transaction:
1. `UPDATE tenants SET status = 'suspended', suspended_at = NOW() WHERE tenant_id = $tenant_id AND status = 'active'`. Uses optimistic locking on the `status` field — if another process already suspended the tenant, the UPDATE affects 0 rows and the transaction rolls back.
2. Inserts `outbox_records` row:
   ```
   event_type = 'tenant.suspended',
   payload = {
     event_id, tenant_id, correlation_id, occurred_at,
     suspended_by = $operator_user_id,
     reason = $reason,
     data_export_deadline = NOW() + INTERVAL '30 days'  (BR011)
   }
   ```
3. Inserts `audit_log` entry:
   ```
   operation = 'suspend', actor_id = $operator_user_id, actor_role = 'operator',
   before_snapshot = { status: 'active' }, after_snapshot = { status: 'suspended', suspended_at, reason }
   ```

Returns HTTP `200 OK`:
```json
{
  "tenant_id": "...",
  "status": "suspended",
  "suspended_at": "2026-06-28T09:15:00Z",
  "data_export_deadline": "2026-07-28T09:15:00Z",
  "message": "Tenant suspended. Admin retains read-only access until 2026-07-28."
}
```

**Step 9 — booking-command-service halts new bookings**

outbox-relay publishes `tenant.suspended` → `booking-command-service-tenant-suspended-queue`.

booking-command-service:
1. InboxRecord deduplication check.
2. Updates its local `TenantConfig` cache: marks tenant as `suspended`.
3. From this point, all new booking requests for this `tenant_id` return `403 Forbidden` with `reason: tenant_suspended`. Existing confirmed bookings are unaffected — they can still be viewed and cancelled.

The tenant admin can still:
- Read all their appointments, settings, and dashboard
- Request a data export (Phase A — always available during suspension, BR011)
- Cancel existing confirmed bookings

The tenant admin cannot:
- Accept new bookings
- Modify services or staff (configuration changes blocked during suspension)

---

### Phase C — Deprovisioning (P4-Initiated, After 30-Day Window)

**Step 10 — Platform Operator confirms deprovisioning**

The Platform Operator calls:
```
POST /api/v1/ops/tenants/{tenant_id}/deprovision
Authorization: Bearer <JWT>   (role = platform_operator)
{
  "operator_note": "30-day window expired. Admin downloaded export on 2026-07-15. Proceeding with deprovisioning."
}
```

ops-service validates:
1. `tenant.status = suspended` — if not suspended → `422 Unprocessable Entity`.
2. `NOW() >= tenant.suspended_at + INTERVAL '30 days'` (BR011). If the 30-day window has not elapsed → `422 Unprocessable Entity`:
   ```json
   {
     "error": "deprovisioning_too_early",
     "message": "The 30-day data export window has not yet elapsed.",
     "data_export_deadline": "2026-07-28T09:15:00Z",
     "days_remaining": 17
   }
   ```

**Step 11 — tenant-service executes the deprovisioning**

Within a single PostgreSQL transaction:
1. `UPDATE tenants SET status = 'deprovisioned', deprovisioned_at = NOW() WHERE tenant_id = $tenant_id AND status = 'suspended'`.
2. Inserts `outbox_records` row:
   ```
   event_type = 'tenant.deprovisioned',
   payload = {
     event_id, tenant_id, correlation_id, occurred_at,
     deprovisioned_by = $operator_user_id,
     hard_deletion_at = NOW() + INTERVAL '90 days'  (BR012)
   }
   ```
3. Inserts `audit_log` entry:
   ```
   operation = 'deprovision', actor_id = $operator_user_id, actor_role = 'operator',
   before_snapshot = { status: 'suspended' }, after_snapshot = { status: 'deprovisioned', deprovisioned_at, hard_deletion_at }
   ```

**Step 12 — ops-service schedules hard deletion**

outbox-relay publishes `tenant.deprovisioned` → `ops-service-tenant-deprovisioned-queue`.

ops-service receives the event and schedules a hard deletion job for `hard_deletion_at`:
```
scheduled_job:
  run_at = hard_deletion_at (90 days post-deprovisioning)
  action = hard_delete_tenant_data(tenant_id)
```

The hard deletion job deletes data in the following order:
1. `notification_records` WHERE tenant_id = $tenant_id
2. `appointment_events` WHERE tenant_id = $tenant_id
3. `appointments` WHERE tenant_id = $tenant_id
4. `slot_locks` WHERE tenant_id = $tenant_id
5. `working_hours` WHERE tenant_id = $tenant_id
6. `staff_blocks` WHERE tenant_id = $tenant_id
7. `staff_members` WHERE tenant_id = $tenant_id
8. `services` WHERE tenant_id = $tenant_id
9. `customers` WHERE tenant_id = $tenant_id
10. `tenant_config` WHERE tenant_id = $tenant_id
11. `tenants` WHERE tenant_id = $tenant_id (the row itself)
12. S3 lifecycle policy expires export archives after 90 days automatically

**Exempt from hard deletion (BR012 exceptions):**
- `payments` WHERE tenant_id = $tenant_id — retained 7 years from payment capture date
- `audit_log` WHERE tenant_id = $tenant_id — retained permanently under system-scoped partition

---

### Phase D — GDPR Customer Erasure (Customer-Initiated, Any Time)

**Step 13 — Customer requests erasure**

A customer submits a GDPR erasure request via the self-serve link in any booking confirmation email, or by contacting the tenant directly:
```
DELETE /api/v1/tenants/{tenant_id}/customers/{customer_id}
Authorization: Bearer <JWT>   (role = customer, sub = customer_id)
```

api-gateway validates: `role = customer` AND `sub = customer_id` (own record only).

**Step 14 — booking-command-service pseudonymises the Customer record**

Within a single PostgreSQL transaction:
1. `UPDATE customers SET name = 'Anonymised Customer', email = 'erased-{uuid}@erasure.internal', phone = NULL, pseudonymised_at = NOW() WHERE customer_id = $customer_id AND tenant_id = $tenant_id`.
2. Inserts `audit_log` entry:
   ```
   operation = 'gdpr_erasure', actor_id = $customer_id, actor_role = 'customer',
   before_snapshot = { name, email, phone, pseudonymised_at: null },
   after_snapshot  = { name: 'Anonymised Customer', email: 'erased-...', phone: null, pseudonymised_at: NOW() }
   ```

**What is preserved:**
- `appointment_events` retain the `customer_id` UUID as a foreign key — the historical booking record is intact for audit and compliance purposes.
- `payments` retain the `customer_id` FK — payment history is preserved for 7 years.
- `notification_records` retain the `customer_id` FK.
- The `customer_id` UUID itself is not erased — only the PII (name, email, phone) is replaced with pseudonymous tokens.

**What is erased:**
- Customer name, email address, and phone number are pseudonymised immediately and irreversibly.

Returns HTTP `200 OK`:
```json
{
  "message": "Your personal data has been pseudonymised. Your booking history remains for operational and compliance purposes."
}
```

---

## 4. Compensation / Failure Flows

### Failure Point 1 — Export S3 upload fails (Step 3)

**Impact:** Archive cannot be stored; signed URL cannot be generated. No OutboxRecord is written (transaction not yet started). Admin is waiting for their download link.

**Compensation:** ops-service retries S3 upload up to 3 times with exponential backoff (1s, 2s, 4s). If all retries fail → returns `503 Service Unavailable` to the admin with `Retry-After: 60` header. No partial state — no OutboxRecord, no AuditLog entry is written. The admin can safely retry the export request.

### Failure Point 2 — Signed URL expires before download (Step 4)

**Impact:** Admin clicks the email link after 24 hours and receives an S3 `AccessDenied` error.

**Compensation:** The S3 object itself is not deleted until the 90-day lifecycle policy expires. The admin simply submits a new export request — a fresh signed URL is generated in a new archive (or the same object key if exports are idempotent within a time window). No data is lost.

### Failure Point 3 — Deprovisioning attempted before 30-day window (Step 10)

**Impact:** Operator mistakenly calls the deprovision endpoint before the 30-day window.

**Compensation:** ops-service validation rejects the request with `422 Unprocessable Entity` and includes `days_remaining` in the response. The tenant status remains `suspended`. No state change occurs.

### Failure Point 4 — Hard deletion scheduler fails (Step 12)

**Impact:** The scheduled hard deletion job does not run on `hard_deletion_at`. Data is retained longer than required — this is not a privacy violation (data is not exposed, just not yet deleted) but it is a compliance drift.

**Compensation:** The hard deletion job failure routes to ops-service DLQ. Operator receives an alert. Operator can manually trigger the deletion via `POST /ops/tenants/{tenant_id}/hard-delete` from the ops console. The deletion is idempotent — re-running against already-deleted rows produces no error.

### Failure Point 5 — tenant.suspended consumer (booking-command-service) is down (Step 9)

**Impact:** New booking requests for the suspended tenant continue to be accepted until booking-command-service processes `tenant.suspended` and updates its cache.

**Compensation:** SQS FIFO retains the `tenant.suspended` message with `maxReceiveCount = 4`. When booking-command-service recovers, it processes the event and blocks new bookings. The tenant.suspended event has `data_export_deadline` in its payload — if the delay causes any bookings to be accepted post-suspension, the operator can cancel those bookings from the ops console and issue refunds directly.

---

## 5. Events Produced

| Event Name | Producer | Consumers | SNS Topic |
|---|---|---|---|
| `tenant.data_exported` | ops-service | notification-service | `tenant-data-exported-topic` |
| `tenant.suspended` | tenant-service (via outbox) | booking-command-service, notification-service (optional) | `tenant-suspended-topic` |
| `tenant.deprovisioned` | tenant-service (via outbox) | ops-service (schedules hard deletion) | `tenant-deprovisioned-topic` |

---

## 6. Business Rules Enforced

| BR ID | Rule Summary | Enforcement Point |
|---|---|---|
| BR005 | Tenant data must never be readable by another tenant | ops-service: `SET LOCAL app.tenant_id` before every export query; RLS enforces scoping at DB layer; api-gateway validates `tid` claim matches path `tenant_id` |
| BR011 | Suspended tenant data must remain read-accessible for 30 days before deprovisioning | ops-service: validates `NOW() >= suspended_at + 30 days` before executing deprovisioning; rejects early with 422 and `days_remaining` |
| BR012 | All tenant data retained for 90 days post-deprovisioning before hard deletion; payments 7 years; audit log permanent | ops-service: `hard_deletion_at = deprovisioned_at + 90 days` enforced in scheduled deletion job; payment and audit tables explicitly excluded from hard deletion |
| BR014 | All write operations on tenant-scoped data must produce an immutable audit log entry | ops-service: AuditLog INSERT in same transaction as every status change (suspend, deprovision, export, gdpr_erasure) |

---

## 7. Consistency Guarantees

| Concern | Consistency Model | Detail |
|---|---|---|
| Export archive generation | Strongly consistent read | RLS-scoped queries at a single point in time; no concurrent modification guarantee (new bookings may occur during generation) |
| Signed URL delivery | Eventually consistent | notification-service is async subscriber; delivery within 60 seconds under normal load |
| Tenant suspension | Strong | Single PostgreSQL transaction with optimistic status check; commit or rollback |
| Booking halt on suspension | Eventually consistent | booking-command-service processes tenant.suspended from SQS; lag < 5 seconds under normal load |
| Deprovisioning execution | Strong | Status transition in single transaction; 30-day window validated before commit |
| Hard data deletion | Eventually consistent | Scheduled job; may run with a delay if scheduler is degraded |
| GDPR pseudonymisation | Strong | Single PostgreSQL transaction; PII replaced atomically |

---

## 8. Idempotency

### Export request idempotency

Export requests are not deduplicated — each request generates a fresh archive and a new signed URL. Multiple exports within a short window are allowed and each produces a new S3 object. This is intentional — the admin may request an export after making configuration changes and expect the fresh archive to reflect those changes.

### Suspension idempotency

The suspension UPDATE uses `WHERE status = 'active'` — if the tenant is already suspended, the UPDATE affects 0 rows, the transaction rolls back, and ops-service returns `422 Unprocessable Entity`. This prevents duplicate suspension events and double AuditLog entries.

### Deprovisioning idempotency

The deprovisioning UPDATE uses `WHERE status = 'suspended'` — if already deprovisioned, 0 rows are affected. The 30-day window validation also rejects re-attempts with a clear error.

### Hard deletion idempotency

DELETE statements on already-deleted rows are safe no-ops in PostgreSQL. The hard deletion job can be run multiple times without side effects.

### tenant.deprovisioned event idempotency

ops-service uses InboxRecord deduplication on `event_id` before scheduling the hard deletion job. A re-delivered `tenant.deprovisioned` event does not schedule a second deletion job.

### GDPR erasure idempotency

If `pseudonymised_at IS NOT NULL`, the record has already been pseudonymised. ops-service checks this before executing — a second erasure request is a safe no-op that returns `200 OK`.
