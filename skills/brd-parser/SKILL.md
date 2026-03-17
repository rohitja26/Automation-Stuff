---
name: brd-parser
description: >
  Load this skill at Step 1 of the Architecture Design Agent. Reads the BRD agent
  output files from /analysis/ and extracts structured context for architecture
  generation. Never reads the original BRD text. Enforces the readiness gate —
  refuses to proceed if P0 questions are unresolved. Treats answered-questions
  decisions as binding constraints, not assumptions.
---

# BRD Parser Skill

## Purpose
Convert BRD agent output files into a clean, structured context object that the architecture agent uses throughout all 4 steps. This is the ONLY place where /analysis/ files are read — subsequent steps work from the context object, not from files directly.

---

## Step 1: Readiness Gate (HARD STOP)

Before reading any content, verify all of the following. If ANY gate fails, STOP and report the specific failure to the user — do not proceed.

**Gate 1**: `architecture_handoff.json` exists in `/analysis/`
**Gate 2**: `architecture_handoff.json` → `readiness_gate.ready_for_architecture = true`
**Gate 3**: `clarification_questions.json` → `summary.p0_blocking = 0` OR all P0 questions have matching entries in answered-questions files
**Gate 4**: `architecture_handoff.json` → `mvp_requirements` array is non-empty
**Gate 5**: `architecture_handoff.json` → `technology_constraints.constraints` array exists (may be empty if no tech constraints specified — that is allowed)

**Gate failure message format**:
```
ARCHITECTURE AGENT — READINESS GATE FAILED

Gate [N] failed: [specific reason]

Required action: [exactly what the user must do]

Example: Gate 3 failed — 2 P0 questions unresolved (Q-P0-001, Q-P0-002).
Action: Add an answered-questions file to /brd/ with decisions for these questions,
then re-run the BRD analyzer before triggering the architecture agent.
```

---

## Step 2: Load Context Object

Read these files in order and build the architecture context object:

### 2a. From architecture_handoff.json
```
context.technology_constraints = handoff.technology_constraints.constraints
  where binding = true → these are FACTS, not preferences
context.mvp_requirements = handoff.mvp_requirements (array of REQ IDs)
context.phase2_requirements = handoff.phase2_requirements
context.nfr_targets = handoff.nfr_targets (numeric NFR values)
context.integrations = handoff.external_integrations
context.compliance = handoff.compliance_requirements
context.scalability_patterns = handoff.scalability_patterns
context.explicit_constraints = handoff.explicit_constraints
```

### 2b. From requirements_catalog.json — MVP requirements only
Load ONLY requirements whose ID is in `context.mvp_requirements`. This is critical for context budget management — do NOT load all requirements at once.

For each MVP requirement, extract:
```
{
  id, title, description, type, category,
  acceptance_criteria,   ← primary source for flow generation
  business_rules,        ← logic constraints
  dependencies,          ← inter-requirement links
  compliance_flags,      ← regulations this req must satisfy
  greenfield_notes,      ← decision points for architecture
  decisions[],           ← answered-questions decisions (BINDING)
  stakeholders           ← actors in this requirement
}
```

Load Phase 2 requirements as a lightweight index only: `{id, title, category}` — do not load full objects.

### 2c. From domain_model_seed.json
```
context.candidate_entities = seed.entities
  (architecture agent confirms/rejects/modifies — does not blindly accept)
```

### 2d. From business_rules.json
```
context.business_rules = rules
  grouped by: rule_type, then by entities_involved
```

### 2e. From glossary.json — terms only
```
context.naming_authority = glossary terms as {term: definition} map
  This is the ONLY authoritative naming source.
  All architecture artifact names (entities, services, APIs) MUST use terms from this map.
  New names must be consistent with existing terms — no synonyms, no abbreviations not in glossary.
```

### 2f. From assumptions_and_risks.json
```
context.risks = risks (for HLD decision constraints)
context.compliance_radar = assumptions_and_risks.compliance_radar
```

### 2g. From clarification_questions.json — P1 questions only
```
context.p1_decision_points = p1_important questions
  These become mandatory entries in the HLD Decision Log.
  Architecture agent must address every P1 question in the decision log.
```

---

## Step 3: Ambiguity Detection

After loading context, scan for any of these conditions. If found, STOP and report ALL ambiguities before proceeding. Do not generate partial output.

| Condition | Ambiguity type |
|---|---|
| A technology decision point (greenfield_notes.decision_point_type) with no corresponding entry in technology_constraints | Missing tech decision |
| A compliance_flag on a requirement with no compliance_requirement in context.compliance | Missing compliance coverage |
| An integration in context.integrations with no requirement describing its behavior | Undefined integration |
| A scalability_pattern with no corresponding NFR target | Unquantified NFR |
| An actor in context.stakeholders not appearing in any requirement | Orphan actor |

**Ambiguity report format**:
```
ARCHITECTURE AGENT — AMBIGUITIES DETECTED BEFORE DESIGN

The following inputs are missing or unclear. Architecture generation cannot proceed
without these being resolved. Please provide explicit answers.

[1] Missing tech decision: REQ-NF-XXX has greenfield note requiring
    "real-time delivery technology decision" but no technology constraint
    addresses this. Please specify: WebSockets, SSE, or polling.

[2] ...

Total ambiguities: N
Provide answers in /brd/answered-questions.md and re-run BRD analyzer.
```

---

## Step 4: Output Context Summary

After successful context load and zero ambiguities, produce a brief confirmation:

```
BRD Parser complete.
Context loaded:
  MVP requirements: N
  Phase 2 requirements: N (index only)
  Candidate entities: N
  Business rules: N
  Technology constraints: N (binding)
  Explicit constraints: N (binding)
  P1 decision points: N (must address in HLD)
  Compliance: [list]
  Integrations: [list]
  Naming authority terms: N

Proceeding to Step 2: Domain Modeling.
```

---

## Constraints This Skill Enforces

- **Never reads `/brd/` or raw BRD files** — reads `/analysis/` output files only
- **Never invents actors, entities, or requirements** not present in loaded context
- **Technology constraints marked `binding: true` are facts** — the architecture agent must not suggest alternatives to these
- **Naming authority is absolute** — if the glossary says "Vendor" not "Seller", all artifacts use "Vendor"
- **Phase 2 requirements are acknowledged but not designed** — architecture documents their existence in a "Future Scope" section only
