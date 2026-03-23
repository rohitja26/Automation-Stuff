---
name: sprint-planning
description: Sprint grouping, capacity planning, and critical path scheduling for PlanningDesign-3.3. Takes a complete list of planned tasks (backend + frontend) with their dependency graph and organizes them into self-contained, dependency-safe sprints. Ensures each sprint delivers a working vertical slice of the system. Enforces capacity limits for LLM execution. Identifies the critical path and flags it for human review.
version: 1.0
used-by: PlanningDesign-3.3
---

# Sprint Planning Skill

## When to Use This Skill

This skill is invoked at the end of Phase 6 (Sprint Mapping) in PlanningDesign-3.3, after all task files have been written for all modules, integration tasks, and E2E tests.

At that point, you have:
- A complete task list (backend + frontend) with `task_id`, `complexity_points`, `dependencies`, and `dependents` for every task
- A dependency graph in `03_planning/dependency_graph.json`
- Module types and their inter-module dependencies from the Index Layer

Your job in this skill: assign every task to a sprint number, respecting all constraints below, then produce `03_planning/sprint_map.json`.

---

## Sprint Structure Rules (Non-Negotiable)

### Rule S1 — Sprint 1 is Bootstrap Only

Sprint 1 contains ONLY Phase 1A (backend bootstrap) and Phase 1B (frontend bootstrap) tasks. No feature module tasks, no foundation tasks, no integration tasks.

If there are more than 10 bootstrap tasks, split them across Sprint 1 and Sprint 1B (relabel as Sprint 1 Part A and Sprint 1 Part B). Never push bootstrap tasks to Sprint 2.

### Rule S2 — Sprint 2 is Foundation Only

Sprint 2 contains ONLY Phase 2A (backend domain foundation: migrations, models, repositories) and Phase 2B (frontend foundation: base UI components, layout components, form primitives). No feature module tasks.

Exception: If the project has very few foundation tasks (fewer than 4), combine Sprint 2 with the first feature module's tasks — but only if the feature module's tasks all depend on foundation tasks that are included in the same sprint and appear earlier in the sprint sequence.

### Rule S3 — Dependency Ordering Within Every Sprint

Within a sprint, tasks must be ordered so that every task's dependencies appear before it in the sprint sequence. If task A depends on task B, B must appear before A — either in an earlier sprint or earlier in the same sprint's sequence.

**Hard constraint:** A task can never be in Sprint N if any of its dependencies are in Sprint N or later.

To verify: for every task T in Sprint N, check every dependency D in T's `dependencies` list. D's sprint assignment must be ≤ N-1, OR D must be earlier in Sprint N's task list and be a within-sprint dependency (see Within-Sprint Dependencies below).

### Rule S4 — Within-Sprint Dependencies

Some tasks within the same sprint have a dependency between them. This is allowed, with conditions:

**Allowed within-sprint dependency:** The dependent task can only start AFTER the dependency task is complete. This creates a sequence within the sprint, not a parallel execution.

**Example:** In Sprint 3, for MOD-003_authentication:
```
Within Sprint 3 sequence:
  TASK-024 (AuthService) → must complete before →
  TASK-025 (AuthController) → must complete before →
  TASK-026 (AuthRouter) → must complete before →
  TASK-038 (Auth API client) → must complete before →
  TASK-039 (Auth store) → must complete before →
  TASK-040 (LoginPage) → must complete before →
  TASK-041 (ProtectedRoute)
```

When within-sprint dependencies exist, the sprint's `sequence` field in `sprint_map.json` must list tasks in the correct execution order.

**Not allowed:** A task cannot be in Sprint N if its dependency is ALSO in Sprint N but would need to be completed FIRST and that dependency itself has dependencies that are NOT yet complete. In other words — no circular within-sprint chains, and no within-sprint dependency that creates a second-order unresolved dependency.

### Rule S5 — Vertical Slice Principle

Each sprint from Sprint 3 onward must deliver a complete, usable, deployable feature — not half a feature. Specifically:

**Acceptable sprint:** "Sprint 3 delivers working user authentication end-to-end: the backend login endpoint is implemented and the frontend login page calls it. An authenticated user can log in, see the dashboard, and their JWT is managed correctly."

**Not acceptable:** "Sprint 3 delivers all backend services for authentication, user management, and product catalog. No frontend."

To enforce this: every sprint from Sprint 3 onward should contain both backend AND frontend tasks for the same module(s), ending with a state where at least one user journey is fully functional.

**Exception:** If a module is `backend_only`, its sprint can be backend-only. That is expected and acceptable.

### Rule S6 — Capacity Limit

Maximum 10 tasks per sprint. Minimum 4 tasks per sprint (unless it's the final sprint, which may have fewer).

If a module's combined backend + frontend tasks exceed 10, split the module across two sprints. Split rule:
1. Backend service + controller + tests go in Sprint N
2. Frontend api_client + store + page + tests go in Sprint N+1
3. The frontend tasks in Sprint N+1 declare dependencies on the backend tasks from Sprint N

If splitting by layer (backend first, frontend next), note it clearly in the sprint goal — this is an exception to the vertical slice principle and must be justified in `planning_review_report.md`.

### Rule S7 — Integration Sprints

Integration tasks are placed in the sprint AFTER both modules they integrate are fully implemented.

Example: `INTEGRATION-FE-MOD003-API-001` (connects frontend auth to backend auth) goes in Sprint 4 if MOD-003's backend and frontend tasks are both in Sprint 3.

If an integration task involves modules from different sprints, it goes in the sprint AFTER the LATER of the two modules' sprints.

### Rule S8 — E2E Testing is Last

All E2E test tasks go into the final sprint(s) of the project, after all integration tasks. E2E tests depend on the entire system being functional.

If there are more than 6 E2E tasks, spread them across the final 2 sprints, prioritizing the most critical user journeys in the second-to-last sprint.

---

## Critical Path Identification

The critical path is the longest chain of dependent tasks in the project. It determines the minimum calendar time for delivery.

### How to Find the Critical Path

1. Build the full DAG from `dependency_graph.json`.
2. Assign each task a weight equal to its `complexity_points`.
3. Find the path from start (no dependencies) to end (no dependents) with the highest total weight.
4. That path is the critical path.

### Output the Critical Path

In `planning_review_report.md`, list the critical path explicitly:

```markdown
## Critical Path

Total weight: 47 complexity points
Estimated duration: ~X sprints minimum (cannot be parallelized)

TASK-001 (2pts) → TASK-006 (3pts) → TASK-009 (5pts) → TASK-024 (5pts)
→ TASK-025 (3pts) → TASK-038 (2pts) → TASK-039 (3pts) → TASK-040 (3pts)
→ TASK-041 (2pts) → INTEGRATION-FE-MOD003-API-001 (3pts)
→ E2E-001 (5pts) → E2E-002 (5pts)

Key bottlenecks on critical path:
- TASK-009 (DB migration for users table, 5pts): blocks AuthService + UserService
- TASK-024 (AuthService, 5pts): blocks entire frontend auth flow
- INTEGRATION-FE-MOD003-API-001 (3pts): blocks E2E tests
```

### What Frontend Tasks on the Critical Path Mean

If a frontend task is on the critical path, it means a delayed frontend implementation blocks the E2E tests (and thus the entire project). Flag this in the review report:

> "Frontend task TASK-040 (LoginPage) is on the critical path. Any delay in frontend implementation directly delays E2E testing. Consider assigning this task with elevated priority."

---

## Sprint Map Output Format

Write to `03_planning/sprint_map.json`:

```json
{
  "sprint_map_id": "SPM-<YYYYMMDD>-<sequence>",
  "generated_at": "<ISO8601 timestamp>",
  "plan_id": "<from execution_plan.json>",
  "total_sprints": 6,
  "total_tasks": 48,
  "total_backend_tasks": 26,
  "total_frontend_tasks": 22,
  "critical_path_length_points": 47,
  "critical_path_task_count": 12,

  "sprints": [
    {
      "sprint_number": 1,
      "name": "Bootstrap",
      "goal": "Project is initialized, dependencies installed, dev environment runs, base test config working. Backend and frontend can run independently.",
      "milestone": "Running `npm run dev` starts both backend and frontend without errors. `npm test` runs and reports 0 tests (scaffolding only).",
      "task_count": 8,
      "backend_task_count": 5,
      "frontend_task_count": 3,
      "total_complexity_points": 14,
      "sequence": [
        "TASK-001", "TASK-002", "TASK-003", "TASK-004",
        "TASK-005", "TASK-006", "TASK-007", "TASK-008"
      ],
      "tasks": [
        {
          "task_id": "TASK-001",
          "layer": "backend",
          "title": "Initialize project repository structure",
          "sprint_position": 1,
          "complexity_points": 2,
          "within_sprint_dependencies": [],
          "parallel_with": ["TASK-007", "TASK-008"]
        },
        {
          "task_id": "TASK-007",
          "layer": "frontend",
          "title": "Frontend project initialization",
          "sprint_position": 1,
          "complexity_points": 2,
          "within_sprint_dependencies": [],
          "parallel_with": ["TASK-001"]
        }
      ],
      "on_critical_path": false,
      "split_from_module": null,
      "notes": null
    },

    {
      "sprint_number": 2,
      "name": "Foundation",
      "goal": "Database migrations complete, ORM models defined, base UI components built. Data layer and design system ready for feature modules.",
      "milestone": "All database migrations run successfully. `npm run migrate` completes. All base UI components render correctly in Storybook or equivalent.",
      "task_count": 9,
      "backend_task_count": 5,
      "frontend_task_count": 4,
      "total_complexity_points": 18,
      "sequence": [
        "TASK-009", "TASK-010", "TASK-011", "TASK-012", "TASK-013",
        "TASK-014", "TASK-015", "TASK-016", "TASK-017"
      ],
      "tasks": [],
      "on_critical_path": true,
      "split_from_module": null,
      "notes": "TASK-009 (users table migration) is on the critical path. Must complete before Sprint 3."
    },

    {
      "sprint_number": 3,
      "name": "Authentication — Full Stack",
      "goal": "User can log in via the frontend login page. JWT is issued by backend, stored in memory by frontend auth store. Protected routes redirect unauthenticated users to /login.",
      "milestone": "Manual test: open browser → navigate to /login → enter valid credentials → backend issues JWT → frontend stores token → redirect to /dashboard succeeds. Navigate to /dashboard without login → redirected to /login.",
      "task_count": 10,
      "backend_task_count": 5,
      "frontend_task_count": 5,
      "total_complexity_points": 26,
      "sequence": [
        "TASK-024", "TASK-025", "TASK-026", "TASK-027", "TASK-028",
        "TASK-038", "TASK-039", "TASK-040", "TASK-041", "TASK-042"
      ],
      "tasks": [],
      "on_critical_path": true,
      "split_from_module": null,
      "notes": "Vertical slice sprint. Backend auth API + frontend auth flow delivered together. ProtectedRoute (TASK-041) depends on auth store (TASK-039)."
    },

    {
      "sprint_number": 4,
      "name": "Authentication Integration + User Management",
      "goal": "Auth API client wired to live backend verified. User management module backend complete.",
      "milestone": "Integration test INTEGRATION-FE-MOD003-API-001 passes. GET /api/v1/users returns correct data with valid JWT.",
      "task_count": 7,
      "backend_task_count": 5,
      "frontend_task_count": 2,
      "total_complexity_points": 20,
      "sequence": [],
      "tasks": [],
      "on_critical_path": true,
      "split_from_module": "MOD-004_user_management",
      "notes": "MOD-004 split across Sprint 4 (backend) and Sprint 5 (frontend) due to capacity. Vertical slice principle exception — documented."
    }
  ]
}
```

---

## Sprint Splitting Decision Tree

When a module's tasks exceed the 10-task sprint cap, use this decision tree:

```
Total module tasks > 10?
│
├── YES → Can you split by backend (Sprint N) + frontend (Sprint N+1)?
│         │
│         ├── YES → Split. Mark split_from_module in both sprints.
│         │         Note vertical slice exception in planning_review_report.md.
│         │         Ensure frontend tasks in Sprint N+1 depend on backend tasks in Sprint N.
│         │
│         └── NO (module has no frontend, or frontend tasks are all tiny)
│               → Split by feature sub-group:
│                 Sprint N: core CRUD (service + controller + basic endpoints)
│                 Sprint N+1: advanced features (filters, pagination, bulk ops)
│
└── NO → Assign all to one sprint. No split needed.
```

---

## How to Handle Modules with Cross-Sprint Dependencies

Sometimes Module B depends on Module A, and Module A's sprint has not been decided yet. Use this resolution order:

1. Assign Sprint 1 (bootstrap) — always fixed
2. Assign Sprint 2 (foundation) — always fixed
3. For feature modules: process in topological sort order (Index Layer gives you this). A module with no unresolved upstream dependencies gets the next available sprint.
4. If two modules have no dependency between them and both fit in the same sprint (combined task count ≤ 10): combine them if their domains are related and their combined milestone makes sense as a deliverable. Otherwise keep them separate.

---

## Sprint Naming Convention

Sprint names should be concise and deliverable-focused. Not "Sprint 3" — use "Sprint 3 — Authentication Full Stack".

Format: `Sprint <N> — <Deliverable Description>`

Good examples:
- `Sprint 1 — Bootstrap`
- `Sprint 2 — Foundation`
- `Sprint 3 — Authentication Full Stack`
- `Sprint 4 — User Management + Auth Integration`
- `Sprint 5 — Product Catalog`
- `Sprint 6 — Shopping Cart + Order Management`
- `Sprint 7 — Integration & E2E Tests`

Avoid:
- `Sprint 3 — Backend Work` (not deliverable-focused)
- `Sprint 4 — Lots of Tasks` (meaningless)
- `Sprint 5 — Frontend Pages` (not specific)

---

## Validation Before Writing sprint_map.json

Before writing the file, run these checks internally:

```
[ ] Every task ID from all task files appears in exactly one sprint
[ ] No task appears in more than one sprint
[ ] Every task's dependencies are all assigned to an earlier sprint, OR
    are within the same sprint and appear earlier in the sequence array
[ ] Sprint 1 contains ONLY bootstrap tasks (layer: backend OR frontend, phase: bootstrap)
[ ] Sprint 2 contains ONLY foundation tasks (phase: domain_foundation OR frontend_foundation)
[ ] No sprint has more than 10 tasks
[ ] No sprint has fewer than 4 tasks (unless it is the final sprint)
[ ] Critical path tasks are identified and noted in their sprint's notes field
[ ] All E2E tasks are in the final sprint(s)
[ ] All integration tasks come after both their involved modules are complete
[ ] Total tasks in sprint_map.json equals total tasks in execution_plan.json
```

If any check fails, correct the sprint assignments before writing. Do not write a sprint_map.json that fails its own validation.
