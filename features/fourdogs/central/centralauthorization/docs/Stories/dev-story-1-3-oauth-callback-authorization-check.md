# Story 1.3: Enforce Authorization Check in OAuth Callback

Status: ready-for-dev

## Story

As a system operator,
I want the OAuth callback to reject any Google account not in the `authorized_users` table,
so that only explicitly authorized users can obtain a session, regardless of their Google account status.

## Acceptance Criteria

1. In `internal/auth/oauth.go`, after extracting `sub` from the verified ID token, call `queries.IsAuthorized(ctx, sub)`
2. If `IsAuthorized` returns `false`:
   - Log: `slog.Warn("unauthorized login attempt", "sub", sub)`
   - Return HTTP 403 with `Content-Type: application/json` and body: `{"error":"forbidden","message":"not authorized"}`
   - Do not proceed to session upsert — function returns immediately after 403 response
3. If `IsAuthorized` returns `true`: continue to session upsert and cookie set as before
4. Unit test: a `sub` not in `authorized_users` → 403 response, no session row written
5. Unit test: a `sub` present in `authorized_users` → callback proceeds, session upsert called
6. All existing OAuth callback tests pass without modification

**Bootstrap procedure (acceptance criteria):**
On first deployment with an empty `authorized_users` table, ALL login attempts return 403 — including Betsy's. This is expected and correct. The operator bootstrap procedure is:
1. Deploy with empty table
2. Betsy attempts login (gets 403 in browser — expected)
3. Operator reads `sub` from pod logs: `kubectl logs <pod> | grep "unauthorized login attempt"`
4. Operator runs: `./fourdogs-central-users add <sub> <email>`
5. Betsy logs in successfully

This is a one-time operator task before handing the URL to Betsy. Document this procedure in a `README` or operator runbook comment in the code.

## Tasks / Subtasks

- [ ] Task 0: Verify Story 1.2 complete — `queries.IsAuthorized` is available (AC: all)
  - [ ] Confirm `internal/db/authorized_users.sql.go` exists and `go build ./...` passes

- [ ] Task 1: Write failing unit tests (AC: 4, 5, 6) — RED phase
  - [ ] In `internal/auth/oauth_test.go` (or alongside the callback test):
    - Test: mock `IsAuthorized` returning `false` → assert response status 403 and JSON body `{"error":"forbidden","message":"not authorized"}` → assert no UpsertSession call
    - Test: mock `IsAuthorized` returning `true` → assert callback proceeds → assert UpsertSession call made
  - [ ] Run tests — confirm compilation fails or tests fail (RED — implementation not yet added)

- [ ] Task 2: Modify `internal/auth/oauth.go` (AC: 1, 2, 3)
  - [ ] After sub extraction and before session upsert, add:
    ```go
    authorized, err := queries.IsAuthorized(ctx, sub)
    if err != nil {
        http.Error(w, `{"error":"internal","message":"auth check failed"}`, http.StatusInternalServerError)
        return
    }
    if !authorized {
        slog.Warn("unauthorized login attempt", "sub", sub)
        w.Header().Set("Content-Type", "application/json")
        w.WriteHeader(http.StatusForbidden)
        w.Write([]byte(`{"error":"forbidden","message":"not authorized"}`))
        return
    }
    ```
  - [ ] Confirm `queries` is wired into the OAuth handler (it should be via the existing `db.Queries` struct — check how `UpsertSession` is currently called to confirm the injection pattern)
  - [ ] Commit: `feat(auth): enforce authorized_users check in OAuth callback`

- [ ] Task 3: Run tests (AC: 4, 5, 6) — GREEN phase
  - [ ] `go test ./internal/auth/...` — all tests pass
  - [ ] Confirm no existing tests were broken

- [ ] Task 4: Verify bootstrap procedure is documented (AC: bootstrap)
  - [ ] Add a code comment above the `IsAuthorized` check block referencing the bootstrap procedure, or confirm a README operator runbook section exists
  - [ ] Commit any documentation additions: `docs(auth): add bootstrap note for empty authorized_users table`

## Dev Notes

- `queries` injection pattern: check the existing OAuth callback handler signature — `UpsertSession` should already be called via a `*db.Queries` receiver. `IsAuthorized` is on the same `*db.Queries` struct, so no new wiring is required.
- Log using `slog.Warn` (not `log.Printf`) — the foundation uses `slog` for structured logging throughout.
- The 403 body format `{"error":"...","message":"..."}` is consistent with the foundation error response contract.
- The authorization check must occur **after** ID token verification and sub extraction, but **before** any call to `UpsertSession`. If there are intermediate steps (email extraction, etc.), keep the authorization gate as early as possible after sub is available.
- Error path for `IsAuthorized` DB failure: return 500, do not proceed. This is a fail-closed design — a DB error is safer than allowing login.

### Project Structure Notes

- `internal/auth/oauth.go` is the correct file — the OAuth callback handler is in the `auth` package per foundation story 3.2
- No new packages needed — `db.Queries` already has `IsAuthorized` from Story 1.2
- The `queries` variable in the callback scope should match the existing pattern for how `UpsertSession` is invoked

### References

- OAuth callback implementation: [_bmad-output/implementation-artifacts/fourdogs-central/dev-story-3-2-google-oauth-redirect-and-callback.md](_bmad-output/implementation-artifacts/fourdogs-central/dev-story-3-2-google-oauth-redirect-and-callback.md)
- Session persistence: [_bmad-output/implementation-artifacts/fourdogs-central/dev-story-3-3-auth-middleware-and-session-persistence.md](_bmad-output/implementation-artifacts/fourdogs-central/dev-story-3-3-auth-middleware-and-session-persistence.md)
- Architecture: [_bmad-output/lens-work/initiatives/fourdogs/central/centralauthorization/architecture.md](_bmad-output/lens-work/initiatives/fourdogs/central/centralauthorization/architecture.md)
- Stories source: [_bmad-output/lens-work/initiatives/fourdogs/central/centralauthorization/stories.md](_bmad-output/lens-work/initiatives/fourdogs/central/centralauthorization/stories.md)

## Dev Agent Record

### Agent Model Used

claude-sonnet-4-5

### Debug Log References

### Completion Notes List

### File List
