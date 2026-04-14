# Story 4.2: Rewire loader.html to Authenticated API

Status: ready-for-dev

## Story

As Betsy,
I want loader.html to fetch the catalog from the live API instead of a static Drive file,
so that the data I see is always current without manual updates.

## Acceptance Criteria

1. `loader.html` fetches the catalog from `GET /v1/items` using session cookie authentication — Google Drive JSON URL is removed (NFR6)
2. First-time or expired-session user is redirected to `/auth/google` (not shown an error state) when API returns HTTP 401
3. Google Drive JSON file is decommissioned at deploy time — not accessible or referenced
4. JSON parsing code in `loader.html` is UNCHANGED — same field names and array format (FR7, NFR1)
5. localStorage session state is preserved across catalog loads (FR23)
6. Compile Order button generates CSV client-side — no network request (FR24)
7. loader.html is served by the API server (or CDN/Helm ingress) — not loaded directly from the filesystem

## Tasks / Subtasks

- [ ] Task 0: Verify Epic 3 complete — `/v1/items` endpoint working with CORS enabled (Story 4.1 done)

- [ ] Task 1: Write failing integration tests (AC: 1, 2, 4) — RED phase
  - [ ] Cypress or Playwright test (or bash integration test):
    - Test: authenticated session → `GET /v1/items` returns items → page renders items
    - Test: no session cookie → API returns 401 → page redirects to `/auth/google`
    - Test: items rendered in UI match expected field names (spot-check `i`, `n`, `b`)
  - [ ] If browser automation is out of scope: manual checklist test (document in AC review)
  - [ ] Run tests — RED

- [ ] Task 2: Locate and modify `loader.html` (AC: 1, 2, 4, 5, 6)
  - [ ] Find current Drive JSON fetch in `loader.html`:
    ```javascript
    // OLD (remove this):
    fetch("https://docs.google.com/spreadsheets/d/.../gviz/tq?tqx=out:json&...")
      .then(response => response.json())
      .then(data => { ... })
    ```
  - [ ] Replace with API fetch using credentials:
    ```javascript
    // NEW:
    fetch('/v1/items', { credentials: 'include' })
      .then(response => {
        if (response.status === 401) {
          window.location.href = '/auth/google';
          return null;
        }
        return response.json();
      })
      .then(items => {
        if (!items) return; // redirecting
        // REUSE existing parse/render logic unchanged
        // items is already a plain array of objects with same field names
        renderItems(items);
      })
      .catch(err => console.error('Catalog load failed:', err));
    ```
  - [ ] `credentials: 'include'` — required for session cookie to be sent cross-origin
  - [ ] RETAIN all existing parsing/rendering code — only replace the fetch source and add 401 redirect
  - [ ] RETAIN localStorage read/write for session state (FR23)
  - [ ] RETAIN client-side "Compile Order" CSV generation — no changes (FR24)
  - [ ] Commit: `feat(ui): rewire loader.html to fetch from /v1/items`

- [ ] Task 3: Serve `loader.html` from the API server (AC: 7)
  - [ ] Add static file serving in `cmd/api/main.go`:
    ```go
    r.Get("/", http.FileServer(http.FS(staticFiles)).ServeHTTP)
    // or embed:
    //go:embed static/loader.html
    var staticFiles embed.FS
    r.Handle("/*", http.FileServer(http.FS(staticFiles)))
    ```
  - [ ] Move `loader.html` to `static/` directory under the Go module root
  - [ ] OR: serve via Helm chart Nginx sidecar — document which approach is used
  - [ ] Verify: `GET /` returns `loader.html` content

- [ ] Task 4: Decommission Drive JSON file (AC: 3)
  - [ ] Remove Drive sharing URL from `loader.html` (done in Task 2)
  - [ ] Update deployment runbook: revoke sharing permissions on Google Drive file at deploy time
  - [ ] Add comment in `loader.html`: `<!-- Drive JSON decommissioned: <date>; catalog served from /v1/items -->`
  - [ ] Commit: `chore(ui): decommission Drive JSON URL; add decommission comment`

- [ ] Task 5: Run GREEN tests and manual verification (AC: 1, 2, 4, 5, 6)
  - [ ] Automated: `go test ./...` passes
  - [ ] Manual: open `loader.html` in browser → items load from API
  - [ ] Manual: clear session cookie → refresh → redirected to Google OAuth
  - [ ] Manual: "Compile Order" button → CSV downloads, no network request observed in DevTools
  - [ ] Manual: localStorage values preserved after catalog reload

- [ ] Task 6: Update sprint status
  - [ ] Set `4-2-rewire-loader-html-to-authenticated-api: done`

## Dev Notes

- NFR6: Drive JSON URL decommissioned = story complete. The URL must be absent from all code; the Drive file's sharing permission should be revoked at deployment
- FR7/NFR1: JSON parsing code MUST NOT be changed. The field names `i`, `u`, `n`, `b`, etc. come from the API response (which mirrors the DB column names) — same as what prototype got from Drive JSON
- `fetch('/v1/items', { credentials: 'include' })`: relative URL preferred — works regardless of domain; `credentials: 'include'` is required to send the `session_id` cookie cross-origin
- 401 redirect: `window.location.href = '/auth/google'` — NOT `window.location.replace` unless you want to prevent back-navigation; `href` is correct for OAuth flows
- FR23 (localStorage): do not modify any `localStorage.setItem/getItem` calls — these persist order state and are unrelated to auth
- FR24 (client-side CSV): do not modify Compile Order logic — verify it has no fetch calls
- `go:embed`: If serving `loader.html` via embed, ensure the file path in the Go source is relative to the package directory containing the `//go:embed` directive
- Helm chart: if `loader.html` is served separately (e.g., Nginx sidecar or as a ConfigMap), document which approach was chosen and ensure it's part of the chart update

### Source Tree

```
static/
  loader.html           ← moved here; Drive URL removed; API fetch added
cmd/
  api/
    main.go             ← serve static/ via //go:embed or fileserver
charts/
  fourdogs-central-api/
    templates/          ← update if static serving changes chart
```

### References

- [Source: phases/devproposal/epics.md — Story 4.2 ACs, NFR6, FR7, NFR1, FR23, FR24]
- [Source: phases/businessplan/prd.md — FR7, FR23 (localStorage), FR24 (client-side CSV)]
- [Source: phases/techplan/architecture.md — Static file serving, CORS context]

## Dev Agent Record

### Agent Model Used

GitHub Copilot / Claude Sonnet 4.6

### Debug Log References

### Completion Notes List

### File List
