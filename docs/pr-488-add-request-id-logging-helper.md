# Add request ID capture + logging helper for faster Support escalations

- New helper to capture Redwood request IDs from responses/errors
- Optional request ID logging hook for easier Support handoff
- Error objects now preserve `requestId` when available (non-breaking)
## Summary
Support escalations frequently stall because customers don’t include request IDs, or because the SDK drops the `x-request-id` header on error paths. This PR adds a small, opt-in helper that (a) captures Redwood request IDs from responses and error responses, (b) exposes them on thrown SDK errors, and (c) provides a safe logging hook so customers can copy/paste request IDs into Support tickets.

This is intentionally not enabled by default to avoid surprising logging behavior in customer apps.

## Motivation
- Support intake + escalation playbook now standardizes on collecting request IDs early (see linked SUP tickets).
- Streaming disconnects/timeouts are hard to reproduce without request IDs.
- Customers often ask “where do I find the request ID?” and we want a canonical answer in the TS SDK.

## What changed
- Added `requestId` extraction utilities (from headers) and an opt-in hook `onRequestId`.
- Updated SDK error shaping so thrown errors preserve a `requestId` field when present.
- Updated docs and examples to show how to log/capture request IDs without logging full payloads.

### Public API
- `getRequestIdFromHeaders(headers: Headers | Record<string, string | string[] | undefined>): string | undefined`
- `createRequestIdLogger({ onRequestId })` -> returns a small helper to attach to SDK request pipeline

No behavior changes unless consumers opt in by passing `onRequestId`.

## Implementation notes
- We normalize header lookups for Node (undici / fetch) and browser environments.
- We parse request ID from:
  - `x-request-id` (canonical)
  - `x-redwood-request-id` (legacy alias; kept for compatibility)
- Errors thrown by the SDK now include:
  - `requestId?: string`
  - `status?: number`
  - `code?: string`

## Example usage
```ts
import { Redwood, createRequestIdLogger } from "@redwood-inference/sdk";

const { onResponse } = createRequestIdLogger({
  onRequestId: (requestId) => {
    // Safe to log: request ID only (no prompts / payloads)
    console.info("Redwood requestId=", requestId);
  },
});

const client = new Redwood({
  apiKey: process.env.REDWOOD_API_KEY!,
  hooks: {
    onResponse,
  },
});

try {
  const res = await client.chat.completions.create({
    model: "redwood/verified-llama",
    messages: [{ role: "user", content: "hello" }],
    stream: false,
  });
} catch (err: any) {
  console.log("Support requestId:", err.requestId);
  throw err;
}
```

## Files changed (high level)
- `src/support/requestId.ts` (new): header normalization + extraction
- `src/support/requestIdLogger.ts` (new): opt-in hook wrapper
- `src/core/errors.ts`: preserve `requestId` on SDK error type
- `src/core/http.ts`: call request-id capture on both success + error responses
- `README.md`: troubleshooting section update
- `examples/node/request-id-logging.ts` (new)
- `test/requestId.test.ts` (new)

### Illustrative snippet
```ts
export function getRequestIdFromHeaders(headers: Headers | Record<string, unknown>): string | undefined {
  const h = normalizeHeaders(headers);
  return (h["x-request-id"] as string | undefined) ?? (h["x-redwood-request-id"] as string | undefined);
}
```

## Validation
- Added unit tests covering:
  - browser `Headers`
  - Node/undici response headers normalization
  - error response paths (4xx/5xx)
  - legacy header alias
- Manual: ran `examples/node/request-id-logging.ts` and confirmed request IDs print for both successful and forced-401 scenarios.

## Rollout / customer guidance
- Support can reference this for customers who ask how to find request IDs.
- Follow-up: we should cross-link this from the streaming troubleshooting doc update (ENG-4827) once merged.

## CI
- ✅ lint
- ✅ typecheck
- ✅ unit tests
- ✅ build

## Review conversation highlights
- Owen Phillips: requested we keep logging opt-in and avoid default console logging. Updated to require explicit `onRequestId` callback and clarified in README.
- Priya Shah: asked that requestId be present on thrown errors to support copy/paste into SUP tickets. Added `requestId?: string` to the public error surface and ensured it’s set on non-2xx responses.
- Ethan Park: flagged header normalization differences between environments. Added normalization helper + tests for both `Headers` and plain objects.

## Links
- Support tickets: SUP-1842, SUP-1854
- Workstreams: PM-914, ENG-4827

Merged via squash.
