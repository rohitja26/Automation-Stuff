---
name: maturity-scoring
description: >
  Load this skill when running the BrdMaturityScorer agent. Defines scoring
  dimensions, weight constants, sub-dimension formulas, safety cap rules,
  cross-validation logic, and score normalization patterns. Provides a
  single source of truth for all maturity scoring calculations.
---

# Maturity Scoring Skill

## Purpose
Define the exact scoring methodology for BRD maturity assessment. The BrdMaturityScorer agent uses this skill to compute a consistent, reproducible quality score across all BRD analysis outputs. All weight constants, formulas, and thresholds are defined here — the agent should not invent scoring logic.

---

## Scoring Dimensions (8 total)

### Dimension 1: Completeness (weight: 20%)
How fully does the BRD cover all expected areas?

**Sub-scores** (each 0-100, averaged):
- Business objectives present → score based on `analysis_summary.json → project_overview`
- Stakeholders identified → from `analysis_summary.json → stakeholder_count`
- Scope boundary stated → check for scope section in requirements
- Functional requirements have acceptance criteria → `(reqs_with_AC / total_functional_reqs) × 100`
- Non-functional requirements present → `min(nfr_count / 5, 1) × 100` (5+ NFRs = 100)
- Constraints documented → from `assumptions_and_risks.json → assumptions[].count`
- Glossary populated → `min(glossary_term_count / 10, 1) × 100`

---

### Dimension 2: Clarity (weight: 15%)
How unambiguous are the requirements?

**Formula**: Start at 100, deduct:
- -3 per requirement with `ambiguity_flags[type=vague-quantifier]` (cap: -30)
- -3 per requirement with `ambiguity_flags[type=undefined-actor]` (cap: -20)
- -5 per requirement with `ambiguity_flags[type=untestable]` (cap: -30)
- -2 per requirement with `ambiguity_flags[type=passive-voice]` (cap: -20)

Floor at 0.

---

### Dimension 3: Testability (weight: 15%)
Can requirements be verified?

**Formula**: `(requirements_with_acceptance_criteria / total_requirements) × 100`

Source: `requirements_catalog.json → requirements[].acceptance_criteria`

---

### Dimension 4: Traceability (weight: 10%)
Can every requirement be traced to source?

**Formula**: `(requirements_with_source_verbatim / total_requirements) × 100`

Source: `requirements_catalog.json → requirements[].source_verbatim` (non-null, non-empty)

---

### Dimension 5: Risk Awareness (weight: 10%)
Are risks and assumptions explicitly identified?

**Sub-scores**:
- Assumptions documented: `min(assumption_count / 3, 1) × 100`
- Risks documented: `min(risk_count / 3, 1) × 100`
- High-impact risks have mitigations: `(risks_with_mitigation / high_impact_risks) × 100`

---

### Dimension 6: Downstream Readiness (weight: 15%)
Is the BRD ready for architecture consumption?

**Sub-scores**:
- `architecture_handoff.json` exists and `readiness_gate.ready_for_architecture = true` → 30 points
- MVP requirements identified → `min(mvp_count / 5, 1) × 30` (30 points max)
- Domain model seed has entities → `min(entity_count / 3, 1) × 20` (20 points max)
- Feature groups defined → `min(group_count / 2, 1) × 20` (20 points max)

---

### Dimension 7: Domain Model Maturity (weight: 10%)
How well-formed is the domain model seed?

**Sub-scores**:
- Entities have fields with data types → `(entities_with_typed_fields / total_entities) × 100`
- Relationships identified → `min(relationship_count / entity_count, 1) × 100`
- Lifecycle states defined → `(entities_with_lifecycle / total_entities) × 100`

---

### Dimension 8: Business Rules Coverage (weight: 5%)
Are business rules explicit and testable?

**Formula**: `min(business_rule_count / max(functional_req_count × 0.3, 1), 1) × 100`

Rationale: Expect roughly 1 business rule per 3 functional requirements.

---

## Overall Score Calculation

```
overall = (completeness × 0.20) + (clarity × 0.15) + (testability × 0.15) +
          (traceability × 0.10) + (risk_awareness × 0.10) +
          (downstream_readiness × 0.15) + (domain_model_maturity × 0.10) +
          (business_rules × 0.05)
```

---

## Safety Caps

These conditions override the calculated score:

| Condition | Cap | Reason |
|---|---|---|
| Zero functional requirements | Cap at 20 | No reqs = not a BRD |
| Zero acceptance criteria across all requirements | Cap at 40 | Untestable BRD |
| Zero user journeys | Cap at 50 | Missing user context |
| P0 blocking questions > 0 | Cap at 55 | Blocking issues present |
| Zero NFRs | Cap at 60 | Missing quality attributes |
| No architecture_handoff.json | Cap at 45 | Not ready for downstream |

If multiple caps apply, use the lowest.

---

## Verdict Thresholds

| Score Range | Verdict | Action |
|---|---|---|
| 75-100 | PASS | Proceed to Architecture Design |
| 50-74 | CONDITIONAL_PASS | Proceed with caveats documented |
| 0-49 | FAIL | Return to BrdAnalyzer for improvement |

---

## Cross-Validation Against BrdAnalyzer Baseline

When `analysis_summary.json` includes pre-computed `quality_scores`:
1. Read BrdAnalyzer's baseline scores (completeness, clarity, testability, traceability)
2. Compare with maturity scorer's independent calculation
3. If any dimension differs by > 15 points, flag as `SCORE_DISCREPANCY` in the report
4. Always use the maturity scorer's calculation as authoritative — but document the discrepancy

---

## Output: maturity-score.json

```json
{
  "scorer_version": "2.0",
  "run_id": "uuid",
  "scored_at": "ISO-8601",
  "brd_run_id": "from analysis_summary.json",
  "overall_score": 0,
  "verdict": "PASS | CONDITIONAL_PASS | FAIL",
  "safety_caps_applied": [],
  "dimensions": {
    "completeness": {"score": 0, "weight": 0.20, "sub_scores": {}},
    "clarity": {"score": 0, "weight": 0.15, "deductions": []},
    "testability": {"score": 0, "weight": 0.15},
    "traceability": {"score": 0, "weight": 0.10},
    "risk_awareness": {"score": 0, "weight": 0.10, "sub_scores": {}},
    "downstream_readiness": {"score": 0, "weight": 0.15, "sub_scores": {}},
    "domain_model_maturity": {"score": 0, "weight": 0.10, "sub_scores": {}},
    "business_rules_coverage": {"score": 0, "weight": 0.05}
  },
  "cross_validation": {
    "brd_analyzer_scores": {},
    "discrepancies": []
  },
  "recommendations": [
    {
      "dimension": "clarity",
      "current_score": 0,
      "target_score": 0,
      "action": "Resolve N ambiguity flags of type vague-quantifier"
    }
  ]
}
```
