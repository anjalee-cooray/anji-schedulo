---
title: Definition of Done
layer: 00-governance
status: current
lastUpdated: 2026-06-28
---

# G5 · Definition of Done

## Purpose

The Definition of Done (DoD) defines the criteria that must be satisfied before any work item — user story, feature, release, or documentation — is considered complete. All contributors, including AI-assisted authoring tools, must apply this checklist. A work item that does not satisfy the relevant DoD level is not done; it is in progress.

The DoD has three levels: **Story Done**, **Feature Done**, and **Release Done**. Each level is a strict superset of the one below it. Documentation-only work items use the **Documentation Done** checklist instead of the code-focused levels.

---

## Story Done

A user story is done when all of the following are true:

- [ ] All acceptance criteria stated in the user story are met and verified
- [ ] Code reviewed by at least one other contributor; for solo development, a self-review with explicit written sign-off is recorded in the pull request
- [ ] Unit tests written and passing for all new business logic paths
- [ ] Integration tests written for all new API endpoints and event handlers
- [ ] **Cross-tenant isolation test written** if the story touches any tenant-scoped data access path (BR005)
- [ ] **InboxRecord deduplication tested** if the story involves consuming an event from SQS (NFR009) — the test must confirm that a second delivery of the same `event_id` produces no duplicate side effects
- [ ] **Idempotency test written** if the story involves a booking or payment operation — the test must confirm that a duplicate request with the same `idempotency_key` returns the original result without reprocessing (BR006)
- [ ] **AuditLog entry verified** for all write operations that modify tenant-scoped data — the test must assert that an `audit_log` row is created with the correct `entity_type`, `entity_id`, `operation`, `actor_role`, and `occurred_at` (BR014)
- [ ] No hardcoded `tenant_id`, credentials, secrets, or PII appear in code or test fixtures
- [ ] New API endpoint documented: path, method, authentication requirement, request body schema, and response schema added to `documentation/02-design/D5-api-design.md`
- [ ] Error responses produced by the story match the error catalogue in `documentation/02-design/D7-error-handling-spec.md`
- [ ] No new compiler warnings introduced; existing warnings not increased
- [ ] All existing tests pass (no regression)

---

## Feature Done

A feature is done when all constituent user stories satisfy Story Done, plus:

- [ ] End-to-end test written and passing for the full feature happy path (from HTTP request to database state to event publication)
- [ ] End-to-end test written for at least one critical failure path (e.g. slot unavailable → 409, payment declined → 402, outside cancellation window → 422)
- [ ] Performance test confirms p95 latency targets are met for all endpoints affected by the feature:
  - Availability query: < 500 ms (FR003, NFR005)
  - Booking saga end-to-end: < 2 seconds (FR004)
  - Dashboard response: < 200 ms (FR008, NFR006)
- [ ] Sequence diagram (D3) for the relevant journey reviewed — if the implementation differs from the spec, the sequence diagram is updated before the feature is marked done
- [ ] Flow spec (D2) for the relevant journey reviewed for accuracy against the implementation
- [ ] Feature flag applied if the risk of regression to existing functionality is assessed as high
- [ ] Runbook updated if the feature introduces new operational concerns: a new DLQ queue, a new retry pattern, a new alert condition, or a new manual remediation step
- [ ] Observability coverage confirmed: a metric, a structured log line, and a trace span exist for the critical path of the feature; no dark code paths
- [ ] Grafana dashboard updated if the feature introduces a new metric that operators need to monitor

---

## Release Done

A release is done when all features in the release satisfy Feature Done, plus:

- [ ] All user-facing strings reviewed: no lorem ipsum, no placeholder text, no TODO comments in any UI component or email template
- [ ] Security review completed for any new authentication flows, new data access patterns, or new external integrations
- [ ] GDPR review completed: any new PII fields are documented in `documentation/02-design/D6-functional-spec.md` and `documentation/03-data/DM1-data-dictionary.md`, with pseudonymisation and erasure strategy confirmed
- [ ] Database migration scripts tested against a full copy of production data (or the most recent staging snapshot if production data is not available)
- [ ] Migration is backwards-compatible: no breaking schema changes applied without a documented migration plan and a rollback script
- [ ] Release notes authored at `documentation/06-operations/version-flows/VER-002-changelog-release-notes.md`
- [ ] Go / No-Go review completed and recorded at `documentation/06-operations/release-flows/REL-003-go-no-go.md`
- [ ] Rollback plan confirmed and tested: the rollback procedure at `documentation/06-operations/O3-rollback-plan.md` has been rehearsed on the staging environment
- [ ] All DLQ queues confirmed empty immediately before the release window opens
- [ ] Tenant-facing communication sent if the release changes any tenant-facing behaviour (booking flow, dashboard, notification content, or pricing): communication plan at `documentation/06-operations/comms-flows/COM-001-internal-release-comms.md`

---

## Documentation Done

For documentation-only work items (spec authoring, document generation, diagram updates):

- [ ] All content is derived from approved specifications in `specs/` — no data invented that is not traceable to a spec field
- [ ] All IDs used in the document exist in the source specs: entity IDs from `domain.json`, persona IDs from `personas.json`, FR/BR/NFR IDs from their respective spec files
- [ ] No placeholder content, `TODO`, `TBD`, or `[FILL IN]` markers appear anywhere in the document body
- [ ] Frontmatter is complete: `title`, `layer`, `status`, and `lastUpdated` fields are all present and correct
- [ ] Per-journey files use the journey ID in the filename: `R6-journey-UJ001.md`, `D2-flow-spec-UJ001.md`, `D3-sequence-UJ001.md`, `D12-user-stories-UJ001.md`
- [ ] Per-persona files use the persona ID in the filename: `R4a-persona-P1.md`
- [ ] All internal cross-references (relative links to other documents) resolve to files that exist in the repository
- [ ] All Mermaid diagrams have been validated in a Mermaid previewer (e.g. mermaid.live) and render without syntax errors

---

## Non-Negotiable Rules

The following rules apply to every work item at every level. They override the DoD checklists in any edge case and cannot be waived by any team member or contributor:

**BR001 — No double-booking.**
No code path may confirm an appointment into a slot that is already reserved. Double-booking prevention must be enforced at the PostgreSQL unique partial index on `slot_locks(tenant_id, staff_id, slot_date, slot_start) WHERE released_at IS NULL`. Application-level checks are complementary, not sufficient.

**BR005 — Absolute tenant data isolation.**
Tenant data must never be readable by another tenant under any condition, including infrastructure failure or application bug. Every feature that touches tenant-scoped data must be accompanied by a cross-tenant isolation test. The safe-by-default RLS policy (zero rows when tenant context is absent) must never be relaxed.

**BR006 — Idempotent booking operations.**
A duplicate booking request carrying the same `idempotency_key` must return the original result without creating a second appointment, charging the card a second time, or emitting a second confirmation. This is enforced at the `booking_idempotency` table with a unique constraint.

**BR007 — Notification failures are fully isolated.**
A notification delivery failure must never block, delay, or roll back the booking operation that triggered it. The notification pipeline is decoupled from all booking and payment operations. This isolation must be preserved in every implementation decision.

**BR013 — Complete event envelopes.**
Every event published to SNS must carry `tenant_id`, `event_id`, `correlation_id`, and `occurred_at`. An event missing any of these fields must be routed to the DLQ rather than processed. The `EventEnvelope` validator must run in every consumer before any business logic is executed.

**BR014 — Immutable audit trail.**
Every write operation that modifies tenant-scoped data must produce an immutable `audit_log` entry. The `booking_app_user` database role has `INSERT` privilege on `audit_log` and no `UPDATE` or `DELETE` privilege. This constraint must be preserved in every database migration and schema change.
