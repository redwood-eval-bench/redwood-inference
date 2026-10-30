# Introduce percentile-variance heatmap canary with adaptive replay and stratified probe replayer

Background: we see recurring noisy tail regressions that only surface when traffic composition shifts (short-text vs long-context, embedding vs generation), which the current percentile canaries either under-react to or produce excessive false positives. Goal: introduce a detector that measures variance across percentiles (a percentile-variance heatmap) and an adaptive replay system that replays stratified micro-bursts from live traces to stress candidate variants deterministically. This PR implements the detector, a lightweight replayer service, integration hooks into the rollout controller, and observability artifacts.

What this PR does: adds a percentile-variance heatmap detector (sliding-window, decay-weighted), exposes heatmap-score and contribution breakdown metrics, implements an adaptive stratified-probe replayer that reconstructs representative bursts by traffic slice, hooks a new evaluation step into the canary pipeline that runs replay probes against new variants, introduces a new rollout guard that combines heatmap thresholds + replay pass/fail signals, exposes a Grafana heatmap dashboard and alert that uses a composite signal, and wires the feature behind a feature flag (perf.heatmap_canary).

Why it matters: the heatmap captures dispersion across percentiles (e.g., simultaneous shift at p50 and p99) which is hard to detect with single-metric detectors; the replayer enables deterministic validation of suspected regressions so we can avoid noisy automated rollbacks while still failing fast on real degradations.

Implementation notes: detector implemented in perf-canary/src/heatmap_detector.go using an incremental KD-style binning for percentiles to keep memory bounded; replayer implemented as an independent process (perf-canary/replayer/) that pulls hashed request signatures and sampled KV caches from the probe broker, reconstructs sequences with preserved inter-arrival times for micro-bursts, and emits structured replay results (latency distributions, token-level error rates). Rollout controller integration: new canary step `heatmap_replay_evaluation` added to pipeline DSL; policy supports soft-hold (pause weight increase) and auto-throttle (reduce concurrency) to allow graceful mitigation during marginal failures.

Backward compatibility and safety: default thresholds are conservative; feature is opt-in via feature flag and Linear ticket triggers for production enable. Replayer does not store raw payloads — only hashed request signatures and anonymized length buckets.

Testing done: unit tests for detector math and decay windowing, integration tests that run replayer against synthetic traffic shapes in perf-lab (short-text long-text mixed, spike/decay patterns), staging end-to-end smoke (canary run at 5% traffic against mock variant), and load-run against dedicated GPU nodes to validate performance overhead (observed <1.5% extra CPU and negligible GPU impact). Added replay fixtures and golden files in tests.

Rollout plan: behind feature flag in main; enable in perf-canary staging for 48h, then gradual enable on canary rollouts (5% -> 25% -> 50%) with automated hold if heatmap_score > 0.18 or replay_failures > 2 in the sample window. Operational runbook added to docs/canaries/percentile_variance.md.

Checklist:
- [x] Implement detector algorithm and metrics emission
- [x] Implement stratified probe replayer and local harness
- [x] Integrate evaluation step into canary pipeline DSL
- [x] Add dashboard and alert panels
- [x] Unit & integration tests
- [x] Run staging canary + smoke tests
- [x] Add rollout runbook and feature flag

Limitations & TODO:
- Thresholds are currently tuned on internal lab traces; we will run an adaptive calibration job (follow-up) to tailor thresholds per-customer traffic profiles.
- Replayer currently supports a limited set of request shapes; adding multimodal fixtures (image/audio) is tracked in ENG-8139.

Related linear/jira: ENG-8124, ENG-8139, PM-341.
Arjun Rao (2026-02-20): Could you add more detail on how the heatmap thresholds are derived and whether they'll be per-customer? Requested: add a calibration job and document defaults.
Maya Chen (2026-02-21): Added calibration follow-up ENG-8139 and documented default conservative thresholds. The detector emits per-customer baseline stats to support later auto-calibration.
Sofia Martinez (2026-02-24): Replayer looks good but we need a privacy review sign-off for trace materials. Please confirm only hashed signatures are stored.
Maya Chen (2026-02-24): Confirmed in description and replayer code comments: raw payloads never persisted, only hashed request signatures and anonymized length buckets. Added a unit test asserting hashing pipeline.
Noah Klein (2026-02-26): CI flake on replayer integration; reran jobs and inspected logs. Approve after green CI.
CI Bot (2026-03-02): Integration job retried and passed. All checks green.
