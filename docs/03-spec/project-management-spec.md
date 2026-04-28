# Feature Specification: Project Management

| Attribute | Value |
|-----------|-------|
| Slice | project-management |
| Source Requirements | [project-management-requirements.md](../01-requirements/project-management-requirements.md) |
| Source Stories | [project-management-stories.md](../02-user-stories/project-management-stories.md) |
| PRD | [agentic_sdlc_control_tower_prd_v0.9.md](../01-requirements/agentic_sdlc_control_tower_prd_v0.9.md) §11.5 |

## Overview

Project Management is the **delivery operating plane** of the Agentic SDLC Control Tower. It is the primary page where a Project Manager edits the plan, allocates capacity, manages risk ownership, and resolves dependencies. Unlike Project Space (read-heavy, per-project view), Project Management is write-heavy and covers both a cross-project Portfolio view and a per-project Plan view.

The slice is scoped to:

- **Rhythm control** — milestones, delivery progress, plan stability
- **Resource control** — member × milestone capacity allocation
- **Risk control** — risk registry lifecycle and AI-assisted mitigation
- **Dependency control** — internal / external dependency approval workflow
- **Transparency** — Portfolio view for PMO, Delivery Manager, Application Owner

It is **not** scoped to project creation / archiving (Platform Center), role permissions (Access Management), lifecycle artifact editing (Requirement / Design / Code / Test / Deploy / Incident), or historical reporting (Report Center).

## Source Stories

| Story | Description |
|-------|-------------|
| S1 | View Portfolio Summary Bar |
| S2 | Scan the Portfolio Milestone Heatmap |
| S3 | Review Cross-Project Capacity & Allocation |
| S4 | Identify Risk Concentration |
| S5 | Detect Dependency Bottlenecks |
| S6 | Inspect Portfolio Delivery Cadence |
| S7 | Open the Plan for a Single Project |
| S8 | Create, Edit, and Reorder Milestones |
| S9 | Move a Milestone Through Its Lifecycle |
| S10 | Review AI Slippage Predictions on Milestones |
| S11 | Edit Member × Milestone Capacity Allocation |
| S12 | Surface Backup Coverage Gaps |
| S13 | Accept or Reject AI Capacity Rebalance Suggestions |
| S14 | Run the Risk Registry |
| S15 | Accept or Edit an AI Mitigation Draft |
| S16 | Resolve Dependencies Through an Approval Workflow |
| S17 | Review AI Dependency Resolution Proposals |
| S18 | Read the Project Delivery Progress Strip |
| S19 | Review the Plan Change Log |
| S20 | Deep-Link into Plan View from Multiple Entry Points |
| S21 | Interact With the Project Management AI Command Panel |
| S22 | Enforce Role-Based Write Authority |
| S23 | Preserve Plan State Across Drill-Outs |
| S24 | Render Loading / Empty / Error States Gracefully |

## Actors / Users

| Actor | Primary Role | Access |
|-------|--------------|--------|
| Project Manager (PM) | Plan view editor — milestones, capacity, risks, deps | Write within the project |
| Delivery Manager / PMO | Portfolio view consumer — slippage, capacity, escalation | Read across the Workspace |
| Application Owner | Escalation approver; portfolio reader across the Application | Read + approve at Workspace level |
| Architect / Tech Lead | Contribute risks and dependency proposals; read plan | Read + propose |
| QA Lead / SRE | Contribute risks and dependency proposals; read plan | Read + propose |
| Auditor | Compliance audit trail consumer | Read-only including change log |
| AI Command Panel | Consumes page context for slippage, mitigation, rebalance suggestions | System actor |

## Functional Scope

### Portfolio View (Workspace-scoped)

1. **Portfolio Summary Bar** (REQ-PM-10, REQ-PM-11, REQ-PM-12)
2. **Portfolio Milestone Heatmap** (REQ-PM-20, REQ-PM-21, REQ-PM-22, REQ-PM-23)
3. **Portfolio Capacity & Allocation matrix** (REQ-PM-30, REQ-PM-31, REQ-PM-32, REQ-PM-33)
4. **Portfolio Risk Concentration** (REQ-PM-40, REQ-PM-41, REQ-PM-42)
5. **Portfolio Dependency Bottlenecks** (REQ-PM-50, REQ-PM-51)
6. **Portfolio Delivery Cadence metrics** (REQ-PM-60, REQ-PM-61)

### Plan View (Project-scoped)

7. **Plan Header** (REQ-PM-70, REQ-PM-71)
8. **Milestone Planner** (REQ-PM-80–REQ-PM-84)
9. **Capacity Allocation editor** (REQ-PM-90–REQ-PM-94)
10. **Risk Registry Editor** (REQ-PM-100–REQ-PM-103)
11. **Dependency Resolver** (REQ-PM-110–REQ-PM-113)
12. **Delivery Progress Strip** (REQ-PM-120, REQ-PM-121, REQ-PM-122)
13. **Plan Change Log** (REQ-PM-130, REQ-PM-131, REQ-PM-132)

### Cross-Cutting

14. **Navigation, context, breadcrumbs** (REQ-PM-140–REQ-PM-144)
15. **AI Command Panel projection** (REQ-PM-150–REQ-PM-152)
16. **Governance & audit** (REQ-PM-160–REQ-PM-162)
17. **Isolation** (REQ-PM-170–REQ-PM-172)
18. **Visual & experience** (REQ-PM-180–REQ-PM-185)

## Routing

| Path | View | Purpose |
|------|------|---------|
| `/project-management` | Portfolio | Workspace-scoped aggregation; reads `workspaceId` from context bar |
| `/project-management?riskSeverity=...&ownerMemberId=...&status=...` | Portfolio (filtered) | Query params preserve filters from deep-links |
| `/project-management/:projectId` | Plan | Single-project editor |
| `/project-management/:projectId?milestoneId=...` | Plan (scrolled) | Plan view scrolled to a specific milestone |

All paths render inside the shared app shell and participate in the breadcrumb pipeline (REQ-PM-142).

## Contracts

The full API contract lives in [../05-design/contracts/project-management-API_IMPLEMENTATION_GUIDE.md](../05-design/contracts/project-management-API_IMPLEMENTATION_GUIDE.md). This section summarizes:

### Portfolio view endpoints (all read)

| Method | Path | Purpose |
|--------|------|---------|
| GET | `/api/v1/project-management/portfolio/summary?workspaceId=...` | Summary Bar counters |
| GET | `/api/v1/project-management/portfolio/heatmap?workspaceId=...&window=week\|month\|milestone` | Milestone heatmap matrix |
| GET | `/api/v1/project-management/portfolio/capacity?workspaceId=...` | Member × project allocation matrix |
| GET | `/api/v1/project-management/portfolio/risks?workspaceId=...&limit=20` | Top risks list + severity × category heatmap |
| GET | `/api/v1/project-management/portfolio/dependencies?workspaceId=...&limit=15` | Top bottlenecks |
| GET | `/api/v1/project-management/portfolio/cadence?workspaceId=...` | Throughput, cycle-time, hit-rate, stability metrics |
| GET | `/api/v1/project-management/portfolio?workspaceId=...` | Aggregate first-paint (all six sections) |

### Plan view read endpoints (project-scoped)

| Method | Path | Purpose |
|--------|------|---------|
| GET | `/api/v1/project-management/plan/{projectId}` | Plan view aggregate first-paint |
| GET | `/api/v1/project-management/plan/{projectId}/header` | Plan Header data |
| GET | `/api/v1/project-management/plan/{projectId}/milestones` | Milestone list |
| GET | `/api/v1/project-management/plan/{projectId}/capacity` | Capacity matrix |
| GET | `/api/v1/project-management/plan/{projectId}/risks` | Risk registry list |
| GET | `/api/v1/project-management/plan/{projectId}/dependencies` | Dependency list |
| GET | `/api/v1/project-management/plan/{projectId}/progress` | 11-node SDLC chain health |
| GET | `/api/v1/project-management/plan/{projectId}/change-log?filters=...&page=...` | Paginated change log |
| GET | `/api/v1/project-management/plan/{projectId}/ai-suggestions` | Active AI suggestions bundle (slippage, rebalance, mitigation, dependency proposals) |

### Plan view mutation endpoints (project-scoped)

| Method | Path | Purpose |
|--------|------|---------|
| POST | `/api/v1/project-management/plan/{projectId}/milestones` | Create milestone |
| PATCH | `/api/v1/project-management/plan/{projectId}/milestones/{milestoneId}` | Update milestone fields |
| POST | `/api/v1/project-management/plan/{projectId}/milestones/{milestoneId}/transition` | Advance milestone state |
| POST | `/api/v1/project-management/plan/{projectId}/milestones/{milestoneId}/archive` | Archive milestone |
| PATCH | `/api/v1/project-management/plan/{projectId}/capacity` | Update capacity cells (batch) |
| POST | `/api/v1/project-management/plan/{projectId}/risks` | Create risk |
| PATCH | `/api/v1/project-management/plan/{projectId}/risks/{riskId}` | Update risk fields |
| POST | `/api/v1/project-management/plan/{projectId}/risks/{riskId}/transition` | Advance risk state |
| POST | `/api/v1/project-management/plan/{projectId}/dependencies` | Create dependency |
| PATCH | `/api/v1/project-management/plan/{projectId}/dependencies/{depId}` | Update dependency fields |
| POST | `/api/v1/project-management/plan/{projectId}/dependencies/{depId}/transition` | Advance dependency state |
| POST | `/api/v1/project-management/plan/{projectId}/dependencies/{depId}/countersign` | Counter-signature for internal approvals |
| POST | `/api/v1/project-management/plan/{projectId}/ai-suggestions/{suggestionId}/accept` | Accept an AI suggestion |
| POST | `/api/v1/project-management/plan/{projectId}/ai-suggestions/{suggestionId}/dismiss` | Dismiss an AI suggestion |

All mutation endpoints emit an audit event via the shared audit pipeline. All responses use the shared `ApiResponse<T>` envelope.

## Data Model Summary

The slice **extends** the existing Project / Milestone / Risk / Dependency / Environment / Member models defined in [project-space-data-model.md](../04-architecture/project-space-data-model.md) and [team-space-data-model.md](../04-architecture/team-space-data-model.md). Extensions:

- **Milestone**: adds `slippageReason`, `archivedAt`, `slippageRiskScore`, `slippageRiskFactors`, `planRevision`
- **Risk**: adds `state` (IDENTIFIED/ACKNOWLEDGED/MITIGATING/RESOLVED/ESCALATED), `mitigationNote`, `resolutionNote`, `escalationApprovalId`
- **Dependency**: adds `resolutionState` (PROPOSED/NEGOTIATING/APPROVED/REJECTED/AT_RISK/RESOLVED), `counterSignatureMemberId`, `contractCommitment`, `rejectionReason`
- **CapacityAllocation (NEW)**: `id`, `projectId`, `memberId`, `milestoneId`, `percent`, `justification`, `windowStart`, `windowEnd`
- **PlanChangeLogEntry (NEW)**: `id`, `projectId`, `actorType` (HUMAN/AI), `actorId`, `action`, `targetType`, `targetId`, `before`, `after`, `at`, `auditLinkId`
- **AiSuggestion (NEW)**: `id`, `projectId`, `kind` (SLIPPAGE/REBALANCE/MITIGATION/DEP_RESOLUTION), `targetType`, `targetId`, `payload`, `confidence`, `state` (PENDING/ACCEPTED/DISMISSED), `skillExecutionId`, `createdAt`

Full ER diagram, DTOs, and DDL live in [project-management-data-model.md](../04-architecture/project-management-data-model.md).

## State Machines

### Milestone state machine

```
NOT_STARTED → IN_PROGRESS → { COMPLETED | AT_RISK }
AT_RISK → { IN_PROGRESS (recovered) | SLIPPED }
SLIPPED → IN_PROGRESS (re-planned, new target)
Any → ARCHIVED (soft delete)
```

Invalid transitions return HTTP 409. `AT_RISK` and `SLIPPED` require `slippageReason`.

### Risk state machine

```
IDENTIFIED → ACKNOWLEDGED → MITIGATING → RESOLVED
Any → ESCALATED (creates a pending Approval routed to Application Owner)
```

`MITIGATING` requires `mitigationNote`; `RESOLVED` requires `resolutionNote`.

### Dependency resolution state machine

```
PROPOSED → NEGOTIATING → { APPROVED (requires counter-signature for internal) | REJECTED | AT_RISK }
AT_RISK → { NEGOTIATING | RESOLVED }
APPROVED → RESOLVED (upon delivery)
```

### AI suggestion state machine

```
PENDING → { ACCEPTED | DISMISSED }
```

Suppression window: a `DISMISSED` suggestion suppresses new suggestions for the same (targetType, targetId) for 24h.

## Performance Budgets

| Scope | First Paint | Full Hydration |
|-------|-------------|----------------|
| Portfolio view | ≤ 400 ms (from workspaceId receipt) | ≤ 800 ms (all cards) |
| Plan view | ≤ 300 ms (from projectId receipt) | ≤ 500 ms (all cards) |
| Single-field mutation | — | ≤ 500 ms p95 local / 1 s p95 staging |

Plan view targets assume project volumes ≤ 30 milestones, ≤ 50 dependencies, ≤ 100 active risks, ≤ 10 environments (same envelope as Project Space REQ-PS-132).

## Security & Authorization

- All writes require authentication and role authorization as per REQ-PM-161
- Authorization failures return HTTP 403 with a structured error code (`PM_AUTH_FORBIDDEN`)
- Input validation failures return HTTP 422 with field-level errors
- State-machine violations return HTTP 409 with the invalid transition named
- The backend is the source of truth; the frontend hides unauthorized affordances for UX but does not rely on hiding for security

## Isolation

- Portfolio view: strict Workspace scoping — projects from other Workspaces must not appear
- Plan view: strict Project scoping — cross-project reads allowed only via explicit drill-out links
- Parent-Workspace auto-resolution when context bar mismatches the Plan-view project's workspaceId

## Audit

Every mutation produces an audit event with:

- Actor type (HUMAN / AI), actor id, skill id (if AI)
- Action verb
- Target type (Milestone / Risk / Dependency / CapacityAllocation / AiSuggestion)
- Target id
- Before state (JSON diff or null for creates)
- After state (JSON diff or null for deletes)
- Timestamp, correlation id
- Evidence references (for AI actions)

Audit events are mirrored into the `plan_change_log` for UI display; the canonical audit store is platform-wide and retained per compliance policy.

## Out of Scope

- Project CRUD and renaming (Platform Center)
- Role / permission grants (Access Management)
- Lifecycle artifact editing (Requirement / Design / Code / Test / Deploy / Incident)
- Historical reporting / export (Report Center)
- WebSocket real-time push (V1 is on-load + manual refresh)
- Multi-Workspace rollup (V2 candidate; V1 is single Workspace)
- Multi-project critical-path scheduling / Gantt (V2)
- Time tracking / cost modeling (V2)

## Acceptance

All 24 stories from [project-management-stories.md](../02-user-stories/project-management-stories.md) are accepted when:

- Their acceptance criteria are met end-to-end
- All referenced REQ-PM-* requirements are satisfied
- Performance budgets hold under typical volumes
- Audit events are recorded for every mutation and AI outcome
- Build passes (`npm run dev`, `npm run build`, `./mvnw verify`)
- Flyway migrations are applied cleanly from an empty H2 and from the existing project-space schema
