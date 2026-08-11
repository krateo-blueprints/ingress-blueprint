---
type: Configuration
title: ingress-blueprint configuration
description: The curated value surface exposed by each edge blueprint — agentgateway, cert-manager, cert-manager-issuers, external-dns and gateway-api-crds.
resource: ingress-blueprint
tags:
  - krateo
  - configuration
  - values
  - gateway-api
timestamp: 2026-08-11
---

# Configuration

Each chart's surface is deliberately small. Anything exposed becomes part of the generated
CRD contract, which is harder to remove later than to add, so upstream defaults apply to
everything not listed. The authoritative contract for each chart is its `values.schema.json`;
see [API](api.md). This page describes the practical surface.

## agentgateway

| Key | Type | Default | Notes |
|---|---|---|---|
| `gatewayClassName` | string | `agentgateway` | Name of the `GatewayClass` this edge creates. Also written to the `agentgateway:` passthrough. |
| `controllerName` | string | `agentgateway.dev/agentgateway` | Written to `GatewayClass.spec.controllerName` **and** the controller's `AGW_CONTROLLER_NAME`; the two must be equal or the class is never reconciled. |
| `gateway.name` | string | `krateo-gateway` | Must match the installer's `exposure.gatewayRef.name` and `cert-manager-issuers`' `acme.gatewayRef.name`. |
| `gateway.listeners` | list | one HTTP `:80` listener | Passed through verbatim to `Gateway.spec.listeners`. The default serves ACME HTTP-01 and HTTP `HTTPRoute`s; add an HTTPS `:443` listener with a cert-manager cert for production. |
| `gateway.addresses` | list | unset (ephemeral) | Optional. Pin the Gateway to a pre-reserved address so it survives a cluster rebuild. Unset lets the cloud assign an ephemeral address that changes on rebuild, invalidating published DNS records. |
| `gateway.annotations` | map | unset | Optional. Set `cert-manager.io/cluster-issuer` here to have cert-manager's gateway-shim issue a Certificate for every listener declaring `tls.certificateRefs`. |

The `agentgateway:` and `agentgateway-crds:` keys are static passthroughs to the upstream
subcharts, set by this blueprint's `values.yaml`. Keep `controllerName`/`gatewayClassName` in
sync between the top-level values and the passthrough — Helm cannot template subchart values.

## cert-manager

Installs the operator and CRDs only; there is no curated top-level surface. Two settings are
fixed in `values.yaml`:

- `cert-manager.crds.enabled: true` — CRDs ship with the operator, so `cert-manager.io/v1` is
  served before any Issuer is applied.
- `cert-manager.config.gatewayAPI.enabled: true` — turns on the gateway-shim. Without it a
  Gateway annotated with `cert-manager.io/cluster-issuer` is silently ignored and HTTPS is
  never served.

## cert-manager-issuers

| Key | Type | Default | Notes |
|---|---|---|---|
| `internalCA.enabled` | boolean | `true` | Always-on self-signed CA chain — the platform trust anchor. Certificates it issues are not publicly trusted. |
| `internalCA.name` | string | `krateo-selfsigned-ca` | Name of the CA `ClusterIssuer`. |
| `acme.enabled` | boolean | `false` | Public HTTP-01 issuer. Enable only once public DNS resolves to the Gateway. |
| `acme.email` | string | `""` | **Required** when `acme.enabled` is true; a render-time guard fails an empty value. |
| `acme.server` | string | Let's Encrypt production | ACME directory URL. |
| `acme.gatewayRef.name` / `.namespace` | string | `krateo-gateway` / `krateo-system` | The Gateway that serves the HTTP-01 challenge (Gateway API, not Ingress). |
| `acmeDns01.enabled` | boolean | `false` | Second ACME issuer solving DNS-01. Required for **wildcard** certificates, which HTTP-01 cannot issue. |
| `acmeDns01.email` | string | `""` | Required when enabled. |
| `acmeDns01.cloudflare.apiTokenSecretRef` | object | `cloudflare-api-token` / `api-token` | Reference to the out-of-band Secret holding the DNS API token, in cert-manager's cluster resource namespace. |

## external-dns

Everything lives under the `external-dns` key, using the upstream chart's own value names,
because Helm cannot carry a curated top-level field into the subchart (see
[Architecture](overview.md)).

| Key (under `external-dns.external-dns`) | Type | Default | Notes |
|---|---|---|---|
| `provider.name` | string | `cloudflare` | One of `cloudflare`, `google`, `aws`, `azure`. |
| `domainFilters` | list | `[]` | **Required.** Zones this instance may manage. An empty list allows every zone the credential can reach — strongly discouraged. |
| `policy` | string | `upsert-only` | `sync` creates **and deletes** records to match cluster state (records vanish on teardown); `upsert-only` never deletes; `create-only` only adds. |
| `sources` | list | `[gateway-httproute, service]` | Resources to derive records from. `gateway-httproute` is required for the agentgateway edge. |
| `txtOwnerId` | string | `""` | **Required**, and must be unique per cluster sharing a zone, or clusters overwrite each other's records. |
| `env` | list | `[]` | **Required.** The provider credential as an env var sourced from an out-of-band Secret. Cloudflare expects `CF_API_TOKEN`. |

The `external-dns` schema does not set `additionalProperties: false` on the subchart key, so
any other upstream value remains reachable as an escape hatch.

## gateway-api-crds

No tunable values. Ships the upstream standard CRDs (bundle-version v1.3.0) verbatim; upgrade
by re-vendoring `files/standard-install.yaml` and bumping the chart `appVersion`.
