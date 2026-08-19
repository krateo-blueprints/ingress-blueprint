# T1 — brokering model traffic through the gateway

**Goal (epic T1):** no model credential ever lives inside a trial child. A stranger holds
cluster-admin in their own vcluster, so anything placed there is theirs to copy and keep
after expiry. The gateway holds the Vertex credential on the parent; children speak the
OpenAI wire format to `https://<llm.hostname>/v1` with a per-client API key.

## Two halves

**Parent (this chart, `llm.enabled`)** — an `AgentgatewayBackend` (Vertex provider), a
backend-auth `AgentgatewayPolicy` (`gcp.secretRef` → operator-placed SA JSON), a client-key
`AgentgatewayPolicy` (`apiKeyAuthentication.secretSelector`, fail-closed), an `HTTPRoute`,
and a dedicated `https-llm` Gateway listener with its own certificate. Neither Secret is
templated — both are placed by hand.

**Child (blueprint config, NOT this chart)** — point the child's kagent `ModelConfig` at
`provider: OpenAI` + `baseUrl: https://<llm.hostname>/v1`, and **pin out** kagent's native
`GeminiVertexAI` / `AnthropicVertexAI` providers, which would reach Vertex directly and
bypass the gateway. Defence in depth: pin the config *and* withhold the credential.

## Current state, stated honestly

The shipped self-service trial profile (`krateo-selfservice-blueprint` values) runs with
`coreAgents:false`, `specialistAgents:false`, `vertexAI.enabled:false` — **a trial child
today runs no agents**, so there is currently nothing in the child to bypass the gateway.
The child-side pin-out therefore becomes load-bearing only for the product direction where
a trial *showcases* the agent fleet. Tracked as a child-blueprint follow-up; the parent
broker here is the prerequisite either way and is complete.

## Exit test (epic T1)

1. From inside a child, a **direct** Vertex call fails — no credential exists there.
2. A call to the gateway with a valid client key **succeeds** and returns a completion.
3. A call with no/invalid key is **401** (fail-closed) — verifiable with `llm.enabled` and
   zero key Secrets provisioned.

## Hook for T2 (credits)

Each client key is a labelled Secret whose entry carries `{"keyHash":"…","metadata":{…}}`.
`metadata` becomes the CEL descriptor for `traffic.rateLimit` (`RateLimitType::Tokens`,
pre-charge + amend, hard stop) — one key per trial, revoked first at teardown (T3). This
chart deliberately stops at the auth boundary; the metering policy is T2.
