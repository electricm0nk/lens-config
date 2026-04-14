# Story 5.4: Deployment artifacts — config and Helm/ArgoCD

**Initiative:** terminus-inference-provider-routing
**Epic:** 5 — Gateway can deploy with routing enabled
**Status:** done

## Story

As an **operator**,
I want a sample `routing-config.yaml` committed alongside the deployment manifests and `ROUTING_CONFIG_PATH` added to the Helm chart,
So that deploying the routing-enabled gateway requires only setting an env var and providing the config file.

## Acceptance Criteria

**Given** the gateway Helm chart / ArgoCD application config
**When** deployment manifests are reviewed
**Then** `ROUTING_CONFIG_PATH` is declared as an environment variable in the chart (value configurable per environment)
**And** a `routing-config.yaml` example file is committed to the gateway deployment directory with both `interactive` and `batch` profiles, two providers (ollama + openai stubs), and valid `schema_version: 1`
**And** the example file contains no real API key values — only `api_key_env` references to env var names

## Technical Notes

- Target repos: `terminus-inference-gateway` (sample config) + `terminus.infra` (Helm/ArgoCD values)
- Sample `routing-config.yaml` location: `deploy/config/routing-config.yaml` (or equivalent)
- Helm chart addition: add `ROUTING_CONFIG_PATH` to `env:` in `values.yaml` with a placeholder value
- Example config content:
  ```yaml
  schema_version: 1
  providers:
    - name: ollama
      base_url: http://ollama.inference.svc.cluster.local:11434
      api_key_env: ""
      daily_budget_usd: 0
      price_per_1k_tokens: 0
    - name: openai
      base_url: https://api.openai.com/v1
      api_key_env: OPENAI_API_KEY
      daily_budget_usd: 5.00
      price_per_1k_tokens: 0.002
  profiles:
    - name: interactive
      provider_chain: [openai, ollama]
    - name: batch
      provider_chain: [ollama, openai]
  ```
- The `OPENAI_API_KEY` env var value is seeded into the pod via ESO/Vault (not set directly in Helm values)
- Commit message should note this is a stub/example — production values via ArgoCD + ESO

## Dependencies

- Epic 1-4 complete (routing package fully implemented)
