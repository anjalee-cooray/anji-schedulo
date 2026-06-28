# HOT-003 · Emergency Change

**Project:** AnjiSchedulo  
**Last Updated:** 2026-06-28

---

## 1. Purpose

An emergency change is a non-code infrastructure or configuration modification made during an active incident, outside of the normal release process. Examples: scaling ECS task count, updating a Terraform variable, re-routing DNS, modifying a security group rule.

Unlike a hotfix, an emergency change does not require a new Docker image build or Flyway migration. It is the fastest possible response when the issue is infrastructure-level rather than code-level.

---

## 2. Types of Emergency Changes

| Type | Example | Tool |
|---|---|---|
| ECS scaling | Increase min/max task count during traffic spike | AWS Console or CLI |
| PgBouncer pool size | Increase connection pool limit | Terraform apply |
| Feature flag disable | Disable a feature immediately | Terraform apply (FLAG-003) |
| DLQ requeue | Requeue failed messages | ops-service API |
| DNS re-routing | Point domain to previous ALB target group | Route 53 console |
| SG rule | Temporarily allow on-call engineer IP to RDS | AWS Console (emergency only) |
| Secrets rotation | Rotate a compromised secret | Secrets Manager + force deploy |

---

## 3. Emergency Change Flow

```
Incident requires an infrastructure change
    │
    ▼
Step 1: Confirm the change is safe
    - What exactly will change?
    - Can it be reverted in < 5 minutes if it makes things worse?
    - Is this change isolated to the affected service/component?
    │
    ▼
Step 2: Notify Slack #incidents
    "🔧 Applying emergency change: {description}
     Reason: {incident summary}
     Reverting if: {failure condition}"
    │
    ▼
Step 3: Apply the change
    Option A: AWS CLI (immediate, no pipeline)
        aws ecs update-service \
          --cluster anji-schedulo-production \
          --service {service-name} \
          --desired-count {N}

    Option B: Terraform (for config changes)
        cd infra/environments/production
        terraform apply -target=module.{resource}

    Option C: Route 53 / Security Group via AWS Console
        (last resort — document every manual change)
    │
    ▼
Step 4: Verify change is effective
    - Check relevant Grafana dashboard
    - Confirm error rate dropping, task count correct, etc.
    │
    ▼
Step 5: Post to Slack #incidents
    "✅ Emergency change applied: {description}
     Effect: {metric before} → {metric after}"
    │
    ▼
Step 6: Codify the change in Terraform
    - If the change was applied via console or CLI, create a follow-up PR
    - Update terraform.tfvars or Terraform module to reflect the new state
    - Merge before end of next business day (avoid config drift)
```

---

## 4. Manual Change Documentation

If a change is made outside of Terraform (console, CLI):

```markdown
## Emergency Change Log — {date}

Incident: {link}
Engineer: {name}
Time (UTC): {HH:MM}

Change made:
{Description of exactly what was changed, in which service/resource}

Reason:
{Why this change was needed}

Verified by:
{Metric or check that confirmed the change worked}

Follow-up PR to codify: {PR link or "TBD"}
```

Post this to the #incidents thread. If the change involved a security group or IAM modification, also notify Engineering Lead.

---

## 5. Changes That Are Never Emergency-Authorised

The following changes always require Engineering Lead sign-off, even during a P1 incident:

| Change | Why |
|---|---|
| Adding a new IAM policy | Security — must be reviewed |
| Disabling ALT005 (isolation violation alert) | Never suppressed — BR005 |
| Deleting RDS snapshots or S3 versioned objects | Irreversible data loss risk |
| Granting cross-tenant access to any service | Violates BR001, BR005 |
| Modifying KMS key policy | Could prevent decryption of all data |

---

## 6. Traceability

| Concern | Doc |
|---|---|
| Decision: kill switch vs. hotfix vs. emergency change | HOT-002-kill-switch-vs-hotfix.md |
| Incident response | O5-incident-response.md |
| Secrets rotation emergency | O6-secrets-rotation-policy.md |
| Rollback | O3-rollback-plan.md |
