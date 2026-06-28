# R2 · Stakeholder Map

**Project:** AnjiSchedulo  
**Version:** 1.0.0  
**Owner:** Pubudu Anjalee Cooray  
**Last Updated:** 2026-06-28

---

## 1. Overview

AnjiSchedulo is a multi-tenant B2B SaaS appointment scheduling platform. Four distinct stakeholder groups interact with the system, each with different goals, influence levels, and expectations. This document maps each stakeholder, their relationship to the product, and their primary concerns.

---

## 2. Internal Stakeholders

### 2.1 Tenant Admin (P1) — Business Owner / Administrator

| Attribute | Detail |
|---|---|
| **Who** | Business owner or operations manager who onboards and runs their business on AnjiSchedulo |
| **Influence** | High — primary paying customer; churn directly impacts revenue |
| **Engagement** | Daily — manages staff, services, schedule, and dashboard |
| **Primary Needs** | Zero double-bookings, automated reminders, simple self-service onboarding, revenue visibility |
| **Pain Points** | Manual scheduling, no-shows with no penalty, lack of insight into popular services, staff sick-day rebooking |
| **Success Metric** | First booking accepted within 30 minutes of sign-up; no-show rate measurably reduced |
| **Escalation Path** | In-app support → dedicated SLA (Enterprise tier) |

**Key interactions:**
- Configures business hours, services, staff roster, and cancellation policy (FR002)
- Views daily operational dashboard (FR008)
- Receives anomaly alerts for cancellation rate spikes and booking velocity (FR009)
- Requests full data export and manages offboarding (FR012)

---

### 2.2 Staff Member (P2) — Service Provider

| Attribute | Detail |
|---|---|
| **Who** | Employee or contractor whose availability is managed on the platform |
| **Influence** | Medium — affects tenant satisfaction; indirect churn risk |
| **Engagement** | Daily to weekly — checks upcoming schedule, blocks personal time |
| **Primary Needs** | Real-time schedule visibility, instant booking/cancellation notifications, self-service availability blocking |
| **Pain Points** | Finding out about bookings late, no self-service time-blocking, schedule changes via informal chat groups |
| **Success Metric** | Zero missed notifications; schedule always current without calling admin |

**Key interactions:**
- Receives booking, cancellation, and rescheduling notifications (FR007)
- Blocks personal availability without admin involvement (FR002)
- Cancels appointments on behalf of customer if needed (FR005)

---

### 2.3 End Customer (P3) — Appointment Booker

| Attribute | Detail |
|---|---|
| **Who** | Person booking an appointment with a tenant's business |
| **Influence** | High volume — satisfaction drives tenant retention and word-of-mouth |
| **Engagement** | Transactional — per appointment lifecycle event |
| **Primary Needs** | Sub-2-minute booking, instant confirmation, self-service cancel/reschedule without calling |
| **Pain Points** | Phone-only booking during limited hours, no confirmation received, showing up to find no record of booking |
| **Success Metric** | Confirmation received within 30 seconds of booking; cancellation/reschedule completed without phone call |

**Key interactions:**
- Browses available slots by service, date, and staff member (FR003)
- Books, cancels, and reschedules appointments (FR004, FR005, FR006)
- Receives all lifecycle notifications — confirmed, cancelled, rescheduled, reminder (FR007)

---

### 2.4 Platform Operator (P4) — SaaS Infrastructure Operator

| Attribute | Detail |
|---|---|
| **Who** | Internal AnjiSchedulo team member responsible for infrastructure health, incident response, and compliance operations |
| **Influence** | Critical — responsible for uptime, data integrity, and tenant lifecycle compliance |
| **Engagement** | On-call — incident-driven; also performs planned maintenance and tenant lifecycle actions |
| **Primary Needs** | Immediate degradation visibility, ability to inspect and resolve stuck events, safe tenant lifecycle tooling |
| **Pain Points** | Silent pipeline failures, no event inspection tooling, cross-tenant data leakage risk |
| **Success Metric** | DLQ events resolved within 4-hour SLA (NFR016); zero cross-tenant isolation violations (NFR012) |

**Key interactions:**
- Monitors DLQ depth and triages failed events via ops console (FR010)
- Triggers tenant event replay to rebuild corrupted derived views (FR011)
- Suspends and deprovisions tenants in compliance with retention policy (FR013)
- Responds to Grafana alerts for service degradation

---

## 3. External Stakeholders

| Stakeholder | Role | Primary Concern |
|---|---|---|
| **Stripe** | Payment processor | PCI-DSS compliance, reliable charge/refund APIs |
| **AWS** | Cloud infrastructure provider | 99.9% SLA, data residency (ap-southeast-1), encryption at rest |
| **Amazon SNS / SQS** | Message bus | At-least-once delivery, DLQ configuration, 14-day message retention |
| **SMTP / SMS Provider** | Notification delivery (email, SMS, WhatsApp) | Delivery reliability, retry tolerance |
| **Regulatory bodies** | GDPR and data protection compliance | PII handling, erasure rights, 90-day data retention minimum (BR012), 7-year payment record retention |

---

## 4. Influence / Interest Matrix

| | High Influence | Low Influence |
|---|---|---|
| **High Interest** | Tenant Admin (P1), Platform Operator (P4), Stripe, AWS | Staff Member (P2), End Customer (P3) |
| **Low Interest** | — | Regulatory bodies |

- **High Influence / High Interest:** Manage closely — decisions require their input or directly affect revenue and uptime.
- **Low Influence / High Interest:** Keep informed — they experience the product daily and their satisfaction drives tenant retention.

---

## 5. Communication Plan

| Stakeholder | Channel | Trigger |
|---|---|---|
| Tenant Admin (P1) | In-app notifications, email, dashboard alerts | Booking events, anomaly alerts, account changes |
| Staff Member (P2) | Email, SMS (Pro+), WhatsApp (Enterprise) | Booking confirmed, cancelled, rescheduled |
| End Customer (P3) | Email, SMS (Pro+), WhatsApp (Enterprise) | Booking confirmed, reminder, cancelled, rescheduled |
| Platform Operator (P4) | Grafana alerts, PagerDuty, ops console | DLQ depth > 0, service degradation, SLO breach |
| Internal engineering | Changelog, release notes, ADRs | Per release cycle |

---

## 6. Traceability

| Persona ID | Stakeholder | Related FRs | Related BRs |
|---|---|---|---|
| P1 | Tenant Admin | FR001, FR002, FR008, FR009, FR012 | BR002, BR003, BR005, BR008, BR010, BR014 |
| P2 | Staff Member | FR005, FR007 | BR002, BR007 |
| P3 | End Customer | FR003, FR004, FR005, FR006, FR007 | BR001, BR002, BR004, BR006, BR008, BR010 |
| P4 | Platform Operator | FR010, FR011, FR013 | BR005, BR011, BR012, BR013, BR014 |
