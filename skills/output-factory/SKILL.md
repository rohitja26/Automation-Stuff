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

## File 8: domain_model_seed.json

Written after Phase 14 (Domain Seed Generation).

```json
{
  "seed_version": "3.2",
  "run_id": "uuid-v4",
  "generated_at": "ISO-8601",
  "entities": [
    {
      "name": "EntityName",
      "description": "One sentence from requirements",
      "source_requirements": ["REQ-F-XXX"],
      "candidate_fields": [
        {"name": "field_name", "type": "string|integer|boolean|datetime|enum|reference", "derived_from": "REQ-F-XXX"}
      ],
      "lifecycle_states": ["state1", "state2"],
      "invariants": ["Business invariant from BR-XXX"],
      "access_control_level": "public | authenticated | role-based | owner-only"
    }
  ],
  "relationships": [
    {"from": "Entity1", "to": "Entity2", "type": "one-to-many", "derived_from": "REQ-F-XXX"}
  ],
  "entity_count": 0,
  "relationship_count": 0
}
```

---

## File 9: business_rules.json

Written after Phase 15 (Business Rules Extraction).

```json
{
  "rules_version": "3.2",
  "run_id": "uuid-v4",
  "generated_at": "ISO-8601",
  "total_rules": 0,
  "rules": [
    {
      "id": "BR-001",
      "rule_type": "validation | calculation | state-transition | constraint | authorization",
      "description": "Rule statement derived from requirements",
      "source_requirements": ["REQ-F-XXX"],
      "entities_involved": ["EntityName"],
      "trigger_condition": "When this condition is true",
      "expected_behavior": "The system shall...",
      "exception_handling": "If condition fails, then...",
      "testable": true
    }
  ]
}
```

---

## File 10: architecture_handoff.json

Written after Phase 16 (Architecture Handoff Assembly).

```json
{
  "handoff_version": "3.2",
  "run_id": "uuid-v4",
  "generated_at": "ISO-8601",
  "readiness_gate": {
    "ready_for_architecture": true,
    "blocking_items": [],
    "conditional_items": []
  },
  "mvp_requirements": ["REQ-F-001"],
  "phase2_requirements": ["REQ-F-050"],
  "nfr_targets": [
    {"nfr_id": "REQ-NF-XXX", "metric": "string", "target_value": "numeric", "measurement_method": "string"}
  ],
  "technology_constraints": {
    "constraints": [
      {"category": "string", "value": "string", "binding": true, "source": "string"}
    ]
  },
  "external_integrations": [
    {"name": "string", "type": "string", "direction": "inbound|outbound|bidirectional"}
  ],
  "compliance_requirements": ["GDPR"],
  "scalability_patterns": ["horizontal-scaling"],
  "explicit_constraints": []
}
```

---

## File 11: user_journeys.json

Written after Phase 15 (User Journey Extraction).

```json
{
  "journeys_version": "3.2",
  "run_id": "uuid-v4",
  "generated_at": "ISO-8601",
  "total_journeys": 0,
  "journeys": [
    {
      "id": "UJ-001",
      "name": "Journey name",
      "actor": "Customer | Vendor | Admin",
      "goal": "What the actor wants to achieve",
      "preconditions": ["string"],
      "steps": [
        {"step": 1, "action": "string", "system_response": "string", "requirement_id": "REQ-F-XXX"}
      ],
      "postconditions": ["string"],
      "requirement_ids": ["REQ-F-XXX"],
      "ui_components": ["screen or component names implied"],
      "services_implied": ["service names implied by the journey"]
    }
  ]
}
```

---

## File 12: api_surface_hints.json

Written after Phase 15 (API Surface Detection).

```json
{
  "hints_version": "3.2",
  "run_id": "uuid-v4",
  "generated_at": "ISO-8601",
  "total_hints": 0,
  "endpoints": [
    {
      "resource": "/entity-name",
      "http_method": "GET | POST | PUT | PATCH | DELETE",
      "operation_summary": "string",
      "input_entities": ["EntityName"],
      "output_entities": ["EntityName"],
      "auth_required": true,
      "auth_level": "public | authenticated | role-based | admin",
      "source_requirements": ["REQ-F-XXX"],
      "confidence": "high | medium | low"
    }
  ]
}
```

---

## File 13: data_flow_map.json

Written after Phase 16 (Data Flow Analysis).

```json
{
  "flow_version": "3.2",
  "run_id": "uuid-v4",
  "generated_at": "ISO-8601",
  "total_flows": 0,
  "flows": [
    {
      "id": "DF-001",
      "from": "source entity or service",
      "to": "target entity or service",
      "data": "what data moves",
      "pattern": "event_driven | request_response | saga | batch | real_time",
      "trigger": "what initiates this flow",
      "source_requirements": ["REQ-F-XXX"],
      "service_dependencies": ["ServiceName"],
      "failure_points": ["what can go wrong"]
    }
  ]
}
```

---

## File 14: feature_groups.json

Written after Phase 14 (Feature Grouping).

```json
{
  "groups_version": "3.2",
  "run_id": "uuid-v4",
  "generated_at": "ISO-8601",
  "total_groups": 0,
  "groups": [
    {
      "id": "FG-001",
      "name": "Feature Group Name",
      "description": "One sentence",
      "requirements": ["REQ-F-XXX"],
      "primary_actors": ["Customer"],
      "bounded_context_hint": "Suggested bounded context"
    }
  ]
}
```

---

## File 15: technology_consultation.json

Written after Phase 9 (Technology Consultation).

```json
{
  "consultation_version": "3.2",
  "run_id": "uuid-v4",
  "generated_at": "ISO-8601",
  "questions_asked": 0,
  "questions_answered": 0,
  "questions": [
    {
      "id": "TC-001",
      "category": "backend | frontend | database | infrastructure | search | messaging | auth",
      "question": "Full question text",
      "answer": "User's answer or null if unanswered",
      "answered_at": "ISO-8601 or null",
      "implications": "What this answer means for architecture"
    }
  ]
}
```

---

## File 16: technology_constraints_binding.json

Written after Phase 9 (from explicit constraints and consultation answers).

```json
{
  "constraints_version": "3.2",
  "run_id": "uuid-v4",
  "generated_at": "ISO-8601",
  "total_constraints": 0,
  "constraints": [
    {
      "category": "backend_language | frontend_framework | database | cloud_provider | search_engine | cache | messaging | auth_provider",
      "value": "Technology name",
      "binding": true,
      "source": "explicit-brd | user-answer-TC-XXX | technology-constraint",
      "cannot_use": ["list of banned alternatives"]
    }
  ]
}
```

---

## File 17: pending_clarifications.json

Written after Phase 18 (Final Assembly). Tracks unresolved items.

```json
{
  "pending_version": "3.2",
  "run_id": "uuid-v4",
  "generated_at": "ISO-8601",
  "total_pending": 0,
  "clarifications": [
    {
      "id": "PC-001",
      "source_question_id": "Q-P1-XXX",
      "question": "Unresolved question text",
      "impact_level": "high | medium | low",
      "blocking_stage": "architecture | development",
      "related_requirements": ["REQ-F-XXX"]
    }
  ]
}
```

---

## greenfield_guidance.json (conditional)

Only written if `greenfield_detected = true`. Schema defined in `greenfield-intelligence` skill.

---

## Additional Fields for requirements_catalog.json

BrdAnalyzer-3.2 adds these fields to each requirement entry (in addition to the base schema above):

```json
{
  "data_entities_involved": ["EntityName"],
  "implied_api_operations": ["GET /resource", "POST /resource"],
  "ui_screens_referenced": ["screen name from UI mockups"],
  "feature_group": "FG-001",
  "test_strategy_hint": "unit | integration | e2e | performance",
  "nfr_quantified_target": {"metric": "response_time", "value": 200, "unit": "ms"},
  "clarification_needed": false,
  "clarification_reason": "null or reason string"
}
```

These are optional per-requirement fields — they may be null if no signal was detected.

---

## Validation Rules (Phase 18)

Before marking output as complete:

1. Parse all 17 JSON files — zero syntax errors allowed
2. `requirements_catalog.json`: `total_requirements` field == `requirements[]` array length
3. `clarification_questions.json`: `summary.total` == `questions[]` array length
4. `traceability_matrix.json`: every `req_id` exists in `requirements_catalog.json`
5. Every `REQ-F-XXX` referenced in `assumptions_and_risks.json` exists in `requirements_catalog.json`
6. No requirement has `source_verbatim` as null, undefined, or empty string
7. No requirement has `priority` as null
8. `analysis_summary.json` `handoff_package.total_requirements` matches `requirements_catalog.json` total
9. `architecture_handoff.json` → `mvp_requirements` is a subset of IDs in `requirements_catalog.json`
10. `domain_model_seed.json` → every entity has at least one `source_requirements` entry
11. `business_rules.json` → every rule has at least one `source_requirements` entry
12. `feature_groups.json` → every group has at least one requirement from `requirements_catalog.json`

**Repair procedure**: If any validation fails, use `vscode/str_replace` to fix the specific field. Log the repair in `analysis_summary.json` under `"repair_log"` array.

