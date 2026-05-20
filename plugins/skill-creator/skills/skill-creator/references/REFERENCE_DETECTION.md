# Automatic detection of reference files

To read when the initial invocation of `skill-creator` contains one or more existing file paths. This workflow replaces free-form prompts like "analyze this file deeply" with a structured sequence of questions, ending with a reference brief injected into Steps 4, 6a, and 6c of the main workflow.

**Core principle**: the user provides one or more reference files (quality templates, well-done examples, files to reproduce). `skill-creator` **reads them, really analyzes them**, and proposes components **DERIVED FROM THE ACTUAL CONTENT** of the files (not a hardcoded list). The user checks the ones the generated skill must reproduce.

---

## -1.1 Scan the invocation message

Look in the full initial user message (the sentence that triggered this skill) for **all tokens that look like file paths**:

- Absolute: `/Users/...`, `/home/...`, `/var/...`, etc.
- With tilde: `~/foo/bar.md`
- Relative: `./docs/spec.txt`, `src/app.py`, `data/users.csv`
- Glob patterns: `docs/*.md`, `src/**/*.py`
- Local URLs: `file:///path/to/file`

**No extension filter.** Any token resembling a path is a candidate.

**Validation**: for each candidate, verify the file (or folder, or glob) **exists** on disk (`ls <path>` or direct attempt at `Read`). If the path doesn't exist, flag it to the user in one line and ask whether to correct or ignore.

For glob patterns: expand them into concrete file lists (`ls docs/*.md`).

If no path detected or validated → SKIP this whole step, return to the main workflow at Step 0.0.

---

## -1.2 Read each detected file

For each validated file:

1. Read with the `Read` tool. The tool automatically handles md, txt, code, json, yaml, csv, PDF, images.
2. For very large files (>1000 lines): read the first 500 lines + a sample from the middle/end to get an overview. Tell the user it was truncated.
3. For an unreadable binary: flag "Binary file unreadable, skipping" and continue with the others.

Show a short recap to the user **before asking questions**:

> "I found N reference file(s):
> - `<path1>` (<format>, <size>) — <1-line content description>
> - `<path2>` ...
>
> I'll ask a few questions to understand how to use them."

---

## -1.3 AskUser: role of each file

`AskUserQuestion` (multiSelect enabled to allow stacking roles):

> "What is the role of this/these file(s) in the skill you want to create?
>
> - **OUTPUT model**: the generated skill must produce files that look like these (same structure, same style, same conventions)
> - **INPUT model**: the generated skill must know how to read / accept this format as input
> - **Domain / rules reference**: the skill must apply the rules, conventions, or domain logic extractable from these files
> - **Example use case**: just to understand context, not a direct template"

**If multiple files** and you suspect roles may differ: first ask "Do all these files share the same role?" — if not, ask per file individually.

This question is **universal**: it works for any file type (domain doc, code, data, mockup, etc.).

---

## -1.4 AskUser: analysis depth

`AskUserQuestion` (single select):

> "How deep should the pattern and rule extraction go?
>
> - **Structural** (fast): just the main sections / functions / columns / elements of the file
> - **Deep** (recommended): structure + intents + style + implicit conventions
> - **Exhaustive**: every detail, every sub-pattern, every deducible domain rule"

This question is **universal**: it tunes the analysis fineness in -1.5.

---

## -1.5 ANALYZE the files at the chosen depth

**This is the most important step.** `skill-creator` actually analyzes the file content to identify the **structuring components present**. The result is used to dynamically generate options for question -1.6.

### What to extract (depending on file type)

Auto-adapt the analysis to the detected format:

**For a markdown / text document**:
- Sections and hierarchy (h1/h2/h3)
- Block types present (mermaid, code blocks, tables, lists, quotes, math formulas, images)
- Diagram style (sequenceDiagram, flowchart, classDiagram, ER, etc. — if mermaid)
- Numbering patterns (1.1/1.2, A/B/C, roman, etc.)
- Naming conventions (titles in French vs English, case, length)
- Writing style (paragraph length, active/passive voice, tone, language, level of detail)
- Table structure (recurring columns, e.g. Code/Name/Type)
- Recurring typical sections (e.g.: Context, Goal, Actor, Scenario, Appendices)

**For a code file (.py, .ts, .js, etc.)**:
- Docstring style (Google, NumPy, reST, JSDoc, none)
- Type hints / annotations (presence, exhaustiveness)
- Naming conventions (snake_case, camelCase, PascalCase)
- Import organization (grouped stdlib/third-party/local, alphabetically sorted)
- Error handling pattern (custom exceptions, return codes, Result types)
- Testing style (pytest fixtures, parametrize, mock patterns)
- Architecture (classes vs functions, files-by-feature or files-by-layer)
- Validation (Pydantic, Zod, manual, etc.)
- Logging (loguru, stdlib, structlog)

**For a data file (CSV, JSON, YAML, XML)**:
- Schema (columns / keys present, types, mandatory vs optional)
- Field naming conventions
- Identifier format (regex, length, prefix)
- Header language vs content language
- Granularity (1 row = 1 entity?)
- Value patterns (detected enums)

**For a PDF / docx**:
- Sections and structure
- Tables, charts, images present
- Style (template, formal/informal)
- Recurrences (header/footer, numbering)

### If multiple files provided

Explicitly detect **COMMON patterns** across files vs file-specific patterns. Typical use case: the user provides 3 Python files "examples of good practices" and wants the generated skill to reproduce patterns present in **all 3** (common patterns are the invariants to enforce).

Tag each detected component with its frequency: "present in 3/3 files" (common) vs "present in 1/3" (specific).

### Depth

- **Structural**: extract only elements visible at first read (sections, blocks, headers).
- **Deep**: add implicit conventions (style, naming, recurring patterns).
- **Exhaustive**: add every detail, every sub-pattern, every deducible domain rule. Slower.

---

## -1.6 AskUser: components to reproduce (DYNAMIC)

**This is where the list is generated from the -1.5 analysis, never a fixed list.**

`AskUserQuestion` (multiSelect enabled) with a list dynamically built from the detected components:

> "Here are the components I detected in your reference files. Check the ones the generated skill must reproduce in its outputs:"

Phrase each option in **concrete**, understandable terms, citing the source file when useful:

**Examples for a domain-doc .md**:
- "Sections Context / Actor / Goal / Scenario / Sequence / Attributes"
- "mermaid sequenceDiagram"
- "Summary table with Code / Name / Type columns"
- "1.1 / 1.2 / 1.3 hierarchical numbering"
- "Narrative style in present tense, active voice, in French"

**Examples for good-practice .py files**:
- "Google-style docstrings on all public functions"
- "Exhaustive type hints (params + returns)"
- "Imports grouped stdlib / third-party / local, sorted"
- "Error handling with custom AuthError exception (common across 3/3 files)"
- "pytest tests with parametrize fixtures"

**Examples for a CSV**:
- "Required columns: id, name, category"
- "Codes matching regex `^[A-Z]{2}\\d{4}$`"
- "Headers in French, values in English"

**Practical limits**: `AskUserQuestion` accepts up to ~10 options. If you detect more than 10 components, group the less important ones or propose the 10 most salient with a note that others can be added via "Other".

---

## -1.7 Synthesize the reference brief

Build in memory a structured brief that will be **auto-injected** into Steps 4, 6a, and 6c of the main workflow:

```
REFERENCES
- File 1: <path> — role: <X>, depth: <Y>
- File 2: ...

COMMON DETECTED PATTERNS (if multiple files)
- <pattern 1> (3/3)
- <pattern 2> (3/3)

SPECIFIC PATTERNS (info)
- <pattern 3> (1/3, file <X>)

COMPONENTS CHECKED BY USER (to reproduce)
- <component 1>
- <component 2>
- ...

GENERATED BRIEF FOR THE SKILL
"The skill <slug> should take <inferred input type> as input, and produce as output a file containing: <component 1>, <component 2>, ... The style must reproduce <detected style>."
```

This brief is then used:

- **Step 4 (description)**: pre-fill a description suggestion based on the brief.
- **Step 6a (interactive mode)**: use the checked components as a starting step skeleton (1 step per major component: "Step 1 — Read the input", "Step 2 — Generate the diagram", etc.).
- **Step 6c (minimal skeleton)**: replace the generic `Step N — TODO` entries with pre-titled steps matching the components.

---

## Edge cases

### User provides a folder instead of a file

If a detected path is a folder: ask via `AskUserQuestion`:
> "`<folder>` is a directory. You want to:
> - Analyze all files in the folder (or a subset by extension)
> - Pick a specific file inside
> - Ignore this path"

### No interesting component detected

If the -1.5 analysis returns nothing usable (empty file, unrecognized format, content too short): warn the user and go straight to Step 0.0 without asking -1.6.

### User wants to add components not detected

At question -1.6, the "Other" option is always available. Free text to add components the analysis didn't capture.
