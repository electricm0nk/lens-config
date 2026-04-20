---
feature: transactions
doc_type: epics
status: draft
goal: "Deliver the eTailPet trigger worker end-to-end: from API research through production-ready scheduled deployment."
key_decisions:
  - Three epics map to the three sprints in sprint-plan.md; no epic crosses sprint boundaries
  - Epic 1 is a hard gate — no implementation code written until all open questions are resolved
  - Epic 3 closes the feature with production enablement and a validation sign-off
open_questions: []
depends_on: []
blocks: []
updated_at: "2026-04-20T00:00:00Z"
---

# Epics: transactions

## Epic Map

| Epic | Title | Sprint | Stories | Outcome |
|---|---|---|---|---|
| EPIC-0 | eTailPet API Discovery | Sprint 0 | transactions-0-01 | All API open questions answered; Vault secret provisioned |
| EPIC-1 | Core Worker Implementation | Sprint 1 | transactions-1-01, 1-02, 1-03, 1-04 | Go binary + client + tests + CI; `--dry-run` validated |
| EPIC-2 | Deploy, Wire, and Validate | Sprint 2 | transactions-2-01, 2-02, 2-03, 2-04 | Worker live in staging; email delivery confirmed; runbook written |

---

## EPIC-0: eTailPet API Discovery

**Goal:** Resolve all unknowns about the eTailPet API before a single line of code is written.

**Why first:** Every implementation decision (auth layer, HTTP client shape, retry classification,
credential field names, Vault secret structure) depends on knowing the API mechanism. Starting
EPIC-1 before EPIC-0 is complete risks implementing the wrong auth pattern and having to rewrite
the client.

**Scope:**
- Confirm auth mechanism (API key / OAuth / basic auth), header name, and credential format
- Confirm trigger endpoint URL, HTTP method, and request body
- Document rate limit / quota
- Document response shape (status codes, job ID or correlation token)
- Provision Vault secret at `secret/terminus/fourdogs/etailpet-api`
- Write `etailpet-api-spike.md` with all findings

**Exit criteria:** All 5 open questions from tech-plan answered; Vault secret exists.

**Time-box:** 5 business days. Escalate to Todd if blocked >3 days without resolution path.

---

## EPIC-1: Core Worker Implementation

**Goal:** A working, tested Go worker that can trigger the eTailPet API in `--dry-run` mode with
full unit test coverage and CI green.

**Depends on:** EPIC-0 complete.

**Scope:**
- `internal/etailpet/` package: client, config, retry, structured logging
- `cmd/etailpet-trigger` binary: env config, `time.Ticker` scheduler, `--dry-run` flag, graceful shutdown
- Unit tests: mock HTTP transport for all client paths; config validation
- CI: `go test ./...` and `go vet ./...` cover new packages

**Exit criteria:** `go test ./...` green; `--dry-run` mode exits 0 with correct log events; no
network calls in unit tests.

---

## EPIC-2: Deploy, Wire, and Validate

**Goal:** Worker deployed to staging via GitOps, credentials injected via ESO, live trigger
confirmed, operator runbook written.

**Depends on:** EPIC-1 complete.

**Scope:**
- Helm chart `fourdogs-etailpet-trigger` with liveness probe, safe `TRIGGER_ENABLED=false` default
- ArgoCD Application in `fourdogs` project
- ESO `ExternalSecret` wiring from Vault to k8s
- CI/CD release infrastructure: Semaphore templates, Ansible seed playbook, dev Vault path, first Temporal release
- Staging validation: live trigger fires; eTailPet email arrives in emailfetcher inbox
- Operator runbook committed to `docs/fourdogs/etailpet-api/transactions/runbook.md`

**Exit criteria:** ArgoCD reports `Synced / Healthy`; `trigger_success` logged in staging;
email delivered to emailfetcher inbox; runbook committed.

**Production enable:** After staging sign-off, set `TRIGGER_ENABLED=true` in production
values.yaml and sync via ArgoCD. Monitor for 7 days before closing feature.
