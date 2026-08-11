---
type: API
title: ingress-blueprint API
description: The generated CRD contract for each edge blueprint, derived from its values.schema.json — required fields, enums and the subchart passthrough keys.
resource: ingress-blueprint
tags:
  - krateo
  - api
  - crd
  - schema
timestamp: 2026-08-11
---

# API

Each blueprint's `values.schema.json` is its API: core-provider reads it to generate a CRD
and register the chart as a `CompositionDefinition`. This page summarizes each schema — the
files themselves are authoritative. See [Configuration](configuration.md) for the meaning of
each field.

## Agentgateway

Schema title *agentgateway edge*. `additionalProperties: false`.

- Required: `gatewayClassName`, `controllerName`, `gateway`.
- `gateway` (required `name`, `listeners`): `name` (string), `listeners` (array of objects),
  optional `addresses` (array; each item requires `value`, `type` enum
  `IPAddress`/`Hostname`/`NamedAddress`), optional `annotations` (map of strings).
- `agentgateway`, `agentgateway-crds` — passthrough objects to the upstream subcharts;
  declared because the schema is `additionalProperties: false`. Not intended for direct use.
- `global` — Helm global values.

## Cert-manager

Schema title *cert-manager*. `additionalProperties: false`. No curated fields:

- `cert-manager` — passthrough object to the upstream chart
  (`x-kubernetes-preserve-unknown-fields`).
- `global` — Helm global values.

## Cert-manager-issuers

Schema title *cert-manager and platform issuers*. `additionalProperties: false`. Required:
`internalCA`, `acme`.

- `internalCA` (required `enabled`, `additionalProperties: false`): `enabled` (boolean,
  default `true`), `name` (string, `minLength: 1`, default `krateo-selfsigned-ca`).
- `acme` (required `enabled`, `additionalProperties: false`): `enabled` (boolean, default
  `false`), `email` (string), `server` (string, default Let's Encrypt production),
  `gatewayRef.name` / `gatewayRef.namespace` (strings, `minLength: 1`).
- `acmeDns01` (optional): `enabled` (boolean), `name` (string, default `krateo-acme-dns01`),
  `email` (string), `server` (string), `cloudflare.apiTokenSecretRef` (required `name`,
  `key`).
- `global` — Helm global values.

## External-dns

Schema title *ExternalDNS*. `additionalProperties: false`. Required: `external-dns`.

- `external-dns` (required `domainFilters`, `txtOwnerId`, `env`; **not**
  `additionalProperties: false`, so extra upstream values remain reachable):
  - `provider.name` — enum `cloudflare`/`google`/`aws`/`azure`, default `cloudflare`.
  - `domainFilters` — array of strings, default `[]`.
  - `policy` — enum `sync`/`upsert-only`/`create-only`, default `upsert-only`.
  - `sources` — array, item enum `gateway-httproute`/`service`/`ingress`/`gateway-grpcroute`,
    default `[gateway-httproute, service]`.
  - `txtOwnerId` — string.
  - `env` — array of objects (each requires `name`; optional `valueFrom.secretKeyRef` with
    required `name`, `key`).
- `global` — Helm global values.

## Gateway-api-crds

Schema title *Gateway API CRDs*. `additionalProperties: false`. Only `global` (Helm global
values). No tunable fields.

## Notes

- Required fields such as `txtOwnerId` and `acme.email` are validated at render time by a
  `fail` guard rather than `minLength`, because a chart's own defaults must satisfy its schema
  for `helm lint` to pass.
- Passthrough keys named after subcharts are present so Helm's value coalescing works under
  `additionalProperties: false`; they are set by each chart's `values.yaml`, not by users.
