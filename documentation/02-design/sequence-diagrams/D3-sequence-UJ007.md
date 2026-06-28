---
title: Sequence Diagram — Tenant Data Export and Offboarding
journeyId: UJ007
layer: 02-design
status: current
lastUpdated: 2026-06-28
---

# D3 · Sequence Diagram — UJ007: Tenant Data Export and Offboarding

## Overview

This diagram shows two linked sub-flows:

1. **Data Export (self-service):** The tenant admin requests a full data archive. ops-service generates the archive, uploads it to S3, and sends a pre-signed download link to the verified owner email.
2. **Deprovisioning (operator-initiated):** After the 30-day suspension window, the Platform Operator (P4) initiates the deprovisioning sequence. This halts all new bookings, transitions the tenant to `deprovisioned`, and schedules hard data deletion 90 days later (BR012).

---

```mermaid
sequenceDiagram
    participant Admin as Tenant Admin (P1)
    participant GW as api-gateway
    participant OPS as ops-service
    participant S3 as AWS S3<br/>(tenant-exports bucket)
    participant OR as outbox-relay
    participant SNS as SNS
    participant NS as notification-service
    participant SG as SendGrid
    participant DB as PostgreSQL
    participant TS as tenant-service
    participant BCS as booking-command-service
    participant Operator as Platform Operator (P4)

    Note over Admin,DB: Sub-flow 1 — Data Export (Self-service, any time)

    Admin->>+GW: POST /api/v1/tenants/{id}/export<br/>Bearer JWT (role=tenant_admin)
    GW->>GW: Validate JWT, verify tid=tenant_id (tenant can only export own data — BR005)
    GW->>+OPS: Forward export request

    OPS->>OPS: Verify tenant status ≠ deprovisioned (hard-deleted data not exportable)
    OPS->>OPS: Create export job record (status=running)

    Note over OPS,DB: Scoped export query — all tables WHERE tenant_id=?
    OPS->>+DB: SELECT tenants, tenant_config, working_hours,<br/>staff_members, services, appointments,<br/>appointment_events, payments,<br/>notification_records<br/>WHERE tenant_id=? (RLS enforced — BR005)
    DB-->>-OPS: Full tenant dataset

    OPS->>OPS: Serialize to structured archive (JSON + CSV)
    OPS->>OPS: Encrypt archive (AES-256)

    OPS->>+S3: PutObject → tenant-exports/{tenant_id}/{timestamp}.zip
    S3-->>-OPS: ETag (upload confirmed)

    OPS->>+S3: GeneratePresignedUrl (expires in 24 hours — FR012)
    S3-->>-OPS: signed_url

    OPS->>DB: INSERT audit_log (op=export, actor_id=admin, tenant_id, occurred_at)

    Note over OPS: BEGIN TRANSACTION
    OPS->>DB: UPDATE export_job (status=completed)
    OPS->>DB: INSERT outbox_records (tenant.data_exported,<br/>{owner_email, signed_url, export_includes[]})
    Note over OPS: COMMIT

    OPS-->>-GW: 202 Accepted {job_id, estimated_delivery: <30s}
    GW-->>-Admin: 202 Accepted — download link will be emailed

    loop Every 500ms
        OR->>OPS: Poll outbox_records WHERE published_at IS NULL
    end

    OR->>+SNS: Publish tenant.data_exported
    SNS-->>-OR: MessageId

    SNS->>NS: SQS: notification-service-tenant-data-exported-queue

    NS->>NS: INSERT inbox_records (event_id, consumer=notification-service)
    NS->>NS: Read owner_email, signed_url from payload (ECST — no back-call needed)
    NS->>+SG: Send download link email to verified owner_email<br/>(signed URL expires in 24 hours)
    SG-->>-NS: 202 Accepted
    NS->>NS: INSERT notification_records (status=sent)

    Admin->>Admin: Receives email → downloads archive

    Note over Admin,Operator: ─────────────────────────────────────────────────────────
    Note over Admin,Operator: Sub-flow 2 — Deprovisioning (Operator-initiated, after 30-day window)

    Operator->>+GW: POST /ops/tenants/{id}/suspend<br/>Bearer JWT (role=platform_operator)<br/>{reason: "non-payment / voluntary offboarding"}
    GW->>GW: Validate JWT, verify role=platform_operator
    GW->>+TS: Forward suspension request

    Note over TS: BEGIN TRANSACTION
    TS->>DB: UPDATE tenants SET status=suspended,<br/>data_export_deadline=NOW()+30days
    TS->>DB: INSERT audit_log (op=suspend, actor_id=operator, tenant_id)
    TS->>DB: INSERT outbox_records (tenant.suspended, {data_export_deadline})
    Note over TS: COMMIT

    TS-->>-GW: 200 OK {tenant_id, status: suspended, export_deadline}
    GW-->>-Operator: 200 OK — tenant suspended (read-only mode for 30 days — BR011)

    OR->>SNS: Publish tenant.suspended
    SNS->>BCS: SQS: booking-command-service-tenant-suspended-queue
    SNS->>NS: SQS: notification-service-tenant-suspended-queue

    Note over BCS: Halt new booking requests for this tenant
    BCS->>BCS: INSERT inbox_records (event_id, consumer=booking-command-service)
    BCS->>BCS: UPDATE local tenant_status_cache (tenant_id → suspended)<br/>All new POST /bookings for this tenant return 403 Forbidden

    Note over NS: Optionally notify tenant admin of suspension (v1 optional)

    Note over Operator,DB: ─── 30 days later (after data_export_deadline) ───

    Operator->>+GW: POST /ops/tenants/{id}/deprovision<br/>Bearer JWT (role=platform_operator)
    GW->>+TS: Forward deprovisioning request

    TS->>+DB: SELECT tenants WHERE tenant_id=?<br/>Validate: status=suspended AND NOW() ≥ data_export_deadline
    DB-->>-TS: Tenant record

    alt 30-day window not yet elapsed
        TS-->>GW: 422 Unprocessable Entity<br/>{error: "export_window_not_elapsed", deadline: ISO8601}
        GW-->>-Operator: 422 — deprovisioning blocked until export deadline
    end

    Note over TS: BEGIN TRANSACTION
    TS->>DB: UPDATE tenants SET status=deprovisioned,<br/>deprovisioned_at=NOW(),<br/>hard_deletion_scheduled_at=NOW()+90days
    TS->>DB: INSERT audit_log (op=deprovision, actor_id=operator)
    TS->>DB: INSERT outbox_records (tenant.deprovisioned,<br/>{hard_deletion_at: NOW()+90days — BR012})
    Note over TS: COMMIT

    TS-->>-GW: 200 OK {tenant_id, status: deprovisioned,<br/>hard_deletion_at: ISO8601}
    GW-->>-Operator: 200 OK — deprovisioned

    OR->>SNS: Publish tenant.deprovisioned
    SNS->>OPS: SQS: ops-service-tenant-deprovisioned-queue

    OPS->>OPS: INSERT inbox_records (event_id, consumer=ops-service)
    OPS->>OPS: Schedule hard deletion job at hard_deletion_at (+90 days — BR012)
    OPS->>OPS: Log: "Payment records retained 7 years (BR012)"<br/>"Audit log retained permanently (BR014)"

    Note over OPS: At hard_deletion_at:
    OPS->>DB: DELETE tenant data (tables scoped to tenant_id)<br/>EXCEPT: audit_log (permanent), payment records (7-year retention)
    OPS->>S3: Delete objects in tenant-exports/{tenant_id}/ (lifecycle policy)
    OPS->>DB: INSERT audit_log (op=hard_delete, actor_id=system, occurred_at)
```

---

## Data Retention Rules

| Data Category | Retention After Deprovisioning | Rule |
|---|---|---|
| All tenant data (appointments, staff, config) | 90 days, then hard deleted | BR012 |
| Payment records | 7 years minimum | BR012 |
| Audit log | Permanent (never deleted) | BR014 |
| Export archives (S3) | Deleted with tenant data at +90 days | S3 lifecycle policy |
| PII (name, email, phone) | Pseudonymised on GDPR erasure request (FR013) | GDPR |

## Key Business Rules Enforced

| Rule | Enforcement Point |
|---|---|
| BR005 — export scoped to own tenant | api-gateway verifies tid=tenant_id; RLS on all export queries |
| BR011 — 30-day read-only window before deprovisioning | tenant-service validates NOW() ≥ data_export_deadline before accepting deprovision |
| BR012 — 90-day retention after deprovisioning | ops-service schedules hard deletion at deprovisioned_at + 90 days |
| BR014 — immutable audit log on all operator actions | audit_log INSERT on suspend, deprovision, hard_delete operations |
| FR012 — signed URL expires in 24 hours | S3 GeneratePresignedUrl with Expires=86400 |
