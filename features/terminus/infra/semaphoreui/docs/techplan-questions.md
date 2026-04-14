---
phase: techplan
initiative: terminus-infra-semaphoreui
generated: 2026-03-28
status: awaiting-answers
---

# TechPlan Batch Questions — Semaphore UI Deployment

Fill in your answers below each question. Leave a question blank to accept the suggested default. When done, respond with `done` or `generate` and I'll build the architecture document from your answers.

---

## Section 1 — Manifest Organization

**Q1.** Where should the Semaphore UI k8s manifests live in `terminus.infra`?

Suggested: `platforms/k3s/manifests/terminus-infra/semaphoreui/`
(Other option: one flat dir like `platforms/k3s/apps/semaphoreui/`)

> **A1:**

---

**Q2.** Should manifests be a single `semaphoreui.yaml` file, or split into per-resource files (e.g., `deployment.yaml`, `service.yaml`, `ingress.yaml`, etc.)?

Suggested: Split per resource — easier to read, diff, and review in PRs.

> **A2:**

---

**Q3.** What namespace should Semaphore UI deploy into?

Suggested: `terminus-infra` (already exists in cluster)

> **A3:**

---

## Section 2 — Container & Deployment

**Q4.** Image tag strategy — `latest` was chosen in the PRD, but pull policy matters. Should the Deployment use `imagePullPolicy: Always` (always re-pull on restart) or `IfNotPresent`?

Suggested: `Always` — since `latest` tag is used, ensures the pod runs the truly current image on each restart.

> **A4:**

---

**Q5.** Resource requests and limits — should we define them now, or leave unset for initial deployment?

Suggested:
```yaml
resources:
  requests:
    cpu: 100m
    memory: 256Mi
  limits:
    memory: 512Mi
```
(No CPU limit — avoids CPU throttling for interactive UI)

> **A5:**

---

**Q6.** How many replicas? Semaphore is not HA-capable with SQLite but with postgres it can theoretically run multiple replicas — however the PVC with `local-path` is `ReadWriteOnce`, which prevents multi-pod.

Suggested: `replicas: 1` (single replica, consistent with `local-path` constraint)

> **A6:**

---

## Section 3 — Persistent Volume

**Q7.** What should the PVC be named?

Suggested: `semaphoreui-data`

> **A7:**

---

**Q8.** What path should the PVC mount inside the container?

Semaphore's default data dir is `/tmp/semaphore`. However, if configured via `SEMAPHORE_TMP_PATH`, it can be changed.

Suggested: `/tmp/semaphore` (default, minimal config)

> **A8:**

---

## Section 4 — Secret Management

**Q9.** What exact Vault KV path will hold Semaphore's credentials?

Suggested: `secret/terminus/infra/semaphoreui` (note: PRD used `semaphore`, but `semaphoreui` is more consistent with the initiative name — confirm which you prefer)

> **A9:**

---

**Q10.** What fields should the `ExternalSecret` sync from Vault into the k8s Secret `semaphoreui-secrets`?

Suggested fields (matching Semaphore environment variable names):
- `admin_user` → `SEMAPHORE_ADMIN`
- `admin_password` → `SEMAPHORE_ADMIN_PASSWORD`
- `admin_email` → `SEMAPHORE_ADMIN_EMAIL`
- `db_password` → `SEMAPHORE_DB_PASS`
- `access_key_encryption` → `SEMAPHORE_ACCESS_KEY_ENCRYPTION` (random 32+ char string)

> **A10:**

---

**Q11.** Where should the Postgres DB password be stored in Vault — co-located with the app secrets, or at a separate path?

Suggested: Co-located at the same Vault path (e.g., `secret/terminus/infra/semaphoreui` with a `db_password` field) — simpler single ExternalSecret.

> **A11:**

---

## Section 5 — Database Connection

**Q12.** How should the Postgres connection string be injected into Semaphore?

Semaphore reads individual env vars (`SEMAPHORE_DB_HOST`, `SEMAPHORE_DB_PORT`, `SEMAPHORE_DB_NAME`, `SEMAPHORE_DB_USER`, `SEMAPHORE_DB_PASS`) rather than a DSN string.

Suggested: Hardcode host/port/name/user as plain env vars in the Deployment manifest (non-secret), inject only `SEMAPHORE_DB_PASS` from the k8s Secret.

> **A12:**

---

**Q13.** Postgres database name and role name — confirm or change from PRD default of `semaphore`?

Suggested: `semaphore` (db and role same name, consistent with module pattern)

> **A13:**

---

## Section 6 — Ingress & TLS

**Q14.** Exact hostname for Semaphore UI ingress?

Suggested: `semaphore.trantor.internal`

> **A14:**

---

**Q15.** Should the Certificate and Ingress be in separate files, or combined (Certificate as an annotation-driven cert-manager resource vs explicit Certificate object)?

Suggested: Explicit `Certificate` object in its own file — more visible, easier to inspect status with `kubectl get certificate`.

> **A15:**

---

**Q16.** TLS secret name (the k8s Secret that cert-manager will create to hold the certificate)?

Suggested: `semaphoreui-tls`

> **A16:**

---

## Section 7 — ArgoCD Configuration

**Q17.** Where should the ArgoCD `Application` manifest live?

Suggested: `platforms/k3s/argocd/apps/semaphoreui.yaml` (in the existing App-of-Apps directory)

> **A17:**

---

**Q18.** ArgoCD sync policy — automated with self-heal and prune, or manual?

Suggested: Automated with self-heal and prune. This is the GitOps reference pattern — changes in git are automatically applied.

> **A18:**

---

**Q19.** Should ArgoCD track the `main` branch of `terminus.infra`, or a specific release tag/SHA?

Suggested: `main` branch (HEAD tracking) — suitable for homelab, reduces deployment ceremony.

> **A19:**

---

## Section 8 — ArgoCD Ingress

**Q20.** What hostname for the ArgoCD ingress?

Suggested: `argocd.trantor.internal`

> **A20:**

---

**Q21.** ArgoCD runs with TLS enabled by default internally. To let Traefik terminate TLS at the edge, ArgoCD server needs to run in `--insecure` mode (or equivalent). Where should this setting be applied?

Suggested: `argocd-cmd-params-cm` ConfigMap, `server.insecure: "true"` — avoids modifying the ArgoCD Deployment directly.

> **A21:**

---

**Q22.** Where should the ArgoCD ingress manifest live?

Suggested: `platforms/k3s/manifests/argocd/ingress.yaml` (in an argocd manifests dir, separate from the App-of-Apps)

> **A22:**

---

## Section 9 — Postgres DB Tofu

**Q23.** Where should the Tofu environment for the semaphore DB provisioning live?

Suggested: `tofu/environments/postgres/semaphore/` (following the existing `postgres/dev/` pattern)

> **A23:**

---

**Q24.** The Tofu postgres-databases module presumably takes a list of databases+roles to create. Should the semaphore environment be a standalone `main.tf` that calls the module, or should semaphore be added to an existing databases environment?

Suggested: Standalone — keeps concerns separate, consistent with standard Tofu env-per-concern pattern. The existing `postgres/dev/` is for the platform DB cluster itself, not app databases.

> **A24:**

---

## Section 10 — Deployment Order & Dependencies

**Q25.** What is the required sequencing for execution? Mark any ordering constraints.

Suggested order:
1. Vault KV path seeded (manual, operator prerequisite — not automated)
2. Tofu: provision postgres `semaphore` DB and role
3. ESO `ExternalSecret` manifests applied (may be part of k8s manifest story)
4. Semaphore `Deployment`, `Service`, `PVC` manifests applied via ArgoCD
5. `Certificate` and `Ingress` for Semaphore applied via ArgoCD
6. ArgoCD `--insecure` ConfigMap patched + ArgoCD `Ingress` and `Certificate` applied
7. DNS records added (Synology: `semaphore.trantor.internal` + `argocd.trantor.internal` → `10.0.0.126`)

> **A25:** (confirm order or adjust)

---

## Section 11 — Anything Else

**Q26.** Are there any constraints, preferences, or decisions not covered above that should be captured in the architecture document?

> **A26:**

---

_When you've filled in your answers, reply with `done` and I'll generate `architecture.md` and `tech-decisions.md`._
