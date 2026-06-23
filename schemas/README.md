# Crayon on-disk data contracts (S5)

JSON Schemas (Draft 2020-12) for every persisted Crayon shape. These are the single source of truth
both the JS extension and the Python CLI validate against. All carry `schemaVersion: 1`.

| Schema | Lives at | Purpose |
|---|---|---|
| [`manifest.schema.json`](manifest.schema.json) | `.crayon/manifest.json` (per branch root) | Whole-tree index; every node tagged `file` / `doc` / `folder`. |
| [`crayon-tabs.schema.json`](crayon-tabs.schema.json) | `<dir>/.crayon-tabs.json` | Marker that a directory IS a tabbed Doc; maps tabs ⇄ files + order. |
| [`frontmatter.schema.json`](frontmatter.schema.json) | YAML head of each `.md` | `title` / `subtitle` / `crayon.docId` (+ `tabId`). |
| [`checkpoint.schema.json`](checkpoint.schema.json) | Google Doc `DeveloperMetadata` (not in repo) | Per-Doc sync checkpoint for divergence detection. |
| [`policy.schema.json`](policy.schema.json) | `.crayon/policy.json` | Outline rules + branch-protection declaration. |

See [`../docs/repo-layout.md`](../docs/repo-layout.md) for how these compose on disk, and
[`../SPEC.md`](../SPEC.md) §"Interfaces & contracts" for the full seam definitions.
