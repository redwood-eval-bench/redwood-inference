# gateway: return 409 + standardized error payload for residency policy violations

Context
We’re introducing Hosted data residency enforcement (region pinning) across routing + gateway. The gateway must return a deterministic, consistent error when a request is blocked by residency policy (instead of falling back, timing out, or returning generic 500s in some streaming paths).

This PR implements the public error contract for residency violations:
- Returns HTTP 409 (Conflict) when residency policy blocks a request
- Standardized JSON error payload with stable `error.code`, `error.type`, and `help_url`
- Works consistently for both streaming and non-streaming endpoints
- Propagates `request_id` and `residency.decision_id` for support/debug (without exposing full policy content)
- Maps internal policy-engine outcomes to a small set of public-facing reason codes

Motivation / user impact
- Customers with EU-only pinning should get a clear, actionable error when their request would route outside allowed regions (or when they call the wrong regional base URL)
- SDKs can reliably detect this as a residency issue and surface guidance (instead of treating as retryable 5xx)
- Support gets deterministic identifiers to locate decision logs and troubleshoot quickly

Behavior
- Non-streaming: respond with 409 immediately
- Streaming: if policy block is known before starting the stream, the gateway responds with a normal 409 JSON error (no stream initiated). If a block is discovered after headers would have been sent (should be rare; mostly during phased rollout), we terminate the stream with a final SSE error event using the same error schema and include `request_id`.

Public error schema (example)
{
  "error": {
    "type": "invalid_request_error",
    "code": "data_residency_policy_violation",
    "message": "Request blocked by data residency policy: allowed regions are [eu-central-1]. Update your project’s residency settings or use an API base URL in an allowed region.",
    "param": null,
    "help_url": "https://docs.redwoodinference.com/errors#data_residency_policy_violation"
  },
  "request_id": "req_01H...",
  "residency": {
    "decision_id": "rdc_01H...",
    "selected_region": null,
    "allowed_regions": "eu-central-1",
    "reason": "target_region_not_allowed"
  }
}

Notes:
- `allowed_regions` is returned as a comma-separated string to keep the payload stable across languages/SDKs and to avoid accidental JSON shape drift.
- We do NOT return `policy_id` in the public response. We do return a `decision_id` which can be looked up in internal logs.
- `help_url` points to docs anchors; docs PR will add that section (tracked separately).

Implementation details
- Adds a dedicated `ResidencyViolationError` type in the gateway domain layer with a strict mapping from internal policy evaluator outcomes.
- Centralizes error serialization in `gateway/errors/public_errors.ts` to guarantee the same shape for REST + streaming adapters.
- Adds regression tests for:
  - /v1/chat/completions (json)
  - /v1/completions (json)
  - /v1/embeddings (json)
  - streaming chat SSE path (ensures no generic 500)
- Adds structured logging fields:
  - `residency.blocked=true`
  - `residency.reason`
  - `residency.decision_id`
  - `http.status=409`

Rollout
- Guarded by feature flag `residency_enforcement_error_contract_v1` (default ON in staging, OFF in prod until routing enforcement is enabled per-region).
- When flag is OFF, behavior is unchanged.

Refs
- Linear: ENG-4833, PM-1490
- Support: SUP-28577 (streaming endpoint returning generic 500)
- ADR: ADR-022 Error contract for residency violations (implemented here)

Checklist
- [x] Error payload matches ADR templates (code, type, help_url)
- [x] Streaming + non-streaming parity
- [x] Unit tests + integration tests
- [x] PII review: no prompt/content in error message
- [x] Added logs for oncall triage
- [ ] Docs link validation (follow-up in docs repo)
Brian Murphy (review) — 2025-02-22
1) Can we confirm the error code matches ADR-022 exactly? I see `data_residency_policy_violation` here; ADR uses the same, good. Please ensure `error.type` is `invalid_request_error` (not `policy_error`).
2) `allowed_regions` is being returned as an array in one handler and as a string in another. We should standardize.

Amara Diallo (author)
1) Yep, updated `error.type` to `invalid_request_error` everywhere and added a snapshot test.
2) Good catch — moved serialization into `public_errors.ts` and made `allowed_regions` a comma-separated string in all responses + SSE.

Marcus Lin (security) — 2025-02-22
- Please do not return `policy_id` or any internal object IDs that could be correlated across orgs. `decision_id` is ok if it’s generated per-request and not guessable.
- Ensure the message doesn’t reveal the customer’s current region if they didn’t provide it.

Amara Diallo (author)
- Removed `policy_id` from the payload; `decision_id` is a random ULID generated server-side per request.
- Message only references allowed regions; does not echo requested/target region.

Ravi Menon (routing) — 2025-02-23
- For the reason mapping: can we align strings with what policy service emits so dashboards don’t fragment? I propose `target_region_not_allowed`, `fallback_not_permitted`, `policy_missing_primary_region`.

Amara Diallo (author)
- Aligned mapping to the three values you listed; added a small unit test that asserts the mapping is stable.

Sean Gallagher (SRE) — 2025-02-23
- Streaming behavior: if headers already sent, do we still show an HTTP 409? That won’t work. Please confirm what the client sees.
- Add a log line when we have to terminate a stream due to residency violation after start; that’s a signal we should investigate.

Amara Diallo (author)
- Confirmed: if stream hasn’t started, client receives standard 409 JSON. If stream has started, we send an SSE `error` event with the same payload, then close. Added `residency.streaming_termination=true` to logs.

Kaitlyn Nguyen (auth) — 2025-02-24
- Any interaction with API key scopes? If key-level policy exists later, does payload remain correct?

Amara Diallo (author)
- Payload is policy-source agnostic; it reports only the evaluated allowed regions + reason. Key-level policy will work without changes.

Approvals
- Brian Murphy: Approved (2025-02-24)
- Marcus Lin: Approved (2025-02-24)
- Sean Gallagher: Approved (2025-02-24)
