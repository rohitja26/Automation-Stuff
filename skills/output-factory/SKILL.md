---
name: output-factory
description: >
  Always load this skill before Phase 12 (output generation). Defines the exact
  JSON schema for all 7 output files, formatting rules, validation requirements,
  and the human-readable analysis summary. Ensures consistent, machine-readable
  output compatible with Stage 2 (BRD Maturity Scoring) agent.
---

# Output Factory Skill

## Purpose
Define exact schemas and formatting rules for all BrdAnalyzer v3.0 output files. Every field, type, and required/optional designation is specified here. The agent must conform to these schemas exactly — Stage 2 agents depend on this structure.

---

## File 1: intake-manifest.json

Written after Phase 1. Updated if re-analysis occurs.

```json
{
  "manifest_version": "3.0",
  "run_id": "uuid-v4",
  "created_at": "ISO-8601 timestamp",
  "brd_directory": "/brd/",
  "artifacts": [
    {
      "filename": "string",
      "extension": "string",
      "classification": "BRD | FRD | TRD | UI-MANIFEST | UI-DESIGN | DATA | UNKNOWN | NON-PROCESSABLE",
      "processing_status": "processed | skipped | failed | requires-external-tool",
      "extraction_method": "direct-read | multi-format-skill | artifact-intake-skill | mcp-tool | manual-required",
      "word_count": 0,
      "confidence": "high | medium | low | none",
      "p0_question_generated": false,
      "notes": "string or null"
    }
  ],
  "totals": {
    "total_files": 0,
    "processed": 0,
    "non_processable": 0,
    "p0_questions_generated": 0
  },
  "project_classification": "GREENFIELD | BROWNFIELD | MIGRATION | UNKNOWN",
  "greenfield_detected": false,
  "compliance_signals_detected": [],
  "companion_documents": {
    "frd_found": false,
    "trd_found": false,
    "ui_designs_found": false
  }
}
```

---

## File 2: requirements_catalog.json

Written incrementally during Phase 3. Finalized in Phase 12.

```json
{
  "catalog_version": "3.0",
  "run_id": "uuid-v4",
  "generated_at": "ISO-8601 timestamp",
  "total_requirements": 0,
  "requirements": [
    {
      "id": "REQ-F-001",
      "title": "Short imperative title",
      "description": "Full normalized requirement statement. First person imperative. Subject is always the system.",
      "type": "functional | non-functional",
      "category": "category-name",
      "priority": "must-have | should-have | nice-to-have",
      "moscow": "must-have | should-have | could-have | wont-have",
      "priority_basis": "explicit-brd-language | linked-to-objective | inferred-default",
      "source_verbatim": "Exact text from source document. Required. Never null.",
      "source_location": {
        "file": "filename.ext",
        "section": "Section Name",
        "page": 0
      },
      "acceptance_criteria": [
        "Given [precondition], when [action], then [expected outcome]"
      ],
      "dependencies": ["REQ-F-XXX"],
      "smart_score": {
        "specific": 0,
        "measurable": 0,
        "achievable": 0,
        "relevant": 0,
        "timebound": 0,
        "total": 0,
        "needs_clarification": false
      },
      "ambiguity_flags": [
        {
          "type": "vague-quantifier | undefined-actor | missing-error-handling | untestable | passive-voice",
          "text": "The specific phrase that is ambiguous",
          "question_id": "Q-P1-XXX"
        }
      ],
      "conflicts": [
        {
          "conflicting_req_id": "REQ-NF-XXX",
          "conflict_type": "security-vs-performance | scope-overlap | priority-contradiction",
          "description": "Brief conflict description",
          "question_id": "Q-P1-XXX"
        }
      ],
      "stakeholders": ["role name"],
      "estimated_complexity": "low | medium | high",
      "frd_refinements": ["REQ-F-001-R01"],
      "trd_constraints": ["ASM-XXX"],
      "compliance_flags": ["GDPR", "COPPA"],
      "greenfield_notes": "Technology implication notes. Null if not greenfield.",
      "mvp_candidate": null
    }
  ],
  "non_functional_categories": {
    "performance": 0,
    "security": 0,
    "compliance": 0,
    "accessibility": 0,
    "reliability": 0,
    "scalability": 0,
    "usability": 0,
    "maintainability": 0
  }
}
```

**Required field enforcement**: Before writing any requirement entry, verify these fields are non-null and non-empty:
- `id` — must match pattern `REQ-[F|NF|C]-\d{3}`
- `source_verbatim` — must be a quoted string from the source document
- `description` — must be a complete sentence
- `priority` — must be set by decision tree, not left as null

---

## File 3: clarification_questions.json

Written after Phase 7.

```json
{
  "questions_version": "3.0",
  "run_id": "uuid-v4",
  "generated_at": "ISO-8601 timestamp",
  "summary": {
    "p0_blocking": 0,
    "p1_important": 0,
    "p2_clarifying": 0,
    "total": 0
  },
  "questions": [
    {
      "id": "Q-P0-001",
      "priority": "P0 | P1 | P2",
      "category": "scope | ambiguity | conflict | compliance | priority | technical | stakeholder | greenfield",
      "question": "Full question text",
      "context": "Why this question matters for requirements completeness",
      "related_requirements": ["REQ-F-XXX"],
      "source_trigger": "What in the BRD triggered this question",
      "blocking": true,
      "blocks_stage": "Stage 2 | Architecture | Development"
    }
  ]
}
```

---

## File 4: assumptions_and_risks.json

Written incrementally during Phase 5 and finalized in Phases 10 and 9.

```json
{
  "ar_version": "3.0",
  "run_id": "uuid-v4",
  "generated_at": "ISO-8601 timestamp",
  "assumptions": [
    {
      "id": "ASM-001",
      "assumption": "What was assumed",
      "rationale": "Why this assumption was made and what evidence supports it",
      "impact_if_wrong": "high | medium | low",
      "related_requirements": ["REQ-F-XXX"],
      "validation_needed": true,
      "validation_question_id": "Q-P1-XXX"
    }
  ],
  "risks": [
    {
      "id": "RISK-001",
      "title": "Short risk title",
      "description": "Risk description",
      "probability": "high | medium | low",
      "impact": "high | medium | low",
      "risk_score": 0,
      "mitigation": "Suggested mitigation approach",
      "related_requirements": ["REQ-F-XXX"],
      "source": "derived-from-gap | derived-from-trd | inferred-from-context | compliance-gap"
    }
  ],
  "compliance_radar": {
    "regulations_applicable": [],
    "detection_basis": {},
    "coverage_by_regulation": {},
    "compliance_gaps": [],
    "suggested_compliance_requirements": [],
    "overall_compliance_readiness": "low | medium | high"
  }
}
```

---

## File 5: traceability_matrix.json

Written after Phase 11.

```json
{
  "rtm_version": "3.0",
  "run_id": "uuid-v4",
  "generated_at": "ISO-8601 timestamp",
  "entries": [
    {
      "req_id": "REQ-F-001",
      "title": "Requirement title",
      "source_document": "filename.ext",
      "source_section": "Section name",
      "frd_refinements": [],
      "trd_constraints": [],
      "risks": ["RISK-XXX"],
      "assumptions": ["ASM-XXX"],
      "clarification_questions": ["Q-P1-XXX"],
      "compliance_flags": ["GDPR"],
      "test_coverage": "full | partial | none",
      "test_coverage_basis": "acceptance criteria present | no acceptance criteria | vague criteria"
    }
  ]
}
```

---

## File 6: glossary.json

Written after Phase 11.

```json
{
  "glossary_version": "3.0",
  "run_id": "uuid-v4",
  "generated_at": "ISO-8601 timestamp",
  "terms": [
    {
      "term": "Term as used in BRD",
      "definition": "Definition as used in BRD",
      "source_file": "filename.ext",
      "source_section": "Section name",
      "first_appearance": "REQ-F-XXX or 'pre-requirements section'",
      "ambiguity_flag": false,
      "ambiguity_note": "If same term appears with different definitions across documents, describe the discrepancy here. Null otherwise.",
      "question_id": "Q-P2-XXX or null"
    }
  ]
}
```

---

## File 7: analysis_summary.json

Written after Phase 12. This is the primary handoff document for Stage 2.

(Full schema defined in main agent — Appendix Phase 12. Ensure all fields are populated.)

**Quality score calculation rules**:

**Completeness score** (0-100):
- Start at 0
- +15 if business objectives present
- +15 if stakeholders identified
- +15 if scope boundary stated
- +15 if functional requirements have acceptance criteria (>80% coverage)
- +15 if non-functional requirements present
- +10 if constraints documented
- +10 if assumptions documented
- +5 if glossary has > 5 terms

**Clarity score** (0-100):
- Start at 100
- -5 per requirement with vague quantifier (max -30)
- -5 per requirement with undefined actor (max -20)
- -10 per requirement marked untestable (max -30)
- -5 per passive-voice ambiguity flag (max -20)

**Testability score** (0-100):
- `(requirements_with_acceptance_criteria / total_requirements) * 100`

**Traceability score** (0-100):
- `(requirements_with_source_verbatim / total_requirements) * 100`

---

## greenfield_guidance.json (conditional)

Only written if `greenfield_detected = true`. Schema defined in `greenfield-intelligence` skill.

---

## Validation Rules (Phase 13)

Before marking output as complete:

1. Parse all JSON files — zero syntax errors allowed
2. `requirements_catalog.json`: `total_requirements` field == `requirements[]` array length
3. `clarification_questions.json`: `summary.total` == `questions[]` array length
4. `traceability_matrix.json`: every `req_id` exists in `requirements_catalog.json`
5. Every `REQ-F-XXX` referenced in `assumptions_and_risks.json` exists in `requirements_catalog.json`
6. No requirement has `source_verbatim` as null, undefined, or empty string
7. No requirement has `priority` as null
8. `analysis_summary.json` `handoff_package.total_requirements` matches `requirements_catalog.json` total

**Repair procedure**: If any validation fails, use `vscode/str_replace` to fix the specific field. Log the repair in `analysis_summary.json` under `"repair_log"` array.
