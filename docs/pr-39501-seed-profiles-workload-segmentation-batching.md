# Seeded workload segmentation and conservative defaults for batching, caching and quant profiles

Motivation: We observe that one-size-fits-all defaults for batching, prefix caching, and quantization lead to suboptimal unit economics for many customers (small chat workloads and high-concurrency APIs). This PR seeds the platform with a compact set of workload-segment profiles and conservative defaults so new customers and freshly-provisioned routes get safer, cost-effective behavior out-of-the-box while still enabling advanced autotuners to adjust later.

What this PR does: Adds a workload-segmentation YAML that encodes three seed profiles (latency-sensitive, throughput-friendly, conservative-cost). Wires a lightweight selector into the defaults layer so newly created model routes get an initial configuration (batch, cache_ttl, quant_profile_hint) based on estimated 95th-percentile seq_len and QPS. Introduces a small runtime hook that logs the chosen profile (for observability) and a migration that backfills unset routes with the conservative-cost profile. Updates docs and console copy to explain rationale and how to override.

Key benefits: Faster time-to-good-unit-econ for new routes, fewer support tickets asking for manual tuning, clearer migration path from conservative defaults to autotuned profiles, and safer behavior in Dedicated/Private where surprise perf regressions are costly.

Risk and scope: Conservative seed profiles intentionally bias toward lower cost (slightly higher latency for long tail requests) to avoid over-provisioning. This is a config-and-docs-first change; no kernel/runtime algorithm changes. Feature gated behind a rollout flag (defaults.seedProfiles.enabled) and opt-out via route config.

Testing and validation: Unit tests for selector logic and YAML parsing added. Integration test simulates route creation with synthetic QPS/seq_len signatures and asserts expected default assignments. E2E perf runs on benchmark-lab show median token cost reduction of ~9% for newly created chat-like routes using the conservative profile. CI: all checks green after one flaky lint failure was fixed.

Rollout plan: feature flag enabled for staging immediately, gradual rollout to 10% of new hosted routes next week, 100% after telemetry and support signoff. Dedicated/Private customers will keep existing defaults unless operator flips the seedProfiles flag.

Files and high-level changes: See files_changed for details.

Related: ENG-6201 (seed profiles design), ENG-6217 (console copy and telemetry events).
Connor Lee: Nice move to default to conservative-cost for new routes. Two asks: add a short metric that counts how many routes have the seed defaults vs user-supplied defaults, and ensure the migration is idempotent for multi-region runs.
Author (Sofia): Added defaults.seed_assigned metric and made migration guarded by a safe idempotent WHERE clause; updated tests to validate metric emission.
Aisha Patel: Can we add a short note in the console explaining why latency-sensitive customers may want to opt into autotune? Also flagged a small nits in console copy.
Author (Sofia): Updated console copy and added an FAQ entry in docs/guides/seed-profiles.md.
Marta Ivanov: CI was failing on a lint rule; after fix I reran the benchmark job and the cost numbers look reasonable. Approve pending green CI.
CI Bot: All checks passed (unit, integration, lint, bench-smoke).
Introduce workload seed profiles and conservative defaults for new routes; feature gated and opt-out capable. Reduces unit token cost for chat-like routes by default; operators can opt into autotune for lower latency.
