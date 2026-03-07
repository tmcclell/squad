# Project Context

- **Owner:** Brady
- **Project:** squad-sdk — the programmable multi-agent runtime for GitHub Copilot (v1 replatform)
- **Stack:** TypeScript (strict mode, ESM-only), Node.js ≥20, @github/copilot-sdk, Vitest, esbuild
- **Created:** 2026-02-21

## Core Context — Drucker's Focus Areas

**CI/CD Engineer:** Drucker owns GitHub Actions workflows, automated validation gates (semver validation, token verification, draft detection), publish pipeline with retry logic, CI health monitoring, and incident response. Pattern: defense in depth — every workflow has validation gates that assume humans will make mistakes.

## Lessons from Kobayashi's Failures

Drucker was created to replace Kobayashi (along with Trejo) after the v0.8.22 release disaster. These CI-specific lessons are foundational to Drucker's approach:

### 📌 v0.8.22 Release Disaster — CI/CD Perspective

**CI FAILED TO CATCH MISTAKES.** Multiple validation gaps allowed invalid state to reach production.

**What went wrong (CI perspective):**
1. **No semver validation gate:** 4-part version (0.8.21.4) made it through CI to `npm publish`. npm mangled it to 0.8.2-1.4.
2. **No NPM_TOKEN type check:** CI failed 5+ times with EOTP errors before token was replaced. No pre-flight check to verify token type.
3. **No retry logic in verify steps:** npm registry propagation delay (5-30 seconds) caused false 404 failures even when publish succeeded.
4. **Workflow triggered on wrong event:** Release created as draft didn't emit `release: published` event, so workflow never ran automatically.

**Root cause (CI perspective):** Workflows assumed inputs were correct. No defensive validation. No retry logic for external dependencies.

**CI Lessons learned:**
1. **Validation gates are mandatory.** CI must validate semver format before `npm publish`. Use `npx semver {version}` and fail build if invalid.
2. **Verify token type before publishing.** Check NPM_TOKEN is an Automation token (not User token with 2FA). Fail fast with actionable error if wrong type.
3. **Retry logic for external dependencies.** ANY step that depends on npm registry, GitHub API, or other external services needs retry logic with backoff.
4. **Defensive programming.** Assume humans will make mistakes. Assume network will be slow. Assume inputs will be wrong. Catch early with clear errors.
5. **Actionable error messages.** Don't just say "failed" — say "To fix: create an Automation token at..." or "Semver validation failed: use X.Y.Z format".

**What we shipped (CI perspective):**
- Comprehensive release runbook: `.squad/skills/release-process/SKILL.md` includes Common Failure Modes section with CI remediation
- Two new agents: Trejo (Release Manager) owns release decisions, Drucker (CI/CD Engineer) owns workflow automation
- Action items for Drucker: implement semver validation gates, add retry logic to verify steps, document token requirements

**Never again.** CI is our safety net. It failed. We fixed it. We document it so it doesn't happen again.

## CI/CD Technical Context

### GitHub Actions Workflows

Squad uses several workflows:
- **publish.yml** — Publishes SDK and CLI to npm (triggered on `release: published`)
- **squad-release.yml** — Release automation (deprecated/legacy?)
- **squad-ci.yml** — Test suite on PR (all checks must pass)
- **squad-preview.yml** — Preview builds to insiders channel
- **squad-docs.yml** — Documentation deployment

**Key focus:** publish.yml is the most critical. It's the final gate before packages go live on npm.

### npm Registry Propagation Delay

**The 5-30 second problem:**
- After `npm publish` succeeds (exit code 0), the package is written to the registry
- But npm uses a CDN with eventual consistency
- Queries via `npm view` may return 404 for 5-30 seconds (sometimes up to 2 minutes)
- This is NORMAL and EXPECTED behavior
- Verify steps MUST have retry logic to handle this

**Retry pattern:**
```bash
MAX_ATTEMPTS=5
WAIT_SECONDS=15

for attempt in $(seq 1 $MAX_ATTEMPTS); do
  if npm view "$PACKAGE@$VERSION" version > /dev/null 2>&1; then
    echo "✅ Package verified"
    exit 0
  fi
  
  if [ $attempt -lt $MAX_ATTEMPTS ]; then
    echo "⏳ Waiting ${WAIT_SECONDS}s for propagation..."
    sleep $WAIT_SECONDS
  fi
done

echo "❌ Failed to verify after $MAX_ATTEMPTS attempts"
exit 1
```

### NPM_TOKEN Types

**Two token types on npmjs.com:**
1. **User tokens (legacy):** Tied to user account, require 2FA/OTP for publish operations
2. **Automation tokens:** No 2FA required, designed for CI/CD, read-write access

**For CI/CD:**
- MUST use Automation tokens
- User tokens will fail with EOTP error (OTP required but can't be provided interactively in CI)
- Create at: npmjs.com → Settings → Access Tokens → Generate New Token → select "Automation" type
- Store as GitHub secret: `NPM_TOKEN`

**Verification strategy:**
- Check token exists: `[ -z "${{ secrets.NPM_TOKEN }}" ]` fails build if missing
- Document requirement: Workflow comments and README must explain token type requirement
- (Future enhancement: pre-publish check that verifies token type via npm API)

### Semver Validation

**3-part semver (X.Y.Z) or prerelease (X.Y.Z-tag.N) ONLY:**
- 4-part versions (0.8.21.4) are NOT valid semver
- npm parser will mangle them (0.8.21.4 becomes 0.8.2-1.4)
- This breaks `latest` dist-tag and causes customer confusion

**CI validation:**
```bash
VERSION="${{ github.event.release.tag_name }}"
VERSION="${VERSION#v}" # Strip 'v' prefix

if ! npx semver "$VERSION" > /dev/null 2>&1; then
  echo "❌ Invalid semver: $VERSION"
  echo "Only 3-part versions (X.Y.Z) or prerelease (X.Y.Z-tag.N) are valid."
  exit 1
fi

echo "✅ Valid semver: $VERSION"
```

### Draft vs. Published Releases

**GitHub Release states:**
- **Draft:** Not visible to public, doesn't emit `release: published` event
- **Published:** Visible to public, emits `release: published` event (triggers workflows)

**Workflow trigger:**
```yaml
on:
  release:
    types: [published]  # NOT 'created' — only fires for published releases
```

**Problem:** If a release is created as draft, the workflow never triggers automatically.

**Detection (for workflow_dispatch fallback):**
```bash
IS_DRAFT=$(gh api repos/${{ github.repository }}/releases/${{ github.event.release.id }} --jq '.draft')

if [ "$IS_DRAFT" = "true" ]; then
  echo "❌ Release is DRAFT. Workflow requires published release."
  exit 1
fi
```

### SKIP_BUILD_BUMP Environment Variable

**bump-build.mjs behavior:**
- Runs during dev builds to increment build number (0.8.21 → 0.8.21.1 → 0.8.21.2)
- Useful for dev iteration but DISASTROUS during releases (creates 4-part versions)
- Respects `SKIP_BUILD_BUMP=1` env var to disable

**CI requirement:**
```yaml
env:
  SKIP_BUILD_BUMP: "1"

- name: Verify SKIP_BUILD_BUMP is set
  run: |
    if [ "$SKIP_BUILD_BUMP" != "1" ]; then
      echo "❌ SKIP_BUILD_BUMP must be set to 1 for release builds"
      exit 1
    fi
    echo "✅ SKIP_BUILD_BUMP is set"
```

## Learnings

### 2026-03-07: First CI/CD Audit (Post-v0.8.22)

**Context:** Brady requested a comprehensive audit of all CI/CD pipelines for a PRD to "make CI/CD suck less." This is my first task as Drucker.

**Key Findings:**

**Critical Issues (P0):**
1. **squad-release.yml is completely broken** — 9+ consecutive failures due to ES module syntax errors in test files. Tests use `require('node:test')` but package.json has `"type": "module"`. This is blocking ALL releases from main.
2. **No semver validation in publish.yml** — 4-part versions can still reach npm and get mangled (same root cause as v0.8.22 disaster). Need to add `npx semver` validation step.
3. **No SKIP_BUILD_BUMP enforcement** — publish.yml doesn't explicitly set or verify this env var, so bump-build.mjs could run during release builds.
4. **bump-build.mjs creates invalid semver** — For non-prerelease versions (0.8.22), it creates 4-part versions (0.8.22.1) which are NOT valid semver. Should use `-build.N` suffix instead (0.8.22-build.1).

**Other Issues (P1/P2):**
- squad-ci.yml has flaky tests (human-journeys.test.ts, 12 failures)
- No dry-run step in publish.yml (`npm publish --dry-run`)
- No NPM_TOKEN existence check
- squad-insider-publish.yml has same validation gaps as publish.yml
- squad-publish.yml and squad-publish.yml.deprecated are redundant/stale
- squad-heartbeat.yml cron is disabled (Ralph not running on schedule)

**Positive Findings:**
- Retry logic in publish.yml works well (5 attempts, 15s intervals)
- Version matching validation exists (package.json vs. target)
- Provenance flag enabled (good supply chain security)
- Squad automation workflows (triage, labels, assignment) are working well
- squad-promote.yml has good design (strips team files before release)

**Pattern Recognition:**
- **Test failures are the primary blocker** — Most workflow failures are due to broken tests, not infrastructure issues
- **Defense in depth is missing** — Workflows assume inputs are correct, no validation gates
- **Redundant workflows cause confusion** — squad-publish.yml vs. publish.yml need clarification

**Technical Learnings:**
1. **ES module vs. CommonJS matters in CI** — package.json `"type": "module"` affects ALL .js files, including tests. Tests must use `import` not `require`.
2. **GitHub Actions has CI=true by default** — bump-build.mjs checks for this, so it's partially protected. But explicit SKIP_BUILD_BUMP is better.
3. **npm registry propagation is real** — Verify steps MUST have retry logic. 5 attempts × 15s = 75s max wait (good).
4. **Workflow naming matters** — squad-publish.yml.deprecated still exists in repo but filename says "deprecated" — delete these to avoid confusion.
5. **Semver validation is cheap** — `npx semver "$VERSION"` is a one-line check that prevents disasters.

**Recommendations for Team:**
- **P0 blockers must be fixed before next release** — squad-release.yml test failures, semver validation, bump-build.mjs format
- **Publish.yml needs hardening** — Add validation gates (semver, SKIP_BUILD_BUMP, dry-run, token check)
- **Test suite needs stability** — Fix or quarantine flaky tests (human-journeys.test.ts)
- **Clean up stale workflows** — Delete deprecated files, clarify redundant workflows

**Collaboration Notes:**
- Trejo (Release Manager) should be aware of squad-release.yml being broken — releases from main are blocked
- Fenster (CLI Engineer) should investigate human-journeys.test.ts failures — CLI error handling may be broken
- Keaton (Architect) should weigh in on bump-build.mjs fix (change format vs. remove script)

**Next Steps:**
- Brady will use this audit to create a PRD
- Trejo is auditing GitOps/release side separately
- Wait for direction on which P0 issues to fix first

**Learnings Applied to Charter:**
- Defense in depth principle confirmed — validation gates are essential
- Retry logic pattern works (5 attempts, 15s intervals, exponential backoff)
- Semver validation is mandatory — one-line check prevents disasters
- SKIP_BUILD_BUMP must be enforced in CI — not just checked by script
- Test failures are often ES module syntax issues — watch for require vs. import

**Documentation Created:**
- `docs/proposals/cicd-gitops-prd-cicd-audit.md` — Comprehensive audit with findings, priorities, code snippets
