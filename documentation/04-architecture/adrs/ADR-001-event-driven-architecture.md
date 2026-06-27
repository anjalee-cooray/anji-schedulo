# ADR-001: Event-Driven Architecture

**Status:** Proposed | Accepted | Deprecated  
**Date:** YYYY-MM-DD  
**Deciders:** Architecture Team

---

## Context

[Describe the problem your system needs to solve in terms of service communication and coordination. What operations span multiple services? What are the failure isolation requirements?]

Alternatives considered:
1. **Synchronous REST chains** — each service calls the next directly
2. **Event-Driven with message bus** — services publish events; subscribers react
3. **Shared database** — all services read/write a common data store directly

---

## Decision

[State your chosen architecture and where you deviate from it. For example: event-driven for async flows, synchronous orchestration for the critical path where immediate failure feedback is required.]

---

## Rationale

### Why event-driven over synchronous REST chains?

| Concern | Synchronous chains | Event-driven |
|---|---|---|
| Downstream failure | Blocks the calling operation | Independent — caller completes regardless |
| Adding new consumers | Requires source service code change | New subscriber, no source service change |
| Service availability coupling | All services must be up simultaneously | Consumers can be down; messages queue up |
| Testability | Must mock all downstream services | Test each service in isolation |

### Why keep synchronous calls for the critical path?

[Explain which flows require synchronous execution and why — typically where atomicity, immediate user feedback, or compensation on failure is required.]

### Why this fits your domain

[Describe the specific characteristics of your domain that make event-driven the right fit — e.g. high decoupling requirements, independent scalability of consumers, analytics/audit needs.]

---

## Consequences

**Positive:**
- [List benefits specific to your system]

**Negative:**
- [List trade-offs — e.g. eventual consistency in derived views, more infrastructure to operate]

**Mitigations:**
- [How you address each negative — e.g. Transactional Outbox, Inbox Pattern, DLQ monitoring]
