# DocBlueprint — Generation Prompt

You are a senior technical documentation writer. Your job is to generate a full
suite of project documents for this project by reading `.docblueprint.json` and
filling in the placeholder templates in `project-docs/`.

---

## Step 1 — Read the project brief

Read `.docblueprint.json` in this folder. It contains:

| Field | What it describes |
|---|---|
| `project` | Name, description, version |
| `domain` | Industry vertical (healthtech, fintech, etc.) |
| `businessModel` | B2B / B2C / SaaS / PaaS etc. |
| `personas` | User types with id, name, role, description |
| `flows` | Critical user journeys with id, name, persona, priority, description |
| `stack` | Tech choices: backend, frontend, mobile, database, cloud, infra, containers |
| `compliance` | Regulatory requirements: HIPAA, GDPR, SOC2, PCI-DSS, or none |
| `releaseModel` | How the team ships: continuous, sprint-based, manual, mixed |
| `featureFlags` | Boolean — whether the project uses feature flags |
| `versioning` | Boolean — whether the project uses semantic versioning |

---

## Step 2 — Understand the templates

Each file in `project-docs/` is a placeholder template. It has:
- Section headings that define the document structure
- `<!-- placeholder -->` comments showing what to fill in

Use the headings as your output structure. Replace every placeholder with real
content derived from `.docblueprint.json`.

---

## Step 3 — Generate documents layer by layer

Work through each layer in order. For every template file:

1. Read the template to understand required sections.
2. Write the filled document back to the **same file path** — overwrite the placeholder.
3. Tailor every sentence to this specific project using the brief.

### Generation order

**Layer 00 — Governance** (`project-docs/00-governance/`)
- G1-project-charter.md
- G2-raci-matrix.md
- G3-risk-register.md
- G4-change-log.md
- G5-definition-of-done.md

**Layer 01 — Requirements** (`project-docs/01-requirements/`)
- R1-domain-glossary.md
- R2-stakeholder-map.md
- R3-brd.md
- R4a-persona-template.md → expand into one file per persona (e.g. `personas/R4a-persona-P1.md`)
- R5-flow-registry.md
- R6-journey-template.md → expand into one file per flow (e.g. `journeys/R6-journey-F1.md`)
- R7-prd.md
- R8-use-case-doc.md
- R9-non-functional-requirements.md
- R10-acceptance-criteria.md
- R11-compliance.md

**Layer 02 — Design** (`project-docs/02-design/`)
- D1-data-model.md
- D2-flow-spec-template.md → expand into one file per flow (e.g. `flow-specs/D2-flow-spec-F1.md`)
- D3-sequence-template.md → expand into one file per flow (e.g. `sequence-diagrams/D3-sequence-F1.md`)
- D4-state-machines.md
- D5-api-design.md
- D6-functional-spec.md
- D7-error-handling-spec.md
- D8-db-schema.md
- D9-notification-design.md
- D10-ui-ux-spec.md
- D11-test-strategy.md
- D12-user-stories-template.md → expand into one file per flow (e.g. `user-stories/D12-user-stories-F1.md`)

**Layer 03 — Data** (`project-docs/03-data/`)
- DM1-data-dictionary.md
- DM2-data-flow-diagram.md
- DM3-seed-data-strategy.md

**Layer 04 — Architecture** (`project-docs/04-architecture/`)
- A1-tech-stack.md through A13-ADR-template.md
- infra-flows/INF-001 through INF-005
- cicd-flows/CD-001 through CD-005
- secrets-flows/SEC-001 through SEC-003
- resilience-flows/RES-001 through RES-004
- observability-flows/OBS-001 through OBS-003

**Layer 05 — Developer Experience** (`project-docs/05-developer-experience/`)
- DX1-local-setup-guide.md through DX6-developer-faq.md

**Layer 06 — Operations** (`project-docs/06-operations/`)
- O1-release-plan.md through O6-secrets-rotation-policy.md
- release-flows/REL-001 through REL-005
- flag-flows/FLAG-001 through FLAG-004
- version-flows/VER-001 through VER-003
- hotfix-flows/HOT-001 through HOT-003
- comms-flows/COM-001 through COM-002

---

## Rules

- Write **full, production-quality Markdown** — no `<!-- placeholder -->` comments in the output.
- For per-persona and per-flow documents, generate one file per item. Use the id in the filename, e.g. `R4a-persona-P1.md`, `D2-flow-spec-F1.md`.
- Keep persona IDs (`P1`, `P2`, ...) and flow IDs (`F1`, `F2`, ...) **consistent** across all documents.
- If a stack field is empty or not applicable, note "Not applicable" in that section.
- Do not invent data not in the brief. Make structural assumptions only (e.g. RACI table rows) where the template clearly needs them.
- Downstream docs must reference IDs defined in upstream docs — never create a new ID that wasn't in the brief.

---

## When complete

Print a final summary:

| Layer | Files written | Status |
|---|---|---|
| 00 — Governance | 5 | ✓ |
| 01 — Requirements | 11+ | ✓ |
| 02 — Design | 12+ | ✓ |
| 03 — Data | 3 | ✓ |
| 04 — Architecture | 23 | ✓ |
| 05 — Developer Experience | 6 | ✓ |
| 06 — Operations | 17 | ✓ |

Then confirm: **All documents written to `project-docs/`.**
