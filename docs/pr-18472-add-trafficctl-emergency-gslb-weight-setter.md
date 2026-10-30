# Add trafficctl emergency GSLB weight setter for api.redwoodinference.com

### Context
During game day and recent regional brownouts, the fastest safe mitigation is to shift Hosted API traffic between regions without waiting for Terraform applies. We currently do this via manual console operations, which is error-prone.

### What this PR does
- Introduces `trafficctl`, an audited CLI wrapper around our GSLB provider API
- Adds commands to:
  - read current targets + weights
  - set weights for api.redwoodinference.com
  - set record TTL (temporary)
- Enforces RBAC via the `infra-traffic-admin` role
- Writes an audit log event for every invocation (who, what record, old/new weights)

### Usage
```
trafficctl gslb get --record api.redwoodinference.com
trafficctl dns set-ttl --record api.redwoodinference.com --ttl 60
trafficctl gslb set-weights --record api.redwoodinference.com --weights us-east-1-gw=0,us-west-2-gw=100
trafficctl dns set-ttl --record api.redwoodinference.com --ttl 300
```

### Rollback
- Disable the `trafficctl` GitHub Actions release workflow and revert to manual console operations
- No data-plane changes are required to roll back this PR

### Changed areas
- `cmd/trafficctl/*`
- `pkg/gslb/*`
- `pkg/auditlog/*`
- `rbac/policies/infra-traffic-admin.yaml`
- `docs/runbooks/global-traffic-routing-controls.md` (synced to Confluence by docs bot)

### Review notes
- Hannah: please verify audit event fields match incident review needs
- Alex: verify gateway target naming conventions (us-east-1-gw, us-west-2-gw)

### Review conversation (excerpt)
Hannah Schmitt: Can we ensure we log *previous weights* as well as the new weights? That's critical for rollback during an incident.

Dev Patel: Yep, updated `auditlog.Event` to include `old_weights` and `new_weights` as strings (serialized).

Ethan Park: Any safety rails to prevent accidental 100% to an unknown target?

Dev Patel: Added validation: target must be in the allowlist for the record; otherwise command aborts.

Alex Martinez: Looks good. Please add a short note that TTL should be restored after the cutover.

Dev Patel: Added to docs + examples.

### Merge
- Squash merged into main
- CI: build/test/lint passed

Internal: `trafficctl` CLI for emergency Hosted API cross-region traffic cutover (audited + RBAC guarded).
