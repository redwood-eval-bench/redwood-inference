# Introduce concurrency rasterizer and kernel-iceberg sampler for micro-contention profiling

Motivation: We frequently observe transient kernel-level contention that standard latency histograms miss because the events are short, high-amplitude, and correlated across co-scheduled requests. This PR introduces two companion pieces: (1) a lightweight "concurrency rasterizer" runner that replays interleaved request traces into fine-grained time-slices to expose cross-request kernel overlap patterns, and (2) a 'kernel-iceberg' sampler and reporter that detects rare, high-impact kernel-starvation events (the iceberg) and surfaces per-kernel attributions in an analysis-friendly Parquet export.

What changed (summary):
- New runner: runners/concurrency_rasterizer.py — deterministic time-slice based replay with tunable stride/window knobs.
- New sampler/reporter: reporters/kernel_iceberg_reporter.py — online reservoir sampling of high-amplitude kernel contention, computes per-kernel score, exports compact event records and Parquet summary.
- Added baseline spec: benchmarks/profiles/iceberg_baseline.yaml with target input mixes and warmup strategy.
- Integration harness: tests/integration/test_iceberg_end2end.py and a small synthetic workload generator under tests/synth/iceberg_workload.py.
- Dashboard/docs: docs/benchmarks/iceberg.md and notebooks/iceberg-analysis.ipynb with example queries and visualizations.
- CI: ci/benchmarks.yml updated to run the rasterizer in a scheduled job and record Golden traces.

Why this is useful: The rasterized replay flattens timing jitter and makes micro-contention reproducible across runs. The iceberg reporter distills noisy telemetry into a small set of candidate events for root-cause investigation, reducing manual triage time on long tail regressions. This complements existing tail-samplers by prioritizing cross-request kernel overlap rather than single-request latency.

Checklist:
- [x] Core algorithm implemented and unit-tested
- [x] End-to-end integration test against synthetic workload
- [x] Exporter produces compact Parquet + JSON artifacts for analysis
- [x] Docs and notebook with example workflow
- [x] CI job added (scheduled)

High-level commits: see commit_summaries field for per-commit descriptions.

Notes and follow-ups:
- Default behavior is disabled for nightly runs (flag --enable-iceberg required) to avoid noise in normal benchmark pipelines.
- We will add an automated signal-to-noise filter in a follow-up (ENG-4477) to only keep iceberg events that show reproducibility across a small cohort.
- The exporter schema is intentionally compact: each event contains timestamp window, kernel-id hint, estimated overlap_ms, sample_score, and a tiny stack fingerprint. Larger traces are left to optional artifact upload.

Related: Linear ENG-4421 (identify micro-contention taxonomy), ENG-4430 (add rasterized replay harness).
Ethan Cole: The reservoir size default looks small for bursty workloads — can we make it adaptive or at least document failure modes?
Sofia Ramirez: Good point — added adaptive-reservoir logic with a 4x cap and documented in the reporter doc (see reporter docstring).
Maya Singh: The integration test is flaky on low-cpu CI nodes, suggested using mocks for timing rather than real sleeps.
Sofia Ramirez: Reworked integration test to use deterministic clock shim from utils/timeutils.py; CI pass observed on rerun.
Liam O'Connor: Approving; left one minor nit about var naming in concurrency_rasterizer.py which I don't think blocks merging.
Sofia Ramirez: Addressed var rename and pushed; thanks for the review.
Add experimental "kernel-iceberg" micro-contention profiler and a concurrency rasterizer runner; disabled by default. Use for in-depth tail analysis and exploratory root-cause hunts.
Experimental feature behind flag --enable-iceberg; will be promoted to default when reproducibility filter is implemented (ENG-4477).
Run `pytest tests/integration/test_iceberg_end2end.py` or invoke `python runners/concurrency_rasterizer.py --config examples/iceberg-demo/runner_config.json --enable-iceberg` to produce Parquet artifacts under ./artifacts/iceberg/
Low-risk: changes are additive and guarded by a feature flag. Potential CPU overhead when enabled; mitigated by sampling reservoir and a conservative default stride.
