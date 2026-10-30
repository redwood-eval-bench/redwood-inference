# split-heavy-integration-tests-add-incremental-test-runner-and-dep-gate

Motivation: Our nightly and PR CI pipelines are dominated by a small set of heavy integration tests that run even when irrelevant files change. This PR: 1) splits heavy integration tests into an explicit heavy-integration job that only runs when infra/runtime or integration test files change (pattern: tests/integration/**, infra/**, runtime/**) 2) introduces an incremental test runner called 'fast-checker' which runs unit tests and a lightweight set of integration smoke tests for most PRs; it uses a file change heuristic to determine coverage 3) adds a dependency viability gate that runs a quick dependency-check step (build-profiler + dep-heuristic) on PRs that touch package manifests or lockfiles and fails the gating stage if a dependency update increases binary size or adds native toolchain requirements beyond configured thresholds 4) augments release automation to defer tag creation until required smoke tests and cost-sanity checks pass (prevents noisy release tags from incomplete CI).

Checklist:
- [x] implement fast-checker incremental runner and integrate with existing jest/runtests
- [x] add heavy-integration job and change patterns
- [x] wire dep-heuristic step and build-profiler binary size check
- [x] update docs/ci.md and README workflow section
- [x] add unit tests for the file-matching heuristics
- [x] smoke-run in staging CI

Commits summary:
- 8 commits: initial fast-checker runner scaffolding; integration job split; dep-heuristic binary-size probe; build-profiler tool (small shim); docs update; test heuristics and unit tests; CI workflow adjustments; fix follow-up from review.

Changed areas (high-level):
- .github/workflows/ci.yml: split jobs into fast-checker, heavy-integration, dep-viability
- tools/build-profiler/: small binary-size probe used by dep-heuristic
- infra/ci/fast-checker/: runner scripts and node harness to select test subsets
- scripts/ci/dep-heuristic.sh: runs quick install, runs build-profiler and verifies thresholds
- docs/ci.md: CI behavior and gating rules

Illustrative snippets (small):
- File-matching heuristic used by fast-checker (in infra/ci/fast-checker/select.sh): "git diff --name-only $BASE..$HEAD | grep -E '(^tests/integration/|^infra/|^runtime/|^package.json|^yarn.lock)' || true"
- build-profiler usage in dep-heuristic: "./tools/build-profiler/profile.sh dist/main.js --report-json > /tmp/profile.json"

Why this approach: We want the common fast-path (iterative development) to be fast without changing test quality. Heavy runs still exist for PRs that touch relevant areas and for nightly jobs. The dependency gate reduces release thrash from inadvertent large binary or native deps sneak-ins.
Ravi Patel (2026-02-13): Nice separation. Two asks: (1) add a short-circuit path when a PR is docs-only (README, docs/*) to skip fast-checker entirely; (2) can the dep-heuristic report be uploaded as an artifact for the release engineer?
Ava Chen (2026-02-13): Added docs-only short-circuit and artifact upload in the latest commit (see commit g7b8c9d). The artifact is a small JSON with size delta and native-module flag.
Elena Morozova (2026-02-14): CI failed initially on the heavy-integration job due to a flaky integration that times out. Recommending we add a retry wrapper and increase timeout to 12m while we triage.
Ava Chen (2026-02-15): Increased timeout and added exponential retry (2 attempts). Reran CI.
CI (2026-02-15): initial run - fast-checker passed, heavy-integration failed (timed out), dep-viability pending
CI (2026-02-16): rerun - heavy-integration passed after retry, dep-viability returned 'warning' once (size delta +18% on a non-critical demo bundle) — threshold is 20%, so gated step passed but reported the warning in artifact
Samir Gupta (2026-02-16): Please add an explicit note in docs/ci.md about how the one-time warning is surfaced to release engineers and how to escalate. Also please add unit test for the artifact content format.
Ava Chen (2026-02-17): Added docs note and unit test for artifact schema. Final CI run green. Requesting approval.
Ravi Patel (2026-02-17): Approved. Small suggestion to later add telemetry for fast-checker hit-rate.
Elena Morozova (2026-02-17): Approved.
Introduce incremental test runner and dependency viability gate to reduce CI cost and make release tagging conditional on smoke checks.
