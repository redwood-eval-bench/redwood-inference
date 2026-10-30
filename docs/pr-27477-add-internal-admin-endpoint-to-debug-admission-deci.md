# Add internal admin endpoint to debug admission decisions

Context: With hot-route capacity protection enabled (tier-aware quotas + priority admission control), on-call and Support have needed a deterministic way to answer “why did this request get a 429 / get shed?” beyond inference from metrics and aggregate logs. During recent escalations we also saw that reason codes alone are not enough to diagnose whether a throttle was quota-window enforcement vs overload admission control vs policy budget exhaustion vs route-group misconfig.

This PR adds a restricted internal admin endpoint on the routing service to inspect admission decisions for a specific request/trace and to dump the evaluated policy inputs (sanitized) that led to the decision.

What’s included: - New internal endpoint: GET /internal/admin/admission/decisions?request_id=… OR ?trace_id=… (prefers request_id) - Response includes: evaluated tier, route group, policy version/hash, decision action (allow|throttle|shed|fallback), reason codes, matched limits/budgets, and a snapshot of capacity signals used by the admission controller (queue depth, scheduler lag bucket, saturation buckets) - Backed by a short-lived in-memory decision ring-buffer (15 min TTL) populated on the request path; no additional synchronous work added to the data plane beyond existing decision logging. The admin handler reads from the buffer and does not touch Redis/DB. - Strict access control: endpoint is disabled by default and gated by (a) feature flag admin_debug_endpoints_enabled, (b) network allowlist (admin ingress / internal mesh only), and (c) RBAC scope “oncall:admission_debug”. - Audit logging: every admin endpoint hit is logged with actor, source, tenant_id (hashed), and request_id; payload does not include prompt, completion, or customer-provided content. - Rate limit: 5 rps per actor to prevent accidental abuse during incidents.

Explicitly not included: - No customer-visible behavior changes - No additional high-cardinality metrics; this is purely an internal on-call tool - No persistence of decision logs (future work; would require privacy and retention review)

Why this approach: We considered querying the main request logs, but those are sampling-based and can lag during incidents. We also considered persisting decisions to Redis, but that adds failure modes and write amplification on hot routes. The ring-buffer approach keeps overhead minimal and still covers the primary incident debugging window.

Example response (redacted fields): {"request_id":"req_01J…","trace_id":"4f3d…","ts":"2025-12-14T22:41:09Z","tenant":"tnt_hash:9b0c…","route":"/v1/chat/completions","tier":"enterprise","route_group":"chat","policy":{"version":"2025-12-14.3","hash":"a12c…"},"decision":{"action":"throttle","reason_code":"overload_admission_control","matched_budget":"enterprise.chat.p95","matched_limit":"rps:1200 burst:200","fallback_attempted":false},"signals":{"queue_depth_bucket":"p95_gt_250","scheduler_lag_bucket":"p95_gt_2s","gpu_saturation_bucket":"gt_92"},"notes":["best_effort shed active: true","enterprise reserve remaining: 3%"]}

Operational notes: - Intended usage is via the on-call toolbox (curl from admin bastion) or incident-bot deep links (future) - If a request_id/trace_id is older than the buffer TTL, the endpoint returns 404 with reason_code=decision_not_found and includes current policy version to help check config drift.

Testing: - Unit tests for RBAC, feature flag gating, and request_id/trace_id lookup - Integration test ensures admin endpoint cannot be reached from public ingress and requires the correct scope - Load test locally: ring-buffer write path adds ~0.7µs avg per request on my laptop benchmark; negligible relative to existing decision evaluation.

Rollout plan: - Land behind admin_debug_endpoints_enabled (default off) - SRE will enable for on-call in us-east canary first after INT-3302 access provisioning is complete - Follow-up: document in runbook and add incident-bot shortcut.

Refs: ENG-4924, ENG-4923, INT-3302, SUP-18421
Jordan Lee: Can we guarantee this won’t leak customer prompts/tokens? The decision object currently carries some request metadata in other paths.
Benji Okafor (author): Good call. I explicitly constructed a sanitized struct for the ring-buffer (tenant hash, route, tier, policy ids, capacity buckets). No headers, no prompt, no token counts. Added a regression test that asserts prompt fields are not present in the serialized response.
Kaitlyn Nguyen: RBAC gating looks correct, but we should also ensure this handler is not registered on the public router by mistake. Can we enforce internal-only build tag or separate mux?
Benji Okafor (author): Updated to register under the existing internalAdminMux which is only mounted on the internal listener. Also added an integration test that hits public ingress and asserts 404 (no route).
Peter Holtz: Request lookup by trace_id could be ambiguous (multiple requests per trace). What do we return?
Benji Okafor (author): Now returns the most recent decision for that trace_id (by timestamp). Added a response field "lookup":"trace_id_most_recent" so on-call understands the behavior.
Rafael Mendes: Please add an explicit rate limit; during incidents folks will script this.
Benji Okafor (author): Added per-actor token bucket limit (5 rps / burst 10) and returns 429 with reason_code=admin_rate_limited.
Ethan Park: Looks good. One more request: include config/policy version + hash to debug rollouts.
Benji Okafor (author): Added policy.version and policy.hash fields; pulls from the same config snapshot used for evaluation.
Jordan Lee: Approved after sanitization test.
Kaitlyn Nguyen: Approved.
Peter Holtz: Approved.
Rafael Mendes: Approved.
