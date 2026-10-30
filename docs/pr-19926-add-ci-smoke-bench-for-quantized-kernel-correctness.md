# Add CI GPU smoke test for FP8/INT8 kernel correctness (small shape set)

Summary
Adds a lightweight GPU CI smoke test that exercises the FP8 + INT8 kernel paths and validates correctness against a baseline implementation on a small, representative shape set.

This is explicitly not a performance benchmark. It is meant to catch:
- obvious numerical blow-ups (NaNs/Infs)
- dispatch / shape-guard regressions (wrong kernel selected, missing fallback)
- alignment/stride edge cases (KV cache and packed layouts)
- GPU-arch-specific correctness differences (within configured tolerances)

Context
We are landing a bunch of FP8/INT8 kernel work during the benchmarking sprint (ENG-7421). We already have unit tests for some kernels, but they are not consistently run on real GPUs in presubmit.

This PR introduces a single, fast test that can be used as a correctness gate for quantized-kernel changes.

What’s in the smoke test
Target: //runtime/tests/quant:quant_kernel_smoke_test

The test runs a small matrix of cases (kept intentionally small for CI):
- Matmul (INT8 and FP8) cases that match common projection shapes:
  - (m,n,k) in {(16,4096,4096), (32,4096,8192), (16,8192,8192)}
  - alignment variants: k % 128 == 0 and a single “near-miss” alignment case that should force fallback
- Attention (FP8 path + baseline) cases:
  - head_dim = 128
  - num_heads in {32, 64}
  - prefill seq_len in {16, 128}
  - decode step with kv_len in {128, 512}
- Batch in {1, 4}

Checks performed per case:
- outputs are finite (no NaNs/Infs)
- max abs / relative error vs baseline within tolerance
- baseline uses deterministic settings where possible; we also seed inputs for reproducibility

Baseline + tolerances
Baseline is the existing (non-quantized) kernel path for the same op, executed in FP16. Tolerances are stored in runtime/tests/quant/quant_kernel_smoke_tolerances.h and are intentionally conservative for CI:
- FP8: slightly looser tolerance than INT8 (expected)
- INT8: tighter but still tolerant of minor implementation differences

CI integration
New job: gpu-quant-smoke
- Runner: self-hosted GPU CI pool (L4)
- Expected runtime: ~6–8 minutes wall clock
- Artifacts: test log + per-case summary (pass/fail, worst error)

Presubmit behavior:
- Default: opt-in for PRs (comment “/run gpu-quant-smoke” or label run-gpu-ci)
- Nightly: runs by default on main to ensure drift is caught even when presubmit is skipped

Rationale: We don’t want to force every PR to wait on GPU capacity, but we do want an easy, standard gate for kernel PRs and a nightly safety net.

How to run locally
1) Build container used by CI (see ci/scripts/run_gpu_quant_smoke.sh)
2) Run:
   ./ci/scripts/run_gpu_quant_smoke.sh \
     --test_filter='*QuantKernelSmoke*' \
     --cuda_visible_devices=0

Notes:
- The script pins the same deterministic flags as CI.
- If you’re on an H100/A100 and see different tolerances needed, please add a per-SKU override in quant_kernel_smoke_tolerances.h (don’t just globally loosen).

Review notes / discussion
- Grace requested we include at least one case that exercises the dispatch fallback (unsupported alignment). Added (m,n,k)=(16,4096,4096) with a deliberately misaligned stride variant.
- Rina asked about flakiness and non-deterministic reductions. We now force deterministic algorithm settings where supported and report the worst-case error (so we can see if noise is growing).
- Noah flagged that the INT8 path previously had a shape-guard bug in one branch; this test includes the shape that triggered it in the Slack thread and will fail if dispatch regresses.

CI results
- gpu-quant-smoke: PASS (L4)
- runtime-unit (cpu): PASS
- clang-tidy: PASS
- bazel test //runtime/...: PASS

Rollout / risk
No production impact. This is test + CI wiring only.

Follow-ups
- Add an H100/A100 weekly run (separate pipeline) to cover Hopper/Ampere-specific paths once capacity is allocated.
- Consider folding the quick quality sanity suite (eval-harness) into a separate non-blocking signal for model-default flips; this smoke test is strictly numerical correctness.

Related
- ENG-7422 Update benchmark harness for quantized kernels
- ENG-7424 INT8 matmul dispatch and shape guardrails
- PR-19841 INT8 matmul dispatch and guardrails
- PR-19827 FP8 attention integration behind flag
