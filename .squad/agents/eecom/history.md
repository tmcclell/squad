# EECOM

> Environmental, Electrical, and Consumables Manager

## Learnings

### CLI Entry Point Architecture
cli-entry.ts is the central router for ~30+ CLI commands using dynamic imports (lazy-loading). Commands are routed via if-else blocks. Has a recurring "unwired command" bug class — implementations exist in cli/commands/ but aren't routed in cli-entry.ts. The cli-command-wiring.test.ts regression test catches this by verifying every .ts file in cli/commands/ is imported.

### ESM Runtime Patch
Module._resolveFilename interceptor in cli-entry.ts (lines 47-54) patches broken ESM import in @github/copilot-sdk@0.1.32 (vscode-jsonrpc/node missing .js extension). Required for Node 24+ strict ESM enforcement. Works on npx cache hits where postinstall scripts don't run.

### Lazy Import Pattern
All command imports use `await import('./cli/commands/xxx.js')` to minimize startup time. Copilot SDK is lazily loaded only when shell is invoked. All .js extensions required for Node 24+ strict ESM.

### CLI Packaging & Distribution
`npm pack` produces a complete, installable tarball (~275KB packed, 1.2MB unpacked). Package includes dist/, templates/, scripts/, README.md per package.json "files" field. Postinstall script (patch-esm-imports.mjs) patches @github/copilot-sdk for Node 24+ compatibility. Tarball can be installed locally (`npm install ./tarball.tgz`) and commands execute via `node node_modules/@bradygaster/squad-cli/dist/cli-entry.js`. Both squad-cli and squad-sdk must be installed together — cli depends on sdk with "*" version specifier. All 27+ CLI commands are lazy-loaded at runtime; `--help` validates command routing without executing full logic.

### Packaging Smoke Test Strategy
test/cli-packaging-smoke.test.ts validates the packaged artifact (not source). Uses npm pack + install in temp dir + command routing verification. Commands are expected to fail (no .squad/ dir) — test verifies routing only (no "Unknown command", no MODULE_NOT_FOUND for the command itself). Exception: node-pty is an optional dependency for the `start` command and MODULE_NOT_FOUND for node-pty is allowed. Windows cleanup requires retry logic due to EBUSY errors — use rmSync with maxRetries + retryDelay options, wrap in try/catch to fail silently since tests have passed.

### v0.8.24 Release Readiness Audit
CLI completeness audit (2026-03-08) confirmed: 26 primary commands routed in cli-entry.ts, all present in smoke test. 4 aliases (watch→triage, workstreams→subsquads, remote-control→rc, streams→subsquads). 3 aliases tested, 1 untested ("streams"). Packaging verified: dist/, templates/, scripts/, README.md in tarball; bin entry points to dist/cli-entry.js; postinstall script included and working. All 32 smoke tests pass. Package.json files array correct. npm pack output shows 318 files, 275KB packed. No missing command implementations. Optional dep (node-pty) handled correctly. Only gap: "streams" alias not in smoke test (routed correctly but test coverage incomplete). Confidence: 95% — all critical paths covered, minor alias test gap non-blocking.

📌 **Team update (2026-03-08T21:18:00Z):** FIDO + EECOM released unanimous GO verdict for v0.8.24. Smoke test approved as release gate. FIDO confirmed 32/32 pass + publish.yml wired correctly. EECOM confirmed 26/26 commands + packaging complete (minor gap: "streams" alias untested, non-blocking).

### SDK Init Flow Deep Dive (2026-03-08)
Traced complete `squad init --sdk` flow end-to-end for unified PRD. Key findings: (1) Init flow has two phases: CLI init creates skeleton files, REPL auto-cast creates team members. (2) Critical gap: squad.config.ts is never updated after auto-cast — members exist in .squad/ but not in config. (3) Ralph is inconsistently created (auto-cast yes, CLI init no). (4) No commands exist for adding/removing members post-init. (5) CastingEngine class exists but is never called during init — LLM-based Init Mode prompt is used instead. Roadmap written to .squad/identity/sdk-init-implementation-roadmap.md with 7 fixes prioritized by dependency graph. Critical path: sync utility → Ralph fixes → CastingEngine integration → hire/remove commands. High-risk items: squad.config.ts AST parsing (considered regex alternative). Open questions: AST vs regex for config sync, CastingEngine augment vs replace LLM, Ralph always-on vs opt-in.

📌 **Team update (2026-03-11T01:25:00Z):** SDK Init decisions finalized: Phase-based quality improvement program, CastingEngine canonical casting, squad.config.ts as source of truth, Ralph always-included, implementation priority order (sync utility first, then Ralph fixes, then CastingEngine integration). All decisions merged to decisions.md. Ready to start Phase 1 implementation.

### Adoption Tracking Tier 1 Implementation (2026-03-10)
Implemented Flight's privacy-first adoption monitoring strategy on PR #326 branch. Moved `.squad/adoption/` → `.github/adoption/` for better GitHub integration. Stripped tracking.md to aggregate-only metrics (removed all individual repo names/URLs). Updated GitHub Action workflow (adoption-report.yml) and monitoring script (scripts/adoption-monitor.mjs) to write reports to `.github/adoption/reports/`. Removed "Built with Squad" showcase link from README.md (deferred to Tier 2 opt-in feature). This honors the principle: collect aggregate metrics via public APIs, but never publish individual repo lists without explicit consent. Test discipline: verified npm run build passes; docs-build.test.ts passed structure tests (Astro build failure unrelated to changes). Committed with clear message explaining privacy rationale.

📌 **Team update (2026-03-10T12-55-49Z):** Adoption tracking Tier 1 complete and merged to decisions.md. Privacy-first architecture confirmed: aggregate metrics only, opt-in for individual repos, public showcase only when 5+ projects opt in. Append-only file governance enforced (no deletions in history.md or decisions.md). Microsoft ampersand style guide adopted for documentation.

### Issue Triage (2026-03-22T06:44:01Z)

**Flight triaged 6 unlabeled issues and filed 1 new issue.**

EECOM assigned:
- **#481 (StorageProvider PRD)** → squad:control + squad:eecom (type system abstraction + runtime integration)
- **#479 (history-shadow race condition)** → squad:eecom + squad:retro (production bug; mitigation through StorageProvider atomicity)

Pattern: Three architectural gaps identified (agent spec, state abstraction, quality tooling) + one production bug. StorageProvider abstraction critical for #479 atomicity fix.

📌 **Team update (2026-03-22T06:44:01Z):** Flight issued comprehensive triage. EECOM owns StorageProvider PRD spec (#481) + history-shadow race mitigation (#479). Ready to begin implementation on next sprint.

### PR #483 Review & Merge — Platform Adapter Timeout Fix (2026-03-22)

Reviewed and merged diberry's fix for platform-adapter test timeouts that were blocking all 8 open PRs. Root cause: `getAvailableWorkItemTypes()` called `execFileSync('az', ...)` with no timeout — in CI where az CLI is installed but no real ADO org exists, it hangs indefinitely until Vitest kills it at 5s.

**Fix pattern:** `{ ...EXEC_OPTS, timeout: 3_000 }` — spread existing exec options and add a 3-second timeout. The existing catch block already returns sensible default work item types, so timeout errors fall through gracefully. This is the correct pattern for any external CLI call that might hang: add a timeout to execFileSync and ensure the catch block has a fallback.

**Rebase note:** Branch was already clean on top of dev (1 commit ahead, no divergence). No rebase was needed.

📌 **Team update:** PR #483 merged (squash). This unblocks CI for all open PRs that were failing on platform-adapter test timeouts. The remaining CI failure across PRs is the broken docs link (separate issue).

### PR #480 Review & Merge — History Race Condition Fix (2026-03-22)

Reviewed and merged PR #480 (async mutex + atomic writes + 14 tests). Addresses race condition in history-shadow file operations under concurrent load.

**Fix pattern:** Race conditions in history operations require three-layer defense: (1) async mutex for write serialization, (2) atomic file operations (write-then-rename), (3) comprehensive test coverage (14 tests for edge cases). This pattern applies to any persistent state under concurrent access.

**Key learning:** File system race conditions aren't just "add a lock" — need atomicity guarantees (rename is atomic), serialization (mutex), and exhaustive test coverage to validate edge cases (concurrent writes, stale reads, partial failures).

### PR #486 Review & Merge — SIGINT Handling (2026-03-22)

Reviewed and merged PR #486 (two-layer signal handling + 22 tests). Improves graceful shutdown under SIGINT (Ctrl+C) by cleaning up both parent and child processes.

**Fix pattern:** Signal handling in Node.js requires two layers: (1) parent process SIGINT handler that triggers graceful shutdown, (2) child process cleanup (kill child processes, close file handles, flush buffers). Incomplete cleanup leaves zombie processes or orphaned file locks. Test coverage essential: 22 tests verify process tree cleanup, signal propagation, and edge cases (nested children, immediate re-signals).

**Key learning:** SIGINT handling is more complex than "add a signal handler" — need explicit child process cleanup logic + comprehensive tests. Pattern applies to any process spawning child processes (CLI spawning subshells, REPL spawning child REPL instances, etc.).
