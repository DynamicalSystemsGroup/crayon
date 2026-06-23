# CLAUDE.md — working contract for the Crayon repo

This file tells Claude (or any LLM agent) how to work in this repository. Read
[SPEC.md](SPEC.md) first (the approved v1 design), then [docs/repo-layout.md](docs/repo-layout.md)
(on-disk layout + the Canonical Markdown profile) and [schemas/](schemas/) (the on-disk data
contracts) before authoring or modifying anything.

## Authorship & accountability
- **The maintainer owns the intellectual content** — intent, design decisions, and the reasoning that
  narrows the implementation space. Claude is the instrument.
- **Git co-author trailer:** add `Co-Authored-By: Claude ...` **only** on commits where the
  maintainer has explicitly delegated decision authority for that work. This is rare. Default = **no
  trailer**.

## Intent-Based-Coding (the paradigm this repo is built around)
Crayon is developed by successively shrinking the space of feasible implementations until the
remaining code matches intent. The mechanism is deliberate and is the point, not overhead:
- **Specs and contracts come before code.** SPEC.md, the Canonical Markdown profile, and the JSON
  Schemas define the envelope; implementation must fit inside it. Do not write implementation ahead
  of the spec/contract that constrains it.
- **The invariants are the acceptance boundary.** INV-1…INV-7 (SPEC.md §"Test strategy") and the
  per-UI acceptance tests (UAT-A…D) are executable definitions of "correct". Tests are written before
  the code that satisfies them.
- **Expect iterative challenge.** Plans get stress-tested (edge cases, alternatives) before approval;
  refine the artifact each round rather than rushing to code.
- **Smallest, simplest, most opinionated** design that satisfies intent — Crayon supports one blessed
  workflow, not all possible workflows (SPEC.md §"Workflows … out of scope").

## Determinism discipline (specific to this codebase)
- The Canonical Markdown profile ([docs/repo-layout.md](docs/repo-layout.md)) is exact. An unedited
  Pull→Push must produce **no diff** (INV-1) and the JS converter and Python CLI must agree on the
  shared golden fixtures (INV-7). Changes to the profile and its fixtures go in the same commit.
- All persisted shapes are schema-validated against [schemas/](schemas/) and carry `schemaVersion`.

## Safety / git
- Standard Claude Code safety: no `--no-verify`, no force-push, no closing issues/PRs unless asked.
- Commit or push only when asked.
