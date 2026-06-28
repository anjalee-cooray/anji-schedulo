# A6 · Data Privacy Architecture

**Project:** AnjiSchedulo  
**Version:** 1.0.0  
**Owner:** Pubudu Anjalee Cooray  
**Last Updated:** 2026-06-28

---

## 1. Overview

This document describes how AnjiSchedulo enforces data privacy at every architectural layer. Privacy controls are structural — not bolted on. PII minimisation, tenant isolation, encryption, and GDPR erasure are enforced through schema design, event envelope rules, log redaction, and database privilege controls.

---

## 2. PII Inventory

| Field | Table | Classification | Retention |
|---|---|---|---|
| `customer.name` | `customers` | PII — direct identifier | Pseudonymised on erasure request |
| `customer.email` | `customers` | PII — contact identifier | Pseudonymised on erasure request |
| `customer.phone` | `customers` | PII — contact identifier | Pseudonymised on erasure request |
| `staff_members.name` | `staff_members` | PII — direct identifier | Pseudonymised on erasure request |
| `staff_members.email` | `staff_members` | PII — contact identifier | Pseudonymised on erasure request |
| `tenants.owner_email` | `tenants` | PII — business contact | Retained for contract duration + 90 days |
| `notifications.recipient_contact` | `notifications` | PII — email or phone | 90 days post-deprovisioning |
| `appointment_events.payload` | `appointment_events` | PII — customer/staff snapshot | Pseudonymised in S3 archives within 24h of erasure |
| Stripe `customer_id`, `payment_intent_id` | `payments` | Pseudonymous — no direct PII | 7 years (legal obligation) |

---

## 3. Privacy-by-Design Principles

### 3.1 Data Minimisation

- Notification Service derives all content from the `AppointmentEvent.payload` snapshot. It never calls back to `booking-command-service` to fetch additional customer data.
- Analytics Service consumes only aggregate counts and timestamps — no PII fields stored in `analytics_summaries`.
- Customer accounts are not required for booking. Customers are identified by email + appointment reference only.

### 3.2 Purpose Limitation

- PII collected during booking is used exclusively for appointment fulfilment and lifecycle notifications.
- No cross-tenant aggregation of customer PII is performed anywhere in the system.
- No PII is shared with third parties beyond Stripe, SendGrid, and Twilio — each under a Data Processing Agreement.

### 3.3 Storage Limitation

- Retention periods are enforced by automated lifecycle jobs, not manual processes.
- Customer PII is pseudonymised on erasure request — not deleted — to preserve appointment event history for audit and dispute resolution.
- S3 tenant export archives expire after 90 days. Audit logs are retained permanently under S3 Object Lock (WORM).

### 3.4 Integrity and Confidentiality

- AES-256 encryption at rest on all data stores (RDS, S3, SQS, SNS, ElastiCache).
- TLS 1.3 enforced on all network paths.
- Secrets exclusively in AWS Secrets Manager — never in environment variables, Dockerfiles, or source code.

---

## 4. Tenant Isolation as a Privacy Control

A breach of tenant isolation is a privacy incident. The three-layer enforcement model is the primary privacy boundary for customer and staff data.

| Layer | Mechanism | Failure Behaviour |
|---|---|---|
| 1 — JWT `tid` claim | Token issued with authenticated `tenant_id`; cannot be self-assigned | Rejected at API Gateway on invalid signature |
| 2 — Application filter | Service reads `X-Tenant-Id` from validated JWT; all queries scoped to this value only | Application bug → caught by Layer 3 |
| 3 — PostgreSQL RLS | `SET LOCAL app.tenant_id` on every transaction; policy on every table | Null context → zero rows (safe default) |
| 4 — Cache key namespacing | Redis keys prefixed with `{tenant_id}:` | Cache miss → RLS-enforced DB query |
| 5 — Event envelope validation | Consumers reject events with missing `tenant_id` → DLQ (BR013) | No silent cross-tenant processing |

**Test coverage:**
- CI mandatory cross-tenant isolation tests: tenant_A JWT cannot read, write, or delete tenant_B data.
- Production isolation monitor: runs every 5 minutes; fires ALT005 (P1) on any violation (NFR012).

---

## 5. GDPR Erasure (Right to be Forgotten)

### 5.1 Scope

Individual customer erasure requests are submitted by the tenant admin (P1) on behalf of the customer, or by Platform Operator (P4) for compliance-driven erasure.

### 5.2 Pseudonymisation Process

```sql
-- Executed by ops-service in a single transaction
UPDATE customers
SET
    name             = 'REDACTED-' || encode(digest(customer_id::text, 'sha256'), 'hex'),
    email            = 'redacted-' || encode(digest(customer_id::text, 'sha256'), 'hex') || '@erased.anji-schedulo.com',
    phone            = NULL,
    pseudonymised_at = NOW()
WHERE tenant_id = $1 AND customer_id = $2;

-- Audit log written in same transaction (BR014)
INSERT INTO audit_log (entity_type, entity_id, operation, actor_id, actor_role, after_snapshot, tenant_id)
VALUES ('Customer', $2, 'pseudonymise', $actor_id, $actor_role, '{"pseudonymised": true}', $1);
```

### 5.3 Impact on Related Records

| Record | Impact on Erasure |
|---|---|
| `appointments` | Retained — `customer_id` FK preserved; PII fields gone from `customers` row |
| `appointment_events.payload` | PII in JSONB snapshot. S3 Object Lambda triggered within 24h to redact customer PII from archived payloads. Live PostgreSQL rows not modified (INSERT-only). |
| `notifications.recipient_contact` | Pseudonymised in-place with same pattern |
| `payments` | Not pseudonymised — 7-year legal retention; no direct PII in `payments` table (Stripe IDs only) |
| `audit_log` | Retained permanently — records the erasure action itself |

### 5.4 Event Archive Handling

1. `ops-service` publishes a `customer.erased` event on erasure request.
2. S3 Object Lambda triggers on the event archive bucket.
3. Lambda reads all tenant event archive files, redacts PII fields for the specific `customer_id`, writes back updated objects.
4. Completion logged in audit trail.

Process completes within 24 hours of the erasure request.

---

## 6. Data Flows and Privacy Controls

### 6.1 Booking Flow (Customer PII)

```
Customer browser
    │  name, email, phone in HTTPS request body (TLS 1.3)
    ▼
API Gateway  →  strips PII from access logs (gateway log filter)
    ▼
booking-command-service
    │  writes Customer row (PII encrypted at rest — AES-256, RDS KMS)
    │  AppointmentEvent.payload contains name/email snapshot (encrypted — RDS + SQS KMS)
    ▼
SNS → SQS (KMS encrypted)
    ├── notification-service: renders email/SMS from payload; stores only recipient_contact
    ├── dashboard-service: counts/timestamps only — no PII consumed
    └── analytics-service: counts/timestamps only — no PII consumed
```

### 6.2 Log Flow (PII Redaction)

```
Spring Boot application → structured JSON logs (stdout, SLF4J + Logback)
    ▼
Fluent Bit (ECS sidecar)
    │  Redact: email, phone, name, recipient_contact, authorization, *secret*key*
    │  Enrich: service_name, task_id, environment, trace_id
    ▼
Grafana Loki (S3 backend)  →  logs indexed WITHOUT any PII
```

### 6.3 Tenant Data Export Flow

```
Tenant Admin requests export → ops-service
    ▼
NDJSON archive generated (all tenant-scoped tables)
    ▼
Archive uploaded to anji-schedulo-tenant-exports-{env} S3 (private, SSE-KMS)
    ▼
Presigned URL (24h expiry) generated by ops-service
    ▼
URL emailed to tenants.owner_email ONLY (verified address from Tenant record)
    ▼
S3 object expires after 90 days (S3 lifecycle policy — BR012)
```

---

## 7. Third-Party Data Sharing

| Third Party | Data Shared | Legal Basis | DPA |
|---|---|---|---|
| **Stripe** | Stripe customer_id, payment_intent_id, billing email | Contractual necessity | Yes — Stripe DPA |
| **SendGrid** | Customer/staff name and email (notification content) | Contractual necessity | Yes — Twilio SendGrid DPA |
| **Twilio** | Customer/staff phone (SMS, Pro+ only) | Contractual necessity | Yes — Twilio DPA |
| **AWS** | All application data (data processor on our behalf) | Contractual necessity | Yes — AWS GDPR DPA |

No PII is shared with any third party for marketing, analytics, or machine learning.

---

## 8. Encryption Summary

| Data Store | At Rest | In Transit |
|---|---|---|
| PostgreSQL RDS | AES-256, AWS KMS | TLS 1.3 (`ssl=require`) |
| ElastiCache Redis | AES-256, AWS KMS | TLS 1.3 |
| Amazon SQS | SSE-KMS | HTTPS |
| Amazon SNS | SSE-KMS | HTTPS |
| Amazon S3 | SSE-KMS | HTTPS |
| AWS Secrets Manager | AES-256, AWS KMS | HTTPS |

---

## 9. Traceability

| Privacy Control | BR | NFR | FR |
|---|---|---|---|
| Tenant isolation (3-layer) | BR005 | NFR012 | — |
| PII pseudonymisation on erasure | — | — | FR013 |
| Audit log (immutable, permanent) | BR014 | — | — |
| Data retention and deletion | BR011, BR012 | — | FR013 |
| Encryption at rest + in transit | — | NFR013 | — |
| Log PII redaction (Fluent Bit) | — | NFR013 | — |
| Event snapshot (no back-calls for PII) | BR013 | NFR002 | FR007 |
| Secrets management | — | NFR013 | — |
