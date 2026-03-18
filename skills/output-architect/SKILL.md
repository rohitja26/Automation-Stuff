---
name: output-architect
description: >
  Load this skill at Step 9 of the Architecture Design Agent (v3.2.2). Handles
  decision log assembly, traceability back-write, and the Master Technical
  Document (MTD) that combines HLD + LLD overview. NOTE: The FDD, SDD, and
  Test Design Document (TDD) are now generated in separate steps (4, 7, 8)
  by the architecture agent directly — this skill no longer generates them.
---

# Output Architect Skill

## Purpose
Finalize all architecture artifacts and write them to `/architecture/`. Also write back the `architecture_components[]` field in `/analysis/traceability_matrix.json` to complete the end-to-end traceability chain.

---

## Artifact 1: Flow Design Document (FDD)

Derive flows from business_rules.json + API contracts. Each flow covers one user journey end-to-end.

### Flow derivation rules

For each MVP requirement that represents a user-visible action, produce one flow. Group related requirements into one flow if they form a single user journey (e.g., all checkout requirements form "Checkout Flow").

**Flow schema**:
```json
{
  "flow_id": "FLOW-001",
  "name": "Vendor Registration Flow",
  "requirement_ids": ["REQ-F-001", "REQ-F-002", "REQ-F-003", "REQ-F-004", "REQ-F-005", "REQ-F-006"],
  "actors": ["Vendor", "System", "OpsTeam", "GSTNApi"],
  "flow_type": "sync | async | mixed",
  "steps": [
    {
      "step": 1,
      "actor": "Vendor",
      "action": "Submits registration form with all mandatory fields",
      "system_response": "System validates fields client-side, submits to POST /vendors",
      "api_called": "API-001"
    },
    {
      "step": 2,
      "actor": "System",
      "action": "Calls GSTN API with submitted GSTN number",
      "system_response": "GSTN validation passes → vendor application stored with status=pending_review",
      "business_rule": "BR-003"
    },
    {
      "step": 3,
      "actor": "System",
      "action": "Notifies ops team of new vendor application",
      "system_response": "Email sent to ops team within 15 minutes",
      "requirement": "REQ-F-004"
    }
  ],
  "happy_path_end": "Vendor receives approval email. Storefront URL active.",
  "edge_cases": [
    {
      "condition": "GSTN already registered on platform",
      "handling": "Return 409 DUPLICATE_GSTN. Registration blocked.",
      "derived_from": "API-001 response 409"
    },
    {
      "condition": "GSTN API is unavailable at submission time",
      "handling": "Return 503. Vendor sees 'Verification service temporarily unavailable, please try again later'. No partial registration stored.",
      "derived_from": "API-001 response 503"
    }
  ],
  "failure_scenarios": [
    {
      "scenario": "Ops team does not review within 2 business days",
      "handling": "System auto-escalates to Vendor Relations Head at T+2. Daily digest of expired applications.",
      "derived_from": "explicit constraint Q-P1-001 (escalation rule)"
    }
  ],
  "async_flows": [
    {
      "trigger": "Ops team approves vendor",
      "background_action": "System creates storefront URL, sends approval email",
      "sla": "Email within 15 minutes of approval"
    }
  ]
}
```

### Mandatory coverage for every flow

- [ ] Happy path (all successful steps)
- [ ] At least 1 edge case (valid but non-standard input)
- [ ] At least 1 failure scenario (external dependency failure or validation failure)
- [ ] Async steps clearly identified and separated from sync steps
- [ ] SLA values from requirements included wherever time constraints exist

---

## Artifact 2: Service Design Document (SDD)

One section per HLD component:

```json
{
  "service_id": "SVC-001",
  "name": "VendorService",
  "hld_component": "COMP-001",
  "responsibility": "All operations related to vendor registration, onboarding, profile, and storefront management",
  "apis_owned": ["API-001", "API-002", "API-003"],
  "data_stores_owned": ["vendors table", "vendor_documents table"],
  "inter_service_communication": [
    {"calls": "NotificationService", "for": "sending approval/rejection emails", "pattern": "async event", "derived_from": "REQ-F-005"}
  ],
  "external_calls": [
    {"system": "GSTN API", "for": "GST number validation", "pattern": "sync REST", "timeout_ms": 5000, "fallback": "return 503", "derived_from": "REQ-F-002"}
  ],
  "slas": [
    {"operation": "vendor registration submission", "target_ms": 2000, "derived_from": "NFR-003 (checkout flow — applies similarly to registration)"}
  ]
}
```

---

## Artifact 3: Master Technical Document (MTD)

The MTD combines HLD + LLD into a single master overview document. It does not add new content — it assembles the two. **NOTE**: This is NOT the Test Design Document (TDD), which is generated separately in Step 8 by the architecture agent.

**MTD.md structure**:
```markdown
# Master Technical Document — {Project Name}
**Version**: 1.0 | **Stage**: Architecture Draft | **Date**: {date}

## 1. Purpose and scope
{2 sentences from project_overview in analysis_summary.json}

## 2. Technology stack
{From technology_stack.json — all confirmed decisions}
| Decision | Value | Source | Justified by |
|---|---|---|---|
| Backend language | Node.js | binding_constraint | Explicit constraint |
| Cloud provider | AWS ap-south-1 | binding_constraint | NFR-018 |
...

If any technology is UNSPECIFIED, show:
| Search engine | UNSPECIFIED — architecture team must decide | | NFR-002 + NFR-012 require decision |

## 3. Architecture overview
{From HLD.md Architecture Overview section}

## 4. System components
{From HLD.md System Components section}

## 5. Domain model
{From domain_model.json — entity list with brief description}

## 6. API summary
{Table: endpoint | method | summary | requirement_ids}

## 7. Database schema summary
{Table: table_name | entity | bounded_context | key_fields}

## 8. Flow summary
{Table: flow_id | name | requirement_ids | actors}

## 9. Test strategy summary
{From TDD.md: Table: suite | type | test_count | coverage}

## 10. Non-functional requirement coverage
{Table: NFR ID | target | addressed by | design decision}

## 11. Security design
{From HLD security architecture section}

## 12. Compliance coverage
{From compliance_radar — regulations and which components address each}

## 13. Decision log
{From decision_log.json — full table}

## 14. Open decisions (UNDECIDED items)
{Any item marked UNDECIDED — these must be resolved before development}

## 15. Future scope
{Phase 2 requirements — acknowledged, not designed}
```

---

## Traceability Back-Write

After all artifacts are produced, update `/analysis/traceability_matrix.json`:

For each requirement ID, populate `architecture_components[]` with the component IDs and artifact references that implement it:

```json
{
  "req_id": "REQ-F-001",
  "architecture_components": [
    {"component": "VendorService", "artifact": "COMP-001 in HLD"},
    {"api_endpoint": "POST /vendors", "artifact": "API-001 in LLD"},
    {"db_table": "vendors", "artifact": "VendorContext schema in LLD"},
    {"flow": "FLOW-001", "artifact": "Vendor Registration Flow in FDD"}
  ]
}
```

Every must-have requirement must have at least 1 `architecture_components` entry. Requirements with empty arrays after this back-write are flagged for the Validator Agent.

---

## Final Artifact Checklist

Before triggering the Validator Agent, verify all of these files exist:

```
/architecture/
  technology_stack.json        ← Technology decisions (Step 2)
  domain_model.json            ← Confirmed entity model (Step 3)
  fdd.json + FDD.md            ← Flow Design Document (Step 4)
  hld.json + HLD.md            ← High-Level Design (Step 5)
  lld.json + LLD.md            ← Low-Level Design (Step 6)
  sdd.json + SDD.md            ← System Design Document (Step 7)
  tdd.json + TDD.md            ← Test Design Document (Step 8)
  decision_log.json            ← All decisions (Step 9)
  MTD.md                       ← Master Technical Document (assembled overview)
  api_contracts/
    {service}.openapi.yaml     ← One per service
  db_schemas/
    {context}.schema.sql       ← One per bounded context
/analysis/traceability_matrix.json  ← UPDATED with architecture_components[]
```
