---
title: Risk Register
layer: 00-governance
status: current
lastUpdated: 2026-06-28
---

# G3 · Risk Register

## Overview

This register records all identified risks to the AnjiSchedulo project — technical, business, compliance, and operational. Each risk is assigned a severity based on likelihood and impact, a mitigation strategy, and an owner. The register is reviewed at the start of each sprint. New risks are added as identified; existing risk statuses are updated as mitigations are implemented or risk conditions change.

**Review cadence:** Per sprint (weekly during active development).
**Owner:** Pubudu Anjalee Cooray (Product & Architecture Owner).

---

## Risk Severity Matrix

| | **Low Impact** | **High Impact** |
|---|---|---|
| **High Likelihood** | Medium | High |
| **Low Likelihood** | Low | High |
| **Very Low Likelihood** | Low | Critical |

> **Critical** risks require immediate mitigation before the associated feature or phase goes live. **High** risks require a documented mitigation and a named owner. **Medium** risks are tracked and reviewed each sprint. **Low** risks are logged and monitored.

---

## Risk Register

### Technical Risks

| ID | Risk | Category | Likelihood | Impact | Severity | Mitigation | Owner | Status |
|---|---|---|---|---|---|---|---|---|
| R001 | Double-booking confirmed for the same slot | Booking | Low | Critical | Critical | Unique partial index on `slot_locks(tenant_id, staff_id, slot_date, slot_start) WHERE released_at IS NULL` enforced at the database level — no application-level lock can substitute this. Saga orchestration releases the lock atomically on any failure path. Integration test suite includes concurrent booking scenarios. | Product Owner | Open |
| R002 | Stripe API outage blocks all bookings requiring payment | Payment | Medium | High | High | Circuit breaker on all outbound Stripe API calls (opens after 5 consecutive failures, half-open after 30 seconds). Tenants that do not require payment capture are completely unaffected. Stripe contractual SLA is 99.9%. Operator runbook covers Stripe incident communication. | Product Owner | Open |
| R003 | outbox-relay message lag grows — events not published to SNS within SLA | Infrastructure | Low | High | High | Grafana alert fires if outbox backlog exceeds 100 unpublished records or relay lag exceeds 10 seconds. outbox-relay is independently deployable and scalable without redeploying domain services. Polling interval is 500 ms. | Platform Operator | Open |
| R004 | PostgreSQL RLS misconfiguration causes cross-tenant data leak | Security | Very Low | Critical | Critical | Three-layer enforcement: (1) api-gateway validates JWT `tid` claim; (2) service layer checks `tenant_id` against token; (3) PostgreSQL RLS enforces `tenant_id` scope at the database layer regardless of application code. Safe-by-default policy returns zero rows when `SET LOCAL app.tenant_id` is missing. CI suite includes explicit cross-tenant isolation tests for every tenant-scoped endpoint. | Product Owner | Open |
| R005 | DLQ builds up silently without triggering an alert | Operations | Low | Medium | Medium | Grafana alert configured to fire within 5 minutes of any DLQ queue receiving its first message. Weekly DLQ depth review logged in the change log. All consumer queues have a paired DLQ with `maxReceiveCount = 4`. | Platform Operator | Open |
| R006 | Event published without required envelope fields (BR013 violation) | Data Integrity | Low | High | High | `EventEnvelope` validator runs in every consumer before processing. Events missing `tenant_id`, `event_id`, `correlation_id`, or `occurred_at` are routed to DLQ and an ops alert fires. Integration test suite validates the full envelope for every event type in `specs/ai/events.json`. | Product Owner | Open |

### Business / Product Risks

| ID | Risk | Category | Likelihood | Impact | Severity | Mitigation | Owner | Status |
|---|---|---|---|---|---|---|---|---|
| R007 | Tenant onboarding experience is too slow or complex, driving early churn | Product | Medium | High | High | Setup flow target: registration to first live booking in under 30 minutes (FR001 acceptance criterion). Guided setup checklist with progress indicator shown in the tenant dashboard. Welcome email includes a direct setup link (not just a login URL). Setup flow is tested end-to-end as part of the MVP acceptance suite. | Product Owner | Open |
| R008 | Notification deliverability falls below acceptable rate | Notifications | Medium | Medium | Medium | Retry with exponential backoff: up to 4 attempts per notification (delays: 1s, 2s, 4s, 8s). DLQ fallback for undeliverable notifications with ops alert. SendGrid dedicated sending IP for Pro and Enterprise tenants to protect sender reputation. Bounce rates monitored in SendGrid dashboard. Notification failures never affect booking operations (BR007). | Platform Operator | Open |
| R009 | Plan limits are too restrictive, creating friction at tier boundaries | Product | Medium | Medium | Medium | Plan limits (Starter: 3 staff, 200 bookings/mo, 5 services; Pro: 20 staff, 2,000 bookings/mo) are set conservatively for MVP. Limits reviewed after the first 10 tenants. Upgrade flow is self-serve with immediate effect. Plan limit rejection returns a clear 422 with the specific limit exceeded and an upgrade prompt. | Product Owner | Open |

### Compliance / Legal Risks

| ID | Risk | Category | Likelihood | Impact | Severity | Mitigation | Owner | Status |
|---|---|---|---|---|---|---|---|---|
| R010 | GDPR customer erasure request not fulfilled within the 30-day statutory SLA | Compliance | Low | High | High | Pseudonymisation flow documented in D6 and FR013. Ops runbook for erasure requests describes the exact steps and SLA timer. `AuditLog` records the erasure event with `occurred_at` timestamp for SLA verification. Platform Operator is Responsible for execution; Product Owner is Accountable. | Platform Operator | Open |
| R011 | Payment records hard-deleted before the 7-year retention minimum | Compliance | Very Low | Critical | Critical | BR012 specifies separate retention policies per record type: general tenant data (90 days post-deprovisioning), payment records (7 years minimum), audit log (permanent). Hard-deletion scheduled job explicitly excludes `payment` records from the 90-day deletion sweep. Separate retention check in the deletion job CI test. | Product Owner | Open |
| R012 | Tenant data export includes records from another tenant | Security | Very Low | Critical | Critical | Export is scoped exclusively by `tenant_id` from the authenticated JWT `tid` claim. PostgreSQL RLS enforces the scope at the query layer. Integration test asserts that an export request for tenant A returns zero records belonging to tenant B. Signed URL is sent only to the verified `owner_email` from the `tenants` table, not the JWT email claim. | Product Owner | Open |

### Operational Risks

| ID | Risk | Category | Likelihood | Impact | Severity | Mitigation | Owner | Status |
|---|---|---|---|---|---|---|---|---|
| R013 | Single-author bus risk — loss of key person halts all development and operations | Team | High | High | Critical | All architecture decisions documented in ADRs (`documentation/04-architecture/adrs/`). All operational procedures in runbooks (`documentation/06-operations/`). All specifications machine-readable in `specs/`. The repository is the complete source of truth — a new contributor can onboard from the documentation alone. This risk is **Accepted**: it is inherent to a solo-author project and cannot be fully mitigated. | Product Owner | Accepted |
| R014 | AWS `eu-west-1` regional outage affects platform availability | Infrastructure | Very Low | High | High | Multi-AZ deployment within `eu-west-1` (RDS Multi-AZ, ECS tasks across three availability zones). Daily automated PostgreSQL backups to `eu-central-1` S3 bucket (cross-region). RTO target < 1 hour for AZ failover, < 4 hours for full region failover. Disaster recovery runbook at `documentation/04-architecture/resilience-flows/RES-003-disaster-recovery.md`. | Platform Operator | Open |

---

## Risk Review History

| Date | Reviewer | Changes Made |
|---|---|---|
| 2026-06-28 | Pubudu Anjalee Cooray | Initial risk register created — 14 risks identified across technical, business, compliance, and operational categories |
