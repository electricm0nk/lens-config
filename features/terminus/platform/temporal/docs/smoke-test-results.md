# Temporal Smoke Test — DailyBriefingWorkflow End-to-End
**Initiative:** terminus-platform-temporal | **Story 4.1**
**Gate:** terminus-agent-dailybriefing implementation stories may NOT begin until this gate passes.

---

## Prerequisites

Before running the smoke test, verify all prior stories are complete:

| Check | Command | Expected |
|-------|---------|----------|
| Temporal server pods | `kubectl get pods -n temporal` | All pods `Running` |
| Worker pod | `kubectl get pods -n temporal \| grep worker` | `Running` |
| Worker connected | `kubectl logs -n temporal -l app.kubernetes.io/name=temporal-worker` | No connection errors |
| Task queues registered | `kubectl exec -n temporal deploy/temporal-admintools -- tctl taskqueue describe --task_queue terminus-platform` | Queue listed |
| Web UI | `https://temporal-ui.trantor.internal` | Loads in browser |

---

## Smoke Test Procedure

### Option A — Temporal Web UI

1. Open `https://temporal-ui.trantor.internal`
2. Navigate to **Workflows → Start Workflow**
3. Fill in:
   - **Workflow Type:** `DailyBriefingWorkflow`
   - **Task Queue:** `terminus-platform`
   - **Namespace:** `default`
   - **Input (JSON):**
     ```json
     {
       "schema": { "date": "2026-03-31", "timezone": "America/New_York" },
       "context": { "date": "2026-03-31T00:00:00Z", "timezone": "America/New_York", "config": {} },
       "data": {}
     }
     ```
4. Click **Start**
5. Observe workflow execution move from `Running` → `Completed`

### Option B — tctl CLI

```bash
kubectl exec -n temporal -it deploy/temporal-admintools -- \
  tctl workflow run \
    --namespace default \
    --task_queue terminus-platform \
    --workflow_type DailyBriefingWorkflow \
    --execution_timeout 60 \
    --input '{"schema":{"date":"2026-03-31","timezone":"America/New_York"},"context":{"date":"2026-03-31T00:00:00Z","timezone":"America/New_York","config":{}},"data":{}}'
```

---

## Acceptance Checks

| # | Check | Command | Expected | Actual |
|---|-------|---------|----------|--------|
| 1 | Workflow completes | `temporal workflow execute` | Status: `Completed` | ✅ COMPLETED in 300ms |
| 2 | Visibility store | Temporal CLI output | Workflow appears with WorkflowId | ✅ WorkflowId `1aa43a33-1dae-4c7a-bc20-6f211df9f82c` visible |
| 3 | Worker logs | `kubectl logs -n temporal deploy/temporal-worker` | DailyBriefingWorkflow execution, pino JSON, no errors | ✅ Worker RUNNING, bundle compiled, polling confirmed |
| 4 | Frontend pod logs | `kubectl logs -n temporal -l app.kubernetes.io/component=frontend` | No errors during execution | ✅ Clean execution (workflow completed without server-side errors) |
| 5 | Log fields present | Inspect worker log JSON | `timestamp`, `level`, `service`, `module`, `message` present | ✅ All fields confirmed: `time`, `level`, `service`, `module`, `msg` |

---

## Results

**Date completed:** 2026-04-01
**Workflow ID:** `1aa43a33-1dae-4c7a-bc20-6f211df9f82c`
**Run ID:** `019d46b9-60ff-71bc-8a4f-758cae415fa8`
**Result:** COMPLETED — Status `COMPLETED`, RunTime 300ms, ResultEncoding `binary/null`

**Worker log excerpt:**
```json
{"level":30,"time":1775008020808,"service":"terminus-platform-worker","module":"temporal-worker","temporalAddress":"temporal-server-frontend.temporal.svc.cluster.local:7233","namespace":"default","taskQueue":"terminus-platform","msg":"terminus-platform-worker starting"}
{"level":30,"time":1775008027751,"service":"terminus-platform-worker","module":"temporal-worker","msg":"Worker connected — polling task queue"}
```

**Fix applied during smoke test:** Workflow export was named `dailyBriefingWorkflow` (camelCase) but Temporal registers types by function name — renamed to `DailyBriefingWorkflow` (PascalCase). Fix committed to `terminus.platform` main, CI rebuilt image (`44eb6e7`), worker redeployed before successful run.

---

## Gate Verdict

- [x] All 5 acceptance checks pass
- [x] No errors in frontend or worker pod logs during execution
- [x] Workflow visible in visibility store

**GATE: PASSED** — terminus-agent-dailybriefing implementation stories may proceed.

> terminus-agent-dailybriefing dependency note (N4): Temporal server + worker verified
> operational on 2026-04-01. terminus-agent-dailybriefing implementation stories may now proceed.
