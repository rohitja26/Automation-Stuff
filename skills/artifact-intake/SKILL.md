---
name: artifact-intake
description: >
  Load this skill automatically when image files, Figma exports, UI design files,
  or wireframes are found in /brd/: .png, .jpg, .gif, .webp, .fig, .xd, .sketch,
  or any file with "mockup", "wireframe", "design", "figma", "ui" in the name.
  Provides vision-based UI analysis, implicit requirement extraction from screen
  flows, and classification of UI artifacts that cannot be read as text.
---

# Artifact Intake Skill

## Purpose
Process visual and design artifacts found in `/brd/`. Extract implicit requirements from UI designs — requirements that are shown visually but never stated in text. Generate P0 questions for artifacts that cannot be processed at all.

---

## Step 1: Classify Every File

Scan using `**/*` pattern (not `*.md`). For each file:

**Extension-based first pass**:
| Extension | Class | Processing |
|---|---|---|
| `.png`, `.jpg`, `.jpeg`, `.gif`, `.webp` | UI-DESIGN or TEXT-IMAGE | Vision analysis |
| `.fig`, `.sketch`, `.xd` | UI-DESIGN (binary) | Cannot read — P0 question |
| `.pdf` | May be UI spec | Attempt text extraction via multi-format skill |
| `.md` | Score for type | Content-based classification |
| `.html` | Web prototype | Fetch and analyze via multi-format skill |

**Content-based classification for `.md` files** — score against signals (threshold: ≥3 signals = confident classification):

BRD signals: "business requirement", "business objective", "stakeholder", "success criteria", "in scope", "out of scope", "business rule"
FRD signals: "functional specification", "feature description", "use case", "system shall", "functional requirement", "user story"
TRD signals: "technical specification", "architecture", "implementation", "database schema", "API endpoint", "technology stack", "infrastructure"
UI-MANIFEST signals: "screen", "flow", "wireframe", "mockup", "component", "navigation", "layout", "prototype"

---

## Step 2: Vision Analysis for Image Files

For each image file, perform vision analysis:

### Text extraction
- Read all visible text in the image (labels, button text, field labels, headings, error messages)
- Extract into a text corpus tagged with position (top-left, center, etc.)

### UI element detection
- Identify: buttons, form fields, dropdowns, checkboxes, radio buttons, tables, modals, navigation menus, tabs, cards, headers, footers
- List detected elements with their visible labels

### Flow analysis
- If multiple screens shown: detect numbered sequence or arrows indicating flow direction
- Reconstruct user journey: Screen A → action → Screen B

---

## Step 3: Implicit Requirement Extraction

From each analyzed image, derive implicit requirements that are shown but not stated. Apply these inference rules:

| Visual element | Implicit requirement generated |
|---|---|
| Login form with username + password | REQ-F: System shall authenticate users with username and password credentials |
| "Forgot password" link | REQ-F: System shall provide a password reset mechanism via registered email |
| Loading spinner | REQ-NF: System shall display a loading indicator for operations taking >500ms |
| Error message field | REQ-F: System shall display descriptive error messages for validation failures |
| Pagination controls | REQ-F: System shall paginate list views; REQ-NF: default page size should be defined |
| Search bar | REQ-F: System shall provide search capability for [detected list context] |
| Filter/sort controls | REQ-F: System shall allow filtering and sorting of [detected list context] |
| File upload button | REQ-F: System shall support file upload; REQ-NF: accepted file types and size limits TBD — P1 question |
| Navigation menu with N items | REQ-F: System shall provide navigation to [N menu items detected] |
| User avatar/profile icon | REQ-F: System shall support user profile management |
| Notification bell | REQ-F: System shall provide in-app notification system |
| Export/download button | REQ-F: System shall support data export functionality |
| Screen-to-screen transition | REQ-NF: latency requirement P1 question generated |
| Modal/dialog | REQ-F: System shall display confirmation dialogs for [detected action]; business interruption handling required |
| Dashboard with charts | REQ-F: System shall provide analytics dashboard; chart types [detected types] |
| Multi-step form (step indicators) | REQ-F: System shall support multi-step wizard for [detected context]; REQ-F: shall preserve form state between steps |
| Role-specific menu items (Admin vs User) | REQ-F: System shall implement role-based access control with at minimum Admin and [User] roles |

**Mark all extracted requirements** with:
- `source_verbatim`: "Implied by UI element: [element] in [filename]"
- `source_location.file`: image filename
- `source_location.section`: "visual-inference"
- `estimated_complexity`: start at "medium" for all visual-inferred requirements

---

## Step 4: P0 Questions for Non-Processable Artifacts

For binary design files (.fig, .sketch, .xd) that cannot be read:

Generate P0 question:
"Design file `{filename}` found in /brd/ but cannot be read (binary format). Please provide one of: (a) exported PNG/JPG screenshots of key screens, (b) a UI-MANIFEST.md describing the screens and flows, (c) paste the relevant acceptance criteria into the BRD directly."

For images with low-confidence text extraction:
Generate P1 question:
"Image `{filename}` processed with low confidence. Extracted text: '{extracted_text_truncated}'. Please verify this is accurate or provide the text content directly."

---

## Step 5: UI-MANIFEST.md Processing

If a `.md` file is classified as UI-MANIFEST (score ≥3 UI signals), process as follows:

1. Extract every screen description as a UI artifact entry
2. For each screen: identify all stated interactions, navigation targets, and data shown
3. Apply implicit requirement extraction (Step 3) to the textual descriptions
4. Cross-reference: if a screen description mentions a field name also mentioned in the BRD, link the UI requirement to the BRD requirement

---

## Output: ui_artifacts section in intake-manifest.json

For each processed visual artifact, add to the `artifacts` array in `intake-manifest.json`:

```json
{
  "filename": "login_screen.png",
  "classification": "UI-DESIGN",
  "processing_status": "processed",
  "extraction_method": "vision-analysis",
  "ui_elements_detected": 12,
  "implicit_requirements_generated": 5,
  "confidence": "high",
  "screens_detected": 1,
  "flows_detected": 0,
  "p0_question_generated": false
}
```

And append generated requirements to `requirements_catalog.json` with all fields populated per Output Factory skill schema.
