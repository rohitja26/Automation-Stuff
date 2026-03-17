---
name: lld-generator
description: >
  Load this skill at Step 4 of the Architecture Design Agent. Produces API
  contracts (OpenAPI), DB schemas, module structures, validation rules, and
  error handling patterns. Derives everything from domain_model.json and
  hld.json. Every API endpoint traces back to a requirement ID. Every DB field
  traces back to a domain model field. Zero invention allowed.
---

# LLD Generator Skill

## Core Rule: Traceability is Mandatory

Every LLD artifact must have a traceability link:
- API endpoint → `requirement_ids: []` listing every FR it satisfies
- DB table → `entity: "EntityName"` from domain_model.json
- DB field → `derived_from: "domain_model.EntityName.field_name"`
- Validation rule → `source: "BR-XXX"` or `"REQ-F-XXX acceptance criteria"`

If a LLD element has no traceability link, it is an invention. Remove it.

---

## Step 1: API Contract Generation

### Derivation rules

For each MVP requirement that describes a user-facing action, derive one or more API endpoints:

| Requirement signal | API derivation |
|---|---|
| "Customer registers using email" | `POST /auth/register` |
| "System shall authenticate users" | `POST /auth/login`, `POST /auth/refresh`, `POST /auth/logout` |
| "Vendor creates product listing" | `POST /vendors/{id}/products` |
| "Customer searches products" | `GET /products/search?q=&filters=` |
| "Customer adds to cart" | `POST /customers/{id}/cart/items` |
| "System processes checkout" | `POST /orders` |
| "Vendor updates order status" | `PATCH /orders/{id}/status` |

**URL naming rules** (enforced by naming authority from glossary):
- Use glossary terms exactly: `/vendors` not `/sellers`, `/customers` not `/users` (unless glossary says "users")
- RESTful resource naming: plural nouns for collections, singular for nested single resources
- No verbs in URLs except for actions that aren't CRUD: `/auth/login`, `/orders/{id}/cancel`

### API contract schema (OpenAPI-compatible)

```json
{
  "endpoint_id": "API-001",
  "path": "/vendors",
  "method": "POST",
  "summary": "Register a new vendor",
  "requirement_ids": ["REQ-F-001", "REQ-F-002"],
  "business_rules": ["BR-003"],
  "auth_required": false,
  "roles_allowed": ["public"],
  "request": {
    "content_type": "application/json",
    "body": {
      "business_name": {"type": "string", "required": true, "max_length": 200, "derived_from": "REQ-F-001"},
      "gstn": {"type": "string", "required": true, "pattern": "^[0-9]{2}[A-Z]{5}[0-9]{4}[A-Z]{1}[1-9A-Z]{1}Z[0-9A-Z]{1}$", "derived_from": "REQ-F-002"},
      "pan": {"type": "string", "required": true, "pattern": "^[A-Z]{5}[0-9]{4}[A-Z]{1}$", "derived_from": "REQ-F-001"},
      "bank_account_number": {"type": "string", "required": true, "derived_from": "REQ-F-001"},
      "ifsc_code": {"type": "string", "required": true, "pattern": "^[A-Z]{4}0[A-Z0-9]{6}$", "derived_from": "REQ-F-001"},
      "contact_email": {"type": "string", "format": "email", "required": true},
      "contact_phone": {"type": "string", "pattern": "^[6-9][0-9]{9}$", "required": true}
    }
  },
  "responses": {
    "201": {
      "description": "Vendor application submitted",
      "body": {"vendor_id": "uuid", "status": "pending_review", "storefront_url": null}
    },
    "400": {
      "description": "Validation error",
      "body": {"error_code": "VALIDATION_ERROR", "errors": [{"field": "string", "message": "string"}]}
    },
    "409": {
      "description": "GSTN already registered",
      "body": {"error_code": "DUPLICATE_GSTN", "message": "string"}
    },
    "422": {
      "description": "GSTN validation failed with GSTN API",
      "body": {"error_code": "INVALID_GSTN", "message": "string"}
    },
    "503": {
      "description": "GSTN API unavailable",
      "body": {"error_code": "DEPENDENCY_UNAVAILABLE", "message": "string", "retry_after": 60}
    }
  },
  "idempotency": false,
  "rate_limit": "10 per IP per hour"
}
```

### Failure scenarios — mandatory for every endpoint

Every endpoint must have at least:
- One 4xx response for invalid input
- One 5xx response for dependency failure (if endpoint calls external service)
- One response for the "entity not found" case (for GET/PATCH/DELETE)
- One response for authentication failure (for auth-required endpoints)

Failure responses must include: `error_code` (machine-readable constant), `message` (human-readable), and where applicable: `retry_after` (seconds), `field` (for validation errors).

---

## Step 2: Database Schema Generation

### Derivation rules

One table per entity in domain_model.json. Table name = snake_case of entity name (from glossary naming authority).

```json
{
  "table_name": "vendors",
  "entity": "Vendor",
  "bounded_context": "VendorContext",
  "fields": [
    {"column": "id", "type": "UUID", "primary_key": true, "default": "gen_random_uuid()", "derived_from": "standard PK pattern"},
    {"column": "business_name", "type": "VARCHAR(200)", "nullable": false, "derived_from": "domain_model.Vendor.business_name"},
    {"column": "gstn", "type": "VARCHAR(15)", "nullable": false, "unique": true, "index": true, "derived_from": "domain_model.Vendor.gstn"},
    {"column": "pan", "type": "VARCHAR(10)", "nullable": false, "derived_from": "domain_model.Vendor.pan"},
    {"column": "bank_account_encrypted", "type": "TEXT", "nullable": false, "encrypted": true, "encryption_standard": "AES-256", "derived_from": "NFR-010"},
    {"column": "tier", "type": "VARCHAR(20)", "nullable": false, "default": "'basic'", "check": "tier IN ('basic', 'professional')", "derived_from": "explicit constraint Q-P0-002"},
    {"column": "status", "type": "VARCHAR(30)", "nullable": false, "default": "'pending_review'", "check": "status IN ('pending_review', 'active', 'suspended', 'rejected')", "derived_from": "REQ-F-004 to REQ-F-006"},
    {"column": "storefront_slug", "type": "VARCHAR(80)", "nullable": true, "unique": true, "derived_from": "REQ-F-006 + explicit constraint Q-P0-008"},
    {"column": "created_at", "type": "TIMESTAMPTZ", "nullable": false, "default": "NOW()", "derived_from": "NFR-019 audit requirement"},
    {"column": "updated_at", "type": "TIMESTAMPTZ", "nullable": false, "default": "NOW()", "derived_from": "NFR-019 audit requirement"}
  ],
  "indexes": [
    {"name": "idx_vendors_gstn", "columns": ["gstn"], "unique": true, "justified_by": "REQ-F-002 GSTN validation — frequent lookup"},
    {"name": "idx_vendors_status", "columns": ["status"], "justified_by": "ops team queries by status (FR-004 escalation flow)"}
  ],
  "foreign_keys": [],
  "row_level_security": false,
  "audit_log_required": true,
  "justified_by": "NFR-019"
}
```

### Indexing strategy rules (from requirements, not best practice)

Add an index ONLY when:
- A requirement describes filtering or searching by that field (e.g., "search products by category" → index on `category_id`)
- An NFR specifies performance target on an operation that reads this field
- A business rule requires uniqueness (unique index)
- An acceptance criterion states lookup by that field

---

## Step 3: Module Structure

Derive module/service structure from HLD components. Each component becomes a module with defined internal structure:

```json
{
  "module": "VendorModule",
  "hld_component": "COMP-001",
  "internal_structure": {
    "controllers": ["VendorRegistrationController", "VendorProfileController", "VendorDocumentController"],
    "services": ["VendorRegistrationService", "GSTNValidationService", "VendorNotificationService"],
    "repositories": ["VendorRepository", "VendorDocumentRepository"],
    "validators": ["VendorRegistrationValidator", "GST NumberValidator"],
    "events_emitted": ["VendorRegistered", "VendorApproved", "VendorRejected"],
    "events_consumed": []
  },
  "dependencies": ["NotificationModule", "AuthModule"],
  "external_calls": ["GSTN API (REQ-F-002)"],
  "justified_by": ["REQ-F-001 through REQ-F-006"]
}
```

Class/service names must use glossary terms. `VendorService` not `SellerService`. `GSTNValidationService` not `TaxValidationService` (GSTN is in the glossary).

---

## Step 4: Validation Rules

Extract validation rules from acceptance criteria and business rules:

```json
{
  "validation_id": "VAL-001",
  "entity": "Vendor",
  "field": "gstn",
  "rules": [
    {"rule": "format", "pattern": "^[0-9]{2}[A-Z]{5}[0-9]{4}[A-Z]{1}[1-9A-Z]{1}Z[0-9A-Z]{1}$", "error_code": "INVALID_GSTN_FORMAT"},
    {"rule": "external-validation", "service": "GSTN API", "error_code": "GSTN_INACTIVE_OR_INVALID", "error_on_service_unavailable": "GSTN_SERVICE_UNAVAILABLE"}
  ],
  "derived_from": "REQ-F-002 acceptance criteria"
}
```

---

## LLD Output Files

```
/architecture/lld.json            ← Full LLD machine-readable
/architecture/LLD.md              ← Human-readable version
/architecture/api_contracts/
  {service_name}.openapi.yaml     ← One OpenAPI YAML per service
/architecture/db_schemas/
  {context_name}.schema.sql       ← One schema file per bounded context
```

**LLD.md structure**:
```markdown
# Low-Level Design — {Project Name}
## API contracts summary (full specs in /api_contracts/)
## Database schema summary (full schemas in /db_schemas/)
## Module structure
## Validation rules catalogue
## Error handling patterns
## Interface definitions
```
