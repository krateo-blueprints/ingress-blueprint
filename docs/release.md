---
type: Runbook
title: Releasing the ingress blueprints
description: The tag → release-tag.yaml → OCI + Marketplace-index publish flow — how each chart is stamped from the CHART_VERSION placeholder, pushed to ghcr.io/krateo-blueprints/charts and published to the operators index.
resource: ingress-blueprint
tags:
  - krateo
  - release
  - runbook
  - oci
timestamp: 2026-11-07T00:00:00+00:00
---

# Release runbook

Every directory containing a `Chart.yaml` is packaged on each semver tag and published twice,
with the same bytes, as the organisation's publishing standard requires
([krateo-blueprints/charts PUBLISHING.md](https://github.com/krateo-blueprints/charts/blob/main/PUBLISHING.md)):
to the GitHub Container Registry as an OCI Helm artifact, and to the Marketplace's `operators`
index at `https://krateo-blueprints.github.io/charts/operators`, which is what the portal
Marketplace installs from. The workflow is shape-agnostic: it discovers charts by locating each
`Chart.yaml` (skipping vendored subcharts under `charts/`), so it works regardless of the repo
layout.

## Why the operators channel

The standard decides the channel from the content: a chart with a `values.schema.json` **and** a
`CompositionDefinition` is a blueprint; every other chart goes to `operators`. None of these
charts ships a `CompositionDefinition` (the installer registers them as components), so all five
are published to `operators`. A chart that gains a `CompositionDefinition` fails the release
until the workflow also publishes the `blueprints` channel.

## How versioning works

Each `Chart.yaml` carries the literal placeholder `version: CHART_VERSION`. This is
intentional — **do not replace it in the repo.** At publish time the `release-tag.yaml`
workflow stamps the git tag into the placeholder and pins the package version to the tag, so
the source stays version-agnostic and the tag is the single source of truth.

## Steps

1. Land the change on the default branch (`main`). The `lint.yaml` workflow renders and
   lints every chart and runs the authoritative `lint-docs` gate on the docs set, on the pull
   request and again on the push to `main`; the release refuses a commit without a successful
   run.
2. Tag a semver, for example `0.4.0`, matching the pattern `[0-9]+.[0-9]+.[0-9]+`, and push
   the tag.
3. `release-tag.yaml` runs on the tag push. It refuses to start unless the tagged commit has a
   successful `lint` run and the org secret `CHARTS_PUBLISH_TOKEN` is set. Then, for every
   discovered chart, it:
   - stamps `CHART_VERSION` → the tag in `Chart.yaml`,
   - runs `helm dependency update` to vendor upstream dependencies (retried; three failures
     stop the release before anything is pushed),
   - runs `helm package --version <tag>`.

   Only when every chart has packaged does it `helm push` each to the OCI registry, and then
   the organisation's publisher (`krateo-blueprints/charts/.github/workflows/publish-chart.yaml@1`)
   adds the same archives to the `operators` index. The publisher succeeds only once the served
   index lists every chart at the tag and `helm pull --repo` installs each anonymously.

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

and the same five versions in `https://krateo-blueprints.github.io/charts/operators/index.yaml`.

The registry owner is derived from `GITHUB_REPOSITORY_OWNER` (lowercased), and the workflow
logs in to GHCR with the built-in `GITHUB_TOKEN` (needs `packages: write`).

## Wiring the new version into the Installer

Once published, bump the chart `version` for each component in the installer registration
(mirrored by `values.ingress-blueprints.yaml` in this repo) to the new tag. The components are
gated behind `features.ingress`, so an unset feature is unaffected by the bump.

## Rollback

The registry is immutable per tag: publish a new higher tag rather than overwriting an
existing one, and re-point the installer registration at it.
