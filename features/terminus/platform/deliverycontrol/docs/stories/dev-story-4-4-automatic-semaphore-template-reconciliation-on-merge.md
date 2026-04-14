# Story 4.4: Automatic Semaphore Template Reconciliation On Merge

Status: done

## Story

As a platform operator,
I want Semaphore project configuration to reconcile automatically after `terminus.infra` merges,
so that onboarding a new release-managed service does not require a manual OpenTofu apply before Temporal can trigger the expected Semaphore tasks.

## Acceptance Criteria

1. Merges that change `tofu/environments/semaphore` trigger a GitHub Actions workflow on the internal self-hosted runner that runs `tofu init`, `tofu plan`, and `tofu apply` for the Semaphore environment
2. The Semaphore OpenTofu environment uses durable remote state suitable for unattended repeated apply instead of runner-local ephemeral state
3. The reconcile workflow sources live Semaphore API access from internal automation prerequisites rather than committed git values and fails clearly when Vault, Consul, or the API token is unavailable
4. Workflow execution is serialized so concurrent `terminus.infra` merges cannot race the Semaphore project state
5. The docs and implementation preserve the contract that PR merge is the normal release trigger and manual OpenTofu apply is no longer required for new Semaphore templates to become live

## Tasks / Subtasks

- [x] Add a path-scoped `terminus.infra` GitHub Actions workflow that reconciles `tofu/environments/semaphore` on merge to `main` and on manual dispatch (AC: 1, 4)
- [x] Move the Semaphore OpenTofu environment off the local backend onto a durable backend suitable for automation (AC: 2)
- [x] Retrieve the Semaphore API token from Vault at workflow runtime and validate internal automation prerequisites before plan/apply (AC: 3)
- [x] Update release-control docs to reflect that Semaphore template registration is now part of the normal merge-driven automation path (AC: 5)

## Dev Notes

### Root Cause

Temporal correctly assumes Semaphore templates already exist and only triggers runtime tasks by name. The actual gap was in `terminus.infra`: there was no workflow applying the Semaphore OpenTofu environment after merge, so new template declarations remained source-only until an operator performed a manual apply.

### Implementation Shape

Keep the ownership boundary intact:

- `terminus.infra` defines and reconciles Semaphore project objects
- Temporal triggers existing Semaphore templates during release execution
- GitHub Actions on the internal runner performs the OpenTofu reconcile after merge

### References

- [docs/terminus/platform/deliverycontrol/epics.md](docs/terminus/platform/deliverycontrol/epics.md)
- [docs/cicd.md](docs/cicd.md)
- [docs/terminus/architecture.md](docs/terminus/architecture.md)
- [TargetProjects/terminus/platform/terminus.platform/internal/activities/release.go](TargetProjects/terminus/platform/terminus.platform/internal/activities/release.go)

## Dev Agent Record

### Agent Model Used

GPT-5.4

### Completion Notes List

- Added merge-driven reconcile workflow for `tofu/environments/semaphore`
- Restored durable Consul backend usage for the Semaphore OpenTofu environment
- Wired runtime retrieval of `SEMAPHORE_API_TOKEN` from Vault before `tofu plan/apply`
- Updated release docs so template registration is treated as part of the normal automation path

### File List

- docs/terminus/platform/deliverycontrol/stories.md
- docs/terminus/platform/deliverycontrol/epics.md
- docs/terminus/platform/deliverycontrol/stories/sprint-status.yaml
- docs/terminus/platform/deliverycontrol/stories/dev-story-4-4-automatic-semaphore-template-reconciliation-on-merge.md