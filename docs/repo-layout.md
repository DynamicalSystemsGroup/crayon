# Repo layout & the Canonical Markdown profile

This document pins the two things that must be *exact* for Crayon's determinism invariants
(INV-1, INV-7) to hold: the on-disk layout of a Crayon-managed branch, and the normalized Markdown
profile both the JS converter (producer) and the Python CLI (consumer/checker) obey.

> This is a **contract document**, not implementation. It accompanies [`../SPEC.md`](../SPEC.md) and
> the JSON Schemas under [`../schemas/`](../schemas/).

---

## 1. On-disk layout (one branch = one Drive folder = one git branch tree)

```
<branch root>/
  intro.md                        # untabbed Doc  → a single file
  method.md                       # untabbed Doc
  appendix/                       # tabbed Doc    → a directory of files…
    a.md                          #   tab "a"
    b.md                          #   tab "b"
    .crayon-tabs.json             #   marker → schemas/crayon-tabs.schema.json
  section/                        # plain Drive folder (NO marker) → recurse
    notes.md
  assets/
    <sha256>.png                  # content-hash-named figure binaries (Tier 1)
  .crayon/
    manifest.json                 # whole-tree index → schemas/manifest.schema.json
    snapshots/
      <docId>.json                # raw Docs documents.get JSON, REPLACED each commit
```

**Directory disambiguation rule (the one-bit test):**

> A directory containing `.crayon-tabs.json` **is a tabbed Google Doc**; any other directory is a
> **plain Drive folder**. A `.md` file (no directory) is an **untabbed Doc**.

**Per-Doc checkpoint** (divergence detection) is *not* a file in the repo — it is stamped into each
Google Doc's `DeveloperMetadata` (see [`../schemas/checkpoint.schema.json`](../schemas/checkpoint.schema.json)).
The repo side of the comparison is the branch HEAD SHA + the committed Markdown content hash.

---

## 2. Canonical Markdown profile (normalized GFM)

The converter MUST emit Markdown in exactly this normalized form so that an unedited Pull→Push
produces **no diff** (INV-1) and the JS and Python readers agree (INV-7).

### File-level
- **Encoding:** UTF-8, no BOM.
- **Newlines:** LF (`\n`) only.
- **Trailing newline:** exactly one at end of file.
- **No trailing whitespace** on any line.
- **Blank lines:** exactly one blank line between block elements; never two or more consecutive.

### Frontmatter (YAML, required)
- Delimited by `---` lines at the very top.
- **Fixed key order:** `title`, then `subtitle` (if present), then `crayon` (object with `docId`,
  optional `tabId`). No other top-level keys in v1.
- Strings unquoted unless they require quoting; quoting style is double-quote when needed.
- Maps to Docs paragraph styles: `title` ⇄ `TITLE`, `subtitle` ⇄ `SUBTITLE`.

```markdown
---
title: Introduction
subtitle: A short preface
crayon:
  docId: 1AbC...xyz
---
```

### Headings
- **ATX only** (`#`…`######`), one space after the hashes, no closing hashes.
- `HEADING_1`…`HEADING_6` ⇄ `#`…`######`. (The document `TITLE` lives in frontmatter, not as `#`.)
- The `(level, text)` sequence is the **outline tree** — the enforced invariant (INV-2).

### Inline
- **Emphasis:** `*italic*`, `**bold**` (asterisks, not underscores).
- **Links:** `[text](url)`.
- **Images:** `![alt](assets/<sha256>.<ext>)` — path is always the content-hash asset path (INV-6).
- **Escaping:** literal `|` inside table cells as `\|`; hard line breaks inside cells as `<br>`.

### Lists
- Unordered marker: `-` (hyphen), one space after.
- Ordered marker: `1.` style, one space after; numbers normalized to ascending from 1.
- Nested lists indented by 2 spaces per level.

### Tables (GFM pipe tables — Tier 1)
- Leading and trailing `|` on every row.
- **Exactly one space** padding inside each cell boundary: `| cell |`.
- Header separator row uses alignment markers reflecting column alignment:
  - left → `:---`, center → `:--:`, right → `---:`, none → `---` (min three dashes; not padded to
    column width — fixed minimal form for stable diffs).
- Cell content is single-line; newlines → `<br>`, pipes → `\|`.
- Merged cells / nested tables / multi-block cells are **Tier 3** (snapshot-only, lint-warned) and
  are NOT emitted as GFM.

### Code
- Fenced code blocks with triple backticks; language tag preserved when present.

---

## 3. Conformance fixtures

`../tests/fixtures/` will hold matched triples used by **both** test suites:
`(docs-json input, expected canonical markdown, expected re-import docs requests)` plus malformed
inputs for the `crayon check` failure modes. These fixtures are the executable definition of this
profile; if a rule here changes, the fixtures change with it in the same commit.
