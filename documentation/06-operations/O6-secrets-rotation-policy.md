# O6 · Secrets Rotation Policy

**Project:** AnjiSchedulo  
**Version:** 1.0.0  
**Last Updated:** 2026-06-28

---

## 1. Overview

All AnjiSchedulo secrets have defined rotation schedules and procedures. Some rotations are fully automated (PostgreSQL passwords), others are manual (third-party API keys, JWT signing keys). This document defines the policy, schedule, and accountability for each secret type.

No secret is ever exempt from rotation. A secret that has not been rotated within its scheduled window must be treated as potentially compromised.

---

## 2. Rotation Schedule

| Secret | Rotation Period | Method | Owner |
|---|---|---|---|
| PostgreSQL `booking_app_user` password | 90 days | Automated (Secrets Manager Lambda) | — |
| PostgreSQL `flyway_user` password | 90 days | Automated (Secrets Manager Lambda) | — |
| Redis AUTH token | 90 days | Manual | On-call engineer |
| JWT RS256 private key | 180 days | Manual — dual-key rotation | Engineering Lead |
| Stripe secret key | 90 days | Manual | Engineering Lead |
| Stripe webhook signing secret | On webhook endpoint change only | Manual | Engineering Lead |
| SendGrid API key | 180 days | Manual | On-call engineer |
| Twilio Auth Token | 90 days | Manual | On-call engineer |

---

## 3. Rotation Procedures

Detailed step-by-step rotation procedures are in [SEC-002 · Secrets Rotation Flow](../04-architecture/secrets-flows/SEC-002-secrets-rotation.md).

Summary of key rotation concerns:

**PostgreSQL password (automated):** Secrets Manager Lambda rotates via four-stage process (createSecret → setSecret → testSecret → finishSecret). PgBouncer reconnects on next pool cycle — no task restart needed.

**JWT RS256 key (dual-key, zero-downtime):**
- Phase 1: Generate new key pair. `api-gateway` signs with new key but verifies with both old and new.
- Phase 2: Wait 15 minutes (access token TTL expires).
- Phase 3 (after 7 days — refresh token TTL): Remove old public key. Update Secrets Manager.

**Stripe key:** Create new restricted key in Stripe dashboard → update Secrets Manager → force ECS task redeploy → verify payment flow → revoke old key.

---

## 4. Rotation Tracking

| Secret | Last Rotated | Next Due | Status |
|---|---|---|---|
| db/password | Automated — check Secrets Manager console | 90-day cycle | Automated |
| db/flyway-password | Automated — check Secrets Manager console | 90-day cycle | Automated |
| redis/auth-token | 2026-04-01 | 2026-07-01 | Manual due |
| jwt/private-key | 2026-01-01 | 2026-07-01 | Manual due |
| stripe/secret-key | 2026-04-01 | 2026-07-01 | Manual due |
| sendgrid/api-key | 2026-01-01 | 2026-07-01 | Manual due |
| twilio/auth-token | 2026-04-01 | 2026-07-01 | Manual due |

Manual rotation reminders are set as recurring calendar events in the Engineering Lead's calendar.

---

## 5. Emergency Rotation (Compromise Response)

If a secret is suspected or confirmed to be compromised:

1. Rotate immediately — do not wait for the scheduled window.
2. Check CloudTrail for unexpected `secretsmanager:GetSecretValue` calls from unrecognised principals.
3. Check external provider logs (Stripe dashboard, SendGrid activity, Twilio logs) for unexpected API calls.
4. Rotate the secret using the standard procedure for that secret type.
5. Force new ECS task deployment to propagate the new value.
6. File a P1 security incident (O5) if there is evidence of unauthorised use.
7. Notify Platform Owner if PII or payment data may have been exposed.

---

## 6. Secret Storage Rules

| Rule | Detail |
|---|---|
| Storage location | AWS Secrets Manager only — never in `.env`, Dockerfiles, source code, or CI variables |
| Injection method | ECS task definition `secrets[]` array referencing ARN — never plaintext `environment[]` |
| Access control | Each service has its own IAM task role with access only to its own secrets |
| Transit protection | All `secretsmanager:GetSecretValue` calls route through VPC endpoint — no internet |
| Log redaction | Fluent Bit strips `*password*`, `*secret*key*`, `authorization` from all log streams |

---

## 7. Traceability

| Policy Area | Doc |
|---|---|
| Secret injection | SEC-001-secrets-injection.md |
| Rotation procedures | SEC-002-secrets-rotation.md |
| Threat model (secret exposure) | A5-threat-model.md (T-I03) |
| Log PII/secret redaction | OBS-001-log-aggregation.md |
| Emergency rotation → incident | O5-incident-response.md |
