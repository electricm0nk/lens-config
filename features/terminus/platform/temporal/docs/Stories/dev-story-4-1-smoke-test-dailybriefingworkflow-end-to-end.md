# Story 4.1: Smoke Test — DailyBriefingWorkflow End-to-End

Status: done

## Story

As a platform developer,
I want to execute a `DailyBriefingWorkflow` end-to-end through the deployed Temporal server and TypeScript worker and observe it complete successfully,
so that the Temporal platform is confirmed operational and the `terminus-agent-dailybriefing` initiative is unblocked to begin implementation stories.

## Acceptance Criteria

1. Using `tctl` or Temporal Web UI (`https://temporal-ui.trantor.internal`), trigger a `DailyBriefingWorkflow` execution on namespace `default`, task queue `terminus-platform`
2. Workflow execution appears in the Temporal Web UI with `Running` → `Completed` status
3. Workflow completion result is visible in the PostgreSQL visibility store (verified via Web UI search or `tctl workflow list`)
4. Worker logs show `DailyBriefingWorkflow` activity execution with expected pino structured output (fields: `timestamp`, `level`, `service`, `module`, `message`, `context`)
5. No errors in Temporal server `temporal-frontend` pod logs during workflow execution
6. Smoke test results documented in `docs/terminus/platform/temporal/smoke-test-results.md`: timestamp, workflow ID, result, screenshot or log excerpt
7. **N4 (dailybriefing sequencing):** Note added to `docs/terminus/platform/temporal/smoke-test-results.md`: "Temporal server + worker verified operational on [date]. terminus-agent-dailybriefing implementation stories may now proceed."

## Tasks / Subtasks

- [ ] Task 1: Pre-smoke-test verification (AC: all preconditions)
  - [ ] Confirm all Story 2.3 pods are Running: `kubectl get pods -n temporal`
  - [ ] Confirm worker pod is Running with real image (not bootstrap): `kubectl get pods -n temporal -l app=temporal-worker`
  - [ ] Confirm `https://temporal-ui.trantor.internal` is accessible
  - [ ] Verify Temporal namespace `default` exists: `kubectl exec -it temporal-admintools-<pod> -n temporal -- tctl namespace list`

- [ ] Task 2: Trigger DailyBriefingWorkflow execution (AC: 1, 2)
  - [ ] Option A — tctl:
    ```bash
    kubectl exec -it temporal-admintools-<pod> -n temporal -- \
      tctl workflow run \
        --taskqueue terminus-platform \
        --workflow_type DailyBriefingWorkflow \
        --execution_timeout 60
    ```
  - [ ] Option B — Web UI: Navigate to `https://temporal-ui.trantor.internal` → Start Workflow → fill in type and task queue
  - [ ] Note the Workflow ID from the trigger output
  - [ ] Monitor execution in Web UI — confirm `Running` → `Completed` transition

- [ ] Task 3: Verify visibility store (AC: 3)
  - [ ] In Temporal Web UI, search for the Workflow ID — confirm it appears in recent executions
  - [ ] Or: `kubectl exec -it temporal-admintools-<pod> -n temporal -- tctl workflow list` — confirm entry shows `Completed`

- [ ] Task 4: Verify worker logs (AC: 4)
  - [ ] `kubectl logs -n temporal -l app=temporal-worker --tail=50`
  - [ ] Confirm pino structured log output with fields: `timestamp`, `level`, `service`, `module`, `message`, `context`
  - [ ] Confirm `DailyBriefingWorkflow` activity execution logged

- [ ] Task 5: Verify frontend pod logs (AC: 5)
  - [ ] `kubectl logs -n temporal -l app=temporal -l component=frontend --tail=50`
  - [ ] Confirm no errors during the smoke test execution window

- [ ] Task 6: Document smoke test results (AC: 6, 7)
  - [ ] Create/update `docs/terminus/platform/temporal/smoke-test-results.md`:
    - Timestamp of execution (ISO 8601)
    - Workflow ID
    - Execution result (Completed)
    - Worker log excerpt showing activity execution
    - N4 gate note: "Temporal server + worker verified operational on [date]. terminus-agent-dailybriefing implementation stories may now proceed."
  - [ ] Commit: `docs(smoke-test): document DailyBriefingWorkflow smoke test results`
  - [ ] Commit to `terminus.platform` and/or update initiative config in control repo

## Dev Notes

### Target Repo Path
```
TargetProjects/terminus/platform/terminus.platform/    (smoke-test-results.md)
```

### Pre-Condition Check
All of the following MUST be true before triggering the smoke test:
- Story 2.3: Temporal server pods Running
- Story 3.1: Worker Deployment Running with real image (not bootstrap)
- Story 3.2: `DailyBriefingWorkflow` registered and polling
- Story 3.3: Real image built and deployed

### tctl Command Reference
```bash
# Find admintools pod
kubectl get pods -n temporal | grep admintools

# Run workflow
kubectl exec -it temporal-admintools-<hash> -n temporal -- \
  tctl workflow run \
    --taskqueue terminus-platform \
    --workflow_type DailyBriefingWorkflow \
    --execution_timeout 60
```

### Critical: Workflow Type Name
The workflow type registration name is **`DailyBriefingWorkflow`** (PascalCase). Using `dailyBriefingWorkflow` (camelCase) will result in `WorkflowNotFound` error. This was a known defect fixed in Story 3.2 — confirm the registered type name matches before triggering.

### N4 Gate
This story is the gate for `terminus-agent-dailybriefing`. The smoke test MUST pass and the gate note MUST be committed before the dailybriefing initiative can begin dev stories. After gate is PASSED, update the initiative config's `dailybriefing_gate` field from `OPEN` to `PASSED`.

### Initial Smoke Test Results
First execution details (from actual run, 2026-04-01):
- WorkflowId: `1aa43a33-1dae-4c7a-bc20-6f211df9f82c`
- Status: COMPLETED in 300ms
- Fix applied: renamed `dailyBriefingWorkflow` → `DailyBriefingWorkflow` (PascalCase) in worker registration (commit `44eb6e7` on `terminus.platform`)
- Gate verdict: PASSED

### Constitutional Override — Develop-First Integration
Implementation work lands on `develop` in `terminus.platform` (Terminus Art. 5 override active).

### Not In Scope
- Full activity implementations (terminus-agent-dailybriefing initiative)
- Load testing or non-functional verification
- Any platform changes — this story is observation and documentation only

## Pre-Implementation Constitution Gates

| Gate | Article | Result |
|------|---------|--------|
| Track declared | Org Art. 1 | ✅ PASS — `tech-change` |
| Architecture documented | Org Art. 3 | ✅ PASS — `docs/terminus/platform/temporal/architecture.md` |
| No confidential data | Org Art. 4 | ✅ PASS — smoke test only, no credentials persisted |
| Git discipline | Org Art. 5 | ✅ PASS — develop-first integration authorized by terminus Art. 5 |
| Security first | Org Art. 9 | ✅ PASS — no credentials involved in smoke test |
| Repo as source of truth | Org Art. 10 | ✅ PASS — all context from committed docs |
| Develop-first integration | Terminus Art. 5 | ✅ OVERRIDE ACTIVE — land implementation work on `develop` in `terminus.platform` |

## Dev Agent Record

### Agent Model Used

Claude Sonnet 4.6 (copilot)

### Debug Log References

### Completion Notes List

### File List
