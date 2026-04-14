# Story 4.3: App-Level Polling Loop and ServiceCard Status Integration

Status: done

## Story

As a user,
I want each service card to display a live health status that updates every 30 seconds,
so that I can see the current reachability of all services at a glance.

## Acceptance Criteria

1. `checkHealth` is called for each service with `healthCheck.enabled: true` every 30s
2. Each `ServiceCard` receives the resolved `status` prop and renders the correct `StatusIndicator`
3. Services with `healthCheck.enabled: false` always show `STATUS.NO_CHECK`, never triggering fetch
4. `Promise.allSettled` prevents a single service failure from blocking others
5. Integration tests mock `fetch` to return success for some, error for others — verify correct status per card

## Tasks / Subtasks

- [ ] Task 1: Write failing integration tests FIRST (TDD) (AC: #5)
  - [ ] `src/__tests__/integration/polling.test.jsx`:
    - [ ] Mock `fetch`: ArgoCD → resolves opaque, Vault → throws, others → resolves
    - [ ] Render `App` (or wrapper with SERVICES + ServiceGrid)
    - [ ] Wait for poll to complete (use `waitFor` from RTL)
    - [ ] ArgoCD card shows `STATUS.ONLINE`, Vault card shows `STATUS.UNREACHABLE`
    - [ ] Fourdogs card shows `STATUS.NO_CHECK` (never calls fetch)
  - [ ] Run tests — confirm FAIL (red)
- [ ] Task 2: Implement polling state in `App.jsx` (AC: #1, #3, #4)
  - [ ] `const [statusMap, setStatusMap] = useState({})` — keyed by service id
  - [ ] `const [isPolling, setIsPolling] = useState(false)`
  - [ ] `const [lastChecked, setLastChecked] = useState(null)`
  - [ ] `checkAllServices` async function:
    ```js
    setIsPolling(true);
    const results = await Promise.allSettled(
      SERVICES.filter(s => s.healthCheck.enabled).map(s => checkOne(s))
    );
    // update statusMap from results
    setLastChecked(new Date());
    setIsPolling(false);
    ```
  - [ ] `checkOne(service)`:
    ```js
    const controller = new AbortController();
    const t = setTimeout(() => controller.abort(), 5000);
    try {
      await fetch(service.healthCheck.url, { signal: controller.signal, mode: 'no-cors' });
      return { id: service.id, status: STATUS.ONLINE };
    } catch {
      return { id: service.id, status: STATUS.UNREACHABLE };
    } finally { clearTimeout(t); }
    ```
  - [ ] Services with `healthCheck.enabled: false` → set `STATUS.NO_CHECK` in initial statusMap
  - [ ] `useEffect`: call `checkAllServices()` on mount, then `setInterval(checkAllServices, 30000)`
  - [ ] Cleanup: `clearInterval` on unmount
- [ ] Task 3: Wire `statusMap` to `ServiceGrid` → `ServiceCard` (AC: #2)
  - [ ] `<ServiceGrid services={SERVICES} statusMap={statusMap} />`
  - [ ] In `ServiceGrid`: pass `status={statusMap[service.id] || STATUS.CHECKING}` to each `ServiceCard`
  - [ ] In `ServiceCard`: pass `status` to `StatusIndicator`
- [ ] Task 4: Run all tests — green (AC: #5)

## Dev Notes

**Target repo:** `TargetProjects/terminus/portal/terminus-portal`
**Depends on:** Story 4.2 — health check logic pattern; Story 3.2 — ServiceGrid accepts statusMap; Story 4.1 — StatusIndicator accepts status.
**Architecture note:** The `useHealthCheck` hook from Story 4.2 encapsulates per-service logic. For the App-level polling loop, you MAY either: (a) use `useHealthCheck` per service, or (b) implement a centralized `checkAllServices` in App with `Promise.allSettled`. Option (b) is preferred for coordinated polling (all services polled together, single lastChecked timestamp).
**Initial state:** All enabled services start as `STATUS.CHECKING` (poll runs immediately on mount). Disabled services start as `STATUS.NO_CHECK`.
**Promise.allSettled (TR4):** Ensures partial failures don't block the statusMap update. Each settled result is `{ status: 'fulfilled', value: { id, status } }` or `{ status: 'rejected', reason: ... }`. Handle rejections as `STATUS.UNREACHABLE`.

### Project Structure Notes

Modified:
- `src/App.jsx` (add polling state, checkAllServices, wire statusMap to ServiceGrid)
- `src/components/ServiceGrid.jsx` (accept statusMap prop, pass to ServiceCard)
- `src/components/ServiceCard.jsx` (pass status to StatusIndicator)

New tests:
```
src/
└── __tests__/integration/
    └── polling.test.jsx
```

### References

- FR2 (health polling, 30s, 5s timeout, Promise.allSettled): `docs/terminus/portal/portal2/epics.md`
- FR3 (header lastChecked comes from this state — wired in Story 5.2): `docs/terminus/portal/portal2/epics.md`
- TR4 (AbortController), TR5 (Vault binary): `docs/terminus/portal/portal2/epics.md`
- Architecture: `docs/terminus/portal/portal2/architecture.md` — Section "Health Polling"
- Story 5.2 (passes `onlineCount`, `totalCount`, `lastChecked` to Header from same App state)

## Dev Agent Record

### Agent Model Used

### Debug Log References

### Completion Notes List

### File List
