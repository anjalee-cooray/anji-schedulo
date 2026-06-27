# docblueprint-engine

Spec-driven documentation engine. Fill in structured JSON specs, open the folder in Claude Code, and get a complete suite of project documents generated automatically.

No CLI. No API key. No install step.

---

## How it works

Specs are split into two categories:

| Category | Folder | Who fills it |
|---|---|---|
| **User specs** | `specs/user/` | You — product knowledge only you have |
| **AI specs** | `specs/ai/` | Claude Code — derived from your user specs |

Once both are complete and reviewed, Claude Code generates all documents in `project-docs/`.

---

## Quickstart

### 1 — Clone the template

```bash
git clone https://github.com/your-org/docblueprint-engine-template my-project
cd my-project
```

### 2 — Fill in `specs/user/`

Open each file in `specs/user/` and replace placeholders with your project's real details:

| File | What to fill in |
|---|---|
| `metadata.json` | Project name, owner, version, repo URL, authors |
| `product.json` | Vision, problem statement, target market, pricing tiers, out of scope |
| `personas.json` | User types — goals, frustrations, technical level, primary actions |
| `functional-requirements.json` | What the product must do, with acceptance criteria |
| `business-rules.json` | Non-negotiable rules the system must always enforce |
| `user-journeys.json` | Critical user flows from the user's perspective |
| `glossary.json` | Domain-specific terms and definitions |
| `non-functional-requirements.json` | Performance, availability, and consistency targets |
| `roadmap.json` | Phases, features per phase, what is deferred |

Do not touch `specs/ai/` — Claude Code fills those.

### 3 — Open in Claude Code

```bash
claude .
```

Claude Code reads `CLAUDE.md` automatically. It then runs four phases:

1. **Validate** — checks all `specs/user/` files are fully filled
2. **Fill AI specs** — derives `specs/ai/` in dependency order (domain → bounded contexts → events → architecture → infra → security → observability → operations)
3. **Review** — prints a summary of every AI-filled file and waits for your confirmation
4. **Generate docs** — on your "generate docs", writes all `project-docs/` layers

---

## What gets generated

| Layer | Folder | Documents |
|---|---|---|
| 00 — Governance | `00-governance/` | Project charter, RACI matrix, risk register, change log, definition of done |
| 01 — Requirements | `01-requirements/` | Glossary, stakeholder map, BRD, personas ×n, flow registry, journeys ×n, PRD, use cases, NFRs, acceptance criteria, compliance |
| 02 — Design | `02-design/` | Data model, flow specs ×n, sequence diagrams ×n, state machines, API design, functional spec, error handling, DB schema, notifications, UI/UX spec, test strategy, user stories ×n |
| 03 — Data | `03-data/` | Data dictionary, data flow diagram, seed data strategy |
| 04 — Architecture | `04-architecture/` | A1–A13 architecture docs + infra/cicd/secrets/resilience/observability flows |
| 05 — Developer Experience | `05-developer-experience/` | Local setup, coding standards, git workflow, PR guide, system walkthrough, developer FAQ |
| 06 — Operations | `06-operations/` | Release plan, feature flags, rollback, runbook, incident response, secrets rotation + release/flag/version/hotfix/comms flows |

Per-persona and per-journey documents are generated as individual files.

---

## Repository structure

```
docblueprint-engine/
├── README.md                               ← you are here
└── docblueprint-engine-template/           ← the template users clone
    ├── CLAUDE.md                           ← four-phase generation prompt
    ├── README.md                           ← step-by-step usage guide
    ├── specs/
    │   ├── user/                           ← YOU fill these (9 files)
    │   │   ├── metadata.json
    │   │   ├── product.json
    │   │   ├── personas.json
    │   │   ├── functional-requirements.json
    │   │   ├── business-rules.json
    │   │   ├── user-journeys.json
    │   │   ├── glossary.json
    │   │   ├── non-functional-requirements.json
    │   │   └── roadmap.json
    │   └── ai/                             ← Claude Code fills these (9 files)
    │       ├── domain.json
    │       ├── bounded-contexts.json
    │       ├── events.json
    │       ├── architecture.json
    │       ├── infrastructure.json
    │       ├── security.json
    │       ├── observability.json
    │       ├── operations.json
    │       └── apis.json
    └── project-docs/                       ← generated documents land here
        ├── 00-governance/
        ├── 01-requirements/
        ├── 02-design/
        ├── 03-data/
        ├── 04-architecture/
        ├── 05-developer-experience/
        └── 06-operations/
```

---

## Contributing

- User spec templates live in `specs/user/` — update placeholder text to stay helpful
- AI spec shells live in `specs/ai/` — update the `_ai` derivation notes when sources change
- Template doc structure lives in `project-docs/` — keep document IDs stable (G1, R1, D1, A1, etc.)
- Update `CLAUDE.md` when adding new spec files, document types, or changing generation order
