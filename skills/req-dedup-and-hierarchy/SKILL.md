---
name: req-dedup-and-hierarchy
description: >
  Load this skill automatically when an FRD or TRD companion document is detected
  alongside a primary BRD. Provides structural fingerprinting for deduplication,
  BRD→FRD refinement hierarchy rules, TRD constraint mapping, and cross-artifact
  conflict detection. Prevents duplicate REQ IDs and maintains BRD as the authority
  document. Never load manually.
---

# Requirement Deduplication and Hierarchy Skill

## Purpose
When multiple document types are present, manage the relationship between them correctly. BRD is always the authority. FRD refines BRD requirements — it does not replace them. TRD contains constraints and implementation decisions — not new functional requirements.

---

## Document Authority Hierarchy

```
BRD (Business Requirements Document)
  ↓ authority
FRD (Functional Requirements Document) — refines BRD requirements
  ↓ constraint
TRD (Technical Requirements Document) — constrains implementation
```

---

## Structural Fingerprinting Algorithm

Before assigning a new REQ ID to any FRD statement, check if it matches an existing BRD requirement.

### Step 1: Normalize both statements
- Lowercase all text
- Remove punctuation except periods
- Canonicalize modal verbs: "must", "shall", "will" → "shall"; "should", "needs to", "expected to" → "should"; "may", "can", "could" → "may"
- Stem domain verbs: "authenticates" → "authenticate", "displays" → "display", "validates" → "validate"

### Step 2: Extract subject:verb:object triplet
- Subject: the first noun phrase (usually "the system", "users", "admin")
- Verb: the main predicate after the modal
- Object: the remainder (what the action is performed on)

### Step 3: Generate fingerprint string
`normalize(subject):normalize(verb):normalize(object)`

### Step 4: Compare fingerprints

| Match level | Condition | Classification |
|---|---|---|
| Exact match | All three triplet parts match | **Definite duplicate** — attach as refinement, no new ID |
| Subject + verb match | Subject and verb match, object differs | **Likely refinement** — attach as refinement with note |
| Verb + object match | Verb and object match, subject differs | **Possible conflict** — flag for review |
| Subject only | Only subject matches | **Independent requirement** — assign new ID |
| No match | No triplet parts match | **New requirement** — assign new ID |

---

## FRD Processing Rules

### For each FRD statement:

1. Run fingerprint comparison against all BRD requirements
2. **If definite duplicate or likely refinement**:
   - Do NOT create a new REQ ID
   - Create a refinement ID: `{parent-REQ-ID}-R{01, 02...}` (e.g., `REQ-F-015-R01`)
   - Attach to parent requirement in `requirements_catalog.json`:
   ```json
   {
     "id": "REQ-F-015",
     "frd_refinements": [
       {
         "refinement_id": "REQ-F-015-R01",
         "source_file": "functional-spec.md",
         "refinement_text": "The FRD statement that refines this requirement",
         "additional_acceptance_criteria": ["Given/When/Then added by FRD"],
         "additional_constraints": ["Specific constraints added by FRD"]
       }
     ]
   }
   ```
3. **If new FRD-only requirement** (no BRD match):
   - Assign a new REQ ID from the counter
   - Set `"frd_only": true` in the requirement
   - Generate P1 question: "Requirement `{REQ-ID}` ('{title}') appears only in the FRD with no corresponding BRD requirement. Should this be elevated to BRD scope, or is it an FRD implementation detail?"

---

## TRD Processing Rules

TRD content falls into three categories. Determine category before processing each TRD statement.

### Category 1: Constraint on existing requirement
**Signal**: TRD statement references a specific capability already in BRD (e.g., "The authentication system shall use OAuth 2.0")
**Action**: Link to existing REQ ID as a constraint, not a new requirement:
```json
{
  "id": "REQ-F-005",
  "trd_constraints": [
    {
      "constraint_id": "ASM-XXX",
      "source_file": "technical-spec.md",
      "constraint_text": "The authentication system shall use OAuth 2.0",
      "constraint_type": "implementation-decision | performance-bound | technology-choice"
    }
  ]
}
```

### Category 2: Architectural assumption
**Signal**: TRD describes a technology decision with no direct requirement link (e.g., "The database will use PostgreSQL 15")
**Action**: Create `ASM-XXX` assumption entry — not a requirement:
```json
{
  "id": "ASM-XXX",
  "assumption": "Database technology: PostgreSQL 15",
  "source": "technical-spec.md",
  "rationale": "Stated in TRD as technical decision",
  "impact_if_wrong": "medium",
  "related_requirements": ["REQ-F-XXX", "REQ-NF-XXX"],
  "validation_needed": false
}
```

### Category 3: TRD contradicts BRD requirement
**Signal**: TRD implementation decision makes a BRD requirement impossible or significantly harder (e.g., BRD says "< 1 second response time" but TRD specifies an architecture known for higher latency)
**Action**: Create `RISK-XXX` entry and flag both documents:
```json
{
  "id": "RISK-XXX",
  "title": "TRD implementation conflicts with BRD performance requirement",
  "description": "BRD REQ-NF-003 requires <1s response; TRD specifies synchronous external API call chain (typical latency 2-5s)",
  "probability": "high",
  "impact": "high",
  "risk_score": 9,
  "mitigation": "Architecture team must resolve: either relax performance requirement or change TRD architecture",
  "related_requirements": ["REQ-NF-003"],
  "source": "derived-from-trd"
}
```

**PROHIBITION**: Never create `REQ-F-XXX` or `REQ-NF-XXX` entries from TRD content. TRD content maps only to: constraint on existing REQ, ASM-XXX assumption, or RISK-XXX risk.

---

## Glossary Deduplication

When multiple documents are present, check for same term with different definitions:

1. Extract all glossary terms from each document
2. For terms appearing in multiple documents, compare definitions
3. If definitions match → single glossary entry with multiple source references
4. If definitions differ → generate P0 question: "Term '{term}' has conflicting definitions across documents: BRD defines it as '{def1}', FRD defines it as '{def2}'. Which definition governs? This ambiguity affects requirements: {REQ-IDs}."

---

## Cross-Document ID Integrity

After processing all companion documents:

1. Every `frd_refinements` entry references an existing REQ ID
2. Every `trd_constraints` entry references an existing REQ ID
3. No orphaned refinement IDs (REQ-F-015-R01 without REQ-F-015 existing)
4. No duplicate REQ IDs across BRD and FRD processing

Run this check before Phase 11 (Traceability Matrix) begins.
