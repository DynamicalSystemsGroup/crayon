# `crayon` Python CLI (placeholder — not yet implemented)

Minimalist dev tooling, run by a developer in an IDE (the UI-3 surface). It is deliberately **thin** —
just enough example machinery that a policy author copies the patterns to build their own. Two
symmetric primitives: **rules** ("check this") and **sinks** ("write this there"). Per
[`../SPEC.md`](../SPEC.md) §Components, the commands are:

- `crayon init` — scaffold a Crayon-shaped repo (starter `intro.md`, `.crayon/` layout incl. a
  starter `rules.yaml` and copy-me `.crayon/checks/` examples, default GitHub Action running
  `crayon check`). It does **not** set branch protection — that is configured by a human in GitHub's
  web UI (S7).
- `crayon check` — pure, deterministic, network-free **rule engine**: a `Rule` ABC → concrete *kinds*
  → instances materialized from `.crayon/rules.yaml` or subclassed in `.crayon/checks/`. Starter kinds:
  `outline_well_formed`, `word_count`, `section_drift`. Also enforces manifest/marker consistency,
  frontmatter validity, Canonical-Markdown conformance. Exit non-zero on violation; this is the
  required status check (S6/S8).
- `crayon publish` — the symmetric **sink engine** ("write this there"): a `Sink` ABC → concrete kinds
  → instances from `.crayon/publish.yaml` or subclassed in `.crayon/sinks/`, routing a file/section to
  an external endpoint post-merge. Ships dependency-free example sinks only (`file`, `http_post` via
  stdlib `urllib`); real targets are user subclasses (S9).
- `crayon doctor` — read-only check that a repo is Crayon-shaped.

Using the CLI to author rules/sinks is **constitutive** work — slow policymaking over the repo, not the
fast operative path (authoring governed docs). Because `crayon check` / `crayon publish` execute
repo-supplied Python (`.crayon/checks/`, `.crayon/sinks/`), those changes are code, reviewed as code:
see [`../docs/screening-guide.md`](../docs/screening-guide.md) and SPEC.md §"Trust model".

Intended tooling: `uv` + `pytest`, PyGithub / GitHub REST (+ PyYAML/PyGithub as base deps), **no
third-party runtime deps per rule / sink / destination**. Nothing here is built yet — see SPEC.md
§"Suggested build sequence" (the CLI is build step 2, since it defines the canonical repo shape the
extension consumes).
