---
name: context-management
description: >
  Always load this skill at Phase 0. Defines the section-by-section processing
  protocol, checkpoint file format, context budget rules, and resume-from-checkpoint
  procedure. Prevents context window overflow on large BRDs by ensuring the agent
  never holds more than one section of source content in context at a time.
  Required for any BRD over 1,000 words.
---

# Context Management Skill

## The Core Problem

An LLM agent has a fixed context window. Every token counts toward it:
- The agent's own instructions (the .agent.md file)
- Skill files that were loaded
- Source BRD text currently being read
- Requirements extracted so far (held in context)
- Tool call results
- Output JSON being assembled

A 100-page BRD is ~25,000 words (~33,000 tokens). Add skills, instructions, and accumulated output — the context fills and the agent silently starts forgetting earlier content. Requirements from page 1 get dropped. Sections are re-processed. IDs conflict. Quality degrades invisibly.

**The fix**: The agent never holds the full BRD in context. It reads one section, processes it, writes results to disk, then discards the section before reading the next. The disk is the memory. The context holds only: current section text + current section results + a compact state summary.

---

## Rule 1: Never Hold the Unified Buffer

**The agent MUST NOT follow the "unified content buffer" pattern for documents over 1,000 words.**

Replace the unified buffer approach with **section-by-section streaming**:

```
For each document:
  For each section in document:
    1. Read section into context
    2. Process section (extract requirements, detect compliance signals, etc.)
    3. Append results to output files on disk immediately
    4. Update checkpoint file
    5. Discard section from context (do not accumulate)
    6. Read next section
```

The only state carried between sections is the **compact context summary** (defined below) — not the section text itself.

---

## Rule 2: The 800-Token Section Budget

**Maximum section size to load at once: 800 tokens (~600 words)**

How to identify sections:
1. Use document headings (H1, H2, H3) as natural section boundaries
2. If a section exceeds 800 tokens: split at paragraph boundaries within the section
3. If no headings exist: split at every 600 words, using paragraph breaks as split points
4. Never split a sentence or a table row mid-way

Before reading any section, estimate its size:
- Count the heading depth and approximate the section length from the document outline
- If unsure: read the first paragraph only to assess density, then decide split strategy

---

## Rule 3: The Compact Context Summary

Between sections, the agent carries only this compact summary (target: under 200 tokens):

```json
{
  "run_id": "uuid",
  "current_phase": 3,
  "current_document": "brd-main.md",
  "current_section_index": 7,
  "total_sections_estimated": 24,
  "sections_completed": ["Introduction", "Background", "Stakeholders", "Scope", "FR Section 1", "FR Section 2", "NFR Section 1"],
  "req_counter": {"functional": 23, "non_functional": 8, "compliance": 0},
  "running_signals": {
    "greenfield_detected": true,
    "compliance_signals": ["GDPR", "COPPA"],
    "document_type": "BRD"
  },
  "checkpoint_file": "analysis/.context-checkpoint.json"
}
```

This summary is reconstructed from the checkpoint file at the start of each section. It tells the agent everything it needs to continue without re-reading prior sections.

---

## Rule 4: The Checkpoint File

Write `/analysis/.context-checkpoint.json` after every section is processed. This is the agent's persistent memory across the run.

**Full checkpoint schema**:

```json
{
  "checkpoint_version": "1.0",
  "run_id": "uuid",
  "agent_version": "BrdAnalyzer-v3.0",
  "started_at": "ISO-8601",
  "last_updated": "ISO-8601",
  "status": "in_progress | completed | failed | interrupted",

  "documents": {
    "brd-main.md": {
      "total_sections": 24,
      "sections_completed": 7,
      "sections": [
        {
          "index": 0,
          "heading": "Introduction",
          "status": "completed",
          "word_count": 312,
          "requirements_extracted": 0,
          "compliance_signals_found": [],
          "greenfield_signals_found": [],
          "completed_at": "ISO-8601"
        }
      ]
    }
  },

  "phase_completion": {
    "phase_0": "completed",
    "phase_1": "in_progress",
    "phase_2": "pending",
    "phase_3": "pending",
    "phase_4": "pending",
    "phase_5": "pending",
    "phase_6": "pending",
    "phase_7": "pending",
    "phase_8": "pending",
    "phase_9": "pending",
    "phase_10": "pending",
    "phase_11": "pending",
    "phase_12": "pending",
    "phase_13": "pending"
  },

  "accumulated_state": {
    "req_counters": {
      "functional": 23,
      "non_functional": 8,
      "compliance": 0,
      "assumptions": 4,
      "risks": 2,
      "questions": 11
    },
    "greenfield_detected": true,
    "compliance_signals_detected": ["GDPR", "COPPA"],
    "document_type_confirmed": "BRD",
    "stakeholders_found": ["CPO", "CTO", "Head of Education"],
    "sections_with_requirements": ["FR Section 1", "FR Section 2"],
    "sections_requiring_revisit": [],
    "p0_questions_generated": 1
  },

  "output_files": {
    "intake-manifest.json": "written",
    "requirements_catalog.json": "in_progress",
    "clarification_questions.json": "pending",
    "assumptions_and_risks.json": "in_progress",
    "traceability_matrix.json": "pending",
    "glossary.json": "pending",
    "analysis_summary.json": "pending"
  }
}
```

**Write frequency**: After every section. Use `vscode/str_replace` to update `last_updated`, the section's status, and the counters. Never rewrite the entire checkpoint file — only patch the changed fields.

---

## Rule 5: What to Hold vs. What to Write

**Hold in context (working memory):**
- Current section text — ONE section only
- Compact context summary (200 tokens)
- The requirement being currently constructed (until it's written to disk)
- The skill instructions relevant to the current phase

**Write to disk immediately:**
- Every completed requirement → appended to `requirements_catalog.json`
- Every question generated → appended to `clarification_questions.json`
- Every assumption/risk → appended to `assumptions_and_risks.json`
- Every checkpoint update → patched in `.context-checkpoint.json`

**Never hold in context:**
- Previously completed sections of the BRD
- Requirements already written to disk (refer to them by ID only)
- Skill files that are no longer needed for the current phase
- Accumulated full requirement objects after they are written

---

## Rule 6: Conflict Detection Without Full Context

The conflict detection phase (Phase 5) normally requires comparing requirements against each other. With section-by-section processing, previously extracted requirements are on disk, not in context.

**Protocol for conflict detection without full context:**

1. Load requirements by category (not all at once):
   - Read `requirements_catalog.json` and extract only the `id`, `category`, `title`, and first 20 words of `description` for each requirement
   - This "requirements index" is ~10 tokens per requirement — 100 requirements = ~1,000 tokens, manageable
2. For each new requirement being processed, compare against the category index only
3. For flagged potential conflicts: load the full text of only the two conflicting requirements to confirm
4. Write conflict record immediately to `assumptions_and_risks.json`
5. Discard loaded requirement text after comparison

**Requirements index format** (kept in context during Phase 5):
```json
[
  {"id": "REQ-F-001", "category": "authentication", "title": "User login", "summary": "System shall authenticate users via username and password"},
  {"id": "REQ-NF-003", "category": "performance", "title": "Login response time", "summary": "Login shall complete within 2 seconds under normal load"}
]
```

---

## Rule 7: Resume From Checkpoint

If the agent run is interrupted (user cancels, timeout, error), it must be able to resume without reprocessing completed sections.

**On Phase 0 startup**, after checking for existing `/analysis/` output:

1. Check if `/analysis/.context-checkpoint.json` exists
2. If yes and `status = "in_progress"`: ask user — "Interrupted run found (started {timestamp}, {N} of {M} sections processed). [R]esume from checkpoint or [S]tart fresh?"
3. If Resume: load checkpoint, reconstruct compact context summary, skip completed sections/phases, continue from where interrupted
4. If Start fresh: archive existing checkpoint and outputs, begin from Phase 0

**Resume procedure:**
1. Read checkpoint file
2. Reconstruct compact context summary from `accumulated_state`
3. For the current document: find the first section with `status != "completed"`
4. Resume processing from that section
5. All phases marked `completed` in `phase_completion` are skipped entirely
6. Phases marked `in_progress` are restarted from the first incomplete section

---

## Rule 8: Section Outline First

Before processing any document, build its section outline:

1. Read the first 100 lines of the document to extract headings (lines starting with `#`, `##`, `###`, or ALL-CAPS lines in PDFs)
2. Write the outline to the checkpoint as the sections array (with `status: "pending"` for all)
3. Estimate total sections and word count per section
4. Determine if any section needs splitting (> 800 tokens)

This outline read is cheap (100 lines) and gives the agent a complete map before processing begins. It prevents the agent from accidentally loading an oversized section.

**Section outline example:**
```json
[
  {"index": 0, "heading": "1. Introduction", "estimated_words": 280, "needs_split": false},
  {"index": 1, "heading": "2. Business Background", "estimated_words": 950, "needs_split": true, "split_into": 2},
  {"index": 2, "heading": "3. Stakeholders", "estimated_words": 420, "needs_split": false},
  {"index": 3, "heading": "4. Functional Requirements", "estimated_words": 4200, "needs_split": true, "split_into": 7}
]
```

---

## Rule 9: Phase Transition Cleanup

When a phase completes and the next phase begins:

1. Mark phase as `completed` in checkpoint
2. **Drop all source text from context** — it has been processed and written to disk
3. The next phase reads from output files, not from BRD source text
4. Phase 5 (conflict detection) reads from `requirements_catalog.json` index
5. Phase 7 (gap analysis) reads section outlines + output file metadata, not BRD text
6. Phase 11 (traceability) reads from all output files — loads metadata only, not full requirement bodies

This means Phases 5, 7, 8, 9, 10, 11 operate on **output files**, not on BRD source content. The BRD is fully consumed and written to disk by the end of Phase 3.

---

## Rule 10: Context Budget Monitoring

Before loading any new content, estimate whether it fits:

**Rough token budget (Sonnet-class model, 200K context):**
- Agent instructions: ~8,000 tokens (reserved, always present)
- Active skill files: ~2,000 tokens each (load at most 2 simultaneously)
- Current section text: max 800 tokens (Rule 2)
- Compact context summary: 200 tokens
- Current requirement being built: 200 tokens
- Requirements index (Phase 5): ~1,000 tokens for 100 requirements
- Tool call overhead: ~500 tokens per call
- **Safe working budget: ~185,000 tokens remaining**

**Warning threshold**: If the agent estimates it would need more than 20,000 tokens for a single operation, it must split that operation further.

**Hard stop**: Never load a full output file into context to read it. Always use targeted reads — load only the fields needed for the current operation.

---

## Skill Load / Unload Protocol

Skills are large (~2,000–4,000 tokens each). Load exactly when needed, then release.

| Phase | Skills to hold in context |
|---|---|
| Phase 0 | context-management only |
| Phase 1 | context-management + multi-format-intake (if needed) |
| Phase 1 (images) | context-management + artifact-intake |
| Phase 3–4 | context-management only (rules are internalized) |
| Phase 5 | context-management only |
| Phase 6 | context-management + req-dedup-and-hierarchy |
| Phase 8 | context-management + greenfield-intelligence |
| Phase 9 | context-management + compliance-radar |
| Phase 12 | context-management + output-factory |

After a skill is no longer needed, do not reference it. It naturally drops out of the rolling context window as new content is loaded.

Never load more than 2 skill files simultaneously (except during Phase 0 setup).
