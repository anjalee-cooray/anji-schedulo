---
title: Change Log
layer: 00-governance
status: current
lastUpdated: 2026-06-28
---

# G4 · Change Log

## Overview

This document is the authoritative record of all significant changes to the AnjiSchedulo project specification, architecture decisions, and documentation suite. Every entry is immutable — past entries must not be edited or deleted. New entries are appended at the bottom of the change log table in chronological order.

The change log captures changes at the specification level (what was decided and documented), not at the code level (use the git commit history for code changes). The git repository at `https://github.com/anjalee-cooray/anji-schedulo` is the complementary source of truth for implementation-level history.

---

## Change Types

| Type | Description |
|---|---|
| **Initial** | First version of a file, document set, or specification layer |
| **AI-Generated** | Specification content derived by an AI model from upstream user specs |
| **Documentation** | Documentation files generated or updated from approved specifications |
| **Architecture** | An Architecture Decision Record (ADR) added, updated, or superseded |
| **Feature** | A new feature added to scope, or an existing feature changed in scope |
| **Fix** | A correction to existing content — data error, broken reference, or factual inaccuracy |
| **Deprecation** | Removal of a feature from scope, or supersession of an existing document |

---

## Change Log

| Version | Date | Author | Type | Summary | Affected Documents |
|---|---|---|---|---|---|
| v1.0.0 | 2026-06-27 | Pubudu Anjalee Cooray | Initial | Initial specification complete — all 9 user-facing spec files defined: project metadata, product vision and pricing, 4 personas, 13 functional requirements, 14 business rules, 7 user journeys, domain glossary, 16 non-functional requirements, and 6-phase roadmap | `specs/user/metadata.json`, `specs/user/product.json`, `specs/user/personas.json`, `specs/user/functional-requirements.json`, `specs/user/business-rules.json`, `specs/user/user-journeys.json`, `specs/user/glossary.json`, `specs/user/non-functional-requirements.json`, `specs/user/roadmap.json` |
| v1.0.1 | 2026-06-27 | AI (DocBlueprint) | AI-Generated | AI-derived specification complete — all 8 AI spec files filled from user specs: domain model (18 entities including 2 aggregate roots, 8 domain entities, 8 infrastructure/read-model entities), 9 bounded contexts, 24 domain events, event-driven microservices architecture with 5 ADRs, infrastructure specification (AWS eu-west-1, PostgreSQL, Redis, SNS/SQS, S3), security model (JWT RS256, 4 RBAC roles, Pool with RLS), observability stack (Grafana LGTM, 6 SLIs, 4 SLOs, 12 alerts), operations specification (14 named DLQ queues, replay pattern, backup and DR policy) | `specs/ai/domain.json`, `specs/ai/bounded-contexts.json`, `specs/ai/events.json`, `specs/ai/architecture.json`, `specs/ai/infrastructure.json`, `specs/ai/security.json`, `specs/ai/observability.json`, `specs/ai/operations.json` |
| v1.0.2 | 2026-06-28 | AI (DocBlueprint) | Documentation | Full documentation suite generated across all 8 layers — 130 documents written. Layer 00: 5 governance documents (project charter, RACI, risk register, change log, definition of done). Layer 01: 25 requirements documents including 4 persona profiles (P1–P4) and 7 journey documents (UJ001–UJ007). Layer 02: 35 design documents including 7 flow specs (D2), 7 sequence diagrams (D3), 7 user story sets (D12), and 9 core design documents. Layer 03: 3 data documents. Layer 04: 27 architecture documents including 5 ADRs, 5 CI/CD flows, 5 infrastructure flows, 3 observability flows, 4 resilience flows, and 3 secrets flows. Layer 05: 6 developer experience documents. Layer 06: 19 operations documents. Layer 07: 5 frontend design documents (F1 lo-fi web+mobile, F2 design tokens, F3 hi-fi web+mobile, F4 component specs) | `documentation/**` |

---

## Upcoming Changes

The v1.0 feature set (Pro plan, SMS notifications, stream processing anomaly alerts, Grafana LGTM full stack, tenant event replay, data export, Enterprise plan) is planned but not yet specced at the time of this change log entry. When v1.0 spec work begins, the following changes will be recorded:

- A new spec version entry when v1.0 user specs are authored
- ADR updates if any v1.0 features require architecture changes
- New or updated documentation files for v1.0-specific journeys and flow specs

The change log will be updated with a new entry at the start of each spec authoring session.
