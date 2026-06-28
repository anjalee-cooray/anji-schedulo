# FLAG-001 · Flag Creation

**Project:** AnjiSchedulo  
**Last Updated:** 2026-06-28

---

## 1. Purpose

This flow defines how to introduce a new feature flag into the AnjiSchedulo codebase. Flags are environment variables managed through Terraform. All new flags start as `false` in all environments and are promoted via code review (FLAG-002).

---

## 2. When to Create a Flag

Create a flag when:
- A feature is being built across multiple PRs and should not be accessible to end users until complete
- A feature is ready to ship but needs staged rollout (dev → staging → production)
- A feature needs a kill switch in case of production issues

Do NOT create a flag for:
- A bug fix (ship directly)
- Infrastructure changes (managed by Terraform variables, not feature flags)
- A feature that affects only operators/internal users with no tenant-visible impact

---

## 3. Flag Creation Flow

```
Engineer decides a feature flag is needed
    │
    ▼
Step 1: Choose flag name
    Convention: FEATURE_{DOMAIN}_{SHORT_DESCRIPTION}
    Example: FEATURE_BOOKING_RESCHEDULE
    │
    ▼
Step 2: Add flag to all environment tfvars (all environments = false to start)
    File: infra/environments/dev/terraform.tfvars
    File: infra/environments/staging/terraform.tfvars
    File: infra/environments/production/terraform.tfvars

    feature_booking_reschedule = "false"
    │
    ▼
Step 3: Add flag to ECS task definition module
    File: infra/modules/ecs-service/variables.tf — add variable
    File: infra/modules/ecs-service/main.tf — add to environment[] block

    { name = "FEATURE_BOOKING_RESCHEDULE", value = var.feature_booking_reschedule }
    │
    ▼
Step 4: Add flag guard in application code
    const rescheduleEnabled = process.env.FEATURE_BOOKING_RESCHEDULE === 'true';
    if (!rescheduleEnabled) throw new FeatureDisabledError('reschedule');
    │
    ▼
Step 5: Add flag to flag inventory table in O2-feature-flag-strategy.md
    │
    ▼
Step 6: Open PR — PR description must include:
    - Flag name
    - Default value in all envs
    - Which feature it guards
    - Planned enable date for dev
    │
    ▼
Step 7: PR review and merge to main
    CI deploys to dev with flag = false
```

---

## 4. Flag Guard Code Pattern

```typescript
// services/{service}/src/guards/feature.guard.ts

export function requireFeature(flag: string): void {
  const enabled = process.env[flag] === 'true';
  if (!enabled) {
    throw new FeatureDisabledError(flag);
  }
}

// Usage in controller or service:
requireFeature('FEATURE_BOOKING_RESCHEDULE');
```

---

## 5. Next Step

After the flag exists in all environments with default `false`, enable it in dev to begin development. See [FLAG-002 · Gradual Rollout](FLAG-002-gradual-rollout.md).

---

## 6. Traceability

| Concern | Doc |
|---|---|
| Flag strategy | O2-feature-flag-strategy.md |
| Gradual rollout | FLAG-002-gradual-rollout.md |
| Config promotion | SEC-003-config-promotion.md |
