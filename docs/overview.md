---
type: Architecture
title: ingress-blueprint architecture
description: How the edge blueprints fit together — the Gateway, DNS and certificate charts, their install ordering, and the curated-surface constraints that shaped the design.
resource: ingress-blueprint
tags:
  - krateo
  - architecture
  - gateway-api
  - cert-manager
timestamp: 2026-08-11
---

# Architecture

The edge-support layer is the set of things every cluster needed on top of the base
platform — a Gateway to front components, DNS to make their hostnames resolvable, and
certificates to serve them over TLS. These were installed by hand for the CMP demo and were
not recoverable when a cluster was rebuilt. Packaging them as blueprints makes the edge
reproducible: `exposure.type: Gateway` plus `features.ingress` stands up the whole thing from
one Installer CR.

## What a blueprint is here

A blueprint is a Helm chart plus a `values.schema.json`. core-provider reads the schema to
generate a CRD and registers the chart as a `CompositionDefinition`; users then create
Composition CRs of the generated Kind. **The schema is the API** — every key it exposes
becomes a field on the generated CRD and in the portal form.

Each chart that wraps an upstream project declares it as a Helm **dependency** rather than
vendoring its templates. Helm places a subchart's values under a top-level key named after
the subchart, so a schema with `additionalProperties: false` must declare that key or the
subchart's values are rejected.

## The five charts

- **`gateway-api-crds`** — ships the upstream Gateway API standard CRDs (bundle-version
  v1.3.0) verbatim. Shared edge infrastructure the Gateway, the `HTTPRoute`s and the ACME
  `gatewayRef` all require. No tunable values.
- **`agentgateway`** — bundles the upstream agentgateway controller (and its CRDs chart) and
  renders the platform `GatewayClass` + `Gateway`, which upstream does not ship. The
  `controllerName` is written to both the `GatewayClass` and the controller; they must match,
  or the controller never reconciles the class.
- **`cert-manager`** — installs the upstream cert-manager operator and CRDs, with Gateway API
  support turned on so the gateway-shim can issue certificates for annotated Gateways.
- **`cert-manager-issuers`** — renders the platform ClusterIssuers: an always-on internal
  self-signed CA chain, an optional public ACME (HTTP-01) issuer, and an optional ACME
  (DNS-01) issuer for wildcards. Depends on `cert-manager`; it deliberately does not install
  it.
- **`external-dns`** — configures the upstream ExternalDNS workload to publish records for
  Gateway API `HTTPRoute`s and `Service`s to an external provider.

## Install ordering

The installer registration dependency-chains the components:

```
gateway-api-crds → agentgateway → cert-manager → cert-manager-issuers / external-dns
```

Ordering is enforced across components, not inside a release, for a reason. cert-manager is
its **own** component, separate from the Issuers, because a single Helm release applies its
manifests in one pass — so `cert-manager.io` CRDs delivered by a subchart are not established
when the same release's `ClusterIssuer` templates are applied, failing with
`no matches for kind "ClusterIssuer" in version "cert-manager.io/v1"`. Splitting the operator
from the Issuers turns a race inside one release into an ordering between two components,
which the installer already models.

## Where a curated surface can and cannot go

This distinction is the whole design, and getting it wrong fails silently.

- **`cert-manager-issuers` renders its own manifests.** `internalCA`, `acme` and `acmeDns01`
  are consumed by this repo's `clusterissuers.yaml`, so they can carry any names the platform
  likes.
- **`external-dns` configures the subchart's workload.** Nothing in this repo renders it, so
  every setting must arrive as a subchart value — and Helm cannot compute subchart values at
  render time (no template, no `import-values` beyond child→parent, no `global:` route). A
  curated name at the top level is simply never read.

So `external-dns` exposes everything under the `external-dns` key using the upstream chart's
own value names. Version 0.2.0 got this wrong — it exposed `domainFilters`, `txtOwnerId` and
`credentialsSecretRef` at the top level, generating a CRD that accepted and stored all three
while the workload received none, and the composition still reported `Ready=True`. The lesson
is baked into the schemas now: **verify a curated field in the running pod, not in the CRD.**

## Credentials

Provider API tokens are never part of any blueprint. They are referenced from a Secret placed
out-of-band, so no token appears in a blueprint's values, a Composition CR, or a rendered
manifest. `external-dns` uses the upstream `env` form; `cert-manager-issuers`' DNS-01 issuer
takes an `apiTokenSecretRef`. An HTTP-01 solver through the Gateway needs no credential at
all.
