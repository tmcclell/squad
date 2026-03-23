# SURGEON

> Flight Surgeon

## Learnings

### Release History
v0.8.24 released successfully. npm packages: @bradygaster/squad-sdk@0.8.24, @bradygaster/squad-cli@0.8.24. publish.yml triggers on `release: published` (NOT draft). Test baseline at release: 3,931 tests, 149 files.

### Version Mutation Bug (P0)
bump-build.mjs mutates versions during local builds despite SKIP_BUILD_BUMP=1 and CI=true env vars. Workaround: set versions with `node -e` script and commit IMMEDIATELY before building. This is a P0 fix item in docs/proposals/cicd-gitops-prd.md.

### Known Incidents
v0.8.22: 4-part version 0.8.21.4 mangled by npm to 0.8.2-1.4. v0.8.23: versions reverted from 0.8.23 to 0.8.22 during build despite env vars. Both resolved with the node -e script + immediate commit workaround.

### v0.9.0 CHANGELOG Written
**Date:** 2026-03-23

v0.9.0 is a MAJOR minor version bump (0.8.25 → 0.9.0) justified by 40+ commits spanning 6+ major features and governance-layer additions. CHANGELOG organized across 12 feature sections + fixes:

**Feature Categories:**
- Personal Squad (governance layer + ambient discovery)
- Worktree Spawning & orchestration
- Machine Capability Discovery
- Cooperative Rate Limiting & circuit breaker
- Economy Mode (cost-conscious model selection)
- Auto-wire Telemetry
- Issue Lifecycle & KEDA templates
- Session Recovery skill
- GAP analysis verification loop
- GitHub Auth Isolation skill
- Astro docs site improvements (10 items)
- Skill migrations (.squad → .copilot)
- ESLint runtime anti-pattern detection

**Fixes:**
- CLI terminal rendering (scroll flicker, Ink remounting)
- Upgrade path & installation (P0 gaps, EPERM handling, template alignment)
- ESM compatibility (Node 22/24 vscode-jsonrpc fixes)
- Runtime stability (signal handling, race conditions, timeouts, memory safety)
- GitHub integration (CI hardening, casting alignment)

**Documentation:** Maintained strict format rules (no npx refs, no "agency" terminology, matched existing CHANGELOG style, grouped by feature subsections). CLI Terminal Rendering fixes kept from [Unreleased] as part of 0.9.0.
