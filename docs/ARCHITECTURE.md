# PlanDoc — System Architecture & Engineering Flows

This document details the core architectural blueprints of **PlanDoc**, emphasizing authorization boundaries, asynchronous consistency, event-driven document certification, and deterministic schedule calculation.

---

## 1. End-to-End Document Lifecycle & 5-Role RBAC Approval Chain

Deliverables originate strictly from a pre-validated Work Breakdown Structure (*Planned Document List*). Creation reserves the unique document code transactionally (`AVAILABLE -> CREATING -> CREATED`) with automatic rollback on binary upload failure.

Once created, revisions follow a server-enforced finite state machine supporting two distinct paths:
- **`INTERNAL` Flow:** Terminates at `Aprovado` after Director deliberation (or authorized Manager bypass).
- **`EXTERNAL` Flow:** Advances through Director deliberation (`Liberado para o Cliente`) before triggering formal Client dispatch (`Enviado para cliente`) and external verdict (`Aprovado`, `Aprovado com Comentários`, or `Rejeitado` spawning `v02`).
- **`AS_BUILT` Coordination Flow (v2):** Field confirmation without technical edits links to the approved numeric revision; modified *As Built* deliverables require Coordination (`Gerente`) approval followed by non-deliberative informational dispatch.

```mermaid
stateDiagram-v2
    [*] --> AVAILABLE: Planned WBS Item Seeded
    AVAILABLE --> CREATING: Reserve Slot and Unique Code
    CREATING --> AVAILABLE: Upload Failure Rollback
    CREATING --> CREATED: Atomic Link Document and Revision v01

    state "Revision Workflow (Enforced by Security Rules)" as WF {
        [*] --> EmElaboracao: Em Elaboração (Revisor / Author)
        EmElaboracao --> AprovacaoCoordenacao: Submit for Review
        AprovacaoCoordenacao --> EmElaboracao: Return for Adjustment (Gerente / Admin)
        AprovacaoCoordenacao --> AprovacaoDirecao: Advance to Executive Gate (Gerente / Admin)
        AprovacaoDirecao --> AprovacaoCoordenacao: Return to Coordination (Diretor)
        AprovacaoDirecao --> Reprovado: Technical Rejection (Diretor)
        Reprovado --> EmElaboracao: Rework Next Revision

        AprovacaoDirecao --> Aprovado: Approve - flowType INTERNAL (Diretor)
        AprovacaoDirecao --> LiberadoCliente: Release Externally - flowType EXTERNAL (Diretor)
        LiberadoCliente --> EnviadoCliente: Dispatch Queue Executed

        EnviadoCliente --> Aprovado: Client Verdict - Approved
        EnviadoCliente --> AprovadoComentarios: Client Verdict - Approved w/ Comments
        EnviadoCliente --> EmElaboracao: Client Verdict - Rejected (Next Rev)
        AprovadoComentarios --> EmElaboracao: Open Corrective Revision

        EmElaboracao --> Suspenso: Suspend (Admin / Gerente / Diretor)
        Suspenso --> EmElaboracao: Resume
        Aprovado --> Obsoleto: Superseded
    }
```

### Role & Permission Gate Flow

```mermaid
flowchart TD
    REQ["Client Action / API Mutation"] --> GATE{"isAccountOperational() & Project Writable?"}
    GATE -- "No" --> DENY["HTTP 403 / Permission Denied"]
    GATE -- "Yes" --> SCOPE{"hasAccessToProject(projectId)?"}
    SCOPE -- "No" --> DENY
    SCOPE -- "Yes" --> ROLE{"Check Actor Role & Stage"}

    ROLE -- "Revisor (Author)" --> REV["Upload Initial/Corrective Draft and Manage Own Holds"]
    ROLE -- "Gerente / Admin" --> GER["Review Coordination Stage, Approve As-Built, Manage Baselines and Holds"]
    ROLE -- "Diretor" --> DIR["Exclusive Executive Gate (Internal Approve or Release to Client)"]
    ROLE -- "Cliente (External)" --> CLI{"Not Restricted and Client-Visible Status?"}
    CLI -- "No" --> DENY
    CLI -- "Yes" --> VER["Inspect and Submit Formal Verdict (Approved / Comments / Rejected)"]
```

---

## 2. Fail-Closed Onboarding & Row-Level Firestore / Storage Isolation

User privileges are never trusted from client claims or browser state. Accounts start pre-authorized and locked behind an onboarding state machine (`TEMP_PASSWORD_SENDING -> TEMP_PASSWORD_SENT -> COMPLETED`, or `TEMP_PASSWORD_UNKNOWN` on ambiguous SMTP responses requiring manual reconciliation).

```mermaid
sequenceDiagram
    autonumber
    participant Admin as Admin Console
    participant CF as Cloud Functions (Admin SDK)
    participant Auth as Firebase Auth
    participant DB as Firestore
    participant Mail as SMTP Transport
    participant User as End User SPA

    Admin->>CF: adminCreateUser(email, role, projectIds)
    CF->>Auth: Create Auth Account
    CF->>DB: Write users/{uid} (active=true, mustChangePassword=true, state=TEMP_PASSWORD_SENDING)
    CF->>Mail: Send Temporary Credential
    alt SMTP Accepted
        CF->>DB: Update state = TEMP_PASSWORD_SENT
    else Ambiguous SMTP / Timeout
        CF->>DB: Update state = TEMP_PASSWORD_UNKNOWN (Blocks Auto-Retry)
    end

    User->>Auth: First Sign-In (Password or Verified SSO)
    User->>CF: completeFirstAccessPasswordChange()
    CF->>Auth: Update Password and Revoke Refresh Tokens
    CF->>DB: Set mustChangePassword=false, state=COMPLETED + Audit Log
    User->>DB: Operational Queries (Rules Evaluate isAccountOperational())
```

### Multi-Service Isolation & Session-Bound Storage

Because Cloud Storage Security Rules impose a strict limit of **two cross-service `firestore.get()` / `exists()` calls** per rule evaluation, new uploads decouple authorization into pre-issued `uploadSessions/{uploadId}` documents:

```mermaid
flowchart TD
    subgraph "Firestore Row-Level Guards"
        U["users/{uid}"] --> OP["isAccountOperational() - exists, active != false, mustChangePassword != true"]
        OP --> LIST["documents list - External Roles Filtered by Root currentStatus and restricted == false"]
        OP --> GET["documents get - Project Membership, Visibility Whitelist, and Non-Restricted Check"]
        OP --> SYS["systemConfig/externalDelivery and externalDispatches - Client List/Write Blocked"]
    end

    subgraph "Cloud Storage 2-Lookup Session Guard"
        REQ_UP["PUT documents/{docId}/{version}/{uploadId}/{fileName}"] --> L1["Lookup 1: uploadSessions/{uploadId} (uid match, exact path, unconsumed, TTL 30m, exact size)"]
        REQ_UP --> L2["Lookup 2: projects/{session.projectId} (ACTIVE, OPEN, unarchived)"]
        L1 --> ALLOW["Allow Immutable Create (resource == null, under 200MB, no .exe/.bat)"]
        L2 --> ALLOW
    end
```

---

## 3. Queue-Based External Delivery & PDF Trailer-Page Stamping

### A. External Delivery Outbox & SMTP Reconciliation

External notifications are separated from client approval state mutations and guarded against duplicate delivery, unauthorized test leaks, and silent network ambiguity:

```mermaid
flowchart TD
    TRIG["Trigger / Callable (Document Released or As-Built Approved)"] --> ELIG{"Fail-Closed Eligibility Gate (unrestricted, EXTERNAL, current rev)"}
    ELIG -- "Fail" --> REJECT["Abort Without Persistence"]
    ELIG -- "Pass" --> MODE{"Delivery Mode Config"}
    MODE -- "DISABLED" --> SKIP["Record Disabled / No SMTP Call"]
    MODE -- "TEST" --> ALLOWLIST{"Recipient in Test Allowlist?"}
    ALLOWLIST -- "No" --> BLOCK["Block Non-Allowlisted Address"]
    ALLOWLIST -- "Yes" --> KEY["Compute Idempotency Key (docId + rev + purpose + seq)"]
    MODE -- "LIVE" --> KEY

    KEY --> LEASE["Acquire Per-Recipient SENDING Lease"]
    LEASE --> SMTP["Send Individualized Email via IMailer"]
    SMTP -- "2xx Accepted" --> ACC["Persist ACCEPTED + Audit Trail"]
    SMTP -- "Definitive 4xx/5xx" --> FAIL["Persist FAILED (Eligible for Controlled Retry)"]
    SMTP -- "Timeout / Network Drop" --> UNK["Persist UNKNOWN (Blocks Auto-Retry Until Admin Attestation)"]
```

### B. Event-Driven PDF Approval Certificate Stamping

When a technical document receives executive approval (`DIRECAO_APPROVE` or `APPROVE_WITHOUT_DIRECTOR`), an asynchronous pipeline produces an immutable certified copy with an appended A4 audit trailer page while preserving the original file intact:

```mermaid
sequenceDiagram
    autonumber
    participant DB as Firestore (documents/{docId})
    participant Prod as Producer (detectApprovalEvent)
    participant Jobs as Firestore (pdf_stamp_jobs/{eventId})
    participant Work as Worker (processPdfStampJob)
    participant GCS as Cloud Storage
    participant Cron as Scheduler (recoverExpiredStampJobs)

    DB-->>Prod: onDocumentUpdated (Status -> Aprovado / Liberado)
    Prod->>DB: Transaction - Re-verify Revision, eventId, actionCode, Path
    Prod->>Jobs: Create PENDING Job and Initialize stampMetadata[eventId]

    Jobs-->>Work: onDocumentWritten (status == PENDING)
    Work->>Jobs: Transaction - Claim Lease (PROCESSING, leaseOwner, +10m TTL)
    Work->>GCS: Download Original PDF (Validate Path, <=20MB, %PDF- Magic Bytes)
    Work->>Work: Compute Original SHA-256 and Append A4 Certificate Page (pdf-lib)
    Work->>GCS: Upload Official PDF (If-Generation-Match 0) + SHA-256 Metadata
    Work->>Jobs: Fencing Transaction - Verify leaseOwner == workerId -> COMPLETED
    Work->>DB: Transaction - Sync stampMetadata (officialStoragePath, SHAs)

    Note over Cron,Jobs: Runs every 10 minutes for orphaned leases
    Cron->>Jobs: Query leaseExpiresAt < now and status == PROCESSING
    alt attemptCount < 3
        Cron->>Jobs: Reset to PENDING (Clear Lease)
    else attemptCount >= 3
        Cron->>Jobs: Mark FAILED (MAX_RETRIES) and Sync stampMetadata
    end
```

---

## 4. Schedule Baseline + Critical Path Method (CPM) $O(V + E)$ Flow

Schedule analytics execute as pure memoized computations over the WBS graph without writing derived projections back to the database. All date arithmetic parses `YYYY-MM-DD` strings into civil calendar integers to prevent daylight-saving or UTC shifts.

```mermaid
flowchart TD
    INPUT["Planned WBS Items + Civil Holiday Calendar + Selected Immutable Baseline"] --> VALID{"Graph and Date Validation (Self-refs, Missing Predecessors, Inverted Dates?)"}
    VALID -- "Invalid" --> ERR["Halt CPM Safely - Return validationErrors and invalidDocIds"]
    VALID -- "Valid" --> TOPO["Kahn Topological Sort O(V + E)"]
    TOPO -- "Visited < V" --> CYCLE["Detect Cycle - Flag validationErrors"]
    TOPO -- "DAG Verified" --> FWD["Forward Pass: ES(v) = max(plannedStart(v), nextBusinessDay(max EF(u)))"]

    FWD --> PROJ_END["Project Finish = max(EF across all V)"]
    PROJ_END --> BWD["Backward Pass: LF(v) = min(prevBusinessDay(LS(w))) or Project Finish"]

    BWD --> FLOAT["Compute Total Float: LS(v) - ES(v) in Business Days"]
    FLOAT --> CRIT{"Total Float Classification"}
    CRIT -- "float == 0" --> IS_CRIT["Critical Path Chain (isCritical = true)"]
    CRIT -- "float < 0" --> NEG_FLT["Conflict Alert (negativeFloatDocIds)"]
    CRIT -- "float > 0" --> SLACK["Non-Critical Buffer (+float business days)"]

    IS_CRIT --> COMP["Compare Against Immutable Baseline Snapshot (Start/Finish Variance)"]
    SLACK --> COMP
    NEG_FLT --> COMP
    COMP --> HH["Net Effort Accounting: Net Hh = (Worked Business Days - Paused Business Days) x 8h"]
```

---

## 5. Two-Phase Project Lifecycle Closure Gate

Project termination (`ACTIVE -> COMPLETED` or `CANCELLED`) decouples operational writability (`writeState: OPEN | CLOSING | CLOSED`) from catalog visibility (`archivedAt`) using a two-phase lock and lease protocol:

```mermaid
stateDiagram-v2
    state "Project Write State" as WS {
        OPEN --> CLOSING: 1. Transactional Lock and Acquire 10m Lease (operationId)
        CLOSING --> OPEN: Validation Blocker Found (Release Same operationId)
        CLOSING --> CLOSED: 2. Paginated Audit Passed -> Persist Closure Snapshot
        CLOSED --> OPEN: Explicit Audited Reopen (Admin / Gerente)
    }
```

While in `CLOSING`, Firestore Rules, Storage session guards, and Admin SDK guarded writes reject new document mutations until either every deliverable/dispatch invariant passes (transitioning to `CLOSED` with an immutable `projectClosureSnapshots/{operationId}`) or an explicit blocker restores `OPEN`.

---

## 6. Transactional External Confirmation Lease & Fail-Closed Outbox

Confirming external recipient configurations or per-document overrides acquires a time-bounded snapshot lease before provisioning temporary credentials or triggering outbound notifications:

```mermaid
stateDiagram-v2
    state "Confirmation Lease Lifecycle" as CL {
        DRAFT --> LEASED: reserveConfirmation() - 10m TTL + Snapshot Fingerprint
        LEASED --> CONFIRMED: All Recipients Provisioned and Fingerprint Matches
        LEASED --> DRAFT: Partial Failure -> releaseConfirmationLease()
        LEASED --> ABORTED: Concurrent Snapshot Mutation Detected
    }

    state "Fail-Closed Credential Retry" as CR {
        PENDING --> SENDING: Transactional Attempt Reservation
        SENDING --> SENT: SMTP Accepted (Persist Completed Marker)
        SENDING --> UNKNOWN: Firestore Failure Post-SMTP (Blocks Auto-Retry)
        SENDING --> FAILED: SMTP Rejected (Eligible for Audited Manual Retry)
    }
```

Separating `UNKNOWN` from `FAILED` guarantees that transient database outages after SMTP acceptance never trigger silent duplicate temporary-password dispatches on retry.