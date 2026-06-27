# ADR-004: Saga Pattern Selection — Orchestration vs Choreography

**Status:** Accepted  
**Date:** 2026-06-27  
**Deciders:** Architecture Team

---

## Context

Distributed transactions spanning multiple services cannot use traditional ACID transactions across service boundaries. The **Saga pattern** provides a mechanism for maintaining consistency via a sequence of local transactions with compensating transactions on failure.

Two implementation styles exist:
- **Orchestration**: A central coordinator calls each step and applies compensations
- **Choreography**: Each service reacts to events and emits the next event independently

BookVault has two primary multi-step flows requiring saga coordination:
1. **Booking creation** — slot reservation → payment → confirmation (failure must compensate immediately)
2. **Appointment cancellation** — cancellation event → refund → slot release → notifications

---

## Decision

We apply **Saga Orchestration** to the booking creation flow and **Saga Choreography** to the cancellation flow. The choice is made per flow, not as a platform-wide decision.

---

## Rationale

### Booking creation → Orchestration

**Requirements:**
- Customer must receive a synchronous response: "confirmed" or a specific failure reason
- If payment fails, slot must be released immediately before responding
- If slot is unavailable, no payment must be attempted
- The sequence is strictly linear: Availability first, then Payment only if Availability succeeds

**Why Orchestration fits:**
- The Booking Command Service acts as the orchestrator — it calls Availability synchronously, then Payment synchronously, then writes the confirmed event
- Compensation is immediate and synchronous: if Payment fails, the orchestrator calls Availability to release the slot before returning the error to the customer
- The customer gets a deterministic, typed error (409 slot unavailable / 402 payment failed) — not an "eventually we'll tell you what happened" model
- A single service owns the saga state — no coordination needed across services to determine whether the saga is complete

**Why Choreography would not fit here:**
- Customer cannot wait for a choreography to complete — they are holding an HTTP connection
- Multiple services would each need to store partial saga state
- Debugging a failed booking would require correlating events across 3+ services

### Cancellation flow → Choreography

**Requirements:**
- Slot must be released regardless of refund outcome
- Notifications must go out regardless of refund outcome
- No single service needs to coordinate the others — they all just need to know the appointment was cancelled
- Refund failure is an ops concern, not a customer-blocking concern

**Why Choreography fits:**
- `appointment.cancelled` is the single trigger event; each service (payment, availability, notification) subscribes independently
- Failure isolation: refund failure routes to DLQ without preventing slot release or customer notification
- No central point of failure: if the Cancellation Saga Service is temporarily down, SQS holds the events and consumers catch up
- The business rule is simple: "on cancellation, do these three things" — not "only do B if A succeeded"

---

## Consequences

**Positive:**
- Booking confirmation is atomic, synchronous, and deterministic
- Cancellation steps are independently resilient and independently scalable
- Each pattern is applied only where it fits — no forcing of one style

**Negative:**
- Two saga patterns increase the mental model complexity for new developers
- Booking orchestrator is a bottleneck — if BookingCommandService is down, no new bookings (mitigated by ECS Fargate autoscaling and multi-AZ deployment)
- Choreography tracing requires correlation_id propagation across all services (required by BR013)

**Mitigations:**
- BookingCommandService is the most heavily tested service (unit, integration, and isolation tests)
- correlation_id is propagated in all event envelopes and HTTP headers — Tempo provides full saga trace waterfall
- Cancellation saga state can be reconstructed from the event log if any consumer misses an event
