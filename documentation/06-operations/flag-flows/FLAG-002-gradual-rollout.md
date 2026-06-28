# FLAG-002 · Gradual Rollout

**Project:** AnjiSchedulo  
**Last Updated:** 2026-06-28

---

## 1. Purpose

This flow defines how to promote a feature flag from `false` (hidden) to `true` (live) through dev → staging → production. Each promotion requires a Terraform apply and is gated on stability observation in the previous environment.

---

## 2. Promotion Flow

```
Flag created at false in all environments (FLAG-001)
    │
    ▼
Phase 1: Enable in dev
    - Update infra/environments/dev/terraform.tfvars:
        feature_booking_reschedule = "true"
    - Open PR: "feat: enable FEATURE_BOOKING_RESCHEDULE in dev"
    - CI applies Terraform to dev, ECS tasks redeploy
    - Engineer validates feature works correctly in dev
    - Observe for 24 hours: no errors in dev Loki logs
    │
    ▼
Phase 2: Enable in staging
    - Update infra/environments/staging/terraform.tfvars:
        feature_booking_reschedule = "true"
    - Open PR
    - CI applies Terraform to staging, ECS tasks redeploy
    - Validate feature in staging (manual smoke test)
    - Observe for 48 hours:
        * No errors in Loki
        * DLQ depth = 0
        * No saga error rate increase
    │
    ▼
Phase 3: Enable in production
    - Update infra/environments/production/terraform.tfvars:
        feature_booking_reschedule = "true"
    - Open PR with sign-off from Engineering Lead
    - Merge triggers CD-003 (or next scheduled release)
    - Post-enable monitoring: 10 minutes active monitoring
    - 48-hour stabilisation period begins
```

---

## 3. Observation Criteria per Environment

| Environment | Minimum observation | Pass criteria |
|---|---|---|
| dev | 24 hours | No flag-related errors in Loki |
| staging | 48 hours | No errors, DLQ = 0, no saga regression |
| production | 48 hours (stabilisation) | Same as staging, plus ALT001/ALT002 stable |

---

## 4. Abort Criteria

Abort the rollout and set the flag back to `false` if:
- Error rate increases by > 1% after flag enable
- DLQ receives messages attributable to the feature
- Any P1 or P2 alert fires within 2 hours of flag enable in staging or production

Use [FLAG-003 · Kill Switch](FLAG-003-kill-switch.md) to disable in production quickly.

---

## 5. Tenant-Scoped Flags (Future)

The current implementation enables/disables a flag globally per environment. If future requirements call for per-tenant or per-plan flag enablement (e.g. "enable for Enterprise tenants only"), this is enforced in application code:

```java
@Value("${feature.tenant.sso:false}")
private boolean ssoEnabled;

if (ssoEnabled && tenant.plan().equals("enterprise")) {
    // feature active
}
```

No separate flag service is required — plan-scoped access is a business rule, not a flag concern.

---

## 6. Traceability

| Concern | Doc |
|---|---|
| Flag creation | FLAG-001-flag-creation.md |
| Kill switch | FLAG-003-kill-switch.md |
| Config promotion | SEC-003-config-promotion.md |
| Flag cleanup | FLAG-004-flag-cleanup.md |
