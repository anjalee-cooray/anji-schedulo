# HOT-001 · Hotfix Release

**Project:** AnjiSchedulo  
**Last Updated:** 2026-06-28

---

## 1. Purpose

A hotfix is a targeted, minimal code change that fixes a confirmed production defect without going through the normal release candidate cycle. Hotfixes always go through staging before production, even under time pressure.

---

## 2. When to Use a Hotfix

Use a hotfix when:
- A confirmed bug is causing a P1 or P2 incident in production
- A kill switch is not sufficient (the issue is a code defect, not a feature flag)
- The fix is small and well-understood

Use a full release instead of a hotfix when:
- The fix requires changes across 5+ files
- The fix requires a DB migration with significant schema changes
- The fix is speculative or exploratory

See [HOT-002 · Kill Switch vs. Hotfix](HOT-002-kill-switch-vs-hotfix.md) for the full decision tree.

---

## 3. Hotfix Flow

```
P1/P2 incident confirmed, root cause identified
    │
    ▼
Step 1: Create hotfix branch from main (NOT from a feature branch)
    git checkout main
    git pull origin main
    git checkout -b hotfix/fix-{description}
    │
    ▼
Step 2: Apply minimal fix
    - Change as few files as possible
    - Do not include unrelated cleanup or refactoring
    - Add a regression test that would have caught this bug
    │
    ▼
Step 3: Open PR targeting main
    Title: "fix: {description} [HOTFIX]"
    Description:
        - Link to incident
        - Root cause in 1–2 sentences
        - What changed and why it fixes the issue
        - Regression test added: yes/no
    Approval: Engineering Lead (1 reviewer sufficient for P1)
    │
    ▼
Step 4: CI runs on hotfix branch
    - All tests must pass
    - Cross-tenant isolation tests must pass
    - No blocking lint errors
    │
    ▼
Step 5: Merge to main
    Squash merge
    │
    ▼
Step 6: Deploy to staging (CD-002 — automatic on main push)
    - Confirm fix resolves the issue in staging
    - Run relevant smoke tests manually
    - 30-minute observation window in staging
    │
    ▼
Step 7: Deploy to production via hotfix pipeline (CD-003 manual trigger)
    - Engineering Lead triggers production deploy
    - Smoke test: booking creation + cancellation
    - 10-minute monitoring window
    │
    ▼
Step 8: Post-hotfix actions
    - Post to Slack #incidents: hotfix deployed, incident resolved
    - Tag the hotfix commit: git tag v{N}.{M}.{PATCH+1}
    - Update changelog (VER-002)
    - Open PIR tracking issue within 24 hours (O5)
```

---

## 4. Hotfix Version Numbering

Hotfixes increment the PATCH version:

| Scenario | Before | After |
|---|---|---|
| Hotfix on v1.0.0 | v1.0.0 | v1.0.1 |
| Second hotfix on v1.0.1 | v1.0.1 | v1.0.2 |

---

## 5. DB Migration in a Hotfix

If the hotfix requires a DB migration:
- The migration must be backward-compatible (old code works on new schema).
- The migration must be a new `V{N}__` Flyway script — never modify an existing migration.
- Add the migration to `services/tenant-service/migrations/`.
- Validate in staging before production deploy.

---

## 6. Traceability

| Concern | Doc |
|---|---|
| Kill switch vs. hotfix decision | HOT-002-kill-switch-vs-hotfix.md |
| Emergency change (non-code) | HOT-003-emergency-change.md |
| CD pipeline | CD-003-production-deploy.md |
| Incident response | O5-incident-response.md |
| PIR template | O5-incident-response.md#pir-template |
