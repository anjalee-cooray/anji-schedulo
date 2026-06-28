# OBS-001 · Log Aggregation Flow

**Project:** AnjiSchedulo  
**Version:** 1.0.0  
**Last Updated:** 2026-06-28

---

## 1. Overview

Every ECS task produces structured JSON logs to stdout. A Fluent Bit sidecar container on each task intercepts these logs, applies PII redaction, enriches them with ECS metadata, and forwards them to Grafana Loki. No log data leaves the VPC unencrypted. PII is stripped before leaving the application tier.

---

## 2. Log Pipeline

```
NestJS application
    │ stdout: structured JSON (pino logger)
    │ Format: {"timestamp":"...","level":"info","service":"booking-command-service",
    │          "tenant_id":"...","correlation_id":"...","trace_id":"...","message":"..."}
    ▼
Fluent Bit (ECS sidecar, FireLens integration)
    │
    ├── FILTER: PII Redaction
    │     Fields redacted (set to "[REDACTED]"):
    │       email, phone, name (standalone), recipient_contact,
    │       authorization, *password*, *secret*key*
    │
    ├── FILTER: ECS Metadata Enrichment
    │     Appended to every log line:
    │       ecs_cluster, ecs_task_id, ecs_service_name,
    │       aws_region, environment
    │
    ├── FILTER: Log Level Routing
    │     error/warn → forwarded immediately
    │     info/debug → batched (5s flush interval)
    │
    └── OUTPUT: Loki (HTTP)
          URL: http://loki.internal:3100/loki/api/v1/push
          Labels:
            service = {service_name}
            env     = {environment}
            level   = {log_level}
          Batch size: 1000 lines or 5s, whichever comes first
          TLS: Loki is internal VPC — HTTP; Loki → S3 backend uses HTTPS
```

---

## 3. Fluent Bit Configuration

```ini
[SERVICE]
    Flush         5
    Grace         10
    Log_Level     warn
    Parsers_File  parsers.conf

[INPUT]
    Name              forward
    Listen            0.0.0.0
    Port              24224
    # FireLens routes ECS stdout here

[FILTER]
    Name              record_modifier
    Match             *
    # Redact PII fields
    Remove_key        email
    Remove_key        phone
    Remove_key        recipient_contact

[FILTER]
    Name              lua
    Match             *
    Script            redact.lua
    Call              redact_sensitive
    # Lua function redacts dynamic field names matching *password*, *secret*key*, authorization

[FILTER]
    Name              record_modifier
    Match             *
    Record            ecs_cluster    ${ECS_CLUSTER}
    Record            ecs_task_id    ${ECS_TASK_ID}
    Record            environment    ${NODE_ENV}

[OUTPUT]
    Name              loki
    Match             *
    Host              loki.internal
    Port              3100
    Labels            service=$service,env=$environment,level=$level
    Label_keys        $tenant_id,$correlation_id
    Auto_Kubernetes_Labels off
    Line_Format       json
```

---

## 4. Grafana Loki Configuration

**Storage backend:** S3 (`anji-schedulo-loki-chunks-{env}`, SSE-KMS)  
**Retention:** 30 days (enforced by Loki compactor)  
**Index:** TSDB  
**Query:** LogQL

Key LogQL queries used by the Platform team:

```logql
# All errors from a specific service
{service="booking-command-service", env="production", level="error"} | json

# All logs for a specific booking (correlation_id)
{env="production"} |= "\"correlation_id\":\"abc-123\""

# All logs for a specific tenant
{env="production"} |= "\"tenant_id\":\"tid-abc\"" | json | level="error"

# DLQ-related log lines
{env="production"} |= "dlq" | json

# Slow saga logs (response time > 3000ms)
{service="booking-command-service", env="production"} | json | saga_duration_ms > 3000
```

---

## 5. Mandatory Log Fields

Every log line emitted by every NestJS service must include:

| Field | Source | Example |
|---|---|---|
| `timestamp` | pino (auto) | `2026-06-28T10:00:00.000Z` |
| `level` | pino | `info` |
| `service` | app config | `booking-command-service` |
| `tenant_id` | TenantContext middleware | `tid-abc-123` |
| `correlation_id` | X-Correlation-Id header | `corr-xyz-789` |
| `user_id` | JWT sub claim | `user-def-456` |
| `trace_id` | OpenTelemetry SDK (auto) | `4bf92f3577b34da6a3ce929d0e0e4736` |
| `span_id` | OpenTelemetry SDK (auto) | `00f067aa0ba902b7` |
| `message` | application code | `Booking saga step: slot_check` |

Fields that must never appear in logs (Fluent Bit blocks these, but applications must not log them either):

- Raw `email`, `phone`, or `name` values
- Authorization header values or JWT tokens
- Card numbers or payment instrument details
- AWS secret values or API keys

---

## 6. Log → Trace Correlation

Every log line includes `trace_id` from the OpenTelemetry SDK. In Grafana, clicking a log line opens the associated Tempo trace directly (Loki → Tempo data link). This provides end-to-end observability from a log message to the full distributed trace.

---

## 7. Retention and Archival

| Log Type | Retention in Loki | Archive |
|---|---|---|
| Application logs (all levels) | 30 days | Not archived beyond Loki retention |
| Flyway migration logs | 30 days | Not archived |
| Audit log (from `audit_log` table) | Permanent (in PostgreSQL + S3 WORM) | Nightly export to `audit-logs` S3 bucket |

Audit logs are stored in PostgreSQL (INSERT-only, BR014) and S3 (WORM), not in Loki. Loki is for operational logs only.

---

## 8. Traceability

| Control | NFR | BR | Doc |
|---|---|---|---|
| PII redaction in Fluent Bit | NFR013 | — | A6-data-privacy-arch.md |
| correlation_id in every log | NFR014 | — | A11-observability.md |
| 30-day log retention | NFR013 | — | A7-infrastructure.md |
| Loki → Tempo trace link | NFR014 | — | OBS-003-distributed-tracing.md |
| Mandatory log fields | — | BR013 | A11-observability.md |
