# A5 · Threat Model

**Project:** AnjiSchedulo  
**Version:** 1.0.0  
**Owner:** Pubudu Anjalee Cooray  
**Last Updated:** 2026-06-28

---

## 1. Overview

This document applies STRIDE threat modelling to AnjiSchedulo. Each threat is assessed against the architecture, mapped to the attack surface, assigned a risk rating, and matched to a control. The system processes personal data (PII), payment references, and multi-tenant business data — making confidentiality, integrity, and tenant isolation the highest-priority concerns.

---

## 2. System Boundaries and Trust Zones

```
[ Internet ]
    │
    ▼
[ CloudFront CDN ]
    │
    ▼
[ AWS ALB ]  ──── TLS 1.3 everywhere below this line ────────
    │
    ▼
[ API Gateway Service ]   ← JWT issuance, route auth, rate limiting
    │
    ├──► [ booking-command-service ]
    ├──► [ availability-service ]
    ├──► [ tenant-service ]
    ├──► [ dashboard-service ]
    └──► [ ops-service ]
              │
              ▼
    [ PostgreSQL RDS ]  [ ElastiCache Redis ]  [ SNS/SQS ]
              │
              ▼
    [ Stripe API ]  [ SendGrid ]  [ Twilio ]
```

**Trust boundaries:**
- Internet → CloudFront: untrusted — all input treated as adversarial
- CloudFront → ALB: AWS-managed; trusted for TLS termination
- ALB → Services: internal VPC; still authenticated via JWT
- Services → RDS/Redis/SQS: VPC-private; no public routing
- Services → Stripe/SendGrid/Twilio: external; validated via HTTPS + API key

---

## 3. STRIDE Threat Catalogue

### 3.1 Spoofing

#### T-S01 — JWT Token Forgery
**Target:** All authenticated API endpoints  
**Description:** An attacker forges a JWT to impersonate a tenant admin, staff member, or operator.  
**Risk:** Critical  
**Controls:**
- RS256 asymmetric signing: forging requires the private key, held exclusively in AWS Secrets Manager
- 15-minute access token expiry limits the window of a stolen token
- `jti` claim enables per-token revocation
- `tid` claim is set at issuance from the authenticated Tenant record — cannot be self-assigned

---

#### T-S02 — Session Cookie Theft
**Target:** Refresh token endpoint (`POST /auth/refresh`)  
**Description:** An attacker steals the refresh token cookie via XSS or network interception.  
**Risk:** High  
**Controls:**
- Refresh token stored in `HttpOnly Secure SameSite=Strict` cookie — not accessible to JavaScript
- TLS 1.3 enforced on all paths
- Refresh token rotation: every `/auth/refresh` call issues a new token and invalidates the old one
- `SameSite=Strict` prevents CSRF-based token use from cross-origin requests

---

#### T-S03 — Stripe Webhook Spoofing
**Target:** `POST /webhooks/stripe` endpoint  
**Description:** An attacker sends a crafted Stripe webhook to trigger `payment.captured` without a real payment.  
**Risk:** High — could confirm a booking without charging the customer  
**Controls:**
- All Stripe webhooks validated using `stripe.webhooks.constructEvent()` with the `Stripe-Signature` header
- Webhook signing secret stored in AWS Secrets Manager
- Requests with invalid signatures rejected with 400 before any processing

---

### 3.2 Tampering

#### T-T01 — Cross-Tenant Data Write
**Target:** PostgreSQL RDS  
**Description:** Buggy application code or an attacker writes data under a different tenant's `tenant_id`.  
**Risk:** Critical  
**Controls:**
- JWT `tid` → application filter → PostgreSQL RLS (three-layer isolation, BR005, NFR012)
- RLS restricts `INSERT` to rows where `tenant_id = current_setting('app.tenant_id')`
- CI cross-tenant isolation tests enforce this at every merge

---

#### T-T02 — Event Payload Tampering
**Target:** SNS/SQS message bus  
**Description:** An attacker modifies an event payload in transit (e.g. changes `tenant_id` in `appointment.confirmed`).  
**Risk:** Medium  
**Controls:**
- All SNS/SQS traffic via HTTPS; VPC endpoints prevent internet routing
- SQS SSE-KMS encryption at rest
- Consumers validate event envelope schema and `tenant_id` before processing (BR013)
- Invalid envelopes → DLQ, never silently processed

---

#### T-T03 — Audit Log Tampering
**Target:** `audit_log` table  
**Description:** An insider or compromised application user modifies audit records to hide activity.  
**Risk:** High  
**Controls:**
- `booking_app_user` has INSERT-only privilege on `audit_log` — UPDATE and DELETE explicitly revoked (BR014)
- Nightly export to `anji-schedulo-audit-logs` S3 bucket with Object Lock (WORM mode)
- CloudTrail logs all DDL and privilege changes

---

### 3.3 Repudiation

#### T-R01 — Actor Denies Performing an Action
**Target:** Booking, cancellation, configuration changes  
**Description:** A tenant admin or operator claims they did not perform an action that modified data.  
**Risk:** Medium  
**Controls:**
- Every write produces an immutable `audit_log` entry with `actor_id`, `actor_role`, `occurred_at`, before/after snapshot (BR014)
- `correlation_id` traces every action from HTTP request through event to audit log
- JWT `jti` uniquely identifies the token used — logged in each audit entry

---

### 3.4 Information Disclosure

#### T-I01 — Cross-Tenant Data Leak via API
**Target:** All tenant-scoped API endpoints  
**Description:** tenant_A admin uses their JWT to read tenant_B's appointments or customer data.  
**Risk:** Critical  
**Controls:**
- Three-layer isolation: JWT `tid` → app filter → RLS (BR005, NFR012)
- RLS safe default: null tenant context returns zero rows — never all rows
- Production isolation monitor runs every 5 minutes; fires ALT005 (P1) on any violation
- CI integration tests: tenant_A JWT on tenant_B resources returns 404/empty — never tenant_B data

---

#### T-I02 — PII Exposure in Logs
**Target:** Application logs forwarded to Grafana Loki  
**Description:** Customer or staff PII (email, phone, name) appears in log messages.  
**Risk:** High — GDPR Article 5 (data minimisation)  
**Controls:**
- Fluent Bit PII redaction filter strips `email`, `phone`, `name`, `recipient_contact`, `authorization` before forwarding to Loki
- Application logging guidelines: log IDs, not contact details
- Fluent Bit redaction runs regardless of application behaviour

---

#### T-I03 — Secrets Exposure
**Target:** Stripe API key, JWT private key, DB credentials  
**Description:** Secrets exposed via source code, environment variables, or CI logs.  
**Risk:** Critical  
**Controls:**
- All secrets in AWS Secrets Manager — never in code, `.env`, Dockerfiles, or CI variables
- ECS task definitions reference secrets by ARN — values injected at container start
- CloudTrail alert on unexpected IAM principal calling `secretsmanager:GetSecretValue`

---

#### T-I04 — Tenant Data Exposure via Export Link
**Target:** Signed S3 URL for tenant data exports (FR012)  
**Description:** An attacker intercepts or guesses a signed URL to download another tenant's archive.  
**Risk:** Medium  
**Controls:**
- Signed URLs are 24-hour expiry, HMAC-SHA256 signed
- URL delivered exclusively to the verified owner email — not returned in the API response
- S3 bucket is private; no public access; no ACL grants

---

### 3.5 Denial of Service

#### T-D01 — API Rate Limit Abuse
**Target:** Public booking endpoints  
**Description:** An attacker floods the booking API to exhaust ECS capacity or hold all slots.  
**Risk:** High  
**Controls:**
- API Gateway rate limiting per `tenant_id` and per source IP (NFR011)
- AWS WAF on CloudFront: rate rule blocks IPs exceeding 1,000 requests/minute
- ECS HPA scales horizontally on CPU/queue depth

---

#### T-D02 — Noisy-Neighbour DB Exhaustion
**Target:** PostgreSQL shared cluster  
**Description:** A high-volume Starter/Pro tenant exhausts PgBouncer connections, degrading other tenants.  
**Risk:** Medium  
**Controls:**
- PgBouncer transaction-mode pooling: max 200 connections per service — shared across tenants
- Per-tenant API rate limiting at API Gateway (NFR011)
- Enterprise tenants on dedicated DB clusters — physically isolated

---

#### T-D03 — Event Bus Flooding
**Target:** SNS/SQS  
**Description:** A malfunctioning service repeatedly publishes events, overwhelming consumer queues.  
**Risk:** Medium  
**Controls:**
- Outbox relay rate-limited: 500 events/second maximum
- ALT006 fires when outbox backlog exceeds 100 records
- SQS FIFO `MessageGroupId = tenant_id` limits blast radius to one tenant's partition

---

### 3.6 Elevation of Privilege

#### T-E01 — Role Escalation via JWT Manipulation
**Target:** RBAC enforcement at API Gateway  
**Description:** An attacker modifies the `role` claim in their JWT to gain `platform_operator` privileges.  
**Risk:** Critical  
**Controls:**
- RS256: modifying any JWT claim invalidates the signature — rejected at API Gateway
- Role validated against route permission matrix before forwarding to services

---

#### T-E02 — Tenant Escalation via `tid` Claim Override
**Target:** Tenant-scoped service endpoints  
**Description:** A tenant user modifies the `tid` claim in their token to access another tenant's resources.  
**Risk:** Critical  
**Controls:**
- RS256 signature validation: any `tid` modification invalidates the token
- Even with a valid signature, three isolation layers block the request

---

#### T-E03 — SSRF via Webhook Callback URL
**Target:** Future webhook integration (v1.5, not in scope v1)  
**Risk:** Low — documented for future mitigation  
**Mitigation (v1.5+):** Webhook URL allowlist; outbound request blocklist for private IP ranges; VPC egress filtering

---

## 4. Risk Summary

| ID | Threat | STRIDE | Risk | Status |
|---|---|---|---|---|
| T-S01 | JWT token forgery | Spoofing | Critical | Mitigated (RS256 + Secrets Manager) |
| T-S02 | Session cookie theft | Spoofing | High | Mitigated (HttpOnly + token rotation) |
| T-S03 | Stripe webhook spoofing | Spoofing | High | Mitigated (signature validation) |
| T-T01 | Cross-tenant data write | Tampering | Critical | Mitigated (3-layer isolation) |
| T-T02 | Event payload tampering | Tampering | Medium | Mitigated (TLS + KMS + envelope validation) |
| T-T03 | Audit log tampering | Tampering | High | Mitigated (INSERT-only + WORM S3) |
| T-R01 | Actor repudiates action | Repudiation | Medium | Mitigated (audit log + correlation_id) |
| T-I01 | Cross-tenant data leak | Info Disclosure | Critical | Mitigated (3-layer isolation + monitor) |
| T-I02 | PII in logs | Info Disclosure | High | Mitigated (Fluent Bit redaction) |
| T-I03 | Secrets exposure | Info Disclosure | Critical | Mitigated (Secrets Manager) |
| T-I04 | Export link exposure | Info Disclosure | Medium | Mitigated (presigned URL + email delivery) |
| T-D01 | API rate limit abuse | DoS | High | Mitigated (WAF + rate limiting + HPA) |
| T-D02 | Noisy-neighbour DB exhaustion | DoS | Medium | Mitigated (PgBouncer + per-tenant quotas) |
| T-D03 | Event bus flooding | DoS | Medium | Mitigated (rate limit + ALT006) |
| T-E01 | Role escalation via JWT | Elevation | Critical | Mitigated (RS256 signature) |
| T-E02 | Tenant escalation via tid | Elevation | Critical | Mitigated (RS256 + 3-layer isolation) |
| T-E03 | SSRF via webhook URL | Elevation | Low | Accepted (not in v1 scope) |

---

## 5. Residual Risks

| Risk | Reason Accepted | Mitigation Plan |
|---|---|---|
| Stripe API key compromise | Manual 90-day rotation; human process risk | Automate rotation in v1.5 |
| Insider threat (platform operator) | Operators have broad DB access by design | CloudTrail alerts on privileged actions; separation of duties |
| T-E03 (SSRF) | Not in v1 scope | URL allowlist in v1.5 with webhook feature |

---

## 6. Traceability

| Threat ID | BR | NFR | Doc |
|---|---|---|---|
| T-S01, T-E01, T-E02 | — | NFR012 | A4-security-model.md |
| T-T01, T-I01 | BR005 | NFR012 | A3-multi-tenancy.md |
| T-T03, T-R01 | BR014 | — | D8-db-schema.md |
| T-I02 | — | NFR013 | A11-observability.md |
| T-I03 | — | NFR013 | O6-secrets-rotation-policy.md |
| T-D01, T-D02 | — | NFR011 | A8-scaling-strategy.md |
