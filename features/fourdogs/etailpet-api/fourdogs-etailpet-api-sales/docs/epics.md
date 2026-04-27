---
feature: sales
doc_type: epics
status: draft
goal: "Deliver the etailpet-sales-trigger worker end-to-end: www auth spike, Go implementation, and production deployment."
key_decisions:
  - Three epics mirror the transactions feature structure; Sprint 0 is a hard gate on www API confirmation
  - Epic 1 depends on Sprint 0 deliverables; no implementation code before spike is complete
  - internal/etailpet package extension (LegacyClient) is the core deliverable of Epic 1
  - Epic 2 reuses established Helm/ArgoCD/ESO patterns from the transactions worker
open_questions: []
depends_on:
  - fourdogs-etailpet-api-transactions
blocks: []
updated_at: "2026-04-27T00:00:00Z"
---

# Epics: fourdogs-etailpet-api-sales

## Epic Map

| Epic | Title | Sprint | Stories | Outcome |
|---|---|---|---|---|
| EPIC-0 | www API Confirmation Spike | Sprint 0 | sales-0-01 | www auth confirmed; sales-line trigger validated; email subject + attachment schema documented |
| EPIC-1 | LegacyClient and Sales Trigger Binary | Sprint 1 | sales-1-01, sales-1-02, sales-1-03, sales-1-04 | cmd/etailpet-sales-trigger with LegacyClient; --dry-run mode; unit tests; CI green |
| EPIC-2 | Deploy, Wire, and Validate | Sprint 2 | sales-2-01, sales-2-02, sales-2-03 | Worker live; email delivery confirmed; runbook written |

---

## EPIC-0: www API Confirmation Spike

**Goal:** Confirm the `www.etailpet.com` legacy token flow works with existing Vault credentials
and that `report_type=sales-line` triggers a real email export. Document the email subject line
and attachment column headers before any implementation begins.

**Why first:** The transactions spike confirmed the `www` token flow works for catalog endpoints,
but not for the report trigger path specifically. All three of the following must be confirmed before
Sprint 1 begins: token validity on the report route, 200 response from the sales-line trigger, and
email delivery with the expected attachment.

**Scope:**
- Obtain `www` Bearer token using existing Vault credentials
- Call `GET /fourdogspetsupplies/api/v1/transaction-report/?report_type=sales-line` with the token
- Document response status code and any response body
- Confirm email received at retailer inbox; document subject line
- Open the xlsx attachment; document the column headers (first row)
- Determine `TO_EMAIL` behavior (default vs. requires explicit param)
- Write `sales-api-spike.md` with all findings

**Exit criteria:** All 6 spike deliverables from tech-plan confirmed; spike doc committed.

**Time-box:** 3 business days. The `www` auth flow is already confirmed on catalog-export, so this
is a targeted validation, not a discovery from scratch. Escalate to Todd if eTailPet support needed.

**Gate:** Sprint 1 does not start until this sprint is complete.

---

## EPIC-1: LegacyClient and Sales Trigger Binary

**Goal:** A working, tested Go binary `cmd/etailpet-sales-trigger` with `--dry-run` mode,
`internal/etailpet.LegacyClient` fully unit-tested, and CI green.

**Depends on:** EPIC-0 complete (www auth confirmed; email delivery model validated).

**Scope:**
- `internal/etailpet/legacy_client.go`: www query-param OAuth2 token flow, trigger GET, retry, structured logging
- `cmd/etailpet-sales-trigger/main.go`: env config, `time.Ticker` scheduler, `--dry-run` flag, SIGTERM shutdown
- Unit tests: mock HTTP transport for LegacyClient; config validation; date-range formatting
- CI: `go test ./...` and `go vet ./...` cover new packages

**Exit criteria:** `go test ./...` green; `--dry-run` mode exits 0 with correct structured log events; no network calls in unit tests; `TRIGGER_ENABLED=false` verified to suppress trigger.

---

## EPIC-2: Deploy, Wire, and Validate

**Goal:** `fourdogs-etailpet-sales-trigger` deployed to staging via GitOps, ESO wired, live trigger
confirmed, and operator runbook written.

**Depends on:** EPIC-1 complete.

**Scope:**
- Helm chart `fourdogs-etailpet-sales-trigger` with liveness probe, `TRIGGER_ENABLED=false` default for staging safety
- ArgoCD Application in `fourdogs` project; same conventions as `fourdogs-etailpet-trigger`
- ESO `ExternalSecret` in namespace `fourdogs-etailpet-sales-trigger` pointing to same Vault path
- CI/CD: Semaphore pipeline, Ansible seed, dev Vault path for non-prod credentials
- Staging validation: live trigger → confirm email delivered to emailfetcher inbox
- Runbook: enable/disable, force trigger, log events reference

**Exit criteria:** Live trigger from staging sends `sales-line` xlsx email to inbox; emailfetcher
confirms receipt; runbook written and committed.
