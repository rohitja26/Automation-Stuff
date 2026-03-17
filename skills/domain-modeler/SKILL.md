---
name: domain-modeler
description: >
  Load this skill at Step 2 of the Architecture Design Agent. Takes the context
  object from the BRD Parser and produces a confirmed domain model: entities,
  relationships, bounded contexts, and business logic constraints. Uses
  domain_model_seed.json as starting point — confirms, rejects, or extends
  candidate entities. All names must come from the glossary naming authority.
  Zero assumptions allowed.
---

# Domain Modeler Skill

## Purpose
Produce `domain_model.json` — the authoritative entity model that ALL subsequent architecture artifacts (HLD, LLD, DB schema, API contracts) must derive from. Any entity name, field name, or relationship not in the domain model must not appear in downstream artifacts.

---

## Step 1: Confirm or Reject Candidate Entities

For each entity in `context.candidate_entities` (from domain_model_seed.json):

**Confirm** if: entity appears as subject or object in 2+ MVP requirements AND has at least 1 candidate field that maps to an explicit requirement field.

**Reject** if: entity appears in glossary only, with no requirement explicitly describing its attributes or behavior. Add to `rejected_entities[]` in domain_model.json with reason.

**Extend** if: confirmed entity is missing fields that are explicitly described in requirements (e.g., requirement mentions "vendor's bank account number" but candidate entity has no financial_info field).

**New entity** if: MVP requirement clearly describes a concept not in the seed (e.g., a concept that is implied by multiple requirements but not in the glossary). Add with `origin: "requirement-derived"` and list the triggering requirement IDs.

---

## Step 2: Field Derivation Rules

For each confirmed entity, derive fields from requirements using these rules:

**Only add a field if**:
- A requirement explicitly describes that attribute (e.g., "vendor must provide GST number" → `gstn: string`)
- An acceptance criterion references it (e.g., "when stock falls below threshold" → `low_stock_threshold: integer`)
- A business rule operates on it (e.g., "commission is 8% of order value" → order needs `total_amount` and `commission_amount`)

**Never add a field because**:
- It is "typical" for this kind of entity in other systems
- You have seen it in similar domain models before
- It seems obvious or logical

**Field schema**:
```json
{
  "field_name": "snake_case_name",
  "data_type": "string | integer | decimal | boolean | datetime | uuid | enum | json | reference",
  "nullable": true,
  "unique": false,
  "derived_from": "REQ-F-XXX or BR-XXX",
  "description": "one sentence from requirement context",
  "enum_values": [],
  "reference_entity": null
}
```

---

## Step 3: Relationship Derivation

Derive relationships from business rules and requirement dependencies only.

**Relationship types and their signals**:

| Signal in requirements | Relationship |
|---|---|
| "A customer places an order" | Customer →(places)→ Order: one-to-many |
| "An order contains products from multiple vendors" | Order →(contains)→ OrderItem: one-to-many |
| "A vendor has multiple products" | Vendor →(owns)→ Product: one-to-many |
| "Many customers can review one product" | Customer →(reviews)→ Product: many-to-many via Review |
| "Each sub-order belongs to one vendor" | SubOrder →(belongs-to)→ Vendor: many-to-one |

**Relationship schema**:
```json
{
  "from_entity": "Order",
  "to_entity": "OrderItem",
  "relationship": "has",
  "cardinality": "one-to-many",
  "derived_from": "REQ-F-028",
  "cascade_delete": true,
  "description": "An order splits into sub-orders per vendor"
}
```

---

## Step 4: Bounded Context Identification

Group entities into bounded contexts based on their primary actor and functional domain (from requirement categories).

**Bounded context rules**:
- Each bounded context owns its entities — no entity belongs to two contexts
- Cross-context references use IDs only (no embedded objects)
- Context name must come from requirement category taxonomy or glossary

**Example grouping** (derived from ShopFlow categories):
```
VendorContext:     Vendor, VendorProfile, VendorDocument, VendorTier
CatalogContext:    Product, ProductVariant, ProductImage, Category
OrderContext:      Order, SubOrder, OrderItem, OrderStatus
PaymentContext:    Payment, Invoice, Payout, PayoutTransaction
CustomerContext:   Customer, Address, Cart, CartItem, Wishlist
ReviewContext:     Review, ReviewFlag, VendorResponse
PromotionContext:  DiscountCode, Campaign, CampaignParticipation
NotificationContext: Notification, NotificationPreference, NotificationLog
```

---

## Step 5: Business Logic Layer

From `context.business_rules`, map each rule to the entity/method that owns it:

```json
{
  "rule_id": "BR-001",
  "owning_entity": "Order",
  "owning_method": "calculateCommission()",
  "logic_summary": "commission = total_amount * 0.08",
  "derived_from": "REQ-F-034 + explicit constraint (8% commission rate)",
  "validation_constraints": ["total_amount must be > 0"]
}
```

State transition rules become explicit state machines:
```json
{
  "entity": "Order",
  "state_field": "status",
  "states": ["New", "Confirmed", "Packed", "Shipped", "Delivered", "Cancelled", "ReturnRequested", "ReturnApproved", "ReturnCompleted"],
  "transitions": [
    {"from": "New", "to": "Confirmed", "trigger": "payment_gateway_callback", "actor": "system"},
    {"from": "Confirmed", "to": "Packed", "trigger": "vendor_action", "actor": "Vendor"},
    {"from": "Packed", "to": "Shipped", "trigger": "vendor_action_with_awb", "actor": "Vendor"},
    {"from": "Shipped", "to": "Delivered", "trigger": "logistics_webhook", "actor": "system"},
    {"from": "New", "to": "Cancelled", "trigger": "customer_or_vendor_action", "actor": "Customer | Vendor"},
    {"from": "Confirmed", "to": "Cancelled", "trigger": "customer_or_vendor_action", "actor": "Customer | Vendor"}
  ],
  "invalid_transitions_note": "Shipped → Cancelled is NOT allowed — must go through return flow",
  "derived_from": "BR-XXX (state-transition rules)"
}
```

---

## Output: domain_model.json

```json
{
  "model_version": "1.0",
  "run_id": "uuid",
  "generated_at": "ISO-8601",
  "naming_authority": "glossary.json",
  "bounded_contexts": [
    {
      "name": "ContextName",
      "description": "one sentence",
      "entities": ["EntityName"],
      "primary_actor": "Vendor | Customer | System | Admin"
    }
  ],
  "entities": [],
  "relationships": [],
  "state_machines": [],
  "business_logic": [],
  "rejected_entities": [
    {"name": "RejectedEntity", "reason": "appears in glossary but no requirement describes its attributes"}
  ],
  "entity_count": 0,
  "relationship_count": 0,
  "bounded_context_count": 0
}
```

---

## Self-Validation Before Proceeding to HLD

After producing domain_model.json, answer these questions. If any answer is NO, fix before proceeding:

- [ ] Every entity has at least 1 field with a `derived_from` requirement or rule ID
- [ ] Every relationship has a `derived_from` requirement ID
- [ ] No entity name contradicts the glossary naming authority
- [ ] Every state machine has transitions covering all valid paths from the business rules
- [ ] Every bounded context has at least 1 entity
- [ ] No field was added because "it is typical" — only because a requirement or rule requires it
