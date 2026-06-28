# VER-001 · Major Version Planning

**Project:** AnjiSchedulo  
**Last Updated:** 2026-06-28

---

## 1. Purpose

A major version increment (e.g. v1.x.x → v2.0.0) signals breaking changes to tenants and integrations. Major versions require a longer planning horizon, formal deprecation periods, and explicit tenant communication. This document defines the process for planning and executing a major version release.

---

## 2. What Constitutes a Major Version

A major version is required when any of the following are true:

| Change | Examples |
|---|---|
| Breaking API change | Removing a field, changing a response schema, removing an endpoint |
| Breaking tenant data model change | Renaming tenant-visible identifiers, changing booking ID format |
| Pricing tier restructure | Removing or merging existing tiers |
| Incompatible event schema change | Changing `appointment.confirmed` payload structure |
| Auth model change | Changing JWT claim names or token format |

Non-breaking additions (new fields, new endpoints, new event consumers) do NOT require a major version.

---

## 3. Major Version Planning Timeline

| Phase | Duration | Activities |
|---|---|---|
| Proposal | 2 weeks | Engineering Lead drafts breaking change proposal; review with team |
| Deprecation notice | 30 days minimum | Notify tenants of deprecated APIs/fields via email; update API docs |
| Parallel operation | 30 days | Old and new API versions run concurrently (API versioning via `Accept-Version` header or URL prefix) |
| Migration support | 30 days | Tooling or migration guide provided to tenants |
| Old version sunset | End of parallel period | Old API version removed; old event schema deprecated |

For MAJOR releases tied to the roadmap (e.g. v2.0.0 — Marketplace), the parallel operation period may extend to 90 days based on tenant contract obligations.

---

## 4. API Versioning Strategy

AnjiSchedulo uses URL-prefix versioning:

```
/v1/appointments/{id}    ← current
/v2/appointments/{id}    ← new major version
```

During the parallel operation period:
- `api-gateway` routes `v1` requests to v1 service images (pinned ECS task revision)
- `api-gateway` routes `v2` requests to current ECS tasks
- Both versions share the same database — the application layer handles schema compatibility

---

## 5. Breaking Change Checklist

Before merging any breaking change:

- [ ] ADR written and approved documenting the decision and why it is breaking
- [ ] Deprecation notice sent to affected tenants (template in COM-001)
- [ ] Old API/field marked as deprecated in OpenAPI spec (`deprecated: true`)
- [ ] Migration guide published in developer documentation
- [ ] VER-003 (API Deprecation) process followed for each deprecated endpoint
- [ ] Parallel operation date range confirmed
- [ ] Old version sunset date communicated and agreed with tenants

---

## 6. Roadmap Major Versions

| Version | Theme | Breaking Changes | Target |
|---|---|---|---|
| v2.0.0 | Marketplace + Recurring Appointments | Tenant hierarchy (parent/child tenants), appointment recurrence model | Roadmap Phase 4 |

---

## 7. Traceability

| Concern | Doc |
|---|---|
| API deprecation | VER-003-api-deprecation.md |
| Changelog | VER-002-changelog-release-notes.md |
| Tenant communication | COM-001-internal-release-comms.md |
| ADR records | documentation/04-architecture/adrs/ |
| Roadmap | specs/user/roadmap.json |
