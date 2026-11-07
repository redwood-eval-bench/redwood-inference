# Introduce temporal segmentation rate-of-change (RoC) canary with adaptive sensitivity scheduler

Motivation: Current canary detectors struggle when traffic exhibits structured temporal phases (e.g., morning ramp, lunch plateau, evening tail). Transient spikes during phase transitions increase false positives or mask gradual regressions. This PR introduces a temporal-segmentation rate-of-change (RoC) detector that segments traffic into short-term phases and applies an adaptive sensitivity scheduler so that detection thresholds scale with expected phase variance.

Design summary: The detector first slices recent traffic into fixed-length segments (configurable window and step) and computes lightweight summary statistics per segment (p50/p95, throughput, request-rate derivative). A robust RoC score is computed per segment using a trimmed-mean baseline and an EWMA of recent RoC values to smooth noise. The adaptive sensitivity scheduler increases sensitivity during low-variance windows and relaxes thresholds during high-variance or high-load windows (configurable hysteresis). To avoid churn we combine the RoC signal with an ensemble anomaly vote: RoC + quantile-cliff detector + load-aware z-score must cross a consensus threshold before raising an alert.

Key characteristics:
- Temporal segmentation parameters are runtime-configurable via canary policy (segment_length_s, step_s, smoothing_alpha, sensitivity_factor).
- Backfill-friendly: supports reprocessing historical slices and computing rolling calibration for p-value thresholds.
- Lightweight: per-segment computation uses O(1) memory per slice and is optimized for streaming ingestion (continuous perf-monitoring mode).

Testing and validation:
- Synthetic traffic harness: injected step/regression, ramp/regression, and diurnal patterns. The new detector reduces false positives on scheduled ramps by ~78% compared to the baseline single-window RoC detector while detecting a 6% median latency regression within the same time budget.
- Replay test: 30-day production replay run on a redwood-model-serving slice; no new alerts during scheduled traffic changes, regression hits identified in postmortem traces reproduced.
- Unit and integration tests added for segmenting logic, EWMA smoothing, and ensemble voting thresholds.

Rollout plan:
1) Feature gated off by default.
2) Canary deploy to small subset of projects (5%) with traffic mirroring enabled for 48h.
3) Metrics review dashboard: RoC score distribution, sensitivity factor histogram, false-positive rate delta.
4) Gradual ramp to 50% over a week, then 100% if stable. Auto-fallback route included (policy-based) to revert to baseline detector on sustained instability.

Checklist:
- [x] Implementation of temporal segmentation RoC detector
- [x] Adaptive sensitivity scheduler and policy bindings
- [x] Unit + integration tests
- [x] Backfill and replay hooks for calibration
- [x] Dashboard panels and alert rules (docs in perf-canary/docs)
- [x] Feature flagging and rollout playbook

Notes: This PR intentionally keeps the default policy conservative to avoid surprise alerts. Config examples are included in docs/examples/temporal_segmentation.md. Related: ENG-1624 implements the scheduler API used here.
Add new temporal-segmentation RoC canary detector and adaptive sensitivity scheduler. Backwards-compatible feature flag off by default. New dashboards and alerts included.
Liam O'Connor: Nice work — can you clarify how the hysteresis prevents flapping between high/low sensitivity on bursty endpoints?
Author (Priya): Hysteresis is implemented using a two-threshold state machine and a minimum dwell time (configurable). I added comments and a unit test to simulate rapid variance toggles.
Nora Patel: The replay numbers look promising; please add a breakdown by model family (chat vs embedding) so platform teams can estimate utility.
Author (Priya): Added replay breakdown in test_roc_replay and updated the docs with model-family caveats.
Ethan Zhao: CI failed on lint for a minor naming mismatch in the scheduler; please fix and re-run.
Author (Priya): Fixed naming and added lint rule exemption where appropriate; all green now.
Squash-merged into main. Feature gated behind 'temporal_segmentation_roc' flag and default policy conservative. Backfill and dashboard artifacts committed. Merge performed after approvals from Liam, Nora, and Ethan and CI green.
