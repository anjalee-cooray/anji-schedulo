# docblueprint-engine — project template

Generate a complete suite of project documents from structured spec files using Claude Code.

No CLI. No API key. No install step.

---

## How it works

Specs are split into two categories:

| Category | Folder | Who fills it |
|---|---|---|
| **User specs** | `specs/user/` | You — product knowledge only you have |
| **AI specs** | `specs/ai/` | Claude Code — derived from your user specs |

Once both categories are complete and reviewed, Claude Code generates all documents in `project-docs/`.

---

## Steps to run

### Step 1 — Install Claude Code

```bash
npm install -g @anthropic/claude-code
```

### Step 2 — Log in to Claude Code

```bash
claude login
```

This opens a browser to authenticate with your Anthropic account. Complete the login and return to the terminal.

### Step 3 — Clone this repo

```bash
git clone https://github.com/anjalee-cooray/docblueprint-engine my-project
cd my-project
```

### Step 4 — Fill in the user specs

Open every file in `specs/user/` and replace the placeholder values with your project's real details:

| File | What to fill in |
|---|---|
| `metadata.json` | Project name, owner, version, repo URL, authors |
| `product.json` | Vision, problem statement, target market, pricing tiers, out of scope |
| `personas.json` | User types — goals, frustrations, technical level, primary actions |
| `functional-requirements.json` | What the product must do, with clear acceptance criteria |
| `business-rules.json` | Non-negotiable rules the system must always enforce |
| `user-journeys.json` | Critical user flows described from the user's perspective |
| `glossary.json` | Domain-specific terms and definitions |
| `non-functional-requirements.json` | Performance, availability, and consistency targets |
| `roadmap.json` | Phases, features per phase, what is deferred |

**Do not touch `specs/ai/`** — Claude Code fills those automatically.

### Step 5 — Open the project folder in Claude Code

```bash
claude .
```

Claude Code reads `CLAUDE.md` as context when the folder opens. It then waits for you to start the session.

### Step 6 — Trigger the workflow

Once Claude Code is open, type:

```
start
```

Claude Code will validate your `specs/user/` files and begin Phase 1.

### Step 7 — Claude Code fills the AI specs

Claude Code reads your `specs/user/` files and derives the technical specs in `specs/ai/`. Each file is filled to production depth — not generic summaries:

| File | What Claude Code derives |
|---|---|
| `domain.json` | Every aggregate root, entity, read model, and infrastructure entity — with lifecycle states, key attributes (including `tenant_id`), and pattern notes citing your BR/NFR IDs |
| `bounded-contexts.json` | Service boundaries, published and subscribed events with reasons, named external dependencies (e.g. `Stripe (payment capture, refund APIs)`), and an explicit Outbox Relay context if the architecture uses a Transactional Outbox |
| `events.json` | Every domain event with producer, consumers, trigger (referencing your user journey steps and FR/BR IDs), delivery guarantee, idempotency strategy, transport topology, and full envelope fields including `tenant_id`, `correlation_id`, `causation_id` |
| `architecture.json` | Architecture style, per-pattern usage tied to NFR/BR IDs, per-service responsibility descriptions, and complete ADRs (context, decision, consequences) for every irreversible decision |
| `infrastructure.json` | Full cloud stack differentiated per pricing tier — shared vs. dedicated resources, per-bucket S3 purposes and lifecycle policies, secrets rotation periods, cache TTLs, message bus topology with DLQ configuration |
| `security.json` | JWT claims (required and optional), full public endpoint list, Enterprise SSO strategy, RBAC roles with permissions per persona, RLS-based tenant isolation layers with safe-default behaviour, PII erasure strategy, and CI test strategy for cross-tenant isolation |
| `observability.json` | Full observability stack, SLI metric queries in actual PromQL, mandatory log fields, redacted PII fields list, trace sampling strategy by endpoint type (100% for critical paths, 10% for reads), named dashboards, and alerting thresholds as real query expressions |
| `operations.json` | Every DLQ queue named explicitly per consumer service, replay scope options with job state tracking, cross-region backup strategy with RPO/RTO tied to NFR IDs, and per-severity incident runbook locations |
| `apis.json` | API surface for each service — paths, methods, auth requirements, request/response shapes |

### Step 8 — Review the AI specs

Claude Code prints a summary of everything it filled. Read through `specs/ai/`. If anything needs correcting, tell Claude Code in plain language:

```
"The architecture should use a modular monolith not microservices —
update architecture.json and re-derive bounded-contexts.json."
```

Claude Code applies the correction and updates any downstream files.

### Step 9 — Confirm and generate docs

When you are happy with both spec folders, tell Claude Code:

```
generate docs
```

Claude Code generates all documents in `project-docs/` layer by layer — using both `specs/user/` and `specs/ai/` as the source of truth. When complete it prints a confirmation table.

---

## What gets generated

| Layer | Folder | Documents |
|---|---|---|
| 00 — Governance | `00-governance/` | G1 Project Charter, G2 RACI Matrix, G3 Risk Register, G4 Change Log, G5 Definition of Done |
| 01 — Requirements | `01-requirements/` | R1 Glossary, R2 Stakeholder Map, R3 BRD, R4a Personas ×n, R5 Flow Registry, R6 Journeys ×n, R7 PRD, R8 Use Cases, R9 NFRs, R10 Acceptance Criteria, R11 Compliance |
| 02 — Design | `02-design/` | D1 Data Model, D2 Flow Specs ×n, D3 Sequence Diagrams ×n, D4 State Machines, D5 API Design, D6 Functional Spec, D7 Error Handling, D8 DB Schema, D9 Notifications, D10 UI/UX Spec, D11 Test Strategy, D12 User Stories ×n |
| 03 — Data | `03-data/` | DM1 Data Dictionary, DM2 Data Flow Diagram, DM3 Seed Data Strategy |
| 04 — Architecture | `04-architecture/` | A1–A13 Architecture docs + INF/CD/SEC/RES/OBS flow docs |
| 05 — Developer Experience | `05-developer-experience/` | DX1 Local Setup, DX2 Coding Standards, DX3 Git Workflow, DX4 PR Guide, DX5 System Walkthrough, DX6 Developer FAQ |
| 06 — Operations | `06-operations/` | O1–O6 Operations docs + REL/FLAG/VER/HOT/COM flow docs |

Per-persona and per-journey documents are generated as individual files.

Sequence diagrams (D3) use Mermaid `sequenceDiagram` syntax. State machine diagrams (D4) use Mermaid `stateDiagram-v2`. API docs (D5) follow OpenAPI structure.

---

## Folder structure

```
my-project/
├── specs/
│   ├── user/                   ← YOU fill these (9 files)
│   │   ├── metadata.json
│   │   ├── product.json
│   │   ├── personas.json
│   │   ├── functional-requirements.json
│   │   ├── business-rules.json
│   │   ├── user-journeys.json
│   │   ├── glossary.json
│   │   ├── non-functional-requirements.json
│   │   └── roadmap.json
│   └── ai/                     ← Claude Code fills these (9 files)
│       ├── domain.json
│       ├── bounded-contexts.json
│       ├── events.json
│       ├── architecture.json
│       ├── infrastructure.json
│       ├── security.json
│       ├── observability.json
│       ├── operations.json
│       └── apis.json
├── CLAUDE.md                   ← generation prompt (do not edit)
└── project-docs/               ← generated documents land here
    ├── 00-governance/
    ├── 01-requirements/
    ├── 02-design/
    ├── 03-data/
    ├── 04-architecture/
    ├── 05-developer-experience/
    └── 06-operations/
```

---

*Built with [docblueprint-engine](https://github.com/anjalee-cooray/docblueprint-engine)*
