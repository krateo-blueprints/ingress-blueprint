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
