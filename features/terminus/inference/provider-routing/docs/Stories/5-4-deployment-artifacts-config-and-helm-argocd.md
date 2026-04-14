# Story 5.4: Deployment artifacts — config and Helm/ArgoCD

Status: ready-for-dev

## Story

As an **operator**,
I want a sample `routing-config.yaml` committed alongside the deployment manifests and `ROUTING_CONFIG_PATH` added to the Helm chart,
So that deploying the routing-enabled gateway requires only setting an env var and providing the config file.

## Acceptance Criteria

1. **Given** the gateway Helm chart / ArgoCD application config  
   **When** deployment manifests are reviewed  
   **Then** `ROUTING_CONFIG_PATH` is declared as an environment variable in the chart (value configurable per environment)

2. **And** a `routing-config.yaml` example file is committed to the gateway deployment directory with both `interactive` and `batch` profiles, two providers (ollama + openai stubs), and valid `schema_version: 1`

3. **And** the example file contains no real API key values — only `api_key_env` references to env var names

## Tasks / Subtasks

- [ ] Add `ROUTING_CONFIG_PATH` env var to Helm chart values (AC: 1)
  - [ ] In `values.yaml`: `routingConfigPath: "/config/routing-config.yaml"` (or appropriate default)
  - [ ] In deployment template: `env: - name: ROUTING_CONFIG_PATH; value: {{ .Values.routingConfigPath }}`
- [ ] Create sample `routing-config.yaml` and commit to deployment directory (AC: 2, 3)
  - [ ] File path: `k8s/routing-config.yaml` or `helm/terminus-inference-gateway/config/routing-config.yaml` (check existing pattern)
  - [ ] Content: `schema_version: 1`, providers: `openai` + `ollama` with `api_key_env` references, profiles: `interactive` + `batch`
  - [ ] `openai.api_key_env: OPENAI_API_KEY` — no real value (AC: 3)
  - [ ] `ollama.api_key_env: ""` — local provider, no key needed (AC: 3)
- [ ] Ensure `ROUTING_CONFIG_PATH` env var is documented in the chart's `values.yaml` comments

## Dev Notes

- Check existing `terminus-inference-gateway` Helm chart structure in the deployment repo (`TargetProjects/terminus/inference` or equivalent) before creating new files — follow the existing patterns
- The sample routing-config.yaml is a deployment artifact, not a test fixture — it should be realistic but safe (no real keys)
- `ROUTING_CONFIG_PATH` value in the chart: the path should be inside a mounted volume or ConfigMap; the chart operator mounts the actual config file and sets this env var to point to it
- Example `routing-config.yaml`:
  ```yaml
  schema_version: 1
  providers:
    - name: openai
      api_key_env: OPENAI_API_KEY
      daily_budget_usd: 5.00
      price_per_1k_tokens: 0.002
    - name: ollama
      api_key_env: ""
      daily_budget_usd: 0
      price_per_1k_tokens: 0.0
  profiles:
    interactive:
      providers: [openai, ollama]
    batch:
      providers: [ollama, openai]
  ```
- ArgoCD application config: if using ArgoCD, ensure `ROUTING_CONFIG_PATH` appears in the `spec.template.spec.containers.env` section of the generated manifest

### Project Structure Notes

- Files: Helm chart `values.yaml`, deployment template, `routing-config.yaml` sample
- Check `TargetProjects/terminus/inference/` for existing Helm chart location
- The gateway's Helm chart for the `releaseorchestrator` initiative shows the existing pattern for ExternalSecret + env var injection — follow the same approach for `ROUTING_CONFIG_PATH`

### References

- [Source: docs/terminus/inference/provider-routing/prd.md#FR14, FR15, FR22, FR23] — YAML config; env var keys; config-only extensibility
- [Source: docs/terminus/inference/provider-routing/architecture.md#Additional Requirements] — `ROUTING_CONFIG_PATH`; sample routing-config.yaml
- [Source: docs/terminus/inference/provider-routing/epics.md#Story 5.4] — All AC scenarios

## Dev Agent Record

### Agent Model Used

Claude Sonnet 4.6

### Debug Log References

### Completion Notes List

### File List
