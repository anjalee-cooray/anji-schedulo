# FLAG-004 · Flag Cleanup

**Project:** AnjiSchedulo  
**Last Updated:** 2026-06-28

---

## 1. Purpose

Feature flags are temporary. A flag that has been `true` in production for > 30 days with no P1/P2 incidents is a candidate for cleanup. Cleaning up flags reduces codebase complexity, eliminates dead conditional branches, and removes the risk of accidentally disabling a production feature.

---

## 2. Eligibility Criteria

A flag is eligible for cleanup when ALL of the following are true:

| Criterion | How to verify |
|---|---|
| Flag has been `true` in production for > 30 days | Check git history of production terraform.tfvars |
| No P1/P2 incidents linked to this feature in the past 30 days | Check incident history in tracking issues |
| The feature has integration test coverage without the flag | Check test suite — tests should not reference the flag |
| Engineering Lead has confirmed feature is permanently live | Verbal or Slack confirmation |

---

## 3. Flag Cleanup Flow

```
Engineering Lead confirms flag is eligible for cleanup
    │
    ▼
Step 1: Remove flag guard from application code
    Before:
        const rescheduleEnabled = process.env.FEATURE_BOOKING_RESCHEDULE === 'true';
        if (!rescheduleEnabled) throw new FeatureDisabledError('reschedule');
        // ... feature code

    After:
        // ... feature code (always active, no guard)
    │
    ▼
Step 2: Remove flag from ECS task definition module
    File: infra/modules/ecs-service/variables.tf — remove variable declaration
    File: infra/modules/ecs-service/main.tf — remove from environment[] block
    │
    ▼
Step 3: Remove flag from all environment tfvars
    File: infra/environments/dev/terraform.tfvars
    File: infra/environments/staging/terraform.tfvars
    File: infra/environments/production/terraform.tfvars
    │
    ▼
Step 4: Remove flag from O2 flag inventory
    Mark as "Removed in v{N}" or delete the row
    │
    ▼
Step 5: Open PR
    Title: "chore: remove FEATURE_BOOKING_RESCHEDULE flag (feature permanently live)"
    Description: feature live since v{M}, no incidents, cleanup per FLAG-004
    │
    ▼
Step 6: Normal PR review and merge
    Standard 2-reviewer process — this is not an emergency change
```

---

## 4. Verifying No Remaining References

Before merging the cleanup PR:

```bash
# Confirm no remaining references to the flag in source code
grep -r "FEATURE_BOOKING_RESCHEDULE" services/ apps/
# Expected: 0 results

# Confirm no remaining references in Terraform
grep -r "feature_booking_reschedule" infra/
# Expected: 0 results
```

---

## 5. Cleanup Schedule

Engineering Lead reviews the O2 flag inventory monthly to identify cleanup candidates. The monthly review is a standing calendar reminder.

---

## 6. Traceability

| Concern | Doc |
|---|---|
| Flag strategy and inventory | O2-feature-flag-strategy.md |
| Flag creation | FLAG-001-flag-creation.md |
| Config promotion | SEC-003-config-promotion.md |
