# Screening guide — reviewing constitutive changes

Crayon separates **operative** work (the fast, primary path: authoring version-controlled docs under
CI/CD) from **constitutive** work (rare, slow policymaking *over* the repo). This guide is for
reviewers of constitutive changes — the only places where repo-supplied code executes in CI.

> Constitutive changes are **code, reviewed as code** (SEC-9). They are gated by branch protection.
> Constraining them to the `Rule`/`Sink` ABC templates (SEC-7) does not by itself prevent malice — it
> shrinks and standardizes what you must screen. This checklist is that screen.

## What counts as a constitutive change
A PR is constitutive if it touches any of:
- `.crayon/checks/*.py` — custom `Rule` subclasses (run by `crayon check` on **PRs**)
- `.crayon/sinks/*.py` — custom `Sink` subclasses (run by `crayon publish` **post-merge**)
- `.crayon/rules.yaml` / `.crayon/publish.yaml` — which rules/sinks are active and their params
- `.github/workflows/*` — the Actions that invoke `crayon check` / `crayon publish`

These deserve a higher bar of review than operative doc edits. When in doubt, treat it as constitutive.

## Checklist

### Workflow files (highest risk)
- [ ] PR-check jobs use `on: pull_request` — **never `pull_request_target`** (SEC-8).
- [ ] `permissions:` is least-privilege (read-only `GITHUB_TOKEN` for the check job); no `write` it
      doesn't need.
- [ ] **No secrets** are exposed to any job that runs PR-supplied code; publish secrets appear only in
      the post-merge job, gated to the protected branch.
- [ ] Jobs invoke only `crayon check` / `crayon publish` — **no arbitrary `run:` scripts**, no
      `curl | sh`, no fetching/executing remote code (SEC-7).
- [ ] No new `actions/*` or third-party action pinned to a mutable ref (pin by SHA if used).

### Custom rules (`.crayon/checks/`)
- [ ] The file defines a `Rule` subclass and nothing else of consequence — no top-level side effects at
      import time (imports run when `crayon check` discovers the file).
- [ ] **Network-free and deterministic** — no sockets, no `urllib`/`requests`, no clock/random, no env
      reads, no filesystem writes outside the working tree. A rule is a pure predicate over its domain.
- [ ] No `subprocess`, `os.system`, `eval`/`exec`, dynamic import of remote code.
- [ ] Acts only within its declared `domain` (`repo`/`file`/`files`/`section`/`sections`); returns a
      `bool` with a descriptive message.

### Custom sinks (`.crayon/sinks/`)
- [ ] Defines a `Sink` subclass; no import-time side effects.
- [ ] Writes only to the destination its config declares; the destination host is expected/allowlisted,
      not attacker-controlled via PR.
- [ ] Reads only the named `source` (file or file/section) — does not vacuum the whole tree or env.
- [ ] Any credential comes from the post-merge job's scoped secret, is used only for the declared
      endpoint, and is never logged.

### Config (`rules.yaml` / `publish.yaml`)
- [ ] Validates against `schemas/rules.schema.json` / `schemas/publish.schema.json`.
- [ ] References only known `kind`s and sane params; no surprising new sink destinations.

## Why this is tractable
Because automation is constrained to the ABC templates (SEC-7), a malicious change cannot hide in an
arbitrary build step — it must appear as a `Rule`/`Sink` subclass or a workflow edit, both of which are
small, shaped, and on this checklist. PR-check code runs without secrets (SEC-8), so the worst case for
a fork PR is wasted CI minutes, not exfiltration.
