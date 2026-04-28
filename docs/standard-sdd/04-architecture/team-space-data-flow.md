# Team Space — Data Flow

## 1. Purpose and Traceability

This document defines the runtime data flows of the Team Space module: how Workspace-scoped data moves from storage, through projection services, across the API boundary, into the store, and onto the cards. It complements the static architecture view in [team-space-architecture.md](team-space-architecture.md) and the data shape defined in [team-space-data-model.md](team-space-data-model.md).

### Traceability

- Spec: [team-space-spec.md](../03-spec/team-space-spec.md)
- Stories: [team-space-stories.md](../02-user-stories/team-space-stories.md) — S1 through S12
- Requirements: [team-space-requirements.md](../01-requirements/team-space-requirements.md)

---

## 2. Runtime Data Flows

### 2.1 Aggregate First-Paint Load (Phase A: mock, Phase B: API)

```mermaid
sequenceDiagram
    participant User
    participant Shell as App Shell
    participant Router as Vue Router
    participant View as TeamSpaceView
    participant Store as teamSpaceStore
    participant API as teamSpaceApiClient
    participant Ctrl as TeamSpaceController
    participant Svc as TeamSpaceService
    participant Facades as Module Facades

    User->>Shell: Click Team Space (from Dashboard)
    Shell->>Router: push /team?workspaceId=ws-123
    Router->>View: Mount with workspaceId
    View->>Store: initWorkspace('ws-123')
    Store->>API: loadAggregate('ws-123')
    API->>Ctrl: GET /api/v1/team-space/ws-123

    Ctrl->>Ctrl: WorkspaceAccessGuard.check(user, 'ws-123')
    Ctrl->>Svc: loadAggregate('ws-123')

    par Parallel projections
        Svc->>Facades: WorkspaceRepositoryFacade.summary
        Svc->>Facades: ConfigInheritanceFacade.resolve
        Svc->>Facades: MemberRepositoryFacade.matrix
        Svc->>Facades: TemplateFacade.grouped
        Svc->>Facades: RequirementReadFacade.pipeline
        Svc->>Facades: MetricsProjection.read
        Svc->>Facades: RiskRadarProjection.read
        Svc->>Facades: ProjectReadFacade.distribution
    end

    Svc-->>Ctrl: TeamSpaceAggregateDto
    Ctrl-->>API: 200 OK + aggregate
    API-->>Store: Aggregate payload
    Store->>Store: Populate per-card state nodes
    Store-->>View: Reactive state update
    View-->>User: Cards hydrate in parallel
```

### 2.2 Workspace Switch Flow

```mermaid
sequenceDiagram
    participant User
    participant Shell
    participant Router
    participant View
    participant Store

    User->>Shell: Click Workspace switcher, select 'ws-456'
    Shell->>Router: push /team?workspaceId=ws-456
    Router->>View: Route query change (not remount)
    View->>Store: switchWorkspace('ws-456')
    Store->>Store: Clear all card states, set to loading
    Store->>Store: loadAggregate('ws-456')
    Note over Store,View: Same fetch path as 2.1
    View-->>User: Per-card skeletons → hydrated cards
```

### 2.3 Per-Card Refresh / Retry Flow

```mermaid
sequenceDiagram
    participant User
    participant Card as RiskRadarCard
    participant Store
    participant API
    participant Ctrl

    User->>Card: Click retry (card in error state)
    Card->>Store: retryCard('risks')
    Store->>Store: Set cards.risks.state = 'loading'
    Store->>API: loadCard('ws-123', 'risks')
    API->>Ctrl: GET /api/v1/team-space/ws-123/risks
    Ctrl-->>API: 200 OK + TeamRiskRadarDto
    API-->>Store: Risk payload
    Store->>Store: Set cards.risks = ready
    Store-->>Card: Reactive update
    Card-->>User: Risks rendered
```

### 2.4 Pipeline Counter → Requirement Management Drill-Down

```mermaid
sequenceDiagram
    participant User
    participant Card as PipelineCard
    participant Shell
    participant Router
    participant ReqView as RequirementListView

    User->>Card: Click "3 blocked Specs"
    Card->>Shell: registerBreadcrumb('Team Space (ws-123)')
    Card->>Router: push /requirements?filter=blocked-specs&workspaceId=ws-123
    Router->>ReqView: Mount with filter + workspaceId
    ReqView-->>User: Pre-filtered requirement list
    Note over User,Shell: Browser Back returns to Team Space
```

### 2.5 Risk Radar Item → Incident Detail Drill-Down

```mermaid
sequenceDiagram
    participant User
    participant Card as RiskRadarCard
    participant Shell
    participant Router

    User->>Card: Click "P1 incident: payment-service down"
    Card->>Shell: registerBreadcrumb('Team Space (ws-123)')
    Card->>Router: push /incidents/INC-789
    Router-->>User: Incident detail page
```

### 2.6 Project Distribution → Project Space (with Fallback)

```mermaid
sequenceDiagram
    participant User
    participant Card as ProjectDistributionCard
    participant Router
    participant FeatureFlag

    User->>Card: Click project card
    Card->>FeatureFlag: isEnabled('project-space-link')
    alt Project Space route enabled
        Card->>Router: push /project-space/proj-42
    else Project Space not yet implemented
        Card->>Router: push /project-stub/proj-42
    end
```

### 2.7 AI Command Panel Context Projection

```mermaid
sequenceDiagram
    participant View as TeamSpaceView
    participant Shell
    participant Panel as AI Command Panel

    View->>Shell: emit 'page.context' { page: 'team-space', workspaceId, topRisks, pipelineCounters }
    Shell->>Panel: dispatch TeamSpaceContext
    Panel->>Panel: Load Team-Space-appropriate suggested prompts
    Panel-->>View: Panel ready
```

### 2.8 AI Skill Invocation Recorded as Skill Execution

```mermaid
sequenceDiagram
    participant User
    participant Panel as AI Command Panel
    participant Skill as SkillRuntime
    participant Audit as Audit Pipeline

    User->>Panel: "Summarize governance posture"
    Panel->>Skill: invoke(team-space-summarizer, { workspaceId })
    Skill->>Audit: record Skill Execution (input, user, timestamp)
    Skill-->>Panel: Summary + evidence refs
    Skill->>Audit: record output, duration
    Panel-->>User: Summary with evidence links
```

### 2.9 Coverage Gap Computation

```mermaid
sequenceDiagram
    participant Svc as MemberProjection
    participant Facade as MemberRepositoryFacade
    participant Sched as OncallSchedule

    Svc->>Facade: listMembers(workspaceId)
    Svc->>Sched: getRotation(workspaceId, nextWindow)
    Svc->>Svc: Compute gaps (no primary, no backup, role unfilled)
    Svc-->>Svc: Return MemberMatrixDto with gaps[]
```

### 2.10 Metric Snapshot Refresh (Scheduled)

```mermaid
sequenceDiagram
    participant Cron as @Scheduled
    participant Job as MetricsSnapshotJob
    participant Repo as MetricSnapshotRepository
    participant DB

    Cron->>Job: run nightly @ 01:00
    loop For each active Workspace
        Job->>Job: compute delivery efficiency
        Job->>Job: compute quality
        Job->>Job: compute stability
        Job->>Job: compute governance maturity
        Job->>Job: compute AI participation
        Job->>Repo: save(MetricSnapshot{workspaceId, period, values})
        Repo->>DB: insert row
    end
```

### 2.11 Risk Signal Refresh (Scheduled)

```mermaid
sequenceDiagram
    participant Cron as @Scheduled
    participant Job as RiskRadarJob
    participant Sources as Risk Sources
    participant Repo as RiskSignalRepository

    Cron->>Job: run every 15 min
    loop For each active Workspace
        Job->>Sources: query project health
        Job->>Sources: query approval backlog
        Job->>Sources: query dependency blocks
        Job->>Sources: query config drift
        Job->>Sources: query incident hotspots
        Job->>Repo: upsert(RiskSignal rows)
    end
```

---

## 3. Error Isolation Strategy

### Aggregate Endpoint: Partial Degradation

When the aggregate endpoint is called, each projection runs in parallel with a 500ms budget. A projection that errors or times out returns a `SectionResult` with `data = null` and a non-null `error`; other projections still return their normal `data` with `error = null`.

```mermaid
graph TD
    AGG[loadAggregate] --> P1[WorkspaceProjection]
    AGG --> P2[OperatingModelProjection]
    AGG --> P3[MemberProjection]
    AGG --> P4[TemplateProjection]
    AGG --> P5[PipelineProjection]
    AGG --> P6[MetricsProjection]
    AGG --> P7[RiskRadarProjection]
    AGG --> P8[ProjectProjection]

    P1 -->|ok| R1[Ready]
    P2 -->|ok| R2[Ready]
    P3 -->|timeout| R3[Error: timeout]
    P4 -->|ok| R4[Ready]
    P5 -->|ok| R5[Ready]
    P6 -->|ok| R6[Ready]
    P7 -->|ok| R7[Ready]
    P8 -->|ok| R8[Ready]

    R1 & R2 & R3 & R4 & R5 & R6 & R7 & R8 --> DTO[TeamSpaceAggregateDto]
```

Each card in the DTO carries its own section envelope — the frontend combines that `data` / `error` payload with local loading flags to render skeleton / error / ready states independently.

### Section Envelope

```typescript
interface SectionResult<T> {
  readonly data: T | null;
  readonly error: string | null;
}
```

### Page-Level Errors

| Condition | Behavior |
|-----------|---------|
| Auth failure | Redirect to login |
| Workspace access denied | Redirect to Dashboard + error banner |
| Invalid `workspaceId` format | 400 + inline page error |
| Both aggregate AND all per-card endpoints fail | Full-page error with reload button |

### Retry Strategy

- Per-card retry: manual, triggered by user click on error state.
- No automatic retry in V1.
- Aggregate endpoint timeout: 2s total; if exceeded, frontend falls back to per-card endpoints.

---

## 4. State Machine

### Per-Card State Transitions

```mermaid
stateDiagram-v2
    [*] --> Loading
    Loading --> Ready: data arrived (non-empty)
    Loading --> Empty: data arrived (empty)
    Loading --> Error: fetch error
    Error --> Loading: retry
    Ready --> Loading: workspace switch / manual refresh
    Empty --> Loading: workspace switch / manual refresh
```

### Card-Level State Values

| Card | Empty example | Error example |
|------|---------------|---------------|
| WorkspaceSummary | N/A (always has identity) | "Failed to load workspace summary" |
| OperatingModel | N/A (always has lineage resolution) | "Failed to resolve configuration" |
| Members | "No members yet" | "Failed to load members" |
| Templates | "No overrides" (partial empty) | "Failed to load templates" |
| Pipeline | "No requirements yet" | "Failed to load pipeline" |
| Metrics | "No snapshots yet" | "Failed to load metrics" |
| Risks | "All green" | "Failed to load risk radar" |
| Projects | "No projects yet" | "Failed to load projects" |

---

## 5. Refresh Strategy

### V1: On-Load + Manual Refresh

- Aggregate fetched on page mount and on Workspace switch.
- Per-card manual refresh via action button on each card.
- No automatic polling.

### Metric / Risk Backing Data

- Metrics: nightly snapshot refresh; displayed with `lastRefreshed` timestamp.
- Risks: 15-minute cron refresh; displayed with `lastRefreshed` timestamp.

### Future (V2+)

- WebSocket push when risk signal changes.
- Configurable polling per-card.
- Optimistic update when Platform Center changes a template that would affect Team Space.

---

## 6. API Client Chain

### Full Request Path

```mermaid
graph LR
    Card --> Store
    Store --> Client[teamSpaceApiClient]
    Client --> Axios[axios instance]
    Axios --> Proxy[Vite dev proxy]
    Proxy --> Ctrl[TeamSpaceController]
    Ctrl --> Svc[TeamSpaceService]
    Svc --> Facades
    Facades --> DB[Database]
```

### Layer Responsibilities

| Layer | Responsibility |
|-------|---------------|
| Card | Dispatches store action; subscribes to state |
| Store (Pinia) | Owns per-card state nodes; dispatches API calls; unwraps envelopes |
| teamSpaceApiClient | Typed wrapper over axios; returns typed DTOs |
| axios | HTTP transport; response envelope interceptor |
| Vite dev proxy | `/api` → backend origin in dev |
| Controller | Endpoint surface; authorization; validation |
| Service | Orchestration; parallel projection fan-out |
| Projections | Read domain data via facades |
| Facades | Repository abstraction over shared tables |

### Vite Proxy Configuration

```ts
// vite.config.ts
server: {
  proxy: {
    '/api': {
      target: 'http://localhost:8080',
      changeOrigin: true,
    },
  },
}
```

### Mock Toggle Pattern

```ts
import { fetchJson } from '@/shared/api/client';

const USE_MOCK = import.meta.env.DEV && !import.meta.env.VITE_USE_BACKEND;

export async function loadAggregate(workspaceId: string): Promise<TeamSpaceAggregateDto> {
  if (USE_MOCK) return mockAggregate(workspaceId);
  return fetchJson<TeamSpaceAggregateDto>(`/team-space/${workspaceId}`);
}
```

---

## 7. Frontend Type to Backend DTO Mapping

| Frontend Type | Backend DTO | Source Projection |
|--------------|-------------|-------------------|
| `WorkspaceSummary` | `WorkspaceSummaryDto` | `WorkspaceProjection` |
| `TeamOperatingModel` | `TeamOperatingModelDto` | `OperatingModelProjection` |
| `MemberMatrix` | `MemberMatrixDto` | `MemberProjection` |
| `TeamDefaultTemplates` | `TeamDefaultTemplatesDto` | `TemplateInheritanceProjection` |
| `RequirementPipeline` | `RequirementPipelineDto` | `RequirementPipelineProjection` |
| `TeamMetrics` | `TeamMetricsDto` | `MetricsProjection` |
| `TeamRiskRadar` | `TeamRiskRadarDto` | `RiskRadarProjection` |
| `ProjectDistribution` | `ProjectDistributionDto` | `ProjectDistributionProjection` |
| `TeamSpaceAggregate` | `TeamSpaceAggregateDto` | `TeamSpaceService` |
| `Lineage` | `LineageDto` | shared primitive |
| `SectionResult<T>` | `SectionResultDto<T>` | shared envelope |

See [team-space-data-model.md](team-space-data-model.md) for full type definitions.

---

## 8. Phase A vs Phase B Data Sources

| Phase | Aggregate Source | Per-Card Source |
|-------|------------------|-----------------|
| Phase A (frontend-first) | `src/features/team-space/mock/aggregate.mock.ts` | same mock file |
| Phase B (full-stack) | `GET /api/v1/team-space/:id` | `GET /api/v1/team-space/:id/{card}` |

Mock structure mirrors the backend DTO shape exactly; swapping via `VITE_USE_BACKEND` does not require rerendering components.
