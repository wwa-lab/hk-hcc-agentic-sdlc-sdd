# Project Space — Data Flow

## 1. Purpose and Traceability

This document defines the runtime data flows of the Project Space module: how Project-scoped data moves from storage, through projection services, across the API boundary, into the store, and onto the summary bar and cards. It complements the static architecture view in [project-space-architecture.md](project-space-architecture.md) and the data shape defined in [project-space-data-model.md](project-space-data-model.md).

### Traceability

- Spec: [project-space-spec.md](../03-spec/project-space-spec.md)
- Stories: [project-space-stories.md](../02-user-stories/project-space-stories.md) — S1 through S12
- Requirements: [project-space-requirements.md](../01-requirements/project-space-requirements.md)

---

## 2. Runtime Data Flows

### 2.1 Aggregate First-Paint Load (Phase A: mock, Phase B: API)

```mermaid
sequenceDiagram
    participant User
    participant Shell as App Shell
    participant Router as Vue Router
    participant View as ProjectSpaceView
    participant Store as projectSpaceStore
    participant API as projectSpaceApiClient
    participant Ctrl as ProjectSpaceController
    participant Svc as ProjectSpaceService
    participant Facades as Module Facades

    User->>Shell: Click project card (from Team Space)
    Shell->>Router: push /project-space/proj-8821?workspaceId=ws-123
    Router->>View: Mount with projectId
    View->>Store: initProject('proj-8821')
    Store->>API: loadAggregate('proj-8821')
    API->>Ctrl: GET /api/v1/project-space/proj-8821

    Ctrl->>Ctrl: ProjectAccessGuard.check(user, 'proj-8821')
    Ctrl->>Svc: loadAggregate('proj-8821')

    par Parallel projections
        Svc->>Facades: ProjectRepositoryFacade.summary
        Svc->>Facades: MemberRepositoryFacade.leadership
        Svc->>Facades: RequirementReadFacade + IncidentReadFacade (chain nodes)
        Svc->>Facades: MilestoneRepository.list
        Svc->>Facades: ProjectDependencyRepository.list
        Svc->>Facades: RiskSignalRepository.listByProject
        Svc->>Facades: EnvironmentRepository + DeploymentRepository
    end

    Svc-->>Ctrl: ProjectSpaceAggregateDto
    Ctrl-->>API: 200 OK + aggregate
    API-->>Store: Aggregate payload
    Store->>Store: Populate per-card state nodes
    Store-->>View: Reactive state update
    View-->>User: Summary bar + cards hydrate in parallel
```

### 2.2 Project Switch Flow (with Workspace Reconciliation)

```mermaid
sequenceDiagram
    participant User
    participant Shell
    participant Router
    participant View
    participant Store

    User->>Shell: Select Project from shared context bar
    Shell->>Shell: Resolve new project -> workspaceId, applicationId
    alt workspaceId differs from current shell context
        Shell->>Shell: Update currentWorkspace + currentApplication
        Shell->>User: Brief toast "Switched to Workspace X"
    end
    Shell->>Router: push /project-space/:newId?workspaceId=:ws
    Router->>View: Route param change (not remount)
    View->>Store: switchProject(':newId')
    Store->>Store: Clear all card states, set to loading
    Store->>Store: loadAggregate(':newId')
    Note over Store,View: Same fetch path as 2.1
    View-->>User: Per-card skeletons -> hydrated cards
```

### 2.3 Per-Card Refresh / Retry Flow

```mermaid
sequenceDiagram
    participant User
    participant Card as EnvironmentMatrixCard
    participant Store
    participant API
    participant Ctrl

    User->>Card: Click retry (card in error state)
    Card->>Store: retryCard('environments')
    Store->>Store: Set cards.environments.state = 'loading'
    Store->>API: loadCard('proj-8821', 'environments')
    API->>Ctrl: GET /api/v1/project-space/proj-8821/environments
    Ctrl-->>API: 200 OK + EnvironmentMatrixDto
    API-->>Store: Environment payload
    Store->>Store: Set cards.environments = ready
    Store-->>Card: Reactive update
    Card-->>User: Environments rendered
```

### 2.4 SDLC Chain Node -> Requirement Management Drill-Down

```mermaid
sequenceDiagram
    participant User
    participant Card as SdlcDeepLinksCard
    participant Shell
    participant Router
    participant ReqView as RequirementListView

    User->>Card: Click Spec node
    Card->>Shell: registerBreadcrumb('Project Space (proj-8821)')
    Card->>Router: push /requirements?projectId=proj-8821&workspaceId=ws-123&node=spec
    Router->>ReqView: Mount with filter + projectId
    ReqView-->>User: Pre-filtered Spec list
    Note over User,Shell: Browser Back returns to Project Space
```

### 2.5 Risk Registry -> Incident Detail Drill-Down

```mermaid
sequenceDiagram
    participant User
    participant Card as RiskRegistryCard
    participant Shell
    participant Router

    User->>Card: Click "P1 incident: payment-service down"
    Card->>Shell: registerBreadcrumb('Project Space (proj-8821)')
    Card->>Router: push /incidents/INC-789
    Router-->>User: Incident detail page
```

### 2.6 Dependency Row -> Adjacent Project Space (with Fallback)

```mermaid
sequenceDiagram
    participant User
    participant Card as OperationalDependencyMapCard
    participant Router
    participant FeatureFlag

    User->>Card: Click dependency "Identity-Service-V2"
    Card->>FeatureFlag: isEnabled('project-space.enabled')
    alt dependency is a registered Project
        Card->>Router: push /project-space/proj-identity-v2
    else dependency is external
        Card->>Card: Show "External — no project link"
    end
```

### 2.7 Milestone Card -> Project Management (with Feature Flag)

```mermaid
sequenceDiagram
    participant User
    participant Card as MilestoneExecutionHubCard
    participant Router
    participant FeatureFlag

    User->>Card: Click "Manage in Project Management"
    Card->>FeatureFlag: isEnabled('project-management-link')
    alt Project Management route enabled
        Card->>Router: push /project-management?projectId=proj-8821
    else Project Management not yet implemented
        Card->>Card: Show disabled CTA with "Coming soon" tooltip
    end
```

### 2.8 Environment Tile -> Deployment Management

```mermaid
sequenceDiagram
    participant User
    participant Tile as EnvironmentTile
    participant Router
    participant FeatureFlag

    User->>Tile: Click STAGING env
    Tile->>FeatureFlag: isEnabled('deployment-link')
    alt Deployment route enabled
        Tile->>Router: push /deployment?projectId=proj-8821&envId=env-staging-1
    else Deployment not yet implemented
        Tile->>Tile: Show disabled CTA with "Coming soon" tooltip
    end
```

### 2.9 AI Command Panel Context Projection

```mermaid
sequenceDiagram
    participant View as ProjectSpaceView
    participant ShellUi as shellUiStore
    participant Panel as AI Command Panel

    View->>ShellUi: setAiPanelContent(ProjectSpaceContext-derived content)
    ShellUi->>Panel: render updated Project Space content
    Panel->>Panel: Load Project-Space-appropriate suggested prompts
    Panel-->>View: Panel ready
```

### 2.10 AI Skill Invocation Recorded as Skill Execution

```mermaid
sequenceDiagram
    participant User
    participant Panel as AI Command Panel
    participant Skill as SkillRuntime
    participant Audit as Audit Pipeline

    User->>Panel: "Summarize why milestone M3 is at risk"
    Panel->>Skill: invoke(project-space-summarizer, { projectId })
    Skill->>Audit: record Skill Execution (input, user, timestamp)
    Skill-->>Panel: Summary + evidence refs
    Skill->>Audit: record output, duration
    Panel-->>User: Summary with evidence links
```

### 2.11 Aggregate Health Computation

```mermaid
sequenceDiagram
    participant Svc as ProjectSummaryProjection
    participant RS as RiskSignalRepository
    participant IN as IncidentReadFacade
    participant MS as MilestoneRepository
    participant AP as ApprovalReadFacade

    Svc->>RS: countCriticalHigh(projectId)
    Svc->>IN: countOpenIncidents(projectId)
    Svc->>MS: countSlippedOrAtRisk(projectId)
    Svc->>AP: countPendingApprovals(projectId)
    Svc->>Svc: derive health = f(inputs)
    Svc-->>Svc: Return ProjectSummaryDto with healthAggregate + healthFactors[]
```

### 2.12 Version Drift Computation

```mermaid
sequenceDiagram
    participant Svc as EnvironmentMatrixProjection
    participant Repo as DeploymentRepository
    participant Git as SourceControlFacade

    Svc->>Repo: latestDeployment(projectId, env=PROD)
    Svc->>Repo: latestDeployment(projectId, env=STAGING)
    Svc->>Git: commitDelta(prodVersion, stagingVersion)
    Svc->>Svc: classify drift band (NONE / MINOR / MAJOR)
    Svc-->>Svc: Return EnvironmentMatrixDto with driftIndicator
```

---

## 3. Error Isolation Strategy

### Aggregate Endpoint: Partial Degradation

When the aggregate endpoint is called, each projection runs in parallel with a 500ms budget. A projection that errors or times out returns a `SectionResult` with `data = null` and a non-null `error`; other projections still return their normal `data` with `error = null`.

```mermaid
graph TD
    AGG[loadAggregate] --> P1[ProjectSummaryProjection]
    AGG --> P2[LeadershipProjection]
    AGG --> P3[ChainNodeProjection]
    AGG --> P4[MilestoneProjection]
    AGG --> P5[DependencyProjection]
    AGG --> P6[RiskRegistryProjection]
    AGG --> P7[EnvironmentMatrixProjection]

    P1 -->|ok| R1[Ready]
    P2 -->|ok| R2[Ready]
    P3 -->|ok| R3[Ready]
    P4 -->|timeout| R4[Error: timeout]
    P5 -->|ok| R5[Ready]
    P6 -->|ok| R6[Ready]
    P7 -->|ok| R7[Ready]

    R1 & R2 & R3 & R4 & R5 & R6 & R7 --> DTO[ProjectSpaceAggregateDto]
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
| Project access denied | Redirect to Team Space + error banner |
| Invalid `projectId` format | 400 + inline page error |
| Project not found | 404 + inline page error with Team Space link |
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
    Ready --> Loading: project switch / manual refresh
    Empty --> Loading: project switch / manual refresh
```

### Card-Level State Values

| Card | Empty example | Error example |
|------|---------------|---------------|
| ProjectSummary | N/A (always has identity) | "Failed to load project summary" |
| Leadership | "No roles assigned yet" | "Failed to load leadership" |
| SdlcDeepLinks | N/A (11 nodes always render) | "Failed to load chain health" |
| Milestones | "No milestones defined" | "Failed to load milestones" |
| Dependencies | "No dependencies configured" | "Failed to load dependency map" |
| Risks | "All green" | "Failed to load risk registry" |
| Environments | "No environments configured" | "Failed to load environment matrix" |

---

## 5. Refresh Strategy

### V1: On-Load + Manual Refresh

- Aggregate fetched on page mount and on Project switch.
- Per-card manual refresh via action button on each card.
- No automatic polling.

### Backing Data Refresh

- Risk signals: 15-minute cron refresh (inherited from Team Space `RiskRadarJob`).
- Deployment records: written on deploy completion by Deployment Management (when available); V1 uses migration-seeded rows + manual admin update.
- Chain node counts: computed on-demand from lifecycle facades.
- Milestones / dependencies / environments: managed by Project Management / Deployment Management when those slices land; V1 reads directly from Project Space-owned tables.

### Future (V2+)

- WebSocket push on milestone status change or deployment completion.
- Configurable polling per-card.
- Event stream integration with Deployment Management.

---

## 6. API Client Chain

### Full Request Path

```mermaid
graph LR
    Card --> Store
    Store --> Client[projectSpaceApiClient]
    Client --> Axios[axios instance]
    Axios --> Proxy[Vite dev proxy]
    Proxy --> Ctrl[ProjectSpaceController]
    Ctrl --> Svc[ProjectSpaceService]
    Svc --> Facades
    Facades --> DB[Database]
```

### Layer Responsibilities

| Layer | Responsibility |
|-------|---------------|
| Card | Dispatches store action; subscribes to state |
| Store (Pinia) | Owns per-card state nodes; dispatches API calls; unwraps envelopes |
| projectSpaceApiClient | Typed wrapper over axios; returns typed DTOs |
| axios | HTTP transport; response envelope interceptor |
| Vite dev proxy | `/api` -> backend origin in dev |
| Controller | Endpoint surface; authorization; validation |
| Service | Orchestration; parallel projection fan-out |
| Projections | Read domain data via facades or own repositories |
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

export async function loadAggregate(projectId: string): Promise<ProjectSpaceAggregateDto> {
  if (USE_MOCK) return mockAggregate(projectId);
  return fetchJson<ProjectSpaceAggregateDto>(`/project-space/${projectId}`);
}
```

---

## 7. Frontend Type to Backend DTO Mapping

| Frontend Type | Backend DTO | Source Projection |
|--------------|-------------|-------------------|
| `ProjectSummary` | `ProjectSummaryDto` | `ProjectSummaryProjection` |
| `LeadershipOwnership` | `LeadershipOwnershipDto` | `LeadershipProjection` |
| `SdlcChainState` | `SdlcChainDto` | `ChainNodeProjection` |
| `MilestoneHub` | `MilestoneHubDto` | `MilestoneProjection` |
| `DependencyMap` | `DependencyMapDto` | `DependencyProjection` |
| `RiskRegistry` | `RiskRegistryDto` | `RiskRegistryProjection` |
| `EnvironmentMatrix` | `EnvironmentMatrixDto` | `EnvironmentMatrixProjection` |
| `ProjectSpaceAggregate` | `ProjectSpaceAggregateDto` | `ProjectSpaceService` |
| `SectionResult<T>` | `SectionResultDto<T>` | shared envelope |

See [project-space-data-model.md](project-space-data-model.md) for full type definitions.

---

## 8. Phase A vs Phase B Data Sources

| Phase | Aggregate Source | Per-Card Source |
|-------|------------------|-----------------|
| Phase A (frontend-first) | `src/features/project-space/mock/aggregate.mock.ts` | same mock file |
| Phase B (full-stack) | `GET /api/v1/project-space/:id` | `GET /api/v1/project-space/:id/{card}` |

Mock structure mirrors the backend DTO shape exactly; swapping via `VITE_USE_BACKEND` does not require rerendering components.
