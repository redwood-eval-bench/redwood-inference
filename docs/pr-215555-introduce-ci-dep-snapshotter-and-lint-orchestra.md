# Introduce dep-snapshotter and lint-orchestra to reduce CI boot latency and dep churn

Motivation: CI boot time and repeated dependency installs are a significant part of our premerge latency (observed 35-55% of overall presubmit time across languages). This PR introduces two coordinated changes: 1) dep-snapshotter, a small Go service and packager that produces deterministic, compressed dependency artifacts (archives keyed by lockfile hash and platform tag) and a ci pipeline hook to publish them to the internal artifact cache; 2) lint-orchestra, a lightweight controller that coordinates linter runs across packages in a change-set so we avoid duplicate work (e.g., shared tsconfigs, monorepo packages) and allow staged language adapters to opt into fast-paths.

What changed (high level): Added a snapshotter binary (ci/snapshotter/*) that consumes lockfiles (pyproject.lock, package-lock.json, go.sum) and produces a signed tarball with a small manifest. CI pipeline ( .github/workflows/ci.yml and ci/pipeline.yaml ) now attempts to warm the runner workspace from a matching snapshot before running installs. We added a sparse-lock-importer script to only materialize install fragments needed for the changed files (reduces pip/npm churn). Lint-orchestra (tools/lint-orchestra/*) exposes an interface for adapters to report file-level ownership and run batches, and we wired Python and TypeScript adapters as prototypes. Tests include an end-to-end smoke that validates warm-path + fallback to normal installs and unit tests for the orchestra controller.

Checklist:
- [x] Implement dep-snapshotter + packager
- [x] Integrate warm-cache hook into CI pipeline
- [x] Add lint-orchestra controller and two adapters (py/ts)
- [x] Add e2e and unit tests
- [x] Document runbook and rollback
- [x] Feature-flag rollout plan for canary teams

Design notes: Snapshots are keyed by (lockfile_hash, platform, toolchain_stamp). The packager attests content using our CI signing key; snapshots are immutable and garbage-collected by an expiry policy. The sparse-lock importer maps changed paths to a minimal install set using heuristics and a small persisted index (built as part of nightly snapshots). Lint-orchestra uses a simple queue + lease model: a coordinator allocates owners to adapters and deduplicates file-level lint invocations; adapters can opt into parallelism hints.

Testing and validation: CI smoke runs were executed against dev and staging with a representative set of PRs. Observed median warm-path success 82% on staging (platform: ubuntu-22.04 images) and ~40-60% reduction in install time for cached PRs. Lint-orchestra reduced duplicate linter invocations in a 50-PR synthetic test by 38% and sped up average presubmit feedback by ~12s for multi-package changes.

Rollout plan: Feature-flag enabled in CI. Phase 1 (canary): opt-in for platform and infra teams for 2 weeks; Phase 2: enable for all presubmit jobs where a snapshot tag exists; Phase 3: default warm-path with monitoring guardrails. Lint-orchestra will be staged by repo and language adapter; default behavior is conservative (no skipping) and a telemetry-driven opt-in for teams.

Risks & mitigations: Broken snapshot keys could cause cache misses — mitigated by strong fallbacks to current install path and an automatic retry. Signing / verification errors are logged and cause a safe-fail to fallback. Lint-orchestra could introduce ordering/regression in linters; adapters run in a strict mode initially and the controller records a replayable trace for debugging. We included a rollback job to purge the pipeline hooks and a toggle to disable the warm path.

Follow-ups: 1) add Ruby/Go adapters for lint-orchestra; 2) increase cross-platform snapshot coverage (mac/linux/arm); 3) metrics exporter for snapshot hit/miss and per-adapter dedupe stats; 4) add artifact retention tuning to infra.

Operational notes: docs/ops/ci-snapshotter-runbook.md contains manual recovery commands for mis-signed artifacts and the steps to force a rebuild of nightly index. The CI team (Evan, Priya) will monitor snapshot-hit and installer duration dashboards for the first 4 weeks.
Adds a deterministic dependency snapshotter that produces lightweight archive artifacts for CI warmup; introduces a coordinated lint orchestration layer to reduce duplicate linter runs across languages and speed up presubmit feedback.
Evan: request_changes - Can we narrow the signer permissions? Priya: replied - moved signing to the CI agent and limited key access to the packager role, added audit logs. Marta: comment - need clearer failure-levels in pipeline to avoid noisy alerts; Priya: replied - introduced explicit metric tags and categorized logs as info/warn/critical. Diego: approved after small lint adapter refactor. CI: all checks passed after two reruns (one flaky npm cache issue), maintainer: squash-merged.
