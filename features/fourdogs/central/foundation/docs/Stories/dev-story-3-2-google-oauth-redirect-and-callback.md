# Story 3.2: Google OAuth Redirect and Callback

Status: ready-for-dev

## Story

As Betsy,
I want to sign in with my Google account using a redirect flow,
so that I can access the catalog without managing a separate password.

## Acceptance Criteria

1. `GET /auth/google` redirects to Google OAuth authorization URL (redirect flow, no popup — NFR5)
2. `GET /auth/callback` receives auth code, exchanges for tokens, extracts `sub` claim from ID token
3. Session stored in `sessions` table via `sqlc UpsertSession` keyed on `google_sub` (ARCH4)
4. After successful auth, handler redirects to app root
5. Second login by same Google account → upserts existing session row, no duplicates
6. `google_sub` is session key — not email; email stored as informational only

## Tasks / Subtasks

- [ ] Task 0: Verify Story 3.1 complete — chi router with stub auth handlers running

- [ ] Task 1: Write failing tests for OAuth handlers (AC: 1, 2, 4, 5) — RED phase
  - [ ] Create `internal/handler/auth_test.go`:
    - Test: `GET /auth/google` → returns HTTP 302 with Location header containing `accounts.google.com`
    - Test: `GET /auth/callback?code=...&state=...` with mock token exchange → HTTP 302 to app root
    - Test: callback with duplicate `google_sub` → second call succeeds (upsert not insert-error)
    - Test: callback with invalid/missing code → HTTP 400 or redirect to error
  - [ ] Use `httptest.NewRecorder()` with a mock OAuth2 token source
  - [ ] Run `go test ./internal/handler/...` — RED

- [ ] Task 2: Add OAuth2 dependencies and configure provider (AC: 1, 2)
  - [ ] `go get golang.org/x/oauth2 golang.org/x/oauth2/google`
  - [ ] `go get github.com/coreos/go-oidc/v3/oidc`
  - [ ] Create `internal/auth/oauth.go`:
    ```go
    package auth

    import (
        "golang.org/x/oauth2"
        "golang.org/x/oauth2/google"
    )

    func NewOAuthConfig(clientID, clientSecret, redirectURL string) *oauth2.Config {
        return &oauth2.Config{
            ClientID:     clientID,
            ClientSecret: clientSecret,
            RedirectURL:  redirectURL,
            Scopes:       []string{"openid", "email", "profile"},
            Endpoint:     google.Endpoint,
        }
    }
    ```
  - [ ] Commit: `feat(auth): add Google OAuth2 config constructor`

- [ ] Task 3: Implement `GET /auth/google` redirect (AC: 1) — GREEN phase
  - [ ] Update `internal/handler/auth.go`:
    ```go
    func AuthGoogleRedirect(oauthCfg *oauth2.Config) http.HandlerFunc {
        return func(w http.ResponseWriter, r *http.Request) {
            state := generateState() // 16-byte random hex string
            // Store state in session/cookie for CSRF validation
            http.SetCookie(w, &http.Cookie{
                Name:     "oauth_state",
                Value:    state,
                HttpOnly: true,
                Secure:   true,
                SameSite: http.SameSiteLaxMode,
                MaxAge:   300,
            })
            url := oauthCfg.AuthCodeURL(state, oauth2.AccessTypeOffline)
            http.Redirect(w, r, url, http.StatusFound)
        }
    }
    ```
  - [ ] `generateState()`: `hex.EncodeToString(rand.Read(16))` using `crypto/rand`
  - [ ] Run `go test ./internal/handler/...` (auth/google tests) — GREEN

- [ ] Task 4: Implement `GET /auth/callback` (AC: 2, 3, 4, 5, 6) — GREEN phase
  - [ ] `AuthCallback(oauthCfg *oauth2.Config, queries *db.Queries) http.HandlerFunc`:
    ```go
    return func(w http.ResponseWriter, r *http.Request) {
        // 1. Validate state cookie matches query param (CSRF protection)
        // 2. Exchange code for tokens: oauthCfg.Exchange(ctx, r.URL.Query().Get("code"))
        // 3. Parse ID token: oidc provider or jwt decode for sub claim
        // 4. Extract google_sub from token claims
        // 5. UpsertSession(ctx, db.UpsertSessionParams{GoogleSub: googleSub, ...})
        // 6. http.Redirect to "/" (app root)
    }
    ```
  - [ ] For ID token parsing: use `go-oidc/v3` OIDC verifier:
    ```go
    provider, _ := oidc.NewProvider(ctx, "https://accounts.google.com")
    verifier := provider.Verifier(&oidc.Config{ClientID: clientID})
    idToken, _ := verifier.Verify(ctx, rawIDToken)
    var claims struct { Sub string `json:"sub"` }
    idToken.Claims(&claims)
    ```
  - [ ] Email capture: extract from claims and store in sessions table informational field if schema supports it
  - [ ] Run ALL `go test ./internal/handler/...` — GREEN
  - [ ] Commit: `feat(auth): implement Google OAuth redirect and callback handlers`

- [ ] Task 5: Wire real handlers into router (replacing stubs from Story 3.1) (AC: 1, 2)
  - [ ] Update `cmd/api/main.go` to pass `oauthCfg` and `queries` to auth handlers
  - [ ] Remove auth stub handlers
  - [ ] Test locally: `curl -v localhost:8080/auth/google` → 302 to Google OAuth URL

- [ ] Task 6: Update sprint status
  - [ ] Set `3-2-google-oauth-redirect-and-callback: done`

## Dev Notes

- NFR5: Redirect-only flow — `AuthCodeURL` redirect; NEVER use popup/implicit flow (iPad Safari incompatibility)
- ARCH4: `google_sub` is the session primary key — not email. Google `sub` is immutable; email can change
- CSRF protection: state parameter validation is required — generate random state → store in HttpOnly cookie → validate on callback
- `offline` access type: needed for `refresh_token` to be returned by Google
- Redirect URL: must be configured in Google Cloud Console exactly — `https://fourdogs.terminus.local/auth/callback`
- go-oidc v3 verifier: validates ID token signature and claims; do not skip this step (security requirement)
- `crypto/rand` for state generation — NOT `math/rand`
- After successful OAuth: redirect to `"/"` — the app root where `loader.html` is served (or the API root)
- Sessions table `email` field: if you want to store email informionally, add it to the sessions schema (migration) — but `google_sub` remains the PK

### Project Structure Notes

- `internal/auth/oauth.go` — OAuth config constructor
- `internal/handler/auth.go` — OAuth handlers (constructor injection pattern)
- `cmd/api/main.go` — constructs OAuth config and injects into handlers

### References

- [Source: phases/techplan/architecture.md — OAuth2 section, Session Management]
- [Source: phases/devproposal/epics.md — Story 3.2 ACs, ARCH4, NFR5, FR10, FR12]
- [Source: phases/businessplan/prd.md — FR10 (Google OAuth), FR11 (token refresh), FR12 (single user)]

## Dev Agent Record

### Agent Model Used

GitHub Copilot / Claude Sonnet 4.6

### Debug Log References

### Completion Notes List

### File List
