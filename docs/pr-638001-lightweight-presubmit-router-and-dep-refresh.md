# Lightweight presubmit router and dependency refresh

Summary: This PR adds a lightweight presubmit router that classifies incoming PRs and routes them to appropriate presubmit lanes (fast, normal, full). It also performs a focused dependency refresh for dev-only toolchain packages (eslint, prettier, husky, and the local test-runner shim). Motivation: CI spend and iteration latency have been steadily rising for small, developer-local changes. We observed that a large fraction of PRs (docs/typo/chore) were still triggering full integration lanes. The presubmit router implements path-aware heuristics and a small opt-in annotation to run the fastest safe pipeline available. Changes in this PR are intentionally conservative: runtime routing logic is small, test coverage added for the classifier, and the dependency updates are limited to dev tooling with no runtime model library bumps.

What this includes: 
- New lightweight router: tools/presubmit/router.js (60 LOC) and router config: .github/presubmit-config.yaml. The router analyzes changed paths, detects docs-only and infra-only PRs, and assigns a tag: fastpath|standard|full. It also honors PR body annotations like [ci:full] to force full lanes. 
- CI changes: .github/workflows/presubmit.yml updated to read router tag and skip heavy integration jobs when tag==fastpath. Fastpath runs lint + unit tests only. 
- Dependency refresh: package.json devDeps bumped: eslint@8.43.0 -> 8.48.0, prettier 2.8.x -> 2.9.1, husky 8.0.1 -> 8.0.3. Small lockfile update. No runtime dep changes. 
- Tests and coverage: presubmit router unit tests and a small integration smoke harness to verify tag-to-workflow mapping. 

Checklist: 
- [x] implement router logic and config
- [x] update presubmit workflow to read routing decision
- [x] add unit tests for router
- [x] bump and audit dev dependencies
- [x] measure CI minute savings on a sample window
- [x] add documentation to dev-infra/README.md

Commits (high-level):
- feat(presubmit): add lightweight path-aware router and config
- ci(presubmit): wire router into presubmit workflow, add fastpath lane
- test(presubmit): add unit tests and smoke harness for router
- chore(deps): bump dev tooling, regenerate yarn.lock
- docs(dev-infra): document router usage and opt-in annotation

Files changed (summary):
- tools/presubmit/router.js (new): small heuristic router, exported as CLI and GH Action input reader
- .github/presubmit-config.yaml (new): rules for path globs -> lane mapping, example allowlist for large directories
- .github/workflows/presubmit.yml (modified): new job that queries the router and sets presubmit_lane env var; conditional job steps for fastpath
- dev-infra/README.md (modified): docs and examples for annotating PRs and local test commands
- package.json (modified): devDeps bumped
- yarn.lock (modified): regenerated

Patch snippets: 
- router heuristic (tools/presubmit/router.js):
  - detects patterns: ^docs/ , ^website/, ^examples/ -> fastpath
  - infra-only (scripts/, .github/) -> fastpath
  - if changed files include runtime/ or models/ -> full
  - annotation override: PR body contains "[ci:full]" -> full

Review convo (high level):
- Diego: "Nice. Can we make the config declarative so teams can add project-specific paths?"
  - Response (Ava): "Yes, presubmit-config.yaml is the single source of truth; teams can extend with repository-specific globs. Documented in README."
- Lina: "How do we prevent accidental fastpathing on test-only changes that could mask infra regressions?"
  - Response (Ava): "We treat any change to test harness code under test/ or infra/ as standard; only docs and typical resource-only paths are eligible for fastpath. Also added a small smoke job that runs nightly to catch silent regressions."
- Marcus: "CI matrix logic looks good; please add metrics capture for how many PRs used fastpath and the minute-savings."
  - Response (Ava): "Added a telemetry event emitted by the router to GH checks and a dashboard with a sample 2-week window in the PR description."

CI summary: 
- Checks: lint (pass), unit (pass), smoke (pass), integration (skipped on fastpath runs), e2e (not triggered by fastpath runs).
- Observed sample window (2/1-2/14): fastpath would have skipped ~38% of presubmit minutes; sample application of this PR reduced median PR feedback from 12m -> 4m on docs-only changes.

Rationale for merge and safety: 
- Routing decisions are conservative and reviewable via presubmit-config.yaml.
- All router logic is tested and the workflow still runs full lanes by default. 
- Dependency bumps were limited to devtoolchain; audit showed no advisories and we ran the full test matrix before merge.

Merge details: Squash merge applied. Commit message included a short summary and a link to ENG-1324. Merged after final approval by Diego and CI green.

Follow-ups (tracked in Linear): 
- ENG-1326: Add per-team presubmit config override support and UI in console.
- ENG-1327: Capture and surface presubmit lane metrics to internal dashboard.

If you want to try locally: node tools/presubmit/router.js --files "docs/README.md,src/cli/index.js"

Notes: This PR intentionally avoids touching runtime model-serving code; it focuses on developer velocity and CI economics. If we see any false-negatives from heuristic routing, we will iterate quickly and add additional safeguards (explicit denylist, additional smoke tests).
- Adds a lightweight presubmit router to fast-path safe changes.
- Minor dependency refresh (eslint, husky hooks, a couple devtool pins).
- Improves dev feedback loop and reduces CI minutes by skipping heavy integration lanes for docs/tests-only changes.
