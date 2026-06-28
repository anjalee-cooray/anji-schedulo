# CD-005 · Database Migration Deployment Flow

**Project:** AnjiSchedulo  
**Version:** 1.0.0  
**Last Updated:** 2026-06-28

---

## 1. Overview

Database schema changes are managed by **Flyway** with versioned, forward-only migration scripts. Migrations run as ECS one-off tasks before the application service rollout in every deployment pipeline. This document covers migration authoring rules, the deployment flow, and recovery procedures.

---

## 2. Migration Authoring Rules

All migrations must satisfy these rules before being merged:

| Rule | Requirement |
|---|---|
| **Forward-only** | No `DROP TABLE`, `DROP COLUMN`, or destructive `ALTER` without a prior data migration step. |
| **Zero-downtime compatible** | No table lock held for > 5 seconds. See section 3 for safe patterns. |
| **Idempotent** | Flyway checksums prevent re-running. Migrations must not assume partial state. |
| **Named correctly** | `V{version}__{description}.sql` — e.g. `V20260628001__add_pseudonymised_at_to_customers.sql` |
| **Tested in dev and staging** | Migration must run successfully in both environments before production deploy. |
| **RLS policy included** | Any new tenant-scoped table must include an RLS policy and an integration test. |

---

## 3. Safe Migration Patterns

### Adding a column

```sql
-- Safe: nullable column with no default — no table lock
ALTER TABLE appointments ADD COLUMN IF NOT EXISTS priority TEXT NULL;

-- Safe: column with a constant default (PostgreSQL 11+) — no table rewrite
ALTER TABLE appointments ADD COLUMN IF NOT EXISTS priority TEXT NOT NULL DEFAULT 'normal';
```

### Renaming a column (two-phase)

**Phase 1 (deploy 1):** Add the new column and dual-write to both columns in application code.

```sql
ALTER TABLE customers ADD COLUMN email_address TEXT NULL;
```

**Phase 2 (deploy 2, after all services are updated):** Backfill and drop the old column.

```sql
UPDATE customers SET email_address = email WHERE email_address IS NULL;
ALTER TABLE customers ALTER COLUMN email_address SET NOT NULL;
ALTER TABLE customers DROP COLUMN IF EXISTS email;
```

### Adding an index (non-blocking)

```sql
-- CONCURRENTLY avoids a full table lock
CREATE INDEX CONCURRENTLY IF NOT EXISTS idx_new_index
    ON appointments (tenant_id, slot_date);
```

### Adding a new table

```sql
CREATE TABLE IF NOT EXISTS new_entity (
    entity_id   UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id   UUID NOT NULL,
    -- ... columns ...
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Always include RLS for tenant-scoped tables
ALTER TABLE new_entity ENABLE ROW LEVEL SECURITY;

CREATE POLICY new_entity_isolation ON new_entity
    USING (tenant_id::TEXT = current_setting('app.tenant_id', true));

-- Required index
CREATE INDEX idx_new_entity_tenant ON new_entity (tenant_id);
```

---

## 4. Migration Deployment Flow

```
Migration file added to migrations/ in PR
          │
          ▼
PR Pipeline (CD-001)
  - Flyway validate (check migration file syntax and checksum)
  - Integration tests run against migrated dev schema
          │
          ▼
Staging Deploy (CD-002)
  ┌────────────────────────────────────────────┐
  │  Step 1: RDS snapshot (staging)            │
  │    aws rds create-db-snapshot              │
  │      --identifier staging-pre-{sha}        │
  └──────────────────────┬─────────────────────┘
                         │
  ┌────────────────────────────────────────────┐
  │  Step 2: Flyway migrate (ECS one-off task) │
  │    Image: flyway/flyway:10                 │
  │    Command: ["migrate"]                    │
  │    FLYWAY_URL: jdbc:postgresql://...       │
  │    FLYWAY_PASSWORD: from Secrets Manager   │
  │    Wait: ECS task EXIT_CODE=0              │
  └──────────────────────┬─────────────────────┘
                         │ success
  ┌──────────────────────────────────────────  ┐
  │  Step 3: ECS service rollout (CD-002)      │
  └──────────────────────┬─────────────────────┘
                         │
  ┌────────────────────────────────────────────┐
  │  Step 4: Extended validation               │
  │    - Smoke tests                           │
  │    - End-to-end booking flow               │
  │    - New schema feature tested             │
  └────────────────────────────────────────────┘
          │ staging confirmed
          ▼
Production Deploy (CD-003)
  ┌────────────────────────────────────────────┐
  │  Step 1: RDS snapshot (production)         │
  │    aws rds create-db-snapshot              │
  │      --identifier prod-pre-{sha}           │
  │    Retained indefinitely                   │
  └──────────────────────┬─────────────────────┘
                         │
  ┌────────────────────────────────────────────┐
  │  Step 2: Flyway migrate (production)       │
  │    Same ECS one-off task as staging        │
  │    Target: production DB                   │
  │    Notify Slack on start                   │
  │    Pipeline halts on EXIT_CODE != 0        │
  └──────────────────────┬─────────────────────┘
                         │ success
  ┌────────────────────────────────────────────┐
  │  Step 3: ECS service rollout (CD-003)      │
  └────────────────────────────────────────────┘
```

---

## 5. Flyway ECS Task Definition

```json
{
  "family": "flyway-migration-{env}",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "512",
  "memory": "1024",
  "executionRoleArn": "arn:aws:iam::...::role/EcsTaskExecutionRole",
  "containerDefinitions": [
    {
      "name": "flyway",
      "image": "flyway/flyway:10",
      "command": ["migrate"],
      "environment": [
        { "name": "FLYWAY_URL", "value": "jdbc:postgresql://{rds-endpoint}:5432/anjischedulo?ssl=true&sslmode=require" },
        { "name": "FLYWAY_USER", "value": "flyway_user" },
        { "name": "FLYWAY_LOCATIONS", "value": "filesystem:/flyway/sql" }
      ],
      "secrets": [
        {
          "name": "FLYWAY_PASSWORD",
          "valueFrom": "arn:aws:secretsmanager:{region}:{account}:secret:anji-schedulo/{env}/db/flyway-password"
        }
      ],
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/anji-schedulo/{env}/flyway",
          "awslogs-region": "eu-west-1",
          "awslogs-stream-prefix": "flyway"
        }
      }
    }
  ]
}
```

---

## 6. Migration Failure Recovery

| Failure Point | Recovery |
|---|---|
| Flyway `validate` fails in CI | Fix migration file in PR; do not push a bypass |
| Flyway `migrate` fails in staging | Investigate error in CloudWatch logs (`/anji-schedulo/staging/flyway`); fix forward; re-deploy |
| Flyway `migrate` fails in production | Pipeline halts; services not updated; restore from `prod-pre-{sha}` snapshot if schema is partially applied; write forward-fix migration |
| Migration applies but app incompatible | ECS rollback (CD-004); write forward-fix migration |
| Data corruption post-migration | PITR from `prod-pre-{sha}` snapshot (A12-disaster-recovery.md); replay events from S3 archive |

---

## 7. Migration Verification Checklist

Before merging a PR containing a migration file:

- [ ] Migration named correctly (`V{version}__{description}.sql`)
- [ ] No destructive operations without prior data migration
- [ ] No table lock > 5 seconds (`CONCURRENTLY` used for index creation)
- [ ] New tenant-scoped tables include RLS policy and index
- [ ] `flyway validate` passes in dev
- [ ] Integration tests cover the new schema
- [ ] `EXPLAIN ANALYZE` run on any new queries against the migrated schema
- [ ] Rollback plan documented in PR description (forward-fix migration or PITR window)
