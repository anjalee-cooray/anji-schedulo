# ADR-005: Service Concurrency Model

**Status:** Proposed | Accepted | Deprecated  
**Date:** YYYY-MM-DD  
**Deciders:** Architecture Team

---

## Context

[Describe your concurrency requirements. How many concurrent requests does your system need to handle? What types of blocking I/O dominate — database queries, external HTTP calls, queue polling, cache access?]

Alternatives for handling concurrent I/O:
1. **Platform threads** (traditional thread-per-request) — bounded pool, simple but limited concurrency
2. **Reactive programming** (e.g. Project Reactor, RxJava) — high concurrency but fundamentally changes the programming model
3. **[Other option relevant to your stack]** — [brief description]

---

## Decision

[State your chosen concurrency model and the configuration or framework feature that enables it.]

---

## Rationale

### Why [chosen model] over [alternative]?

| Dimension | [Alternative] | [Chosen model] |
|---|---|---|
| **Programming model** | [describe] | [describe] |
| **Learning curve** | [describe] | [describe] |
| **Stack traces** | [describe] | [describe] |
| **Library compatibility** | [describe] | [describe] |
| **Concurrency ceiling** | [describe] | [describe] |
| **Debugging** | [describe] | [describe] |

[Explain the decisive factor for your team and codebase — typically the trade-off between concurrency ceiling and development complexity.]

### Known compatibility issues

[List any libraries or patterns in your stack that interact poorly with the chosen model — e.g. libraries using blocking synchronization primitives. Explain how you mitigate each.]

---

## Consequences

**Positive:**
- [List benefits specific to your chosen model and stack]

**Negative:**
- [List trade-offs — e.g. minimum runtime version required, library audit needed]

**Mitigations:**
- [How you address each negative]
