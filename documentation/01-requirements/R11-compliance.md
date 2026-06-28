# R11 · Compliance Document

**Project:** AnjiSchedulo  
**Version:** 1.0.0  
**Owner:** Pubudu Anjalee Cooray  
**Last Updated:** 2026-06-28

---

## 1. Overview

This document defines the compliance obligations for AnjiSchedulo and maps each obligation to the technical and procedural controls that satisfy it. AnjiSchedulo operates as a multi-tenant B2B SaaS platform handling personal data of end customers (P3), staff members (P2), and tenant business owners (P1). Compliance spans data protection, payment security, data residency, and operational integrity.

---

## 2. Applicable Regulations and Standards

| Regulation / Standard | Scope | Relevance |
|---|---|---|
| **GDPR** (EU) 2016/679 | Personal data of EU-based data subjects | End customers, staff members, tenant admins whose data is processed |
| **UK GDPR / DPA 2018** | UK data subjects post-Brexit | Same scope as EU GDPR for UK-based tenants and customers |
| **PCI-DSS** | Payment card data | Payment processing via Stripe; AnjiSchedulo does not store raw card data |
| **ISO 27001 (principles)** | Information security management | Controls applied to encryption, access management, incident response |

---

## 3. Data Protection (GDPR / UK GDPR)

### 3.1 Lawful Basis for Processing

| Processing Activity | Lawful Basis | Data Subjects |
|---|---|---|
| Appointment booking | Contractual necessity (Art. 6(1)(b)) | End customers |
| Sending booking confirmations and reminders | Contractual necessity (Art. 6(1)(b)) | End customers, staff members |
| Billing and payment processing | Contractual necessity (Art. 6(1)(b)) | Tenant admins |
| Audit logging | Legitimate interests (Art. 6(1)(f)) | All data subjects |
| Payment record retention (7 years) | Legal obligation (Art. 6(1)(c)) | Tenant admins, end customers |

---

### 3.2 Personal Data Inventory

| Data Category | Fields | Storage Location | Retention Period |
|---|---|---|---|
| **End Customer PII** | name, email, phone | `appointments` table + event log | Duration of tenant subscription + 90 days post-deprovisioning |
| **Staff Member PII** | name, email, phone, working hours | `staff_members` table | Duration of employment record + 90 days |
| **Tenant Admin PII** | name, business email | `tenants` table | Duration of subscription + 90 days post-deprovisioning |
| **Payment records** | Stripe customer ID, transaction IDs (no raw card data) | `payments` table | 7 years (legal obligation) |
| **Audit log** | Actor identity, operation, timestamp | `audit_log` table | Permanent (INSERT-only, never deleted) |

---

### 3.3 Data Subject Rights

#### 3.3.1 Right of Access (Art. 15 GDPR)
- **Control:** Tenant admins can export all tenant-scoped data via the self-service data export (FR012). Export is scoped strictly to the requesting tenant's data and delivered to the verified owner email only (BR005).
- **SLA:** Export generated and link delivered within 72 hours of request.

#### 3.3.2 Right to Erasure / Right to be Forgotten (Art. 17 GDPR)
- **Control:** Platform Operator pseudonymises PII fields for specific customer records on verified erasure request (FR013).
- **Pseudonymisation fields:** `customer_name`, `customer_email`, `customer_phone` in `appointments` and event log snapshots.
- **Exceptions:** Payment records cannot be erased during the 7-year legal retention period. Audit log entries are permanent.
- **SLA:** Pseudonymisation completed within 30 days of verified request.

#### 3.3.3 Right to Data Portability (Art. 20 GDPR)
- **Control:** Tenant data export (FR012) provides a structured, machine-readable archive of all tenant data including appointments, configuration, and analytics.
- **Format:** JSON archive delivered via signed S3 URL.

#### 3.3.4 Right to Restriction of Processing (Art. 18 GDPR)
- **Control:** Tenant suspension (FR013) halts all active processing for the tenant while preserving read access. No new bookings or data modifications occur during suspension.

---

### 3.4 Data Retention Policy

| Data Category | Retention Period | Deletion Mechanism |
|---|---|---|
| Tenant operational data | 90 days post-deprovisioning | Hard deletion via scheduled job |
| PII (post-erasure request) | Immediately pseudonymised | Replace PII fields with anonymised tokens |
| Payment records | 7 years | Excluded from all deletion jobs |
| Audit log | Permanent | Never deleted; INSERT-only table privilege |
| S3 export archives | 24 hours (signed URL expiry) | S3 lifecycle policy auto-deletes after 24h |
| SQS messages (DLQ) | 14 days | SQS retention policy |

*Enforced by:* Business Rules BR011, BR012.

---

### 3.5 Data Transfers

- **Primary deployment region:** AWS ap-southeast-1 (Singapore). Data residency can be configured per Enterprise tenant.
- **Transfers to Stripe:** Payment data transferred to Stripe under Stripe's Data Processing Agreement. Stripe is PCI-DSS Level 1 certified.
- **Transfers to notification providers:** Notification content (name, email, appointment details) transferred to SMTP/SMS/WhatsApp providers under DPA agreements. Content is event-snapshot derived and does not include payment data.
- **Cross-region replication:** S3 export archives are not replicated cross-region by default. Enterprise tenants may configure data residency to restrict processing to a single region.

---

### 3.6 Privacy by Design

| Principle | Implementation |
|---|---|
| Data minimisation | Notification content derived from event snapshot only; no back-calls to fetch additional PII (BR013) |
| Purpose limitation | PII collected for booking fulfilment only; not used for marketing without explicit consent |
| Storage limitation | Retention periods enforced by automated lifecycle jobs; no indefinite data retention |
| Integrity and confidentiality | AES-256 at rest (RDS, S3, SQS, ElastiCache); TLS 1.3 in transit (NFR013) |
| Accountability | Immutable audit log for all write operations (BR014); operator actions logged with actor identity |

---

## 4. Payment Security (PCI-DSS)

### 4.1 Scope Reduction

AnjiSchedulo deliberately reduces its PCI-DSS scope by delegating all payment handling to Stripe:

- **No raw card data is stored, transmitted, or processed by AnjiSchedulo.** Customers enter card details directly into Stripe's hosted payment form or Stripe Elements (rendered in the customer's browser).
- AnjiSchedulo stores only: Stripe `customer_id`, `payment_intent_id`, `charge_id`, and transaction status.

### 4.2 Controls

| Control | Detail |
|---|---|
| Stripe webhook signature validation | All Stripe webhooks are validated using Stripe's webhook signing secret before processing |
| Stripe API key rotation | Stripe API keys stored in AWS Secrets Manager; rotation period: 90 days |
| Refund audit trail | Every refund action is recorded in the audit log with actor identity and Stripe transaction reference |
| Payment failure handling | Saga compensation releases slot on payment failure; no partial booking state persists (BR004) |

---

## 5. Information Security

### 5.1 Encryption

| Layer | Standard | Key Management |
|---|---|---|
| Data at rest (RDS) | AES-256 | AWS KMS — CMK per environment |
| Data at rest (S3) | AES-256 | AWS KMS — CMK per environment |
| Data at rest (SQS) | AES-256 | AWS KMS |
| Data at rest (ElastiCache) | AES-256 | AWS KMS |
| Data in transit | TLS 1.3 | AWS ACM managed certificates |

### 5.2 Access Control

- **Authentication:** JWT RS256 signed tokens with 15-minute access token expiry (see A4-security-model.md).
- **Authorisation:** RBAC with four roles: `tenant_admin`, `staff`, `customer`, `platform_operator`. Enforced at API Gateway, service layer, and DB (RLS).
- **Tenant isolation:** PostgreSQL RLS enforced as a safety net independent of application code. Safe default — null tenant context returns zero rows (BR005, NFR012).
- **Principle of least privilege:** Each service uses a dedicated DB user with only the permissions required for its operations. `booking_app_user` has INSERT-only on `audit_log`.

### 5.3 Secrets Management

| Secret Type | Storage | Rotation Period |
|---|---|---|
| Database credentials | AWS Secrets Manager | 30 days (automated rotation) |
| JWT signing key (RS256 private key) | AWS Secrets Manager | 90 days |
| Stripe API keys | AWS Secrets Manager | 90 days |
| SMTP / SMS API keys | AWS Secrets Manager | 90 days |
| AWS KMS CMKs | AWS KMS | Annual key rotation |

---

## 6. Operational Compliance

### 6.1 Audit and Logging

- Every write operation modifying tenant-scoped data produces an immutable audit log entry (BR014).
- Audit log is INSERT-only; `booking_app_user` has no UPDATE or DELETE privilege on `audit_log`.
- Log fields: `timestamp`, `actor_id`, `actor_role`, `tenant_id`, `operation`, `entity_type`, `entity_id`, `before_state`, `after_state`, `correlation_id`.
- Audit log is permanent and excluded from all data deletion jobs.

### 6.2 Incident Response

- All security incidents are classified by severity (P1–P3).
- P1 incidents (data breach, tenant isolation violation, payment fraud) trigger immediate PagerDuty escalation and require notification of affected tenants within 72 hours (GDPR Art. 33).
- Incident runbooks are documented in O5-incident-response.md.

### 6.3 Data Breach Notification

| Obligation | Deadline | Mechanism |
|---|---|---|
| Notify supervisory authority | Within 72 hours of becoming aware (GDPR Art. 33) | Platform Operator initiates notification via legal/compliance process |
| Notify affected data subjects | Without undue delay if high risk (GDPR Art. 34) | Direct email to affected customers via verified email addresses |
| Internal incident record | Immediately | Incident recorded in audit log and incident tracker |

---

## 7. Compliance Traceability

| Obligation | Control | BR / NFR | Document |
|---|---|---|---|
| GDPR data minimisation | Event-snapshot notifications; no back-calls | BR013 | A4-security-model.md |
| GDPR data retention | 90-day post-deprovisioning deletion; 7-year payment retention | BR011, BR012 | O1-release-plan.md |
| GDPR right to erasure | PII pseudonymisation on request | FR013 | — |
| GDPR right to portability | Full data export (FR012) | BR005, BR014 | — |
| GDPR audit accountability | Immutable audit log | BR014 | DM1-data-dictionary.md |
| GDPR data breach notification | 72-hour supervisory notification | — | O5-incident-response.md |
| PCI-DSS scope reduction | No card data stored; Stripe delegation | — | A4-security-model.md |
| Encryption at rest | AES-256 across all data stores | NFR013 | A7-infrastructure.md |
| Encryption in transit | TLS 1.3 | NFR013 | A7-infrastructure.md |
| Tenant isolation | RLS + JWT + app filter | BR005, NFR012 | A3-multi-tenancy.md |
| Secrets management | AWS Secrets Manager + rotation | — | O6-secrets-rotation-policy.md |
