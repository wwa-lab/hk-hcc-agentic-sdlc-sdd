# System Architecture: Incident Management

## Overview
- **Architecture Summary**: The Incident Management module is an AI-native operational command center that inverts the traditional incident lifecycle — AI is the primary executor (detection, diagnosis, remediation) while humans govern (approve, escalate, override). The module plugs into the existing shared app shell and follows the same layered architecture as the Dashboard module: Vue 3 frontend with Pinia stores, Spring Boot REST API, and relational persistence.
- **Design Objective**: Provide a high-density, auditable incident command surface where AI skill executions, governance decisions, and SDLC chain traceability are first-class concerns — not afterthoughts.
- **Architectural Style**: Layered service architecture within a modular monolith (package-by-feature), consistent with the existing Dashboard domain.

---

## Source Specification
- **Feature / System Name**: Incident Management
- **Scope Summary**: Incident list with triage, detail view with AI diagnosis feed, skill execution timeline, action governance (approve/reject), human governance audit trail, SDLC chain traceability, and AI learning section. Phase A delivers frontend with mocked data; Phase B adds backend API.

---

## Architectural Drivers

### Key Functional Drivers
- AI-first incident lifecycle with explicit state machine (Detected → ... → Closed)
- Per-incident skill execution timeline (detection, correlation, diagnosis, remediation, learning)
- Action governance with approve/reject controls and policy-triggered approval gates
- SDLC chain traceability linking incidents to upstream artifacts (Requirement → Spec → Code → Deploy)
- Independent card-level loading and error isolation (SectionResult pattern)

### Key Non-Functional Drivers
- Auditability: every governance action (approve, reject, escalate, override) must be recorded with actor, timestamp, and reason
- Workspace isolation: all incident data scoped to current workspace context
- Consistency: follows the same API envelope, design tokens, and frontend patterns established by Dashboard

### Constraints and Assumptions
- Frontend framework: Vue 3 / Vite / Vue Router / Pinia / TypeScript (project standard)
- Backend framework: Spring Boot 3.x / Java 21 / JPA (project standard)
- Database: H2 (local dev), Oracle (production) via Flyway migrations
- API envelope: `ApiResponse<T>` with `data` and `error` fields (verified: `shared/dto/ApiResponse.java`)
- Per-card error isolation: `SectionResult<T>` pattern (verified: `dashboard/types/dashboard.ts` and `dashboard/dto/SectionResultDto.java`)
- [ASSUMPTION] V1 uses on-load fetch for data; no WebSocket or server-sent events
- [ASSUMPTION] Incident count per workspace is small enough (<50 active) that pagination is not required in V1

---

## System Context

### Primary Actors
| Actor | Role |
|---|---|
| SRE / DevOps Engineer | Triages incidents, follows AI diagnosis, approves/rejects AI-proposed actions |
| Team Lead / Delivery Manager | Reviews incident impact, governance posture, SDLC chain links |
| Platform Admin / Auditor | Audits governance decisions, reviews AI execution history |

### External Systems
| System | Integration Purpose |
|---|---|
| Shared App Shell | Navigation, context bar, AI Command Panel — hosts the incident page |
| Dashboard | Stability card links to `/incidents` for drill-down |
| SDLC Module Pages | Incident SDLC chain links navigate to module pages (may show placeholder) |
| AI Center (future) | Skill registry and execution history |
| ServiceNow / PagerDuty (future) | Bi-directional incident sync — out of scope for V1 |

### System Boundary
The Incident Management module owns all incident-specific UI components, state management, API endpoints, domain services, and persistence. It depends on the shared app shell for navigation and workspace context. It reads SDLC chain links but does not own the linked artifacts — those belong to their respective domain modules. External monitoring/alerting integrations are outside V1 scope.

---

## High-Level Architecture

```
┌──────────────────────────────────────────────────────────────────┐
│  Users                                                           │
│  SRE · Team Lead · Platform Admin · Auditor                      │
└──────────────────────────┬───────────────────────────────────────┘
                           │ HTTPS
                           ▼
┌──────────────────────────────────────────────────────────────────┐
│  Vue 3 Frontend (within Shared App Shell)                        │
│  ┌──────────────┐  ┌──────────────┐  ┌───────────────────────┐  │
│  │ Incident List │  │Incident Detail│  │ Pinia Store + API     │  │
│  │ View         │  │View (7 cards) │  │ Client                │  │
│  └──────────────┘  └──────────────┘  └───────────────────────┘  │
└──────────────────────────┬───────────────────────────────────────┘
                           │ REST / JSON
                           ▼
┌──────────────────────────────────────────────────────────────────┐
│  Spring Boot API Layer                                           │
│  ┌───────────────────────────────────────────────────────────┐   │
│  │  Incident Controller (REST endpoints)                     │   │
│  └───────────────────────────────────────────────────────────┘   │
├──────────────────────────────────────────────────────────────────┤
│  Domain Layer                                                    │
│  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────┐   │
│  │ Incident Domain   │  │ Skill Execution  │  │ Governance    │   │
│  │ (lifecycle, state │  │ (timeline,       │  │ (approval,    │   │
│  │  machine, triage) │  │  audit trail)    │  │  escalation)  │   │
│  └──────────────────┘  └──────────────────┘  └──────────────┘   │
├──────────────────────────────────────────────────────────────────┤
│  Shared Infrastructure                                           │
│  ApiResponse<T> · SectionResultDto<T> · ApiConstants             │
├──────────────────────────┬───────────────────────────────────────┤
│  Persistence (JPA/H2/Oracle)                                     │
└──────────────────────────┴───────────────────────────────────────┘
```

### Layer Summary

The module is organized into four primary layers:

- **Presentation Layer** — Vue 3 components split into an incident list view and a detail view with 7 independent cards (header, diagnosis, timeline, actions, governance, SDLC chain, learning). Pinia store manages state; API client handles REST communication.
- **API Layer** — Spring Boot REST controller exposing incident list and detail endpoints. Uses the shared `ApiResponse<T>` envelope.
- **Domain Layer** — Three capability groups: Incident Domain (lifecycle state machine, triage logic), Skill Execution (timeline tracking), and Governance (approval workflow, audit trail). All state is represented as immutable Java records (DTOs).
- **Persistence Layer** — JPA entities with Flyway-managed schema. H2 for local dev, Oracle for production. Phase A seeds mock data via Flyway migration; Phase B adds real data access.

---

## Component Breakdown

### Frontend Components

- **Incident List View**: Table of incidents with filtering (priority, status, handler type, date range), sorting, severity distribution summary, and row-click navigation to detail
- **Incident Detail View**: Composite view hosting 7 independent cards, each using `SectionResult<T>` for error isolation
- **Incident Header Card**: ID, title, priority, status badge, handler type indicator, autonomy level, control mode, timestamps
- **AI Diagnosis Card**: Chronological log-style feed with timestamped entries (analysis / finding / suggestion / conclusion), root cause hypothesis with confidence
- **Skill Timeline Card**: Ordered list of skill executions (detection → correlation → diagnosis → remediation → learning) with status indicators
- **AI Actions Card**: List of actions taken/proposed with approve/reject controls, impact assessment, rollback indicator
- **Governance Card**: Historical audit trail of approve/reject/escalate/override decisions with actor and reason
- **SDLC Chain Card**: Compressed chain visualization with Spec always visible, expand control, navigation links
- **AI Learning Card**: Root cause, pattern, prevention recommendations — visible only after resolution
- **Incident Store (Pinia)**: Manages incident list, selected incident detail, section-level loading/error states, filter state
- **Incident API Client**: REST client for incident endpoints using shared `fetchJson<T>`

### Backend Components

- **Incident Controller**: REST endpoints for incident list and detail. Returns `ApiResponse<T>`.
- **Incident Service**: Assembles the full incident detail from domain sub-services. Workspace-scoped.
- **Incident DTOs**: Immutable Java records matching frontend TypeScript interfaces. Includes per-section `SectionResultDto<T>` wrappers.

### Monitoring / Audit

- **Governance audit entries**: Every approve/reject/escalate/override action recorded with actor, timestamp, reason, and policy reference. Feeds into platform audit system.
- **Skill execution records**: Each AI skill invocation recorded with input/output summary, timestamps, and status.

---

## Data Architecture

### Conceptual Entities
| Entity | Description | Key Attributes |
|---|---|---|
| Incident | An operational incident in a workspace | id, title, priority, status, handlerType, controlMode, autonomyLevel, timestamps, workspaceId |
| DiagnosisEntry | One entry in the AI reasoning feed | timestamp, text, entryType |
| SkillExecution | An AI skill invocation record | skillName, startTime, endTime, status, inputSummary, outputSummary, evidenceRefs |
| IncidentAction | An AI-proposed or executed action | description, actionType, executionStatus, timestamp, impactAssessment, isRollbackable |
| GovernanceEntry | A human governance decision | actor, timestamp, actionTaken, reason, policyRef |
| SdlcChainLink | A link to an upstream SDLC artifact | artifactType, artifactId, artifactTitle, routePath |
| AiLearning | Post-resolution AI analysis | rootCause, patternIdentified, preventionRecommendations, knowledgeBaseEntryCreated |

### State / Status Models

**Incident Status State Machine:**

```
DETECTED → AI_INVESTIGATING → AI_DIAGNOSED → ACTION_PROPOSED
    → PENDING_APPROVAL → EXECUTING → RESOLVED → LEARNING → CLOSED

Override transitions:
    AI_INVESTIGATING → ESCALATED (AI cannot diagnose)
    AI_DIAGNOSED → ESCALATED (human takes over)
    PENDING_APPROVAL → MANUAL_OVERRIDE (human overrides)
    ESCALATED → RESOLVED (human resolves)
    MANUAL_OVERRIDE → RESOLVED (human resolves)
```

**Action Execution Status:**
`PENDING → APPROVED → EXECUTING → EXECUTED | ROLLED_BACK`
`PENDING → REJECTED`

**Skill Execution Status:**
`RUNNING → COMPLETED | FAILED | PENDING_APPROVAL`

### Persistence Responsibilities
- Phase A: All data mocked via frontend mock data files and/or Flyway seed migration
- Phase B: Incident domain entities persisted via JPA. Governance entries are append-only (immutable audit trail)

---

## Integration Architecture

### Shared App Shell
- **Interaction Pattern**: Incident page renders inside the shell via Vue Router lazy-load
- **Data exchanged**: Workspace context (from shell store), navigation events
- **Verified**: Route configured at `router/index.ts:33` with `coming-soon: false`

### Dashboard Module
- **Interaction Pattern**: Dashboard stability card links to `/incidents`
- **Data exchanged**: Navigation link only; no data coupling
- **Verified**: Dashboard StabilityIncidentCard navigates to `/incidents`

### SDLC Module Pages
- **Interaction Pattern**: SDLC chain links in incident detail navigate to module pages
- **Data exchanged**: Route path + artifact ID via Vue Router
- **Constraint**: Unimplemented modules show "Coming Soon" placeholder

---

## Workflow / Runtime Architecture

### Request Flow (Page Load)

1. User clicks "Incident Management" in shell navigation → Vue Router loads incident list view
2. Incident store dispatches `fetchIncidentList()` → API client calls `GET /api/v1/incidents`
3. List view renders with severity distribution summary and incident rows
4. User clicks an incident row → Vue Router navigates to `/incidents/:id`
5. Incident store dispatches `fetchIncidentDetail(id)` → API client calls `GET /api/v1/incidents/:id`
6. Detail response contains sections wrapped in `SectionResult<T>` for independent card loading
7. Each card renders independently — failed sections show per-card error state

### Governance Flow (Action Approval)

1. User reviews pending action in AI Actions card
2. User clicks Approve or Reject
3. Frontend calls `POST /api/v1/incidents/:id/actions/:actionId/approve` or `/reject`
4. Backend validates, transitions action status, creates governance entry
5. Frontend refreshes action and governance cards

### State Transitions
- `DETECTED → AI_INVESTIGATING` — triggered when first detection skill starts
- `AI_INVESTIGATING → AI_DIAGNOSED` — triggered when diagnosis skill completes with hypothesis
- `AI_DIAGNOSED → ACTION_PROPOSED` — triggered when remediation skill proposes actions
- `ACTION_PROPOSED → PENDING_APPROVAL` — triggered when action requires human approval
- `PENDING_APPROVAL → EXECUTING` — triggered by human approval
- `EXECUTING → RESOLVED` — triggered by successful action completion
- `RESOLVED → LEARNING` — triggered when learning skill starts
- `LEARNING → CLOSED` — triggered when learning is captured

### Failure and Retry Handling
- V1 does not implement automated retry for failed actions
- Failed AI skills are recorded in the timeline with `FAILED` status
- Frontend retry: manual refresh via user action or page reload

---

## API / Interface Boundaries

### Major Inbound Interfaces
| Interface | Consumer | Purpose |
|---|---|---|
| `GET /api/v1/incidents` | Frontend incident list | List incidents with optional filters |
| `GET /api/v1/incidents/:id` | Frontend incident detail | Full incident detail with all sections |
| `POST /api/v1/incidents/:id/actions/:actionId/approve` | Frontend governance | Approve a pending action |
| `POST /api/v1/incidents/:id/actions/:actionId/reject` | Frontend governance | Reject a pending action with reason |

### Internal Module Boundaries
- Incident domain consumes workspace context from the shared platform module
- Incident domain does not call other domain modules directly — SDLC chain links are stored as references (IDs + types), not live joins

---

## Deployment / Environment Considerations

- **Supported Environments**: Local dev (H2 + Vite dev server), production (Oracle + deployed frontend)
- **Configuration Separation**: Flyway migrations handle schema; `application.yml` profiles handle env-specific config
- **Secrets Handling**: No incident-specific secrets in V1; workspace context inherited from shell
- **Operational Concerns**: Governance audit entries must be append-only and durable

---

## Security / Reliability / Observability

### Access Control
- All incident data workspace-scoped (enforced at service layer)
- Governance actions (approve/reject) require authenticated user identity
- V1 does not implement per-action role-based access control

### Auditability
- All governance actions recorded with actor, timestamp, reason, and policy reference
- Skill execution records provide a complete audit trail of AI activity
- Governance entries are append-only — no updates or deletes

### Resilience
- Per-card error isolation via `SectionResult<T>` — one card failure does not break the page
- V1 uses synchronous request/response; no async job processing

### Monitoring / Logging
- Standard Spring Boot logging for API requests
- Frontend console logging for development diagnostics

---

## Risks / Tradeoffs

| # | Risk / Tradeoff | Notes |
|---|---|---|
| 1 | Skill execution timeline and diagnosis feed may overlap in content | Architecture separates them: feed is narrative reasoning; timeline is discrete skill invocations. Design must enforce visual distinction. |
| 2 | Governance and AI Actions cards show related but distinct data | Governance is the audit-historical view; Actions is the operational-current view. Consider combining in a future iteration if user feedback suggests confusion. |
| 3 | SDLC chain links are stored as references, not live joins | If upstream artifacts change ID or are deleted, chain links become stale. V1 accepts this risk; future versions may add link validation. |
| 4 | No real-time data delivery in V1 | Diagnosis feed and timeline are static at page load. Users must manually refresh. This may reduce the "command center" feel. |
| 5 | Autonomy level definitions are not specified in the PRD | Architecture assumes a 3-level model (Manual / Suggest+Approve / Auto+Post-audit). Needs product confirmation. |

---

## Open Questions

1. What are the concrete autonomy level definitions? (Suggest 3 levels for V1)
2. Should governance entries be stored in the incident domain or in a shared platform audit table?
3. What is the incident data retention policy? (Impacts schema design for archival)
4. Should the approve/reject API be synchronous or eventually consistent?
5. Will V2 require WebSocket for real-time diagnosis feed updates?
