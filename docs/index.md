---
type: Component
title: ingress-blueprint
description: Krateo blueprint bundle for the platform edge — Gateway, DNS publication and certificate issuance, stood up from one Installer CR behind features.ingress.
resource: ingress-blueprint
tags:
  - krateo
  - ingress
  - gateway-api
  - blueprint
timestamp: 2026-08-11
---

# ingress-blueprint

`ingress-blueprint` is a Krateo blueprint bundle for the **edge-support layer**: the
Gateway itself, DNS publication, and certificate issuance. Each directory that contains a
`Chart.yaml` is a Helm chart plus a `values.schema.json`; core-provider reads the schema,
generates a CRD, and registers the chart as a `CompositionDefinition`. Enabling
`features.ingress` on the Installer stands up the whole edge from a single Installer CR — no
by-hand step, no BYO Gateway.

The bundle follows the platform principle that *everything is a blueprint, even an external
tool*: the Gateway (agentgateway) and the Gateway API CRDs are packaged as blueprints too,
wrapping the upstream charts as dependencies rather than vendoring their templates.

## The charts

| Chart | Installs | Upstream |
|---|---|---|
| `gateway-api-crds` | The Kubernetes Gateway API **standard CRDs** (`GatewayClass`, `Gateway`, `HTTPRoute`, `ReferenceGrant`, `GRPCRoute`), bundle-version v1.3.0. | kubernetes-sigs/gateway-api |
| `agentgateway` | The upstream **agentgateway** controller plus the platform `GatewayClass` + `Gateway`. | agentgateway/agentgateway |
| `cert-manager` | Upstream **cert-manager** operator + CRDs (with Gateway API support on). | cert-manager/cert-manager |
| `cert-manager-issuers` | The platform's **ClusterIssuers**: an internal self-signed CA chain and optional public ACME issuers. | cert-manager/cert-manager |
| `external-dns` | Upstream **ExternalDNS**, publishing records for Gateway API and Service resources. | kubernetes-sigs/external-dns |

**Install order** (dependency-chained in the installer registration):
`gateway-api-crds` → `agentgateway` → `cert-manager` → `cert-manager-issuers` /
`external-dns`. The CRDs must be served before the Gateway; the Gateway must exist before the
ACME `gatewayRef` and the per-component `HTTPRoute`s attach to it.

## Documentation

- [Overview](overview.md) — architecture, the charts, and the design constraints.
- [Usage](usage.md) — enabling `features.ingress` and driving the edge from the Installer.
- [Configuration](configuration.md) — the curated value surface, per chart.
- [API](api.md) — the generated CRD contract (from each `values.schema.json`).
- [Examples](examples.md) — index of runnable examples.
- [Release](release.md) — the tag → OCI publish flow.
- [Changelog](log.md) — notable documentation changes.
