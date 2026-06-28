# HOT-002 · Kill Switch vs. Hotfix Decision

**Project:** AnjiSchedulo  
**Last Updated:** 2026-06-28

---

## 1. Purpose

When an incident occurs, the on-call engineer must quickly decide whether to use a kill switch (FLAG-003), initiate a full rollback (O3, CD-004), or develop a hotfix (HOT-001). This document provides the decision tree.

---

## 2. Decision Tree

```
Production incident confirmed
    │
    ▼
Q1: Is this incident related to a recent deploy (< 1 hour ago)?
    │
    ├── YES
    │       │
    │       ▼
    │   Q2: Is the incident caused by a specific feature that has a flag?
    │       │
    │       ├── YES → Kill switch (FLAG-003)
    │       │         Fastest option. Disable the flag, ECS redeploys.
    │       │         Use when: single feature broken, rest of system fine.
    │       │
    │       └── NO  → Full rollback (O3, CD-004)
    │                 Multiple services affected or root cause unclear.
    │                 Use when: cross-service issue, deploy-wide regression.
    │
    └── NO (incident is from existing code, not a new deploy)
            │
            ▼
        Q3: Can the issue be contained without a code change?
            │
            ├── YES → Config or infrastructure change (HOT-003)
            │         E.g. scale up ECS, increase PgBouncer pool size,
            │         update a Terraform variable.
            │
            └── NO → Hotfix (HOT-001)
                      Bug in existing code that requires a fix.
                      Go through: hotfix branch → staging → production.
```

---

## 3. Option Comparison

| Option | Time to apply | Code change | Staging required | Risk |
|---|---|---|---|---|
| Kill switch (FLAG-003) | 5–10 min | Terraform only | No (emergency PR) | Low — only disables a feature |
| Full rollback (CD-004) | 10–15 min | None — reverts ECS | No | Medium — reverts all changes |
| Hotfix (HOT-001) | 30–60 min | Yes | Yes | Medium — new code in production |
| Emergency config (HOT-003) | 5–15 min | Terraform/config | No | Low–Medium |

---

## 4. Cross-Tenant Isolation Violation (ALT005) — Special Case

If ALT005 fires (cross-tenant isolation violation):

```
IMMEDIATE ACTION — do not evaluate decision tree:
1. Scale booking-command-service and tenant-service to 0
2. Page Engineering Lead
3. Preserve all evidence (logs, traces)
4. Engineering Lead decides the remediation path
5. Full rollback or hotfix only — no kill switch (isolation is not flag-guarded)
```

---

## 5. Special Case: DB Migration Broke Schema

If a DB migration caused schema issues:
- **Do NOT rollback the migration** — Flyway down migrations are not used.
- Write a forward-fix migration (new `V{N+1}__fix_...sql`) immediately.
- Deploy via hotfix pipeline.

---

## 6. After Stabilisation

Regardless of which option was used:
1. Open PIR within 24 hours (O5).
2. Identify why the issue was not caught in staging.
3. Add tests to prevent regression.
4. If kill switch was used: plan the permanent hotfix and re-enable timeline.

---

## 7. Traceability

| Option | Doc |
|---|---|
| Kill switch | FLAG-003-kill-switch.md |
| Full rollback | O3-rollback-plan.md, CD-004-rollback.md |
| Hotfix | HOT-001-hotfix-release.md |
| Emergency config change | HOT-003-emergency-change.md |
| Incident response | O5-incident-response.md |
