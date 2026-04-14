---
stepsCompleted: [1, 2, 3, 4, 5, 6, 7, 8]
inputDocuments:
  - "User-provided project brief (brainstorm session)"
  - "https://github.com/electricm0nk/terminus-portal (repo inspection)"
workflowType: architecture
project_name: terminus-portal-portal2
user_name: CrisWeber
date: "2026-04-13"
initiative_root: terminus-portal-portal2
track: tech-change
phase: techplan
---

# Architecture — Terminus Platform Portal (portal2)

**Initiative:** `terminus-portal-portal2`
**Track:** `tech-change`
**Date:** 2026-04-13
**Author:** Perturabo (Architect) via lens-work techplan

---

## Project Context Analysis

### What We Are Building

A self-hosted homelab portal — a React single-page application served behind Nginx on k3s. It replaces the existing static HTML dashboard (`v1`) at `https://portal.trantor.internal`.

This is **not** a monitoring platform. It is a structured launchpad and live health surface:
- One-click navigation to every Terminus service UI
- Live health status via periodic fetch-based health checks (binary ONLINE/UNREACHABLE)
- Metric stub panel wired to future Prometheus and InfluxDB feeds
- Two selectable themes persisted to localStorage

### Existing System State

The current `terminus-portal` repo (`ghcr.io/electricm0nk/terminus-portal`) is:
- A single `index.html` static file served by `nginx:1.27-alpine`
- Helm chart at `deploy/helm/` — full chart with Deployment, Service, Ingress
- Live at `https://portal.trantor.internal`, namespace `terminus-portal`
- Built via `Makefile` → `docker build` → `ghcr.io` push → ArgoCD reconcile

**This initiative is an in-place replacement.** The chart, namespace, ingress hostname, and image registry path are preserved. Only the application layer (HTML→React/Vite) and Dockerfile (single-stage→multi-stage) change.

### Requirements Summary

**Functional:**
- Service registry: 8 services, categorized, with optional health check URLs
- Health polling: every 30s, 5s timeout, fetch no-cors
- Metric stub panel: 6 cards (CPU, Memory, Net In/Out, Pods, Disk)
- Theme system: Terminal (dark green/CRT) and Space Ops (navy/HUD)
- Header: cluster identity, live summary, last-checked timestamp, theme toggle, manual refresh

**Non-Functional:**
- TDD required — full test pyramid (unit, integration, E2E)
- Static build (`dist/`) — no backend, no SSR
- No Docker Compose — k3s only
- No secrets in portal — all URLs are config
- Accessible — semantic HTML, keyboard navigation, screen reader labels

### Constraints

- `no-cors` fetch means opaque responses — status codes unreadable from cross-origin health endpoints
- No external CSS framework — CSS-in-JS via inline token objects only
- No external state management — React state is sufficient
- Build target is `dist/` folder served by Nginx

---

## Architectural Decisions

### Decision 1 — Health Check Strategy

**Decision:** Accept opaque responses. Health status is binary: `ONLINE` / `UNREACHABLE`.

**Rationale:** `fetch` with `mode: 'no-cors'` returns an opaque response for cross-origin requests — `status: 0`, body unreadable. A successful opaque response (no network error) signals ONLINE. A `TypeError` (network failure, timeout) signals UNREACHABLE.

**Vault special case:** The brief specifies distinguishing `200 / 429 / 503` for Vault. This is architecturally incompatible with `no-cors` without Vault serving `Access-Control-Allow-Origin` headers. This distinction is **deferred** — Vault is treated as binary ONLINE/UNREACHABLE in portal2.

**Future wiring point:** `VAULT_STATUS_DETAIL` — when a proxy tier is introduced (Caddy reverse proxy rule or lightweight sidecar that exposes a same-origin `/api/health/vault` endpoint), replace the Vault health check URL in `services.js` with `'/api/health/vault'` and update the status resolver to read the response body. No component changes required.

**Status enum:**
```js
// src/constants/status.js
export const STATUS = {
  CHECKING:     'CHECKING',
  ONLINE:       'ONLINE',
  UNREACHABLE:  'UNREACHABLE',
  NO_CHECK:     'NO_CHECK',
}
```

**Health check logic (per service, on interval):**
```js
async function checkHealth(url, timeoutMs = 5000) {
  const controller = new AbortController()
  const id = setTimeout(() => controller.abort(), timeoutMs)
  try {
    await fetch(url, { mode: 'no-cors', signal: controller.signal })
    return STATUS.ONLINE        // opaque but received — no network error
  } catch {
    return STATUS.UNREACHABLE   // TypeError: network failure or abort
  } finally {
    clearTimeout(id)
  }
}
```

---

### Decision 2 — Metric Card Data Shape

**Decision:** Prop-driven `metric` object. Parent component owns the static array. Context upgrade path documented but not implemented.

**Metric prop shape:**
```js
/**
 * @typedef {Object} MetricDef
 * @property {string}            id        - unique identifier
 * @property {string}            label     - display label
 * @property {string}            unit      - display unit (e.g. '%', 'GB', 'pods')
 * @property {'prometheus'|'influxdb'} source - data source identifier
 * @property {number|string|null} value    - current value; null = unwired
 * @property {boolean}           wired     - false = stub display
 */
```

**Current static definitions (all unwired):**
```js
// src/config/metrics.js
export const METRICS = [
  { id: 'cpu-load',    label: 'CPU Load',     unit: '%',   source: 'prometheus', value: null, wired: false },
  { id: 'mem-used',    label: 'Memory Used',  unit: 'GB',  source: 'prometheus', value: null, wired: false },
  { id: 'net-in',      label: 'Net In',       unit: 'MB/s',source: 'influxdb',   value: null, wired: false },
  { id: 'net-out',     label: 'Net Out',      unit: 'MB/s',source: 'influxdb',   value: null, wired: false },
  { id: 'pods-running',label: 'Pods Running', unit: 'pods',source: 'prometheus', value: null, wired: false },
  { id: 'disk-used',   label: 'Disk Used',    unit: 'GB',  source: 'influxdb',   value: null, wired: false },
]
```

**Future wiring:** When Prometheus/InfluxDB feeds are available, extract `METRICS` into a `MetricsContext` provider that polls and updates `value`. `MetricCard` components require zero changes — they consume the same prop shape.

---

### Decision 3 — Theme Architecture

**Decision:** `ThemeContext` React context provider. Components consume via `useTheme()` hook. localStorage persistence lives in the provider.

**Token structure:**
```js
// src/themes/terminal.js  — Terminal: dark green on near-black, monospaced, CRT scanlines
export const terminal = {
  name: 'terminal',
  bg: '#0a0a0a',
  bgSurface: '#0f110f',
  bgCard: '#111411',
  text: '#b5d9b5',
  textMuted: '#5a7a5a',
  accent: '#00ff41',
  accentDim: '#00cc33',
  border: '#1a2e1a',
  borderActive: '#00ff41',
  statusOnline: '#00ff41',
  statusUnreachable: '#ff3333',
  statusChecking: '#cccc00',
  statusNoCheck: '#444444',
  fontFamily: '"JetBrains Mono", "Fira Code", "Courier New", monospace',
  scanlines: true,
}

// src/themes/spaceops.js  — Space Ops: dark navy, HUD aesthetic, blue accent
export const spaceops = {
  name: 'spaceops',
  bg: '#0d1b2a',
  bgSurface: '#112236',
  bgCard: '#132840',
  text: '#e9eefc',
  textMuted: '#9aa7c7',
  accent: '#4fc3f7',
  accentDim: '#0288d1',
  border: '#1e3a5f',
  borderActive: '#4fc3f7',
  statusOnline: '#4caf50',
  statusUnreachable: '#ef5350',
  statusChecking: '#ffa726',
  statusNoCheck: '#546e7a',
  fontFamily: '"Space Grotesk", "Avenir Next", "Segoe UI", sans-serif',
  scanlines: false,
}

// src/themes/index.js
export { terminal } from './terminal'
export { spaceops } from './spaceops'
export const THEMES = { terminal, spaceops }
export const DEFAULT_THEME = 'spaceops'
```

**Context provider:**
```jsx
// src/context/ThemeContext.jsx
const STORAGE_KEY = 'terminus-portal-theme'

export function ThemeProvider({ children }) {
  const [themeName, setThemeName] = useState(
    () => localStorage.getItem(STORAGE_KEY) ?? DEFAULT_THEME
  )
  const tokens = THEMES[themeName] ?? THEMES[DEFAULT_THEME]

  const toggleTheme = useCallback(() => {
    setThemeName(prev => {
      const next = prev === 'terminal' ? 'spaceops' : 'terminal'
      localStorage.setItem(STORAGE_KEY, next)
      return next
    })
  }, [])

  return (
    <ThemeContext.Provider value={{ tokens, themeName, toggleTheme }}>
      {children}
    </ThemeContext.Provider>
  )
}

export function useTheme() {
  return useContext(ThemeContext)
}
```

**CRT scanline overlay:** When `tokens.scanlines === true`, `App` renders a fixed-position pseudo-element overlay via inline style. No CSS file required.

---

### Decision 4 — Service Registry

**Decision:** Static `src/config/services.js`, version-controlled, imported at build time.

**Service object shape:**
```js
/**
 * @typedef {Object} ServiceDef
 * @property {string}  id          - unique slug
 * @property {string}  name        - display name
 * @property {string}  description - short description
 * @property {string}  category    - display category label
 * @property {string}  url         - service UI URL (opens in new tab)
 * @property {string}  icon        - Simple Icons slug (https://cdn.simpleicons.org/{slug}/{color})
 * @property {string}  iconColor   - hex color for Simple Icons CDN URL
 * @property {{ url: string, enabled: boolean }} healthCheck
 * @property {boolean} enabled     - false = disabled card (fourdogs pending)
 */
```

**Initial registry:**
```js
// src/config/services.js
export const SERVICES = [
  {
    id: 'argocd',
    name: 'ArgoCD',
    description: 'GitOps continuous delivery',
    category: 'platform',
    url: 'https://argocd.trantor.internal',
    icon: 'argo',
    iconColor: 'f75b3b',
    healthCheck: { url: 'https://argocd.trantor.internal/healthz', enabled: true },
    enabled: true,
  },
  {
    id: 'vault',
    name: 'Vault',
    description: 'Secrets management — active/standby/sealed status',
    category: 'infra',
    url: 'https://vault.trantor.internal',
    icon: 'vault',
    iconColor: 'ffd74a',
    // VAULT_STATUS_DETAIL: binary ONLINE/UNREACHABLE only (no-cors opaque responses).
    // Future: replace url with '/api/health/vault' when proxy tier available.
    healthCheck: { url: 'https://vault.trantor.internal/v1/sys/health', enabled: true },
    enabled: true,
  },
  {
    id: 'semaphore',
    name: 'Semaphore',
    description: 'Pipeline and automation task management',
    category: 'platform',
    url: 'https://semaphore.trantor.internal',
    icon: 'semaphoreci',
    iconColor: '62d84e',
    healthCheck: { url: 'https://semaphore.trantor.internal/api/ping', enabled: true },
    enabled: true,
  },
  {
    id: 'k3s-dashboard',
    name: 'k3s Dashboard',
    description: 'Kubernetes cluster management',
    category: 'infra',
    url: 'https://k3s-dashboard.trantor.internal',
    icon: 'kubernetes',
    iconColor: '326ce5',
    healthCheck: { url: '', enabled: false },
    enabled: true,
  },
  {
    id: 'pgadmin',
    name: 'pgAdmin',
    description: 'PostgreSQL administration',
    category: 'data',
    url: 'https://pgadmin.trantor.internal',
    icon: 'postgresql',
    iconColor: '4169e1',
    healthCheck: { url: 'https://pgadmin.trantor.internal/misc/ping', enabled: true },
    enabled: true,
  },
  {
    id: 'ollama-gateway',
    name: 'Ollama Gateway',
    description: 'Local LLM inference gateway',
    category: 'ai',
    url: 'https://ollama.trantor.internal',
    icon: 'ollama',
    iconColor: 'ffffff',
    healthCheck: { url: 'https://ollama.trantor.internal/api/version', enabled: true },
    enabled: true,
  },
  {
    id: 'proxmox',
    name: 'Proxmox',
    description: 'Hypervisor management UI on trantor',
    category: 'infra',
    url: 'https://proxmox.trantor.internal:8006',
    icon: 'proxmox',
    iconColor: 'ff6f00',
    healthCheck: { url: '', enabled: false },
    enabled: true,
  },
  {
    id: 'consul',
    name: 'Consul',
    description: 'Service mesh and health registry',
    category: 'infra',
    url: 'http://vault.trantor.internal:8500/ui',
    icon: 'consul',
    iconColor: 'f24c53',
    healthCheck: { url: 'http://vault.trantor.internal:8500/v1/status/leader', enabled: true },
    enabled: true,
  },
  {
    id: 'fourdogs',
    name: 'Fourdogs',
    description: 'Fourdogs application — URL pending',
    category: 'app',
    url: '',
    icon: 'dogecoin',
    iconColor: 'f2a900',
    healthCheck: { url: '', enabled: false },
    enabled: false,  // disabled until URL is confirmed
  },
]
```

---

### Decision 5 — Helm Chart & Deployment

**Decision:** Adapt existing `deploy/helm/` chart. Replace single-stage Dockerfile with multi-stage Vite build.

**Updated Dockerfile:**
```dockerfile
# Stage 1 — build
FROM node:20-alpine AS build
WORKDIR /app
COPY package*.json ./
RUN npm ci --prefer-offline
COPY . .
RUN npm run build

# Stage 2 — serve
FROM nginx:1.27-alpine
RUN rm -rf /usr/share/nginx/html/*
COPY --from=build /app/dist /usr/share/nginx/html
COPY nginx.conf /etc/nginx/conf.d/default.conf
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

**nginx.conf (SPA routing required):**
```nginx
server {
  listen 80;
  server_tokens off;
  root /usr/share/nginx/html;
  index index.html;

  location / {
    try_files $uri $uri/ /index.html;
  }

  location ~* \.(js|css|png|jpg|svg|ico|woff2?)$ {
    expires 1y;
    add_header Cache-Control "public, immutable";
  }
}
```

**Helm chart changes required:**
- `values.yaml`: update `image.repository` to `ghcr.io/electricm0nk/terminus-portal`, `image.tag` managed by CI
- Ingress host remains `portal.trantor.internal`
- Namespace remains `terminus-portal`
- No new Helm resources required

**Makefile update:**
```makefile
IMAGE = ghcr.io/electricm0nk/terminus-portal
TAG   ?= latest

.PHONY: build run push

build:
	docker build -t $(IMAGE):$(TAG) .

run:
	docker run --rm -p 8080:80 $(IMAGE):$(TAG)

push:
	docker push $(IMAGE):$(TAG)
```
No changes to Makefile — multi-stage build is transparent.

---

## Component Architecture

### Directory Structure
```
terminus-portal/
├── src/
│   ├── config/
│   │   ├── services.js         # ServiceDef array — all 8 services
│   │   └── metrics.js          # MetricDef array — 6 stub cards
│   ├── constants/
│   │   └── status.js           # STATUS enum
│   ├── themes/
│   │   ├── terminal.js         # Terminal token object
│   │   ├── spaceops.js         # Space Ops token object
│   │   └── index.js            # THEMES map, DEFAULT_THEME
│   ├── context/
│   │   └── ThemeContext.jsx    # ThemeProvider, useTheme hook
│   ├── hooks/
│   │   └── useHealthCheck.js   # polling hook for a single service
│   ├── components/
│   │   ├── Header/
│   │   │   ├── Header.jsx
│   │   │   └── Header.test.jsx
│   │   ├── ServiceCard/
│   │   │   ├── ServiceCard.jsx
│   │   │   └── ServiceCard.test.jsx
│   │   ├── ServiceGrid/
│   │   │   ├── ServiceGrid.jsx
│   │   │   └── ServiceGrid.test.jsx
│   │   ├── MetricCard/
│   │   │   ├── MetricCard.jsx
│   │   │   └── MetricCard.test.jsx
│   │   ├── MetricsPanel/
│   │   │   ├── MetricsPanel.jsx
│   │   │   └── MetricsPanel.test.jsx
│   │   └── StatusIndicator/
│   │       ├── StatusIndicator.jsx
│   │       └── StatusIndicator.test.jsx
│   ├── App.jsx
│   ├── App.test.jsx
│   └── main.jsx
├── e2e/
│   └── portal.spec.js          # Playwright E2E
├── deploy/
│   └── helm/                   # existing chart — adapted
│       ├── Chart.yaml
│       ├── values.yaml
│       └── templates/
├── Dockerfile                  # multi-stage (node:20 build + nginx:1.27 serve)
├── nginx.conf
├── Makefile
├── vite.config.js
├── package.json
└── index.html                  # Vite entry point (minimal)
```

### Component Responsibilities

| Component | Responsibility |
|---|---|
| `App` | Root; owns service health state array; renders `Header`, `ServiceGrid`, `MetricsPanel`; manages poll interval |
| `Header` | Cluster identity, live summary (X/Y ONLINE), last-checked, theme toggle, manual refresh |
| `ServiceGrid` | Groups `ServiceCard` by category; receives services with resolved status |
| `ServiceCard` | Renders one service — name, icon, description, URL, status badge; disabled state for `enabled: false` |
| `StatusIndicator` | Colored dot + label for ONLINE/UNREACHABLE/CHECKING/NO_CHECK; aria-label required |
| `MetricsPanel` | Renders 6 `MetricCard` components from `METRICS` config |
| `MetricCard` | Renders one metric — label, value (`—` when unwired), unit, source badge, wired/unwired label |
| `ThemeProvider` | Context root; holds theme name + tokens; localStorage read/write; `toggleTheme` |
| `useHealthCheck` | Custom hook — polls one service URL, returns `status`; handles abort on unmount |

### Health Polling Design

`App` owns the composite health state to enable the header summary (X/Y ONLINE):

```js
// App.jsx (simplified)
const [healthState, setHealthState] = useState(
  () => Object.fromEntries(SERVICES.map(s => [s.id, STATUS.CHECKING]))
)

useEffect(() => {
  const checkAll = async () => {
    const results = await Promise.allSettled(
      SERVICES.filter(s => s.healthCheck.enabled)
               .map(s => checkHealth(s.healthCheck.url)
                           .then(status => ({ id: s.id, status })))
    )
    setHealthState(prev => {
      const next = { ...prev }
      results.forEach(r => { if (r.status === 'fulfilled') next[r.value.id] = r.value.status })
      return next
    })
  }

  checkAll()
  const interval = setInterval(checkAll, 30_000)
  return () => clearInterval(interval)
}, [])
```

Services with `healthCheck.enabled: false` are initialized to `STATUS.NO_CHECK` and never polled.

---

## Testing Strategy

**Requirement:** TDD — tests before implementation, full pyramid.

| Layer | Tool | Scope |
|---|---|---|
| Unit | Vitest | Pure functions (`checkHealth`, status utils, theme token access) |
| Component | Vitest + React Testing Library | Each component in isolation; mock `useTheme`, mock health state |
| Integration | Vitest + RTL | `App` with mocked `fetch`; verify health state updates flow to `Header` summary |
| E2E | Playwright | Full render; theme toggle persists; service cards render; disabled card has no link |

**Key test cases (pre-implementation):**
1. `checkHealth` returns `ONLINE` for resolved fetch (even opaque)
2. `checkHealth` returns `UNREACHABLE` for aborted fetch (timeout)
3. `MetricCard` renders `—` and `WIRE TO PROMETHEUS TO ENABLE` when `wired: false`
4. `ServiceCard` renders as non-interactive when `enabled: false`
5. `StatusIndicator` has correct `aria-label` for each status value
6. `ThemeContext` reads from localStorage on init, writes on toggle
7. Header summary shows correct `X/Y ONLINE` count

---

## Security Notes

- No secrets in portal — all service URLs are static config, no auth tokens
- `nginx.conf` disables `server_tokens` (no version exposure)
- All external links use `rel="noopener noreferrer"`
- Health check fetch uses `AbortController` to enforce timeout — no hanging requests
- Simple Icons CDN dependency (CDN-loaded images) — service icons may not render in air-gapped environments; `alt` text is required and sufficient for accessibility

---

## Future Wiring Points

| Identifier | Location | What to do |
|---|---|---|
| `VAULT_STATUS_DETAIL` | `services.js` vault entry, `checkHealth` comments | When proxy tier available, replace health URL with same-origin proxy endpoint; update status resolver to read JSON body |
| `MetricsContext upgrade` | `App.jsx`, `metrics.js` | Extract `METRICS` array into a `MetricsContext` provider with polling; `MetricCard` requires no changes |
| `fourdogs URL` | `services.js` fourdogs entry | Set `url`, `healthCheck.url`, `enabled: true` when confirmed |
