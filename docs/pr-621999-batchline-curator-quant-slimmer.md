# Batchline Curator + Quant Slimmer: tenant-aware batching defaults and deploy-time quant trimming

Motivation: Many tenants run with overly conservative batching and wide quantization profiles, increasing per-token cost and GPU memory usage. This PR introduces a new component (Batchline Curator) that computes tenant-aware batching/caching defaults and a small deploy-time tool (Quant Slimmer) that suggests a minimal, safe quant profile per model family based on recent traffic and eval traces. The goal is to reduce unit economics by default while providing telemetry-driven rollback and safety checks.

What this PR does (high level):
- Add batchline-curator service (lightweight scheduler) that emits tenant default suggestions and a ranked list of safe batch shapes. Suggestions are written to configs/presets/batchline-defaults.yaml and exposed via the admin API.
- Add quant-slimmer CLI (rwprefill quant-slimmer) that runs during model onboarding or rollout to propose narrower quantization parameters; integrates with the existing model-registry/quality checks.
- Wire telemetry: new metrics (batchline.suggester.* and quant.slimmer.recommendation.*) and traces to Redwood Console to allow rollback if quality regressions are observed.
- Safety: multi-stage rollout — suggestion -> shadow-simulate -> canary -> apply. Automatic rollback triggers on eval set drift or QA alert counts.
- Docs: quickstart and a troubleshooting guide in redwood-docs/docs/ops/batching-and-quant.md.

Checklist: 
- [x] Design doc: Linear ENG-9407 (link: https://linear.app/redwood/issue/ENG-9407/batchline-curator-design)
- [x] Unit tests for core suggestion logic
- [x] Integration tests: perf-canary scenario added
- [x] E2E smoke on staging
- [x] Docs updated

Compatibility notes: The Batchline Curator writes opinions as tenant defaults that can be overridden by tenant config. Quant Slimmer outputs candidate profiles; we do not apply any quant changes automatically without explicit rollout approval.

Rollout plan: staged on staging -> internal dedicated tenants -> opt-in beta customers -> default for newly created tenants after 2 weeks of monitoring.

Related work: ties into ENG-8310 (tenant-prefill heuristics) and PRs that added telemetry-driven rollbacks.

1) feat(batchline): add curator skeleton and API (pri: 68a7b2c)
2) feat(quant): implement quant-slimmer CLI and lightweight analyzer (pri: d4f3e89)
3) perf: batch shape ranking and cost estimator (pri: b2c4a31)
4) test: add unit+integration tests for suggestion engine (pri: 92f1a4e)
5) docs: ops guide and rollout checklist (pri: c7e0b12)
Reviewer: Alex Martinez — Please clarify the rollback thresholds and whether we have per-model family calibration.
Author (Priya): Added thresholds to metrics.yml and included per-family calibration map stored in model-registry under 'quant_profiles/meta'.
Reviewer: Samir Gupta — Can the quant-slimmer produce a conservative fallback automatically when an explicit profile is missing?
Author: It now emits a 'fallback' profile which must be applied by a human; auto-application is behind a feature flag.
Reviewer: Elena Rossi — Tests flaked on CI twice (integration perf-canary).
Author: Fixed timing race in the perf scenario and added retry logic; CI green on rerun.
Approvals: Alex Martinez (approved), Samir Gupta (approved after changes), Elena Rossi (approved).
