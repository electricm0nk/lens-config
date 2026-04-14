---
initiative: fourdogs-central-oauthredesign
phase: devproposal
track: tech-change
completedAt: '2026-04-06'
---

# Stories — fourdogs-central-oauthredesign

## Epic 1 stories (DONE — shipped in TechPlan phase)

All Epic 1 stories were implemented atomically during the TechPlan phase because the scope was trivial (< 30 LoC across 5 files). No separate sprint execution was needed.

### Story 1.1 — Add `OAuthRedirectURI` to config struct
- **Status:** done
- **Commit:** `fourdogs-central@8278288`
- **Change:** `internal/config/config.go` — added `OAuthRedirectURI string` field; added `"OAUTH_REDIRECT_URI"` to required-vars slice; mapped `os.Getenv("OAUTH_REDIRECT_URI")` in return struct.

### Story 1.2 — Wire `cfg.OAuthRedirectURI` through main.go
- **Status:** done
- **Commit:** `fourdogs-central@8278288`
- **Change:** `cmd/api/main.go` line 39 — replaced hardcoded `"https://fourdogs.trantor.internal/auth/callback"` with `cfg.OAuthRedirectURI`.

### Story 1.3 — Add `OAUTH_REDIRECT_URI` to ExternalSecret
- **Status:** done
- **Commit:** `fourdogs-central@8278288`
- **Change:** `deploy/helm/fourdogs-central/templates/externalsecret.yaml` — added key block for `OAUTH_REDIRECT_URI` between `OAUTH_CLIENT_SECRET` and `CORS_ALLOWED_ORIGIN`.

### Story 1.4 — Fix `ingress.host` hostname typo
- **Status:** done
- **Commit:** `fourdogs-central@8278288`
- **Change:** `deploy/helm/fourdogs-central/values.yaml` — corrected `ingress.host` from `fourdogs.trantor.internal` to `fourdogs-central.trantor.internal`.

### Story 1.5 — Add `oauth_redirect_uri` extra-var to seed playbook
- **Status:** done
- **Commit:** `terminus.infra@00da8d6`
- **Change:** `platforms/k3s/ansible/playbooks/seed-fourdogs-secrets.yml` — added `oauth_redirect_uri` as 6th required extra-var in assert, URI write, verify, and success message tasks; updated var count from 4 to 5.

---

## Epic 2 stories (Deployment ceremony)

### Story 2.1 — Run `Seed fourdogs secrets` in Semaphore
- **Status:** in-progress (Semaphore task 6 submitted)
- **Acceptance criteria:**
  - Semaphore task completes with status `success`
  - `vault kv get secret/terminus/fourdogs/central` shows `OAUTH_REDIRECT_URI = https://central.fourdogspetsupplies.com/auth/google/callback` and `CORS_ALLOWED_ORIGIN = https://central.fourdogspetsupplies.com`

### Story 2.2 — Verify ESO refresh + pod env
- **Status:** pending (blocked on 2.1)
- **Acceptance criteria:**
  - ArgoCD shows `fourdogs-central` app `Synced / Healthy`
  - ESO `ExternalSecret` reports `Ready`
  - Running pod has `OAUTH_REDIRECT_URI` in its env (verified via `kubectl exec`)

### Story 2.3 — E2E smoke test
- **Status:** pending (blocked on 2.2, requires browser)
- **Acceptance criteria:**
  - `https://central.fourdogspetsupplies.com` loads the UI without TLS errors
  - Clicking "Sign in with Google" completes the OAuth flow and returns to the app (no `redirect_uri_mismatch` error)
  - Betsy can browse catalogue, add to cart, and complete an ordering session
  - No CORS errors in browser console
