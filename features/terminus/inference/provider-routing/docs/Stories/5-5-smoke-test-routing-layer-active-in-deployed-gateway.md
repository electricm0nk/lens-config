# Story 5.5: Smoke test — routing layer active in deployed gateway

Status: ready-for-dev

## Story

As an **operator**,
I want a smoke test confirming the deployed gateway routes inference requests through the routing layer,
So that a broken deployment is detected before it reaches production.

## Acceptance Criteria

1. **Given** the gateway is deployed with a valid `routing-config.yaml` and `ROUTING_CONFIG_PATH` set  
   **When** a test inference request is sent with `Profile: "interactive"`  
   **Then** the response includes a selected provider name and HTTP 200  
   **And** `/readyz` returns 200 with routing health populated

2. **Given** a test inference request is sent with `Profile: "unknown"`  
   **Then** the response is HTTP 400

3. **Given** no provider is reachable (simulated by setting budgets to near-zero in the test config and exhausting them)  
   **When** a test inference request is sent  
   **Then** the response is HTTP 503 and `/readyz` reports unhealthy

## Tasks / Subtasks

- [ ] Write smoke test script or test file (AC: 1, 2, 3)
  - [ ] AC 1: POST `Profile: "interactive"` → expect 200 + `provider` field in response
  - [ ] AC 1: GET `/readyz` → expect 200 + routing health JSON
  - [ ] AC 2: POST `Profile: "unknown"` → expect 400
  - [ ] AC 3: Deplete all providers via exhaustion requests, then POST → expect 503 and `/readyz` shows unhealthy
- [ ] Place smoke test in appropriate location (e.g. `tests/smoke/` or `e2e/` in the target inference repo)
- [ ] Ensure the smoke test instructions note the required env vars (`ROUTING_CONFIG_PATH`, `OPENAI_API_KEY`) and the required config file

## Dev Notes

- The smoke test is an **integration/smoke test**, not a unit test — it runs against a deployed (or locally started) gateway instance
- AC 3 (budget exhaustion) can be simulated using a `routing-config.yaml` with `daily_budget_usd: 0` or an extremely small value, which causes `BudgetTracker.IsExhausted()` to return true immediately
- The test should be idempotent; if the gateway doesn't have a running inference backend (ollama, openai), the routing layer itself should still respond correctly for known error cases
- Prefer using `curl` or a simple Go `testing` integration test with `httptest.NewServer` or a real HTTP call (if a test instance is available)
- `/readyz` response shape (from Story 4.2):
  ```json
  {
    "status": "ok|degraded",
    "providers": {
      "openai": { "exhausted": false, "budget_remaining_tokens": 5000 },
      "ollama": { "exhausted": false, "budget_remaining_tokens": -1 }
    }
  }
  ```
- The smoke test should be runnable independently from the unit test suite so it can be used in CD pipelines as a post-deploy gate

### Project Structure Notes

- Files: `tests/smoke/routing_smoke_test.go` or `scripts/smoke-test.sh` in the inference gateway repo
- If using Go integration tests, tag with `//go:build smoke` so they are excluded from regular CI unit runs
- Dependencies: running gateway, valid `ROUTING_CONFIG_PATH` pointing to a test config

### References

- [Source: docs/terminus/inference/provider-routing/prd.md#NFR3] — reliably detects degraded state
- [Source: docs/terminus/inference/provider-routing/architecture.md#Observability] — /readyz shape and health reporting
- [Source: docs/terminus/inference/provider-routing/epics.md#Story 5.5] — All AC scenarios
- [Source: Stories/4-2-healthcheck-returns-per-provider-budget-state.md] — /readyz implementation
- [Source: Stories/5-2-http-handler-calls-resolve-and-recordusage.md] — HTTP handler routing logic

## Dev Agent Record

### Agent Model Used

Claude Sonnet 4.6

### Debug Log References

### Completion Notes List

### File List
