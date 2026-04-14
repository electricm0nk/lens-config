# Story 3.1: Service Config and ServiceCard Component

Status: done

## Story

As a platform operator,
I want all 9 services defined in `src/config/services.js` and rendered as individual cards,
so that the portal displays every service with its name, description, category, and URL.

## Acceptance Criteria

1. `SERVICES` array contains all 9 service entries
2. Exactly 9 `ServiceCard` components render on the portal
3. Each card displays: service name, description, category label, icon from Simple Icons CDN
4. Each ENABLED card is `<a>` with `href=serviceURL`, `target="_blank"`, `rel="noopener noreferrer"`
5. Fourdogs card (`enabled: false`) renders as non-interactive `<div>` with "pending" indicator
6. Icon `<img>` elements have non-empty `alt` attributes
7. No service URL/healthCheck URL stored outside `src/config/services.js`

## Tasks / Subtasks

- [ ] Task 1: Write failing component tests FIRST (TDD) (AC: #4, #5, #6)
  - [ ] `src/__tests__/components/ServiceCard.test.jsx`:
    - [ ] Given `enabled: true` → renders as `<a>` with correct href, target, rel
    - [ ] Given `enabled: false` → renders as `<div>`, NOT `<a>`
    - [ ] Icon `<img>` has non-empty `alt` attribute
    - [ ] Card displays name and description
  - [ ] Run tests — confirm FAIL (red)
- [ ] Task 2: Populate `src/config/services.js` with all 9 services (AC: #1, #7)
  - [ ] Each entry shape: `{ id, name, description, category, url, iconSlug, healthCheck: { url, enabled }, enabled }`
  - [ ] Services (id, name, category, url, healthCheck.url):
    - `argocd` / ArgoCD / infra / `https://argo.trantor.internal` / `https://argo.trantor.internal/healthz`
    - `vault` / Vault / infra / `https://vault.trantor.internal` / `https://vault.trantor.internal/v1/sys/health`
    - `semaphore` / Semaphore / infra / `https://semaphore.trantor.internal` / `https://semaphore.trantor.internal/api/ping`
    - `k3s-dashboard` / k3s Dashboard / infra / `https://k3s.trantor.internal` / `https://k3s.trantor.internal/healthz`
    - `pgadmin` / pgAdmin / data / `https://pgadmin.trantor.internal` / `https://pgadmin.trantor.internal/misc/ping`
    - `ollama` / Ollama Gateway / ai / `https://ollama.trantor.internal` / `https://ollama.trantor.internal/api/version`
    - `proxmox` / Proxmox / infra / `https://proxmox.trantor.internal:8006` / `https://proxmox.trantor.internal:8006/api2/json/version`
    - `consul` / Consul / infra / `http://vault.trantor.internal:8500/ui` / `http://vault.trantor.internal:8500/v1/status/leader`
    - `fourdogs` / Fourdogs / app / `#` / — , `enabled: false`, `healthCheck.enabled: false`
  - [ ] `iconSlug` values (Simple Icons slugs): argocd, vault, semaphoreui, kubernetes, postgresql, ollama, proxmox, consul, — (fourdogs: placeholder)
- [ ] Task 3: Create `src/components/ServiceCard.jsx` (AC: #2, #3, #4, #5, #6)
  - [ ] Props: `{ service, status }` (status passed later in Story 4.3 — default `null` for now)
  - [ ] If `service.enabled === true`: render `<a href={service.url} target="_blank" rel="noopener noreferrer">`
  - [ ] If `service.enabled === false`: render `<div>` with "PENDING" badge, no link behavior
  - [ ] Icon: `<img src={`https://cdn.simpleicons.org/${service.iconSlug}`} alt={service.name} />`
  - [ ] Category label, name, description all displayed
  - [ ] All colors/styles from `useTheme()` tokens — no hardcoded hex
- [ ] Task 4: Render 9 `ServiceCard` components in `App.jsx` (AC: #2)
  - [ ] `SERVICES.map(s => <ServiceCard key={s.id} service={s} />)`
- [ ] Task 5: Run all tests — green (AC: #4–#6)

## Dev Notes

**Target repo:** `TargetProjects/terminus/portal/terminus-portal`
**Depends on:** Story 2.1 — `useTheme()` must exist.
**Simple Icons CDN:** `https://cdn.simpleicons.org/{slug}` returns an SVG. Use `<img>` tag, not inline SVG. Set `width` and `height` via token or fixed value (e.g. 24px).
**No health status yet:** `ServiceCard` accepts an optional `status` prop (used in Story 4.3). For this story, omit or pass `null`.
**Fourdogs:** `enabled: false` — no href, no healthCheck. Renders as div with visual "PENDING" or "COMING SOON" indicator using `textMuted` token color.
**NFR6:** All URLs (service URLs + healthCheck URLs) must live ONLY in `services.js` — no duplication elsewhere.

### Project Structure Notes

Modified:
- `src/config/services.js` (fully populated)

New:
```
src/
└── components/
    └── ServiceCard.jsx
src/
└── __tests__/components/
    └── ServiceCard.test.jsx
```

Modified:
- `src/App.jsx` (render ServiceCard loop)

### References

- FR1 (service registry, 9 services): `docs/terminus/portal/portal2/epics.md`
- FR6 (disabled fourdogs card): `docs/terminus/portal/portal2/epics.md`
- FR9 (Simple Icons CDN): `docs/terminus/portal/portal2/epics.md`
- TR2 (SERVICES array in services.js), NFR6 (no secrets): `docs/terminus/portal/portal2/epics.md`
- Architecture: `docs/terminus/portal/portal2/architecture.md` — Section "Service Registry"

## Dev Agent Record

### Agent Model Used

### Debug Log References

### Completion Notes List

### File List
