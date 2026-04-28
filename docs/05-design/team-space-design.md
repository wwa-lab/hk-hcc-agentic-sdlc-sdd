# Team Space Design

## Purpose

This document defines the concrete implementation design of the Team Space slice: file structure, component APIs, state shape, routing, data contracts, visual decisions, database schema summary, error/empty states, and integration boundaries. It is the bridge between the architecture doc and the implementation work.

## Traceability

- Spec: [team-space-spec.md](../03-spec/team-space-spec.md)
- Architecture: [team-space-architecture.md](../04-architecture/team-space-architecture.md)
- Data Model: [team-space-data-model.md](../04-architecture/team-space-data-model.md)
- API Guide: [contracts/team-space-API_IMPLEMENTATION_GUIDE.md](contracts/team-space-API_IMPLEMENTATION_GUIDE.md)
- Visual System: [design.md](design.md) (project-root visual system)

---

## 1. File Structure

### Frontend

```
frontend/src/
├── features/
│   └── team-space/
│       ├── TeamSpaceView.vue
│       ├── components/
│       │   ├── WorkspaceSummaryCard.vue
│       │   ├── TeamOperatingModelCard.vue
│       │   ├── MemberMatrixCard.vue
│       │   ├── CoverageGapBanner.vue
│       │   ├── TeamTemplatesCard.vue
│       │   ├── TemplateGroup.vue
│       │   ├── ExceptionOverrideList.vue
│       │   ├── PipelineCard.vue
│       │   ├── PipelineCounters.vue
│       │   ├── PipelineBlockerList.vue
│       │   ├── SdlcChainStrip.vue
│       │   ├── MetricsCard.vue
│       │   ├── MetricItem.vue
│       │   ├── RiskRadarCard.vue
│       │   ├── RiskItem.vue
│       │   ├── ProjectDistributionCard.vue
│       │   └── ProjectCard.vue
│       ├── stores/
│       │   └── teamSpaceStore.ts
│       ├── api/
│       │   └── teamSpaceApi.ts
│       ├── types/
│       │   ├── enums.ts
│       │   ├── workspace.ts
│       │   ├── operatingModel.ts
│       │   ├── members.ts
│       │   ├── templates.ts
│       │   ├── pipeline.ts
│       │   ├── metrics.ts
│       │   ├── risks.ts
│       │   ├── projects.ts
│       │   └── aggregate.ts
│       ├── mock/
│       │   ├── aggregate.mock.ts
│       │   ├── summary.mock.ts
│       │   ├── operatingModel.mock.ts
│       │   ├── members.mock.ts
│       │   ├── templates.mock.ts
│       │   ├── pipeline.mock.ts
│       │   ├── metrics.mock.ts
│       │   ├── risks.mock.ts
│       │   └── projects.mock.ts
│       └── __tests__/
│           ├── TeamSpaceView.spec.ts
│           ├── teamSpaceStore.spec.ts
│           └── SdlcChainStrip.spec.ts
├── shared/
│   ├── components/
│   │   └── LineageBadge.vue    // reused by Team Space
│   ├── types/
│   │   ├── lineage.ts
│   │   └── section.ts
│   └── api/
│       └── client.ts           // fetchJson<T>(), postJson<T>()
└── router/
    └── index.ts               // wire existing /team route to TeamSpaceView
```

### Backend

```
backend/src/main/java/com/sdlctower/
├── domain/
│   └── teamspace/
│       ├── TeamSpaceController.java
│       ├── TeamSpaceService.java
│       ├── WorkspaceAccessGuard.java
│       ├── projection/
│       │   ├── WorkspaceProjection.java
│       │   ├── OperatingModelProjection.java
│       │   ├── MemberProjection.java
│       │   ├── TemplateInheritanceProjection.java
│       │   ├── RequirementPipelineProjection.java
│       │   ├── MetricsProjection.java
│       │   ├── RiskRadarProjection.java
│       │   └── ProjectDistributionProjection.java
│       ├── persistence/
│       │   ├── RiskSignalEntity.java
│       │   ├── MetricSnapshotEntity.java
│       │   ├── RiskSignalRepository.java
│       │   └── MetricSnapshotRepository.java
│       └── dto/
│           ├── TeamSpaceAggregateDto.java
│           ├── WorkspaceSummaryDto.java
│           ├── ResponsibilityBoundaryDto.java
│           ├── TeamOperatingModelDto.java
│           ├── FieldDto.java
│           ├── AccountableOwnerDto.java
│           ├── OncallOwnerDto.java
│           ├── MemberMatrixDto.java
│           ├── MemberMatrixRowDto.java
│           ├── CoverageGapDto.java
│           ├── TeamDefaultTemplatesDto.java
│           ├── TemplateEntryDto.java
│           ├── ExceptionOverrideDto.java
│           ├── RequirementPipelineDto.java
│           ├── PipelineCountersDto.java
│           ├── PipelineBlockerDto.java
│           ├── ChainNodeHealthDto.java
│           ├── TeamMetricsDto.java
│           ├── TeamMetricItemDto.java
│           ├── TeamRiskRadarDto.java
│           ├── RiskItemDto.java
│           ├── ProjectDistributionDto.java
│           ├── ProjectCardDto.java
│           ├── LineageDto.java
│           ├── LineageHopDto.java
│           └── LinkDto.java
├── shared/
│   ├── dto/
│   │   ├── ApiResponse.java
│   │   └── SectionResultDto.java
│   └── ApiConstants.java
└── src/main/resources/db/migration/
    ├── V7__create_team_space_tables.sql
    └── V8__seed_team_space_data.sql
```

---

## 2. Layout Composition

### 2.1 Default Team Space Layout

Desktop-first, 12-column grid inside the shared shell content area.

```
+--------------------------------------------------------------------------+
| Context bar: Workspace / Application / SNOW Group / Project / Environment |
+--------------------------------------------------------------------------+
| SDLC 11-node chain strip                                                 |
+--------------------------------------------------------------------------+
| WorkspaceSummaryCard  (col 1-6)       | OperatingModelCard   (col 7-12)  |
+--------------------------------------------------------------------------+
| MemberMatrixCard (col 1-6)            | TeamTemplatesCard    (col 7-12)  |
+--------------------------------------------------------------------------+
| PipelineCard (col 1-12)                                                  |
+--------------------------------------------------------------------------+
| MetricsCard (col 1-8)                 | RiskRadarCard        (col 9-12)  |
+--------------------------------------------------------------------------+
| ProjectDistributionCard (col 1-12)                                       |
+--------------------------------------------------------------------------+
```

Card order and span are configuration-driven (REQ-TS-04). In V1 the defaults are hardcoded as a layout constant and can be overridden by Workspace preference in a future Platform Center feature.

### 2.2 Responsive Behavior

- ≥ 1600px: 12-column grid as above.
- 1200–1599px: 12-column grid, slightly reduced typography; Risk Radar collapses to a compact list.
- 900–1199px: 8-column grid; cards stack (each full width); keeps high-density style.
- < 900px: out of scope for V1 (desktop-first per PRD §1).

---

## 3. Component API Contracts

### 3.1 `TeamSpaceView.vue`

```typescript
// Props: none (reads workspaceId from route query)
// Dependencies: useRoute(), useRouter(), useWorkspaceStore(), useTeamSpaceStore()

interface TeamSpaceViewState {
  workspaceId: string;
  isInitialLoad: boolean;
}

// Lifecycle:
// onMounted → store.initWorkspace(route.query.workspaceId ?? resolvedWorkspaceId)
// onBeforeUnmount → unregister AI Command Panel context
// watch(route.query.workspaceId) → store.switchWorkspace(newId)
```

### 3.2 `WorkspaceSummaryCard.vue`

```typescript
interface Props {
  section: SectionResult<WorkspaceSummary>;
}
interface Emits {
  retry: () => void;
}
```

Renders: identity line, health LED, project/env counters, responsibility boundary chip row, owner avatar.

### 3.3 `TeamOperatingModelCard.vue`

```typescript
interface Props {
  section: SectionResult<TeamOperatingModel>;
}
interface Emits {
  retry: () => void;
  'navigate-platform': (section: string) => void;
}
```

Each operating-mode field row uses `<LineageBadge>` to display lineage.

### 3.4 `LineageBadge.vue` (shared)

```typescript
interface Props {
  lineage: Lineage;
  compact?: boolean; // default true
}
```

Renders a small chip showing origin (e.g., "Workspace override") with a tooltip revealing the full chain.

### 3.5 `MemberMatrixCard.vue`

```typescript
interface Props {
  section: SectionResult<MemberMatrix>;
}
interface Emits {
  retry: () => void;
  'navigate-access': () => void;
  'focus-member': (memberId: string) => void;
}
```

Uses `<CoverageGapBanner>` at the top if gaps exist.

### 3.6 `PipelineCard.vue`

```typescript
interface Props {
  section: SectionResult<RequirementPipeline>;
}
interface Emits {
  retry: () => void;
  'navigate-requirement': (filterKey: string) => void;
}
```

Contains `<PipelineCounters>`, `<PipelineBlockerList>`, and `<SdlcChainStrip>`.

### 3.7 `SdlcChainStrip.vue`

```typescript
interface Props {
  chain: ChainNodeHealth[]; // always 11
  compact?: boolean;        // default false
}
```

Always renders 11 nodes. Spec node always visually highlighted as the execution hub. Per-node color follows the health LED palette.

### 3.8 `MetricsCard.vue`

```typescript
interface Props {
  section: SectionResult<TeamMetrics>;
}
interface Emits {
  retry: () => void;
  'navigate-report': (metricKey: MetricKey) => void;
}
```

### 3.9 `RiskRadarCard.vue`

```typescript
interface Props {
  section: SectionResult<TeamRiskRadar>;
}
interface Emits {
  retry: () => void;
  'open-action': (url: string) => void;
}
```

Renders groups in severity order; critical risks render with crimson accent.

### 3.10 `ProjectDistributionCard.vue`

```typescript
interface Props {
  section: SectionResult<ProjectDistribution>;
}
interface Emits {
  retry: () => void;
  'open-project': (projectId: string) => void;
}
```

Tabs for strata: Healthy / At-Risk / Critical / Archived with counters.

---

## 4. Visual Design Decisions

### 4.1 Color Usage

| Surface | Token |
|---------|-------|
| Card background | `--surface-card` |
| Card border | `--border-subtle` |
| Critical risk accent | `--accent-crimson` |
| Health GREEN LED | `--status-success` |
| Health YELLOW LED | `--status-warn` |
| Health RED LED | `--status-danger` |
| Health UNKNOWN LED | `--status-muted` |
| Lineage badge | `--chip-neutral` |
| Override badge | `--chip-emphasis` |
| Trend UP (good direction) | `--trend-positive` |
| Trend DOWN (good direction) | `--trend-positive` |
| Trend in bad direction | `--trend-negative` |
| Trend FLAT | `--trend-neutral` |

Tokens already defined in the root `design.md` visual system. Team Space reuses them without introducing new palette entries.

### 4.2 Typography

- Card title: `text-md`, weight 600
- Card subtitle / metadata: `text-sm`, weight 400, muted
- Metric value: `text-2xl`, weight 600
- Metric delta: `text-xs`, weight 500
- Risk title: `text-sm`, weight 600 (critical: 700)
- Member row: `text-sm`, weight 400

### 4.3 Card Style

- 1px border, 4px radius, no drop shadow (flat tactical aesthetic).
- Header row: title left, action overflow right (ellipsis menu).
- 16px internal padding on desktop; compact mode uses 12px.

### 4.4 SDLC Chain Strip Rules

- 11 equal-width cells with connecting arrows between cells.
- Spec cell is 1.2x the others and always highlighted (even if health is GREEN).
- Per-cell color is health LED; cell label is node name.
- Hovering a cell shows a popover with the node's workspace-scoped summary.

### 4.5 Empty / Error / Loading States

- Loading: shimmer skeleton matching card's resting shape.
- Error: subtle red outline, error message + retry button.
- Empty: illustrative glyph + calm copy (no exclamation).
- Partial: cards independently render their states; no global spinner.

---

## 5. State Management

### 5.1 Pinia Store Shape

```typescript
// frontend/src/features/team-space/stores/teamSpaceStore.ts

interface TeamSpaceState {
  workspaceId: string | null;
  cards: {
    summary: SectionResult<WorkspaceSummary>;
    operatingModel: SectionResult<TeamOperatingModel>;
    members: SectionResult<MemberMatrix>;
    templates: SectionResult<TeamDefaultTemplates>;
    pipeline: SectionResult<RequirementPipeline>;
    metrics: SectionResult<TeamMetrics>;
    risks: SectionResult<TeamRiskRadar>;
    projects: SectionResult<ProjectDistribution>;
  };
  loadingCards: Record<TeamSpaceCardKey, boolean>;
  pageState: 'idle' | 'loading' | 'ready' | 'error';
  pageError: string | null;
}
```

### 5.2 Store Actions

| Action | Behavior |
|--------|----------|
| `initWorkspace(workspaceId)` | Set workspaceId, set loading flags, call `loadAggregate` |
| `switchWorkspace(workspaceId)` | Reset transient state, set workspaceId, reload aggregate |
| `loadAggregate()` | Fetch aggregate endpoint; populate per-card section envelopes |
| `retryCard(cardKey)` | Fetch a single card endpoint and replace only that section |
| `refreshCard(cardKey)` | Alias of `retryCard(cardKey)` for manual refresh affordances |
| `reset()` | Clear state on unmount |

### 5.3 Phase A / Phase B Toggle

```typescript
const USE_MOCK = import.meta.env.DEV && !import.meta.env.VITE_USE_BACKEND;

async function fetchAggregate(workspaceId: string) {
  if (USE_MOCK) return mockAggregate(workspaceId);
  return teamSpaceApi.getAggregate(workspaceId);
}
```

---

## 6. Routing

### 6.1 Route Configuration

```typescript
// frontend/src/router/index.ts (update existing Team Space entry)
PAGE_CONFIGS.team = {
  subtitle: 'Workspace operating home for risks, pipeline, and ownership',
  actions: [{ key: 'ai-sync', label: 'AI SYNC', variant: 'ai' }],
};

COMPONENT_MAP.team = () => import('@/features/team-space/TeamSpaceView.vue');

// Team Space keeps the existing shell route namespace: /team
// Workspace scope is carried in route.query.workspaceId for deep links.
```

### 6.2 Navigation Vocabulary

| From | To | URL Pattern |
|------|----|-------------|
| Dashboard drill-in | Team Space | `/team?workspaceId=:workspaceId` |
| Pipeline counter "blocked specs" | Requirement Management | `/requirements?filter=blocked-specs&workspaceId=:id` |
| Pipeline blocker requirement | Requirement detail | `/requirements/:id` |
| Risk item — incident | Incident detail | `/incidents/:id` |
| Risk item — approval | Platform Center approvals stub | `/platform?view=approvals&workspaceId=:id` |
| Risk item — config drift | Platform Center | `/platform?view=config&workspaceId=:id&section=...` (stub) |
| Project card | Project Space | `/project-space/:projectId` (or `/project-stub/:projectId`) |
| Metric history | Report Center | `/reports/metric/:key?workspaceId=:id` (stub) |
| "View in Platform Center" | Platform Center | `/platform?view=config&workspaceId=:id&section=...` (stub) |
| "Manage in Access Management" | Access Management | `/platform?view=access&workspaceId=:id` (stub) |

---

## 7. API Contracts (Summary)

### 7.1 Endpoints

| Method | Path | Purpose |
|--------|------|---------|
| GET | `/api/v1/team-space/:workspaceId` | Aggregate first-paint |
| GET | `/api/v1/team-space/:workspaceId/summary` | Workspace summary |
| GET | `/api/v1/team-space/:workspaceId/operating-model` | Operating model |
| GET | `/api/v1/team-space/:workspaceId/members` | Members matrix |
| GET | `/api/v1/team-space/:workspaceId/templates` | Templates |
| GET | `/api/v1/team-space/:workspaceId/pipeline` | Requirement pipeline |
| GET | `/api/v1/team-space/:workspaceId/metrics` | Metrics |
| GET | `/api/v1/team-space/:workspaceId/risks` | Risk Radar |
| GET | `/api/v1/team-space/:workspaceId/projects` | Project Distribution |

Full contracts in [team-space-API_IMPLEMENTATION_GUIDE.md](contracts/team-space-API_IMPLEMENTATION_GUIDE.md).

### 7.2 Error Responses

| HTTP Status | Example Error Message |
|-------------|-----------------------|
| `400` | `Invalid workspaceId: INVALID` |
| `403` | `Workspace access denied: ws-other` |
| `404` | `Workspace ws-missing not found` |
| `500` | `Internal server error` |

---

## 8. Database Schema (Summary)

Two new tables introduced in migration `V7__create_team_space_tables.sql`:

- `risk_signals` — Workspace-scoped risk rows with category / severity / source / action link
- `metric_snapshots` — Workspace-scoped daily metric snapshots with previous value + trend

Full DDL and seed data in [team-space-data-model.md §5](../04-architecture/team-space-data-model.md).

---

## 9. Error and State Handling

### 9.1 Per-Card Isolation

Every card accepts a `SectionResult<T>` prop and renders:

- `section.data !== null`: data view
- `section.error !== null`: error view with retry button emitting `@retry`
- Success payload with empty arrays / counters: empty-state component (per-card specific)
- `loadingCards[cardKey] === true`: skeleton

### 9.2 Page-Level Errors

Only these trigger page-level errors:

- Authentication failure (redirect to login)
- Workspace access denied (redirect to Dashboard + banner)
- Both aggregate and all per-card calls fail (full-page error + reload)

### 9.3 AI Operation States

AI Command Panel actions do not affect Team Space card states. AI responses render inside the panel; clicking an evidence link navigates, not refreshes.

---

## 10. Validation and Error Handling

### 10.1 Frontend Validation

- `workspaceId` query value, when present, must match pattern `^ws-[a-z0-9\-]+$` (configurable).
- Invalid format → route to an invalid-workspace error page.

### 10.2 Backend Validation

- `@PathVariable workspaceId` validated via `@Pattern` annotation.
- `WorkspaceAccessGuard` enforces authorization pre-service.
- Controller returns 400 for malformed input, 403 for access denied, 404 for missing workspace, 500 for downstream projection failure (with per-projection degradation).

---

## 11. Integration Boundary

### 11.1 Upstream Dependencies

| Dependency | Team Space Usage |
|-----------|------------------|
| Shared App Shell | Context bar, AI Command Panel mount, breadcrumb |
| Dashboard slice | Drill-in entry point |
| Requirement slice | Query param vocabulary for drill-down |
| Incident slice | Deep-link navigation targets |

### 11.2 Downstream Dependencies

| Dependency | Team Space Produces |
|-----------|--------------------|
| Project Space (future) | Entry URL pattern `/project-space/:projectId` |
| Platform Center (future) | `/platform?view=config&workspaceId=:id&section=...` |
| Report Center (future) | `/reports/metric/:key?workspaceId=:id` |

### 11.3 Shared Components

- `<LineageBadge>` — used by Team Space, Platform Center (future), Dashboard config drift card (future)
- `<SdlcChainStrip>` — variant of Dashboard's chain; Team Space version scoped to single Workspace

### 11.4 Module Facades Consumed

- `WorkspaceRepositoryFacade`
- `MemberRepositoryFacade`
- `TemplateInheritanceFacade`
- `RequirementReadFacade`
- `IncidentReadFacade`
- `ProjectReadFacade`

Team Space never writes to upstream tables.

---

## 12. Testing Considerations

### 12.1 Backend Tests

- `TeamSpaceControllerTest` (MockMvc): endpoint responses, error codes, access control
- `TeamSpaceServiceTest`: projection orchestration, parallel fan-out, per-projection timeout degradation
- Each `*Projection` has its own unit test with a facade stub
- `WorkspaceAccessGuardTest`: authorization filter

### 12.2 Frontend Tests

- `TeamSpaceView.spec.ts`: mounts with mock store, verifies card rendering per state
- `teamSpaceStore.spec.ts`: action behavior, per-card retry, workspace switch
- `SdlcChainStrip.spec.ts`: always 11 nodes, Spec always highlighted
- Component-level snapshot tests for each card

---

## 13. Risks / Design Tradeoffs

| Risk / Tradeoff | Decision |
|----------------|---------|
| Aggregate endpoint size grows large | Start with single aggregate; if payload exceeds a threshold, split into smaller bundles |
| Inheritance resolution latency | Cache per-Workspace lineage for a short TTL (60s) |
| Metric snapshot freshness | Surface `lastRefreshed` timestamp on card; manual refresh hits live data path (V2) |
| Stub navigation for Platform Center / Project Space / Report Center | Feature flags; stub pages explicitly labeled "coming soon" |
| Shared components drift | Place shared primitives under `src/shared/`; version via ESLint import path rules |

---

## 14. Open Questions

Inherited from [team-space-spec.md](../03-spec/team-space-spec.md):

- Metric definitions for Governance Maturity and AI Participation
- Project lifecycle stage enum authoritative source
- Default pipeline blocker threshold (proposed 3 days)
- Team-Space-specific AI skill vs reuse of dashboard skills
- Permission-gated link strategy (hide vs disable)
