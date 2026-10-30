# Add optional VPN/DX placeholders and routing hooks (no provider assumptions)

Title: Add optional VPN/DX placeholders and routing hooks (no provider assumptions)

Context
Design partners keep asking, "Where do we plug in our VPN / Direct Connect / Transit Gateway routes?" Our alpha modules intentionally avoid building VPN/DX/TGW resources (provider + enterprise topology specific), but we also need to stop forcing customers to fork the VPC module just to add a couple of routes.

This PR adds:
- Stable outputs for route table IDs and subnet groupings so customers can attach routes externally (VPN/TGW/DX) without modifying our modules.
- Optional, generic "routing hooks" inputs for adding additional routes to the private/public route tables when the target is an ID we already know (e.g., a pre-existing TGW attachment, VGW, or NAT instance) while keeping the module free of assumptions about what creates that target.
- Docs + example showing recommended patterns (customer-owned networking module creates VPN/TGW, then consumes our outputs to add routes).

Scope / non-goals
- NOT creating aws_vpn_gateway / aws_customer_gateway / aws_dx_gateway / aws_ec2_transit_gateway or attachments.
- NOT standardizing on any single enterprise connectivity topology.
- NOT attempting to manage propagation/associations for TGW route tables (customers vary widely).

Implementation details
1) New outputs in modules/vpc
- outputs for:
  - vpc_id
  - private_route_table_ids_by_az
  - public_route_table_ids_by_az
  - private_subnet_ids_by_az
  - public_subnet_ids_by_az
  - intra_subnet_ids_by_az (if enabled)
  - azs

These are intentionally shaped for common enterprise patterns (per-AZ route tables) and are stable identifiers for external modules.

2) Optional route injection (generic)
- Added variable extra_routes with schema:
  - destination_cidr (string)
  - route_table_scope ("private"|"public"|"intra")
  - target_type ("tgw"|"vgw"|"eni"|"nat_instance"|"gateway")
  - target_id (string)

We keep it as a simple switch to set the right aws_route attribute (transit_gateway_id, gateway_id, network_interface_id, etc.) without creating the target resource.

Important: We validate that only one supported target type is set per route entry and we hard-fail on "unknown" to avoid silent misroutes.

3) Example + docs
- Added examples/private-vpc-with-customer-connectivity/
  - shows a placeholder customer module creating a TGW attachment (commented/pseudocode) and then adding routes via our outputs.
  - callouts for: DNS expectations, ECR endpoints, and making sure installer can reach required services.

Why this shape?
- SE feedback (Markus/Colin): customers want a clear "attachment point" without us dictating whether it’s VPN, DX, TGW, or something else.
- Security feedback (Marcus): we should not implicitly open egress or create VPN constructs; keep connectivity owned by customer networking team.
- Ops feedback (Nina): avoid forcing route table recreation (high risk) and avoid any behavior that changes NAT strategy.

Files changed (high level)
- modules/vpc/variables.tf (extra_routes input + validation)
- modules/vpc/outputs.tf (route table + subnet outputs)
- modules/vpc/routes.tf (optional aws_route resources per scope)
- examples/private-vpc-with-customer-connectivity/main.tf
- examples/private-vpc-with-customer-connectivity/README.md
- docs/extension-points.md (new section: Customer connectivity / VPN / DX hooks)

Testing
- terraform fmt / validate
- tflint
- checkov (no new high severity findings)
- integration test harness: ephemeral VPC apply/destroy (routing hooks exercised with dummy ENI target in test)

Notes for reviewers
- I intentionally did not add any new required variables. Default behavior is unchanged.
- Outputs are additive; no renames.
- Route injection only creates routes when extra_routes is non-empty.

Review conversation (abridged)

Markus Klein (Solutions Eng): Can we make the docs explicit that this is where DX/VPN plugs in? Our pilots keep asking for "Direct Connect support" and we need to explain we support it via hooks, not by creating it.
Bruno Silva (author): Yep. Added a short section in docs/extension-points.md and an example directory. Also added wording: "customer-owned connectivity resources".

Victor Alvarez (Networking): output names should be consistent with endpoints module naming. Prefer *_ids_by_az and also provide azs output so consumers don’t have to recompute.
Bruno Silva: Updated outputs to *_ids_by_az and added azs.

Marcus Lin (Security): extra_routes target_type list is OK, but make sure we validate we don’t accidentally set multiple route target fields. Also, call out that we don’t manage route propagation.
Bruno Silva: Added input validation and README note. No propagation logic.

Nina Petrova (Private Deployments): Please ensure we don’t force route table replacement for existing users when they adopt this module version.
Bruno Silva: Confirmed in plan output: additive only. No diff unless extra_routes set.

Irene Choi (Infra): CI runtime increased by ~2 min from the new integration test case. Fine for now but keep it minimal.
Bruno Silva: Acknowledged; kept it to a single route injection case.

CI
- redwood-terraform / validate: pass
- tflint: pass
- checkov: pass
- integration-tests (ephemeral-vpc): pass

Merge
- Squashed and merged to main.
- Follow-up: I’ll open a small PR to add a snippet to the main VPC module README linking to docs/extension-points.md (missed in this PR).
