---
title: Flow Registry
layer: 01-requirements
status: current
lastUpdated: 2026-06-27
---

# R5 · Flow Registry

This registry catalogues every user journey defined for AnjiSchedulo. Each entry identifies the primary persona, maps the journey to its functional requirements and architectural patterns, and links to the detailed per-journey file. The persona cross-reference and dependency map at the end of this document show how journeys relate to one another.

---

## Journey Registry

| Journey ID | Name | Primary Persona | Priority | Related FRs | Architecture Pattern | Status |
|---|---|---|---|---|---|---|
| [UJ001](#uj001--tenant-onboarding) | Tenant Onboarding | P1 — Tenant Admin | Critical | FR001, FR002 | Pub/Sub Fan-out, Transactional Outbox | Current |
| [UJ002](#uj002--appointment-booking) | Appointment Booking | P3 — End Customer | Critical | FR003, FR004 | Saga Orchestration, Transactional Outbox, Idempotent Consumer | Current |
| [UJ003](#uj003--appointment-cancellation) | Appointment Cancellation | P3 — End Customer | Critical | FR005 | Saga Choreography, Dead Letter Queue, Retry + Backoff | Current |
| [UJ004](#uj004--appointment-rescheduling) | Appointment Rescheduling | P3 — End Customer | High | FR006 | Saga Orchestration, Transactional Outbox | Current |
| [UJ005](#uj005--tenant-dashboard-review) | Tenant Dashboard Review | P1 — Tenant Admin | High | FR008 | CQRS, Materialised View, Event Aggregator | Current |
| [UJ006](#uj006--platform-operator-dlq-triage) | Platform Operator DLQ Triage | P4 — Platform Operator | High | FR010, FR011 | Dead Letter Queue, Replay Pattern | Current |
| [UJ007](#uj007--tenant-data-export-and-offboarding) | Tenant Data Export and Offboarding | P1 — Tenant Admin | Medium | FR012, FR013 | Pub/Sub (tenant.deprovisioned), Status Machine | Current |

---

## Journey Descriptions

### UJ001 — Tenant Onboarding

A new business registers on AnjiSchedulo, receives a welcome email, and completes initial configuration — business hours, service catalogue, and staff roster — before going live. The journey ends when the tenant's public booking page is active and accepting customer appointments. Provisioning is event-driven: a `tenant.provisioned` event triggers parallel fan-out to the billing, notification, and tenant setup services via SNS/SQS.

**Detailed file:** `documentation/01-requirements/journeys/R6-journey-UJ001.md`

---

### UJ002 — Appointment Booking

An end customer visits a tenant's public booking page, selects a service and staff member, chooses an available time slot, and completes payment where required. The booking saga guarantees an atomic outcome: either a confirmed appointment with a notification delivered within 30 seconds, or a clean failure with no charge levied and no slot held. The full journey completes in under 2 minutes.

**Detailed file:** `documentation/01-requirements/journeys/R6-journey-UJ002.md`

---

### UJ003 — Appointment Cancellation

A customer cancels a confirmed appointment within the tenant's configured cancellation policy window. The platform releases the slot, initiates a refund where a payment was taken, and notifies the customer and relevant staff. Refund failure is handled independently via the Dead Letter Queue and never blocks slot release or customer notification.

**Detailed file:** `documentation/01-requirements/journeys/R6-journey-UJ003.md`

---

### UJ004 — Appointment Rescheduling

A customer moves their confirmed appointment to a different available time slot. The platform checks the new slot's availability before releasing the original booking, so a failed availability check leaves the original appointment fully intact. Once both operations succeed, a rescheduling confirmation is sent to the customer.

**Detailed file:** `documentation/01-requirements/journeys/R6-journey-UJ004.md`

---

### UJ005 — Tenant Dashboard Review

A tenant admin opens the operational dashboard at the start of the business day to review today's confirmed bookings, revenue, cancellation rate, and peak hours, and acts on any surfaced alerts. The dashboard is served from a pre-aggregated materialised view in under 200 milliseconds, with data reflecting booking events within 5 seconds.

**Detailed file:** `documentation/01-requirements/journeys/R6-journey-UJ005.md`

---

### UJ006 — Platform Operator DLQ Triage

A platform operator receives an automated Grafana alert when a failed event lands in a dead letter queue, inspects the full event payload in the ops console, identifies the failure cause, corrects and re-queues the event, and confirms successful reprocessing. The complete audit trail — original failure, correction, and resolution — is preserved.

**Detailed file:** `documentation/01-requirements/journeys/R6-journey-UJ006.md`

---

### UJ007 — Tenant Data Export and Offboarding

A tenant admin requests a full structured export of all tenant-scoped data before leaving the platform. After a mandatory 30-day window the platform operator initiates deprovisioning, and all tenant data is permanently deleted 90 days post-deprovisioning in accordance with the data retention policy (BR012), with the exception of audit logs which are retained permanently.

**Detailed file:** `documentation/01-requirements/journeys/R6-journey-UJ007.md`

---

## Persona Cross-Reference

The table below maps each persona to the journeys they participate in as either the primary actor or a secondary participant receiving a side effect (notification, alert, or operator action).

| Persona | Role | Primary Journeys | Secondary Involvement |
|---|---|---|---|
| P1 — Tenant Admin | Business Owner / Administrator | UJ001, UJ005, UJ007 | Receives booking velocity and cancellation rate alerts surfaced during UJ002 and UJ003 via FR009 |
| P2 — Staff Member | Service Provider | — | UJ002 (receives booking confirmation notification), UJ003 (receives cancellation notification), UJ004 (receives rescheduling notification) |
| P3 — End Customer | Appointment Booker | UJ002, UJ003, UJ004 | — |
| P4 — Platform Operator | SaaS Infrastructure Operator | UJ006 | UJ007 (initiates formal deprovisioning after the 30-day data export window closes) |

---

## Journey Dependency Map

The following structural dependencies must be respected during implementation and end-to-end testing. A downstream journey cannot be executed without the upstream journey having produced the required system state.

```
UJ001 — Tenant Onboarding  [MUST PRECEDE]
  ├─ UJ002 — Appointment Booking
  ├─ UJ005 — Tenant Dashboard Review
  └─ UJ007 — Tenant Data Export and Offboarding

UJ002 — Appointment Booking  [MUST PRECEDE]
  ├─ UJ003 — Appointment Cancellation
  └─ UJ004 — Appointment Rescheduling

UJ007 — Tenant Data Export and Offboarding  [PRECEDES]
  └─ UJ006 — Platform Operator DLQ Triage (deprovisioning triggers DLQ events)
```

---

## Coverage Gaps and Deferred Journeys

The following journeys are known but not yet specified. They will be added in future iterations once the corresponding roadmap phases are approved.

| Deferred Journey | Reason |
|---|---|
| Staff Availability Block (P2) | Scoped to v1.1 — P2 self-service availability management not in MVP |
| Enterprise SSO Setup (P1) | Scoped to Enterprise tier; requires IdP integration not in v1.0 |
| Booking Reminder Flow (P3) | Covered by FR007 notification triggers; no separate journey required |

---

*Per-journey files are located in `documentation/01-requirements/journeys/`.*
*Persona definitions are in `documentation/01-requirements/personas/`.*
*Functional requirements source: `specs/user/functional-requirements.json`.*
*User journey source: `specs/user/user-journeys.json`.*
