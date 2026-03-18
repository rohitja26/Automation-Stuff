---
name: architecture-validator
description: >
  The only skill loaded by the Architecture Validator Agent. Runs 8 validation
  checks against all architecture artifacts: flow completeness, API consistency,
  schema completeness, assumption detection (hedge-word scan + citation check),
  traceability completeness, technology consistency, TDD coverage, and SDD-flow
  alignment. Produces validation_report.json. Can trigger a fix cycle for
  auto-correctable issues.
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

## Check 6: Technology Consistency

### What to verify
Load `technology_stack.json` and verify every component in the architecture uses only technologies from the confirmed stack.

### 6a. Component-to-stack mapping
For each component in `hld.json → components`:
- Verify the component's technology references (language, framework, database, infrastructure) match entries in `technology_stack.json`
- If a component uses a technology not in the stack: `TECH_NOT_IN_STACK`

### 6b. Constraint compliance
For each technology decision in `technology_stack.json`:
- If `source = "binding_constraint"` or `source = "must_use"`: verify at least one component uses it: `BINDING_TECH_UNUSED`
- If `cannot_use` list exists: verify no component references a banned technology: `BANNED_TECH_USED`

### 6c. Undecided items
- Flag any entry where `status = "UNDECIDED"` or value is blank: `TECH_UNDECIDED`

**Finding types**: `TECH_NOT_IN_STACK` | `BINDING_TECH_UNUSED` | `BANNED_TECH_USED` | `TECH_UNDECIDED`

---

## Check 7: TDD Coverage

### What to verify
Load `tdd.json` (Test Design Document) and verify completeness against requirements and architecture.

### 7a. Requirement → Test mapping
For every MVP functional requirement in `architecture_handoff.json`:
- Verify at least one test suite in `tdd.json` covers this requirement ID: `REQUIREMENT_NO_TEST`

### 7b. Business rule → Unit test mapping
For each rule in `business_rules.json`:
- Verify a corresponding unit test exists in `tdd.json → suites[type=unit]`: `RULE_NO_UNIT_TEST`

### 7c. API endpoint → API test mapping
For each endpoint in `lld.json → api_contracts`:
- Verify a corresponding test exists in `tdd.json → suites[type=api]`: `ENDPOINT_NO_API_TEST`

### 7d. Flow → Integration test mapping
For each flow in `fdd.json → flows`:
- Verify at least one integration test references this flow: `FLOW_NO_INTEGRATION_TEST`

### 7e. NFR → Performance test mapping
For each NFR in `architecture_handoff.json → nfr_targets`:
- Verify a performance or load test targets this NFR: `NFR_NO_PERF_TEST`

**Finding types**: `REQUIREMENT_NO_TEST` | `RULE_NO_UNIT_TEST` | `ENDPOINT_NO_API_TEST` | `FLOW_NO_INTEGRATION_TEST` | `NFR_NO_PERF_TEST`

---

## Check 8: SDD-Flow Alignment

### What to verify
Load `sdd.json` (System Design Document) and cross-reference against FDD and HLD.

### 8a. Flow → Sequence diagram mapping
For every flow in `data_flow_map.json → flows`:
- Verify a corresponding sequence diagram exists in `sdd.json → sequence_diagrams[]`: `FLOW_NO_SEQUENCE_DIAGRAM`

### 8b. Participants match HLD components
For each sequence diagram in `sdd.json`:
- Verify every `participant` matches a component in `hld.json → components`: `SDD_UNKNOWN_PARTICIPANT`

### 8c. API calls match OpenAPI paths
For each API call step in sequence diagrams:
- Verify the called endpoint exists in `api_contracts/` OpenAPI files: `SDD_INVALID_API_CALL`

### 8d. Inter-service communication consistency
For each `inter_service_communication` entry in `sdd.json`:
- Verify the pattern (sync/async/event) is consistent with the HLD component's declared communication pattern: `SDD_PATTERN_MISMATCH`

**Finding types**: `FLOW_NO_SEQUENCE_DIAGRAM` | `SDD_UNKNOWN_PARTICIPANT` | `SDD_INVALID_API_CALL` | `SDD_PATTERN_MISMATCH`

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
  "coverage_metrics": {
    "req_to_flow_coverage": "N/M (percentage)",
    "req_to_test_coverage": "N/M (percentage)",
    "api_hints_to_endpoints": "N/M (percentage)",
    "flows_to_sequence_diagrams": "N/M (percentage)",
    "rules_to_unit_tests": "N/M (percentage)",
    "entities_to_tables": "N/M (percentage)"
  },
  "findings": [
    {
      "finding_id": "VF-001",
      "check": "Check 1 | Check 2 | Check 3 | Check 4 | Check 5 | Check 6 | Check 7 | Check 8",
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
    "traceability_completeness": false,
    "technology_consistency": true,
    "tdd_coverage": true,
    "sdd_flow_alignment": true
  },
  "undecided_items": ["list of any UNDECIDED markers found in artifacts — these block development even if validation passes"],
  "pending_clarifications": ["items from pending_clarifications.json still unresolved"],
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
| `BANNED_TECH_USED` | High | Component uses a technology on cannot_use list |
| `REQUIREMENT_NO_TEST` | High | Must-have requirement with no test coverage |
| `FLOW_NO_SEQUENCE_DIAGRAM` | High | Data flow has no SDD sequence diagram |
| `TECH_NOT_IN_STACK` | Medium | Component uses tech not in technology_stack.json |
| `TECH_UNDECIDED` | Medium | Technology decision still pending |
| `NAMING_INCONSISTENCY` | Medium | Wrong term in artifact — maintainability issue |
| `FLOW_NO_EDGE_CASES` | Medium | Flow covers only happy path |
| `DECISION_BROKEN_CITATION` | Medium | Decision references non-existent requirement |
| `RULE_NO_UNIT_TEST` | Medium | Business rule with no unit test |
| `ENDPOINT_NO_API_TEST` | Medium | API endpoint with no API test |
| `SDD_UNKNOWN_PARTICIPANT` | Medium | SDD participant not in HLD components |
| `SDD_INVALID_API_CALL` | Medium | SDD references API not in OpenAPI specs |
| `SDD_PATTERN_MISMATCH` | Medium | Communication pattern mismatch between SDD and HLD |
| `FLOW_NO_INTEGRATION_TEST` | Low | Flow with no integration test |
| `NFR_NO_PERF_TEST` | Low | NFR with no performance test |
| `INDEX_NO_CITATION` | Low | Index without business justification |
| `BINDING_TECH_UNUSED` | Low | Binding tech specified but not used by any component |
| `VERIFY_MANUALLY` | Low | Cannot be auto-verified |

---

## Auto-Fix Capability

The Validator can auto-fix these finding types without human intervention:
- `NAMING_INCONSISTENCY`: Replace incorrect term with glossary-authoritative term in artifact JSON
- `DECISION_BROKEN_CITATION`: Remove the broken requirement ID from `justified_by[]` and add finding note
- `INDEX_NO_CITATION`: Add `"justified_by": "NEEDS_CITATION"` placeholder
- `COUNT_MISMATCH`: Fix summary count fields to match array lengths

All other findings require the Architecture Design Agent to re-run the relevant step or human architect review.

**Step re-trigger mapping** (when FAILED, re-run only the step that produced the failing artifact):

| Finding source | Re-trigger step |
|---|---|
| Check 1 (Flow issues) | Step 4 (FDD) |
| Check 2 (API issues) | Step 6 (LLD) |
| Check 3 (Schema issues) | Step 6 (LLD) |
| Check 4 (Assumption issues) | Step that produced the flagged artifact |
| Check 5 (Traceability issues) | Step 9 (Decision Log + Traceability) |
| Check 6 (Tech consistency) | Step 2 (Technology Consultation) |
| Check 7 (TDD coverage) | Step 8 (TDD) |
| Check 8 (SDD alignment) | Step 7 (SDD) |

---

## Pass/Fail Criteria

**PASSED**: Zero Critical findings, zero High findings
**PASSED_WITH_WARNINGS**: Zero Critical findings, one or more Medium/Low findings
**FAILED**: One or more Critical or High findings

If FAILED: Report is written. Architecture Design Agent is re-triggered for the specific steps that need revision (not a full re-run). Only the step that produced the failing artifact is re-executed.
