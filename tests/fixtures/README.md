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
  `crayon check`.

Each conversion fixture is a matched triple: `(docs-json input, expected canonical markdown, expected
re-import Docs requests)`. If a rule in the profile changes, its fixtures change in the same commit.
