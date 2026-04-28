# Requirement Management Tasks

## Objective

Implement the Requirement Management page — the SDD chain entry point of the
Agentic SDLC Control Tower product. This is where requirements are captured,
classified, prioritized, and decomposed into User Stories and Specs.

## Implementation Strategy

Frontend and backend are developed in separate phases with different tools:

| Phase   | Scope                 | Tool        | Goal                                             |
| ------- | --------------------- | ----------- | ------------------------------------------------ |
| Phase A | Frontend (Vue 3)      | Claude Code | Build requirement UI with mocked data            |
| Phase B | Backend (Spring Boot) | Claude Code | Build API and connect frontend to live data      |

Phase A is self-contained and produces a fully functional requirement page with mocked data.
Phase B adds the backend and replaces mocked data with real API calls.

## Traceability

- Requirements: [requirement-requirements.md](../01-requirements/requirement-requirements.md)
- Stories: [requirement-stories.md](../02-user-stories/requirement-stories.md)
- Spec: [requirement-spec.md](../03-spec/requirement-spec.md)
- Architecture: [requirement-architecture.md](../04-architecture/requirement-architecture.md)
- Design: [requirement-design.md](../05-design/requirement-design.md)
- Data Flow: [requirement-data-flow.md](../04-architecture/requirement-data-flow.md)
- Data Model: [requirement-data-model.md](../04-architecture/requirement-data-model.md)
- API Guide: [requirement-API_IMPLEMENTATION_GUIDE.md](../05-design/contracts/requirement-API_IMPLEMENTATION_GUIDE.md)

## Planning Assumptions

- `SectionResult<T>` already exists in `shared/types/section.ts` (moved during incident slice)
- `fetchJson<T>` and `postJson<T>` already exist in `shared/api/client.ts`
- API envelope: `ApiResponse<T>` exists in backend `shared/dto/`
- `SectionResultDto<T>` already exists in backend `shared/dto/` (moved during incident slice)
- Mock toggle pattern follows incident/dashboard: `import.meta.env.DEV && !import.meta.env.VITE_USE_BACKEND`
- `SdlcChainLink` / `SdlcChain` types can be reused from `incident/types/incident.ts` or extracted to a shared location
- Route at `/requirements` currently points to the placeholder page (with `comingSoon: true` in router)
- Nested routing pattern established by incident slice can be followed
- `CHILD_ROUTES` map in `router/index.ts` supports adding requirement child routes
- Flyway migrations V1–V4 already exist; requirement tables will be V5
- V1 ships with two hardcoded pipeline profiles: "Standard SDD" (default) and "IBM i"
- The IBM i profile delegates to a single skill (`ibm-i-workflow-orchestrator`) that automatically determines path and tier
- Profile definitions are TypeScript constants in `features/requirement/profiles/`; no backend API for profiles in V1
- Profile-specific skill invocation is stubbed in V1 (shows confirmation toast); actual skill execution requires AI Center integration
- IBM i skill family (`build-agent-skill`) is an external dependency; this module only references the orchestrator skill ID
- File parsing libraries: SheetJS (xlsx) for Excel, pdf.js for PDF, mailparser for email — all client-side in Phase A
- AI normalization is stubbed in Phase A (returns mock RequirementDraft based on input type)
- Import panel design follows the glassmorphism overlay pattern from the design system
- Maximum file size: 10MB; maximum batch rows: 50

---

## Phase A: Frontend

### A0: Shared Infrastructure Preparation

- Remove `comingSoon: true` from the `requirements` entry in `NAVIGATION_ITEMS` (`router/index.ts:27`)
- Add page config for `requirements` in `PAGE_CONFIGS`:
  ```
  requirements: {
    subtitle: 'SDD chain entry point — capture, classify, decompose',
    actions: [
      { key: 'ai-analyze', label: 'AI ANALYZE', variant: 'ai' },
    ],
  }
  ```
- Add `requirements` to `COMPONENT_MAP` pointing to `@/features/requirement/RequirementManagementView.vue`
- Add child routes to `CHILD_ROUTES`:
  ```
  requirements: [
    { path: '', name: 'requirements', component: () => import('@/features/requirement/views/RequirementListView.vue') },
    { path: ':requirementId', name: 'requirement-detail', component: () => import('@/features/requirement/views/RequirementDetailView.vue') },
  ]
  ```
- Extract `SdlcChainLink`, `SdlcChain`, and `SdlcArtifactType` to `shared/types/sdlc-chain.ts` if not already shared, and re-export from incident types so existing imports are not broken
- Verify shared types (`SectionResult`, `SdlcChainLink`) are importable from the shared path
- Add requirement-specific design tokens to `variables.css` if needed (category colors: functional=cyan, non-functional=amber, technical=purple, business=green)

Depends on: none
Done when: `/requirements` route is active (no longer "Coming Soon"); child routes registered; shared types importable; dashboard and incident still build clean

### A0.5: Pipeline Profile Infrastructure

- Create `frontend/src/features/requirement/profiles/index.ts` — profile registry with `getActiveProfile(workspaceId)` resolver (returns hardcoded profile based on workspace config; V1 defaults to Standard SDD)
- Create `frontend/src/features/requirement/profiles/standardSddProfile.ts` — Standard SDD profile constant with 11 chain nodes, 2 skills (req-to-user-story, user-story-to-spec), no tiering, single entry path, per-layer traceability, `usesOrchestrator: false`
- Create `frontend/src/features/requirement/profiles/ibmIProfile.ts` — single IBM i profile constant with 10 chain nodes, 1 skill (`ibm-i-workflow-orchestrator`), L1/L2/L3 tiering (orchestrator-determined), 3 entry paths (orchestrator-determined, not user-selected), shared-br traceability, `usesOrchestrator: true`
- Add `PipelineProfile`, `ChainNode`, `SpecTier`, `EntryPath`, `SkillBinding`, `OrchestratorResult` interfaces to `types/requirement.ts`

Depends on: A0
Done when: Both profiles are importable and type-safe; `getActiveProfile()` returns Standard SDD by default; IBM i profile has single skill binding (ibm-i-workflow-orchestrator)

### A1: Define Requirement Types and Mock Data

- Create `frontend/src/features/requirement/types/requirement.ts` with all TypeScript interfaces:
  - Enums/union types: `RequirementPriority` (`Critical` | `High` | `Medium` | `Low`), `RequirementStatus` (`Draft` | `In Review` | `Approved` | `In Progress` | `Delivered` | `Archived`), `RequirementCategory` (`Functional` | `Non-Functional` | `Technical` | `Business`), `RequirementSource` (`Manual` | `Imported` | `AI-Generated`), `StoryStatus`, `SpecStatus`, `AnalysisConfidence` (`High` | `Medium` | `Low`)
  - List types: `StatusDistribution` (counts per status), `RequirementListItem` (id, title, priority, status, category, storyCount, specCount, completeness, updatedAt), `RequirementFilters` (priority?, status?, category?, search?, showCompleted), `RequirementList` (statusDistribution, requirements[])
  - Detail types: `RequirementHeader` (id, title, priority, status, category, source, assignee, createdAt, updatedAt), `RequirementDescription` (summary, businessJustification, acceptanceCriteria[], assumptions[], constraints[]), `AcceptanceCriterion` (id, text, isMet), `LinkedStory` (id, title, status, specId?, specStatus?), `LinkedSpec` (id, title, status, version), `LinkedStoriesSection` (stories[], totalCount), `LinkedSpecsSection` (specs[], totalCount), `AiAnalysis` (completenessScore, missingElements[], similarRequirements[], impactAssessment, suggestions[]), `SdlcChain` (reuse from shared), `RequirementDetail` with 6 `SectionResult<T>` fields (header, description, linkedStories, linkedSpecs, aiAnalysis, sdlcChain)
- Create `frontend/src/features/requirement/mockData.ts` with comprehensive mock data:
  - 10 requirements (REQ-0001 through REQ-0010) across all statuses and priorities:
    - REQ-0001: Critical, Approved, Functional — "User Authentication and SSO Integration"
    - REQ-0002: High, In Progress, Functional — "Role-Based Access Control"
    - REQ-0003: High, Draft, Non-Functional — "API Response Time Under 200ms"
    - REQ-0004: Medium, In Review, Technical — "Database Migration to Oracle 23ai"
    - REQ-0005: Critical, In Progress, Business — "Audit Trail for Compliance"
    - REQ-0006: Low, Delivered, Functional — "User Profile Management"
    - REQ-0007: Medium, Approved, Non-Functional — "99.9% Uptime SLA"
    - REQ-0008: High, Draft, Technical — "Event-Driven Architecture Migration"
    - REQ-0009: Medium, Archived, Business — "Legacy Report Export"
    - REQ-0010: High, In Review, Functional — "AI-Powered Requirement Analysis"
  - 2–4 linked stories per requirement (varying statuses)
  - 0–2 linked specs per requirement (some without specs to show gaps)
  - AI analysis data for requirements in Approved/In Progress states
  - SDLC chain data showing forward chain from requirement
  - Status distribution matching the 10 requirements above
- Mock data values must be realistic and demonstrate all UI states

Depends on: A0
Done when: types compile; mock data is importable and type-safe; all status/priority/category combinations covered

### A2: Build Reusable Badge Components

- Create `frontend/src/features/requirement/components/PriorityBadge.vue` — Critical (crimson), High (amber), Medium (cyan), Low (muted)
- Create `frontend/src/features/requirement/components/RequirementStatusBadge.vue` — LED indicator + status text, active statuses pulse
- Create `frontend/src/features/requirement/components/CategoryBadge.vue` — category label with subtle background tint (Functional=cyan, Non-Functional=amber, Technical=purple, Business=green)
- Create `frontend/src/features/requirement/components/RequirementCard.vue` — reusable card wrapper following IncidentCard/DashboardCard pattern (surface-container-high, 4px radius, no borders, per-card error handling via SectionResult)

Depends on: A0
Done when: all 4 components render correctly with sample props; visual style matches design system

### A2.5: Profile-Adaptive Components

- Create `ProfileBadge.vue` — shows active profile name in page header (label-md, uppercase, HUD style)
- Create `ProfileChainCard.vue` — renders chain nodes dynamically from profile, highlights execution hub with cyan glow; replaces hardcoded SdlcChainCard
- Create `EntryPathSelector.vue` — read-only badge showing orchestrator-determined path; hidden until orchestrator returns a result; hidden for single-path profiles
- Create `SpecTierSelector.vue` — read-only badge showing orchestrator-determined tier (L1/L2/L3); hidden when specTiering is null or until orchestrator returns a result
- Create `ProfileSkillActions.vue` — for Standard SDD: renders action buttons from profile skill bindings; for IBM i: renders single "Send to Orchestrator" button

Depends on: A0.5, A2
Done when: All 5 components render correctly with both Standard SDD and IBM i profile data; tier/path displays show orchestrator results as read-only; IBM i shows single "Send to Orchestrator" button

### A3: Build List View

- Create `frontend/src/features/requirement/views/RequirementListView.vue` with:
  - `StatusDistribution.vue` — summary strip at top showing counts per status (Draft, In Review, Approved, In Progress, Delivered, Archived)
  - Filter bar: status dropdown, priority dropdown, category dropdown, search input
  - View toggle: List / Kanban buttons (top-right, adjacent to filters)
  - `RequirementListTable.vue` — sortable table with columns: ID, Title, Priority, Status, Category, Stories, Specs, Completeness, Updated
  - Each row uses PriorityBadge, RequirementStatusBadge, CategoryBadge
  - Row click emits `@select` with requirement ID
  - Completeness column shows a mini progress indicator (percentage or bar)
- Create `frontend/src/features/requirement/components/StatusDistribution.vue`
- Create `frontend/src/features/requirement/components/RequirementListTable.vue`
- Implement sort by priority, status, recency, title
- Empty state: "No requirements yet — create your first requirement to begin the SDD chain"
- Loading state: skeleton rows

Depends on: A1, A2
Done when: list renders 10 mock requirements; sorting works; filtering works; empty state renders; view toggle switches to kanban route

### A4: Build Kanban View

- Create `frontend/src/features/requirement/views/RequirementKanbanView.vue` with columns per status
- Create `frontend/src/features/requirement/components/KanbanColumn.vue` — single column with count badge and stacked requirement cards
- Columns: Draft | In Review | Approved | In Progress | Delivered | Archived
- Each card shows: ID, title, priority badge, category badge, story/spec counts
- Read-only in V1 (no drag-drop)
- Same filter bar as list view — share filter state via store
- All 6 statuses are primary columns (no override/collapsed section)

Depends on: A1, A2
Done when: kanban renders with cards distributed across correct columns; filter bar shared with list view; column counts match

### A5: Build Detail View — Header and Description Cards

- Create `frontend/src/features/requirement/views/RequirementDetailView.vue` — layout host for all detail cards (CSS grid)
- Create `frontend/src/features/requirement/components/RequirementHeaderCard.vue`:
  - Requirement ID (JetBrains Mono), title, priority badge, status badge, category badge
  - Source label (Manual / Imported / AI-Generated)
  - Assignee name
  - Created and last modified timestamps
  - Full-width layout in detail view (row 1)
- Create `frontend/src/features/requirement/components/DescriptionCard.vue`:
  - Summary text (rich text display)
  - Business justification section
  - Acceptance criteria as interactive checkbox list (each criterion shows met/unmet)
  - Assumptions list (bulleted)
  - Constraints list (bulleted)
  - Card spans left column, rows 2–3

Depends on: A2
Done when: header renders REQ-0001 data with all fields; description card shows acceptance criteria with checkboxes

### A6: Build Detail View — Stories and Specs Cards

- Create `frontend/src/features/requirement/components/LinkedStoriesCard.vue`:
  - List of derived user stories with: story ID, title, status badge, linked spec indicator
  - "Generate Stories" button (emits `@generate-stories`, no-op in V1 but wired)
  - Empty state: "No stories derived yet"
  - Navigation link per story (emits `@navigate(storyPath)`)
  - Shows total count and decomposition completeness indicator
- Create `frontend/src/features/requirement/components/LinkedSpecsCard.vue`:
  - List of linked specs with: spec ID, title, status badge, version
  - "Generate Spec" button (emits `@generate-spec`, no-op in V1 but wired)
  - Spec entries highlighted with a subtle accent to emphasize Spec-as-execution-hub
  - Empty state: "No specs linked — generate from stories"
  - Navigation link per spec (emits `@navigate(specPath)`)

Depends on: A2
Done when: stories card renders linked stories with status; specs card shows spec entries with version; generate buttons visible; empty states render

**Profile integration notes:**
- LinkedSpecsCard should use `SpecTierSelector` when `activeProfile.specTiering` is not null
- "Generate" buttons should delegate to `ProfileSkillActions` component instead of hardcoding skill names

### A7: Build Detail View — Chain and AI Analysis Cards

- Create `frontend/src/features/requirement/components/SdlcChainCard.vue`:
  - Reuse pattern from incident but show **forward** chain from requirement
  - Chain direction: Requirement → User Story → Spec → Architecture → Design → Tasks → Code → Test → Deploy
  - Compressed view: shows only linked artifact types (not all 9 nodes)
  - Spec node always visible and highlighted even if not directly linked
  - Each link: artifact type icon, ID, title, clickable
  - Click emits `@navigate(routePath)`
  - Empty state: "No downstream artifacts yet"
  - Expand/collapse control to show full chain context
- Create `frontend/src/features/requirement/components/AiAnalysisCard.vue`:
  - Completeness score displayed as a progress ring (0–100%)
  - Missing elements list (e.g., "Missing acceptance criteria", "No business justification")
  - Similar requirements section (list of IDs with similarity percentage)
  - Impact assessment text
  - Suggestions list (bulleted actionable items)
  - "Run Analysis" button (emits `@run-analysis`, no-op in V1 but wired)
  - When no analysis available: "Run AI analysis to assess requirement quality"

Depends on: A2
Done when: chain card renders forward chain with Spec highlighted; AI analysis card shows completeness ring, missing elements, and suggestions; empty states render

**Profile integration note:**
- Replace hardcoded `SdlcChainCard` with `ProfileChainCard` that reads chain nodes from active profile

### A8: Build Priority Matrix View

- Create `frontend/src/features/requirement/components/PriorityMatrix.vue`:
  - 2x2 grid: X-axis = Effort (Low/High), Y-axis = Impact (Low/High)
  - Quadrants labeled: Quick Wins (high impact, low effort), Strategic (high impact, high effort), Fill-ins (low impact, low effort), Deprioritize (low impact, high effort)
  - Requirements plotted as dots/chips in quadrants (using priority badge colors)
  - Click on a dot navigates to requirement detail
  - Accessible as a sub-view toggle from list view (List / Kanban / Matrix)
- Required view mode — matrix is part of the core three-view toggle (List / Kanban / Matrix)

Depends on: A1
Done when: matrix renders with requirements plotted in quadrants; clicking navigates to detail

### A9: Build Pinia Store and API Client

- Create `frontend/src/features/requirement/stores/requirementStore.ts` (Pinia store):
  - Store shape: `list` (requirements[], statusDistribution, isLoading, error, filters) + `detail` (6 SectionResult fields, isLoading, error) + `selectedRequirementId` + `activeProfile: PipelineProfile` + `orchestratorResult: OrchestratorResult | null`
  - Actions: `fetchRequirementList()`, `fetchRequirementDetail(id)`, `setFilters(filters)`, `clearDetail()`, `generateStories(requirementId)` (stub), `generateSpec(storyId)` (stub), `runAnalysis(requirementId)` (stub), `loadActiveProfile(workspaceId)`, `invokeProfileSkill(requirementId, skillId)` (stubbed in Phase A; for IBM i, returns mock OrchestratorResult with determined path and tier)
  - Mock toggle: `import.meta.env.DEV && !import.meta.env.VITE_USE_BACKEND`
  - Phase A: imports from `mockData.ts`; Phase B: calls `requirementApi.ts`
  - Filter logic: client-side filtering of mock data by priority, status, category, search text
- Create `frontend/src/features/requirement/api/requirementApi.ts`:
  - `getRequirementList(filters?)` → `fetchJson<RequirementList>('/requirements')`
  - `getRequirementDetail(id)` → `fetchJson<RequirementDetail>('/requirements/${id}')`
  - `triggerStoryGeneration(requirementId)` → `postJson('/requirements/${requirementId}/generate-stories')`
  - `triggerSpecGeneration(storyId)` → `postJson('/requirements/stories/${storyId}/generate-spec')`
  - `triggerAnalysis(requirementId)` → `postJson('/requirements/${requirementId}/analyze')`

Depends on: A1
Done when: store is importable; `fetchRequirementList()` returns mock data; reactive state works; filters apply correctly

### A10: Build Navigation and State Integration

- Create `frontend/src/features/requirement/RequirementManagementView.vue` as `<router-view />` host (same pattern as `IncidentManagementView.vue`)
- Wire `RequirementListView.vue` to store:
  - Filter changes → `setFilters()` → re-filter mock data
  - Row click → `router.push({ name: 'requirement-detail', params: { requirementId } })`
  - View toggle → query param or route switch between list/kanban
  - Status distribution wired to store data
- Wire `RequirementDetailView.vue` to store:
  - On mount: `fetchRequirementDetail(route.params.requirementId)`
  - Cards wired to store detail state via SectionResult
  - Card events: generate-stories, generate-spec, run-analysis → store actions
  - Navigate events → `router.push`
- Breadcrumb integration: Requirements > [Requirement Title]
- Deep linking support: `/requirements/REQ-0001` loads detail directly
- Loading/error/empty states per card via SectionResult pattern

Depends on: A3, A4, A5, A6, A7, A9
Done when: list view renders from store; filtering works; clicking navigates to detail; detail view renders all cards from store; browser back navigation works; deep-linking works

### A11: Validate Requirement Frontend

- Verify all 10 mock requirements render in the list with correct badges
- Verify status distribution shows correct counts matching the 10 requirements
- Verify filtering by priority, status, category works
- Verify text search filters by title
- Verify sorting by priority, recency, status, title works
- Verify view toggle switches between list and kanban views
- Verify kanban columns show correct requirement distribution
- Verify kanban column counts match filter results
- Verify REQ-0001 detail renders all 6 cards correctly:
  - Header card: ID, title, priority, status, category, assignee, dates
  - Description card: summary, business justification, acceptance criteria (checkable), assumptions, constraints
  - Linked stories card: stories with status badges
  - Linked specs card: specs with version and status, Spec highlighted
  - SDLC chain card: forward chain with Spec visible
  - AI analysis card: completeness ring, missing elements, suggestions
- Verify "Generate Stories" and "Generate Spec" buttons are visible and emit events
- Verify "Run Analysis" button is visible and emits event
- Verify SDLC chain links navigate correctly (to placeholder pages for unimplemented modules)
- Verify empty states render for requirements without stories/specs/analysis
- Verify browser back navigation between list and detail
- Verify deep-linking to `/requirements/REQ-0001` works
- Verify `npm run dev` and `npm run build` both succeed

Depends on: A10
Done when: all checks pass; requirement page is complete with mocked data

### A12: Requirement Import — Single Input Flow

- Create `ImportPanel.vue` — full-screen modal with glassmorphism overlay (840px width), hosts import flow state machine
- Create `ImportSourceTabs.vue` — horizontal tabs: Paste Text, Upload File, Email, Meeting Summary
- Create `ImportTextInput.vue` — textarea for paste input with character count
- Create `ImportDropZone.vue` — drag-and-drop zone accepting .xlsx/.csv/.pdf/.eml/.msg/.vtt/.txt/.png/.jpg/.jpeg/.webp (10MB limit)
- Create `NormalizationResultCard.vue` — AI draft review form with:
  - All extracted fields (title, priority, category, description, acceptance criteria, etc.)
  - AI badge on suggested values (cyan pill)
  - Editable fields with inline editing
  - Missing info amber banners (MissingInfoBanner.vue)
  - Open questions collapsible section
  - Three actions: Edit Draft, Confirm & Create (AI gradient button), Discard
- Add "Import Requirement" button to RequirementListView.vue page header
- Wire import panel open/close to store state
- Add import state management to requirementStore.ts:
  - `importState: ImportState` with step tracking
  - `setRawInput()`, `triggerNormalization()`, `confirmDraft()`, `discardDraft()` actions
  - Phase A: `triggerNormalization()` returns mock draft from template based on input
- File parsing (client-side):
  - Install SheetJS (`xlsx` package) for Excel parsing
  - Install `pdfjs-dist` for PDF text extraction
  - Basic email parsing: extract subject/body from .eml text (full .msg parsing is V2)
  - Meeting transcript: treat as plain text input

Depends on: A1 (types), A2 (badges), A9 (store)
Done when: User can paste text or upload a file, see a mock AI draft, edit fields, and confirm to add a requirement to the list

### A13: Requirement Import — Batch Excel Flow

- Create `BatchPreviewTable.vue` — table with checkbox column, parsed rows, auto-detected column mappings
- Create `ColumnMappingEditor.vue` — dropdown per column to map source → target field
- Create `BatchProgressBar.vue` — progress bar with "3 of 12 normalized..." text
- Extend import flow in requirementStore.ts:
  - `batchPreview`, `batchDrafts`, `batchProgress` state
  - `parseBatchFile()` action using SheetJS
  - `normalizeSelected()` action (sequential, with progress updates)
  - `confirmAllDrafts()`, `confirmDraft(index)` actions
- Wire batch flow into ImportPanel.vue (detect .xlsx/.csv → switch to batch-preview step)
- Column auto-detection: match common header names (Title, Name, Description, Priority, Category, Type, Status)

Depends on: A12
Done when: User can upload an Excel file, preview rows, select rows, see mock normalization, and batch-create requirements

### Phase A Definition of Done

- Requirement list view renders 10 requirements with status distribution
- Requirement kanban view renders cards across status columns
- Requirement detail view renders all 6 cards inside the shared shell
- Acceptance criteria display as interactive checkboxes
- Linked stories and specs cards show decomposition status
- Spec entries are visually highlighted as the execution hub
- SDLC forward chain card shows chain with Spec always visible
- AI analysis card shows completeness score, missing elements, and suggestions
- "Generate Stories", "Generate Spec", and "Run Analysis" buttons are wired (stub actions)
- View toggle switches between List and Kanban views
- Navigation between list and detail works; deep-linking works
- Filtering and sorting work across both list and kanban views
- Loading, error, and empty states all render correctly
- No backend dependency; all data is mocked
- `npm run dev` and `npm run build` both succeed

---

## Phase B: Backend

### B0: Shared Infrastructure (Prerequisite)

- Add requirement endpoint constants to `ApiConstants.java`:
  ```
  REQUIREMENTS = API_V1 + "/requirements"
  REQUIREMENT_DETAIL = REQUIREMENTS + "/{requirementId}"
  REQUIREMENT_GENERATE_STORIES = REQUIREMENT_DETAIL + "/generate-stories"
  REQUIREMENT_STORY_GENERATE_SPEC = REQUIREMENTS + "/stories/{storyId}/generate-spec"
  REQUIREMENT_ANALYZE = REQUIREMENT_DETAIL + "/analyze"
  ```
- Verify existing dashboard and incident tests still pass after the addition

Depends on: none (can start in parallel with Phase A)
Done when: `ApiConstants` has requirement paths; existing tests pass

### B1: Create Backend Domain Package Structure

- Create package `domain/requirement/` with sub-packages:
  - `domain/requirement/controller/`
  - `domain/requirement/service/`
  - `domain/requirement/dto/`

Depends on: B0
Done when: package structure exists

### B2: Create Requirement DTOs

- Create Java records in `domain/requirement/dto/`:
  - List: `RequirementListDto`, `RequirementListItemDto`, `StatusDistributionDto`
  - Detail: `RequirementDetailDto`, `RequirementHeaderDto`, `RequirementDescriptionDto`, `AcceptanceCriterionDto`, `LinkedStoriesSectionDto`, `LinkedStoryDto`, `LinkedSpecsSectionDto`, `LinkedSpecDto`, `AiAnalysisDto`, `SdlcChainDto`, `SdlcChainLinkDto`
  - Request: `GenerateStoriesRequestDto`, `GenerateSpecRequestDto`, `RunAnalysisRequestDto`
  - Response: `GenerationResultDto` (for stub responses)
- All DTOs are Java records (immutable)
- Field names match frontend TypeScript interfaces exactly (camelCase JSON via Jackson defaults)
- `RequirementDetailDto` wraps 6 sections in `SectionResultDto<T>` (header, description, linkedStories, linkedSpecs, aiAnalysis, sdlcChain)

Depends on: B1
Done when: all DTOs compile; Jackson serializes to JSON matching the API guide examples

### B3: Implement Requirement Service

- Create `RequirementService.java` in `domain/requirement/service/`
- Implement `getRequirementList(filters)` returning `RequirementListDto` with seed data
- Implement `getRequirementDetail(id)` returning `RequirementDetailDto` with 6 `SectionResultDto` sections
- Implement `generateStories(requirementId)` — returns stub `GenerationResultDto` with message "Story generation queued"
- Implement `generateSpec(storyId)` — returns stub `GenerationResultDto` with message "Spec generation queued"
- Implement `runAnalysis(requirementId)` — returns stub `GenerationResultDto` with message "Analysis queued"
- Seed data must match the frontend mock data / API guide JSON examples
- Service is workspace-aware (accepts workspace ID parameter for future isolation)
- Filtering logic: filter by priority, status, category, search text server-side

Depends on: B2
Done when: service returns valid DTOs with realistic seed data matching frontend mocks

### B3.5: Pipeline Profile API

- Create `GET /api/v1/pipeline-profiles/active` endpoint in a new `platform/profile/` package (or `shared/profile/`)
- Returns resolved `PipelineProfileDto` based on workspace/project inheritance
- V1 implementation: hardcoded profiles (same as frontend — "Standard SDD" and "IBM i"); future: database-backed with Platform Center UI
- IBM i profile has `usesOrchestrator: true` and single skill binding (`ibm-i-workflow-orchestrator`)
- Create `POST /api/v1/requirements/{id}/invoke-skill` endpoint in RequirementController
- Accepts `{ skillId }` body (no entryPathId or tierId — orchestrator determines these automatically for IBM i)
- Returns `SkillExecutionResultDto` with execution ID and optional `orchestratorResult` (populated for IBM i with determined path and tier)
- V1 implementation: creates execution record, returns TRIGGERED status with mock orchestratorResult for IBM i (actual skill execution is future)

Depends on: B1, B2
Done when: Both endpoints return valid responses; IBM i invoke-skill returns OrchestratorResult; frontend can switch from hardcoded profiles to API

### B4: Implement Requirement Controller

- Create `RequirementController.java` in `domain/requirement/controller/` with:
  - `GET /api/v1/requirements` → `listRequirements(filters)` returns `ApiResponse<RequirementListDto>`
  - `GET /api/v1/requirements/{requirementId}` → `getRequirementDetail(id)` returns `ApiResponse<RequirementDetailDto>`
  - `POST /api/v1/requirements/{requirementId}/generate-stories` → returns `ApiResponse<GenerationResultDto>`
  - `POST /api/v1/requirements/stories/{storyId}/generate-spec` → returns `ApiResponse<GenerationResultDto>`
  - `POST /api/v1/requirements/{requirementId}/analyze` → returns `ApiResponse<GenerationResultDto>`
- Proper error handling: 404 for requirement not found, 400 for invalid request

Depends on: B3
Done when: all 5 endpoints return valid JSON matching the API guide

### B5: Create Flyway Seed Migration

- Create `V5__create_requirement_tables.sql` with:
  - `requirement` table: id, title, summary, business_justification, priority, status, category, source, assignee, completeness_score, created_at, updated_at, workspace_id
  - `requirement_acceptance_criterion` table: id, requirement_id, criterion_text, is_met
  - `requirement_assumption` table: id, requirement_id, assumption_text
  - `requirement_constraint` table: id, requirement_id, constraint_text
  - `user_story` table: id, requirement_id, title, status, spec_id, spec_status
  - `requirement_spec` table: id, requirement_id, title, status, version
  - `requirement_ai_analysis` table: id, requirement_id, completeness_score, impact_assessment, analyzed_at
  - `requirement_ai_analysis_element` table: id, analysis_id, element_type (missing/suggestion/similar), element_text
  - `requirement_sdlc_chain_link` table: id, requirement_id, artifact_type, artifact_id, artifact_title, route_path, link_order
- Seed data: 10 requirements matching frontend mock data, with linked stories, specs, acceptance criteria, and analysis data
- Seed data values must match the frontend mock data and API guide examples

Depends on: B2
Done when: migration runs cleanly on H2; tables and seed data visible in H2 console

### B6: Write Backend Tests

- Create `RequirementControllerTest.java` using MockMvc:
  - GET /api/v1/requirements returns 200 with requirements[] and statusDistribution
  - GET /api/v1/requirements?priority=Critical returns filtered results
  - GET /api/v1/requirements?category=Functional returns filtered results
  - GET /api/v1/requirements/REQ-0001 returns 200 with all 6 sections
  - GET /api/v1/requirements/REQ-9999 returns 404
  - POST /api/v1/requirements/REQ-0001/generate-stories returns 200 with result message
  - POST /api/v1/requirements/REQ-0001/analyze returns 200 with result message
  - POST /api/v1/requirements/stories/US-001/generate-spec returns 200 with result message
- Create `RequirementServiceTest.java` for unit tests:
  - getRequirementList returns all 10 requirements
  - getRequirementList with filters returns subset
  - getRequirementDetail returns valid sections
  - getRequirementDetail with invalid ID throws not found

Depends on: B4
Done when: all tests pass; coverage meets 80% threshold

### B7: Connect Frontend to Backend

- Update `requirementApi.ts` to call all 5 endpoints with proper path construction
- Update `requirementStore.ts` to use API instead of mock data when `VITE_USE_BACKEND` is set
- Configure Vite dev server proxy for `/api/v1/requirements/*` in `vite.config.ts` (verify proxy wildcard covers new paths)
- Handle loading and error states from real API responses
- Retain mock data as fallback for development without backend

Depends on: A11, B4
Done when: frontend renders live data from Spring Boot when `VITE_USE_BACKEND=true`

### B8: Validate Full Stack

- Verify frontend + backend integration end to end
- Verify requirement list loads from backend API with 10 requirements
- Verify requirement detail loads all 6 sections from backend
- Verify filtering by priority, status, category works through API
- Verify search text filtering works through API
- Verify generate-stories, generate-spec, and analyze stub endpoints return success
- Verify API error responses (404, 400) produce graceful frontend fallbacks
- Verify H2 profile works for local development
- Verify existing dashboard and incident functionality is not broken

Depends on: B7
Done when: full stack runs locally; requirement page displays backend seed data

### B9: Intake API Endpoints

- Create `POST /api/v1/requirements/normalize` endpoint:
  - Accepts `{ rawInput, profileId }` JSON body for pasted text intake
  - Invokes profile's normalizer skill (stubbed in V1: returns hardcoded draft)
  - Returns `ApiResponse<RequirementDraftDto>`
- Create `POST /api/v1/requirements/imports` endpoint:
  - Accepts KB-backed `multipart/form-data` for file intake
  - Multipart contract uses required `kb_name`, repeatable `file` fields, optional `profileId`
  - Supports batch upload with a `100 MB` total-request limit
  - Supports `.txt`, `.md`, `.pdf`, `.html`, `.htm`, `.xlsx`, `.xls`, `.docx`, `.csv`, and `.zip`
  - Returns an async receipt with `importId`, `taskId`, `datasetId`, file counts, supported/unsupported types, and per-file provider status
- Create `GET /api/v1/requirements/imports/{importId}` endpoint:
  - Polls KB/provider processing status
  - Returns `RequirementImportStatusDto`
  - Includes `draft` once KB indexing + normalization complete
- Create `POST /api/v1/requirements` endpoint:
  - Accepts confirmed draft as request body
  - Creates requirement record with status DRAFT
  - Stores source attachment metadata
  - Returns `ApiResponse<RequirementListItemDto>`
- Add import audit logging: source format, file name, skill ID, user, timestamp, outcome
- Add requirement-side KB provider abstraction so local stub and Dify-backed KB can share the same contract
- Switch frontend from mock normalization to JSON normalize for pasted text and async import receipt/polling for uploaded files

Depends on: B1, B2, B3
Done when: Import flow works end-to-end through the API; requirements persist in database

### Phase B Definition of Done

- Spring Boot serves requirement list and detail via REST API
- Generate-stories, generate-spec, and analyze stub endpoints return success responses
- API response shapes match frontend TypeScript contracts
- Frontend fetches live data from backend
- All 6 detail cards render from backend data
- Filtering and search work through the API
- Controller and service tests pass (12+ test cases)
- `./mvnw spring-boot:run` and `npm run dev` work together locally
- Existing dashboard and incident functionality unaffected

---

## Dependency Plan

### Critical Path

```
A0 → A1 → A9 → A10 → A11
         → A2 → A3 → A10
              → A5 ─┐
              → A6  ├→ A10 → A11
              → A7  ┘
A1 → A4 → A10
A1 → A8 (optional, parallel)
B0 → B1 → B2 → B3 → B4 → B6
                   → B5
A11 + B4 → B7 → B8
```

### Parallel Workstreams

| Stream                      | Tasks            | Can Run Alongside            |
| --------------------------- | ---------------- | ---------------------------- |
| Frontend types + data       | A0, A1           | — (first)                   |
| Badge components            | A2               | A1 (after A0)                |
| List view                   | A3               | After A1, A2                 |
| Kanban view                 | A4               | After A1, A2 (parallel w/ A3)|
| Detail cards — header/desc  | A5               | After A2 (parallel w/ A3–A4) |
| Detail cards — stories/specs| A6               | After A2 (parallel w/ A3–A5) |
| Detail cards — chain/AI     | A7               | After A2 (parallel w/ A3–A6) |
| Priority matrix (optional)  | A8               | After A1 (parallel w/ all)   |
| Store + API client          | A9               | After A1 (parallel w/ A2–A8) |
| Assembly + navigation       | A10              | After A3–A9                  |
| Backend DTOs + service      | B0, B1, B2, B3   | Parallel with all Phase A    |
| Backend controller + tests  | B4, B5, B6       | After B3                     |

---

## Risks / Blockers

| # | Risk                                                                                      | Impact            | Resolution                                                                  |
| - | ----------------------------------------------------------------------------------------- | ----------------- | --------------------------------------------------------------------------- |
| 1 | Extracting `SdlcChainLink` to shared may break incident imports                           | Build failure     | Re-export from incident types so existing imports are unchanged (A0)        |
| 2 | Kanban view performance with many requirements (500+ per REQ-REQ-113)                     | UI lag            | Virtualize column lists; test at scale during A4; paginate if needed        |
| 3 | Forward chain differs from incident reverse chain — SdlcChainCard may need forking        | Code duplication  | Parameterize direction in shared component or create requirement-specific variant |
| 4 | Priority matrix requires impact/effort data not in current mock schema                    | Incomplete A8     | Add impact/effort fields to requirement type; mark A8 optional              |
| 5 | AI stub endpoints (generate-stories, generate-spec, analyze) set user expectations        | UX confusion      | Show "Coming soon" toast on button click in V1; clear stub responses        |
| 6 | Route change from placeholder to nested may affect "Coming Soon" page behavior            | Navigation broken | Test route registration early (A0) before building components              |
| 7 | 6 cards in detail view CSS grid may have layout issues on narrower screens                | Visual breakage   | Test at 1280px minimum width; allow horizontal scroll if needed             |

---

## Open Questions

1. Should `SdlcChainLink` and `SdlcArtifactType` be extracted to `shared/types/sdlc-chain.ts` now (benefiting both incident and requirement), or should each module maintain its own copy? (Decision needed before A0)
2. Should the Kanban view support drag-and-drop status transitions in V1, or stay read-only? (V1 assumes read-only per this doc)
3. Should the Priority Matrix (A8) be a full view toggle (List/Kanban/Matrix) or a sub-component within the list view? (Design decision needed)
4. What is the minimum viewport width for the requirement detail grid? (1280px assumed, same as incident)
5. Should the "Generate Stories" and "Generate Spec" buttons show a modal confirmation or fire immediately? (V1 assumes immediate fire with toast feedback)
6. Should filter state be persisted in URL query parameters (e.g., `/requirements?priority=Critical&status=Draft`)? (Not required for V1 but recommended)
