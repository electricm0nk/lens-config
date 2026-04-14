---
initiative: terminus-infra-postgres
phase: businessplan
mode: batch
generated: 2026-03-24
instructions: >
  Fill in all answers below each question. When done, save the file and
  tell Copilot "batch responses complete" to generate the PRD from your answers.
---

# BusinessPlan Batch Questionnaire — terminus-infra-postgres

---

## Section 1: Project Classification

**Q1.1 — Project type**
What best describes this initiative? (e.g., infrastructure service, platform component, CLI tool, API, data store)

> Answer:  Build a new highly available postgres cluster that is able to host databases needed for anything we might through at it in the home lab.  web apps, k3s, etc.  

---

**Q1.2 — Project context**
Is this greenfield (new from scratch) or brownfield (adding to / replacing something that exists)?

> Answer:  greenfield

---

**Q1.3 — Domain complexity**
How complex is the operational/compliance domain? (low / medium / high)
Consider: Are there security requirements, data durability constraints, credential rotation concerns, backup/recovery obligations?

> Answer:  fintech, with relaxed requirements for things like SoD, since it's just me.  We just need to document the risk as accepted

---

## Section 2: Product Vision

**Q2.1 — The core problem**
What is the real problem this feature solves — not just "we need postgres", but the underlying ops/infra pain?

> Answer:  we need somewhere robust that can host databases.  I need a place to train and sharpen my skills in the home lab that can be direction transferable to my work life

---

**Q2.2 — What success feels like**
When this is done and working well, what does the operator (you) experience that you don't have today?

> Answer:  I'm able to create multiple databases on the cluster.  The cluster is highly available.  The cluster accepts distinct logins for each service/use.  The cluster can be monitored with standard monitoring tooling to be defined later.  The cluster build is completely scripted so that it can be blasted and repaved.  Backup and other maint jobs are setup to run automatically on all newly created dbs.  Always use security first/best practices

---

**Q2.3 — Why now**
Why does postgres need to be resolved at this point in the terminus build-out? What's blocked until this is done?

> Answer:  I can't installed k3s without postgres.

---

**Q2.4 — Differentiating approach**
Is there a specific approach or design insight that matters here? (e.g., not in k3s, dedicated VM, OpenTofu-provisioned, Vault-managed credentials)

> Answer:  i think we want dedicated vms.  we want opentofu-provisioned.  we want all credentials vaulted.  

---

## Section 3: Success Criteria

**Q3.1 — Operator success**
What specific outcomes would make you say "postgres is done and working"? List 3–5 concrete, checkable outcomes.

> Answer:  I can can login to the ui  We creat a db via automation alond with required login credentials which are then vaulted.  We create a test table and populate it with data via automation.  We do a backup of the server.  We are able to see all these things happening in the logs.  

---

**Q3.2 — Consumer success**
How will downstream consumers (e.g. Temporal) know postgres is ready and trustworthy for them to depend on?

> Answer:  not applicable.  i'm the only user

---

**Q3.3 — Operational success**
What does "postgres is healthy and maintainable" look like in 6 months? (backups, monitoring, credential rotation, etc.)

> Answer:  backups, monitoring, and alerting

---

**Q3.4 — Failure modes to explicitly avoid**
What outcomes would mean this feature failed? (e.g., credentials committed to git, no backup strategy, no runbook)

> Answer:  failure of anything in q3.1

---

## Section 4: User / Operator Journeys

**Q4.1 — Day-0: Initial provisioning**
Walk through what the operator does to stand up postgres for the first time. What's the sequence of commands/steps? What does success look like at each stage?

> Answer:  Opentofu?  

---

**Q4.2 — Day-1: Onboarding a new consumer (e.g. Temporal)**
What does the process look like for a new platform service to get a postgres database and credentials? Who does what?

> Answer:  not defined

---

**Q4.3 — Day-2: Routine operations**
What are the recurring operational tasks expected? (e.g., credential rotation, backup verification, schema migrations, version upgrades)

> Answer:  mostly releases and version upgrades

---

**Q4.4 — Break-glass / recovery**
If postgres goes down or the VM is lost, what is the operator's recovery path? What must be available and documented?

> Answer:  deploy a new server using automation.  restore backups.  RTO 4 hours.

---

## Section 5: Scope — MVP vs. Growth

**Q5.1 — MVP must-haves**
What is the absolute minimum postgres capability that unblocks Temporal (and other consumers)?
List only what MUST be true for MVP.

> Answer:  i don't know

---

**Q5.2 — Explicitly out of MVP scope**
What is deliberately deferred? (e.g., HA/replication, connection pooling, automated failover, PITR, metrics export)

> Answer:  setup up monitoring and alerting (we don't have a solution yet)

---

**Q5.3 — Growth phase candidates**
What do you want to eventually add but not now? (these give shape to the architecture without over-engineering MVP)

> Answer:  plan on maybe a dozen small database.  no more than 100gig in total

---

## Section 6: Functional Requirements

**Q6.1 — Provisioning**
What must be true about how postgres is provisioned? (e.g., OpenTofu only, no manual steps, specific VM config)

> Answer:  there should be multiple avenues.  opentofu, ui, script.

---

**Q6.2 — Database and user management**
How are databases and roles created? Who can create them and how? (e.g., OpenTofu, Ansible playbook, manual with runbook)

> Answer:  don't know

---

**Q6.3 — Credential delivery**
How do consumers get database credentials at runtime? (e.g., Vault KV v2, static secrets, env vars — align with secrets initiative)

> Answer:  don't know

---

**Q6.4 — Backup and restore**
What backup strategy is required? What restore capability must be demonstrable at MVP?

> Answer:  Daily backups at a minimum.

---

**Q6.5 — Connection model**
How do consumers connect? (direct TCP, connection pooler like pgBouncer, service mesh, k3s service)

> Answer:  not defined

---

**Q6.6 — Schema management**
How are schema migrations handled? (e.g., each consumer owns its own migrations, centralized tooling, Flyway/Liquibase, raw SQL)

> Answer:  not sure.  maybe liquibase?

---

**Q6.7 — Monitoring / health**
What health and monitoring signals are required at MVP? (e.g., basic health endpoint, Prometheus metrics, log export)

> Answer:  out of scope.  

---

## Section 7: Non-Functional Requirements

**Q7.1 — Availability target**
What availability is required? (homelab SLA, best-effort, defined RTO/RPO?)

> Answer:  99.95%

---

**Q7.2 — Data durability**
What are the durability requirements? What is the acceptable data loss window (RPO)?

> Answer:  24 hours

---

**Q7.3 — Security**
What security properties are non-negotiable? (e.g., no plaintext passwords, no direct root access, network isolation, encrypted connections)

> Answer:  use best practice security protocols.  Security comes first.

---

**Q7.4 — Performance**
Any specific performance constraints? (e.g., query latency budget for Temporal, max connection count, storage size estimates)

> Answer:  we have a finite amount of resources on the server and they need to host a lot of other services beyond postgres, let's not get crazy just because it's the first thing onboarding

---

**Q7.5 — Operational constraints**
Any operational constraints from the infra constitution or team preference? (e.g., no manual changes outside OpenTofu, all changes via PR)

> Answer:  nothing yet

---

## Section 8: Architecture Constraints & Decisions

**Q8.1 — Postgres version**
What version should be pinned for MVP?

> Answer:  latest

---

**Q8.2 — VM vs. container**
Confirm: postgres runs on a dedicated Proxmox VM (not in k3s), correct? Any notes on sizing?

> Answer:  dedicated vm, k3s doesn't exist yet

---

**Q8.3 — State / provisioning authority**
Confirm OpenTofu is the only provisioning authority. Any exceptions?

> Answer:  OpenTofu is preferred but not required

---

**Q8.4 — Replication / HA**
MVP is single-node. Is there any constraint on the architecture that would make HA harder to add later? Anything to preserve in the design?

> Answer:  MVP is a cluster.  Full HA.

---

**Q8.5 — Integration with secrets**
Confirm: all postgres credentials delivered via Vault KV v2 at `secret/terminus/{env}/postgres/...`. Any variations?

> Answer:  confirmed

---

## Section 9: Open Questions / Notes

Any aspects not covered above that should influence the PRD?

> Answer:  not so far

---
