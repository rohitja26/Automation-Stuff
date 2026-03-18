---
name: multi-format-intake
description: >
  Load this skill automatically when any non-markdown file is detected in /brd/:
  PDF, Word (.docx/.doc), Confluence URL, web page URL, Excel/CSV, or plain-text
  URL files. Provides extraction protocols for each format, content normalization
  rules, and source-tagging requirements. Never load manually — the agent detects
  format and loads this skill without user instruction.
---

# Multi-Format Intake Skill

## Purpose
Define extraction protocols for every non-markdown input format. The agent follows whichever protocol matches the detected file type. All extracted content must be source-tagged and written to disk immediately after extraction — never accumulated in a buffer.

---

## Format 1: PDF Files

### Tool approach
Use `github/fetch_url` if the PDF is accessible via URL. If the file is local in `/brd/`, note it as a file-path PDF.

### Extraction protocol
1. If the PDF is URL-accessible: fetch with `github/fetch_url` → extract text content from response
2. If the PDF is a local file path: generate a P0 question — "PDF file `{filename}` found in /brd/ but cannot be read as raw bytes. Please either: (a) paste the text content into a .md file in /brd/, or (b) provide a URL to the PDF, or (c) confirm your Copilot environment has a PDF-parsing MCP connected."
3. After extraction: strip headers/footers (page numbers, running titles), preserve table structure as markdown tables, preserve numbered/bulleted lists
4. Tag every paragraph: `{source: "filename.pdf", page: N, section: "detected heading"}`

### Table extraction
Detect tables by looking for consistent column spacing or explicit table markup in extracted text. Convert to markdown table format. If a table contains requirements, extract each row as a candidate requirement statement.

### Multi-page assembly
Process page by page. If a sentence is split across a page break (detected by incomplete sentence at end of page), join with the start of next page before tagging.

---

## Format 2: Word Documents (.docx, .doc)

### Tool approach
Word documents cannot be read as raw text by the agent. Apply this decision tree:

1. Check if a Word-reading MCP tool is connected (look for tools with "docx", "word", or "document" in their name)
2. If MCP available: use it to extract text, preserving heading hierarchy and table structure
3. If no MCP available: generate a P0 question — "Word document `{filename}` found in /brd/. Please either: (a) save as .md or .txt and place in /brd/, (b) paste content directly into chat, or (c) connect a document-reading MCP to your Copilot environment."

### After extraction (if MCP available)
1. Preserve heading hierarchy (H1, H2, H3 → map to section depth)
2. Extract tracked changes and comments as a separate `document_comments` array — these often contain stakeholder clarifications    
3. Extract tables as markdown tables
4. Tag every paragraph with source file, heading path, and paragraph index
5. Treat document styles: "Requirement", "User Story", "Acceptance Criteria" as explicit requirement signals

---

## Format 3: Confluence Pages

### Tool approach
1. Extract the URL from the `.txt` file or from wherever it appears in the BRD
2. Use `github/fetch_url` to fetch the Confluence page URL
3. If authentication error (401/403): generate P0 question — "Confluence page `{URL}` requires authentication. Please either: (a) export the page as PDF/Word and place in /brd/, or (b) configure Confluence MCP with credentials."

### After fetch
1. Parse HTML: extract `<h1>`, `<h2>`, `<h3>` as section headers
2. Expand Confluence macros represented in HTML: status macros (TODO, DONE, IN PROGRESS) → append status to adjacent text
3. Extract `<table>` elements as markdown tables
4. Remove navigation chrome (sidebar, breadcrumbs, page metadata)
5. If page has child pages: generate P1 question — "Confluence page has {N} child pages. Should I fetch those as well?"
6. Tag all content with `{source: "confluence:{page-title}", url: "url", section: "heading"}`

---

## Format 4: Web Pages

### Tool approach
Use `github/fetch_url` to fetch the URL. If the URL is in a `.txt` file in `/brd/`, read the file first to get the URL.

### After fetch
1. Strip navigation, headers, footers, ads (look for `<nav>`, `<header>`, `<footer>`, `<aside>` and exclude)
2. Extract main content from `<main>`, `<article>`, or the largest `<div>` by text content
3. Preserve headings, lists, and tables
4. If page links to sub-pages containing requirements, generate P1 question listing the linked pages and asking whether to fetch them
5. Tag all content with `{source: "web:{domain}", url: "url", section: "detected heading"}`

---

## Format 5: Excel / CSV Files

### Tool approach
1. CSV files: read directly with `vscode/readFile`
2. XLSX files: check for spreadsheet MCP. If none, generate P0 question same as Word.

### After extraction
1. Detect if the spreadsheet is a requirements list: look for columns named "ID", "Requirement", "Priority", "Status", "Acceptance Criteria"
2. If yes: each row is a candidate requirement — extract with column values mapped to requirement schema fields
3. If no: treat as reference data — extract as a data table artifact and generate P1 question asking how it relates to requirements

---

## Format 6: Images with Embedded Text (non-UI)

For images that contain text (screenshots of requirements documents, photos of whiteboard sessions):
1. Use vision capabilities to read the image if the file is passed as an image attachment
2. Extract all readable text
3. Note confidence level: if text is partially illegible, flag with P1 question
4. Tag: `{source: "filename.png", type: "text-from-image", confidence: "high|medium|low"}`

---

## Universal Source Tagging Rule

Every piece of extracted content must carry this tag before being written to disk:
```json
{
  "text": "extracted content",
  "source_file": "filename.ext",
  "source_type": "pdf | docx | confluence | web | csv | image-text",
  "source_section": "Section heading at extraction point",
  "source_page_or_url": "page number or URL",
  "extraction_confidence": "high | medium | low"
}
```

Low-confidence extractions (garbled OCR, partially visible text) generate a P1 question asking the user to verify the extracted content.

---

## Content Normalization Rules (apply to all formats)

After extraction, normalize before writing to disk:
1. Remove duplicate whitespace and normalize line breaks
2. Standardize heading levels (if PDF uses all-caps for headings, detect and convert)
3. Expand common abbreviations: "NFR" → "Non-Functional Requirement", "FR" → "Functional Requirement", "AC" → "Acceptance Criteria"
4. Detect and tag requirement ID patterns: "FR-001", "NFR-2.1", "UC-03" — these are explicit requirement IDs from the source document; preserve them in `source_verbatim`
5. Do NOT alter the source text — only add tags and normalize structure

---

## Quality Checklist (complete before Phase 2 begins)

- [ ] Every file in artifact manifest has been processed or has a P0 question generated
- [ ] Every extracted paragraph has a source tag
- [ ] All tables have been converted to markdown format
- [ ] Tracked changes/comments from Word documents captured
- [ ] Low-confidence extractions flagged with P1 questions
- [ ] All extracted content written to disk with source tags — no in-memory accumulation
