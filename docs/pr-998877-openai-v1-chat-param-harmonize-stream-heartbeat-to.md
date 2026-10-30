# openai-v1-chat-compat-param-harmonize-stream-heartbeat-tool-negotiation

Harmonize OpenAI /v1/chat.completions compatibility surface with Redwood runtime to improve streaming stability and tool calling negotiation. This PR normalizes parameter mappings, consolidates stop token handling, maps OpenAI logprobs to the internal top_logprobs shape, and introduces a lightweight stream heartbeat plus fragment coalescer to reduce partial-json drops in adverse network conditions. Includes a handshake extension to the functioncall streaming envelope to avoid ambiguous tool invocation boundaries and a structured output sanitizer for JSON-mode responses. The change set contains unit and integration tests, updated docs for the compatibility layer, and compatibility shims to preserve behavior for legacy clients. Checklist: 1) Param normalization table and conversion layer added. 2) Stop handling unified across adapters and serving runtime. 3) Streaming heartbeat and fragment coalescing implemented. 4) Functioncall handshake and tool metadata framing added. 5) Mapping of logprob fields to OpenAI v1 shape. 6) Tests and docs updated. Related work: ENG-4512, PM-112.
feat(openai-adapter): add param normalization and alias mapping for v1 chat fields
fix(stream): introduce heartbeat sentinel and coalescer to reduce malformed fragments
feat(tooling): extend functioncall streaming envelope with handshake and boundary metadata
refactor(stop): centralize stop token processing into serving stop processor
test: add integration tests for stop arrays, role aliasing, and streaming reconnection
docs: document v1 chat compatibility decisions and migration notes
Diego Marquez: Please clarify how stop arrays containing non-string tokens are handled. Are we coercing or failing?
Asha Patel: We coerce to string following OpenAI behavior and add a defensive test. See stop_processor.cpp change.
Lina Becker: The heartbeat sentinel looks reasonable but we need to ensure it is disabled for non-streaming clients. Add a feature gate.
Asha Patel: Added runtime flag 'stream_heartbeat.enabled' and document opt-out in the compatibility doc.
Ravi Singh: Integration tests flaked locally once. Can we increase the backoff window on the reconnection test?
Asha Patel: Increased reconnection jitter and added deterministic seed for CI to reduce flakiness.
Improvements to OpenAI /v1/chat.completions compatibility. Adds param normalization, unified stop handling, streaming heartbeat and fragment coalescing for more robust streams, functioncall handshake for clearer tool boundaries, and logprob mapping to OpenAI v1 shapes. Behavioral changes are opt-in for legacy clients via compatibility flag. See docs/openai-chat-compatibility.md for migration steps.
