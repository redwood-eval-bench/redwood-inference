# add-unit-econ-reporter-cli-and-quant-presets-exporter

Motivation: customers asked for a lightweight, local-first tool to surface unit-economics signals for Redwood-hosted workloads and to export quantization profile suggestions from observed inference traces. This PR adds a small CLI (redwood-econ) and SDK helpers to produce tenant-level CSV heatmaps, sample-weighted unit-cost estimates, and an opinionated exporter for quant-presets (candidate quantization/bitwidth recommendations) so downstream ops teams can run audits and seed Dedicated/Private defaults.

What this PR delivers: 1) python/redwood/econ/reporter.py: core reporter that ingests request/trace samples and produces per-tenant unit-econ CSV and JSON summaries; supports prefix aggregation, warm/cold split, and per-model shape buckets. 2) python/redwood/econ/quant_export.py: exporter that synthesizes quant-presets from token-cost curves and per-shape loss estimates; includes conservative fallback rules and a compact YAML output compatible with on-prem deploy tooling. 3) CLI entrypoint (bin/redwood-econ) to run local dry-runs, simulate tenant rollups, and export artifacts. 4) integration tests and fixtures (tests/test_econ_reporter.py, tests/fixtures/*) and minimal docs (docs/econ-reporter.md) with examples and recommended thresholds.

Design notes: the reporter is intentionally sync-first and memory-bounded (streaming aggregator) to be runnable in dev environments. Quant exporter applies a three-stage filter: safety thresholds, sample-weighted cost delta, and a stability window to avoid noisy recommendations. We avoided changing runtime codepaths to reduce risk; SDK helpers are pure computation and optional.

Checklist: - [x] Unit tests for aggregator and exporter logic - [x] CLI basic e2e smoke - [x] Docs and examples - [x] Telemetry hooks (opt-in) - [x] Manual smoke on staging with a tenant snapshot (ENG-3421)

Usage: redwood-econ summarize --input traces.ndjson --tenant tenant_123 --out tenant_123_econ.csv
redwood-econ export-quant --input traces.ndjson --out quant-presets/tenant_123.yaml

Compatibility and rollout: Non-breaking. The CLI and SDK live in the python package; no API surface change for hosted inference. We will ship as a minor SDK release and recommend ops teams run the CLI against 24-72h trace slices to seed quant-presets.

Related: ENG-3421 (unit-econ UX), ENG-3495 (quant export safety rules).
Aisha Patel (reviewer): Nice direction — can we tank the memory usage on pathological tenants? Suggest switching the in-memory shape-bucket map to an LRU capped at 10k keys or streaming to disk after threshold. Also request a small sample of a produced quant YAML in the PR body.
Author (Marina): Addressed — switched to a capped LRU map with a configurable cap via env var REDWOOD_ECON_BUCKET_CAP. Added an example quant YAML (docs/econ-reporter.md). Updated tests to cover boundary evictions.
Liam Chen (reviewer): Tests look good. Two small nits: prefer explicit decimal formatting for cost deltas and add an opt-out flag for telemetry hooks for compliance customers.
Author (Marina): Added cost formatting and --no-telemetry flag to CLI; updated docs and changelog note.
Ruben Alvarez (approver): Ran a staged dry-run against a 48h snapshot from staging-qa. Output looks reasonable; quant presets were conservative and matched our heuristic baseline. Approving for merge after CI passes.
Added a lightweight SDK CLI (redwood-econ) to generate tenant unit-economics reports and export candidate quantization presets. Useful for ops and cost-analysis workflows; non-breaking SDK addition. See docs/econ-reporter.md for examples and recommended thresholds.
