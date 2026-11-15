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
timestamp: 2026-11-15T00:00:00+00:00
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
  default `true`), `name` (string, `minLength: 1`, default `krateo-selfsigned-ca`),
  `namespace` (string, a DNS-1123 label or empty, `maxLength: 63`, default `""`).
- `acme` (required `enabled`, `additionalProperties: false`): `enabled` (boolean, default
  `false`), `email` (string), `server` (string, default Let's Encrypt production),
  `gatewayRef.name` / `gatewayRef.namespace` (strings, `minLength: 1`).
- `acmeDns01` (optional): `enabled` (boolean), `name` (string, default `krateo-acme-dns01`),
  `email` (string), `server` (string), `cloudflare.apiTokenSecretRef` (required `name`,
  `key`).
- `global` — Helm global values.

## Trust-manager

Schema title *trust-manager*. `additionalProperties: false`. No `required` at any depth.

- `trust-manager` — passthrough object to the upstream chart
  (`x-kubernetes-preserve-unknown-fields`), with two typed fields:
  - `app.trust.namespace` — string, a DNS-1123 label, `maxLength: 63`.
  - `app.targetNamespaces` — array of DNS-1123 labels (`maxLength: 63`), `uniqueItems`. The
    generated CRD types its items as strings (upstream's own schema leaves them untyped).
  `app` and `app.trust` keep `x-kubernetes-preserve-unknown-fields`, so every other upstream
  value under them passes through.
- `global` — Helm global values.

## Internal-ca-bundle

Schema title *Internal CA trust bundle*. `additionalProperties: false`. Required: `name`,
`target`.

- `name` — string, a DNS-1123 subdomain, `maxLength: 253`, default `krateo-internal-ca`.
- `target` (required `key`, `namespaces`, `additionalProperties: false`): `key` (string,
  pattern `^[-._a-zA-Z0-9]+$`, default `ca.crt`), `namespaces` (array of DNS-1123 labels,
  `uniqueItems`). An empty `namespaces` passes the schema and fails the render (see Notes).
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

- Required fields such as `txtOwnerId`, `acme.email` and `internal-ca-bundle`'s
  `target.namespaces` are validated at render time by a
  `fail` guard rather than `minLength`, because a chart's own defaults must satisfy its schema
  for `helm lint` to pass.
- Passthrough keys named after subcharts are present so Helm's value coalescing works under
  `additionalProperties: false`; they are set by each chart's `values.yaml`, not by users.
