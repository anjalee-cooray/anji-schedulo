# ADR-001: Event-Driven Architecture

**Status:** Accepted  
**Date:** 2026-06-27  
**Deciders:** Architecture Team

---

## Context

BookVault orchestrates multi-step business operations (booking, cancellation, onboarding) that span multiple independently deployable services. We needed to decide how these services communicate and coordinate state.

The core challenge: a confirmed appointment triggers work in at least five systems — availability release, payment capture, query view update, dashboard refresh, and customer notification. These must be reliable, independently evolvable, and must not cascade failures.

Alternatives considered:
1. **Synchronous REST chains** — each service calls the next directly
2. **Event-Driven with message bus** — services publish events; subscribers react
3. **Shared database** — all services read/write a common data store directly

---

## Decision

We adopt an **event-driven architecture** using a message bus (AWS SNS + SQS) as the primary integration mechanism for post-booking flows, combined with targeted **synchronous orchestration** (Saga pattern) for the critical booking path where atomicity and immediate failure feedback are required.

---

## Rationale

### Why event-driven over synchronous REST chains?

| Concern | Synchronous chains | Event-driven |
|---|---|---|
| Notification failure | Blocks booking confirmation | Independent — booking completes regardless |
| Dashboard lag | Must wait for all views to update | Acceptable eventual consistency |
| Adding new consumers | Requires Booking Service code change | New SQS subscriber, no Booking Service change |
| Service availability coupling | All services must be up for booking to succeed | Consumers can be down; messages queue up |
| Testability | Must mock all downstream services | Test each service in isolation |

### Why keep synchronous orchestration for the booking critical path?

The booking saga (slot reservation → payment → confirmation) requires atomicity. If payment fails, the slot must be released immediately and the customer must receive an error response — not an eventual email. Asynchronous choreography does not provide the synchronous response semantics the customer experience requires.

We use **Saga Orchestration** here: the Booking Command Service is the orchestrator, calls Availability and Payment synchronously, and applies compensations on failure. All other flows use **Saga Choreography** (e.g. cancellation), where each step reacts to the previous event independently.

### Why this is correct for a scheduling platform

- **Notification failure frequency** is non-trivial (SMTP timeouts, SMS provider outages). Decoupling protects booking throughput.
- **Dashboard updates** are not blocking — a 5-second lag is acceptable for operational monitoring.
- **Analytics** are inherently retrospective — stream processing on Kinesis is the right fit.
- **New features** (CRM, waitlist, loyalty) can subscribe to existing events with zero impact on core services.

---

## Consequences

**Positive:**
- Booking throughput unaffected by notification/analytics failures
- New consumers added without touching core services
- Full event history enables replay, audit, and time-travel debugging

**Negative:**
- Eventual consistency in dashboard and analytics (accepted — SLO is < 5s lag)
- More infrastructure to operate (SNS, SQS, DLQ, relay)
- Developers must reason about out-of-order delivery and idempotency

**Mitigations:**
- Transactional Outbox eliminates dual-write risk
- Inbox Pattern and event_id watermarks ensure idempotency
- DLQ + Ops Service ensures no event is silently lost
