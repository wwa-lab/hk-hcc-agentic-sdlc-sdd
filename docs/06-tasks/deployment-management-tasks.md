# Deployment Management Tasks

## Objective

Deliver the Deployment Management slice end-to-end in two phases: Phase A (frontend with mocks, Codex) and Phase B (backend wiring, Codex). V1 scope is a **read-only observability viewer** over deployments executed by **Jenkins**, with six views — Catalog, Application Detail, Release Detail, Deploy Detail, Environment Detail, and Story ↔ Commit ↔ Build ↔ Deploy Traceability — plus two AI surfaces (release notes / deploy summary). No triggering, approving, promoting, cancelling, or rolling back Jenkins builds from the Control Tower in V1, per the scope decisions recorded in `deployment-management-architecture.md` §Decisions (D1–D19). Implementation must conform to the 8 upstream SDD documents.

## Implementation Strategy

- **Frontend-first (Phase A):** mock-driven, no backend dependency. Ship six browsable views — Catalog, Application, Release, Deploy, Environment, Traceability — each assembled from independently mockable, independently retryable cards. AI regenerate is the only mutation; it round-trips through a mock command layer that simulates `DP_AI_UNAVAILABLE` (5%), `DP_JENKINS_UNREACHABLE` (2%), `DP_RELEASE_NOTES_EVIDENCE_MISMATCH` (3%), `DP_BUILD_ARTIFACT_PENDING` (2%), and `DP_RELEASE_UNRESOLVED` (1%).
- **Backend (Phase B):** implement projections, command services, controllers, Jenkins ingestion (webhook receiver + outbox + dispatcher), backfill, resync, entities, and Flyway migrations V60–V67. Swap frontend from mock mode to live API by toggling `VITE_USE_BACKEND=true`. No `ddl-auto`.
- **Per-card isolation:** each card on each view is independently developable, mockable, and testable. Per-projection 500ms timeout degrades to `SectionResult<T>(data=null, error=...)` — never a page-level failure.
- **Net-new entities only:** Deployment Management introduces 12 net-new tables per data-model §2; `Project`, `Workspace`, `Member`, `Story`, `BuildArtifact`, `Commit`, `CommitStoryLink` are reused read-only via existing cross-slice facades (Code & Build for artifact/commit/story-link; Requirement for story lookup).
- **Observability-first:** no triggering, no config mutation, no Jenkins credentials management in V1. AI regenerate is the only write path reachable from the UI. All state changes observed from Jenkins arrive via Notification-Plugin webhooks; backfill + nightly resync are the recovery paths.
- **Release composition resolved read-side:** a `release` row stores `release_version` + `build_artifact_slice_id` + `build_artifact_id` only. The list of commits and linked stories is resolved at query time via `CodeBuildFacade` — **no persisted story↔deploy link table**. Code & Build remains the single source of truth for Story↔Commit.
- **Release row lazy creation:** on first `job.started` with a given `RELEASE_VERSION` on a given application, a `release` row is upserted (state=PREPARED). If `RELEASE_VERSION` is absent, the deploy is staged in the outbox as `DP_RELEASE_UNRESOLVED` and retried once the Jenkins build metadata becomes available.
- **Dual-signal rollback detection:** `trigger=ROLLBACK` from Jenkins OR `releaseVersion < priorSuccessfulDeployReleaseVersion` on the same environment. The firing signal is persisted as `rollback_detection_signal` on the deploy row for audit.
- **Shared primitives:** live under `src/shared/` / `com.sdlctower.shared` — Deployment Management does not fork shell-level behavior.

## Traceability

- Requirements: [deployment-management-requirements.md](../01-requirements/deployment-management-requirements.md)
- Stories: [deployment-management-stories.md](../02-user-stories/deployment-management-stories.md)
- Spec: [deployment-management-spec.md](../03-spec/deployment-management-spec.md)
- Architecture: [deployment-management-architecture.md](../04-architecture/deployment-management-architecture.md)
- Data Flow: [deployment-management-data-flow.md](../04-architecture/deployment-management-data-flow.md)
- Data Model: [deployment-management-data-model.md](../04-architecture/deployment-management-data-model.md)
- Design: [deployment-management-design.md](../05-design/deployment-management-design.md)
- API Guide: [deployment-management-API_IMPLEMENTATION_GUIDE.md](../05-design/contracts/deployment-management-API_IMPLEMENTATION_GUIDE.md)

## Planning Assumptions

- `shared-app-shell`, `dashboard`, `requirement`, `incident`, `team-space`, `project-space`, `project-management`, `design-management`, `code-build-management` slices are merged and operational.
- Shared shell primitives (`SectionResult<T>`, `ApiResponse<T>`, `fetchJson<T>`, `LineageBadge.vue`, `shellUiStore.setBreadcrumbs`, `shellUiStore.setAiPanelContent`, route guard `requireWorkspaceMember(projectId)`) already exist and are reused as-is.
- The Code & Build slice exposes `CodeBuildFacade` returning `(buildArtifactId, sourceRunId, headSha, commits[], linkedStories[])` keyed by `(applicationId, releaseVersion)` and `(releaseId)`. If not yet available, stub in `com.sdlctower.shared.integration.codebuild` and document as an upstream blocker.
- The Requirement slice exposes a read-only `StoryLookup` facade returning `(storyId, title, projectId, state)` — already in use by Code & Build; reused here.
- The Incident slice accepts a deep-link `context` object `{ deployId, releaseVersion, environmentName, failingStageId, summary }` — deep-link contract must be verified before Phase B close.
- The Dashboard and Project Management slices expect a cross-slice facade `DeploymentHealthFacade` returning per-project deploy health (Deployment Frequency, CFR, MTTR, last deploy) sourced from Deployment Management; ship the facade in Phase B.
- `WorkspaceAutonomyLookup` facade is reused; AI regenerate gated by `aiAutonomyLevel >= SUPERVISED`.
- H2 is the local dev DB; Oracle is the deployed DB. Flyway migrations V60–V67 must run on both (split per-dialect if DDL diverges — notably `CLOB` for `ai_release_notes.body_markdown`, the `approval_event.rationale_cipher CLOB`, and the `uq_deploy_jenkins` unique constraint).
- Java 21 + Spring Boot 3.x; Vue 3 + Vite + Pinia + TypeScript.
- No localStorage / sessionStorage (per shared shell rules).
- Base path for all endpoints: `/api/v1/deployment-management` (per API guide §1).
- Mock toggle pattern follows repo convention: `import.meta.env.DEV && !import.meta.env.VITE_USE_BACKEND`.
- Access: all six read views are visible to any workspace member on the owning project. AI regenerate requires PM or Tech Lead role. Approval `rationale` and approver display-name are role-gated to `deployment:view-approvals` (PM + Tech Lead + Release Manager by default).
- Jenkins auth: per-workspace `JenkinsInstance` row carries a workspace-scoped webhook secret and a read-only API token (used only on backfill paths). Secrets stored encrypted at rest.
- Webhook signature header: `X-Jenkins-Signature` (HMAC-SHA-256 against raw request body, secret resolved from `X-Jenkins-Instance` header); constant-time compare (D3 of architecture).
- Supported Jenkins event types for V1: `job.started`, `job.completed`, `stage.completed`, `input.approved`, `input.rejected`, `input.timedOut`, `build.promoted` (logged), `build.deleted` (soft-hide).
- Ingestion is **outbox-backed** — webhook receiver verifies the signature synchronously, persists to `deployment_ingestion_outbox`, and returns 202 immediately; the dispatcher worker parses and projects (sync-verify, async-dispatch).
- Jenkins freshness SLO: 45s P95 from Jenkins timestamp to DB write for happy-path events. If exceeded, `FreshnessIndicator` renders `Degraded` / `Stale` tier and the UI shows a banner.
- Release version comparator is semver-ish: dot-segmented numeric, with lexicographic fallback for non-numeric segments. Ties break on `created_at` on the `release` row.
- AI release notes retention: 365 days (longer than CB's 90d) — compliance-aligned per data-model §6.
- AI deployment summary is **skipped** (not failed) when `change-summary.commitCount == 0` (same release re-deployed or a rollback to a known version).
- Approval rationale redaction applies to both the `rationale` field and the `approver` display name when principal lacks `deployment:view-approvals`.
- Log redaction patterns (applied at ingestion on `deploy_stage.log_excerpt.text`): AWS access key id, GitHub tokens, generic `Bearer …`, and Jenkins token patterns `(?i)token[= ]+[A-Za-z0-9]{20,}`.

---

## Phase A: Frontend (Claude Code)

### A0: Shared Infrastructure Preparation

- [ ] Reuse `src/shared/types/section.ts` for `SectionResult<T>`
- [ ] Reuse `src/shared/api/client.ts` for `fetchJson<T>`
- [ ] Reuse `src/shared/components/LineageBadge.vue` for provenance badges
- [ ] Reuse `shellUiStore.setBreadcrumbs(...)` + `shellUiStore.setAiPanelContent(...)`
- [ ] Confirm Tactical Command tokens already exported; add only if absent: `--dp-state-pending`, `--dp-state-in-progress`, `--dp-state-succeeded`, `--dp-state-failed`, `--dp-state-cancelled`, `--dp-state-rolled-back`, `--dp-env-dev`, `--dp-env-qa`, `--dp-env-staging`, `--dp-env-prod`, `--dp-env-canary`, `--dp-trigger-manual`, `--dp-trigger-scheduled`, `--dp-trigger-webhook`, `--dp-trigger-rollback`, `--dp-freshness-fresh`, `--dp-freshness-degraded`, `--dp-freshness-stale`, `--dp-release-prepared`, `--dp-release-deployed`, `--dp-release-superseded`, `--dp-release-abandoned`
- [ ] Confirm route guard helper `requireWorkspaceMember(projectId)` exists; if not, coordinate with shell owners before merge1

### A1: Define Deployment Management Types

- [ ] `src/features/deployment-management/types/enums.ts` — all enums per data-model §2.2 (`DeployState`, `DeployTrigger`, `DeployStageState`, `ReleaseState`, `EnvironmentKind`, `ApprovalEventState`, `AiRowStatus`, `RollbackDetectionSignal`, `ChangeLogEntryType`)
- [ ] `src/features/deployment-management/types/catalog.ts` — `CatalogSummary`, `CatalogApplicationRow`, `CatalogFilters`, `CatalogSection` (grouped by project)
- [ ] `src/features/deployment-management/types/application.ts` — `ApplicationHeader`, `ApplicationEnvironmentRow`, `RecentReleaseRow`, `RecentDeployRow`, `ApplicationMetrics`, `ApplicationAiInsights`
- [ ] `src/features/deployment-management/types/release.ts` — `ReleaseHeader`, `ReleaseComposition`, `ReleaseDeployRow`, `ReleaseLinkedStoryRow`, `AiReleaseNotesPayload`
- [ ] `src/features/deployment-management/types/deploy.ts` — `DeployHeader`, `DeployStageRow`, `DeployApprovalRow`, `DeployChangeSummary`, `AiDeploymentSummaryPayload`
- [ ] `src/features/deployment-management/types/environment.ts` — `EnvironmentHeader`, `EnvironmentCurrent`, `EnvironmentTimelineRow`, `EnvironmentStability`
- [ ] `src/features/deployment-management/types/traceability.ts` — `TraceabilityStoryChip`, `TraceabilityReleaseRow`, `TraceabilityDeployRow`
- [ ] `src/features/deployment-management/types/requests.ts` — mutation request/response records for `regenerateWorkspaceSummary`, `regenerateReleaseNotes`, `regenerateDeploySummary`
- [ ] `src/features/deployment-management/types/aggregate.ts` — `CatalogAggregate`, `ApplicationDetailAggregate`, `ReleaseDetailAggregate`, `DeployDetailAggregate`, `EnvironmentDetailAggregate`, `TraceabilityAggregate`, `DeploymentManagementState`

### A2: Build Mock Data

- [ ] `src/features/deployment-management/mock/catalog.mock.ts` — seeds ~14 applications across 4 projects: each has 4 environments (DEV/QA/STAGING/PROD); mix of current deploy states; at least one application with a ROLLED_BACK on PROD; at least one application with an IN_PROGRESS deploy
- [ ] `src/features/deployment-management/mock/catalogSummary.mock.ts` — totals + state bucket counts + DF/CFR/MTTR + env distribution
- [ ] `src/features/deployment-management/mock/applicationDetail.mock.ts` — full detail for 3 canonical applications: one healthy (30d clean), one flaky (2 rollbacks in 30d), one new (only dev/qa deploys, no prod yet)
- [ ] `src/features/deployment-management/mock/releaseDetail.mock.ts` — 4 releases: one DEPLOYED with AI notes SUCCESS + evidence intact, one DEPLOYED with notes EVIDENCE_MISMATCH (evidence references a fabricated commit), one PREPARED (no deploys yet), one SUPERSEDED
- [ ] `src/features/deployment-management/mock/deployDetail.mock.ts` — 5 deploys: one SUCCEEDED prod, one FAILED prod at smoke stage, one ROLLED_BACK prod (rollback_detection_signal=`trigger=rollback`), one ROLLED_BACK staging (signal=`version-older-than-prior`), one IN_PROGRESS dev with stages streaming
- [ ] `src/features/deployment-management/mock/environmentDetail.mock.ts` — 4 environments: DEV with weekly cadence, PROD with one rollback last week, STAGING clean, QA with mixed outcomes
- [ ] `src/features/deployment-management/mock/approvals.mock.ts` — canned approval events: APPROVED (rationale present), REJECTED (rationale present), TIMED_OUT (no rationale), PROMPTED (no resolver yet); one pair with `rationale` marked for role-gated redaction
- [ ] `src/features/deployment-management/mock/aiReleaseNotes.mock.ts` — SUCCESS, PENDING, FAILED (retry), EVIDENCE_MISMATCH (banner + locked content)
- [ ] `src/features/deployment-management/mock/aiDeploymentSummary.mock.ts` — SUCCESS, SKIPPED (no meaningful change), FAILED
- [ ] `src/features/deployment-management/mock/traceability.mock.ts` — ~12 stories with known story ids spanning multiple releases/deploys; ~3 stories with no deploys yet (empty); one story that spans 2 applications via shared library commit
- [ ] `src/features/deployment-management/mock/webhookFixtures/` — canned Jenkins Notification-Plugin bodies per event type (`job.started`, `job.completed` SUCCESS/FAILURE/UNSTABLE/ABORTED, `stage.completed` SUCCESS/FAILURE, `input.approved`, `input.rejected`, `input.timedOut`, `build.promoted`, `build.deleted`); used by the mock command loop for round-trip ingestion simulation
- [ ] Mock command layer: `src/features/deployment-management/mock/commandLoop.ts` — simulates:
  - `DP_AI_UNAVAILABLE` (5% injected) on AI regenerate
  - `DP_JENKINS_UNREACHABLE` (2% injected) on backfill-triggered reloads
  - `DP_RELEASE_NOTES_EVIDENCE_MISMATCH` (3% injected) on notes regeneration
  - `DP_BUILD_ARTIFACT_PENDING` (2% injected) on fresh release view
  - `DP_RELEASE_UNRESOLVED` (1%) when `RELEASE_VERSION` is missing
  - `DP_AI_AUTONOMY_INSUFFICIENT` when workspace autonomy < SUPERVISED
  - `DP_ROLE_REQUIRED` for non-PM / non-Tech-Lead invokers
  - `DP_WORKSPACE_FORBIDDEN` for cross-workspace entity ids
  - `DP_STORY_NOT_FOUND` surfaced only in Traceability view, never blocking
  - `DP_INGEST_SIGNATURE_INVALID` when signature is tampered in `__SIGNATURE_TRIGGER__` fixture
  - IN_PROGRESS → SUCCEEDED / FAILED transitions on a 15-second mock clock for the in-progress deploy fixture

### A3: Build Reusable Primitives

- [ ] `DeployStateBadge.vue` — PENDING / IN_PROGRESS / SUCCEEDED / FAILED / CANCELLED / ROLLED_BACK with shape + color + text; running state animates
- [ ] `DeployTriggerChip.vue` — MANUAL / SCHEDULED / WEBHOOK / ROLLBACK with tooltip; actor display redacts to `(redacted)` when principal lacks role
- [ ] `EnvironmentChip.vue` — DEV / QA / STAGING / PROD / CANARY; PROD gets a solid border
- [ ] `ReleaseStateBadge.vue` — PREPARED / DEPLOYED / SUPERSEDED / ABANDONED
- [ ] `StageConclusionChip.vue` — SUCCESS / UNSTABLE / FAILURE / ABORTED / SKIPPED / IN_PROGRESS (UNSTABLE = amber)
- [ ] `StoryChip.vue` (reused from Code & Build if shared; otherwise re-implement against shared tokens) — VERIFIED / UNVERIFIED / UNKNOWN_STORY
- [ ] `DurationPill.vue` — humanised duration (`3m 12s`) with absolute-timestamp tooltip; renders `—` when null
- [ ] `FreshnessIndicator.vue` — Fresh (≤45s) / Degraded (≤5m) / Stale (>5m); exposes tier via `aria-label`
- [ ] `JenkinsLinkOut.vue` — `target=_blank`, `rel="noopener noreferrer"`, Jenkins crown icon, deep-link chevron
- [ ] `ReleaseVersionPill.vue` — monospace; hover reveals the full build artifact id + Jenkins job name
- [ ] `HealthLed.vue` (reused) — GREEN / AMBER / RED / UNKNOWN
- [ ] `AiRowStatusBanner.vue` (reused) — PENDING / FAILED / STALE / SUPERSEDED / EVIDENCE_MISMATCH variants
- [ ] `RedactedLogExcerpt.vue` — monospace block, "redacted" watermark
- [ ] `RateLimitBanner.vue` (reused) — sticky top-of-card when `DP_RATE_LIMITED` returns
- [ ] `JenkinsStaleBanner.vue` — shown when the application / release / deploy's `lastIngestedAt` is > 5 minutes stale AND a non-terminal state is in play
- [ ] `ReleaseNotesMarkdown.vue` — sanitized markdown renderer (no script/style), with evidence footer linking out to commits and stories

### A4: Build Catalog View Cards

- [ ] `CatalogSummaryBarCard.vue` — aggregate across visible projects: application count, deploys last 7d, state buckets, DF / CFR / MTTR, env distribution
- [ ] `CatalogGridCard.vue` — grouped by project; each group a row of application tiles; each tile shows per-env pills (DEV/QA/STAGING/PROD) with current state LEDs; sticky project header
- [ ] `CatalogFilterBar.vue` — filters by project, environment kind, deploy state, trigger, search string, time window; emits `change` debounced 250ms
- [ ] `CatalogAiSummaryCard.vue` — sticky on desktop; autonomy-gated; Regenerate button admin-only + rate-limited 1/min

### A5: Build Application Detail View Cards

- [ ] `ApplicationHeaderCard.vue` — app header + Jenkins job full name + instance + current version per env
- [ ] `ApplicationEnvironmentsCard.vue` — environment strip with current release, deploy state, last-deployed-at
- [ ] `ApplicationRecentReleasesCard.vue` — paged 20; click → Release Detail
- [ ] `ApplicationRecentDeploysCard.vue` — paged 20; click → Deploy Detail
- [ ] `ApplicationMetricsCard.vue` — 30-day window: DF, CFR, MTTR, mean lead time; per-env breakdown
- [ ] `ApplicationAiInsightsCard.vue` — top risks + notable recent deploys; autonomy-gated

### A6: Build Release Detail View Cards

- [ ] `ReleaseHeaderCard.vue` — release version, state, created-at, owning application, build artifact pill
- [ ] `ReleaseCompositionCard.vue` — resolved build artifact → commits list (capped at 100 with `capNotice`); first and last SHA
- [ ] `ReleaseDeploysCard.vue` — all deploys of this release across envs; grouped by env
- [ ] `ReleaseLinkedStoriesCard.vue` — derived read-side from commits; shows `capNotice` when >100 commits
- [ ] `ReleaseAiNotesCard.vue` — Release Notes markdown (SUCCESS); evidence-mismatch banner + locked content on mismatch; PENDING polling; FAILED retry for admin

### A7: Build Deploy Detail View Cards

- [ ] `DeployHeaderCard.vue` — application, environment, release, state, trigger chip, rollback marker, Jenkins build link, `rollbackDetectionSignal` surfaced as a tooltip
- [ ] `DeployStagesCard.vue` — Jenkins pipeline stages with durations and conclusions; IN_PROGRESS polling every 5s while view is open
- [ ] `DeployApprovalsCard.vue` — approval events (PROMPTED / APPROVED / REJECTED / TIMED_OUT) with rationale; redacts to `(redacted)` for non-privileged role
- [ ] `DeployChangeSummaryCard.vue` — commits delta vs prior successful deploy on same env; linked story ids
- [ ] `DeployAiSummaryCard.vue` — per-deploy summary; autonomy-gated; skipped-state rendering when no meaningful change

### A8: Build Environment Detail View Cards

- [ ] `EnvironmentHeaderCard.vue` — env name, kind, current live release, last stable release
- [ ] `EnvironmentCurrentCard.vue` — currently-serving deploy summary
- [ ] `EnvironmentTimelineCard.vue` — chronological deploys on this env (paged 50)
- [ ] `EnvironmentStabilityCard.vue` — 30-day CFR, MTTR, last failure, last rollback

### A9: Build Traceability View Cards

- [ ] `TraceabilityInputCard.vue` — typeahead against recent verified story ids (via Code & Build facade proxy); keyboard-navigable
- [ ] `TraceabilityReleasesCard.vue` — releases that include any commit linked to the story; grouped by application
- [ ] `TraceabilityDeploysCard.vue` — all deploys of those releases, grouped by env; `capNotice` when >100

### A10: Build Pinia Store + API Client

- [ ] `stores/deploymentStore.ts` — state per design doc §6; actions: `initCatalog`, `refreshCatalogCard`, `openApplication/Release/Deploy/Environment`, `refreshXCard`, `closeX`, `lookupStory`, `regenerateWorkspaceSummary`, `regenerateReleaseNotes`, `regenerateDeploySummary`, `startAiPolling`, `stopAiPolling`, `startDeployPolling`, `stopDeployPolling`, `reset`
- [ ] `api/deploymentApi.ts` — per API guide §6; `deploymentApi = USE_BACKEND ? liveClient : mockClient`
- [ ] Poll lifecycle: 3s → 10s backoff for AI PENDING rows, cap 5m; 5s tick for IN_PROGRESS deploys while view is open; tear down in `reset()` and view `onBeforeUnmount`

### A11: Assemble Views + Routing

- [ ] `views/CatalogView.vue`, `ApplicationDetailView.vue`, `ReleaseDetailView.vue`, `DeployDetailView.vue`, `EnvironmentDetailView.vue`, `TraceabilityView.vue`
- [ ] `router.ts` entries per design doc §7 (order matters — `/deployment/traceability` before wildcard paths)
- [ ] All routes guarded by `requireWorkspaceMember(resolveWorkspaceId(to))`
- [ ] Shell nav config: remove `comingSoon` for Deployment

### A12: Empty / Error / Loading States

- [ ] Every card wires skeleton + error + empty variants per design doc §8
- [ ] Canonical copies used verbatim
- [ ] Per-card isolation: a failure in one card never blanks neighbors
- [ ] `JenkinsStaleBanner` shown when `lastIngestedAt > 5m` and deploy is non-terminal

### A13: Frontend Validation

- [ ] Unit tests for every primitive + every store action (Vitest)
- [ ] Component tests for every card (`@vue/test-utils`): loading / success / error / empty / stale
- [ ] BLOCKER role-gated redaction (approvals) test passes for both privileged and non-privileged principals
- [ ] AI PENDING polling test passes
- [ ] IN_PROGRESS deploy polling test passes
- [ ] Evidence-mismatch banner test passes
- [ ] Rate-limit banner test passes
- [ ] Accessibility: axe-core smoke pass on each view with no serious/critical issues; keyboard tab order verified

### Phase A Definition of Done

- All six views render in mock mode; every card hydrates from mock; every error code is reachable via mock command-loop
- IN_PROGRESS deploys stream stage updates via polling
- AI PENDING rows poll until SUCCESS / FAILED / EVIDENCE_MISMATCH
- Role-gated redaction for approvals verified in tests and manual
- Deep-links work end-to-end (Catalog → Application → Release / Deploy / Environment → back)
- Traceability inverse lookup by story id works against mock story registry
- Vite build passes; Vitest unit + component suite passes; `npm run build` green

---

## Phase B: Backend (Claude Code)

### B0: Shared Backend Preparation

- [ ] Reuse `com.sdlctower.shared.api.ApiResponse<T>`, `SectionResult<T>`
- [ ] Reuse `com.sdlctower.shared.api.ControllerAdvice` for error → `ApiResponse.error(code, message)` mapping
- [ ] Reuse `com.sdlctower.shared.audit.AuditLogEmitter`
- [ ] Reuse `com.sdlctower.shared.lineage.LineageEmitter`
- [ ] Reuse `com.sdlctower.shared.ai.AiSkillClient` interface; inject via Spring
- [ ] Reuse `com.sdlctower.shared.integration.requirement.StoryLookup` and `com.sdlctower.shared.integration.designmanagement.WorkspaceAutonomyLookup`
- [ ] Add (if absent) `com.sdlctower.shared.integration.codebuild.CodeBuildFacade` interface, with Code & Build supplying the implementation

### B1: Package-by-feature Skeleton

- [ ] Package: `com.sdlctower.domain.deploymentmanagement` with subpackages per design doc §2: `controller`, `service`, `ingestion`, `projection`, `policy`, `integration`, `persistence/{entity,repository,converter}`, `dto`, `events`

### B2: DTOs (records, immutable)

- [ ] All records per data-model §4: `CatalogAggregateDto`, `CatalogSummaryDto`, `CatalogGridRowDto`, `ApplicationHeaderDto`, `ApplicationEnvironmentDto`, `RecentReleaseRowDto`, `RecentDeployRowDto`, `ApplicationMetricsDto`, `ApplicationAiInsightsDto`, `ReleaseHeaderDto`, `ReleaseCompositionDto`, `ReleaseDeployRowDto`, `ReleaseLinkedStoryRowDto`, `AiReleaseNotesDto`, `DeployHeaderDto`, `DeployStageRowDto`, `DeployApprovalRowDto`, `DeployChangeSummaryDto`, `AiDeploymentSummaryDto`, `EnvironmentHeaderDto`, `EnvironmentCurrentDto`, `EnvironmentTimelineRowDto`, `EnvironmentStabilityDto`, `TraceabilityAggregateDto`, `AiWorkspaceSummaryDto`
- [ ] All enums mirror the frontend `enums.ts` (backend → frontend is the source of truth)

### B3: Projections (per-card, per-view)

- [ ] One projection class per card listed in design doc §2; each has a single `build(...)` method returning `SectionResult<T>`
- [ ] Each projection wraps its data access in a 500ms timeout (`CompletableFuture` + `orTimeout`) and converts timeout to `SectionResult.error(DP_RATE_LIMITED|internal, "timeout")`
- [ ] `CatalogGridProjection` applies role-based redaction (`approver`, `rationale` stripped to `(redacted)` on approval preview if applicable)
- [ ] `ReleaseCompositionProjection` uses `CodeBuildFacade.resolveCommits(applicationId, releaseVersion)` and caps at 100 with `capNotice`
- [ ] `ReleaseLinkedStoriesProjection` uses `CodeBuildFacade.resolveLinkedStories(releaseId)` — no local story table
- [ ] `DeployChangeSummaryProjection` computes delta via prior successful deploy on same env, using `CodeBuildFacade.commitRangeBetween(priorHeadSha, headSha)`
- [ ] `ApplicationMetricsProjection` computes DF / CFR / MTTR / mean lead time from local `deploy` table + outer join with `deployment_change_log`; 30-day window
- [ ] `EnvironmentStabilityProjection` same windowed metrics, scoped per-env

### B4: Read Services

- [ ] `CatalogService.loadAggregate(filters, principal)` — composes Summary + Grid + AI Summary
- [ ] `ApplicationDetailService.loadAggregate(applicationId, principal)` — composes 6 cards
- [ ] `ReleaseDetailService.loadAggregate(releaseId, principal)` — composes 5 cards
- [ ] `DeployDetailService.loadAggregate(deployId, principal)` — composes 5 cards; applies approval-redaction policy
- [ ] `EnvironmentDetailService.loadAggregate(applicationId, environmentName, principal)` — composes 4 cards
- [ ] `TraceabilityService.inverseLookup(storyId, principal)` — composes Release + Deploy rows from `CodeBuildFacade.storyCommitReleases(storyId)` + local deploy index; caps at 100
- [ ] Each service enforces `DeploymentAccessGuard` at entry (never inside projections)

### B5: Command Services (AI regenerate paths)

- [ ] `AiReleaseNotesService.enqueueRegeneration(releaseId, principal)` — enforces role + autonomy + debounce 60s keyed on `releaseId`; emits `SkillInvocation`
- [ ] `AiReleaseNotesService.regenerateWorkspaceSummary(workspaceId, principal)` — token-bucket 1/min burst 1
- [ ] `AiDeploymentSummaryService.enqueueRegeneration(deployId, principal)` — role + autonomy + debounce 30s keyed on `deployId`; skips when `change-summary.commitCount == 0` (returns SUCCESS with `status=SKIPPED`)
- [ ] All AI results routed through `ReleaseNotesEvidenceValidator` / `DeploymentSummaryEvidenceValidator`; mismatches persisted as `EVIDENCE_MISMATCH` and never surface as user-facing errors
- [ ] `AiRowStatus.PENDING` row written immediately; the actual skill call is async (fire-and-forget with completion writing the SUCCESS/FAILED row)

### B6: Policies, Extractors, Redactors

- [ ] `DeploymentAccessGuard` — workspace membership + project role gating
- [ ] `AiAutonomyPolicy` — DISABLED / OBSERVATION / SUPERVISED / AUTONOMOUS; regenerate requires `>= SUPERVISED`
- [ ] `ApprovalRedactionPolicy` — strips `rationale` and `approver` display name when principal lacks `deployment:view-approvals`
- [ ] `ReleaseNotesEvidenceValidator` — every `evidence.commit` must exist in `CodeBuildFacade.listCommitsForRelease(releaseId)`; every `evidence.story` must exist in `CodeBuildFacade.listStoryIdsForRelease(releaseId)`
- [ ] `DeploymentSummaryEvidenceValidator` — every `evidence.stage` must exist in `deploy_stage` for the deploy; every `evidence.deploy` must exist on the same application
- [ ] `LogRedactor` — AWS access key id, GitHub tokens, `Bearer …`, `(?i)token[= ]+[A-Za-z0-9]{20,}` Jenkins tokens; applied at ingestion on `deploy_stage.log_excerpt.text`; constant-time scan; never re-applied to already-redacted content
- [ ] `ReleaseVersionResolver` — see API guide §5.2; lazy-creates `release` rows keyed on `(applicationId, releaseVersion)`; defers with `DP_RELEASE_UNRESOLVED` when `RELEASE_VERSION` absent
- [ ] `RollbackDetector` — dual-signal per API guide §5.3; persists `rollback_detection_signal` on the deploy row
- [ ] `ReleaseVersionComparator` — dot-segmented numeric with lexicographic fallback; ties break on `created_at`

### B7: Jenkins Integration

- [ ] `JenkinsRestClient` — read-only; used only on backfill and IN_PROGRESS stage fill; circuit breaker + retry with jitter; returns `DP_JENKINS_UNREACHABLE` on failure
- [ ] `JenkinsAuth` — resolves per-instance API token from encrypted storage; never logged
- [ ] `JenkinsSignatureVerifier` — HMAC-SHA-256; constant-time compare; secret resolved from `JenkinsInstance` row keyed on `X-Jenkins-Instance` header
- [ ] `JenkinsPayloadParser` — accepts Notification-Plugin JSON shape; normalizes to internal `TypedEvent` enum (`job.started`, `job.completed`, `stage.completed`, `input.approved`, `input.rejected`, `input.timedOut`, `build.promoted`, `build.deleted`)

### B8: Ingestion Pipeline

- [ ] `JenkinsWebhookController` — sync-verify-signature, persist to `deployment_ingestion_outbox`, return 202; see API guide §5.1
- [ ] `IngestionDispatcher` — reads outbox, routes by `TypedEvent`:
  - `job.started` → `ReleaseVersionResolver.resolveOrCreate` → upsert `deploy` row (state=IN_PROGRESS, trigger resolved from Jenkins metadata)
  - `job.completed` → update `deploy` state (`SUCCEEDED`/`FAILED`/`CANCELLED`); `UNSTABLE` → `SUCCEEDED` with led=AMBER; run `RollbackDetector`
  - `stage.completed` → append `deploy_stage` row with conclusion + duration; apply `LogRedactor` if log excerpt attached
  - `input.approved|rejected|timedOut` → append `approval_event` row with encrypted rationale (`approval_event.rationale_cipher`)
  - `build.promoted` → log-only (V1)
  - `build.deleted` → soft-hide the deploy row (set `hidden_at`)
  - Unknown event type → mark DONE forward-compatible; never FAILED
- [ ] Every handler updates `deploy.last_ingested_at` at completion
- [ ] Every handler emits `DeploymentChangeLogEntry` via `DeploymentChangeLogPublisher` for auditable state transitions

### B9: Backfill + Resync

- [ ] `JenkinsBackfillService.backfill(jenkinsInstanceId)`:
  - Triggered on `JenkinsInstance` registration and on webhook-drop detection
  - Enumerates jobs in the instance (scoped to applications owned by the workspace)
  - For each job: fetches last 14 days of builds including stage details
  - Batches upserts through the same event handlers (never direct DB writes) to preserve audit
  - Idempotent under repeat via `uq_deploy_jenkins UNIQUE (jenkins_instance_id, jenkins_job_full_name, jenkins_build_number)`
  - Persists progress in `jenkins_backfill_checkpoint`
- [ ] `NightlyResyncJob` — Spring `@Scheduled(cron = "0 15 2 * * *")` (02:15 UTC):
  - Walks `JenkinsInstance` rows; for each: fetches builds modified in last 72h
  - Reconciles delta; any row observed on Jenkins but missing locally → routed through the same event handlers
  - Emits a `ResyncRun` record to the change log for auditability

### B10: Controllers

- [ ] `DeploymentController` with endpoints per API guide §2–§3; all read endpoints return 200 with `ApiResponse<T>` envelope; regenerate endpoints return 200 with updated entity state
- [ ] `@Pattern` path validation: `applicationId` (`^application-[a-z0-9\-]+$`), `releaseId` (`^release-[a-z0-9\-]+$`), `deployId` (`^deploy-[a-z0-9\-]+$`), environment `name` (`^[a-z][a-z0-9_\-]{0,62}$`), `storyId` (`^STORY-[A-Z0-9\-]+$`)
- [ ] `JenkinsWebhookController.POST /webhooks/jenkins` returns 202 Accepted; never returns the envelope body — Jenkins expects an empty body
- [ ] Cross-slice facades: `/internal/story/{storyId}/deploy-aggregate` and `/internal/release/{releaseId}/commit-slice` under `X-Service-Token` filter, never browser-exposed

### B11: Flyway Migrations (V60–V67)

- [ ] `V60__create_jenkins_instance_and_application.sql` — `jenkins_instance`, `application` with FK to workspace, project, jenkins_instance; unique `(workspace_id, slug)`
- [ ] `V61__create_application_environment.sql` — `application_environment`; unique `(application_id, name)`; `kind` enum
- [ ] `V62__create_release.sql` — `release`; unique `(application_id, release_version)`; indexes on `(build_artifact_slice_id, build_artifact_id)`, `(state)`
- [ ] `V63__create_deploy_and_deploy_stage.sql` — `deploy`, `deploy_stage`; `uq_deploy_jenkins UNIQUE (jenkins_instance_id, jenkins_job_full_name, jenkins_build_number)` (idempotency); indexes on `(application_id, environment_name, completed_at DESC)`, `(release_id)`, `(state)`, `(trigger)`, `(last_ingested_at)`
- [ ] `V64__create_approval_event.sql` — `approval_event` with `rationale_cipher CLOB` (encrypted at rest); FK to `deploy`, `deploy_stage`
- [ ] `V65__create_ai_release_notes_and_summary.sql` — `ai_release_notes` (unique `(release_id, skill_version)`), `ai_deployment_summary` (unique `(deploy_id, skill_version)`); evidence hash columns
- [ ] `V66__create_deployment_change_log_and_outbox.sql` — `deployment_change_log` with index `(entity_type, entity_id, created_at DESC)`; `deployment_ingestion_outbox` with unique `delivery_id`, index `(status, received_at)`; `jenkins_backfill_checkpoint`
- [ ] `V67__seed_deployment_local.sql` — local H2 seed: 1 Jenkins instance, 3 applications, 4 envs each, 10 releases, 30 deploys (mix of SUCCEEDED/FAILED/ROLLED_BACK/IN_PROGRESS), 6 approval events (1 per kind + extras), 4 AI release notes (SUCCESS, PENDING, FAILED, EVIDENCE_MISMATCH), 4 AI deploy summaries
- [ ] Verify all migrations run on H2 without error
- [ ] Produce Oracle variant if DDL diverges (notably `CLOB`, encrypted column syntax) under `db/migration/oracle/`

### B12: Entities + Repositories

- [ ] JPA entities (12 net-new per data-model §3): `JenkinsInstanceEntity`, `ApplicationEntity`, `ApplicationEnvironmentEntity`, `ReleaseEntity`, `DeployEntity`, `DeployStageEntity`, `ApprovalEventEntity`, `AiReleaseNotesEntity`, `AiDeploymentSummaryEntity`, `DeploymentChangeLogEntity`, `DeploymentIngestionOutboxEntity`, `JenkinsBackfillCheckpointEntity`
- [ ] Map `ai_release_notes.body_markdown`, `ai_deployment_summary.narrative`, and `approval_event.rationale_cipher` as `@Lob String` with `FetchType.LAZY` — never eagerly loaded via catalog projections
- [ ] Map `deployment_ingestion_outbox.raw_body` as `@Lob byte[]` with `FetchType.LAZY`
- [ ] Repositories: one per entity with representative query methods:
  - `ApplicationRepository.findByWorkspaceIdIn(workspaceIds, Pageable)`
  - `ApplicationRepository.findBySlugAndWorkspaceId(slug, workspaceId)`
  - `ReleaseRepository.findByApplicationAndVersion(applicationId, releaseVersion)`
  - `ReleaseRepository.findTop20ByApplicationIdOrderByCreatedAtDesc(applicationId)`
  - `DeployRepository.findTop20ByApplicationIdOrderByCompletedAtDesc(applicationId)`
  - `DeployRepository.findByJenkinsInstanceIdAndJenkinsJobFullNameAndJenkinsBuildNumber(...)` — idempotency lookup
  - `DeployRepository.findLastSuccessByApplicationAndEnvironment(applicationId, environmentName)` — prior-deploy resolution for change summary + rollback
  - `DeployRepository.findLastSuccessBeforeCompletedAt(applicationId, environmentName, completedAt)` — rollback comparison
  - `DeployStageRepository.findByDeployIdOrderByOrderAsc(deployId)`
  - `ApprovalEventRepository.findByDeployIdOrderByPromptedAtAsc(deployId)`
  - `AiReleaseNotesRepository.findTopByReleaseIdAndSkillVersionOrderByGeneratedAtDesc(releaseId, skillVersion)`
  - `AiDeploymentSummaryRepository.findTopByDeployIdAndSkillVersionOrderByGeneratedAtDesc(deployId, skillVersion)`
  - `DeploymentIngestionOutboxRepository.findByDeliveryId(deliveryId)` — idempotency
  - `DeploymentIngestionOutboxRepository.lockAndClaimOldest(batch)` — worker drain with row lock
- [ ] All cross-slice reads go through facades — no direct JPA association from Deployment Management to `build_artifact`, `commit`, `story`, `member`, `project`

### B13: Governance, Lineage, Cross-Slice Facade

- [ ] Every write path writes a `LineageEvent` identifying Deployment Management as the authoring domain
- [ ] Every regenerate endpoint writes an `AuditLogEntry`
- [ ] AI regenerate paths emit `SkillInvocation` events (invoker, autonomy level at time of call, skill version, latency, outcome, request token count, evidence integrity outcome)
- [ ] `DeploymentHealthFacade` — cross-slice read facade exposed to Dashboard + Project Management:
  - `healthByProject(projectId)` → `{deploymentFrequencyPerDay, changeFailureRate, mttrMinutes, lastDeployState, lastDeployAt}`
  - `healthByWorkspace(workspaceId)` → workspace-level rollup
  - `recentRollbacksByWorkspace(workspaceId, since)` → list for dashboard rollback feed
- [ ] Incident slice deep-link handshake verified: `context` object `{ deployId, releaseVersion, environmentName, failingStageId, summary }` accepted and pre-fills the Incident create form

### B14: Backend Tests

- [ ] `DeploymentControllerTest` (MockMvc):
  - Catalog aggregate happy + per-projection error isolation
  - Application detail happy + 404
  - Release detail happy + 404 + EVIDENCE_MISMATCH surfaces as AI row status, not HTTP error
  - Deploy detail happy + ROLLED_BACK rendering + IN_PROGRESS polling projection
  - Deploy detail approvals: privileged principal sees rationale/approver, non-privileged sees `(redacted)` on both
  - Environment detail happy + 404
  - Traceability happy + story with no deploys (empty releases/deploys) + cap notice
  - Every regenerate endpoint: happy + role denied + autonomy insufficient + debounce → 429 + validation failure
  - Workspace isolation: cross-workspace id → 403 `DP_WORKSPACE_FORBIDDEN`
- [ ] `JenkinsWebhookControllerTest` (MockMvc):
  - Valid signature → 202 + outbox row created
  - Invalid signature → 401 `DP_INGEST_SIGNATURE_INVALID`
  - Unknown `X-Jenkins-Instance` → 404 `DP_JENKINS_INSTANCE_UNKNOWN`
  - Malformed payload → 400 `DP_INGEST_PAYLOAD_INVALID`
  - Duplicate `delivery_id` → 202 but no second outbox row (idempotent)
  - Duplicate `(jenkinsInstanceId, jobFullName, buildNumber)` → handler no-ops; `uq_deploy_jenkins` holds
  - Unsupported event type → 202 + dispatcher marks DONE without touching domain tables
- [ ] `IngestionDispatcherTest`: one test per event type asserting handler invocation + correct upsert behavior; `UNSTABLE` → state=SUCCEEDED with led=AMBER
- [ ] `ReleaseVersionResolverTest`: lazy creation on first seen version; deferred with `DP_RELEASE_UNRESOLVED` when `RELEASE_VERSION` missing
- [ ] `RollbackDetectorTest`: explicit `trigger=ROLLBACK` → positive; version-older-than-prior → positive with signal recorded; same version re-deploy → negative
- [ ] `ApprovalRedactionPolicyTest`: privileged sees rationale + approver; non-privileged sees both as `(redacted)`
- [ ] `ReleaseNotesEvidenceValidatorTest`: valid evidence → persisted; fabricated commit → persisted as `EVIDENCE_MISMATCH`
- [ ] `DeploymentSummaryEvidenceValidatorTest`: fabricated stage → `EVIDENCE_MISMATCH`; no change → `SKIPPED`
- [ ] `LogRedactorTest`: each pattern / false-positive safe / idempotent
- [ ] `AiReleaseNotesServiceTest`: happy + autonomy + role + debounce + skill 5xx retry
- [ ] `AiDeploymentSummaryServiceTest`: happy + skipped (zero change) + autonomy + evidence mismatch
- [ ] `DeploymentAccessGuardTest`: read vs admin gating; cross-workspace blocked
- [ ] `JenkinsBackfillServiceTest`: idempotent under repeat; 14-day window; all events go through handlers (no direct DB writes)
- [ ] `NightlyResyncJobTest`: delta reconciliation; no duplicate change log for already-seen state
- [ ] Repository tests against H2 for every entity + edge queries (idempotency, rollback comparator, metrics windowing)
- [ ] `FlywayMigrationIntegrationTest`: applies V60–V67 on a clean H2; verifies target schema (including `uq_deploy_jenkins` rejects duplicates; `release` unique constraint holds) + seed counts
- [ ] Golden-file test for API envelopes — compare response JSON to canonical examples in API guide §3 (including redacted-approvals shape for non-privileged principal)
- [ ] Webhook signature fixture test — canned body + known secret + expected HMAC → verifier passes; flipped byte → verifier rejects with constant-time compare

### B15: Connect Frontend to Backend

- [ ] Set `VITE_USE_BACKEND=true` in dev config
- [ ] Verify Vite proxy routes `/api/v1/deployment-management/*` to backend
- [ ] Smoke: browse `/deployment` against live backend — 3 catalog cards hydrated from seed data
- [ ] Smoke: browse `/deployment/applications/{seedAppId}` — 6 app cards hydrated
- [ ] Smoke: browse `/deployment/releases/{seedReleaseId}` — 5 release cards hydrated; composition shows commits resolved via `CodeBuildFacade`
- [ ] Smoke: browse `/deployment/deploys/{seedDeployId}` — 5 deploy cards hydrated; rollback variant renders `rollback_detection_signal`
- [ ] Smoke: browse `/deployment/applications/{seedAppId}/environments/prod` — 4 env cards hydrated
- [ ] Smoke: browse `/deployment/traceability?storyId=STORY-…` — releases + deploys populated
- [ ] Smoke: execute each mutation:
  - AI release notes regenerate (PM principal) → new row keyed by `(releaseId, skillVersion)`; change log entry
  - AI deploy summary regenerate (Tech Lead principal) → summary refreshed or `SKIPPED`
  - AI workspace summary regenerate (admin) → rate-limited to 1/min
- [ ] Smoke: force evidence mismatch (inject skill response with fabricated commit) → UI locks content + banner, no hard error
- [ ] Smoke: attempt each error path:
  - Non-PM user tries regenerate → controls hidden (not merely disabled)
  - Cross-workspace deploy id → `DP_WORKSPACE_FORBIDDEN`
  - Debounce hit on regenerate → `DP_RATE_LIMITED` + `<RateLimitBanner>`
  - AI skill 5xx → `DP_AI_UNAVAILABLE` toast + Retry in card
- [ ] Verify deep-links: Catalog → Application → Deploy; Catalog → Application → Release; Deploy → Incident create (pre-fills); Traceability → Requirement story page (KNOWN) and Requirement search (UNKNOWN_STORY); Jenkins chevrons open with `rel="noopener noreferrer"`

### B16: Validate Full Stack

- [ ] `./mvnw verify` passes (all new tests + existing suites)
- [ ] `npm run build` passes
- [ ] End-to-end flow on seed data:
  - Simulate webhook: `job.started` with `RELEASE_VERSION=2026.04.18-0001` on a fresh application → release row lazy-created (state=PREPARED); deploy row IN_PROGRESS
  - Simulate webhook: `stage.completed SUCCESS` for stages 1–2, `stage.completed FAILURE` for stage 3 → stages appear on Deploy Detail; last stage FAILURE
  - Simulate webhook: `job.completed FAILURE` → deploy transitions to FAILED; release stays PREPARED; metrics update
  - Simulate webhook: `job.started trigger=ROLLBACK` for the prior release → `RollbackDetector` marks it as rollback; prior deploy flagged `rolled_back=true` with `rollback_detection_signal=trigger=rollback`
  - Simulate webhook: `job.started` with older `RELEASE_VERSION` than last-success → `RollbackDetector` marks it with signal=`version-older-than-prior`
  - Simulate webhook: `input.approved` with rationale → approval row appears on Deploy Detail (rationale visible to PM, redacted to Developer)
  - Regenerate AI release notes → notes row persisted; evidence validator passes; markdown rendered
  - Regenerate AI release notes with fabricated evidence → persisted as `EVIDENCE_MISMATCH`; UI banner + locked content; no hard error
  - Regenerate AI deploy summary on zero-change deploy → persisted as `SKIPPED`
  - Traceability by story id → returns releases + deploys resolved read-side; no local story table touched
- [ ] Webhook idempotency: replay the same `delivery_id` → no second outbox row; replay the same `(instance, job, build)` triple → no second deploy row
- [ ] Approval role-gating: Developer principal sees `(redacted)` in rationale + approver; PM sees actual values
- [ ] Jenkins backfill on new `JenkinsInstance` registration: fetches 14 days; all data arrives via handlers (no direct DB writes); change log populated
- [ ] Nightly resync: simulate drift (delete a local deploy row) → next resync re-hydrates without duplicate change log entries
- [ ] Audit log contains entries for every regenerate; Lineage graph shows Deployment Management as authoring domain; SkillInvocation records include autonomy level + evidence outcome
- [ ] Freshness SLO: on happy-path webhook, `lastIngestedAt` updates within 45s P95 of Jenkins timestamp; on injected delay, `FreshnessIndicator` renders Degraded / Stale tier
- [ ] Per-projection 500ms timeout: inject a projection that sleeps 1s → the containing card returns `SectionResultDto(error=...)`; other cards unaffected; page-level status remains 200
- [ ] Dashboard / Project Management deploy health badges populate via `DeploymentHealthFacade` (no regressions in those slices)
- [ ] Incident deep-link: from a FAILED Deploy Detail, clicking "Open Incident" pre-fills deploy/release/env/stage/summary; no write to Jenkins

### Phase B Definition of Done

- All read endpoints return correct envelopes for happy path
- All regenerate endpoints enforce role + autonomy + debounce + evidence integrity
- Webhook receiver is signature-verified, idempotent on `delivery_id` AND on `(instance, job, build)`, outbox-backed, returns 202 within SLO
- Dispatcher routes every supported event type to the correct handler; unknown events are forward-compatible
- Log redaction runs at ingestion; raw secrets never persisted to `deploy_stage.log_excerpt.text`
- Evidence integrity check surfaces mismatched AI rows as `EVIDENCE_MISMATCH` without failing the command
- Rollback detection is dual-signal and the firing signal is persisted for audit
- Release row lazy creation handles `DP_RELEASE_UNRESOLVED` deferral path
- Approval rationale + approver display name correctly gated to `deployment:view-approvals`
- Per-projection 500ms timeout degrades to section-level error only
- Flyway V60–V67 migrations run cleanly on H2; Oracle variants prepared under `db/migration/oracle/` if divergent
- Frontend works in live mode (`VITE_USE_BACKEND=true`)
- `DeploymentHealthFacade` serves Dashboard + Project Management cross-slice reads without direct table access
- Code & Build remains the single source of truth for Story↔Commit — no story-link table added in this slice
- Write authority for all 12 Deployment Management entities is demonstrably owned by this slice (no other slice mutates these tables)

---

## Dependency Plan

### Critical Path

```
A0 (shared infra) → A1 (types) → A2 (mocks) → A3 (primitives)
   → A4 (catalog) + A5 (application) + A6 (release) + A7 (deploy) + A8 (environment) + A9 (traceability)  [parallel]
   → A10 (store/API) → A11 (views + routing) → A12 (states) → A13 (validate)
        │
        ↓
B0 (shared backend) → B1 (package) → B2 (DTOs)
   → B3 (projections in parallel) → B4 (read services)
   → B6 (policies + resolver + detector + redactor) → B7 (Jenkins integration) → B5 (command services)
   → B8 (ingestion pipeline) → B9 (backfill / resync)
   → B10 (controllers) → B11 (Flyway V60-V67) → B12 (entities/repos)
   → B13 (governance / lineage / DeploymentHealthFacade) → B14 (tests)
   → B15 (connect FE) → B16 (validate full stack)
```

### Upstream Blockers (must be resolved before B5/B14)

- `CodeBuildFacade.resolveCommits(applicationId, releaseVersion)` and `resolveLinkedStories(releaseId)` — owned by Code & Build
- `StoryLookup` — already in place from Requirement slice
- `WorkspaceAutonomyLookup` — already in place from Design Management
- Incident deep-link handshake — coordinate with Incident slice maintainer; contract defined in architecture doc §Integration

### Ship Gate

1. Phase A green (Vitest + component tests + axe smoke)
2. Phase B green (MockMvc + Flyway + golden-file + webhook signature)
3. End-to-end smoke on seed data
4. `DeploymentHealthFacade` consumed by Dashboard without regression
5. Incident deep-link verified
6. CLAUDE.md Lesson compliance — all 9 docs present, Mermaid 8.x-compatible, cross-references clean
