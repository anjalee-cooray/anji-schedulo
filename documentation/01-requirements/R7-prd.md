# R7 · Product Requirements Document

**Project:** AnjiSchedulo  
**Version:** 1.0.0  
**Owner:** Pubudu Anjalee Cooray  
**Last Updated:** 2026-06-28

---

## 1. Product Vision

Make professional appointment scheduling accessible to every business — from a solo therapist to a multi-location clinic — without requiring technical expertise or per-tenant infrastructure.

---

## 2. Problem Statement

Small and mid-sized service businesses lose revenue and customers to double-bookings, missed reminders, and manual scheduling. Existing tools are either too simple (no automation) or too expensive and complex (enterprise-only). AnjiSchedulo delivers enterprise-grade scheduling as a self-serve SaaS at accessible price points.

---

## 3. Target Personas

| Persona | Role | Key Need |
|---|---|---|
| P1 — Tenant Admin | Business Owner / Administrator | Self-serve onboarding, no-show reduction, operational visibility |
| P2 — Staff Member | Service Provider | Schedule visibility, instant notifications, self-service blocking |
| P3 — End Customer | Appointment Booker | Sub-2-minute booking, instant confirmation, self-service cancel/reschedule |
| P4 — Platform Operator | SaaS Infrastructure Operator | Zero silent failures, DLQ triage, safe tenant lifecycle management |

---

## 4. Pricing Tiers

| Tier | Target | Staff | Bookings/Month | Notifications | Infrastructure |
|---|---|---|---|---|---|
| **Starter** | Solo operators and micro-businesses | Up to 3 | 200 | Email only | Shared |
| **Pro** | Growing SMBs with multiple staff | Up to 20 | 2,000 | Email + SMS | Shared |
| **Enterprise** | Multi-location businesses requiring SLA | Unlimited | Unlimited | Email + SMS + WhatsApp | Dedicated DB, custom domain, SLA, SSO, data residency |

---

## 5. Feature Requirements

### 5.1 Tenant Onboarding and Configuration

#### FR001 — Tenant Self-Service Onboarding
A new business can register, configure their profile, and become active without operator intervention.

**Acceptance Criteria:**
- Tenant submits registration with business name, owner email, and plan selection
- Billing subscription activated automatically on registration
- Welcome email sent automatically on registration
- Tenant status transitions to active once all provisioning steps confirm
- Tenant can set business hours, services, and staff before going live

**Personas:** P1  
**Business Rules:** BR005

---

#### FR002 — Tenant Configuration Management
A tenant admin can configure all operational parameters for their business.

**Acceptance Criteria:**
- Admin can set business hours per staff member per day
- Admin can create, update, and deactivate services
- Admin can add, update, and remove staff members
- Admin can set cancellation policy window in hours
- Admin can set booking advance window in days
- Admin can set max active bookings per customer

**Personas:** P1  
**Business Rules:** BR002, BR003, BR008, BR010

---

### 5.2 Booking

#### FR003 — Customer Slot Browsing
A customer can browse available slots for a tenant's services and staff.

**Acceptance Criteria:**
- Customer queries available slots by service, date, and staff member
- Available slots exclude confirmed bookings, staff blocks, and outside-hours periods
- Slots beyond the tenant's booking window are not shown
- Response time under 500ms

**Personas:** P3  
**Business Rules:** BR003, BR010

---

#### FR004 — Appointment Booking
A customer can book an appointment through an atomic, compensatable saga.

**Acceptance Criteria:**
- Booking request initiates availability check, payment (if required), and confirmation in sequence
- If slot is unavailable, booking fails with a 409 response and no charge
- If payment fails, slot reservation is released and booking fails with 402
- Duplicate requests with the same idempotency key return the original result without re-processing
- Customer receives confirmation notification within 30 seconds of booking

**Personas:** P3  
**Business Rules:** BR001, BR004, BR006, BR007, BR008

---

#### FR005 — Appointment Cancellation
A customer or staff member can cancel a confirmed appointment within the cancellation policy window.

**Acceptance Criteria:**
- Cancellation outside policy window is rejected with a clear error
- Cancellation within window triggers: refund (if applicable), slot release, notifications
- Slot is released regardless of refund outcome
- Customer and staff are notified of cancellation
- Full appointment history including cancellation is preserved

**Personas:** P2, P3  
**Business Rules:** BR002, BR009

---

#### FR006 — Appointment Rescheduling
A customer can move a confirmed appointment to a new slot.

**Acceptance Criteria:**
- Rescheduling atomically cancels the current slot and books the new slot
- New slot availability is checked before releasing the original slot
- If new slot is unavailable, original booking remains unchanged
- Customer receives rescheduling confirmation notification
- Appointment event history records the reschedule

**Personas:** P3  
**Business Rules:** BR001, BR004

---

### 5.3 Notifications

#### FR007 — Lifecycle Notifications
Customers and staff receive automated communications at key appointment lifecycle events.

**Acceptance Criteria:**
- Notifications sent at: booking requested, confirmed, cancelled, rescheduled, upcoming reminder
- All notification content derived from event snapshot — no back-calls to booking services
- Failed deliveries retried up to 4 times with exponential backoff
- After 4 failures, event routes to DLQ and ops alert is raised
- Delivery failures never delay or cancel the triggering booking operation

**Personas:** P2, P3  
**Business Rules:** BR007, BR013

---

### 5.4 Dashboard and Analytics

#### FR008 — Tenant Operational Dashboard
A tenant admin can view a real-time summary of their booking operations.

**Acceptance Criteria:**
- Dashboard shows: today's bookings, confirmed/cancelled/rescheduled counts, net revenue, cancellation rate, peak hour
- Dashboard data reflects events within 5 seconds under normal load
- Dashboard response time under 200ms (served from materialized view)
- Dashboard can be fully rebuilt from event history if corrupted

**Personas:** P1  
**Business Rules:** —  
**NFRs:** NFR006, NFR008

---

#### FR009 — Stream Processing Alerts
The platform continuously monitors per-tenant booking patterns and surfaces anomaly alerts.

**Acceptance Criteria:**
- Booking velocity spike (> 3× historical average in 5-minute window) triggers alert
- Cancellation rate > 30% over last 50 bookings triggers alert
- Peak hour data updated continuously and available to tenant admins
- Alerts delivered to tenant admin and platform operator within 60 seconds of detection

**Personas:** P1, P4

---

### 5.5 Platform Operations

#### FR010 — Dead Letter Queue Triage
Platform operators can inspect, correct, and reprocess failed events.

**Acceptance Criteria:**
- Failed events routed to DLQ after retry exhaustion with full payload preserved
- Ops console shows DLQ entries filterable by tenant_id, event_type, failure_reason
- Operator can re-queue a corrected event or discard with audit log entry
- DLQ entry triggers automated ops alert within 5 minutes

**Personas:** P4  
**NFRs:** NFR016

---

#### FR011 — Tenant Event Replay
Platform operators can replay historical events for a tenant to rebuild derived views.

**Acceptance Criteria:**
- Replay can be scoped to a tenant, date range, or specific event IDs
- Target services rebuild state idempotently from replayed events
- Replay does not affect live booking operations
- Replay job status is tracked and visible to operators

**Personas:** P4  
**NFRs:** NFR009

---

### 5.6 Tenant Lifecycle

#### FR012 — Tenant Data Export
A tenant admin can request a full export of their data at any time.

**Acceptance Criteria:**
- Export includes: config, staff, services, appointment events, payments, notifications, analytics
- Export is scoped strictly to the requesting tenant
- Download link delivered to verified owner email only
- Signed URL expires after 24 hours

**Personas:** P1  
**Business Rules:** BR005, BR014

---

#### FR013 — Tenant Suspension and Deprovisioning
Platform operators can suspend and deprovision tenants in a controlled sequence.

**Acceptance Criteria:**
- Suspension immediately halts new bookings while preserving read access
- Deprovisioning requires explicit operator action after 30-day data export window
- Data retained for 90 days post-deprovisioning before hard deletion
- PII pseudonymised on customer erasure requests

**Personas:** P4  
**Business Rules:** BR011, BR012

---

## 6. Non-Functional Requirements Summary

| Category | Requirement | Target | NFR ID |
|---|---|---|---|
| Availability | Core booking operations uptime | 99.9% monthly | NFR001 |
| Performance | Booking confirmation end-to-end p95 | < 3 seconds | NFR004 |
| Performance | Slot availability query p95 | < 500ms | NFR005 |
| Performance | Dashboard response p95 | < 200ms | NFR006 |
| Consistency | No double-booking under concurrency | Zero violations | NFR007 |
| Consistency | Dashboard event lag | < 5 seconds | NFR008 |
| Security | Cross-tenant data isolation violations | Zero | NFR012 |
| Security | Encryption in transit and at rest | TLS 1.3 + AES-256 | NFR013 |
| Recoverability | DLQ resolution SLA | Within 4 hours | NFR016 |

---

## 7. Roadmap

### MVP (Design — current phase)
Goal: Core booking platform live with three paid tenants.

Features:
- Tenant self-service onboarding (Starter plan)
- Service and staff configuration
- Slot availability browsing
- Appointment booking (payment optional)
- Appointment cancellation and rescheduling
- Email notifications (confirmation, cancellation, reminder)
- Tenant operational dashboard
- Platform operator DLQ console
- Basic Grafana observability

*Excludes:* SMS notifications, Analytics alerts, Enterprise plan, Data export

---

### v1.0 (Planned)
Goal: Production-ready platform with full observability and Pro plan.

Features:
- Pro plan with SMS notifications
- Stream processing anomaly alerts (velocity spike, cancellation rate)
- Analytics daily summaries
- Full Grafana LGTM stack with SLO dashboards
- Tenant event replay
- Data export and GDPR erasure
- Enterprise plan (dedicated DB, custom domain, SLA)
- Peak hour detection and recommendations

---

### v1.5 (Planned)
Goal: Platform scalability improvements and channel expansion.

Features: WhatsApp channel, Google Calendar/Outlook sync, waitlist management, group bookings, recurring appointments, custom tenant branding.

---

### v2.0 (Roadmap)
Goal: Platform intelligence and integration ecosystem.

Features: AI-driven scheduling suggestions, CRM integration (Salesforce, HubSpot), video conferencing (Zoom, Google Meet), multi-language support, native iOS/Android apps, advanced reporting.

---

### Marketplace / Platform Expansion (Vision)
Public tenant directory, customer accounts spanning multiple tenants, review and rating system, staff marketplace, POS integration, inventory management, loyalty programme.

---

## 8. Out of Scope (v1)

- Video conferencing integration
- Multi-language / internationalisation
- Native mobile applications
- Marketplace / staff-to-tenant matching
- AI-driven scheduling suggestions
- POS / inventory management
- CRM integration

---

## 9. Success Metrics

| Metric | Target |
|---|---|
| Time to first booking from sign-up | < 30 minutes |
| Booking confirmation notification delivery | < 30 seconds |
| Double-booking incidents | Zero |
| Dashboard event lag | < 5 seconds |
| Core booking uptime | 99.9% per month |
| DLQ event resolution | < 4 hours |
