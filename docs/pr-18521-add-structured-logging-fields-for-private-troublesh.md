# Add structured logging fields + redaction safeguards for Private troubleshooting

Title: Add structured logging fields + redaction safeguards for Private troubleshooting

Context
Private customers and Support/SRE need a reliable way to correlate failures across the gateway, control plane, and serving runtime without requiring payload logging or environment-specific bespoke parsing. Today, several components log different key names (or omit them entirely), and a few code paths still allow unredacted headers to slip into debug logs.

This PR aligns the log schema for Private installs with the minimum pack expectations and adds redaction safeguards suitable for regulated environments.

Summary of changes
1) Standardized structured fields (JSON logs)
- request_id: propagated from API gateway to downstream control-plane calls and runtime request handling. If an inbound X-Request-Id is present, we honor it (bounded length + charset validation); otherwise generate a UUIDv4.
- tenant_id_hash: stable per-cluster HMAC-SHA256 of tenant_id using a Private-only secret key (redwood-logging/tenant-hmac-key). This allows correlation across services without exposing raw tenant identifiers.
- route: normalized route identifier (route_id if available; otherwise a sanitized route name). Avoids high-cardinality raw path logging.
- model: normalized model identifier (model_id / registry handle) instead of raw model display strings.
- Additional alignment fields where available: component, service, environment, cluster_id, namespace, pod, node, version, build_sha, trace_id (if present), span_id (if present).

2) Redaction + safe defaults
- Added a central redaction utility used by gateway, policy/routing, and runtime request logging.
- Default redactions:
  - Headers: Authorization, Cookie, Set-Cookie, X-Api-Key, X-Redwood-Api-Key, X-Amz-Security-Token, X-Google-* access tokens
  - Query params: api_key, token, access_token, refresh_token
  - Body/payload: request prompt/messages are not logged (even at debug). We only log prompt_bytes (length), max_tokens, and sampling flags.
- Added guardrails so "log_request_body" style flags in legacy configs do not enable payload logging in Private builds; instead they emit a warning and remain disabled.

3) Cardinality controls
- model and route are truncated and normalized to a bounded set of characters.
- Removed a few ad-hoc fields that were previously logging full URLs and raw endpoint paths.
- Ensured tenant_id_hash is constant-size and does not create unbounded labels/fields.

4) Backwards compatibility
- Kept legacy keys (req_id, tenant) for one release cycle with a deprecation warning in logs.
- Introduced log_schema_version: "2025-08-structured-v1" for downstream collectors/parsers.

5) Tests + docs
- Added unit tests for redaction (headers/query params/body-like fields) and request_id propagation.
- Updated Private troubleshooting docs to reference the new fields and recommended queries in Loki/ELK/Splunk.

Example (runtime)
{"ts":"2025-08-11T18:42:15.219Z","level":"info","service":"runtime","component":"inference","log_schema_version":"2025-08-structured-v1","request_id":"c16d7fd8-5f7d-4f9e-8b9e-8b835f8ac3ad","tenant_id_hash":"hmac256:1d2d...a91f","route":"route:chat-completions","model":"model:llama3.1-70b-instruct@2025-07-29","status":"ok","latency_ms":183,"prompt_bytes":842,"max_tokens":256,"token_out":211,"gpu":"A100","pod":"runtime-7c6b8c7b7d-qk9h2"}

Why HMAC for tenant_id_hash?
- A plain hash can be vulnerable to offline enumeration if tenant IDs are low-entropy. Using an HMAC with a per-cluster key prevents that while still allowing stable correlation.

Operational notes
- New secret: redwood-logging/tenant-hmac-key is created automatically by the installer if not present. Rotation is supported but will break cross-time correlation for tenant_id_hash; we document this for customers.
- For air-gapped installs, no new images are introduced; only code/config changes.

Rollout / risk
- Low runtime risk: schema changes only, no request-path changes.
- Some downstream log filters might rely on legacy keys; we keep aliases for one release.

Checklist
- [x] Security review for redaction rules (Marcus Lin / Kevin Osei)
- [x] Added tests for request_id propagation and redaction
- [x] Updated documentation for field names + example queries
- [x] Verified no prompts/messages are logged (search in runtime + gateway code paths)

Related
- ENG-19149 (Private log pipeline default collector config)
- ENG-19157 (Security review: log redaction and retention)
- SUP-49277 (Splunk HEC 401: improved request_id correlation made this easier to debug; doc updates include HEC field mapping)

Notes for Support/SRE
Recommended primary key for correlation is request_id. tenant_id_hash should be used only for aggregation and multi-request analysis; do not attempt to reverse it (not possible without the cluster key).
Private installs now emit consistent structured log fields (request_id, tenant_id_hash, route, model) across gateway/control-plane/runtime, with default redaction of sensitive headers and payload-like fields. No config changes required; optional knobs added for sink-specific field allow/deny lists.
