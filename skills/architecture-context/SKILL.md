---
name: architecture-context
description: >
  Always load this skill at Step 1 of the Architecture Design Agent and at
  Phase 0 of the Architecture Validator Agent. Defines context budget rules,
  bounded-context-by-bounded-context processing protocol, checkpoint format,
  and resume-from-checkpoint procedure for architecture generation. Prevents
  context window overflow when processing large requirements catalogs, multiple
  OpenAPI contracts, and multiple DB schemas simultaneously. Separate from
  context-management skill (which is BRD-agent-specific).
---

# Architecture Context Management Skill

## Why Architecture Agents Need Their Own Context Rules

The BRD agent's context problem is **reading large source documents** — it reads prose sections of a BRD one at a time. The context-management skill solves that.

The architecture agents' context problem is different: **accumulating large structured outputs**. A ShopFlow-scale project produces 60+ requirements, 8 bounded contexts, 10+ services, 50+ API endpoints, and 8+ DB schemas. Loading all of this simultaneously overflows context just as badly as a 100-page BRD. But the solution is different — instead of section-by-section prose reading, the architecture agents use **bounded-context-by-bounded-context processing**.

**The same core principle applies**: the disk is the memory, not the context window. Write to disk and discard before loading the next unit of work.

---

## Rule 1: The Bounded-Context Processing Unit

The BRD agent's unit of work is one document section (800 tokens of prose).
The architecture agent's unit of work is one **bounded context** (all entities, APIs, and DB tables for one context at a time).

**Processing loop for LLD generation**:
```
For each bounded context in domain_model.json:
  1. Load ONLY that context's entities + relationships from domain_model.json
  2. Load ONLY that context's MVP requirements from requirements_catalog.json
  3. Load ONLY that context's business rules from business_rules.json
  4. Generate API contracts for this context
  5. Write to /architecture/api_contracts/{context}.openapi.yaml immediately
  6. Generate DB schema for this context
  7. Write to /architecture/db_schemas/{context}.schema.sql immediately
  8. Update checkpoint
  9. Discard all loaded content for this context
  10. Load next context
```

Never load two bounded contexts simultaneously.

---

## Rule 2: Context Budget for Architecture Agents

Rough token budget (Sonnet-class, 200K context):

| Content | Tokens | Notes |
|---|---|---|
| Agent instructions | ~5,000 | Always present |
| Active skill (1 at a time) | ~3,000–4,000 | Load, use, discard |
| Compact state summary | ~300 | Always present |
| Current bounded context entities | ~1,000–3,000 | 5–15 entities with fields |
| Current context requirements | ~2,000–4,000 | 10–20 requirements at 200 tokens each |
| Current context business rules | ~500–1,500 | Rules for this context only |
| Output being constructed | ~2,000–5,000 | API contract or schema being written |
| **Safe working budget** | ~180,000+ | Well within limit per context |

**What NEVER gets loaded simultaneously**:
- All requirements at once (load by bounded context only)
- All API contracts at once (generate and write one context, then next)
- All DB schemas at once (same)
- Full domain_model.json (load one context's slice at a time)
- Both hld.json and lld.json fully in context at the same time

**Warning threshold**: If a single bounded context has more than 20 requirements, split it into two processing passes — first generate APIs, write to disk, then load the same context again for DB schema.

---

## Rule 3: Step-Level Checkpoint (not section-level)

The architecture agent checkpoints at the **step level** (coarser than BRD agent's section level), because each step produces a complete, self-contained artifact file. A completed artifact on disk IS the checkpoint — no partial-step recovery is needed.

**Checkpoint file**: `/architecture/.arch-checkpoint.json`

```json
{
  "checkpoint_version": "1.0",
  "run_id": "uuid",
  "started_at": "ISO-8601",
  "last_updated": "ISO-8601",
  "status": "in_progress | completed | failed | interrupted",
  "brd_run_id": "uuid from architecture_handoff.json",

  "steps": {
    "step_1_brd_parser":        {"status": "completed | in_progress | pending", "completed_at": null},
    "step_2_tech_consultation":  {"status": "pending", "completed_at": null},
    "step_3_domain_model":       {"status": "pending", "completed_at": null},
    "step_4_fdd":                {"status": "pending", "completed_at": null},
    "step_5_hld":                {"status": "pending", "completed_at": null},
    "step_6_lld":                {"status": "pending", "completed_at": null, "contexts_completed": [], "contexts_pending": []},
    "step_7_sdd":                {"status": "pending", "completed_at": null},
    "step_8_tdd":                {"status": "pending", "completed_at": null},
    "step_9_decision_log":       {"status": "pending", "completed_at": null},
    "step_10_self_validation":   {"status": "pending", "completed_at": null}
  },

  "step_6_progress": {
    "api_contracts_written": [],
    "db_schemas_written": [],
    "traceability_backwrite_done": false
  },

  "context_object_summary": {
    "mvp_req_count": 0,
    "bounded_context_count": 0,
    "technology_constraints_count": 0,
    "compliance_count": 0,
    "integration_count": 0
  }
}
```

**Write frequency**:
- After Step 1 completes: mark `step_1_brd_parser: completed`
- After technology_stack.json is written: mark `step_2_tech_consultation: completed`
- After domain_model.json is written: mark `step_3_domain_model: completed`
- After fdd.json is written: mark `step_4_fdd: completed`
- After hld.json is written: mark `step_5_hld: completed`
- After each API contract written: append to `step_6_progress.api_contracts_written[]`
- After each DB schema written: append to `step_6_progress.db_schemas_written[]`
- After Step 6 complete: mark `step_6_lld: completed`
- After sdd.json is written: mark `step_7_sdd: completed`
- After tdd.json is written: mark `step_8_tdd: completed`
- After decision_log.json + traceability back-write: mark `step_9_decision_log: completed`
- After self-validation passes: mark `step_10_self_validation: completed`

Use `vscode/str_replace` to patch only the changed fields. Never rewrite the entire checkpoint.

---

## Rule 4: What to Hold vs. What to Write — Architecture Version

**Hold in context (working memory)**:
- Compact state summary (300 tokens) — always
- Current bounded context slice (entities + requirements + rules for ONE context)
- The artifact currently being constructed (API contract or schema for current context)
- The active skill instructions

**Write to disk immediately and discard**:
- Every completed API contract → `/architecture/api_contracts/{context}.openapi.yaml`
- Every completed DB schema → `/architecture/db_schemas/{context}.schema.sql`
- domain_model.json → after Step 2 completes
- hld.json → after Step 3 completes
- Each FDD flow → append to fdd.json as completed
- Each SDD entry → append to sdd.json as completed
- Checkpoint update → after each unit of work

**Never hold in context after writing**:
- A bounded context's full requirements after its API contract is written
- Any previously completed context's entities or schemas
- The full hld.json while generating LLD (load only the component list — not full HLD)

---

## Rule 5: How Architecture Agent Reads hld.json During LLD

The HLD can be large (dozens of components with full decision logs). The LLD generator does NOT need the full HLD — it needs only the component-to-context mapping.

**Load this slice of hld.json for LLD generation** (not the full file):
```json
{
  "components": [
    {"component_id": "COMP-001", "name": "VendorService", "bounded_context": "VendorContext"},
    {"component_id": "COMP-002", "name": "CatalogService", "bounded_context": "CatalogContext"}
  ]
}
```

Then for each context, load the full component detail ONE AT A TIME:
```
Load hld.json → components[where bounded_context = "VendorContext"]
Generate VendorContext artifacts
Write to disk, discard
Load hld.json → components[where bounded_context = "CatalogContext"]
...
```

---

## Rule 6: Validator Agent — One Artifact at a Time

The Architecture Validator reads multiple artifact files. It must NEVER load all of them simultaneously. Process checks artifact-by-artifact:

**Check 1 (Flow completeness)**:
- Load: `architecture_handoff.json` → mvp_requirements[] (IDs only, ~500 tokens)
- Load: `fdd.json` → one flow at a time, check, discard

**Check 2 (API consistency)**:
- Load: `requirements_catalog.json` → MVP requirement IDs + categories only
- Load: one `api_contracts/` YAML at a time, check, discard

**Check 3 (Schema completeness)**:
- Load: `domain_model.json` → entity list (names + field names only, no full objects)
- Load: one `db_schemas/` SQL file at a time, check, discard

**Check 4 (Assumption detection)**:
- Load: one artifact file at a time, scan text fields, discard
- Do NOT load hld.json + lld.json + fdd.json simultaneously

**Check 5 (Traceability)**:
- Load: `traceability_matrix.json` → req_id + architecture_components[] only
- Cross-reference against mvp_requirements from architecture_handoff.json

**Interim findings**: Write findings to `/architecture/validation_interim_{check_N}.json` after each check before loading next artifact. This prevents finding accumulation from filling context.

---

## Rule 7: Resume from Checkpoint — Architecture Version

On Architecture Design Agent startup, after readiness gate passes:

1. Check if `/architecture/.arch-checkpoint.json` exists
2. If yes and `status = "in_progress"`: ask user — "Interrupted architecture run found (started {timestamp}, Step {N} in progress). [R]esume or [S]tart fresh?"
3. If Resume:
   - Read checkpoint
   - Skip all steps marked `completed` entirely — do not re-run them
   - For `step_4_lld` if `in_progress`: check `contexts_completed[]` — skip those contexts, resume from first context in `contexts_pending[]`
   - Re-load the context object (reconstruct from `architecture_handoff.json` — already on disk)
4. If Start fresh: archive `/architecture/` to `/architecture/archive-{timestamp}/`, delete checkpoint, start from Step 1

**Why step-level resume is sufficient**: Each step produces a complete file (domain_model.json, hld.json, fdd.json, etc.). If step 5 completed successfully, hld.json is valid and on disk. No need for sub-step recovery — just skip the completed step.

**Exception**: Step 6 (LLD) is long and context-intensive. It has context-level granularity: `api_contracts_written[]` and `db_schemas_written[]` track which contexts are done. Resume within Step 6 is supported.

---

## Rule 8: Skill Load/Unload Protocol — Architecture Version

Architecture agents load one skill at a time, maximum. After a step completes and its skill is no longer needed, stop referencing it — it will naturally drop from context as the next skill is loaded.

| Architecture agent step | Skill in context |
|---|---|
| Step 1 — BRD Parser + Quality Gate | architecture-context + brd-parser |
| Step 2 — Technology Consultation | architecture-context only (interactive) |
| Step 3 — Domain Modeler | architecture-context + domain-modeler |
| Step 4 — FDD Generation | architecture-context only (uses feature_groups + user_journeys) |
| Step 5 — HLD Generator | architecture-context + hld-generator |
| Step 6 — LLD Generator | architecture-context + lld-generator |
| Step 7 — SDD Generation | architecture-context only (uses data_flow_map) |
| Step 8 — TDD Generation | architecture-context only (uses requirements + business_rules) |
| Step 9 — Decision Log + Traceability | architecture-context + output-architect |
| Step 10 — Self-Validation | architecture-context only |
| Validator Check 1–8 | architecture-context + architecture-validator |

Never load two non-context skills simultaneously. The architecture-context skill stays loaded throughout the entire run.

---

## Rule 9: Context Budget Monitoring — Architecture Version

Before starting each bounded context in LLD generation, estimate whether it fits:

**Estimate context needed for one bounded context**:
- Entity count × ~200 tokens per entity = entity budget
- Requirement count × ~200 tokens per requirement = requirements budget
- Total estimate = entity budget + requirements budget + 3,000 (skill) + 2,000 (output in progress) + 300 (summary)

**If estimate > 20,000 tokens**: Split the context into two passes:
- Pass 1: API contract generation only (load requirements + entities, write APIs, discard)
- Pass 2: DB schema generation only (reload entities only, write schema, discard)

**Hard limit**: Never load more than 25 requirements at once. A context with 30+ requirements must be split into sub-batches of 15 for processing.
