# krateo-ingress-blueprint

Krateo blueprints for the edge-support layer: the **Gateway** itself, **DNS publication**
and **certificate issuance**. These are things the CMP demo needed on every cluster, were
installed by hand, and were not recoverable when a cluster was rebuilt — which is the reason
they exist here as blueprints rather than as runbook steps.

## What is this

A bundle of Krateo blueprints — each a Helm chart plus a `values.schema.json` that
core-provider turns into a `CompositionDefinition`. Per the platform principle that
**everything is a blueprint, even an external tool**, the Gateway (agentgateway) and the
Gateway API CRDs are blueprints too, so `exposure.type: Gateway` + `features.ingress` stands
up the whole edge from one Installer CR, with no BYO step.

| blueprint | what it installs |
|---|---|
| `gateway-api-crds` | The upstream Kubernetes Gateway API **standard CRDs** (`GatewayClass`, `Gateway`, `HTTPRoute`, `ReferenceGrant`, `GRPCRoute`), bundle-version v1.3.0. |
| `agentgateway` | Upstream **agentgateway** controller plus the platform `GatewayClass` + `Gateway`. |
| `cert-manager` | Upstream **cert-manager** operator + CRDs, with Gateway API support on. |
| `cert-manager-issuers` | The platform's **ClusterIssuers**: an internal self-signed CA chain and optional public ACME issuers. |
| `external-dns` | Upstream **ExternalDNS**, publishing records for Gateway API `HTTPRoute`s and `Service`s. |

A blueprint is a Helm chart plus a `values.schema.json`. The schema *is* the API — every key
it exposes becomes a field in the generated CRD and in the portal's form. Each chart wrapping
an upstream project declares it as a Helm **dependency** rather than vendoring its templates.

## Install

The blueprints register as installer components behind a `features.ingress` flag (default
off), dependency-chained in this order:

```
gateway-api-crds → agentgateway → cert-manager → cert-manager-issuers / external-dns
```

The CRDs must be served before the Gateway; the Gateway must exist before the ACME
`gatewayRef` and the per-component `HTTPRoute`s attach to it. `values.ingress-blueprints.yaml`
in this repo is the reference registration. See [docs/usage.md](docs/usage.md) for the full
flow.

## Configure

Each chart's surface is deliberately small — anything exposed becomes part of the generated
CRD contract. The Gateway is referenced by name from three places that must all agree:
`agentgateway`'s `gateway.name`, the installer's `exposure.gatewayRef.name`, and
`cert-manager-issuers`' `acme.gatewayRef.name` (default `krateo-gateway`).

`external-dns` puts everything under the `external-dns` key using the upstream chart's own
value names, because Helm cannot compute subchart values at render time — a curated top-level
field would be accepted by the CRD and then silently ignored (the 0.2.0 bug). **Verify a
curated field in the running pod, not in the CRD.** Provider API tokens are never part of a
blueprint; they are referenced from a Secret placed out-of-band.

Full reference: [docs/configuration.md](docs/configuration.md) and
[docs/api.md](docs/api.md).

## Examples

- [examples/basic](examples/basic/README.md) — a minimal edge: `features.ingress` on, an HTTP
  `:80` Gateway, the internal self-signed CA, and ExternalDNS publishing `HTTPRoute` and
  `Service` records.

Index: [docs/examples.md](docs/examples.md).

## Docs

The full documentation set lives under [docs/](docs/index.md):

- [Overview](docs/overview.md) — architecture and design constraints.
- [Usage](docs/usage.md) — enabling `features.ingress`.
- [Configuration](docs/configuration.md) — the value surface, per chart.
- [API](docs/api.md) — the generated CRD contract.
- [Examples](docs/examples.md) — runnable examples.
- [Release](docs/release.md) — the tag → OCI publish flow.
- [Changelog](docs/log.md).

## Develop & release

Each `Chart.yaml` carries the literal placeholder `version: CHART_VERSION`; do not replace it
in the repo. On a semver tag, `release-tag.yaml` discovers every chart (skipping vendored
subcharts under `charts/`), stamps the tag into the placeholder, vendors dependencies,
packages, and pushes each to:

```
oci://ghcr.io/krateo-blueprints/charts/<chart>:<tag>
```

Pull requests run `lint.yaml`, which renders and lints every chart and runs the authoritative
`lint-docs` gate on the docs set. See [docs/release.md](docs/release.md) for the runbook.
