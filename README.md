# AnjiSchedulo

**Multi-tenant B2B SaaS appointment scheduling platform** for salons, clinics, fitness studios, and professional consultancies.

Built with Java 25 · Spring Boot 3.x · Next.js 14 · PostgreSQL 15 · AWS ECS Fargate · Event-Driven Microservices

---

## What is AnjiSchedulo?

AnjiSchedulo lets service businesses publish their availability online and let customers book appointments in real time — without phone calls or back-and-forth emails. Each business (tenant) gets their own branded booking page, staff calendar, and notification system.

**Key capabilities:**
- Real-time slot availability with conflict-free booking
- Online payment capture via Stripe at the time of booking
- Automated email + SMS notifications (booking confirmation, reminders, cancellations)
- Multi-staff scheduling with per-staff service assignments
- Tenant dashboard with live booking counts and revenue summaries
- GDPR-compliant customer data management with right-to-erasure support

---

## Pricing Tiers

| Plan | Staff | Bookings/month | Notifications | Database |
|---|---|---|---|---|
| **Starter** | Up to 3 | 200 | Email only | Shared cluster |
| **Pro** | Up to 20 | 2,000 | Email + SMS | Shared cluster |
| **Enterprise** | Unlimited | Unlimited | Email + SMS + WhatsApp | Dedicated cluster |

Enterprise also includes SSO (SAML 2.0 / OIDC), custom domain, and a dedicated KMS key.

---

## Architecture

AnjiSchedulo uses an **Event-Driven Microservices** architecture with CQRS and the Transactional Outbox pattern.

```
Customer Browser
    │ HTTPS (TLS 1.3)
    ▼
CloudFront → ALB → api-gateway
    │ JWT RS256 auth · per-tenant rate limiting
    ├──► booking-command-service   (saga orchestrator)
    ├──► availability-service      (slot lock via Redis)
    ├──► payment-service           (Stripe integration)
    ├──► tenant-service            (tenant provisioning)
    ├──► dashboard-service         (read projections)
    └──► ops-service               (operator tools)

Domain events flow via Transactional Outbox → SNS → SQS FIFO per consumer:
    ├──► notification-service      (SendGrid + Twilio)
    ├──► dashboard-service         (materialized view updates)
    └──► analytics-service         (aggregate summaries)
```

**Multi-tenancy:** Pool model with PostgreSQL Row-Level Security. All Starter/Pro tenants share a cluster; Enterprise tenants get a dedicated RDS instance.

**Core patterns:** Saga Orchestration (booking), Saga Choreography (cancellation), Idempotent Consumer, Event-Carried State Transfer, Transactional Outbox.

---

## Tech Stack

| Layer | Technology |
|---|---|
| Backend | Java 25 · Spring Boot 3.x |
| Frontend | Next.js 14 · React |
| Monorepo | Gradle 8 (multi-module) |
| Database | PostgreSQL 15 · PgBouncer · Flyway |
| Cache | Redis 7 (ElastiCache) · Lettuce |
| Message bus | AWS SNS + SQS FIFO |
| Compute | AWS ECS Fargate |
| Infrastructure | Terraform · AWS eu-west-1 |
| Observability | Grafana LGTM (Loki · Mimir · Tempo · Grafana) |
| CI/CD | GitHub Actions |

---

## Repository Structure

```
anji-schedulo/
├── apps/
│   ├── api-gateway/                  # JWT auth, routing, rate limiting
│   ├── booking-command-service/      # Booking saga orchestrator
│   ├── availability-service/         # Slot availability + Redis slot locks
│   ├── payment-service/              # Stripe integration
│   ├── tenant-service/               # Tenant provisioning + config
│   ├── notification-service/         # SendGrid + Twilio
│   ├── dashboard-service/            # Read projections + materialized views
│   ├── analytics-service/            # Analytics summaries
│   ├── ops-service/                  # Platform operator tools (DLQ, replay)
│   ├── outbox-relay/                 # Outbox → SNS publisher
│   └── web/                          # Next.js customer + tenant admin UI
├── shared/
│   ├── shared-events/                # Event envelope types, domain records
│   ├── shared-logging/               # SLF4J/Logback configuration
│   ├── shared-telemetry/             # OpenTelemetry Java agent config
│   ├── shared-db/                    # Base repository (RLS), HikariCP config
│   └── shared-security/              # JWT filter, TenantContext (ThreadLocal)
├── database/
│   ├── migrations/                   # Flyway SQL migration files
│   └── seeds/                        # Local dev seed data
├── infra/
│   ├── modules/                      # Reusable Terraform modules
│   └── environments/                 # dev / staging / production tfvars
├── specs/
│   ├── user/                         # Product, personas, requirements, journeys
│   └── ai/                           # Derived architecture specs
└── documentation/                    # Full documentation suite (see below)
```

---

## Documentation

The `documentation/` folder contains a complete, production-quality documentation suite generated from the structured specs in `specs/`.

| Layer | Folder | Contents |
|---|---|---|
| 00 — Governance | `00-governance/` | Project charter, RACI matrix, risk register, change log, definition of done |
| 01 — Requirements | `01-requirements/` | Glossary, stakeholder map, BRD, personas (P1–P4), flow registry, user journeys (UJ001–UJ007), PRD, use cases, NFRs, acceptance criteria, compliance (GDPR, PCI-DSS) |
| 02 — Design | `02-design/` | Data model, flow specs, Mermaid sequence diagrams, state machines, API design (OpenAPI), functional spec, error handling, DB schema (DDL + RLS), notifications, UI/UX spec, test strategy, user stories |
| 03 — Data | `03-data/` | Data dictionary, data flow diagram, seed data strategy |
| 04 — Architecture | `04-architecture/` | Tech stack, system architecture, multi-tenancy, security model, threat model (STRIDE), data privacy, infrastructure, scaling strategy, deployment, integrations, observability, disaster recovery · ADRs · cicd-flows · infra-flows · observability-flows · resilience-flows · secrets-flows |
| 05 — Developer Experience | `05-developer-experience/` | Local setup guide, coding standards, git workflow, PR review guide, system walkthrough, developer FAQ |
| 06 — Operations | `06-operations/` | Release plan, feature flags, runbook, incident response, secrets rotation, operations flows |
| 07 — Frontend Design | `07-frontend-design/` | Lo-fi wireframes, hi-fi designs, design tokens, component specs |

---

## Local Development

### Prerequisites

Java 25 JDK, Docker Desktop, AWS CLI. See [DX1 · Local Setup Guide](documentation/05-developer-experience/DX1-local-setup-guide.md) for the full setup.

### Quick start

```bash
# Start backing services (PostgreSQL, Redis, LocalStack, Grafana LGTM)
docker compose up -d

# Run database migrations
./gradlew flywayMigrate

# Seed demo data
./gradlew :shared:seed-data:run

# Start a service (repeat per service)
./gradlew :services:booking-command-service:bootRun
```

Services start on ports 4000–4009. The Next.js web app runs on port 3001. Grafana is at http://localhost:3000.

### Tests

```bash
./gradlew test                    # unit tests (all modules)
./gradlew integrationTest         # integration tests (Testcontainers)
./gradlew test jacocoTestReport   # unit tests + coverage report
```

---

## Key Design Decisions

**Why pool tenancy with RLS instead of schema-per-tenant?**  
Simpler operational model — one migration applies to all tenants atomically. RLS enforces isolation at the database level as a hard guarantee, not just application-level filtering. Enterprise tenants who need stronger isolation get a dedicated cluster. See [ADR-002](documentation/04-architecture/adrs/ADR-002-shared-database-rls.md).

**Why SNS + SQS instead of Kafka?**  
Managed service — no broker cluster to operate. SQS FIFO gives per-tenant ordering via `MessageGroupId = tenant_id`. At the expected scale (thousands of tenants, not millions of events/second), SNS/SQS is cost-effective and operationally simpler. See [ADR-003](documentation/04-architecture/adrs/ADR-003-sns-sqs-over-kafka.md).

**Why saga orchestration for booking but choreography for cancellation?**  
Booking requires strict ordering and compensation on failure (slot lock → payment → confirm). Cancellation participants (slot release, refund, notification) can fail independently — a failed refund must not block slot release. Choreography gives each participant autonomy. See [ADR-004](documentation/04-architecture/adrs/ADR-004-saga-pattern-selection.md).

---

## Owner

**Pubudu Anjalee Cooray**  
[github.com/anjalee-cooray](https://github.com/anjalee-cooray)
