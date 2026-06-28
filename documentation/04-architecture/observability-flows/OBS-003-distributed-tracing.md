# OBS-003 · Distributed Tracing Flow

**Project:** AnjiSchedulo  
**Version:** 1.0.0  
**Last Updated:** 2026-06-28

---

## 1. Overview

Distributed traces link every step of a request — from the initial HTTP call through multiple microservices, database queries, Redis operations, and SQS messages — into a single observable timeline. AnjiSchedulo uses the OpenTelemetry SDK for Node.js (auto-instrumentation) and exports traces to Grafana Tempo via OTLP gRPC.

The `correlation_id` is propagated through all spans and log lines, enabling a single search term to pull all related logs and the full distributed trace.

---

## 2. Tracing Architecture

```
HTTP Request arrives at api-gateway
    │ TraceContext extracted from incoming W3C traceparent header
    │ (or new trace created if no header present)
    ▼
api-gateway (Span: http.server)
    │ Sets: X-Correlation-Id header (correlation_id = trace_id)
    │ Propagates W3C traceparent to downstream services
    ▼
booking-command-service (Span: http.server)
    ├── Span: db.query (PostgreSQL — slot availability check)
    ├── Span: redis.set (slot lock acquisition)
    ├── Span: http.client (→ payment-service)
    │       └── payment-service (Span: http.server)
    │               ├── Span: http.client (→ Stripe API)
    │               └── Span: db.query (write payment record)
    ├── Span: db.query (write appointment + outbox_records)
    └── Span: messaging.produce (outbox_records INSERT)

outbox-relay (polling loop)
    └── Span: messaging.produce (SNS Publish)

notification-service (SQS consumer)
    └── Span: messaging.receive (SQS ReceiveMessage)
        └── Span: http.client (→ SendGrid)

All spans → OTLP gRPC → Grafana Tempo
```

---

## 3. OpenTelemetry Configuration

Auto-instrumentation is applied via the OpenTelemetry Node.js SDK at process startup:

```typescript
// apps/shared/telemetry/src/index.ts
import { NodeSDK } from '@opentelemetry/sdk-node';
import { OTLPTraceExporter } from '@opentelemetry/exporter-trace-otlp-grpc';
import { Resource } from '@opentelemetry/resources';
import { SemanticResourceAttributes } from '@opentelemetry/semantic-conventions';
import { getNodeAutoInstrumentations } from '@opentelemetry/auto-instrumentations-node';

const sdk = new NodeSDK({
  resource: new Resource({
    [SemanticResourceAttributes.SERVICE_NAME]: process.env.SERVICE_NAME,
    [SemanticResourceAttributes.DEPLOYMENT_ENVIRONMENT]: process.env.NODE_ENV,
  }),
  traceExporter: new OTLPTraceExporter({
    url: 'grpc://tempo.internal:4317',
  }),
  instrumentations: [getNodeAutoInstrumentations()],
});

sdk.start();
```

**Auto-instrumented libraries:** NestJS HTTP (incoming + outgoing), `pg` PostgreSQL driver, `ioredis` Redis client, AWS SDK v3 (SNS, SQS, Secrets Manager calls).

---

## 4. Sampling Strategy

| Path | Sample Rate | Configuration |
|---|---|---|
| Booking saga (POST /bookings) | 100% | `AlwaysSampleSampler` on route match |
| Cancellation saga (DELETE /appointments/{id}) | 100% | `AlwaysSampleSampler` on route match |
| DLQ triage operations | 100% | `AlwaysSampleSampler` on operator routes |
| Replay operations | 100% | `AlwaysSampleSampler` on operator routes |
| Slot availability query (GET /availability) | 10% | `TraceIdRatioSampler(0.1)` |
| Dashboard read (GET /dashboard) | 5% | `TraceIdRatioSampler(0.05)` |
| Health check (GET /health) | 0% | `NeverSampleSampler` |

---

## 5. Mandatory Span Attributes

Every span, regardless of service, must carry:

| Attribute | Source | Example |
|---|---|---|
| `tenant_id` | TenantContext middleware | `tid-abc-123` |
| `correlation_id` | Propagated from X-Correlation-Id | `corr-xyz-789` |
| `user_id` | JWT sub claim | `user-def-456` |
| `service.name` | SDK resource | `booking-command-service` |
| `deployment.environment` | SDK resource | `production` |

Additional attributes for specific span types:

| Span Type | Extra Attributes |
|---|---|
| Booking saga spans | `appointment_id`, `saga_step` (`slot_check\|payment_capture\|confirmation`) |
| Cancellation saga spans | `appointment_id`, `saga_step` (`slot_release\|refund\|notification`) |
| DB spans | `db.statement` (sanitised — no PII in parameters), `db.name`, `db.operation` |
| SQS spans | `messaging.destination`, `messaging.message_id`, `event_type` |
| HTTP client spans | `http.url` (domain only, no path params with PII), `http.status_code` |
| Payment spans | `payment_id`, `stripe_payment_intent_id` |

---

## 6. Trace Propagation Through SQS

SQS does not natively propagate trace context in message headers. AnjiSchedulo injects trace context into the event envelope:

```json
{
  "event_id": "...",
  "event_type": "appointment.confirmed",
  "tenant_id": "...",
  "correlation_id": "corr-xyz-789",
  "trace_context": {
    "traceparent": "00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01",
    "tracestate": ""
  },
  ...
}
```

Consumer services extract `trace_context` from the event envelope and use it to continue the trace as a child span. This creates an unbroken trace from the original HTTP request through SNS/SQS into the consumer.

---

## 7. Grafana Tempo Configuration

**Retention:** 7 days  
**Storage:** S3 (`anji-schedulo-tempo-chunks-{env}`, SSE-KMS)  
**Search:** TraceQL  
**Sampling rate written:** 100% of sampled traces (no head-based drop at Tempo)

Key TraceQL queries used by the platform team:

```traceql
# All spans for a specific booking saga
{ .appointment_id = "appt-abc-123" }

# All traces with booking saga errors
{ .saga_step = "payment_capture" && status = error }

# Slow booking sagas (> 3s)
{ .service.name = "booking-command-service" } | duration > 3s

# All traces for a specific tenant
{ .tenant_id = "tid-abc" }

# All traces linked to a correlation_id
{ .correlation_id = "corr-xyz-789" }
```

---

## 8. Log → Trace Correlation

In Grafana, every log line in Loki contains `trace_id`. Clicking the trace_id opens the Tempo trace directly. Every Tempo span contains `correlation_id`, which can be used to search Loki for related logs.

This bidirectional linking means an engineer can start from either a log alert or a slow trace and traverse to the other view instantly.

---

## 9. Traceability

| Tracing Concern | NFR | Doc |
|---|---|---|
| correlation_id propagation | NFR014 | A11-observability.md |
| 100% booking saga sampling | NFR004 | A11-observability.md |
| Trace context in event envelope | NFR014 | A2-system-architecture.md |
| PII exclusion from span attributes | NFR013 | A6-data-privacy-arch.md |
| 7-day trace retention | — | A7-infrastructure.md |
