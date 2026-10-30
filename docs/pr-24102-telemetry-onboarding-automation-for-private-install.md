# telemetry-onboarding: automates Prometheus/Grafana/Vector for Private installs, adds Terraform module and safe rollback flow

Motivation: Private installations currently require manual wiring of telemetry stacks (Prometheus + Grafana + log shipping via Vector/Fluent Bit) which increases time-to-production and makes upgrades error-prone. This PR adds opinionated, configurable automation for observability during Private installs and upgrades, plus a documented safe rollback flow.

What this PR does (single-line summary): Introduces a Helm 'telemetry' umbrella chart, a Terraform observability module for private VPC networking and IAM wiring, installer playbooks for on-prem air-gapped installs, Grafana dashboards, PrometheusServiceMonitor manifests, alert rules, and an explicit upgrade/rollback playbook.

Details: - Helm changes: new chart at helm/telemetry that deploys Prometheus Operator (kube-prometheus-stack), Grafana with pre-provisioned dashboards, and a lightweight Vector agent for logs (configurable to use Fluent Bit). Values schema extended to allow: controlPlaneTelemetry.enabled, telemetry.persistence.storageClass, telemetry.resources.cpu/memory, telemetry.remote_write endpoints, and telemetry.disablePrometheusOperator (for clusters that provide a managed Prometheus). Includes templates for ServiceMonitor, PodMonitor, and GrafanaDataSource.
- Terraform: new module terraform/modules/observability that creates VPC endpoints, IAM roles/policies for pushing metrics/logs to optional cloud sinks, and automates DNS records used by the installer. Module is idempotent and supports existing installs via a 'migration' flag.
- Installer automation: installer/ansible/telemetry.yml playbook that wires Helm via the private chart, waits for CRDs, runs post-deploy smoke checks (metrics ingestion, datasource connectivity), and can toggle Vector vs Fluent Bit based on inventory. Added installer/upgrade/rollback.md documenting a tested rollback path that removes the telemetry chart, reverts Terraform state using a safe snapshot, and restores prior Grafana provisioning if necessary.
- Dashboards & alerts: shipped dashboards/dashboards/redwood_private_overview.json (custom panels for request latency, token consumption, KV cache hit rates, tail latency heatmap) and prometheus/rules/private_alerts.yaml with tuned thresholds for disk pressure, prometheus scrape lag, and ingestion error rates.
- Documentation: docs/private/observability.md with step-by-step deployment notes, example helmvalues snippets, and troubleshooting.

Backward compatibility and opt-in: All telemetry automation is opt-in by default (controlPlaneTelemetry.enabled = false). The installer detects existing Prometheus/Grafana deployments and will skip operator deployment when disablePrometheusOperator=true. Terraform module exposes a migration path for teams that already have cloud telemetry resources.

Upgrade & rollback strategy: 1) Dry-run Helm install into a staging namespace and run smoke checks. 2) Backup current Terraform state (automated via terraform workspace snapshot helper included). 3) Run Helm upgrade with --atomic and rollout status checks. 4) If metrics fail to appear within the timeout, the installer triggers a rollback routine that: a) rolls back the Helm release b) triggers terraform state rollback c) restores Grafana provisioning from snapshot. The playbook and docs include exact commands and timelines.

Security and secrets: Vector/Grafana secrets are injected through existing KMS/K8s secret provider interfaces used by Private installer. Terraform module supports KMS key ARNs for encrypting any cloud-side telemetry sinks. No telemetry signals are sent to Redwood by default; remote_write must be explicitly configured in values.yaml.

Testing & validation: - Unit: Helm template tests added (helm unittest) for values matrix. - Integration: CI pipeline spins up a small Kind cluster in the private CI runner and runs the playbook end-to-end (prometheus scrape + grafana provisioning) — added to ci/integ/telemetry. - Manual: smoke tested in two customer-like private environments (VPC + air-gapped). CI run green after one flaky grafana provisioning retry was hardened.

Related changes in this PR: see commits list below.

Rollout notes: Recommended to enable telemetry automation in a staging environment first and leave controlPlaneTelemetry.enabled=false for 24–48h when migrating to observe resource consumption. Default storageClass is not supplied; operator will fail fast if persistence isn't configured — installer will detect and inject a best-effort storageClass if present.

Open items: - Add support for Loki in a follow-up (ENG-1345). - Add auto-rotate retention policies for long-term metric storage (UX required).

References: Linear ENG-1298 (telemetry automation), SUP-452 (customer request for easier Private observability), helm chart upstream: kube-prometheus-stack docs.

Checklist: - [x] Helm chart scaffolded and unit-tested - [x] Terraform module with migration flag - [x] Installer playbook with rollback - [x] Dashboards and alerts committed - [x] Docs updated

- New: optional telemetry automation for Private installs (Prometheus, Grafana, Vector/Fluent Bit)
- New: terraform/modules/observability for VPC and sink wiring
- New: installer playbook with tested rollback flow
- Docs: private/observability.md with usage and troubleshooting
