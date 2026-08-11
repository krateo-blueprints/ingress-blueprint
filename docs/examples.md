---
type: ExampleIndex
title: ingress-blueprint examples
description: Index of runnable examples for the edge blueprints — starting with a minimal internal-CA edge on an HTTP :80 Gateway.
resource: ingress-blueprint
tags:
  - krateo
  - examples
  - gateway-api
timestamp: 2026-08-11
---

# Examples

Runnable examples for the edge blueprints. Each lives under `examples/<name>/` with a
`README.md` and the manifests it references.

| Example | What it shows |
|---|---|
| [basic](../examples/basic/README.md) | A minimal edge: `features.ingress` on, an HTTP `:80` Gateway, the internal self-signed CA, and ExternalDNS publishing `HTTPRoute` and `Service` records. Public ACME and DNS-01 stay off. |

## Running an example

Examples are applied through the Installer, not chart-by-chart. Enable `features.ingress` and
supply the `componentValues` shown in the example. See [Usage](usage.md) for the full flow and
the three Gateway names that must match.
