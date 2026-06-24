# Crayon Chrome extension (placeholder — not yet implemented)

Manifest V3, Node/JS. This directory will hold the browser-side "git client". Per
[`../SPEC.md`](../SPEC.md) §Components, it comprises:

- **Service worker** — owns the GitHub App device flow (repo-scoped user-to-server tokens) + Google
  PKCE OAuth, token storage (`chrome.storage.session`), and all calls to `api.github.com` / Docs /
  Drive. Implements the S1 message API and the S2/S3/S4 contracts.
- **Popup UI** — the git controls: Pull, Push, Open PR, New/Delete branch, Open on GitHub, plus the
  branch's **green/red CI status** (✓/✗/◦ via `checks.get`).
- **Content scripts** — thin status chip on `docs.google.com` (branch, in-sync/diverged, green/red CI);
  navigation glue on `github.com`.
- **Converter** — Docs JSON ⇄ Canonical Markdown (the profile in
  [`../docs/repo-layout.md`](../docs/repo-layout.md)).

Nothing here is built yet. Build order and the TDD plan are in SPEC.md §"Suggested build sequence"
and §"Test strategy (TDD)".
