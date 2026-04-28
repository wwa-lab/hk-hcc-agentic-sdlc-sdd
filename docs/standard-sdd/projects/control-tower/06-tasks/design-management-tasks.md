# Design Management Tasks

## Objective

Deliver the Design Management slice end-to-end in two phases: Phase A (frontend with mocks, Gemini) and Phase B (backend wiring, Codex). V1 scope is a **lightweight read-only viewer** for internally-authored Stitch / HTML design mocks, with Spec→Design traceability and AI summary (no critique, no generation), per the scope decisions recorded in `design-management-architecture.md` §Decisions. Implementation must conform to the 8 upstream SDD documents.

## Implementation Strategy

- **Frontend-first (Phase A):** mock-driven, no backend dependency. Ship a browsable Catalog view, a per-artifact Viewer view with CSP-hardened iframe preview, and a Traceability view. All admin mutations (register, publish version, link spec, regenerate AI summary) round-trip through a mock command layer that simulates `prevVersionId` stale-token rejection, `DM_PII_DETECTED`, and `DM_ARTIFACT_TOO_LARGE`.
- **Backend (Phase B):** implement projections, command services, controller, entities, and Flyway migrations V30–V36. Swap frontend from mock mode to live API by toggling `VITE_USE_BACKEND=true`. No `ddl-auto`.
- **Per-card isolation:** each Catalog section, each Viewer card, and the Traceability matrix is independently developable, mockable, and testable.
- **Net-new entities only:** Design Management introduces 5 net-new tables (`design_artifact`, `design_artifact_version`, `design_spec_link`, `design_ai_summary`, `design_change_log`). Project, Spec, Member, Workspace are reused read-only from upstream slices.
- **Coverage computed, not persisted:** link coverage (OK / PARTIAL / STALE / MISSING / UNKNOWN) is derived at request time from `(link.coversRevision, spec.latestRevision, spec.state, link.declaredCoverage)`. Never stored.
- **Shared primitives:** live under `src/shared/` / `com.sdlctower.shared` — Design Management does not fork shell-level behavior.

## Traceability

- Spec: [design-management-spec.md](../03-spec/design-management-spec.md)
- Design: [design-management-design.md](../05-design/design-management-design.md)
- Architecture: [design-management-architecture.md](../04-architecture/design-management-architecture.md)
- Data Flow: [design-management-data-flow.md](../04-architecture/design-management-data-flow.md)
- Data Model: [design-management-data-model.md](../04-architecture/design-management-data-model.md)
- API Guide: [design-management-API_IMPLEMENTATION_GUIDE.md](../05-design/contracts/design-management-API_IMPLEMENTATION_GUIDE.md)
- Requirements: [design-management-requirements.md](../01-requirements/design-management-requirements.md)
- Stories: [design-management-stories.md](../02-user-stories/design-management-stories.md)

## Planning Assumptions

- `shared-app-shell`, `dashboard`, `requirement`, `incident`, `team-space`, `project-space`, `project-management` slices are merged and operational.
- The Requirement slice exposes a read-only `SpecRevisionLookup` facade returning `(specId, latestRevision, state, title, projectId)` — if not yet available, stub it in `com.sdlctower.shared.integration.requirement` and document as a blocker.
- Code & Build, Testing, Deployment Management, Platform Center remain stubbed; Design Management does not deep-link into them in V1.
- H2 is the local dev DB; Oracle is the deployed DB. Flyway migrations V30–V36 must run on both (split per-dialect if DDL diverges — notably the `BLOB`/`CLOB` column for `html_payload` and `CHECK` constraints).
- Java 21 + Spring Boot 3.x; Vue 3 + Vite + Pinia + TypeScript.
- No localStorage / sessionStorage (per shared shell rules).
- `SectionResult<T>`, `ApiResponse<T>`, `fetchJson<T>`, `LineageBadge.vue` already exist and are reused as-is.
- Base path for all endpoints: `/api/v1/design-management` (per API guide §1).
- Mock toggle pattern follows repo convention: `import.meta.env.DEV && !import.meta.env.VITE_USE_BACKEND`.
- All admin mutations that touch a version require `prevVersionId`; stale tokens cause `DM_STALE_VERSION` (409).
- Access: Catalog + Viewer + Traceability are read-visible to any workspace member on the owning project. Admin mutations require Designer or PM role on that project.
- HTML payload limit: **2 MiB** per version, enforced server-side via `CHECK (html_size_bytes <= 2097152)` and client-side before upload.
- Preview iframe CSP: `default-src 'self' 'unsafe-inline' cdn.tailwindcss.com fonts.googleapis.com fonts.gstatic.com; sandbox allow-scripts allow-same-origin`. No other origins allowed — extending the list requires a platform review.

---

## Phase A: Frontend (Codex)

### A0: Shared Infrastructure Preparation

- [ ] Reuse `src/shared/types/section.ts` for `SectionResult<T>`
- [ ] Reuse `src/shared/api/client.ts` for `fetchJson<T>`
- [ ] Reuse `src/shared/components/LineageBadge.vue` for provenance badges
- [ ] Reuse `shellUiStore.setBreadcrumbs(...)` + `shellUiStore.setAiPanelContent(...)`
- [ ] Confirm Tactical Command tokens already exported from shared design tokens; add `--dm-coverage-ok`, `--dm-coverage-partial`, `--dm-coverage-stale`, `--dm-coverage-missing`, `--dm-coverage-unknown` to `src/shared/styles/tokens.css` only if absent (LED greens / ambers / crimsons already exist)
- [ ] Confirm route guard helper `requireWorkspaceMember(projectId)` exists; if not, coordinate with shell owners before merge

### A1: Define Design Management Types

- [ ] `src/features/design-management/types/enums.ts` — all enums per data-model §2.2 (`ArtifactLifecycle`, `ArtifactFormat`, `CoverageStatus`, `CoverageDeclaration`, `AiSummaryStatus`, `ChangeLogEntryType`)
- [ ] `src/features/design-management/types/catalog.ts` — `CatalogSummary`, `CatalogArtifactRow`, `CatalogFilter`, `CatalogSection` (grouped by project)
- [ ] `src/features/design-management/types/viewer.ts` — `ArtifactDetail`, `ArtifactHeader`, `ArtifactVersionRef`, `LinkedSpecRow`, `AiSummaryPayload`, `ChangeLogEntry`
- [ ] `src/features/design-management/types/traceability.ts` — `TraceabilityMatrixRow`, `TraceabilityCell`, `CoverageBucket`
- [ ] `src/features/design-management/types/requests.ts` — admin mutation request/response records (register, publish version, link specs, unlink, regenerate summary, retire)
- [ ] `src/features/design-management/types/aggregate.ts` — `CatalogAggregate`, `ViewerAggregate`, `TraceabilityAggregate`, `DesignManagementState`

### A2: Build Mock Data

- [ ] `src/features/design-management/mock/catalog.mock.ts` — seeds ~14 artifacts across 5 projects: mix of PUBLISHED / DRAFT / RETIRED, each artifact with 1–4 versions; includes one artifact with no linked specs (MISSING coverage) and one with STALE coverage for visual coverage
- [ ] `src/features/design-management/mock/catalogSummary.mock.ts` — totals + coverage bucket counts
- [ ] `src/features/design-management/mock/artifactDetail.mock.ts` — full detail for 3 canonical artifacts: one healthy (all OK), one mixed (partial/stale/missing), one draft (no AI summary yet)
- [ ] `src/features/design-management/mock/artifactVersions.mock.ts` — version history with author, timestamp, changelog note
- [ ] `src/features/design-management/mock/linkedSpecs.mock.ts` — links across all 5 coverage states
- [ ] `src/features/design-management/mock/aiSummary.mock.ts` — SUCCESS for 2 artifacts, PENDING for 1, FAILED with retry for 1
- [ ] `src/features/design-management/mock/changeLog.mock.ts` — 20 entries covering all 7 entry types (REGISTERED, VERSION_PUBLISHED, SPEC_LINKED, SPEC_UNLINKED, RETIRED, AI_SUMMARY_REGENERATED, LIFECYCLE_TRANSITIONED)
- [ ] `src/features/design-management/mock/traceabilityMatrix.mock.ts` — ~30 specs × ~14 artifacts; deliberately sparse with clustered gaps
- [ ] `src/features/design-management/mock/htmlPayloads/` — 3 canned Stitch HTML files (copy-pasted from `docs/standard-sdd/projects/control-tower/05-design/*.html`) served through the mock as `/artifacts/:id/raw`
- [ ] Mock command layer: `src/features/design-management/mock/commandLoop.ts` — simulates:
  - `prevVersionId` mismatch → `DM_STALE_VERSION` (5% injected)
  - `__PII_TRIGGER__` sentinel in uploaded HTML → `DM_PII_DETECTED`
  - Payload > 2 MiB → `DM_ARTIFACT_TOO_LARGE`
  - Invalid lifecycle transition (e.g. RETIRED → DRAFT) → `DM_INVALID_LIFECYCLE_TRANSITION`
  - AI regeneration under insufficient workspace autonomy → `DM_AI_AUTONOMY_INSUFFICIENT`
  - Non-designer / non-PM principal → `DM_ROLE_REQUIRED`
  - Cross-workspace artifact id → `DM_WORKSPACE_FORBIDDEN`

### A3: Build Reusable Primitives

- [ ] `CoverageStatusChip.vue` — OK / PARTIAL / STALE / MISSING / UNKNOWN with LED palette and tooltip text
- [ ] `LifecycleBadge.vue` — DRAFT / PUBLISHED / RETIRED with distinct typography per state
- [ ] `VersionPill.vue` — `v{n}` monospace pill with current-version marker and relative timestamp tooltip
- [ ] `ArtifactThumbnailTile.vue` — small square preview card with title, lifecycle, version, coverage summary, click → Viewer
- [ ] `PreviewFrame.vue` — CSP-hardened iframe host; sets `sandbox="allow-scripts allow-same-origin"`, `csp="..."`, `referrerpolicy="no-referrer"`; emits `load`, `error`, and `resize` events
- [ ] `PreviewFrameToolbar.vue` — width toggles (desktop / tablet / mobile), fit-to-width, open-raw-in-new-tab
- [ ] `AiSummaryPanel.vue` — renders SUCCESS payload (bullet list + focus areas + risks); shows PENDING spinner with ETA; shows FAILED state with Retry button (admins only)
- [ ] `ChangeLogRow.vue` — timestamp, actor, entry type, entity ref, inline detail, optional reason
- [ ] `TraceabilityMatrixCell.vue` — cell color-coded by coverage status with tooltip showing `(specId, artifactId, coversRevision vs latestRevision, declaredCoverage)`
- [ ] `LinkedSpecRow.vue` — spec id, title, state, declared coverage, computed coverage, last updated, deep-link chevron to Requirement slice
- [ ] `PiiWarningBanner.vue` — renders when mock command loop returns `DM_PII_DETECTED` with detected pattern kind (email / SSN / credit-card)
- [ ] `StaleVersionBanner.vue` — red banner when `prevVersionId` collision occurs; offers Refresh & Retry
- [ ] `HtmlSizeMeter.vue` — live byte counter for upload / paste flows; turns amber at 1.5 MiB, crimson at 2.0 MiB

### A4: Build Catalog View Cards

- [ ] `CatalogSummaryBarCard.vue` — aggregate across visible projects: total artifacts, DRAFT count, PUBLISHED count, RETIRED count, coverage-bucket pie (OK / PARTIAL / STALE / MISSING / UNKNOWN)
- [ ] `CatalogGridCard.vue` — grouped by project, each group a row of `<ArtifactThumbnailTile>`; sticky project header with artifact count and coverage bucket summary; supports filter by lifecycle / coverage / project / search string
- [ ] `CatalogFilterBar.vue` — filter chips (lifecycle, coverage status, project), text search, sort toggle (recent / alphabetical / coverage-worst-first)
- [ ] `CatalogEmptyState.vue` — 3 distinct copies: "No artifacts in workspace yet" / "No artifacts match your filters" / "No permission to view this workspace's artifacts"
- [ ] `CatalogErrorState.vue` — per-card retry UI

### A5: Build Viewer Cards

- [ ] `ViewerHeaderCard.vue` — artifact title + project badge + `<LifecycleBadge>` + current `<VersionPill>` + author list + last updated + admin actions menu (publish new version / retire / edit metadata)
- [ ] `ViewerPreviewCard.vue` — `<PreviewFrameToolbar>` + `<PreviewFrame>` bound to `/api/v1/design-management/artifacts/:id/raw?version={n}`; handles load / error / empty
- [ ] `ViewerVersionHistoryCard.vue` — reverse-chron version list with `<VersionPill>`, author, timestamp, changelog note; clicking a version scopes Preview + Linked Specs + AI Summary to that version
- [ ] `ViewerLinkedSpecsCard.vue` — `<LinkedSpecRow>` list with per-row `<CoverageStatusChip>`; admin action: Link Spec / Unlink
- [ ] `ViewerAiSummaryCard.vue` — `<AiSummaryPanel>` + Regenerate button (admin only, gated by `Workspace.aiAutonomyLevel >= SUPERVISED`)
- [ ] `ViewerChangeLogCard.vue` — reverse-chron `<ChangeLogRow>` list filtered to this artifact; filter by entry type
- [ ] `ViewerEmptyStates/` — 9 canonical empty-state copies per design doc §8 (no versions, no linked specs, AI summary pending, AI summary failed, preview unavailable, etc.)

### A6: Build Traceability View

- [ ] `TraceabilityMatrixCard.vue` — specs on Y, artifacts on X; cells render `<TraceabilityMatrixCell>`; supports sort by coverage-worst-first; hover highlights row+column
- [ ] `TraceabilityCoverageSummaryCard.vue` — big-number breakdown: OK / PARTIAL / STALE / MISSING / UNKNOWN counts and percentages
- [ ] `TraceabilityGapsCard.vue` — table of specs with no linked artifact; sortable by project / spec age
- [ ] `TraceabilityFiltersCard.vue` — filter by project, coverage status, lifecycle; export button (CSV, admin only in V1.1 — stubbed in V1)
- [ ] `TraceabilityEmptyState.vue` — "No specs in scope" / "No artifacts in scope" variants

### A7: Build Pinia Store and API Client

- [ ] `frontend/src/features/design-management/api/designManagementApi.ts` using `fetchJson<T>()` — one function per endpoint from API guide §2–§3 (17 total)
- [ ] `frontend/src/features/design-management/stores/designManagementStore.ts` with:
  - [ ] `initCatalog(filters?)`, `refreshCatalogCard(cardKey)`
  - [ ] `openArtifact(artifactId, versionId?)`, `refreshViewerCard(cardKey)`, `closeArtifact()`
  - [ ] `initTraceability(filters?)`, `refreshTraceabilityCard(cardKey)`
  - [ ] Admin actions: `registerArtifact`, `publishVersion`, `retireArtifact`, `linkSpecs`, `unlinkSpec`, `regenerateAiSummary`, `updateMetadata`
  - [ ] `prevVersionId` tracking: every admin mutation reads current version id, sends it, stores returned version on success, surfaces `DM_STALE_VERSION` as a refresh-prompt toast that refetches viewer aggregate
  - [ ] `reset()` on unmount
- [ ] Mock toggle: `import.meta.env.DEV && !import.meta.env.VITE_USE_BACKEND`
- [ ] Per-card retry preserves other cards' state
- [ ] Mutations update only the affected card's state + log entry (no full refetch of the entire viewer aggregate)

### A8: Build Top-Level Views

- [ ] `CatalogView.vue` — top-level route view for `/design-management`; mounts 4 catalog cards in 12-column grid per design §3.1; breadcrumb `Dashboard / Design Management`; AI panel scoped to catalog
- [ ] `ViewerView.vue` — top-level route view for `/design-management/artifacts/:id`; mounts 6 viewer cards in 12-column grid per design §3.2; breadcrumb `Dashboard / Design Management / {artifactTitle}`; watches `route.params.id` + `route.query.version` for deep-link switching
- [ ] `TraceabilityView.vue` — top-level route view for `/design-management/traceability`; mounts matrix + summary + gaps + filters; breadcrumb `Dashboard / Design Management / Traceability`
- [ ] `RawArtifactRoute.vue` (or plain redirect handler) — `/design-management/artifacts/:id/raw` must be served by backend as an iframe-ready HTML resource, not a Vue route; frontend only uses it as an `iframe.src` target

### A9: Routing and Navigation Wiring

- [ ] Add `/design-management` route → `CatalogView`
- [ ] Add `/design-management/artifacts/:id` route → `ViewerView` (with optional `?version=` query)
- [ ] Add `/design-management/traceability` route → `TraceabilityView`
- [ ] Backend-served route `/api/v1/design-management/artifacts/:id/raw?version=` is not a Vue route — never register it in the SPA router
- [ ] Remove Design Management `comingSoon` flag from shared shell nav config
- [ ] Deep-link: Catalog tile → Viewer at latest published version
- [ ] Deep-link: Viewer `<ViewerLinkedSpecsCard>` row → Requirement slice spec detail
- [ ] Deep-link: Traceability gap row → Requirement slice spec detail
- [ ] Deep-link: Traceability filled cell → Viewer at linked artifact+version
- [ ] Ensure keyboard navigation and ARIA labels (tab order per card, Enter to open artifacts, Esc to close modals, tooltip-exposed coverage states)

### A10: Admin Modals and Flows

- [ ] `RegisterArtifactModal.vue` — title, project, format, lifecycle, initial version HTML upload/paste, author list; validates PII pre-submit via `commandLoop.ts`; surfaces `<PiiWarningBanner>` / `<HtmlSizeMeter>`
- [ ] `PublishNewVersionModal.vue` — HTML upload/paste, changelog note, `prevVersionId` hidden field; on stale collision shows `<StaleVersionBanner>`
- [ ] `RetireArtifactModal.vue` — reason field (required), confirms lifecycle transition, bumps version
- [ ] `LinkSpecsModal.vue` — spec picker (typeahead against Requirement slice), declared coverage selector (FULL / PARTIAL), covers-revision dropdown
- [ ] `UnlinkSpecConfirmDialog.vue` — confirms unlink with reason
- [ ] `RegenerateAiSummaryDialog.vue` — confirms regeneration; shows current autonomy level; disables when below `SUPERVISED`
- [ ] All modals gated by `designAccessGuard.canAdmin(projectId)` — non-admin never sees the triggers

### A11: States and Error Isolation

- [ ] Per-card skeletons during load (all three views)
- [ ] Per-card error with retry
- [ ] Per-card empty states (distinct copy per card)
- [ ] Page-level error for access denial (`DM_WORKSPACE_FORBIDDEN`), invalid artifact id, or 404
- [ ] Mutation errors surfaced via toast + inline validation (PII, stale version, too large, invalid transition, autonomy gate)
- [ ] AI summary FAILED: admins see Retry; non-admins see `AI summary unavailable — pending regeneration by an admin`
- [ ] Preview iframe error: show `PreviewFrame error — open raw in new tab` with deep-link to raw endpoint
- [ ] Coverage status tooltip always explains WHY (e.g. "STALE: link covers spec revision 7; latest is 9")

### A12: Validate Design Management Frontend

- [ ] `npm run dev` — browses `/design-management`, `/design-management/artifacts/art-2041`, `/design-management/traceability`
- [ ] `npm run build` passes
- [ ] Visual spot-check against design §3 layouts and the Stitch HTML mocks
- [ ] Catalog → Viewer drill works; back-navigation preserves scroll + filters
- [ ] Version switch re-loads Preview + Linked Specs + AI Summary without full page reload
- [ ] Preview iframe renders the 3 canned Stitch HTML payloads correctly under the CSP; no console CSP violations against allowed origins (cdn.tailwindcss.com, fonts.googleapis.com, fonts.gstatic.com)
- [ ] Every admin mutation updates state locally; intentional stale-token injection surfaces `<StaleVersionBanner>`
- [ ] PII trigger (`__PII_TRIGGER__` in uploaded HTML) blocks registration with `<PiiWarningBanner>`
- [ ] Size > 2 MiB blocks registration via `<HtmlSizeMeter>`
- [ ] Traceability matrix filters + sort behave; gap list accurate
- [ ] Vitest unit tests pass for: primitives, store actions (including version fencing), mock commandLoop, coverage status tooltip derivation

### Phase A Definition of Done

- Catalog view renders 4 cards with realistic mock data covering all 5 coverage states and all 3 lifecycle states
- Viewer view renders 6 cards; Preview iframe loads canned Stitch HTML under CSP without violations
- Traceability view renders matrix + summary + gaps with worst-first sort option
- All admin mutations round-trip through mock `commandLoop.ts`; stale / PII / oversize / autonomy / role / workspace errors all reachable and surfaced correctly
- Coverage status for every link is derived at render time from `(coversRevision, latestRevision, state, declaredCoverage)` — never from a stored `status` field
- Per-card loading / error / empty states behave correctly
- Build passes; no console errors; mutations produce change log entries in the Viewer

---

## Phase B: Backend (Codex)

### B0: Shared Infrastructure (Prerequisite)

- [ ] Verify `com.sdlctower.shared.dto.ApiResponse` exists; reuse for Design Management endpoints
- [ ] Verify `com.sdlctower.shared.dto.SectionResultDto` exists; reuse for aggregate section envelopes
- [ ] Add `DESIGN_MANAGEMENT` constant to `com.sdlctower.shared.ApiConstants` with base path `/api/v1/design-management`
- [ ] Confirm `SpecRevisionLookup` facade from the Requirement slice returns `(specId, latestRevision, state, title, projectId)`; if missing, stub in `com.sdlctower.shared.integration.requirement` and open a follow-up ticket
- [ ] Confirm `WorkspaceAutonomyLookup` facade exists (returns current `aiAutonomyLevel` for a workspace); if missing, stub and note blocker
- [ ] Confirm `LineageEmitter` and `AuditLogEmitter` are available in shared

### B1: Create Design Management Package Structure

- [ ] Create `com.sdlctower.domain.designmanagement` package with:
  - `controller/` — `DesignManagementController` (one controller; Catalog + Viewer + Traceability + Admin + Raw)
  - `service/` — `CatalogService`, `ViewerService`, `TraceabilityService`, `ArtifactCommandService`, `SpecLinkCommandService`, `AiSummaryService`, `CoverageService`, `HtmlPayloadService`
  - `policy/` — `DesignAccessGuard` (read vs admin gating), `VersionFencingPolicy` (`prevVersionId` optimistic concurrency), `LifecyclePolicy` (DRAFT / PUBLISHED / RETIRED transitions), `AiAutonomyPolicy`
  - `projection/` — all read-path projections
  - `dto/` — records per data-model §4
  - `persistence/` — JPA entities + repositories
  - `events/` — change log publisher, AI summary lifecycle events, lineage emitter integration
  - `integration/` — adapters for `SpecRevisionLookup`, `WorkspaceAutonomyLookup`, AI skill invocation (`AiSkillClient`)
  - `security/` — `PiiScanner`, `HtmlSanitizer`
- [ ] No cross-cutting logic leaks into `controller/` — controller calls service, never projection/repository directly

### B2: Create Design Management DTOs

- [ ] All DTOs per data-model §4 as Java 21 records
- [ ] All enums per data-model §2.2 mirrored as Java enums with JSON codecs (`ArtifactLifecycle`, `ArtifactFormat`, `CoverageStatus`, `CoverageDeclaration`, `AiSummaryStatus`, `ChangeLogEntryType`)
- [ ] Command request records: `RegisterArtifactRequest`, `PublishVersionRequest`, `RetireArtifactRequest`, `UpdateMetadataRequest`, `LinkSpecsRequest`, `UnlinkSpecRequest`, `RegenerateAiSummaryRequest`
- [ ] Command response records include updated `currentVersionId` / `artifactId` as applicable
- [ ] Reuse `LinkDto`, `MemberRefDto`, `SpecRefDto` from shared
- [ ] `SectionResultDto<T>` wrapper per aggregate card
- [ ] `RawArtifactPayload` value type for the `/raw` endpoint — carries bytes + content type + cache headers

### B3: Implement Projections

- [ ] `CatalogSummaryProjection` — totals + coverage-bucket counts across artifacts the caller can read
- [ ] `CatalogGridProjection` — artifact rows grouped by project; coverage summary per artifact computed via `CoverageService` fan-out across the artifact's linked specs
- [ ] `ArtifactHeaderProjection` — artifact identity + lifecycle + current version + author list + last updated
- [ ] `ArtifactVersionHistoryProjection` — versions ordered DESC with author, timestamp, changelog note, size bytes
- [ ] `LinkedSpecsProjection` — `design_spec_link` rows joined with Requirement `SpecRevisionLookup`; `CoverageService` computes live status per row
- [ ] `AiSummaryProjection` — latest `design_ai_summary` row for the artifact+version+skillVersion tuple; returns PENDING if none exists yet and a regeneration request is in flight
- [ ] `ChangeLogProjection` — artifact-scoped entries, reverse-chron, filterable by entry type
- [ ] `TraceabilityMatrixProjection` — builds specs × artifacts matrix; batched Requirement facade call (one call per spec id set); `CoverageService.compute()` per cell
- [ ] `TraceabilityGapsProjection` — specs in scope with no `design_spec_link` row; efficiently computed via LEFT JOIN
- [ ] `TraceabilityCoverageSummaryProjection` — per-bucket counts and percentages

### B4: Implement Read Services

- [ ] `CatalogService.loadAggregate(filters)` with parallel fan-out over catalog projections, per-projection 500ms timeout → `SectionResultDto(data=null, error=...)` on failure
- [ ] `CatalogService.load{Summary,Grid,Filters}` for per-card endpoints
- [ ] `ViewerService.loadAggregate(artifactId, versionId?)` with parallel fan-out over 6 viewer projections
- [ ] `ViewerService.load{Header,VersionHistory,LinkedSpecs,AiSummary,ChangeLog}` for per-card endpoints
- [ ] `ViewerService.loadRawPayload(artifactId, versionId?)` — returns raw HTML bytes + content type; enforces `DesignAccessGuard.requireRead`; sets cache headers per API guide §5; never returns the payload if the artifact's lifecycle is RETIRED unless requester is admin
- [ ] `TraceabilityService.loadAggregate(filters)` with parallel fan-out over matrix/summary/gaps
- [ ] All read methods emit zero writes; safe to retry

### B5: Implement Command Services

- [ ] `ArtifactCommandService.register(request)` — validates workspace access; runs `PiiScanner.scan(html)` → `DM_PII_DETECTED` on match; enforces 2 MiB limit → `DM_ARTIFACT_TOO_LARGE`; creates `DesignArtifact` + initial `DesignArtifactVersion`; writes change log REGISTERED + VERSION_PUBLISHED; fires AI summary regenerate request asynchronously
- [ ] `ArtifactCommandService.publishVersion(artifactId, request)` — enforces `VersionFencingPolicy.check(artifactId, request.prevVersionId)` FIRST → `DM_STALE_VERSION` (409); PII scan; size enforcement; creates new `DesignArtifactVersion`; bumps `DesignArtifact.currentVersionId`; writes change log VERSION_PUBLISHED; queues AI summary regeneration
- [ ] `ArtifactCommandService.retire(artifactId, request)` — enforces `LifecyclePolicy.canTransition(current, RETIRED)`; requires non-empty reason; bumps lifecycle; writes change log RETIRED + LIFECYCLE_TRANSITIONED
- [ ] `ArtifactCommandService.updateMetadata(artifactId, request)` — title / authors / format metadata only; no version bump; writes change log METADATA_UPDATED (treat as `VERSION_PUBLISHED` sibling type — see data-model §2.2)
- [ ] `SpecLinkCommandService.linkSpecs(artifactId, request)` — for each spec, validates via `SpecRevisionLookup`; idempotent upsert into `design_spec_link` (unique `(artifact_id, spec_id)`); writes change log SPEC_LINKED per spec
- [ ] `SpecLinkCommandService.unlink(artifactId, specId, request)` — removes row; requires reason; writes change log SPEC_UNLINKED
- [ ] `AiSummaryService.regenerate(artifactId, request)` — enforces `AiAutonomyPolicy.requireAtLeast(workspaceId, SUPERVISED)` → `DM_AI_AUTONOMY_INSUFFICIENT`; invokes `AiSkillClient.summarize(versionPayload, linkedSpecs)`; persists `DesignAiSummary` keyed by `(artifactId, versionId, skillVersion)`; writes change log AI_SUMMARY_REGENERATED
- [ ] All command services enforce `DesignAccessGuard.requireAdmin(projectId, principal)` — role must be Designer or PM on that project → `DM_ROLE_REQUIRED`
- [ ] All command services enforce workspace membership → `DM_WORKSPACE_FORBIDDEN`
- [ ] Every successful mutation publishes a `DesignChangeLogEntry` via `DesignChangeLogPublisher` + a `LineageEvent` via shared `LineageEmitter` naming Design Management as the authoring domain

### B6: Implement CoverageService

- [ ] `CoverageService.compute(LinkedSpecRaw link, SpecRevisionInfo spec) → CoverageStatus`:

```java
public CoverageStatus compute(LinkedSpecRaw link, SpecRevisionInfo spec) {
  if (spec == null) return CoverageStatus.UNKNOWN;
  if (spec.state() == SpecState.ARCHIVED || spec.state() == SpecState.DELETED) return MISSING;
  if (link.declaredCoverage() == PARTIAL) return PARTIAL;
  if (link.coversRevision() < spec.latestRevision()) return STALE;
  return OK;
}
```

- [ ] `CoverageService.computeForArtifact(artifactId)` — batched: one call to `SpecRevisionLookup` for all linked spec ids; returns `Map<SpecId, CoverageStatus>`
- [ ] `CoverageService.computeForMatrix(artifactIds, specIds)` — batched two-axis; reuses the same `SpecRevisionInfo` map
- [ ] Never persist results — always compute at request time

### B7: Implement Policies and Security

- [ ] `DesignAccessGuard.requireRead(projectId, principal)` — workspace membership check
- [ ] `DesignAccessGuard.requireAdmin(projectId, principal)` — role is Designer or PM on that project
- [ ] `VersionFencingPolicy.check(artifactId, clientPrevVersionId)` — compares to `design_artifact.current_version_id`
- [ ] `LifecyclePolicy.canTransition(from, to)` — DRAFT→PUBLISHED, DRAFT→RETIRED, PUBLISHED→RETIRED; RETIRED is terminal; forbids RETIRED→PUBLISHED and RETIRED→DRAFT → `DM_INVALID_LIFECYCLE_TRANSITION`
- [ ] `AiAutonomyPolicy.requireAtLeast(workspaceId, level)` — looks up workspace autonomy, compares to required floor
- [ ] `PiiScanner` — regex patterns:
  - Email: `[A-Za-z0-9._%+\-]+@[A-Za-z0-9.\-]+\.[A-Za-z]{2,}`
  - US SSN: `\b\d{3}-\d{2}-\d{4}\b`
  - Credit card (Luhn-checked): `\b(?:\d[ -]*?){13,19}\b` with Luhn post-filter
  - Returns `PiiMatch(kind, sample)` list; any non-empty → reject
- [ ] `HtmlSanitizer` — strips `<script>` tags that reference off-allowlist origins; leaves allowlisted CDN tags alone; used during registration only (never mutates already-stored payloads)
- [ ] Exception classes: `DesignAccessDeniedException`, `InvalidLifecycleTransitionException`, `StaleVersionException`, `PiiDetectedException`, `ArtifactTooLargeException`, `AiAutonomyInsufficientException`, `ArtifactNotFoundException`
- [ ] `@ExceptionHandler` entries in `GlobalExceptionHandler` mapping each to its error code + HTTP status per API guide §4

### B8: Implement Controller

- [ ] `DesignManagementController` with endpoints per API guide §2–§3:
  - GET `/catalog` — aggregate
  - GET `/catalog/summary`, `/catalog/grid` — per-card
  - GET `/artifacts/{id}` — viewer aggregate (optional `?version=`)
  - GET `/artifacts/{id}/header`, `/versions`, `/linked-specs`, `/ai-summary`, `/change-log` — per-card
  - GET `/artifacts/{id}/raw` — raw payload; produces `text/html; charset=utf-8`; sets `Content-Security-Policy: default-src 'self' 'unsafe-inline' cdn.tailwindcss.com fonts.googleapis.com fonts.gstatic.com` and `X-Frame-Options: SAMEORIGIN`; `Cache-Control: private, max-age=300`
  - GET `/traceability` — aggregate
  - GET `/traceability/matrix`, `/gaps`, `/summary` — per-card
  - POST `/artifacts` — register
  - POST `/artifacts/{id}/versions` — publish version
  - POST `/artifacts/{id}/retire` — retire
  - PATCH `/artifacts/{id}` — update metadata
  - POST `/artifacts/{id}/spec-links` — link specs (bulk)
  - DELETE `/artifacts/{id}/spec-links/{specId}` — unlink
  - POST `/artifacts/{id}/ai-summary/regenerate` — regen
- [ ] `@Pattern` path validation on `artifactId` (`^art-[a-z0-9\-]+$`) and `specId` (`^spec-[a-z0-9\-]+$`)
- [ ] `@RequestBody @Valid` on all command endpoints
- [ ] Standard `ApiResponse<T>` envelope wrap via shared `ResponseBodyAdvice` or explicit `ApiResponse.ok(...)` calls (the `/raw` endpoint is an exception — it returns raw HTML, not an envelope)
- [ ] Mutation endpoints return `200 OK` with updated version/artifact state; not `204`

### B9: Create Flyway Migrations

- [ ] `V30__create_design_artifact.sql` — columns: `id` (PK, string), `project_id` (FK), `workspace_id` (FK), `title`, `format`, `lifecycle`, `current_version_id` (nullable FK — set after V31), `created_at`, `updated_at`; indexes on `(project_id, lifecycle)` and `(workspace_id, updated_at DESC)`
- [ ] `V31__create_design_artifact_version.sql` — `id` (PK), `artifact_id` (FK), `version_number` (INT), `html_payload` (BLOB/CLOB), `html_size_bytes` (BIGINT), `content_sha256` (CHAR(64)), `changelog_note`, `created_by`, `created_at`; `CHECK (html_size_bytes <= 2097152)`; indexes on `(artifact_id, version_number DESC)` and `(artifact_id, created_at DESC)`; add `FOREIGN KEY (current_version_id) REFERENCES design_artifact_version(id)` to `design_artifact`
- [ ] `V32__create_design_artifact_author.sql` — `artifact_id` + `member_id` composite PK; FKs; index on `member_id`
- [ ] `V33__create_design_spec_link.sql` — `id` (PK), `artifact_id` (FK), `spec_id`, `covers_revision` (INT), `declared_coverage` (enum FULL|PARTIAL), `linked_by`, `linked_at`, `unlink_reason` nullable (for soft-delete-via-audit); UNIQUE `(artifact_id, spec_id)` for idempotency; index on `(spec_id)` for reverse lookup
- [ ] `V34__create_design_ai_summary.sql` — `id` (PK), `artifact_id` (FK), `version_id` (FK), `skill_version`, `status` (PENDING|SUCCESS|FAILED), `payload_json` (CLOB), `error_json` nullable, `generated_at`; UNIQUE `(artifact_id, version_id, skill_version)` so the natural cache key rotates on version publish
- [ ] `V35__create_design_change_log.sql` — `id` (PK), `artifact_id` (FK), `entry_type`, `actor_id`, `version_id` nullable, `spec_id` nullable, `before_json` nullable, `after_json` nullable, `reason` nullable, `created_at`; index on `(artifact_id, created_at DESC)` and `(actor_id, created_at DESC)`
- [ ] `V36__seed_design_management_local.sql` — local-only seed: ~6 artifacts across 2 workspaces, ~10 versions, ~14 spec links exercising all 5 coverage states, ~3 AI summary rows (one PENDING, one SUCCESS, one FAILED), ~12 change log entries
- [ ] Verify all migrations run on H2 (local) without error
- [ ] Produce Oracle variant if DDL diverges (notably `BLOB` vs `CLOB`, `CHECK` constraint syntax, identity columns) and land under `db/migration/oracle/` per repo convention

### B10: Implement Entities and Repositories

- [ ] JPA entities: `DesignArtifact`, `DesignArtifactVersion`, `DesignArtifactAuthor` (`@IdClass`), `DesignSpecLink`, `DesignAiSummary`, `DesignChangeLogEntry`
- [ ] Map `html_payload` as `@Lob byte[]` with `FetchType.LAZY` — never eagerly loaded via catalog/viewer projections
- [ ] Repositories: one per entity with query methods:
  - `DesignArtifactRepository.findByWorkspaceIdIn(workspaceIds, Pageable)`
  - `DesignArtifactVersionRepository.findByArtifactIdOrderByVersionNumberDesc(artifactId)`
  - `DesignArtifactVersionRepository.findByArtifactIdAndId(artifactId, versionId)` — used by `/raw`
  - `DesignSpecLinkRepository.findByArtifactId(artifactId)`
  - `DesignSpecLinkRepository.findBySpecIdIn(specIds)` — for traceability matrix
  - `DesignAiSummaryRepository.findTopByArtifactIdAndVersionIdOrderByGeneratedAtDesc(artifactId, versionId)`
  - `DesignChangeLogEntryRepository.findByArtifactIdOrderByCreatedAtDesc(artifactId, Pageable)`
- [ ] Ensure all cross-slice reads (Requirement spec lookups) go through `SpecRevisionLookup` facade — no direct JPA association from Design Management to Requirement tables

### B11: Governance & Lineage Integration

- [ ] Every write path writes a `LineageEvent` identifying Design Management as the authoring domain (reuse shared `LineageEmitter`)
- [ ] Every write path writes an `AuditLogEntry` via shared `AuditLogEmitter` — mutation endpoints are permanently audited even if the artifact is later retired
- [ ] AI summary regeneration emits a `SkillInvocation` event via shared AI governance plumbing (tracks invoker, autonomy level at time of call, skill version, latency, outcome)
- [ ] Confirm Dashboard "recent design activity" tile (if present) reads from `design_change_log` via a cross-slice projection facade rather than the table directly

### B12: Backend Tests

- [ ] `DesignManagementControllerTest` (MockMvc):
  - Catalog aggregate happy path + per-projection error isolation
  - Viewer aggregate happy + 404 artifact + 403 workspace forbidden
  - `/raw` returns HTML with correct CSP and `X-Frame-Options` headers
  - `/raw` for RETIRED artifact returns 403 for non-admin, 200 for admin
  - Traceability aggregate + matrix / gaps / summary
  - Every mutation endpoint: happy + stale version + PII detected + too large + invalid transition + access denied + validation failure
- [ ] `ArtifactCommandServiceTest`: register flow (PII blocked, size blocked, happy), publishVersion (stale rejected, version bumped, AI regen queued), retire (lifecycle enforced), updateMetadata (no version bump)
- [ ] `SpecLinkCommandServiceTest`: bulk link idempotency; unlink with reason; unknown spec → facade returns empty → 400 `DM_SPEC_NOT_FOUND`
- [ ] `AiSummaryServiceTest`: autonomy gating; skill client retry + failure path → FAILED row persisted; cache key rotates on version publish
- [ ] `CoverageServiceTest`: every branch of compute() — UNKNOWN (missing spec), MISSING (archived/deleted), PARTIAL (declared), STALE (stale revision), OK (happy)
- [ ] `VersionFencingPolicyTest`: stale → 409
- [ ] `LifecyclePolicyTest`: every forbidden transition blocked
- [ ] `DesignAccessGuardTest`: read vs admin gating; cross-workspace blocked
- [ ] `AiAutonomyPolicyTest`: DISABLED / OBSERVATION / SUPERVISED / AUTONOMOUS gating
- [ ] `PiiScannerTest`: email / SSN / credit-card (Luhn-valid + Luhn-invalid) / clean HTML
- [ ] `HtmlSanitizerTest`: off-allowlist `<script>` stripped; allowlisted CDN tags preserved
- [ ] Repository tests against H2 for every entity + edge queries
- [ ] `FlywayMigrationIntegrationTest`: applies V30–V36 on a clean H2; verifies target schema (including the `html_size_bytes` CHECK constraint rejects >2 MiB inserts) + seed counts
- [ ] Golden-file test for API envelopes — compare response JSON to canonical examples in API guide §2–§3 (including the `/raw` response headers)

### B13: Connect Frontend to Backend

- [ ] Set `VITE_USE_BACKEND=true` in dev config
- [ ] Verify Vite proxy routes `/api/v1/design-management/*` to backend (including `/raw`, which the iframe hits directly — no envelope unwrap)
- [ ] Smoke: browse `/design-management` against live backend — 4 catalog cards hydrated from seed data
- [ ] Smoke: browse `/design-management/artifacts/{seedId}` — 6 viewer cards hydrated; Preview iframe loads `/raw` under CSP without violations
- [ ] Smoke: browse `/design-management/traceability` — matrix populated; gap list correct
- [ ] Smoke: execute one mutation of each kind (register / publish version / retire / link / unlink / regen AI) with a Designer account; verify change log entries appear and version/lifecycle updates on the viewer
- [ ] Smoke: force stale version (via second browser tab) → verify `<StaleVersionBanner>` + refetch
- [ ] Smoke: attempt each error path (PII HTML, 3 MiB HTML, non-designer user, cross-workspace id, RETIRED→PUBLISHED, AI regen under OBSERVATION autonomy) → verify correct error code + UI treatment
- [ ] Verify deep-links: Catalog tile → Viewer, Viewer linked spec → Requirement detail, Traceability gap → Requirement detail, Traceability filled cell → Viewer

### B14: Validate Full Stack

- [ ] `./mvnw verify` passes (all new tests + existing suites)
- [ ] `npm run build` passes
- [ ] End-to-end: register → publish v2 → link 2 specs (one FULL, one PARTIAL) → regenerate AI → retire — each step produces a change log entry visible in `<ViewerChangeLogCard>`; AI summary cache key rotates on v2 publish
- [ ] Traceability matrix reflects the new links immediately (no stale projection cache)
- [ ] Coverage status transitions correctly when the linked spec's `latestRevision` advances upstream (bump Requirement seed revision; verify STALE appears)
- [ ] Requirement slice still renders correctly (no regressions from `SpecRevisionLookup` read load)
- [ ] CSP header on `/raw` blocks inline scripts that reference off-allowlist origins when tested manually in browser devtools
- [ ] Audit log contains entries for every mutation; Lineage graph shows Design Management as authoring domain for each entry

### Phase B Definition of Done

- All 17 endpoints return correct envelopes for happy path (plus `/raw` returning HTML with correct headers)
- Every mutation enforces version fencing, lifecycle transitions, access gating, PII scan, size limit, and AI autonomy policy
- Per-projection timeout degrades to section-level error only (never page-level failure for read)
- Flyway V30–V36 migrations run cleanly on H2
- Frontend works in live mode (`VITE_USE_BACKEND=true`)
- Backend + frontend builds pass; golden-file API tests match API guide exactly
- Coverage status is always computed at request time — no persisted `coverage_status` column
- AI summary cache keyed by `(artifactId, versionId, skillVersion)` rotates correctly on version publish
- `/raw` endpoint serves HTML with the specified CSP + `X-Frame-Options`; iframe renders canned Stitch mocks without violations
- Write authority for DesignArtifact / DesignArtifactVersion / DesignSpecLink / DesignAiSummary is demonstrably owned by Design Management (no other slice mutates these tables)

---

## Dependency Plan

### Critical Path

```
A0 (shared infra) → A1 (types) → A2 (mocks) → A3 (primitives)
   → A4 (catalog cards in parallel) + A5 (viewer cards in parallel) + A6 (traceability cards in parallel)
   → A7 (store/API) → A8 (views) → A9 (routing) → A10 (admin modals) → A11 (states) → A12 (validate)
        │
        ↓
B0 (shared backend) → B1 (package) → B2 (DTOs)
   → B3 (projections in parallel) → B6 (CoverageService) → B4 (read services)
   → B7 (policies + PII + sanitizer) → B5 (command services)
   → B8 (controller) → B9 (migrations) → B10 (entities/repos)
   → B11 (governance/lineage) → B12 (tests) → B13 (connect) → B14 (validate)
```

### Parallel Workstreams

- A4 (catalog cards), A5 (viewer cards), A6 (traceability cards) can be built in parallel by multiple contributors
- A10 admin modals can be built in parallel with A4/A5/A6 once A3 primitives land
- B3 projections can be built in parallel once DTOs are defined
- B5 command services can be split per capability (Artifact / SpecLink / AiSummary)
- Frontend Phase A can proceed independently of backend work up through A12
- Flyway migrations (B9) can be drafted in parallel with entity work (B10) but must land before entity tests pass
- `PiiScanner` / `HtmlSanitizer` (B7) can be built in parallel with everything else — pure utilities with self-contained tests

---

## Risks / Blockers

| Risk                                                                             | Mitigation                                                                                                                                                                                    |
| -------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `SpecRevisionLookup` facade not yet implemented in Requirement slice           | Stub in `com.sdlctower.shared.integration.requirement` returning static lookup; flag as upstream blocker; document cutover plan                                                             |
| CSP on `/raw` too restrictive, breaks Stitch HTML rendering                    | Seed data includes the exact HTML files from `docs/standard-sdd/projects/control-tower/05-design/*.html`; CI verifies they render under the configured CSP without violations; extending allowlist requires platform review     |
| CSP too permissive allowing third-party origins to exfiltrate                    | Allowlist is locked to the three CDN origins the internal mocks currently use (cdn.tailwindcss.com, fonts.googleapis.com, fonts.gstatic.com); adding a new origin is a platform-review change |
| 2 MiB limit bypassed if `html_size_bytes` is client-reported                   | Server recomputes size from the uploaded bytes before the CHECK constraint fires; client value is advisory only                                                                               |
| PII scanner false positives on legitimate HTML (e.g. mailto links in footer)     | PII scanner takes the sample + offset, returns structured `PiiMatch`; admin sees WHICH pattern matched WHERE; explicit override requires a platform approval (out of scope for V1)          |
| AI skill latency makes regeneration UI feel stuck                                | AI summary persists as PENDING immediately; frontend polls viewer aggregate at 3s cadence until status changes; timeout at 60s → FAILED with Retry                                           |
| Coverage drift if `SpecRevisionLookup` caches aggressively                     | `SpecRevisionLookup` is a request-scoped read; no cache within a request; per-projection timeout 500ms protects user-facing latency                                                         |
| Cross-slice churn when Requirement slice changes its spec revision semantics     | Coverage derivation is isolated in `CoverageService.compute`; any semantic change requires updating this one method + its unit tests                                                        |
| Oracle BLOB handling differs from H2                                             | Oracle-variant migration planned under `db/migration/oracle/`; verified on Oracle-in-Docker before deployment                                                                               |
| Shell-level nav for Design Management still feature-flagged in some environments | A9 removes `comingSoon`; coordinate with shell owners before merge                                                                                                                          |
| Admin mutations bypass workspace isolation when artifactId is spoofed            | `DesignAccessGuard.requireAdmin` always resolves `projectId → workspaceId` from the artifact row itself; never trusts the request body's workspace id                                    |

---

## Open Questions

- AI summary auto-regeneration on version publish — synchronous (block publish until AI ready) or fire-and-forget? Default: fire-and-forget; publish returns immediately with AI status PENDING.
- Should RETIRED artifacts be hidden from non-admin users in Catalog by default? Default: yes, hide behind a "Show retired" admin toggle.
- Traceability CSV export scope — V1 stub only; V1.1 will add export? Default: V1 button disabled with tooltip; implementation deferred.
- Version deletion — permitted at all? Default: NO in V1; only RETIRE. Hard-delete (with governance gate) deferred to V2.
- PII allowlist overrides (e.g. test fixtures that legitimately contain demo SSNs) — how to permit? Default: not supported in V1; sample data uses clearly synthetic patterns.
- Preview width breakpoint defaults — use shell's device breakpoints or dedicated design-management breakpoints? Default: reuse shell breakpoints (desktop 1280+, tablet 1024–1279, mobile <1024).
- HTML sanitization aggressiveness — mutate on upload or verify-only? Default: verify-only with hard-reject on off-allowlist `<script>`; never mutate stored bytes (preserves the designer's intent verbatim).
- Who can regenerate AI summary — any workspace Designer or only project-PM? Default: any Designer or PM on the owning project; gated additionally by workspace autonomy level.
