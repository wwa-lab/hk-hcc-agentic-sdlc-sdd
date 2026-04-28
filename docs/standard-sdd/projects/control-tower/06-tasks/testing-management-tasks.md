# Testing Management Tasks

## Objective

Deliver the Testing Management slice end-to-end in seven phases: Phase 0 (scaffolding & Flyway baseline), Phase 1 (backend read-only MVP), Phase 2 (frontend read-only MVP), Phase 3 (CRUD mutations), Phase 4 (test-run ingestion pipeline), Phase 5 (AI test-case drafting), and Phase 6 (traceability & cross-slice integration). V1 scope is a **full QA lifecycle viewer** over test plans, cases, runs, coverage, and AI-drafted test candidates, with deep traceability to requirements. No test triggering from the Control Tower, no cross-workspace analytics in V1, per the scope decisions recorded in `testing-management-requirements.md` §3. Implementation must conform to the 8 upstream SDD documents (requirements, user-stories, spec, architecture, data-flow, data-model, design, API guide).

## Implementation Strategy

- **Phase 0 (Scaffolding):** Backend package structure, shared entities (Workspace, Project, Member), Flyway baseline, and backend shared infrastructure reuse.
- **Phase 1 (Backend MVP read-only):** Implement all projections, read services, controllers for Catalog, Plan Detail, Case Detail, Run Detail, and Traceability views. No mutations.
- **Phase 2 (Frontend MVP read-only):** Frontend types, mocks, mock service layer, Catalog, Plan Detail, Case Detail, Run Detail, and Traceability views. Codex-driven via mocks.
- **Phase 3 (CRUD mutations):** Plan, Case, and Environment CRUD endpoints and services; frontend forms and mutations via Codex.
- **Phase 4 (Run ingestion):** Webhook receiver, JUnit/TestNG/Playwright/Cypress parsers, outbox-backed dispatcher, log redaction, test-case-result persistence.
- **Phase 5 (AI drafting):** AI test-case-drafter skill invocation, DRAFT ↔ ACTIVE ↔ DEPRECATED state machine, approval flows, autonomy gating, stale-draft re-draft.
- **Phase 6 (Traceability & polishing):** Traceability aggregate endpoint, Requirement-slice "Tests" tab integration, per-card error handling, empty states, accessibility, performance tuning, a11y, i18n.

## Traceability

- Requirements: [testing-management-requirements.md](../01-requirements/testing-management-requirements.md)
- User Stories: [testing-management-stories.md](../02-user-stories/testing-management-stories.md)
- Spec: [testing-management-spec.md](../03-spec/testing-management-spec.md)
- Architecture: [testing-management-architecture.md](../04-architecture/testing-management-architecture.md)
- Data Flow: [testing-management-data-flow.md](../04-architecture/testing-management-data-flow.md)
- Data Model: [testing-management-data-model.md](../04-architecture/testing-management-data-model.md)
- Design: [testing-management-design.md](../05-design/testing-management-design.md)
- API Guide: [testing-management-API_IMPLEMENTATION_GUIDE.md](../05-design/contracts/testing-management-API_IMPLEMENTATION_GUIDE.md)

## Planning Assumptions

- `shared-app-shell`, `dashboard`, `requirement`, `team-space`, `project-space` slices are merged and operational.
- Shared shell primitives (`SectionResult<T>`, `ApiResponse<T>`, `fetchJson<T>`, `LineageBadge.vue`, `shellUiStore.setBreadcrumbs`) already exist and are reused as-is.
- The Requirement slice exposes a read-only `RequirementLookup` facade returning `(storyId, reqId, title, projectId, state)` — if not yet available, stub and document as upstream blocker.
- `WorkspaceAutonomyLookup` facade (from Platform Center) is reused; AI draft invocation gated by `aiAutonomyLevel >= OBSERVATION` (read-only view) or `SUPERVISED` (approval allowed).
- H2 is the local dev DB; Oracle is the deployed DB. Flyway migrations V50–V57 must run on both (split per-dialect if DDL diverges — notably CLOB for markdown bodies and failure excerpts).
- Java 21 + Spring Boot 3.x; Vue 3 + Vite + Pinia + TypeScript.
- Base path for all endpoints: `/api/v1/testing-management`.
- Mock toggle pattern: `import.meta.env.DEV && !import.meta.env.VITE_USE_BACKEND`.
- Access: all read views visible to workspace members with appropriate roles (PM, QA Lead, Tech Lead, Engineer); write access: QA Lead / Tech Lead (plans, cases, approvals); Run ingestion: webhooks + manual upload by QA Lead.
- No localStorage / sessionStorage (per shared shell rules).
- Both Frontend and Backend use **Codex** (per 2026-04-18 session decision, overriding CLAUDE.md default Gemini for FE).

---

## Phase 0: Scaffolding & Flyway Baseline

### Goal
Establish backend package structure, reuse shared entities, lay Flyway baseline, prepare for MVP implementation.

### Tasks — Backend (Codex)

- [ ] Confirm shared entities exist: `Workspace`, `Project`, `Member`, `ProjectRole`; stub reading facades if missing
- [ ] Create `com.sdlctower.domain.testingmanagement` package structure:
  - `controller/` — all REST endpoints
  - `service/` — read + command services
  - `projection/` — query projections
  - `ingestion/` — webhook receiver, parsers, outbox dispatcher
  - `integration/` — `RequirementLookup`, `AiSkillClient`, `ProjectRoleLookup` adapters
  - `policy/` — access guards, autonomy gate, log redactor, evidence validation
  - `dto/` — request/response records, enums
  - `persistence/` — JPA entities, repositories
  - `events/` — `TestingManagementChangeLogPublisher`, audit/lineage emitters
  - `resync/` — re-resolver jobs for unresolved REQ-IDs
- [ ] Create `src/shared/types/testing.ts` — shared enum mappings (if needed for FE/BE contract)
- [ ] Confirm `LineageEmitter`, `AuditLogEmitter` available in shared; reuse for all mutations (REQ-TM-82)
- [ ] Confirm `ProjectRoleLookup` facade exists; flag upstream if missing

### Tasks — Frontend (Codex)

- [ ] Reuse `src/shared/types/section.ts` for `SectionResult<T>`
- [ ] Reuse `src/shared/api/client.ts` for `fetchJson<T>`
- [ ] Reuse `src/shared/components/LineageBadge.vue`
- [ ] Reuse design tokens for coverage LEDs: `--tm-coverage-green`, `--tm-coverage-amber`, `--tm-coverage-red`, `--tm-coverage-grey`

### Tasks — Database (Flyway)

- [ ] `V50__create_testing_core.sql` — `test_plan`, `test_case`, `test_environment`; indexes on `(workspace_id, project_id)`, `(plan_id, state)`, `(case_id, plan_id)`, `(environment_id, workspace_id)`
- [ ] `V51__create_test_run_entities.sql` — `test_run`, `test_case_result`, `test_failure_summary`; indexes on `(plan_id, created_at DESC)`, `(case_id, run_id)`, `(run_id, status)`
- [ ] `V52__create_test_traceability.sql` — `test_case_req_link`; indexes on `(req_id)`, `(case_id)`, `(link_status)`
- [ ] Verify all migrations run on H2 (local) without error
- [ ] Produce Oracle variant if DDL diverges under `db/migration/oracle/`

### Phase 0 Definition of Done
- Package structure in place; no package leaks (policy/dto access via service boundary only)
- Flyway V50–V52 baseline applied; H2 schema matches design-model DDL
- Shared facades stubbed or confirmed; upstream blockers documented
- Ready for Phase 1 backend work

---

## Phase 1: Backend Read-Only MVP (Codex)

### Goal
Implement all read projections, services, and controllers for Catalog, Plan Detail, Case Detail, Run Detail, and Traceability views. No mutations, no ingestion yet.

### Tasks — Backend: Entities & Repositories

- [ ] Create JPA entities per data-model: `TestPlan`, `TestCase`, `TestCaseReqLink`, `TestRun`, `TestCaseResult`, `TestFailureSummary`, `TestEnvironment`, `TestingManagementChangeLogEntry`
- [ ] Map `TestCase.steps` and `TestCase.expectedResult` as `@Lob String` (markdown CLOB)
- [ ] Map `TestFailureSummary.failureExcerpt` as `@Lob String` with hard limit 4 KB (enforced at ingestion)
- [ ] Create repositories with representative query methods:
  - `TestPlanRepository.findByWorkspaceIdInAndState(workspaceIds, state, Pageable)`
  - `TestPlanRepository.findByWorkspaceIdAndProjectId(workspaceId, projectId)`
  - `TestCaseRepository.findByPlanIdAndState(planId, state, Pageable)`
  - `TestCaseRepository.findByPlanIdAndStateBetweenDates(planId, state, since, until)`
  - `TestCaseReqLinkRepository.findByReqId(reqId)` — for traceability
  - `TestCaseReqLinkRepository.findByLinkStatus(UNKNOWN_REQ, Pageable)` — nightly resolver
  - `TestRunRepository.findTop20ByPlanIdOrderByCreatedAtDesc(planId)`
  - `TestCaseResultRepository.findByRunId(runId)`
  - `TestFailureSummaryRepository.findByRunIdAndStatus(runId, FAILED)`
  - `TestEnvironmentRepository.findByWorkspaceId(workspaceId)`

### Tasks — Backend: Projections

- [ ] `CatalogSummaryProjection` — total plans, active cases, runs in last 7d, pass rate % last 7d, mean duration last 7d (REQ-TM-02)
- [ ] `CatalogRepoGridProjection` — plan rows grouped by project; per-plan case count, coverage LED, state, release target (REQ-TM-01)
- [ ] `PlanHeaderProjection` — identity, name, description, state, owner, release target, created/updated, owning Project (REQ-TM-11)
- [ ] `PlanCasesProjection` — case rows with type, priority, state, linked-REQ chips, last-run status, last-run timestamp (REQ-TM-12)
- [ ] `PlanCoverageProjection` — REQ-ID rows with case count and latest-run status per REQ (REQ-TM-13)
- [ ] `PlanRecentRunsProjection` — last 20 runs with ID, environment, trigger source, state, duration, pass/fail/skip counts, actor (REQ-TM-14)
- [ ] `CaseDetailProjection` — full case body (title, type, priority, state, owner, preconditions, steps, expected, linked-REQs, linked-incidents), last 20 run outcomes as sparkline + table (REQ-TM-20)
- [ ] `RunHeaderProjection` — run metadata (plan, environment, trigger, actor, duration, start/end, pass/fail/skip counts) (REQ-TM-30)
- [ ] `RunCaseResultsProjection` — per-case outcomes (PASS / FAIL / SKIP / ERROR) with failure excerpts capped at 4 KB (REQ-TM-30)
- [ ] `RunCoverageProjection` — union of REQ-IDs from all ACTIVE cases in the run (REQ-TM-36)
- [ ] `TraceabilityReqRowsProjection` — stories linked from active cases; batched `RequirementLookup` fan-out (REQ-TM-41)
- [ ] All projections honour workspace isolation at query level; per-projection 500ms timeout → `SectionResultDto(error=...)`

### Tasks — Backend: Read Services

- [ ] `CatalogService.loadAggregate(filters)` with parallel fan-out over catalog projections; no card failure blocks page (REQ-TM-01, REQ-TM-02, REQ-TM-03, REQ-TM-04)
- [ ] `CatalogService.load{Summary,Grid,Filters}` for per-card endpoints
- [ ] `PlanDetailService.loadAggregate(planId)` — parallel fan-out over plan projections; per-card isolation (REQ-TM-10)
- [ ] `PlanDetailService.load{Header,Cases,Coverage,RecentRuns}` for per-card endpoints
- [ ] `CaseDetailService.load(caseId)` — markdown rendering + REQ chip color coding per `RequirementLookup` (GREEN / AMBER / RED, REQ-TM-22)
- [ ] `RunDetailService.loadAggregate(runId)` — parallel fan-out over run projections (REQ-TM-30)
- [ ] `RunDetailService.load{Header,CaseResults,Coverage}` for per-card endpoints
- [ ] `TraceabilityService.loadAggregate(filters)` — coverage view per REQ, story-level rollup (REQ-TM-41, REQ-TM-42, REQ-TM-44)
- [ ] All read methods emit zero writes; safe to retry; timeout-safe

### Tasks — Backend: Controllers

- [ ] `TestingManagementController` (all read endpoints, per API guide):
  - GET `/catalog` — aggregate
  - GET `/catalog/summary`, `/catalog/grid` — per-card
  - GET `/plans/{planId}` — plan detail aggregate
  - GET `/plans/{planId}/header`, `/cases`, `/coverage`, `/recent-runs` — per-card
  - GET `/cases/{caseId}` — case detail
  - GET `/runs/{runId}` — run detail aggregate
  - GET `/runs/{runId}/header`, `/case-results`, `/coverage` — per-card
  - GET `/traceability` — aggregate
  - GET `/traceability/req-rows` — per-card
- [ ] `@Pattern` path validation on IDs
- [ ] Standard `ApiResponse<T>` envelope wrap via shared response advice
- [ ] Per-projection 500ms timeout returns `SectionResultDto(data=null, error=...)`

### Tasks — Backend: Tests

- [ ] `CatalogServiceTest`: aggregate happy + per-projection error isolation (each card fails independently)
- [ ] `PlanDetailServiceTest`: happy + 404 plan not found
- [ ] `CaseDetailServiceTest`: REQ chip color coding (GREEN / AMBER / RED) per `RequirementLookup` state
- [ ] `RunDetailServiceTest`: happy + failure excerpt truncation (4 KB hard limit)
- [ ] `TraceabilityServiceTest`: coverage computation per REQ (PASS / MIXED / FAIL / NOT_RUN)
- [ ] `TestingManagementControllerTest` (MockMvc): every endpoint happy path + status codes
- [ ] Repository tests against H2 for every entity + edge queries
- [ ] `FlywayMigrationIntegrationTest`: V50–V52 clean on H2; schema matches design-model

### Phase 1 Definition of Done
- All 13 read endpoints return correct envelopes + data
- Per-projection 500ms timeout degrades to card-level error only
- REQ-ID chips render with correct colors (GREEN / AMBER / RED / grey) via `RequirementLookup`
- Workspace isolation enforced at query level
- Markdown rendering safe (no raw HTML)
- Unit tests ≥80% line coverage on services + projections
- Flyway V50–V52 applied successfully

---

## Phase 2: Frontend Read-Only MVP (Codex)

### Goal
Build all read-path views: Catalog, Plan Detail, Case Detail, Run Detail, Traceability. Codex-driven, mock-backed initially.

### Tasks — Frontend: Types & Enums

- [ ] `src/features/testing-management/types/enums.ts` — PlanState, CaseState, CaseType, CasePriority, RunStatus, RunTrigger, CaseResultOutcome, CoverageStatus, LinkStatus, AiRowStatus
- [ ] `src/features/testing-management/types/catalog.ts` — CatalogSummary, CatalogPlanRow, CatalogFilter, CatalogAggregate (REQ-TM-01, REQ-TM-02)
- [ ] `src/features/testing-management/types/plan.ts` — PlanHeader, PlanCaseRow, PlanCoverageRow, PlanRunRow, PlanDetailAggregate (REQ-TM-10, REQ-TM-12, REQ-TM-13, REQ-TM-14)
- [ ] `src/features/testing-management/types/case.ts` — CaseDetail, CaseRunOutcome, CaseSparkline (REQ-TM-20)
- [ ] `src/features/testing-management/types/run.ts` — RunHeader, RunCaseResultRow, RunCoverageAggregate, RunDetailAggregate (REQ-TM-30, REQ-TM-36)
- [ ] `src/features/testing-management/types/traceability.ts` — TraceabilityReqRow, TraceabilityCoverageStatus, TraceabilityAggregate (REQ-TM-41)

### Tasks — Frontend: Mock Data

- [ ] `src/features/testing-management/mock/catalog.mock.ts` — ~8 plans across 3 projects; mix of DRAFT / ACTIVE / ARCHIVED; coverage LED distribution (green / amber / red / grey)
- [ ] `src/features/testing-management/mock/planDetail.mock.ts` — full detail for 3 canonical plans (healthy, mixed, no-runs-yet)
- [ ] `src/features/testing-management/mock/caseDetail.mock.ts` — 4 cases with markdown steps + preconditions; REQ chips in all 4 colors (GREEN / AMBER / RED / grey)
- [ ] `src/features/testing-management/mock/runDetail.mock.ts` — 3 runs (PASSED, FAILED with excerpt, IN_PROGRESS); failure excerpts pre-redacted for secrets
- [ ] `src/features/testing-management/mock/traceability.mock.ts` — 12 REQs with coverage status distribution (PASS / MIXED / FAIL / NOT_RUN)
- [ ] Mock command layer: error injection for rate limits, unknown REQs, AI unavailable (used in Phase 5)

### Tasks — Frontend: Reusable Components & Composables

- [ ] `src/features/testing-management/components/CoverageIndicator.vue` — LED + percentage; color-blind safe (shape + text)
- [ ] `src/features/testing-management/components/ReqChip.vue` — linked REQ-ID with color (GREEN / AMBER / RED / grey); hover tooltip
- [ ] `src/features/testing-management/components/CaseOutcomeSparkline.vue` — sparkline chart of case outcomes over last 20 runs
- [ ] `src/features/testing-management/components/RunStatusBadge.vue` — PASSED / FAILED / IN_PROGRESS / ABORTED with icon
- [ ] `src/features/testing-management/components/EmptyStatePlaceholder.vue` — contextual empty states (no plans, no cases, no runs, AI drafts disabled, AI draft failed)
- [ ] `src/features/testing-management/composables/useTestingManagement.ts` — API client, mock toggle, error handling
- [ ] `src/features/testing-management/stores/testingStore.ts` — Pinia store for plan/case/run state; aggregate card hydration

### Tasks — Frontend: Views

- [ ] `src/features/testing-management/views/CatalogView.vue` — 6 cards: Header, Summary, Grid (filterable + searchable per REQ-TM-03, REQ-TM-04), Filters sidebar; deep-link to Plan Detail (REQ-TM-05)
- [ ] `src/features/testing-management/views/PlanDetailView.vue` — 6 cards: Header, Cases, Coverage, Recent Runs, AI Draft Inbox (Phase 5), AI Insights (Phase 5); per-card error + retry
- [ ] `src/features/testing-management/views/CaseDetailView.vue` — title, type, priority, state, owner, preconditions, steps, expected, REQ chips, incident chips (Phase 3), sparkline + run history table; edit CTA (Phase 3)
- [ ] `src/features/testing-management/views/RunDetailView.vue` — metadata, case-result table with status + failure excerpt (truncated), coverage rollup; deep-link to Requirement (REQ-TM-17, REQ-TM-31)
- [ ] `src/features/testing-management/views/TraceabilityView.vue` — Catalog by REQ-ID; coverage status per REQ; linked cases per REQ; inverse view from Requirement story (REQ-TM-41)

### Tasks — Frontend: Routing & Navigation

- [ ] Add routes: `/testing-management`, `/testing-management/plans/{planId}`, `/testing-management/cases/{caseId}`, `/testing-management/runs/{runId}`, `/testing-management/traceability`
- [ ] Update shell breadcrumbs on navigation via `shellUiStore.setBreadcrumbs`
- [ ] Confirm `requireWorkspaceMember` guard works for each view

### Tasks — Frontend: Mock Validation

- [ ] Smoke: browse `/testing-management` — all 6 catalog cards render with mock data
- [ ] Smoke: navigate to plan detail — 6 cards hydrated
- [ ] Smoke: click case from plan → case detail view; REQ chips rendered in 4 colors
- [ ] Smoke: click run from plan → run detail; failure excerpt visible + truncated
- [ ] Smoke: browse traceability view — REQ rows with coverage status

### Phase 2 Definition of Done
- All 5 views render correctly with mock data
- Every deep-link works (Catalog → Plan → Case → Run → Requirement)
- REQ chips correctly color-coded
- Empty states match Figma; error boundaries per card visible
- `mock` toggle in dev tools works (switch between mock + backend coming Phase 1B)
- Accessibility baseline (keyboard nav, ARIA roles, color-blind safety)

---

## Phase 3: CRUD Mutations & Forms (Codex)

### Goal
Implement Plan, Case, and Environment CRUD; frontend forms; edit/create flows.

### Tasks — Backend: DTOs & Commands

- [ ] `CreatePlanRequest` — name, description, releaseTarget (REQ-TM-01)
- [ ] `UpdatePlanRequest` — name, description, state, releaseTarget (REQ-TM-10)
- [ ] `CreateCaseRequest` — title, type, priority, preconditions, steps, expected, linkedReqIds (REQ-TM-12)
- [ ] `UpdateCaseRequest` — all above + record revision entry (REQ-TM-25)
- [ ] `CreateEnvironmentRequest` — name, description, kind, url (REQ-TM-60)
- [ ] `UpdateEnvironmentRequest` — all above (REQ-TM-60)
- [ ] Command response records include created/updated entity state + audit log entry id

### Tasks — Backend: Command Services

- [ ] `PlanCommandService.create(request, principal)`:
  - `CodeBuildAccessGuard.requireRole(projectId, principal, QA_LEAD | TECH_LEAD | PM)` → `TM_ROLE_REQUIRED` (REQ-TM-83)
  - Validate release target format
  - Persist `TestPlan` with state=ACTIVE
  - Publish `LineageEvent` + `AuditLogEntry`
- [ ] `PlanCommandService.update(planId, request, principal)`:
  - Same role gate; state transition validation (DRAFT/ACTIVE/ARCHIVED)
  - Update plan row
  - Publish changelog entry
- [ ] `PlanCommandService.archive(planId, principal)`:
  - Role gate; transition to ARCHIVED
  - Publish `LineageEvent`
- [ ] `CaseCommandService.create(planId, request, principal)`:
  - Role gate (QA_LEAD | TECH_LEAD)
  - Validate case preconditions + steps + expected (non-empty, markdown-safe)
  - For each linkedReqId: validate via `RequirementLookup`; if UNKNOWN → persist link with UNKNOWN_REQ status (REQ-TM-43)
  - Persist `TestCase` + `TestCaseReqLink` rows
  - Publish `CodeBuildChangeLogEntry` + `LineageEvent` (REQ-TM-84)
- [ ] `CaseCommandService.update(caseId, request, principal)`:
  - Role gate (QA_LEAD | TECH_LEAD)
  - Capture revision entry (actor, timestamp, field diffs) in side drawer (REQ-TM-25)
  - On linkedReqIds change: invalidate downstream Plan Coverage + Requirement inverse view within 30s P95 (REQ-TM-26)
  - Publish changelog + lineage
- [ ] `CaseCommandService.deprecate(caseId, reason, principal)`:
  - Role gate; transition to DEPRECATED with reason
  - Deprecated cases remain visible in history but excluded from coverage (REQ-TM-24)
- [ ] `EnvironmentCommandService.create(workspaceId, request, principal)`:
  - Role gate (QA_LEAD | TECH_LEAD | PM)
  - Validate name unique within workspace (REQ-TM-60)
  - Persist `TestEnvironment` with archived=false
  - Publish changelog
- [ ] `EnvironmentCommandService.update(envId, request, principal)`:
  - Role gate; update row
- [ ] `EnvironmentCommandService.archive(envId, principal)`:
  - Mark archived=true; block new runs (REQ-TM-62)
- [ ] All command services enforce workspace isolation → `TM_WORKSPACE_FORBIDDEN` on cross-workspace ids
- [ ] All mutations publish `CodeBuildChangeLogEntry` + `LineageEvent` (REQ-TM-84)

### Tasks — Backend: Mutation Controllers

- [ ] `TestingManagementController`:
  - POST `/plans` → create
  - PUT `/plans/{planId}` → update
  - POST `/plans/{planId}/archive` → archive
  - POST `/plans/{planId}/cases` → create case
  - PUT `/cases/{caseId}` → update case
  - POST `/cases/{caseId}/deprecate` → deprecate
  - POST `/environments` → create environment
  - PUT `/environments/{envId}` → update
  - POST `/environments/{envId}/archive` → archive
- [ ] `@Valid` on request bodies; validation error → 400 with field errors
- [ ] Successful mutation → 200 with updated entity state
- [ ] Workspace isolation check on every write

### Tasks — Backend: Tests

- [ ] `PlanCommandServiceTest`: happy create/update/archive + role denied + cross-workspace blocked
- [ ] `CaseCommandServiceTest`: happy create/update/deprecate + unknown REQ → persists with UNKNOWN_REQ status + revision tracking
- [ ] `EnvironmentCommandServiceTest`: happy create/update/archive + name uniqueness + archived blocks new runs
- [ ] `TestingManagementControllerTest`: mutation endpoints happy + validation errors + status codes
- [ ] Workspace isolation integration test: cross-workspace ID → 403

### Tasks — Frontend: Forms & Mutations

- [ ] `src/features/testing-management/components/PlanForm.vue` — name, description, releaseTarget input fields; mode=create|edit
- [ ] `src/features/testing-management/components/CaseForm.vue` — title, type, priority dropdowns; preconditions/steps/expected markdown editors; ReqChip multiselect for linkedReqIds; revision history side drawer on edit (REQ-TM-25)
- [ ] `src/features/testing-management/components/EnvironmentForm.vue` — name, description, kind dropdown, url input
- [ ] Mutation API calls via `useTestingManagement()` composable; form validation (non-empty, markdown safety)
- [ ] On successful mutation: refetch aggregate + show success toast
- [ ] Error handling: role denied, unknown REQ, conflict → show error toast + retain form state
- [ ] Revision history drawer: list of (actor, timestamp, field-diffs) per case edit (REQ-TM-25)

### Tasks — Frontend: Views (Mutation Paths)

- [ ] `PlanDetailView`: Add Plan button → modal with PlanForm
- [ ] `PlanDetailView.Cases` card: Add Case button → modal with CaseForm
- [ ] `CaseDetailView`: Edit button → full-page form with revision history drawer
- [ ] New `EnvironmentsView` (accessible from settings or Plan Detail): list + create/edit/archive UI

### Tasks — Frontend: Validation & UX

- [ ] Markdown preview for steps + preconditions + expected (read-only on detail)
- [ ] REQ multiselect: autocomplete from `RequirementLookup`; unknown IDs show AMBER chip with warning
- [ ] on case linkedReqIds change: background refresh of Plan Coverage card (REQ-TM-26)
- [ ] Optimistic updates: show pending state during submission

### Phase 3 Definition of Done
- Plan CRUD happy path works (create → read → update → archive)
- Case CRUD happy path works (create → read → update → deprecate)
- Environment CRUD happy path works
- Revision history tracked + visible
- Unknown REQ-IDs persist and resolve asynchronously
- All mutations publish LineageEvent + AuditLogEntry
- Workspace isolation enforced (cross-workspace ID → 403)
- Unit tests ≥80% coverage on command services
- Frontend forms validated + error-handled
- Optimistic updates + loading states visible

---

## Phase 4: Test-Run Ingestion Pipeline (Codex)

### Goal
Implement webhook receiver, JUnit/TestNG/Playwright/Cypress parsers, outbox-backed dispatcher, result persistence, secret redaction.

### Tasks — Backend: Entities & Repositories (Additional)

- [ ] `TestRunIngestionOutbox` — `(deliveryId UNIQUE, sourceType, rawPayload, receivedAt, status=PENDING)` (REQ-TM-35)
- [ ] `TestRunIngestionSourceEnum` — MANUAL_UPLOAD, CI_WEBHOOK
- [ ] Repositories:
  - `TestRunIngestionOutboxRepository.findByDeliveryId(id)` — idempotency
  - `TestRunIngestionOutboxRepository.lockAndClaimOldest(batch)` — outbox worker drain

### Tasks — Backend: Ingestion Pipeline

- [ ] `TestRunIngestionController.POST /webhooks/ingest`:
  - Accept file upload or webhook payload per sourceType
  - On CI webhook: verify HMAC signature (per sourceType: GitHub, GitLab, Jenkins) → `TM_WEBHOOK_SIGNATURE_INVALID` (401) (REQ-TM-85)
  - Persist `TestRunIngestionOutbox` row with `deliveryId UNIQUE` (idempotency) → 202 Accepted
- [ ] `ManualRunUploadController.POST /runs/{planId}/upload`:
  - Accept JUnit/TestNG/Playwright/Cypress file upload by QA Lead
  - Validate format (magic header or root element)
  - Persist outbox row → 202 (REQ-TM-35)
- [ ] `TestRunIngestionOutboxWorker` — `@Scheduled` drain:
  - Claim oldest PENDING rows (row-level lock or CAS)
  - Mark IN_PROGRESS
  - Dispatch to parser
  - Mark DONE or FAILED with error payload
  - At-least-once + `deliveryId` uniqueness = effectively-once
- [ ] Parsers (one per format, REQ-TM-34):
  - `JunitXmlParser` — reads `testcase` / `testsuites` elements; maps to `TestCaseResult` (name, classname, status, time, failure/error message)
  - `TestngXmlParser` — reads TestNG XML structure
  - `PlaywrightJsonParser` — reads Playwright JSON structure
  - `CypressJsonParser` — reads Cypress Mochawesome JSON structure
  - All parsers return a list of `(caseTitle, classname?, status, duration, failureExcerpt?)`
- [ ] `TestRunPayloadNormalizer` — unifies all parser outputs into `IngestedRunPayload`
- [ ] `TestRunIngestionDispatcher`:
  - Parse payload → determine format
  - For each case result:
    - Match case to `TestCase` by name (or name + classname if provided)
    - If no match: log warning, skip (do not create fake case) (REQ-TM-74)
    - If match: create `TestCaseResult` with status, duration, failureExcerpt
    - Apply `LogRedactor` to failureExcerpt before persist (REQ-TM-80)
    - Truncate failureExcerpt to 4 KB (REQ-TM-30, REQ-TM-34)
  - Create `TestRun` row with metadata (planId, environment, trigger source, actor, duration, pass/fail/skip counts)
  - Persist all changes in a transaction (atomicity — REQ-TM-74)
  - If parse fails: persist run in state=INGEST_FAILED with error message (first 2 KB); never partial-write results (REQ-TM-74)
  - Update Plan's `lastSyncedAt`
  - Emit `CodeBuildChangeLogEntry(RUN_INGESTED)`
- [ ] `LogRedactor` — applied at ingestion time (REQ-TM-80):
  - Pattern: `AKIA[0-9A-Z]{16}` → `***REDACTED:AWS_KEY***`
  - Pattern: `ghp_[A-Za-z0-9]{36}` / `gho_[A-Za-z0-9]{36}` / `ghs_[A-Za-z0-9]{36}` → `***REDACTED:GH_TOKEN***`
  - Pattern: `(?i)Bearer\s+[A-Za-z0-9._\-]{20,}` → `Bearer ***REDACTED:BEARER***`
- [ ] `RunInvocationValidator`:
  - Reject re-ingestion of same `externalRunId` unless `force=true` by admin → 409 Conflict (REQ-TM-33)
  - On force: mark prior run as superseded + audit

### Tasks — Backend: Commands (Run Creation)

- [ ] `TestRunCommandService.ingestRun(planId, payload, principal)`:
  - Resolve plan → workspace/project
  - Role gate (QA_LEAD can upload; CI webhook verified by signature)
  - Dispatch ingestion pipeline
  - Return `IngestedRunSummary` (run id, case count, pass/fail/skip counts, failures)
- [ ] `TestRunCommandService.reuploadRun(runId, force, principal)`:
  - Allow reupload if prior run = INGEST_FAILED
  - If force=true: mark prior as superseded, create new run (admin only)

### Tasks — Backend: Integration

- [ ] `CiWebhookSignatureVerifier` — HMAC-SHA-256 verification per CI system (GitHub, GitLab, Jenkins) (REQ-TM-85)
- [ ] `TestCaseNameMatcher` — fuzzy match of ingested case name to persisted `TestCase` by name ± classname
- [ ] Confirm `LogRedactor` is idempotent (redaction doesn't re-redact if already applied)

### Tasks — Backend: Tests

- [ ] `TestRunIngestionControllerTest` (MockMvc):
  - Manual upload happy + validation errors
  - Webhook signature valid → 202 + outbox row
  - Webhook signature invalid → 401
  - Duplicate `externalRunId` → 409 unless force=true
- [ ] `JunitXmlParserTest`: canned JUnit XML → correct `IngestedRunPayload`
- [ ] `TestngXmlParserTest`: canned TestNG XML
- [ ] `PlaywrightJsonParserTest`: canned Playwright JSON
- [ ] `CypressJsonParserTest`: canned Cypress JSON
- [ ] `TestRunIngestionDispatcherTest`:
  - Happy path: parse + match cases + create run + results
  - Case name not found → warning logged, case skipped
  - Parse failure → run persisted in INGEST_FAILED state with error; no partial results
  - Secrets redacted in failure excerpts + idempotent redaction
  - Failure excerpt truncated to 4 KB
- [ ] `RunInvocationValidatorTest`: duplicate externalRunId → 409; force=true → supersedes
- [ ] `LogRedactorTest`: each secret pattern redacted correctly; false positives handled; idempotency
- [ ] `CiWebhookSignatureVerifierTest`: valid signature passes; flipped byte rejects with constant-time compare
- [ ] `TestCaseNameMatcherTest`: exact match + fuzzy match + no match

### Tasks — Frontend: Run Upload

- [ ] `src/features/testing-management/components/RunUploadForm.vue` — file input (accepts `.xml`, `.json`), environment dropdown, manual/webhook option
- [ ] `src/features/testing-management/views/RunUploadView.vue` — form + progress indicator + success/error summary (case count, pass/fail/skip)
- [ ] Modal triggered from Plan Detail "Upload Results" button
- [ ] On success: refetch Plan Detail Recent Runs card

### Phase 4 Definition of Done
- Webhook receiver signature-verified, idempotent, outbox-backed
- All 4 parsers (JUnit, TestNG, Playwright, Cypress) working
- Cases matched by name (with fuzzy matching fallback)
- Secrets redacted from failure excerpts; no raw tokens persisted
- Failure excerpts truncated to 4 KB max
- Run ingestion atomic — no partial results on parse failure
- Duplicate `externalRunId` rejected (409) unless force=true
- Manual upload works from frontend
- Unit tests ≥80% coverage on ingestion + parsers
- Change log entry per run ingested
- Workspace isolation enforced

---

## Phase 5: AI Test-Case Drafting (Codex)

### Goal
Implement AI test-case-drafter skill invocation, DRAFT ↔ ACTIVE ↔ DEPRECATED state machine, approval flows, autonomy gating, role-based approval, stale-draft re-draft.

### Tasks — Backend: Entities & Repositories (Additional)

- [ ] `TestCase.origin` — enum: MANUAL, AI_DRAFT
- [ ] `TestCase.state` — extend to: ACTIVE, DRAFT (pre-approval), DEPRECATED (rejected or archived)
- [ ] `AiDraftMetadata` — (sourceReqId, skillVersion, draftTimestamp, rejectionReason?)

### Tasks — Backend: Command Services

- [ ] `AiTestCaseDraftCommandService.draft(planId, reqId, principal)`:
  - `CodeBuildAccessGuard.requireRole` (QA_LEAD | TECH_LEAD) → `TM_ROLE_REQUIRED` (REQ-TM-50, REQ-TM-53)
  - `CodeBuildAiAutonomyPolicy.requireAtLeast(workspaceId, OBSERVATION)` — OBSERVATION allows view; SUPERVISED/AUTONOMOUS allow approval (REQ-TM-56)
  - Validate reqId via `RequirementLookup` → if unknown, return `TM_UNKNOWN_REQ`
  - Check daily rate limit: ≤20 drafts per REQ per day per workspace → `TM_DRAFT_RATE_LIMIT` (REQ-TM-58)
  - Invoke `AiSkillClient.generateTestCases(reqId, reqText, priorCaseTitles)` — return candidate cases list (title, preconditions, steps, expected, type, priority)
  - For each candidate: persist `TestCase` with state=DRAFT, origin=AI_DRAFT, keyed by (reqId, skillVersion, planId) (REQ-TM-51)
  - Cite source REQ text excerpt in draft metadata (REQ-TM-57)
  - Publish `SkillInvocationEvent` (invoker, autonomy, version, latency, outcome) (REQ-TM-82)
  - Return list of draft cases with source excerpt visible for human review (REQ-TM-57)
- [ ] `AiTestCaseDraftCommandService.approve(draftCaseId, principal)`:
  - `CodeBuildAccessGuard.requireRole` (QA_LEAD | TECH_LEAD) → role-based (REQ-TM-53)
  - `CodeBuildAiAutonomyPolicy.requireAtLeast(workspaceId, SUPERVISED)` — OBSERVATION blocks approval (REQ-TM-56)
  - Transition state: DRAFT → ACTIVE
  - Record `AuditLogEntry` (approver, timestamp)
  - Publish `CodeBuildChangeLogEntry(AI_DRAFT_APPROVED)` (REQ-TM-53)
  - Draft case now appears in public Cases card + counts toward coverage
- [ ] `AiTestCaseDraftCommandService.reject(draftCaseId, reason, principal)`:
  - Role gate (QA_LEAD | TECH_LEAD)
  - Transition state: DRAFT → DEPRECATED with rejectionReason
  - Publish `CodeBuildChangeLogEntry(AI_DRAFT_REJECTED)` (REQ-TM-54)
  - Rejected draft remains visible in history but excluded from coverage
- [ ] `AiTestCaseDraftCommandService.editBeforeApprove(draftCaseId, updates, principal)`:
  - Allow edits while in DRAFT state (title, steps, expected, type, priority) (REQ-TM-53)
  - Record revision entry
  - Publish changelog
- [ ] `AiTestCaseDraftCommandService.redraft(draftCaseId, principal)`:
  - On stale draft (skill version advanced): allow admin to re-invoke drafter with current skill version (REQ-TM-55, REQ-TM-72)
  - Mark prior draft stale (`invalidatedAt` = now)
  - Invoke drafter again; persist new draft keyed by new skillVersion
  - Publish changelog

### Tasks — Backend: Stale-Draft Resolution

- [ ] `AiDraftVersionResolver` — `@Scheduled` job:
  - On skill version bump: detect drafts with old skillVersion
  - Mark with STALE badge (REQ-TM-55)
  - Surface re-draft CTA in UI (REQ-TM-72)
- [ ] `UnresolvedReqLinkResolver` — `@Scheduled` job (hourly):
  - Find all `TestCaseReqLink` with linkStatus=UNKNOWN_REQ
  - Re-invoke `RequirementLookup` for each reqId
  - Update link status if now resolved (REQ-TM-43)

### Tasks — Backend: Controllers (AI Endpoints)

- [ ] `TestingManagementController`:
  - POST `/ai/test-cases/draft` — request: (planId, reqId), response: list of draft cases with source excerpt (REQ-TM-50)
  - POST `/cases/{draftCaseId}/ai/approve` → 200 with case now ACTIVE (REQ-TM-53)
  - POST `/cases/{draftCaseId}/ai/reject` → request: (reason) → 200 (REQ-TM-54)
  - POST `/cases/{draftCaseId}/ai/redraft` → 200 with new DRAFT cases (REQ-TM-72)

### Tasks — Backend: Integration

- [ ] `AiSkillClient.generateTestCases(reqId, reqText, priorCaseTitles, workspace)`:
  - Invoke test-case-drafter skill
  - Secure prompt: REQ text + prior case titles only; no secrets, no environment credentials (REQ-TM-81)
  - 60s timeout; surfaces 5xx as `TM_AI_UNAVAILABLE` (REQ-TM-50)
  - Return skill version in response
- [ ] Autonomy gating: DISABLED → 403; OBSERVATION → draft visible but Approve button disabled (read-only preview); SUPERVISED/AUTONOMOUS → full flow (REQ-TM-56)

### Tasks — Backend: Tests

- [ ] `AiTestCaseDraftCommandServiceTest`:
  - Happy draft + approve + reject + redraft flow
  - Role denied → 403
  - Autonomy insufficient (DISABLED or OBSERVATION) → appropriate error
  - Stale draft detected → STALE badge + re-draft CTA
  - Daily rate limit hit → `TM_DRAFT_RATE_LIMIT`
  - Unknown REQ → `TM_UNKNOWN_REQ`
- [ ] `AiSkillClientTest`: skill invocation happy + 5xx timeout + autonomy gating
- [ ] `AiDraftVersionResolverTest`: skill version bump → detect stale, mark, surface CTA
- [ ] `UnresolvedReqLinkResolverTest`: UNKNOWN_REQ → re-resolve on hourly job, update link status
- [ ] `TestingManagementControllerTest`: all AI endpoints (draft, approve, reject, redraft) + status codes

### Tasks — Frontend: AI Draft Flows

- [ ] `src/features/testing-management/components/AiDraftInboxCard.vue` — list of DRAFT cases with source REQ excerpt (REQ-TM-15); approve/reject/edit inline buttons
- [ ] `src/features/testing-management/components/AiDraftForm.vue` — trigger draft from Plan Detail; form shows REQ selection + rate-limit quota (REQ-TM-58)
- [ ] `src/features/testing-management/components/StaleDraftBadge.vue` — marks old skill version drafts; "re-draft" button (REQ-TM-55, REQ-TM-72)
- [ ] On approve: move case to public Cases card; show success toast
- [ ] On reject: move to deprecated view; show success toast
- [ ] Edit before approve: inline form; record revision
- [ ] Rate limit: show remaining quota (e.g. "18 / 20 drafts used today") (REQ-TM-58)
- [ ] Autonomy gating in UI: OBSERVATION → Approve button disabled with tooltip; DISABLED → entire draft endpoint disabled (REQ-TM-56)

### Tasks — Frontend: AI Insights Card

- [ ] `src/features/testing-management/components/AiInsightsCard.vue` — AI narrative summary (REQ-TM-16): "what changed in last 7d, which cases trending red, which REQs lost coverage"
- [ ] Read from `AiWorkspaceSummary` backend aggregate (if available) or mock for Phase 5

### Phase 5 Definition of Done
- Draft endpoint works; returns candidate cases with source REQ text (REQ-TM-57)
- Approve / Reject / Edit / Redraft flows work
- Autonomy gating enforced: DISABLED blocks endpoint; OBSERVATION allows view, not approval; SUPERVISED/AUTONOMOUS allow approval
- Role-based approval (QA_LEAD | TECH_LEAD only) enforced
- Daily rate limit (20 drafts/REQ/day) enforced; UI shows quota (REQ-TM-58)
- Stale-draft badge visible; re-draft button works
- Unresolved REQ links re-resolve hourly
- All mutations publish LineageEvent + AuditLogEntry + SkillInvocationEvent
- Unit tests ≥80% coverage on AI command service
- Frontend: draft inbox displays; approve/reject buttons work
- Autonomy + role UI controls visible / disabled correctly

---

## Phase 6: Traceability & Cross-Slice Integration (Codex)

### Goal
Implement Traceability view, Requirement-slice "Tests" tab integration, per-card error handling, empty states, accessibility, performance tuning, final polish.

### Tasks — Backend: Traceability Aggregate

- [ ] `TraceabilityService.loadAggregate(filters)`:
  - Parallel fan-out over 4 projections: ReqRows, UnresolvedReqs, Summary
  - Per-projection 500ms timeout → `SectionResultDto(error=...)`
  - Never page-level failure
  - Coverage computation (REQ-TM-44): PASS if ≥1 ACTIVE case passed in last 7d; AMBER if oldest run >7d or mixed; RED if latest FAILED; GREY if no runs
- [ ] `TraceabilityStoryRowsProjection` — for each req visible to workspace: linked ACTIVE cases, plans, most-recent case run status (REQ-TM-41)
- [ ] `TraceabilityUnresolvedReqsProjection` — test cases with linkedReqIds that could not be resolved (link_status=UNKNOWN_REQ); surfaced in UI (REQ-TM-73)
- [ ] `TraceabilitySummaryProjection` — count of PASS / MIXED / FAIL / NOT_RUN buckets (REQ-TM-44)

### Tasks — Backend: Cross-Slice Facade

- [ ] `TestCoverageFacade` — cross-slice read facade exposed to Requirement slice:
  - `coverageByReq(reqId, workspaceId)` → list of test cases + their recent run status (REQ-TM-42)
  - `coverageByStory(storyId, projectId)` → same (REQ-TM-42)
  - Used by Requirement slice "Tests" tab integration
- [ ] Confirm Requirement slice integrates "Tests" tab via this facade (downstream PR, not in Testing Management)

### Tasks — Backend: Error Handling & Empty States

- [ ] Per-card error state + Retry button for every projection (REQ-TM-70)
- [ ] Empty-state definitions (REQ-TM-71):
  - "No plans yet in this workspace" → Catalog view, no plans found
  - "No cases in this plan" → Plan Detail Cases card
  - "No runs yet for this plan" → Plan Detail Recent Runs card
  - "No AI drafts pending" → Plan Detail AI Draft Inbox card, or "AI drafts disabled" (DISABLED autonomy), or "AI draft generation failed"
  - "No REQs with test coverage" → Traceability view
  - "Unresolved REQ links found" → Plan Detail or Traceability, link_status=UNKNOWN_REQ (REQ-TM-73)
- [ ] Unverified REQ chips (REQ lookup failed) render GREY + tooltip "Unable to verify requirement" (REQ-TM-73)
- [ ] `<SectionResultError>` component — per-card error handling + retry logic

### Tasks — Backend: Validation & Constraints

- [ ] Run ingestion rejects unknown environment name; auto-creates with kind=OTHER and flags in platform diagnostics (REQ-TM-61)
- [ ] Test case validation: title non-empty, type/priority in enum, steps non-empty markdown
- [ ] Plan validation: name non-empty, releaseTarget format, owning project exists

### Tasks — Backend: Audit & Governance

- [ ] Verify all write paths (plan/case/run CRUD, AI approvals, ingestion) emit:
  - `CodeBuildChangeLogEntry` (what, when, by whom, entity_id, entity_type)
  - `LineageEvent` (Testing Management as authoring domain) (REQ-TM-84)
  - `AuditLogEntry` (per shared audit infra)
- [ ] Verify AI operations emit `SkillInvocationEvent` (autonomy level, skill version, latency, outcome)

### Tasks — Frontend: Traceability View

- [ ] `src/features/testing-management/views/TraceabilityView.vue`:
  - 4 cards: Summary, ReqRows, UnresolvedReqs, Export/Report CTA (V2)
  - Summary card shows pass/mixed/fail/grey bucket counts
  - ReqRows table: REQ-ID, coverage status (LED), case count, plan count, last-run status
  - Unresolved REQs table: candidate REQ-IDs that failed lookup; retry button re-runs resolver (REQ-TM-73)
  - Deep-link to Requirement story detail (REQ-TM-17); deep-link to Requirement search for unknown IDs
- [ ] Empty state: "No REQs with test coverage in this workspace"

### Tasks — Frontend: Cross-Slice Integration

- [ ] Requirement slice integration (downstream PR in Requirement slice):
  - Story Detail page: add "Tests" tab
  - Call `GET /api/v1/testing-management/traceability/req/{reqId}` (facade endpoint)
  - Render test cases + their recent run status
  - Show coverage status (LED) for this REQ
  - Deep-link to Plan Detail + Case Detail

### Tasks — Frontend: Accessibility & i18n

- [ ] All cards meet WCAG 2.1 AA:
  - Keyboard navigation (Tab / Shift+Tab / Enter / Space)
  - ARIA roles + labels on interactive elements
  - Coverage LEDs color-blind safe (shape + text + pattern, not color alone) (REQ-TM-90)
  - Focus visible on buttons / links
  - Alt text on icons
- [ ] All text keys in shared i18n token set; no hard-coded English (REQ-TM-91)
  - `tm.catalog.title`, `tm.catalog.summary.totalPlans`, `tm.plan.detail.header.title`, `tm.case.detail.linkedReqs`, `tm.run.detail.failedCases`, `tm.traceability.coverage.pass`, `tm.ai.draft.inbox.title`, `tm.ai.draft.rate.remaining`, etc.
- [ ] RTL layout considerations (if supported by shell)

### Tasks — Frontend: Performance Tuning

- [ ] Catalog P95 load ≤1200ms (including summary + grid + filters cards) (REQ-TM-92)
- [ ] Plan Detail P95 ≤1500ms (6 cards in parallel, 500ms per projection timeout)
- [ ] Run Detail P95 ≤1500ms (6 cards)
- [ ] No card >300 KB gzipped on critical path (REQ-TM-93)
- [ ] Run failure-excerpt rendering: virtualize if >100 KB (REQ-TM-93)
- [ ] Sparkline chart: canvas-based or SVG, not DOM nodes per data point

### Tasks — Frontend: Validation & UX Polish

- [ ] Form validation:
  - Plan name: non-empty, ≤255 chars
  - Case title: non-empty, ≤255 chars
  - Case steps: non-empty, ≤10 KB markdown
  - Environment name: non-empty, unique within workspace
  - REQ multiselect: warn if unknown REQ selected
- [ ] Error toast styling: respect error / warning / info levels
- [ ] Success toast on mutation: "Plan created", "Case updated", "Run uploaded", etc.
- [ ] Loading skeleton on paginated tables
- [ ] Optimistic updates: show pending state during submission; revert on failure
- [ ] Confirmation dialogs: "Archive this plan?" (destructive state transitions)

### Tasks — Frontend: Deep-Link & Navigation

- [ ] Catalog → Plan Detail: `/testing-management/plans/{planId}`
- [ ] Plan Detail → Case Detail: `/testing-management/cases/{caseId}`
- [ ] Plan Detail → Run Detail: `/testing-management/runs/{runId}`
- [ ] Traceability → Requirement Story: `/requirement/stories/{storyId}` (external)
- [ ] Traceability → Requirement Search (for unknown REQ-IDs): `/requirement/search?q={reqId}`
- [ ] All external links: `rel="noopener noreferrer"`, CSP headers respected

### Tasks — Backend: Final Tests

- [ ] `TraceabilityServiceTest`: all 4 projections happy; coverage computation (PASS / MIXED / FAIL / GREY) correct
- [ ] `TestCoverageFacadeTest`: Requirement-slice integration facade working
- [ ] `TestingManagementControllerTest`: all error paths + empty states
- [ ] `FlywayMigrationIntegrationTest`: V50–V57 run on H2; schema matches data-model + foreign keys enforced
- [ ] End-to-end smoke test:
  - Create plan + cases + link REQs
  - Upload run (JUnit) → cases matched + results persisted + secrets redacted
  - Approve AI drafts → appear in coverage
  - View Traceability → coverage status (LED) correct
  - Traceability REQ → click to Requirement story "Tests" tab (downstream verification)

### Tasks — Frontend: Final Tests

- [ ] Catalog view: filtering + searching working
- [ ] Plan Detail: 6 cards hydrated; deep-links working
- [ ] Case Detail: REQ chips correct colors; edit form + revisions
- [ ] Run Detail: failure excerpts truncated; secrets redacted; "Open Incident" link (Phase 3 spike)
- [ ] Traceability: coverage status correct; unresolved REQ links visible + retry
- [ ] AI Draft flows: draft → approve/reject → visible in coverage
- [ ] Accessibility audit: keyboard nav, ARIA roles, color-blind safety
- [ ] Performance: Catalog <1200ms P95, Plan <1500ms, Run <1500ms
- [ ] i18n: all labels via token keys
- [ ] CSP + external link safety verified

### Phase 6 Definition of Done
- Traceability aggregate endpoint works; per-projection timeout + error isolation
- Requirement-slice "Tests" tab can integrate via `TestCoverageFacade` (downstreamPR)
- Per-card error + Retry visible on all views
- Empty states match Figma per REQ-TM-71
- Unresolved REQ links visible; re-resolver runs hourly
- All error paths tested + handled
- Audit log + LineageEvent emitted for every write
- Accessibility: WCAG 2.1 AA on all views
- i18n: no hard-coded English; all tokens in shared catalog
- Performance: P95 load times within budget
- CSP + external link safety verified
- All deep-links working end-to-end
- Smoke test: create plan → upload run → approve AI draft → view traceability → verify coverage

---

## Definition of Done (Per Phase)

### Phase 0: Scaffolding
- [x] Package structure created; no cross-package leaks
- [x] Flyway V50–V52 baseline applied on H2
- [x] Shared facades stubbed or confirmed available

### Phase 1: Backend MVP
- [x] All 13 read endpoints return correct envelopes
- [x] Per-projection 500ms timeout degrades to card error
- [x] REQ chips GREEN / AMBER / RED / grey per lookup
- [x] Workspace isolation at query level
- [x] Unit tests ≥80% coverage

### Phase 2: Frontend MVP
- [x] All 5 views render with mock data
- [x] Deep-links work (Catalog → Plan → Case → Run)
- [x] Accessibility baseline (keyboard, ARIA, color-blind)
- [x] Mock toggle works

### Phase 3: CRUD
- [x] Plan / Case / Environment CRUD happy paths
- [x] Revision history tracked + visible
- [x] Unknown REQ-IDs persist; hourly resolver runs
- [x] LineageEvent + AuditLogEntry per mutation
- [x] Workspace isolation (403 on cross-workspace)

### Phase 4: Ingestion
- [x] Webhook receiver signature-verified, idempotent
- [x] All 4 parsers (JUnit, TestNG, Playwright, Cypress) working
- [x] Cases matched by name; secrets redacted
- [x] Ingestion atomic; no partial results
- [x] Manual upload works from frontend

### Phase 5: AI Drafting
- [x] Draft → Approve / Reject / Redraft flows work
- [x] Autonomy gating enforced
- [x] Role-based approval (QA_LEAD | TECH_LEAD)
- [x] Daily rate limit + UI quota display
- [x] Stale-draft badge + re-draft
- [x] SkillInvocationEvent emitted

### Phase 6: Traceability & Polish
- [x] Traceability aggregate + 4 projections working
- [x] `TestCoverageFacade` ready for Requirement-slice integration
- [x] Per-card error + empty states visible
- [x] Audit + LineageEvent complete
- [x] Accessibility + i18n + performance within budget
- [x] All deep-links end-to-end
- [x] End-to-end smoke test passing

---

## Doc Quality Gate Before Merge

Before merging any Phase, verify:

1. **All 9 SDD documents exist and are complete** (CLAUDE.md rule #6):
   - `01-requirements/testing-management-requirements.md`
   - `02-user-stories/testing-management-stories.md`
   - `03-spec/testing-management-spec.md`
   - `04-architecture/testing-management-architecture.md`
   - `04-architecture/testing-management-data-flow.md`
   - `04-architecture/testing-management-data-model.md`
   - `05-design/testing-management-design.md`
   - `05-design/contracts/testing-management-API_IMPLEMENTATION_GUIDE.md`
   - `06-tasks/testing-management-tasks.md` (this file)

2. **Architecture docs include Mermaid 8.x diagrams** (CLAUDE.md rule #7):
   - System context diagram
   - Component breakdown diagram
   - Data flow sequence diagram
   - State boundaries diagram
   - Integration diagram
   - Use Mermaid 8.x syntax only (no C4Context, no direction in subgraphs, no `→` in node text)

3. **Design docs include concrete implementation details** (CLAUDE.md rule #8):
   - File structure map (actual Vue/Java paths)
   - Component API contracts (props, inputs, sources)
   - Data model (frontend types AND backend DTOs/entities)
   - API contracts (endpoints, request/response JSON, HTTP status codes)
   - Visual design decisions (tokens, typography, animation)
   - Database schema DDL (with column types, constraints)
   - Error and empty state design (per REQ-TM-71)
   - Integration boundary diagram

4. **Data-flow, data-model, and API implementation guide are complete** (CLAUDE.md rule #9):
   - `data-flow.md`: runtime data flows, sequence diagrams, state machine, refresh strategy, API client chain
   - `data-model.md`: domain model ER diagram, frontend type catalog, backend DTO definitions, DB schema DDL (current + future), type mapping table
   - `API_IMPLEMENTATION_GUIDE.md`: full endpoint contract with JSON examples, error handling, backend/frontend implementation skeleton, testing contracts

5. **Every REQ-TM-* ID is referenced in the tasks** — at least once per major feature area.

6. **Flyway migrations explicitly named** with version numbers and naming convention `V{version}__{description}.sql`.

7. **Package-by-feature structure used for backend** — no flat `controller/` / `service/` / `repository/` packages (CLAUDE.md rule #3).

8. **No ddl-auto beyond local H2 dev** — all schema changes via Flyway (CLAUDE.md rule #4).

---

## Risks & Blockers

| Risk | Mitigation |
|------|-----------|
| `RequirementLookup` facade not implemented | Stub with static map; flag upstream blocker; document cutover plan |
| Webhook payload >25 MiB | Raise Spring body limit to 25 MiB for `/webhooks/ingest`; reject >25 MiB with 413 |
| AI skill unavailable during V1 development | Use mock skill client returning canned responses; feature-flag AI endpoints; document cutover plan |
| Per-projection timeout cascades | Each projection has 500ms hard limit; timeout → `SectionResultDto(error=...)` per card only; never page-level |
| Workspace isolation bypass via entity-id spoofing | Access guard always resolves workspace from entity row itself; never trusts request body |
| REQ chip lookup fails silently | Render as UNVERIFIED (grey) + tooltip; never hide — absence of verification ≠ absence of link (REQ-TM-73) |
| Stale REQ links accumulate | Hourly resolver retries; no expiry in V1 (deferred to V2) |
| Log redaction false positives | Redactor runs only on captured logs; patterns tuned to AWS + GitHub + Bearer shapes; changes require reviewed changelog |
| AI evidence integrity bypass | `AiTriageEvidenceIntegrityPolicy` verifies per-row set membership; mismatches persist as `FAILED_EVIDENCE`, never block command |
| Run ingestion partial write on error | All ingestion in transaction; parse failure → run persisted in `INGEST_FAILED` state; no partial results (REQ-TM-74) |
| Oracle CLOB diverges from H2 | Oracle-variant migration under `db/migration/oracle/`; tested before deployment |
| Daily AI draft rate limit gaming | Rate limit per (workspace, reqId); stored per day; reset at UTC midnight; backend enforced + UI quota display |

---

## Open Questions

- Should AI draft approval be automatic at `AUTONOMOUS` autonomy level? Default: **NO** — autonomy never grants authoring authority; human approval always required (REQ-TM-56).
- Should DRAFT cases appear in "recent runs" coverage for their plan? Default: **NO** — DRAFT cases excluded from all active views until ACTIVE (REQ-TM-52).
- Should unresolved REQ links expire after N days? Default: **NO** in V1 — nightly resolver keeps trying; expiry deferred to V2.
- Should run redaction auto-heal broken cases on subsequent runs? Default: **NO** — deferred to V2 AI self-healing capability.
- How long to retain test run logs? Default: 90d for SUCCESS, 365d for FAILURE, forever for runs linked to an incident.
- Should environment-level entitlements / approval gates ship in V1? Default: **NO** — deferred; simple environment registry only.

---

## Dependency Plan

### Critical Path

```
Phase 0: Scaffolding
  ↓
Phase 1: Backend MVP (projections, services, controllers)
  ↓
Phase 2: Frontend MVP (types, mocks, views)
  ↓
Phase 3: CRUD mutations (backend commands, frontend forms)
  ↓
Phase 4: Run ingestion (webhook, parsers, dispatcher)
  ↓
Phase 5: AI drafting (skill invocation, state machine, approvals)
  ↓
Phase 6: Traceability & polish (facade, cross-slice integration, accessibility)
```

### Parallel Workstreams

- Phase 1 projections can be built in parallel by multiple contributors
- Phase 2 component development (ReqChip, CoverageIndicator, etc.) can proceed in parallel with Phase 1
- Phase 3 CRUD (plan, case, env) forms can be built in parallel with Phase 1 backend
- Phase 4 parsers (JUnit, TestNG, Playwright, Cypress) can be built in parallel with Phase 1/2
- Phase 5 AI skill integration can start once Phase 4 is near completion
- Phase 6 polishing (a11y, i18n, perf) can start once Phase 5 core is working

---

## Summary

**Testing Management Tasks** defines a 7-phase phased implementation plan for the Testing Management slice:

- **Phase 0**: Scaffolding & Flyway baseline
- **Phase 1**: Backend read-only MVP (Catalog, Plan, Case, Run, Traceability views)
- **Phase 2**: Frontend read-only MVP (Vue components, mocks)
- **Phase 3**: CRUD mutations (Plan, Case, Environment forms)
- **Phase 4**: Test-run ingestion (Webhook, JUnit/TestNG/Playwright/Cypress parsers, secret redaction)
- **Phase 5**: AI test-case drafting (Skill invocation, DRAFT→ACTIVE→DEPRECATED state machine, autonomy gating, approval)
- **Phase 6**: Traceability & cross-slice integration (Requirement-slice "Tests" tab, accessibility, i18n, performance)

Every phase includes explicit Codex prompts for Frontend and Backend, Flyway migration names, test requirements, and definition-of-done checklist. All 7 phases anchor tasks to specific REQ-TM-* IDs and design/architecture/data-model document sections.
