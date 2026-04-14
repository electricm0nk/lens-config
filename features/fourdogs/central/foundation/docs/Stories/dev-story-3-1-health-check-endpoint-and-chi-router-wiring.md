# Story 3.1: Health Check Endpoint and Chi Router Wiring

Status: ready-for-dev

## Story

As a system operator,
I want a live health check endpoint and a fully wired chi router,
so that the API's routing, middleware chain, and response conventions are established before business endpoints are added.

## Acceptance Criteria

1. `cmd/api/main.go` constructs `pgxpool` from `DATABASE_URL`, reads `Config`, wires chi router, starts HTTP server
2. Chi router registers: `GET /v1/health`, `GET /auth/google`, `GET /auth/callback`
3. Auth middleware placeholder on `/v1/*` routes (returns HTTP 401 for all routes except `/v1/health` until Story 3.3)
4. `GET /v1/health` returns `HTTP 200 {"status": "ok"}` (no auth required)
5. All error responses use `{"error": "error_code", "message": "human string"}` format
6. `pgxpool` constructed exactly once at startup and injected into handlers — no per-request construction

## Tasks / Subtasks

- [ ] Task 0: Verify Story 1.5 complete — API is deployed to k3s with stub health handler

- [ ] Task 1: Write failing tests for health handler (AC: 4, 5) — RED phase
  - [ ] Create `internal/handler/health_test.go`:
    - Test: `GET /v1/health` → `HTTP 200`, body is `{"status":"ok"}`
    - Test: `GET /v1/health` (with Accept: application/json) → `Content-Type: application/json`
  - [ ] Create `internal/handler/testhelpers_test.go` with a `newTestRouter()` helper
  - [ ] Run `go test ./internal/handler/...` — RED

- [ ] Task 2: Implement chi router wiring in `cmd/api/main.go` (AC: 1, 2, 3, 6)
  - [ ] Add dependencies:
    ```
    go get github.com/go-chi/chi/v5
    go get github.com/jackc/pgx/v5
    ```
  - [ ] Update `cmd/api/main.go`:
    ```go
    func main() {
        cfg, err := config.Load()
        if err != nil {
            log.Fatalf("config error: %v", err)
        }

        ctx := context.Background()
        pool, err := pgxpool.New(ctx, cfg.DatabaseURL)
        if err != nil {
            log.Fatalf("failed to connect to database: %v", err)
        }
        defer pool.Close()

        r := chi.NewRouter()
        r.Use(middleware.Logger)
        r.Use(middleware.Recoverer)

        // Health (no auth)
        r.Get("/v1/health", handler.Health)

        // Auth routes (no auth middleware)
        r.Get("/auth/google", handler.AuthGoogleRedirect)
        r.Get("/auth/callback", handler.AuthCallback)

        // Protected routes (auth middleware — stub returns 401 until Story 3.3)
        r.Group(func(r chi.Router) {
            r.Use(handler.AuthMiddlewarePlaceholder)
            r.Get("/v1/items", handler.GetItems)
        })

        addr := ":8080"
        log.Printf("listening on %s", addr)
        if err := http.ListenAndServe(addr, r); err != nil {
            log.Fatalf("server error: %v", err)
        }
    }
    ```
  - [ ] Commit: `feat(api): wire chi router, pgxpool, and route registration`

- [ ] Task 3: Implement `internal/handler/health.go` (AC: 4, 5) — GREEN phase
  - [ ] Create `internal/handler/health.go`:
    ```go
    func Health(w http.ResponseWriter, r *http.Request) {
        w.Header().Set("Content-Type", "application/json")
        w.WriteHeader(http.StatusOK)
        json.NewEncoder(w).Encode(map[string]string{"status": "ok"})
    }
    ```
  - [ ] Create `internal/handler/errors.go` with helper:
    ```go
    func writeError(w http.ResponseWriter, status int, code, message string) {
        w.Header().Set("Content-Type", "application/json")
        w.WriteHeader(status)
        json.NewEncoder(w).Encode(map[string]string{
            "error":   code,
            "message": message,
        })
    }
    ```
  - [ ] Run `go test ./internal/handler/...` — GREEN

- [ ] Task 4: Implement stub handlers and placeholder auth middleware (AC: 2, 3)
  - [ ] Create `internal/handler/auth.go` with stub functions:
    - `AuthGoogleRedirect` — `writeError(w, 501, "not_implemented", "OAuth not yet implemented")`
    - `AuthCallback` — same stub
  - [ ] Create `internal/handler/middleware.go`:
    ```go
    // AuthMiddlewarePlaceholder returns 401 for all requests until Story 3.3 implements real auth
    func AuthMiddlewarePlaceholder(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            writeError(w, http.StatusUnauthorized, "unauthorized", "authentication required")
        })
    }
    ```
  - [ ] Create stub `internal/handler/items.go`:
    - `GetItems` — stub, never reached until auth middleware replaced in Story 3.3
  - [ ] Commit: `feat(api): add health handler, stub auth, placeholder middleware`

- [ ] Task 5: Integration test (AC: 1, 4, 5, 6)
  - [ ] Start API locally: `DATABASE_URL=... OAUTH_CLIENT_ID=... OAUTH_CLIENT_SECRET=... CORS_ALLOWED_ORIGIN=... go run ./cmd/api`
  - [ ] `curl -s localhost:8080/v1/health` → `{"status":"ok"}` with HTTP 200
  - [ ] `curl -s localhost:8080/v1/items` → HTTP 401 `{"error":"unauthorized",...}`
  - [ ] Verify: pgxpool constructed once (log statement in main)
  - [ ] Commit: `test(api): add health endpoint integration verification`

- [ ] Task 6: Update sprint status
  - [ ] Set `3-1-health-check-endpoint-and-chi-router-wiring: done`

## Dev Notes

- ARCH5: `pgxpool.New()` called exactly once in `main()` and injected into handlers via closure or dependency struct
- chi v5: `github.com/go-chi/chi/v5` — use `chi.NewRouter()` + `r.Group()` for route grouping
- `middleware.Logger` and `middleware.Recoverer` from chi's built-in middleware package
- Error response format must be consistent across ALL handlers (UX5): `{"error": "error_code", "message": "human string"}`
  - Use the `writeError` helper in every handler
- Health endpoint: not protected — the chi router registers it outside the auth group
- Port: `8080` hardcoded for now; could be env var but not required for MVP
- Pool injection pattern: pass pool to handler via constructor/closure; avoid global state
  - Example: `r.Get("/v1/items", handler.GetItemsHandler(pool, queries))`

### Project Structure Notes

- `internal/handler/` — all HTTP handlers and middleware
- `internal/handler/errors.go` — shared error writing helper
- `cmd/api/main.go` — router wiring and server startup only; no business logic

### References

- [Source: phases/techplan/architecture.md — API Server, chi Router]
- [Source: phases/devproposal/epics.md — Story 3.1 ACs, ARCH5, UX5, UX6, FR9]
- [Source: phases/businessplan/prd.md — FR9 (health check), FR6 (GET /v1/items)]

## Dev Agent Record

### Agent Model Used

GitHub Copilot / Claude Sonnet 4.6

### Debug Log References

### Completion Notes List

### File List
