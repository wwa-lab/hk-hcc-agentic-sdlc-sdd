# Project Space Tasks

## Objective

Deliver the Project Space slice end-to-end in two phases: Phase A (frontend with mock data, Codex) and Phase B (backend wiring, Codex), following the SDD pipeline and conforming to the 9 upstream design documents.

## Implementation Strategy

- Frontend-first (Phase A): mock-driven, no backend dependency. Ship an end-to-end browsable Project Space for the seeded Project.
- Backend (Phase B): implement projections, controller, and migrations. Swap frontend from mock mode to live API. Flyway migrations only.
- Per-card isolation: each card is independently developable, mockable, and testable.
- Shared primitives live under `src/shared/` / `com.sdlctower.shared` to avoid Project-Space-owned drift.

## Traceability

- Spec: [project-space-spec.md](../03-spec/project-space-spec.md)
- Design: [project-space-design.md](../05-design/project-space-design.md)
- Architecture: [project-space-architecture.md](../04-architecture/project-space-architecture.md)
- Data Flow: [project-space-data-flow.md](../04-architecture/project-space-data-flow.md)
- Data Model: [project-space-data-model.md](../04-architecture/project-space-data-model.md)
- API Guide: [project-space-API_IMPLEMENTATION_GUIDE.md](../05-design/contracts/project-space-API_IMPLEMENTATION_GUIDE.md)
- Requirements: [project-space-requirements.md](../01-requirements/project-space-requirements.md)
- Stories: [project-space-stories.md](../02-user-stories/project-space-stories.md)

## Planning Assumptions

- `shared-app-shell`, `dashboard`, `requirement`, `incident`, `team-space` slices are merged and operational.
- Project Management, Design Management, Code & Build, Testing, Deployment Management, Platform Center are NOT yet built; stub navigation fallbacks (feature-flagged) are acceptable and expected.
- H2 is the local dev DB; Oracle is the deployed DB. Flyway migrations must be compatible with both or split per-dialect.
- Java 21 + Spring Boot 3.x; Vue 3 + Vite + Pinia + TypeScript.
- No localStorage / sessionStorage (per shared shell rules).
- `SectionResult<T>` already exists in `shared/types/section.ts`.
- `fetchJson<T>` already exists in `shared/api/client.ts`.
- `ApiResponse<T>` and `SectionResultDto<T>` already exist in backend `shared/dto/`.
- `LineageBadge.vue` from Team Space is reusable.
- `risk_signals` table from Team Space exists and will be extended with `project_id`.
- Mock toggle pattern follows dashboard / incident / requirement / team-space: `import.meta.env.DEV && !import.meta.env.VITE_USE_BACKEND`.

---

## Phase A: Frontend (Codex)

### A0: Shared Infrastructure Preparation

- [ ] Reuse `src/shared/types/section.ts` for `SectionResult<T>`
- [ ] Reuse `src/shared/components/LineageBadge.vue` (no changes)
- [ ] Reuse shared shell primitives already in repo: `shellUiStore.setBreadcrumbs(...)` + `shellUiStore.setAiPanelContent(...)`
- [ ] Support route-driven Project switching first; treat shell-level cross-workspace reconciliation as an optional integration if shared shell support is available

### A1: Define Project Space Types

- [ ] `src/features/project-space/types/enums.ts` — all enums per data-model §2.2
- [ ] `src/features/project-space/types/summary.ts` — `ProjectSummary`, `ProjectCounters`, `HealthFactor`
- [ ] `src/features/project-space/types/leadership.ts` — `LeadershipOwnership`, `RoleAssignment`
- [ ] `src/features/project-space/types/chain.ts` — `SdlcChainState`, `ChainNodeHealth`
- [ ] `src/features/project-space/types/milestones.ts` — `MilestoneHub`, `Milestone`
- [ ] `src/features/project-space/types/dependencies.ts` — `DependencyMap`, `Dependency`
- [ ] `src/features/project-space/types/risks.ts` — `RiskRegistry`, `RiskItem`
- [ ] `src/features/project-space/types/environments.ts` — `EnvironmentMatrix`, `Environment`, `VersionDrift`
- [ ] `src/features/project-space/types/aggregate.ts` — `ProjectSpaceAggregate`, `ProjectSpaceState`

### A2: Build Mock Data

- [ ] `src/features/project-space/mock/summary.mock.ts`
- [ ] `src/features/project-space/mock/leadership.mock.ts`
- [ ] `src/features/project-space/mock/chain.mock.ts` — always 11 nodes, Spec emphasized
- [ ] `src/features/project-space/mock/milestones.mock.ts`
- [ ] `src/features/project-space/mock/dependencies.mock.ts`
- [ ] `src/features/project-space/mock/risks.mock.ts`
- [ ] `src/features/project-space/mock/environments.mock.ts` — include a drift example
- [ ] `src/features/project-space/mock/aggregate.mock.ts` — composes the above into a full aggregate

### A3: Build Reusable Primitives

- [ ] `ChainNodeTile.vue` — 11-node tile with emphasis + disabled state
- [ ] `MilestoneRow.vue` — label, date, status chip, percent bar, slippage reason
- [ ] `DependencyRow.vue` — target, relationship chip, owner team, health, blocker
- [ ] `RiskItem.vue` — title, severity chip, category, owner, age, action
- [ ] `EnvironmentTile.vue` — label, version (mono), gate chip, last-deployed, drift indicator
- [ ] `DriftIndicator.vue` — band color, commit delta, tooltip
- [ ] `HealthFactorPopover.vue` — hover details for aggregate health LED
- [ ] `RoleRow.vue` — role label, person, oncall chip, backup chip

### A4: Build Project Summary Bar

- [ ] `ProjectSummaryBar.vue` — identity + workspace back-link + lifecycle chip + health LED + PM/Tech Lead avatars + active milestone + counters + last-updated
- [ ] Hover on health LED triggers `<HealthFactorPopover>`
- [ ] Workspace back-link navigates to `/team?workspaceId=:ws`
- [ ] Loading / error states (no empty — always has identity)

### A5: Build Leadership & Ownership Card

- [ ] `LeadershipOwnershipCard.vue` — renders 6 `<RoleRow>` entries in fixed order
- [ ] Unassigned roles render "Not assigned" muted chip
- [ ] Missing backups render "no backup" chip
- [ ] "Manage in Access Management" link stays disabled until `/platform?view=access&projectId=:id` is available

### A6: Build SDLC Deep Links Card

- [ ] `SdlcDeepLinksCard.vue` — 11 `<ChainNodeTile>` components in canonical order
- [ ] Spec tile rendered 1.2x with cyan emphasis
- [ ] Disabled tiles (target feature flag off) render muted with "Coming soon" tooltip
- [ ] Click emits `navigate-chain-node` with node key; router deep-links with `projectId` + `workspaceId`

### A7: Build Milestone Execution Hub Card

- [ ] `MilestoneExecutionHubCard.vue`
- [ ] Chronological timeline rendering via `<MilestoneRow>`
- [ ] Current milestone marked with cyan ring
- [ ] At-Risk / Slipped rows use crimson accent + inline reason
- [ ] Empty state "No milestones defined yet" with CTA to Project Management (feature-flagged)
- [ ] "Manage in Project Management" link (feature-flagged)

### A8: Build Operational Dependency Map Card

- [ ] `OperationalDependencyMapCard.vue`
- [ ] Two subsections: Upstream / Downstream
- [ ] `<DependencyRow>` renders per entry with external chip when `external === true`
- [ ] Blockers use crimson accent with reason inline
- [ ] Primary action navigates to adjacent Project Space OR incident detail OR disabled for externals

### A9: Build Risk & Vulnerability Registry Card

- [ ] `RiskRegistryCard.vue` — `<RiskItem>` list (already pre-ordered server-side)
- [ ] Critical risks render in crimson accent
- [ ] Empty state "All green" with illustrative glyph
- [ ] Primary action opens incident / approval / task depending on `action.url`

### A10: Build Environment & Version Matrix Card

- [ ] `EnvironmentMatrixCard.vue` — one `<EnvironmentTile>` per environment
- [ ] Gate status chip: `AUTO` / `APPROVAL_REQUIRED` / `BLOCKED`
- [ ] `<DriftIndicator>` rendered when `drift != null`
- [ ] Deployment-Management link gated by feature flag

### A11: Build Pinia Store and API Client

- [ ] `frontend/src/features/project-space/api/projectSpaceApi.ts` using `fetchJson<T>()`
- [ ] `frontend/src/features/project-space/stores/projectSpaceStore.ts` with `initProject`, `switchProject`, `retryCard`, `refreshCard`, `reset`
- [ ] Mock toggle follows repo convention: `import.meta.env.DEV && !import.meta.env.VITE_USE_BACKEND`
- [ ] Per-card retry preserves other cards' state
- [ ] Store exposes `workspaceId` derived from aggregate for context-bar reconciliation

### A12: Build ProjectSpaceView

- [ ] `ProjectSpaceView.vue` as top-level route view
- [ ] Mounts summary bar + 6 cards (leadership / chain / milestones / dependencies / risks / environments)
- [ ] Registers breadcrumb on mount: `Dashboard / Team Space (ws) / Project Space (prj)`
- [ ] Projects Project-scoped AI summary content via `shellUiStore.setAiPanelContent(...)`
- [ ] Watches `route.params.projectId` for Project switch
- [ ] Preserves optional `workspaceId` query on cross-workspace drill-downs; do not block V1 on a dedicated shell Project switcher

### A13: Routing and Navigation Wiring

- [ ] Add `/project-space/:projectId?` route pointing to `ProjectSpaceView`
- [ ] Carry optional upstream Workspace context via query string when drilling from Team Space
- [ ] Remove Project Space `comingSoon` behavior from shared shell nav config
- [ ] Wire Team Space Project Distribution cards to deep-link into `/project-space/:projectId`
- [ ] Feature-flag stub navigation for Project Management / Design / Code / Testing / Deployment / Platform Center
- [ ] Ensure keyboard navigation and ARIA labels

### A14: States and Error Isolation

- [ ] Per-card skeletons during load
- [ ] Per-card error with retry
- [ ] Per-card empty states (distinct copy per card)
- [ ] Page-level error for access denial / 404 / auth
- [ ] Invalid `projectId` format → invalid-project page

### A15: Validate Project Space Frontend

- [ ] `npm run dev` — browses Project Space at `/project-space/proj-8821?workspaceId=ws-default-001`
- [ ] `npm run build` passes
- [ ] Visual spot-check against design §2 layout and the mockup at [../05-design/Project Space.html](../05-design/Project%20Space.html)
- [ ] Project switch re-loads cards without full reload
- [ ] Cross-workspace switch reconciles shell context bar
- [ ] Breadcrumb trail records on drill-down
- [ ] Vitest unit tests pass

### Phase A Definition of Done

- Summary bar + 6 cards render with mock data
- SDLC chain strip renders 11 nodes with Spec highlighted; disabled tiles render muted
- Project switch works (including cross-workspace reconciliation)
- Drill-down navigation works (to Requirement Management + Incident Management; stubs for others)
- Per-card loading / error / empty states behave correctly
- Build passes; no console errors

---

## Phase B: Backend (Codex)

### B0: Shared Infrastructure (Prerequisite)

- [ ] Verify `com.sdlctower.shared.dto.ApiResponse` exists; reuse for Project Space endpoints
- [ ] Verify `com.sdlctower.shared.dto.SectionResultDto` exists; reuse for aggregate section envelopes
- [ ] Add `PROJECT_SPACE` constant to `com.sdlctower.shared.ApiConstants`

### B1: Create Project Space Package Structure

- [ ] Create `com.sdlctower.domain.projectspace` package with controller, service, `dto/`, `projection/`, and `persistence/`

### B2: Create Project Space DTOs

- [ ] All DTOs per data-model §4 (records)
- [ ] All enums per data-model §2.2 (mirrored as Java enums)
- [ ] `LinkDto`, `MemberRefDto`, `MilestoneRefDto`, `HealthFactorDto`, `SkillAttributionDto`
- [ ] `SectionResultDto<T>` wrapper per aggregate card

### B3: Implement Projections

- [ ] `ProjectSummaryProjection` reading `ProjectRepositoryFacade` + aggregate health computation
- [ ] `LeadershipProjection` reading `MemberRepositoryFacade` + role/backup/oncall resolution
- [ ] `ChainNodeProjection` reading `RequirementReadFacade` + `IncidentReadFacade` (+ future facades as available, defaulting to `UNKNOWN` / `enabled=false` when absent)
- [ ] `MilestoneProjection` reading `MilestoneRepository`
- [ ] `DependencyProjection` reading `ProjectDependencyRepository` + optional health enrichment
- [ ] `RiskRegistryProjection` reading `risk_signals` filtered by `project_id`; server-side ordered by severity DESC, ageDays DESC
- [ ] `EnvironmentMatrixProjection` reading `EnvironmentRepository` + `DeploymentRepository` + version drift computation

### B4: Implement Project Space Service

- [ ] `ProjectSpaceService.loadAggregate` with parallel fan-out + per-projection 500ms timeout
- [ ] `ProjectSpaceService.load{Summary,Leadership,Chain,Milestones,Dependencies,Risks,Environments}` for per-card endpoints
- [ ] Per-projection exception yields `SectionResultDto(data=null, error=...)`, not a top-level failure

### B5: Implement Access Guard

- [ ] `ProjectAccessGuard.check(projectId)` resolving the authenticated principal + verifying read access (typically via parent workspace membership)
- [ ] Throws `ProjectAccessDeniedException` on failure; add a matching `@ExceptionHandler` in `GlobalExceptionHandler` so 403 responses keep the shared `ApiResponse` envelope

### B6: Implement Project Space Controller

- [ ] `ProjectSpaceController` with aggregate + 7 per-card endpoints
- [ ] `@Pattern` path validation on `projectId` (`^proj-[a-z0-9\-]+$`)
- [ ] Standard response envelope wrap

### B7: Create Flyway Migrations

- [ ] `V9__create_project_space_tables.sql` — `milestones`, `project_dependencies`, `environments`, `deployments`
- [ ] `V10__add_project_id_to_risk_signals.sql` — add nullable `project_id` column + project-scoped index
- [ ] `V11__seed_project_space_data.sql` — seed rows for local dev
- [ ] Verify migrations run on H2 (local) without error
- [ ] Oracle variant migration if DDL needs adjustment (separate V-file or conditional DDL)

### B8: Implement Entities and Repositories

- [ ] `MilestoneEntity`, `ProjectDependencyEntity`, `EnvironmentEntity`, `DeploymentEntity` (JPA)
- [ ] `MilestoneRepository`, `ProjectDependencyRepository`, `EnvironmentRepository`, `DeploymentRepository` (Spring Data)
- [ ] Query methods for Project-scoped reads (ordered, filtered)

### B9: Risk Signal Extension Support

- [ ] Extend existing `RiskSignalEntity` with nullable `projectId` field
- [ ] Add repository method `findActiveByProjectId(projectId)` returning list ordered by severity DESC, detectedAt DESC
- [ ] Verify Team Space `RiskRadarJob` still functions after column addition

### B10: Backend Tests

- [ ] `ProjectSpaceControllerTest` (MockMvc): happy path, 400 invalid id, 403 denied, 404 not found, per-projection error isolation, chain always 11 nodes
- [ ] `ProjectSpaceServiceTest`: parallel fan-out behavior, timeout degradation
- [ ] Per-projection unit tests with stub facades
- [ ] `EnvironmentMatrixProjectionTest`: version drift band thresholds (0 / 10 / 11)
- [ ] `RiskRegistryProjectionTest`: ordering correctness
- [ ] `ProjectAccessGuardTest`
- [ ] Repository tests against H2

### B11: Connect Frontend to Backend

- [ ] Set `VITE_USE_BACKEND=true` in dev config
- [ ] Verify Vite proxy routes `/api` to backend
- [ ] Smoke test: browse `/project-space/proj-8821` against live backend
- [ ] Verify error cases (invalid project id, denied access, 404) render correctly
- [ ] Verify Team Space project distribution deep-link lands correctly

### B12: Validate Full Stack

- [ ] `./mvnw verify` passes
- [ ] `npm run build` passes
- [ ] End-to-end browse with mock=false shows summary bar + all 6 cards hydrated from real data
- [ ] Per-card retry works end-to-end
- [ ] Project switch re-fetches from backend; cross-workspace switch reconciles context
- [ ] 11-node SDLC chain always rendered; disabled tiles behave

### Phase B Definition of Done

- All endpoints return correct envelopes for happy path
- Per-projection timeout degrades to section-level error only
- Flyway V9 + V10 + V11 migrations run cleanly on H2
- Frontend works in live mode (`VITE_USE_BACKEND=true`)
- Backend + frontend builds pass
- Project Space accessible from Team Space Project Distribution drill-in and direct URL

---

## Dependency Plan

### Critical Path

```
A0 (shared infra) → A1 (types) → A2 (mocks) → A3 (primitives)
   → A4 (summary bar) → A5…A10 (6 cards in parallel) → A11 (store/API) → A12 (view) → A13 (routing) → A14 (states) → A15 (validate)
        │
        ↓
B0 (shared backend) → B1 (package) → B2 (DTOs) → B3 (projections in parallel)
   → B4 (service) → B5 (guard) → B6 (controller) → B7 (migrations) → B8 (entities/repo) → B9 (risk extension)
   → B10 (tests) → B11 (connect) → B12 (validate)
```

### Parallel Workstreams

- A5 through A10 (six card components) can be built in parallel by multiple contributors
- B3 projections can be built in parallel once DTOs are defined
- Frontend Phase A can proceed independently of Phase B backend work up through A15

---

## Risks / Blockers

| Risk | Mitigation |
|------|-----------|
| Upstream facades (Design / Code / Test / Deploy) not yet exposed | Chain node projection defaults to `UNKNOWN` health + `enabled=false`; feature flags hide disabled tiles gracefully |
| Milestone ownership ambiguity before Project Management lands | Project Space is interim read-only owner via seed data; flag for transfer when Project Management lands |
| Dependency health probe requires upstream health service | V1 uses static registry values; real-time probe deferred to V2 |
| Environment / deployment data ownership overlaps with future Deployment Management | Use Project Space tables as interim source of truth with read-only HTTP surface; migration path documented |
| Cross-workspace project switch surprises users | Shell auto-reconciles Workspace / Application; brief toast explains the switch |
| Oracle DDL incompatibility for new tables | Portable DDL; split per-dialect migration if needed; verify on Oracle-in-Docker during B7 |
| Adding `project_id` to `risk_signals` affects Team Space query plans | Additive + nullable; verify Team Space `RiskRadarJob` unaffected in B9 |

---

## Open Questions

Inherited from spec:

- Milestone completion % source (derived vs manual) — confirm before B3
- Dependency health probe strategy (static vs real-time) — confirm before B3
- Version drift threshold default — proposed 10 commits; confirm before B3
- AI Adoption owner as a first-class role vs derived — confirm before A5
- Team-Space vs Project-Space AI skill pack separation — confirm before A12
- Permission-gated link strategy (hide vs disable) — reuse Team Space precedent unless overridden
