# Add L40S memory guardrails and KV cache budgeting

Context: During the workload-matrix runs on L40S runners we saw intermittent OOMs and allocator fragmentation spikes when long-context prompts coincided with bursts of concurrent prefill. The existing KV cache sizing logic assumes H100-class HBM headroom and tends to over-commit on L40S (48GB) when queue depth is high.

This PR introduces L40S-focused memory guardrails and KV cache budgeting defaults that are safe-by-default and intended to stabilize tail latency and avoid OOM/retry storms. The changes are gated behind a runtime toggle so we can roll back per pool/region if needed.

What’s in this PR (high level): (1) GPU-SKU-aware defaults for KV cache budget on L40S (target utilization and hard cap), (2) admission/backpressure hooks that avoid starting prefill when projected KV reservation would exceed budget, (3) periodic logging + metrics for effective KV budget and rejection/deferral counts, (4) explicit rollback toggle to revert to prior behavior.

Non-goals: This does not add new kernels or change attention implementations. It also does not attempt to perfectly predict memory usage for every model; it aims to prevent the worst-case over-commit scenarios and provide oncall-friendly observability.

Rollout plan: Enable in L40S canary pool for 24h (workload matrix + production traffic), then expand by region. If we observe increased deferrals / latency above SLO (p95) or throughput regression beyond agreed thresholds, disable via the feature flag and fall back to legacy sizing.

Runtime logs (example, new):
- [kv_budget] gpu_sku=L40S total_mem_bytes=51539607552 reserved_os_bytes=2147483648 target_util=0.72 hard_cap_bytes=34359738368 computed_budget_bytes=33171271680
- [kv_budget] enforcement=on projected_kv_bytes=35651584000 budget_bytes=33171271680 action=defer request_id=... route=chat.completions
- [kv_budget] enforcement=on deferred=37 rejected=0 avg_deferral_ms=18 p95_deferral_ms=44 window=60s

Rollback toggle:
- env: REDWOOD_DISABLE_L40S_KV_GUARDRAILS=1 (forces legacy behavior; still emits a one-line warning at startup)

Testing: unit tests for budget computation, integration tests around config/override behavior, and a targeted stress test that reproduces the prior OOM pattern using the workload-matrix harness settings for long-context buckets.

Notes for SRE/oncall: the new metrics are tagged with gpu_sku and profile selection. See added runbook comments in code for the expected alert interpretation (high deferrals usually means concurrency too high for the chosen context length distribution; high rejects should remain ~0 except under pathological load).
Omar El-Sayed (review): Can you clarify what happens when we exceed budget—do we reject or defer? I’m worried about turning OOMs into user-visible 429/5xx.
Samir Qureshi (author): Good call. Default behavior is deferral (bounded wait) for prefill start; reject is only used if the request is already past max_queue_ms or if queue is saturated and we’d violate backpressure limits. In practice, for our L40S settings rejects should stay ~0; the point is to smooth bursts.

Sofia Ivanova (review): Admission control changes can shift tail latency. Do we have knobs to keep streaming stable? Also, is the projection model aware of streaming decode vs non-streaming?
Samir Qureshi: Projection is conservative and based on max_new_tokens reservation + context length bucket. It doesn’t distinguish streaming at the KV level (KV is needed either way), but streaming stability benefits from avoiding OOM/retry storms. I added metrics so we can correlate deferrals with p95/p99 by route; also added a per-route override hook (not exposed yet) so we can tune queue limits independently if needed.

Ethan Park (review): Please ensure defaults only apply when detection is confident. Also add explicit startup logging of which path was taken (L40S defaults vs legacy vs override).
Samir Qureshi: Addressed in latest push. We only apply L40S defaults when gpu_sku_normalized == "L40S" and vram_bytes is within an expected range. Otherwise we fall back to legacy sizing and log a warning with detection details.

Sean Gallagher (SRE review): Need a fast rollback and clear signals. Can we get a single env var to disable the whole behavior and still keep the logs? Also what are the suggested alert thresholds?
Samir Qureshi: Added REDWOOD_DISABLE_L40S_KV_GUARDRAILS=1 (forces legacy). Logs remain but include enforcement=off. For thresholds: deferrals can be >0 during bursts; sustained deferral_rate > 5% for 10m likely indicates mismatch between concurrency and context distribution; rejects should be <0.1% sustained. I added comments near the metric definitions; runbook updates are coming via the rollout doc.

Rina Kobayashi (perf): Confirmed the previously failing long-context (16k/32k bucket) matrix run no longer OOMs on L40S. Throughput regression within 1–2% in the steady-state; p95 improved in the pathological burst case because we avoided retries.

Keiko Yamamoto (infra): Any interaction with autoscaling / queue depth? We don’t want the guardrail to mask capacity issues.
Samir Qureshi: Deferrals are counted and exported; autoscaling can still use queue depth + deferral rate. Guardrails should make the signal cleaner (less crash-looping).

Final approvals: Omar El-Sayed, Sofia Ivanova, Ethan Park, Sean Gallagher. Merged after CI green + perf smoke pass.
- Runtime: add L40S-specific KV cache budgeting defaults + admission guardrails to prevent OOMs under long-context/high-concurrency workloads. Includes rollback toggle and additional startup/periodic logging for oncall triage.
