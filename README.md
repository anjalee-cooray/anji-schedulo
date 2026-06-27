# docblueprint-engine

Spec-driven documentation engine. Fill in a JSON file, open the folder in Claude Code, and get a complete suite of project documents generated automatically.

No CLI. No API key. No install step.

---

## How it works

```
docblueprint-engine/
└── docblueprint-engine-template/   ← copy this folder to start a new project
    ├── .docblueprint.json          ← fill this in with your project details
    ├── CLAUDE.md                   ← Claude Code reads this and generates all docs
    └── project-docs/               ← 90+ placeholder templates, filled by Claude Code
```

### 1 — Clone the template

```bash
git clone https://github.com/your-org/docblueprint-engine-template my-project
cd my-project
```

### 2 — Fill in `.docblueprint.json`

Open `.docblueprint.json` and replace the example values with your project's details:

- `project` — name, description, version
- `domain` — industry vertical (healthtech, fintech, edtech, etc.)
- `businessModel` — B2B, B2C, SaaS, PaaS, etc.
- `personas` — user types with id, name, role, description
- `flows` — critical user journeys with id, name, persona, priority, description
- `stack` — backend, frontend, database, cloud, infra, containerisation
- `compliance` — HIPAA, GDPR, SOC2, PCI-DSS, or none
- `releaseModel` — continuous, sprint-based, manual, or mixed

### 3 — Open in Claude Code

```bash
claude .
```

Claude Code reads `CLAUDE.md` automatically, reads your `.docblueprint.json`, and generates all documents into `project-docs/` — layer by layer in dependency order.

---

## What gets generated

| Layer | Folder | Documents |
|---|---|---|
| 00 — Governance | `00-governance/` | Project charter, RACI matrix, risk register, change log, definition of done |
| 01 — Requirements | `01-requirements/` | Glossary, stakeholder map, BRD, personas (×n), flow registry, journeys (×n), PRD, use cases, NFRs, acceptance criteria, compliance |
| 02 — Design | `02-design/` | Data model, flow specs (×n), sequence diagrams (×n), state machines, API design, functional spec, error handling, DB schema, notifications, UI/UX spec, test strategy, user stories (×n) |
| 03 — Data | `03-data/` | Data dictionary, data flow diagram, seed data strategy |
| 04 — Architecture | `04-architecture/` | System arch, tech stack, security model, threat model, data privacy, infra, scaling, deployment, integrations, observability, DR, multi-tenancy, ADRs + 20 flow docs |
| 05 — Developer Experience | `05-developer-experience/` | Local setup, coding standards, git workflow, PR guide, system walkthrough, developer FAQ |
| 06 — Operations | `06-operations/` | Release plan, feature flags, rollback, runbook, incident response, secrets rotation + 17 flow docs |

Per-persona and per-flow documents are expanded automatically from your `.docblueprint.json`.

---

## Repository structure

```
docblueprint-engine/
├── README.md                           ← you are here
└── docblueprint-engine-template/       ← the template users copy
    ├── .docblueprint.json              ← project brief (user fills this in)
    ├── CLAUDE.md                       ← generation prompt for Claude Code
    ├── README.md                       ← template usage guide
    └── project-docs/                   ← 90+ placeholder .md files
```

---

## Contributing

- Template structure changes go in `docblueprint-engine-template/project-docs/`
- Keep document IDs stable (G1, R1, D1, A1, etc.) — `CLAUDE.md` references them by name
- Update `.docblueprint.json` example values to stay representative
- Update `CLAUDE.md` when adding new document types or changing the generation order
