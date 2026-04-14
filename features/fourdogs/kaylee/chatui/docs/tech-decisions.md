# Tech Decisions Log — fourdogs-kaylee-chatui

**Initiative:** fourdogs-kaylee-chatui
**Track:** tech-change
**Phase:** TechPlan
**Date:** 2026-04-13

---

## TDL-001 — Vanilla HTML/JS, zero build toolchain

| | |
|---|---|
| **Status** | Accepted |
| **Decision** | Single `index.html` with inline JS and CSS. No bundler, no npm, no TypeScript. |
| **Rationale** | This is a developer test/debug tool, not a production UI. A zero-dependency approach means any developer can open, read, and modify it directly. There is no maintenance burden for node_modules, build scripts, or version upgrades. |
| **Alternatives Considered** | React/Vite (rejected: overkill + build friction), HTMX (rejected: unfamiliar dep, no real benefit here), Alpine.js (rejected: added complexity without need) |
| **Consequences** | No type safety, no component reuse, no HMR. Acceptable for a dev tool. If this grows into a real UI, revisit. |

---

## TDL-002 — FastAPI StaticFiles mount (same container, same port)

| | |
|---|---|
| **Status** | Accepted |
| **Decision** | Mount `ui/` directory via `app.mount("/ui", StaticFiles(...))` in `main.py`. |
| **Rationale** | No new containers, no new ports, no nginx config, no CORS headers. The UI and API share the same origin (`localhost:8000`), making direct `fetch()` calls trivial. Deployment changes are minimal. |
| **Alternatives Considered** | Separate nginx container (rejected: deployment complexity), separate dev server with proxy (rejected: adds local setup friction), vite dev server (rejected: requires npm ecosystem) |
| **Consequences** | `StaticFiles` mount is added to `main.py`. Should be gated behind `KAYLEE_DEV_UI_ENABLED=true` env var to prevent the mount from appearing in production images. |

---

## TDL-003 — Direct REST to Kaylee (no adapter layer)

| | |
|---|---|
| **Status** | Accepted |
| **Decision** | Browser calls `/sessions` and `/sessions/{id}/messages` directly. |
| **Rationale** | Since UI and API share the same origin, there is no CORS problem and no proxy is needed. This also means the test UI exercises the exact same code path that fourdogs-central uses when proxying Kaylee — making it a genuine integration test tool. |
| **Alternatives Considered** | Proxy through fourdogs-central (rejected: defeats the purpose of testing Kaylee directly), custom WebSocket bridge (rejected: unnecessary complexity) |
| **Consequences** | The UI will only work when the kaylee-agent service is running. Not an issue — this is a dev tool. |

---

## TDL-004 — Future-proof streaming display (sync now, SSE-ready)

| | |
|---|---|
| **Status** | Accepted |
| **Decision** | Write the UI with two rendering paths: sync (current) and SSE stream (future). A `KAYLEE_STREAM_ENABLED` JS constant controls which path is active. Default: `false`. |
| **Rationale** | `POST /sessions/{id}/stream` is a 501 stub today. When it is implemented, the UI should be able to switch without an architectural rewrite. Writing the SSE path as a thin layer over the display logic costs ~20 lines and avoids a future migration. |
| **Alternatives Considered** | Sync-only, rewrite when streaming is ready (rejected: creates future rework), streaming-only (rejected: blocks on 501 implementation) |
| **Consequences** | Slightly more code than absolutely necessary. The `EventSource` path is dead code until streaming is implemented. |

---

## TDL-005 — Session created on page load, no persistence

| | |
|---|---|
| **Status** | Accepted |
| **Decision** | `POST /sessions` fires on page load. `session_id` is held in a JS variable. History is loaded via `GET /sessions/{id}/messages` on load. No `localStorage`, no cookies. |
| **Rationale** | This is a stateless test tool. Persisting session IDs adds complexity with no benefit — each test run is typically a fresh conversation. Session TTL in Kaylee's DB handles cleanup. |
| **Alternatives Considered** | `localStorage` persistence (rejected: stale session IDs cause confusing errors), cookie-based session (rejected: unnecessary overhead) |
| **Consequences** | Refreshing the page starts a new session. Expected behaviour for this use case. |

---

## TDL-006 — Production exclusion via env var gate

| | |
|---|---|
| **Status** | Accepted |
| **Decision** | `StaticFiles` mount is conditional on `KAYLEE_DEV_UI_ENABLED=true`. Default is `false`. |
| **Rationale** | The chat UI is a dev tool and should not be exposed on production deployments. A env var gate is the simplest way to control this without maintaining separate Dockerfiles. The same image can be used in all environments. |
| **Alternatives Considered** | `.dockerignore` exclusion (rejected: requires separate image builds), separate `dev` image tag (rejected: complicates CI/CD pipeline), always-on (rejected: surface area / security risk in prod) |
| **Consequences** | Dev deployments must set `KAYLEE_DEV_UI_ENABLED=true`. This should be in the docker-compose dev config but NOT in the k8s prod values. |
