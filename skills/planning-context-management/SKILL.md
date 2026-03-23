---
name: planning-context-management
description: Hierarchical context loading strategy for PlanningDesign-3.3. The architecture files for a full-stack project are large and deeply interconnected — loading all of them at once causes context overflow and cross-reference confusion. This skill defines a 3-level hierarchical approach: Index Layer (always in context), Module Layer (loaded one module at a time), and Task Layer (loaded for individual task planning). Used exclusively by PlanningDesign-3.3. Do NOT use for BrdAnalyzer — that agent has its own context-management skill.
version: 1.0
used-by: PlanningDesign-3.3
---

# Planning Context Management Skill

## Why This Skill Exists

BrdAnalyzer processes one document sequentially. PlanningDesign-3.3 processes a fundamentally different workload: **8+ interconnected architecture files simultaneously**, with cross-file references at every step.

The naive approach — load all 8 architecture files into context at once — fails for two reasons:

1. **Context overflow.** A fully specified full-stack architecture across 8 files can easily exceed 50,000 tokens. Loading everything at once pushes earlier content out of the context window before it is needed.
2. **Reference confusion.** When the LLM holds `hld.json`, `api_contracts.json`, `database_schema.json`, `security_design.json`, `frontend_design.json`, and `tech_stack.json` all at once, it loses track of which field came from which file. Architecture references in task outputs become inaccurate.

The solution is a **3-level hierarchical loading strategy** that keeps a compact summary always in context, loads one module at a time when planning tasks, and drills to the task level when finalizing individual task details.

---

## The 3 Levels

```
Level 1 — Index Layer        (~500–800 tokens, always in context)
  What it contains: Module registry, dependency graph, technology summary, phase sequence
  When loaded: Once at the start of planning. Never unloaded.
  Purpose: Global orientation. Answers "what modules exist, in what order, with what tech?"

Level 2 — Module Layer       (~2,000–4,000 tokens, one module at a time)
  What it contains: Full details for one module: its architecture component definition,
                    its API endpoints, its frontend design spec, its security config
  When loaded: When PlanningDesign begins planning that specific module
  Unloaded: When that module's task files are written and the next module begins
  Purpose: Complete context for one module's backend + frontend planning

Level 3 — Task Layer         (~500–1,000 tokens, one task at a time)
  What it contains: The specific architecture section relevant to one task
  When loaded: When finalizing acceptance criteria, architecture_references, and rollback_scope for a task
  Unloaded: After the task entry is written to the task file
  Purpose: Precision. Ensures exact JSON paths in architecture_references are correct.
```

---

## Level 1 — Index Layer

### What to Extract

Read all 8 architecture files once at the start of planning. Extract ONLY the index-level information. Do not keep the full file content in your working memory — extract the summary and let the full file content recede.

**From `hld.json` — extract:**
- Array of module names, IDs, types (`full_stack` / `backend_only` / `frontend_only`)
- Module dependency relationships (which module depends on which)
- Module processing order (topological sort of the dependency graph)

**From `tech_stack.json` — extract:**
- Backend runtime name + version
- Backend framework name + version
- Frontend framework name + version
- State management library name + version
- HTTP client name + version
- Test runner names (backend + frontend)
- Build tool name + version

**From `api_contracts.json` — extract:**
- Count of total endpoints
- List of endpoint paths grouped by module (path + method only, not full schemas)

**From `database_schema.json` — extract:**
- Table names and which module owns each table
- Foreign key relationships between tables (for migration ordering)

**From `frontend_design.json` — extract:**
- Route list (path + component name only)
- State slice names and which module owns each
- User journey names

**From `security_design.json` — extract:**
- Auth strategy name (JWT / session / OAuth)
- List of protected routes
- Token storage strategy (memory / localStorage / httpOnly cookie)

**From `infra_design.json` — extract:**
- List of all environment variable names (backend + frontend, not values)
- CI/CD tool name
- Container tool (Docker / Podman / none)

**From `domain_model.json` — extract:**
- Entity names and their direct relationships

### Index Layer Output Format

Write the index to a working memory object. Reference it throughout planning without reloading architecture files.

```
INDEX LAYER (working memory — do not write to disk):

MODULES (processing order):
  1. MOD-001_environment_setup      [backend_only]  deps: none
  2. MOD-002_foundation             [full_stack]     deps: MOD-001
  3. MOD-003_authentication         [full_stack]     deps: MOD-002
  4. MOD-004_user_management        [full_stack]     deps: MOD-003
  5. MOD-005_product_catalog        [full_stack]     deps: MOD-002
  6. MOD-006_shopping_cart          [full_stack]     deps: MOD-004, MOD-005
  7. MOD-007_order_management       [full_stack]     deps: MOD-006, MOD-007
  8. MOD-008_notification_service   [backend_only]   deps: MOD-004

TECH SUMMARY:
  Backend:  Node.js 20.11.0 / Express 4.18.2 / TypeScript 5.3.3
  DB ORM:   Prisma 5.10.2 / PostgreSQL 16
  Frontend: React 18.2.0 / Vite 5.1.4 / TypeScript 5.3.3
  State:    Zustand 4.5.2
  HTTP:     Axios 1.6.7
  Tests:    Vitest 1.3.1 (FE) / Jest 29.7.0 (BE) / Playwright 1.42.1 (E2E)
  Build:    Docker / GitHub Actions

API ENDPOINTS BY MODULE:
  MOD-003_authentication:    POST /api/v1/auth/login, POST /api/v1/auth/logout, POST /api/v1/auth/refresh
  MOD-004_user_management:   GET /api/v1/users, GET /api/v1/users/:id, PUT /api/v1/users/:id, DELETE /api/v1/users/:id
  [continue for all modules]

TABLES BY MODULE:
  MOD-002_foundation:        users, sessions
  MOD-003_authentication:    refresh_tokens
  MOD-005_product_catalog:   products, categories, product_images
  [continue for all modules]

FRONTEND ROUTES BY MODULE:
  MOD-003_authentication:    /login, /logout, /register
  MOD-004_user_management:   /profile, /profile/edit, /admin/users
  [continue for all modules]

STATE SLICES BY MODULE:
  MOD-003_authentication:    authStore (Zustand)
  MOD-004_user_management:   userStore (Zustand)
  [continue for all modules]

ENV VARS: DATABASE_URL, JWT_SECRET, JWT_EXPIRY, REFRESH_TOKEN_EXPIRY,
          VITE_API_BASE_URL, VITE_APP_ENV, SMTP_HOST, SMTP_PORT, REDIS_URL

AUTH: JWT (access token in memory, refresh token in httpOnly cookie)
PROTECTED ROUTES: /profile, /profile/edit, /admin/*, /cart, /orders
```

### How to Use the Index Layer

Once the Index Layer is built:
- Use it to determine module processing order
- Use it to plan sprint assignments (tech_stack is always available without reloading)
- Use it to identify which API endpoints belong to which module (without reloading `api_contracts.json`)
- Use it to check table ownership when planning migration ordering

When you need more detail than the index provides → load Level 2 for that module.

---

## Level 2 — Module Layer

### When to Load

Load Level 2 when you begin planning a specific module. You are building `backend_tasks.json` and `frontend_tasks.json` for that module.

### What to Load

For the module you are currently planning, load the **relevant sections** from the architecture files — not the whole files.

**From `hld.json` — load:**
```
components[n]   (the entry for this specific module — full object)
```

**From `api_contracts.json` — load:**
```
All endpoints where endpoint.module === "<module_id>"
(or filter by the path prefix if module field is absent)
```

**From `database_schema.json` — load:**
```
All tables where table.owned_by === "<module_id>"
Plus: foreign key target tables (even if owned by another module)
```

**From `security_design.json` — load:**
```
module_permissions["<module_name>"]   (if this field exists)
frontend.route_guards for this module's routes
```

**From `frontend_design.json` — load:**
```
routing.routes filtered by module
component_hierarchy["<module_name>"]  (if this field exists)
state_management.slices["<module_name>"]
user_journeys["<module_name>"]        (if this field exists)
```

**From `tech_stack.json` — do NOT reload.** The Index Layer already captured the version summary. Only reload `tech_stack.json` if you need a specific dependency version not captured in the index.

### Module Layer Working Memory Format

```
MODULE LAYER: MOD-003_authentication (full_stack)
Architecture source: hld.json → components[2]

COMPONENT DEFINITION:
  Responsibilities: [list from hld.json]
  Interfaces: [list from hld.json]
  Upstream deps: MOD-002_foundation (provides: db connection, base error handler, base logger)
  Downstream consumers: MOD-004_user_management (needs: validateToken), MOD-006_shopping_cart (needs: isAuthenticated)

BACKEND — API ENDPOINTS (from api_contracts.json):
  [0] POST /api/v1/auth/login
      request:  { email: string, password: string }
      response: { data: { accessToken: string, refreshToken: string } }
      errors:   [INVALID_CREDENTIALS, VALIDATION_ERROR]
      source:   api_contracts.json → endpoints[0]

  [1] POST /api/v1/auth/logout
      request:  (Authorization header: Bearer <token>)
      response: { data: { message: "Logged out successfully" } }
      errors:   [TOKEN_INVALID, TOKEN_EXPIRED]
      source:   api_contracts.json → endpoints[1]

  [2] POST /api/v1/auth/refresh
      request:  { refreshToken: string }
      response: { data: { accessToken: string } }
      errors:   [REFRESH_TOKEN_INVALID, REFRESH_TOKEN_EXPIRED]
      source:   api_contracts.json → endpoints[2]

BACKEND — DATABASE TABLES (from database_schema.json):
  refresh_tokens: id (uuid), user_id (fk → users.id), token_hash (varchar 255),
                  expires_at (timestamp), created_at (timestamp)
  source: database_schema.json → tables.refresh_tokens

BACKEND — SECURITY (from security_design.json):
  JWT payload: { userId, email, role }
  Access token expiry: 24h
  Refresh token expiry: 7d
  Password hashing: bcrypt, 12 salt rounds
  Token storage: access token in memory (frontend store), refresh token in httpOnly cookie
  source: security_design.json → auth

FRONTEND — ROUTES (from frontend_design.json):
  /login    → LoginPage   [public]
  /logout   → (action, no page)
  /register → RegisterPage [public]
  source: frontend_design.json → routing.routes[0..2]

FRONTEND — COMPONENT HIERARCHY (from frontend_design.json):
  LoginPage
    └── AuthLayout
        └── LoginForm
              ├── EmailInput    (reuse from foundation design system)
              ├── PasswordInput (reuse from foundation design system)
              └── SubmitButton  (reuse from foundation design system)
  source: frontend_design.json → component_hierarchy.authentication

FRONTEND — STATE (from frontend_design.json):
  authStore (Zustand):
    state:   { accessToken, currentUser, isLoading, authError }
    actions: { login, logout, refreshToken, setUser, clearError }
    selectors: { isAuthenticated, currentUser, authError }
    persistence: accessToken NOT persisted (memory only per security_design.json)
  source: frontend_design.json → state_management.slices.auth

FRONTEND — USER JOURNEY (from frontend_design.json):
  login journey:
    1. User lands on /login
    2. Enters email + password
    3. On success → redirect to /dashboard
    4. On failure → show inline error message
  source: frontend_design.json → user_journeys.login
```

### When to Unload

After you have written `backend_tasks.json` and `frontend_tasks.json` for this module, release the Module Layer from working memory. Load the next module's Layer 2.

Do not carry two modules' Level 2 data simultaneously. The only exception: when planning an integration task between Module A and Module B, load both modules' Level 2 data simultaneously. This is the only case where two Level 2 loads overlap.

---

## Level 3 — Task Layer

### When to Use

Level 3 is used for precision verification, not broad planning. Use it when:
- Writing `architecture_references` for a specific task (you need the exact JSON path, not an approximation)
- Writing `acceptance_criteria` for a task that references specific field values (you need the actual value, not your remembered summary)
- Writing `rollback_scope` for a task involving database changes (you need the exact column list)
- Resolving an ambiguity between what your Level 2 summary says and what the architecture file actually says

### How to Load

Do not reload the full architecture file. Use targeted extraction.

For a specific task — for example, `TASK-024: Implement AuthService`:

```
Level 3 — TASK-024 (AuthService implementation)

LOAD: security_design.json → auth.jwt.payload_schema
  Result: { "userId": "string (uuid)", "email": "string", "role": "enum: user|admin|moderator" }

LOAD: security_design.json → auth.jwt.access_token_expiry
  Result: "24h"

LOAD: security_design.json → auth.password_hashing
  Result: { "algorithm": "bcrypt", "salt_rounds": 12 }

LOAD: database_schema.json → tables.refresh_tokens
  Result: {
    "columns": [
      { "name": "id",          "type": "uuid",         "constraints": ["primary_key", "default: gen_random_uuid()"] },
      { "name": "user_id",     "type": "uuid",         "constraints": ["not_null", "foreign_key: users.id on_delete: cascade"] },
      { "name": "token_hash",  "type": "varchar(255)", "constraints": ["not_null", "unique"] },
      { "name": "expires_at",  "type": "timestamp",    "constraints": ["not_null"] },
      { "name": "created_at",  "type": "timestamp",    "constraints": ["not_null", "default: now()"] }
    ]
  }

ARCHITECTURE REFERENCES (to embed in task):
  { "file": "security_design.json", "path": "auth.jwt.payload_schema",           "reason": "Defines JWT payload fields" }
  { "file": "security_design.json", "path": "auth.jwt.access_token_expiry",      "reason": "Token expiry duration" }
  { "file": "security_design.json", "path": "auth.password_hashing.algorithm",   "reason": "Hashing algorithm (bcrypt)" }
  { "file": "security_design.json", "path": "auth.password_hashing.salt_rounds", "reason": "Salt rounds (12)" }
  { "file": "database_schema.json", "path": "tables.refresh_tokens",             "reason": "refresh_tokens table schema for token storage" }
```

### The JSON Path Accuracy Rule

**Every JSON path you write in `architecture_references` must be a path you have verified at Level 3 — not a path you remembered or guessed from the Level 2 summary.**

If you write `security_design.json → auth.jwt.payload_schema` in a task's `architecture_references`, you must have loaded that path at Level 3 and confirmed the field exists and contains what you expect.

TicketValidator check A-02 will verify every architecture reference path. A remembered-but-wrong path fails the validation and sends the ticket back to JiraOrchestrator.

---

## Context Load Sequence: Full Planning Run

This is the complete sequence of context loads for a full planning run, so you can verify you are following it correctly.

```
START:
  Load Index Layer (Level 1) — one-time read of all 8 files, extract summaries
  Build working memory Index Layer object
  Write 05_context/ skeleton files if they don't exist

PHASE 1 PLANNING (Bootstrap):
  Load Level 2: hld.json → bootstrap_config (if defined)
  Load Level 2: infra_design.json → environment, cicd, containers
  Load Level 2: tech_stack.json → backend.dependencies (full list, all packages)
  Load Level 2: tech_stack.json → frontend.dependencies (full list, all packages)
  Load Level 2: frontend_design.json → design_system, design_tokens, shared_components
  Plan Phase 1A + 1B tasks
  For each bootstrap task: Load Level 3 for precise architecture references
  Write: 03_planning/module_breakdown/MOD-001_environment_setup/

PHASE 2 PLANNING (Foundation):
  Load Level 2: domain_model.json → full entity list and relationships
  Load Level 2: database_schema.json → all tables with full column definitions
  Load Level 2: frontend_design.json → layouts, typography, form_primitives
  Plan Phase 2A + 2B tasks
  For each foundation task: Load Level 3 as needed
  Write: 03_planning/module_breakdown/MOD-002_foundation/

PHASE 3 PLANNING (Feature Modules — one module at a time):
  FOR EACH MODULE (in Index Layer processing order, skipping MOD-001 and MOD-002):
    Load Level 2 for this module (see Module Layer section above)
    Plan backend_tasks (service, controller, router, middleware, tests)
    Plan frontend_tasks (api_client, store, pages, components, route_guard, tests)
    For each task: Load Level 3 for precise architecture references
    Write: 03_planning/module_breakdown/MOD-XXX_<n>/
    Unload Level 2 for this module
    Load Level 2 for next module

PHASE 4 PLANNING (Integration):
  For each integration task:
    Load Level 2 for Module A (the two modules being integrated)
    Load Level 2 for Module B
    Plan integration task with references to both modules
    Unload both Level 2s
  Write: 03_planning/module_breakdown/integration/integration_tasks.json

PHASE 5 PLANNING (E2E Testing):
  Load Level 2: frontend_design.json → user_journeys (full)
  Load Level 2: non_functional_requirements.json (if referenced in e2e)
  Plan E2E tasks
  Write: 03_planning/module_breakdown/e2e_testing/e2e_tasks.json

PHASE 5.5 (Shared Utilities Extraction):
  Re-read all task files written so far (not architecture files — task files)
  Identify repeated patterns
  Create shared utility tasks
  Update dependency graph
  Re-write affected task files with updated dependencies

PHASE 6 (Sprint Mapping):
  Use Index Layer + all task files to build sprint_map.json
  No architecture file loads needed in this phase
  Write: 03_planning/sprint_map.json

FINALIZATION:
  Write: 03_planning/execution_plan.json
  Write: 03_planning/dependency_graph.json
  Write: 03_planning/planning_review_report.md
  Halt. Await human review.
```

---

## Common Mistakes and How to Avoid Them

### Mistake 1: Writing a JSON path you haven't verified at Level 3

**Wrong:**
```json
{ "file": "security_design.json", "path": "auth.jwtPayload", "reason": "..." }
```
This path was guessed from memory. The actual field is `auth.jwt.payload_schema`. TicketValidator will fail A-02.

**Right:** Load Level 3 for that file/path, confirm the exact field exists, then write the path.

### Mistake 2: Loading the same architecture file multiple times when the Index Layer is enough

**Wrong:** Reloading all of `api_contracts.json` to check which endpoints belong to MOD-005 — the Index Layer already has this grouped.

**Right:** Consult the Index Layer endpoint-by-module grouping. Only reload `api_contracts.json` when you need the full request/response schema for a specific endpoint (Level 3 load).

### Mistake 3: Carrying two full modules in context simultaneously

**Wrong:** Planning MOD-003 and MOD-004 at the same time because they have a dependency relationship.

**Right:** Finish MOD-003 completely. Write its task files. Unload its Level 2. Then load MOD-004's Level 2 — and reference MOD-003's completed task files (from disk) rather than its Level 2 context.

### Mistake 4: Forgetting the dependency direction when loading integration tasks

Integration tasks between Module A and Module B are planned AFTER both individual modules are planned. You load both Level 2s simultaneously for integration tasks only. After writing the integration task, unload both.

### Mistake 5: Using Level 1 Index summaries as the source for acceptance criteria values

**Wrong:** Writing `"JWT expiry must be 24h"` in acceptance criteria because the Index Layer says "24h" — without verifying that `security_design.json → auth.jwt.access_token_expiry` actually says `"24h"` and not `86400` or `"1d"`.

**Right:** Always load Level 3 to verify the actual value before writing it into acceptance criteria. The Index Layer is approximate. Architecture files are authoritative.

---

## Token Budget Guidance

Use these approximate token counts to estimate your working memory usage:

| Layer | Approximate Tokens |
|---|---|
| Index Layer (Level 1 summary) | 500–800 |
| One Module Layer (Level 2) | 2,000–4,000 |
| One Task Layer (Level 3) | 200–600 |
| Two simultaneous Module Layers (integration tasks only) | 4,000–8,000 |
| Full agent prompt + rules | ~3,000–5,000 |

Total working memory at any point: **~6,000–9,000 tokens** (Index + one Module Layer + agent instructions). Well within safe context bounds for a modern LLM, even when accounting for the growing task files being written.
