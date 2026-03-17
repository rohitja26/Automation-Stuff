---
name: data-flow-analysis
description: >
  Load this skill when the domain model has more than 3 entities (non-trivial
  data model). Maps how data flows between entities, services, and external
  systems based on requirements and user journeys. Produces data_flow_map.json
  with step-by-step data movement chains. Critical input for the LLD agent
  to design service interactions, message queues, and data pipelines.
---

# Data Flow Analysis Skill

## Purpose
Map how data moves between entities, services, and external systems. While `domain_model_seed.json` captures what entities exist and their relationships, and `user_journeys.json` captures what users do, this skill captures how data actually flows through the system — the read/write chains, event propagation, and cross-service data dependencies. This is the most critical input for the LLD agent designing service interactions.

---

## Step 1: Identify Data Flow Triggers

Read from `/analysis/user_journeys.json` and `/analysis/api_surface_hints.json`.

Each data flow is triggered by one of:
1. **User action** → mapped from a user journey step
2. **System event** → background process, scheduled job, external webhook
3. **Data change** → cascading update when one entity changes

For each journey in `user_journeys.json`, identify steps where data is created, modified, or consumed. Each such step is a potential data flow trigger.

---

## Step 2: Trace Data Movement

For each trigger, trace the complete data flow from source to final destination:

**Flow construction rules**:
1. Start at the entry point (frontend, API gateway, external webhook)
2. Follow the data through each processing step
3. For each step, record: source component, destination component, data transferred, operation type
4. End at the final storage or output destination

**Component types** (use these as `from`/`to` values):
- `Frontend` — client-side application
- `API Gateway` — request entry point
- `{Entity} Service` — business logic service (inferred from entity name)
- `Database` — primary data store
- `Cache` — caching layer (if detected from requirements)
- `{External} Gateway` — external service adapter (payment, email, SMS)
- `Message Queue` — async communication (if event-driven pattern detected)
- `Notification Service` — handles email/SMS/push delivery
- `File Storage` — for upload/download flows
- `Search Index` — if search service is involved

---

## Step 3: Detect Data Flow Patterns

Categorize each flow by its pattern:

| Pattern | Detection Signal | Architecture Implication |
|---|---|---|
| **Request-Response** | Synchronous CRUD operations, simple read/write | Standard REST API |
| **Write-Heavy** | Multiple writes per user action, data transformations | Consider write-optimized storage, CQRS |
| **Read-Heavy** | Dashboard, reporting, search queries | Consider caching, read replicas |
| **Event-Driven** | Notifications, cascading updates, async processing | Consider message queues, event bus |
| **Saga/Orchestrated** | Multi-step transactions, payment flows, order processing | Consider saga pattern, compensating transactions |
| **Batch** | Import, export, report generation, data migration | Consider job queue, background workers |
| **Real-Time** | Chat, live updates, collaborative editing | Consider WebSocket, SSE |

---

## Step 4: Identify Cross-Service Dependencies

From the traced flows, build a service dependency map:

For each pair of services that exchange data:
- Record the direction (A → B, B → A, or bidirectional)
- Record what data is exchanged
- Record if the communication is synchronous (blocking) or asynchronous (non-blocking)
- Flag circular dependencies as risks

---

## Step 5: Build Flow Map

Write to `/analysis/data_flow_map.json`:

```json
{
  "flow_version": "1.0",
  "run_id": "uuid",
  "generated_at": "ISO-8601",
  "derivation_note": "Data flows inferred from requirements, user journeys, and API surface hints. Architecture agent validates and refines.",
  "flows": [{
    "flow_id": "DF-001",
    "name": "User Registration Flow",
    "trigger": "User submits registration form",
    "trigger_type": "user_action | system_event | data_change",
    "source_journey": "UJ-001",
    "source_requirement": "REQ-F-001",
    "steps": [
      {
        "step": 1,
        "from": "Frontend",
        "to": "Auth Service",
        "data": ["RegistrationData"],
        "operation": "create",
        "api_hint": "API-001",
        "sync": true
      },
      {
        "step": 2,
        "from": "Auth Service",
        "to": "Database",
        "data": ["User"],
        "operation": "write",
        "api_hint": null,
        "sync": true
      },
      {
        "step": 3,
        "from": "Auth Service",
        "to": "Notification Service",
        "data": ["VerificationEmail"],
        "operation": "publish",
        "api_hint": null,
        "sync": false
      }
    ],
    "entities_involved": ["User", "VerificationToken"],
    "services_involved": ["Auth Service", "Database", "Notification Service"],
    "pattern": "event-driven",
    "has_async_step": true,
    "failure_points": [
      {"at_step": 2, "failure": "Email uniqueness violation", "handling": "Return 409 Conflict"},
      {"at_step": 3, "failure": "Email service unavailable", "handling": "Queue for retry"}
    ]
  }],
  "service_dependencies": [
    {
      "from_service": "Auth Service",
      "to_service": "Notification Service",
      "data_exchanged": ["VerificationEmail", "PasswordResetEmail"],
      "communication_type": "async",
      "flows_using": ["DF-001", "DF-005"]
    }
  ],
  "flow_count": 0,
  "patterns_detected": {
    "request_response": 0,
    "write_heavy": 0,
    "read_heavy": 0,
    "event_driven": 0,
    "saga": 0,
    "batch": 0,
    "real_time": 0
  }
}
```

---

## Validation Rules

- Every flow must reference a valid `source_journey` from `user_journeys.json` OR a valid `source_requirement`
- Every `api_hint` must reference a valid ID from `api_surface_hints.json`
- Every entity in `entities_involved` must exist in `domain_model_seed.json`
- `flow_count` must equal actual array length
- `patterns_detected` counts must sum to `flow_count`
- No step can have `from` and `to` pointing to the same component
- Circular service dependencies must be flagged as `RISK-XXX` entries
