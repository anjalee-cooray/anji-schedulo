---
title: Sequence Diagram — Tenant Onboarding
journeyId: UJ001
layer: 02-design
status: current
lastUpdated: 2026-06-28
---

# D3 · Sequence Diagram — UJ001: Tenant Onboarding

## Overview

This diagram shows the full technical sequence for tenant onboarding: the synchronous registration transaction, asynchronous event fan-out via Transactional Outbox → SNS → SQS, parallel downstream processing by notification-service and billing-service, tenant activation on billing confirmation, and the subsequent configuration steps that make the booking page live.

---

```mermaid
sequenceDiagram
    participant Client as Client (Browser)
    participant GW as api-gateway
    participant TS as tenant-service
    participant OR as outbox-relay
    participant SNS as SNS
    participant NS as notification-service
    participant BS as billing-service
    participant Stripe as Stripe
    participant SG as SendGrid
    participant BCS as booking-command-service
    participant AS as availability-service

    Note over Client,AS: Phase A — Registration (Synchronous)

    Client->>+GW: POST /auth/register<br/>{business_name, owner_email, plan}
    GW->>GW: Inject X-Correlation-ID
    GW->>+TS: Forward registration request

    TS->>TS: Validate: email format, plan enum,<br/>no duplicate owner_email (active/provisioning)

    alt Validation fails
        TS-->>GW: 400 Bad Request (field errors)
        GW-->>-Client: 400 Bad Request
    end

    TS->>+Stripe: POST /v1/customers<br/>{email, name}
    Stripe-->>-TS: stripe_customer_id

    Note over TS: BEGIN TRANSACTION
    TS->>TS: INSERT tenants (status=provisioning)
    TS->>TS: INSERT tenant_config (plan defaults)
    TS->>TS: INSERT outbox_records (tenant.provisioned)
    TS->>TS: INSERT audit_log (entity=Tenant, op=create)
    Note over TS: COMMIT

    TS-->>-GW: 202 Accepted {tenant_id, status: provisioning}
    GW-->>-Client: 202 Accepted {tenant_id, correlation_id}

    Note over Client,AS: Phase B — Event Fan-out (Asynchronous)

    loop Every 500ms
        OR->>+TS: Poll outbox_records WHERE published_at IS NULL
        TS-->>-OR: [tenant.provisioned record]
    end

    OR->>+SNS: Publish tenant.provisioned<br/>(MessageGroupId=tenant_id)
    SNS-->>-OR: MessageId
    OR->>TS: UPDATE outbox_records SET published_at=NOW()

    SNS->>NS: SQS: notification-service-tenant-provisioned-queue
    SNS->>BS: SQS: billing-service-tenant-provisioned-queue

    Note over NS: Process tenant.provisioned
    NS->>NS: INSERT inbox_records (event_id, consumer=notification-service)<br/>— skip if duplicate
    NS->>NS: Read owner_email, owner_name, plan from payload
    NS->>+SG: Send welcome email (setup link, plan details)
    SG-->>-NS: 202 Accepted
    NS->>NS: INSERT notification_records (status=sent)

    Note over BS: Process tenant.provisioned
    BS->>BS: INSERT inbox_records (event_id, consumer=billing-service)
    BS->>BS: Read stripe_customer_id, plan from payload
    BS->>+Stripe: POST /v1/subscriptions {customer, price_id}
    Stripe-->>-BS: Subscription created
    BS->>BS: INSERT outbox_records (billing.subscription_activated)

    OR->>SNS: Publish billing.subscription_activated
    SNS->>TS: SQS: tenant-service-billing-activated-queue

    Note over TS: Process billing.subscription_activated
    TS->>TS: INSERT inbox_records (event_id, consumer=tenant-service)
    TS->>TS: UPDATE tenants SET status=active
    TS->>TS: INSERT audit_log (op=update, status: provisioning→active)
    TS->>TS: INSERT outbox_records (tenant.configured, config_snapshot)
    OR->>SNS: Publish tenant.configured
    SNS->>BCS: SQS: booking-command-service-tenant-configured-queue
    SNS->>AS: SQS: availability-service-tenant-configured-queue

    BCS->>BCS: Refresh local TenantConfig cache
    AS->>AS: Invalidate Redis slot cache for tenant

    Note over Client,AS: Phase C — Tenant Configuration (Synchronous)

    Client->>+GW: PUT /api/v1/tenants/{id}/working-hours<br/>Bearer JWT (role=tenant_admin)
    GW->>GW: Validate JWT, check role=tenant_admin
    GW->>+TS: Forward request
    TS->>TS: Validate hours, INSERT working_hours rows
    TS->>TS: INSERT outbox_records (tenant.configured)
    TS->>TS: INSERT audit_log
    TS-->>-GW: 200 OK
    GW-->>-Client: 200 OK

    Client->>+GW: POST /api/v1/tenants/{id}/services<br/>{name, duration_minutes, price}
    GW->>+TS: Forward request
    TS->>TS: Check plan service limit (5 for Starter)
    TS->>TS: INSERT services row
    TS->>TS: INSERT outbox_records (tenant.configured)
    TS-->>-GW: 201 Created
    GW-->>-Client: 201 Created {service_id}

    Client->>+GW: POST /api/v1/tenants/{id}/staff<br/>{name, email, services}
    GW->>+TS: Forward request
    TS->>TS: Check plan staff limit (3 Starter, 20 Pro)
    TS->>TS: INSERT staff_members row
    TS->>TS: INSERT outbox_records (tenant.configured)
    TS-->>-GW: 201 Created
    GW-->>-Client: 201 Created {staff_id}

    Note over Client,AS: Booking page /{tenant_slug}/book is now live
```

---

## Key Business Rules Enforced

| Step | Rule | Enforcement |
|---|---|---|
| Registration transaction | BR005 — tenant isolation from creation | tenant_id on all rows; RLS active immediately |
| Registration transaction | BR014 — immutable audit log | audit_log INSERT in same transaction |
| Event envelope | BR013 — event carries tenant_id, event_id, correlation_id, occurred_at | EventEnvelope validator before outbox write |
| Service creation | Plan limits (5 Starter, unlimited Pro/Enterprise) | Enforced by tenant-service |
| Staff creation | Plan limits (3 Starter, 20 Pro, unlimited Enterprise) | Enforced by tenant-service |

## Consistency Model

| Concern | Model |
|---|---|
| Tenant row + outbox write | Strong (single transaction) |
| Welcome email delivery | Eventually consistent (< 30s) |
| Billing activation | Eventually consistent (< 60s) |
| Cache refresh in booking / availability services | Eventually consistent (< 5s) |
