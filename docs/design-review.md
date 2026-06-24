# Design review — decision log

A pre-implementation design review of the v1 contracts (SPEC.md, the schemas, repo-layout.md) and the
work breakdown (docs/wbs.md), conducted 2026-06-23. Each issue was flagged with a severity and
remediation options; the maintainer chose a resolution. This log is the source of record for the
`(DR-n)` references elsewhere in the docs.

**Severity:** 🔴 Critical (threatens a headline guarantee) · 🟠 High (a contract likely won't hold as
written) · 🟡 Medium (real gap, localized) · ⚪ Low (clarity/hygiene). All items are **resolved**.

| ID | Sev | Issue | Resolution | Landed in |
|----|-----|-------|------------|-----------|
| **DR-1** | 🔴 | `drive.file` can't deliver a *shared* multiplayer Doc across users (per-app, per-user file access). | A branch's Drive folder is **one canonical location** (service identity / repo owner), collaborators added per-branch-role (**commenter on `main`, editor on branches** — see the read-only-mirror addition below); each does a one-time Picker **"adopt"** per Doc. | S3, INV-5, constraint #2, W0/W3, open-risk #6 |
| **DR-2** | 🔴 | INV-1 (no-diff round-trip) depends on Google Docs being byte-stable on re-export. | **Settling rule**: checkpoint hash taken over the Doc's *re-export*; first Pull may emit one settling commit; INV-1 restated as **idempotent-after-settle**. | INV-1, constraint #4, repo-layout §"Round-trip & settling" |
| **DR-3** | 🟠 | `crayon check` executes untrusted PR-supplied Python in CI. | **Trust model**: automation constrained to the `Rule`/`Sink` ABCs (SEC-7); least-privilege PR CI, no secrets (SEC-8); reviewed-as-code (SEC-9); **operative vs constitutive** framing; screening guide. | SEC-7/8/9, "Trust model" §, operative/constitutive §, [screening-guide.md](screening-guide.md) |
| **DR-4** | 🟠 | Device Flow gives the extension a broad `repo`-scoped token on a public client. | **GitHub App** device flow → user-to-server tokens scoped to installed repos. Public repos need no install (encouraged); private repos require installing the App. | constraint #1, S2, service-worker bullet, open-risk #7 |
| **DR-5** | 🟠 | Sinks POST repo content to arbitrary URLs from CI (exfiltration/SSRF). | Folded into the DR-3 trust model: publish secrets scoped to the post-merge job; `publish.yaml`/`sinks/` reviewed as code (SEC-8/9). | SEC-8/9, "Trust model" § |
| **DR-6** | 🟡 | Per-commit raw Docs-JSON snapshots bloat history and bury the reviewable diff. | `crayon init` writes a `.gitattributes` marking `.crayon/snapshots/**` `linguist-generated -diff` — kept for restore, collapsed in PR review. | S5/commit-contains, repo-layout, `init`, WBS 2.5/4.5 |
| **DR-7** | 🟡 | Pull may clobber un-Pushed local Doc edits. | New **INV-8 Pull safety**: refuse/confirm on `local-ahead`/`diverged`. | INV-8, Pull step 0, WBS 4.4 |
| **DR-8** | 🟡 | First import of pre-existing Markdown (no docIds) is underspecified. | Explicit **first-import** step (mint docIds, write back as a settling commit) + **UAT-E4**. | Pull step 1, UAT-E4, WBS 4.4/O.3 |
| **DR-9** | 🟡 | Branch-create copies Docs → new docIds; manifest/frontmatter/checkpoint rewrite is non-trivial. | Documented as an explicit **transactional rewrite** step (one Drive call per Doc; rate-limit risk). | Branch-lifecycle §, WBS 7 |
| **DR-10** | 🟡 | No determinism contract on custom rules/sinks. | S6 requires custom rules be **pure/deterministic/network-free**; run-twice-and-diff spot check + screening guide. | S6, WBS 2.2 |
| **DR-11** | 🟡 | `lastSyncContentHash` domain unspecified. | Pinned to **`sha256:<hex>` over the re-export** (tabbed Doc = ordered tab concat). | checkpoint.schema.json, repo-layout |
| **DR-12** | ⚪ | "No third-party deps" contradicts PyYAML needed to parse the YAML configs. | Reworded: **no third-party deps *per rule/sink/destination***; PyYAML/PyGithub are base CLI deps. | scope line, cli/README |
| **DR-13** | ⚪ | INV-4 "zero remote writes" is literally false (orphan blobs created before ref update). | Reworded to **"zero branch/ref mutations"** (orphan objects gc'd). | INV-4, S1 invariant, W5 diagram |
| **DR-14** | ⚪ | `severity: warn` × exit code × green/red unspecified. | Specified: `error` → non-zero / **red**; `warn` → exit 0 / **green**; only `error` gates a merge. | S6, WBS 2.2 |

## Post-review design addition: the read-only `main` mirror
The Google-side counterpart to branch protection, resolving what "protection" means on Drive:
- **The default-branch (`main`) Drive mirror is a read-only receiver, not an authoring surface**
  (**INV-9**): shared **commenter** (read/comment/suggest, no edit); **Push from `main` is refused**;
  it is overwritten by canon, never the reverse.
- **Refresh — both** (maintainer's choice): a **post-merge Action + Google service identity** when
  configured (Drive sink / automated Pull, secret post-merge-scoped per SEC-8), **else triggered
  Pull** of `main`.
- **Read-only enforcement — Drive permissions** (commenter), which also enables the review surface.
- **Capturing review — "branch from recommended changes":** reviewers leave Google Docs *suggestions*
  on `main`; Crayon **materializes** them onto a new branch via
  `documents.get?suggestionsViewMode=PREVIEW_SUGGESTIONS_ACCEPTED` (constraint #8; accept-all in v1) —
  the suggested diff becomes a real git diff, push + PR. Free-form comments stay on the mirror (Tier-3,
  not versioned in v1); per-suggestion selection and comment→GitHub capture are deferred seams.

Landed in: core model, "The read-only `main` mirror" §, S3 (per-branch-role perms), Pull/Push +
Branch-lifecycle, constraint #8, INV-9, W8, UAT-A6/UAT-B4, scope; WBS 4.1/4.9, 7, 9, coverage.

**Follow-on hazard — mirror refresh destroys review state (resolved, INV-10).** The refresh overwrites
the mirror, wiping comments/suggestions that live only there, and `main` is never Pushed — so naive
triggering both *loses* the review state and can *loop* (a capture that advances `main` re-fires the
post-merge Action → another wipe). Resolved with an explicit **capture-before-overwrite protocol**
(MR-1..MR-7) + **INV-10**: capture each Doc's review state (Docs JSON + suggestions + Drive comments)
to a discoverable **git tag** `crayon-review/<sha>` **before** any overwrite; a projected-SHA
idempotence guard (re-run = no-op, no second wipe); a tag is not a branch and the Action excludes tags,
so the capture can't trigger a refresh; serialized triggers; owner-executed; and a **mandatory
time-bounded lock** (auto-expiring, length configurable with a small floor) closing the
capture→overwrite race. Distinct from the deferred *structured* comment→GitHub capture — MR guarantees
nothing is silently lost, with explicit tests T-MR-1..T-MR-7.
Landed in: "Mirror refresh: capture-before-overwrite" §, INV-10, UAT-B5, Test strategy (T-MR-*),
scope; WBS 9m, coverage.

## Scope added during this review
- **Citable public example (WBS O.8):** a public GitHub repo (`crayon-example`, lipsum content, full
  `.crayon/` layout, a sample custom rule) + a public view-only Drive folder mirroring it, linkable
  from the doc site and README as the canonical worked example.

## Process note
This review followed the Intent-Based-Coding discipline: the spec/contracts were stress-tested and
amended **before** implementation, so the invariants (now INV-1…INV-10), UAT cases (UAT-A…E), and
security criteria (SEC-1…SEC-9) remain the executable acceptance boundary. Commits: `9291518`
(rules/sinks + onboarding/security), `9b55eed` (DR-1…DR-5), and the DR-6…DR-14 + consistency pass.
