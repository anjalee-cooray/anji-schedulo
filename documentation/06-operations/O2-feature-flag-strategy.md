# O2 · Feature Flag Strategy

**Project:** AnjiSchedulo  
**Version:** 1.0.0  
**Last Updated:** 2026-06-28

---

## 1. Overview

AnjiSchedulo uses environment-variable feature flags managed through Terraform and ECS task definitions. Flags control whether a feature is active in a given environment. They are not runtime-togglable (no flag service) — enabling or disabling a flag requires a Terraform apply and ECS task redeployment. This is intentional: it keeps the flag system simple, auditable, and consistent with how all other configuration is managed.

---

## 2. Flag Naming Convention

All feature flags are environment variables prefixed with `FEATURE_`:

```
FEATURE_{DOMAIN}_{SHORT_DESCRIPTION}
```

Examples:
- `FEATURE_BOOKING_RESCHEDULE=true`
- `FEATURE_NOTIFICATION_WHATSAPP=true`
- `FEATURE_TENANT_SSO=true`
- `FEATURE_PAYMENT_SAVED_CARDS=false`

Flag names use SCREAMING_SNAKE_CASE. The domain segment matches the Spring Boot service that owns the feature.

---

## 3. Flag Lifecycle

```
FLAG-001: Flag creation (added to Terraform, deployed to dev)
    │
    ▼
FLAG-002: Gradual rollout (dev → staging → production)
    │
    ▼
Feature fully live in production (flag still present)
    │
    ▼
FLAG-004: Flag cleanup (flag removed from code and Terraform)
```

If a feature causes a P1 or P2 incident after production rollout:

```
FLAG-003: Kill switch (flag set to false, ECS redeployed)
```

---

## 4. Flag Storage and Injection

Flags are stored in `infra/environments/{env}/terraform.tfvars`:

```hcl
# infra/environments/production/terraform.tfvars
feature_booking_reschedule    = "false"   # v1.5.0 — not yet live
feature_notification_sms      = "true"    # v1.0.0 — live for Pro+
feature_tenant_sso            = "false"   # v1.5.0 — Enterprise only
```

Terraform passes these to ECS task definitions as environment variables:

```hcl
# infra/modules/ecs-service/main.tf
environment = [
  { name = "FEATURE_BOOKING_RESCHEDULE", value = var.feature_booking_reschedule },
  { name = "FEATURE_NOTIFICATION_SMS",   value = var.feature_notification_sms },
]
```

ECS tasks redeploy when the task definition changes. The new flag value takes effect on the next task start.

---

## 5. Flag Usage in Code

```java
// ✅ Simple boolean check — no flag service, no SDK
// In application.yml the env var is bound to a typed property:
// feature.notification.sms: ${FEATURE_NOTIFICATION_SMS:false}

@Value("${feature.notification.sms:false}")
private boolean smsEnabled;

if (smsEnabled && !tenant.plan().equals("starter")) {
    twilioClient.send(...);
}
```

Flags are injected once at bean construction (`@Value` fields) — not read on every request.

---

## 6. Flag Inventory

| Flag | Default (prod) | Introduced | Purpose |
|---|---|---|---|
| `FEATURE_NOTIFICATION_SMS` | `true` | v1.0.0 | Enable Twilio SMS for Pro+ tenants |
| `FEATURE_BOOKING_RESCHEDULE` | `false` | v1.5.0 | Enable reschedule flow |
| `FEATURE_TENANT_SSO` | `false` | v1.5.0 | Enable SAML/OIDC for Enterprise |
| `FEATURE_NOTIFICATION_WHATSAPP` | `false` | v1.5.0 | Enable WhatsApp for Enterprise |
| `FEATURE_PAYMENT_SAVED_CARDS` | `false` | v2.0.0 | Enable Stripe saved payment methods |

Flags with default `false` in production are not yet released. Flags with `true` are live and candidates for cleanup once stable.

---

## 7. Flag Retirement Policy

A flag is eligible for cleanup (FLAG-004) when:
- It has been `true` in production for > 30 days with no P1/P2 incidents linked to it
- The feature is confirmed stable by the Engineering Lead
- The code path guarded by the flag has integration test coverage without the flag

---

## 8. Traceability

| Concern | Doc |
|---|---|
| Flag creation | FLAG-001-flag-creation.md |
| Gradual rollout | FLAG-002-gradual-rollout.md |
| Kill switch | FLAG-003-kill-switch.md |
| Flag cleanup | FLAG-004-flag-cleanup.md |
| Config promotion between envs | SEC-003-config-promotion.md |
