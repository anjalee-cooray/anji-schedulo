# FLAG-003 · Kill Switch

**Project:** AnjiSchedulo  
**Last Updated:** 2026-06-28

---

## 1. Purpose

A kill switch is an emergency flag disable in production. Use it when a recently enabled feature is causing errors or performance degradation and a hotfix is not yet available. A kill switch is faster than a full service rollback because it only redeploys the affected services.

---

## 2. When to Use a Kill Switch

Use FLAG-003 (kill switch) when:
- A specific feature is causing errors, but the rest of the deploy is functioning correctly
- You can identify the flag guarding the problematic feature
- The feature has been enabled in production for < 7 days

Use full rollback (O3, CD-004) instead when:
- Multiple services are affected
- The issue is not attributable to a specific feature flag
- You cannot determine the root cause quickly

See [HOT-002 · Kill Switch vs. Hotfix Decision](../hotfix-flows/HOT-002-kill-switch-vs-hotfix.md) for the full decision tree.

---

## 3. Kill Switch Flow

```
On-call engineer identifies a feature-flag-attributable incident
    │
    ▼
Confirm the correct flag name:
    cat infra/environments/production/terraform.tfvars | grep FEATURE
    │
    ▼
Set flag to false in production tfvars:
    File: infra/environments/production/terraform.tfvars
    feature_booking_reschedule = "false"   # was "true"
    │
    ▼
Open emergency PR (bypass normal review for P1):
    Title: "fix: kill switch — disable FEATURE_BOOKING_RESCHEDULE"
    Description: include incident link and reason
    Approval: Engineering Lead approves in Slack or GitHub
    │
    ▼
Merge PR → CD-003 triggers production deploy
    - Terraform apply updates ECS task definitions
    - ECS rolling redeploy (affected services only, ~5 min)
    │
    ▼
Verify kill switch is effective:
    - Check error rate drops in Grafana → Booking Operations
    - Confirm DLQ stops growing
    - Confirm no new P1/P2 alerts
    │
    ▼
Post to Slack #incidents:
    "🔴 Kill switch activated: FEATURE_{NAME} disabled in production
     Error rate: {before}% → {after}%
     Root cause investigation: ongoing"
    │
    ▼
Continue root cause investigation and plan hotfix (HOT-001)
```

---

## 4. Emergency PR Process

For a P1 kill switch, the normal 2-reviewer PR rule is suspended:
- Engineering Lead approval alone is sufficient
- PR must still be created (not a direct push to main)
- PR description must link the incident

---

## 5. After Kill Switch

After stabilisation:
1. Root cause investigation — why did the feature cause the issue?
2. Fix the code, add tests covering the failure case.
3. Re-validate in dev and staging (FLAG-002).
4. Re-enable in production with heightened monitoring.
5. Update O2 flag inventory with incident reference.

---

## 6. Traceability

| Concern | Doc |
|---|---|
| Kill switch vs. rollback decision | HOT-002-kill-switch-vs-hotfix.md |
| Full rollback | O3-rollback-plan.md, CD-004-rollback.md |
| Hotfix for permanent fix | HOT-001-hotfix-release.md |
| Flag re-enablement | FLAG-002-gradual-rollout.md |
