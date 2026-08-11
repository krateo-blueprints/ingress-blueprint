---
type: Usage
title: Using the ingress blueprints
description: How to stand up the edge — enabling features.ingress on the Installer, wiring the Gateway name to exposure and ACME, and placing the provider credential.
resource: ingress-blueprint
tags:
  - krateo
  - usage
  - installer
  - gateway-api
timestamp: 2026-08-11
---

# Usage

The edge blueprints are meant to be driven from the Installer, not installed one Helm chart
at a time. They register as installer components behind a `features.ingress` flag (default
off), so a cluster fronting Krateo some other way — an existing ingress controller, a cloud
LB, a service mesh — is unaffected.

## Prerequisites

1. The blueprints published to an OCI registry the installer can pull (see [Release](release.md)).
2. The Gateway API CRDs served — provided by the `gateway-api-crds` blueprint, ordered first.
3. The provider API token Secret (for `external-dns`, and for a DNS-01 issuer) placed
   out-of-band, in the namespace the composition renders into. The blueprint takes a
   **reference** only; no credential is ever part of a blueprint's values or CR.

## Enable the edge

Turn on the feature gate and let the installer register and order the components. The
registration mirrors `values.ingress-blueprints.yaml` in this repo:

```yaml
features:
  ingress: true

componentValues:
  agentgateway:
    gatewayClassName: agentgateway
    controllerName: agentgateway.dev/agentgateway
    gateway:
      name: krateo-gateway
      listeners:
        - name: http
          protocol: HTTP
          port: 80
          allowedRoutes:
            namespaces:
              from: All

  cert-manager-issuers:
    internalCA:
      enabled: true
    acme:
      enabled: false
      email: ""
      gatewayRef:
        name: krateo-gateway
        namespace: krateo-system

  external-dns:
    external-dns:
      provider:
        name: cloudflare
      policy: upsert-only
      domainFilters:
        - krateo.dev
      sources:
        - gateway-httproute
        - service
      txtOwnerId: ""
      env:
        - name: CF_API_TOKEN
          valueFrom:
            secretKeyRef:
              name: cloudflare-api-token
              key: api-token
```

The installer chains the components in order: `gateway-api-crds` → `agentgateway` →
`cert-manager` → `cert-manager-issuers` / `external-dns`.

## Three names that must match

The Gateway is referenced by name from three places, and they must all agree:

- `agentgateway` — `gateway.name`
- the Installer — `exposure.gatewayRef.name`
- `cert-manager-issuers` — `acme.gatewayRef.name`

The default is `krateo-gateway`. Likewise `agentgateway`'s `controllerName` is written to both
the `GatewayClass` and the controller, and the two must be identical.

## Turn on public TLS

`internalCA` is always rendered — the platform needs a trust anchor before any public
certificate exists. Public ACME is off by default because it needs a real account email and a
reachable Gateway to answer the HTTP-01 challenge.

Enable `acme` **only once public DNS resolves to the Gateway**: HTTP-01 cannot succeed before
the hostname points at the edge that serves the challenge. Set `acme.enabled: true`, provide
`acme.email`, and add an HTTPS `:443` listener to the Gateway with a cert-manager-issued cert.
Wildcard certificates require the DNS-01 issuer (`acmeDns01`) instead — HTTP-01 cannot issue
them.

## Pin an address

Leave `agentgateway`'s `gateway.addresses` unset and the cloud assigns an ephemeral address,
which it reassigns on every cluster rebuild — silently invalidating every DNS record already
published for the old address. Pin a pre-reserved address so it survives teardown and gets
reattached. See [Configuration](configuration.md) for the field.

## Verify

Confirm the Gateway is programmed and has an address, then confirm curated fields landed in
the **running pod**, not just the CRD:

```sh
kubectl get gateway -A
kubectl get gatewayclass
kubectl get clusterissuers
kubectl -n <ns> get deploy external-dns -o yaml | grep -A3 CF_API_TOKEN
```

See [Examples](examples.md) for a minimal walkthrough.
