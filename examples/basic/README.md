---
type: Example
title: Basic edge example
description: A minimal edge — features.ingress on, an HTTP :80 Gateway, the internal self-signed CA, and ExternalDNS publishing HTTPRoute and Service records. Public ACME and DNS-01 stay off.
resource: ingress-blueprint
tags:
  - krateo
  - example
  - gateway-api
  - external-dns
timestamp: 2026-08-11T00:00:00+00:00
---

# Basic edge

The smallest useful edge: a single HTTP `:80` Gateway named `krateo-gateway`, the internal
self-signed CA as the trust anchor, and ExternalDNS publishing records for Gateway API
`HTTPRoute`s and `Service`s. Public ACME (HTTP-01) and the DNS-01 issuer stay off — enable
them once public DNS resolves to the Gateway (see [Usage](../../docs/usage.md)).

## Prerequisites

- The blueprints published to an OCI registry the installer can pull (see
  [Release](../../docs/release.md)).
- A `cloudflare-api-token` Secret placed out-of-band in the namespace the `external-dns`
  composition renders into. The blueprint references it; it never carries the token.

## Apply

The edge is driven from the Installer, not chart-by-chart. Enable the feature gate and supply
the values in [`installer-values.yaml`](installer-values.yaml):

```sh
helm upgrade --install krateo-installer <installer-chart> \
  -f examples/basic/installer-values.yaml
```

The installer chains the components: `gateway-api-crds` → `agentgateway` → `cert-manager` →
`cert-manager-issuers` / `external-dns`.

## Verify

```sh
kubectl get gatewayclass
kubectl get gateway -A                 # krateo-gateway should be Programmed with an address
kubectl get clusterissuers             # krateo-selfsigned-ca present
kubectl -n <ns> get deploy external-dns -o yaml | grep -A3 CF_API_TOKEN
```

Confirm the curated `external-dns` fields (for example `txtOwnerId`) landed in the running
pod, not just the CRD — that is the check version 0.2.0 taught us to make. See
[Configuration](../../docs/configuration.md) for every field.
