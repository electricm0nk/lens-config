---
status: complete
initiative: fourdogs-central-oauthredesign
track: tech-change
phase: techplan
completedAt: '2026-04-06'
inputDocuments:
  - _bmad-output/lens-work/initiatives/fourdogs/central/phases/techplan/architecture.md
  - TargetProjects/fourdogs/central/fourdogs-central/internal/auth/oauth.go
  - TargetProjects/fourdogs/central/fourdogs-central/cmd/api/main.go
  - TargetProjects/fourdogs/central/fourdogs-central/internal/config/config.go
  - TargetProjects/fourdogs/central/fourdogs-central/deploy/helm/fourdogs-central/values.yaml
  - TargetProjects/fourdogs/central/fourdogs-central/deploy/helm/fourdogs-central/templates/externalsecret.yaml
  - TargetProjects/terminus/infra/terminus.infra/platforms/k3s/ansible/playbooks/seed-fourdogs-secrets.yml
  - _bmad-output/lens-work/initiatives/fourdogs/central/.todo/2026-04-06-oauth-redirect-redesign.md
---

# Architecture: fourdogs-central OAuth Redesign

_Scoped amendment to the foundation architecture. All decisions from
`phases/techplan/architecture.md` remain in force unless explicitly
superseded here._

---

## Problem Statement

Google OAuth 2.0 does not permit private/intranet hostnames as redirect
URIs. The foundation deployed the app exclusively on
`fourdogs-central.trantor.internal` (internal Vault PKI cert) and
hardcoded that hostname as the OAuth redirect URI in `cmd/api/main.go`.
The Google Cloud Console project has `central.fourdogspetsupplies.com`
registered. These never matched — every OAuth callback returned
`redirect_uri_mismatch`.

A secondary defect: the internal ingress hostname was misregistered in
`values.yaml` as `fourdogs.trantor.internal` instead of
`fourdogs-central.trantor.internal`.

---

## Scope

**In scope:**
- Establish a single canonical public URL for all app access
- Make the public hostname swappable without code changes
- Fix the redirect URI configuration chain end-to-end
- Fix the internal hostname misregistration

**Out of scope:**
- Dual-hostname CORS support (not needed — single canonical URL)
- Any changes to auth logic, session handling, or the OAuth library
- Any changes to the items API, ingestion pipeline, or data model
- Changes to the Traefik, cert-manager, or ESO infrastructure (already correct)

---

## Root Cause Analysis

| # | Defect | Location |
|---|--------|----------|
| 1 | `OAUTH_REDIRECT_URI` hardcoded as `https://fourdogs.trantor.internal/auth/callback` | `cmd/api/main.go:39` |
| 2 | `OAUTH_REDIRECT_URI` not managed as a Vault secret — cannot be updated without a code change and redeploy | `internal/config/config.go`, `templates/externalsecret.yaml`, playbook |
| 3 | Internal ingress hostname in `values.yaml` is `fourdogs.trantor.internal` (wrong) | `deploy/helm/fourdogs-central/values.yaml` |

---

## Architecture Decision: Single Canonical Public URL

**Decision:** `central.fourdogspetsupplies.com` (or whatever `values.yaml`
`ingress.externalHost` is set to) is the **only** URL Betsy uses for the
app. The internal URL (`fourdogs-central.trantor.internal`) is reserved
for infrastructure and operator access only.

**Rationale:**
- Google OAuth always redirects to the registered callback URL. If Betsy
  starts sign-in from the internal URL, Google drops her back at the
  external URL — split-brain session state.
- CORS: `loader.html` is served by the same origin as the API, so
  cross-origin requests never occur in normal usage. `CORS_ALLOWED_ORIGIN`
  is set to the public URL and is effectively a no-op defence layer.
- Single URL eliminates the redirect confusion entirely.

---

## Architecture Decision: `OAUTH_REDIRECT_URI` as Vault-sourced Env Var

**Decision:** Add `OAUTH_REDIRECT_URI` to the Vault secret at
`secret/terminus/fourdogs/central`. Surface it as an env var via ESO →
k8s Secret → pod env. `cmd/api/main.go` reads it from `cfg.OAuthRedirectURI`.

**Rationale:** The redirect URI is operationally coupled to the public
hostname, not to the code. It must be updatable without a code change or
image rebuild. Vault is already the authoritative secret store for this
service — this is a natural extension of the existing pattern.

**Value:** `https://{ingress.externalHost}/auth/google/callback`
Current: `https://central.fourdogspetsupplies.com/auth/google/callback`

---

## Hostname Swap Procedure

When the public hostname changes, the complete procedure is:

1. **`values.yaml`** — update `ingress.externalHost` to the new hostname.
   ArgoCD sync triggers: new cert requested from `letsencrypt-prod` via
   HTTP-01, new ingress rule added.
2. **Vault** — run `Seed fourdogs secrets` in Semaphore with updated
   `oauth_redirect_uri` = `https://{new-host}/auth/google/callback` and
   `cors_allowed_origin` = `https://{new-host}`. ESO refreshes the k8s
   Secret within 1h (or immediately on pod restart).
3. **Google Cloud Console** — register the new redirect URI, remove the old
   one.

No code changes. No image rebuild. DNS propagation + cert issuance are the
only wait times.

---

## Changes Inventory

### `TargetProjects/fourdogs/central/fourdogs-central`

| File | Change |
|------|--------|
| `internal/config/config.go` | Add `OAuthRedirectURI string` field; add `"OAUTH_REDIRECT_URI"` to required vars list |
| `cmd/api/main.go` | Pass `cfg.OAuthRedirectURI` to `auth.NewOAuthConfig(...)` instead of hardcoded string |
| `deploy/helm/fourdogs-central/values.yaml` | Fix `ingress.host` from `fourdogs.trantor.internal` → `fourdogs-central.trantor.internal` |
| `deploy/helm/fourdogs-central/templates/externalsecret.yaml` | Add `OAUTH_REDIRECT_URI` secret key from Vault |

### `TargetProjects/terminus/infra/terminus.infra`

| File | Change |
|------|--------|
| `platforms/k3s/ansible/playbooks/seed-fourdogs-secrets.yml` | Add `oauth_redirect_uri` to required extra-vars; add it to the Vault write task and verification assertion |

---

## What Does Not Change

All decisions from the foundation `architecture.md` are unchanged:
- Go + chi + pgx/v5 + sqlc stack
- `golang.org/x/oauth2` + `go-oidc/v3` library selection
- `google_sub` as session primary key
- State cookie CSRF pattern (`oauth_state` HttpOnly cookie → SameSite Lax)
- Token persistence in `sessions` table (pod-restart safe)
- ESO → Vault → k8s Secret pipeline
- Traefik ingress + cert-manager dual-cert pattern (already correct)
- All naming conventions, error formats, and API contracts
