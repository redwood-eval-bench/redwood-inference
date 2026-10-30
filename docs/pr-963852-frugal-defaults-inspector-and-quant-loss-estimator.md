# Add frugal defaults inspector, tenant dry-run and quantization loss estimator

Motivation: customers repeatedly ask for safer, tenant-specific defaults for batching, caching and quantization that minimize unit cost without silently degrading quality. This PR introduces a lightweight inspector workflow (CLI + API) to dry-run candidate defaults against historical traces, a quantization loss estimator that predicts aggregate quality delta for a given quant profile, and a small Redwood Console preview for unit-econ impact. Changes are intentionally conservative: nothing auto-applies without an explicit tenant opt-in or a dry-run approval.

Summary of changes: add a 'frugal defaults inspector' CLI command (redwoodctl defaults inspect --tenant <id> --dry-run) that replays sampled traces and reports latency, token-cost, and estimated quality shift; embed a quant-loss estimator service that runs offline evals and returns a numeric risk score for each quant profile; surface a UnitEcon preview in the Console route /tenant/{id}/defaults-preview showing expected cost delta, 95th pct latency impact, and loss-estimator score; wire new telemetry into the existing cost-sim pipeline so ROI graphs can show proposed defaults. The description below lists commits, files changed, review threads, and CI checks.
Diego: Can the inspector scale when a tenant has millions of traces? Please document sampling guarantees and approximations.
Author (Maya): Inspector uses stratified sampling and a max-simulated-requests cap; added docs and clarified error bounds.
Renee: Quant-loss-estimator looks promising but how do we calibrate it to perceptual quality? Is it just token-level perplexity?
Author: Estimator runs lightweight evals using our existing eval harness and BLEU/ROUGE-like metrics for structured outputs and an internal human-calibrated mapper for chat quality; added explanation and a 'risk-to-action' mapping in the docs.
Owen: UI load could leak tenant traces; ensure PII is not surfaced in previews.
Author: previews only display aggregate stats and hashed sample ids, we added an RBAC gate and sanitized the sample pipeline.
Follow-ups: add a compact 'approve' flow to apply defaults (separate PR), and an operator-runbook for long dry-runs.
Introduces a Defaults Inspector (CLI + API + Console preview) and a quantization loss estimator to help teams safely evaluate tenant-specific batching, caching, and quant profiles. Nothing auto-applies; operator approval is required. See docs/defaults-inspector.md for usage and safety guidance.
