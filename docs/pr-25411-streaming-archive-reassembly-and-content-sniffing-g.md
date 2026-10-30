# streaming archive reassembly and content-type sniffing guardrails for tool attachments

Motivation: tool/function-calling workloads increasingly accept file/attachment payloads from third-party clients. We observed two classes of failures in production: (1) malformed/fragmented multipart streams that cause the streaming parser to drop the connection or OOM when client resumes with overlapping offsets; (2) clients that upload compressed archives (zip/tar/gzip) containing large uncompressed payloads which bypass per-file byte limits because limits were applied to compressed size only. This PR introduces a conservative, backwards-compatible set of guardrails to make the OpenAI-compatibility layer and streaming ingestion more robust.

Summary of changes (high level):
- Add streaming reassembly layer that normalizes chunk sequence and rejects overlapping/duplicated ranges with clear structured errors (422/409 mapping) instead of panics. The layer supports resumable uploads by validating a server-side digest and an expected "resume_at" offset in the protocol.
- Implement content-type sniffing using a lightweight signature check (magic bytes) for commonly abused archive formats (zip, tar, gzip, 7z) and image/video containers. We now compute both compressed_size and an estimated_uncompressed_size using heuristics when streaming (for gz/tar) and enforce per-tool uncompressed byte budgets.
- Introduce per-tool policy configuration: max_files_per_call, max_compressed_bytes_per_file, max_uncompressed_bytes_per_file, max_total_uncompressed_bytes. Policies default to permissive but are tunable via the compat layer config and Dedicated/Private runtime flags.
- Add a controlled server-side archive inspection path: small archives are inspected inline; larger archives are offloaded to a temporary local streaming scanner which checks for nested archives, large uncompressed members, and forbidden filetypes. Offload path uses a bounded temp-store and is subject to eviction/backpressure to protect the runtime.
- Improve OpenAI-compat error semantics: return structured error blobs for attachment validation failures with machine-readable reason codes (attachment_too_large, nested_archive, invalid_mime, stream_overlap) and suggested client actions. Map these to compat HTTP codes (413, 415, 422, 409) so existing OpenAI clients get expected behavior.
- Add integration tests that exercise resumable uploads, compressed->uncompressed size estimation, nested archive detection, and error mappings. Tests include a simulated flaky client that intentionally resends overlapping ranges to ensure we fail fast.
- Docs: add a new guide in redwood-docs describing the attachment lifecycle, recommended client upload patterns (use the new resumable headers), and sample error handling for SDKs.

Compatibility notes: the guardrails are additive. Existing clients that do not stream large archives will not observe changes. The resumable header is optional; when absent, we fall back to existing behavior with stricter streaming sanity checks. We intentionally avoid a breaking-change rollout: feature gated behind a runtime flag that is enabled for API-internal dogfooding then gradually flipped for customers.

Testing and CI: unit tests added for the reassembler and sniffers; integration tests added to the e2e suite. One flaky test (integration/archive-offload) failed on first CI run due to a timing race around temp-store eviction; we added a deterministic lease in the test and re-ran. Final CI status: all checks passed.

Operational playbook: when an attachment_too_large or nested_archive error is seen, the recommended steps are: (1) ask client to follow resumable upload flows (SDKs have helpers), (2) if customer needs to send large compressed archives, increase per-tool budgets and enable offload scanner for that org, (3) for suspicious nested archives escalate to security for static scanning.

Files changed summary (high level): streaming reassembly and sniffing in redwood/runtime/streaming; compatibility layer changes in redwood/compat/openai/attachments; per-tool policy config in redwood/api/config; offload scanner and temp-store in redwood/runtime/offload; docs under redwood-docs/guides/attachments.md; tests under integration-tests/attachments.

This PR touches runtime code paths so reviewers: please pay attention to memory accounting in the scanner and the choice of heuristics for uncompressed size estimation.

Related: Linear ENG-3421 (attachment resiliency), ENG-3498 (compat layer error mappings).
Diego Alvarez: Left a couple of comments around edge cases in reassembler.handleResume — what happens when client sends a zero-length chunk with resume_at equal to current offset? Please add a defensive check.
Maya Chen: Added check and unit test to assert zero-length chunks are rejected with 400 + reason=invalid_chunk. Also tightened logging to include upload_id for easier tracing.
Samir Patel: Offload scanner looks good, but please ensure we don't create unbounded goroutines per upload. Suggest using a worker limiter and add a metric for active scanners.
Maya Chen: Added worker limiter and active_scanners gauge; updated tests to assert limiter behaviour.
Rina Kapoor: Approving after changes. Verified error payloads and docs.
Add streaming reassembly and attachment inspection to improve robustness for tool attachments; introduces per-tool attachment budgets and structured error responses for invalid uploads. Backwards-compatible and feature-gated.
