---
updated_at: 2026-03-15T15:50:00Z
focus_area: Post-sprint — SDK work + remaining bugs
version: v0.8.25-build.4
branch: dev
tests_passing: ~4157
tests_todo: 46
tests_skipped: 5
test_files: 154
team_size: 19 active agents + Scribe + Ralph + @copilot
team_identity: Apollo 13 / NASA Mission Control
process: All work through PRs. Branch naming squad/{issue-number}-{slug}. Never commit to main directly.
---

# What We're Focused On

**Status:** Irritating bugs sprint COMPLETE. 7 issues closed, 8 PRs merged to dev, 5 community PRs merged. CI lockfile fix resolved 8-day build failure. Reskill needed, then dive into SDK work and remaining bugs.

## Current State

**Version:** v0.8.25-build.4 (on dev, not yet released)
- **Packages:** @bradygaster/squad-sdk, @bradygaster/squad-cli
- **Branch:** dev
- **Build:** ✅ clean (0 errors, CI green)
- **Tests:** ~4,157 passed, 46 todo, 5 skipped, ~154 test files
  - Only failure: aspire-integration.test.ts (needs Docker, pre-existing)
  - New: docs-links.test.ts (internal link + anchor validation from diberry)

**Stack:**
- TypeScript (strict mode, ESM-only)
- Node.js ≥20
- @github/copilot-sdk
- Vitest (test runner)
- esbuild (bundler)

**Team:** Apollo 13 / NASA Mission Control
- 19 active agents + Scribe + Ralph + @copilot

## What Just Shipped (Irritating Bugs Sprint — 2026-03-15)

### Bug Fixes (our PRs)
- **PR #409** — Version stamp in agent charter.md (#321)
- **PR #411** — SDK init trio: Ralph in init (#338), config sync after cast (#337), @copilot removed from routing templates (#339)
- **PR #412** — Base roles opt-in via `--roles` flag (#379) — @spboyer pinged
- **PR #414** — CI lockfile fix (stale nested SDK entry caused 8-day build failure)

### Community PRs (Tamir Dresher)
- **PR #415** — Rework rate OTEL metrics in squad-sdk (5th DORA metric)
- **PR #381** — Rework rate CLI command (`squad rework`) — cherry-picked to dev

### Community PRs (Dina Berry)
- **PR #389** — Docs consolidation & reduction (-1,471 net lines, closes #258/#351)
- **PR #393** — baseBranch alignment to dev (closes #350)
- **PR #396** — Docs quality CI (markdownlint, cspell, link validation)

### Issues Closed
#321, #337, #338, #339, #348, #356, #379, #258, #350, #351

## Next Up (Post-Reskill)

### Remaining Sprint Bugs
- **#342** — CastingEngine bypass (casting doesn't use the engine)
- **#363** — WSL transient error handling
- **#340** — SDK feature parity audit

### SDK Work
- SDK builder improvements
- Feature parity between CLI and SDK paths

### Upgrade Notes for PAO
- Captured in session files — PAO needs to review and update docs for:
  - `--roles` flag for `squad init`
  - Version stamp in charters
  - Ralph auto-inclusion in init
  - Config sync after casting (SDK users)

## Process

All work through PRs. Branch naming: `squad/{issue-number}-{slug}`. Never commit to main directly. Squad member review before merge. Always use bradygaster (personal) GitHub account for this repo.
