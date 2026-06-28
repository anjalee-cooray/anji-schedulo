# REL-001 · Release Candidate Cut

**Project:** AnjiSchedulo  
**Last Updated:** 2026-06-28

---

## 1. Purpose

This flow describes how to cut a release candidate (RC) branch from `main`, prepare it for extended validation, and record the RC SHA. An RC branch isolates the release from ongoing feature development and is the only branch that gets deployed to staging for pre-release validation.

---

## 2. Prerequisites

Before cutting an RC, confirm:

- [ ] All planned feature PRs for this release are merged to `main`
- [ ] GitHub Actions CI is green on `main` (all tests, type checks, linting)
- [ ] No P1 or P2 incidents open on production
- [ ] Engineering Lead has confirmed the scope of the release

---

## 3. RC Cut Flow

```
main (all feature PRs merged, CI green)
    │
    ▼
Engineering Lead creates release branch:
    git checkout main
    git pull origin main
    git checkout -b release/v{MAJOR}.{MINOR}.{PATCH}
    git push origin release/v{MAJOR}.{MINOR}.{PATCH}
    │
    ▼
Record the RC SHA:
    git rev-parse HEAD
    # Post to Slack #deployments:
    # "🔖 RC cut: release/v{N} @ {sha} — entering validation"
    │
    ▼
GitHub Actions: rc-validation.yml triggers on push to release/* branch
    │
    ├── Run full test suite (unit + integration + isolation)
    ├── Run E2E smoke tests against a fresh staging environment
    ├── Run cross-tenant isolation test suite
    └── Build + push Docker images to ECR with tag: v{N}-rc.{date}
    │
    ▼
Deploy RC to staging (CD-002):
    - Update staging terraform.tfvars → image_tag = v{N}-rc.{date}
    - terraform apply (staging workspace)
    - Run Flyway migrations against staging DB
    - Confirm ECS tasks healthy
    │
    ▼
Post to Slack #deployments:
    "✅ RC deployed to staging — v{N}-rc.{date}
     Validation period: {RC_DATE} to {GO_NO_GO_DATE}
     Go/No-Go decision: {GO_NO_GO_DATE} at 09:00 UTC"
```

---

## 4. Version Numbering

Follow semantic versioning (SemVer):

| Change type | Example |
|---|---|
| MINOR — new features, backward-compatible | v1.0.0 → v1.1.0 |
| PATCH — bug fixes only | v1.0.0 → v1.0.1 |
| MAJOR — breaking changes (rare) | v1.0.0 → v2.0.0 |

RC tags use `v{N}-rc.{YYYYMMDD}` format (e.g. `v1.1.0-rc.20260628`).

---

## 5. Hotfixes to the RC Branch

If a critical bug is found on the RC branch during validation:

```bash
git checkout release/v{N}
git checkout -b hotfix/fix-{description}
# Apply fix
git commit -m "fix: {description}"
git push origin hotfix/fix-{description}
# Open PR into release/v{N} (not main)
```

After the hotfix merges into the RC branch, also merge back into `main`:

```bash
git checkout main
git merge release/v{N}
git push origin main
```

---

## 6. Next Step

After RC is deployed to staging and CI passes, proceed to [REL-002 · RC Validation](REL-002-rc-validation.md).

---

## 7. Traceability

| Concern | Doc |
|---|---|
| Staging CD pipeline | CD-002-staging-deploy.md |
| Go/No-Go decision | REL-003-go-no-go.md |
| Hotfix process | HOT-001-hotfix-release.md |
| Git branching strategy | DX3-git-workflow.md |
