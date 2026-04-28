# Shared App Shell Tasks

## Objective

Implement the first SDD foundation slice for the product:

- shared application shell

## Implementation Strategy

Frontend and backend are developed in separate phases with different tools:

| Phase | Scope | Tool | Goal |
|-------|-------|------|------|
| Phase A | Frontend (Vue 3) | Gemini | Build shell UI, verify visual output with static/mocked data |
| Phase B | Backend (Spring Boot) | Codex | Build API layer, persistence, and connect frontend to live data |

Phase A is self-contained and produces a fully navigable shell with mocked data.
Phase B adds the backend and replaces mocked data with real API calls.

## Traceability

- Stories: [shared-app-shell-stories.md](/Users/leo/wwa-lab/GitHub/Agentic-SDLC-Control-Tower/docs/standard-sdd/02-user-stories/shared-app-shell-stories.md:1)
- Spec: [shared-app-shell-spec.md](/Users/leo/wwa-lab/GitHub/Agentic-SDLC-Control-Tower/docs/standard-sdd/03-spec/shared-app-shell-spec.md:1)
- Architecture: [shared-app-shell-architecture.md](/Users/leo/wwa-lab/GitHub/Agentic-SDLC-Control-Tower/docs/standard-sdd/04-architecture/shared-app-shell-architecture.md:1)
- Design reference: [design.md](/Users/leo/wwa-lab/GitHub/Agentic-SDLC-Control-Tower/docs/standard-sdd/05-design/design.md:82)

---

## Phase A: Frontend (Gemini)

### A0: Scaffold Frontend Project

- Initialize Vue 3 project with Vite and TypeScript
- Configure Vue Router and Pinia
- Set up project structure: `src/layouts/`, `src/views/`, `src/components/`, `src/composables/`, `src/types/`
- Add basic CSS reset and dark theme baseline (control-tower style)

Depends on: none
Done when: `npm run dev` serves a blank Vue app at localhost

### A1: Confirm Foundation Contract

- Review nav vocabulary against PRD §10 and confirm final 13 labels
- Confirm `WorkspaceContext` type shape
- Confirm shell ownership boundary versus page ownership
- Confirm AI panel is a shell region, not page-local layout

Depends on: A0
Done when: all four items reviewed; any changes reflected back into spec and stories

### A2: Build Shared Layout Components

- Create `AppShell.vue` with slot-based layout regions (nav, top bar, content, AI panel)
- Create `PrimaryNav.vue` with 13 navigation entries and left-edge active indicator
- Create `TopContextBar.vue` rendering `WorkspaceContext` with empty-safe fallbacks
- Create `GlobalActionBar.vue` with search, notification, and audit entry points
- Create `AiCommandPanel.vue` with summary, reasoning, evidence, and action zones (320px right rail)

Depends on: A1
Done when: all five components render in isolation with static props; visual output matches design HTML prototypes

### A3: Wire Example Pages Through The Shell

- Create route configuration with metadata (navKey, title) for all 13 entries
- Mount Dashboard placeholder view through the shell
- Mount Project Space placeholder view through the shell
- Mount Incident Management placeholder view through the shell
- Mount Platform Center placeholder view through the shell
- Remaining 9 routes point to a shared "Coming Soon" placeholder

Depends on: A2
Done when: all 13 routes are navigable; each route highlights the correct nav item; four placeholder pages render distinct content inside the shared shell

### A4: Add Shell State Contracts

- Define `WorkspaceContext` TypeScript type in `src/types/`
- Define `ShellPageConfig` TypeScript type in `src/types/`
- Create `useWorkspaceContext()` composable with static mocked data
- Implement route-to-nav-key mapping via Vue Router route metadata
- Add safe fallback rendering for missing optional context fields (`snowGroup`, `project`, `environment`)

Depends on: A3
Done when: context bar renders from composable state; nav active state driven by route metadata; missing optional fields render gracefully

### A5: Validate Frontend Baseline

- Verify active nav state is unique per route
- Verify top context field order is stable across all pages
- Verify global utilities (search, notifications, audit) render consistently
- Verify page content errors do not collapse the surrounding shell
- Verify AI panel remains layout-stable when page content changes
- Verify story S1–S5 acceptance criteria against the running app

Depends on: A4
Done when: all checks pass; the frontend shell is visually complete and navigable with mocked data

### Phase A Definition Of Done

- Shared shell renders all 13 routes through one layout abstraction
- Four Round 1 pages show placeholder content inside the shell
- `WorkspaceContext` displays with empty-safe fallbacks
- AI Command Panel is a persistent right-side region
- All story S1–S5 acceptance criteria verifiable in the browser
- No backend dependency; all data is static or mocked
- `npm run dev` and `npm run build` both succeed

---

## Phase B: Backend (Codex)

### B0: Scaffold Backend Project

- Initialize Spring Boot project (Java 17+, Maven or Gradle)
- Configure Spring profiles: `local` (H2 in-memory) and `prod` (Oracle)
- Set up JPA/Hibernate with profile-based datasource switching
- Create base package structure: `controller`, `service`, `repository`, `model`, `config`
- Add health check endpoint (`/actuator/health`)

Depends on: none (can start in parallel with Phase A or after A5)
Done when: `./mvnw spring-boot:run -Dspring.profiles.active=local` starts with H2; health endpoint returns 200

### B1: Implement Workspace Context API

- Create `WorkspaceContext` JPA entity and repository
- Create `GET /api/v1/workspace-context` REST endpoint
- Return workspace, application, snowGroup, project, environment
- Seed H2 with sample workspace context data via `data.sql` or `CommandLineRunner`
- Validate response shape matches frontend `WorkspaceContext` TypeScript type

Depends on: B0
Done when: API returns valid JSON matching the frontend contract; H2 seed data renders correctly

### B2: Implement Navigation Configuration API

- Create `GET /api/v1/nav/entries` REST endpoint returning the 13 navigation items
- Include route key, label, icon identifier, and order
- Navigation data is static in V1 but served from backend to allow future permission filtering

Depends on: B0
Done when: API returns the 13 entries in correct order

### B3: Connect Frontend to Backend

- Replace mocked `useWorkspaceContext()` data with API call to `GET /api/v1/workspace-context`
- Add API client utility in frontend (`src/api/`)
- Configure Vite dev server proxy to Spring Boot backend
- Handle loading and error states in context bar

Depends on: A5, B1, B2
Done when: frontend renders live data from Spring Boot; context bar shows H2 seed data; loading/error states work

### B4: Validate Full Stack

- Verify frontend + backend integration end to end
- Verify H2 profile works for local development
- Verify Oracle profile configuration is correct (connection test against Oracle instance if available)
- Verify API error responses produce graceful frontend fallbacks

Depends on: B3
Done when: full stack runs locally with H2; Oracle config is validated or documented for later connection

### Phase B Definition Of Done

- Spring Boot backend starts with H2 (local) profile
- REST APIs serve workspace context and navigation entries
- Frontend fetches live data from backend instead of mocks
- API response shapes match frontend TypeScript contracts
- `./mvnw spring-boot:run` and `npm run dev` work together locally
- Oracle profile is configured and ready for production connection
