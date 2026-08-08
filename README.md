# krateo-ingress-blueprint

Krateo blueprints for the edge-support layer: the **Gateway** itself, **DNS publication**
and **certificate issuance**. These are things the CMP demo needed on every cluster, were
installed by hand, and were not recoverable when a cluster was rebuilt — which is the reason
they exist here as blueprints rather than as runbook steps. Per the platform principle that
**everything is a blueprint, even an external tool**, the Gateway (agentgateway) and the
Gateway API CRDs are blueprints too — so `exposure.type: Gateway` + `features.ingress` stands
up the whole edge from one Installer CR, with no BYO step.

| blueprint | what it installs |
|---|---|
| `gateway-api-crds` | The upstream Kubernetes Gateway API **standard CRDs** (`GatewayClass`, `Gateway`, `HTTPRoute`, `ReferenceGrant`, `GRPCRoute`), bundle-version v1.3.0. Shared edge infra everything below (and the installer's `HTTPRoute`s) requires. |
| `agentgateway` | Upstream **agentgateway** controller plus the platform `GatewayClass` + `Gateway` that per-component `HTTPRoute`s, external-dns and cert-manager's ACME `gatewayRef` all attach to. |
| `external-dns` | Upstream ExternalDNS, publishing records for Gateway API `HTTPRoute`s and `Service`s to an external provider. |
| `cert-manager-issuers` | Upstream cert-manager plus the platform's ClusterIssuers: an internal self-signed CA chain, and optionally a public ACME issuer solving HTTP-01 through the Gateway API edge. |

**Install order** (dep-chained in the registration below): `gateway-api-crds` →
`agentgateway` → `cert-manager-issuers` / `external-dns`. The CRDs must be served before the
Gateway; the Gateway must exist before the ACME `gatewayRef` and the `HTTPRoute`s attach to it.

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

**`agentgateway`** — `gatewayClassName`, `controllerName`, `gateway{name,listeners}`.
`controllerName` is written to both the `GatewayClass` and the controller (they must match);
`gateway.name` must equal the installer's `exposure.gatewayRef.name` and cert-manager's
`acme.gatewayRef.name`. `gateway.listeners` passes through to the `Gateway` spec (default: an
HTTP `:80` listener for ACME HTTP-01 + HTTPRoutes; add an HTTPS `:443` listener with a
cert-manager cert for production).

**`gateway-api-crds`** — none. Ships the upstream standard CRDs verbatim; upgrade by
re-vendoring `files/standard-install.yaml` and bumping the chart appVersion.

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

- **Kyverno tenant onboarding** — owned by the C2 workstream in `krateo-saas`.

> **agentgateway is now included** (`agentgateway` + `gateway-api-crds` above). It was
> previously deferred (D22b) and kept as a manual provisioner step; per the
> everything-is-a-blueprint principle it is now a blueprint, so the edge no longer has a BYO
> Gateway. The upstream agentgateway chart (`oci://ghcr.io/agentgateway/charts/agentgateway`)
> is bundled as a dependency; the blueprint adds only the `GatewayClass` + `Gateway` it does
> not ship. HBONE mTLS and the deeper controller surface stay at upstream defaults (not
> exposed on the generated CRD) until there is a reason to curate them.
