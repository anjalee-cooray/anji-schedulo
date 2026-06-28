# SEC-001 · Secrets Injection Flow

**Project:** AnjiSchedulo  
**Version:** 1.0.0  
**Last Updated:** 2026-06-28

---

## 1. Overview

No secret ever appears in source code, `.env` files, Dockerfiles, CI/CD logs, or environment variables stored in the ECS task definition as plaintext. All secrets are stored in AWS Secrets Manager and injected into ECS containers at task startup via the ECS secrets mechanism. The injected values are never logged by the application or by Fluent Bit.

---

## 2. Secrets Inventory

| Secret Path | Contents | Consumers |
|---|---|---|
| `anji-schedulo/{env}/db/password` | PostgreSQL application user password | All DB-connected services |
| `anji-schedulo/{env}/db/flyway-password` | PostgreSQL Flyway migration user password | Flyway ECS task only |
| `anji-schedulo/{env}/jwt/private-key` | RS256 JWT signing private key (PEM) | `api-gateway` only |
| `anji-schedulo/{env}/jwt/public-key` | RS256 JWT verification public key (PEM) | All services that validate JWTs |
| `anji-schedulo/{env}/stripe/secret-key` | Stripe API secret key (`sk_live_...`) | `payment-service` only |
| `anji-schedulo/{env}/stripe/webhook-signing-secret` | Stripe webhook HMAC secret | `api-gateway` only |
| `anji-schedulo/{env}/sendgrid/api-key` | SendGrid API key | `notification-service` only |
| `anji-schedulo/{env}/twilio/auth-token` | Twilio Auth Token | `notification-service` only |
| `anji-schedulo/{env}/redis/auth-token` | ElastiCache Redis AUTH token | All Redis-connected services |

---

## 3. Secrets Injection Flow

```
Terraform provisions AWS Secrets Manager secret:
    aws_secretsmanager_secret:
      name: anji-schedulo/{env}/db/password
      kms_key_id: {env CMK ARN}
      recovery_window_in_days: 7
    (Initial value: random placeholder — rotated via SEC-002)
          │
          ▼
IAM policy attached to EcsTaskExecutionRole:
    Statement:
      Effect: Allow
      Action: secretsmanager:GetSecretValue
      Resource:
        - arn:aws:secretsmanager:{region}:{account}:secret:anji-schedulo/{env}/db/password
        - arn:aws:secretsmanager:{region}:{account}:secret:anji-schedulo/{env}/jwt/public-key
        # Only the specific secrets this service needs
          │
          ▼
ECS task definition references secret by ARN:
    "secrets": [
      {
        "name": "DB_PASSWORD",
        "valueFrom": "arn:aws:secretsmanager:eu-west-1:{account}:secret:anji-schedulo/{env}/db/password"
      }
    ]
          │
          ▼
ECS task starts (new deployment or new task):
    ECS Task Executor calls secretsmanager:GetSecretValue
    AWS Secrets Manager returns plaintext value
    Value is injected into container as environment variable DB_PASSWORD
    Value is NEVER written to ECS task definition in plaintext
    Value is NEVER logged (Fluent Bit redacts *password* field names)
          │
          ▼
NestJS service reads process.env.DB_PASSWORD at startup
    Connects to PostgreSQL via PgBouncer
    Secret value is in memory only — never written to disk or logs
```

---

## 4. IAM Least Privilege

Each service has its own IAM task role with access only to the secrets it needs:

| Service | Secrets Access |
|---|---|
| `api-gateway` | `jwt/private-key`, `jwt/public-key`, `stripe/webhook-signing-secret` |
| `booking-command-service` | `db/password`, `jwt/public-key` |
| `payment-service` | `db/password`, `jwt/public-key`, `stripe/secret-key` |
| `notification-service` | `db/password`, `jwt/public-key`, `sendgrid/api-key`, `twilio/auth-token` |
| `availability-service` | `db/password`, `jwt/public-key`, `redis/auth-token` |
| `dashboard-service` | `db/password`, `jwt/public-key`, `redis/auth-token` |
| `tenant-service` | `db/password`, `jwt/public-key` |
| `analytics-service` | `db/password`, `jwt/public-key` |
| `ops-service` | `db/password`, `jwt/public-key` |
| `outbox-relay` | `db/password` |
| `flyway` (migration task) | `db/flyway-password` |

No service has access to another service's secrets. Cross-service secret access is denied by IAM resource policy.

---

## 5. VPC Endpoint for Secrets Manager

All `secretsmanager:GetSecretValue` calls from ECS tasks are routed through a VPC Interface Endpoint:

```
ECS task → VPC endpoint → AWS Secrets Manager
         (stays within AWS network — no internet routing)
```

The VPC endpoint ensures that secret retrieval is not exposed to the public internet and does not route through the NAT Gateway.

---

## 6. Secret Value Updates

When a secret value changes (rotation, compromise response):

1. Operator updates the secret value in Secrets Manager via AWS CLI or console.
2. New secret value does NOT automatically reach running ECS tasks — tasks have the old value in memory.
3. To propagate the new value: trigger a new ECS deployment (`aws ecs update-service --force-new-deployment`).
4. New tasks start, pull the new secret value from Secrets Manager, and begin using it.
5. Old tasks drain and stop (30-second deregistration delay).

For the JWT key rotation case specifically, see SEC-002-secrets-rotation.md for the dual-key rotation procedure.

---

## 7. Secret Access Monitoring

CloudTrail logs all `secretsmanager:GetSecretValue` calls. A CloudWatch metric filter and alarm detects unexpected access:

```
Alert: Unexpected Secret Access
Condition: secretsmanager:GetSecretValue called by a principal
           that is not an ECS task execution role or approved CLI role
Action: Notify Slack #security immediately
```

This covers: compromised IAM credentials, developer accidentally using production credentials locally, or a misconfigured service accessing the wrong secrets.

---

## 8. Prohibited Patterns

The following patterns are prohibited and enforced via pre-commit hooks and CI linting:

| Prohibited | Why |
|---|---|
| `process.env.DB_PASSWORD = 'hardcoded'` in source code | Plaintext secret in repo |
| `.env` file committed to git | Exposes secrets via git history |
| `environment:` array in ECS task definition JSON with secret values | Appears in CloudTrail and ECS console plaintext |
| `console.log(process.env.DB_PASSWORD)` | PII/secret in logs |
| Secret values in Terraform `.tfvars` files | Appears in Terraform state |

Terraform uses `data.aws_secretsmanager_secret_version` to reference existing secrets — it does not create secrets with hardcoded values.

---

## 9. Traceability

| Security Control | NFR | BR | Doc |
|---|---|---|---|
| All secrets in Secrets Manager | NFR013 | — | A4-security-model.md |
| No plaintext in task definitions | NFR013 | — | A9-deployment.md |
| Least-privilege IAM per service | NFR013 | — | A4-security-model.md |
| VPC endpoint for Secrets Manager | NFR013 | — | INF-005-network-dns.md |
| Fluent Bit redacts secret field names | NFR013 | — | OBS-001-log-aggregation.md |
| CloudTrail monitoring of secret access | — | — | A5-threat-model.md (T-I03) |
