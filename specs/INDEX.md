# BookVault Specification Index

> **Single Source of Truth** for the BookVault platform. Designed for Spec-Driven Development and AI-assisted code generation.

---

## Layer Map

| Layer | File | Concern |
|---|---|---|
| 1 | [metadata.json](metadata.json) | Project identity, version, authors, changelog |
| 2 | [product.json](product.json) | Vision, problem statement, target market, pricing tiers |
| 3 | [personas.json](personas.json) | 4 personas with goals, frustrations, and primary actions |
| 4 | [glossary.json](glossary.json) | 20 domain terms with definitions and relationships |
| 5 | [domain.json](domain.json) | Entities, aggregate roots, value objects, lifecycle states |
| 6 | [business-rules.json](business-rules.json) | BR001–BR014 with enforcement mechanism per rule |
| 7 | [functional-requirements.json](functional-requirements.json) | FR001–FR013 with acceptance criteria and persona links |
| 8 | [non-functional-requirements.json](non-functional-requirements.json) | NFR001–NFR016: performance, availability, security, consistency |
| 9 | [user-journeys.json](user-journeys.json) | UJ001–UJ007 in business language + implementation detail separated |
| 10 | [bounded-contexts.json](bounded-contexts.json) | 10 bounded contexts, owned entities, events published/consumed |
| 11 | [events.json](events.json) | 19 events: producer, consumers, delivery guarantee, key fields |
| 12 | [apis.json](apis.json) | All REST endpoints: method, path, auth, request, response, errors |
| 13 | [architecture.json](architecture.json) | All 15 patterns with placement rationale, communication model, consistency |
| 14 | [infrastructure.json](infrastructure.json) | Full AWS + Java 21 + Spring Boot 3.3 + Maven 3.9 stack |
| 15 | [security.json](security.json) | Auth, RBAC, tenant isolation layers, PII, audit, compliance |
| 16 | [observability.json](observability.json) | Grafana LGTM, SLIs/SLOs, 6 alerts, 5 dashboards, cross-signal links |
| 17 | [operations.json](operations.json) | DLQ, replay, backups, incident response, migration constraints, tenant lifecycle |
| 18 | [roadmap.json](roadmap.json) | MVP → v1.0 → v1.5 → v2.0 → Marketplace → Platform Expansion |

---

## Architecture Decision Records

| ADR | Title | Status |
|---|---|---|
| [ADR-001](adrs/ADR-001-event-driven-architecture.md) | Why Event-Driven Architecture? | Accepted |
| [ADR-002](adrs/ADR-002-shared-database-rls.md) | Why Shared Database with RLS (Pool Model)? | Accepted |
| [ADR-003](adrs/ADR-003-sns-sqs-over-kafka.md) | Why SNS + SQS over Kafka? | Accepted |
| [ADR-004](adrs/ADR-004-saga-pattern-selection.md) | Why Orchestration for Booking, Choreography for Cancellation? | Accepted |
| [ADR-005](adrs/ADR-005-java-virtual-threads.md) | Why Java 21 Virtual Threads over Reactive? | Accepted |

---

## Design Documents

| Document | Purpose |
|---|---|
| [PRD.md](../PRD.md) | Product requirements, personas, functional/non-functional requirements |
| [SDD.md](../SDD.md) | High-level and detailed system design, all 15 patterns, sequence diagrams |
| [TENANCY_MODEL.md](../TENANCY_MODEL.md) | Full multi-tenant data isolation model, DDL, RLS policies |
| [BACKEND_STACK.md](../BACKEND_STACK.md) | Java 21 + Spring Boot 3.3 + Maven 3.9 full implementation reference |

---

## Cross-Reference Index

### Business Rules → Functional Requirements

| Business Rule | Enforced By |
|---|---|
| BR001 (no double booking) | FR004, NFR007 |
| BR002 (cancellation window) | FR005 |
| BR004 (saga atomicity) | FR004, FR006 |
| BR005 (tenant isolation) | FR001, NFR012, security.json |
| BR006 (idempotency) | FR004 |
| BR007 (notifications decouple) | FR007, NFR002 |
| BR013 (event envelope) | NFR014 |

### Patterns → User Journeys

| Pattern | Used In |
|---|---|
| Saga Orchestration | UJ002 (booking) |
| Saga Choreography | UJ003 (cancellation) |
| CQRS + Materialized View | UJ005 (dashboard) |
| Dead Letter Queue + Replay | UJ006 (ops triage) |
| Pub/Sub + Outbox | UJ001 (onboarding) |

### SLOs → NFRs → Alerts

| SLO | NFR | Alert |
|---|---|---|
| SLO001 (booking p95 < 3s) | NFR004 | ALT001 |
| SLO002 (< 1% error rate) | NFR001 | ALT006 |
| SLO003 (availability < 500ms) | NFR005 | — |
| SLO004 (dashboard < 5s lag) | NFR006, NFR008 | ALT002 |
| SLO005 (notifications 99%) | — | ALT005 |
| SLO006 (outbox < 30s) | NFR003 | ALT002 |
