# T2 — per-tenant token credits

**Goal (epic T2):** a trial gets a finite credit; when it is spent, further model calls stop
with a clear message, not a mysterious error. Correctness here is a **billing property**, so
this doc states exactly what is guaranteed and what is not.

## What the gateway gives us, verified (not assumed)

Checked against the live CRD (`agentgatewaypolicies.agentgateway.dev`, controller
**v1.4.1**). The epic brief assumed upstream provided "RateLimitType::Tokens (pre-charge +
amend, hard stop)" and "an llm_cost catalog". **Neither exists in this version:**

- `traffic.rateLimit.global` → an external rate-limit service (`backendRef`), with CEL
  `descriptors` (per-tenant keying) and a per-request `cost` CEL. **This is the standard
  Envoy RLS gRPC protocol.**
- `unit: Tokens` cost is **evaluated after the request completes** — the CRD says verbatim
  *"token-based rate limits apply to future requests only."* There is **no pre-charge**.
- There is **no built-in price catalog**; currency cost must be a CEL expression over the
  `llm` token attributes.

### Consequence: bounded overshoot, not zero overshoot

Because cost is known only post-request, enforcement is *"block the **next** request once the
tenant's cumulative cost has crossed the budget."* A single in-flight request can therefore
overshoot the budget by at most its own output. The chart bounds that with
`credits.perRequestCap` (a per-request `local.tokens` ceiling). **This must be surfaced as a
product decision, not hidden:** "a trial may exceed its credit by up to one capped request."
If zero overshoot is required, that needs a gateway version with real pre-charge, or an
inline ext-proc that estimates and reserves before the call — out of scope for v1.4.1.

## The config this chart ships (`credits.enabled`)

An `AgentgatewayPolicy` on the `llm` HTTPRoute with:
- `rateLimit.global`: `domain`, `failureMode: FailClosed` (an unmeterable trial must not run
  for free), `backendRef` → the balance service, and a `Tokens` descriptor keyed by
  `entries[tenant] = apiKey.tenant` (the metadata the T1 client key carries).
- `rateLimit.local[tokens]`: the per-request overshoot cap.

That is the whole config surface. The balance lives in the service.

## The service contract (the remaining build)

A small stateless service implementing the **Envoy RateLimit gRPC v3** API
(`envoy.service.ratelimit.v3.RateLimitService/ShouldRateLimit`), backed by a durable
per-tenant counter (Redis or Postgres). Standard `envoyproxy/ratelimit` does **fixed-window**
counters and will NOT do a depleting balance — so this is a **custom RLS**, which is exactly
the "remote rate-limit service" the epic brief said to build. It must:

1. **Decrement, not window.** On each `ShouldRateLimit(domain, descriptor{tenant, hits_addend=cost})`,
   subtract `hits_addend` from that tenant's remaining balance. Return `OVER_LIMIT` once the
   balance is ≤ 0, `OK` otherwise. No time-based reset (a credit is bought, not refilled).
2. **Persist the balance** so a service restart or a second gateway replica sees the same
   number (the reason the balance cannot live in the per-proxy `local` limiter).
3. **Expose remaining credit** on a read endpoint (`GET /credit/{tenant}` → remaining,
   spent, cap) so the portal can show it. *Exhaustion must not present as a bug* (T2's
   explicit requirement): the UI reads this and shows "credit exhausted", and the gateway's
   `OVER_LIMIT` maps to a 429 with a body that says so.
4. **Be provisioned per trial** by T3: create the tenant's balance at signup, and
   **revoke/zero it first at teardown** (before deleting the child — a copied client key
   outlives the child, so the balance must refuse it immediately).

Align with **C5/S4**: the currency `costExpression` (per-model $/token) is the same rating
model as ClickHouse showback and budget alerts — compute it once, feed both. Do not build
metering three times.

## Exit test

With `llm.enabled` + `credits.enabled`, a per-tenant balance seeded to a small number, and a
valid T1 client key:

1. Calls succeed and the balance decrements by each call's token cost (observable on
   `GET /credit/{tenant}`).
2. Once the balance hits 0, the **next** call returns **429** (FailClosed on OVER_LIMIT),
   and the portal shows credit exhausted — not an opaque error.
3. Overshoot is ≤ `perRequestCap` tokens on the request that crossed zero.
4. Killing the balance service makes calls 429 (FailClosed), never run un-metered.

## Status

- **Config layer: done** (this chart, `credits.enabled`, rendered + schema-validated).
- **Balance service: not built** — needs the durable RLS above, and it cannot be
  live-validated until (a) a trial profile actually enables agents (today's slim profile
  runs none — see T1) and (b) the business credit size/currency are set. Both are inputs,
  not code. The config is ready to point at the service the moment it exists.
