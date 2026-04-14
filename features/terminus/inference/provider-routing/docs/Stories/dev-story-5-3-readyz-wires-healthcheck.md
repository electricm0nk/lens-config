# Story 5.3: /readyz wires HealthCheck

**Initiative:** terminus-inference-provider-routing
**Epic:** 5 — Gateway can deploy with routing enabled
**Status:** review

## Story

As an **operator**,
I want the gateway's `/readyz` endpoint to include routing layer health in its response,
So that orchestration infrastructure (k3s, ArgoCD) can detect when all providers are exhausted.

## Acceptance Criteria

**Given** the `/readyz` handler exists in the gateway
**When** `router.HealthCheck(ctx)` returns `HealthStatus.AllExhausted: false`
**Then** `/readyz` returns HTTP 200 and the routing section of the response shows healthy

**Given** `router.HealthCheck(ctx)` returns `HealthStatus.AllExhausted: true`
**When** `/readyz` is called
**Then** it returns HTTP 503 — the gateway reports itself unhealthy
**And** the response body includes per-provider budget state from `HealthStatus.Providers`

## Technical Notes

- Target repo: `terminus-inference-gateway`
- Modify existing `/readyz` handler to call `router.HealthCheck(ctx)` and merge result into readiness response
- Response structure addition:
  ```json
  {
    "status": "unhealthy",
    "routing": {
      "all_exhausted": true,
      "degraded": true,
      "providers": [
        {"name": "openai", "budget_exhausted": true, "budget_used_usd": 5.00, "budget_limit_usd": 5.00}
      ]
    }
  }
  ```
- `AllExhausted: true` → HTTP 503; otherwise existing readyz logic determines status code
- No credentials in the response — `ProviderStatus` never contains key values (enforced by story 4.2)

## Dependencies

- Story 4.2 (HealthCheck implemented)
- Story 5.1 (Router injected into server — readyz handler has access to router)
