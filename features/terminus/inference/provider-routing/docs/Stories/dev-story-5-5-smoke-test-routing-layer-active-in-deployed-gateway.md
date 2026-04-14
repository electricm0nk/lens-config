# Story 5.5: Smoke test — routing layer active in deployed gateway

**Initiative:** terminus-inference-provider-routing
**Epic:** 5 — Gateway can deploy with routing enabled
**Status:** backlog

## Story

As an **operator**,
I want a smoke test confirming the deployed gateway routes inference requests through the routing layer,
So that a broken deployment is detected before it reaches production.

## Acceptance Criteria

**Given** the gateway is deployed with a valid `routing-config.yaml` and `ROUTING_CONFIG_PATH` set
**When** a test inference request is sent with `Profile: "interactive"`
**Then** the response includes a selected provider name and HTTP 200
**And** `/readyz` returns 200 with routing health populated

**Given** a test inference request is sent with `Profile: "unknown"`
**When** the handler processes it
**Then** the response is HTTP 400

**Given** no provider is reachable (simulated by setting budgets to near-zero in the test config and exhausting them)
**When** a test inference request is sent
**Then** the response is HTTP 503 and `/readyz` reports unhealthy

## Tasks

- [ ] Deploy gateway with routing config (`ROUTING_CONFIG_PATH` set, both providers configured)
- [ ] `GET /readyz` → HTTP 200, `all_exhausted: false`
- [ ] `POST /v1/chat/completions` with `X-Workload-Class: interactive` → HTTP 200 or 401 (auth) — verify response shape includes provider routing info
- [ ] `POST /v1/chat/completions` with `X-Workload-Class: unknown` → HTTP 400
- [ ] Exhaust provider budget via RecordUsage test endpoint or config override → `GET /readyz` → HTTP 503
- [ ] Record results in story acceptance table

## Acceptance Table

| Check | Expected | Result | Notes |
|---|---|---|---|
| `/readyz` pre-test | HTTP 200, `all_exhausted: false` | — | |
| Interactive routing | HTTP 200/401, provider in response | — | |
| Unknown workload class | HTTP 400 | — | |
| All providers exhausted | HTTP 503, `/readyz` unhealthy | — | |

## Dependencies

- Stories 5.1–5.4 complete (full deployment with routing)
