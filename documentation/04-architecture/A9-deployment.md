# A9 · Deployment

**Project:** AnjiSchedulo  
**Version:** 1.0.0  
**Owner:** Pubudu Anjalee Cooray  
**Last Updated:** 2026-06-28

---

## 1. Overview

AnjiSchedulo deploys containerised NestJS services to AWS ECS Fargate via a GitHub Actions CI/CD pipeline. Three environments exist: `dev`, `staging`, and `production`. All deployments are zero-downtime rolling updates. Database migrations run as ECS one-off tasks before service rollout. Rollback to the previous task definition is automatic on smoke test failure.

---

## 2. Environments

| Environment | Branch | Purpose | Auto-Deploy |
|---|---|---|---|
| `dev` | `feature/*` (on PR open) | Developer integration testing | Yes |
| `staging` | `release/*` | Pre-production validation | Yes |
| `production` | `main` (on merge) | Live platform | Yes, with manual approval gate |

All environments deploy to **AWS eu-west-1 (Ireland)**.

---

## 3. Infrastructure as Code

All infrastructure is defined in Terraform (`infra/` monorepo package):

```
infra/
  modules/
    ecs-service/          # Reusable ECS Fargate module
    rds-cluster/          # PostgreSQL RDS + PgBouncer sidecar
    sqs-pair/             # SQS FIFO queue + DLQ pair
    sns-topic/            # SNS topic with KMS encryption
  environments/
    dev/ staging/ production/
  global/                 # ECR repositories, KMS keys, IAM roles
```

Terraform state: S3 (`anji-schedulo-tf-state-{env}`) with DynamoDB state lock.  
`infra/` changes trigger `terraform plan` in CI; `terraform apply` requires manual approval for staging and production.

---

## 4. Container Build

```dockerfile
# Build stage
FROM node:20-alpine AS build
WORKDIR /app
COPY package*.json turbo.json ./
RUN npm ci
COPY . .
RUN npx turbo build --filter=<service-name>

# Production — distroless
FROM gcr.io/distroless/nodejs20-debian12
WORKDIR /app
COPY --from=build /app/apps/<service-name>/dist ./dist
COPY --from=build /app/node_modules ./node_modules
USER nonroot
EXPOSE 3000
CMD ["dist/main.js"]
```

- Non-root user, distroless base — minimal attack surface
- Built with esbuild via Turborepo — fast incremental builds
- Images pushed to **Amazon ECR**: `anji-schedulo/{env}/{service-name}:{git-sha}`
- ECR lifecycle: retain last 10 images; delete untagged after 7 days
- Amazon Inspector scans on push — P1 vulnerabilities block deployment

---

## 5. Deployment Pipeline

```
PR opened (feature/*)
├── 1. Lint + type-check (tsc --noEmit, ESLint)
├── 2. Unit tests (Jest)
├── 3. Integration tests (Docker Compose: PostgreSQL + Redis + LocalStack)
│       ├── Cross-tenant isolation tests (NFR012)
│       ├── Booking saga tests
│       └── Event envelope validation tests
├── 4. Build Docker images (multi-stage, esbuild)
├── 5. Push to ECR (dev tag: {git-sha})
└── 6. Deploy to dev

Merge to release/* (staging)
├── Steps 1–5 above
├── 7. Run Flyway migrations (ECS one-off task, staging DB)
├── 8. Deploy ECS services — rolling update
├── 9. Smoke tests (GET /health on each service)
└── 10. Auto-rollback on smoke test failure

Merge to main (production)
├── Steps 1–5 above
├── 11. Manual approval gate (GitHub Actions environment protection rule)
├── 12. RDS manual snapshot (taken before migration)
├── 13. Run Flyway migrations (ECS one-off task, production DB)
├── 14. Deploy ECS services — rolling update (dependency order)
├── 15. Smoke tests
└── 16. Auto-rollback on smoke test failure
```

### ECS Rolling Update Parameters

| Parameter | Value |
|---|---|
| `minimumHealthyPercent` | 50% |
| `maximumPercent` | 200% |
| Deployment circuit breaker | Enabled — auto-rollback if task fails to reach RUNNING |
| Health check grace period | 30 seconds |

### Service Deployment Order (Production)

1. `outbox-relay`
2. `availability-service`
3. `payment-service`
4. `booking-command-service`
5. `notification-service`
6. `dashboard-service`
7. `analytics-service`
8. `tenant-service`
9. `ops-service`
10. `api-gateway` (last — routes to all services)

---

## 6. Database Migrations

Migrations managed by **Flyway** as ECS one-off tasks before service rollout.

**Rules:**
- Forward-only: no `DROP COLUMN` or `DROP TABLE` without a prior data migration step
- Zero-downtime: column additions are nullable or have a default; renames done in two-phase (add + dual-write, then remove)
- Idempotent: Flyway checksums prevent re-running applied migrations
- Manual RDS snapshot taken in pipeline before every production migration

```yaml
# ECS one-off migration task
family: flyway-migration-{env}
containerDefinitions:
  - name: flyway
    image: flyway/flyway:10
    command: ["migrate"]
    environment:
      - FLYWAY_URL: jdbc:postgresql://{rds-endpoint}:5432/anjischedulo
    secrets:
      - name: FLYWAY_PASSWORD
        valueFrom: arn:aws:secretsmanager:{region}:{account}:secret:anji-schedulo/{env}/db/password
```

Migration failure halts the pipeline and alerts the engineering team. Service rollout does not proceed.

---

## 7. Secrets Injection

Secrets are never in code, `.env` files, or Docker images. ECS task definitions reference Secrets Manager ARNs:

```json
{
  "secrets": [
    {
      "name": "DB_PASSWORD",
      "valueFrom": "arn:aws:secretsmanager:eu-west-1:123456789:secret:anji-schedulo/production/db/password"
    },
    {
      "name": "JWT_PRIVATE_KEY",
      "valueFrom": "arn:aws:secretsmanager:eu-west-1:123456789:secret:anji-schedulo/production/jwt/private-key"
    }
  ]
}
```

The IAM task execution role grants `secretsmanager:GetSecretValue` only on the specific ARNs for that service (least-privilege).

---

## 8. Health Checks and Smoke Tests

Every service exposes `GET /health` (no auth):

```json
{
  "status": "ok",
  "service": "booking-command-service",
  "checks": { "database": "ok", "redis": "ok", "sqs": "ok" }
}
```

ALB target group: `GET /health` must return 200 within 5 seconds. Unhealthy after 3 consecutive failures. Task replaced automatically.

Post-deployment smoke tests validate each service before marking the deploy successful. Failure triggers automatic rollback.

---

## 9. Rollback

**Automatic:** Smoke test failure → pipeline calls `ecs:UpdateService` with the previous task definition revision → ECS rolling update reverts.

**Manual (post-smoke-test issue):**
```bash
aws ecs update-service \
  --cluster anji-schedulo-production \
  --service booking-command-service \
  --task-definition booking-command-service:<previous-revision>
```

Database rollback: migrations are forward-only. Data corruption → PITR (point-in-time recovery) from RDS. See A12-disaster-recovery.md.

---

## 10. Production Deploy Checklist

| Step | Owner | Method |
|---|---|---|
| All integration tests pass | CI | Automated |
| ECR scan: no P1 vulnerabilities | CI | Automated (Inspector) |
| Manual approval granted | Engineering Lead | GitHub Actions environment rule |
| RDS snapshot taken before migration | CI | Automated (pipeline step) |
| Flyway migration completes | CI | Automated |
| Smoke tests pass | CI | Automated |
| Grafana SLO dashboards stable (10 min) | On-call engineer | Manual observation |

---

## 11. Traceability

| Concern | NFR | Mechanism |
|---|---|---|
| Zero-downtime | NFR001 | ECS rolling update (50%/200%) |
| Migration safety | NFR007 | Forward-only Flyway + PITR backup |
| Secrets security | NFR013 | Secrets Manager ARN injection |
| Post-deploy validation | NFR001 | Smoke tests + Grafana |
| Rollback capability | NFR001 | Previous task definition revision |
