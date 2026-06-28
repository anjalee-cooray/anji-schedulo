# CD-002 · Staging Deployment Flow

**Project:** AnjiSchedulo  
**Version:** 1.0.0  
**Last Updated:** 2026-06-28

---

## 1. Overview

The staging deployment runs automatically when a commit is pushed to a `release/*` branch. Staging is a production-equivalent environment used for pre-release validation, load testing, and final QA. It uses the same ECS Fargate infrastructure, RDS configuration, and secrets injection pattern as production — only the data and scale targets differ.

**Trigger:** Push to `release/*` branch  
**Environment:** `staging` (eu-west-1, separate VPC from production)  
**Duration:** ~20–25 minutes  
**Auto-deploy:** Yes — no manual approval gate (unlike production)

---

## 2. Pipeline Stages

```
Push to release/*
       │
       ▼
┌──────────────────────────────┐
│  All PR Pipeline stages      │
│  (lint, tests, build, scan)  │
│  run again against the       │
│  release branch HEAD         │
└──────────────┬───────────────┘
               │ all pass
               ▼
┌──────────────────────────────────────────┐
│  Stage 1: RDS Snapshot (pre-migration)   │
│  - aws rds create-db-snapshot            │
│    --identifier staging-pre-{sha}        │
│  - Wait for snapshot completion          │
│  - Snapshot retained indefinitely        │
└──────────────┬───────────────────────────┘
               │
               ▼
┌──────────────────────────────────────────┐
│  Stage 2: Database Migration             │
│  - ECS one-off task: flyway migrate      │
│  - Target: staging DB                    │
│  - Wait for task EXIT_CODE=0             │
│  - Pipeline halts on migration failure   │
│    (snapshot available for rollback)     │
└──────────────┬───────────────────────────┘
               │ success
               ▼
┌──────────────────────────────────────────┐
│  Stage 3: ECS Rolling Update             │
│  Deploy services in dependency order:    │
│    1. outbox-relay                       │
│    2. availability-service               │
│    3. payment-service                    │
│    4. booking-command-service            │
│    5. notification-service               │
│    6. dashboard-service                  │
│    7. analytics-service                  │
│    8. tenant-service                     │
│    9. ops-service                        │
│   10. api-gateway (last)                 │
│  Parameters:                             │
│    minHealthyPercent: 50%                │
│    maxPercent: 200%                      │
│    circuitBreaker: enabled               │
└──────────────┬───────────────────────────┘
               │
               ▼
┌──────────────────────────────────────────┐
│  Stage 4: Smoke Tests                    │
│  - POST /auth/token (api-gateway)        │
│  - GET /tenants/{slug}/availability      │
│  - GET /health (all 10 services)         │
│  - Assert all return 200                 │
└──────────────┬───────────────────────────┘
               │ pass
               ▼
┌──────────────────────────────────────────┐
│  Stage 5: Extended Validation            │
│  - End-to-end booking flow test          │
│    (create tenant → book → cancel)       │
│  - Cross-tenant isolation test           │
│  - Dashboard data reflects booking       │
│    within 5 seconds (NFR008)             │
│  - Notification delivered (mock SMTP)    │
└──────────────┬───────────────────────────┘
               │ pass
               ▼
  Staging deployment complete
  Notify: Slack #deployments + PR comment
```

---

## 3. Staging Environment Characteristics

| Characteristic | Staging | Production |
|---|---|---|
| ECS cluster | `anji-schedulo-staging` | `anji-schedulo-production` |
| RDS instance | `db.t4g.medium` (same as Starter/Pro) | `db.t4g.medium` (same) |
| ECS task count | Min 1 per service (vs. min 2 in prod) | Min 2 per service |
| Stripe | Stripe test mode (`sk_test_...`) | Stripe live mode |
| SendGrid | Sandbox mode (no real emails sent) | Live SendGrid |
| Twilio | Test credentials | Live Twilio |
| Data | Synthetic seed data; no real PII | Real tenant data |
| Manual approval | Not required | Required |

---

## 4. Rollback (Staging)

On smoke test or extended validation failure:

1. Pipeline automatically runs `ecs:UpdateService` to the previous task definition revision.
2. If migration caused the failure: restore from `staging-pre-{sha}` snapshot.
3. Pipeline notifies Slack #deployments with failure details.
4. Engineer investigates before re-deploying.

---

## 5. When Staging Deployment is a Gate for Production

A staging deployment is **required** before any production deployment:

- Production deploy pipeline checks that the same `{git-sha}` was successfully deployed to staging.
- If staging was skipped or failed, the production pipeline refuses to proceed.
- This is enforced via a GitHub Actions environment protection rule on the `production` environment.

---

## 6. Traceability

| Step | Purpose | NFR |
|---|---|---|
| Pre-migration RDS snapshot | Safe rollback point if migration fails | NFR001 |
| Flyway migration before service update | Zero-downtime schema changes | NFR007 |
| Dependency-ordered ECS rollout | Prevent version mismatch during deploy | NFR001 |
| End-to-end booking flow test | Validate critical path in staging | NFR004 |
| Cross-tenant isolation test | Catch isolation regressions pre-production | NFR012 |
