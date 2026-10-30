# runtime: export KV cache stats + prefill/decode token accounting for cost model

Problem: Cost Model v1 needs stable, runtime-sourced inputs for (a) per-stage token accounting (prefill vs decode) and (b) KV cache behavior to correctly model cache hits, cache reuse, and batch efficiency. Today we only emit total tokens + ad-hoc cache counters in a couple of codepaths, which diverges from what Console/Optimize/billing need and makes drift analysis noisy.

This PR adds runtime-side instrumentation and export for KV cache stats and per-stage token accounting. The exported payload is designed to be: (1) low-cardinality, (2) deterministic for a given request execution, and (3) compatible with streaming + speculative decode disabled/enabled paths.

What changed: - Introduced a request-scoped "TokenAccounting" struct that accumulates tokens_processed_prefill and tokens_processed_decode. - Added KV cache counters captured at the scheduler boundary (before execution) and at completion: kv_cache_prompt_tokens_hit, kv_cache_prompt_tokens_miss, kv_cache_prefix_bytes_hit, kv_cache_prefix_bytes_miss, and kv_cache_evictions. - Emitted a single unified telemetry event "runtime.inference_accounting.v1" with stage token counts + KV cache stats, keyed by request_id and model_execution_id (NOT by prompt hash or cache key).

Semantics (v1): - Prefill tokens = tokens processed in the prefill phase for the chosen execution graph (includes prompt tokens after any truncation). - Decode tokens = tokens processed in decode iterations (includes all generated tokens, including EOS token if produced). - KV cache prompt token hit/miss are measured relative to the runtime's prefix-cache lookup result for the request (token-level units), and are emitted even when cache is disabled (0/total).

Bounds/perf: - No per-layer metrics, no per-head, no per-cache-entry output. - Event payload is bounded; no arrays. - Counters are computed from existing internal bookkeeping; no additional GPU sync points are introduced.

Testing: - Added unit tests for token accounting across streaming and non-streaming paths. - Added an integration test fixture that runs a tiny model with cache enabled and verifies hit/miss invariants (hits+misses == prefill tokens after truncation). - Added a debug-only assertion (guarded) to catch negative counters during development.

Rollout: - Gate emission behind feature flag RUNTIME_EXPORT_ACCOUNTING_V1 (default off). - Once telemetry pipeline is ready, we will enable in staging for 5% and validate drift dashboards before prod.

Notes / follow-ups: - This PR intentionally does not rename existing legacy metrics; Telemetry will map these new fields and then we can deprecate the old ones (see ENG-4823). - Speculative decode: we currently attribute speculative draft tokens to decode tokens_processed_decode; we do not emit a separate speculative bucket in v1.

Checklist: - [x] Metrics names reviewed with Telemetry - [x] Payload bounded (no high-cardinality tags) - [x] Backward compatible (flagged) - [x] Added tests + fixture

Risk: Moderate. Touches request execution accounting and telemetry emission; guarded by feature flag and includes invariants in tests.
Sofia Ivanova (review): Can we confirm the counters are taken post-truncation? Cost model needs consistency with what actually ran. Also, is the scheduler boundary the right place for cache hit/miss measurement given fallback paths?
Omar El-Sayed (author): Yes—prefill tokens are counted after truncation (same value used for the actual execution). For cache hit/miss: we record lookup result after routing picks the final execution plan, before running the graph. If we fall back (e.g., cache disabled for the run), counters emit 0/total and we set kv_cache_enabled=false in the event.
Nadia Rahman (review): Please explicitly define whether decode includes EOS. Billing currently includes it in generated tokens; Console expects same. Also, please avoid emitting prompt_hash or cache_key to keep this low-risk for PII.
Omar El-Sayed (author): Updated semantics: decode includes EOS if produced. Confirmed no prompt hash/cache key in the event; only request_id + model_execution_id which are already in telemetry.
Caleb Johnson (review): Telemetry pipeline prefers stable field names and units. Can we align token units (int) and bytes (int) and avoid floats? Also, name suggestion: kv_cache_prefix_bytes_hit/miss instead of just bytes_hit/miss.
Omar El-Sayed (author): Done. All counters are u64 serialized as int. Renamed to kv_cache_prefix_bytes_hit/miss and updated event schema + tests.
Grace Kim (review): How does this behave with quantized kernels where we may fuse prefill? Is stage detection reliable or will prefill/decode be ambiguous?
Omar El-Sayed (author): Stage is derived from the high-level execution loop (prefill call vs decode step calls). Even if kernels fuse internal ops, we still know which invocation is prefill vs decode. Added a comment in engine.rs clarifying this assumption.
Thomas Baird (review): Please double-check payload size and cardinality. No per-batch arrays. Also ensure we don't emit per-token timing in this event—Console hot path can't handle high volume of large events.
Omar El-Sayed (author): Payload is fixed fields only (no arrays). No timing fields included here; queue time remains in existing scheduler event. Added a size assertion in the schema test to catch accidental growth.
Sofia Ivanova (approve): Looks good with the rename and clarifications. Please keep speculative decode as-is for v1; we can add a separate bucket later if needed.
Nadia Rahman (approve): Approved. Thanks for documenting EOS behavior and ensuring no cache-key leakage.
Caleb Johnson (approve): Approved once staging flag default is OFF. Confirmed in flags.yaml.
Internal only: runtime now can emit bounded KV cache hit/miss + prefill/decode token accounting event (runtime.inference_accounting.v1) behind feature flag RUNTIME_EXPORT_ACCOUNTING_V1 for Standard Cost Model inputs.
