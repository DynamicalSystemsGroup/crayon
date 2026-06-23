# Crayon v1 — Work Breakdown Structure

This is the bridge between SPEC.md's prose build-sequence and executable work. Each **work package**
names its *deliverable*, the *contract(s)* it satisfies, and the *acceptance gate* (the named
invariant / UAT / security criterion) that defines "done" — consistent with the Intent-Based-Coding
discipline in [`../CLAUDE.md`](../CLAUDE.md): contracts and tests precede code; invariants are the
acceptance boundary.

> Read [`../SPEC.md`](../SPEC.md), [`repo-layout.md`](repo-layout.md), and
> [`../schemas/`](../schemas/) first — this document references their contracts (S1–S8), invariants
> (INV-1…INV-7), UAT suites (UAT-A…E), and security criteria (SEC-1…n) by id.

## How to read a package

- **WBS id** — hierarchical (`2.3`). Numbered sections 1–9 track SPEC.md §"Suggested build sequence";
  lettered sections (S, O) are cross-cutting concerns that span the numbered work.
- **Deliverable** — the concrete artifact produced.
- **Satisfies** — the contract(s) from SPEC.md §"Interfaces & contracts" (S1–S8) and the
  layout/profile in [`repo-layout.md`](repo-layout.md).
- **Acceptance gate** — the executable definition of done: named invariants, UAT cases, security
  criteria, and the test level (unit / contract / integration-with-fakes / E2E).
- **Depends on** — upstream WBS ids that must be green first.

Each package follows the per-step TDD loop from SPEC.md §"Test strategy": (a) extend contract+schema,
(b) failing unit+contract tests, (c) failing integration test against the fakes, (d) implement to
green, (e) add the invariant fixture. **No package is done until its gate is green.**

---

## Credentials, identity & test-loop model (cross-cutting acceptance constraint)

Authentication is a *per-surface contract*, not just an extension concern. These are acceptance
criteria attached to the relevant packages:

- **Web user (Chrome extension, UI-1):** **never** sees a separate login or an API key. Identity is
  the serverless dual-OAuth only — GitHub App device flow + Google PKCE, tokens in
  `chrome.storage.session`. Any design that would require the web user to paste a token fails the gate
  for WBS 3.2 / 3.3.
- **IDE user (`crayon` CLI, UI-3):** may supply credentials via a **gitignored `.env`** (GitHub token
  / Google credentials). Optional-by-surface: `crayon check` / `doctor` stay network-free and need no
  credentials; only `init` (S7) reads them. Branch protection is set by a human in GitHub's web UI —
  Crayon never writes it, so the CLI needs no policy-write scope.
- **Three test rungs** (the spec's pyramid, made concrete against available access):
  1. **Fakes** (`FakeGitHub`/`FakeDrive`, WBS 1.3) — fast, deterministic, network-free; the default CI
     gate for all invariants.
  2. **Live build/test loop** (WBS 1.6) — real `api.github.com` + Docs/Drive against throwaway repos /
     a dedicated Drive area, run as the real logged-in maintainer. Validates that the fakes model
     reality. **Never spoof actors.**
  3. **User-acceptance testing** (the UATs) — the maintainer drives the real surfaces: the extension
     authenticates through the browser's own login, the CLI through its `.env`/`gh` session. No
     injected credentials at this rung.

---

## WBS 0 — Baseline (complete)

SPEC.md, [`repo-layout.md`](repo-layout.md), and `schemas/*.schema.json` are committed.
= build-sequence step 1. Recorded as the WBS root so dependencies resolve.

---

## WBS 0.1 — Spec amendment: onboarding + security (precursor)

Per Intent-Based-Coding, the missing user story and the security boundary enter the **spec** before
they become code. Amends SPEC.md (maintainer owns/approves the intent; the proposed text is drafted
for review):

- **New surface — UI-0 "Install & onboard":** *one cannot use the plugin without installing it.* Adds
  install→authenticate→bind-first-repo→first-Pull as a first-class surface alongside UI-1/2/3.
- **New workflow W0 — install & onboard:** discover (doc site) → install (dev-install now / Web Store
  later) → first-run OAuth (both providers, in-browser) → bind a repo → first successful Pull. CLI
  variant: `uv`/`pipx` install → `crayon init` quickstart.
- **New UAT-E — onboarding:** a first-time user, starting from nothing, reaches a working in-sync
  state by following only the doc site, with **no separate login or API key** (web) / a gitignored
  `.env` (IDE).
- **New security acceptance criteria SEC-1…n** referenced by the spec's Test strategy: MV3
  least-privilege, no remote code, token-handling, supply-chain integrity, content-script injection
  safety — the executable boundary for "nothing malicious reaches the user's browser."

**Satisfies:** the spec contract itself (precedes WBS S and WBS O). **Acceptance gate:** maintainer
approval of the spec text; every package in WBS S / WBS O traces to it. **Depends on:** WBS 0.

---

## WBS 1 — Shared foundations (cross-cutting; unblock everything)

Not in the spec's numbered sequence but prerequisites its Test strategy implies. Built first because
every later package's gate runs against them.

### 1.1 Golden fixture corpus — `tests/fixtures/`
- **Deliverable:** the matched-triple fixtures from [`../tests/fixtures/README.md`](../tests/fixtures/README.md)
  — `docs-json/` inputs, `canonical-md/` expected outputs, expected re-import Docs requests;
  `tier1-figure-table.*`, `tier3-merged-drawing.*`; `outline-wellformed.md`,
  `outline-skipped-level.md`, `outline-multi-top.md`.
- **Satisfies:** [`repo-layout.md`](repo-layout.md) §Canonical Markdown profile (its executable
  definition).
- **Acceptance gate:** contract — language-neutral, consumed by both suites; the substrate for INV-7.
  Seeded minimally, grown TDD-style alongside the code that needs them; the harness + directory
  contract land now.
- **Depends on:** WBS 0.

### 1.2 Shared schema-validation utility (both languages)
- **Deliverable:** a thin validator each side uses against `schemas/` (`manifest`, `crayon-tabs`,
  `frontmatter`, `checkpoint`, `rules`, `publish`), asserting `schemaVersion`. YAML configs are parsed
  to JSON before validation.
- **Satisfies:** S5. **Acceptance gate:** contract — every S5 fixture validates; malformed fixtures
  fail. **Depends on:** WBS 0.

### 1.3 API fakes — `FakeGitHub`, `FakeDrive`/`FakeDocs`
- **Deliverable:** in-memory test doubles exercising real control flow without network. FakeGitHub
  models refs, blobs/trees/commits (Git Data API), branch + PR endpoints with expected-SHA
  preconditions; FakeDrive/FakeDocs models folders, files, `DeveloperMetadata`, `documents.get`,
  `batchUpdate`.
- **Satisfies:** S2, S3 (as test surfaces). **Acceptance gate:** integration — the harness all of WBS
  3–4 run against; correctness proven by the first flows that use them (4.4–4.6). **Depends on:**
  WBS 0.

### 1.4 Cross-impl conformance harness (INV-7 runner)
- **Deliverable:** a runner each suite invokes that feeds `docs-json/` through the converter/reader and
  diffs against `canonical-md/`, byte-for-byte.
- **Satisfies:** S4 invariant (kills the two-readers-drift risk). **Acceptance gate:** **INV-7** — JS
  converter output and Python reader agree on every shared fixture. **Depends on:** 1.1.

### 1.5 Repo + CI tooling bootstrap
- **Deliverable:** `cli/pyproject.toml` (uv + pytest), `extension/package.json` + MV3 `manifest.json`
  skeleton, `.github/workflows/` running both suites on PRs, and a gitignored `.env.example` +
  `.gitignore` entry establishing the CLI credential convention (no secrets ever committed).
- **Satisfies:** tooling envelope in `cli/README.md` / `extension/README.md`; the IDE-user credential
  rule. **Acceptance gate:** CI green on a trivial test in each suite; `.env` is gitignored.
  **Depends on:** WBS 0.

### 1.6 Live build/test-loop harness (real-access rung)
- **Deliverable:** opt-in config + helpers that run flows against real `api.github.com` and Docs/Drive
  using the maintainer's `gh` session and Google accounts, scoped to throwaway repos / a dedicated
  Crayon Drive area; skipped by default in CI.
- **Satisfies:** validates S2/S3 fakes against reality; never spoofs actors. **Acceptance gate:** a
  smoke flow (throwaway repo → init → pull → push → cleanup) succeeds live and matches the fakes.
  **Depends on:** 1.3, 1.5.

---

## WBS 2 — `crayon` CLI (build step 2; rules/publish/governance from step 9)

Built before the extension because it **defines the canonical repo shape the extension consumes**. Kept
deliberately thin — two symmetric engines (rules = "check this", sinks = "write this there"), starter
kinds, and copy-the-pattern custom extension; **no third-party runtime deps**.

### 2.1 CLI skeleton + harness — `cli/`
- **Deliverable:** `crayon` entry point, arg parsing, exit-code contract, pytest harness.
  **Satisfies:** S6 (exit-code surface). **Depends on:** 1.2, 1.5.

### 2.2 Rule engine + starter kinds (`crayon check`)
- **Deliverable:** the `Rule` ABC + `Domain` enum (`repo`/`file`/`files`/`section`/`sections`) + a
  `kind` registry (auto-register on subclass) + YAML materialization from `.crayon/rules.yaml`
  (`rules.schema.json`) + custom discovery importing `.crayon/checks/*.py`. Starter concrete kinds:
  `outline_well_formed`, `word_count`, `section_drift`. Plus manifest/marker + frontmatter +
  Canonical-Markdown conformance checks folded into the run.
- **Satisfies:** S6; the outline invariant; the rule model. **Acceptance gate:** **INV-2**
  (`outline_well_formed`); **UAT-B2** (skipped-level PR fails); **UAT-C1** (deterministic, network-free
  pass/fail); **UAT-C2** (author a rule — low-code YAML enable *and* a custom subclass from
  `.crayon/checks/` both enforced). `section_drift` uses the pinned section-text definition
  ([`repo-layout.md`](repo-layout.md) §Sections). Unit over the `outline-*` + rule fixtures.
  **Depends on:** 2.1, 1.1.

### 2.3 Manifest / marker / frontmatter consistency check
- **Deliverable:** validation that `manifest.json` tree, `.crayon-tabs.json` markers, and `.md`
  frontmatter agree (docId presence per `kind`; `order` is a permutation of `tabs[].file`; frontmatter
  `docId`/`tabId` match the manifest).
- **Satisfies:** S5, S6. **Acceptance gate:** contract — consistent fixtures pass, each injected
  inconsistency fails. Unit. **Depends on:** 2.1, 1.2.

### 2.4 Canonical-Markdown conformance check (Python reader)
- **Deliverable:** the Python side of the profile — parse + verify a `.md` is canonical.
- **Satisfies:** S4 (consumer side), [`repo-layout.md`](repo-layout.md) profile. **Acceptance gate:**
  **INV-7** via 1.4; non-canonical inputs fail. Contract. **Depends on:** 2.1, 1.4.

### 2.5 `crayon init`
- **Deliverable:** scaffold a Crayon-shaped repo — starter `intro.md`, `.crayon/` layout (incl. a
  starter `rules.yaml` + copy-me `.crayon/checks/` examples), default Action running `crayon check`.
  **Does not set branch protection** (human act in GitHub's web UI).
- **Satisfies:** S7; reads GitHub creds from the gitignored `.env`/`gh` session. **Acceptance gate:**
  integration — against a throwaway repo (FakeGitHub or recorded fixtures), assert repo shape + the
  Action is written; `crayon doctor` then passes. Also exercised live via 1.6. **Depends on:** 2.1,
  1.3.

### 2.6 Sink engine + starter kinds (`crayon publish`)
- **Deliverable:** the symmetric "write this there" half — a `Sink` ABC (source = `file`/`section`) +
  `kind` registry + YAML materialization from `.crayon/publish.yaml` (`publish.schema.json`) + custom
  discovery importing `.crayon/sinks/*.py`. Dependency-free starter kinds: `file` (write to a local
  path), `http_post` (POST via stdlib `urllib`).
- **Satisfies:** S9. **Acceptance gate:** **UAT-C3** — an example sink routes a file/section
  deterministically; a custom destination is added by subclassing (same pattern as a rule). No
  third-party deps. **Depends on:** 2.1, 2.2 (shares the section-resolution + config-materialization
  machinery).

### 2.7 `crayon doctor`
- **Deliverable:** read-only "is this repo Crayon-shaped?" check. **Satisfies:** S6 (read-only).
  **Acceptance gate:** passes on a 2.5 scaffold, fails on a malformed tree. **Depends on:** 2.2, 2.3.

### 2.8 Default GitHub Action wiring
- **Deliverable:** the workflow that runs `crayon check` (and, post-merge, `crayon publish`) — the
  check's exit code is the status a human requires via branch protection. **Satisfies:** S8.
  **Acceptance gate:** **UAT-B2** (failing check blocks merge) end-to-end against branch protection
  the maintainer configured in GitHub settings. **Depends on:** 2.2, 2.6.

---

## WBS 3 — Extension foundations: auth spike + converter (build step 3)

The auth spike de-risks the serverless dual-OAuth assumption; the converter is the central seam (S4).

### 3.1 Extension skeleton — `extension/`
- **Deliverable:** MV3 manifest, service-worker entry, popup shell, build/test setup. **Depends on:**
  1.5.

### 3.2 Dual-OAuth auth spike
- **Deliverable:** **GitHub App device flow** (user-to-server tokens scoped to installed repos, not a
  broad `repo` scope) + Google PKCE in the service worker; token storage in `chrome.storage.session`;
  proven serverless (public client_id, `drive.file` scope). Public-repo path needs no install;
  private-repo path requires installing the App on the repo (documented, heavier).
- **Satisfies:** S1 (`auth.status`/`auth.login`), hard constraints #1–2; **SEC-1** (least-privilege
  GitHub token); **the web-user credential rule** — zero separate logins, zero API keys; identity is
  in-browser OAuth only.
- **Acceptance gate:** E2E manual smoke — log in to both providers with no backend and no pasted
  token; confirm the token is repo-scoped (cannot touch an unrelated repo). Unblocks all sync work.
  **Depends on:** 3.1.

### 3.3 S1 message API + error taxonomy
- **Deliverable:** `chrome.runtime` request/response layer with the full command set + stable error
  taxonomy (`AUTH_REQUIRED`, `NOT_BOUND`, `DIVERGED_CONFLICT`, …, `TIER3_PRESENT`).
- **Satisfies:** S1. **Acceptance gate:** contract — every command returns
  `{ok,data}`/`{ok:false,error}`; error codes asserted. Unit. **Depends on:** 3.2.

### 3.4 Converter: Docs JSON → Canonical Markdown (Push)
- **Deliverable:** JS converter emitting the exact profile (frontmatter order, ATX headings,
  blank-line rules, Tier-1 tables, `assets/<sha256>` images, escaping, LF + single trailing newline).
- **Satisfies:** S4 (producer), [`repo-layout.md`](repo-layout.md). **Acceptance gate:** **INV-7** via
  1.4; **INV-2** (outline isomorphism); **INV-6** (asset path stability). Unit + contract.
  **Depends on:** 3.1, 1.1, 1.4.

### 3.5 Converter: Canonical Markdown → Docs `batchUpdate` (Pull)
- **Deliverable:** JS emitter of Docs `batchUpdate` requests from canonical Markdown. **Satisfies:**
  S4 (`md→docs→md == md`). **Acceptance gate:** **INV-1** round-trip no-diff at the converter level;
  expected-requests fixtures match. Unit + contract. **Depends on:** 3.4.

---

## WBS 4 — N=1 Pull/Push: the MVP seam (build step 4)

One untabbed Doc ⇄ one `.md` + snapshot + checkpoint + conflict refusal. Proves the seam end to end
with **no tree-walking** — the integration milestone where INV-1/3/4 first hold over the real flow.

| WBS | Deliverable | Satisfies | Acceptance gate | Depends on |
|-----|-------------|-----------|-----------------|------------|
| 4.1 | Drive binding + branch-folder layout — **one canonical folder** under `Crayon/<owner>/<repo>/<branch>/` (account or Shared Drive), collaborators as editors; **one-time Picker "adopt" per Doc** for files another user created (`drive.file`) | S3 | integration vs FakeDrive; adopt path covered | 3.2, 1.3 |
| 4.2 | Checkpoint read/write — `DeveloperMetadata` per `checkpoint.schema.json`; `lastSyncContentHash` taken over the Doc's **re-export** (settling), tabbed-Doc hash = ordered tab concat | S3, S5 | contract + integration | 4.1, 1.2 |
| 4.3 | Divergence state machine — `status.get` → `in-sync\|local-ahead\|remote-ahead\|diverged` | S1 | unit over the truth table | 4.2 |
| 4.4 | Pull (N=1) — read HEAD, convert via 3.5, write Doc, **settle** (re-export → hash → at most one settling commit), stamp checkpoint, write snapshot | S2 (read), S3 | **INV-1** settling fixed point; **INV-3** (Pull then Pull = no Drive change); **UAT-A1**. integration vs fakes | 3.5, 4.2, 4.3 |
| 4.5 | Push (N=1, atomic) — export via 3.4, one commit via Git Data API (blobs→tree→commit→update-ref, expected-SHA), replace snapshot, optional PR | S2 | **INV-1** (unedited Pull→Push = empty diff); **UAT-A2**. integration vs fakes | 3.4, 4.4 |
| 4.6 | Conflict refusal — both sides moved → `DIVERGED_CONFLICT`, **zero writes** | S1, S2 | **INV-4**; **UAT-A3** / **UAT-D2**. integration vs fakes | 4.5 |
| 4.7 | Minimal popup + status chip — Pull/Push buttons, branch/repo display, docs.google.com chip | S1, content script | E2E manual smoke (SPEC.md step-4 verification) | 4.4, 4.5, 4.6 |
| 4.8 | CI status green/red — `checks.get` reads the GitHub Checks API; popup + chip render ✓ green / ✗ red / ◦ pending for the branch PR | S1 (`checks.get`), S2 (read) | **UAT-A5** (color matches GitHub Checks). integration vs FakeGitHub | 3.3, 4.7 |

---

## WBS 5+ — Later build steps (lower resolution; expand when reached)

Each expands to full deliverable/acceptance packages when its predecessor is green.

- **5 — Tree reconciliation (step 5):** multiple untabbed Docs + sub-folders; idempotent Pull; atomic
  multi-file Push; manifest generation. Gate: **INV-3** at tree scale, **INV-1** multi-file. Depends
  on WBS 4.
- **6 — Figures + tables Tier 1+2 (step 6):** images → content-hash `assets/` re-inserted via Drive
  staging; rectangular tables ⇄ canonical GFM with alignment; Tier-3 lint/warn chip. Gate: **INV-6**,
  **UAT-A4** (`TIER3_PRESENT` warn). Depends on WBS 3.4/3.5, 5.
- **7 — Branch lifecycle (step 7):** create/switch/delete with recursive folder copy/trash mirroring.
  Gate: **INV-5** (branch ⇄ exactly one folder; delete mirrors). Depends on WBS 5.
- **8 — Tabbed Docs (step 8):** opt-in `.crayon-tabs.json`; read/write existing tabs by `tabId` (no
  tab CRUD). Gate: `TAB_STRUCTURE_REQUIRED` path; tabbed round-trip. Depends on WBS 5.
- **9 — PR + navigation glue (step 9):** Open-PR flow, github.com ⇄ Docs navigation links; finalize
  CLI governance (2.5–2.8 land here if not earlier). Gate: **UAT-B1/B3**, **UAT-D1** cross-UI
  converge. Depends on WBS 2, 4, 7.

---

## WBS S — Security & supply chain (continuous gate + external review)

Cross-cutting. **Continuous release-gate sweep in CI, plus a third-party/store security review before
any public listing.** Traces to SEC-1…n (WBS 0.1). "Nothing malicious reaches the user's browser" is
the headline guarantee.

- **S.1 MV3 least-privilege + CSP audit** — minimal `permissions`/`host_permissions` (only
  `docs.google.com`, `github.com`, the API origins); strict CSP; **no remote code / no `eval`** (MV3
  forbids it — verified, not assumed). Gate: automated manifest+CSP lint blocks the build. Depends on:
  3.1.
- **S.2 Supply-chain integrity** — pinned lockfiles (`package-lock`/`uv.lock`), dependency audit
  (`npm audit`/`pip-audit`), no surprise `postinstall`, vendored/reviewed bundle, release provenance.
  Gate: CI fails on advisories above threshold or lockfile drift. Depends on: 1.5.
- **S.3 Token & data-handling review** — tokens only in `chrome.storage.session`; `drive.file`
  least-scope; no token logging/exfiltration; secrets never in the bundle or repo. Gate: static check
  + per-release review checklist. Depends on: 3.2, 1.5.
- **S.4 Content-script injection safety** — the `docs.google.com`/`github.com` scripts have no XSS
  sinks, no untrusted-HTML injection, no exfiltration; the message API validates origins. Gate: SAST +
  targeted tests on the injection surface. Depends on: 3.3, 4.7.
- **S.5 Continuous gate wiring** — fold S.1–S.4 into CI as a required check; reuse the repo's
  `security-review` skill on each PR touching the extension. Gate: the sweep is a required status
  check (parallel to `crayon-check`). Depends on: S.1–S.4, 2.8.
- **S.6 External / store security review** — third-party (or Chrome Web Store) review of OAuth-scope
  justification + bundle before public listing. Gate: external sign-off is a prerequisite for O.5.
  Depends on: S.5.
- **S.7 Rule/sink trust model + screening (constitutive surface)** — the default Action templates
  invoke only `crayon check` / `crayon publish` (no arbitrary `run:` steps); PR checks use
  `pull_request` + least-privilege `GITHUB_TOKEN` + no secrets; publish secrets are post-merge only;
  ship [`docs/screening-guide.md`](screening-guide.md) as the reviewer checklist for `.crayon/checks/`,
  `.crayon/sinks/`, configs, and workflows. Gate: **SEC-7/8/9** — a fork-PR custom rule runs with no
  secrets and cannot escape the ABC templates; the screening checklist catches a planted
  `run:`/network/secret-read. Depends on: 2.2, 2.6, 2.8.

---

## WBS O — Onboarding, install & distribution

Cross-cutting; realizes UI-0 / W0 / UAT-E (WBS 0.1). **GitHub Pages doc site; both install paths,
gated by review** (dev-install now, Web Store later).

- **O.1 GitHub Pages doc site (autogenerated)** — a `docs/`-sourced Pages build: getting-started,
  install guide, the one blessed-workflow walkthrough, generated from the repo's existing Markdown
  (SPEC/repo-layout + a new quickstart) so it can't drift. Gate: Pages builds in CI; links resolve.
  Depends on: 0.1.
- **O.2 Dev-install path + one-page guide** — unpacked/`.crx` artifact + a crisp install page; reach
  onboarding without blocking on store review. Gate: a fresh profile installs and reaches first OAuth
  by following only the guide. Depends on: 3.1, O.1.
- **O.3 Extension first-run onboarding UX** — post-install flow: prompt dual-OAuth (in-browser, no
  keys) → bind first repo → **one-time Picker "adopt"** when joining an existing shared folder → first
  Pull, with the status chip explaining state. Gate: **UAT-E** (web variant) end-to-end incl. a
  second collaborator adopting; honors the web-user-no-keys rule. Depends on: 3.2, 4.1, 4.7, O.2.
- **O.4 CLI onboarding/quickstart** — `uv`/`pipx` install instructions + `crayon init` quickstart;
  `.env.example` walkthrough for the IDE user. Gate: **UAT-E** (IDE variant) — clone-to-`check` with
  only the gitignored `.env`. Depends on: 2.5, 1.5, O.1.
- **O.5 Chrome Web Store listing (one-click, review-gated)** — store listing, privacy disclosures,
  OAuth-scope justification. Gate: published listing; **prerequisite = S.6 external review green**.
  Depends on: O.3, S.6.
- **O.6 Policy-author worked example** — `docs/policy-authoring.md` on the Pages site: enable a starter
  rule in `rules.yaml` (low-code), then copy a starter class into `.crayon/checks/` to author a custom
  rule; run `crayon check`; wire the status into branch protection via the GitHub web UI. Realizes
  **W6**. Gate: the example's commands reproduce **UAT-C2** verbatim. Depends on: 2.2, O.1.
- **O.7 Publication recipes** — `docs/recipes/` example GitHub Actions for post-merge `crayon publish`
  (read-only publish; push to an external API, e.g. the Substack case), each **dependency-free** and a
  copy-the-pattern custom `Sink`. Realizes **W7**. Gate: a recipe runs `crayon publish` against an
  example sink in CI; scope note states targets are examples, not Crayon deps. Depends on: 2.6, O.1.

---

## Critical path, parallelism & coverage

- **Critical path:** WBS 0 → 1.1/1.3/1.4 → 3.4 (converter) → 3.5 → 4.4 → 4.5 → 4.6. The converter
  (S4) and the N=1 Push are the load-bearing risk.
- **Parallelizable:** WBS 2 (CLI) runs alongside WBS 3 once 1.1/1.2/1.5 exist — different language,
  different surface, joined only at INV-7 via 1.4. The auth spike (3.2) runs concurrently with the
  converter (3.4) — they don't touch.
- **Invariant ownership:** INV-1 → 3.5/4.5 · INV-2 → 2.2/3.4 · INV-3 → 4.4 · INV-4 → 4.6 · INV-5 →
  WBS 7 · INV-6 → 3.4/WBS 6 · INV-7 → 1.4/2.4/3.4.
- **UAT / SEC ownership:** UAT-A1/2/3 → 4.4/4.5/4.6 · **A5 (green/red CI) → 4.8** · UAT-B → 2.2/2.8/9 ·
  **C2 (author a rule) → 2.2/O.6** · **C3 (route content out) → 2.6/O.7** · C4 (interop) → WBS 5/9 ·
  UAT-D → 4.6/9 · **UAT-E → O.3/O.4** · **SEC-1…6 → WBS S.1–S.6** · **SEC-7/8/9 (rule/sink trust) →
  S.7**. Every invariant, UAT case, and security criterion has an owning package.
- **Governance modes:** the WBS mirrors the spec's operative/constitutive split — the fast operative
  path is WBS 3–4 (+5–8); the slow, screened constitutive surface (rules, sinks, workflows) is WBS 2.2
  / 2.6 / 2.8 with its trust model in S.7. Repo-supplied code executes only in the constitutive layer.
- **Symmetry note:** the rule engine (2.2, "check this") and the sink engine (2.6, "write this there")
  share one ABC→kinds→YAML-materialization→custom-subclass pattern, so 2.6 reuses 2.2's machinery and
  the worked examples (O.6/O.7) teach a single pattern twice.
- **Launch gate:** public listing (O.5) cannot proceed until the security sweep is a green required
  check (S.5) and external review signs off (S.6) — onboarding and security are hard prerequisites,
  not afterthoughts.
