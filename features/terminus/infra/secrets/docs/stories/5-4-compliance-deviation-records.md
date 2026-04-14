# Story 5.4: Compliance Deviation Records — SoD and Dual-Control Deviation Documentation (FR21, NFR14)

Status: ready-for-dev

## Story

As the compliance owner,
I want formal deviation records documenting the separation-of-duties (SoD) and dual-control deviations inherent in this single-operator architecture,
so that the system's compliance posture is explicit, the deviations are justified with compensating controls, and any future auditor can understand the risk acceptance.

## Acceptance Criteria

1. **[SoD deviation record exists]** Given `docs/deviations/sod-deviation.md` is created, when I read it, then it contains:
   - The deviation: "single operator fulfills all roles (infrastructure provisioner, secret author, secret consumer, auditor)"
   - Reference to the standard deviated from (e.g., NIST SP 800-53 AC-5: Separation of Duties)
   - Compensating controls (at least 3 listed)
   - Risk acceptance statement with explicit acknowledgment by the operator

2. **[SoD compensating controls are specific]** Given the SoD deviation record's compensating controls section, when I read it, then it contains all three of:
   - "Policy-driven access control: Vault policies enforce least-privilege even for the operator identity (opentofu AppRole cannot read agent/openclaw secrets)"
   - "All operations logged: Vault audit log captures every operation regardless of performer identity"
   - "Code-reviewed automation: OpenTofu and Ansible are the primary provisioning mechanism; manual actions are minimized and logged"

3. **[Dual-control deviation record exists]** Given `docs/deviations/dual-control-deviation.md` is created, when I read it, then it contains:
   - The deviation: "1-of-1 Shamir unseal key departs from NIST SP 800-57 multi-custodian key ceremony"
   - Reference to NIST SP 800-57 or equivalent standard
   - Compensating controls (at least 2 listed)
   - Reference to the AD (Architecture Decision) that made this choice: "D3 — 1-of-1 Shamir for homelab simplicity"

4. **[Dual-control compensating controls reference offline custody]** Given the dual-control deviation record, when I read the compensating controls, then it includes: "Offline custody: unseal key is encrypted with age and stored in `sops/bootstrapcreds.sops.yaml` — accessible only with the SOPS age private key."

5. **[Both deviation records reference the architecture document]** Given both deviation documents, when I inspect them, then each contains a `## References` section that links to `docs/terminus/infra/secrets/architecture.md` (or relative path equivalent) for the architecture decisions that drive the deviations.

6. **[FR21 compliance — deviations are documented and justified]** Given both deviation documents are complete, when I evaluate them against FR21 ("Compliance deviations are documented with justification and compensating controls"), then both include: the regulatory/standards reference, the deviation description, the justification, the compensating controls, and a risk acceptance statement.

## Tasks / Subtasks

- [ ] **Task 1: Create `docs/deviations/` directory structure** (AC: 1, 3)
  - [ ] Create the directory: `docs/deviations/`
  - [ ] Note: this directory will hold all formal deviation records for the terminus-infra-secrets project

- [ ] **Task 2: Create `docs/deviations/sod-deviation.md`** (AC: 1, 2, 5, 6)
  - [ ] Create the file with content:

    ```markdown
    # Compliance Deviation Record — Separation of Duties (SoD)

    **Document ID:** DEV-001
    **Standard Deviated From:** NIST SP 800-53 Rev 5 — AC-5 (Separation of Duties)
    **Risk Level:** MEDIUM
    **Status:** Accepted — Compensating Controls in Place
    **Date:** <date>
    **Owner:** <operator name>

    ## Deviation Description

    This deployment uses a single-operator model in which one person fulfills all operational roles:
    - Infrastructure provisioner (provisions Vault VM via OpenTofu)
    - Secret author (writes secrets to Vault via SOPS/OpenTofu pipeline)
    - Secret consumer (reads secrets for operational tasks)
    - Auditor (reviews audit logs for compliance)

    NIST SP 800-53 AC-5 requires that personnel be assigned different roles to prevent any one
    individual from having unconstrained access to the full system. A single-operator homelab
    architecture cannot satisfy this control natively.

    ## Justification

    This is a homelab/development infrastructure deployment for learning and experimentation.
    The system manages no production workloads with external SLA obligations. The single-operator
    model is intentional for this scope and phase of the project.

    ## Compensating Controls

    The following compensating controls reduce risk from the SoD deviation:

    1. **Policy-driven access control:** Vault policies enforce least-privilege even within the
       single-operator context. Each workload identity (opentofu, k3s, openclaw) is scoped to
       its minimum required paths. The operator identity does not use AppRole credentials —
       it uses higher-privilege tokens only for administrative tasks.

    2. **All operations logged:** The Vault audit log captures every operation performed,
       regardless of initiator. Manual and automated operations are equally visible in
       `/var/log/vault/audit.log`. This provides after-the-fact detectability for unauthorized
       or erroneous operations.

    3. **Code-reviewed automation:** OpenTofu and Ansible are the primary provisioning
       mechanisms. Manual operations are minimized and documented. All infrastructure changes
       are committed to version control and visible in git history. This creates a lightweight
       peer-review path (even if only one operator, future reviewers can audit git history).

    4. **Break-glass audit trail:** Emergency root access is documented in `docs/ops-log.md`
       per FR31/FR32. Each break-glass session has a mandatory ops-log entry.

    ## Risk Acceptance Statement

    I, the system operator, acknowledge that this deployment deviates from NIST SP 800-53 AC-5
    (Separation of Duties). I accept this deviation for the current homelab/development scope,
    with the understanding that:
    - Compensating controls listed above are in place
    - This deviation will be reviewed if the system scope expands to production workloads
    - Any external audit of this system may flag this as a finding

    Accepted by: <operator name>
    Date: <date>

    ## References

    - NIST SP 800-53 Rev 5 — AC-5: https://csrc.nist.gov/publications/detail/sp/800-53/rev-5/final
    - Architecture decisions: `docs/terminus/infra/secrets/architecture.md`
    - Related deviation: `docs/deviations/dual-control-deviation.md`
    ```

- [ ] **Task 3: Create `docs/deviations/dual-control-deviation.md`** (AC: 3, 4, 5, 6)
  - [ ] Create the file with content:

    ```markdown
    # Compliance Deviation Record — Dual-Control Key Management

    **Document ID:** DEV-002
    **Standard Deviated From:** NIST SP 800-57 Part 1 Rev 5 — Key Management (Section 8.2.3: Cryptographic Key Management)
    **Risk Level:** MEDIUM (AR-S5)
    **Status:** Accepted — Compensating Controls in Place
    **Date:** <date>
    **Owner:** <operator name>

    ## Deviation Description

    NIST SP 800-57 recommends multi-custodian key ceremonies for cryptographic key material
    (e.g., requiring 2 of N custodians to reconstruct a key). A production Vault deployment
    typically uses Shamir Secret Sharing with N≥3 and a quorum (e.g., 3-of-5) to require
    multiple parties to unseal Vault.

    This deployment uses a **1-of-1 Shamir unseal key** (architecture decision D3): a single
    unseal key is generated at Vault initialization. One operator with the single key can unseal
    Vault without any other party's involvement.

    ## Architecture Decision Reference

    **D3 — 1-of-1 Shamir for homelab simplicity:**
    "Vault initialized with 1-of-1 Shamir; unseal key stored in SOPS. Single operator model
    does not justify key ceremony overhead."
    Source: `docs/terminus/infra/secrets/architecture.md#Vault Provisioning Decisions — D3`

    ## Justification

    For a single-operator homelab environment, a multi-custodian key ceremony would require
    designating multiple trusted key holder parties who do not exist in this context. The
    1-of-1 Shamir key simplifies operations while maintaining encryption of the unseal key
    at rest via SOPS+age.

    ## Compensating Controls

    1. **Offline custody:** The 1-of-1 Shamir unseal key is age-encrypted and stored in
       `sops/bootstrapcreds.sops.yaml`. It is accessible only to the holder of the SOPS age
       private key (`SOPS_AGE_KEY_FILE`). Physical access to the unseal key requires both the
       repository and the age private key.

    2. **Mandatory ops-logging for unsealing:** Any unseal operation performed outside of
       automated workloads (i.e., break-glass) must be recorded in `docs/ops-log.md` per FR31.
       This provides accountability for unseal events.

    3. **Audit log captures all post-unseal operations:** Once Vault is unsealed, every
       operation is captured in the Vault audit log at `/var/log/vault/audit.log`. Unauthorized
       use of the unseal key would be detectable through audit log analysis.

    ## Risk Acceptance Statement

    I, the system operator, acknowledge that this deployment deviates from NIST SP 800-57
    multi-custodian key management recommendations. I accept this deviation for the current
    homelab/development scope with the compensating controls listed above in place.

    Accepted by: <operator name>
    Date: <date>

    ## References

    - NIST SP 800-57 Part 1 Rev 5: https://csrc.nist.gov/publications/detail/sp/800-57-pt-1/rev-5/final
    - Architecture decision D3: `docs/terminus/infra/secrets/architecture.md`
    - Related deviation: `docs/deviations/sod-deviation.md`
    - Break-glass procedure: `docs/runbooks/break-glass.md`
    - AR-S5: AppRole Vault Integration risk item in architecture.md
    ```

- [ ] **Task 4: Personalize both deviation records** (AC: 1, 3, 6)
  - [ ] Replace `<operator name>` and `<date>` placeholders in both documents with actual values
  - [ ] Review the risk acceptance statements — they must genuinely reflect the operator's understanding
  - [ ] Confirm both documents contain all required fields per AC6 (standards reference, deviation, justification, compensating controls, acceptance)

## Dev Notes

### Architecture Constraints

- **FR21 — Documented compliance deviations:** "Deviations from security standards documented with justification and compensating controls." The two deviation documents in this story are the compliance artifacts for FR21. They must be production-quality documents, not placeholder stubs.

- **Deviation records are immutable once committed:** Like ops-log.md entries, deviation records should not be amended. If the risk level or compensating controls change, create a new version of the document with an updated date and status.

- **SoD is an inherent property of this architecture:** The single-operator model is not a temporary gap — it is a deliberate design decision for the homelab context. The deviation record normalizes this rather than treating it as a defect to be fixed.

- **Reference document IDs:** DEV-001 (SoD) and DEV-002 (Dual-Control) should be cross-referenced from the architecture document's AR-S5 entry and from the break-glass runbook. Consider adding a note to `architecture.md`: "Full deviation records: docs/deviations/sod-deviation.md (DEV-001), docs/deviations/dual-control-deviation.md (DEV-002)."

### Project Structure Notes

- Creates `docs/deviations/` (new directory)
- Creates `docs/deviations/sod-deviation.md` (new file)
- Creates `docs/deviations/dual-control-deviation.md` (new file)

### References

- [Source: docs/terminus/infra/secrets/epics.md#Story 5.4]
- [Source: docs/terminus/infra/secrets/architecture.md#Additional Requirements — FR21]
- [Source: docs/terminus/infra/secrets/architecture.md#Vault Provisioning Decisions — D3/AR-S5]

## Dev Agent Record

### Agent Model Used

claude-sonnet-4-5

### Debug Log References

### Completion Notes List

### File List
