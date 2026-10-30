# Introduce single-pass lint with sparse-change detection to speed CI

Context: The monorepo's lint job has regressed in wall-clock time as the codebase grew; CI lint runs now dominate PR feedback latency. This change introduces a single-pass lint pipeline plus a sparse-change detector so that CI runs lint only the minimal impacted set of files for most PRs, while preserving a full-lint fallback for cases where the sparse detector cannot safely determine dependencies.

Motivation: faster feedback cycles for engineers, reduced CI cost, and fewer queueing delays for runners.

High-level changes:
- Add a single-pass parser/lint runner that walks file ASTs once and emits rules incrementally (packages/tools/lint/single_pass_runner.py).
- Implement a sparse-change detector that computes a minimal affected file set based on import graph traversal and test/infra heuristics (packages/tools/lint/sparse_detector/*).
- Introduce a new composite GitHub Action (ci/actions/lint-single-pass-action.yml) that chooses between fast-path (sparse) and full-path based on detector output and branch age.
- Wire the composite into the main PR workflow, replacing the old monolithic lint job (ci/workflows/pr-tests.yml).
- Add telemetry: histograms for lint duration, fast-path hit rate, and skipped-file counts (telemetry/metrics.yaml + instrumentation in runner).
- Update CONTRIBUTING.md and dev docs with troubleshooting and how to run the fast-path locally.

Implementation notes:
- The single-pass runner reuses existing rule implementations but orchestrates them in a single AST traversal to avoid repeated parsing. This required small adapters for a few rules (e.g., no-async-await-in-legacy-tests) to be streamable.
- Sparse-change detector relies primarily on static import edges and a lightweight heuristic that treats build/infra files as always impacted. For cross-language modules we conservatively mark sibling files as impacted.
- We added a safety gate: if a PR touches more than 250 files or cannot compute a reliable graph (e.g., new generated code patterns), the action falls back to full lint.
- We intentionally did not change any lint rules or thresholds; this is performance-only.

Commits (summary):
- feat(lint): single-pass AST runner and adapter shims
- feat(lint): sparse change detector and import graph builder
- ci: composite action for fast lint path with fallback
- docs: add contributor guide and operation runbook for fast lint
- test: add unit + integration tests for detector and runner
- perf: add telemetry hooks and dashboards

Changed areas (high level):
- packages/tools/lint/single_pass_runner.py
- packages/tools/lint/sparse_detector/core.py
- ci/actions/lint-single-pass-action.yml
- ci/workflows/pr-tests.yml (lint job replaced with composite)
- telemetry/metrics.yaml
- docs/CONTRIBUTING.md and docs/dev/lint-fastpath.md
- tests/lint/* (new unit/integration tests)

Review conversation (highlights):
- Liam O'Connor: "This looks promising. Can we add a metric for false negatives where fast-path skipped something caught by the full run?"
- Maya Patel (author): "Added histogram lint.fastpath.false_negatives_count and a daily alert if non-zero in the last 24h; see telemetry/metrics.yaml."
- Priya Desai: "Please ensure pre-commit hooks still run locally and that the action exits non-zero when the full-lint would fail."
- Maya Patel: "Pre-commit configuration updated (tools/pre-commit-config.yml). The composite action runs local pre-commit step in the checkout stage and will error if any hook fails. Also added explicit instructions to the CONTRIBUTING doc."
- Noah Kim: "Concern: this could hide cross-package contract changes (e.g., typed interface changes). How do we ensure safety?"
- Maya Patel: "Sparse detector marks direct dependents and a shallow transitive closure up to 2 hops by default; we made this tunable via CI env var LINT_SPARSE_HOPS. We also added a daily full lint on main to catch any missed regressions."
- Priya Desai (after tweaks): "Approve after telemetry and pre-commit checks added."

CI behavior during review:
- Initial run: fast-path failed intermittently due to a flake in the detector's file-watching mock in integration tests.
- Fix: stabilize test by using deterministic timestamps (tests/lint/test_detector_integration.py).
- Rerun: all checks passed.

Manual smoke: local dev tried running the action with --fast-path and observed ~70-80% reduction in elapsed time for small PRs.

Operational notes / roll-out plan:
- Feature flagged via ci/actions/lint-single-pass-action.yml input; rollout enabled for internal team first, then org-wide after 72h with monitoring.
- Telemetry dashboards added to SLO board: lint.duration.p50/p90, lint.fastpath.hit_rate, lint.fastpath.false_negatives_count.
- Alert: high false_negatives_count or >1% error rate will auto-disable fast-path via alert-runbook.

Backwards compatibility / risk:
- No lint rules changed.
- Pre-commit still enforced.
- Full-lint remains available as fallback.
- Large PRs/complex new code patterns trigger full-lint.

Follow-ups planned:
- ENG-5022: expand graph accuracy for generated code and cross-language modules.
- ENG-5099: experiment with caching parsed ASTs across CI jobs.
- Add a small migration note for contributors in release notes.

Merge notes: squashed and merged after approvals; commit message reflects the combined summary and points to linked ticket ENG-4981.
Speeds up lint job by ~3x on typical PRs by using a single-pass parser and linting only files that changed or depend on changed files. Falls back to full lint for large/old branches. No behavior changes to lint rules.
