# Add shared Cost Model API client library (timeouts, retries, schema negotiation)

Context
Console and Optimize are both adopting the Cost Model API (see ENG-4821/4822). We need a single, opinionated client that standardizes:
- request timeouts (hot path safety for Console)
- retries (bounded + only when safe)
- schema version negotiation (so callers can be forward-compatible and we can roll schema changes safely)
- consistent auth/tenancy + tracing/metrics

What this PR does
- Adds a shared client library for service-to-service calls to Cost Model API:
  - package: //go/platform/costmodel/client (Go)
  - exported type: costmodel.Client
  - supports endpoints: EstimateCost, GetCostBreakdown, WhatIf
- Timeouts
  - default request timeout: 250ms (overridable per-call)
  - default connect timeout: 50ms
  - caller can pass a context with deadline; client will respect the stricter of (ctx deadline, configured timeout)
- Retries
  - retries are bounded: max 2 retries, exponential backoff + full jitter
  - only retries on: 502/503/504, connection reset/timeouts, and explicit server retryable errors
  - never retries on: 4xx (except 429), schema/version errors, validation errors
  - idempotency: for POST requests we attach X-Redwood-Request-Id; server can safely dedupe if it chooses (client side still assumes retries may result in duplicate work)
  - 429 behavior: honor Retry-After (bounded by a per-call max budget)
- Schema version negotiation
  - client sends:
    - X-Redwood-Cost-Model-Schema-Version: <preferred_version>
    - X-Redwood-Cost-Model-Schema-Versions-Accepted: <csv of supported versions>
  - server responds with:
    - X-Redwood-Cost-Model-Schema-Version-Applied: <applied_version>
  - client surfaces applied version in response metadata and emits a metric when fallback occurs (preferred != applied)
  - unknown response fields are ignored by default (forward compatibility); strict mode is available for conformance tests
- Auth/tenancy plumbing
  - client requires tenant context (tenant_id) and propagates:
    - X-Redwood-Tenant-Id
    - X-Redwood-Caller-Service (for attribution)
  - supports standard internal service auth transport (mTLS) and attaches per-service auth token when available via ctx
- Observability
  - OpenTelemetry spans: costmodel.client.<method>
  - metrics:
    - costmodel_client_requests_total{method,status,retried}
    - costmodel_client_latency_ms_bucket{method}
    - costmodel_client_schema_applied_total{preferred,applied}
    - costmodel_client_retry_total{method,reason}
  - adds log fields (structured): request_id, tenant_id, schema_preferred, schema_applied

Why now
We have multiple in-flight integrations (Console + Optimize) and we’ve already seen early endpoint performance discussions in eng-platform threads. Centralizing client behavior prevents each consumer from inventing its own timeouts/retry logic and makes schema rollouts measurable.

Not in scope
- circuit breaking (we’ll rely on existing service mesh / caller-side fail-open behavior per integration)
- client-side caching (Console caching guidance will live in the integration how-to)
- non-Go clients (we can port the semantics to TS later; this PR establishes the canonical behavior)

Test plan
- Unit tests for:
  - retry policy (retryable vs non-retryable statuses)
  - 429 Retry-After handling
  - schema negotiation header behavior + strict mode
  - context deadline behavior
- Integration-style tests with httptest server verifying:
  - headers for tenant/auth/version negotiation
  - metrics/spans are emitted

Rollout
- Library is additive.
- Console/Optimize adoption will be done in separate PRs behind feature flags.

Follow-ups
- Add conformance tests that run against staging Cost Model API to validate negotiation behavior end-to-end (ENG-4831)
- Evaluate lowering default timeout for Console once we have p95 data in the new dashboards

Chloe Martin (Console): Default 250ms seems high for Console hot paths. Can we make it 150ms? We’ll have pages calling this per-route per-model and we want failures to fail fast and fall back to cached/approx numbers.
Benji Okafor (author): Agree on failing fast. I set 250ms to start because staging p95 for estimate endpoint isn’t stable yet. I added per-call override + documented recommended 150ms for Console in README. We can drop the default after we get a week of p95/p99 data from dashboards.
Rafael Mendes (SRE): Please ensure retry budget is bounded and doesn’t amplify outages. Also: avoid retrying on validation/version errors. Can you add a metric for retries + a label with retry reason?
Benji Okafor (author): Done. Retries capped at 2, only on 502/503/504 + transport timeouts, and 429 honoring Retry-After (bounded). Added costmodel_client_retry_total{reason} and a retried=true label on requests_total.
Kaitlyn Nguyen (Auth/Tenancy): Tenant propagation: can we enforce tenant_id required? Also please make sure we don’t accidentally log auth tokens.
Benji Okafor (author): Tenant is required in ClientOptions; constructor returns error if missing. Logging is only request_id/tenant_id/schema fields/status; no auth material. Added a unit test ensuring headers don’t get dumped in error strings.
Allison Grant (Observability): Span naming looks good. Please add an attribute for schema_applied and schema_preferred so we can correlate fallback rates. Also avoid high-cardinality attributes (tenant id) on spans.
Benji Okafor (author): Added span attrs schema.preferred/schema.applied and omitted tenant_id from span attrs. Tenant remains in structured logs (already standard in platform services).
Ethan Park (API lead): Version negotiation headers: can we align names with the API standards doc? Also strict mode is great—please ensure default behavior is permissive (unknown fields ignored) for forward-compat.
Benji Okafor (author): Renamed headers to the X-Redwood-Cost-Model-* set we agreed on in the design review. Default is permissive; strict mode only enabled in tests via option WithStrictDecoding(true).
Ethan Park: Approved after header rename + docs update.
Rafael Mendes: Approved after retry metrics landed.
