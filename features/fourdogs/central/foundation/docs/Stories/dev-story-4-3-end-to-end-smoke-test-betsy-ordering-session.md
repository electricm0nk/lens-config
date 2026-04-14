# Story 4.3: End-to-End Smoke Test — Betsy Ordering Session

Status: ready-for-dev

## Story

As a product owner,
I want to validate the complete ordering workflow end-to-end on Betsy's actual device,
so that I can confirm the system is ready for production use.

## Acceptance Criteria

1. System is live on k3s; smoke test performed on iPad (Betsy's actual device)
2. Navigating to the app URL shows the Google sign-in screen
3. After signing in with Google account → item list loads with item count matching pre-API prototype parity
4. localStorage session state is preserved across page navigation (refresh, re-open on same device)
5. "Compile Order" button generates a downloadable CSV — no network request observed in Safari DevTools
6. Google Drive JSON URL is unreachable or returns a 403/404 error (decommissioned — NFR6)
7. Health check endpoint `GET /v1/health` returns HTTP 200 throughout the test
8. Smoke test result (pass/fail, regressions noted) is recorded in story completion notes

## Tasks / Subtasks

- [ ] Task 0: Verify Story 4.2 complete — loader.html rewired to API, deployed to k3s

- [ ] Task 1: Pre-smoke-test deployment checklist
  - [ ] Verify ArgoCD shows `fourdogs-central-api` app as `Synced` and `Healthy`
  - [ ] Verify current `loader.html` version deployed: check file hash or timestamp
  - [ ] Verify `CORS_ALLOWED_ORIGIN` matches the k3s ingress hostname
  - [ ] Verify Google OAuth client ID/secret are in Vault and ESO has synced the secret
  - [ ] Verify `GOOGLE_REDIRECT_URL` matches the deployed IngressRoute URL exactly
  - [ ] Run `curl https://fourdogs.terminus.local/v1/health` → HTTP 200 ✅
  - [ ] Note prototype item count baseline (from last Drive JSON snapshot) for parity check (AC: 3)

- [ ] Task 2: Execute smoke test on iPad (AC: 1, 2, 3, 4, 5, 6, 7)
  - [ ] Open Safari on iPad; navigate to `https://fourdogs.terminus.local`
  - [ ] AC2: Confirm Google sign-in page appears (OAuth redirect working)
  - [ ] Sign in with Betsy's Google account
  - [ ] AC3: Item list loads; count items visible — compare to prototype baseline
  - [ ] AC4: Navigate away and back (or close/reopen tab); verify items still loaded (localStorage preserved)
  - [ ] AC5: Check checkboxes for 2-3 items; tap "Compile Order"; verify CSV downloads; open Safari Network tab → confirm no fetch/XHR on compile
  - [ ] AC6: Attempt to fetch old Drive JSON URL directly in Safari → confirm 403 or 404
  - [ ] AC7: `curl https://fourdogs.terminus.local/v1/health` from terminal during test → HTTP 200
  - [ ] Note any regressions or unexpected behavior

- [ ] Task 3: Record smoke test results (AC: 8)
  - [ ] Complete the test result table in story completion notes:
    ```
    | AC  | Check                              | Result   | Notes              |
    |-----|------------------------------------|----------|--------------------|
    | AC2 | Google sign-in screen              | PASS/FAIL |                   |
    | AC3 | Item list loads with parity count  | PASS/FAIL | Count: X vs Y     |
    | AC4 | localStorage persists              | PASS/FAIL |                   |
    | AC5 | CSV download, no network request   | PASS/FAIL |                   |
    | AC6 | Drive URL unreachable              | PASS/FAIL |                   |
    | AC7 | Health check 200                   | PASS/FAIL |                   |
    ```
  - [ ] If any AC FAILS: open regression issue; do NOT mark story done

- [ ] Task 4: Post-smoke-test cleanup and artifacts
  - [ ] Commit smoke test result summary to story file (completion notes)
  - [ ] Update sprint status: `4-3-end-to-end-smoke-test-betsy-ordering-session: done`
  - [ ] Tag the release if all ACs pass: `git tag fourdogs-central-v0.1.0`

- [ ] Task 5: Mark Epic 4 and initiative foundation COMPLETE
  - [ ] Set `epic-4-authenticated-api-and-triage-ui: done` in sprint-status.yaml
  - [ ] Set sprint retrospective entry in sprint-status.yaml
  - [ ] Notify product owner (Betsy) of readiness for live use

## Dev Notes

- This story is a MANUAL test — it cannot be automated at this stage (requires Betsy's Google account + real iPad hardware)
- If Betsy is unavailable for iPad test: a desktop Chrome test with devtools mobile emulation is an acceptable interim; document the deviation
- **Item count parity (AC3):** Before decommissioning, capture the item count from the Drive JSON snapshot as the baseline. The API should return the same count (assuming same data was ingested via Story 2.1)
- **localStorage verify (AC4):** Use Safari Developer Tools → Storage → Local Storage to inspect key values before and after navigation
- **CSV no-network (AC5):** Safari DevTools → Network → filter by Fetch/XHR — should show zero requests on compile action
- **Drive decommission (AC6):** The Drive file sharing permission should be revoked as part of Story 4.2 deployment. If story 4.2 was deployed but permission not revoked, this is a story 4.2 defect, not a blocker for this story (note and track separately)
- **ARCH9 pod-restart during smoke test:** You may optionally kill the pod during the smoke test (after authentication) to verify session persistence — this fulfills the ARCH9 demonstration requirement if it was not covered in Story 3.3
- **Foundation complete:** When this story passes, the fourdogs-central-foundation initiative is feature-complete. The sprintplan PR (large-sprintplan → large) and any subsequent audience promotions can proceed

### References

- [Source: phases/devproposal/epics.md — Story 4.3 ACs, end-to-end acceptance]
- [Source: phases/businessplan/prd.md — FR23 (localStorage), FR24 (client-side CSV), NFR6 (no Drive dependency)]
- [Source: phases/techplan/architecture.md — Acceptance criteria, ARCH9 (session persistence)]

## Dev Agent Record

### Agent Model Used

GitHub Copilot / Claude Sonnet 4.6

### Debug Log References

### Completion Notes List

| AC  | Check                              | Result | Notes |
|-----|------------------------------------|--------|-------|
| AC2 | Google sign-in screen displayed    |        |       |
| AC3 | Item list loads — parity count     |        |       |
| AC4 | localStorage session preserved     |        |       |
| AC5 | CSV download, no network request   |        |       |
| AC6 | Drive URL unreachable (403/404)     |        |       |
| AC7 | Health check returns HTTP 200      |        |       |

### File List
