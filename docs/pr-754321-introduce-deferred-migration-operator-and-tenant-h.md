# introduce deferred-migration operator, tenant-level health probes, and hook-priority templates

Motivation: Customers running Redwood Private reported upgrade stalls when per-tenant migration jobs collide with init hooks during rolling upgrades. This PR provides a safer default by adding a deferred migration operator (helm-templated job + controller-side knob) and richer hook templating so operators can express priority and maintenance windows.

Goals: 1) allow tenant migrations to be scheduled after the critical init hooks finish and cluster stabilizes; 2) give Helm values to configure tenant probe templates so health checks are tenant-aware (namespace, annotation-based overrides); 3) introduce hook-priority annotations and sequencer helpers to order non-deterministic init jobs without hardcoding names.

What changed (high level):
- New templates/ deferred-migration/ operator-job.yaml: templated Job + RBAC snippets that are idempotent and keyed to tenant labels. Controlled by values.deferredMigration.enabled and values.deferredMigration.window.
- templates/_hook_helpers.tpl: added helper functions `hookPriority` and `maintenanceWindow` for composing annotations on jobs and cronjobs.
- values.schema.json: extended schema with `deferredMigration`, `tenantProbes`, and `hookPriority` sections; schema includes validation for cron expressions and duration strings.
- templates/tenant-probe.yaml: tenant-aware liveness/readiness probe templates with optional annotation overrides and nodeAffinity suggestions for on-prem node pools.
- NOTES.txt and values.example.yaml: documented operators on how to enable safe migrations and recommended window defaults for large clusters.

Files changed (illustrative):
- templates/deferred-migration/operator-job.yaml (+220 additions)
- templates/_hook_helpers.tpl (+48 additions)
- templates/tenant-probe.yaml (+120 additions)
- values.schema.json (+75 additions)
- values.example.yaml (+80 additions)
- charts/README.md (+12 additions)
- tests/helm/unit/deferred_migration_test.yaml (+40 additions)
- ci/helm-lint.yaml (updated to include schema validation)
- templates/NOTES.txt (+17 additions)

Commits:
- 7f3c1d0: add deferred migration operator and RBAC; initial tests
- 93b7a42: introduce hookPriority helper and annotation format docs
- a1e5bf0: add tenant probe templates and nodeAffinity knobs
- b4c2e88: extend values.schema.json and values.example.yaml; update helm-lint CI
- d2f909f: tweak templates after reviewer feedback; fix idempotency on Job names

Design notes:
- Deferred migration operator is intentionally thin: it's a templated Job that can optionally be deployed as a CronJob by operators. We expect larger installations to replace the Job with a controller (outside this chart) that watches for values.deferredMigration.enabled. The Job includes a guard that checks for `redwood.io/upgrade-in-progress` namespace annotation and respects `values.deferredMigration.window` before starting migrations.
- hookPriority uses numeric annotation `redwood.io/hook-priority` (0-100) to allow sequenced processing by init orchestrators; lower numbers run first. These values are advisory; chart consumers can implement leader-election watchers to enforce strict ordering.
- tenantProbes are rendered per-tenant via `{{ .Values.tenantProbes.template }}` and support annotation-based overrides so cluster-operators can opt into shorter probe timeouts for canary tenants.

Testing and CI:
- Added unit-style helm tests that render templates with different values sets: single-tenant, multi-tenant, and maintenance-window scenarios.
- CI changes: helm lint now runs schema validation; one flaky test (helm/render:deferred-migration job name collision) failed on first run; rerun succeeded. Integration tests run in the private CI pool where on-prem nodeAffinity examples are validated.

Security consideration: RBAC rules for the Job are minimal (Jobs can list/watch migrations in their tenant namespace only). Docs highlight that operators should run deferred-migration Job in a management namespace with cross-namespace RBAC if they want cluster-scoped migration orchestration.

Migration notes for operators:
- Default behavior unchanged. To enable: set values.deferredMigration.enabled=true and optionally tune values.deferredMigration.window (RFC3339 duration-like string).
- You can opt to render the Job as a CronJob by setting values.deferredMigration.cron="0 2 * * *" which will cause periodic non-blocking migration runs during off-hours.

Request for reviewers: please check the schema validation coverage and RBAC scope. Special ask to SRE: review the recommended default `window` value for clusters with >100 tenants.

Review conversation (high level):
- Jonas Park (reviewer): "Can the Job avoid naming collisions if multiple tenants request overlapping migrations?"
- Maya Kline (author): "Yes — Job name now includes tenant hash and a stable prefix; added tests to assert name length and uniqueness. See commit d2f909f."
- Arun Patel (reviewer): "We should not implicitly change nodeAffinity; operators must opt-in."
- Maya: "Agreed — nodeAffinity is off by default and the example shows how to enable it per-tenant. Updated values.example.yaml."
- Noah Chen (approver): "Schema looks good; validated helm lint locally. Approving pending CI pass."
- CI: first run failed a flaky render test; rerun passed.

Merge outcome: Squash-merged after approvals and green CI. Backported as a non-breaking patch to the 1.12.x maintenance release in a follow-up PR (not included here).

Notes for docs/ops: follow-up PR planned to add an operator-runbook that shows recommended maintenance windows for different cluster sizes and how to integrate with cluster schedulers (K8s CronJob vs external controller).

Related work: addresses part of ENG-8421 (upgrade resiliency) and ENG-8473 (tenant-level observability). Also referenced in SUP-1297 where customer reported upgrade hang causing P0 incident.

Testing commands used locally: helm template ./charts/redwood-private -f values.example.yaml --set deferredMigration.enabled=true | kubectl apply --dry-run=client -f -

If any follow-ups are requested I will open small focused PRs: (1) chart-provided controller scaffolding; (2) runbook + examples for migrating existing clusters; (3) installer option to toggle render-as-cronjob during bootstrap.

Adds optional deferred migration operator to make tenant migrations non-blocking during upgrade windows; introduces configurable tenant-level liveness/ready probe templates and hook-priority annotations. No default behavior change unless values.enableDeferredMigration=true.
Added deferred migration Job templates, hook-priority helpers, tenant-level probe templates, and extended values.schema.json to validate maintenance windows and hook priorities.
This PR was intentionally authored as a minimal-behavior change with opt-in features. The deferral path is designed so operators can enable it per-customer without changing core upgrade semantics.
