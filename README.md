# PlanDoc — Enterprise Engineering Document Management & Project Planning Architecture

> **Portfolio & Architecture Showcase Notice**  
> This repository is an **architectural case study** prepared for technical evaluation and recruitment. To protect proprietary intellectual property, client confidentiality, and operational security rules, **all production source code, Firebase security rule implementations, internal deployment identifiers, and project datasets remain in a private repository**.

---

## Executive Summary

**PlanDoc** is a full-stack **Electronic Document Management System (EDMS)** and **Integrated Engineering Planning Platform** designed for multi-discipline consulting and capital-engineering projects. It unifies WBS-driven Deliverable Lists (*Lista de Documentos — LD*), multi-tier internal and external approval workflows, immutable audit trails, automated PDF certification, queue-driven stakeholder notifications, and deterministic Critical Path Method (CPM) scheduling on a serverless **React 19 + TypeScript + Firebase** architecture.

### The Engineering Problem

In multi-discipline engineering delivery (Civil, Mechanical, Electrical, Process, Instrumentation, Automation, Piping, and Project Management), teams typically face three fragmented failure modes:

1. **Spreadsheet-Driven Drift:** Deliverable lists, schedule dependencies, and revision numbers live in disconnected spreadsheets, causing duplicate codes, unapproved uploads, and stale status reporting.
2. **Unbounded Access & Unaudited Approvals:** Informal email attachments bypass formal deliberation gates (Engineering Reviewer $\to$ Project Manager $\to$ Technical Director $\to$ External Client) and lack verifiable evidence of who approved which file revision and when.
3. **Opaque Schedule & Effort Accounting:** Traditional trackers conflate gross calendar elapsed days with net engineering effort (*Man-Hours / Hh*), ignore external hold periods, and lack automated predecessor locking or baseline variance tracking.

PlanDoc resolves these gaps by binding **controlled WBS planning**, **fail-closed authorization rules**, **event-driven certification and dispatch workers**, and **pure deterministic schedule math** into a single cohesive platform.

---

## Core Architectural Pillars

### 1. Five-Tier RBAC & Multi-Tenant Project Isolation

Access control is enforced across five operational personas (plus an optional read-only observer tier for passive external auditing) combined with explicit project allowlists (`projectIds`) and flow segmentation (`INTERNAL` vs `EXTERNAL`):

| Role | Operational Scope | Planning & Schedule | Approval Authority |
| :--- | :--- | :--- | :--- |
| **`Admin`** | Global platform governance, user provisioning, delivery config | Full read/write across projects | Can delegate/act at Manager gate; exclusive manual retry & reconciliation |
| **`Gerente` (Project Manager)** | Operational coordination, team allocation, WBS & baselines | Full schedule, baselines, priority shift & pauses | Reviews drafts, returns for rework, advances to Director, approves *As Built* |
| **`Diretor` (Technical Director)** | Executive technical governance across projects | Read-only analytical visibility | Mandatory executive gate prior to external release or internal closure |
| **`Revisor` (Lead Engineer / Author)** | Discipline execution within assigned projects | Can declare/manage pauses on own deliverables | Creates initial revisions from WBS slots and submits corrections |
| **`Cliente` (External Stakeholder)** | External client / fiscal inspector (project-scoped) | No access to internal planning or effort metrics | Issues formal verdict (`Approved`, `Approved with Comments`, `Rejected`) |

- **Sensitive Document Isolation:** Management-restricted artifacts (`restricted: true`) are blocked at both Firestore `get`/`list` rules and Cloud Storage download rules for external roles, and hard-rejected by external email dispatchers.
- **Root Status Denormalization:** `currentStatus` is maintained in lockstep with the latest embedded revision so Firestore `list` queries can filter client-visible deliverables server-side without scanning nested arrays.

---

### 2. Fail-Closed Security Rules & Operational Account Gating

PlanDoc follows a **fail-closed** security posture across browser SDK access, Cloud Storage paths, and Admin SDK Cloud Functions:

- **`isAccountOperational()` Gate:** Every privileged Firestore rule, Storage rule, and operational Cloud Function verifies not only Firebase Auth identity, but also that the canonical `users/{uid}` document exists, `active !== false`, and `mustChangePassword !== true`.
- **Self-Service Escalation Prevention:** Client-side user self-creation is blocked (`allow create` requires operational `Admin`). First-access onboarding and pre-authorized profile claiming run exclusively inside Admin SDK transactions using explicit field allowlists. Self-updates are constrained to cosmetic profile fields (`name`, `avatar`, `deviceTokens`) with immutable `role`, `projectIds`, `email`, and `active` flags.
- **Server-Side Finite State Machine:** Document status transitions (`isStatusTransitionAllowed()`) validate `fromStatus -> toStatus` edges, actor role, and `flowType` directly inside Firestore Rules, preventing stale clients or direct API calls from bypassing review stages or swapping binary URLs on approved revisions.
- **Session-Scoped Storage Uploads:** File writes use short-lived server-issued upload sessions (`uploadSessions/{uploadId}` with 30-minute TTL, exact byte-size matching, immutable random path `documents/{docId}/{version}/{uploadId}/{fileName}`, and `.bat`/`.exe` blocking), engineered specifically around Cloud Storage's cross-service two-Firestore-lookup limit.

---

### 3. Asynchronous External Delivery & PDF Approval Stamping

External communication and official artifact certification are decoupled from synchronous UI actions using transactional outbox/job queues:

- **Queue-Driven External Delivery (`externalDelivery`):**
  - Mode-gated dispatch engine (`DISABLED`, `TEST` with allowlist enforcement, and `LIVE`) prevents accidental emails in non-production environments.
  - Deterministic idempotency keys (`documentId + revision + purpose + sequence`) prevent duplicate dispatches across retries.
  - Per-recipient SMTP tracking distinguishes confirmed failures (`FAILED`) from ambiguous network outcomes (`UNKNOWN`), requiring explicit operator reconciliation with SMTP log attestation rather than blind automatic resends.
  - Separates formal approval dispatches (`purpose: APPROVAL`) from *As Built* informational notices (`purpose: INFORMATION` via `externalDeliveryOutbox`) so informational notices never reopen client voting or overwrite approved status.
- **Event-Driven PDF Trailer-Page Certificate Stamping (`pdfStamp`):**
  - A Firestore `onDocumentUpdated` producer detects final approval transitions (`Aprovado` / `Liberado para o Cliente`) with verified `actionCode` and `eventId`, creating an idempotent job in `pdf_stamp_jobs/{approvalEventId}` transactionally.
  - A worker claims `PENDING` jobs via atomic lease (`leaseOwner`, 10-minute `leaseExpiresAt`), validates `%PDF-` magic bytes, 20 MB bounds, and non-encryption, computes the original **SHA-256** digest, and uses `pdf-lib` to append an A4 electronic approval certificate trailer page containing event ID, actor snapshot, timezone-normalized timestamp, and original hash.
  - Official artifacts are written immutably using Storage precondition `If-Generation-Match: 0` and verified via SHA-256 comparison on conflicts, followed by a final lease-fencing transaction before updating `stampMetadata`.
  - A scheduled 10-minute reaper (`recoverExpiredStampJobs`) reclaims timed-out leases (up to 3 attempts), complemented by an `Admin`-only manual retry callable (`retryFailedStampJob`).

---

### 4. Immutable Baselines & Critical Path Method (CPM) Schedule Engine

The Management Module brings deterministic project controls into the browser without mutating production databases during analysis:

- **Finish-to-Start (FS) Precedence Locking:** A deliverable remains locked until 100% of its upstream predecessors record an actual completion date (`dataFimReal`). Circular dependencies and self-references are rejected prior to persistence.
- **Pure Civil Calendar Arithmetic:** Business-day calculations operate on parsed civil date tuples `[YYYY, MM, DD]` with national holiday sets, eliminating UTC midnight offset bugs across client timezones.
- **Immutable Contractual Baselines:** Managers freeze schedule snapshots into append-only `projectBaselines` (`allow update, delete: if false`) bounded by an 800 KB payload guard, enabling dual-bar Gantt overlays and Start/Finish Variance calculations in business days.
- **Deterministic $O(V + E)$ CPM Engine:** Memoized pure topological sort (Kahn's algorithm) validates graph integrity (cycles, missing predecessors, inverted dates) and runs Forward Pass (`ES`, `EF` constrained by planned start) and Backward Pass (`LS`, `LF`) to isolate zero-float critical chains and surface `negativeFloatDocIds` schedule conflicts.
- **Priority Shift & Net Effort Accounting:** Hold intervals (`pauseHistory`) are deducted from worked business days ($\text{Net Hh} = (\text{Worked Business Days} - \text{Paused Business Days}) \times 8\text{h/day}$) and optionally shift schedule capacity to higher-priority projects with previewed cascading impact.

---

### 5. Emulator-Isolated CI & Fail-Closed Test Harness

Automated verification executes against isolated local Firebase Emulators (`Auth`, `Firestore`, `Storage`, `Functions`) using strict safety invariants:

- **Demo-Only Project Enforcement:** The test runner (`run_plandoc_tests`) rejects any project ID not matching `/^demo-[a-z0-9-]+$/` and forces loopback-only emulator host bindings (`127.0.0.1`).
- **Environment & Secret Sanitization:** Ambient `SMTP_*`, `GOOGLE_APPLICATION_CREDENTIALS`, `CLOUDSDK_AUTH_CREDENTIAL_FILE_OVERRIDE`, and `FIREBASE_CONFIG` variables are stripped from test subprocesses, and any local `.env*`/`.secret*` files inside `functions/` trigger a fail-closed abort.
- **Hermetic Transport & Runtime Parity:** Real SMTP transport is replaced by a `FakeMailer` boundary guarded by `NODE_ENV === 'test' && FUNCTIONS_EMULATOR === 'true'`, while integration suites run sequentially on **Node 22** with isolated emulator project namespaces to avoid cross-suite fixture collisions.

---

## Technology Stack

| Layer | Technologies |
| :--- | :--- |
| **Frontend SPA** | React 19, TypeScript (~5.8, `strict`), Vite 6, Tailwind CSS 4, Framer Motion, Lucide React |
| **Client Data & Exports** | Firebase JS SDK v12 (Auth, Firestore, Storage, Functions), ExcelJS, UTF-8 BOM CSV Exporter |
| **Serverless Backend** | Firebase Cloud Functions v2 (Node 22), TypeScript, `firebase-admin`, `pdf-lib`, Nodemailer |
| **Security & Storage** | Cloud Firestore Security Rules v2, Cloud Storage Security Rules v2, Firebase Hosting HTTP Headers |
| **Quality & Testing** | Vitest 3, `@firebase/rules-unit-testing`, Firebase Emulator Suite (`Auth`, `Firestore`, `Storage`, `Functions`) |

---

## Architecture Documentation Index

- **[`docs/ARCHITECTURE.md`](./docs/ARCHITECTURE.md)** — Deep-dive Mermaid diagrams covering the document approval state machine, `isAccountOperational()` onboarding and row-level isolation, outbox external delivery, leased PDF stamping pipeline, and $O(V + E)$ CPM baseline flow.
- **[`docs/QUALITY_AND_SECURITY.md`](./docs/QUALITY_AND_SECURITY.md)** — ISO/IEC 25010 quality alignment, fail-closed CI emulator harness, Hosting security header policy, upload session reservation mechanics, and two-phase project closure locks.
- **[`LICENSE`](./LICENSE)** — Copyright and proprietary IP protection notice.

---

*Authored by **Helder Murilo** — Software & Quality Engineering Architecture Showcase.*
