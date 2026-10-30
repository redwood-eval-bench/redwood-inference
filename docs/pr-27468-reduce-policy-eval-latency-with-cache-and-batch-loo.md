# policy-engine: reduce policy eval latency with request-scope cache + batch lookups

Context: Hot-route capacity protection adds a policy evaluation hop on the request path (gateway/routing -> policy engine) that must remain low-overhead to avoid shifting tail latency. In canary load tests we were seeing policy eval add ~2.0–3.5ms p95 at peak QPS, largely due to repeated lookups (tenant/tier/route-group) and per-check datastore reads for quota + budget state. This PR reduces end-to-end policy eval time by introducing (1) a request-scope evaluation cache and (2) batch retrieval for the quota/budget state used to decide admission/priority.

What changed: - Add request-scope cache keyed by (tenant_id, route_group, priority_class, region) for derived policy artifacts: resolved route-group match, tier defaults, priority class mapping, and compiled rule set (post-validation) - Add batch read path to quota/budget state store: evaluator builds a set of required keys (route budget, per-tier budget, per-tenant quota windows) and fetches them in a single call rather than N sequential reads - Short-circuit evaluation for common allow paths (enterprise allowlisted traffic + no overload signal) to avoid constructing unused decision objects and reduce allocations - Add microbench + production-facing metrics: policy_eval_cache_hit, policy_eval_batch_fetch_ms, policy_eval_total_ms, policy_eval_store_calls (guardrails for regression) - Preserve determinism/auditability: decision log still includes resolved rule IDs + store read versions; cache only stores derived artifacts, not mutable state - Add safe fallback: if batch fetch fails or returns partial results, evaluator falls back to prior per-key reads and annotates decision metadata with "degraded_store_mode=true"

Performance results (local bench + canary): - Local microbench (go test -bench): eval p95 improved from 2.8ms -> 1.1ms at 1000 concurrent with synthetic ruleset; allocations/op reduced ~35% - us-east canary (routing -> policy): policy eval overhead improved ~1.9ms -> ~0.7ms p95 at peak; store calls per eval from ~6.2 -> ~1.4 average - No change in decision outcomes in replay tests (golden logs) and integration tests for tiered quota windows

Why this is safe: - Cache invalidation: request-scope only (lives for the evaluation span); no cross-request stale policy issues - Batch fetch is read-only aggregation; decision semantics unchanged (verified by replay tests) - The fallback path ensures we don’t fail-closed on store hiccups; we keep existing behavior and visibility

Rollout plan: - Ship behind feature flag policy.eval.batch_fetch (default OFF) - Enable in us-east canary allowlist first (paired with ENG-4928 rollout guardrails) - Monitor new metrics and alert on policy_eval_store_calls regression + policy_eval_total_ms p95

Related: - ENG-4923: priority class evaluation + deterministic decision logging - PR #27441: overload reason codes for 429 responses - PR #27427: routing admission control driven by capacity signals

Testing: - Unit tests for cache keys + rule compilation caching - Integration tests covering quota windows and overload/admission behavior unchanged - Added bench tests in policy/engine/evaluator_bench_test.go

Notes for reviewers: Most of the diff is evaluator refactor; please focus on correctness parity (decision equivalence tests) and the store batch API contract. Telemetry naming follows the overload metrics cardinality guidance (no per-tenant high-card tags).
Mina Haddad (review): Can you confirm the cache is strictly request-scope? I’m worried about stale rules if policy bundles update mid-rollout. Also, do we log whether a decision used cache?

Jordan Lee (author): Yep, cache lives inside EvalContext created per request in the policy service handler; it’s dropped after response. No cross-request memoization. Added decision metadata field eval_cache.hit=true/false (not included in customer-facing payload, only internal decision log) so we can debug performance and parity.

Vikram Singh (review): BatchGet returning partial results is tricky. In fallback mode, are we potentially double-reading and increasing store load under pressure?

Jordan Lee (author): We only fallback for missing keys; we don’t re-fetch keys that were returned. Also we annotate degraded_store_mode so we can quantify it. In normal operation it’s a single call. Under store flakiness this may increase reads but preserves correctness. If it becomes frequent we can choose fail-open for best-effort only, but I’d prefer visibility first.

Logan Wright (review): Metric names look fine but please avoid tagging route_group as raw string if it’s high-card. Can we cap it to protected route groups only or use an enum?

Jordan Lee (author): Good call. Updated to tag route_group_class = {protected|unprotected|unknown} and keep the raw route_group only in logs (not metrics). Also added a unit test asserting no tenant_id tag usage.

Cole Summers (review): For quotas: we should keep window alignment semantics identical. Does batch path reuse the same rounding logic?

Jordan Lee (author): It does; BatchGet keys are built by the same KeyForWindow() helper used by the old per-key reads. Added integration tests comparing decision output for a set of window boundaries.

Amara Diallo (review): Any impact on 429 reason codes? We’ve had customer confusion (SUP-18421).

Jordan Lee (author): No behavior change; reason code generation happens after policy decision. This PR should reduce the chance of policy timeouts causing ambiguous 5xx -> which was contributing to confusing client behavior.

CI note: Initial run failed due to flaky integration test (timeout) in routing/policyclient; re-run passed after increasing test server deadline by 2s.

Approvals: Mina (approved), Vikram (approved), Logan (approved w/ metrics change), Cole (approved).

Merged via squash after addressing metric cardinality feedback and adding degraded_store_mode annotation.
Internal: Reduced policy-engine evaluation overhead via request-scope caching and batch quota/budget reads (behind feature flag policy.eval.batch_fetch). Added metrics to monitor eval latency and store call rate.
