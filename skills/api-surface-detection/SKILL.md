---
name: api-surface-detection
description: >
  Load this skill when functional requirements implying system behavior are
  detected (nearly always). Identifies implied API endpoints from requirements,
  user journeys, and UI mockup interactions. Produces api_surface_hints.json
  with HTTP methods, resource paths, input/output entities, and auth requirements.
  These are hints for the architecture agent — not final API contracts.
---

# API Surface Detection Skill

## Purpose
Identify implied API endpoints from functional requirements. Every requirement that describes system behavior ("user shall be able to search products", "system shall send email notification") implies one or more API operations. This skill extracts those API surface hints so the HDD/LLD downstream agents can design the actual API contracts without guessing.

---

## Step 1: Scan Requirements for API Signals

Read from `/analysis/requirements_catalog.json`. For each functional requirement, check for API-implying patterns:

**Strong API signals** (high confidence):
| Pattern | Implied Operation | HTTP Method |
|---|---|---|
| "create", "register", "add", "submit", "upload" | Create resource | POST |
| "view", "display", "show", "list", "search", "retrieve", "get" | Read resource | GET |
| "update", "edit", "modify", "change" | Update resource | PUT / PATCH |
| "delete", "remove", "cancel", "deactivate" | Delete resource | DELETE |
| "login", "authenticate", "sign in" | Auth operation | POST |
| "export", "download" | Read + transform | GET |
| "import", "bulk upload" | Batch create | POST |
| "approve", "reject", "verify" | State transition | PATCH / POST |
| "send notification", "trigger email", "push alert" | Async action | POST |

**Weak API signals** (medium confidence — need corroboration):
- "the system shall" + any verb → likely an API operation
- Acceptance criteria with "Given/When/Then" → the "When" clause often implies an API call
- UI mockup buttons with action labels → each button likely triggers an API call

---

## Step 2: Construct Endpoint Hints

For each detected API operation, construct the endpoint hint:

1. **Resource identification**: Extract the primary entity from the requirement's `data_entities_involved` field
2. **Path construction**: Follow REST conventions — `/{resource-plural}` for collections, `/{resource-plural}/{id}` for specific items
3. **Method mapping**: Use the pattern table from Step 1
4. **Input entities**: What data the client sends (from requirement's form fields, user inputs)
5. **Output entities**: What data the server returns (from requirement's expected outcomes)
6. **Auth requirement**: Does this endpoint need authentication? Check if requirement references roles or has access-control related context

**Endpoint grouping**: Group related endpoints by resource. Multiple requirements acting on the same entity should produce endpoints on the same resource path.

---

## Step 3: Cross-Reference with User Journeys

If `/analysis/user_journeys.json` exists (Phase 6 completed), cross-reference:
- Each journey step that involves system interaction likely maps to an API call
- Link each endpoint hint to the journey step that triggers it via `source_journey`
- This provides the architecture agent with the call sequence, not just individual endpoints

---

## Step 4: Detect Non-CRUD Patterns

Not all API endpoints are simple CRUD. Detect special patterns:

| Pattern | Signal | Endpoint Style |
|---|---|---|
| Search/filter with multiple criteria | "search", "filter", "query" | `GET /resource?query=...` or `POST /resource/search` |
| Batch operations | "bulk", "batch", "import", "export" | `POST /resource/batch` |
| Real-time updates | "real-time", "live", "streaming" | WebSocket or SSE endpoint |
| File operations | "upload", "download", "attachment" | `POST /resource/{id}/files`, `GET /resource/{id}/files/{fileId}` |
| Aggregation/reporting | "dashboard", "report", "analytics", "summary" | `GET /analytics/...` or `GET /reports/...` |
| Webhooks/callbacks | "notify", "callback", "webhook" | `POST /webhooks/...` |

---

## Step 5: Link to NFRs

For each API endpoint hint, check if related non-functional requirements apply:
- Performance requirements → add to `related_nfrs` (the architecture agent needs to know which endpoints have SLAs)
- Security requirements → flag auth level (public, authenticated, admin-only)
- Rate limiting mentioned → note in endpoint hint

---

## Output: api_surface_hints.json

Write to `/analysis/api_surface_hints.json`:

```json
{
  "api_version": "1.0",
  "run_id": "uuid",
  "generated_at": "ISO-8601",
  "derivation_note": "These are hints inferred from requirements — architecture agent confirms, modifies, or rejects each.",
  "endpoints": [{
    "hint_id": "API-001",
    "implied_endpoint": "POST /api/auth/register",
    "http_method_hint": "POST",
    "resource": "User",
    "operation": "create",
    "source_requirement": "REQ-F-001",
    "source_journey": "UJ-001",
    "actors": ["New User"],
    "input_entities": ["RegistrationData"],
    "output_entities": ["UserProfile", "AuthToken"],
    "auth_required": false,
    "auth_level": "public | authenticated | role-specific",
    "related_nfrs": ["REQ-NF-001"],
    "special_pattern": null,
    "confidence": "high | medium | low",
    "notes": "Registration endpoint — no auth required"
  }],
  "endpoint_count": 0,
  "by_resource": {},
  "by_method": {"GET": 0, "POST": 0, "PUT": 0, "PATCH": 0, "DELETE": 0}
}
```

**Validation rules**:
- Every endpoint must reference a valid `source_requirement` from `requirements_catalog.json`
- `endpoint_count` must equal actual array length
- `by_method` counts must sum to `endpoint_count`
- No duplicate endpoints (same method + same path = merge into one with multiple source requirements)
