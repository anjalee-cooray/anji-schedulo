# R3 · Business Requirements Document

**Project:** AnjiSchedulo  
**Version:** 1.0.0  
**Owner:** Pubudu Anjalee Cooray  
**Last Updated:** 2026-06-28

---

## 1. Executive Summary

AnjiSchedulo is a multi-tenant B2B SaaS appointment scheduling platform targeting small and mid-sized service businesses. The platform eliminates manual scheduling, prevents double-bookings, automates appointment reminders, and provides real-time operational visibility — all at self-serve price points accessible to solo operators through to multi-location enterprises.

This document defines the business requirements that AnjiSchedulo must satisfy to achieve commercial viability and deliver value to its stakeholders.

---

## 2. Business Context

### 2.1 Problem Statement

Small and mid-sized service businesses lose revenue and customers to three avoidable problems:

1. **Double-bookings** caused by manual coordination across phone, spreadsheet, and chat.
2. **No-shows** with no automated reminder or policy enforcement.
3. **Inaccessible booking** — customers can only book during business hours via phone, driving them to competitors.

Existing tools are either too simple (no automation, no multi-staff support) or too expensive and technically complex (enterprise-only, require IT involvement). AnjiSchedulo fills this gap with enterprise-grade scheduling delivered as self-serve SaaS.

### 2.2 Target Market

| Vertical | Example Businesses |
|---|---|
| Salon & Beauty | Hair salons, nail studios, beauty therapists |
| Clinic & Healthcare | GP practices, physiotherapy, dental |
| Fitness & Wellness | Personal trainers, yoga studios, massage therapists |
| Consultancy & Coaching | Business coaches, life coaches, career advisors |
| Legal & Financial Services | Solicitors, financial advisors, tax consultants |
| Automotive Services | MOT centres, detailing services |

### 2.3 Business Model

AnjiSchedulo operates as a B2B SaaS with monthly subscription pricing across three tiers:

| Tier | Target | Staff | Bookings/Month | Notifications | Infrastructure |
|---|---|---|---|---|---|
| **Starter** | Solo operators and micro-businesses | Up to 3 | 200 | Email only | Shared |
| **Pro** | Growing SMBs with multiple staff | Up to 20 | 2,000 | Email + SMS | Shared |
| **Enterprise** | Multi-location businesses requiring SLA | Unlimited | Unlimited | Email + SMS + WhatsApp | Dedicated DB, custom domain, SLA, SSO, data residency |

---

## 3. Business Objectives

| ID | Objective | Measure of Success |
|---|---|---|
| BO001 | Eliminate double-bookings for all tenants | Zero double-booking incidents (enforced by BR001) |
| BO002 | Reduce no-show rate through automated reminders | Measurable no-show reduction reported by tenants |
| BO003 | Enable self-serve onboarding with no operator involvement | Tenant live and accepting bookings within 30 minutes of registration |
| BO004 | Deliver real-time operational visibility to tenant admins | Dashboard updated within 5 seconds of each booking event |
| BO005 | Achieve 99.9% availability for core booking operations | Monthly uptime measured via Grafana SLO dashboard |
| BO006 | Comply with GDPR and data protection obligations | PII erasure on request; 90-day retention post-deprovisioning |

---

## 4. Business Requirements

### 4.1 Tenant Management

**BR-BIZ-001 — Self-Service Onboarding**  
A new business must be able to register, configure their profile, and begin accepting bookings without any operator intervention. The platform must activate the billing subscription automatically on registration and send a welcome email. *(FR001)*

**BR-BIZ-002 — Tenant Configuration**  
Tenant admins must be able to independently configure all operational parameters: business hours per staff per day, service catalogue (name, duration, price), staff roster, cancellation policy window, booking advance window, and maximum active bookings per customer. *(FR002)*

**BR-BIZ-003 — Tenant Data Portability**  
A tenant must be able to export their complete data (configuration, staff, appointments, payments, notifications, analytics) at any time. Export must be scoped to the requesting tenant only. Download link must be delivered to the verified owner email and expire after 24 hours. *(FR012)*

### 4.2 Booking Operations

**BR-BIZ-004 — Atomic Appointment Booking**  
The appointment booking flow must be atomic. If any step fails (slot unavailable, payment declined), all partial state must be compensated. A customer must never be charged without a confirmed appointment, and a slot must never be held without a confirmed appointment. *(FR004, BR004)*

**BR-BIZ-005 — No Double-Booking**  
Under no circumstances — including concurrent requests — may two customers hold the same slot for the same staff member at the same time. This constraint is non-negotiable and must be enforced at the database level. *(BR001)*

**BR-BIZ-006 — Cancellation Policy Enforcement**  
A customer may cancel an appointment only within the cancellation window configured by the tenant admin. Late cancellation requests must be rejected with a clear error. *(FR005, BR002)*

**BR-BIZ-007 — Safe Rescheduling**  
Rescheduling must check new slot availability before releasing the original slot. If the new slot is unavailable, the original booking must remain unchanged. *(FR006, BR004)*

**BR-BIZ-008 — Idempotent Booking**  
Duplicate booking requests carrying the same idempotency key must produce exactly one confirmed appointment with no duplicate charge. *(BR006)*

### 4.3 Notifications

**BR-BIZ-009 — Lifecycle Notifications**  
Customers and staff must receive automated notifications at every key appointment lifecycle event: booking requested, confirmed, cancelled, rescheduled, and upcoming reminder. *(FR007)*

**BR-BIZ-010 — Notification Isolation**  
A notification delivery failure must never block, delay, or roll back the booking operation that triggered it. Notification and booking pipelines must be independently fault-tolerant. *(BR007)*

### 4.4 Operational Visibility

**BR-BIZ-011 — Tenant Dashboard**  
Tenant admins must have access to a real-time summary of booking operations: today's bookings, confirmed/cancelled/rescheduled counts, net revenue, cancellation rate, and peak hour. Dashboard must reflect events within 5 seconds. *(FR008)*

**BR-BIZ-012 — Anomaly Alerts**  
The platform must surface anomaly alerts to tenant admins when booking velocity exceeds 3× historical average in a 5-minute window, or when cancellation rate exceeds 30% over the last 50 bookings. *(FR009)*

### 4.5 Platform Operations

**BR-BIZ-013 — Dead Letter Queue Triage**  
Platform operators must be able to inspect, correct, and reprocess failed events. Failed events must be preserved in a DLQ with their full payload. DLQ entries must trigger an automated ops alert within 5 minutes. *(FR010)*

**BR-BIZ-014 — Event Replay**  
Platform operators must be able to replay historical events for a tenant to rebuild corrupted derived views. Replay must not disrupt live booking operations. *(FR011)*

**BR-BIZ-015 — Tenant Lifecycle Management**  
Platform operators must be able to suspend a tenant (halting new bookings while preserving read access) and deprovision a tenant (after the 30-day data export window). Data must be retained for 90 days post-deprovisioning before hard deletion. Payment records must be retained for 7 years. *(FR013, BR011, BR012)*

### 4.6 Data and Security

**BR-BIZ-016 — Tenant Isolation**  
A tenant's data must never be readable by another tenant under any condition, including infrastructure failure or application bug. This must be enforced at the database layer, not solely in application code. *(BR005)*

**BR-BIZ-017 — Audit Trail**  
All write operations that modify tenant-scoped data must produce an immutable audit log entry. The audit log must be permanent (never deleted). *(BR014)*

---

## 5. Constraints

| Constraint | Description |
|---|---|
| **Infrastructure** | Starter and Pro tiers share database infrastructure. Enterprise tier requires dedicated database cluster. |
| **No-code onboarding** | Tenant onboarding must require zero technical expertise — P1 persona is low-to-medium technical level. |
| **Third-party payment** | Payment processing is delegated to Stripe; the platform must not store raw card data. |
| **GDPR compliance** | PII must be pseudonymisable on customer erasure request. Tenant data must be exportable. |
| **Notification decoupling** | Notification delivery must never block booking operations (BR007). |

---

## 6. Out of Scope (v1)

The following are explicitly excluded from version 1:

- Video conferencing integration
- Multi-language / internationalisation
- Native mobile applications (iOS/Android)
- Marketplace / staff-to-tenant matching
- AI-driven scheduling suggestions
- POS / inventory management
- CRM integration

---

## 7. Assumptions

1. Tenants self-manage their configuration without support intervention (Starter and Pro tiers).
2. Payment is optional per tenant — the platform supports both free and paid appointment booking.
3. Customer accounts are not required for booking — customers are identified by email.
4. All tenants operate in a single language (English) for v1.
5. Platform operators have direct access to AWS infrastructure and Grafana dashboards.

---

## 8. Dependencies

| Dependency | Type | Risk |
|---|---|---|
| Stripe API | External — payment processing | Medium: payment failure handling via saga compensation |
| AWS SNS/SQS | External — event bus | Low: managed service with high availability |
| Email/SMS provider | External — notification delivery | Low: failures isolated to notification pipeline (BR007) |
| PostgreSQL RLS | Technical — tenant isolation | Low: enforced at DB layer independent of application |

---

## 9. Traceability

| Business Requirement | Related FRs | Related BRs | Related NFRs |
|---|---|---|---|
| BR-BIZ-001 | FR001 | BR005 | — |
| BR-BIZ-002 | FR002 | BR002, BR003, BR008, BR010 | — |
| BR-BIZ-003 | FR012 | BR005, BR014 | — |
| BR-BIZ-004 | FR004 | BR004 | NFR007 |
| BR-BIZ-005 | FR004 | BR001 | NFR007 |
| BR-BIZ-006 | FR005 | BR002 | — |
| BR-BIZ-007 | FR006 | BR004 | — |
| BR-BIZ-008 | FR004 | BR006 | — |
| BR-BIZ-009 | FR007 | BR007, BR013 | — |
| BR-BIZ-010 | FR007 | BR007 | NFR002 |
| BR-BIZ-011 | FR008 | — | NFR006, NFR008 |
| BR-BIZ-012 | FR009 | — | — |
| BR-BIZ-013 | FR010 | BR013 | NFR016 |
| BR-BIZ-014 | FR011 | — | NFR009 |
| BR-BIZ-015 | FR013 | BR011, BR012 | — |
| BR-BIZ-016 | — | BR005 | NFR012 |
| BR-BIZ-017 | — | BR014 | — |
