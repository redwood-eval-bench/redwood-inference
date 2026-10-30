# Introduce lint-weight trimmer and release-window prober to reduce presubmit footprint

Motivation: Presubmit pipelines have grown costly and noisy — a large fraction of lint and heavyweight test shards run for tiny surface-area changes. This PR introduces two complementary mechanisms: (1) a lint-weight trimmer that computes a per-change weight and selectively runs lighter lint shards for small changes, and (2) a release-window prober that defers/re-schedules heavyweight gating tasks to controlled windows and proactively checks for release collisions.

What this change does (single-string summary): Adds a lightweight Go CI shim and a deterministic 'lint weight' heuristic, configures presubmit workflow to opt into trimmed lint paths via a 'ci:trim' label and change-size thresholds, introduces a 'deferred-heavy' tagging flow for tests that can be run on nightly/release windows, and adds a small release-window prober that validates mergeability and dependency drift during release windows. Also adds migration tooling to backfill historical file-surface metadata for accurate weight computation.

Why now: We observed median presubmit runtime increasing 28% year-over-year and SRE/infra costs up ~12% attributable to redundant lint/test runs on low-risk changes (docs, README edits, vendor updates). This is a low-risk opt-in mechanism that yields immediate CI cost and latency improvements.

Checklist:
- [x] Implement lint-weight compute algorithm (deterministic, language-aware, cached per-commit)
- [x] Replace targeted bash flows with a small Go shim for stability and observability (ci-shim)
- [x] Integrate trimmed lint path into .github/workflows/presubmit.yml with safe opt-in label and change thresholds
- [x] Add release-window prober that runs during configured release hours and posts status to the PR and release dashboard
- [x] Add migration script to backfill file-metrics (scripts/ci/migrate-lint-weights.go)
- [x] Add telemetry hooks (statsd/datadog) and opt-out env var for Private deployments
- [x] Unit and integration tests for weight calc and shim behavior
- [x] Docs for contributors and runbook for SRE

Rollout plan:
1) Merge behind feature flag (CI env: CI_LINT_TRIMMER=off by default).
2) Enable in opt-in mode: contributors can add 'ci:trim' label or maintainers can rely on automatic thresholds for non-critical paths.
3) Monitor Telemetry: presubmit latency, shard counts, false-negative rate for lint and test coverage delta.
4) Gradual tightening of thresholds over 3 weeks with fallback to full-run on any coverage regression.

Rollback plan: revert feature flag to off, or remove 'ci:trim' label from merges; migration/backfill script is non-destructive. Release-window prober is read-only by default and can be disabled via RELEASE_PROBER_ENABLED=false.

Risk & mitigations:
- Risk: missed lint finding because trimmed lint skipped an important rule. Mitigation: trimmed path still runs a core rule set; any failure in post-merge nightly job triggers an auto-rollout-block and alerts the author.
- Risk: hidden test regressions. Mitigation: heavy tests are scheduled to run in the next release-window and a short-circuit fallback runs full test on push-to-release branches.

Notes on CI cost impact (internal projection): expected presubmit runner-time reduction ~34% for small PRs, cluster CPU savings ~20% across working set.

Marcus Chen: Please add an integration test that exercises a multi-language change (TS + GO) to validate weighting across mixed diffs. Also suggest renaming 'weight' to 'lint_score' to be explicit.
Ava Martinez (author): Added tests under tests/ci/weight_calc_test.go and kept 'weight' name after discussion — we standardize on 'lint_weight' in code and map to 'lint_score' in external metrics. Added a comment explaining the naming.
Priya Nair: CI shim logs look good. Request: add retry/backoff on remote cache fetches and surface cache hit ratio in telemetry. Also can we scope Datadog tags by repo (repo:redwood) and environment?
Ava Martinez: Added backoff and cache metrics (internal/telemetry/metrics.go). Datadog tags now include repo and env. See last commit.
Ethan Cole: Verified release-window prober logic. One nit: the prober should not block merges if it cannot reach external dependency registry — made it read-only and log a warning instead. Approved after change.
Introduce lint-weight trimming and a release-window prober to reduce presubmit cost and schedule heavy tests. Opt-in via 'ci:trim' label; rollout behind CI_LINT_TRIMMER flag. See docs/ci/README.md for contributor guidance.
Run 'make ci-shim-test' for local shim integration. To simulate trimmed lint path: add label 'ci:trim' to a PR or set CI_LINT_TRIMMER=on in CI. Use scripts/ci/migrate-lint-weights.go to populate historical cache.
If presubmit regressions occur after enabling, flip CI_LINT_TRIMMER=false. Nightly backstop jobs will run heavy tests — check statsd.lint_trim.false_negatives and ci.release_prober.alerts. Migration script is idempotent.
