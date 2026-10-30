# Epochal shadow reservoir with traffic reverberation and autoscale mitigation

Motivation: Production shadow traffic often creates transient load spikes during promotion rehearsals and phased seeding. Those spikes can cause autoscalers to misinterpret demand and overprovision, increasing cost and masking regression signals. This PR implements an "epochal shadow reservoir": a lightweight in-memory reservoir sampler and traffic reverberation buffer that smooths shadow bursts, preserves fidelity for replayable smoke, and exposes autoscale-aware heuristics so the inference autoscaler treats shadow rehearsal load as a first-class signal with damped impact.

Design summary: The feature attaches to the shadow ingress path and groups shadow traffic into short-lived epochs (configurable, default 10s). Each epoch keeps a capped reservoir per-model-cohort and emits a controlled, rate-limited replay stream to backend replicas for rehearsal and telemetry. Replays are tagged as rehearsal=true and carry an epoch_id to support deterministic sampling and subsequent bisect/replay. An autoscale adapter publishes a damped load metric (shadow_effective_rps) that is weighted by a tunable damp factor and a decay window. The design aims to: 1) prevent autoscaler overreaction to instantaneous shadow spikes, 2) ensure stable, reproducible rehearsal streams for evaluation, and 3) provide telemetry hooks for signal-level diagnostics and causal attribution.

What changed (high level): added epochal reservoir and replay pipeline; introduced autoscale adapter metric and config knobs; plumbing for rehearsal tagging and epoch provenance; unit and integration tests simulating spike profiles; docs and migration notes; small SDK shim to expose epoch_id in audit logs.

Commits summary: 1) add epochal reservoir core and interfaces; 2) implement epoch grouping and reservoir sampling strategy; 3) replay dispatcher with rate limiter and backoff; 4) autoscale adapter and metric exporter; 5) tests (unit + integration spike harness); 6) docs and config defaults; 7) address reviewer feedback and flake fixes.

Files changed (summary): see changed_files.

Rollout plan: land behind runtime flag `FEATURE_EPOCHAL_SHADOW` default off. Stage rollout in three steps: dev staging (flag on, no autoscale adapter), canary (small percentage of shadow traffic to reservoir, adapter in read-only mode), and progressive promote (enable replay and adapter for monitored clusters). Follow-playbook: monitor shadow_effective_rps, rehearsal success ratio, and autoscale decisions for 48h. Canary guardrails are included in the config and console charting.

Risks & mitigations: incorrect damp factor may hide real load; mitigation: conservative default (0.25) and canary-only initial enablement. Reservoir memory leak risk: bounded caps and epoch eviction implemented; validated via integration memory profiles.

Related work and tickets: ENG-54321 (cohort replay infra), PM-234 (canary observability), INT-88 (incident follow-up for autoscale overprovisioning).
Ethan Cole (2026-02-14): Nice design. A few asks: (1) please justify damp_factor default and add a Helm override, (2) ensure epoch_id is present in audit traces, (3) add e2e scenario where autoscaler sees both production and rehearsal streams.
Priya Nambiar (2026-02-15): Addressed: added helm value `epochalShadow.dampFactor`, surfaced epoch_id in SDK shim and audit events, added integration test simulating combined streams. PTAL.
Marina Lopez (2026-02-16): Request small change: make reservoir caps per-cohort configurable via runtime config and not compile-time constant. Also noted a potential N+1 metric export — tracked and fixed in the fixup commit.
Priya Nambiar (2026-02-17): Updated config to make caps tunable and optimized metric aggregation to avoid N+1. Added unit test to assert no duplicate exporter registrations.
Sasha Ivanov (2026-02-18): Approve after small comment about doc phrasing. Left an approval: 'LGTM with doc nits'.
Marina Lopez (2026-02-19): Approve — observed integration run is stable and autoscaler behaved within expected bounds in staging run.
Conservative rollout defaults retained; feature flag gating required. Approvals from core SRE and runtime reviewers. CI green after fixups. Proceed with staged rollout as documented.
Introduce epochal shadow reservoir to smooth shadow traffic spikes, provide damped shadow_effective_rps metric for autoscaler, and add replayable epoch provenance. Feature disabled by default; enable via FEATURE_EPOCHAL_SHADOW or Helm value epochalShadow.enabled.
If autoscaler starts scaling unexpectedly after enabling, set epochalShadow.enabled=false and observe shadow_effective_rps metric. Relevant dashboards: epochal-shadow. Alert hooks: autoscale-unexpected-surge (watch shadow_effective_rps > production_rps * 0.5 for 5m).
1) Enable FEATURE_EPOCHAL_SHADOW in staging. 2) Run tests/integration/test_replay_spike_profile.py against staging cluster. 3) Validate console dashboard values and check autoscaler logs for damped contributions. 4) Run smoke traffic with and without reservoir to compare autoscale behavior.
This PR avoids changing production default behavior. Follow-up work proposed: persistent epoch store for long-running replays and sampled persisted traces for offline bisecting (tracked in ENG-56000).
