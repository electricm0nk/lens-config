---
initiative: fourdogs-central-oauthredesign
phase: devproposal
track: tech-change
completedAt: '2026-04-06'
---

# Epics — fourdogs-central-oauthredesign

## Epic 1: OAuth redirect URI externalization

**Goal:** Move the hardcoded OAuth redirect URI to a Vault-sourced env var, fix the internal hostname typo, and wire the new var through the ESO/Ansible pipeline.

**Status:** DONE — code shipped to `fourdogs-central` (commit `8278288`) and `terminus.infra` (commit `00da8d6`) as part of the TechPlan phase.

| Story | Title | Status |
|-------|-------|--------|
| 1.1 | Add `OAuthRedirectURI` to config struct + required env var list | done |
| 1.2 | Use `cfg.OAuthRedirectURI` in `cmd/api/main.go` | done |
| 1.3 | Add `OAUTH_REDIRECT_URI` key to `externalsecret.yaml` | done |
| 1.4 | Fix `ingress.host` typo (`fourdogs.trantor.internal` → `fourdogs-central.trantor.internal`) | done |
| 1.5 | Add `oauth_redirect_uri` extra-var to `seed-fourdogs-secrets.yml` | done |

---

## Epic 2: Deployment ceremony

**Goal:** Seed the new secret to Vault, trigger a fresh deployment, and execute the end-to-end smoke test that was blocked by the OAuth failure.

| Story | Title | Status |
|-------|-------|--------|
| 2.1 | Run `Seed fourdogs secrets` in Semaphore with 6 vars (incl. `OAUTH_REDIRECT_URI`) | — |
| 2.2 | Verify ArgoCD sync + ESO picks up `OAUTH_REDIRECT_URI` in pod env | — |
| 2.3 | E2E smoke test: OAuth sign-in + Betsy ordering session at `central.fourdogspetsupplies.com` | — |
