---
initiative: fourdogs-central-oauthredesign
phase: techplan
completedAt: '2026-04-06'
---

# Tech Decisions — fourdogs-central OAuth Redesign

## TD-1: `OAUTH_REDIRECT_URI` as required env var (not hardcoded)

**Decision:** Move the OAuth redirect URI from a hardcoded string in
`main.go` to a required env var `OAUTH_REDIRECT_URI`, sourced from Vault
via ESO.

**Alternatives considered:**
- Derive it from `ingress.externalHost` at startup via a downward API env
  var — rejected: adds Helm/k8s coupling to the Go binary; Vault is a
  cleaner and already-established pattern
- Keep hardcoded, update it manually per deploy — rejected: requires code
  change and image rebuild to swap hostnames

**Consequence:** Vault secret `secret/terminus/fourdogs/central` gains one
new key. The Semaphore seed playbook gains one required extra-var. No
change to the binary's public interface.

---

## TD-2: Single canonical public URL — no dual-URL support

**Decision:** `CORS_ALLOWED_ORIGIN` is set to the public hostname only.
No multi-origin CORS support. Internal URL is operator-only.

**Alternatives considered:**
- Multi-origin CORS (`strings.Split` on comma-separated env var) — rejected:
  adds code complexity; the real problem is OAuth redirect split-brain,
  not CORS. Solving CORS doesn't solve the session confusion.
- Dual-URL with separate Google OAuth clients — rejected: operational
  complexity; two redirect URIs to maintain; confusing for a single-user app

**Consequence:** Betsy uses the public URL for everything. Internal URL
remains accessible for operators but OAuth will redirect to the public
URL if initiated from internal.

---

## TD-3: `ingress.externalHost` in values.yaml as single swap point

**Decision:** The public hostname lives in exactly one place:
`deploy/helm/fourdogs-central/values.yaml` `ingress.externalHost`. All
downstream resources (cert, ingress rule) derive from it via Helm
templating. Vault secrets are updated separately via Semaphore.

**Consequence:** Swapping hostname = `values.yaml` edit + Semaphore run +
Google Console update. No code change. No image rebuild.
