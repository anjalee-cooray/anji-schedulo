# REL-002 · RC Validation

**Project:** AnjiSchedulo  
**Last Updated:** 2026-06-28

---

## 1. Purpose

RC validation is the structured 24–48 hour period between cutting the release candidate and the Go/No-Go decision. It verifies the RC behaves correctly in staging before any commitment to a production deploy.

---

## 2. Validation Checklist

All items must be checked. Items marked ❌ block the Go/No-Go unless explicitly waived by the Engineering Lead.

### 2.1 Automated (run by CI on RC deploy)

| Check | Status |
|---|---|
| Unit tests: all pass | ✓ / ❌ |
| Integration tests (real DB): all pass | ✓ / ❌ |
| Cross-tenant isolation tests: all pass | ✓ / ❌ |
| E2E smoke tests (staging): all pass | ✓ / ❌ |
| Docker image vulnerability scan (ECR): no CRITICAL findings | ✓ / ❌ |
| Flyway migrations: applied cleanly to staging DB | ✓ / ❌ |
| All ECS services healthy after migration | ✓ / ❌ |

### 2.2 Manual — Core Booking Flow

| Check | Tester | Status |
|---|---|---|
| Create booking as a new customer (Starter tenant) | | ✓ / ❌ |
| Stripe payment captured correctly | | ✓ / ❌ |
| `appointment.confirmed` email notification received | | ✓ / ❌ |
| Booking appears in staff dashboard | | ✓ / ❌ |
| Cancel booking — slot released, refund processed | | ✓ / ❌ |
| `appointment.cancelled` email notification received | | ✓ / ❌ |

### 2.3 Manual — Multi-Tenant Isolation

| Check | Tester | Status |
|---|---|---|
| Tenant A cannot see Tenant B's bookings via API | | ✓ / ❌ |
| Staff login scoped to own tenant — no cross-tenant navigation | | ✓ / ❌ |
| `isolation_monitor_violations_total` metric = 0 throughout validation | | ✓ / ❌ |

### 2.4 Manual — Observability

| Check | Tester | Status |
|---|---|---|
| Booking saga traces visible in Grafana Tempo | | ✓ / ❌ |
| No unexpected ERROR-level logs in Loki | | ✓ / ❌ |
| DLQ depth = 0 throughout validation | | ✓ / ❌ |
| No spurious alerts firing in Grafana | | ✓ / ❌ |

### 2.5 Manual — Notifications (Pro tier)

| Check | Tester | Status |
|---|---|---|
| SMS notification delivered within 30 seconds of booking | | ✓ / ❌ |
| 24h reminder sent correctly | | ✓ / ❌ |

### 2.6 Migration Checks (if DB migrations are included)

| Check | Tester | Status |
|---|---|---|
| Migration is backward-compatible (old code reads new schema) | | ✓ / ❌ |
| Migration tested on a staging DB snapshot matching production size | | ✓ / ❌ |
| Migration runtime < 30 seconds (avoid locking) | | ✓ / ❌ |

---

## 3. Validation Period

| Release type | Minimum validation period |
|---|---|
| PATCH (bug fix only) | 12 hours |
| MINOR (new features) | 24 hours |
| MAJOR (breaking change) | 48 hours |

---

## 4. Blocking vs. Non-Blocking Issues

| Issue type | Action |
|---|---|
| Automated test failure | Blocking — fix on RC branch, re-deploy, restart validation |
| Cross-tenant isolation failure | Blocking — P1, escalate to Engineering Lead |
| Manual smoke test failure (booking flow) | Blocking |
| Minor UI cosmetic issue (no UX impact) | Non-blocking — log as issue, ship with release |

---

## 5. Next Step

When all blocking checks pass, proceed to [REL-003 · Go/No-Go Decision](REL-003-go-no-go.md).

---

## 6. Traceability

| Concern | Doc |
|---|---|
| RC cut | REL-001-rc-cut.md |
| Go/No-Go | REL-003-go-no-go.md |
| Cross-tenant isolation testing | A3-multi-tenancy.md |
| DLQ checks | O4-runbook.md |
