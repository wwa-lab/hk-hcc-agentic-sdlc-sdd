# Incident Management Tasks

## Objective

Implement the Incident Management page — the AI-native operational command center
of the Agentic SDLC Control Tower product.

## Implementation Strategy

Frontend and backend are developed in separate phases with different tools:

| Phase   | Scope                 | Tool        | Goal                                        |
| ------- | --------------------- | ----------- | ------------------------------------------- |
| Phase A | Frontend (Vue 3)      | Claude Code | Build incident UI with mocked data          |
| Phase B | Backend (Spring Boot) | Claude Code | Build API and connect frontend to live data |

Phase A is self-contained and produces a fully functional incident page with mocked data.
Phase B adds the backend and replaces mocked data with real API calls.

## Traceability

- Requirements: [incident-requirements.md](../01-requirements/incident-requirements.md)
- Stories: [incident-stories.md](../02-user-stories/incident-stories.md)
- Spec: [incident-spec.md](../03-spec/incident-spec.md)
- Architecture: [incident-architecture.md](../04-architecture/incident-architecture.md)
- Design: [incident-design.md](../05-design/incident-design.md)
- Data Model: [incident-data-model.md](../04-architecture/incident-data-model.md)
- API Guide: [incident-API_IMPLEMENTATION_GUIDE.md](../05-design/contracts/incident-API_IMPLEMENTATION_GUIDE.md)

## Planning Assumptions

- `SectionResultDto<T>` will be moved from `domain/dashboard/dto/` to `shared/dto/` as a prerequisite
- Frontend `SectionResult<T>` interface already exists in `dashboard/types/dashboard.ts` and will be extracted to a shared location or re-exported
- Mock toggle pattern follows dashboard: `import.meta.env.DEV && !import.meta.env.VITE_USE_BACKEND` (verified: `dashboardStore.ts:7`)
- API client pattern follows dashboard: `fetchJson<T>` from `shared/api/client.ts` (verified: `dashboardApi.ts:1`)
- Route already exists at `/incidents` in `router/index.ts:33` (verified)

---

## Phase A: Frontend

### A0: Shared Infrastructure Preparation

- Move `SectionResult<T>` interface to a shared location accessible by both dashboard and incident modules (e.g., `frontend/src/shared/types/section.ts` or re-export from dashboard types)
- Ensure the existing `SectionResult<T>` import in dashboard components is not broken
- Add `postJson<T>(path, body?)` function to `frontend/src/shared/api/client.ts` alongside the existing `fetchJson<T>`. The new function sends POST requests with JSON body and unwraps the `ApiEnvelope<T>` response. This is needed for approve/reject endpoints. (See API guide §7.1 for the exact signature.)
- Verify `--color-incident-crimson` and `--color-incident-tint` design tokens exist in `variables.css` (verified: they exist)

Depends on: none
Done when: `SectionResult<T>` is importable from a shared path; `postJson<T>` is exported from `client.ts`; dashboard still builds clean

### A1: Define Incident Types and Mock Data

- Create `frontend/src/features/incident/types/incident.ts` with all TypeScript interfaces:
  - Enums: `Priority`, `IncidentStatus`, `HandlerType`, `ControlMode`, `AutonomyLevel`, `DiagnosisEntryType`, `SkillExecutionStatus`, `ActionType`, `ActionExecutionStatus`, `GovernanceActionType`, `ConfidenceLevel`, `SdlcArtifactType`, `SortField`
  - List types: `SeverityDistribution`, `IncidentListItem`, `IncidentFilters`, `IncidentList`
  - Detail types: `IncidentHeader`, `DiagnosisEntry`, `RootCauseHypothesis`, `DiagnosisFeed`, `SkillExecution`, `SkillTimeline`, `IncidentAction`, `IncidentActions`, `GovernanceEntry`, `Governance`, `SdlcChainLink`, `SdlcChain`, `AiLearning`
  - Aggregate: `IncidentDetail` with 7 `SectionResult<T>` fields
- Replace existing `frontend/src/features/incident/mockData.ts` with comprehensive mock data:
  - 5 incidents with varying priorities (P1–P3), statuses (active + resolved + learning + closed), handler types (AI / Human / Hybrid)
  - INC-0422: P1, PENDING_APPROVAL, AI handler — the primary demo incident
  - Diagnosis feed: 6 entries (analysis, finding, suggestion, conclusion)
  - Skill timeline: 4 executions (detection → correlation → diagnosis → remediation)
  - Actions: 1 pending action (scale replicas), policy reference
  - Governance: empty for INC-0422 (pending); entries for resolved incidents
  - SDLC chain: 3 links (spec, code, deploy)
  - AI learning: data for resolved incidents, null for active
- Mock data values must match the API Implementation Guide JSON examples exactly

Depends on: A0
Done when: types compile; mock data is importable and type-safe; matches API guide examples

### A2: Build Reusable Badge Components

- Create `SeverityBadge.vue` — P1 (crimson), P2 (error 70%), P3 (tertiary), P4 (muted)
- Create `StatusBadge.vue` — LED indicator + status text, active states pulse
- Create `HandlerTypeBadge.vue` — AI (cyan), Human (standard), Hybrid (gradient)
- Create `IncidentCard.vue` — reusable card wrapper following DashboardCard pattern (surface-container-high, 4px radius, no borders, per-card error handling)

Depends on: A0
Done when: all 4 components render correctly with sample props

### A3: Build Incident List View

- Create `IncidentListView.vue` with:
  - `SeverityDistribution.vue` strip at top (P1/P2/P3/P4 counts)
  - Filter bar: priority dropdown, status dropdown, handler type dropdown, date range
  - Active/Resolved tab toggle
  - `IncidentListTable.vue` — sortable table with columns: ID, Title, Priority, Status, Handler, Duration
  - Each row uses SeverityBadge, StatusBadge, HandlerTypeBadge
  - Row click emits `@select` with incident ID
- Implement sort by priority, recency, duration
- Empty state: "All clear — no active incidents in this workspace"
- Loading state: skeleton rows

Depends on: A1, A2
Done when: list renders 5 mock incidents; sorting works; filtering works; empty state renders

### A4: Build Incident Header Card

- Create `IncidentHeaderCard.vue`:
  - Incident ID (JetBrains Mono), title, priority badge, status badge
  - Handler type badge, control mode label, autonomy level label
  - Timestamps: detected, acknowledged, resolved (or duration timer)
  - Full-width layout in detail view

Depends on: A2
Done when: header renders INC-0422 data with all fields visible

### A5: Build AI Diagnosis Feed Card

- Create `DiagnosisFeedCard.vue`:
  - Chronological log feed in JetBrains Mono (0.75rem, line-height 1.6)
  - Each entry: `[timestamp] text`
  - Entry types: analysis (standard), finding (standard), suggestion (cyan accent), conclusion (bold)
  - Root cause hypothesis section with confidence badge (High/Medium/Low)
  - Affected components list
  - Card spans 2 rows in the detail grid (left column)

Depends on: A2
Done when: feed renders 6 mock entries; suggestions highlighted in cyan; root cause visible

### A6: Build Skill Timeline Card

- Create `SkillTimelineCard.vue`:
  - Vertical timeline with chronological entries
  - Each entry: skill name, start→end timestamps, duration, status badge
  - Status colors: running (cyan pulse), completed (green), failed (crimson), pending_approval (amber)
  - Input/output summaries collapsible or shown inline

Depends on: A2
Done when: timeline renders 4 mock skill executions with correct status colors

### A7: Build AI Actions Card

- Create `AiActionsCard.vue`:
  - List of actions with: description, action type, status, timestamp
  - Impact assessment text visible per action
  - Rollback indicator (icon) for rollbackable actions
  - Policy reference text when present
  - Approve/Reject buttons for `pending` actions
  - Approve click emits `@approve(actionId)`
  - Reject click shows reason input, then emits `@reject({ actionId, reason })`

Depends on: A2
Done when: actions card renders with approve/reject buttons; clicking approve/reject changes local state in mock mode

### A8: Build Governance Card

- Create `GovernanceCard.vue`:
  - Chronological list of governance entries
  - Each entry: actor, timestamp, action taken (approve/reject/escalate/override), reason
  - Policy reference when present
  - Empty state: "No governance actions yet"
  - Escalation path indicator when applicable

Depends on: A2
Done when: governance card renders entries for resolved incidents; empty state for pending incidents

### A9: Build SDLC Chain Card

- Create `SdlcChainCard.vue`:
  - Horizontal chain of SDLC artifact links
  - Compressed view: shows only linked artifact types (not all 11 nodes)
  - Spec node always visible even if not directly linked (per PRD §13.1)
  - Each link: artifact type icon, ID, title, clickable
  - Click emits `@navigate(routePath)`
  - Empty state: "No linked artifacts"
  - Expand/collapse control to show full chain context

Depends on: A2
Done when: chain renders 3 mock links; Spec visible; clicking navigates via router

### A10: Build AI Learning Card

- Create `AiLearningCard.vue`:
  - Root cause (confirmed), pattern identified, prevention recommendations (list)
  - Knowledge base entry indicator (created/not created)
  - When incident is not resolved: shows "Learning will be available after resolution"
  - Full-width layout (bottom of detail grid)

Depends on: A2
Done when: learning card renders for resolved incidents; placeholder for active incidents

### A11: Build Incident Store

- Create `frontend/src/features/incident/stores/incidentStore.ts` (Pinia store)
- Store shape: `list` (incidents, severity distribution, isLoading, error, filters) + `detail` (7 SectionResult fields, isLoading, error) + `selectedIncidentId`
- Actions: `fetchIncidentList()`, `fetchIncidentDetail(id)`, `approveAction(incidentId, actionId)`, `rejectAction(incidentId, actionId, reason)`, `setFilters(filters)`, `clearDetail()`
- Mock toggle: `import.meta.env.DEV && !import.meta.env.VITE_USE_BACKEND` (same as dashboard)
- Phase A: imports from mockData.ts; Phase B: calls incidentApi.ts
- Create `frontend/src/features/incident/api/incidentApi.ts` — GET endpoints use `fetchJson<T>`, POST endpoints (approve/reject) use `postJson<T>`, both from `shared/api/client.ts`

Depends on: A1
Done when: store is importable; `fetchIncidentList()` returns mock data; reactive state works

### A12: Set Up Routing (Nested Routes)

**NOTE: This is the first nested route in the codebase.** The current router (`router/index.ts`) uses only flat, top-level routes. This task modifies the shared router infrastructure.

- Modify `router/index.ts`: change the `incidents` entry from a flat route to a parent route with `children`:
  - Parent: `/incidents` → `IncidentManagementView.vue` (becomes a `<router-view>` host)
  - Child default: `''` (empty path) → `IncidentListView.vue` (name: `'incidents'`)
  - Child detail: `':incidentId'` → `IncidentDetailView.vue` (name: `'incident-detail'`)
- Modify `IncidentManagementView.vue` to replace current placeholder content with `<router-view />`
- Create `views/IncidentListView.vue` as the list page
- Create `views/IncidentDetailView.vue` as the detail layout host (CSS grid for 7 cards)
- Verify: all existing shell navigation still works after the router change; dashboard, platform, project-space routes unaffected

Depends on: none (can start early)
Done when: navigating to `/incidents` shows list; `/incidents/INC-0422` shows detail; browser back works; existing routes unbroken

### A13: Assemble Incident Detail View

- Wire all 7 cards into `IncidentDetailView.vue` using CSS grid layout:
  - Row 1: Header (full width)
  - Row 2–3: Diagnosis feed (left, span 2) + Skill timeline (right top) + AI Actions (right bottom)
  - Row 4: Governance (left) + SDLC chain (right)
  - Row 5: AI Learning (full width)
- Wire cards to incident store detail state
- Wire card events: approve/reject → store actions; navigate → router.push
- Implement per-card loading and error states via SectionResult

Depends on: A4, A5, A6, A7, A8, A9, A10, A11, A12
Done when: detail view renders all 7 cards from mock data; approve/reject works locally; navigation works

### A14: Assemble Incident List View Integration

- Wire `IncidentListView` to incident store
- Wire filter changes → `setFilters()` → re-filter mock data
- Wire row click → `router.push({ name: 'incident-detail', params: { incidentId } })`
- Wire severity distribution to store data
- Implement loading, error, and empty states

Depends on: A3, A11, A12
Done when: list view renders from store; filtering works; clicking navigates to detail

### A15: Validate Incident Frontend

- Verify all 5 mock incidents render in the list with correct badges
- Verify severity distribution shows correct counts
- Verify filtering by priority, status, handler type works
- Verify sorting by severity, recency, duration works
- Verify INC-0422 detail renders all 7 cards correctly
- Verify diagnosis feed shows entries with suggestion highlighting
- Verify skill timeline shows 4 executions with correct status colors
- Verify approve/reject buttons work on pending action
- Verify SDLC chain links navigate correctly (to placeholder pages)
- Verify AI learning shows for resolved incidents, placeholder for active
- Verify empty state "All clear" renders when filter excludes all incidents
- Verify browser back navigation between list and detail
- Verify deep-linking to `/incidents/INC-0422` works
- Verify `npm run dev` and `npm run build` both succeed

Depends on: A13, A14
Done when: all checks pass; incident page is complete with mocked data

### Phase A Definition of Done

- Incident list view renders 5 incidents with severity distribution
- Incident detail view renders all 7 cards inside the shared shell
- AI diagnosis feed displays with suggestion highlighting and root cause
- Skill timeline shows execution history with status indicators
- Approve/reject flow works locally with mock data
- SDLC chain card shows compressed chain with Spec visible
- AI learning shows for resolved incidents
- Navigation between list and detail works; deep-linking works
- Loading, error, and empty states all render correctly
- No backend dependency; all data is mocked
- `npm run dev` and `npm run build` both succeed

---

## Phase B: Backend

### B0: Shared Infrastructure (Prerequisite)

- Move `SectionResultDto<T>` from `domain/dashboard/dto/` to `shared/dto/`
- Update `DashboardService.java` imports to reference `shared.dto.SectionResultDto`
- Add incident endpoint constants to `ApiConstants.java`:
  ```
  INCIDENTS = API_V1 + "/incidents"
  INCIDENT_DETAIL = INCIDENTS + "/{incidentId}"
  INCIDENT_ACTION_APPROVE = INCIDENT_DETAIL + "/actions/{actionId}/approve"
  INCIDENT_ACTION_REJECT = INCIDENT_DETAIL + "/actions/{actionId}/reject"
  ```
- Verify existing dashboard tests still pass after SectionResultDto move

Depends on: none (can start in parallel with Phase A)
Done when: `SectionResultDto` is in shared; ApiConstants has incident paths; dashboard tests pass

### B1: Create Incident DTOs

- Create Java records in `domain/incident/dto/`:
  - List: `IncidentListDto`, `IncidentListItemDto`, `SeverityDistributionDto`
  - Detail: `IncidentDetailDto`, `IncidentHeaderDto`, `DiagnosisFeedDto`, `DiagnosisEntryDto`, `RootCauseHypothesisDto`, `SkillTimelineDto`, `SkillExecutionDto`, `IncidentActionsDto`, `IncidentActionDto`, `GovernanceDto`, `GovernanceEntryDto`, `SdlcChainDto`, `SdlcChainLinkDto`, `AiLearningDto`
  - Request: `ActionApprovalRequestDto` (with `reason` field)
- All DTOs are Java records (immutable)
- Field names match frontend TypeScript interfaces exactly (camelCase JSON)

Depends on: B0
Done when: all DTOs compile; Jackson serializes to JSON matching the API guide examples

### B2: Implement Incident Service

- Create `IncidentService.java` in `domain/incident/`
- Implement `getIncidentList(filters)` returning `IncidentListDto` with seed data
- Implement `getIncidentDetail(id)` returning `IncidentDetailDto` with 7 `SectionResultDto` sections
- Implement `approveAction(incidentId, actionId)` — validates state, returns result
- Implement `rejectAction(incidentId, actionId, reason)` — validates state + reason, returns result
- Seed data must match the frontend mock data / API guide JSON examples
- Service is workspace-aware (accepts workspace ID for future isolation)

Depends on: B1
Done when: service returns valid DTOs with realistic seed data; approve/reject work

### B3: Implement Incident Controller

- Create `IncidentController.java` with:
  - `GET /api/v1/incidents` → `listIncidents(filters)` returns `ApiResponse<IncidentListDto>`
  - `GET /api/v1/incidents/{incidentId}` → `getIncidentDetail(id)` returns `ApiResponse<IncidentDetailDto>`
  - `POST /api/v1/incidents/{incidentId}/actions/{actionId}/approve` → returns `ApiResponse<ActionApprovalResultDto>`
  - `POST /api/v1/incidents/{incidentId}/actions/{actionId}/reject` → returns `ApiResponse<ActionApprovalResultDto>` with `@RequestBody ActionApprovalRequestDto`
- Proper error handling: 404 for not found, 400 for invalid state/missing reason

Depends on: B2
Done when: all 4 endpoints return valid JSON matching the API guide

### B4: Create Flyway Seed Migration

- Create `V4__seed_incident_data.sql` with seed data for incidents
- Seed data values must match the frontend mock data and API guide examples
- Include: 5 incidents, diagnosis entries, skill executions, actions, governance entries, SDLC chain links, AI learning data

Depends on: B1
Done when: migration runs cleanly; H2 console shows seeded data

### B5: Write Backend Tests

- Create `IncidentControllerTest.java` using MockMvc:
  - GET /api/v1/incidents returns 200 with incidents[] and severityDistribution
  - GET /api/v1/incidents/INC-0422 returns 200 with all 7 sections
  - GET /api/v1/incidents/INC-9999 returns 404
  - POST approve returns 200 with newStatus "approved"
  - POST reject returns 200 with newStatus "rejected"
  - POST reject without reason returns 400

Depends on: B3
Done when: all tests pass

### B6: Connect Frontend to Backend

- Update `incidentApi.ts` to call all 4 endpoints
- Update `incidentStore.ts` to use API instead of mock data when `VITE_USE_BACKEND` is set
- Configure Vite dev server proxy for `/api/v1/incidents/*`
- Handle loading and error states from real API responses
- Retain mock data as fallback for development without backend

Depends on: A15, B3
Done when: frontend renders live data from Spring Boot

### B7: Validate Full Stack

- Verify frontend + backend integration end to end
- Verify incident list loads from backend API
- Verify incident detail loads all 7 sections from backend
- Verify approve/reject actions work through the full stack
- Verify API error responses produce graceful frontend fallbacks
- Verify H2 profile works for local development

Depends on: B6
Done when: full stack runs locally; incident page displays backend seed data

### Phase B Definition of Done

- Spring Boot serves incident list and detail via REST API
- Approve/reject endpoints work with state validation
- API response shapes match frontend TypeScript contracts
- Frontend fetches live data from backend
- All 7 detail cards render from backend data
- Controller tests pass (6 test cases)
- `./mvnw spring-boot:run` and `npm run dev` work together locally

---

## Dependency Plan

### Critical Path

```
A0 → A1 → A11 → A14 → A15
         → A2 → A3 → A14
              → A4 ─┐
              → A5  │
              → A6  ├→ A13 → A15
              → A7  │
              → A8  │
              → A9  │
              → A10 ┘
A12 (parallel with A2–A10) → A13, A14
```

### Parallel Workstreams

| Stream                     | Tasks                       | Can Run Alongside         |
| -------------------------- | --------------------------- | ------------------------- |
| Frontend types + data      | A0, A1                      | — (first)                |
| Badge components           | A2                          | A1 (after A0)             |
| Detail cards               | A4, A5, A6, A7, A8, A9, A10 | All parallel after A2     |
| List view                  | A3                          | After A1, A2              |
| Store + routing            | A11, A12                    | After A1                  |
| Backend DTOs + service     | B0, B1, B2                  | Parallel with all Phase A |
| Backend controller + tests | B3, B4, B5                  | After B2                  |

---

## Risks / Blockers

| # | Risk                                                                              | Impact            | Resolution                                                      |
| - | --------------------------------------------------------------------------------- | ----------------- | --------------------------------------------------------------- |
| 1 | Moving `SectionResultDto` to shared may break dashboard imports                 | Build failure     | Run dashboard tests immediately after the move (B0)             |
| 2 | Nested routes in incident page may conflict with existing shell route config      | Navigation broken | Test route registration early (A12) before building components  |
| 3 | 7 cards in detail view CSS grid may have layout issues on narrower screens        | Visual breakage   | Test at 1280px minimum width; allow horizontal scroll if needed |
| 4 | Approve/reject mock flow may not feel realistic without backend state persistence | UX confusion      | Store locally mutates mock data to simulate state transitions   |

---

## Open Questions

1. Should `SectionResult<T>` be extracted to `shared/types/` or re-exported from dashboard? (Decision needed before A0)
2. What is the minimum viewport width for the incident detail grid? (1280px assumed)
3. Should the reject reason dialog be a modal or inline input? (Design assumes inline)
4. Should the list view support URL-based filter state (e.g., `/incidents?priority=P1`)? (Not required for V1)
