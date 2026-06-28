# RES-001 · Auto-Scaling Flow

**Project:** AnjiSchedulo  
**Version:** 1.0.0  
**Last Updated:** 2026-06-28

---

## 1. Overview

AnjiSchedulo scales application services horizontally via ECS Application Auto Scaling. Scaling decisions are based on CPU utilisation (all services) and SQS queue depth (consumer services). The database connection layer is decoupled from task count via PgBouncer — adding ECS tasks does not add DB connections proportionally.

---

## 2. Scale-Out Flow (CPU Trigger)

```
ECS tasks running under load
    │
    ▼
CloudWatch collects ECSServiceAverageCPUUtilization
    (1-minute period, sum of all tasks in service)
    │
    ▼ CPU > 60% for 1 consecutive period
Application Auto Scaling evaluates target tracking policy
    Target: 60% CPU utilisation
    Cooldown: 60s (scale-out), 300s (scale-in)
    │
    ▼
Auto Scaling calculates desired task count:
    desired = ceil(current_tasks × (current_cpu / target_cpu))
    Example: 2 tasks × (80% / 60%) = ceil(2.67) = 3 tasks
    │
    ▼ (respects MaxCapacity limit)
ECS UpdateService: desiredCount = 3
    │
    ▼
Fargate schedules new task:
    - Pull image from ECR (cached layer reuse)
    - Inject secrets from Secrets Manager
    - Start Fluent Bit sidecar
    - Health check: GET /health (30s grace period)
    │
    ▼ Task is RUNNING and healthy
ALB target group registers new task IP
    │
    ▼
Traffic distributed across 3 tasks
CloudWatch emits metric: DesiredTaskCount=3
```

---

## 3. Scale-Out Flow (SQS Queue Depth Trigger — Consumer Services)

```
SQS queue backlog grows (events arriving faster than consumed)
    │
    ▼
CloudWatch custom metric: ApproximateNumberOfMessagesVisible
    (per consumer queue)
    │
    ▼ Queue depth > threshold (per service, see table)
Application Auto Scaling evaluates target tracking policy
    │
    ▼
ECS UpdateService: desiredCount += N
    (same Fargate task start flow as CPU-triggered)
    │
    ▼
New consumer tasks start polling SQS
Queue depth begins falling
```

### Consumer Service Scaling Thresholds

| Service | Queue Depth Threshold | Min | Max |
|---|---|---|---|
| `notification-service` | 100 messages | 1 | 10 |
| `dashboard-service` | 200 messages | 1 | 5 |
| `analytics-service` | 500 messages | 1 | 5 |
| `outbox-relay` | outbox_backlog > 100 rows | 1 | 3 |

---

## 4. Scale-In Flow

```
Load decreases — CPU drops below target or queue drains
    │
    ▼
Scale-in cooldown period: 300 seconds
    (prevents oscillation — system waits before removing tasks)
    │
    ▼ CPU still below target after 300s
Auto Scaling calculates reduced desired count
    Minimum enforced: MinCapacity (prevents scaling to zero)
    │
    ▼
ECS initiates task draining:
    ALB deregisters task IP (deregistration delay: 30s)
    In-flight requests complete during deregistration period
    │
    ▼ Task has 0 connections or 30s passes
ECS stops drained task
    │
    ▼
CloudWatch emits metric: DesiredTaskCount reduced
```

---

## 5. Scaling Constraints

| Constraint | Value | Reason |
|---|---|---|
| Scale-out cooldown | 60 seconds | Fast response to load spikes |
| Scale-in cooldown | 300 seconds | Prevent thrashing |
| Minimum tasks per service | 1–2 (see INF-002) | HA — never scale to zero |
| Maximum tasks per service | 3–20 (see INF-002) | Cost cap + DB connection limit |
| DB connections (PgBouncer) | Max 200 per service | PgBouncer decouples from task count |

---

## 6. Database Connection Behaviour During Scale-Out

PgBouncer operates in transaction mode. Adding ECS tasks does not increase DB connections proportionally:

```
2 ECS tasks × 10 threads each → 20 application threads
  → PgBouncer pool: up to 200 connections to RDS
  
4 ECS tasks × 10 threads each → 40 application threads
  → PgBouncer pool: still up to 200 connections to RDS
     (PgBouncer multiplexes — application threads queue for a pool slot)
```

RDS is not the bottleneck during horizontal scaling. CPU and network are.

---

## 7. CloudFront and ALB Scaling

CloudFront and ALB scale automatically — no task-based Auto Scaling applies. AWS manages capacity:

- CloudFront: scales globally at edge with no operator action
- ALB: scales load balancer node count automatically based on traffic

---

## 8. Scaling Failure Modes and Responses

| Failure | Detection | Response |
|---|---|---|
| Tasks at MaxCapacity, CPU still critical | ALT001 fires (saga error rate) | Raise MaxCapacity via Terraform; investigate root cause |
| Scale-out blocked by ECR pull failure | ECS deployment event; task stays PENDING | Check ECR image tag; re-push if needed |
| PgBouncer connection saturation | RDS `connection_count` near max; query latency spikes | Increase `max_server_connections`; add PgBouncer instance |
| Scale-out doesn't reduce queue depth | ALT003 (DLQ growing); queue not draining | Check consumer DLQ; may be poison-pill message blocking queue |

---

## 9. Traceability

| Scaling Mechanism | NFR | BR | Doc |
|---|---|---|---|
| ECS CPU auto-scaling | NFR001, NFR010 | — | A8-scaling-strategy.md |
| SQS queue-depth scaling | NFR003 | — | A8-scaling-strategy.md |
| PgBouncer connection pooling | NFR001 | — | INF-003-managed-services.md |
| Per-tenant rate limits (prevent noisy-neighbour) | NFR011 | — | A8-scaling-strategy.md |
