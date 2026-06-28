---
title: RACI Matrix
layer: 00-governance
status: current
lastUpdated: 2026-06-28
---

# G2 · RACI Matrix

## Overview

The RACI matrix defines who is Responsible, Accountable, Consulted, and Informed for each activity in the AnjiSchedulo project.

| Code | Meaning |
|---|---|
| **R** | **Responsible** — does the work |
| **A** | **Accountable** — owns the outcome; approves the result; only one per activity |
| **C** | **Consulted** — provides input before or during the work |
| **I** | **Informed** — notified of the outcome after the fact |
| **—** | No involvement |

---

## Roles

| Role | Description |
|---|---|
| **Product & Architecture Owner** | Pubudu Anjalee Cooray — sole author; owns all product, architecture, and specification decisions |
| **Platform Operator** | P4 — SRE / infrastructure function; responsible for platform health, incident response, and operational tooling |
| **Tenant Admin** | P1 — the business customer; accountable for their own tenant configuration and data |
| **End Customer** | P3 — books appointments via the tenant booking page; external to the platform |
| **Staff Member** | P2 — employee or contractor of a tenant; manages own availability; external to the platform |

---

## RACI Matrix

### Governance & Architecture

| Activity | Product & Architecture Owner | Platform Operator | Tenant Admin | End Customer | Staff Member |
|---|---|---|---|---|---|
| Define product vision and roadmap | R/A | I | C | — | — |
| Author and approve technical specifications | R/A | C | — | — | — |
| Architecture decisions (ADRs) | R/A | C | — | — | — |
| Define security model and data retention policies | R/A | C | I | — | — |

### Tenant Lifecycle

| Activity | Product & Architecture Owner | Platform Operator | Tenant Admin | End Customer | Staff Member |
|---|---|---|---|---|---|
| Tenant registration and self-service onboarding | I | I | R/A | — | — |
| Configure business hours, services, and staff | I | — | R/A | — | C |
| Request data export | I | R | A | — | — |
| Respond to suspension / deprovisioning notice | A | R | I | — | — |

### Booking Operations (Daily)

| Activity | Product & Architecture Owner | Platform Operator | Tenant Admin | End Customer | Staff Member |
|---|---|---|---|---|---|
| Browse available appointment slots | — | — | — | R/A | — |
| Book / cancel / reschedule appointments | — | — | — | R/A | — |
| Block staff availability | — | — | C | — | R/A |
| Receive booking lifecycle notifications | — | — | I | I | I |

### Platform Operations

| Activity | Product & Architecture Owner | Platform Operator | Tenant Admin | End Customer | Staff Member |
|---|---|---|---|---|---|
| Monitor infrastructure health and alerts | I | R/A | — | — | — |
| Triage and resolve DLQ events | C | R/A | — | — | — |
| Trigger tenant event replay | A | R | — | — | — |
| Initiate tenant suspension and deprovisioning | A | R | I | — | — |
| Respond to incidents (P1 / P2 severity) | A | R | I | — | — |
| Rotate secrets and manage infrastructure access | A | R | — | — | — |

### Compliance & Data

| Activity | Product & Architecture Owner | Platform Operator | Tenant Admin | End Customer | Staff Member |
|---|---|---|---|---|---|
| GDPR customer erasure requests | A | R | I | C | — |
| Audit log review | A | R | C | — | — |
| Data retention enforcement | A | R | — | — | — |
| Security incident response | A | R | I | — | — |

---

## Notes

**Tenant Admin is Accountable for tenant-scoped data.** The Tenant Admin (P1) is the data controller for all data within their tenant account. AnjiSchedulo acts as data processor. The Tenant Admin is responsible for their customers' consent and their staff data, within the constraints set by AnjiSchedulo's privacy policy and data retention rules (BR012).

**Platform Operator has cross-tenant read access only for operational purposes.** The Platform Operator role can read across tenant boundaries only via the ops console, only for events in Dead Letter Queues or replay jobs, and only with a full audit trail written for every action (BR014). The `platform_operator` JWT role is verified at the api-gateway layer and cannot access tenant-scoped data via the standard tenant API endpoints.

**Product & Architecture Owner is Accountable for all architectural decisions.** In a single-author project, the owner carries sole accountability for architecture, security, and compliance decisions. The RACI reflects this — the Platform Operator is Consulted on operational implications of architecture choices, but cannot override them.

**End Customers and Staff Members are external actors.** They interact with the platform through well-defined public or authenticated endpoints. They are never Responsible or Accountable for platform-level activities. Their "I" entries reflect the automated notifications they receive from the platform (appointment confirmations, cancellation notices) rather than any active participation in platform governance.
