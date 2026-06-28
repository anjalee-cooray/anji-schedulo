# INF-005 · Network and DNS Setup Flow

**Project:** AnjiSchedulo  
**Version:** 1.0.0  
**Last Updated:** 2026-06-28

---

## 1. Overview

This document describes the VPC topology, subnet design, security group rules, and DNS configuration for AnjiSchedulo. The network is designed so that no application data store or internal service is reachable from the internet — all external traffic enters through CloudFront and the ALB only.

---

## 2. VPC Design

| Environment | VPC CIDR |
|---|---|
| `dev` | 10.10.0.0/16 |
| `staging` | 10.20.0.0/16 |
| `production` | 10.30.0.0/16 |

### Subnet Layout (per environment, 2 AZs)

| Subnet | CIDR (AZ-a) | CIDR (AZ-b) | Purpose |
|---|---|---|---|
| Public subnet A | 10.{env}.1.0/24 | 10.{env}.2.0/24 | ALB, NAT Gateway |
| Private subnet A | 10.{env}.10.0/24 | 10.{env}.11.0/24 | ECS Fargate tasks |
| Private subnet B | 10.{env}.20.0/24 | 10.{env}.21.0/24 | RDS, ElastiCache |

- ECS tasks run in Private subnet A — no public IP, outbound via NAT Gateway
- RDS and ElastiCache run in Private subnet B — not reachable from ECS directly except via security group rules
- NAT Gateways in Public subnets — 1 per AZ for HA

---

## 3. Traffic Flow

```
Internet
   │ HTTPS :443
   ▼
AWS CloudFront (edge)
   │ WAF rate-limit rule
   │ TLS 1.3 termination
   │
   ├── Static assets → S3 or Amplify (direct)
   │
   └── API/app requests → ALB (origin)
          │ HTTP :80 (CloudFront → ALB, internal TLS)
          ▼
       AWS ALB (public subnets)
          │ Target group routing by path/host
          ▼
       ECS tasks (private subnets)
          │
          ├── PostgreSQL RDS (private subnet B) :5432
          ├── ElastiCache Redis (private subnet B) :6379
          └── SNS/SQS (VPC Endpoints — no internet routing)
                   │
                   └── Outbound only:
                       Stripe, SendGrid, Twilio → via NAT Gateway → Internet
```

---

## 4. Security Groups

### 4.1 ALB Security Group

| Direction | Protocol | Port | Source |
|---|---|---|---|
| Inbound | HTTPS | 443 | 0.0.0.0/0 (CloudFront only via WAF) |
| Inbound | HTTP | 80 | 0.0.0.0/0 (redirect to HTTPS) |
| Outbound | HTTP | 3000 | ECS Private Subnet A CIDR |

### 4.2 ECS Tasks Security Group

| Direction | Protocol | Port | Source |
|---|---|---|---|
| Inbound | HTTP | 3000 | ALB security group |
| Inbound | HTTP | 3000 | ECS tasks security group (inter-service) |
| Outbound | PostgreSQL | 5432 | RDS security group |
| Outbound | Redis | 6379 | ElastiCache security group |
| Outbound | HTTPS | 443 | 0.0.0.0/0 (via NAT — external APIs, SNS/SQS endpoints) |

### 4.3 RDS Security Group

| Direction | Protocol | Port | Source |
|---|---|---|---|
| Inbound | PostgreSQL | 5432 | ECS tasks security group |
| Outbound | None | — | — |

### 4.4 ElastiCache Security Group

| Direction | Protocol | Port | Source |
|---|---|---|---|
| Inbound | Redis | 6379 | ECS tasks security group |
| Outbound | None | — | — |

---

## 5. VPC Endpoints

To prevent SNS/SQS traffic from routing through the public internet, Interface VPC Endpoints are provisioned:

| Endpoint | Service | Type |
|---|---|---|
| `sns-endpoint` | `com.amazonaws.eu-west-1.sns` | Interface |
| `sqs-endpoint` | `com.amazonaws.eu-west-1.sqs` | Interface |
| `secretsmanager-endpoint` | `com.amazonaws.eu-west-1.secretsmanager` | Interface |
| `ecr-api-endpoint` | `com.amazonaws.eu-west-1.ecr.api` | Interface |
| `ecr-dkr-endpoint` | `com.amazonaws.eu-west-1.ecr.dkr` | Interface |
| `s3-endpoint` | `com.amazonaws.eu-west-1.s3` | Gateway |

All endpoints use the private subnets and the ECS tasks security group. Traffic to these services never leaves the AWS network.

---

## 6. DNS Configuration

### 6.1 Route 53 Hosted Zone

Public hosted zone: `anji-schedulo.com`

| Record | Type | Target |
|---|---|---|
| `api.anji-schedulo.com` | ALIAS | ALB DNS name |
| `app.anji-schedulo.com` | ALIAS | CloudFront distribution |
| `*.anji-schedulo.com` | ALIAS | CloudFront (tenant custom subdomains, Enterprise only) |

### 6.2 ACM Certificates

| Certificate | Domain | Used By |
|---|---|---|
| `api.anji-schedulo.com` | `api.anji-schedulo.com` | ALB HTTPS listener |
| `*.anji-schedulo.com` | `*.anji-schedulo.com`, `anji-schedulo.com` | CloudFront |

Certificates provisioned via AWS ACM in us-east-1 (required for CloudFront) and eu-west-1 (ALB).

### 6.3 Internal DNS (ECS Service Connect)

Internal service discovery uses ECS Service Connect namespace `anji-schedulo.internal`:

| Service | Internal DNS |
|---|---|
| `booking-command-service` | `booking-command-service.anji-schedulo.internal:3000` |
| `availability-service` | `availability-service.anji-schedulo.internal:3000` |
| `payment-service` | `payment-service.anji-schedulo.internal:3000` |
| `tenant-service` | `tenant-service.anji-schedulo.internal:3000` |
| `dashboard-service` | `dashboard-service.anji-schedulo.internal:3000` |
| `ops-service` | `ops-service.anji-schedulo.internal:3000` |

---

## 7. TLS Configuration

| Layer | Protocol | Enforced By |
|---|---|---|
| CloudFront → User | TLS 1.2 minimum, TLS 1.3 preferred | CloudFront security policy |
| CloudFront → ALB | HTTPS (TLS 1.3) | CloudFront origin protocol policy |
| ALB → ECS | HTTP (ALB handles TLS termination) | ALB listener |
| ECS → RDS | `sslmode=require` | PostgreSQL connection string |
| ECS → Redis | `transit_encryption_enabled: true` | ElastiCache config |
| ECS → SNS/SQS | HTTPS via VPC endpoint | AWS SDK default |
| ECS → Secrets Manager | HTTPS via VPC endpoint | AWS SDK default |

---

## 8. Traceability

| Network Component | NFR | BR | Doc |
|---|---|---|---|
| VPC private subnets | NFR013 | — | A7-infrastructure.md |
| CloudFront + WAF rate limit | NFR011 | — | A8-scaling-strategy.md |
| TLS 1.3 | NFR013 | — | A4-security-model.md |
| VPC endpoints (SNS/SQS/Secrets) | NFR013 | — | A10-integrations.md |
| ECS Service Connect | NFR001 | — | INF-002-cluster-setup.md |
| ACM certificates | NFR013 | — | A9-deployment.md |
