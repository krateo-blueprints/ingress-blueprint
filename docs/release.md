---
type: Runbook
title: Releasing the ingress blueprints
description: The tag → release-tag.yaml → OCI publish flow — how each chart is stamped from the CHART_VERSION placeholder and pushed to ghcr.io/krateo-blueprints/charts.
resource: ingress-blueprint
tags:
  - krateo
  - release
  - runbook
  - oci
timestamp: 2026-08-11
---

# Release runbook

Every directory containing a `Chart.yaml` is packaged and published to the GitHub Container
Registry as an OCI Helm artifact on each semver tag. The workflow is shape-agnostic: it
discovers charts by locating each `Chart.yaml` (skipping vendored subcharts under `charts/`),
so it works regardless of the repo layout.

## How versioning works

Each `Chart.yaml` carries the literal placeholder `version: CHART_VERSION`. This is
intentional — **do not replace it in the repo.** At publish time the `release-tag.yaml`
workflow stamps the git tag into the placeholder and pins the package version to the tag, so
the source stays version-agnostic and the tag is the single source of truth.

## Steps

1. Land the change on the default branch (`blueprints`). The `lint.yaml` workflow renders and
   lints every chart and runs the authoritative `lint-docs` gate on the docs set.
2. Tag a semver, for example `0.4.0`, matching the pattern `[0-9]+.[0-9]+.[0-9]+`, and push
   the tag.
3. `release-tag.yaml` runs on the tag push. For every discovered chart it:
   - stamps `CHART_VERSION` → the tag in `Chart.yaml`,
   - runs `helm dependency update` to vendor upstream dependencies,
   - runs `helm package --version <tag>`,
   - runs `helm push` to the OCI registry.

## Where charts land

```
oci://ghcr.io/krateo-blueprints/charts/<chart>:<tag>
```

For example, tag `0.4.0` publishes:

- `oci://ghcr.io/krateo-blueprints/charts/gateway-api-crds:0.4.0`
- `oci://ghcr.io/krateo-blueprints/charts/agentgateway:0.4.0`
- `oci://ghcr.io/krateo-blueprints/charts/cert-manager:0.4.0`
- `oci://ghcr.io/krateo-blueprints/charts/cert-manager-issuers:0.4.0`
- `oci://ghcr.io/krateo-blueprints/charts/external-dns:0.4.0`

The registry owner is derived from `GITHUB_REPOSITORY_OWNER` (lowercased), and the workflow
logs in to GHCR with the built-in `GITHUB_TOKEN` (needs `packages: write`).

## Wiring the new version into the Installer

Once published, bump the chart `version` for each component in the installer registration
(mirrored by `values.ingress-blueprints.yaml` in this repo) to the new tag. The components are
gated behind `features.ingress`, so an unset feature is unaffected by the bump.

## Rollback

The registry is immutable per tag: publish a new higher tag rather than overwriting an
existing one, and re-point the installer registration at it.
