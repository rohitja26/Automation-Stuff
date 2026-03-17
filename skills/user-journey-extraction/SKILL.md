---
name: user-journey-extraction
description: >
  Load this skill when user-facing flows are described in the BRD or UI mockup
  images are detected. Extracts structured user journeys from BRD narratives,
  acceptance criteria, and UI mockup analysis. Produces user_journeys.json with
  step-by-step flows, screen references, error paths, and implied service
  boundaries. Also detects reusable UI components across mockup screens.
---

# User Journey Extraction Skill

## Purpose
Extract structured, step-by-step user journeys from BRD narratives, acceptance criteria, and UI mockup analysis. Each journey maps a complete user workflow from trigger to completion — including happy path, error paths, screens involved, data entities touched, and services implied. The output feeds directly into HDD (service design) and FDD (feature boundary definition) downstream agents.

---

## Step 1: Identify Journey Candidates

Scan requirements in `/analysis/requirements_catalog.json` for journey signals:

**Journey trigger patterns** (any of these implies a user journey exists):
- User story clusters: multiple `REQ-F-XXX` entries sharing the same actor AND related business objective
- Screen-to-screen references: requirements that reference different UI mockup screens in sequence
- Multi-step acceptance criteria: Given/When/Then chains that describe sequential actions
- Flow keywords in requirement descriptions: "after", "then", "next", "proceeds to", "redirected to", "navigates to", "completes"
- State transitions: requirements describing status changes (created → pending → approved → completed)

**Journey boundary detection**:
- A journey STARTS when a user initiates an action (clicks, submits, navigates, logs in)
- A journey ENDS when: (a) the user achieves their goal, (b) an error terminates the flow, or (c) the user is handed off to another actor/system
- If a requirement chain crosses actor boundaries, split into separate journeys linked by `handoff_to_journey`

---

## Step 2: Build Journey Steps

For each identified journey, construct the step sequence:

1. **Read related requirements** one at a time from disk (context-management discipline)
2. **Order steps** chronologically based on:
   - Explicit sequence language ("first", "then", "after", "next")
   - Logical dependency (can't checkout before adding to cart)
   - UI mockup screen order (if screens are numbered or show navigation arrows)
3. **For each step, extract**:
   - `action`: what the user does (verb + object)
   - `screen`: which UI mockup image shows this step (from `ui_screens_referenced`)
   - `requirements`: which REQ-IDs are satisfied by this step
   - `data_input`: what data the user provides (form fields, selections, uploads)
   - `data_output`: what data is created or modified (entities written)
   - `system_response`: what the system does after this step (validation, processing, redirect)

---

## Step 3: Extract Error & Alternate Paths

For each step in a journey, identify error conditions:

**Error detection sources**:
- Acceptance criteria with "Given [error condition]" patterns
- Requirements mentioning validation, error messages, or failure handling
- UI mockups showing error states, validation messages, or error pages
- Business rules with failure scenarios

**For each error path**:
```json
{
  "error_id": "UJ-001-ERR-01",
  "at_step": 2,
  "condition": "Email already registered",
  "expected_behavior": "Display error message with login link",
  "recovery_action": "User clicks login link or enters different email",
  "requirement": "REQ-F-003",
  "severity": "recoverable | terminal | redirect"
}
```

**Alternate paths** (not errors, but different valid routes):
- Optional steps (e.g., "Apply coupon code" during checkout)
- Conditional branches (e.g., "If existing customer → skip registration")
- A/B flow variants mentioned in BRD

---

## Step 4: Detect Reusable UI Components

While analyzing UI mockups across journeys, identify UI components that appear on multiple screens:

| Component Pattern | How to Detect |
|---|---|
| Navigation header | Same menu items visible across multiple mockup images |
| Sidebar navigation | Consistent side panel across screens |
| Data table | Tabular data display with consistent column headers |
| Form component | Input fields with labels following consistent pattern |
| Modal/dialog | Overlay pattern appearing in contextual actions |
| Card layout | Repeated card patterns across list views |
| Footer | Consistent bottom section across pages |

**Record each component**:
```json
{
  "component_id": "UC-001",
  "name": "Navigation Header",
  "type": "navigation | form | display | feedback | layout",
  "appears_in_screens": ["mockup-dashboard.png", "mockup-settings.png", "mockup-profile.png"],
  "related_requirements": ["REQ-F-050"],
  "interactive_elements": ["logo-home-link", "menu-items", "user-avatar-dropdown", "notification-bell"],
  "reuse_count": 5
}
```

---

## Step 5: Infer Service Boundaries

For each journey, analyze which logical services are implied by the step sequence:

**Service inference rules**:
- Steps involving authentication → `auth-service`
- Steps involving payment → `payment-service`
- Steps involving notifications (email, SMS, push) → `notification-service`
- Steps involving file upload/download → `storage-service`
- Steps involving search/filtering → `search-service`
- Steps involving CRUD on a specific entity → `{entity}-service` (e.g., `order-service`)
- Steps involving external APIs → `{integration}-adapter`

Record in `services_implied` array on each journey.

---

## Output: user_journeys.json

Write to `/analysis/user_journeys.json`:

```json
{
  "journeys_version": "1.0",
  "run_id": "uuid",
  "generated_at": "ISO-8601",
  "journeys": [{
    "journey_id": "UJ-001",
    "name": "User Registration Flow",
    "actor": "New User",
    "trigger": "User clicks 'Sign Up' on landing page",
    "steps": [
      {
        "step": 1,
        "action": "Fill registration form",
        "screen": "mockup-signup.png",
        "requirements": ["REQ-F-001"],
        "data_input": ["name", "email", "password", "confirm_password"],
        "data_output": [],
        "system_response": "Client-side validation of form fields"
      }
    ],
    "happy_path": true,
    "error_paths": [],
    "alternate_paths": [],
    "data_entities_touched": ["User", "VerificationToken"],
    "services_implied": ["auth-service", "email-service"],
    "related_requirements": ["REQ-F-001", "REQ-F-002", "REQ-F-003"],
    "handoff_to_journey": null,
    "estimated_step_count": 3
  }],
  "ui_components": [],
  "journey_count": 0,
  "component_count": 0
}
```

**Validation rules**:
- Every journey step must reference at least 1 valid REQ-ID from `requirements_catalog.json`
- Every `screen` value must match a filename in `intake-manifest.json` (if UI mockups exist)
- No orphan journeys (every journey must have at least 2 steps)
- `journey_count` must equal actual array length
