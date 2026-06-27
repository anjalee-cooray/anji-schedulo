# ADR-003: AWS SNS + SQS over Apache Kafka

**Status:** Accepted  
**Date:** 2026-06-27  
**Deciders:** Architecture Team

---

## Context

BookVault requires a reliable, ordered, at-least-once message bus with:
- Fan-out (one event → multiple independent consumers)
- Per-tenant event ordering
- Dead letter queue support
- Local development parity
- Low operational overhead at MVP stage

Two serious candidates were evaluated: **Apache Kafka** and **AWS SNS + SQS**.

---

## Decision

We use **AWS SNS (fan-out) + SQS FIFO (ordered delivery per tenant) + AWS Kinesis (analytics stream processing)** for the initial platform, with a documented migration path to Kafka at the 200+ tenant scale threshold.

---

## Detailed Comparison

| Dimension | Kafka | SNS + SQS FIFO |
|---|---|---|
| **Operations** | Self-managed or MSK ($500–$2,000/month) | Fully managed, serverless |
| **Fan-out** | Requires consumer groups on same topic | SNS → multiple SQS subscribers |
| **Per-tenant ordering** | Manual: partition key per tenant | Built-in: `MessageGroupId` = `tenant_id` |
| **DLQ** | Custom implementation required | Native `RedrivePolicy` on every queue |
| **At-least-once** | Yes | Yes |
| **Exactly-once** | With transactions + idempotent producer | Not native (mitigated via Inbox Pattern) |
| **Replay** | Native: reset consumer offset | Via PostgreSQL event store → Relay re-publish |
| **Retention** | Configurable, days to forever | 14 days max (SQS) |
| **Local dev** | Requires Kafka container (~2GB RAM) | LocalStack (lightweight) |
| **Cost at 100 tenants** | Fixed MSK baseline regardless of traffic | Pay per message (near zero at low volume) |
| **Cost at 10,000 tenants** | Fixed + storage | Scales proportionally |
| **Kafka Streams** | Native | Not available (use Kinesis Data Streams) |
| **Schema registry** | Confluent / Glue | Custom versioning in envelope |

---

## Rationale

### Why SNS + SQS wins for MVP

1. **Zero ops burden**: No cluster to size, patch, or rebalance. Critical for a small team at launch.
2. **Native DLQ**: Every queue gets a `RedrivePolicy` in one line of IaC. With Kafka, we'd implement custom retry/DLQ logic.
3. **Per-tenant ordering for free**: `MessageGroupId = tenant_id` in SQS FIFO. In Kafka we'd manage partition-to-tenant assignment and handle rebalances.
4. **LocalStack parity**: SQS, SNS, and Kinesis all emulate faithfully in LocalStack 3.4. Kafka requires a real container and adds startup time.
5. **Cost model fits SaaS**: At low message volumes (MVP), cost is near zero. We pay as we grow.

### Why Kafka eventually wins at scale

- **Native replay**: Reset a consumer group offset. No need to re-publish from PostgreSQL event store.
- **Kafka Streams**: Stateful stream processing with windowed aggregations is native. Kinesis requires custom aggregation logic.
- **Longer retention**: Kinesis max 7 days; Kafka indefinite. As analytics needs grow, this matters.
- **Exactly-once semantics**: Idempotent producer + transactional API. At high fan-out, the Inbox Pattern overhead (one DB row per message per consumer) becomes a bottleneck.

### Migration path at 200+ tenants

1. Run Kafka alongside SQS FIFO (dual-publish via relay)
2. Migrate consumers one by one
3. Decomission SQS queues after all consumers migrate
4. Replace Kinesis with Kafka Streams

The Transactional Outbox + Relay architecture intentionally isolates the application from the bus choice — only the Relay Service changes.

---

## Consequences

**Positive:**
- Zero infrastructure at launch
- Native DLQ, fan-out, and per-tenant ordering
- LocalStack enables complete local environment without containers for Kafka

**Negative:**
- No native replay (mitigated by PostgreSQL event store + Relay re-publish)
- 14-day message retention limit (mitigated by event store as source of truth)
- No exactly-once (mitigated by Inbox Pattern deduplication)
- At 500M+ messages/month, SNS+SQS costs more than MSK

**Decision trigger for Kafka migration:**
- Tenant count > 200 AND
- Inbox Pattern DB overhead measurable in load tests OR
- Analytics requires > 7-day event retention for stream processing
