# ADR-002: Shared Database with Row-Level Security (Pool Model)

**Status:** Accepted  
**Date:** 2026-06-27  
**Deciders:** Architecture Team

---

## Context

BookVault is a multi-tenant SaaS platform. Every piece of data belongs to exactly one tenant. We need a data isolation strategy that:
- Guarantees no tenant can read or write another tenant's data
- Is cost-efficient at the scale of Starter and Pro plan tenants
- Supports Enterprise tenants requiring stronger guarantees
- Is enforceable at the database layer (not just application code)

Three standard multi-tenancy models were evaluated:

| Model | Description |
|---|---|
| **Silo** | Separate database cluster per tenant |
| **Bridge** | Shared cluster, separate schema per tenant |
| **Pool** | Shared schema, `tenant_id` column on every table, enforced via RLS |

---

## Decision

We adopt the **Pool model with PostgreSQL Row-Level Security (RLS)** as the default for Starter and Pro tenants. Enterprise tenants receive a **dedicated RDS cluster** (Silo model) with the same application code and a different connection string.

---

## Rationale

### Why not Silo for all tenants?

At MVP launch targeting dozens of tenants, provisioning and maintaining a dedicated RDS cluster per tenant is operationally unsustainable and cost-prohibitive (each Multi-AZ RDS instance costs ~$100–$300/month). Starter plan pricing would be unviable.

### Why not Bridge (separate schema per tenant)?

- PostgreSQL schema-per-tenant requires runtime `SET search_path` — connection pooling (PgBouncer/HikariCP) does not hold schema state reliably across borrowed connections
- Each schema multiplication (pg_class, pg_attribute) bloats the system catalogue, degrading planning performance at hundreds of tenants
- Migrations must be run per schema, adding operational complexity

### Why Pool + RLS?

- **Defence in depth**: RLS is enforced at the database engine level. Even if application code has a bug and omits a `WHERE tenant_id = ?` clause, the RLS policy prevents data leakage
- **Safe default**: `SET LOCAL app.current_tenant_id = ''` (missing context) causes `current_setting('app.current_tenant_id', true)` to return NULL, and RLS evaluates the policy to FALSE — returning zero rows, never all rows
- **Connection pool friendly**: `SET LOCAL` is transaction-scoped; the setting resets automatically when the transaction ends, regardless of connection reuse
- **Cost efficient**: All tenants share one RDS cluster until the 200+ tenant threshold where a read replica becomes justified
- **Single migration path**: Flyway runs one migration against the shared cluster (plus separately against each enterprise dedicated cluster)

### Enterprise escape hatch

Enterprise tenants with contractual data residency, regulatory, or security requirements get a dedicated RDS cluster. The application routes to the correct cluster via tenant_id lookup in TenantContext. No code change required — only connection string configuration.

---

## Consequences

**Positive:**
- Zero infrastructure provisioning per new Starter/Pro tenant
- Database isolation guaranteed at engine level, independent of application bugs
- Cost scales with usage, not tenant count, up to ~200 tenants

**Negative:**
- Shared cluster is a noisy-neighbour risk at high tenant count (mitigated by per-tenant query quotas and read replicas)
- All service schemas must carry `tenant_id` on every table — no exceptions
- RLS requires explicit `SET LOCAL` in every transaction — a missing call is a security bug, not a correctness bug (mitigated by integration tests that assert cross-tenant isolation)

**Mitigations:**
- Isolation test suite (12 test cases) runs in CI on every PR
- Continuous isolation monitor queries the DB every 5 minutes in production
- `booking_app_user` role is never granted `BYPASSRLS` or superuser
- Code review checklist includes: every new table has RLS policy, every query passes tenant context
