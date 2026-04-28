# Codex Prompt: Dashboard Phase B — Backend Implementation

## Task

Add the **Dashboard API** to the existing Spring Boot backend for Agentic SDLC Control Tower. The frontend already renders a dashboard with mocked data. Your job is to create a backend endpoint that serves the same data, so the frontend can switch from mocks to a live API call.

## Existing Backend (DO NOT break)

The backend is already running under `backend/` with:
- `platform/workspace/` — WorkspaceContext entity, repository, service, controller
- `platform/navigation/` — NavItem DTO, NavigationService, NavigationController
- `shared/dto/ApiResponse.java` — response envelope `{ data, error }`
- `shared/exception/` — GlobalExceptionHandler, ResourceNotFoundException
- `config/CorsConfig.java` — CORS for Vite dev server
- Flyway migrations in `src/main/resources/db/migration/`
- Tests in `src/test/`

## Tech Stack

| Layer | Technology |
|-------|-----------|
| Framework | Spring Boot 3.x (Java 21) |
| Build | Maven |
| ORM | JPA / Hibernate |
| DB (local) | H2 in-memory (profile: `local`) |
| DB (production) | Oracle (profile: `prod`) |

## What To Build

### 1. DTOs in `domain/dashboard/dto/`

Create Java records matching the frontend TypeScript types exactly (camelCase JSON):

```java
// SectionResultDto.java — per-section envelope for independent failure
public record SectionResultDto<T>(T data, String error) {
    public static <T> SectionResultDto<T> ok(T data) {
        return new SectionResultDto<>(data, null);
    }
    public static <T> SectionResultDto<T> fail(String error) {
        return new SectionResultDto<>(null, error);
    }
}
```

Other DTOs (all as Java records):
- `DashboardSummaryDto` — top-level, each field is `SectionResultDto<XxxDto>`
- `SdlcStageHealthDto` — `key`, `label`, `status`, `itemCount`, `isHub`, `routePath`
- `MetricValueDto` — `label`, `value`, `trend`, `trendIsPositive`
- `DeliveryMetricsDto` — `leadTime`, `deployFrequency`, `iterationCompletion`, `bottleneckStage`
- `AiParticipationDto` — `usageRate`, `adoptionRate`, `autoExecSuccess`, `timeSaved`, `stageInvolvement`
- `QualityMetricsDto` — `buildSuccessRate`, `testPassRate`, `defectDensity`, `specCoverage`
- `StabilityMetricsDto` — `activeIncidents`, `criticalIncidents`, `changeFailureRate`, `mttr`, `rollbackRate`
- `GovernanceMetricsDto` — `templateReuse`, `configDrift`, `auditCoverage`, `policyHitRate`
- `ActivityEntryDto` — `id`, `actor`, `actorType`, `action`, `stageKey`, `timestamp`
- `RecentActivityDto` — `entries`, `total`
- `ValueStoryDto` — `headline`, `metrics`

See `docs/05-design/dashboard-design.md` §4 and §5 for complete type definitions.

### 2. DashboardService in `domain/dashboard/`

- `getDashboardSummary()` returns `DashboardSummaryDto` with hardcoded seed data
- Seed data must be realistic and match the mock data values in the frontend
- Each section wrapped in `SectionResultDto.ok(...)` (no failures in V1 seed data)
- Route paths in SDLC stage health must use actual frontend routes:

| Stage | routePath |
|-------|-----------|
| requirement | `/requirements` |
| user-story | `/requirements` |
| spec | `/requirements` |
| architecture | `/design` |
| design | `/design` |
| tasks | `/project-management` |
| code | `/code` |
| test | `/testing` |
| deploy | `/deployment` |
| incident | `/incidents` |
| learning | `/ai-center` |

- Spec stage must have `isHub: true`
- Include at least one `critical` status and one `warning` status

### 3. DashboardController in `domain/dashboard/`

```
GET /api/v1/dashboard/summary
```

- Returns `ApiResponse<DashboardSummaryDto>` using the existing shared envelope
- Delegates to `DashboardService`
- Path constant in `shared/ApiConstants.java` (add if not existing)

### 4. Tests in `domain/dashboard/`

Create `DashboardControllerTest.java` using MockMvc:

- Test: GET returns 200 with valid JSON
- Test: response contains all 8 sections (sdlcHealth, deliveryMetrics, etc.)
- Test: sdlcHealth contains exactly 11 stages
- Test: Spec stage has `isHub: true`
- Test: each section has `data` and `error` fields (SectionResult envelope)

### 5. Frontend Wiring

Update the following frontend files to connect to the backend:

- `frontend/src/features/dashboard/api/dashboardApi.ts` — call `GET /api/v1/dashboard/summary` using shared `fetchJson<T>`
- `frontend/src/features/dashboard/stores/dashboardStore.ts` — use API instead of mock data
- `frontend/vite.config.ts` — add proxy rule for `/api/v1/dashboard` if not already covered by existing proxy

## API Response Shape

```json
{
  "data": {
    "sdlcHealth": { "data": [...], "error": null },
    "deliveryMetrics": { "data": {...}, "error": null },
    "aiParticipation": { "data": {...}, "error": null },
    "qualityMetrics": { "data": {...}, "error": null },
    "stabilityMetrics": { "data": {...}, "error": null },
    "governanceMetrics": { "data": {...}, "error": null },
    "recentActivity": { "data": {...}, "error": null },
    "valueStory": { "data": {...}, "error": null }
  },
  "error": null
}
```

Top-level `error` = total API failure. Section-level `error` = individual section failure. V1 seed data always returns `error: null` for all sections.

## Package Structure

```
backend/src/main/java/com/sdlctower/
├── domain/
│   └── dashboard/
│       ├── DashboardController.java
│       ├── DashboardService.java
│       └── dto/
│           ├── SectionResultDto.java
│           ├── DashboardSummaryDto.java
│           ├── SdlcStageHealthDto.java
│           ├── MetricValueDto.java
│           ├── DeliveryMetricsDto.java
│           ├── AiParticipationDto.java
│           ├── QualityMetricsDto.java
│           ├── StabilityMetricsDto.java
│           ├── GovernanceMetricsDto.java
│           ├── ActivityEntryDto.java
│           ├── RecentActivityDto.java
│           └── ValueStoryDto.java
```

## What NOT To Do

- Do NOT modify existing `platform/` or `shared/` code (except adding a constant to `ApiConstants.java`)
- Do NOT create JPA entities or database tables for dashboard (V1 is seed-data-only)
- Do NOT add Flyway migrations for dashboard data
- Do NOT add authentication or security
- Do NOT modify shell components (`frontend/src/shell/`)
- Do NOT modify shared frontend types or stores (`frontend/src/shared/`)
- Do NOT modify the router (`frontend/src/router/index.ts`)
- Do NOT use Lombok
- Do NOT break existing tests

## Acceptance Criteria

- [ ] `./mvnw spring-boot:run -Dspring.profiles.active=local` starts without errors
- [ ] `GET http://localhost:8080/api/v1/dashboard/summary` returns 200 with correct JSON shape
- [ ] Response contains all 8 sections, each wrapped in `SectionResult` (`{ data, error }`)
- [ ] SDLC health contains exactly 11 stages; Spec has `isHub: true`
- [ ] Route paths in SDLC stages match actual frontend routes
- [ ] JSON field names are camelCase (matching frontend TypeScript types)
- [ ] `./mvnw test` passes all tests (existing + new dashboard tests)
- [ ] Frontend renders backend data when both servers are running
- [ ] `npm run dev` and `npm run build` still succeed
