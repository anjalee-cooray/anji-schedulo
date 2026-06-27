# ADR-003: Message Bus Technology Selection

**Status:** Proposed | Accepted | Deprecated  
**Date:** YYYY-MM-DD  
**Deciders:** Architecture Team

---

## Context

[Describe your message bus requirements — fan-out, ordering guarantees, DLQ support, local development parity, operational overhead constraints, and cost model at your expected scale.]

Candidates evaluated: [list the technologies you compared, e.g. Apache Kafka, AWS SNS + SQS, Google Pub/Sub, RabbitMQ]

---

## Decision

[State your chosen technology and any hybrid use — e.g. managed queue service for transactional events, stream processing service for analytics.]

---

## Detailed Comparison

| Dimension | Option A | Option B |
|---|---|---|
| **Operations** | [describe] | [describe] |
| **Fan-out** | [describe] | [describe] |
| **Ordering** | [describe] | [describe] |
| **DLQ** | [describe] | [describe] |
| **Replay** | [describe] | [describe] |
| **Local dev** | [describe] | [describe] |
| **Cost at launch scale** | [describe] | [describe] |
| **Cost at target scale** | [describe] | [describe] |

---

## Rationale

### Why [chosen option] wins for launch

[Explain the decisive factors — typically operational simplicity, cost model, native features that replace custom code.]

### Migration path (if applicable)

[If your chosen option has known scale limits, describe the trigger conditions and migration steps — e.g. "at N tenants, switch to X because of Y".]

---

## Consequences

**Positive:**
- [List benefits]

**Negative:**
- [List trade-offs]

**Mitigations:**
- [How you address each negative]
