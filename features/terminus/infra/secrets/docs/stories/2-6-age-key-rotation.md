# Story 2.6: Age Key Rotation (FR25)

Status: ready-for-dev

## Story

As the operator,
I want a documented, tested age key rotation procedure that re-encrypts all SOPS-managed files with a new age key,
so that key rotation is a structured, low-risk operation that leaves no files encrypted to the old key.

## Acceptance Criteria

1. **[Rotation procedure documented]** Given `docs/runbooks/age-key-custody.md`, when I read the age key rotation procedure, then it covers: generate new age key B, update `sops/.sops.yaml` recipient, run `sops updatekeys` on every SOPS file, verify decrypt with key B, commit all re-encrypted files, document old key destruction.

2. **[Old key decryption fails after rotation]** Given all SOPS files re-encrypted with age key B, when I run `sops -d sops/bootstrapcreds.sops.yaml` with age key A (old key), then decryption fails with a key-not-found or recipient-mismatch error — key A can no longer decrypt any SOPS file.

3. **[New key decryption succeeds]** Given the re-encrypted files with age key B, when I run `sops -d sops/bootstrapcreds.sops.yaml` with age key B, then decryption succeeds and values match the pre-rotation content.

4. **[sops/.sops.yaml updated]** Given the rotation procedure is complete, when I inspect `sops/.sops.yaml`, then the age recipient entry shows the new public key B (not the old key A).

5. **[All SOPS files re-encrypted]** Given the rotation is complete, when I grep for the old public key in any committed file, then zero matches are found — no file still references the old key.

## Tasks / Subtasks

- [ ] **Task 1: Write rotation procedure in `docs/runbooks/age-key-custody.md`** (AC: 1)
  - [ ] Add `## Age Key Rotation Procedure` section:
    ```
    Step 1: Generate new age key B
      age-keygen -o /tmp/new-key-B.txt
      # Note: public key B is on the first line (starts with "# public key:")
      # Private key B is the second line (starts with "AGE-SECRET-KEY-")

    Step 2: Update sops/.sops.yaml
      # Replace the 'age:' recipient value with public key B
      
    Step 3: Re-encrypt all SOPS files
      export SOPS_AGE_KEY_FILE=/path/to/old-key-A.txt  # needed to decrypt for re-encryption
      for f in sops/*.sops.yaml; do
        sops updatekeys --yes "$f"
      done
      # sops updatekeys reads the new recipient from sops/.sops.yaml

    Step 4: Verify decryption with key B
      export SOPS_AGE_KEY_FILE=/tmp/new-key-B.txt
      for f in sops/*.sops.yaml; do
        sops -d "$f" > /dev/null && echo "OK: $f" || echo "FAIL: $f"
      done

    Step 5: Verify old key CANNOT decrypt
      export SOPS_AGE_KEY_FILE=/path/to/old-key-A.txt
      sops -d sops/bootstrapcreds.sops.yaml
      # Must fail — if it succeeds, updatekeys did not run correctly

    Step 6: Commit all re-encrypted files
      git add sops/.sops.yaml sops/*.sops.yaml
      git commit -m "chore(sops): rotate age key — all files re-encrypted to new key"

    Step 7: Document old key destruction
      # Record in ops-log.md: date, reason for rotation, confirmation old key destroyed
      # Destroy old key: shred -vzu old-key-A.txt (if on disk)

    Step 8: Move new key to offline custody
      # Store /tmp/new-key-B.txt on offline media per custody runbook
      # Delete from workstation: shred -vzu /tmp/new-key-B.txt
    ```

- [ ] **Task 2: Test the rotation procedure end-to-end** (AC: 2, 3, 4, 5)
  - [ ] Generate key A (simulated old key) and key B (new key) in a test environment
  - [ ] Encrypt a test SOPS file with key A
  - [ ] Run `updatekeys` procedure to re-encrypt to key B
  - [ ] Confirm: decrypt with key A fails, decrypt with key B succeeds
  - [ ] Confirm `sops/.sops.yaml` shows key B recipient
  - [ ] Check no file references old key: `grep -r "age1<key-A-pubkey>" sops/`
  - [ ] Record test results in story completion notes

- [ ] **Task 3: Add rotation to `age-key-custody.md` index** (AC: 1)
  - [ ] Update `docs/runbooks/age-key-custody.md` table of contents to include rotation section
  - [ ] Add cross-reference: "Key rotation should be performed immediately if: (a) old key media is lost/stolen, (b) key may have been exposed on a networked system, (c) periodic rotation schedule (recommend annual)"

## Dev Notes

### Architecture Constraints

- **FR25 — Age key rotation:** Rotation must re-encrypt ALL SOPS-managed files. Partial rotation (some files on old key, some on new key) leaves the system in a split state and means compromise of either key affects some files.

- **`sops updatekeys` mechanism:** The `updatekeys` command reads the current `.sops.yaml` creation rules to determine the new recipients, then re-encrypts the file's data key to those recipients. The old age key must be available to DECRYPT the data key — this is why `SOPS_AGE_KEY_FILE` must be set to the old key during the `updatekeys` run. After `updatekeys`, the file is re-encrypted to the new recipient set.

- **Atomic rotation:** The rotation is not atomic across all files — if interrupted mid-rotation, some files will be on old key and some on new key. The procedure should be run to completion in a single session. If interrupted, check which files still reference the old key and re-run `updatekeys` on those files.

- **terraform.tfstate rotation:** The `terraform.tfstate` is SOPS-encrypted (D4). It must also be re-encrypted during key rotation: `sops updatekeys --yes terraform.tfstate`. This is easy to forget — add explicitly to the rotation procedure.

- **No git history rewrite needed:** After rotation, the git history contains files encrypted to old key A. This is acceptable — those historical commits are only decryptable with old key A, which is being destroyed. Once old key A is destroyed, historical encrypted content is permanently inaccessible (which is the desired outcome of key rotation with destruction).

### Project Structure Notes

- This story only modifies `docs/runbooks/age-key-custody.md` (adds rotation section).
- No OpenTofu or SOPS file changes are made to the committed repo state by this story — the rotation procedure is demonstrated and tested but the repo continues to use the production age key.
- The `terraform.tfstate` rotation step is documented in the runbook — actual execution happens at rotation time, not as part of this story's AC.

### References

- [Source: docs/terminus/infra/secrets/epics.md#Story 2.6]
- [Source: docs/terminus/infra/secrets/architecture.md#Bootstrap & Credential Delivery — D1]
- [Source: docs/terminus/infra/secrets/architecture.md#Naming Patterns — SOPS File Naming]

## Dev Agent Record

### Agent Model Used

claude-sonnet-4-5

### Debug Log References

### Completion Notes List

### File List
