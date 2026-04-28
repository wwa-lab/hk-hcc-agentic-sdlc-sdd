# Dashboard / Control Tower Architecture

## Purpose

This document defines the technical architecture for the Dashboard / Control Tower
page — the main landing page of the Agentic SDLC Control Tower.

## 1. Architecture Goal

Build a dashboard that aggregates cross-stage SDLC signals into a card-based
layout, tells a delivery story through the 11-node chain, and provides drill-down
navigation to downstream module pages.

## 2. System Context

The dashboard is a page view rendered within the shared app shell. It consumes
aggregate data from the backend (Phase B) or mocked data (Phase A).

```mermaid
graph LR
    USER["SDLC Operator<br/>any role"]
    SHELL["Shared App Shell<br/>Nav + Context + AI Panel"]
    DASH["Dashboard View<br/>Card Grid + SDLC Chain"]
    API["Spring Boot API<br/>GET /api/v1/dashboard/summary"]
    DB["H2 / Oracle<br/>Aggregated metrics"]

    USER -->|navigates to /| SHELL
    SHELL -->|router-view| DASH
    DASH -->|fetch| API
    API -->|query| DB
```

## 3. Technology Stack

Inherits from the shared app shell architecture:

| Layer | Technology |
|-------|-----------|
| Frontend framework | Vue 3 (Composition API, `<script setup>`) |
| Build tool | Vite |
| Routing | Vue Router |
| Client state | Pinia |
| Backend framework | Spring Boot (Java 21) |
| ORM | JPA / Hibernate |
| Database | H2 (local) / Oracle (prod) |

Dashboard-specific additions:

| Layer | Technology | Rationale |
|-------|-----------|-----------|
| Charts (future) | TBD (e.g., ECharts, Chart.js) | V1 uses styled HTML/CSS; charting library deferred |
| API layer | `fetchJson<T>` from shared client | Reuses existing API infrastructure |

## 4. Component Breakdown

```mermaid
graph TD
    subgraph Shell["Shared App Shell"]
        NAV["PrimaryNav"]
        CTX["TopContextBar"]
        ACT["GlobalActionBar"]
        AI["AiCommandPanel"]
        RV["router-view"]
    end

    subgraph Dashboard["DashboardView"]
        CHAIN["SdlcChainHealth"]
        DM["DeliveryMetricsCard"]
        AIP["AiParticipationCard"]
        QT["QualityTestingCard"]
        SI["StabilityIncidentCard"]
        GOV["GovernanceTrustCard"]
        VS["ValueStoryCard"]
        RA["RecentActivityStream"]
    end

    RV --> CHAIN
    CHAIN --> DM
    CHAIN --> AIP
    DM --> QT
    AIP --> SI
    QT --> GOV
    SI --> VS
    GOV --> RA
```

### Component Responsibilities

| Component | Responsibility | Data Source |
|-----------|---------------|-------------|
| `DashboardView` | Layout grid, data fetching orchestration | Dashboard store |
| `SdlcChainHealth` | 11-node pipeline visualization | `sdlcHealth[]` |
| `DeliveryMetricsCard` | Lead time, deploy freq, completion | `deliveryMetrics` |
| `AiParticipationCard` | AI usage, adoption, time saved | `aiParticipation` |
| `QualityTestingCard` | Build, test, defect, spec coverage | `qualityMetrics` |
| `StabilityIncidentCard` | Incidents, MTTR, change failure | `stabilityMetrics` |
| `GovernanceTrustCard` | Template reuse, drift, audit, policy | `governanceMetrics` |
| `ValueStoryCard` | AI value proof headline and metrics | `valueStory` |
| `RecentActivityStream` | Chronological activity list | `recentActivity` |

## 5. Data Flow

```mermaid
sequenceDiagram
    participant User
    participant Router
    participant DashboardView
    participant DashboardStore
    participant API as API Client
    participant Backend as Spring Boot

    User->>Router: Navigate to /
    Router->>DashboardView: Mount component
    DashboardView->>DashboardStore: load()
    DashboardStore->>API: fetchJson /dashboard/summary
    API->>Backend: GET /api/v1/dashboard/summary
    Backend-->>API: ApiResponse DashboardSummary
    API-->>DashboardStore: DashboardSummary
    DashboardStore-->>DashboardView: reactive state update
    DashboardView->>DashboardView: render cards with data
```

### Phase A (Frontend Only)

In Phase A, the `DashboardStore` returns mocked data instead of calling the API.
The mock data module provides a complete `DashboardSummary` object matching the
spec contract.

### Phase B (Full Stack)

In Phase B, the store calls the backend API. The backend aggregates data from
domain tables (or seed data in V1) and returns the dashboard summary.

## 6. State Boundaries

```mermaid
graph TD
    subgraph Global["Global State -- Pinia"]
        WS["workspaceStore<br/>WorkspaceContext"]
        DASH_S["dashboardStore<br/>DashboardSummary"]
    end

    subgraph Page["Page-Local State"]
        FILTER["selected filters<br/>future"]
        EXPAND["expanded cards<br/>future"]
    end

    subgraph Component["Component-Local State"]
        HOVER["hover state"]
        TOOLTIP["tooltip visibility"]
        ANIM["animation state"]
    end

    WS --> DASH_S
    DASH_S --> FILTER
    DASH_S --> EXPAND
    FILTER --> HOVER
    EXPAND --> TOOLTIP
```

| State | Scope | Owner |
|-------|-------|-------|
| WorkspaceContext | Global (Pinia) | `workspaceStore` |
| DashboardSummary | Global (Pinia) | `dashboardStore` |
| Loading/error per section | Store | `dashboardStore` |
| Card hover, tooltip, animation | Component-local | Each card component |
| Future: filter selections | Page-local | `DashboardView` |

## 7. Backend Architecture (Phase B)

### 7.1 Package Structure

```
backend/src/main/java/com/sdlctower/
├── domain/
│   └── dashboard/
│       ├── DashboardController.java
│       ├── DashboardService.java
│       └── dto/
│           ├── DashboardSummaryDto.java
│           ├── SdlcStageHealthDto.java
│           ├── DeliveryMetricsDto.java
│           ├── AiParticipationDto.java
│           ├── QualityMetricsDto.java
│           ├── StabilityMetricsDto.java
│           ├── GovernanceMetricsDto.java
│           ├── RecentActivityDto.java
│           └── ValueStoryDto.java
```

### 7.2 API Design

```
GET /api/v1/dashboard/summary
  Query: workspaceId (derived from context)
  Response: ApiResponse<DashboardSummaryDto>
```

### 7.3 Data Aggregation Strategy

V1 uses seed data. The `DashboardService` returns pre-built summary objects.

Future versions will aggregate from domain tables:
- SDLC health from requirement, spec, code, test, deploy, incident counts
- Delivery metrics from project timeline data
- AI participation from skill execution history
- Quality from build and test result tables
- Stability from incident and deployment tables
- Governance from template, audit, and policy tables

## 8. Integration Points

```mermaid
graph LR
    DASH["Dashboard"]
    IM["Incident Management<br/>/incidents"]
    PC["Platform Center<br/>/platform"]
    PS["Project Space<br/>/project-space"]
    CB["Code and Build<br/>/code"]
    REQ["Requirements<br/>/requirements"]

    DASH -->|incident badge click| IM
    DASH -->|governance click| PC
    DASH -->|SDLC node click| REQ
    DASH -->|SDLC node click| CB
    DASH -->|project drill-down future| PS
```

### Navigation from Dashboard

| User Action | Target Route | Context Preserved |
|-------------|-------------|-------------------|
| Click SDLC chain node | Module page route | Yes (workspace) |
| Click incident badge | `/incidents` | Yes |
| Click governance indicator | `/platform` | Yes |
| Click "View All" activity | `/platform` | Yes |

## 9. Error and Resilience

- Dashboard fetches data on mount via a single API call
- Each section in the response has its own `SectionResult` envelope
- A section-level error renders only that card in error state; others still display
- A top-level API failure (network, auth) puts all cards into error state
- Shell (nav, context, AI panel) remains functional during dashboard errors
- Retry is manual (page refresh) in V1

## 10. Security Considerations

- Dashboard API is workspace-scoped; backend must validate workspace access
- No sensitive data exposed in dashboard summary (aggregate metrics only)
- Activity stream shows action descriptions but not full payloads
- Authentication is deferred to a later slice but API structure must not
  make it structurally difficult to add

## 11. Performance Considerations

- Single API call fetches all dashboard data (avoids waterfall)
- V1 data is small (seed data); no pagination needed for summary
- Activity stream limited to 10 entries
- Future: consider caching dashboard summary with short TTL
- Future: lazy-load below-fold cards

## 12. Risks and Mitigations

| Risk | Mitigation |
|------|-----------|
| Dashboard becomes chart-heavy and loses story | Design emphasizes narrative flow and SDLC chain |
| Too many cards create visual overload | Limit to 8 core cards; group related metrics |
| Backend aggregation becomes slow with real data | Single summary endpoint; future: materialized views |
| Mock data doesn't represent realistic scenarios | Use realistic enterprise-scale mock values |
