# resync-sse-offset-skew-and-toolcall-boundary-mapper

Context: We observed sporadic user reports where structured tool/function call outputs (tool call JSON objects) emitted over SSE could arrive with mismatched framing when upstream fragmentation produced skewed byte offsets. In practice this manifested as: (1) toolcall JSON emitted across multiple fragments with fragment offsets shifted by preceding heartbeat/event fragments, and (2) downstream OpenAI-compatible adapters sometimes reassembled fragments using a strict append order, producing truncated or duplicated JSON boundaries under backpressure. This PR introduces an offset-resynchronization layer and a toolcall-boundary mapper to make reassembly idempotent and order-agnostic.
fix(sse/parser): introduce fragment offset map and resync algorithm
feat(sse/handler): add toolcall_boundary_mapper to align JSON span start/end against resync table
chore(tests): add deterministic replay tests and new fixtures for interleaved heartbeats
docs: update CHANGELOG and observability notes
Samir: Can you explain how this interacts with the existing checkpoint/replay flow? Are there cases where we might drop a fragment due to a stale seq?
Author (Marina): The resync table is keyed by stream id + seq window. We never drop fragments on mismatch; instead we record an offset correction and replay locally. Only fragments that fail checksum validation after correction are dropped and surfaced to metrics.
Jordan: Tests look thorough; one nit around naming in offset_map.go (suggested an exported method rename). Also please add a short explanation to sse_metrics.md for the new 'sse.fragment_offset_corrections' counter.
Aisha: Verified that OpenAI adapter changes preserve function-call sentinel ordering. I ran the repro from SUP-7783 locally and saw consistent reconstruction across multiple client replay strategies.
Samir (approval): LGTM after CI green and the rename addressed. approving for squash merge.
Fixes a class of SSE fragmentation bugs where interleaved heartbeats and skewed fragment offsets could break tool/function call JSON reassembly. Adds mitigation via offset-resynchronization and improves diagnostics. No API changes.
