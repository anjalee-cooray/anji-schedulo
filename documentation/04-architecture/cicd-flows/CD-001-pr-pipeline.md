# CD-001 · PR Pipeline Flow

**Project:** AnjiSchedulo  
**Version:** 1.0.0  
**Last Updated:** 2026-06-28

---

## 1. Overview

Every pull request targeting any branch triggers the PR pipeline. It validates code quality, runs all tests, builds Docker images, and deploys to the `dev` environment. The pipeline must pass before a PR can be merged.

**Trigger:** PR opened or updated against `feature/*`, `fix/*`, `release/*`, or `main`  
**Environment:** `dev`  
**Duration:** ~12–18 minutes

---

## 2. Pipeline Stages

```
PR Opened / Commit Pushed
         │
         ▼
┌─────────────────────────────┐
│  Stage 1: Code Quality      │
│  - ESLint (all packages)    │
│  - tsc --noEmit (strict)    │
│  - Prettier check           │
└─────────────┬───────────────┘
              │ pass
              ▼
┌─────────────────────────────┐
│  Stage 2: Unit Tests        │
│  - Jest (all services)      │
│  - Coverage threshold: 80%  │
└─────────────┬───────────────┘
              │ pass
              ▼
┌──────────────────────────────────────────┐
│  Stage 3: Integration Tests              │
│  Environment: Docker Compose             │
│    - PostgreSQL 15 (real DB, not mock)   │
│    - Redis 7                             │
│    - LocalStack (SNS, SQS)               │
│  Tests:                                  │
│    - Booking saga (slot lock, payment)   │
│    - Cross-tenant isolation (NFR012)     │
│    - Event envelope validation (BR013)   │
│    - Idempotency key deduplication       │
│    - Cancellation saga choreography      │
└─────────────┬────────────────────────────┘
              │ pass
              ▼
┌─────────────────────────────────────────┐
│  Stage 4: Build Docker Images           │
│  - Turborepo builds only changed        │
│    services (affected graph)            │
│  - Multi-stage esbuild build            │
│  - Tag: {git-sha}                       │
└─────────────┬───────────────────────────┘
              │ pass
              ▼
┌─────────────────────────────────────────┐
│  Stage 5: Security Scan                 │
│  - Amazon Inspector scan on ECR push    │
│  - P1 CVE → pipeline fails, PR blocked  │
│  - P2 CVE → warning, PR can merge       │
└─────────────┬───────────────────────────┘
              │ pass
              ▼
┌─────────────────────────────────────────┐
│  Stage 6: Push to ECR                   │
│  - Tag: {git-sha} (immutable)           │
│  - Repo: anji-schedulo/dev/{service}    │
└─────────────┬───────────────────────────┘
              │ pass
              ▼
┌─────────────────────────────────────────┐
│  Stage 7: Deploy to dev                 │
│  - Flyway migration (ECS one-off task)  │
│  - ECS rolling update (all services,    │
│    dependency order)                    │
│  - Smoke tests (GET /health each svc)   │
│  - Auto-rollback on smoke test failure  │
└─────────────┬───────────────────────────┘
              │ pass
              ▼
         PR Status: ✓ All checks passed
         Reviewer can approve and merge
```

---

## 3. Cross-Tenant Isolation Tests (Stage 3)

These tests are mandatory and run on every PR. They verify the three-layer isolation model (BR005, NFR012):

```typescript
// Example: tenant_A token cannot read tenant_B data
it('returns 404 when tenant_A reads tenant_B appointment', async () => {
  const tokenA = await getJwtToken({ tenant_id: tenantA.id, role: 'tenant_admin' });
  const response = await request(app)
    .get(`/appointments/${tenantBAppointment.id}`)
    .set('Authorization', `Bearer ${tokenA}`);
  expect(response.status).toBe(404);
});

// DB-level: null tenant context returns zero rows
it('returns zero rows when app.tenant_id is not set', async () => {
  const rows = await db.query('SELECT * FROM appointments');
  expect(rows.rowCount).toBe(0); // RLS safe default
});
```

---

## 4. GitHub Actions Workflow (Abbreviated)

```yaml
name: PR Pipeline
on:
  pull_request:
    types: [opened, synchronize, reopened]

jobs:
  quality:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: '20' }
      - run: npm ci
      - run: npx turbo lint type-check

  unit-tests:
    needs: quality
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm ci
      - run: npx turbo test -- --coverage

  integration-tests:
    needs: quality
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:15
        env: { POSTGRES_DB: anjischedulo_test, POSTGRES_PASSWORD: test }
      redis:
        image: redis:7-alpine
    steps:
      - uses: actions/checkout@v4
      - run: npm ci
      - run: npx turbo test:integration

  build-and-deploy-dev:
    needs: [unit-tests, integration-tests]
    runs-on: ubuntu-latest
    environment: dev
    steps:
      - uses: actions/checkout@v4
      - uses: aws-actions/configure-aws-credentials@v4
        with: { role-to-assume: ${{ secrets.AWS_DEV_ROLE_ARN }} }
      - uses: aws-actions/amazon-ecr-login@v2
      - run: npx turbo docker:build docker:push
        env: { IMAGE_TAG: ${{ github.sha }}, ENV: dev }
      - run: ./scripts/deploy.sh dev ${{ github.sha }}
```

---

## 5. Failure Behaviour

| Stage Failure | Effect |
|---|---|
| Lint / type-check fails | PR blocked; no further stages run |
| Unit tests fail | PR blocked |
| Integration tests fail (incl. isolation tests) | PR blocked |
| Docker build fails | PR blocked |
| P1 CVE in ECR scan | PR blocked |
| Dev deployment smoke test fails | PR flagged; dev env auto-rolled back; PR can still be reviewed |

---

## 6. PR Requirements Checklist

- [ ] All pipeline stages green
- [ ] At least 1 reviewer approval
- [ ] No merge conflicts with target branch
- [ ] If schema change: Flyway migration file included in PR
- [ ] If new event type: event envelope test added
- [ ] If new tenant-scoped table: RLS policy included and integration tested
