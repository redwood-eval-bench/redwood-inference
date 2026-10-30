# Add rolling lagged-anchor calibrator and soft holdback

Context and motivation: We observed recurring false-positive regressions caused by anchor reference sets that lag behind recent traffic changes (time-lagged drift). The existing anchor snapshot process is snapshot-based and can be brittle when prompt distributions shift within short windows (e.g., marketing campaigns, A/B experiments). This PR implements a lightweight rolling/lagged anchor resampler that maintains time-windowed anchor slices and computes a calibrated, age-weighted threshold for regression CI gates. The goal is to reduce noisy CI failures while preserving sensitivity to genuine regressions.

What this PR does (summary): Adds RollingAnchorResampler that: - keeps a ring of recent anchor snapshots keyed by window (hour/day) - computes weighted statistics (decay by recency) for metric baselines - exposes a resample_decision API used by CI hooks; integrates a SoftHoldback CI hook that: - when a gating assertion would otherwise fail, the hook can mark the run as soft-failed, route examples to quarantine for manual triage, and optionally delay hard-fail until N successive soft-fails across windows - emits telemetry so ops can track soft-holdback rates and replays.

Checklist: - [x] Core algorithm for age-weighted baseline and threshold fusion - [x] CI hook integration with toggles (env + feature flag) - [x] Unit tests for resampler behavior across synthetic time-series - [x] Integration test covering holdback + replay path - [x] Docs describing config knobs and migration guidance - [x] Telemetry metrics and dashboards snippets included

High-level design notes: RollingAnchorResampler is intentionally conservative: it will only resample to a new anchor if there is sustained shift (configurable min-duration and min-sample-count). Thresholds are fused across windows using exponential decay weights to avoid abrupt jumps. The soft holdback path is a policy layer that can be enabled per-prompt-set; default is opt-in for canary cohorts.

Files changed (high level): - eval_harness/anchors/rolling_resampler.py: core implementation + public API (ResamplerConfig, RollingAnchorResampler, _fusion helpers) - eval_harness/ci/hooks/soft_holdback.py: CI-side hook used by the eval runner; integrates with existing reporter and failure routing - eval_harness/telemetry/anchors_metrics.yaml: new metrics (anchor_age_seconds, resample_decision_count, soft_holdback_rate) - tests/test_rolling_resampler.py: unit tests for fusion logic, boundary conditions, and resample gating - docs/anchors/rolling-calibrator.md: design rationale and operational guidance - scripts/recompute_anchors.py: helper to backfill rolling anchor windows for historical data

Review timeline & conversation summary: - 2025-08-14 initial draft opened. Jonah requested clearer separation between resampling and threshold fusion and asked for deterministic unit tests for edge cases (low-sample cohorts). - 2025-08-15 author pushed tests + refactor to expose deterministic fusion helper; CI unit tests failed lint and a flaky integration test. - 2025-08-16 fixed lint, increased test timeouts for an unstable integration that depends on local test clock; Ethan asked for added telemetry tags to correlate anchor_age with prompt_set; author added tags. - 2025-08-19 Priya ran canary evals against a staging workspace and reported reduced noisy CI failures on a recent marketing burst. Priya approved with minor comments. - 2025-08-20 final cleanup, squash and rebase onto main; CI passed all gates.

CI status and troubleshooting: The PR initially had a failing integration test (test_time_window_rollover) caused by non-deterministic clock usage in a helper; replaced with a frozen-time fixture. Lint failures were fixed in a separate commit. After rerun, all checks passed (unit, integration, lint, flake). GitHub Actions summary: unit-tests (pass), integration-tests (pass), linter (pass), security-scan (pass), canary-smoke (pass).

Merge and deployment notes: Merged on 2025-08-21 via squash merge (merge commit: "squash! introduce rolling anchor resampler + soft holdback (Maya Chen)"). Migration: Ops should enable the soft-holdback flag for one staging workspace first. Default resampler config is conservative; recommended rollback: set RESAMPLER_MIN_DURATION to a large value to effectively disable resampling.

Why this is different from other anchor work: prior PRs introduced static anchor refreshers and one-shot rollforwards; this change introduces an online-ish, age-aware fusion strategy and an explicit soft-fail CI mode that reduces alert noise while preserving auditability and replayability.

Follow-ups (recommended): - ENG-4991: dashboard for soft-holdback trends and cohort-level breakdown - ENG-5032: backfill historical anchor windows and run a re-evaluation sweep against the last 30 days - small UX: add a console toggle to visualize anchor-age heatmap

Release notes: "Introduce rolling anchor calibrator (age-weighted fusion) and a soft-holdback CI hook to reduce transient false-positive regressions due to prompt distribution shifts. Configurable and opt-in; includes telemetry and migration guidance."

Merge method: squash

Merge commit: "squash! introduce rolling anchor resampler + soft holdback (Maya Chen)"

Breaking change: false
Introduce rolling anchor calibrator (age-weighted fusion) and a soft-holdback CI hook to reduce transient false-positive regressions due to prompt distribution shifts. Configurable and opt-in; includes telemetry and migration guidance.
