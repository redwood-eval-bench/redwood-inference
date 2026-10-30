# Longitudinal fidelity runner with adaptive kernel probing and GPU memory safety

Motivation: We need a repro-friendly runner that simulates long-running production inference workloads while exercising kernel selection and memory-pressure edge cases. Current bench runners focus on short bursts or single-phase warmups and miss gradual drift in sequence mixes and KV-cache pressure that lead to late regressions. This PR introduces a longitudinal fidelity runner and an accompanying reporting pipeline to capture phase transitions, kernel selection traces, and GPU memory-safety events.

What this PR adds (high level):
- A new runner: bench/longitudinal_runner.py (config-driven, supports sequence-mix schedules, time-decayed priming, and cohort splitting).
- Adaptive kernel probe: runtime/probes/kernel_probe.cc + probe harness that issues benign synthetic micro-requests that trigger alternate kernel paths without changing model weights. Probe backs off if it detects memory fragmentation or OOM proximity.
- Memory-safety monitor: tools/telemetry/gpu_memory_watcher which annotates traces with allocation failures, fragmentation heuristics, and resident set samples.
- Reporting pipeline: reports/longitudinal_exporter (CSV + compact flame segments) and an HTML quicklook summarizer for offline triage.
- New test fixtures and CI job: ci/workflows/longitudinal-bench.yaml that runs a fast smoke run in CI and stores artifacts.

Design notes:
- Runner is config-first: YAML schedule with phases (steady, drift, spike, decay) and a stochastic token-length generator to emulate nonstationary seq distributions.
- Prober is conservative: probes run on a separate worker thread group, honor runtime SLO settings, and use a kernel-candidate list derived from the model profile. Probes are annotated in traces as 'probe.'
- Memory watcher uses a lightweight sampling policy to avoid perturbing the system; it prefers resident set sampling and CUDA meminfo when available.

Checklist:
- [x] Unit tests for schedule parser and token generator
- [x] End-to-end smoke benchmark that produces CSV output
- [x] Kernel probe safety backoff verified in local stress test (no OOMs triggered)
- [x] CI pipeline added and green on main smoke run

Why different from existing runners: This focuses on longitudinal behavior (hours-scale simulated via compressed replay), adaptive kernel probing tied to observed kernel selection drift, and explicit GPU memory-safety annotations to help correlate late failures with kernel choices and sequence-mix drift.

Potential follow-ups:
- Integrate probe results into model-routing policies to enable automatic fallback to conservative kernels when drift is detected.
- Add per-cohort QoS budgeting to measure economic impact of probe traffic.

Diego Ramos: 'Nice shape — I like the conservative probe approach. Can we reduce probe privilege to avoid touching the KV cache directly?'

Aisha (author): 'Probe traffic uses read-only prefix patterns and never injects priming that would mutate KV cache entries; added explicit assertion in probe harness and unit test to ensure no writes.'

Linus Park: 'Memory watcher looks good; can we expose fragmentation metric as a tag on traces so alerting can use it?'

Aisha: 'Added fragmentation tag in the exporter and a minimal histogram metric. See reports/longitudinal_exporter.py changes.'

Maya Bennett (approval): 'Approach validated locally on p3 and p4 hardware. Approving for merge.'
Introduce longitudinal benchmark runner with adaptive kernel probing and GPU memory-safety telemetry. Adds CI smoke coverage and reporting exporter for longitudinal traces.
