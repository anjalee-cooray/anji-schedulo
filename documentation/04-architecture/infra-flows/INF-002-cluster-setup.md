# INF-002 · Cluster Setup Flow

**Project:** AnjiSchedulo  
**Version:** 1.0.0  
**Last Updated:** 2026-06-28

---

## 1. Overview

This document describes how the ECS Fargate cluster and its 10 application services are configured, deployed, and connected to supporting infrastructure. The cluster is created as part of the environment provisioning flow (INF-001 Phase 5), but this document covers the detailed configuration of each service.

---

## 2. ECS Cluster Configuration

```
ECS Cluster: anji-schedulo-{env}
  Launch type: FARGATE
  Container Insights: ENABLED
  CloudWatch log group: /anji-schedulo/{env}/ecs
```

No EC2 instances are managed. Fargate handles compute allocation per task.

---

## 3. Service Topology

```
Internet
   │ HTTPS (TLS 1.3)
   ▼
CloudFront → ALB
   │
   ├──► api-gateway             (public listener :443)
   │         │
   │         ├──► booking-command-service   (internal)
   │         ├──► availability-service      (internal)
   │         ├──► tenant-service            (internal)
   │         ├──► dashboard-service         (internal)
   │         ├──► ops-service               (internal)
   │         └──► payment-service           (internal)
   │
   ├──► notification-service    (SQS consumer only — no HTTP)
   ├──► analytics-service       (SQS consumer only — no HTTP)
   └──► outbox-relay            (PostgreSQL → SNS — no HTTP)
```

Services marked "internal" are not reachable from the internet directly. All inbound traffic enters through `api-gateway`.

---

## 4. Service Specifications

| Service | CPU | Memory | Min Tasks | Max Tasks | Inbound |
|---|---|---|---|---|---|
| `api-gateway` | 512 | 1024 MB | 2 | 20 | ALB :443 |
| `booking-command-service` | 1024 | 2048 MB | 2 | 10 | api-gateway internal |
| `availability-service` | 512 | 1024 MB | 2 | 10 | api-gateway internal |
| `payment-service` | 512 | 1024 MB | 1 | 5 | booking-command-service internal |
| `tenant-service` | 256 | 512 MB | 1 | 5 | api-gateway internal |
| `notification-service` | 256 | 512 MB | 1 | 10 | SQS only |
| `dashboard-service` | 256 | 512 MB | 1 | 5 | SQS + api-gateway internal |
| `analytics-service` | 256 | 512 MB | 1 | 5 | SQS only |
| `ops-service` | 256 | 512 MB | 1 | 3 | api-gateway internal (operator only) |
| `outbox-relay` | 256 | 512 MB | 1 | 3 | PostgreSQL outbox_records |

---

## 5. Task Definition Structure

Every service task definition follows the same pattern:

```json
{
  "family": "{service-name}-{env}",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "executionRoleArn": "arn:aws:iam::...::role/EcsTaskExecutionRole",
  "taskRoleArn": "arn:aws:iam::...::role/{service-name}-task-role-{env}",
  "containerDefinitions": [
    {
      "name": "{service-name}",
      "image": "{ecr-uri}/anji-schedulo/{env}/{service-name}:{image-tag}",
      "portMappings": [{ "containerPort": 3000 }],
      "healthCheck": {
        "command": ["CMD-SHELL", "curl -sf http://localhost:3000/health || exit 1"],
        "interval": 30,
        "timeout": 5,
        "retries": 3,
        "startPeriod": 30
      },
      "secrets": [
        { "name": "DB_PASSWORD", "valueFrom": "arn:aws:secretsmanager:...:secret:anji-schedulo/{env}/db/password" },
        { "name": "JWT_PUBLIC_KEY", "valueFrom": "arn:aws:secretsmanager:...:secret:anji-schedulo/{env}/jwt/public-key" }
      ],
      "environment": [
        { "name": "NODE_ENV", "value": "{env}" },
        { "name": "SERVICE_NAME", "value": "{service-name}" }
      ],
      "logConfiguration": {
        "logDriver": "awsfirelens",
        "options": {}
      }
    },
    {
      "name": "fluent-bit",
      "image": "public.ecr.aws/aws-observability/aws-for-fluent-bit:latest",
      "firelensConfiguration": { "type": "fluentbit" },
      "environment": [
        { "name": "LOKI_URL", "value": "http://loki.internal:3100/loki/api/v1/push" },
        { "name": "PII_REDACT_FIELDS", "value": "email,phone,name,recipient_contact,authorization" }
      ]
    }
  ]
}
```

---

## 6. ALB Target Group Configuration

For each HTTP-serving service:

| Parameter | Value |
|---|---|
| Target type | IP (Fargate) |
| Protocol | HTTP (TLS terminated at ALB) |
| Health check path | `/health` |
| Health check interval | 30 seconds |
| Healthy threshold | 2 |
| Unhealthy threshold | 3 |
| Deregistration delay | 30 seconds |

---

## 7. Service Auto-Scaling Policies

Each service has two scaling policies registered with ECS Application Auto Scaling:

**Policy 1 — CPU-based (all services):**
```
TargetTrackingScaling
  MetricType: ECSServiceAverageCPUUtilization
  TargetValue: 60%
  ScaleInCooldown: 300s
  ScaleOutCooldown: 60s
```

**Policy 2 — SQS queue depth (consumer services only):**
```
TargetTrackingScaling
  CustomMetric: SQS ApproximateNumberOfMessagesVisible
  TargetValue: {per-service threshold — see A8}
  ScaleInCooldown: 300s
  ScaleOutCooldown: 60s
```

---

## 8. Inter-Service Communication

All inter-service HTTP calls use the ECS Service Connect namespace (`anji-schedulo.internal`). Services discover each other by short name:

```
http://booking-command-service.anji-schedulo.internal:3000/internal/...
```

No public URLs are used for internal calls. Security groups permit traffic only between known service pairs.

---

## 9. Deployment Order

When deploying new task definitions, services must be updated in this order to prevent version mismatch:

1. `outbox-relay`
2. `availability-service`
3. `payment-service`
4. `booking-command-service`
5. `notification-service`
6. `dashboard-service`
7. `analytics-service`
8. `tenant-service`
9. `ops-service`
10. `api-gateway` (last — routes to all services)

---

## 10. Traceability

| Configuration | NFR | Doc |
|---|---|---|
| Task resource limits | NFR010, NFR011 | A8-scaling-strategy.md |
| Fluent Bit PII redaction | NFR013 | OBS-001-log-aggregation.md |
| Health check endpoint | NFR001 | A9-deployment.md |
| Secrets injection via ARN | NFR013 | SEC-001-secrets-injection.md |
| ALB + CloudFront TLS 1.3 | NFR013 | INF-005-network-dns.md |
