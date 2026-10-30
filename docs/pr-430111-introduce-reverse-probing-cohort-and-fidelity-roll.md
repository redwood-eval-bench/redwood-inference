# Introduce reverse-probing cohort and fidelity rollback adapter

Context and motivation: We observed a class of spurious CI failures where late-arriving contextual drift or transient model nondeterminism caused the primary smoke cohorts to surface regression alerts, triggering interrupts and engineer investigation. This PR introduces a lightweight "reverse-probing" cohort and a fidelity rollback adapter that works as a CI-layer soft-fallback: when a regression is detected, the reverse-probing cohort re-evaluates the failing prompt subset with inverse perturbations and seed-variance traces to detect instability vs. true quality degradation. If the reverse probe indicates high instability (configurable threshold), the fidelity rollback adapter will mark the CI gate as 'soft-fail' and trigger a prioritized re-run and targeted replay rather than a full blocking failure. This reduces noise and gives a fast path for automated triage.

What this PR includes (single-line summary): implement reverse-probe cohort sampling, probe scheduler, rollback adapter, CI hook, unit and integration tests, docs and lightweight telemetry.

Checklist:
- [x] Implement reverse-probe sampling strategy and scheduler
- [x] Implement fidelity rollback adapter and CI hook integration
- [x] Add unit tests for probe behaviors and adapter decision logic
- [x] Add integration test that simulates nondeterministic churn and ensures soft-fail semantics
- [x] Add docs: rationale and runbook for ops teams
- [x] Add telemetry hooks: probe outcome, rollback.reason, replay.job_id

Design notes: The reverse-probe cohort uses a complementary perturbation set (token-suffix shuffles, paraphrase templates, and inverse temperature sweep) to exercise orthogonal failure modes compared to standard anchors. The adapter uses a two-step decision function: (1) compute instability score = weighted combination of cross-seed divergence and paraphrase disagreement, (2) if instability_score > instability_threshold and absolute regression delta < small_delta_threshold, mark as soft-fail and schedule targeted replay with higher sample concentration. Thresholds are configurable via ci.config.replay.

Commit summary:
- 10f4a6c: introduce reverse_prober module and sampling primitives
- 2b9c4de: add fidelity_rollback adapter and CI hook glue
- a3d2f71: tests: unit tests for instability scoring and adapter pathing
- f4e1b58: integration test for nondeterministic churn and soft-fail behavior
- 9c8b41d: docs: add runbook and configuration example

Files changed (high level):
- eval_harness/replay/reverse_prober.py (new)
- eval_harness/adapters/fidelity_rollback.py (new)
- ci/hooks/replay_gate.py (modified)
- tests/unit/test_reverse_prober.py (new)
- tests/integration/test_fidelity_rollback_simulation.py (new)
- docs/ci/reverse_probe_runbook.md (new)
- telemetry/metrics.yaml (modified) -- adds probe.outcome, rollback.reason
- examples/ci_configurations/reverse_probe_example.yml (new)

Testing and CI: The PR wired up a staged CI job: unit -> integration(replay-mock) -> e2e(smoke). Flaky network on first integration run caused a single rerun; final CI status is green. The integration includes a mocked replay service so this can run in CI without requiring cluster access. Local tests report ~98% determinism on synthetic seeds and a measured 18% reduction in blocking regressions in our internal canary simulation when enabling the soft-fail path.

Review conversation (high-level highlights):
- Ethan Liu: "Core idea looks solid. Can we avoid adding too many knobs? Suggest default thresholds that are conservative. Also, include a short circuit for 'known flaky prompts' so we don't hit the adapter repeatedly." (author reply: added known_flaky_cache and conservative defaults)
- Priya Nair: "Please ensure telemetry tags include model_version and seed_range. Also ensure adapter doesn't hide real regressions with large deltas." (author reply: telemetry tags added; adapter enforces absolute regression delta check)
- Liam O'Connor: "Tests look good; add one more test for paraphrase-induced false positives." (author reply: added paraphrase scenario)

Author notes on risk: The adapter only affects CI gating and is gated behind a feature flag ci.features.reverse_probe_rollout. It does not change production serving behavior. Config defaults are conservative; the rollout will start on the eval-harness canary cohort.

Operational runbook (short): When the adapter marks a gate as soft-fail, it emits rollback.reason with values {"seed_variance","paraphrase_disagreement","late_arrival"}. Operator steps: (1) inspect telemetry dashboard for rollback.job_id and failing_prompt_ids, (2) if instability is reproducible, create a prompt ticket with the failing set, (3) if a true regression is determined, escalate via ENG-2101.

Backward compatibility: No public API changes. CI configuration example included. Documentation updated with configuration snippet and recommended thresholds.

Merge details: squash-merged after approvals from Ethan and Priya. Changelog and docs updated in the same PR to make rollout smoother.
Adds a reverse-probing cohort and a new fidelity rollback adapter to reduce false regression flags and allow graceful CI soft-failures for transient prompt-set instability. No API surface changes for consumers.
This PR aims to reduce noisy CI interruptions by distinguishing transient instability from true regressions. Expected impact: fewer blocking alerts, faster triage; follow rollout and monitor probe.outcome and rollback.rate.
Feature flagged rollout, conservative defaults, telemetry to detect regressions, explicit absolute-delta check to avoid masking real failures. Ops runbook included.
Ethan Liu: Looks good overall; recommend conservative defaults and known_flaky cache. Author: added known_flaky_cache + defaults.
Priya Nair: Include model_version and seed_range tags in telemetry and guard against hiding large deltas. Author: telemetry tags added; adapter enforces absolute delta check.
Liam O'Connor: Add paraphrase-induced false positive test. Author: added new paraphrase scenario test.
