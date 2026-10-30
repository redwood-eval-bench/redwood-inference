# Align compat routing and stream-degradation policy for fncall/tooling

Motivation: Several customers encountered noisy fncall streams where backend GPU preemption plus transient network loss caused duplicated events or stalled fncall handshakes when going through the OpenAI compatibility layer. This PR updates routing decisions in the gateway and slides in a deterministic degradation strategy that preserves structured fncall semantics while preventing long-lived stream stalls.

Summary of changes: 1) Route scoring: add stability-weight to route selection so compatible-but-flaky model variants are deprioritized for streaming toolcalls. 2) Degradation policy: new 'stream->chunked-poll' fallback that converts streaming toolcall responses into ordered chunks with explicit structured-fncall-complete events when streaming exceeds a latency/oom threshold. 3) Per-client edge buffer caps: enforce soft caps at the proxy layer and emit client_buffer_trim metrics; these prevent single noisy client sessions from blowing node memory and impacting other routes. 4) Payload limits: move route payload quotas into runtime config with a small, latency-optimized fast-path for small payloads (<8KB) and stricter enforcement for legacy OpenAI-compat endpoints. 5) KV-prefetch hints: add a light-weight prefetch hint in the fncall path to warm model KV for expected long-context calls, reducing the first-chunk latency on dedicated pools. 6) Compat mapping: OpenAI-compat translator now surfaces a structured 'fncall_fallback' event when a tool call is downgraded so SDKs can surface a clear actionable state.

Checklist: - [x] Add route selection tests and unit coverage - [x] E2E stream tests for degraded paths - [x] Observability dashboards and alert thresholds - [x] Docs snippet for compat behavior (compat/README) - [x] Manual smoke on dedicated cluster

References: Linear ENG-7219, incident SUP-4421, design doc: https://linear.app/rw/issue/ENG-7219/compat-degradation-design
Gateway: introduce route-level degradation policies that gracefully downgrade streaming to chunked-polling for unstable backends; OpenAI-compat: map fncall tool failures to structured fallback events; Observability: new metrics for compat_routing.degrades and client_buffer_trims;
Jonah Klein: Suggests adding a small cooldown on prefetched KV when the same client flips between routes rapidly — risk of thrash.
Author (Samira): added a per-client cooldown counter in edge_buffer.go and rebalanced prefetch TTLs (see commit e3c8f2b).
Priya Raman: Can we ensure the compatibility event schema won't break older SDKs?
Author (Samira): Translator emits the fallback as an additive event; old SDKs will ignore unknown event types. Added tests to prove backward compatibility.
Marco Silva: Integration tests flaky in k8s-ci on first run (time-based). CI rerun passed. Approved with minor nits.
Jonah Klein: approved — perf numbers look good on dedicated staging (p50 first-chunk down ~8%).
