# T2 — per-tenant token credits

**Goal (epic T2):** a tenant gets a finite LLM credit. When it is spent, further model calls
are refused with a 429. Calls never run unmetered.

## Model: a per-tenant token bucket, policy only

There is no balance service. `credits.enabled` renders one `AgentgatewayPolicy`
(`llm-credits`) on the `llm` HTTPRoute:

- `traffic.rateLimit.conditional`: one branch per plan, matched on `credits.planExpression`
  (default `apiKey.plan`, the `plan` in the client key's metadata). It ends with the
  `credits.defaultPlan` branch with no condition, so a key without a plan, or with an
  unknown one, is still metered.
- Each branch is a `local` token limit with `key: credits.tenantExpression` (default
  `apiKey.tenant`), so **every tenant gets its own bucket**.

Behaviour, verified live on agentgateway **v1.6.0** (GKE demo):

| Property | Observed |
|---|---|
| Starting balance | the bucket starts full at `burst` |
| Per-tenant isolation | an exhausted tenant gets 429 while another tenant gets 200 |
| Plan routing | a `plan: standard` key used the standard bucket; a key without a plan used the default plan's bucket |
| Exhaustion | `429 rate limit exceeded` |
| Overshoot | tokens are counted after the request completes, so the call that crosses zero is allowed and the next call is refused. Overshoot is at most one request. |

## Sizing a plan

- **One-off credit** (e.g. a trial): `burst` = the credit in tokens, `tokensPerHour: 1`.
  This holds for any trial length, because the trial's client key is revoked at teardown.
- **Recurring credit** (e.g. per month): `tokensPerHour` = credit tokens ÷ hours in the
  period, plus a bounded `burst`, so that spend in a period ≤ credit + burst.

**Converting money to tokens.** A local limit counts raw tokens (input + output) and takes
no cost expression, so a money amount is converted at a measured **effective $/token**:
total `agw.ai.usage.cost.total` ÷ total tokens over representative traffic, read from the
gateway's access logs (agentgateway ≥1.6 prices each request from its built-in catalog).
Input tokens dominate agent traffic, and prompt caching lowers the effective rate well
below the list input price, so measure the rate rather than assuming it. Re-measure when
the model, the price or the agents change.

The amounts are business configuration. Set `credits.plans` at deploy time; the chart
ships `plans: {}` and refuses to render credits without them.

## Known limits

- **Tokens, not currency.** The credit is exact in tokens and approximate in money, as
  accurate as the effective rate.
- **In memory, per proxy.** Buckets live in the gateway pod: a restart refills them, and
  with several replicas each holds its own bucket. Run one gateway replica, or divide by
  the replica count.
- **No balance readout.** Remaining credit is not exposed. Usage per tenant is on the
  `tenant` label of `agentgateway_gen_ai_client_token_usage_*` (`llm.metering.tenantLabel`).

## Exit test

With `llm.enabled`, `credits.enabled`, a plan with a small `burst` and a valid T1 client
key: calls succeed until the tenant has used `burst` tokens; the next call returns 429;
another tenant's calls are unaffected.
