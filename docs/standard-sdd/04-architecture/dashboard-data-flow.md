# Dashboard Data Flow

## Purpose

This document details the runtime data flows for the Dashboard / Control Tower page,
covering both Phase A (mocked) and Phase B (full-stack) execution paths.

## Traceability

- Architecture: [dashboard-architecture.md](dashboard-architecture.md)
- Design: [dashboard-design.md](../05-design/dashboard-design.md)
- Spec: [dashboard-spec.md](../03-spec/dashboard-spec.md)

---

## 1. Page Load Flow (Phase A — Mocked Data)

```mermaid
sequenceDiagram
    participant User
    participant Router as Vue Router
    participant DV as DashboardView
    participant DS as dashboardStore
    participant Mock as mockData.ts

    User->>Router: Navigate to /
    Router->>DV: Mount DashboardView
    DV->>DS: onMounted() calls fetchSummary()
    DS->>DS: Set isLoading = true
    DS->>Mock: Import MOCK_DASHBOARD_DATA
    Mock-->>DS: DashboardSummary (complete object)
    DS->>DS: Set summary = data
    DS->>DS: Set isLoading = false
    DS-->>DV: Reactive state update
    DV->>DV: Render 8 cards from summary sections
```

Phase A bypasses the API layer entirely. The store imports static data from
`mockData.ts` and returns it as if it came from the network. Each section is
wrapped in a `SectionResult<T>` envelope, so the rendering path is identical
whether the data is mocked or live.

---

## 2. Page Load Flow (Phase B — Full Stack)

```mermaid
sequenceDiagram
    participant User
    participant Router as Vue Router
    participant DV as DashboardView
    participant DS as dashboardStore
    participant API as dashboardApi.ts
    participant Client as fetchJson (client.ts)
    participant Backend as DashboardController
    participant Service as DashboardService
    participant DB as H2 / Oracle

    User->>Router: Navigate to /
    Router->>DV: Mount DashboardView
    DV->>DS: onMounted() calls fetchSummary()
    DS->>DS: Set isLoading = true
    DS->>API: dashboardApi.getSummary()
    API->>Client: fetchJson('/dashboard/summary')
    Client->>Backend: GET /api/v1/dashboard/summary
    Backend->>Service: getDashboardSummary(workspaceId)
    Service->>DB: Query aggregate metrics (V2+)
    DB-->>Service: Raw metrics
    Service->>Service: Assemble DashboardSummaryDto
    Service-->>Backend: DashboardSummaryDto
    Backend-->>Client: ApiResponse with DashboardSummaryDto
    Client->>Client: Unwrap envelope, throw on error
    Client-->>API: DashboardSummary
    API-->>DS: DashboardSummary
    DS->>DS: Set summary = data
    DS->>DS: Set isLoading = false
    DS-->>DV: Reactive state update
    DV->>DV: Render 8 cards from summary sections
```

---

## 3. Section-Level Error Isolation

```mermaid
sequenceDiagram
    participant Service as DashboardService
    participant Backend as DashboardController

    Service->>Service: Load SDLC health OK
    Service->>Service: Load delivery metrics OK
    Service->>Service: Load quality metrics FAIL
    Service->>Service: Wrap failure in SectionResult.fail("timeout")
    Service->>Service: Continue loading remaining sections
    Service-->>Backend: DashboardSummaryDto (quality section has error)
    Backend-->>Backend: Return 200 with partial failure
```

```mermaid
sequenceDiagram
    participant DS as dashboardStore
    participant DV as DashboardView
    participant QC as QualityTestingCard
    participant DC as DeliveryMetricsCard

    DS-->>DV: summary with qualityMetrics.error set
    DV->>QC: Pass qualityMetrics section
    QC->>QC: Detect error != null, render error state
    DV->>DC: Pass deliveryMetrics section
    DC->>DC: Detect data != null, render normally
```

Each card reads its own `SectionResult<T>` independently. A failure in one
section does not cascade to other cards. The top-level API returns 200 even
when individual sections fail.

---

## 4. Navigation Flow

```mermaid
sequenceDiagram
    participant User
    participant Node as SdlcStageNode
    participant Chain as SdlcChainHealth
    participant DV as DashboardView
    participant Router as Vue Router
    participant Shell as Shared App Shell

    User->>Node: Click "Incident" node
    Node->>Chain: Emit @click(stageKey)
    Chain->>DV: Emit @navigate(routePath)
    DV->>Router: router.push('/incidents')
    Router->>Shell: Route change
    Shell->>Shell: Preserve workspace context
    Shell->>Shell: Load target module view
```

### Navigation Routes

| Source Component | User Action | Target Route |
|------------------|-------------|-------------|
| SdlcChainHealth | Click any stage node | `stage.routePath` |
| StabilityIncidentCard | Click incident badge | `/incidents` |
| GovernanceTrustCard | Click governance indicator | `/platform` |
| RecentActivityStream | Click "View All" | `/platform` |

---

## 5. Store State Machine

```mermaid
graph LR
    IDLE["idle<br/>summary: null<br/>isLoading: false<br/>error: null"]
    LOADING["loading<br/>summary: null<br/>isLoading: true<br/>error: null"]
    LOADED["loaded<br/>summary: DashboardSummary<br/>isLoading: false<br/>error: null"]
    ERROR["error<br/>summary: null<br/>isLoading: false<br/>error: string"]

    IDLE -->|"fetchSummary()"| LOADING
    LOADING -->|"success"| LOADED
    LOADING -->|"failure"| ERROR
    LOADED -->|"reload()"| LOADING
    ERROR -->|"retry()"| LOADING
```

---

## 6. Data Refresh Strategy

| Trigger | Behavior |
|---------|----------|
| Page mount | Full load via `store.fetchSummary()` |
| Manual refresh | Page reload (V1); future: dedicated refresh button |
| Workspace switch | `workspaceStore` watcher triggers `dashboardStore.fetchSummary()` |
| Background | Not implemented in V1; future: short-TTL polling or WebSocket |

---

## 7. API Client Chain

```mermaid
graph LR
    subgraph Frontend["Frontend Layer"]
        DV["DashboardView"]
        DS["dashboardStore"]
        DA["dashboardApi.ts"]
        CL["client.ts<br/>fetchJson"]
    end

    subgraph Network["Network"]
        PROXY["Vite dev proxy<br/>/api -> localhost:8080"]
    end

    subgraph Backend["Backend Layer"]
        CTRL["DashboardController"]
        SVC["DashboardService"]
    end

    DV -->|reads| DS
    DS -->|calls| DA
    DA -->|calls| CL
    CL -->|HTTP GET| PROXY
    PROXY -->|proxy| CTRL
    CTRL -->|delegates| SVC
    SVC -->|returns DTO| CTRL
    CTRL -->|ApiResponse JSON| CL
```

### Frontend API Chain

1. `DashboardView.vue` reads reactive state from `dashboardStore`
2. `dashboardStore.fetchSummary()` calls `dashboardApi.getSummary()`
3. `dashboardApi.ts` calls `fetchJson<DashboardSummary>('/dashboard/summary')`
4. `client.ts` sends `GET /api/v1/dashboard/summary`, unwraps `ApiResponse` envelope

### Backend Processing Chain

1. `DashboardController` receives GET request
2. Delegates to `DashboardService.getDashboardSummary(workspaceId)`
3. Service assembles `DashboardSummaryDto` (seed data in V1; aggregate queries in V2+)
4. Controller wraps in `ApiResponse.ok(dto)` and returns JSON
