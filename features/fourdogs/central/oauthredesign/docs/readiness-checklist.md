---
initiative: fourdogs-central-oauthredesign
phase: devproposal
track: tech-change
status: PASS
completedAt: '2026-04-06'
---

# DevProposal Readiness Checklist — fourdogs-central-oauthredesign

## Track context

- **Track:** `tech-change` — targeted fix to existing service, no new tenants, no schema migrations, no new infrastructure
- **Audience gate:** small → medium (no adversarial review required per constitution §5.3)

---

## 1. Problem definition ✅

- [x] Root cause identified and documented in `architecture.md`
- [x] Three discrete defects catalogued (hardcoded redirect URI, wrong internal hostname, Vault gap)
- [x] User-facing impact described (OAuth `redirect_uri_mismatch` blocks all sign-in)

## 2. Solution design ✅

- [x] Architecture decision: single canonical public URL (`central.fourdogspetsupplies.com`)
- [x] CORS strategy: Option A — same-origin approach, single allowed origin
- [x] Hostname swap procedure documented (swap-without-rebuild supported)
- [x] No new infrastructure required; no schema changes; no tenant changes

## 3. Code changes ✅

- [x] `config.go` — `OAuthRedirectURI` field + required-var wired
- [x] `main.go` — hardcoded string eliminated, env var used
- [x] `externalsecret.yaml` — `OAUTH_REDIRECT_URI` key added to ESO mapping
- [x] `values.yaml` — `ingress.host` hostname typo corrected
- [x] `seed-fourdogs-secrets.yml` — 6th extra-var added to seed playbook
- [x] `go build ./...` passes clean on `fourdogs-central`

## 4. Deployment pipeline ✅

- [x] Vault seed playbook updated (`terminus.infra@00da8d6`)
- [x] Semaphore template "Seed fourdogs secrets" (ID 3) supports all 6 required vars
- [x] ArgoCD will detect Helm change and re-deploy automatically
- [x] ESO will refresh `fourdogs-central-secrets` k8s Secret within 15 minutes

## 5. Observability ✅

- [x] `OAUTH_REDIRECT_URI` will be visible in pod env (verifiable via `kubectl exec`)
- [x] ArgoCD sync status is observable at `https://argocd.trantor.internal`
- [x] Semaphore task log confirms Vault write success

## 6. Risk assessment ✅

- [x] **Low risk:** change is limited to 5 files, < 30 LoC
- [x] **Rollback:** revert `fourdogs-central@8278288` and re-seed Vault with old values
- [x] **No downtime:** env var change triggers pod restart (< 30 s); OAuth was already broken pre-fix
- [x] Google Cloud Console already has `https://central.fourdogspetsupplies.com/auth/google/callback` registered

## 7. Definition of done ✅

- [x] All Epic 1 stories completed and committed
- [ ] Epic 2.1 — Semaphore seed task completes successfully
- [ ] Epic 2.2 — Pod env verified with new var
- [ ] Epic 2.3 — E2E smoke test passes (requires browser)
