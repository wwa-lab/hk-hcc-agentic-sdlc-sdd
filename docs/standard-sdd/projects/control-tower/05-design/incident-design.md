# Incident Management Design

## Purpose

This document defines the concrete component APIs, file structure, data model,
visual design decisions, and API contracts for the Incident Management page.
It bridges the [architecture](../04-architecture/incident-architecture.md) and the
[implementation tasks](../06-tasks/incident-tasks.md).

## Traceability

- Requirements: [incident-requirements.md](../01-requirements/incident-requirements.md)
- Stories: [incident-stories.md](../02-user-stories/incident-stories.md)
- Spec: [incident-spec.md](../03-spec/incident-spec.md)
- Architecture: [incident-architecture.md](../04-architecture/incident-architecture.md)
- Data Model: [incident-data-model.md](../04-architecture/incident-data-model.md)
- API Guide: [incident-API_IMPLEMENTATION_GUIDE.md](contracts/incident-API_IMPLEMENTATION_GUIDE.md)
- Visual design system: [visual-design-system.md](visual-design-system.md) (§The Tactical Command)

## 1. File Structure

### Frontend

```
frontend/src/
├── features/
│   └── incident/
│       ├── IncidentManagementView.vue       # page view — list/detail router-view host
│       ├── views/
│       │   ├── IncidentListView.vue         # incident list with filtering and sorting
│       │   └── IncidentDetailView.vue       # detail layout — hosts 7 cards
│       ├── components/
│       │   ├── IncidentCard.vue             # reusable card wrapper (like DashboardCard)
│       │   ├── IncidentListTable.vue        # incident table rows
│       │   ├── SeverityBadge.vue            # P1–P4 priority badge
│       │   ├── StatusBadge.vue              # incident status badge with LED
│       │   ├── HandlerTypeBadge.vue         # AI / Human / Hybrid badge
│       │   ├── IncidentHeaderCard.vue       # header with AI posture
│       │   ├── DiagnosisFeedCard.vue        # AI diagnosis log feed
│       │   ├── SkillTimelineCard.vue        # skill execution timeline
│       │   ├── AiActionsCard.vue            # actions with approve/reject
│       │   ├── GovernanceCard.vue           # governance audit trail
│       │   ├── SdlcChainCard.vue            # related SDLC chain
│       │   ├── AiLearningCard.vue           # post-resolution learning
│       │   └── SeverityDistribution.vue     # P1/P2/P3/P4 summary strip
│       ├── stores/
│       │   └── incidentStore.ts             # Pinia store — list + detail state
│       ├── api/
│       │   └── incidentApi.ts               # API client for /incidents/*
│       ├── types/
│       │   └── incident.ts                  # TypeScript interfaces
│       └── mockData.ts                      # Phase A mocked data (replaces current)
```

### Backend

```
backend/src/main/java/com/sdlctower/
├── domain/
│   └── incident/
│       ├── IncidentController.java          # REST endpoints
│       ├── IncidentService.java             # domain logic + data assembly
│       └── dto/
│           ├── IncidentListDto.java
│           ├── IncidentListItemDto.java
│           ├── SeverityDistributionDto.java
│           ├── IncidentDetailDto.java
│           ├── IncidentHeaderDto.java
│           ├── DiagnosisFeedDto.java
│           ├── DiagnosisEntryDto.java
│           ├── RootCauseHypothesisDto.java
│           ├── SkillTimelineDto.java
│           ├── SkillExecutionDto.java
│           ├── IncidentActionsDto.java
│           ├── IncidentActionDto.java
│           ├── GovernanceDto.java
│           ├── GovernanceEntryDto.java
│           ├── SdlcChainDto.java
│           ├── SdlcChainLinkDto.java
│           ├── AiLearningDto.java
│           └── ActionApprovalRequestDto.java
├── shared/
│   ├── dto/
│   │   ├── ApiResponse.java                 # existing shared envelope
│   │   └── SectionResultDto.java            # move from dashboard to shared [Assumption]
│   └── ApiConstants.java                    # add incident endpoint constants
├── src/main/resources/
│   └── db/migration/
│       └── V4__seed_incident_data.sql       # seed data for incidents
├── src/test/java/com/sdlctower/domain/incident/
│   └── IncidentControllerTest.java
```

---

## 2. Layout Composition

### 2.1 Incident List View

```
┌──────────────────────────────────────────────────────────────────┐
│  Context Bar (from shell)                                        │
├──────────────────────────────────────────────────────────────────┤
│  Page Header: "Incident Management" + subtitle + actions         │
├──────────────────────────────────────────────────────────────────┤
│  ┌──────────────────────────────────────────────────────────────┐│
│  │  Severity Distribution Strip                                 ││
│  │  [P1: 2] [P2: 3] [P3: 1] [P4: 0]                          ││
│  ├──────────────────────────────────────────────────────────────┤│
│  │  Filter Bar: Priority | Status | Handler | Date Range        ││
│  ├──────────────────────────────────────────────────────────────┤│
│  │  Active | Resolved (tab toggle)                              ││
│  ├──────────────────────────────────────────────────────────────┤│
│  │  Incident Table                                              ││
│  │  ID | Title | Priority | Status | Handler | Duration         ││
│  │  ─────────────────────────────────────────────────────────── ││
│  │  INC-0422 | API Gateway Latency... | P1 | AI_INVEST... | AI ││
│  │  INC-0421 | DB Connection Pool...  | P2 | RESOLVED   | Hyb  ││
│  │  INC-0420 | Cache Miss Spike...    | P3 | CLOSED     | AI   ││
│  └──────────────────────────────────────────────────────────────┘│
└──────────────────────────────────────────────────────────────────┘
```

### 2.2 Incident Detail View

```
┌──────────────────────────────────────────────────────────────────┐
│  Context Bar (from shell)                                        │
├──────────────────────────────────────────────────────────────────┤
│  ← Back to List | INC-0422 | Page Actions                       │
├──────────────────────────────────────────────────────────────────┤
│  ┌──────────────────────────────────────────────────────────────┐│
│  │  Incident Header Card (full width)                           ││
│  │  Title | Priority | Status | Handler: AI | Mode: Approval    ││
│  │  Autonomy: Level 2 | Detected: 09:40 | Duration: 1h23m      ││
│  └──────────────────────────────────────────────────────────────┘│
│                                                                  │
│  ┌────────────────────────────┐  ┌──────────────────────────────┐│
│  │  AI Diagnosis Feed         │  │  Skill Execution Timeline    ││
│  │  (left column, spans 2     │  │  (right column)              ││
│  │   rows)                    │  ├──────────────────────────────┤│
│  │                            │  │  AI Actions Card             ││
│  │  [09:41:02] Analyzing...  │  │  ┌─────────────────────────┐ ││
│  │  [09:41:05] Pattern: SSL  │  │  │ Scale Replicas (v2.4.0) │ ││
│  │  [09:41:10] SUGGESTION:   │  │  │ [REJECT] [APPROVE]      │ ││
│  │   Scale to 5 replicas     │  │  └─────────────────────────┘ ││
│  └────────────────────────────┘  └──────────────────────────────┘│
│                                                                  │
│  ┌────────────────────────────┐  ┌──────────────────────────────┐│
│  │  Governance Card           │  │  SDLC Chain Card             ││
│  │  Pending | History         │  │  Req → Spec → Code → Deploy  ││
│  └────────────────────────────┘  └──────────────────────────────┘│
│                                                                  │
│  ┌──────────────────────────────────────────────────────────────┐│
│  │  AI Learning Card (full width, visible after resolution)     ││
│  │  Root Cause | Pattern | Prevention | KB Entry                 ││
│  └──────────────────────────────────────────────────────────────┘│
└──────────────────────────────────────────────────────────────────┘
```

Detail view uses CSS grid:
- **Row 1**: Header card (full width)
- **Row 2–3**: Diagnosis feed (left, spans 2 rows) + Skill timeline (right top) + AI Actions (right bottom)
- **Row 4**: Governance (left) + SDLC chain (right)
- **Row 5**: AI Learning (full width)

---

## 3. Component API Contracts

### 3.1 IncidentListTable

| Prop | Type | Required | Description |
|------|------|----------|-------------|
| `incidents` | `readonly IncidentListItem[]` | Yes | Filtered incident list |
| `sortField` | `SortField` | Yes | Current sort column |
| `sortOrder` | `'asc' \| 'desc'` | Yes | Sort direction |

| Event | Payload | Description |
|-------|---------|-------------|
| `@select` | `incidentId: string` | User clicks a row |
| `@sort` | `{ field: SortField, order: 'asc' \| 'desc' }` | User changes sort |

### 3.2 SeverityBadge

| Prop | Type | Required | Description |
|------|------|----------|-------------|
| `priority` | `Priority` | Yes | P1, P2, P3, or P4 |

Visual: P1 uses `--color-incident-crimson`, P2 uses `--color-error` at 70%, P3 uses `--color-tertiary`, P4 uses `--color-on-surface-variant`.

### 3.3 StatusBadge

| Prop | Type | Required | Description |
|------|------|----------|-------------|
| `status` | `IncidentStatus` | Yes | Current incident status |

Visual: LED indicator + status text. Active statuses pulse subtly.

### 3.4 HandlerTypeBadge

| Prop | Type | Required | Description |
|------|------|----------|-------------|
| `handlerType` | `HandlerType` | Yes | AI, Human, or Hybrid |

Visual: AI uses `--color-secondary` (cyan), Human uses `--color-on-surface`, Hybrid uses gradient.

### 3.5 IncidentHeaderCard

| Prop | Type | Required | Description |
|------|------|----------|-------------|
| `header` | `SectionResult<IncidentHeader>` | Yes | Header data with error isolation |

Renders: ID, title, priority badge, status badge, handler badge, control mode, autonomy level, timestamps, duration.

### 3.6 DiagnosisFeedCard

| Prop | Type | Required | Description |
|------|------|----------|-------------|
| `diagnosis` | `SectionResult<DiagnosisFeed>` | Yes | Feed data with error isolation |

Renders: Chronological log entries in JetBrains Mono. Suggestions highlighted in cyan. Root cause hypothesis with confidence badge. Affected components list.

### 3.7 SkillTimelineCard

| Prop | Type | Required | Description |
|------|------|----------|-------------|
| `timeline` | `SectionResult<SkillTimeline>` | Yes | Timeline data |

Renders: Vertical timeline with skill name, duration, status badge, input/output summaries.

### 3.8 AiActionsCard

| Prop | Type | Required | Description |
|------|------|----------|-------------|
| `actions` | `SectionResult<IncidentActions>` | Yes | Actions data |

| Event | Payload | Description |
|-------|---------|-------------|
| `@approve` | `actionId: string` | User approves an action |
| `@reject` | `{ actionId: string, reason: string }` | User rejects with reason |

Renders: Action list with status, impact, rollback indicator, and approve/reject buttons for pending actions.

### 3.9 GovernanceCard

| Prop | Type | Required | Description |
|------|------|----------|-------------|
| `governance` | `SectionResult<Governance>` | Yes | Governance data |

Renders: Chronological list of governance entries with actor, action, reason, policy ref.

### 3.10 SdlcChainCard

| Prop | Type | Required | Description |
|------|------|----------|-------------|
| `chain` | `SectionResult<SdlcChain>` | Yes | Chain links |

| Event | Payload | Description |
|-------|---------|-------------|
| `@navigate` | `routePath: string` | User clicks a chain link |

Renders: Compressed SDLC chain with Spec always visible. Expand control for collapsed nodes.

### 3.11 AiLearningCard

| Prop | Type | Required | Description |
|------|------|----------|-------------|
| `learning` | `SectionResult<AiLearning>` | Yes | Learning data |

Renders: Root cause, pattern, prevention list, KB indicator. Shows "Learning will be available after resolution" if data is null and incident is not resolved.

---

## 4. Visual Design Decisions

### 4.1 Color Usage

| Element | Token | Value |
|---------|-------|-------|
| P1 Critical | `--color-incident-crimson` | `var(--color-error)` |
| P1 tint background | `--color-incident-tint` | `rgba(255, 180, 171, 0.1)` |
| AI handler / actions | `--color-secondary` | Electric Cyan `#89ceff` |
| Suggestion highlight | `--color-secondary` | Electric Cyan |
| Human handler | `--color-on-surface` | Standard text |
| Approved action | `--color-status-healthy` | Green |
| Rejected action | `--color-incident-crimson` | Red |
| Pending action | `--color-tertiary` | Amber/warning |

### 4.2 Typography

| Element | Font | Size | Style |
|---------|------|------|-------|
| Incident ID | JetBrains Mono | `text-tech` (0.75rem) | Uppercase |
| Diagnosis feed entries | JetBrains Mono | 0.75rem | Log-style, line-height 1.6 |
| Timestamps | JetBrains Mono | 0.6875rem | Muted color |
| Card headers | Inter | `text-label` | Uppercase, letter-spacing 0.05em |
| Body text | Inter | `body-sm` (0.75rem) | Standard |

### 4.3 Card Style

All cards follow `design.md §5`:
- Background: `--color-surface-container-high`
- Radius: `4px` (strict `--radius-sm`)
- No borders (the "No-Line" rule)
- Separation via background shifts and 16px whitespace
- Error state: card body replaced with error message + retry button

### 4.4 Status LED Indicators

Status LEDs follow the existing pattern from the prototype:
- `led-crimson` — P1 / critical / error states
- `led-amber` — warning / pending states
- `led-green` — healthy / resolved / approved states
- `led-muted` — inactive / closed states

Active states (AI_INVESTIGATING, EXECUTING) use a subtle CSS pulse animation.

---

## 5. State Management

### 5.1 Store Shape

```
incidentStore
├── list
│   ├── incidents: IncidentListItem[]
│   ├── severityDistribution: SeverityDistribution
│   ├── isLoading: boolean
│   ├── error: string | null
│   └── filters: IncidentFilters
├── detail
│   ├── header: SectionResult<IncidentHeader>
│   ├── diagnosis: SectionResult<DiagnosisFeed>
│   ├── skillTimeline: SectionResult<SkillTimeline>
│   ├── actions: SectionResult<IncidentActions>
│   ├── governance: SectionResult<Governance>
│   ├── sdlcChain: SectionResult<SdlcChain>
│   ├── learning: SectionResult<AiLearning>
│   ├── isLoading: boolean
│   └── error: string | null
└── selectedIncidentId: string | null
```

### 5.2 Store Actions

| Action | Triggers | Effect |
|--------|----------|--------|
| `fetchIncidentList()` | List view mount, filter change | Fetches list from API or mock |
| `fetchIncidentDetail(id)` | Detail view mount | Fetches detail from API or mock |
| `approveAction(incidentId, actionId)` | User clicks Approve | POST approve, refresh actions + governance |
| `rejectAction(incidentId, actionId, reason)` | User clicks Reject | POST reject, refresh actions + governance |
| `setFilters(filters)` | User changes filters | Updates filter state, re-fetches list |
| `clearDetail()` | Navigate back to list | Resets detail state |

### 5.3 Phase A / Phase B Toggle

The store checks `import.meta.env.DEV && !import.meta.env.VITE_USE_BACKEND`:
- When `true` (dev without backend): imports from `mockData.ts` directly
- When `false` (backend available): calls `incidentApi.ts` functions

This matches the verified pattern in `dashboardStore.ts:7`.

---

## 6. Routing

### 6.1 Route Configuration

The incident page uses nested routes within the existing shell:

| Route | Component | Purpose |
|-------|-----------|---------|
| `/incidents` | `IncidentListView.vue` | Default list view |
| `/incidents/:incidentId` | `IncidentDetailView.vue` | Detail view for specific incident |

`IncidentManagementView.vue` becomes a `<router-view>` host for these child routes.

### 6.2 Navigation

- Shell nav → `/incidents` → List view
- List row click → `router.push({ name: 'incident-detail', params: { incidentId } })`
- Detail back button → `router.push({ name: 'incidents' })`
- SDLC chain link → `router.push(routePath)`

---

## 7. Error and State Handling

### 7.1 List View States

| State | Behavior |
|-------|----------|
| Loading | Skeleton rows in table area |
| Error | Error card with message + retry button |
| Empty (no incidents) | "All clear — no active incidents in this workspace" with positive icon |
| Loaded | Full incident table |

### 7.2 Detail View States

| State | Behavior |
|-------|----------|
| Loading | All cards show skeleton placeholders |
| Error (full) | Error card replacing entire detail view |
| Partial error | Individual cards show inline error with retry |
| Loaded | All cards render their data |

### 7.3 Per-Card Error Isolation

Each card receives `SectionResult<T>`. Component template:

```
if section.error → render error message + retry
else if section.data → render card content
else → render loading skeleton
```

---

## 8. Validation and Error Handling

### 8.1 Frontend Validation

| Action | Validation | Error Handling |
|--------|-----------|----------------|
| Filter change | Validate date range (from ≤ to) | Show inline validation message |
| Reject action | Reason must be non-empty string | Disable reject submit until reason entered |
| URL param | Validate `:incidentId` matches `INC-NNNN` | Redirect to list if invalid |

### 8.2 Backend Validation

| Endpoint | Validation | Error Response |
|----------|-----------|----------------|
| `GET /incidents/:id` | ID exists in workspace | 404 with "Incident not found" |
| `POST /approve` | Action exists and is PENDING | 400 with "Action not in PENDING state" |
| `POST /reject` | Action exists, is PENDING, reason non-empty | 400 with validation message |

---

## 9. Testing Considerations

### 9.1 Backend Tests

| Test | Assertions |
|------|-----------|
| GET /incidents returns 200 | Response contains incidents array and severity distribution |
| GET /incidents/:id returns 200 | Response contains all 7 sections |
| GET /incidents/:id with invalid ID returns 404 | Error response |
| POST approve returns 200 | Action status transitions to APPROVED |
| POST reject returns 200 | Action status transitions to REJECTED, governance entry created |

### 9.2 Frontend Tests (future)

| Test | Assertions |
|------|-----------|
| List renders incidents from store | Table rows match incident count |
| Severity badges use correct colors | P1 uses crimson |
| Approve button calls store action | Store method invoked with correct params |
| Detail view loads all 7 cards | All cards render from mock data |
| Empty state shows positive message | "All clear" text visible |

---

## 10. Risks / Design Tradeoffs

| # | Tradeoff | Decision |
|---|----------|----------|
| 1 | Nested routes vs. single view with conditional rendering | Nested routes — cleaner URL support, code splitting, browser back works naturally |
| 2 | Diagnosis feed + skill timeline overlap | Keep as separate cards — feed is narrative reasoning, timeline is structured audit |
| 3 | Governance + AI Actions overlap | Keep as separate cards — actions is operational, governance is audit-focused |
| 4 | SectionResultDto in shared vs. dashboard package | Move to shared — it's used by both dashboard and incident modules |

---

## 11. Open Questions

1. Should `SectionResultDto` be moved from `dashboard/dto/` to `shared/dto/` now, or kept duplicated?
2. What is the polling interval for the "live feel" of the diagnosis feed (if any in V1)?
3. Should the severity distribution strip be clickable to filter by priority?
4. Should incident detail support keyboard shortcuts for approve/reject?
