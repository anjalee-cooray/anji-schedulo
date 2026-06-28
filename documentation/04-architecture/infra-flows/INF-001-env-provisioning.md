# INF-001 · Environment Provisioning Flow

**Project:** AnjiSchedulo  
**Version:** 1.0.0  
**Last Updated:** 2026-06-28

---

## 1. Overview

A new environment (`dev`, `staging`, or `production`) is provisioned entirely through Terraform. No resource is created manually in the AWS Console. This document describes the sequence in which infrastructure components are created and the dependencies between them.

**Trigger:** Engineer runs `terraform apply` from `infra/environments/{env}/` after a `terraform plan` review.  
**Environments:** `dev` · `staging` · `production`  
**AWS Region:** eu-west-1 (Ireland)

---

## 2. Provisioning Sequence

```
terraform apply (infra/environments/{env}/)
          │
          ▼
┌──────────────────────────────────────────┐
│  Phase 1: Foundation                     │
│  Order matters — all others depend here  │
│                                          │
│  1a. VPC                                 │
│      - CIDR: 10.{env}.0.0/16            │
│      - Public subnets (2 AZs): ALB      │
│      - Private subnets (2 AZs): ECS,    │
│        RDS, ElastiCache                  │
│      - No public IP on private subnets   │
│                                          │
│  1b. Internet Gateway + NAT Gateway      │
│      - 1 NAT per AZ (HA)                │
│      - Outbound-only for private subnets │
│                                          │
│  1c. KMS Customer Managed Keys           │
│      - 1 CMK per environment             │
│      - Enterprise: 1 additional CMK per  │
│        Enterprise tenant                 │
│      - Used by: RDS, S3, SQS, SNS,      │
│        ElastiCache, Secrets Manager      │
│                                          │
│  1d. IAM Roles                           │
│      - EcsTaskExecutionRole              │
│      - EcsTaskRole (per service)         │
│      - FlywayMigrationRole               │
│      - TerraformDeployRole               │
└──────────────────┬───────────────────────┘
                   │
                   ▼
┌──────────────────────────────────────────┐
│  Phase 2: Data Stores                    │
│  (depends on VPC + KMS)                  │
│                                          │
│  2a. RDS PostgreSQL 15                   │
│      - Multi-AZ enabled                  │
│      - Subnet group: private subnets     │
│      - Security group: port 5432 from    │
│        ECS private subnet only           │
│      - Encrypted with Phase 1 CMK        │
│      - Backup window: 03:00–04:00 UTC    │
│      - Retention: 7 days                 │
│      - Parameter group: custom           │
│        (pg_stat_statements, pgcrypto)    │
│                                          │
│  2b. ElastiCache Redis 7                 │
│      - Cluster mode disabled (single     │
│        primary + read replica)           │
│      - Auth token from Secrets Manager   │
│      - Encrypted in transit + at rest    │
│      - Subnet group: private subnets     │
│                                          │
│  2c. S3 Buckets (3)                      │
│      - tenant-exports-{env}              │
│      - event-archives-{env}              │
│      - audit-logs-{env} (Object Lock)    │
│      All: SSE-KMS, versioning, private   │
└──────────────────┬───────────────────────┘
                   │
                   ▼
┌──────────────────────────────────────────┐
│  Phase 3: Messaging                      │
│  (depends on KMS)                        │
│                                          │
│  3a. SNS Topics (1 per event type)       │
│      - SSE-KMS on all topics             │
│      - Resource policy: outbox-relay     │
│        IAM role allowed to publish       │
│                                          │
│  3b. SQS FIFO Queues + DLQs             │
│      - 1 queue per (consumer × topic)    │
│      - maxReceiveCount: 4                │
│      - DLQ: {queue-name}-dlq             │
│      - SSE-KMS, 14-day retention         │
│      - Subscribe to SNS topic            │
└──────────────────┬───────────────────────┘
                   │
                   ▼
┌──────────────────────────────────────────┐
│  Phase 4: Secrets                        │
│  (depends on KMS)                        │
│                                          │
│  4a. AWS Secrets Manager                 │
│      - anji-schedulo/{env}/db/password   │
│      - anji-schedulo/{env}/jwt/private-key│
│      - anji-schedulo/{env}/jwt/public-key│
│      - anji-schedulo/{env}/stripe/secret │
│      - anji-schedulo/{env}/sendgrid/key  │
│      - anji-schedulo/{env}/twilio/token  │
│      - anji-schedulo/{env}/redis/token   │
│      Initial values: Terraform generates │
│      random placeholders; operators      │
│      update via SEC-002 rotation flow    │
└──────────────────┬───────────────────────┘
                   │
                   ▼
┌──────────────────────────────────────────┐
│  Phase 5: Compute                        │
│  (depends on VPC, IAM, Secrets)          │
│                                          │
│  5a. ECR Repositories (1 per service)    │
│      - Lifecycle: keep last 10 images    │
│      - Scan on push (Amazon Inspector)   │
│                                          │
│  5b. ECS Cluster                         │
│      - Fargate launch type               │
│      - Container Insights enabled        │
│      - Cluster name: anji-schedulo-{env} │
│                                          │
│  5c. ECS Task Definitions (10 services)  │
│      - Secrets injected by ARN           │
│      - Fluent Bit sidecar (log routing)  │
│      - Resource limits per service       │
│                                          │
│  5d. ECS Services (10)                   │
│      - Attached to ALB target groups     │
│      - Auto-scaling policies             │
│      - Circuit breaker enabled           │
└──────────────────┬───────────────────────┘
                   │
                   ▼
┌──────────────────────────────────────────┐
│  Phase 6: Edge                           │
│  (depends on ECS Services)               │
│                                          │
│  6a. Application Load Balancer           │
│      - Public subnets                    │
│      - HTTPS listener (ACM certificate)  │
│      - HTTP → HTTPS redirect             │
│      - Target groups: 1 per ECS service  │
│                                          │
│  6b. CloudFront Distribution             │
│      - Origin: ALB                       │
│      - AWS WAF rate-limit rule attached  │
│      - HTTPS only, TLS 1.3              │
│      - Cache behaviour per path pattern  │
│                                          │
│  6c. Route 53 Records                    │
│      - api.anji-schedulo.com → ALB       │
│      - app.anji-schedulo.com → CF        │
└──────────────────────────────────────────┘
          │
          ▼
  Environment provisioning complete
  Run: flyway migrate (INF-003)
  Run: seed data (DM3)
```

---

## 3. Terraform State

| Environment | State Backend | Lock Table |
|---|---|---|
| `dev` | `s3://anji-schedulo-tf-state-dev` | DynamoDB `tf-state-lock-dev` |
| `staging` | `s3://anji-schedulo-tf-state-staging` | DynamoDB `tf-state-lock-staging` |
| `production` | `s3://anji-schedulo-tf-state-production` | DynamoDB `tf-state-lock-production` |

State bucket and DynamoDB table are bootstrapped separately (Terraform bootstrap in `infra/bootstrap/`). They are the only resources created manually.

---

## 4. Deprovisioning

To tear down an environment:

```bash
# Drain ECS services first (zero tasks)
for svc in api-gateway booking-command-service ...; do
  aws ecs update-service --cluster anji-schedulo-{env} --service $svc --desired-count 0
done

# Then destroy with Terraform
terraform destroy -var-file=environments/{env}/terraform.tfvars
```

Production must not be destroyed unless explicitly authorized by the platform owner. S3 Object Lock on the audit log bucket prevents deletion — the bucket must be manually drained through the Object Lock console before `terraform destroy` will succeed.

---

## 5. Traceability

| Phase | NFR | Architecture Doc |
|---|---|---|
| VPC / Network | NFR013 | INF-005-network-dns.md |
| KMS encryption | NFR013 | A4-security-model.md |
| RDS provisioning | NFR001, NFR007 | A7-infrastructure.md |
| Secrets Manager | NFR013 | SEC-001-secrets-injection.md |
| ECS Cluster + Services | NFR001, NFR010 | INF-002-cluster-setup.md |
| CloudFront + WAF | NFR011 | A8-scaling-strategy.md |
