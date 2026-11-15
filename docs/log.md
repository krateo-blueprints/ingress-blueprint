---
type: Log
title: ingress-blueprint changelog
description: Notable documentation changes for the edge blueprint bundle.
resource: ingress-blueprint
tags:
  - krateo
  - changelog
  - log
timestamp: 2026-11-15T00:00:00+00:00
---

# Changelog

## Unreleased

- `cert-manager-issuers`: the internal root CA keeps its private key across renewals
  (`privateKey.rotationPolicy: Never`, `duration: 87600h`, `renewBefore: 8760h`). On cert-manager's
  defaults (90 days, and since v1.18 a new key at every re-issue) the root re-keyed at each
  renewal. `internal-ca-bundle` then switched clients to the new root while every certificate it
  had signed stayed on the old key until its own renewal, so for that gap every client refused
  them. On clusters already running the chart the upgrade re-issues nothing: the current 90-day
  root stays until its scheduled renewal, which keeps the key and issues a 10-year certificate
  (measured on cert-manager v1.21.1). Values are unchanged. Changing the key is now a documented
  procedure.

- Added the `trust-manager` chart: upstream trust-manager v0.20.0 as a dependency, versioned
  by this repository's tag. jetstack's chart version `v0.20.0` made the installer's
  CompositionDefinition serve `vv0-20-0` while the installer looked for `v0-20-0`, so
  trust-manager was silently never installed. `trust-manager.app.trust.namespace` and
  `trust-manager.app.targetNamespaces` are typed (the latter as unique DNS-1123 labels; upstream
  leaves its items untyped); every other upstream value passes through.
- Releases also publish every chart to the Marketplace's `operators` index
  (`https://krateo-blueprints.github.io/charts/operators`) through the organisation's publisher,
  with the same archives pushed to GHCR. A release refuses to start without a successful `lint`
  run on the tagged commit and the `CHARTS_PUBLISH_TOKEN` secret, and every chart must list this
  repository in `sources`, so the upstream wrappers now name it first. See
  [release.md](release.md).

## 2026-11-15 (0.16.0)

- Releases stamp `org.opencontainers.image.{revision,source,version,ref.name}` into every
  chart and refuse to push a package that doesn't carry the tag's commit.
- `cert-manager-issuers` and `internal-ca-bundle` list this repository as their `sources`.
  They render only cert-manager/trust-manager resources and ship no upstream code, so the
  upstream project was not a build input; it stays named in their descriptions. Values and
  templates are unchanged.

## 2026-11-15

- Added the `internal-ca-bundle` chart: one trust-manager `Bundle` copying the internal CA's
  public certificate (`krateo-root-ca-key-pair`, key `ca.crt`) into a ConfigMap in each
  namespace listed in the required `target.namespaces` (authn#145). It is a chart of its own so
  the installer can order it after trust-manager's API.
- `cert-manager-issuers`: `internalCA.namespace` sets where the root CA `Certificate` and its
  Secret are written (default: the release namespace, unchanged). It must be cert-manager's
  cluster resource namespace; with cert-manager installed elsewhere, the CA `ClusterIssuer`
  otherwise never becomes Ready.

## 2026-08-11

- Added the Krateo Open Knowledge Format (OKF) documentation set: `index`, `overview`,
  `usage`, `configuration`, `api`, `examples`, `release`, `log`, and `llms.txt`, plus the
  `basic` example. Content grounded in the repo's five charts (`gateway-api-crds`,
  `agentgateway`, `cert-manager`, `cert-manager-issuers`, `external-dns`) and their
  `values.schema.json` contracts.
