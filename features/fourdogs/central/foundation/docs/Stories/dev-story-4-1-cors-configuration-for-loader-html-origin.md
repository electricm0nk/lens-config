# Story 4.1: CORS Configuration for loader.html Origin

Status: ready-for-dev

## Story

As a system operator,
I want the API to send correct CORS headers for the loader.html origin,
so that Betsy's browser can call the API without cross-origin errors.

## Acceptance Criteria

1. `go-chi/cors` middleware is registered and applied to all API routes
2. `CORS_ALLOWED_ORIGIN` environment variable drives allowed origins — NOT hardcoded (ARCH7)
3. Preflight `OPTIONS` request to any `/v1/*` route returns HTTP 200 with correct `Access-Control-Allow-Origin`, `Access-Control-Allow-Methods`, and `Access-Control-Allow-Headers` headers
4. Request from a disallowed origin → response does NOT include `Access-Control-Allow-Origin` header
5. Missing or empty `CORS_ALLOWED_ORIGIN` at startup → startup error (fail-fast; do not run without CORS config)

## Tasks / Subtasks

- [ ] Task 0: Verify Epic 3 complete — OAuth, middleware, and catalog API in place

- [ ] Task 1: Write failing CORS tests (AC: 1, 3, 4) — RED phase
  - [ ] Create `internal/handler/cors_test.go` or update integration test:
    - Test: `OPTIONS /v1/items` with `Origin: https://fourdogs.terminus.local` → HTTP 200 + `Access-Control-Allow-Origin: https://fourdogs.terminus.local`
    - Test: `OPTIONS /v1/items` with `Origin: https://evil.example.com` → NO `Access-Control-Allow-Origin` header
    - Test: `GET /v1/health` with allowed origin → response has CORS header
    - Test: `GET /v1/health` with disallowed origin → no CORS header
  - [ ] Run `go test ./internal/handler/...` — RED

- [ ] Task 2: Add go-chi/cors dependency (AC: 1)
  - [ ] `go get github.com/go-chi/cors`
  - [ ] Run `go mod tidy`
  - [ ] Commit: `chore: add go-chi/cors dependency`

- [ ] Task 3: Implement CORS middleware wiring in `cmd/api/main.go` (AC: 1, 2, 3, 4)
  - [ ] Read `CORS_ALLOWED_ORIGIN` from environment in server startup:
    ```go
    corsOrigin := os.Getenv("CORS_ALLOWED_ORIGIN")
    if corsOrigin == "" {
        log.Fatal("CORS_ALLOWED_ORIGIN is required")
    }
    ```
  - [ ] Register CORS middleware on the root router BEFORE route definitions:
    ```go
    r.Use(cors.Handler(cors.Options{
        AllowedOrigins:   []string{corsOrigin},
        AllowedMethods:   []string{"GET", "OPTIONS"},
        AllowedHeaders:   []string{"Accept", "Authorization", "Content-Type", "Cookie"},
        AllowCredentials: true,
        MaxAge:           300,
    }))
    ```
  - [ ] `AllowCredentials: true` — required for browser to send session cookie cross-origin
  - [ ] Run `go test ./...` — GREEN

- [ ] Task 4: Add `CORS_ALLOWED_ORIGIN` to Helm values and ESO secret config (AC: 2, 5)
  - [ ] Add to `charts/fourdogs-central-api/values.yaml`:
    ```yaml
    env:
      corsAllowedOrigin: ""  # set per environment
    ```
  - [ ] Add to `charts/fourdogs-central-api/templates/deployment.yaml` env block:
    ```yaml
    - name: CORS_ALLOWED_ORIGIN
      value: {{ .Values.env.corsAllowedOrigin | quote }}
    ```
  - [ ] Add to `charts/fourdogs-central-api/values.terminus-local.yaml` (or environment overlay):
    ```yaml
    env:
      corsAllowedOrigin: "https://fourdogs.terminus.local"
    ```
  - [ ] Commit: `feat(cors): configure CORS_ALLOWED_ORIGIN via Helm values`

- [ ] Task 5: Validate startup error on missing CORS config (AC: 5)
  - [ ] Test: `CORS_ALLOWED_ORIGIN="" go run ./cmd/api/` → immediate `log.Fatal` before HTTP server starts
  - [ ] Confirm behavior in test: override `os.Getenv` via env injection or refactor to accept config struct

- [ ] Task 6: Update sprint status
  - [ ] Set `4-1-cors-configuration-for-loader-html-origin: done`

## Dev Notes

- ARCH7: All origins/secrets via env vars — never hardcode `fourdogs.terminus.local` or any domain in Go source
- `AllowCredentials: true`: MANDATORY — session cookie must be sent cross-origin; without this, browser silently omits `Cookie` header
- When `AllowCredentials: true`, `AllowedOrigins` CANNOT be `["*"]` — must be explicit origin list (CORS spec requirement)
- `MaxAge: 300`: 5-minute preflight cache — reduces OPTIONS round-trips
- Allowed methods: `GET` and `OPTIONS` are sufficient for this API (read-only catalog, auth via redirect not XHR)
- `Accept, Authorization, Content-Type, Cookie`: standard headers; add `X-Forwarded-For` if Traefik injects it
- cors middleware applies to ALL routes including `/auth/*` and `/v1/health` — this is correct and harmless
- Startup fail-fast (AC5): do not use a default fallback origin — missing config means misconfigured deployment, not a recoverable state

### Source Tree

```
cmd/
  api/
    main.go             ← register cors middleware, read CORS_ALLOWED_ORIGIN
charts/
  fourdogs-central-api/
    values.yaml         ← corsAllowedOrigin: ""
    values.terminus-local.yaml  ← corsAllowedOrigin: "https://fourdogs.terminus.local"
    templates/
      deployment.yaml   ← CORS_ALLOWED_ORIGIN env var
```

### References

- [Source: phases/devproposal/epics.md — Story 4.1 ACs, ARCH7]
- [Source: phases/techplan/architecture.md — CORS configuration, environment variable policy]
- [Source: go-chi/cors: https://github.com/go-chi/cors]

## Dev Agent Record

### Agent Model Used

GitHub Copilot / Claude Sonnet 4.6

### Debug Log References

### Completion Notes List

### File List
