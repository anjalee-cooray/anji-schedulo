---
title: Project Charter
layer: 00-governance
status: current
lastUpdated: 2026-06-28
---

# G1 · Project Charter

## Project Overview

| Field | Value |
|---|---|
| **Project Name** | AnjiSchedulo |
| **Version** | 1.0.0 |
| **Owner** | Pubudu Anjalee Cooray |
| **Role** | Product & Architecture |
| **Status** | Design |
| **Repository** | https://github.com/anjalee-cooray/anji-schedulo |
| **Last Updated** | 2026-06-28 |

AnjiSchedulo is a multi-tenant SaaS appointment scheduling platform that gives service businesses — from solo therapists to multi-location clinics — a self-serve, enterprise-grade booking system without requiring technical expertise or dedicated infrastructure. Businesses register, configure their services and staff, and begin accepting customer bookings in under 30 minutes. Customers browse availability, book appointments, and manage cancellations or rescheduling entirely online. The platform enforces double-booking prevention at the database level, delivers automated lifecycle notifications, and provides real-time operational dashboards — all underpinned by an event-driven microservices architecture with strict multi-tenant data isolation.

---

## Problem Statement

Small and mid-sized service businesses collectively lose significant revenue each year to preventable operational failures: double-bookings that create immediate customer-facing crises, missed appointment reminders that increase no-show rates, and manual scheduling processes that consume hours of staff time that would otherwise serve customers. A salon owner spending four hours per week on phone bookings and spreadsheet reconciliation is not only losing productivity — they are accepting a risk of errors that erodes customer trust.

Existing scheduling tools fail these businesses at opposite ends of the spectrum. Consumer-grade tools (shared calendars, simple booking widgets) offer no automation, no payment integration, and no multi-staff support — they digitise the problem without solving it. Enterprise scheduling platforms, by contrast, require IT departments, long onboarding cycles, and pricing models that exclude the businesses that need them most. There is a large and underserved middle market of businesses with two to twenty staff members that need professional-grade scheduling capability at accessible price points.

AnjiSchedulo closes this gap by delivering the core scheduling capabilities that enterprise businesses take for granted — atomic double-booking prevention, automated notifications, real-time dashboards, saga-based payment integration, and GDPR-compliant data management — in a self-serve SaaS product that a non-technical business owner can configure unassisted in under 30 minutes.

---

## Product Vision

Make professional appointment scheduling accessible to every business — from a solo therapist to a multi-location clinic — without requiring technical expertise or per-tenant infrastructure.

In three years, AnjiSchedulo will be the default scheduling infrastructure for small and mid-sized service businesses across the UK and Europe. A business owner in any of the platform's target verticals will be able to register, configure, and go live in a single afternoon, confident that every booking is protected against double-confirmation, every cancellation triggers the correct automated workflows, and every customer interaction is tracked in a compliant and auditable way. The platform will grow with businesses: a solo therapist on the Starter plan will be able to upgrade to Pro as their team grows, and to Enterprise as they expand to multiple locations — all without migrating to a different tool or re-training staff.

---

## Target Market

AnjiSchedulo targets B2B SaaS customers across six primary verticals:

| Vertical | Description |
|---|---|
| **Salon & Beauty** | Hair salons, nail studios, beauty therapists, and barbers that manage bookings across multiple stylists with varying availability and service specialisations. |
| **Clinic & Healthcare** | Private GP practices, physiotherapy clinics, dental surgeries, and specialist consultancies that require appointment-level compliance, payment capture, and GDPR-grade data handling. |
| **Fitness & Wellness** | Personal trainers, yoga studios, and massage therapists whose schedules are driven by individual practitioner availability and recurring weekly patterns. |
| **Consultancy & Coaching** | Business coaches, career advisors, and independent consultants who need a clean booking page they can share with clients without relying on email back-and-forth. |
| **Legal & Financial Services** | Solicitors, accountants, and financial advisers who book client appointments that may require pre-payment or deposit capture at time of booking. |
| **Automotive Services** | Garages, valeting services, and MOT centres that operate on fixed-duration service slots and need to manage workshop capacity across technicians. |

---

## Pricing Tiers

| Tier | Target | Staff Limit | Bookings / Month | Services | Notification Channels | Infrastructure |
|---|---|---|---|---|---|---|
| **Starter** | Solo operators and micro-businesses | 3 | 200 | 5 | Email | Shared |
| **Pro** | Growing SMBs with multiple staff | 20 | 2,000 | Unlimited | Email, SMS | Shared |
| **Enterprise** | Multi-location businesses requiring SLA, custom domain, dedicated infrastructure | Unlimited | Unlimited | Unlimited | Email, SMS, WhatsApp | Dedicated database cluster, custom domain, SSO, data residency |

Enterprise plan extras: dedicated PostgreSQL cluster, custom booking page domain, contractual SLA, SAML 2.0 / OIDC SSO, configurable data residency region.

---

## Project Scope

### In Scope — MVP

The MVP delivers the core scheduling platform sufficient for the first three paid tenants to go live:

- Tenant self-service registration and onboarding (no operator intervention required)
- Service catalogue and staff roster configuration
- Slot availability browsing by customers (public, unauthenticated)
- Appointment booking with optional payment capture (Stripe integration)
- Appointment cancellation and rescheduling
- Email lifecycle notifications (booking confirmed, cancelled, rescheduled, upcoming reminder)
- Tenant operational dashboard (today's bookings, revenue, cancellation rate, peak hour)
- Platform operator Dead Letter Queue console (inspect, re-queue, discard failed events)
- Basic Grafana observability (service health dashboards, DLQ depth alerts)

### Out of Scope — v1

The following capabilities are explicitly deferred beyond MVP:

- Video conferencing integration (Zoom, Google Meet)
- Multi-language and internationalisation support
- Native iOS and Android mobile applications
- Customer marketplace (public tenant directory, cross-tenant customer accounts)
- AI-driven scheduling suggestions and smart slot recommendations
- POS and inventory management
- CRM integration (Salesforce, HubSpot)

---

## Roadmap Summary

| Phase | Status | Goal |
|---|---|---|
| **MVP** | Design | Core booking platform live with three paid tenants |
| **v1.0** | Planned | Production-ready platform with full observability, Pro plan, and Enterprise plan |
| **v1.5** | Planned | Platform scalability improvements and channel expansion (WhatsApp, calendar sync, waitlists) |
| **v2.0** | Roadmap | Platform intelligence and integration ecosystem (AI suggestions, CRM, video conferencing) |
| **Marketplace** | Vision | Open platform where customers discover and book businesses across a public directory |
| **Platform Expansion** | Vision | Vertical expansion beyond appointment booking (POS, payroll, loyalty, franchise management) |

---

## Stakeholders

| Role | Name / Persona | Responsibilities |
|---|---|---|
| **Product & Architecture Owner** | Pubudu Anjalee Cooray | Overall product vision, roadmap ownership, architecture decisions (ADRs), specification authorship, quality sign-off |
| **Platform Operator** | P4 | Infrastructure health monitoring, incident response, DLQ triage, tenant suspension and deprovisioning, secrets management |
| **Tenant Admin** | P1 | Primary business customer; configures the platform for their business; responsible for all tenant-scoped data within their account |
| **End Customer** | P3 | Books appointments via the tenant's public booking page; no direct relationship with AnjiSchedulo as a platform |
| **Staff Member** | P2 | Employee or contractor of a tenant; manages their own availability; receives booking lifecycle notifications |

---

## Success Criteria

| Criterion | Target | Source |
|---|---|---|
| First paying tenant live | Within 30 days of MVP deployment | roadmap.json — MVP goal |
| Booking saga p95 latency | < 2 seconds end-to-end | functional-requirements.json — FR004 |
| Slot availability query p95 | < 500 ms | functional-requirements.json — FR003 |
| Tenant dashboard response time | < 200 ms | functional-requirements.json — FR008 |
| Double-bookings in production | Zero | business-rules.json — BR001 |
| Notification delivery rate | > 99% (excluding DLQ-routed failures) | functional-requirements.json — FR007 |
| DLQ event resolution SLA | < 4 hours from alert | operations.json — resolutionSla |
| Platform uptime | 99.9% over any 30-day window | non-functional-requirements.json — NFR001 |

---

## Project Constraints

1. **Single author.** AnjiSchedulo is designed and built by one person (Pubudu Anjalee Cooray). All architecture decisions, specifications, and implementation work are the sole responsibility of the owner. This is identified as a key risk (R013) and mitigated through complete documentation of all decisions, runbooks, and ADRs.

2. **Shared infrastructure for Starter and Pro tiers.** Both tiers run on shared PostgreSQL clusters with Row-Level Security providing tenant isolation. Dedicated infrastructure is available only to Enterprise tenants. This is a deliberate cost constraint that must not be relaxed without a formal ADR.

3. **No native mobile applications in v1.** The customer-facing booking page is a responsive web application. Native iOS and Android apps are deferred to v2.0. The booking flow must be fully functional on mobile browsers at 375px viewport width.

4. **GDPR compliance from day one.** The platform is designed for UK and European customers. All PII handling, data retention policies (BR012), pseudonymisation flows, and audit logging (BR014) must be implemented from the first production deployment — not retrofitted later.

5. **Multi-tenant isolation is a hard non-negotiable.** Tenant data isolation (BR005) cannot be traded off for performance or simplicity at any point in the roadmap. Every architectural decision that touches cross-tenant data access must be reviewed against BR005 and tested with an explicit cross-tenant isolation test in CI.
