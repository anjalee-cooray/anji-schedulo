# ADR-002: Multi-Tenancy Data Isolation Strategy

**Status:** Proposed | Accepted | Deprecated  
**Date:** YYYY-MM-DD  
**Deciders:** Architecture Team

---

## Context

[Describe your multi-tenancy requirements. How many tenants are expected at launch and at scale? What data isolation guarantees must you provide? What are the cost constraints?]

Three standard multi-tenancy models were evaluated:

| Model | Description |
|---|---|
| **Silo** | Separate database cluster per tenant |
| **Bridge** | Shared cluster, separate schema per tenant |
| **Pool** | Shared schema, `tenant_id` column on every table, enforced via RLS |

---

## Decision

[State your chosen model and any exceptions — e.g. Pool with RLS as default, dedicated cluster for enterprise tenants.]

---

## Rationale

### Why not Silo for all tenants?

[Explain why dedicated infrastructure per tenant is not viable at your scale — typically cost and operational overhead at low tenant counts.]

### Why not Bridge (separate schema per tenant)?

[Explain the drawbacks of schema-per-tenant for your stack — e.g. connection pooling issues, system catalogue bloat, migration complexity.]

### Why your chosen model?

[Explain the specific benefits — e.g. defence in depth at the DB engine level, safe-by-default behaviour on missing context, cost efficiency at your expected tenant count.]

### Enterprise escape hatch

[If applicable, describe how higher-tier tenants get stronger isolation — e.g. dedicated cluster with the same application code and a different connection string.]

---

## Consequences

**Positive:**
- [List benefits — e.g. zero provisioning per new tenant, engine-level isolation]

**Negative:**
- [List trade-offs — e.g. shared cluster noisy-neighbour risk, every table must carry tenant_id]

**Mitigations:**
- [How you address each negative — e.g. isolation test suite in CI, continuous isolation monitor in production]
