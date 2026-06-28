# A11 · Observability

**Project:** AnjiSchedulo  
**Version:** 1.0.0  
**Owner:** Pubudu Anjalee Cooray  
**Last Updated:** 2026-06-28

---

## 1. Overview

AnjiSchedulo uses the **Grafana LGTM stack** (Loki · Grafana · Tempo · Mimir) deployed on AWS ECS Fargate in the same VPC as the application services. Grafana Cloud serves as the fallback if self-hosted LGTM is unavailable during an incident.

The observability system is designed to answer four questions at any moment:
1. Are the SLOs being met? (Mimir metrics → Grafana SLO dashboards)
2. What happened for a specific booking or tenant? (Tempo traces + Loki logs via `correlation_id`)
3. Are there any stuck events or tenant data isolation issues? (Grafana alerts)
4. What is the operational health of the infrastructure? (Infrastructure Health dashboard)

---

## 2. Metrics

**Collection:** Prometheus scrape. Each NestJS service exposes `/metrics` via `prom-client`. Mimir scrapes via Prometheus `remote_write` every 15 seconds.  
**Storage:** Grafana Mimir (horizontally scalable, S3 backend).  
**Retention:** 90 days.

### Key Metrics

| Metric | Description | PromQL Query | Satisfies |
|---|---|---|---|
| `bookings_saga_duration_seconds` | End-to-end booking saga latency | `histogram_quantile(0.95, sum(rate(bookings_saga_duration_seconds_bucket[5m])) by (le, tenant_id))` | NFR004, SLI001 |
| `booking_success_rate` | Ratio of confirmed bookings to total attempts | `sum(rate(bookings_saga_duration_seconds_count{status='confirmed'}[5m])) / sum(rate(bookings_saga_duration_seconds_count[5m]))` | NFR001, SLI002 |
| `booking_saga_errors_total` | Booking saga failures by `failure_reason` | `sum(rate(booking_saga_errors_total[5m])) by (failure_reason)` | NFR001 |
| `http_server_requests_seconds` | HTTP p95 latency by route and status | `histogram_quantile(0.95, sum(rate(http_server_requests_seconds_bucket{route=~'.*/availability.*'}[5m])) by (le))` | NFR005, SLI003 |
| `dashboard_view_response_seconds` | Dashboard p95 latency | `histogram_quantile(0.95, sum(rate(dashboard_view_response_seconds_bucket[5m])) by (le))` | NFR006, SLI004 |
| `dashboard_view_lag_seconds` | Lag between latest event and dashboard update | `max(dashboard_view_lag_seconds) by (tenant_id)` | NFR008, SLI004 |
| `dlq_depth_total` | Messages in each DLQ | `sum(dlq_depth_total) by (queue_name)` | NFR016, SLI005 |
| `outbox_backlog_total` | Unpublished outbox records (relay lag) | `sum(outbox_backlog_total) by (service)` | NFR003 |
| `notification_delivery_failures_total` | Notification failures by channel | `sum(rate(notification_delivery_failures_total[5m])) by (channel, failure_reason)` | NFR002 |
| `isolation_monitor_violations_total` | Cross-tenant isolation violations — must always be zero | `sum(isolation_monitor_violations_total)` | NFR012, BR005 |
| `tenant_active_bookings_total` | Confirmed appointments per tenant | `sum(tenant_active_bookings_total) by (tenant_id)` | FR008 |

---

## 3. Logs

**Collection:** Fluent Bit deployed as ECS sidecar on every task. Collects stdout/stderr, applies PII redaction, enriches with ECS task metadata, forwards to Grafana Loki via HTTP.  
**Storage:** Grafana Loki (S3 backend, eu-west-1).  
**Retention:** 30 days.

### Mandatory Log Fields

Every log line emitted by every service must include:

| Field | Description |
|---|---|
| `timestamp` | ISO 8601 timestamp |
| `level` | `debug` \| `info` \| `warn` \| `error` |
| `service` | NestJS app name (e.g. `booking-command-service`) |
| `tenant_id` | From request TenantContext; `'system'` for non-tenant-scoped ops |
| `correlation_id` | Propagated from API Gateway `X-Correlation-Id` header (NFR014) |
| `user_id` | From JWT `sub` claim; `'anonymous'` for unauthenticated requests |
| `trace_id` | OpenTelemetry trace ID (enables Loki → Tempo correlation) |
| `span_id` | OpenTelemetry span ID |
| `message` | Human-readable log message |

### PII Fields Redacted by Fluent Bit

The following fields are stripped from all log streams before forwarding to Loki:

- `email`
- `phone`
- `name` (when standalone field — not service/staff names in business context)
- `recipient_contact`
- Any field matching `*secret*key*` or `*password*`
- `authorization` header value

---

## 4. Traces

**Instrumentation:** OpenTelemetry SDK for Node.js (auto-instrumentation for NestJS HTTP, `pg` PostgreSQL driver, Redis, AWS SDK calls).  
**Exporter:** OTLP (gRPC) to Grafana Tempo.  
**Retention:** 7 days.

### Sampling Strategy

| Path | Sample Rate | Rationale |
|---|---|---|
| Booking saga | 100% | Critical path — every booking traced end-to-end (NFR004) |
| Cancellation saga | 100% | Every cancellation traced across choreography participants |
| DLQ triage operations | 100% | Operator actions always fully traced |
| Replay operations | 100% | Operator actions always fully traced |
| Slot availability query | 10% | High-volume read; sampled to control Tempo storage |
| Dashboard read | 5% | Very high volume; O(1) response; minimal tracing needed |

### Mandatory Span Attributes

Every span carries:

- `tenant_id`
- `correlation_id`
- `user_id`
- `appointment_id` (when applicable)
- `payment_id` (when applicable)
- `saga_step` (`slot_check` \| `payment_capture` \| `confirmation` \| `slot_release` \| `refund`)
- `db.statement` (sanitised — no PII in SQL parameters)
- `messaging.destination` (SNS topic or SQS queue name)
- `http.route`
- `http.status_code`

---

## 5. Dashboards

| Dashboard | Contents |
|---|---|
| **Platform Overview** | Service health (up/down), request rate, error rate, p95 latency per service, ECS task counts, active alerts |
| **Booking Operations** | Saga success rate, saga p95 vs 3s SLO, errors by `failure_reason`, active slot locks, idempotency key hit rate, outbox backlog |
| **Tenant Operations** | Bookings per tenant per day (heatmap), cancellation rate per tenant, net revenue per tenant, active tenants by tier, new registrations |
| **DLQ Triage** | DLQ depth per queue, oldest unresolved message age, resolution rate, failed event breakdown by consumer and `failure_reason`, notification failure rate by channel |
| **Infrastructure Health** | RDS CPU and connections, ElastiCache hit rate and memory, SQS queue depth, ECS CPU/memory per service, outbox relay lag |
| **SLO Burn Rate** | Error budget remaining per SLO (SLO001–SLO004), burn rate over 1h/6h/24h windows, alert state, projected exhaustion date |
| **Security and Isolation** | Isolation monitor violation count (must be zero), failed auth attempts/min, JWT validation errors by service, Secrets Manager access anomalies |

---

## 6. SLIs

| ID | Name | Query | NFR |
|---|---|---|---|
| SLI001 | Booking Saga p95 Latency | `histogram_quantile(0.95, sum(rate(bookings_saga_duration_seconds_bucket[5m])) by (le))` | NFR004 |
| SLI002 | Booking Success Rate | `sum(rate(bookings_saga_duration_seconds_count{status='confirmed'}[5m])) / sum(rate(bookings_saga_duration_seconds_count[5m]))` | NFR001 |
| SLI003 | Availability Query p95 Latency | `histogram_quantile(0.95, sum(rate(http_server_requests_seconds_bucket{route=~'.*/availability.*',status!~'5..'}[5m])) by (le))` | NFR005 |
| SLI004 | Dashboard Response p95 Latency | `histogram_quantile(0.95, sum(rate(dashboard_view_response_seconds_bucket[5m])) by (le))` | NFR006 |
| SLI005 | DLQ Depth | `sum(dlq_depth_total)` | NFR016 |

---

## 7. SLOs

| ID | Name | Target | Window | Error Budget | SLI |
|---|---|---|---|---|---|
| SLO001 | Booking Saga Latency | 99.5% of requests complete p95 < 3s | 30 days | 0.5% (~7.2 hours) | SLI001 |
| SLO002 | Platform Availability | 99.9% core booking uptime | 30 days | 0.1% (~43.2 min) | SLI002 |
| SLO003 | Availability Query Latency | 99.5% of queries p95 < 500ms | 30 days | 0.5% | SLI003 |
| SLO004 | Dashboard Response Latency | 99.5% of requests p95 < 200ms | 30 days | 0.5% | SLI004 |

---

## 8. Alerts

| ID | Name | Condition | Severity | Channels |
|---|---|---|---|---|
| ALT001 | Booking Saga Error Rate Critical | `sum(rate(booking_saga_errors_total{failure_reason!~'slot_unavailable\|booking_limit_exceeded'}[5m])) > 0.05` | P1 | PagerDuty, Slack #incidents |
| ALT002 | Platform Availability Below SLO | `sum(rate(bookings_saga_duration_seconds_count{status='confirmed'}[15m])) / sum(rate(bookings_saga_duration_seconds_count[15m])) < 0.999` | P1 | PagerDuty, Slack #incidents |
| ALT003 | DLQ Depth Non-Zero | `sum(dlq_depth_total) > 0` | P2 | PagerDuty, Slack #incidents |
| ALT004 | Booking Saga p95 > 3s | `histogram_quantile(0.95, sum(rate(bookings_saga_duration_seconds_bucket[10m])) by (le)) > 3` | P2 | PagerDuty, Slack #incidents |
| ALT005 | Cross-Tenant Isolation Violation | `sum(isolation_monitor_violations_total) > 0` | P1 | PagerDuty, Slack #incidents |
| ALT006 | Outbox Relay Lag | `sum(outbox_backlog_total) > 100` | P2 | PagerDuty, Slack #incidents |
| ALT007 | RDS CPU High | `avg(aws_rds_cpu_utilization_average) > 80` | P2 | Slack #incidents |
| ALT008 | Notification Failure Rate High | `sum(rate(notification_delivery_failures_total[5m])) / sum(rate(notification_attempts_total[5m])) > 0.10` | P3 | Slack #incidents |

**Alert routing:** P1 and P2 → PagerDuty (on-call engineer paged immediately). All severities → Slack #incidents. P3 → email daily digest.

---

## 9. Traceability

| Observability Concern | NFR | FR |
|---|---|---|
| Booking saga latency SLO | NFR004 | FR004 |
| Platform availability SLO | NFR001 | FR004 |
| Availability query latency SLO | NFR005 | FR003 |
| Dashboard response SLO | NFR006 | FR008 |
| Distributed tracing via correlation_id | NFR014 | — |
| DLQ depth and resolution SLA | NFR016 | FR010 |
| Cross-tenant isolation monitoring | NFR012 | — |
| PII redaction in logs | NFR013 | — |
