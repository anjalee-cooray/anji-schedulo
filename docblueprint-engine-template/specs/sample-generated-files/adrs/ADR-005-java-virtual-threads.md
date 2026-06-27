# ADR-005: Java 21 Virtual Threads for Service Concurrency

**Status:** Accepted  
**Date:** 2026-06-27  
**Deciders:** Architecture Team

---

## Context

BookVault's services make extensive use of blocking I/O: PostgreSQL queries via JDBC, HTTP calls to Payment provider and other internal services, SQS message polling, and Redis cache access. The traditional thread-per-request model (platform threads) limits concurrency to the size of the thread pool — typically 200–400 threads per JVM instance before context-switching overhead degrades performance.

Alternatives for handling high-concurrent I/O in Java:
1. **Platform threads** (traditional) — bounded pool, cheap but limited concurrency
2. **Reactive programming** (Project Reactor / WebFlux) — high concurrency but fundamentally changes the programming model
3. **Virtual Threads** (Project Loom, Java 21 GA) — lightweight threads that block cheaply without consuming an OS thread

---

## Decision

We use **Java 21 Virtual Threads** (`spring.threads.virtual.enabled=true`) as the concurrency model for all BookVault services. Spring Boot 3.2+ enables this with a single configuration flag.

---

## Rationale

### Why Virtual Threads over Reactive?

| Dimension | Reactive (WebFlux) | Virtual Threads |
|---|---|---|
| **Programming model** | Non-blocking callbacks / Mono/Flux | Conventional blocking code |
| **Learning curve** | High — entire team must learn reactive patterns | None — code looks identical to synchronous code |
| **Stack traces** | Fragmented — hard to trace across operators | Full stack traces as expected |
| **JDBC compatibility** | Requires R2DBC (reactive JDBC) | Standard JDBC — no changes |
| **Library compatibility** | Must use reactive-compatible versions | All blocking libraries work unchanged |
| **Concurrency** | Very high (no thread blocking at all) | High (millions of virtual threads possible) |
| **Debugging** | Complex — async context breaks debuggers | Standard debuggers work |

For a booking platform with a conventional team, reactive programming's complexity cost outweighs its concurrency benefits. Virtual Threads deliver most of the concurrency gain with zero programming model change.

### Why Virtual Threads over platform thread pools?

The booking saga makes 2 synchronous HTTP calls per booking request (Availability + Payment). With platform threads and a 200-thread pool, throughput peaks at roughly 200 concurrent bookings-in-flight. With Virtual Threads, the JVM can schedule millions of lightweight virtual threads — the HTTP blocking time doesn't consume an OS thread.

At target load (500 bookings/hour per tenant, 200 tenants peak = ~28 concurrent bookings), platform threads are sufficient. Virtual Threads provide headroom for growth and eliminate the need to carefully tune pool sizes.

### HikariCP compatibility

HikariCP uses `synchronized` blocks internally. Virtual threads parked inside a `synchronized` block **pin the carrier thread**, negating the benefit. We set:
```yaml
spring.datasource.hikari.maximum-pool-size: 20
```
And use `ReentrantLock` instead of `synchronized` in any custom connection management code.

The small pool size (20) is intentional — virtual threads block cheaply waiting for a JDBC connection, so we don't need 200 connections. This reduces database connection overhead.

---

## Consequences

**Positive:**
- Standard blocking JDBC, RestTemplate, and SQS polling code — no reactive overhead
- Full Java stack traces — simpler debugging and profiling
- Spring Boot 3.3 + `spring.threads.virtual.enabled=true` — one line of config
- Headroom for 10x+ concurrent request growth without architecture change

**Negative:**
- `synchronized` blocks pin carrier threads — must audit third-party library usage
- Virtual threads still consume memory per thread (stack allocated lazily) — under extreme concurrency, heap pressure increases
- Java 21 LTS required — no support for Java 17 or 11

**Mitigations:**
- HikariCP pool capped at 20 connections (avoids thread pinning at connection acquisition)
- Libraries using `synchronized` internally: Lettuce (Redis) uses `synchronized` — replaced with connection pool mode to avoid pinning
- Java 21 is LTS — supported until September 2031
