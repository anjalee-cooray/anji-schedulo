# DX4 · PR Review Guide

**Project:** AnjiSchedulo  
**Version:** 1.0.0  
**Last Updated:** 2026-06-28

---

## 1. PR Checklist — Author

Before marking a PR ready for review, confirm:

### Code quality
- [ ] `npm run lint` passes with zero errors
- [ ] `npm run type-check` passes with zero errors
- [ ] No `any`, `@ts-ignore`, `console.log`, or commented-out code
- [ ] No secrets, API keys, or PII in code, logs, or tests

### Tests
- [ ] Unit tests written for new business logic
- [ ] Business rules covered by test name (e.g. "throws when monthly limit exceeded (BR003)")
- [ ] Integration tests cover any new saga step or event flow
- [ ] If new tenant-scoped table: cross-tenant isolation test added
- [ ] `npm run test` passes locally

### Data and migrations
- [ ] If schema change: Flyway migration file included (`V{version}__{description}.sql`)
- [ ] Migration is forward-only (no destructive `DROP` without prior data migration step)
- [ ] New tables include RLS policy and tenant_id index
- [ ] Rollback plan described in PR description

### Events and outbox
- [ ] New event types use the `EventEnvelope` type with all mandatory fields
- [ ] Event envelope test added if new event type
- [ ] Outbox record written in same DB transaction as domain write

### PR description
- [ ] Summary explains what changed and why (not just what)
- [ ] Linear ticket linked
- [ ] Screenshots/logs for UI or behaviour changes

---

## 2. PR Size Guidelines

Keep PRs small and focused. Large PRs are harder to review and more likely to introduce bugs.

| PR type | Ideal size |
|---|---|
| Feature PR | < 400 lines changed |
| Bug fix | < 100 lines changed |
| Refactor | < 300 lines changed |
| Migration | Schema file + tests only |
| Chore (deps, tooling) | Isolated from feature changes |

If a PR is growing large, split it:
1. Infrastructure/schema change PR (merged first)
2. Feature implementation PR (depends on schema PR)
3. Tests and cleanup PR (optional)

---

## 3. Reviewer Checklist

When reviewing a PR, check these areas in order:

### Architecture and design

- [ ] Does this change respect the bounded context boundaries? (No direct DB calls to another service's tables)
- [ ] Does business logic live in the service layer, not the controller or repository?
- [ ] Does the change need to be in the outbox for durability, or is it fire-and-forget acceptable?
- [ ] If adding a new event: does the envelope include all mandatory fields (`tenant_id`, `correlation_id`, `occurred_at`, etc.)?

### Tenant isolation

- [ ] Does every new query scope by `tenant_id`?
- [ ] Is there a corresponding RLS policy for new tables?
- [ ] Is the cross-tenant isolation test present and meaningful?

```typescript
// Look for this pattern in integration tests:
it('returns 404 when tenant_A reads tenant_B data', ...)
it('returns zero rows when app.tenant_id is not set', ...)
```

### Business rule compliance

- [ ] Are the relevant BRs referenced in test names or comments?
- [ ] Is idempotency handled for operations that could be retried? (BR006)
- [ ] If payment is involved: does the saga compensate on failure? (BR004, BR009)
- [ ] If audit-required: is an `audit_log` entry written in the same transaction? (BR014)

### Security

- [ ] No plaintext secrets in code or tests
- [ ] No PII in log messages (check for email, phone, name in log calls)
- [ ] SQL uses parameterised queries (`$1`, `$2`), not string interpolation
- [ ] New endpoints protected by JWT auth (check `@UseGuards(JwtAuthGuard)`)

### Observability

- [ ] New code paths emit structured logs with `tenant_id` and `correlation_id`
- [ ] New saga steps are traced with span attributes (`appointment_id`, `saga_step`)
- [ ] New failure modes produce logs at `error` level with enough context to debug

### Testing

- [ ] Tests describe behaviour, not implementation
- [ ] Business rules referenced in failing-case test names
- [ ] No `expect(true).toBe(true)` or other vacuous assertions

---

## 4. Review Comment Labels

Use these prefixes to communicate comment intent:

| Label | Meaning |
|---|---|
| `[blocking]` | Must be fixed before merge |
| `[suggestion]` | Take it or leave it — non-blocking improvement |
| `[question]` | Clarification needed — may become blocking |
| `[nit]` | Minor style/formatting — non-blocking |
| `[praise]` | Good approach — no action needed |

Example:

```
[blocking] This query doesn't scope by tenant_id. Without this, RLS is the
only guard — but we enforce defence-in-depth at the application layer too (BR005).

[suggestion] Consider extracting this logic into a TenantQuotaService — it's
now duplicated in three places.

[nit] Prefer `const` over `let` here since the value is never reassigned.
```

---

## 5. Approval Requirements

| Branch target | Required approvals | Required CI |
|---|---|---|
| `feature/*` → `release/*` | 1 reviewer | PR pipeline (CD-001) |
| `release/*` → `main` | 2 reviewers (incl. Engineering Lead) | PR pipeline + staging deploy (CD-002) |
| `hotfix/*` → `main` | 1 reviewer + Engineering Lead notification | PR pipeline |

Stale reviews are dismissed when new commits are pushed. The author must re-request reviews after addressing blocking comments.

---

## 6. Merge Strategy

All PRs are **squash-merged** into the target branch. The PR title becomes the single commit message on `main`. Write PR titles in Conventional Commits format:

```
feat(booking): add reschedule flow with saga orchestration
fix(availability): correct slot boundary calculation for DST transitions
chore: upgrade @nestjs/core to 10.3.9
```

After merging, delete the feature branch. GitHub is configured to auto-delete branches on merge.

---

## 7. When to Request a Separate Review

Request an additional review from a specialist when:

| Change | Specialist |
|---|---|
| Flyway migration or schema change | DBA or senior backend engineer |
| Security-sensitive change (auth, JWT, tenant isolation) | Engineering Lead |
| Infrastructure/Terraform change | Infrastructure engineer |
| Payment flow (Stripe) | Engineering Lead |
| GDPR erasure or PII handling | Engineering Lead |

Tag the specialist with `@mention` in the PR description, not just a review request.

---

## 8. Turnaround Expectations

| PR type | First review within | Resolution within |
|---|---|---|
| Bug fix | 4 business hours | 1 business day |
| Feature PR | 1 business day | 2 business days |
| Release PR | 4 business hours | 4 business hours |
| Hotfix | 1 hour (P1) | 2 hours (P1) |

If a PR has been waiting more than 1 business day without review, ping the team in Slack #engineering.
