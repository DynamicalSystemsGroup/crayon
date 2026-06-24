# Crayon

**A deliberately small seam between Google Docs (ergonomic editor) and GitHub (canonical record).**

Write in Google Docs; govern in GitHub. Crayon assigns each system the role it's good at, using
git's own local/remote mental model:

- **GitHub = remote.** Canon is the remote `main` branch. Branches, history, PRs, and policy live here.
- **Google Doc/Drive = shared local working copy.** Ergonomic, multiplayer, disposable. (The `main`
  mirror is **read-only** — a projection of canon; you edit on branches.)
- **Crayon Chrome extension = the working-copy tooling** — pull, push, branch, open PR, navigate.
- **`crayon` Python CLI = minimalist governance tooling** — scaffold a repo, author CI **rules**
  (`check`) and content **routes** (`publish`); branch protection is configured in GitHub's web UI.

Crayon is **opinionated and serverless**: it supports one blessed workflow well, and the extension
talks directly to the GitHub + Google APIs from the browser (no backend to host).

## Status

**Specification stage — no implementation code yet.** The full design is in **[SPEC.md](SPEC.md)**.
This repository currently contains:

| Path | What it is |
|---|---|
| [`SPEC.md`](SPEC.md) | The approved v1 specification (the source of truth for the design). |
| [`docs/wbs.md`](docs/wbs.md) | The work breakdown structure — packages mapped to contracts + acceptance gates. |
| [`docs/repo-layout.md`](docs/repo-layout.md) | The on-disk repo layout + the **Canonical Markdown** profile. |
| [`docs/design-review.md`](docs/design-review.md) | The design-review decision log (DR-1…DR-14 + resolutions). |
| [`docs/screening-guide.md`](docs/screening-guide.md) | Reviewer checklist for constitutive changes (rules/sinks/workflows). |
| [`schemas/`](schemas/) | JSON Schemas for the on-disk data contracts (S5). |
| [`tests/fixtures/`](tests/fixtures/) | Where shared golden fixtures will live (JS + Python consume them). |
| [`extension/`](extension/) | Placeholder for the Chrome extension (not yet implemented). |
| [`cli/`](cli/) | Placeholder for the `crayon` Python CLI (not yet implemented). |
| [`LICENSE`](LICENSE) | Apache License 2.0. |

## The core model in one picture

```
GitHub repo  ──────────────  Google Drive  "Crayon / <owner>/<repo>"
  ├ branch: main     ⇄  Folder "main"
  │    ├ intro.md            ⇄  Doc "intro"           (untabbed Doc = one file)
  │    └ appendix/           ⇄  Doc "appendix" + tabs (tabbed Doc = a directory of files)
  │         ├ a.md                ⇄  tab "a"
  │         └ .crayon-tabs.json   (marker: this dir IS a tabbed Doc, not a plain folder)
  └ branch: draft   ⇄  Folder "draft"
```

A branch is a Drive folder; a file is a Doc; tabs are optional within-file grouping. See
[SPEC.md](SPEC.md) for the invariants, sync protocol, fidelity tiers, interface contracts, sequence
diagrams, and the TDD/UAT plan.

## Next steps (build sequence)

Per SPEC.md §"Suggested build sequence": (1) ✅ repo skeleton + spec, (2) `crayon init` CLI,
(3) extension auth spike, (4) N=1 Pull/Push, … Each step is TDD — tests and the named invariants
(INV-1…INV-10) come before code. See [`docs/wbs.md`](docs/wbs.md) for the full breakdown.

## License

[Apache License 2.0](LICENSE).
