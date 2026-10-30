# introduce-tls-rotation-orchestrator-and-stepwise-reencrypt-operator

Motivation:
This PR implements a controlled, auditable TLS/key rotation flow for private deployments where customers use Vault + external HSM/KMS backing keys. Existing installers required manual sequence: rotate root key, rewrap sealed secrets, update certificates, and coordinate kube-system components. That process was error-prone during upgrades and didn't provide a safe dry-run or staged rollback.

What this change does (high level):
- Introduces a new controller (sealedkeys-reencryptor-controller) that can re-encrypt SealedSecrets and Helm secret manifests against a new envelope key while preserving original metadata and rotation audit entries. The operator supports a "compare-and-preview" dry-run mode that emits a ReencryptPlan CR containing diffs.
- Adds a rotation-orchestrator service (stateless) used during upgrades to coordinate: preflight checks, create rotation IAM/KMS bindings, trigger staged key promotion, initiate rewrap jobs, and verify TLS chains before cutover. It communicates over mTLS with the controller and uploads audit artifacts to a protected S3 bucket.
- Helm chart changes: pre-upgrade and post-upgrade hooks to trigger the orchestrator in dry-run and then execute the staged promotion. New values: rotation.enabled, rotation.window, rotation.dryRun, rotation.retries.
- Terraform additions: a rotation_schedule module to create Cloud resources (KMS key aliases, IAM bindings, scheduler cron, S3 artifact bucket with lifecycle rules) and outputs that feed the installer upgrade pipeline.
- Add an upgrade hook script and a rollback path that re-applies previous sealed secret envelopes in case verification fails.

Commits summary:
- feat(controller): add sealedkeys-reencryptor controller and CRD scaffolding
- feat(orchestrator): implement state machine and mTLS client/server stubs
- chore(helm): add rotation hooks and new values schema
- infra(terraform): rotation_schedule module with example usage and docs
- test: e2e harness for dry-run and staged rollout, unit tests for rewrap logic
- docs: runbook for operators and playbook for on-call during rotation windows

Files/areas changed (high level):
- helm/values/installer/rotation-values.yaml (new)
- helm/charts/installer/templates/hooks/pre-rotation-job.yaml (new)
- helm/charts/installer/templates/hooks/post-rotation-job.yaml (new)
- cmd/reencryptor/main.go (new controller entrypoint)
- api/v1alpha1/reencryptplan_types.go (new CRD types)
- internal/reencryptor/rewrap.go (rewrap implementation with envelope abstraction)
- orchestrator/service/*.go (orchestrator state machine and mTLS client)
- infra/terraform/modules/rotation_schedule/main.tf (new TF module)
- infra/terraform/modules/rotation_schedule/README.md
- test/e2e/rotation_dryrun_test.go
- docs/runbooks/tls-rotation-playbook.md

Notes on implementation details:
- The rewrap pipeline prefers a KMS envelope approach: the controller fetches the current sealed secret payload, decrypts it using the cluster's local unwrap key, then re-encrypts with the newly-provided envelope key ID (supplied by the orchestrator). For Vault-backed setups with HSM, the orchestrator will call Vault to create a key-version alias and return an alias ARN.
- To avoid double-write races during rewrap we use an optimistic annotation check: the controller will only swap the envelope if the object resourceVersion matches the read value; otherwise it retries with exponential backoff.
- All actions produce audit events persisted to a rotation artifacts bucket and an in-cluster Event with a link to the artifact. Artifact retention is 90 days by default (configurable).

Review conversation (selected threads):
- Ethan Ruiz (review): "Can we add a fast-path for small clusters that skips artifact upload to avoid S3 costs? Also request unit tests around the resourceVersion race."
  - Priya Kapoor (author): "Added a flag rotation.internal.skip_artifacts for opt-in; added unit tests for rewrap race and bumped coverage to 78%."
- Marta Silva (security review): "HSM integration looks good but we need proof that the orchestrator can't be tricked into promoting a key without operator approval."
  - Priya Kapoor: "Orchestrator requires a two-step approval when running against production alias namespaces: (1) generate promotion token, (2) operator approve token via CLI which signs a JWT consumed by the orchestrator. Added docs and an example CLI flow to docs/runbooks.")
- Lucas Grant (SRE): "CI failed on the first run due to flaky e2e (timing of helm hook). We should sequence the hook to wait for controller readiness."
  - Priya Kapoor: "Adjusted helm hooks to include readinessProbe checks and a polling backoff. Rerun CI and confirmed flake resolved."

CI summary:
- Initial run: build (pass), unit-tests (pass), lint (pass), e2e/rotation_dryrun (fail - timing), security-scan (pass)
- Fixes applied (helm hook wait + additional e2e stabilization): rerun -> all checks pass. Final CI status: pass.

Testing and verification:
- Ran local minikube dry-run: verified ReencryptPlan CR contains encrypted/diff summary and no writes were performed.
- Performed staged rollout in a staging tenant: rotate with promotion token, rewrap completed for ~120 secrets across 3 namespaces, TLS chain verified by orchestrator, cutover performed with zero downtime for HTTP traffic.
- Chaos testing: introduced transient KMS API latency; orchestrator retried and completed within escalation window.

Backward compatibility and rollback:
- Migration is additive: existing sealed secrets continue to be readable by the cluster until rewrap occurs.
- Rollback path applies the previously-stored envelope snapshot from artifacts and re-applies original sealed objects. The orchestrator flags any missing snapshots and aborts if snapshots are incomplete.

Operational runbook (short):
- Create promotion window in Terraform module and run terraform apply.
- Trigger dry-run via helm hook with rotation.dryRun=true and inspect ReencryptPlan CRs.
- Operator generates promotion token and approves when ready.
- Run orchestrator with rotation.dryRun=false. Monitor artifacts S3 and orchestrator logs.
- On failure, invoke rollback mode pointing to artifacts snapshot ID.

Follow-ups / future work: 
- Add support for multi-KMS hybrid rewrap (cross-cloud) and an idempotent rewrap index for extremely large clusters.
- Add RBAC grant tooling so customers can automate the promotion token signing via their OPA flows.

Merge outcome:
- After sign-off from security and SRE, PR was squash-merged to main on 2026-02-18. Change is behind default-off feature flag (rotation.enabled=false) to avoid surprise behavior on existing installs.

If you want to run locally: see infra/terraform/modules/rotation_schedule/README.md for a minimal demo and test/e2e/README.md for running the e2e harness.

Contact: Priya Kapoor (author) for implementation questions or to get added to the rotation operator on-call rota.
- Add TLS Rotation Orchestrator: stepwise automated rotation with dry-run and staged rollback.
- New reencrypt operator to rewrap sealed secrets against new KMS/HSM envelopes.
- Helm chart hooks for safe pre-upgrade key-sync and post-upgrade verification.
- Terraform module to schedule rotation windows and create rotation IAM/KMS bindings.
