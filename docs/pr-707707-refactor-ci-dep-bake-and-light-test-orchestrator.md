# Refactor CI: introduce dependency-bake layers and lightweight test orchestrator

Motivation: CI feedback time has been a recurring blocker for developer productivity and release velocity. This PR reduces cold-start overhead for CI by pre-baking dependency layers (dep-bake) for common language toolchains and introduces a lightweight test orchestrator that dispatches only the relevant test profile(s) for a changed package. The change targets: (1) reduce median CI run time for PRs touching single packages, (2) avoid costly monolithic test runs on unrelated changes, and (3) make lint/test configuration easier to extend across the monorepo.

What changed: prebuilt dependency layers are produced by .github/actions/dep-bake and referenced by CI runners; runner images now mount cached dep layers when available. The orchestrator (scripts/ci/orchestrator.py) inspects git diff against base and maps touched paths to test profiles (unit, integration-smoke, e2e-nightly). Linting was decoupled into a dynamic sharder that runs only affected linters (python, ts, go) when relevant files change. A small compatibility shim was added to preserve existing matrix jobs for release branches.

Performance: in our staging runs the average PR CI time dropped from ~18m to ~9m for single-package changes; cache-hit runs can be <5m for trivial docs/README edits. Integration-heavy changes still run full suites as before.

Rollout plan: staged rollout: (a) land dep-bake and orchestrator behind a feature-flag in main, (b) opt-in the infra team and a sample of active repos for two weeks, (c) enable org-wide after one week of stable runs and two successful rollback drills. This PR implements the feature-flag and the opt-in mapping table (ci/config/profile-mapping.yaml).

Compatibility and safety: full-matrix jobs remain for release/* branches and for PRs marked [full-ci]. We added a fail-open behavior: if the orchestrator fails, CI falls back to running the legacy matrix rather than blocking merges. The dep-bake artifacts are versioned by a short build-tag so older pipelines can continue to reference previous bakes.

Testing performed: multiple self-hosted runner runs, cross-runs on GitHub-hosted images, manual validation of fallback behavior, and a Canary namespace enabling the dep-bake artifacts for a subset of runners. Metrics and traces were collected to validate KV cache pressure and network I/O on runners.

Migration notes: CI docs updated (docs/ci/dep-bake.md) with guidelines on adding new language layers and how to regrow a stale bake. Teams with custom runner images must add the dep-bake mountpoint or use the bake-helper container.

Checklist:
- [x] Dep-bake action and Dockerfile
- [x] Orchestrator script + mapping config
- [x] Dynamic lint shard implementation
- [x] Fallback matrix and feature-flag gating
- [x] Docs and runbook updates
- [x] Stage rollout and validation

Notes: This PR intentionally avoids changing release signing or artifact upload semantics — those remain identical to current main.
Maya Thompson: Left a few nits on orchestrator.py re: the mapping precedence (package-level overrides should win). Also asked for an integration test that simulates fallback behavior.
Evan Chen: Addressed mapping precedence; added unit tests and a small integration harness that intentionally triggers orchestrator exceptions to validate fallback (added test/infra/runner_timeout_fix).
Ravi Kapoor: Concern about bake tag rotation and stale caches in long-lived runners. Requested that we add a TTL policy and a debug endpoint for bake info.
Evan Chen: Added TTL policy (scripts/ci/bake_helper.sh) and a /_bake_info endpoint on helper container; documented fallback in runbook.
Leah Martinez: Approved with comment: please annotate the canary-run metrics in Grafana and add alert if cache-hit rate drops below 60% for two hours.
CI Bot: All required checks passed after rerun (integration tests were flaky first run, then passed on rerun).
7a9d3c2 - feat(ci): add dep-bake action and bake Dockerfile (scripts/ci/dep-bake)
4f2b1e0 - feat(ci): implement lightweight orchestrator and profile mapping (scripts/ci/orchestrator.py, ci/config/profile-mapping.yaml)
b6c8a44 - refactor(ci): decouple lint runners into dynamic-sharder and update workflows (.github/workflows/ci.yml)
c3e5d98 - chore(docs): add dep-bake.md and orchestrator runbook (docs/ci/dep-bake.md)
d12fa71 - ci(test): add canary flag and fallback to legacy matrix when orchestrator errors
e8f7b22 - fix(tests): address race in integration-smoke runner (test/infra/runner_timeout_fix)
fa1b2c3 - perf(ci): tune cache keys and make bake tag stable across minor tool updates
Introduce pre-baked dependency layers to speed up CI runner cold starts; teams can opt-in via ci/config/profile-mapping.yaml.
Add a lightweight CI orchestrator that runs only relevant test profiles for changed packages, dramatically reducing median PR CI time for single-package changes.
Dynamic lint sharding to run only language-specific linters when relevant files are changed.
Fallback to legacy full matrix on orchestrator errors; no breaking changes to release pipelines.
Docs and runbook updated: docs/ci/dep-bake.md (how to add language layers and regrow bakes).
Deploys a dep-bake system and a fast test orchestrator to cut CI latency; staged rollout with safe fallback to legacy matrix.
