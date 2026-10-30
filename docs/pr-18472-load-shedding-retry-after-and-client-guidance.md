# Load shedding: consistent Retry-After + correlation IDs; document client retry contract

Context: During canary we saw overload responses missing Retry-After on some routes (notably chat completions + a subset of streaming paths), which caused clients to implement ad-hoc retry behavior and in one case a retry storm. This PR standardizes how we emit Retry-After and correlation IDs for overload/load-shedding responses, and adds a short internal "client retry contract" comment block to keep behavior stable across gateway/routing/runtime.

What changes: 1) Retry-After is now set for all overload-class responses emitted by the API gateway (admission control + breaker open + dependency brownout mapping) and for propagated overload errors from downstream services when they bubble back through the gateway. 2) Correlation ID plumbing is tightened: we prefer an existing inbound X-Request-ID (if present), otherwise generate a Redwood request id, and we always return it to clients in both X-Request-ID and X-Redwood-Correlation-ID for a deprecation window. 3) Centralizes overload response construction into a shared helper that sets headers + JSON payload consistently (status, error.code, error.message, retry guidance, request_id). 4) Adds a code comment (internal contract) that documents semantics: when clients should retry, how to treat 429 vs 503 overload, and that Retry-After is in seconds.

Notes/constraints: - Retry-After is emitted as an integer number of seconds (RFC-compliant); we do not emit HTTP-date format. - For streaming (SSE) requests, we only attempt overload rejection before the response is committed; once headers have been sent, we cannot reliably add Retry-After. This PR ensures the admission decision happens before initiating the stream in gateway. - We do not change error codes/status mappings here (see ADR-014); this PR only enforces header + correlation id consistency.

Testing: - Unit tests for overload response helper (header presence, seconds formatting, correlation id behavior). - Integration test covering /v1/chat/completions streaming: overload rejection happens before stream start and includes Retry-After + request id. - Regression test for SUP-2860 reproduction (route missing header).

Rollout: Safe for dark launch; header additions are backwards compatible (additive). Client-visible change is improved consistency (some clients previously saw no Retry-After).
Ethan Park (review): Can we confirm Retry-After is always seconds (not date) and consistent across 429/503? Also please ensure we don't accidentally set it on non-overload 5xx.

Christine Wong (author): Yep. Helper only used by overload paths (breaker open, admission shed, downstream mapped overload). Added a guard test to ensure generic 5xx errors do not set Retry-After.

Aisha Bello (review): Please add a metric for missing Retry-After as a canary safety check. We should be able to alert if this regresses.

Christine Wong (author): Added api_gateway.overload.retry_after_missing counter (dimensions: route_group, region, status) and wired it into logging when helper isn't used (should stay at 0). No tenant_id dimension.

Ibrahim Khan (review): For SDKs we key off X-Request-ID today. Returning both X-Request-ID and X-Redwood-Correlation-ID is fine, but we should document the precedence and avoid breaking existing clients.

Christine Wong (author): Updated request_id.ts to prefer inbound X-Request-ID and always echo it back; X-Redwood-Correlation-ID is additive for now. Added comment in helper about deprecation window.

Mina Haddad (review): On routing fallback, we sometimes return an overload from routing with a retryable code. Are we certain Retry-After is preserved when routing returns its own error payload?

Christine Wong (author): Gateway now overwrites/sets Retry-After on mapped overload regardless of downstream body, but preserves downstream error.code/message. Also ensures request_id is our canonical id.

Nikhil Sharma (review): Please keep the "client retry contract" short and explicit. We should avoid promising exact backoff durations.

Christine Wong (author): Tightened language: we only promise Retry-After is present and in seconds; clients should add jitter and cap retries. No guarantees on exact window beyond header value.
Hosted API: overload/load-shedding responses now consistently include Retry-After (seconds) and return a stable correlation ID header for easier client retry/backoff and support debugging.
