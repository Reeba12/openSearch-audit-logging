# Knowledge Transfer Document

---

## BUILD 1.1

### FRMS Expansion Reports

Documents BaseMessage expansion in FRMS-COE-LIB: flexible, schema-agnostic transaction envelope letting Tazama process any non-Pacs002 message type without new TypeScript interfaces or protobuf defs per format. Payload validation deferred to runtime via Redis-stored schemas — new formats onboard through config alone.

- **Note:** SafeObject implementation not yet in place. For type safety, created a "set variable with type" node — used for variable declaration, adds types alongside in code.
- **KT-FRMS-COE-LIB.md** — knowledge transfer doc, full four-phase BaseMessage wiring: interface/protobuf expansion, SafeObject schema enforcement, rule executor Redis integration, CIMS service extension. Includes design rationale, source references.
- **SundayStatus.md** — weekend progress, Build 1.1 work-in-progress tracker.
- **Report-FRMS-COE-LIB-2Week-FunctionalDelta.md** — runtime-impacting changes only. Per-file deltas w/ commits, risk notes on breaking/integration-sensitive changes.
- **Report-FRMS-COE-LIB-2Week-ChangeInventory.md** — full file-by-file change inventory across source, tests, docs, package metadata, w/ commit references.
- **Report-FRMS-COE-LIB-2Week-ChangeInventory-Compact.md** — condensed three-table version of same inventory, for quick PR/status review.

**Reference Documents:**
- https://github.com/ahmad-paysys/frms-expansion-docs
- https://github.com/ahmad-paysys/frms-expansion-docs/blob/main/Report-FRMS-COE-LIB-2Week-FunctionalDelta.md

---

### Audit Lib

This document provides guidance for building an audit logging sub-system for Tazama, a rules-based fraud and money-laundering transaction monitoring platform that currently has system logging but lacks application-level audit logging. It outlines the Tazama system and ecosystem architecture (TMS, CIMS, BIAR, Rule Studio, Data Lakehouse), distinguishes audit logging from system logging, and describes the current logging stack (Processor → Event-Sidecar → NATS → Lumberjack → ELK). The core of the document is 17 principles (P1-P17) covering licensing, RBAC via Keycloak, multi-tenancy, atomicity, tamper-evidence, real-time logging, and compliance with PCI-DSS, SOX, GDPR, and ISO 27001, plus 7 assumptions (A1-A7) on scope, encryption, and async write patterns. It proposes a synchronous audit logging architecture using OpenSearch, where the application blocks until the write is acknowledged, with a future batch sync to the Data Lakehouse for analytics. An appendix compares candidate tech stacks: Fluent Bit + Data Prepper + OpenSearch versus the Grafana LGTM stack, and NATS versus Data Prepper for log routing.

- **Live OpenSearch:** http://10.10.80.30:5601/
- **FSD Link:** https://github.com/Reeba12/openSearch-audit-logging/blob/main/Tazama%20Audit%20Logging%20Product%20Guidance%20v1.0%2020251111.md

---

### Simulation Studio

Repo holds full planning docs, architecture diagrams, DB schema, dev workflow for commercial Simulation Studio initiative (Single-Rule Simulation, Release 4). Spans two codebases: `rule-studio/backend` (NestJS — new SimulationStudioModule), `rule-studio/frontend` (React+Vite — UI/wizard), `admin-service` (Fastify — persistence layer).

- **Scope:** Single-Rule Simulation only; Integration Testing deferred.
- **Timeline:** 8 working days (21 May – 4 Jun 2026), 3 devs.
- **Repos/branches:** Two codebases, one feature branch each (`feat-paysys-simulation-studio`); flow: task branch → feature branch → `dev` (protected, PR-only) → `main`.
- **Database:** New PostgreSQL `simulation_studio` DB, 17 tables (suites, generations, contexts, triggers, enrichment, runs, results). Init via `plan/database-migration.sql`.
- **New components:** Backend `SimulationStudioModule` + `simulation-studio.logic.service.ts`; Frontend SimulationStudio pages + `simulationStudioApi` RTK slice; ERD/architecture/data-flow diagrams (Mermaid).
- **Reused, not duplicated:** TazamaAuthGuard, AdminServiceClient, AJV, StatusCard, Table, DropDown, etc. Connection-studio frozen (read-only, no new files).
- **Flow:** Browser (React) → JWT → rule-studio/backend (NestJS) → JWT forwarded → admin-service (Fastify) → PostgreSQL, + Docker orchestration for run execution, Docker Hub for rule-processor images, Keycloak for identity.

**Key docs:** `plan/summary-of-report.md` (exec summary, 5 min), `plan/implementation_report.md` (authoritative tech decisions, 20 min), `plan/branch-plan.md` (git workflow, 15 min), `plan/database-migration.sql` (schema), `plan/trs_simulation_gantt.md` (8-day timeline), `plan/User-Stories-20260520-EOD.md` (21 stories + acceptance criteria).

**Status:** architecture finalized, DB schema designed, API contracts specified, dev workflow documented, diagrams reviewed — ready for development.

**Reference document:** https://github.com/ahmad-paysys/simulation-documentation

---

### Rule Studio

All the decisions and discussions for rule studio are in the user stories.

- **User stories:** https://github.com/frmscoe/paysys-pmo/issues/42
- **Requirement documentation:** Rule Builder Supplementary Doc.docx
