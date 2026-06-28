# O1 · Release Plan

**Project:** AnjiSchedulo  
**Version:** 1.0.0  
**Owner:** Pubudu Anjalee Cooray  
**Last Updated:** 2026-06-28

---

## 1. Release Strategy

AnjiSchedulo uses **continuous delivery** with gated promotion through three environments. Every commit to `main` is a candidate for production. Releases are tagged with semantic versions and deployed via the CD pipeline (CD-003) with a manual approval gate.

Release cadence:
- **Feature releases (MINOR):** every 2–4 weeks
- **Patch releases (PATCH):** as needed (bug fixes, hotfixes)
- **Major releases (MAJOR):** planned — breaking API or tenant data model changes require tenant communication 30 days in advance

---

## 2. Release Phases

### Phase 1 — MVP (v0.1.0) — Internal

**Goal:** End-to-end booking flow working for a single tenant in production.

| Feature | Status |
|---|---|
| Tenant provisioning (FR001) | Planned |
| Staff and service configuration (FR002) | Planned |
| Real-time slot availability (FR003) | Planned |
| Booking creation with Stripe payment (FR004, FR005) | Planned |
| Email notification on confirmation (FR007) | Planned |
| Basic tenant dashboard (FR008) | Planned |

**Exit criteria:** 1 pilot tenant live, 0 P1 incidents in first 48 hours.

---

### Phase 2 — v1.0.0 — General Availability

**Goal:** Full feature set for Starter and Pro tiers, publicly available.

| Feature | FR | Notes |
|---|---|---|
| Cancellation + refund | FR006 | Choreography saga with Stripe refund |
| SMS notifications | FR007 | Pro+ only via Twilio |
| Appointment reminders | FR007 | 24h before slot |
| Booking history + management | FR009 | Customer and staff views |
| DLQ management console | FR010 | Operator tooling |
| Event replay | FR011 | Operator tooling |
| Tenant data export | FR012 | GDPR FR013 dependency |
| GDPR customer erasure | FR013 | Required before GA |

**Exit criteria:** All 13 FRs implemented, cross-tenant isolation CI tests green, GDPR erasure flow tested.

---

### Phase 3 — v1.5.0 — Enterprise Features

| Feature | Notes |
|---|---|
| Enterprise SSO (SAML 2.0 / OIDC) | FR001 Enterprise tier |
| Dedicated RDS cluster provisioning | Infrastructure |
| Webhook delivery (outbound) | New — with URL allowlist (T-E03 mitigation) |
| WhatsApp notifications | Enterprise tier only |
| Custom domain support | Enterprise tier only |

---

### Phase 4 — v2.0.0 — Platform Expansion

| Feature | Notes |
|---|---|
| Recurring appointments | New domain model changes |
| Marketplace / multi-location | Breaking: tenant hierarchy changes |
| Native mobile apps | iOS + Android |
| AI-assisted scheduling | Suggested slots based on history |

---

## 3. Release Checklist (per release)

| Step | Owner | When |
|---|---|---|
| All PRs merged, feature branch deleted | Engineer | Before RC cut |
| Release candidate cut (`release/v{N}` branch) | Engineering Lead | Release day -2 |
| Staging deploy and extended validation | QA / Engineer | Release day -1 |
| Go/No-Go decision (REL-003) | Engineering Lead | Release day, 09:00 UTC |
| Production deploy with manual approval (CD-003) | Engineering Lead | Release day, 10:00 UTC |
| 10-minute post-deploy monitoring window | On-call engineer | After deploy |
| Git tag created (`v{N}`) | Engineer | After smoke tests pass |
| Changelog updated (VER-002) | Engineer | Same day |
| Internal release comms (COM-001) | Engineering Lead | Same day |
| Post-release stabilisation period | On-call | 48 hours |

---

## 4. Rollback Policy

Any P1 incident within 48 hours of a release triggers an immediate rollback assessment. See:
- [CD-004 · Rollback Flow](../04-architecture/cicd-flows/CD-004-rollback.md)
- [O3 · Rollback Plan](O3-rollback-plan.md)

---

## 5. Communication Plan

| Audience | Channel | When |
|---|---|---|
| Engineering team | Slack #deployments | Before and after deploy |
| Tenant admins (breaking change) | Email from owner account | 30 days before MAJOR release |
| Status page | status.anji-schedulo.com | During maintenance window if applicable |

---

## 6. Traceability

| Release Concern | Doc |
|---|---|
| CD pipeline | CD-003-production-deploy.md |
| Go/No-Go criteria | REL-003-go-no-go.md |
| Rollback | CD-004-rollback.md, O3-rollback-plan.md |
| Versioning and changelog | VER-002-changelog-release-notes.md |
| Internal comms | COM-001-internal-release-comms.md |
