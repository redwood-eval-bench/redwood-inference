# Introduce invocation diffusion map and alert attenuation

Motivation: long-tail cost and latency incidents often originate from subtle cross-invocation context propagation and noisy downstream stages. Existing per-invocation summaries and span rollups capture local signals but provide limited surface for exploring how token-cost and latency diffuse across an invocation graph. This PR introduces a lightweight "invocation diffusion map" that aggregates token-cost propagation and latency influence across call trees and integrates an "alert attenuation" system that dampens noisy low-confidence alerts while preserving SLO visibility.

What this does (single-string summary): Adds a dataframe-style rollup that computes per-invocation token-cost flow and representative latency influence scores, exposes a Grafana dashboard with an interactive diffusion map and heatmap layers, instruments a server-side attenuation controller that reduces alert fanout based on diffusion confidence and historical hit-rate, and provides a small simulator to validate attenuation behavior under bursty traffic.

Key behaviors and guarantees: - Diffusion map computes a directional, weighted graph where nodes are invocation sites (route+model+stage) and edges represent aggregated token-cost and p95 latency transfer over a rolling 15m window. - The attenuation controller applies a two-stage filter: (1) confidence scoring using diffusion weight + recent alert hit-rate, (2) dynamic attenuation factor that reduces alert propagation to downstream teams and auto-suppresses duplicates within a decaying window. - All computations are bounded by cardinality guards; expensive joins are approximated with reservoir samples and a lightweight Count-Min sketch for edge weights.

Why near-term: Reduces alert fatigue for platform + customer SREs by suppressing cascaded alarms from noisy prefetched prompts and transient kernel backpressure; helps root-cause by surfacing likely upstream cost drivers in the diffusion path. Linked Linear: ENG-1124 (diffusion map), PM-431 (alert tuning scope).

Testing and validation: - Unit tests for diffusion aggregator logic and attenuation policy (coverage for edge cases where nodes appear/disappear). - Integration test simulates bursty multi-stage invocations and asserts attenuation retains alerts for high-confidence upstream causes while muting low-impact downstream duplicates. - Local replay bench used to tune attenuation thresholds.

Backward compatibility and rollout: default attenuation is disabled for 2 weeks behind a feature flag and the dashboard is read-only until opt-in. Runtime configs are annotated in the Console for Dedicated/Private deployments so operators can adjust sensitivity per tenant.

Notes: this PR focuses on observability surface and runbook ergonomics; it intentionally defers automated remediation strategies to follow-ups (ENG-1189).
Priya Desai: Nice approach. Two asks before merge: 1) add a brief section in the runbook about when to disable attenuation for smoke tests, 2) include an operator-visible metric for 'alerts suppressed' per-tenant.
Jordan Lee (author): Added the runbook blurb and exported observability.metric.alerts_suppressed_count; see commit e5aa2b7.
Marco Ruiz: Attenuation logic looks good but the default thresholds feel aggressive. Recommend softening the default attenuation factor and adding telemetry to track false-negatives during the opt-in period.
Jordan Lee (author): Lowered the default attenuation multiplier to 0.6 and added 'attenuation.false_negative_ratio' gauge to the internal telemetry. Updated flags.yaml to make threshold and decay configurable.
Samir Patel: Integration tests are comprehensive. Suggest adding one extra case for churned node reappearance (node disappears then reappears with new id).
Jordan Lee (author): Added simulate_diffusion_test.go case for churned node reappearance (commit d4f9a11). CI run green afterwards.
Add invocation diffusion map (observability) and an opt-in alert attenuation mechanism to reduce cascaded alert fanout. Dashboard panels added. Feature off by default; operators can enable per-tenant. See runbook for tuning guidance.
Merged via squash on 2025-10-02. Merge note: feature gated and flagged off by default; operators and SRE will enable with canary tenants first. Backport not required.
Monitor attenuation.false_negative_ratio during canary window (2 weeks)
Enable alerts_suppressed_count dashboard panel for platform SREs
Follow-up: ENG-1189 to explore automated remediation hooks
