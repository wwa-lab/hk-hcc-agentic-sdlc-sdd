# Project Management Data Flow

## Purpose

Runtime sequences, state machines, error cascade, and refresh strategy for the Project Management slice. This document operationalizes the architecture in [project-management-architecture.md](project-management-architecture.md) and the contracts in [../05-design/contracts/project-management-API_IMPLEMENTATION_GUIDE.md](../05-design/contracts/project-management-API_IMPLEMENTATION_GUIDE.md).

All Mermaid diagrams use Mermaid 8.x-compatible syntax.

---

## 1. Load Lifecycle

### 1.1 Portfolio view — Phase A (frontend-only, mocked)

```mermaid
sequenceDiagram
    autonumber
    participant U as User
    participant Router as Vue Router
    participant View as PortfolioView
    participant Store as PM Store
    participant Api as PM API Client
    participant Mock as Mock Fixtures

    U->>Router: Navigate /project-management
    Router->>View: Mount
    View->>Store: initPortfolio(workspaceId)
    Store->>Api: getPortfolioAggregate(workspaceId)
    Api->>Mock: load aggregate.mock.ts
    Mock-->>Api: PortfolioAggregateDto
    Api-->>Store: cards populated
    Store-->>View: reactive render
    View-->>U: Portfolio paint (summary, heatmap, capacity, risks, deps, cadence)
```

### 1.2 Portfolio view — Phase B (backend integration)

```mermaid
sequenceDiagram
    autonumber
    participant U as User
    participant View as PortfolioView
    participant Store as PM Store
    participant Api as PM API Client
    participant BE as ProjectManagementController
    participant Svc as PortfolioService
    participant Proj as 6x Projections
    participant DB as DB

    U->>View: /project-management?workspaceId=ws-42
    View->>Store: initPortfolio(ws-42)
    Store->>Api: getPortfolioAggregate(ws-42)
    Api->>BE: GET /api/v1/project-management/portfolio?workspaceId=ws-42
    BE->>Svc: aggregate(ws-42, caller)
    par 6 parallel projections
      Svc->>Proj: summaryProjection(ws-42)
      Proj->>DB: SELECT counters
      DB-->>Proj: rows
      Proj-->>Svc: Summary data
    and
      Svc->>Proj: heatmapProjection(ws-42, week)
      Proj->>DB: SELECT milestone snapshot
      DB-->>Proj: rows
      Proj-->>Svc: Heatmap data
    and
      Svc->>Proj: capacityMatrixProjection(ws-42)
      Proj->>DB: SELECT capacity_allocation JOIN member
      DB-->>Proj: rows
      Proj-->>Svc: Capacity data
    and
      Svc->>Proj: riskConcentrationProjection(ws-42)
      Proj->>DB: SELECT risk_signal
      DB-->>Proj: rows
      Proj-->>Svc: Risks data
    and
      Svc->>Proj: bottleneckProjection(ws-42)
      Proj->>DB: SELECT project_dependencies
      DB-->>Proj: rows
      Proj-->>Svc: Bottleneck data
    and
      Svc->>Proj: cadenceProjection(ws-42)
      Proj->>DB: SELECT milestone history
      DB-->>Proj: rows
      Proj-->>Svc: Cadence data
    end
    Svc-->>BE: PortfolioAggregateDto(SectionResult&lt;T&gt; x6)
    BE-->>Api: 200 JSON
    Api-->>Store: cards populated
    Store-->>View: reactive render
    View-->>U: Portfolio paint
```

### 1.3 Plan view — Phase B initial load

```mermaid
sequenceDiagram
    autonumber
    participant U as PM
    participant View as PlanView
    participant Store as PM Store
    participant Api as PM API Client
    participant BE as ProjectManagementController
    participant Guard as PlanAccessGuard
    participant Svc as PlanService
    participant DB as DB

    U->>View: /project-management/proj-8821
    View->>Store: initPlan(proj-8821)
    Store->>Api: getPlanAggregate(proj-8821)
    Api->>BE: GET /api/v1/project-management/plan/proj-8821
    BE->>Guard: authorize(caller, proj-8821, READ)
    Guard->>DB: resolve project + role
    DB-->>Guard: ok
    Guard-->>BE: ok
    BE->>Svc: aggregate(proj-8821)
    par 9 parallel projections (header, milestones, capacity, risks, deps, progress, changeLog-p0, aiSuggestions)
      Svc->>DB: SELECT projections
      DB-->>Svc: rows
    end
    Svc-->>BE: PlanAggregateDto
    BE-->>Api: 200 JSON
    Api-->>Store: cards populated
    Store-->>View: reactive render
    View-->>U: Plan paint
```

---

## 2. Error Cascade & Isolation

Every card hydrates independently. A failure in one projection must not block the rest.

```mermaid
flowchart TB
    Start[Aggregate request]
    Start --> P1[Projection 1]
    Start --> P2[Projection 2]
    Start --> P3[Projection 3]
    Start --> PN[Projection N]
    P1 -->|success| S1[SectionResult.ok]
    P1 -->|failure| E1[SectionResult.error with code]
    P2 -->|success| S2[SectionResult.ok]
    P2 -->|failure| E2[SectionResult.error]
    P3 -->|success| S3[SectionResult.ok]
    PN -->|success| SN[SectionResult.ok]
    S1 --> Merge[Aggregate DTO assembled]
    E1 --> Merge
    S2 --> Merge
    E2 --> Merge
    S3 --> Merge
    SN --> Merge
    Merge -->|HTTP 200| UI[UI renders card states]
    UI --> UIOK[Card OK]
    UI --> UIErr[Card Error + Retry button]
```

- Aggregate endpoint returns HTTP 200 even when some projections fail.
- Each projection result is wrapped in `SectionResult<T>` with `status: OK | ERROR | EMPTY` and, on error, a structured code + correlation id.
- UI renders per-card states; a "Retry" action calls the per-section endpoint rather than the aggregate.

---

## 3. Mutation Flows

### 3.1 Milestone transition (e.g., IN_PROGRESS → AT_RISK)

```mermaid
sequenceDiagram
    autonumber
    participant U as PM
    participant UI as MilestonePlanner
    participant Store as Pinia Store
    participant Api as API Client
    participant Ctrl as Controller
    participant Guard as PlanAccessGuard
    participant Svc as MilestoneService
    participant Pol as PlanPolicy
    participant Evt as ChangeEventPublisher
    participant Audit as Audit Pipeline
    participant Out as Outbox
    participant DB as DB

    U->>UI: Select transition with reason (≥10 chars)
    UI->>Store: transitionMilestone(payload)
    Store->>Store: optimistic update + lock row
    Store->>Api: POST .../milestones/{id}/transition
    Api->>Ctrl: HTTP
    Ctrl->>Guard: authorize(WRITE)
    Guard-->>Ctrl: ok
    Ctrl->>Svc: transition(projectId, milestoneId, to, reason)
    Svc->>Pol: allowed?(from→to) + reason check
    Pol-->>Svc: ok
    Svc->>DB: BEGIN TX
    Svc->>DB: UPDATE milestone SET status, slippage_reason, plan_revision++
    Svc->>Evt: publish(MilestoneChanged(before, after))
    Evt->>DB: INSERT plan_change_log
    Evt->>Out: enqueue OutboxEvent
    Svc->>DB: COMMIT
    Evt->>Audit: record event (after commit)
    Svc-->>Ctrl: MilestoneDto
    Ctrl-->>Api: 200 MilestoneDto + audit link
    Api-->>Store: replace entity
    Store->>Store: release lock
    Store-->>UI: render + toast
```

**Failure branches**:

- `PlanPolicy` rejects → Service raises `InvalidTransition` → Controller returns 409 with `{ code: "PM_INVALID_TRANSITION", from, to }` → Store rolls back optimistic update → UI toast "Transition not allowed".
- `Guard` denies → 403 `{ code: "PM_AUTH_FORBIDDEN" }` → UI toast "You do not have permission".
- `DB` constraint violation → 422 → UI toast with validation errors.
- Missing or too-short reason → 422 `{ code: "PM_SLIPPAGE_REASON_REQUIRED" }`.

### 3.2 Capacity cell batch update

```mermaid
sequenceDiagram
    autonumber
    participant U as PM
    participant UI as CapacityAllocationEditor
    participant Store as PM Store
    participant Api as API Client
    participant Ctrl as Controller
    participant Svc as CapacityService
    participant Pol as PlanPolicy
    participant Evt as ChangeEventPublisher
    participant DB as DB

    U->>UI: Edit cells (memberA × M2 = 60, memberA × M3 = 50)
    UI->>Store: queueCapacityEdits(batch)
    Store->>Store: debounce 400ms
    Store->>Api: PATCH .../capacity (batch)
    Api->>Ctrl: HTTP
    Ctrl->>Svc: batchUpdate(projectId, edits)
    Svc->>Pol: validate(overallocation rule, justification)
    alt row total > 100 without justification
      Pol-->>Svc: reject
      Svc-->>Ctrl: 422 PM_OVERALLOCATION_JUSTIFICATION_REQUIRED
      Ctrl-->>Api: 422
      Api-->>Store: error
      Store-->>UI: inline banner + revert cells
    else ok (or justified)
      Pol-->>Svc: ok
      Svc->>DB: BEGIN TX
      Svc->>DB: UPSERT capacity_allocation rows
      Svc->>Evt: publish(CapacityChanged(before, after))
      Evt->>DB: INSERT plan_change_log x N
      Svc->>DB: COMMIT
      Svc-->>Ctrl: CapacityMatrixDto
      Ctrl-->>Api: 200
      Api-->>Store: replace matrix
      Store-->>UI: render
    end
```

### 3.3 Risk escalation

```mermaid
sequenceDiagram
    autonumber
    participant U as PM
    participant UI as RiskRegistryEditor
    participant Store as PM Store
    participant Api as API Client
    participant Svc as RiskService
    participant ApprovalSvc as Approval Pipeline
    participant Evt as ChangeEventPublisher
    participant DB as DB
    participant Notif as Notification

    U->>UI: Escalate risk
    UI->>Store: escalateRisk(riskId, reason)
    Store->>Api: POST .../risks/{id}/transition (to=ESCALATED)
    Api->>Svc: escalate
    Svc->>DB: UPDATE risk state=ESCALATED
    Svc->>ApprovalSvc: createApproval(target=RISK, riskId, approver=AppOwner)
    ApprovalSvc->>DB: INSERT approval
    ApprovalSvc-->>Svc: approvalId
    Svc->>DB: UPDATE risk.escalation_approval_id
    Svc->>Evt: publish(RiskEscalated)
    Evt->>DB: INSERT plan_change_log
    Svc->>Notif: enqueue AppOwner notification
    Svc-->>Api: 200 RiskDto with approvalId
    Api-->>Store: replace
    Store-->>UI: render escalated state + link to approval
```

### 3.4 Dependency approval with counter-signature

```mermaid
sequenceDiagram
    autonumber
    participant PM_A as PM (source project)
    participant UI_A as DependencyResolver (source)
    participant BE as DependencyService
    participant PM_B as PM (target project)
    participant UI_B as DependencyResolver (target)
    participant DB as DB

    PM_A->>UI_A: Propose dependency to target
    UI_A->>BE: POST .../dependencies (state=PROPOSED)
    BE->>DB: INSERT dependency
    BE-->>UI_A: 201 DependencyDto
    PM_A->>UI_A: Negotiate (edit, chat, attach contract notes)
    UI_A->>BE: PATCH .../dependencies/{id}
    BE->>DB: UPDATE
    PM_A->>UI_A: Request approval (state → NEGOTIATING)
    UI_A->>BE: POST .../dependencies/{id}/transition NEGOTIATING
    BE->>DB: UPDATE
    Note over PM_B,UI_B: Target-side PM sees pending counter-signature in target project's Plan view
    PM_B->>UI_B: Open target Plan view → deps
    PM_B->>UI_B: Counter-sign dependency
    UI_B->>BE: POST .../dependencies/{id}/countersign
    BE->>DB: UPDATE counter_signature_member_id
    BE->>DB: UPDATE state=APPROVED
    BE-->>UI_B: 200
    BE-->>UI_A: Webhook/outbox → refresh source view
```

If the target is external (not a registered internal project), `countersign` is skipped and `APPROVED` instead requires a `contractCommitment` string ≥ 20 chars.

### 3.5 AI suggestion accept / dismiss

```mermaid
sequenceDiagram
    autonumber
    participant Skill as AI Skill Runtime
    participant BE as AiSuggestionService
    participant PV as Plan View
    participant Store as PM Store
    participant U as PM
    participant DB as DB
    participant Audit as Audit Pipeline

    Skill->>BE: POST /internal/ai-suggestions (kind, target, payload, confidence)
    BE->>DB: UPSERT ai_suggestion (PENDING, skill_execution_id)
    PV->>Store: periodic refresh (onFocus or refresh button)
    Store->>BE: GET .../plan/{projectId}/ai-suggestions
    BE-->>Store: list of PENDING
    U->>PV: Accept suggestion-X
    PV->>Store: accept(suggestionId)
    Store->>BE: POST .../ai-suggestions/{id}/accept
    BE->>DB: UPDATE ai_suggestion state=ACCEPTED
    BE->>DB: INSERT plan_change_log (AI actor)
    BE->>Audit: record skill-outcome ACCEPTED
    BE-->>Store: 200 with audit link
    Store-->>PV: mark accepted; hide chip
    U->>PV: Dismiss suggestion-Y
    PV->>Store: dismiss(suggestionId, reason)
    Store->>BE: POST .../ai-suggestions/{id}/dismiss
    BE->>DB: UPDATE state=DISMISSED; set suppression_until=now+24h
    BE->>DB: INSERT plan_change_log
    BE->>Audit: record skill-outcome DISMISSED
    BE-->>Store: 200
    Store-->>PV: hide chip; add to log
```

---

## 4. State Machines

### 4.1 Milestone

```mermaid
stateDiagram-v2
    [*] --> NOT_STARTED
    NOT_STARTED --> IN_PROGRESS
    IN_PROGRESS --> COMPLETED
    IN_PROGRESS --> AT_RISK
    AT_RISK --> IN_PROGRESS
    AT_RISK --> SLIPPED
    SLIPPED --> IN_PROGRESS : with new target
    NOT_STARTED --> ARCHIVED
    IN_PROGRESS --> ARCHIVED
    AT_RISK --> ARCHIVED
    SLIPPED --> ARCHIVED
    COMPLETED --> ARCHIVED
    ARCHIVED --> [*]
```

**Guards**:
- `IN_PROGRESS → AT_RISK`: `slippageReason` required, min 10 chars.
- `AT_RISK → SLIPPED`: `slippageReason` required (may be updated).
- `SLIPPED → IN_PROGRESS`: new `targetDate` required, ≥ today.
- `IN_PROGRESS → COMPLETED`: server stamps `completedAt`.
- Re-opening `COMPLETED` is **not allowed** in V1; archive and recreate instead.

### 4.2 Risk

```mermaid
stateDiagram-v2
    [*] --> IDENTIFIED
    IDENTIFIED --> ACKNOWLEDGED
    ACKNOWLEDGED --> MITIGATING
    MITIGATING --> RESOLVED
    IDENTIFIED --> ESCALATED
    ACKNOWLEDGED --> ESCALATED
    MITIGATING --> ESCALATED
    ESCALATED --> ACKNOWLEDGED : after approval
    RESOLVED --> [*]
```

**Guards**:
- `ACKNOWLEDGED → MITIGATING`: `mitigationNote` required (≥ 20 chars).
- `MITIGATING → RESOLVED`: `resolutionNote` required.
- `→ ESCALATED`: creates a pending Approval; `escalationApprovalId` captured.

### 4.3 Dependency resolution

```mermaid
stateDiagram-v2
    [*] --> PROPOSED
    PROPOSED --> NEGOTIATING
    NEGOTIATING --> APPROVED
    NEGOTIATING --> REJECTED
    NEGOTIATING --> AT_RISK
    AT_RISK --> NEGOTIATING
    AT_RISK --> RESOLVED
    APPROVED --> RESOLVED
    REJECTED --> [*]
    RESOLVED --> [*]
```

**Guards**:
- Internal `APPROVED`: requires counter-signature from a user with write role on the target project.
- External `APPROVED`: requires `contractCommitment` ≥ 20 chars.
- `REJECTED`: requires `rejectionReason` ≥ 10 chars.

### 4.4 AI suggestion

```mermaid
stateDiagram-v2
    [*] --> PENDING
    PENDING --> ACCEPTED
    PENDING --> DISMISSED
    ACCEPTED --> [*]
    DISMISSED --> [*]
```

Dismissal sets `suppressionUntil = now + 24h`; the skill runtime must check this before emitting a new suggestion for the same `(targetType, targetId, kind)`.

---

## 5. Refresh Strategy

V1 is on-load + manual refresh, matching Project Space conventions.

```mermaid
flowchart LR
    InitLoad[Initial load on mount] --> Paint[Paint cards]
    Paint --> Idle[Idle]
    Idle -->|manual refresh click| Reload[Re-fetch aggregate]
    Idle -->|tab regains focus + &gt;5min| AutoReload[Background refresh]
    Idle -->|mutation success| PartialRefresh[Refresh affected card only]
    Reload --> Paint
    AutoReload --> Paint
    PartialRefresh --> Paint
```

- A "last refreshed" timestamp is displayed on the Portfolio Summary Bar and the Plan Header.
- A "stale" banner appears when `now − lastRefreshed > 5 min` on the Portfolio view.
- Mutations refresh **only the affected card** and append to the Change Log card.
- No WebSocket push in V1. The backend emits outbox events for future consumers (Project Space projection refresh, Dashboard counters).

---

## 6. API Client Chain

```mermaid
flowchart LR
    ViewComp[Vue Component] --> Store[Pinia Store action]
    Store --> ApiClient[projectManagementApi.ts]
    ApiClient --> Fetch[fetchJson&lt;T&gt;]
    Fetch --> HTTP[HTTP + ApiResponse&lt;T&gt;]
    HTTP --> Backend[Backend controller]

    ApiClient -->|VITE_USE_MOCK=true| MockFixtures[mock fixtures]
    MockFixtures --> Store
```

- `fetchJson<T>` unwraps the `ApiResponse<T>` envelope, returning either the payload or throwing a typed error.
- A Vite env flag (`VITE_USE_MOCK=true`) swaps the real HTTP client for mock fixtures — used through all of Phase A and during offline development.

---

## 7. Mutation Concurrency & Optimistic UI

- Every mutation takes a `planRevision` fencing token on the relevant entity (e.g., `milestone.plan_revision`).
- If the client sends a stale `planRevision`, the server returns 409 `PM_STALE_REVISION` with the latest revision; the client reloads the card and re-asks the user.
- Optimistic UI updates are applied locally before the server round-trips; rollback on non-2xx.
- Batch capacity edits are debounced 400ms on the client; the server treats the batch atomically (all or nothing).

---

## 8. Access Isolation at Runtime

```mermaid
flowchart TB
    Req[Incoming HTTP request with auth] --> Caller[Resolve caller identity]
    Caller --> WS{workspaceId on request}
    WS -->|portfolio| CheckWSRole[Caller role in Workspace]
    WS -->|plan| ResolveProj[Load Project]
    ResolveProj --> CheckProjRole[Caller role in Project + parent Workspace]
    CheckWSRole -->|authorized| Continue[Proceed]
    CheckProjRole -->|authorized| Continue
    CheckWSRole -->|unauthorized| Deny[403 PM_AUTH_FORBIDDEN]
    CheckProjRole -->|unauthorized| Deny
    Continue --> Svc[Service]
    Svc -->|write gating| WriteCheck{caller in PM / Workspace Admin / Project Admin?}
    WriteCheck -->|yes| Execute[Execute]
    WriteCheck -->|no| Deny
```

Read authority: Auditor, Project Contributor, PM, Tech Lead, Workspace Admin, Application Owner.
Write authority: Project Admin (includes PM), Workspace Admin, Application Owner (limited to approvals).

---

## 9. Plan Change Log Emission Rules

For every mutation:

1. Inside the database transaction, `PlanChangeEventPublisher` inserts a row into `plan_change_log` with:
   - `actorType` (`HUMAN` or `AI`), `actorId`, `skillExecutionId` (nullable)
   - `action` (`CREATE`, `UPDATE`, `TRANSITION`, `ARCHIVE`, `ACCEPT_AI_SUGGESTION`, `DISMISS_AI_SUGGESTION`, `ESCALATE`, `COUNTERSIGN`)
   - `targetType` (`MILESTONE`, `RISK`, `DEPENDENCY`, `CAPACITY_ALLOCATION`, `AI_SUGGESTION`)
   - `targetId`, `projectId`
   - `before` and `after` JSON (before is null for creates, after is null for archives)
   - `at`, `correlationId`, `auditLinkId`
2. After transaction commit, an outbox event is emitted to the shared platform audit pipeline.
3. The Plan Change Log card subscribes to `projectId` and reads paginated entries.

---

## 10. Caching & Invalidation

- Frontend: Pinia store caches aggregate responses per `projectId` or `workspaceId` with a soft TTL of 5 min; a successful mutation invalidates only the affected card cache.
- Backend: per-projection in-memory cache (Caffeine) with 30s TTL for the Portfolio aggregate under heavy read; Plan aggregate is not cached (edit-heavy).
- Outbox event consumers invalidate downstream caches (Project Space, Dashboard) on best-effort basis.

---

## 11. Telemetry

- Every aggregate endpoint logs: `workspaceId` or `projectId`, caller role, projection timings per section, total latency, section failure count.
- Every mutation logs: actor, target, before→after, correlation id, duration.
- Every AI suggestion logs: skill id, confidence, accept/dismiss outcome latency.
- Structured JSON logs for correlation with the platform audit.

---

## 12. Summary

| Flow | Endpoint(s) | State machine | Audit entry kind |
|------|-------------|---------------|------------------|
| Portfolio aggregate | `GET /portfolio`, `GET /portfolio/{section}` | — | — |
| Plan aggregate | `GET /plan/{projectId}`, `GET /plan/{projectId}/{section}` | — | — |
| Milestone CRUD + transition + archive | `POST/PATCH/POST/POST .../milestones*` | Milestone SM | `CREATE`, `UPDATE`, `TRANSITION`, `ARCHIVE` |
| Capacity batch update | `PATCH .../capacity` | — | `UPDATE` per cell |
| Risk CRUD + transition + escalate | `POST/PATCH/POST .../risks*` | Risk SM | `CREATE`, `UPDATE`, `TRANSITION`, `ESCALATE` |
| Dependency CRUD + transition + countersign | `POST/PATCH/POST/POST .../dependencies*` | Dependency SM | `CREATE`, `UPDATE`, `TRANSITION`, `COUNTERSIGN` |
| AI suggestion accept / dismiss | `POST .../ai-suggestions/{id}/{accept\|dismiss}` | AI SM | `ACCEPT_AI_SUGGESTION`, `DISMISS_AI_SUGGESTION` |
