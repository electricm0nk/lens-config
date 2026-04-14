# Story 3.3: Auth Middleware and Session Persistence

Status: ready-for-dev

## Story

As a system operator,
I want every request to `/v1/*` (except `/v1/health`) to be gated by a session lookup,
so that Betsy's data stays protected and session state survives pod restarts.

## Acceptance Criteria

1. `internal/auth/middleware.go` validates session by looking up the session cookie in the `sessions` table via `sqlc GetSession` — no in-memory store (ARCH9)
2. Request without a session cookie or with an invalid/unknown session → HTTP 401 with `WWW-Authenticate: Bearer realm="fourdogs"` header
3. **Pod-restart smoke test (ARCH9):** authenticate → kill pod → restart → send request with same session cookie → receive HTTP 200, NOT 401
4. Expired session (where `expires_at < now()`) → HTTP 401; client must re-initiate OAuth
5. `/auth/google`, `/auth/callback`, and `/v1/health` are NOT protected by the middleware
6. All other `/v1/*` routes ARE protected by the middleware

## Tasks / Subtasks

- [ ] Task 0: Verify Story 3.2 complete — OAuth round-trip working end-to-end

- [ ] Task 1: Write failing middleware tests (AC: 1, 2, 4, 5, 6) — RED phase
  - [ ] Create `internal/auth/middleware_test.go`:
    - Test: request with valid session cookie → handler called (HTTP 200)
    - Test: request with no session cookie → HTTP 401 with `WWW-Authenticate` header
    - Test: request with unrecognized session cookie value → HTTP 401
    - Test: request with expired session (`expires_at` in past) → HTTP 401
    - Test: `/v1/health` request (no cookie) → handler is NOT blocked (HTTP 200)
    - Test: `/auth/google` request (no cookie) → handler is NOT blocked (HTTP 302)
  - [ ] Use `httptest.NewRecorder()` with a mock `db.Querier` interface
  - [ ] Run `go test ./internal/auth/...` — RED

- [ ] Task 2: Implement `internal/auth/middleware.go` (AC: 1, 2, 4, 5)
  - [ ] Middleware constructor:
    ```go
    func SessionMiddleware(queries db.Querier) func(http.Handler) http.Handler {
        return func(next http.Handler) http.Handler {
            return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
                sessionID := extractSessionCookie(r) // read "session_id" cookie
                if sessionID == "" {
                    unauthorised(w)
                    return
                }
                session, err := queries.GetSession(r.Context(), sessionID)
                if err != nil || session.ExpiresAt.Before(time.Now()) {
                    unauthorised(w)
                    return
                }
                // Optionally store session in context for downstream handlers
                ctx := context.WithValue(r.Context(), sessionContextKey, session)
                next.ServeHTTP(w, r.WithContext(ctx))
            })
        }
    }

    func unauthorised(w http.ResponseWriter) {
        w.Header().Set("WWW-Authenticate", `Bearer realm="fourdogs"`)
        w.WriteHeader(http.StatusUnauthorized)
    }
    ```
  - [ ] `extractSessionCookie`: read `Cookie: session_id=<value>` header
  - [ ] Run `go test ./internal/auth/...` — GREEN

- [ ] Task 3: Wire middleware into router with path exclusions (AC: 5, 6)
  - [ ] In `cmd/api/main.go`, update chi router to apply middleware selectively:
    ```go
    r.Group(func(r chi.Router) {
        r.Use(auth.SessionMiddleware(queries))
        r.Get("/v1/items", handler.GetItems(queries))
        // ... other protected /v1 routes
    })
    // Unprotected routes:
    r.Get("/v1/health", handler.HealthCheck())
    r.Get("/auth/google", handler.AuthGoogleRedirect(oauthCfg))
    r.Get("/auth/callback", handler.AuthCallback(oauthCfg, queries))
    ```
  - [ ] Commit: `feat(auth): implement session middleware with DB-backed validation`
  - [ ] Run all tests: `go test ./...` — GREEN

- [ ] Task 4: Execute pod-restart smoke test (AC: 3 — ARCH9)
  - [ ] Prerequisites: Story 3.1 deployed to k3s (Helm chart from 1.5)
  - [ ] Step 1: Authenticate via `/auth/google` on real k3s instance — verify session cookie set
  - [ ] Step 2: Confirm `/v1/items` or `/v1/health` returns 200 with session cookie
  - [ ] Step 3: Kill the pod — `kubectl delete pod -n fourdogs-central -l app=fourdogs-central-api`
  - [ ] Step 4: Wait for pod to restart (watch: `kubectl get pods -n fourdogs-central -w`)
  - [ ] Step 5: Send same request with original session cookie — EXPECT HTTP 200
  - [ ] Step 6: Record smoke test result in story completion notes
  - [ ] If AC3 FAILS: session table is not being used (in-memory leakage) — check DB connection and `GetSession` query

- [ ] Task 5: Update sprint status
  - [ ] Set `3-3-auth-middleware-and-session-persistence: done`

## Dev Notes

- ARCH9: No in-memory session cache — every request hits the DB (`GetSession` query). This is intentional and ensures session state survives pod restarts without shared memory or Redis dependency
- Session cookie name: `session_id` — set as HttpOnly + Secure + SameSite=Lax during `upsertSession` in Story 3.2
- `db.Querier` interface: sqlc generates this interface — use it for mock injection in tests
- `expires_at` check: do this in Go application layer as belt-and-suspenders (even though SQL query could filter it); session expiry should be validated server-side
- Session table `session_id`: generated as UUID on insert (uuidv4 via `github.com/google/uuid` or DB equivalent); this value is the cookie value
- Context propagation: storing session in context (`context.WithValue`) makes it accessible to downstream handlers without repeated DB queries
- Do NOT store session in Redis, memcached, or any external cache at this stage — ARCH9 explicitly requires DB persistence
- Pod restart verification: ARCH9 is a hard requirement — this smoke test blocks story completion

### Source Tree

```
internal/
  auth/
    middleware.go       ← this story
    middleware_test.go  ← this story
    oauth.go            ← story 3.2
  handler/
    auth.go             ← story 3.2
    health.go           ← story 3.1
    items.go            ← story 3.4
cmd/
  api/
    main.go             ← wire middleware into router
```

### References

- [Source: phases/techplan/architecture.md — Session Management, ARCH9]
- [Source: phases/devproposal/epics.md — Story 3.3 ACs, ARCH9]
- [Source: phases/businessplan/prd.md — FR13 (session management), FR14 (session expiry)]

## Dev Agent Record

### Agent Model Used

GitHub Copilot / Claude Sonnet 4.6

### Debug Log References

### Completion Notes List

### File List
