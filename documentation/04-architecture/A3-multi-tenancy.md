---
title: Multi-Tenancy Architecture
layer: 04-architecture
status: current
lastUpdated: 2026-06-28
---

# A3 · Multi-Tenancy Architecture

## Strategy

AnjiSchedulo uses **Pool multi-tenancy with PostgreSQL Row-Level Security (RLS)** for Starter and Pro tiers. Enterprise tenants receive a dedicated PostgreSQL cluster. The strategy is governed by ADR002.

---

## Tenant Isolation Model

### Starter and Pro — Shared Cluster with RLS

All Starter and Pro tenants share a single RDS PostgreSQL 15 cluster. Isolation is enforced at the database layer via Row-Level Security, not at the schema or cluster level.

**How it works:**

1. Every table that stores tenant-scoped data has a `tenant_id UUID NOT NULL` column.
2. Every table has an RLS policy of the form:
   ```sql
   CREATE POLICY tenant_isolation ON <table>
     USING (tenant_id = current_setting('app.tenant_id')::uuid);
   ```
3. The application database role (`booking_app_user`) has `FORCE ROW LEVEL SECURITY` enabled — RLS applies even when the role owns the table.
4. At the start of every database connection, the service calls:
   ```sql
   SET LOCAL app.tenant_id = '<tenant_uuid>';
   ```
   This scopes every subsequent query in that transaction to the calling tenant.
5. **Safe default:** If `app.tenant_id` is not set (null or missing), `current_setting('app.tenant_id')` returns null. The `= null` comparison evaluates to `UNKNOWN` in SQL, which filters to zero rows. **A missing tenant context never returns all rows** (BR005).

### Enterprise — Dedicated Cluster

Enterprise tenants receive a dedicated RDS PostgreSQL 15 cluster. This provides:
- No resource contention with other tenants (CPU, IOPS, connections).
- Schema-level separation — Enterprise databases contain only the Enterprise tenant's data.
- Separate credentials per Enterprise tenant.
- The same RLS policies are applied as in the shared cluster (defence in depth).

The dedicated cluster is provisioned by the platform operator via the ops-service provisioning workflow, documented in O5-tenant-management.md.

---

## Enforcement Layers

Tenant isolation is enforced at three independent layers. All three must be intact; compromise of any one layer is not sufficient to cause a cross-tenant data breach.

### Layer 1 — JWT Claim (api-gateway)

Every authenticated request to the platform must carry a JWT token signed by api-gateway's RS256 private key. The token includes a mandatory `tenant_id` claim:

```json
{
  "sub": "<user_id>",
  "tenant_id": "<tenant_uuid>",
  "role": "tenant_admin | staff | customer | platform_operator",
  "iat": ...,
  "exp": ...
}
```

api-gateway validates the JWT on every inbound request. It forwards `tenant_id` to downstream services via a trusted `X-Tenant-ID` header. Services must not accept `tenant_id` from the request body or query string — only from this trusted header (BR005).

**Public endpoints** (booking page availability query, registration) carry no JWT. These endpoints use only the `tenant_id` embedded in the public URL (the tenant slug), and are limited to read-only queries scoped to that tenant. No write operations are permitted on public endpoints.

### Layer 2 — Application Filter (service layer)

Each Spring Boot service uses a `TenantContextFilter` that:
1. Extracts `tenant_id` from the `X-Tenant-ID` header injected by api-gateway.
2. Validates that it is a well-formed UUID.
3. Stores it in a `ThreadLocal`-backed `TenantContext` holder.
4. Passes it to every PostgreSQL transaction as `SET LOCAL app.tenant_id`.

No service layer query is issued without first establishing the tenant context. If `tenant_id` is missing or malformed, the service returns `403 Forbidden` before any query executes.

### Layer 3 — Database RLS (PostgreSQL)

As described above. This layer is the last line of defence. Even if the application layer fails to set the tenant context correctly, the RLS policy prevents cross-tenant reads from returning data. The `booking_app_user` role cannot bypass RLS — it is not the table owner and does not have `BYPASSRLS` privilege.

---

## Tables Protected by RLS

Every tenant-scoped table has an RLS policy. The full list of tables:

| Table | Tenant Column | RLS Policy |
|---|---|---|
| tenants | tenant_id | Own row only |
| tenant_config | tenant_id | Own rows only |
| staff_members | tenant_id | Own rows only |
| services | tenant_id | Own rows only |
| working_hours | tenant_id | Own rows only |
| appointments | tenant_id | Own rows only |
| appointment_events | tenant_id | Own rows only |
| slot_locks | tenant_id | Own rows only |
| customers | tenant_id | Own rows only |
| payments | tenant_id | Own rows only |
| notification_records | tenant_id | Own rows only |
| outbox_records | tenant_id | Own rows only |
| inbox_records | tenant_id | Own rows only |
| audit_log | tenant_id | Own rows only (INSERT-only privilege for booking_app_user) |
| tenant_dashboard_views | tenant_id | Own rows only |
| booking_idempotency_keys | tenant_id | Own rows only |
| replay_jobs | tenant_id | Own rows only |

Tables without `tenant_id` (e.g. infrastructure tables without tenant affinity) are accessible only to specific service roles, not to `booking_app_user`.

---

## Tenant Provisioning

When a new tenant is provisioned (UJ001), tenant-service:

1. Generates a `tenant_id` UUID v4.
2. Creates the `tenants` row with this `tenant_id`.
3. All subsequent rows created by the tenant inherit this `tenant_id` in every write.
4. For Enterprise tenants: ops-service triggers dedicated cluster provisioning. The tenant's data is never written to the shared cluster.

Tenant provisioning is an atomic transaction (Transactional Outbox) — the `tenant_id` is established in the same transaction that creates the tenant record, ensuring there is never a state where a tenant exists without a `tenant_id`.

---

## Safe Default Guarantee

The safe default (BR005) is tested in CI at every merge to main:

```sql
-- Test: missing tenant context returns zero rows
BEGIN;
-- Do not SET LOCAL app.tenant_id
SELECT COUNT(*) FROM appointments;
-- Expected: 0 (not the total count of all appointments)
ROLLBACK;
```

This test is run against a populated test database with multiple test tenants. If it returns any rows, the pipeline fails.

---

## Cross-Tenant Access (platform_operator role only)

Platform Operators (P4) have a special `platform_operator` role that is authorised to perform cross-tenant operations for support, DLQ triage, and deprovisioning. Cross-tenant access is implemented by:

1. The platform operator's JWT carries `role: platform_operator` and no `tenant_id` claim.
2. A separate application role (`booking_ops_user`) is used for operator queries. This role has `BYPASSRLS` privilege.
3. All operator queries are logged to the audit_log with `actor_role = platform_operator` and the target `tenant_id` explicitly recorded.
4. Operator tools (ops-service) always require explicit `tenant_id` input — there is no "query all tenants" operation available in the UI.

---

## Tier Limits Enforcement

Plan limits (BR001, BR002) are enforced in the application layer, not the database layer:

| Limit | Starter | Pro | Enterprise |
|---|---|---|---|
| Staff members | 3 | 20 | Unlimited |
| Monthly bookings | 200 | 2,000 | Unlimited |
| Notification channels | Email only | Email + SMS | Email + SMS + WhatsApp |
| Infrastructure | Shared cluster | Shared cluster | Dedicated cluster |

Before any write that would exceed a plan limit, tenant-service checks the current count against the plan's maximum. Exceeding the limit returns `403 Forbidden` with a `PLAN_LIMIT_EXCEEDED` error code and a prompt to upgrade (BR001, BR002).

Limits are cached in Redis (key: `tenant_config:{tenant_id}`) with a 5-minute TTL. The cache is invalidated on `tenant.configured` events.

---

## Data Residency

All data is stored in `eu-west-1` (Ireland). There is no cross-region replication of tenant data for GDPR compliance. Disaster recovery failover uses a cross-region read replica in `eu-central-1` (Frankfurt) for RTO recovery, but this replica is not used for normal reads and data does not leave the EU region at rest.
