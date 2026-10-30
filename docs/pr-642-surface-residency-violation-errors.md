# sdk: raise ResidencyPolicyViolationError for data residency enforcement blocks

Context: Hosted data residency enforcement returns a deterministic error when a request would violate the org/project/API-key region policy (see ADR-022 + gateway work in ENG-4833). Today the Python SDK surfaces this as a generic APIError, which makes it hard for customers to catch and handle (e.g. prompt the admin to update allowed regions) and for support to quickly pull request traces.

What this PR does: Adds a dedicated exception type, ResidencyPolicyViolationError, and maps residency enforcement responses into it. The exception exposes (a) error_code (machine readable), (b) request_id (for support + audit correlation), and (c) help_url (docs link for remediation). The rest of the response payload is still preserved on the base error for debugging.

Behavior: - If the API returns HTTP 409 with error.code matching residency enforcement (e.g. RESIDENCY_POLICY_VIOLATION), raise ResidencyPolicyViolationError. - For all other errors, keep existing behavior (APIError subclasses like RateLimitError, AuthError, etc.). - For streaming: the exception is raised when the initial response is an error; mid-stream transport failures remain unchanged.

Why 409: Aligns with the public error contract for policy blocks (conflict with configured policy) and matches the gateway rollout.

Testing: Added unit tests for mapping and for preserving request_id/help_url. Also added a small integration-style fixture to ensure unknown codes do not get misclassified.

Docs: This PR does not update public docs; it only changes the SDK surface. A separate docs PR will reference the new exception in the Python examples.

Checklist: - [x] Linked to PM-1490 acceptance criteria (machine readable error + actionable guidance) - [x] Added tests - [x] No breaking change (new exception is a subclass of APIError) - [x] Backwards compatible parsing (help_url is optional; request_id may be absent on older gateways)
2025-03-04 10:18 Naomi Feldman: Can we ensure the exception name matches product language? We’ve been using “residency policy violation” in the Console copy.
2025-03-04 10:27 Brian Murphy (author): Yep—renamed from RegionPinningError -> ResidencyPolicyViolationError and aligned docstrings with the Console terminology.

2025-03-04 13:02 Marcus Lin: Please confirm we’re not exposing any internal policy identifiers or region routing internals beyond what the gateway returns. Also, make sure request_id is carried through even when error body is missing fields.
2025-03-04 13:16 Brian Murphy (author): Agreed. This maps only code/request_id/help_url + message. No policy_id exposure. Added guard rails so request_id defaults to None but is still available on the base error via headers when present.

2025-03-05 09:41 Wei Chen: Can we keep it as a subclass of APIError so existing try/except blocks still work? And can we add a short note in the changelog?
2025-03-05 09:55 Brian Murphy (author): Yes, subclassing APIError. Added release note entry and docstring showing how to catch specifically.

2025-03-06 11:08 Marcus Lin: LGTM after the request_id fallback change.
2025-03-06 11:21 Naomi Feldman: Approved. Please link this in PM-1490 so we can close acceptance criteria.
2025-03-06 11:33 Brian Murphy (author): Done (linked in Linear + release notes).
