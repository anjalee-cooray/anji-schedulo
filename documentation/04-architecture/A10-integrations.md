# A10 · Integrations

**Project:** AnjiSchedulo  
**Version:** 1.0.0  
**Owner:** Pubudu Anjalee Cooray  
**Last Updated:** 2026-06-28

---

## 1. Overview

AnjiSchedulo integrates with four external systems: Stripe (payment), SendGrid (email), Twilio (SMS), and AWS-native services. All external integrations follow the same design principles:

- **Decoupled from core booking** — provider failures never block booking confirmation (BR007)
- **Secrets in AWS Secrets Manager** — no API keys in code or environment variables
- **TLS on all outbound calls** — no plaintext HTTP to external providers
- **Idempotent retries** — duplicate calls produce no additional charges or messages

---

## 2. Stripe — Payment Processing

### 2.1 Purpose

Stripe handles all payment capture and refund operations. AnjiSchedulo never stores, processes, or transmits raw card data — card input is handled directly by Stripe Elements in the customer's browser.

### 2.2 Flow

```
Customer browser
    │  Card entered into Stripe Elements (JS SDK)
    │  Stripe returns PaymentIntent.client_secret
    ▼
Customer → API Gateway
    │  POST /tenants/{slug}/bookings { stripe_payment_intent_id, idempotency_key }
    ▼
booking-command-service (saga orchestrator)
    ├── Step 1: Slot availability check
    ├── Step 2: payment-service → Stripe API (POST /v1/payment_intents/{id}/confirm)
    └── Step 3: Write appointment.confirmed → Outbox → SNS → notification-service
```

### 2.3 Data Exchanged

| Direction | Data |
|---|---|
| AnjiSchedulo → Stripe | `payment_intent_id`, `amount`, `currency` — no card data |
| Stripe → AnjiSchedulo | `PaymentIntent.status`, `stripe_charge_id`, `stripe_refund_id` |
| Stripe → AnjiSchedulo (webhook) | `payment_intent.succeeded`, `charge.refunded` events |

### 2.4 Failure Handling

| Failure | Behaviour | Business Rule |
|---|---|---|
| Payment declined | Saga compensates: slot released; 402 returned | BR004 |
| Stripe API timeout | Retry up to 3× with exponential backoff; 502 after 3 failures | BR004 |
| Refund failure on cancellation | DLQ; slot still released; customer still notified | BR009 |
| Stripe webhook delivery failure | Stripe retries up to 3 days; AnjiSchedulo processes idempotently via signature validation | — |

### 2.5 Security

- Stripe secret key: `anji-schedulo/{env}/stripe/secret-key` in Secrets Manager; rotated every 90 days
- All webhook events validated via `Stripe-Signature` header + webhook signing secret
- PCI-DSS scope: SAQ A (no card data handled by AnjiSchedulo servers)

---

## 3. SendGrid — Email Notifications

### 3.1 Purpose

SendGrid delivers all transactional email notifications for all tiers (Starter, Pro, Enterprise).

### 3.2 Flow

```
appointment.confirmed → SNS → notification-service (SQS consumer)
    │  Reads customer name, email, appointment details from event payload
    │  Selects template by event_type
    ▼
notification-service → SendGrid API (POST /v3/mail/send)
    ▼
SendGrid → Customer/Staff inbox
```

### 3.3 Email Templates

| Template | Trigger | Recipients |
|---|---|---|
| `booking-confirmation` | `appointment.confirmed` | Customer, Staff |
| `booking-cancellation` | `appointment.cancelled` | Customer, Staff |
| `booking-rescheduled` | `appointment.rescheduled` | Customer, Staff |
| `appointment-reminder` | Scheduled job (24h before slot) | Customer |
| `tenant-welcome` | `tenant.provisioned` | Tenant Admin |

### 3.4 Failure Handling

| Failure | Behaviour |
|---|---|
| SendGrid 4xx | Marked `failed`; not retried (client error) |
| SendGrid 5xx / timeout | Retry up to 4× with exponential backoff (30s, 60s, 120s, 240s) |
| 4 retries exhausted | Status → `dlq`; ALT008 fires; DLQ entry created |
| Email bounce | SendGrid webhook fires; logged; no automatic retry |

Notification failures never block or roll back the triggering booking operation (BR007).

### 3.5 Security

- SendGrid API key: `anji-schedulo/{env}/sendgrid/api-key`; rotated every 180 days
- Outbound connection HTTPS only
- PII in notification content — encrypted in SQS in transit; never written to application logs (Fluent Bit redaction)

---

## 4. Twilio — SMS Notifications

### 4.1 Purpose

Twilio delivers SMS notifications for Pro and Enterprise tier tenants. All notification types mirror the email templates.

### 4.2 Flow

```
notification-service checks tenant.plan:
    Starter  → email only (SendGrid)
    Pro/Enterprise → email + SMS (SendGrid + Twilio)

notification-service → Twilio API
    POST /2010-04-01/Accounts/{AccountSid}/Messages
    { To: customer_phone, From: +{twilio_number}, Body: "{message}" }
```

### 4.3 Failure Handling

Same retry and DLQ pattern as SendGrid. SMS failures are independent of email failures.

### 4.4 Security

- Twilio Account SID and Auth Token: `anji-schedulo/{env}/twilio/auth-token`; rotated every 90 days
- Twilio webhook signature validated via `X-Twilio-Signature` header
- Phone numbers (PII) only in `notifications` table (encrypted at rest) — never in application logs

---

## 5. AWS Native Service Integrations

### 5.1 SNS + SQS — Event Bus

| Integration | Pattern | Detail |
|---|---|---|
| Service → SNS | Transactional Outbox | `outbox-relay` reads `outbox_records`, publishes to SNS |
| SNS → SQS | Fan-out | Each consumer has its own SQS FIFO queue per subscribed topic |
| SQS → Service | Long-poll consumer | Max 20 messages per receive; 30s visibility timeout |
| SQS → DLQ | Automatic | `maxReceiveCount=4`; DLQ: `{source-queue-name}-dlq` |

All SNS topics and SQS queues encrypted with AWS KMS. Access restricted via resource-based IAM policies — only the designated consumer service IAM role can receive messages from its queue.

### 5.2 AWS Secrets Manager

| Secret | Consumer |
|---|---|
| `anji-schedulo/{env}/db/password` | All DB-connected services |
| `anji-schedulo/{env}/jwt/private-key` | `api-gateway` |
| `anji-schedulo/{env}/jwt/public-key` | All services that validate JWTs |
| `anji-schedulo/{env}/stripe/secret-key` | `payment-service` |
| `anji-schedulo/{env}/sendgrid/api-key` | `notification-service` |
| `anji-schedulo/{env}/twilio/auth-token` | `notification-service` |
| `anji-schedulo/{env}/redis/auth-token` | All Redis-connected services |

### 5.3 Amazon S3

| Bucket | Purpose | Access |
|---|---|---|
| `anji-schedulo-tenant-exports-{env}` | Tenant data export archives (FR012) | Private; presigned URL 24h expiry |
| `anji-schedulo-event-archives-{env}` | Monthly `appointment_events` export for DR | Private; ops-service only |
| `anji-schedulo-audit-logs-{env}` | Nightly `audit_log` export (WORM, permanent) | Private; write-only from export job |

### 5.4 AWS KMS

Separate CMKs per environment. Enterprise tenants have their own CMK — can rotate independently. All KMS keys are used for RDS, S3, SQS, SNS, and ElastiCache encryption.

---

## 6. Enterprise SSO (SAML 2.0 / OIDC)

Available on Enterprise plan. `api-gateway` acts as the SAML/OIDC Service Provider.

```
Enterprise user → tenant SSO page
    │  Redirect to IdP (Okta / Azure AD)
    ▼
IdP authenticates → returns SAML assertion or OIDC ID token
    ▼
api-gateway validates assertion → maps IdP role to AnjiSchedulo role
    │  Issues AnjiSchedulo JWT with tid = Enterprise tenant_id (from SSO config — not from IdP)
    ▼
Standard AnjiSchedulo JWT flow
```

**Security constraint:** `tid` is always set from the SSO configuration record — an IdP cannot issue a token for a different tenant.

---

## 7. Integration Health Monitoring

| Integration | Metric | Alert |
|---|---|---|
| Stripe | `payment_service_errors_total` | ALT001 (P1) if > 5% saga failures |
| SendGrid | `notification_delivery_failures_total{channel='email'}` | ALT008 (P3) if > 10% |
| Twilio | `notification_delivery_failures_total{channel='sms'}` | ALT008 (P3) if > 10% |
| SNS/SQS | `outbox_backlog_total` | ALT006 (P2) if > 100 |
| Secrets Manager | CloudTrail unexpected `GetSecretValue` | Security alert |

---

## 8. Traceability

| Integration | FR | BR | NFR |
|---|---|---|---|
| Stripe | FR004, FR005 | BR004, BR009 | — |
| SendGrid / Twilio | FR007 | BR007, BR013 | NFR002 |
| SNS/SQS | FR004–FR011 | BR013 | NFR003, NFR015 |
| Secrets Manager | — | — | NFR013 |
| S3 | FR012 | BR012, BR014 | — |
| Enterprise SSO | FR001 | BR005 | NFR012 |
