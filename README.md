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
`additionalProperties: false` will reject them unless that key is declared.

### Where a curated surface can and cannot go

This distinction is the whole design, and getting it wrong fails silently:

- **`cert-manager-issuers` renders its own manifests.** `internalCA` and `acme`
  are consumed by this repo's `clusterissuers.yaml`, so they can carry any names
  we like. Only `cert-manager.installCRDs` reaches the subchart, and it is static.
- **`external-dns` configures the subchart's workload.** Nothing in this repo
  renders it, so every setting has to arrive as a subchart value — and Helm
  **cannot compute subchart values at render time**. No template, no
  `import-values` (child→parent only), no `global:` (subcharts read `.Values.x`,
  not `.Values.global.x`). A curated name at the top level is simply never read.

So `external-dns` puts everything under the `external-dns` key, using the upstream
chart's own value names. Less pretty; it takes effect.

**0.2.0 got this wrong.** It exposed `domainFilters`, `txtOwnerId` and
`credentialsSecretRef` at the top level, generating a CRD that accepted, validated
and stored all three — while the workload received none of them. external-dns
started with `DomainFilter:[] TXTOwnerID:default` and crash-looped on a missing
`CF_API_TOKEN`, and the composition still reported `Ready=True`. `helm lint`,
`helm template` and CRD generation all passed. **Verify a curated field in the
running pod, not in the CRD.** Fixed in 0.3.0, which is a breaking change to the
CR shape.

## Surface

Deliberately small. Upstream's full values surface is not exposed: anything exposed
becomes part of the generated CRD contract, and is harder to remove later than to
add. Upstream defaults apply to everything not listed. `external-dns` does not set
`additionalProperties: false` on its subchart key, so any other upstream value
remains reachable as an escape hatch.

**`external-dns`** — all under `external-dns`: `provider.name`, `domainFilters`,
`policy`, `sources`, `txtOwnerId`, `env[]`

**`cert-manager-issuers`** — `internalCA{enabled,name}`,
`acme{enabled,email,server,gatewayRef}`

### Two settings that bite

- **`policy: sync`** creates *and deletes* records to match cluster state. That is
  usually what you want for drift correction, but it also means records disappear
  when their Service or Gateway does — including during a cluster teardown. The
  default here is `upsert-only`.
- **`txtOwnerId`** must be unique per cluster when several clusters share a DNS
  zone. Two clusters with the same owner ID will overwrite each other's records.
  It is `required`, and a render-time guard rejects an empty value — `minLength`
  cannot be used, because a chart's own defaults must satisfy its schema for
  `helm lint` to pass.

## Credentials

The provider API token is **not** part of any blueprint. It is referenced from a
Secret placed out-of-band, so no token appears in a blueprint's values, in a
Composition CR, or in a rendered manifest.

For `external-dns` the reference is the upstream `env` form — for the reason in
"Where a curated surface can and cannot go", a friendlier `credentialsSecretRef`
would never reach the workload:

```yaml
external-dns:
  env:
    - name: CF_API_TOKEN          # Cloudflare; other providers differ
      valueFrom:
        secretKeyRef:
          name: cloudflare-api-token
          key: api-token
```

cert-manager needs a DNS credential only for a **DNS-01** solver, which is also
what a wildcard certificate requires. An **HTTP-01** solver through the Gateway
needs none.

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
