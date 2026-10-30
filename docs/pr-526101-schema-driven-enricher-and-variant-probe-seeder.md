# Introduce schema-driven metadata enricher and variant probe seeder

Context: In practice, a significant fraction of inbound model uploads omit non-required intake fields (hardware_affinity, seq_distribution, licensing). That reduces the registry's ability to generate useful performance profiles and automated rollout recommendations. This PR introduces a lightweight, schema-driven metadata enricher and a companion variant probe seeder to produce initial performance seeds for downstream tooling.

What this PR does:
- Adds a schema-driven enricher that inspects uploaded manifests and artifact contents (model weights, tokenizer artifacts, config blobs) and auto-populates conservative but useful defaults plus provenance tags. The enricher writes a `registry_metadata` annex that downstream services can opt into.
- Introduces a variant probe seeder that schedules short, bounded probe jobs (sampled sequences, cold/warm KV cache runs, single/generic batch sizes) for each model variant to generate a small performance sample used to create an initial PerformanceProfile. These probes are intentionally low-cost (<= 30s) and asynchronous.
- Adds an autogen release-note stub generator that consumes probe outputs to create a short performance blurb (median latency, 95th, tokens/sec) and includes it in the onboard review checklist.

Motivation: The goal is to make onboarding higher fidelity without asking model authors to provide more manual metadata. Seeded perf profiles improve Recommend/Optimize suggestions (quantization/batching) and reduce iteration between teams.

Implementation notes:
- The enricher is implemented as `onboard/enricher.py` (pluggable pipeline stage); schema lives in `schemas/model_intake_schema.yaml`. Enrichment rules are conservative and non-destructive: we only fill missing fields and attach `provenance.enricher=v1` to the metadata.
- Probe seeder lives in `onboard/probe_seeder.py` and uses the existing lightweight harness (new `harness/probe_runner` module) to run short jobs. Probe results are stored under `perf_seeds/<model_id>/<variant>/seed.json`.
- Release-note autogen templates are under `docs/onboarding/release_blurb_templates.md` and produce a single-line blurb attached to the model intake as `autoblurb.v1`.

Checklist:
- [x] Add enricher pipeline stage and schema
- [x] Add probe seeder and lightweight harness integration
- [x] Unit tests for enrichment heuristics and probe result parsing
- [x] Integration test that runs seeder in CI matrix (short mode)
- [x] Docs and onboarding guide updates
- [x] Migration/rollback plan included

Testing and CI:
- Unit tests added under `tests/test_enricher.py` and `tests/test_probe_seeder.py` (fast, deterministic).
- Integration matrix includes a short-run probe job (runs under 45s in CI). To reduce flakiness, probe runner uses a local deterministic RNG seed and a small synthetic input.

Rollout plan:
- Canary: enable enricher+seeder for 10% of new onboard jobs (flagged) and monitor ingestion error rate + probe failure rate for 48h.
- Full rollout: after successful canary, enable for all new onboards. Existing models will be backfilled via a one-off worker (`tools/backfill_perf_seeds`) that will be run by platform infra during the agreed maintenance window.

Migration notes:
- The enricher appends `registry_metadata` only when fields are missing; existing metadata remains unchanged. Backfill is optional but recommended to seed older models.

Security/privacy notes:
- Probe payloads are synthetic and do not include customer data. Probe jobs run in the same sandbox as existing tests and respect per-tenant isolation.

Related: ENG-8113 (schema-driven metadata); PM-293 (onboard UX for perf seeds).

Adds automated metadata enrichment during model intake and seeds per-variant probe profiles for performance baselines. Introduces no public API changes; runtime and Optimize will consume new artifacts.
Aaron Cole (2025-01-16): Nice direction. Two requests: 1) add an opt-out flag at intake level so customers can disable probes, 2) we need a migration plan for existing models (backfill cost estimates).
Sofia Martinez (2025-01-16): Added opt-out flag (onboard.enrich.disable_probes) and included a backfill tool + cost estimate in PR description. Backfill is optional and rate-limited. Thanks!
Liang Zhao (2025-01-17): Probe runner looks good but CI had a flake on the integration run. Can we use a deterministic seed for the synthetic inputs?
Sofia Martinez (2025-01-17): Updated probe_runner to accept RNG seed via env var and wired CI to use a fixed seed. Reran integration matrix.
