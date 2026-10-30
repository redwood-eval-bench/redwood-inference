# Introduce priority concurrency cartographer and background-noise injector

Motivation: Current workload runners in benchmark-lab model concurrent request load as homogeneous traffic which hides emergent contention between high-priority short requests and low-priority long requests. This PR introduces a priority-conscious cartography simulator and a configurable background-noise injector to produce representative steady-state traces that surface kernel saturation and queueing interactions across priority classes.

What this PR does: 1) Implements PriorityConcurrencyCartographer (sim/runner) that can simulate tagged request classes with per-class arrival processes (Poisson, bursty, and Schedule-based) and enforce priority-preemptive or priority-nonpreemptive queue semantics. 2) Adds BackgroundNoiseInjector that can inject low-priority background work (I/O-bound emulation, long-seq model calls) with tunable intensity and correlation with high-priority load. 3) Adds steady-state reporters: priority_heatmap (per-priority p50/p95/p99 latency and token-throughput heatmaps), kernel_saturation_exporter (per-kernel occupancy and stall-time histogram), and a compact CSV/Parquet export that plugs into the existing reporting pipeline to S3. 4) Adds integration tests and a microbench harness that validates cartographer behaviour under deterministic seeds.

Checklist: - [x] Design + benchmarks for priority queueing model (ENG-3124) - [x] Implementation of cartographer + background injector - [x] End-to-end pipeline export to S3/LOCAL - [x] Unit tests and seeded stochastic tests - [x] Documentation and usage example in benchmark-lab/docs/runbooks/priority-cartographer.md - [x] Perf baseline upload and sample visualizations

Commits summary: - initial: add PriorityConcurrencyCartographer core simulation types and API - add: background noise injector and correlator - add: priority_heatmap reporter and kernel_saturation exporter - add: s3 export adapter and parquet writer - tests: deterministic seed tests and integration harness - docs: runbook + example orchestration script - fix: minor race in reporter flush on shutdown

Changed files (high-level): - benchmark-lab/runners/priority_cartographer.py - benchmark-lab/runners/noise_injector.py - benchmark-lab/reporters/priority_heatmap.py - benchmark-lab/reporters/kernel_saturation.py - benchmark-lab/exporters/parquet_s3_adapter.py - benchmark-lab/tests/test_priority_cartographer.py - benchmark-lab/docs/runbooks/priority-cartographer.md - benchmark-lab/examples/priority-mix-compose.yaml - benchmark-lab/ci/benchmark-ci.yml - benchmark-lab/ci/scripts/upload_baseline.sh - benchmark-lab/README.md - benchmark-lab/CHANGELOG.md

CI summary: New benchmark CI workflow (benchmark-ci.yml) runs the deterministic harness with a small schedule and compares generated baseline metrics against the committed golden files. All jobs passed on first run; a cached parquet writer was swapped to a smaller memory footprint after review.

Review conversation (high level): - Oliver Park: "Nice addition. Two asks: expose preemption toggle in the run config and add a lightweight mode that avoids parquet to speed local dev." 
- Maya Chen (author): "Added 'preemption_mode' flag to the top-level config and a 'dev_exporter' option that writes compact CSV; both are present in the latest push." 
- Ravi Singh: "Can we add a small assertion that prevents unbounded background intensity from being set by mistake? I've seen runs OOM with very aggressive noise profiles." 
- Maya Chen: "Added a sanity check with a 95th percentile cap and documented recommended ranges in the runbook." 
- Leah Martinez: "Exporter looks good; one nit on column ordering for parquet schema to match existing reporting consumers." 
- Maya Chen: "Reordered schema and added a compatibility test; updated release notes."

Performance notes: In our baseline comparison runs (3 seeds x 5 replicates) the new simulator reproduced a 12-18% higher p99 latency for high-priority short tasks when a correlated background intensity of 0.6 was enabled versus uniform noise — this surfaced a kernel-level stall pattern that the kernel_saturation exporter captures. Parquet exports average ~1.6MB per 10k events with the new compact schema.

Merge: Squash and merge. Merged by Maya Chen after approvals from Oliver Park and Ravi Singh.

Related work: several benchmark-lab PRs modeled background and bursty traffic; this one focuses on explicit priority semantics and exportable saturation diagnostics to help runtime engineers tune preemption policies and batching windows.
