# Add model version metadata types + changelog fetcher

Adds typed SDK support for model metadata + release notes/changelog endpoints (including pagination helpers) and an example showing how to render a simple customer-visible changelog feed.
Context: As part of the model metadata schema v1 + automated changelog work, we need first-class SDK affordances so customers can programmatically query: (1) model/version metadata (compatibility, deprecations, availability) and (2) generated release notes for a given model/version or time window. This PR adds TypeScript types that mirror the control plane responses, plus a small client wrapper (fetchers) and examples.

What’s included: 1) New public types (generated-by-hand for now, not OpenAPI-generated yet): ModelVersionMetadataV1, ModelArtifactRef, DeploymentAvailability, ReleaseNoteEntry, ReleaseNoteCategory, ReleaseNoteSeverity, and RedactionLevel (public-only; internal fields are not represented). 2) New client surface: changelog.getReleaseNotes({ model, after?, before?, limit?, cursor? }) and changelog.getModelVersionMetadata({ model, version }) with cursor-based pagination helper. 3) Error handling: maps 401/403/404 into existing RedwoodError types; adds a discriminated union for “not available in this deployment” vs “model/version not found”. 4) Docs/examples: adds examples/release-notes-feed.ts and updates README snippet for “Change visibility / Release notes”.

Notes/constraints: - This intentionally only models the public/customer-visible fields. Internal-only notes are not exposed in the SDK types even if a user is an admin in Console; the API’s redaction layer remains the source of truth. - Field names follow the API payloads (snake_case) but we expose camelCase accessors via a lightweight transform to keep the rest of the SDK consistent. - We did not add any dependency on the Console endpoints; this uses the control plane endpoints intended for SDK consumption.

Testing: - Unit tests for cursor pagination and response transforms. - Contract-ish tests using recorded fixtures from staging (sanitized). - Example runs in CI to ensure examples compile.

Rollout: Safe to ship as a minor version. No existing API behavior changes; only additive exports.

References: - ENG-9821 (schema) - ENG-9823 (changelog generator internal API) - Docs PR will reference the example in the “Release notes API” guide once endpoints are GA.
Quentin Harris: Types look good. Can we align enum values with the taxonomy doc (e.g., ‘performance’ vs ‘perf’)? Also confirm we never represent internal-only fields in the SDK.
Nora Ahmed (author): Updated enum values to match the release note taxonomy (security, breaking, performance, quality, bugfix, deprecation). Also removed the internalNotes placeholder type and added a comment explaining why we intentionally omit internal fields.
Sahana Iyer: Pagination helper returns {items, nextCursor}. Could we standardize naming to match other list() calls in the SDK? Also please add a unit test for empty-page behavior.
Nora Ahmed (author): Renamed to {data, nextCursor} to match existing listResponses and added a test for empty page + missing cursor.
Wei Chen: Example is helpful. Please ensure wording in README doesn’t promise notifications/subscriptions yet. ‘Poll for updates’ is fine.
Nora Ahmed (author): Adjusted README to say customers can query/poll release notes; removed mention of webhooks/notifications.
Jada Williams: One concern: the client is doing snake->camel transform. Please keep raw response accessible for debugging.
Nora Ahmed (author): Added an optional { raw: true } flag to return the unmodified payload for both endpoints, documented in JSDoc, and added a test to ensure no transform occurs when raw=true.
Dylan Brooks: Looks ready. Please bump minor version and add changelog entry for the SDK release.
Nora Ahmed (author): Version bump + CHANGELOG.md entry added. Marked as minor since additive exports.
Coordinate with docs team to reference the example in public docs once release notes endpoints are GA.
After OpenAPI spec is finalized for metadata v1/changelog endpoints, revisit generating these types instead of hand-maintaining.
