# Shadow traffic: deterministic request fingerprint + bucketing v1

Context: Shadow traffic needs consistent comparisons across time windows and across control/shadow targets. The earlier routing mirror MVP used per-request RNG sampling which made deltas noisy and made it hard to correlate control vs shadow outputs for the same logical request. This PR adds a deterministic request fingerprint and a stable bucketing/assignment helper used by the data plane when deciding whether to mirror a request and which shadow target(s) to send it to.

What’s in this PR: 1) Request fingerprinting (fingerprint_v1): Computes a 256-bit fingerprint as HMAC-SHA256(tenant_shadow_secret, canonical_request_bytes). Canonical bytes are derived from a canonical JSON encoding of a strictly limited set of fields: route_id, request_kind (chat/completions/embeddings), model_group (logical), normalized generation params (temperature/top_p/max_tokens/etc), and the request content payload (messages/prompt) after canonicalization (stable key ordering, normalized whitespace for JSON tool args, and explicit UTF-8 encoding). The HMAC key is per-tenant (and per-environment) and is never emitted in telemetry. This ensures: (a) the same logical request maps to the same fingerprint across instances/regions; (b) fingerprints are not reversible without the secret; (c) we do not store raw prompt text to achieve determinism.

2) Bucketing + assignment: Adds helpers for (a) deterministic sampling: sample = (uint64(fingerprint[0:8]) / 2^64) < sample_rate; (b) stable variant selection: variant_index = uint64(fingerprint[8:16]) % num_variants; (c) optional multi-shadow fanout selection using independent lanes: lane_i uses bytes [16+8*i : 24+8*i] to avoid correlated selection when we add more than one shadow target.

3) Collision + privacy notes: Includes a short ADR note in-code and in docs about collision risk and privacy posture. Collision risk: We still treat the fingerprint as an identifier, not a cryptographic guarantee of uniqueness; we store both (fingerprint_prefix_64) and (fingerprint_full_256) internally, but only emit a truncated 96-bit form in telemetry events to reduce cardinality and avoid join keys that are too powerful. For collision math: probability is negligible at our expected sample sizes with 96 bits, but we still design joins to tolerate rare collisions (e.g., shadow pairing includes request_id + timestamp bucket as additional join constraints).

4) Guardrails: The canonicalization step caps included content to 16KB for fingerprinting (after UTF-8 bytes) and includes a fast-path for streaming requests: fingerprint is computed from the initial request envelope only (before any streamed tokens) so we do not block the user path on stream completion.

Why HMAC vs plain hash: A plain SHA256(prompt) can be dictionary-attacked for common prompts; HMAC with a per-tenant secret meaningfully raises the bar and prevents cross-tenant correlation of fingerprints.

Rollout plan: This PR wires the helper behind ShadowTrafficDeterministicBucketing (default off). Routing mirror code will start using this in pr-24833 once merged. No behavior change unless the flag is enabled.

Test plan: Unit tests for canonicalization stability (key ordering, whitespace), determinism across processes, sampling distribution sanity (chi-square vs expected), and variant selection stability. Added a microbenchmark to ensure fingerprinting stays < ~15µs p50 on typical chat payloads (1–4 messages).
Mina Haddad (review): Canonicalization worries me. What fields are we including exactly, and is there any chance we accidentally include request_id or timestamp and break determinism across retries/regions? Also, does including model_group mean we won’t be able to compare control model A vs shadow model B for the same prompt? 

Vikram Singh (author): Good callouts. We explicitly exclude request_id, trace_id, timestamp, headers, and any per-hop metadata. CanonicalRequest is constructed from the user-visible API payload + route_id + normalized params only. On model_group: we include the logical model group requested (e.g., "chat-default") rather than the resolved concrete model id. That way control vs shadow variants (different concrete ids) still share the same fingerprint.

Benji Okafor (review): Please include route_id (or route stable key) in the fingerprint; otherwise two different routes with identical prompts will collide and we’ll pair apples-to-oranges in dashboards. Also, can we ensure the per-tenant secret is not shared across Dedicated pools? 

Vikram Singh (author): Added route_id to the canonical set (it was in the draft but not plumbed through for all request types). Secret is sourced from tenant config; Dedicated pools will have a tenant-scoped secret in their control plane namespace. I also added a comment that secrets must be unique per tenant+env and should not be reused across stacks.

Marcus Lin (review): Privacy: emitting even truncated fingerprints can become a stable correlation handle across datasets. What’s the retention and how are we preventing cross-product joins? Also: how do we handle secret rotation without breaking longitudinal experiments? 

Vikram Singh (author): For telemetry we emit only a 96-bit truncation and we do not expose it via customer-facing APIs (internal-only). Pairing in the pipeline uses the full 256-bit value stored in the shadow artifacts table (with short retention per policy). On rotation: we add a key version prefix in the stored fingerprint metadata; when we rotate, we will still be able to pair within the same key-version window. Longitudinal experiments spanning rotations will show a discontinuity (documented). Follow-up: add dual-key support for a transition period (separate PR; Marcus requested).

Sean Gallagher (review): Performance: HMAC per request on the hot path is OK if bounded, but I want an explicit cap on canonical payload bytes. Also ensure any failure in canonicalization is fail-open (i.e., shadow disabled, user path unaffected). 

Vikram Singh (author): Added max 16KB cap (bytes) for content contribution; beyond that we hash only the first 16KB and include a "truncated" bit in the canonical bytes. Any canonicalization error returns (fingerprint="", sampled=false) and logs at debug with a counter; it never blocks the request.

Nadia Rahman (review): Telemetry note: if you emit 96-bit fingerprint, do not allow it as a dimension in high-cardinality metrics. Can you name the fields carefully (request_fingerprint_trunc) and mark as "join-only"? 

Vikram Singh (author): Done. Updated naming to request_fingerprint_trunc96 and added comments in the event schema docs. We also added a lint rule stub in our metrics emitter (TODO in ENG-5410) to prevent accidental tag emission.

Resolution: Approved after changes (route_id inclusion, payload cap, truncation naming, added key-version metadata).
CI: build (pass), unit tests (pass), gofmt (pass), lint (pass), security-scan (pass). Perf microbench: router/shadow BenchmarkFingerprintV1 p50 9.8µs (c6i.2xlarge), p99 17.1µs; no regression against main (±3%).
Internal: adds deterministic request fingerprinting and bucketing helpers for Shadow Traffic experiments (flagged off by default).
Uses per-tenant HMAC secret for non-reversible fingerprinting; avoids emitting full fingerprint to telemetry; caps canonicalized content bytes and excludes volatile identifiers (request_id/timestamp/headers).
