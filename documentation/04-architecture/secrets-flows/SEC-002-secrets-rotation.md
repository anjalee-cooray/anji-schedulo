# SEC-002 · Secrets Rotation Flow

**Project:** AnjiSchedulo  
**Version:** 1.0.0  
**Last Updated:** 2026-06-28

---

## 1. Overview

All secrets have defined rotation periods. Some are automated (database passwords, Redis token), others are manual (third-party API keys, JWT signing keys). This document describes the rotation procedure for each secret type, including the dual-key rotation pattern for the JWT signing key (which requires zero-downtime handling).

---

## 2. Rotation Schedule

| Secret | Rotation Period | Method |
|---|---|---|
| PostgreSQL `booking_app_user` password | 90 days | Automated (Secrets Manager + Lambda rotator) |
| PostgreSQL `flyway_user` password | 90 days | Automated |
| Redis AUTH token | 90 days | Manual (see section 4) |
| JWT private key (RS256) | 180 days | Manual — dual-key rotation (see section 5) |
| Stripe secret key | 90 days | Manual (Stripe dashboard + Secrets Manager update) |
| Stripe webhook signing secret | On new webhook endpoint only | Manual |
| SendGrid API key | 180 days | Manual |
| Twilio Auth Token | 90 days | Manual |

---

## 3. Automated Rotation — PostgreSQL Password

Secrets Manager automated rotation uses a Lambda rotation function:

```
Day 0: Rotation scheduled
          │
          ▼
Secrets Manager triggers Lambda rotator:
    Stage 1 (createSecret):
        Generate new strong password
        Store as AWSPENDING version
    │
    Stage 2 (setSecret):
        ALTER USER booking_app_user PASSWORD '{new_password}';
        (on production RDS via admin credentials)
    │
    Stage 3 (testSecret):
        Connect to RDS with AWSPENDING password
        Run: SELECT 1
        Verify connection succeeds
    │
    Stage 4 (finishSecret):
        Promote AWSPENDING → AWSCURRENT
        Demote AWSCURRENT → AWSPREVIOUS
          │
          ▼
Running ECS tasks still use AWSPREVIOUS (old password via PgBouncer)
    → PgBouncer reconnects on next pool cycle using Secrets Manager SDK
    → Within 5 minutes: all pool connections use new password
    → No task restart needed (Secrets Manager SDK caches + refreshes)
```

**Connection continuity:** PgBouncer uses the Secrets Manager SDK's `GetSecretValue` on each new server connection. As old pool connections close naturally (idle timeout), new connections use the new password. No downtime.

---

## 4. Manual Rotation — Redis AUTH Token

Redis token rotation requires a brief connection interruption. Perform during a low-traffic window:

```
Step 1: Generate new AUTH token
    NEW_TOKEN=$(openssl rand -base64 32)

Step 2: Update Secrets Manager
    aws secretsmanager put-secret-value \
      --secret-id anji-schedulo/production/redis/auth-token \
      --secret-string "$NEW_TOKEN"

Step 3: Update ElastiCache cluster to accept BOTH old and new token
    (ElastiCache supports up to 2 AUTH tokens simultaneously during rotation)
    aws elasticache modify-replication-group \
      --replication-group-id anji-schedulo-production \
      --auth-token "$NEW_TOKEN" \
      --auth-token-update-strategy ROTATE

Step 4: Force new ECS task deployment (all Redis-connected services)
    for SVC in availability-service dashboard-service booking-command-service; do
      aws ecs update-service --cluster anji-schedulo-production \
        --service $SVC --force-new-deployment
    done
    New tasks pick up new token from Secrets Manager

Step 5: Remove old token from ElastiCache (after all old tasks drained)
    aws elasticache modify-replication-group \
      --replication-group-id anji-schedulo-production \
      --auth-token "$NEW_TOKEN" \
      --auth-token-update-strategy SET
```

---

## 5. Manual Rotation — JWT RS256 Signing Key (Zero-Downtime)

JWT key rotation requires care: in-flight tokens signed with the old key must continue to be valid during the transition. This uses a dual-key rotation approach.

```
Phase 1: Add new key alongside old key
──────────────────────────────────────
Step 1: Generate new RS256 key pair
    openssl genrsa -out new_private.pem 2048
    openssl rsa -in new_private.pem -pubout -out new_public.pem

Step 2: Add new public key to Secrets Manager
    aws secretsmanager create-secret \
      --name anji-schedulo/production/jwt/public-key-new \
      --secret-string "$(cat new_public.pem)"

Step 3: Update api-gateway to SIGN with new key but VERIFY with BOTH keys
    Update api-gateway task definition:
      JWT_PRIVATE_KEY   → new private key ARN
      JWT_PUBLIC_KEY    → new public key ARN
      JWT_PUBLIC_KEY_OLD → old public key ARN  (NEW env var)
    Deploy api-gateway (force new deployment)

    All new tokens are now signed with the new key.
    All services verify with new key (valid for new tokens).
    api-gateway falls back to old public key for tokens issued before rotation.

Phase 2: Expire old tokens (wait for access token TTL = 15 minutes)
──────────────────────────────────────────────────────────────────
    Wait 15 minutes: all tokens signed with the old key have expired.
    (Refresh tokens: 7 days — users will need to refresh, which
     issues a new access token signed with the new key.)

Phase 3: Remove old key (after 7 days for refresh token expiry)
────────────────────────────────────────────────────────────────
Step 4: Update api-gateway to remove fallback to old public key
    Remove JWT_PUBLIC_KEY_OLD from task definition
    Deploy api-gateway

Step 5: Update Secrets Manager
    Update jwt/public-key → new_public.pem
    Update jwt/private-key → new_private.pem
    Delete jwt/public-key-new (consolidated into jwt/public-key)

Step 6: Secure deletion of old key material
    Delete old private key PEM from all local machines
    Confirm old public key ARN is not referenced in any task definition
```

---

## 6. Manual Rotation — Stripe API Key

```
Step 1: Log in to Stripe Dashboard → Developers → API keys
    Create new Restricted key (not Standard key) with permissions:
      - charges: read
      - payment_intents: write
      - refunds: write

Step 2: Update Secrets Manager
    aws secretsmanager put-secret-value \
      --secret-id anji-schedulo/production/stripe/secret-key \
      --secret-string "rk_live_{new-key}"

Step 3: Force new deployment of payment-service
    aws ecs update-service --cluster anji-schedulo-production \
      --service payment-service --force-new-deployment

Step 4: Verify: make a test booking in production (smoke test)
    Confirm payment-service logs show successful Stripe API call

Step 5: Revoke old Stripe key in Stripe Dashboard
    (Only after Step 4 confirms new key works)
```

---

## 7. Rotation Tracking

Rotation dates are tracked in an internal ops log (outside source control):

| Secret | Last Rotated | Next Due | Owner |
|---|---|---|---|
| db/password | 2026-06-01 | 2026-09-01 | Automated |
| redis/auth-token | 2026-04-01 | 2026-07-01 | On-call |
| jwt/private-key | 2026-01-01 | 2026-07-01 | Engineering Lead |
| stripe/secret-key | 2026-04-01 | 2026-07-01 | Engineering Lead |
| sendgrid/api-key | 2026-01-01 | 2026-07-01 | On-call |
| twilio/auth-token | 2026-04-01 | 2026-07-01 | On-call |

Rotation reminders are set as recurring calendar events for the owner.

---

## 8. Traceability

| Rotation Control | NFR | Doc |
|---|---|---|
| JWT dual-key rotation | NFR013 | A4-security-model.md |
| Automated DB password rotation | NFR013 | SEC-001-secrets-injection.md |
| All keys in Secrets Manager | NFR013 | SEC-001-secrets-injection.md |
| Zero-downtime rotation requirement | NFR001 | A9-deployment.md |
