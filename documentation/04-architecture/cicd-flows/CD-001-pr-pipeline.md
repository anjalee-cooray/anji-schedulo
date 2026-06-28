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
│  - Checkstyle (all modules) │
│  - SpotBugs (all modules)   │
│  - Gradle build (compile)   │
└─────────────┬───────────────┘
              │ pass
              ▼
┌─────────────────────────────┐
│  Stage 2: Unit Tests        │
│  - JUnit 5 + Mockito        │
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
│  - Gradle builds only changed           │
│    modules (incremental build)          │
│  - Multi-stage Gradle bootJar build     │
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

```java
// Example: tenant_A token cannot read tenant_B data
@Test
void returns404WhenTenantAReadsTenantBAppointment() throws Exception {
    String tokenA = getJwtToken(tenantA.getId(), "tenant_admin");
    mockMvc.perform(get("/appointments/{id}", tenantBAppointment.getId())
            .header("Authorization", "Bearer " + tokenA))
        .andExpect(status().isNotFound());
}

// DB-level: null tenant context returns zero rows
@Test
void returnsZeroRowsWhenTenantContextIsNotSet() {
    int count = jdbcTemplate.queryForObject(
        "SELECT COUNT(*) FROM appointments", Integer.class);
    assertThat(count).isEqualTo(0); // RLS safe default
}
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
      - uses: actions/setup-java@v4
        with: { java-version: '25', distribution: 'temurin' }
      - run: ./gradlew checkstyleMain spotbugsMain --no-daemon

  unit-tests:
    needs: quality
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with: { java-version: '25', distribution: 'temurin' }
      - run: ./gradlew test jacocoTestReport --no-daemon

  integration-tests:
    needs: quality
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with: { java-version: '25', distribution: 'temurin' }
      - run: ./gradlew integrationTest --no-daemon
      # Testcontainers starts PostgreSQL 15 and Redis 7 automatically

  build-and-deploy-dev:
    needs: [unit-tests, integration-tests]
    runs-on: ubuntu-latest
    environment: dev
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with: { java-version: '25', distribution: 'temurin' }
      - uses: aws-actions/configure-aws-credentials@v4
        with: { role-to-assume: ${{ secrets.AWS_DEV_ROLE_ARN }} }
      - uses: aws-actions/amazon-ecr-login@v2
      - run: ./gradlew bootJar dockerBuildPush --no-daemon
        env: { IMAGE_TAG: ${{ github.sha }}, ENV: dev }
      - run: ./scripts/deploy.sh dev ${{ github.sha }}
```

---

## 5. Failure Behaviour

| Stage Failure | Effect |
|---|---|
| Checkstyle / SpotBugs fails | PR blocked; no further stages run |
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
