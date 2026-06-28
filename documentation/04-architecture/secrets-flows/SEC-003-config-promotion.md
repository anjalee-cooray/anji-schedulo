# SEC-003 · Config Promotion Flow

**Project:** AnjiSchedulo  
**Version:** 1.0.0  
**Last Updated:** 2026-06-28

---

## 1. Overview

Config promotion describes how configuration values — not secrets — move from `dev` to `staging` to `production`. Configuration includes environment-specific settings like feature flags, rate limits, tenant plan quotas, and infrastructure endpoints. It is distinct from secrets (covered in SEC-001 and SEC-002), which are never promoted between environments.

---

## 2. Configuration Categories

| Category | Storage | Promotion Method |
|---|---|---|
| Infrastructure config (DB host, Redis host, SNS topic ARNs) | ECS task definition environment variables; set by Terraform per env | Terraform workspace per env — not promoted, derived from Terraform outputs |
| Application tuning (rate limits, booking limits, timeouts) | `config/{env}.json` in repo | Code change + PR review + standard deploy pipeline |
| Tenant plan limits (per-plan booking caps, staff limits) | `business-rules.json` spec → hard-coded in `tenant-service` | Code change requiring spec update + PR |
| Feature flags | Environment variable `FEATURE_{FLAG}=true` in ECS task definition | Terraform change per env |
| Observability config (alert thresholds, sampling rates) | Grafana provisioning YAML (`infra/grafana/`) | Terraform apply |

---

## 3. Infrastructure Config Promotion Flow

Infrastructure configuration values (DB endpoints, SNS ARNs, SQS URLs) are outputs of Terraform and injected automatically into ECS task definitions. They are never manually copied between environments.

```
infra/environments/dev/   → terraform apply → dev ECS task defs
infra/environments/staging/ → terraform apply → staging ECS task defs
infra/environments/production/ → terraform apply → prod ECS task defs

Each environment has its own:
  - RDS endpoint
  - ElastiCache endpoint
  - SNS topic ARNs
  - SQS queue URLs
  
These values are derived by Terraform from the resources it created —
they are not configurable by humans.
```

---

## 4. Application Config Promotion Flow

Application tuning values (timeouts, retry counts, rate limit overrides) live in the codebase and follow the standard deploy pipeline:

```
Engineer changes config/{env}.json or environment-specific value
          │
          ▼
PR opened → PR pipeline (CD-001)
    - Lint + type-check
    - Unit tests
    - Integration tests (verify config value doesn't break behaviour)
          │
          ▼
PR reviewed + merged to release/*
          │
          ▼
Staging deploy (CD-002)
    - New config applied to staging ECS tasks
    - Smoke tests + extended validation
          │
          ▼
Production deploy (CD-003)
    - Manual approval gate
    - Config applied to production ECS tasks
```

Config changes follow the same pipeline as code changes — they cannot bypass staging or the manual approval gate.

---

## 5. Feature Flag Promotion Flow

Feature flags control whether a feature is active in a given environment. They are environment variables in ECS task definitions, managed by Terraform.

```
Feature being tested in dev:
    infra/environments/dev/terraform.tfvars:
      feature_new_reschedule_flow = "true"
    → terraform apply → dev ECS tasks have FEATURE_NEW_RESCHEDULE_FLOW=true

Feature promoted to staging:
    infra/environments/staging/terraform.tfvars:
      feature_new_reschedule_flow = "true"
    → terraform apply → staging ECS tasks updated
    → QA validates feature in staging

Feature promoted to production:
    Engineer creates PR:
      infra/environments/production/terraform.tfvars:
        feature_new_reschedule_flow = "true"
    → PR review (same standards as code change)
    → Merge → CD-003 production deploy
    → Terraform applies → ECS tasks redeployed with flag enabled
```

**Flag rollback:** set flag to `"false"` and re-apply Terraform. ECS tasks redeploy with the flag disabled. No code change needed.

---

## 6. What Is Never Promoted Between Environments

| Item | Reason |
|---|---|
| Secret values (API keys, passwords, JWT keys) | Each env has its own credentials — never shared |
| Production tenant data | PII; must stay in production |
| Pre-signed S3 URLs | Environment-specific; expire quickly |
| Stripe live keys | Never used outside production |
| Cross-env Terraform state | Each env has isolated state |

If an engineer needs to test with production-like data, they use anonymised seed data in staging (DM3-seed-data-strategy.md). Production secrets never leave the production environment.

---

## 7. Drift Detection

To detect when an environment's configuration has drifted from the Terraform state:

```bash
# Check for drift in production environment
cd infra/environments/production
terraform plan

# If terraform plan shows unexpected changes:
#   - Review which resource changed outside Terraform
#   - Restore via terraform apply (bring back to desired state)
#   - Investigate who made the manual change via CloudTrail
```

A weekly CI job runs `terraform plan` on all environments and posts a diff to Slack #infra. Any non-empty plan triggers a review.

---

## 8. Tenant-Level Configuration

Tenant configuration (business hours, services, staff assignments, booking rules) is stored in the PostgreSQL `tenants` and `staff_members` tables, managed by the Tenant Admin via the product UI. It is not part of infrastructure configuration and is not promoted between environments — it exists independently in each environment's database.

Seed data for staging is generated from `DM3-seed-data-strategy.md` scripts, which create representative but synthetic tenant configurations.

---

## 9. Traceability

| Config Type | NFR | Doc |
|---|---|---|
| Infrastructure config via Terraform | NFR013 | INF-001-env-provisioning.md |
| Feature flags via ECS env vars | NFR001 | A9-deployment.md |
| Secrets never promoted | NFR013 | SEC-001-secrets-injection.md |
| Drift detection via terraform plan | NFR013 | INF-001-env-provisioning.md |
