---
name: architecture-validator
description: >
  The only skill loaded by the Architecture Validator Agent. Runs 5 validation
  checks against all architecture artifacts: flow completeness, API consistency,
  schema completeness, assumption detection (hedge-word scan + citation check),
  and traceability completeness. Produces validation_report.json. Can trigger
  a fix cycle for auto-correctable issues.
---

# Architecture Validator Skill

## Purpose
Independently verify every architecture artifact against two criteria:
1. Every design element traces back to a requirement, rule, or explicit constraint
2. Every must-have requirement is addressed by at least one design element

The Validator Agent reads architecture artifacts and the /analysis/ output files. It does NOT re-read the BRD or answered-questions files — it works from the structured outputs only.

---

## Check 1: Flow Completeness

### What to verify
For every `REQ-F-XXX` in `architecture_handoff.json → mvp_requirements`:
- [ ] A flow in `fdd.json` includes this requirement ID in its `requirement_ids[]`
- [ ] That flow has at least 1 `edge_cases[]` entry
- [ ] That flow has at least 1 `failure_scenarios[]` entry
- [ ] Every acceptance criterion in `requirements_catalog.json` for this requirement maps to a step, edge case, or failure scenario in the flow

**Finding type**: `FLOW_MISSING` | `FLOW_NO_EDGE_CASES` | `FLOW_NO_FAILURE_SCENARIO` | `ACCEPTANCE_CRITERIA_UNCOVERED`

### Special case: async flows
Any requirement with `category: notification` or any acceptance criteria containing "within X minutes/hours" must have a corresponding `async_flows[]` entry in its flow. If missing: `ASYNC_FLOW_MISSING`.

---

## Check 2: API Consistency

### 2a. Orphan requirements — requirements with no API
For each MVP functional requirement (`type: functional`):
- Find matching endpoint(s) in `lld.json → api_contracts` where `requirement_ids[]` includes this REQ ID
- If none found AND requirement describes a user-visible action (not purely background system behavior): `ORPHAN_REQUIREMENT`

### 2b. Orphan endpoints — APIs with no requirement
For each endpoint in `api_contracts`:
- Verify `requirement_ids[]` is non-empty
- Verify every ID in `requirement_ids[]` exists in `requirements_catalog.json`
- If `requirement_ids` is empty: `ORPHAN_ENDPOINT`
- If a requirement ID in the list doesn't exist: `BROKEN_REQUIREMENT_REFERENCE`

### 2c. Naming consistency
For every path segment, request field, and response field in API contracts:
- Check if the term matches or is clearly derived from a glossary term
- Flag if a synonym is used: e.g., path uses `/sellers` when glossary says `Vendor` → `NAMING_INCONSISTENCY`
- Flag if an abbreviation not in glossary is used: `/vnd` instead of `/vendors` → `NAMING_INCONSISTENCY`

### 2d. Schema mismatch
For each request/response field in API contracts:
- Check if a corresponding field exists in the DB schema for the primary entity
- Flag if a field is in the API but not in any DB table AND not marked as computed/derived → `SCHEMA_API_MISMATCH`

---

## Check 3: Schema Completeness

### 3a. Entity coverage
For each entity in `domain_model.json → entities`:
- Verify a corresponding table exists in db_schemas
- If missing: `ENTITY_NO_TABLE`

### 3b. Field coverage
For each field in each entity:
- Verify the corresponding column exists in the DB table
- If missing: `ENTITY_FIELD_NO_COLUMN`
- If column exists but type is incompatible (e.g., entity says `datetime`, column is `VARCHAR`) → `FIELD_TYPE_MISMATCH`

### 3c. Relationship coverage
For each relationship in `domain_model.json → relationships`:
- Verify a foreign key or join table exists in the DB schema
- If missing: `RELATIONSHIP_NO_FK`

### 3d. Compliance field coverage
For each `compliance_flags[]` value on any requirement:
- Verify the entity/table handling that requirement has appropriate fields:
  - GDPR: `deleted_at` field (soft delete) + `consent_given_at` field on user-facing entity
  - PCI-DSS: no raw card data fields anywhere in any schema — presence of card number, CVV fields = `PCI_VIOLATION`
  - HIPAA: PHI fields must have `encrypted: true` in schema
  - Data residency: no schema elements indicating non-Indian storage (this is an infrastructure concern — flag as `VERIFY_MANUALLY`)

---

## Check 4: Assumption Detection (DUAL METHOD)

### 4a. Hedge-word scan
Scan ALL text fields in all architecture artifact JSON files and markdown files for these phrases:

**Hard flags** (always wrong — design cannot proceed with these):
- "typically", "usually", "generally", "commonly", "often", "standard practice", "best practice", "industry standard", "common approach", "recommended"

**Soft flags** (require citation — may be valid if a requirement ID is nearby):
- "may", "might", "could", "should consider", "it is advisable", "it would be good to"

For each hit, record: `{file, field_path, phrase_found, surrounding_text_snippet}`

### 4b. Citation check
For every decision in `decision_log.json`:
- Verify `justified_by[]` is non-empty
- Verify every requirement ID in `justified_by[]` exists in `requirements_catalog.json`
- If `justified_by` is empty: `DECISION_NO_CITATION`
- If a requirement ID doesn't exist: `DECISION_BROKEN_CITATION`

For every component in `hld.json → components`:
- Verify `justified_by[]` is non-empty: `COMPONENT_NO_CITATION`

For every DB table in db_schemas:
- Verify `justified_by` field exists: `TABLE_NO_CITATION`

For every index in db_schemas:
- Verify `justified_by` field exists: `INDEX_NO_CITATION`

---

## Check 5: Traceability Completeness

For each `REQ-F-XXX` in `architecture_handoff.json → mvp_requirements`:
- Read `traceability_matrix.json` entry for this requirement
- Verify `architecture_components[]` is non-empty: `REQUIREMENT_NOT_IMPLEMENTED`
- Verify at least one entry is a component reference, one is an API reference, and one is a DB or flow reference (complete coverage): `REQUIREMENT_PARTIALLY_COVERED`

For each compliance requirement in `architecture_handoff.json → compliance_requirements`:
- Verify at least one component in HLD has this regulation in its `nfr_constraints[]` or security section: `COMPLIANCE_NOT_ADDRESSED`

---

## Validation Report Schema

```json
{
  "report_version": "1.0",
  "run_id": "uuid",
  "validated_at": "ISO-8601",
  "architecture_run_id": "uuid from hld.json",
  "overall_status": "PASSED | FAILED | PASSED_WITH_WARNINGS",
  "summary": {
    "total_findings": 0,
    "critical": 0,
    "high": 0,
    "medium": 0,
    "auto_fixable": 0,
    "manual_required": 0
  },
  "findings": [
    {
      "finding_id": "VF-001",
      "check": "Check 1 | Check 2 | Check 3 | Check 4 | Check 5",
      "type": "FLOW_MISSING | ORPHAN_ENDPOINT | SCHEMA_API_MISMATCH | DECISION_NO_CITATION | REQUIREMENT_NOT_IMPLEMENTED | ...",
      "severity": "critical | high | medium | low",
      "artifact_file": "fdd.json",
      "artifact_path": "flows[2].requirement_ids",
      "affected_requirement": "REQ-F-051",
      "description": "REQ-F-051 has no flow coverage in fdd.json",
      "auto_fixable": false,
      "fix_suggestion": "Add REQ-F-051 to an existing flow or create FLOW-012 covering the review moderation requirement"
    }
  ],
  "checks_passed": {
    "flow_completeness": true,
    "api_consistency": false,
    "schema_completeness": true,
    "assumption_detection": true,
    "traceability_completeness": false
  },
  "undecided_items": ["list of any UNDECIDED markers found in artifacts — these block development even if validation passes"],
  "manual_review_required": ["items flagged VERIFY_MANUALLY"]
}
```

---

## Severity Classification

| Finding type | Severity | Reason |
|---|---|---|
| `REQUIREMENT_NOT_IMPLEMENTED` | Critical | Must-have requirement with no design |
| `PCI_VIOLATION` | Critical | Raw card data in schema = compliance violation |
| `DECISION_NO_CITATION` | Critical | Assumption in design = violates core principle |
| `FLOW_MISSING` | High | User journey with no design coverage |
| `ORPHAN_ENDPOINT` | High | API with no business justification |
| `SCHEMA_API_MISMATCH` | High | API promises data that DB cannot provide |
| `COMPLIANCE_NOT_ADDRESSED` | High | Regulation with no design coverage |
| `NAMING_INCONSISTENCY` | Medium | Wrong term in artifact — maintainability issue |
| `FLOW_NO_EDGE_CASES` | Medium | Flow covers only happy path |
| `DECISION_BROKEN_CITATION` | Medium | Decision references non-existent requirement |
| `INDEX_NO_CITATION` | Low | Index without business justification |
| `VERIFY_MANUALLY` | Low | Cannot be auto-verified |

---

## Auto-Fix Capability

The Validator can auto-fix these finding types without human intervention:
- `NAMING_INCONSISTENCY`: Replace incorrect term with glossary-authoritative term in artifact JSON
- `DECISION_BROKEN_CITATION`: Remove the broken requirement ID from `justified_by[]` and add finding note
- `INDEX_NO_CITATION`: Add `"justified_by": "NEEDS_CITATION"` placeholder

All other findings require the Architecture Design Agent to re-run the relevant step or human architect review.

---

## Pass/Fail Criteria

**PASSED**: Zero Critical findings, zero High findings
**PASSED_WITH_WARNINGS**: Zero Critical findings, one or more Medium/Low findings
**FAILED**: One or more Critical or High findings

If FAILED: Report is written. Architecture Design Agent is re-triggered for the specific steps that need revision (not a full re-run). Only the step that produced the failing artifact is re-executed.
