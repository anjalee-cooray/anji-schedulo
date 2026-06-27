# ADR-004: Saga Pattern Selection — Orchestration vs Choreography

**Status:** Proposed | Accepted | Deprecated  
**Date:** YYYY-MM-DD  
**Deciders:** Architecture Team

---

## Context

[Describe the distributed transactions in your system that cannot use traditional ACID transactions across service boundaries. List the multi-step flows that need coordination and their consistency requirements.]

Two implementation styles exist:
- **Orchestration**: A central coordinator calls each step and applies compensations
- **Choreography**: Each service reacts to events and emits the next event independently

---

## Decision

[State which pattern you apply to which flows, and why the choice differs per flow rather than being a platform-wide decision.]

---

## Rationale

### [Flow A] → Orchestration

**Requirements:**
- [List the specific requirements that demand synchronous, coordinated execution — e.g. user receives a typed error response, compensation must be immediate]

**Why Orchestration fits:**
- [Explain the specific reasons — e.g. linear sequence, immediate compensation, single service owns saga state]

**Why Choreography would not fit:**
- [Explain what breaks — e.g. user cannot wait for async completion, debugging requires correlating events across services]

### [Flow B] → Choreography

**Requirements:**
- [List the requirements that allow independent, decoupled execution — e.g. multiple steps must succeed regardless of each other's outcome]

**Why Choreography fits:**
- [Explain the specific reasons — e.g. single trigger event, failure isolation between steps, no central coordination needed]

---

## Consequences

**Positive:**
- [List benefits — e.g. atomic and deterministic for the critical path, independently resilient for async flows]

**Negative:**
- [List trade-offs — e.g. increased mental model complexity, orchestrator is a bottleneck]

**Mitigations:**
- [How you address each negative — e.g. correlation_id propagation, heavily tested orchestrator service]
