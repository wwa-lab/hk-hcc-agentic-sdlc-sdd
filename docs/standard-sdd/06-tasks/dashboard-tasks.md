# Dashboard / Control Tower Tasks

## Objective

Implement the Dashboard / Control Tower page — the main landing page of the
Agentic SDLC Control Tower product.

## Implementation Strategy

Frontend and backend are developed in separate phases with different tools:

| Phase | Scope | Tool | Goal |
|-------|-------|------|------|
| Phase A | Frontend (Vue 3) | Gemini | Build dashboard UI with mocked data |
| Phase B | Backend (Spring Boot) | Codex | Build API and connect frontend to live data |

Phase A is self-contained and produces a fully functional dashboard with mocked data.
Phase B adds the backend and replaces mocked data with real API calls.

## Traceability

- Requirements: [dashboard-requirements.md](../01-requirements/dashboard-requirements.md)
- Stories: [dashboard-stories.md](../02-user-stories/dashboard-stories.md)
- Spec: [dashboard-spec.md](../03-spec/dashboard-spec.md)
- Architecture: [dashboard-architecture.md](../04-architecture/dashboard-architecture.md)
- Design: [dashboard-design.md](../05-design/dashboard-design.md)
- Visual prototype: [Control Tower.html](../05-design/Control%20Tower.html)

---

## Phase A: Frontend (Gemini)

### A0: Define Dashboard Types and Mock Data

- Create `frontend/src/features/dashboard/types/dashboard.ts` with all TypeScript interfaces including `SectionResult<T>` envelope for per-card error isolation
- Create `frontend/src/features/dashboard/mockData.ts` with realistic mock data matching all interfaces
- Mock data must cover: SDLC health (11 stages), delivery metrics (4 metrics with trends), AI participation (4 metrics + stage involvement), quality metrics, stability metrics, governance metrics, recent activity (10 entries), and value story
- Ensure Spec node has `isHub: true` and at least one stage has `status: 'critical'` to verify visual treatment

Depends on: none (existing shell infrastructure)
Done when: types compile; mock data is importable and type-safe

### A1: Build Reusable Sub-Components

- Create `MetricCard.vue` — displays a single metric with label, value, trend arrow, and trend color
- Create `SdlcStageNode.vue` — single SDLC chain node with status LED, label, item count, clickable
- Create `ActivityEntry.vue` — single activity row with actor icon (AI/human), action, stage badge, timestamp
- All components accept props matching the TypeScript interfaces
- All components use the existing design tokens (colors, typography, spacing)

Depends on: A0
Done when: all three components render correctly with sample props in isolation

### A2: Build SDLC Chain Health Component

- Create `SdlcChainHealth.vue` — horizontal pipeline of 11 `SdlcStageNode` instances connected by arrow connectors
- Spec node receives distinct visual treatment: highlighted border, "HUB" micro-badge
- Nodes are clickable and emit a `@navigate` event with the stage's route path
- Chain supports all status colors: healthy (green), warning (amber), critical (red), inactive (muted)
- Chain scrolls horizontally if viewport is too narrow

Depends on: A1
Done when: chain renders 11 nodes from mock data; Spec node is visually distinct; clicking a node emits the correct route path

### A3: Build Dashboard Metric Cards

- Create `DeliveryMetricsCard.vue` — grid of `MetricCard` instances for lead time, deploy frequency, iteration completion, plus bottleneck stage callout
- Create `AiParticipationCard.vue` — headline metrics plus stage involvement indicator strip
- Create `QualityTestingCard.vue` — 4 quality metrics
- Create `StabilityIncidentCard.vue` — incident count badge (with severity color), MTTR, change failure rate, rollback rate; incident badge is clickable
- Create `GovernanceTrustCard.vue` — 4 governance metrics with warning flags for non-compliant values; clickable to Platform Center
- Create `ValueStoryCard.vue` — headline text and 3-4 proof metrics with descriptions

Depends on: A1
Done when: all 6 cards render from mock data with correct styling and interactions

### A4: Build Recent Activity Stream

- Create `RecentActivityStream.vue` — chronological list of `ActivityEntry` instances
- Show 10 most recent entries
- AI entries distinguished with AI icon and cyan accent
- Human entries use person icon
- "View All" link at the bottom
- Empty state: "No recent activity" message

Depends on: A1
Done when: activity stream renders 10 mock entries; AI and human entries are visually distinct

### A5: Build Dashboard Store

- Create `frontend/src/features/dashboard/stores/dashboardStore.ts` (Pinia store)
- Store loads the full `DashboardSummary` — in Phase A, from mock data
- Each section uses `SectionResult<T>` so cards handle errors independently
- Exposes reactive `summary`, `isLoading` (top-level), `error` (top-level) state
- `fetchSummary()` function that returns mock data (Phase A via `MOCK_DASHBOARD_DATA`) or calls API (Phase B via `dashboardApi.getSummary()`)
- Create `frontend/src/features/dashboard/api/dashboardApi.ts` — API function using shared `fetchJson<T>`

Depends on: A0
Done when: store is importable; `fetchSummary()` returns mock data; reactive state works

### A6: Assemble Dashboard View

- Replace the existing placeholder `DashboardView.vue` with the full dashboard layout
- Use CSS grid: 2 columns, full-width rows for SDLC chain and activity stream
- Wire all cards to the dashboard store
- Implement card-level loading and error states
- Wire navigation: SDLC chain click → router.push(stage.routePath), incident click → `/incidents`, governance click → `/platform`, activity "View All" → `/platform`
- Note: Routes for unimplemented modules (e.g., `/requirements`, `/design`) navigate to placeholder views with "Coming Soon" state. This is handled by the shared shell's route configuration in `application.yml` (`coming-soon: true`).
- Ensure page renders inside the shared shell with context bar and AI panel

Depends on: A2, A3, A4, A5
Done when: dashboard renders all 8 sections from mock data; navigation works; page loads within the shell

### A7: Validate Dashboard Frontend

- Verify all 11 SDLC stages render with correct status colors
- Verify Spec node is visually distinct with "HUB" badge
- Verify all metric cards show values, labels, and trend indicators
- Verify AI participation card shows stage involvement strip
- Verify activity stream shows 10 entries with AI/human distinction
- Verify value story card renders headline and proof metrics
- Verify navigation: clicking SDLC nodes routes correctly
- Verify navigation: clicking incident badge routes to `/incidents`
- Verify navigation: clicking governance routes to `/platform`
- Verify loading, error, and empty states render correctly
- Verify stories S1–S8 acceptance criteria against the running app
- Verify `npm run dev` and `npm run build` both succeed

Depends on: A6
Done when: all checks pass; the dashboard is visually complete and functional with mocked data

### Phase A Definition of Done

- Dashboard renders all 8 content sections within the shared shell
- SDLC chain displays 11 nodes with Spec highlighted as hub
- All metric cards display with trend indicators
- AI participation is visually prominent, not a sidebar metric
- Recent activity stream distinguishes AI from human actions
- Navigation from dashboard to downstream pages works
- All story S1–S8 acceptance criteria verifiable in the browser
- No backend dependency; all data is mocked
- `npm run dev` and `npm run build` both succeed

---

## Phase B: Backend (Codex)

### B0: Create Dashboard DTOs

- Create Java records in `domain/dashboard/dto/`:
  - `DashboardSummaryDto`
  - `SdlcStageHealthDto`
  - `MetricValueDto`
  - `AiParticipationDto`
  - `QualityMetricsDto`
  - `StabilityMetricsDto`
  - `GovernanceMetricsDto`
  - `ActivityEntryDto`
  - `RecentActivityDto`
  - `ValueStoryDto`
- All DTOs are Java records (immutable)
- Field names match frontend TypeScript interfaces exactly (camelCase JSON serialization)

Depends on: none (can start in parallel with Phase A)
Done when: all DTOs compile; Jackson serializes to JSON matching the frontend contract

### B1: Implement Dashboard Service

- Create `DashboardService.java` in `domain/dashboard/`
- Implement `getDashboardSummary()` returning a `DashboardSummaryDto` with seed data
- Seed data matches the mock data from frontend Phase A
- Service is workspace-aware (accepts workspace ID parameter for future isolation)

Depends on: B0
Done when: service returns valid `DashboardSummaryDto` with realistic seed data

### B2: Implement Dashboard Controller

- Create `DashboardController.java` with `GET /api/v1/dashboard/summary`
- Return `ApiResponse<DashboardSummaryDto>` using the shared response envelope
- Include proper error handling and logging

Depends on: B1
Done when: endpoint returns valid JSON matching the frontend contract; status 200 for success

### B3: Write Backend Tests

- Create `DashboardControllerTest.java` using MockMvc
- Test: GET /api/v1/dashboard/summary returns 200 with valid JSON
- Test: response contains all required sections (sdlcHealth, deliveryMetrics, etc.)
- Test: SDLC health contains exactly 11 stages
- Test: Spec stage has `isHub: true`

Depends on: B2
Done when: all tests pass; controller behavior is verified

### B4: Connect Frontend to Backend

- Update `dashboardApi.ts` to call `GET /api/v1/dashboard/summary`
- Update `dashboardStore.ts` to use the API instead of mock data
- Configure Vite dev server proxy for `/api/v1/dashboard/*`
- Handle loading and error states from real API responses
- Retain mock data as fallback for development without backend

Depends on: A7, B2
Done when: frontend renders live data from Spring Boot; dashboard shows seed data from backend

### B5: Validate Full Stack

- Verify frontend + backend integration end to end
- Verify dashboard loads data from backend API
- Verify all 8 sections render correctly with backend data
- Verify API error responses produce graceful frontend fallbacks
- Verify H2 profile works for local development

Depends on: B4
Done when: full stack runs locally; dashboard displays backend seed data

### Phase B Definition of Done

- Spring Boot serves dashboard summary via REST API
- API response shape matches frontend TypeScript contracts
- Frontend fetches live data from backend
- All 8 dashboard sections render from backend data
- Controller tests pass
- `./mvnw spring-boot:run` and `npm run dev` work together locally
