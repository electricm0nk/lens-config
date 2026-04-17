---
feature: proxmox-vm-boot-order
phase: techplan
gate: adversarial-review
source: phase-complete
verdict: pass-with-warnings
reviewed_artifacts:
  - architecture.md
date: "2026-04-17"
reviewer: lens-adversarial-review
---

# TechPlan Adversarial Review — proxmox-vm-boot-order

**Verdict:** pass-with-warnings  
**Blocking criticals:** 0 (all resolved during review session)  
**Remaining warnings:** 2 (documented, non-blocking)

---

## Critical Findings — Resolved

### C-01 — `on_boot = true` missing from component change examples
**Status:** ✅ Resolved  
All module component change sections in `architecture.md` now explicitly include `on_boot = true` alongside each `startup {}` block. The architecture also documents this as a required precondition: without `on_boot`, the `startup {}` block has no effect.

### C-02 — automation-host import strategy underspecified
**Status:** ✅ Resolved  
ADR-002 updated: the decision is to use an `import {}` block declared in the module. Rationale documented: keeps import intent in version control, allows any operator to run `tofu apply` safely without a manual pre-step. Pre-apply `tofu plan` review is required to confirm no destructive changes before first apply against VM 9001.

---

## High Findings — Addressed

### H-01 — up_delay values are estimates, not calibrated
**Status:** ⚠️ Documented (non-blocking)  
Added as `open_questions` in architecture frontmatter and as Assumption 4 in the architecture body. `[BOOT-ORDER-ACCEPTANCE]` in `.todo` tracks the operator acceptance validation. Values are explicitly labeled as initial estimates to be tuned after first controlled reboot.

### H-02 — Patroni split-brain risk on cold start
**Status:** ⚠️ Documented (non-blocking)  
The 90s `up_delay` on postgres-primary gates on guest-agent response, not Patroni-ready. This is an accepted residual risk: Patroni has its own DCS election timeout recovery, and the 90s window provides a substantial buffer. Documented in architecture Assumptions.

### H-03 — Single Proxmox node assumption not stated
**Status:** ✅ Resolved  
Assumption 1 added to architecture: all VMs must reside on the same Proxmox physical node (`trantor`). Cross-node boot ordering is not supported by Proxmox autostart.

---

## Medium Findings — Accepted

### M-01 — `down_delay` ordering not rationalized
**Status:** ✅ Acceptable  
Shutdown runs in descending `order` value, which correctly inverts the boot sequence. Workers shut down before control-plane, control-plane before postgres, postgres before automation host. No documentation change required; behavior is implicit in the `bpg/proxmox` provider's shutdown sequencing.

### M-02 — Worker count extensibility
**Status:** ✅ Acceptable  
The formula `var.worker_startup_order_base + index` correctly handles any `worker_count` value without hardcoded order assignments. The architecture narrative calls this out explicitly.

---

## Blind-Spot Challenge Responses

| Question | Response |
|----------|----------|
| Import block vs manual `tofu import`? | **Import block** — declared in module, in version control |
| Recovery time SLO? | No SLO. ~12.5 min sequenced boot is the correct tradeoff; out-of-order startup causes hours of cascading job recycling |
| Single Proxmox node confirmed? | **Yes** — all VMs on `trantor` |
| Acceptance test plan? | Operator-led controlled reboot, tracked in `[BOOT-ORDER-ACCEPTANCE]` in `.todo` |

---

## Remaining Warnings (non-blocking)

1. **up_delay calibration** — initial estimates; validate and tune during first maintenance-window reboot
2. **Patroni split-brain window** — residual risk on cold start; Patroni's own recovery handles this, but the gap between guest-agent-ready and Patroni-ready is not gated

---

## Verdict: pass-with-warnings

Architecture is complete and implementation-ready. TechPlan phase may advance to FinalizePlan.
