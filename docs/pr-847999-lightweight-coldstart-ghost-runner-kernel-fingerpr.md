# lightweight coldstart ghost-runner and kernel fingerprint reconciler

Motivation: we need a low-overhead harness to (1) reproduce cold-start micro-latency artifacts seen in noisy multi-tenant clusters and (2) tie those artifacts to per-kernel execution signatures across GPU families. This PR introduces two components: a lightweight 'ghost-runner' that injects deterministic, intermittent coldstart-like sequences into existing workload traces, and a kernel fingerprint reconciler that captures compact kernel execution signatures for later offline correlation.

What this changes: ghost-runner: a user-space benchmark runner that can replay a subset of sequences from the benchmark corpus with configurable primer sparsity and partial KV priming. It supports: deterministic seedable replay, per-request metadata tagging (ghost=true), an option to inject micro-bursts with configurable inter-burst jitter, and a CLI to export a compact run manifest. Kernel fingerprint reconciler: a small agent that hooks into the profiler probe points and emits kernel-level fingerprints (opcode hash, duration distro summary, occupancy hint) tied to the ghost-runner manifest id. A post-processor reconciles run telemetry and kernel signatures into an epochal report used by the perf dashboards.

Backward compatibility: no changes to runtime product code paths. All instrumentation is gated behind benchmark lab flags and the reconciler runs in a sidecar/prometheus-exporter mode for offline ingestion.

How to run locally: ./bench/ghost_runner/run.sh --manifest examples/manifests/mini-ghost.json --seed 42 --primer-sparsity 0.15 --burst-size 8 --output /tmp/ghost-out.json Then: python tools/reconcile_fingerprints.py --runs /tmp/ghost-out.json --profiles /tmp/prof-*.ndjson --out /tmp/epoch-report.json

Testing and verification: added deterministic unit tests for manifest parsing, a small e2e benchmark that runs on the CI ephemeral GPU job (approx 6 minutes), and offline reconciler smoke checks that validate signature stability across two representative kernels. The CI perf-check step is configured to run on the nightly perf runner for broader time series comparison.

Checklist: - [x] Unit tests - [x] Small e2e bench - [x] CI integration + job config - [x] Dashboard export schema update - [x] Documentation for running and interpreting reports

Rationale and tradeoffs: We intentionally keep the ghost-runner lightweight: it emits metadata and relies on existing tracer/profiler pipelines for heavy telemetry. The reconciler produces compact signatures (not full traces) to avoid storage blowup while still enabling root-cause correlation for tail events.

Lena Horvath: Can we make the primer-sparsity parameter a fraction or absolute count? Prefer fraction for portability across manifests.
Arjun Mehta: Updated manifest schema to accept 'primer_sparsity' as float in [0,1] and added validation tests.
Diego Ramos: I observed nondeterministic seeds on the ephemeral CI; looks like thread scheduling changed ordering. Please add an explicit single-threaded replay mode.
Arjun Mehta: Added --single-threaded flag and updated CI job to run the bench under that mode for the unit e2e.
Priya Kapoor: Reconciler emits a field 'kernel_hint' that overlaps with existing profiler tags; ensure namespaced export to avoid collisions.
Arjun Mehta: Namespaced reconciler fields under 'ghost_reconciler' and updated dashboard mapping.
Initial micro-bench on T4 and A100 showed a 0.8%-2.4% p95 tail shift when ghost sequences are interleaved with steady-state traffic; reconciler signatures matched a kernel variant on older driver stacks. Nightly runs will confirm stability.
Register epoch-report exporter in perf pipeline (done)
Schedule nightly ghost-run in perf lab (ticket ENG-5401)
Add reconciler signatures to kernel-attribution index (follow-up PR)
Key files to inspect: bench/ghost_runner/main.py for replay determinism, bench/reconciler/reconciler_agent.py for fingerprint format, and ci/jobs/ephemeral-gpu-bench.yml for reproducibility in CI. The reconciler intentionally emits compact summaries; if more detail is required we should gate it behind a verbose flag.
