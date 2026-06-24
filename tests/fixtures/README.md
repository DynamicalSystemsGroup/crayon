# Shared golden fixtures (language-neutral)

These fixtures are consumed by **both** test suites — the JS extension converter and the Python CLI
checker — so the two implementations cannot drift (INV-7). They are the executable definition of the
Canonical Markdown profile in [`../../docs/repo-layout.md`](../../docs/repo-layout.md).

Planned contents (added alongside the code that needs them, TDD-style):

- `docs-json/` — raw Google Docs `documents.get` JSON inputs.
- `canonical-md/` — the expected Canonical Markdown output for each `docs-json` input.
- `tier1-figure-table.*` — a Doc exercising Tier-1 inline image + rectangular aligned table.
- `tier3-merged-drawing.*` — a Doc exercising Tier-3 (merged cells / Drawing): asserts the lint
  warning fires and the snapshot retains it.
- `outline-wellformed.md` / `outline-skipped-level.md` / `outline-multi-top.md` — pass/fail cases for
  the `outline_well_formed` rule.
- `rules/` — pass/fail inputs for the other starter rule kinds (`word_count`, `section_drift`) plus a
  sample custom `Rule` under `.crayon/checks/` (exercises auto-discovery + the pure/deterministic
  contract).
- `publish/` — a `publish.yaml` + an example `Sink` (e.g. `file`) routing a file/section, asserting a
  deterministic, dependency-free write (`crayon publish`).
- `settling/` — a Docs import whose first re-export differs from the source Markdown, asserting the
  single **settling commit** and that the subsequent unedited cycle is a no-op (INV-1).
- `mirror-refresh/` — a `main`-mirror Doc carrying **comments + suggestions** plus an advanced canon,
  for the capture-before-overwrite tests (T-MR-1..T-MR-7 / INV-10): captured state, the
  `crayon-review/<sha>` tag, idempotence, no-loop, and the timelock.
- `template/` — a docId-less **template seed** (`.crayon/template.json` + lipsum `.md`), asserting
  template-mode `crayon check` and that instantiation mints docIds to a valid live repo (UAT-E5).

Each conversion fixture is a matched triple: `(docs-json input, expected canonical markdown, expected
re-import Docs requests)`. If a rule in the profile changes, its fixtures change in the same commit.
