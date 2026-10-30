# phase-shifted-latency-sweep-and-coldstart-prober

Motivation: We need a lightweight, reproducible runner that exercises both steady-state and phase-shifted burst patterns while simultaneously measuring cold-start latency across different model variants and runtime kernel configs. This PR introduces a phase-shifted latency sweep runner and a paired cold-start prober that can be composed to produce joint distributions (latency vs cold-start tail) used by the regression detector pipeline.
Design:
- phase_sweep_runner: Python runner that generates a sequence of workload phases with configurable duration, RPS, sequence length distribution and an adjustable phase-offset parameter to produce controlled interleaving across client shards.
- coldstart_prober: small Go probe that issues warmup-less single-shot requests on a configurable model selection (quantized/FP16/FP32) to capture cold-start percentiles under concurrent load.
- reporter bridge: new CSV+JSON reporter that tags samples with phase-id, shard-id, kernel-profile and coldstart-epoch for downstream aggregation.
- CI harness changes: bench job integration and a perf-regression threshold test that runs an abbreviated sweep in CI with synthetic small-models.
How to run: bench/README contains examples; typical run: python bench/runners/phase_sweep_runner.py --profile profiles/low_mem.yaml --phases 6 --phase-duration 30s --phase-offset 7s --out reports/run-<ts>.csv
The coldstart probe is compiled with go build and invoked alongside the runner to annotate early-epoch samples.
Implementation notes:
- Uses existing bench client abstraction for request generation and adds a deterministic pseudo-random seed to make runs reproducible.
- Phase IDs are encoded into the request metadata header X-Bench-Phase so runtime traces can correlate tokens to phase.
- Reporter writes a compact JSON lines stream and a summary CSV compatible with the regression detector.
- Adds a small integration test that validates phase interleaving logic and a smoke perf check for the probe binary.
Commits:
- "initial phase_sweep_runner implementation, core API and simple CLI"
- "add coldstart_prober (go) and CI smoke job"
- "reporter: csv + jsonlines bridge and header tagging"
- "docs: bench user guide and example profiles"
- "fix: deterministic seeding + durable temp file cleanup"
Files changed (high level):
- bench/runners/phase_sweep_runner.py (new): core runner, phase scheduler, sampling hooks
- bench/probers/coldstart_prober/main.go (new): standalone probe binary
- bench/reporters/csv_json_bridge.go (new): reporter bridge for CSV/JSONL outputs
- bench/configs/profiles/phase_sweep_low_mem.yaml (new): example profile
- integration/test_phase_sweep.py (new): lightweight unit/integration checks
- docs/bench/phase-sweep.md (new): usage and interpretation
- ci/bench.yml (modified): adds perf sweep smoke job
Reviewers requested small API changes (explicit --seed flag, clearer exit codes for the probe) and asked for a more conservative default for phase-offset to avoid self-synchronization artifacts on noisy CI hosts. I iterated on the runner CLI, added --seed and made the default phase-offset randomized within a small jitter when running in CI mode.
Tests: unit tests for scheduler logic and an integration runner simulation were added.
CI summary:
- build: pass
- unit-tests: pass
- integration-tests: pass
- bench-smoke: pass (flaky on first run; retried once)
- perf-regression-check: pass (abbreviated)
Merge: squashed and merged to main.
Notes on impact and usage:
- This enables the benchmark lab to capture correlated latency tailing behavior during workload phase transitions and quantify cold-start exposure for short-lived containers.
- New CSV schema is backwards-compatible with the regression detector after a small mapping layer; a follow-up will add direct ingestion.
Checklist:
- [x] Runner implementation
- [x] Coldstart probe
- [x] Reporter bridge and CSV schema
- [x] CI integration
- [x] Docs and examples
- [x] Tests and smoke checks
Related: ENG-4821 tracks adding cold-start monitoring to the nightly suites; ENG-4890 is a follow-up to integrate these runs into the SLA dashboard.
Ethan Park: Left a couple of comments on CLI ergonomics. Please add --seed flag and ensure probe exit codes are in [0,1,2].
Priya Shah: Added explicit --seed and updated probe to return 2 on misconfig, 1 on transient error, 0 on success.
Aisha Malik: Can we make the default phase-offset smaller on CI to reduce spurious contention? Noted flaky behavior on small hosts.
Priya Shah: Added randomized jitter and CI-only conservative defaults, documented in docs/bench/phase-sweep.md.
Jonah Reed: Approve after one nit (typo in README).
