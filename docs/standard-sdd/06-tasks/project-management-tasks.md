# Project Management Tasks

## Objective

Deliver the Project Management slice end-to-end in two phases: Phase A (frontend with mocks, Gemini) and Phase B (backend wiring, Codex). Scope covers BOTH the portfolio-level cross-project view AND the per-project plan execution view in a single slice, per the scope decision recorded in `project-management-architecture.md` §Decisions (D1). Implementation must conform to the 9 upstream SDD documents.

## Implementation Strategy

- **Frontend-first (Phase A):** mock-driven, no backend dependency. Ship a browsable Portfolio view and a Plan view for a seeded project. All mutations go through a mock API layer that round-trips realistic envelopes (including `planRevision` stale-token simulation).
- **Backend (Phase B):** implement projections, command services, controller, entities, and Flyway migrations V20–V26. Swap frontend from mock mode to live API by toggling `VITE_USE_BACKEND=true`. No `ddl-auto`.
- **Per-card isolation:** each Portfolio card (6 total) and each Plan card (9 total) is independently developable, mockable, and testable.
- **Reuse-and-extend:** Milestone, Risk, Dependency, Project, and Member entities are reused from Project Space / Team Space and extended with PM-specific columns. Only CapacityAllocation, PlanChangeLogEntry, and AiSuggestion are net-new entities.
- **Write authority:** all mutations of Milestone / Risk / Dependency are rooted in Project Management from this slice onward. Project Space reads them (no longer stewards). Contract enforced by the backend `/plan/*` mutation endpoints.
- **Shared primitives:** live under `src/shared/` / `com.sdlctower.shared` — PM does not fork shell-level behavior.

## Traceability

- Spec: [project-management-spec.md](../03-spec/project-management-spec.md)
- Design: [project-management-design.md](../05-design/project-management-design.md)
- Architecture: [project-management-architecture.md](../04-architecture/project-management-architecture.md)
- Data Flow: [project-management-data-flow.md](../04-architecture/project-management-data-flow.md)
- Data Model: [project-management-data-model.md](../04-architecture/project-management-data-model.md)
- API Guide: [project-management-API_IMPLEMENTATION_GUIDE.md](../05-design/contracts/project-management-API_IMPLEMENTATION_GUIDE.md)
- Requirements: [project-management-requirements.md](../01-requirements/project-management-requirements.md)
- Stories: [project-management-stories.md](../02-user-stories/project-management-stories.md)

## Planning Assumptions

- `shared-app-shell`, `dashboard`, `requirement`, `incident`, `team-space`, `project-space` slices are merged and operational.
- Design Management, Code & Build, Testing, Deployment Management, Platform Center remain stubbed; PM deep-links to them are feature-flagged.
- H2 is the local dev DB; Oracle is the deployed DB. Flyway migrations V20–V26 must run on both (split per-dialect if DDL diverges).
- Java 21 + Spring Boot 3.x; Vue 3 + Vite + Pinia + TypeScript.
- No localStorage / sessionStorage (per shared shell rules).
- `SectionResult<T>`, `ApiResponse<T>`, `fetchJson<T>`, `LineageBadge.vue` already exist and are reused as-is.
- Base path for all endpoints: `/api/v1/project-management` (per API guide §1).
- Mock toggle pattern follows repo convention: `import.meta.env.DEV && !import.meta.env.VITE_USE_BACKEND`.
- All mutations return an updated `planRevision`; stale tokens cause `PM_STALE_REVISION` (409).
- Access: portfolio visible to users with workspace-level membership; plan mutations require PM or Tech Lead role on that project.

---

## Phase A: Frontend (Codex)

### A0: Shared Infrastructure Preparation

- [ ] Reuse `src/shared/types/section.ts` for `SectionResult<T>`
- [ ] Reuse `src/shared/api/client.ts` for `fetchJson<T>`
- [ ] Reuse `src/shared/components/LineageBadge.vue` for provenance badges
- [ ] Reuse `shellUiStore.setBreadcrumbs(...)` + `shellUiStore.setAiPanelContent(...)`
- [ ] Confirm Tactical Command tokens already exported from shared design tokens (crimson `--pm-crimson`, amber `--pm-amber`, LED `--pm-led-*`); add missing tokens to `src/shared/styles/tokens.css` only if absent

### A1: Define Project Management Types

- [ ] `src/features/project-management/types/enums.ts` — all enums per data-model §2.2 (MilestoneState, RiskState, DependencyState, AiSuggestionState, CapacityBucket, PlanChangeEntryType)
- [ ] `src/features/project-management/types/portfolio.ts` — `PortfolioSummary`, `PortfolioHeatmapRow`, `CapacityAllocationSummary`, `RiskConcentration`, `DependencyBottleneck`, `CadenceMetrics`
- [ ] `src/features/project-management/types/plan.ts` — `PlanHeader`, `MilestonePlan`, `RiskRegistry`, `DependencyGraph`, `CapacityPlan`, `DeliveryProgress`, `PlanChangeLog`
- [ ] `src/features/project-management/types/mutations.ts` — command request/response types for every mutation endpoint (§3 of API guide)
- [ ] `src/features/project-management/types/aggregate.ts` — `PortfolioAggregate`, `PlanAggregate`, `ProjectManagementState`
- [ ] `src/features/project-management/types/suggestions.ts` — `AiSuggestion`, `AiSuggestionAction`

### A2: Build Mock Data

- [ ] `src/features/project-management/mock/portfolio.mock.ts` — seeds 8 projects across 3 workspaces, mixed health LEDs, one at-risk cluster
- [ ] `src/features/project-management/mock/milestoneHeatmap.mock.ts`
- [ ] `src/features/project-management/mock/capacity.mock.ts` — includes one over-allocation case
- [ ] `src/features/project-management/mock/riskConcentration.mock.ts`
- [ ] `src/features/project-management/mock/dependencyBottlenecks.mock.ts`
- [ ] `src/features/project-management/mock/cadence.mock.ts`
- [ ] `src/features/project-management/mock/planHeader.mock.ts`
- [ ] `src/features/project-management/mock/planMilestones.mock.ts` — all 6 milestone states represented
- [ ] `src/features/project-management/mock/planRisks.mock.ts` — IDENTIFIED / MITIGATING / ESCALATED
- [ ] `src/features/project-management/mock/planDependencies.mock.ts` — internal + external examples
- [ ] `src/features/project-management/mock/planCapacity.mock.ts`
- [ ] `src/features/project-management/mock/planProgress.mock.ts`
- [ ] `src/features/project-management/mock/planChangeLog.mock.ts` — 15 entries covering all 7 entry types
- [ ] `src/features/project-management/mock/aiSuggestions.mock.ts` — 3 PENDING, 1 ACCEPTED, 1 DISMISSED
- [ ] `src/features/project-management/mock/aggregate.mock.ts` — composes portfolio + plan aggregates
- [ ] Mock mutation layer: `src/features/project-management/mock/commandLoop.ts` — simulates `planRevision` increment, stale-token rejection (5% injected), invalid transitions, access errors

### A3: Build Reusable Primitives

- [ ] `PlanRevisionToken.vue` — small monospace badge rendering current `planRevision`
- [ ] `StateTransitionChip.vue` — milestone / risk / dependency state chip with crimson / amber / LED palette
- [ ] `SlippageReasonField.vue` — textarea + validation when transition requires reason
- [ ] `OverallocationBanner.vue` — red banner for capacity > 100% with justification field
- [ ] `HeatmapCell.vue` — milestone heatmap cell with tooltip, color-coded by aggregate state
- [ ] `AiSuggestionCard.vue` — PENDING suggestion with Accept / Dismiss actions, 24h suppress after dismiss
- [ ] `ChangeLogRow.vue` — timestamp, actor, action, entity ref, before→after, reason
- [ ] `CapacityBar.vue` — stacked bar showing allocation across projects with over-allocation indicator
- [ ] `RiskSeverityBadge.vue` — severity chip (CRITICAL / HIGH / MEDIUM / LOW) with LED ring
- [ ] `DependencyHealthPill.vue` — internal / external chip with health LED
- [ ] `CadenceGauge.vue` — mini gauge for throughput / cycle time / WIP

### A4: Build Portfolio View Cards (6 cards)

- [ ] `PortfolioSummaryBarCard.vue` — aggregate across all visible projects: LED, active milestones, risk count, over-allocated people, on-track %
- [ ] `MilestoneHeatmapCard.vue` — project × week grid, each cell `<HeatmapCell>`; click cell deep-links to Plan view
- [ ] `CapacityAllocationCard.vue` — person-level `<CapacityBar>` rows, over-allocated shown first; filter by workspace
- [ ] `RiskConcentrationCard.vue` — projects sorted by weighted risk; CRITICAL rows crimson
- [ ] `DependencyBottlenecksCard.vue` — bottleneck edges (blocked dependencies) with source / target / owner / age
- [ ] `CadenceMetricsCard.vue` — 3 `<CadenceGauge>` instances (throughput / cycle time / WIP) per workspace

### A5: Build Plan View Cards (9 cards)

- [ ] `PlanHeaderCard.vue` — project identity + PM + Tech Lead + current milestone + LED + `<PlanRevisionToken>` + last updated
- [ ] `MilestonePlannerCard.vue` — editable milestone list, row-level state transition via `<StateTransitionChip>`, slippage reason on SLIPPED, creates change log entry on save
- [ ] `CapacityAllocationPlanCard.vue` — per-person allocation editor, weekly buckets, batch save with `<OverallocationBanner>` when applicable
- [ ] `RiskRegistryEditorCard.vue` — risk rows with editable severity / owner / state; ESCALATED transition requires incident link
- [ ] `DependencyResolverCard.vue` — upstream / downstream list, state transitions require counter-sign for APPROVED; external blockers highlighted
- [ ] `DeliveryProgressStripCard.vue` — 11-node SDLC chain mini-strip showing count of open items per node (read-only, deep-links into each domain)
- [ ] `PlanChangeLogCard.vue` — reverse-chronological `<ChangeLogRow>` list with filter by entry type / actor / date range
- [ ] `AiSuggestionsCard.vue` — PENDING suggestions list, Accept / Dismiss, 24h suppression window
- [ ] `GovernancePanelCard.vue` — approvals needed (milestone baseline change, dependency counter-sign), routed via shared governance pipeline

### A6: Build Pinia Store and API Client

- [ ] `frontend/src/features/project-management/api/projectManagementApi.ts` using `fetchJson<T>()` — one function per endpoint from API guide §2–§3 (30 total)
- [ ] `frontend/src/features/project-management/stores/projectManagementStore.ts` with:
  - [ ] `initPortfolio(workspaceIds?)`, `refreshPortfolioCard(cardKey)`
  - [ ] `initPlan(projectId)`, `refreshPlanCard(cardKey)`
  - [ ] Mutation actions: `transitionMilestone`, `saveMilestoneBaseline`, `saveCapacityBatch`, `escalateRisk`, `resolveRisk`, `counterSignDependency`, `rejectDependency`, `acceptAiSuggestion`, `dismissAiSuggestion`
  - [ ] `planRevision` tracking: every mutation action reads current revision, sends it, stores returned revision on success, surfaces `PM_STALE_REVISION` as a refresh-prompt toast
  - [ ] `reset()` on unmount
- [ ] Mock toggle: `import.meta.env.DEV && !import.meta.env.VITE_USE_BACKEND`
- [ ] Per-card retry preserves other cards' state
- [ ] Mutations update only the affected card's state + log entry (no full refetch)

### A7: Build PortfolioView

- [ ] `PortfolioView.vue` as top-level route view for `/project-management`
- [ ] Mounts 6 portfolio cards in 12-column grid per design §3.1
- [ ] Breadcrumb: `Dashboard / Project Management`
- [ ] Projects AI summary via `shellUiStore.setAiPanelContent(...)` — portfolio-scoped
- [ ] Watches `route.query.workspaceIds` for scope filter
- [ ] Renders per-card loading / error / empty states

### A8: Build PlanView

- [ ] `PlanView.vue` as top-level route view for `/project-management/plan/:projectId`
- [ ] Mounts `<PlanHeaderCard>` + 8 other plan cards in 12-column grid per design §3.2
- [ ] Breadcrumb: `Dashboard / Project Management / Plan: {projectName}`
- [ ] Projects plan-scoped AI summary
- [ ] Watches `route.params.projectId` for plan switch
- [ ] Preserves optional `workspaceId` query on drill-in from portfolio heatmap

### A9: Routing and Navigation Wiring

- [ ] Add `/project-management` route → `PortfolioView`
- [ ] Add `/project-management/plan/:projectId` route → `PlanView`
- [ ] Remove Project Management `comingSoon` flag from shared shell nav config
- [ ] Deep-link: Portfolio heatmap cell → Plan view for that project at that milestone
- [ ] Deep-link: Plan `<DeliveryProgressStripCard>` nodes → corresponding domain pages (Requirement, Incident, Project Space, etc.)
- [ ] Deep-link: Plan `<RiskRegistryEditorCard>` escalated row → incident detail
- [ ] Deep-link: Plan `<DependencyResolverCard>` row → counterpart Project Space or external ticket
- [ ] Ensure keyboard navigation and ARIA labels (tab order per card, Enter to open editors, Esc to cancel inline edits)

### A10: States and Error Isolation

- [ ] Per-card skeletons during load (both views)
- [ ] Per-card error with retry
- [ ] Per-card empty states (distinct copy per card; e.g. "No milestones yet" vs. "No risks registered")
- [ ] Page-level error for access denial / invalid projectId / 404
- [ ] Mutation errors surfaced via toast + inline validation (slippage reason missing, overallocation missing justification, stale revision)
- [ ] AI suggestions: when `commandLoop.ts` returns DISMISSED with 24h suppress, card hides that suggestion until mocked clock advances

### A11: Validate Project Management Frontend

- [ ] `npm run dev` — browses `/project-management` and `/project-management/plan/proj-8821`
- [ ] `npm run build` passes
- [ ] Visual spot-check against design §3 layouts and any referenced mockup
- [ ] Portfolio → Plan drill works; back-navigation preserves portfolio scroll / scope filter
- [ ] Plan switch re-loads cards without full page reload
- [ ] Every mutation updates `planRevision`; intentional stale-token injection surfaces `PM_STALE_REVISION` prompt
- [ ] AI suggestion accept → entity updates, dismiss → 24h suppression
- [ ] Vitest unit tests pass for: primitives, store actions (including revision fencing), mock commandLoop

### Phase A Definition of Done

- Portfolio view renders 6 cards with realistic mock data including an at-risk cluster + over-allocation example
- Plan view renders 9 cards with realistic mock data covering all state values
- All mutations round-trip through mock `commandLoop.ts` and update `planRevision`
- Stale-revision path produces the refresh-prompt toast
- AI suggestion lifecycle (Pending → Accept / Dismiss) works including 24h suppress
- Per-card loading / error / empty states behave correctly
- Build passes; no console errors; mutations produce change log entries

---

## Phase B: Backend (Codex)

### B0: Shared Infrastructure (Prerequisite)

- [ ] Verify `com.sdlctower.shared.dto.ApiResponse` exists; reuse for Project Management endpoints
- [ ] Verify `com.sdlctower.shared.dto.SectionResultDto` exists; reuse for aggregate section envelopes
- [ ] Add `PROJECT_MANAGEMENT` constant to `com.sdlctower.shared.ApiConstants` with base path `/api/v1/project-management`
- [ ] Confirm governance pipeline hook (`GovernanceRequestPublisher`) is available in shared; if not, document as a blocker for approval-gated mutations

### B1: Create Project Management Package Structure

- [ ] Create `com.sdlctower.domain.projectmanagement` package with:
  - `controller/` — `PortfolioController`, `PlanController`
  - `service/` — `PortfolioService`, `PlanService`, `MilestoneCommandService`, `RiskCommandService`, `DependencyCommandService`, `CapacityCommandService`, `AiSuggestionService`
  - `policy/` — `PlanPolicy` (role gating), `RevisionFencingPolicy` (optimistic concurrency), `TransitionPolicy` (state machines)
  - `projection/` — all read-path projections
  - `dto/` — records per data-model §4
  - `persistence/` — JPA entities + repositories
  - `events/` — plan change log entry publisher, AI suggestion lifecycle events
- [ ] No cross-cutting logic leaks into `controller/` — controllers call service, never projection/repository directly

### B2: Create Project Management DTOs

- [ ] All DTOs per data-model §4 as Java 21 records
- [ ] All enums per data-model §2.2 mirrored as Java enums with JSON codecs
- [ ] Command request records: `TransitionMilestoneRequest`, `SaveMilestoneBaselineRequest`, `SaveCapacityBatchRequest`, `EscalateRiskRequest`, `ResolveRiskRequest`, `CounterSignDependencyRequest`, `RejectDependencyRequest`, `AcceptAiSuggestionRequest`, `DismissAiSuggestionRequest`
- [ ] Command response records include updated `planRevision`
- [ ] Reuse `LinkDto`, `MemberRefDto`, `MilestoneRefDto` from shared
- [ ] `SectionResultDto<T>` wrapper per aggregate card

### B3: Implement Projections

- [ ] `PortfolioSummaryProjection` — aggregates across projects the caller can read; per-project LED + counts
- [ ] `MilestoneHeatmapProjection` — project × week matrix keyed by milestone aggregate state
- [ ] `CapacityAllocationProjection` — person-level allocation rows pulled from `capacity_allocation` joined with `member`
- [ ] `RiskConcentrationProjection` — project-level weighted risk score; ordered DESC
- [ ] `DependencyBottlenecksProjection` — blocked dependencies across visible projects, age DESC
- [ ] `CadenceMetricsProjection` — throughput / cycle time / WIP per workspace (delegates to analytics platform if available; stub otherwise)
- [ ] `PlanHeaderProjection` — project identity + PM + Tech Lead + current milestone + `planRevision`
- [ ] `MilestonePlanProjection` — extended milestone entity rows for the project, ordered by target date
- [ ] `RiskRegistryProjection` — project-scoped `risk_signals` extended rows, ordered severity DESC, ageDays DESC
- [ ] `DependencyGraphProjection` — `project_dependency` extended rows split upstream / downstream
- [ ] `CapacityPlanProjection` — allocation rows for members of that project
- [ ] `DeliveryProgressProjection` — counts per SDLC chain node; delegates to domain facades (Requirement, Incident, etc.)
- [ ] `PlanChangeLogProjection` — reverse-chron entries filtered by project id
- [ ] `AiSuggestionsProjection` — PENDING suggestions for that plan (excluding suppressed-window dismissals)

### B4: Implement Read Services

- [ ] `PortfolioService.loadAggregate(workspaceIds)` with parallel fan-out over 6 projections, per-projection 500ms timeout → `SectionResultDto(data=null, error=...)` on failure
- [ ] `PortfolioService.load{Summary,Heatmap,Capacity,RiskConcentration,DependencyBottlenecks,Cadence}` for per-card endpoints
- [ ] `PlanService.loadAggregate(projectId)` with parallel fan-out over 9 projections, per-projection 500ms timeout
- [ ] `PlanService.load{Header,Milestones,Capacity,Risks,Dependencies,Progress,ChangeLog,AiSuggestions,Governance}` for per-card endpoints
- [ ] All read methods emit zero writes; safe to retry

### B5: Implement Command Services

- [ ] `MilestoneCommandService.transition(projectId, milestoneId, request)` — validates transition via `TransitionPolicy.milestone()`, enforces slippage reason, bumps `planRevision`, writes `PlanChangeLogEntry`
- [ ] `MilestoneCommandService.saveBaseline(projectId, request)` — governance-gated (publishes approval request); on approval, writes baseline + change log
- [ ] `RiskCommandService.escalate(projectId, riskId, request)` — requires incident link; transitions to ESCALATED, bumps revision, writes change log
- [ ] `RiskCommandService.resolve(projectId, riskId, request)` — transitions to RESOLVED, writes change log
- [ ] `DependencyCommandService.counterSign(projectId, dependencyId, request)` — requires counter-signer to be PM/Tech Lead on target project (counter-sign check via `DependencyCounterSignPolicy`); transitions NEGOTIATING → APPROVED, bumps revision on both projects
- [ ] `DependencyCommandService.reject(projectId, dependencyId, request)` — transitions to REJECTED, writes change log entries on both sides
- [ ] `CapacityCommandService.saveBatch(projectId, request)` — validates per-person total ≤ 100% unless `justification` present; bulk upserts `capacity_allocation`; writes aggregated change log entry
- [ ] `AiSuggestionService.accept(projectId, suggestionId)` — applies suggested action via the relevant command service (milestone baseline / capacity rebalance / dependency flag), marks ACCEPTED
- [ ] `AiSuggestionService.dismiss(projectId, suggestionId, reason)` — marks DISMISSED, sets `suppressUntil = now() + 24h`
- [ ] All command services enforce `RevisionFencingPolicy.check(projectId, request.planRevision)` FIRST; mismatch → `PM_STALE_REVISION` (409)
- [ ] All command services enforce `PlanPolicy.requireWrite(projectId, principal)` — role must be PM or Tech Lead on that project
- [ ] Every successful mutation publishes a `PlanChangeLogEntry` via `PlanChangeLogPublisher`

### B6: Implement Policies

- [ ] `PlanPolicy.requireRead(projectId, principal)` — workspace membership check
- [ ] `PlanPolicy.requireWrite(projectId, principal)` — role is PM or Tech Lead on that project
- [ ] `RevisionFencingPolicy.check(projectId, clientRevision)` — compares to `plan_revision` table
- [ ] `TransitionPolicy.milestone(from, to)` / `.risk(...)` / `.dependency(...)` — matches state machines in data-flow doc
- [ ] `DependencyCounterSignPolicy.canCounterSign(dependencyId, principal)` — counter-signer must be PM/Tech Lead on the TARGET project
- [ ] Exception classes: `PlanAccessDeniedException`, `InvalidTransitionException`, `StaleRevisionException`, `SlippageReasonRequiredException`, `OverallocationJustificationRequiredException`, `IncidentLinkRequiredException`
- [ ] `@ExceptionHandler` entries in `GlobalExceptionHandler` mapping each to its error code + HTTP status per API guide §4

### B7: Implement Controllers

- [ ] `PortfolioController` with 7 endpoints (aggregate + 6 per-card) per API guide §2.1
- [ ] `PlanController` with 9 read endpoints + 13 mutation endpoints + 1 internal endpoint per API guide §2.2 + §3
- [ ] `@Pattern` path validation on `projectId` (`^proj-[a-z0-9\-]+$`)
- [ ] `@RequestBody @Valid` on all command endpoints
- [ ] Standard `ApiResponse<T>` envelope wrap via shared `ResponseBodyAdvice` or explicit `ApiResponse.ok(...)` calls
- [ ] Mutation endpoints return `200 OK` with updated revision; not `204`

### B8: Create Flyway Migrations

- [ ] `V20__extend_milestone_pm_columns.sql` — add `pm_state`, `baseline_target_date`, `slippage_reason`, `slippage_acknowledged_at`, `ai_suggestion_id`, `plan_revision_at_update` columns to existing `milestone` table
- [ ] `V21__extend_risk_signal_pm_columns.sql` — add `pm_state`, `escalated_incident_id`, `owner_member_id`, `plan_revision_at_update` columns to existing `risk_signal` table
- [ ] `V22__extend_project_dependency_pm_columns.sql` — add `pm_state`, `counter_signed_by`, `counter_signed_at`, `plan_revision_at_update` columns to existing `project_dependency` table
- [ ] `V23__create_capacity_allocation.sql` — new table with `id`, `project_id`, `member_id`, `week_start`, `allocation_percent`, `justification`, `created_at`, `updated_at`, index on `(project_id, week_start)` and `(member_id, week_start)`
- [ ] `V24__create_plan_change_log.sql` — new table with `id`, `project_id`, `entry_type`, `actor_id`, `entity_kind`, `entity_id`, `before_json`, `after_json`, `reason`, `created_at`, index on `(project_id, created_at DESC)`
- [ ] `V25__create_ai_suggestion.sql` — new table with `id`, `project_id`, `suggestion_kind`, `payload_json`, `state`, `suppress_until`, `accepted_by`, `dismissed_by`, `dismissed_reason`, `created_at`, `updated_at`, index on `(project_id, state)`
- [ ] `V26__seed_project_management_local.sql` — local-only seed rows: extend existing seed projects with PM revisions, seed 5 capacity rows, 3 AI suggestions, 8 change log entries
- [ ] Also create `plan_revision` tracking column or table (decide: lightweight is a `plan_revision` column on `project` table; if present in V9, reuse; if not, add in V20 as a dedicated column `project.plan_revision BIGINT NOT NULL DEFAULT 0`)
- [ ] Verify all migrations run on H2 (local) without error
- [ ] Produce Oracle variant if DDL diverges (e.g. `JSON` column type, identity columns)

### B9: Implement Entities and Repositories

- [ ] Extend `MilestoneEntity` with new PM columns (map to V20 additions)
- [ ] Extend `RiskSignalEntity` with new PM columns (V21)
- [ ] Extend `ProjectDependencyEntity` with new PM columns (V22)
- [ ] New: `CapacityAllocationEntity`, `PlanChangeLogEntryEntity`, `AiSuggestionEntity`
- [ ] Repositories: `CapacityAllocationRepository`, `PlanChangeLogEntryRepository`, `AiSuggestionRepository`
- [ ] Query methods:
  - `findByProjectIdAndWeekStartBetween(projectId, from, to)` on capacity
  - `findByProjectIdOrderByCreatedAtDesc(projectId, Pageable)` on change log
  - `findByProjectIdAndState(projectId, state)` on AI suggestion; exclude those with `suppressUntil > now`
- [ ] Verify Project Space queries still pass against extended tables (additive-only columns)

### B10: Governance & Lineage Integration

- [ ] `MilestoneBaselineApprovalHandler` — consumes governance approval events; on approval, applies the queued baseline change
- [ ] `DependencyCounterSignApprovalHandler` — optional if counter-sign is governance-routed rather than direct; confirm against governance platform
- [ ] Ensure every write path writes a `LineageEvent` identifying PM as the authoring domain (reuse shared `LineageEmitter`)
- [ ] Confirm Project Space reads do NOT bypass PM write authority — Project Space repository reads become read-only views on the extended tables

### B11: Backend Tests

- [ ] `PortfolioControllerTest` (MockMvc): happy path, 400 invalid query, 403 denied, per-projection error isolation
- [ ] `PlanControllerTest`: all read endpoints + every mutation endpoint (happy + stale revision + invalid transition + access denied + validation failure)
- [ ] `MilestoneCommandServiceTest`: transitions follow state machine; slippage reason enforcement; change log entry written; revision bumped
- [ ] `RiskCommandServiceTest`: escalation requires incident link; resolution writes log
- [ ] `DependencyCommandServiceTest`: counter-sign requires PM/Tech Lead on target; rejection path
- [ ] `CapacityCommandServiceTest`: over-allocation requires justification; batch upsert correctness
- [ ] `AiSuggestionServiceTest`: accept applies downstream command; dismiss sets 24h suppress
- [ ] `RevisionFencingPolicyTest`: stale → 409 `PM_STALE_REVISION`
- [ ] `TransitionPolicyTest`: every forbidden transition blocked
- [ ] `PlanPolicyTest`: read vs write gating, counter-sign target rule
- [ ] Repository tests against H2 for new entities + extended queries
- [ ] `FlywayMigrationIntegrationTest`: applies V20–V26 on a clean H2; verifies target schema + seed counts
- [ ] Golden-file test for API envelopes — compare response JSON to canonical examples in API guide §2–§3

### B12: Connect Frontend to Backend

- [ ] Set `VITE_USE_BACKEND=true` in dev config
- [ ] Verify Vite proxy routes `/api/v1/project-management/*` to backend
- [ ] Smoke: browse `/project-management` against live backend — 6 portfolio cards hydrated
- [ ] Smoke: browse `/project-management/plan/proj-8821` — 9 plan cards hydrated; planRevision visible
- [ ] Smoke: execute one mutation of each kind; verify `planRevision` increments on response and store updates
- [ ] Smoke: force stale revision (via second browser tab); verify refresh-prompt toast
- [ ] Verify error cases: invalid projectId → 400; non-member user → 403; unknown project → 404
- [ ] Verify deep-links from Portfolio heatmap → Plan view function end-to-end

### B13: Validate Full Stack

- [ ] `./mvnw verify` passes (all new tests + existing suites)
- [ ] `npm run build` passes
- [ ] End-to-end: execute one milestone transition, one capacity save, one risk escalation, one dependency counter-sign, one AI suggestion accept — each produces a change log entry visible in `<PlanChangeLogCard>`
- [ ] Project Space slice still renders correctly against extended tables (no regressions)
- [ ] Team Space risk radar job still functions against extended `risk_signal` table
- [ ] Governance approval round-trip: milestone baseline change queued → governance approves → plan reflects new baseline + change log entry

### Phase B Definition of Done

- All 30 endpoints return correct envelopes for happy path
- Every mutation enforces revision fencing, role gating, and state-machine transitions
- Per-projection timeout degrades to section-level error only (never page-level failure for read)
- Flyway V20–V26 migrations run cleanly on H2
- Frontend works in live mode (`VITE_USE_BACKEND=true`)
- Backend + frontend builds pass; golden-file API tests match API guide exactly
- Write authority for Milestone / Risk / Dependency demonstrably owned by PM (Project Space does not mutate)
- Governance round-trip functional for baseline changes

---

## Dependency Plan

### Critical Path

```
A0 (shared infra) → A1 (types) → A2 (mocks) → A3 (primitives)
   → A4 (portfolio cards 6x in parallel) + A5 (plan cards 9x in parallel) → A6 (store/API)
   → A7 (PortfolioView) + A8 (PlanView) → A9 (routing) → A10 (states) → A11 (validate)
        │
        ↓
B0 (shared backend) → B1 (package) → B2 (DTOs)
   → B3 (projections in parallel) + B5 (command services depend on B2) → B4 (read services)
   → B6 (policies) → B7 (controllers) → B8 (migrations) → B9 (entities/repos)
   → B10 (governance/lineage) → B11 (tests) → B12 (connect) → B13 (validate)
```

### Parallel Workstreams

- A4 (six portfolio cards) can be built in parallel by multiple contributors
- A5 (nine plan cards) can be built in parallel by multiple contributors
- B3 projections can be built in parallel once DTOs are defined
- B5 command services can be split per entity (Milestone / Risk / Dependency / Capacity / AiSuggestion)
- Frontend Phase A can proceed independently of backend work up through A11
- Flyway migrations (B8) can be drafted in parallel with entity work (B9) but must land before entity tests pass

---

## Risks / Blockers

| Risk                                                                                                                    | Mitigation                                                                                                                                                                                  |
| ----------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Adding columns to `milestone` / `risk_signal` / `project_dependency` risks Project Space / Team Space regressions | Additive + nullable with safe defaults; dedicated regression runs on dependent slices in B13                                                                                                |
| Write-authority transfer (Project Space previously stewarded milestones) may leak residual writes                       | Remove all Project Space mutation endpoints that touch these entities; verify via controller test audit; Project Space becomes read-only on these tables                                    |
| Governance pipeline (milestone baseline approval) not yet available in shared platform                                  | Implement command service with feature flag `pm.governance.enabled`; when off, apply baseline change immediately but still write change log entry; re-enable when shared governance lands |
| Counter-sign semantics (dependency) require cross-project role resolution                                               | `DependencyCounterSignPolicy` explicitly queries target project's PM/Tech Lead; document failure mode if target project is in a different tenant                                          |
| `planRevision` contention on active projects (multiple PMs editing)                                                   | Revision fencing is per-project scalar; clients prompted to refresh on 409; explore entity-scoped fencing in V2 if contention is observed                                                   |
| AI suggestion payload schema churn                                                                                      | `ai_suggestion.payload_json` stored as opaque JSON; versioned by `suggestion_kind`; handlers must tolerate unknown kinds by marking the suggestion INELIGIBLE rather than erroring      |
| Oracle-specific DDL (JSON column, IDENTITY) differences from H2                                                         | Produce Oracle-variant migration V20–V26 if needed; verify on Oracle-in-Docker before deployment                                                                                           |
| Capacity over-allocation justifications pile up without review                                                          | Portfolio view's Capacity card highlights justified over-allocations separately; flagged as open question below                                                                             |
| Shell-level nav for Project Management still feature-flagged in some environments                                       | A9 removes `comingSoon`; coordinate with shell owners before merge                                                                                                                        |
| 11-node SDLC chain counts (Delivery Progress Strip) require upstream domain facades                                     | Use the same facade pattern as Project Space's `ChainNodeProjection`; default to `UNKNOWN` for absent domains                                                                           |

---

## Open Questions

- Milestone baseline approval — governance-routed (asynchronous) or PM-immediate with audit? Default: governance-routed, behind `pm.governance.enabled`.
- Capacity justification review workflow — does portfolio view surface a dedicated queue, or is inspection ad-hoc via filter? Default: filter-only in V1.
- AI suggestion re-generation cadence — event-driven vs. scheduled? Default: scheduled nightly + event-driven on major plan changes.
- Counter-sign tenancy — can a dependency cross tenants? Default: no for V1; flag for V2.
- Plan revision fencing scope — per-project scalar vs per-entity? Default: per-project scalar; measure contention before reconsidering.
- External dependency edges — is an external counterpart an email/ticket link, or an actual adjacent record? Default: link-only in V1.
- Governance "reason" copy — should slippage / overallocation reasons be persisted verbatim or NL-summarized by the platform? Default: verbatim; summarization deferred.
