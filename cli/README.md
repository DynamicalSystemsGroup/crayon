# `crayon` Python CLI (placeholder — not yet implemented)

The governance layer, run by a developer in an IDE (the UI-3 surface). Per [`../SPEC.md`](../SPEC.md)
§Components, the commands are:

- `crayon init` — scaffold a Crayon-shaped repo (starter `intro.md`, `.crayon/` layout, default
  GitHub Action running `crayon check`, branch protection on `main`).
- `crayon check` — pure, deterministic, network-free policy gate: outline well-formedness +
  manifest/marker consistency + frontmatter validity + Canonical-Markdown conformance. Exit non-zero
  on violation. This is the required status check (S6/S8).
- `crayon policy apply` — apply the declarative `.crayon/policy.json` (branch protection / required
  checks) via the GitHub API; idempotent (S7).
- `crayon doctor` — read-only check that a repo is Crayon-shaped.

Intended tooling: `uv` + `pytest`, PyGithub / GitHub REST. Nothing here is built yet — see SPEC.md
§"Suggested build sequence" (the CLI is build step 2, since it defines the canonical repo shape the
extension consumes).
