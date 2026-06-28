---
title: Security Model
layer: 04-architecture
status: current
lastUpdated: 2026-06-28
---

# A4 · Security Model

## Overview

AnjiSchedulo's security model is built on three enforcement layers: authentication at the gateway, authorisation at the service, and isolation at the database. Each layer is independently capable of preventing unauthorised access. The model is designed so that a failure in any single layer does not expose tenant data.

---

## Authentication

### Mechanism

JWT RS256 (asymmetric signing). api-gateway holds the private key and issues tokens. All downstream services hold the public key and validate tokens independently — no round-trip to a centralised auth service on every request.

### Token Issuance

api-gateway issues tokens after verifying credentials (email + password hash comparison against tenant_service's user store). Token issuance endpoints:

| Endpoint | Auth Required | Returns |
|---|---|---|
| `POST /api/v1/auth/login` | No | `{ access_token, refresh_token }` |
| `POST /api/v1/auth/refresh` | Refresh token (Bearer) | `{ access_token }` |
| `POST /api/v1/auth/logout` | Access token (Bearer) | 204 No Content |

### Token Claims

Every access token contains:

```json
{
  "sub": "<user_id>",
  "tenant_id": "<tenant_uuid>",
  "role": "tenant_admin | staff | customer | platform_operator",
  "plan": "starter | pro | enterprise",
  "email": "<user_email>",
  "iat": 1751088000,
  "exp": 1751091600,
  "jti": "<token_uuid>"
}
```

Refresh tokens carry only `sub`, `jti`, `iat`, `exp`.

### Token Expiry

| Token Type | TTL | Rationale |
|---|---|---|
| Access token | 60 minutes | Long enough to avoid mid-session expiry; short enough to limit blast radius of token compromise |
| Refresh token | 7 days | Matches expected session duration for recurring admin users |

Refresh tokens are single-use (rotation on every refresh). A compromised refresh token can only be exploited once before it is invalidated.

### Public Endpoints

The following endpoints require no token:

| Endpoint | Purpose |
|---|---|
| `POST /api/v1/tenants` | Tenant registration |
| `POST /api/v1/auth/login` | Authentication |
| `POST /api/v1/auth/refresh` | Token refresh |
| `GET /api/v1/tenants/{slug}/availability` | Customer-facing slot browsing |
| `GET /api/v1/tenants/{slug}/services` | Customer-facing service catalogue |
| `POST /api/v1/tenants/{slug}/bookings` | Customer booking submission |
| `GET /health` | Healthcheck |

Public booking endpoints (`/{slug}/availability`, `/{slug}/services`, `/{slug}/bookings`) are scoped strictly to the tenant identified by the URL slug. They are read-only except for booking submission, which creates records only within the slug-identified tenant.

### Enterprise SSO

Enterprise tenants can configure SAML 2.0 or OIDC federation. The identity provider (IDP) is configured per tenant by the Platform Operator. On SSO login:

1. The user is redirected to the tenant's IDP.
2. The IDP returns a SAML assertion or OIDC ID token.
3. api-gateway validates the assertion/token against the configured IDP metadata.
4. api-gateway issues a standard AnjiSchedulo JWT with `tenant_id` and `role` derived from the IDP claims mapping configured by the Platform Operator.
5. All downstream services see a standard JWT — SSO is transparent to services beyond api-gateway.

---

## Authorisation

### Model

Role-Based Access Control (RBAC) with three enforcement layers: gateway (rate limiting by role), service (role claim validation per endpoint), database (RLS by tenant_id).

### Roles

| Role | Persona | Scope | Permissions |
|---|---|---|---|
| `tenant_admin` | P1 — Tenant Admin | Own tenant | Full CRUD on tenant config, staff, services, working hours. Read own bookings and dashboard. Export own data. |
| `staff` | P2 — Staff Member | Own tenant, own records | Read own schedule, upcoming bookings. Block own unavailability. Cannot manage other staff or view tenant config. |
| `customer` | P3 — End Customer | Own bookings only | Create booking, view own upcoming bookings, cancel own bookings. Cannot read any tenant admin data. |
| `platform_operator` | P4 — Platform Operator | Cross-tenant (explicit) | DLQ triage, replay, data export, tenant deprovisioning. All actions are audit-logged. Cannot modify tenant application data. |

### Endpoint Permission Matrix

| Endpoint | tenant_admin | staff | customer | platform_operator |
|---|---|---|---|---|
| `POST /api/v1/tenants` | — (public) | — | — | — |
| `GET /api/v1/tenants/{id}` | ✓ own | ✗ | ✗ | ✓ any |
| `PUT /api/v1/tenants/{id}/config` | ✓ own | ✗ | ✗ | ✗ |
| `POST /api/v1/tenants/{id}/staff` | ✓ own | ✗ | ✗ | ✗ |
| `GET /api/v1/tenants/{slug}/availability` | — (public) | — | — | — |
| `POST /api/v1/tenants/{slug}/bookings` | — (public) | — | — | — |
| `GET /api/v1/bookings/{id}` | ✓ own tenant | ✓ own | ✓ own | ✓ any |
| `DELETE /api/v1/bookings/{id}` | ✓ own tenant | ✗ | ✓ own | ✗ |
| `GET /api/v1/dashboard` | ✓ own tenant | ✗ | ✗ | ✓ any |
| `GET /api/v1/ops/dlq` | ✗ | ✗ | ✗ | ✓ |
| `POST /api/v1/ops/replay` | ✗ | ✗ | ✗ | ✓ |
| `POST /api/v1/tenants/{id}/export` | ✓ own | ✗ | ✗ | ✓ any |

### Enforcement

1. **api-gateway:** Validates JWT signature and expiry on every authenticated request. Rejects tokens with invalid signatures or expired `exp`. Rate-limits by `tenant_id` to prevent individual tenant abuse from affecting shared infrastructure.
2. **Service layer:** Each NestJS service endpoint is decorated with a `@Roles(...)` guard. Requests with a `role` claim that does not match the required role are rejected with `403 Forbidden`. The `tenant_id` from the token is compared to the `tenant_id` in the URL path or request body — mismatches are rejected.
3. **Database layer:** PostgreSQL RLS enforces `tenant_id` scoping at the database level. Even if a service bug passes the wrong `tenant_id`, the RLS policy prevents the query from returning another tenant's data.

---

## Tenant Isolation

See [A3 · Multi-Tenancy Architecture](A3-multi-tenancy.md) for the full three-layer isolation model.

**Key guarantees:**
- `tenant_id` is established at JWT issuance and validated at the gateway, service, and database layers.
- A missing tenant context returns zero rows at the database layer (safe default, BR005).
- Platform Operator cross-tenant access uses a separate DB role with `BYPASSRLS`, is always explicitly scoped to a single `tenant_id`, and every action is audit-logged.

---

## Data Protection

### Encryption in Transit

| Connection | Protocol |
|---|---|
| Client → CloudFront | TLS 1.3 |
| CloudFront → ALB | TLS 1.3 |
| ALB → ECS services | TLS 1.3 (within VPC) |
| ECS services → PostgreSQL (via PgBouncer) | TLS 1.3 |
| ECS services → Redis (ElastiCache) | TLS 1.3 |
| ECS services → AWS SNS/SQS | TLS 1.3 (HTTPS) |
| ECS services → Stripe API | TLS 1.3 (HTTPS) |
| ECS services → SendGrid API | TLS 1.3 (HTTPS) |

### Encryption at Rest

| Service | Algorithm | Key Management |
|---|---|---|
| RDS PostgreSQL | AES-256 | AWS KMS managed key per environment |
| ElastiCache Redis | AES-256 | AWS KMS managed key |
| S3 buckets (exports, backups) | AES-256 (SSE-S3 and SSE-KMS) | AWS KMS managed key |
| SQS queues | AES-256 | AWS KMS managed key |
| SNS topics | AES-256 | AWS KMS managed key |
| Secrets Manager | AES-256 | AWS KMS managed key |

### PII Fields

The following fields are classified as Personally Identifiable Information:

| Field | Entity | Storage | Pseudonymisation | Erasure on Offboarding |
|---|---|---|---|---|
| `owner_email` | Tenant | Plain (unique index) | No | Deleted after 90 days (FR013) |
| `owner_name` | Tenant | Plain | No | Deleted after 90 days |
| `customer_email` | Customer | Plain | No | Deleted after 90 days |
| `customer_name` | Customer | Plain | No | Deleted after 90 days |
| `customer_phone` | Customer | Plain | No | Deleted after 90 days |
| `staff_email` | Staff | Plain | No | Deleted after 90 days |
| `card_last_four` | Payment | Plain (non-recoverable fragment) | n/a — not a full PAN | Deleted after 90 days |
| `stripe_payment_method_id` | Payment | Encrypted (Stripe-side) | Stripe token — not stored raw | Deleted after 90 days |

No full payment card numbers (PAN) are stored by AnjiSchedulo. Stripe Tokens are stored as references only.

**Log redaction:** PII fields (`email`, `name`, `phone`, `stripe_payment_method_id`) are always redacted in logs. Fluent Bit applies a `record_modifier` filter to scrub these fields before log lines reach Loki. This is enforced at the collection layer — individual services do not need to implement their own redaction.

**GDPR right to erasure:** Handled by the data export and deprovisioning flow (UJ007, FR012, FR013). On deprovisioning request, tenant data is soft-deleted immediately (access revoked). Permanent deletion of all PII occurs 90 days after the deprovisioning request. A deletion confirmation record (containing only `tenant_id` and deletion timestamp) is retained for compliance.

---

## Secrets Management

All secrets are stored in AWS Secrets Manager, encrypted with KMS. Secrets are injected into ECS tasks at startup via ECS Secret references in the task definition — secrets are never written to environment variables in plaintext or included in container images.

| Secret | Rotation Period | Used By |
|---|---|---|
| JWT private key (RS256) | 90 days | api-gateway |
| JWT public key | On private key rotation | All services |
| PostgreSQL master password | 30 days | ECS task (via PgBouncer) |
| PostgreSQL shared pool credentials | 30 days | All services |
| Stripe API secret key | 90 days | payment-service |
| SendGrid API key | 90 days | notification-service |
| Twilio API key | 90 days | notification-service |

On secret rotation, ECS tasks are rolled with a forced deployment so they pick up the new secret values. Zero-downtime rotation: old and new key versions are valid simultaneously for 5 minutes during the rolling update window.
