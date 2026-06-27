# docblueprint-engine — project template

Generate a complete suite of project documents from a single JSON file using Claude Code.

No CLI. No API key. No install step.

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
git clone https://github.com/your-org/docblueprint-engine-template my-project
cd my-project
```

### Step 4 — Fill in `.docblueprint.json`

Open `.docblueprint.json`. It contains example values with comments explaining each field. Replace every value with your project's real details.

```json
{
  "project": {
    "name": "My Project",
    "description": "What this product does and who it is for."
  },
  "domain": "healthtech",
  "businessModel": ["B2B", "SaaS"],
  "personas": [
    { "id": "P1", "name": "Admin", "role": "Administrator", "description": "..." },
    { "id": "P2", "name": "End User", "role": "Customer", "description": "..." }
  ],
  "flows": [
    { "id": "F1", "name": "User Signup", "persona": "P2", "priority": "critical", "description": "..." }
  ],
  "stack": {
    "backend": "Node.js",
    "frontend": "React",
    "database": "PostgreSQL",
    "cloud": "AWS"
  },
  "compliance": ["HIPAA"],
  "releaseModel": "continuous",
  "featureFlags": true,
  "versioning": true
}
```

**Tips:**
- Add as many personas and flows as your project needs — documents expand automatically per item
- Leave stack fields empty (`""`) for technologies you're not using
- Use `"none"` for compliance if there are no requirements

### Step 5 — Open the project folder in Claude Code

```bash
claude .
```

Claude Code automatically reads `CLAUDE.md` when it opens the folder. No extra command is needed.

### Step 6 — Let Claude Code generate the documents

Claude Code will:

1. Read `.docblueprint.json` — your project brief
2. Read each placeholder template in `project-docs/`
3. Generate full, tailored document content for your project
4. Write the filled documents back into `project-docs/`

Documents are generated layer by layer in dependency order — governance first, then requirements, design, data, architecture, developer experience, and operations.

When complete, Claude Code prints a summary table confirming every file written.

### Step 7 — Review the output

Open any file in `project-docs/` to read the generated content. If something needs adjusting, tell Claude Code directly in plain language:

```
"The personas in R4a don't match our actual user types — update them and regenerate the downstream documents."
```

Claude Code will apply the correction and update any affected files.

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

Per-persona documents (R4a) and per-flow documents (R6, D2, D3, D12) are generated as individual files using the IDs from your `.docblueprint.json`.

---

## Folder structure

```
my-project/
├── .docblueprint.json              ← your project brief (fill this in)
├── CLAUDE.md                       ← generation prompt (do not edit)
└── project-docs/
    ├── 00-governance/
    ├── 01-requirements/
    │   ├── personas/               ← one file per persona after generation
    │   └── journeys/               ← one file per flow after generation
    ├── 02-design/
    │   ├── flow-specs/
    │   ├── sequence-diagrams/
    │   └── user-stories/
    ├── 03-data/
    ├── 04-architecture/
    │   ├── adrs/
    │   ├── infra-flows/
    │   ├── cicd-flows/
    │   ├── secrets-flows/
    │   ├── resilience-flows/
    │   └── observability-flows/
    ├── 05-developer-experience/
    ├── 06-operations/
    │   ├── release-flows/
    │   ├── flag-flows/
    │   ├── version-flows/
    │   ├── hotfix-flows/
    │   └── comms-flows/
    └── fe-design/
        ├── lofi/
        ├── hifi/
        └── component-specs/
```

---

## Why dependency order matters

Each document is context for the next. Claude Code generates them in strict order so later documents are always consistent with earlier ones:

- Glossary → BRD (shared vocabulary before requirements)
- Personas → User Journeys (who before what)
- Data Model → Sequence Diagrams (entities before interactions)
- Sequence Diagrams → API Design (interactions before contracts)
- All of the above → Architecture (decisions informed by requirements)
- Architecture → Developer Experience (system understood before onboarding docs)
- Everything → Operations (operations planned last, informed by everything)

---

*Built with [docblueprint-engine](https://github.com/org/docblueprint-engine)*
