# Add FP8/INT8 model variants and default-policy stubs (server-flag controlled)

Summary: This PR extends the model registry schema to represent quantized serving variants (FP8 and INT8) alongside existing baseline variants, and adds placeholders for a default-selection policy that is controlled by server-side flags.

Motivation: As part of the FP8/INT8 kernel benchmarking sprint, we need a place to (a) register which models have quantized artifacts available per GPU family, and (b) express default eligibility without hard-coding a product decision in the registry. Runtime will use server-side flags + heuristics for rollout; the registry should only provide structured metadata (available variants, constraints, and safety guardrails).

What’s included: - Adds `quantization_mode` to variant records (`bf16`, `fp16`, `fp8`, `int8`) and a `variant_family` grouping to keep the catalog tidy. - Adds `availability` and `constraints` fields for quantized variants (supported gpu_arch list, min_cuda, known shape caveats, and “requires_calibration” semantics for INT8). - Adds `default_policy_stub` fields (no behavior change): `eligible_by_default` remains false for fp8/int8 variants, but can be overridden by server-side flags (runtime/control-plane). The stub exists so we can attach notes and per-model guardrails (e.g. “requires eval sign-off for long-context”). - Adds validation + tests for schema correctness and backwards compatibility with older consumers (unknown fields ignored).

Non-goals: - This PR does not flip any defaults in hosted production. - No registry-driven auto-selection logic is added here; runtime dispatch remains policy owner.

Rollout/compatibility: - Backwards compatible: existing baseline variant IDs are unchanged. - New fields are additive. - Consumers that deserialize strictly will need to update (runtime already tolerates unknown keys; console uses a permissive parser).

Testing: - Unit tests for schema validation and fixture decoding. - Golden fixtures updated for top hosted model entries (kept minimal; detailed policy lives in runtime flags/ADR).

Follow-ups: - After benchmarks + eval sign-off, we will populate model-specific constraints more fully and add a registry linter rule to prevent enabling fp8/int8 without required guardrails (tracked in ENG-7421).
Jonas Weber (review): Can we make sure we don’t imply “default FP8/INT8” anywhere in the registry? I want the eval sign-off criteria to remain the gate, and runtime flags to be the mechanism.

Arjun Desai (author): Agree. I’ve set `eligible_by_default=false` for all fp8/int8 variants and renamed the field group to `default_policy_stub` to make it explicit this is not an active policy engine. Added doc text: “server-side flags are policy owner; registry is metadata only.”

Claire Dubois (review): Please add a note about measurement/claim hygiene: if we expose these variants in any UI later, we’ll need the benchmarking methodology footnotes. Can we attach a freeform field for “benchmark_notes” without hard-typing it yet?

Arjun Desai (author): Added `notes` under the variant record, with a convention that this is internal-only and may include “benchmark footnotes required.” Kept it optional and unvalidated beyond type.

Yuki Sato (review): For runtime heuristics we’ll need to know supported GPU arch per variant. Your `availability.supported_gpu_arch` is good, but can we also include a min driver/CUDA constraint? We’ve seen mismatches across clusters.

Arjun Desai (author): Added `availability.min_cuda` (string) and `availability.min_driver` (string). Validator enforces they exist for fp8/int8 variants.

Rina Kobayashi (review): Requesting a small fixture: one model with both fp8 and int8 entries + a “known bad shape” caveat so we can test consumer logic.

Arjun Desai (author): Added fixture example and a validation test ensuring `constraints.known_caveats` is permitted and preserved.

Jonas Weber (final): LGTM. Please ensure docs mention that any quality gating/evals live outside the registry (Applied ML sign-off).

Arjun Desai (author): Done; updated docs/registry_fields.md with an explicit callout + link placeholder to internal policy doc (to be wired in follow-up).
Internal-only: Registry now supports FP8/INT8 variant metadata and policy stubs for server-flag controlled defaults. No production default changes.
