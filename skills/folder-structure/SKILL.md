---
name: folder-structure
description: Canonical output folder structure and file naming conventions for the full-stack AI delivery pipeline. Every agent in the pipeline references this skill to determine WHERE to write output files, HOW to name them, and WHAT to do when naming decisions are ambiguous. This is the single source of truth for all output paths — agents must never invent ad-hoc paths.
version: 1.0
used-by: PlanningDesign-3.3, JiraOrchestrator-3.2, TicketValidator-1.0
---

# Folder Structure Skill

## Why This Skill Exists

Every agent in this pipeline produces output files. Without a single source of truth for paths and naming, agents drift — one writes to `output/planning/`, another to `03_planning/`, a third to `plan/`. The result is broken cross-references, agents that can't find each other's outputs, and context files that never get updated.

This skill defines the complete, canonical directory layout for every pipeline stage. Every agent reads this skill before writing any file. No agent invents paths. No agent writes to directories not listed here.

---

## Pipeline Stage → Folder Map

```
<project-root>/
├── 01_brd_analysis/          ← Stage 1–2: BrdAnalyzer + BrdMaturityScorer
├── 02_architecture/          ← Stage 3–4: ArchitectureDesign + ArchitectureValidator
├── 03_planning/              ← Stage 5: PlanningDesign
├── 04_tickets/               ← Stage 6–7: JiraOrchestrator + TicketValidator
└── 05_context/               ← Shared state: written and read by stages 5–7 continuously
```

The number prefix (`01_`, `02_`, etc.) is mandatory. It ensures filesystem sorting matches pipeline order, which makes debugging and human review significantly easier.

---

## Full Directory Tree

```
<project-root>/

├── 01_brd_analysis/
│   ├── brd_summary.md
│   ├── functional_requirements.json
│   ├── non_functional_requirements.json
│   ├── stakeholder_map.json
│   ├── assumptions_log.json
│   ├── glossary.json
│   └── maturity_score.json

├── 02_architecture/
│   ├── hld.json
│   ├── tech_stack.json
│   ├── domain_model.json
│   ├── api_contracts.json
│   ├── database_schema.json
│   ├── security_design.json
│   ├── infra_design.json
│   ├── frontend_design.json
│   └── validation_report.json

├── 03_planning/
│   ├── execution_plan.json
│   ├── dependency_graph.json
│   ├── sprint_map.json
│   ├── planning_review_report.md
│   └── module_breakdown/
│       ├── MOD-001_<module_name>/
│       │   ├── module_plan.json
│       │   ├── backend_tasks.json
│       │   └── frontend_tasks.json
│       ├── MOD-002_<module_name>/
│       │   ├── module_plan.json
│       │   ├── backend_tasks.json
│       │   └── frontend_tasks.json
│       ├── [one folder per module — see Module Folder Naming below]
│       ├── integration/
│       │   └── integration_tasks.json
│       └── e2e_testing/
│           └── e2e_tasks.json

├── 04_tickets/
│   ├── raw/
│   │   └── TASK-<id>-<kebab-title>.json
│   ├── validated/
│   │   └── TASK-<id>-<kebab-title>.json
│   ├── validation_report.json
│   ├── validation_feedback/
│   │   └── TASK-<id>-feedback.json
│   └── jira_board_config.json

└── 05_context/
    ├── project_context.json
    ├── api_state.json
    ├── schema_state.json
    └── component_state.json
```

---

## Folder-by-Folder Rules

---

### `01_brd_analysis/`

**Written by:** BrdAnalyzer, BrdMaturityScorer
**Read by:** ArchitectureDesign, PlanningDesign (for NFR references)

| File | Description | Agent |
|---|---|---|
| `brd_summary.md` | Human-readable summary of the BRD | BrdAnalyzer |
| `functional_requirements.json` | Structured list of all functional requirements with IDs | BrdAnalyzer |
| `non_functional_requirements.json` | Performance, security, scalability, availability requirements | BrdAnalyzer |
| `stakeholder_map.json` | Roles, responsibilities, approval authority | BrdAnalyzer |
| `assumptions_log.json` | Every assumption made during BRD analysis | BrdAnalyzer |
| `glossary.json` | Domain-specific terms and their definitions | BrdAnalyzer |
| `maturity_score.json` | BRD quality score, gaps, readiness verdict | BrdMaturityScorer |

**Rules:**
- Files in this folder are **read-only after Stage 2**. No downstream agent modifies them.
- If a downstream agent discovers a BRD gap, it documents it in its own output — it does NOT edit `01_brd_analysis/` files.

---

### `02_architecture/`

**Written by:** ArchitectureDesign, ArchitectureValidator
**Read by:** PlanningDesign, JiraOrchestrator, TicketValidator

| File | Description | Agent |
|---|---|---|
| `hld.json` | High-level design: modules, components, module types | ArchitectureDesign |
| `tech_stack.json` | All technologies with exact versions: backend + frontend + testing + build tools | ArchitectureDesign |
| `domain_model.json` | Entity definitions, relationships, cardinality | ArchitectureDesign |
| `api_contracts.json` | All API endpoints: path, method, request schema, response schema, error codes | ArchitectureDesign |
| `database_schema.json` | Tables, columns, types, constraints, foreign keys, indexes, seed data | ArchitectureDesign |
| `security_design.json` | Auth strategy, JWT config, roles, permissions, frontend route guards, token storage | ArchitectureDesign |
| `infra_design.json` | Environment variables (backend + frontend), CI/CD, containers, deployment targets | ArchitectureDesign |
| `frontend_design.json` | UI framework, routing, component hierarchy, state management, design system, user journeys, layouts | ArchitectureDesign |
| `validation_report.json` | Architecture validation result: PASS/FAIL, issues list, architecture version hash | ArchitectureValidator |

**Rules:**
- Files in this folder are **read-only after Stage 4**. PlanningDesign reads but never writes here.
- `validation_report.json` must exist and contain `"status": "PASS"` before Stage 5 begins.
- `frontend_design.json` is **required**. Its absence is a hard blocker for PlanningDesign.
- The `architecture_version` field in `validation_report.json` is the canonical version identifier. All downstream agents embed this value in their outputs for traceability.

---

### `03_planning/`

**Written by:** PlanningDesign
**Read by:** JiraOrchestrator, TicketValidator

#### Top-level files

| File | Description |
|---|---|
| `execution_plan.json` | Master plan: all module IDs, task counts, sprint assignments, shared utilities, dependency summary |
| `dependency_graph.json` | Full DAG of all task dependencies (backend + frontend). Adjacency list format. Must have no cycles. |
| `sprint_map.json` | Sprint breakdown: for each sprint — goal, backend tasks, frontend tasks, deployable milestone |
| `planning_review_report.md` | Human review gate. Must be stamped APPROVED before JiraOrchestrator can proceed. |

#### Module breakdown subfolder

Each module gets exactly one subfolder. The subfolder contains exactly three files. No more, no less.

```
03_planning/module_breakdown/MOD-XXX_<module_name>/
├── module_plan.json       ← module overview, type, tech stack, dependencies, risk summary
├── backend_tasks.json     ← all backend tasks for this module (empty array + note if frontend_only)
└── frontend_tasks.json    ← all frontend tasks for this module (empty array + note if backend_only)
```

**See Module Folder Naming section below for the exact folder naming rules.**

#### Integration subfolder

```
03_planning/module_breakdown/integration/
└── integration_tasks.json    ← ALL integration tasks: BE↔BE, FE↔BE, FE↔FE
```

All integration tasks go into one file. Do not create sub-files per module pair.

#### E2E testing subfolder

```
03_planning/module_breakdown/e2e_testing/
└── e2e_tasks.json    ← all E2E test tasks, one per user journey from frontend_design.json
```

---

### `04_tickets/`

**Written by:** JiraOrchestrator (raw/), TicketValidator (validated/, validation_report.json, validation_feedback/)
**Read by:** JIRA MCP integration (reads from validated/ only)

#### `04_tickets/raw/`

JiraOrchestrator writes here. TicketValidator reads from here. Nothing reads from `raw/` except TicketValidator.

File naming: `TASK-<zero-padded-3-digit-id>-<kebab-case-title>.json`

Examples:
```
TASK-001-repo-initialization.json
TASK-012-auth-login-endpoint.json
TASK-038-auth-api-client-module.json
```

Rules:
- The `TASK-XXX` prefix in the filename must exactly match the `ticket_id` field inside the file.
- The kebab-case title is derived from the task `title` field: lowercase, spaces→hyphens, remove special characters, truncate at 50 characters.
- Never create subdirectories inside `raw/`. All raw tickets are flat in one folder.

#### `04_tickets/validated/`

TicketValidator writes here — only after an entire sprint passes all validation checks. JIRA MCP reads from here.

Same naming convention as `raw/`. A file in `validated/` is a copy of the corresponding file from `raw/` — its content is identical. TicketValidator does not modify ticket content. It only copies passing tickets.

Rules:
- Never write directly to `validated/`. Only TicketValidator moves files here.
- If a sprint has any FAIL tickets, zero tickets from that sprint move to `validated/`.

#### `04_tickets/validation_report.json`

Written by TicketValidator after every validation run. Overwritten on each run (not appended). Contains the full validation results for the most recent run.

#### `04_tickets/validation_feedback/`

TicketValidator writes one feedback file per failed ticket:
```
TASK-<id>-feedback.json
```
JiraOrchestrator reads these files when correcting tickets and resubmitting.

#### `04_tickets/jira_board_config.json`

Written by JiraOrchestrator once. Contains the JIRA board column definitions, sprint config, epic structure, and label taxonomy. Read by the JIRA MCP integration to configure the board before tickets are pushed.

---

### `05_context/`

**Written by:** All agents (after ticket completion), read by JiraOrchestrator when populating Section 4 of each ticket.

This folder holds the shared state of the project as it is being built. It starts empty and grows as tickets are completed. JiraOrchestrator reads these files to populate "Existing Implementation State" (Section 4) in every new ticket.

| File | Contains | Updated by |
|---|---|---|
| `project_context.json` | Progress tracker: last completed task, current sprint, task counts, last updated timestamp | Every ticket completion |
| `api_state.json` | All implemented API endpoints: path, method, handler file, request/response schema, error codes, implemented_by task | Every backend endpoint ticket completion |
| `schema_state.json` | All implemented database tables: table name, columns, migration file, implemented_by task | Every migration ticket completion |
| `component_state.json` | All implemented services, stores, components, hooks: class/function name, file path, exports, implemented_by task | Every service/component ticket completion |

**Initialization rule:** On first run (before any tickets are completed), all four files must exist with empty/skeleton content. PlanningDesign creates empty skeletons if they don't exist. JiraOrchestrator reads them on every ticket generation — a missing file is an error, not a handled case.

**Skeleton format for `project_context.json`:**
```json
{
  "project_id": "<from execution_plan.json>",
  "architecture_version": "<from validation_report.json>",
  "last_completed_task": null,
  "completed_tasks_count": 0,
  "total_tasks": 0,
  "current_sprint": 1,
  "last_updated": null,
  "status": "not_started"
}
```

**Skeleton format for `api_state.json`:**
```json
{ "endpoints": [] }
```

**Skeleton format for `schema_state.json`:**
```json
{ "tables": [] }
```

**Skeleton format for `component_state.json`:**
```json
{ "services": [], "stores": [], "components": [], "hooks": [], "utilities": [] }
```

---

## Module Folder Naming

This is the single most important naming rule in the entire pipeline. Every agent that creates or references a module folder must follow this exactly.

### Format

```
MOD-<zero-padded-3-digit-sequence>_<snake_case_module_name>
```

### Rules

1. **`MOD-` prefix** — always uppercase, always with a hyphen.
2. **Zero-padded 3-digit sequence** — `001`, `002`, `003`, not `1`, `2`, `3`. Supports up to 999 modules.
3. **Underscore separator** — exactly one underscore between the ID and the name.
4. **Snake_case name** — all lowercase, spaces replaced with underscores, no hyphens, no special characters.
5. **Name comes from `hld.json`** — derive the name from `hld.json → components[n].name`. Apply snake_case transformation. Do not invent names.
6. **Sequence matches `hld.json` order** — the first component in `hld.json → components` is MOD-001, the second is MOD-002, etc. Do not reorder.
7. **Names are immutable once assigned** — if `hld.json` is updated after planning begins, the folder name does NOT change. The sequence and name are locked at planning time.

### Examples

| `hld.json` component name | Folder name |
|---|---|
| `"Environment Setup"` | `MOD-001_environment_setup` |
| `"User Authentication"` | `MOD-002_user_authentication` |
| `"User Management"` | `MOD-003_user_management` |
| `"Product Catalog"` | `MOD-004_product_catalog` |
| `"Shopping Cart"` | `MOD-005_shopping_cart` |
| `"Order Management"` | `MOD-006_order_management` |
| `"Payment Processing"` | `MOD-007_payment_processing` |
| `"Admin Dashboard"` | `MOD-008_admin_dashboard` |
| `"Notification Service"` | `MOD-009_notification_service` |
| `"Reporting & Analytics"` | `MOD-010_reporting_and_analytics` |

### What to do with special characters in module names

| Character in source name | Transformation |
|---|---|
| Space | `_` (underscore) |
| Hyphen `-` | `_` (underscore) |
| Ampersand `&` | `and` |
| Slash `/` | `_or_` |
| Parentheses `()` | remove |
| Any other special character | remove |

---

## File Naming Rules

### JSON files

All JSON files use `snake_case.json`. No camelCase, no kebab-case for JSON files.

Examples: `execution_plan.json`, `backend_tasks.json`, `api_state.json`

### Markdown files

All markdown files use `snake_case.md`.

Example: `planning_review_report.md`

### Ticket files

Ticket files are the only exception — they use a mixed format because the TASK ID prefix is significant:

```
TASK-<zero-padded-3-digit-id>-<kebab-case-title>.json
```

The `TASK-XXX` prefix uses uppercase and hyphen (matching the ticket ID format). The title portion uses lowercase kebab-case.

### Deriving kebab-case title from task title

Given task title: `"Implement JWT login endpoint with bcrypt verification"`

1. Lowercase: `"implement jwt login endpoint with bcrypt verification"`
2. Replace spaces with hyphens: `"implement-jwt-login-endpoint-with-bcrypt-verification"`
3. Remove special characters: (none to remove)
4. Truncate at 50 characters: `"implement-jwt-login-endpoint-with-bcrypt-verifi"` — actually 50 chars is fine here, count carefully
5. Result: `TASK-012-implement-jwt-login-endpoint-with-bcrypt.json`

When in doubt, keep the title short and descriptive. The TASK ID is the primary identifier — the title is just for human readability.

---

## What to Do When a File Already Exists

This is the most dangerous edge case. An agent should never silently overwrite a file that may contain valid work from a prior run.

| Scenario | Rule |
|---|---|
| Agent is writing a file for the first time (clean run) | Write directly. |
| Agent is re-running after a failure and the file already exists | **Check the file's `generated_at` or `created_at` timestamp field.** If it is from the current pipeline run (same `plan_id` or same day), overwrite. If it is from a prior pipeline run, halt and report: "FILE_EXISTS_FROM_PRIOR_RUN — [filename]. Confirm overwrite or archive the existing file before proceeding." |
| Agent is correcting and resubmitting a failed ticket | Write to `04_tickets/raw/` with the same filename. This is an expected overwrite. No confirmation needed. |
| Two agents try to write to the same file simultaneously | This pipeline is sequential — simultaneous writes should not occur. If they do, halt and report: "CONCURRENT_WRITE_CONFLICT — [filename]. Pipeline may have branched unexpectedly." |

---

## Cross-Reference Integrity

When one agent references a file written by another agent, it must use the exact path from this skill — not a path it infers or remembers.

### Correct cross-reference pattern

```
JiraOrchestrator reading planning output:
  → 03_planning/execution_plan.json
  → 03_planning/module_breakdown/MOD-003_authentication/backend_tasks.json
  → 03_planning/module_breakdown/MOD-003_authentication/frontend_tasks.json
  → 05_context/api_state.json
```

### Incorrect patterns (never do these)

```
❌ planning/execution_plan.json           (missing numeric prefix)
❌ 3_planning/execution_plan.json         (missing zero-padding)
❌ 03_planning/modules/MOD-3_auth/        (wrong module folder format)
❌ output/tickets/TASK-12.json            (wrong folder, wrong ID format)
❌ context/api_state.json                 (missing numeric prefix)
```

---

## Quick Reference Card

Use this when you need a fast lookup without reading the full spec.

```
01_brd_analysis/                              ← BRD outputs (read-only after Stage 2)
02_architecture/                              ← Architecture files (read-only after Stage 4)
03_planning/                                  ← Planning outputs
03_planning/module_breakdown/                 ← One subfolder per module
03_planning/module_breakdown/MOD-XXX_<n>/     ← module_plan + backend_tasks + frontend_tasks
03_planning/module_breakdown/integration/     ← integration_tasks.json
03_planning/module_breakdown/e2e_testing/     ← e2e_tasks.json
04_tickets/raw/                               ← JiraOrchestrator writes here
04_tickets/validated/                         ← TicketValidator writes PASS tickets here
04_tickets/validation_feedback/               ← TicketValidator writes FAIL feedback here
05_context/                                   ← Shared state, updated after each ticket
```

```
Module folder name:   MOD-<001>_<snake_case_name>
Ticket file name:     TASK-<001>-<kebab-case-title>.json
JSON files:           snake_case.json
Markdown files:       snake_case.md
```
