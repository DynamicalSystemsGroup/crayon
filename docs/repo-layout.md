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
    rules.yaml                    # crayon check rule set → schemas/rules.schema.json
    publish.yaml                  # crayon publish routes → schemas/publish.schema.json (optional)
    checks/                       # custom Rule subclasses (copy-the-pattern); auto-discovered
      *.py
    sinks/                        # custom Sink subclasses (copy-the-pattern); auto-discovered
      *.py
    snapshots/
      <docId>.json                # raw Docs documents.get JSON, REPLACED each commit
.gitattributes                    # marks .crayon/snapshots/** linguist-generated + -diff
```

`crayon init` writes a `.gitattributes` so the per-commit JSON snapshots stay in history (fidelity
backstop) but **collapse in PR review** — reviewers see the Markdown diff, not the snapshot blob:

```gitattributes
.crayon/snapshots/** linguist-generated -diff
```

`.crayon/rules.yaml` and `.crayon/publish.yaml` are YAML for low-code authoring but validate against
the JSON Schemas above (YAML is parsed to JSON first). Custom rules/sinks live as plain Python files
under `.crayon/checks/` and `.crayon/sinks/`; the CLI imports them so each subclass auto-registers its
`kind`. **Branch protection is not on disk** — it is configured by a human in GitHub's web UI.

**Read-only `main` review-state archive (a tag, not in the working tree).** Before the `main` mirror is
overwritten by a refresh, its review state (Docs JSON + suggestions + Drive comments) is committed and
pointed to by a **git tag** — `crayon-review/<defaultBranchSha>` (capture commit parented at the canon
SHA, so it is *not* on `main`'s history). A tag is **discoverable** (`git tag`, GitHub tags UI) yet not
a branch, and the refresh Action triggers only on default-branch pushes/merges — **never on tags** — so
the capture cannot fire a refresh (INV-10 / MR-1..MR-7). It is intentionally *not* part of any branch's
checked-out tree.

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

### Round-trip & settling (the basis of INV-1)
The end-to-end cycle is `git Markdown → Docs batchUpdate → Google Doc → documents.get → Markdown`.
Google Docs may **normalize content on import** (named-style coercion, list numbering, whitespace), so
the Markdown re-exported immediately after a Pull can differ from the Markdown that was pushed in.

To keep this deterministic:
- A Doc's **checkpoint content hash** (`checkpoint.lastSyncContentHash`) is taken over the canonical
  Markdown **re-exported from the Doc** at the last successful Pull/Push — *not* over the git source.
  So an unedited Push re-exports, hashes, matches the checkpoint, and commits nothing.
- The **first** Pull of a Doc may therefore produce one **settling commit** (the normalization delta
  between the git source and the Doc's faithful re-export). After it settles, the branch is a fixed
  point: Pull→Push with no edits is a no-op (INV-1, *idempotent after settle*).
- For a **tabbed Doc** (one `docId` ↔ many `.md`), the hash domain is the ordered concatenation of the
  tab files' canonical bytes, in `.crayon-tabs.json` `order`.

Changes to this profile that affect re-export must be proven against the live round-trip rung before
they are trusted (see SPEC.md §"Test strategy").

### Sections (for `section_drift` and file/section sources)
- A **section** is the heading-bounded region from a heading down to (but excluding) the next heading
  of equal-or-higher level — its nested subsections included.
- A section's **text**, for drift comparison and for `crayon publish` file/section sources, is the
  Canonical Markdown body of that region (the heading line excluded), normalized by this profile. Two
  sections are "the same" iff their canonical bodies are byte-identical — so drift checks are
  deterministic for the same reason round-trips are (INV-1).

---

## 3. Conformance fixtures

`../tests/fixtures/` will hold matched triples used by **both** test suites:
`(docs-json input, expected canonical markdown, expected re-import docs requests)` plus malformed
inputs for the `crayon check` failure modes. These fixtures are the executable definition of this
profile; if a rule here changes, the fixtures change with it in the same commit.
