# Story 4.2: useHealthCheck Hook

Status: done

## Story

As a developer,
I want a `useHealthCheck(url, enabled)` hook that polls a health endpoint,
so that health check logic is encapsulated and testable independently of components.

## Acceptance Criteria

1. `useHealthCheck(url, true)` → fetch resolves (opaque or any non-error) → returns `STATUS.ONLINE`
2. Fetch throws `TypeError` or `AbortController` fires after 5000ms → returns `STATUS.UNREACHABLE`
3. `useHealthCheck(url, false)` → returns `STATUS.NO_CHECK` immediately, never calls `fetch`
4. Hook uses `AbortController` to enforce 5s timeout
5. `AbortController` and interval are cleaned up on component unmount
6. Unit tests mock `fetch` to cover all 4 scenarios

## Tasks / Subtasks

- [ ] Task 1: Write failing unit tests FIRST (TDD) (AC: #6)
  - [ ] `src/__tests__/hooks/useHealthCheck.test.js`:
    - [ ] Mock `fetch` with opaque response (mode: 'no-cors', `response.type === 'opaque'`) → `STATUS.ONLINE`
    - [ ] Mock `fetch` throwing `TypeError('Failed to fetch')` → `STATUS.UNREACHABLE`
    - [ ] Mock `AbortController.abort()` being called → `STATUS.UNREACHABLE`
    - [ ] `enabled: false` → `STATUS.NO_CHECK`, `fetch` never called
    - [ ] Unmount during pending fetch → no state update after unmount (no `act()` warning)
  - [ ] Use `vi.stubGlobal('fetch', vi.fn(...))` for mock
  - [ ] Run tests — confirm FAIL (red)
- [ ] Task 2: Create `src/hooks/useHealthCheck.js` (AC: #1–#5)
  - [ ] Initial state: `STATUS.CHECKING` (enabled) or `STATUS.NO_CHECK` (disabled)
  - [ ] `enabled === false` → return `STATUS.NO_CHECK`, no fetch, no interval
  - [ ] Single check function:
    ```js
    const controller = new AbortController();
    const timeoutId = setTimeout(() => controller.abort(), 5000);
    try {
      await fetch(url, { signal: controller.signal, mode: 'no-cors' });
      setStatus(STATUS.ONLINE);
    } catch {
      setStatus(STATUS.UNREACHABLE);
    } finally {
      clearTimeout(timeoutId);
    }
    ```
  - [ ] Run check on mount, then set interval every 30s
  - [ ] Cleanup: `clearInterval` + `AbortController.abort()` in `useEffect` return
- [ ] Task 3: Run all tests — green (AC: #6)

## Dev Notes

**Target repo:** `TargetProjects/terminus/portal/terminus-portal`
**Depends on:** Story 1.1 — `STATUS` enum exists; Story 4.1 — `StatusIndicator` exists (though hook is independent).
**Key constraint (TR5):** Vault health check is no-cors — response will be opaque (`response.type === 'opaque'`). An opaque response does NOT throw — it resolves successfully. So `fetch` resolving to an opaque response → `STATUS.ONLINE`. Only a thrown error → `STATUS.UNREACHABLE`.
**AbortController timeout pattern (TR4):** Use `setTimeout` calling `controller.abort()` after 5000ms. In the catch block, check `if (e.name === 'AbortError')` to still set `STATUS.UNREACHABLE`.
**Cleanup guard:** Set a `mounted` flag in `useEffect` to prevent `setState` after unmount:
```js
let mounted = true;
// ... in async callback: if (mounted) setStatus(...)
return () => { mounted = false; clearInterval(id); controller.abort(); };
```
**Note:** `useHealthCheck` drives a single service. App-level polling loop (Story 4.3) coordinates all services via `Promise.allSettled`.

### Project Structure Notes

New:
```
src/
├── hooks/
│   └── useHealthCheck.js
└── __tests__/hooks/
    └── useHealthCheck.test.js
```

### References

- FR2 (health polling, 30s interval, 5s timeout, no-cors): `docs/terminus/portal/portal2/epics.md`
- TR4 (AbortController), TR5 (Vault binary — opaque response): `docs/terminus/portal/portal2/epics.md`
- Architecture: `docs/terminus/portal/portal2/architecture.md` — Section "Health Polling"
- Story 4.3 (App-level polling — uses results from this hook or equivalent logic)

## Dev Agent Record

### Agent Model Used

### Debug Log References

### Completion Notes List

### File List
