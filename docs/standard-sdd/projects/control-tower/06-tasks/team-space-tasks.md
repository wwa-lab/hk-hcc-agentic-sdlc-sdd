# Team Space Tasks

## Objective

Deliver the Team Space slice end-to-end in two phases: Phase A (frontend with mock data, Gemini) and Phase B (backend wiring, Codex), following the SDD pipeline and conforming to the 9 upstream design documents.

## Implementation Strategy

- Frontend-first (Phase A): mock-driven, no backend dependency. Ship an end-to-end browsable Team Space for the seeded Workspace.
- Backend (Phase B): implement projections, controller, and migrations. Swap frontend from mock mode to live API. Flyway migrations only.
- Per-card isolation: each card is independently developable, mockable, and testable.
- Shared primitives live under `src/shared/` / `com.sdlctower.shared` to avoid Team-Space-owned drift.

## Traceability

- Spec: [team-space-spec.md](../03-spec/team-space-spec.md)
- Design: [team-space-design.md](../05-design/team-space-design.md)
- Architecture: [team-space-architecture.md](../04-architecture/team-space-architecture.md)
- Data Flow: [team-space-data-flow.md](../04-architecture/team-space-data-flow.md)
- Data Model: [team-space-data-model.md](../04-architecture/team-space-data-model.md)
- API Guide: [team-space-API_IMPLEMENTATION_GUIDE.md](../05-design/contracts/team-space-API_IMPLEMENTATION_GUIDE.md)
- Requirements: [team-space-requirements.md](../01-requirements/team-space-requirements.md)
- Stories: [team-space-stories.md](../02-user-stories/team-space-stories.md)

## Planning Assumptions

- `shared-app-shell`, `dashboard`, `requirement`, `incident` slices are merged and operational.
- Project Space and Platform Center are NOT yet built; stub navigation fallbacks are acceptable and expected.
- H2 is the local dev DB; Oracle is the deployed DB. Flyway migrations must be compatible with both or split per-dialect.
- Java 21 + Spring Boot 3.x; Vue 3 + Vite + Pinia + TypeScript.
- No localStorage / sessionStorage (per shared shell rules).
- `SectionResult<T>` already exists in `shared/types/section.ts`.
- `fetchJson<T>` already exists in `shared/api/client.ts`.
- `ApiResponse<T>` and `SectionResultDto<T>` already exist in backend `shared/dto/`.
- Mock toggle pattern follows dashboard / incident / requirement: `import.meta.env.DEV && !import.meta.env.VITE_USE_BACKEND`.

---

## Phase A: Frontend (Codex)

### A0: Shared Infrastructure Preparation

- [ ] Reuse `src/shared/types/section.ts` for `SectionResult<T>`; extend only if the change is shared across slices
- [ ] Create `src/shared/types/lineage.ts` (`Lineage`, `LineageOrigin`, `LineageHop`)
- [ ] Create `src/shared/components/LineageBadge.vue`
- [ ] Verify shared shell exposes Workspace context bar + AI Command Panel mount API

### A1: Define Team Space Types

- [ ] `src/features/team-space/types/enums.ts` — all enums per data model §2.3
- [ ] `src/features/team-space/types/workspace.ts` — `WorkspaceSummary`, `ResponsibilityBoundary`
- [ ] `src/features/team-space/types/operatingModel.ts` — `TeamOperatingModel`, `TeamOperatingModelField<T>`, `OncallOwner`
- [ ] `src/features/team-space/types/members.ts` — `MemberMatrix`, `MemberMatrixRow`, `CoverageGap`
- [ ] `src/features/team-space/types/templates.ts` — `TeamDefaultTemplates`, `TemplateEntry`, `ExceptionOverride`
- [ ] `src/features/team-space/types/pipeline.ts` — `RequirementPipeline`, `PipelineCounters`, `PipelineBlocker`, `ChainNodeHealth`
- [ ] `src/features/team-space/types/metrics.ts` — `TeamMetrics`, `TeamMetricItem`
- [ ] `src/features/team-space/types/risks.ts` — `TeamRiskRadar`, `RiskItem`
- [ ] `src/features/team-space/types/projects.ts` — `ProjectDistribution`, `ProjectCardDto`
- [ ] `src/features/team-space/types/aggregate.ts` — `TeamSpaceAggregate`, `TeamSpaceState`

### A2: Build Mock Data

- [ ] `src/features/team-space/mock/summary.mock.ts`
- [ ] `src/features/team-space/mock/operatingModel.mock.ts`
- [ ] `src/features/team-space/mock/members.mock.ts`
- [ ] `src/features/team-space/mock/templates.mock.ts`
- [ ] `src/features/team-space/mock/pipeline.mock.ts`
- [ ] `src/features/team-space/mock/metrics.mock.ts`
- [ ] `src/features/team-space/mock/risks.mock.ts`
- [ ] `src/features/team-space/mock/projects.mock.ts`
- [ ] `src/features/team-space/mock/aggregate.mock.ts` — composes the above into a full aggregate

### A3: Build Reusable Card Primitives

- [ ] Card shell component (skeleton / error / empty slots) — reuse existing shared component if present
- [ ] `CoverageGapBanner.vue`
- [ ] `SdlcChainStrip.vue` — always renders 11 nodes, Spec highlighted

### A4: Build Workspace Summary Card

- [ ] `WorkspaceSummaryCard.vue` — renders identity line, health LED, counters, responsibility boundary chip row
- [ ] Handles compatibility mode (null SNOW group) with neutral chip
- [ ] Loading / error / (no empty state — always has identity)

### A5: Build Team Operating Model Card

- [ ] `TeamOperatingModelCard.vue`
- [ ] Each field row uses `<LineageBadge>` with hover tooltip
- [ ] "View in Platform Center" link gated by feature flag

### A6: Build Member & Role Matrix Card

- [ ] `MemberMatrixCard.vue`
- [ ] `CoverageGapBanner` at top if gaps exist
- [ ] Member rows sortable by last-active
- [ ] "Manage in Access Management" link

### A7: Build Team Default Templates Card

- [ ] `TeamTemplatesCard.vue`
- [ ] `TemplateGroup.vue` sub-component per template kind
- [ ] `ExceptionOverrideList.vue` section
- [ ] Empty state when no overrides

### A8: Build Pipeline Card

- [ ] `PipelineCard.vue`
- [ ] `PipelineCounters.vue` grid
- [ ] `PipelineBlockerList.vue`
- [ ] `SdlcChainStrip.vue` integrated
- [ ] Counter and blocker clicks deep-link to Requirement Management with filter vocabulary

### A9: Build Metrics Card

- [ ] `MetricsCard.vue` with 5 metric groups
- [ ] `MetricItem.vue` shows value, previous, trend indicator, history link
- [ ] Last-refreshed timestamp

### A10: Build Risk Radar Card

- [ ] `RiskRadarCard.vue` with groups in severity order
- [ ] `RiskItem.vue` with primary action
- [ ] Critical risks use crimson accent
- [ ] Empty state "All green"

### A11: Build Project Distribution Card

- [ ] `ProjectDistributionCard.vue` with stratum tabs
- [ ] `ProjectCard.vue` with name, stage, health, risk, counts
- [ ] Click navigates to `/project-space/:id` or stub

### A12: Build Pinia Store and API Client

- [ ] `frontend/src/features/team-space/api/teamSpaceApi.ts` using `fetchJson<T>()`
- [ ] `frontend/src/features/team-space/stores/teamSpaceStore.ts` with `initWorkspace`, `switchWorkspace`, `retryCard`, `refreshCard`, `reset`
- [ ] Mock toggle follows repo convention: `import.meta.env.DEV && !import.meta.env.VITE_USE_BACKEND`
- [ ] Per-card retry preserves other cards' state

### A13: Build TeamSpaceView

- [ ] `TeamSpaceView.vue` as top-level route view
- [ ] Mounts all 8 cards + SDLC chain strip
- [ ] Registers breadcrumb on mount
- [ ] Projects context to AI Command Panel
- [ ] Watches `route.query.workspaceId` for Workspace switch

### A14: Routing and Navigation Wiring

- [ ] Wire existing `/team` route to `TeamSpaceView`
- [ ] Carry Workspace scope via `workspaceId` query param (`/team?workspaceId=:id`)
- [ ] Remove Team Space `comingSoon` behavior from the shared shell nav config when the slice is enabled
- [ ] Add Dashboard drill-in link to Team Space
- [ ] Wire deep-link navigation for pipeline / risks / projects / metrics using `/platform` and `/reports` namespaces
- [ ] Feature-flag stub navigation for Platform Center / Project Space / Report Center

### A15: States and Error Isolation

- [ ] Per-card skeletons during load
- [ ] Per-card error with retry
- [ ] Per-card empty states (distinct copy per card)
- [ ] Page-level error for access denial / auth

### A16: Validate Team Space Frontend

- [ ] `npm run dev` — browses Team Space at `/team?workspaceId=ws-default-001`
- [ ] `npm run build` passes
- [ ] Visual spot-check against design §2 layout
- [ ] Workspace switch re-loads cards without full reload
- [ ] Breadcrumb trail records on drill-down
- [ ] Vitest unit tests pass

### Phase A Definition of Done

- All 8 cards render with mock data
- SDLC chain strip renders 11 nodes with Spec highlighted
- Workspace switch works
- Drill-down navigation works (to Requirement Management with filter; stubs for Project Space / Platform Center)
- Per-card loading / error / empty states behave correctly
- Build passes; no console errors

---

## Phase B: Backend (Codex)

### B0: Shared Infrastructure (Prerequisite)

- [ ] Verify `com.sdlctower.shared.dto.ApiResponse` exists; reuse it for Team Space endpoints
- [ ] Verify `com.sdlctower.shared.dto.SectionResultDto` exists; reuse it for aggregate section envelopes
- [ ] Create Team Space-specific lineage / enum types only where no shared primitive already exists

### B1: Create Team Space Package Structure

- [ ] Create `com.sdlctower.domain.teamspace` package with controller, service, `dto/`, `projection/`, and `persistence/`

### B2: Create Team Space DTOs

- [ ] All DTOs per data-model §4 (records)
- [ ] All enums per data-model §2.3 (mirrored as Java enums)
- [ ] `FieldDto<T>`, `LineageDto`, `LineageHopDto` for inheritance display
- [ ] `SectionResultDto<T>` wrapper per aggregate card

### B3: Implement Projections

- [ ] `WorkspaceProjection` reading `WorkspaceRepositoryFacade` + `ApplicationRepositoryFacade`
- [ ] `OperatingModelProjection` resolving 4-layer inheritance chain via `ConfigInheritanceFacade`
- [ ] `MemberProjection` reading `MemberRepositoryFacade` + oncall computation + coverage gap detection
- [ ] `TemplateInheritanceProjection` reading `TemplateFacade` + grouping by kind
- [ ] `RequirementPipelineProjection` reading `RequirementReadFacade` + computing blockers per threshold
- [ ] `MetricsProjection` reading `metric_snapshots` table
- [ ] `RiskRadarProjection` reading `risk_signals` + grouping by category
- [ ] `ProjectDistributionProjection` reading `ProjectReadFacade` + stratification

### B4: Implement Team Space Service

- [ ] `TeamSpaceService.loadAggregate` with parallel fan-out + per-projection 500ms timeout
- [ ] `TeamSpaceService.load{Summary,OperatingModel,Members,Templates,Pipeline,Metrics,Risks,Projects}` for per-card endpoints
- [ ] Per-projection exception yields `SectionResultDto(data=null, error=...)`, not a top-level failure

### B5: Implement Access Guard

- [ ] `WorkspaceAccessGuard.check(workspaceId)` resolving the authenticated principal + verifying read access
- [ ] Throws `WorkspaceAccessDeniedException` on failure; mapped to 403 by global exception handler

### B6: Implement Team Space Controller

- [ ] `TeamSpaceController` with aggregate + 8 per-card endpoints
- [ ] `@Pattern` path validation on `workspaceId`
- [ ] Standard response envelope wrap

### B7: Create Flyway Migrations

- [ ] `V7__create_team_space_tables.sql` — `risk_signals`, `metric_snapshots`
- [ ] `V8__seed_team_space_data.sql` — seed rows for local dev
- [ ] Verify migrations run on H2 (local) without error
- [ ] Oracle variant migration if DDL needs adjustment (separate V-file or conditional DDL)

### B8: Implement Entities and Repositories

- [ ] `RiskSignalEntity`, `MetricSnapshotEntity` (JPA)
- [ ] `RiskSignalRepository`, `MetricSnapshotRepository` (Spring Data)
- [ ] Query methods for Workspace-scoped reads

### B9: Scheduled Jobs (Risk + Metric Refresh)

- [ ] `RiskRadarJob` @Scheduled every 15 minutes — computes risks per Workspace
- [ ] `MetricsSnapshotJob` @Scheduled nightly — computes metric snapshots
- [ ] Both jobs idempotent and resumable

### B10: Backend Tests

- [ ] `TeamSpaceControllerTest` (MockMvc): happy path, 400, 403, 404, per-projection error isolation
- [ ] `TeamSpaceServiceTest`: parallel fan-out behavior, timeout degradation
- [ ] Per-projection unit tests with stub facades
- [ ] `WorkspaceAccessGuardTest`
- [ ] Repository tests against H2

### B11: Connect Frontend to Backend

- [ ] Set `VITE_USE_BACKEND=true` in dev config
- [ ] Verify Vite proxy routes `/api` to backend
- [ ] Smoke test: browse `/team?workspaceId=ws-default-001` against live backend
- [ ] Verify error cases (invalid workspace id, denied access) render correctly

### B12: Validate Full Stack

- [ ] `./mvnw verify` passes
- [ ] `npm run build` passes
- [ ] End-to-end browse with mock=false shows all 8 cards hydrated from real data
- [ ] Per-card retry works end-to-end
- [ ] Workspace switch re-fetches from backend

### Phase B Definition of Done

- All endpoints return correct envelopes for happy path
- Per-projection timeout degrades to section-level error only
- Flyway V7 + V8 migrations run cleanly
- Frontend works in live mode (`VITE_USE_BACKEND=true`)
- Backend + frontend builds pass
- Team Space accessible from Dashboard drill-in and direct URL

---

## Dependency Plan

### Critical Path

```
A0 (shared infra) → A1 (types) → A2 (mocks) → A3 (primitives)
   → A4…A11 (cards in parallel) → A12 (store/API) → A13 (view) → A14 (routing) → A15 (states) → A16 (validate)
        │
        ↓
B0 (shared backend) → B1 (package) → B2 (DTOs) → B3 (projections in parallel)
   → B4 (service) → B5 (guard) → B6 (controller) → B7 (migrations) → B8 (entities/repo) → B9 (jobs)
   → B10 (tests) → B11 (connect) → B12 (validate)
```

### Parallel Workstreams

- A4 through A11 (eight card components) can be built in parallel by multiple contributors
- B3 projections can be built in parallel once DTOs are defined
- Frontend Phase A can proceed independently of Phase B backend work up through A16

---

## Risks / Blockers

| Risk | Mitigation |
|------|-----------|
| Upstream facades (WorkspaceRepositoryFacade, TemplateFacade, etc.) not yet exposed | Confirm with shared-platform team before Phase B kickoff; scaffold stubs if absent |
| Scheduled job scheduling collides with existing cron infrastructure | Reuse existing `@Scheduled` config; verify no conflicting fixed-rate jobs |
| Metric / risk source data unavailable in dev | Use seed data from V8 migration; scheduled jobs degrade gracefully when sources empty |
| Platform Center / Project Space stubs feel like broken navigation | Feature flags + clearly labeled placeholder pages |
| Inheritance resolution latency | Cache with short TTL (60s); benchmark in B10 |
| Oracle DDL incompatibility | Split migrations per-dialect if needed; verify on Oracle-in-Docker during B7 |

---

## Open Questions

Inherited from spec:

- Metric definitions for Governance Maturity / AI Participation — confirm with Platform Center owners before B3 metric projection
- Project lifecycle stage enum — confirm authoritative source before B3 project projection
- Default pipeline blocker threshold — proposed 3 days; confirm before B4
- Team-Space-specific AI skill — if none, reuse dashboard skills in A13
- Permission-gated link strategy (hide vs disable) — confirm before A5 / A6
