# Story 4.5: Automated Rotation Hook Design — Forward Stub and Mechanism Analysis (AR-S9 LOW, FR24 Growth)

Status: ready-for-dev

## Story

As the architect,
I want a documented growth hook in the rotation playbook and a design analysis of two automated rotation trigger mechanisms in the runbook,
so that future implementers have a clear, opinionated starting point for automating secret rotation without requiring immediate implementation.

## Acceptance Criteria

1. **[Growth hook comment exists in day2-rotate-secrets.yml]** Given `ansible/playbooks/day2-rotate-secrets.yml` exists (Story 4.4), when I inspect it for the growth hook, then there is a comment block reading exactly: `# GROWTH HOOK: automated rotation trigger point` at a logical point in the playbook (e.g., before Step 2 where credential generation begins or at the end of the workflow).

2. **[Two candidate mechanisms documented in runbook]** Given `docs/runbooks/secret-rotation.md` contains a `## Growth: Automated Rotation` section (from Story 4.4 stub), when I read it, then it documents both:
   - **Mechanism A:** Vault Agent `auto_auth` with lease stanza — renewable token-based trigger
   - **Mechanism B:** Ansible AWX/cron-scheduled rotation — external scheduler approach

3. **[Both mechanisms include key trade-offs]** Given the `## Growth: Automated Rotation` section, when I read Mechanism A and Mechanism B descriptions, then each contains:
   - A 2–3 sentence technical description of how it would work
   - Pros (at least 2 bullet points)
   - Cons / constraints (at least 2 bullet points)

4. **[Recommendation is documented]** Given the trade-off analysis in the runbook, when I read the section conclusion, then a recommendation is stated — either choosing one mechanism over the other or providing clear criteria under which each is preferred (e.g., "Use Mechanism B (cron) for single-operator homelab; consider Mechanism A (Vault Agent) when a Vault Agent deployment is already established").

5. **[FR24 Growth reference is present]** Given the runbook section, when I read it, then it includes a reference to `FR24` and a note: "Automated rotation is a Growth feature — not required for initial deployment. Implement when rotation frequency requirements exceed manual capacity."

6. **[AR-S9 risk item is referenced and acknowledged]** Given the `## Growth: Automated Rotation` section, when I read the section header or opening paragraph, then it references `AR-S9` and acknowledges: "Design without implementation carries AR-S9 risk (LOW) — automated rotation is not built, only planned."

## Tasks / Subtasks

- [ ] **Task 1: Add growth hook comment to `day2-rotate-secrets.yml`** (AC: 1)
  - [ ] Edit `ansible/playbooks/day2-rotate-secrets.yml` to add a comment block after the `pre_tasks` section or before the credential generation step:
    ```yaml
    # GROWTH HOOK: automated rotation trigger point
    # FR24 (Growth): Replace manual invocation with an automated trigger mechanism.
    # See docs/runbooks/secret-rotation.md ## Growth: Automated Rotation for candidate designs.
    # AR-S9 (LOW): Automated rotation is not implemented — this hook marks the extension point.
    ```

- [ ] **Task 2: Expand `## Growth: Automated Rotation` section in `secret-rotation.md`** (AC: 2, 3, 4, 5, 6)
  - [ ] Replace or expand the stub section from Story 4.4 with the full analysis:

    ```markdown
    ## Growth: Automated Rotation

    > **AR-S9 (LOW):** Automated rotation is not implemented in this baseline.
    > FR24 is a Growth feature — not required for initial deployment.
    > Implement when rotation frequency requirements exceed manual capacity.

    Two candidate mechanisms are documented here for future implementers.

    ### Mechanism A: Vault Agent `auto_auth` with Lease Stanza

    Vault Agent can be configured with an `auto_auth` block that maintains a renewable token,
    and a `template` or `exec` stanza that triggers when a secret lease approaches renewal.
    The agent can be extended to invoke a rotation script when a monitored lease TTL drops below
    a threshold.

    **Pros:**
    - Native Vault integration — no external scheduler required
    - Respects Vault's lease model; trigger fires based on actual credential expiry
    - Audit trail in Vault logs captures agent-triggered operations

    **Cons:**
    - Requires Vault Agent to be deployed and maintained (additional operational surface)
    - KV v2 static secrets have no native leases — trigger must be time-based, not lease-based
    - More complex configuration; Vault Agent version must be kept in sync with Vault server

    ### Mechanism B: Ansible AWX / cron-scheduled Rotation

    Schedule `day2-rotate-secrets.yml` as a recurring cron job (or AWX/Rundeck job template)
    on the operator machine. The playbook is already idempotent per-invocation — triggering
    it on a weekly or monthly schedule achieves automated rotation at predictable cadence.

    **Pros:**
    - No new infrastructure required — leverages existing Ansible + cron setup
    - Simple to implement: `0 2 1 * * ansible-playbook day2-rotate-secrets.yml -e "target_path=..."`
    - Easy to audit: cron logs + git commit history capture all rotation events

    **Cons:**
    - Requires environment variables (VAULT_TOKEN, SOPS_AGE_KEY_FILE) to be available to cron
    - Time-based (not event-based) — credential expiry and rotation schedule may drift
    - AWX/Rundeck adds operational overhead; plain cron lacks secret injection integration

    ### Recommendation

    For a single-operator homelab environment (this baseline deployment), use **Mechanism B
    (cron-scheduled Ansible)**. It requires no new infrastructure and leverages the existing
    playbook. Configure cron to run monthly on the 1st (aligning with NFR3's 90-day ceiling):

    ```cron
    0 3 1 * * cd /path/to/terminus.infra && ansible-playbook ansible/playbooks/day2-rotate-secrets.yml -e "target_path=terminus/infra/postgres/app_role_password" >> /var/log/vault/rotation.log 2>&1
    ```

    Consider **Mechanism A (Vault Agent)** when:
    - Vault Agent is already deployed for other workloads (k3s, openclaw)
    - Dynamic secret backends (database, PKI) are introduced — these have native leases
    - Rotation frequency requirements increase to weekly or more frequent

    **FR24 reference:** Automated rotation is marked as Growth in the requirements. Implement
    only when manual rotation capacity is exceeded.
    ```

- [ ] **Task 3: Review and confirm both files** (AC: 1, 2)
  - [ ] Verify `day2-rotate-secrets.yml` contains the `# GROWTH HOOK:` comment (exact text)
  - [ ] Verify `secret-rotation.md` contains the `## Growth: Automated Rotation` section with both mechanisms, pros/cons, recommendation, FR24 reference, and AR-S9 acknowledgment

## Dev Notes

### Architecture Constraints

- **AR-S9 — LOW risk, design without implementation:** "No automated rotation mechanism is implemented — rotation relies on manual Ansible playbook invocation." This is an accepted risk for the initial deployment. The resolution path (Mechanism A or B from this story's analysis) must be documented.
  [Source: docs/terminus/infra/secrets/architecture.md#Vault Provisioning Decisions]

- **FR24 — Growth feature classification:** FR24 is explicitly marked as a Growth requirement in the requirements inventory. Growth features are acknowledged as valuable but not required for the initial production deployment. Marking FR24 as Growth and documenting the hook is the correct implementation.

- **No new code:** This story produces documentation artifacts only — a comment in an existing file and an expansion of a pre-existing section. There is no infrastructure to provision, no playbook to test operationally.

- **KV v2 static secrets and Vault Agent:** A common misunderstanding is that Vault Agent can monitor KV v2 secrets for "expiry" and trigger rotation. KV v2 secrets do NOT have built-in leases or TTLs — they are static. Vault Agent cannot natively trigger on KV v2 "expiry." The only way to automate KV v2 rotation is via external scheduling (Mechanism B) or by moving to dynamic secrets that do have leases (PKI, database backends). Document this clearly in the Mechanism A cons.

### Project Structure Notes

- Modifies `ansible/playbooks/day2-rotate-secrets.yml` (adds growth hook comment)
- Modifies `docs/runbooks/secret-rotation.md` (expands Growth section)

### References

- [Source: docs/terminus/infra/secrets/epics.md#Story 4.5]
- [Source: docs/terminus/infra/secrets/architecture.md#Vault Provisioning Decisions — AR-S9]
- [Source: docs/terminus/infra/secrets/architecture.md#Additional Requirements — FR24]

## Dev Agent Record

### Agent Model Used

claude-sonnet-4-5

### Debug Log References

### Completion Notes List

### File List
