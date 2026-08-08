# krateo-ingress-blueprint

Krateo blueprints for the edge-support layer: **DNS publication** and **certificate
issuance**. Both are things the CMP demo needed on every cluster, were installed by
hand, and were not recoverable when a cluster was rebuilt — which is the reason they
exist here as blueprints rather than as runbook steps.

| blueprint | what it installs |
|---|---|
| `external-dns` | Upstream ExternalDNS, publishing records for Gateway API `HTTPRoute`s and `Service`s to an external provider. |
| `cert-manager-issuers` | Upstream cert-manager plus the platform's ClusterIssuers: an internal self-signed CA chain, and optionally a public ACME issuer solving HTTP-01 through the Gateway API edge. |

## What a blueprint is here

A Helm chart plus a **`values.schema.json`**. core-provider reads the schema to
generate a CRD and registers the chart as a `CompositionDefinition`; users then
create Composition CRs of that generated Kind. The schema *is* the API — every key
it exposes becomes a field in the generated CRD and in the portal's form.

Both charts declare their upstream chart as a Helm **dependency** rather than
vendoring its templates. That requires one non-obvious thing: Helm places a
subchart's values under a top-level key named after the subchart, so a schema with
`additionalProperties: false` will reject them unless that key is declared. Both
schemas therefore carry the subchart key as a bare `{"type": "object"}` — the same
approach `krateo-platformops/installer` uses for its `core-provider` subchart.

## Surface

Deliberately small. Upstream's full values surface is not exposed: anything exposed
becomes part of the generated CRD contract, and is harder to remove later than to
add. Upstream defaults apply to everything not listed.

**`external-dns`** — `provider`, `domainFilters`, `policy`, `sources`,
`txtOwnerId`, `credentialsSecretRef{name,key}`

**`cert-manager-issuers`** — `internalCA{enabled,name}`,
`acme{enabled,email,server,gatewayRef}`

### Two settings that bite

- **`policy: sync`** creates *and deletes* records to match cluster state. That is
  usually what you want for drift correction, but it also means records disappear
  when their Service or Gateway does — including during a cluster teardown. The
  default here is `upsert-only`.
- **`txtOwnerId`** must be unique per cluster when several clusters share a DNS
  zone. Two clusters with the same owner ID will overwrite each other's records.
  It is `minLength: 1`, so an empty value fails the render rather than producing
  silent conflicts later.

## Credentials

The provider API token is **not** part of any blueprint. `credentialsSecretRef`
takes a `{name, key}` reference to a Secret placed out-of-band — the same shape
`KeycloakUser` uses for its credential values — so no token appears in a
blueprint's values, in a Composition CR, or in a rendered manifest.

cert-manager needs **no** DNS credential: the ACME issuer solves **HTTP-01 through
the Gateway**, not DNS-01.

## Registration

`values.ingress-blueprints.yaml` registers both as installer components behind a
`features.ingress` flag (default off), merged additively into the installer
composition's `.spec.components` — the same pattern
`krateo-acmp/assembly/installer-catalog` uses for the D25 auth blueprints.

## Release

Tag a semver (`0.1.0`); CI packages every directory containing a `Chart.yaml`,
vendors its dependencies, and pushes to `oci://ghcr.io/<owner>/charts`.

## Not included

- **agentgateway** — the Gateway API control/data plane (decision D22b). Large
  surface with its own CRDs, `GatewayClass` and HBONE mTLS config; folding it in
  would couple installer releases to work that is still settling. It stays the
  documented provisioner step in
  `krateo-acmp/assembly/hardening/transit/install/install.md`.
- **Kyverno tenant onboarding** — owned by the C2 workstream in `krateo-saas`.
