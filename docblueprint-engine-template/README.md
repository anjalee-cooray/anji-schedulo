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

Claude Code reads `CLAUDE.md` automatically when the folder opens. No extra command needed.

### Step 6 — Claude Code fills the AI specs

Claude Code reads your `specs/user/` files and derives the technical specs in `specs/ai/`:

| File | What Claude Code derives |
|---|---|
| `domain.json` | All domain entities, aggregates, lifecycles |
| `bounded-contexts.json` | Service boundaries, events published and subscribed |
| `events.json` | Every domain event — producer, consumers, payload fields |
| `architecture.json` | Architecture style and patterns suited to your NFRs |
| `infrastructure.json` | Full cloud and tech stack |
| `security.json` | Auth mechanism, RBAC roles, tenant isolation strategy |
| `observability.json` | SLIs, SLOs, metrics, alerts |
| `operations.json` | DLQ strategy, replay, backups, incident runbooks |

### Step 7 — Review the AI specs

Claude Code prints a summary of everything it filled. Read through `specs/ai/`. If anything needs correcting, tell Claude Code in plain language:

```
"The architecture should use a modular monolith not microservices —
update architecture.json and re-derive bounded-contexts.json."
```

Claude Code applies the correction and updates any downstream files.

### Step 8 — Confirm and generate docs

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
