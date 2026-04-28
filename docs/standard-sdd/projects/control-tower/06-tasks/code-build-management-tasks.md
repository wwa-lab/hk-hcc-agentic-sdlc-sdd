# Code & Build Management Tasks

## Objective

Deliver the Code & Build Management slice end-to-end in two phases: Phase A (frontend with mocks, Codex) and Phase B (backend wiring, Codex). V1 scope is a **read-only observability viewer** over GitHub + GitHub Actions, with five views — Catalog, Repo Detail, PR Detail, Run Detail, and Story ↔ Commit ↔ Build Traceability — plus three AI surfaces (workspace summary, PR review assist, run failure triage). No build triggering, no config mutation, no secrets management in V1, per the scope decisions recorded in `code-build-management-architecture.md` §Decisions (D1–D14). Implementation must conform to the 8 upstream SDD documents.

## Implementation Strategy

- **Frontend-first (Phase A):** mock-driven, no backend dependency. Ship five browsable views — Catalog, Repo Detail, PR Detail, Run Detail, Traceability — each assembled from independently mockable, independently retryable cards. AI regenerate + Run re-run are the only mutations; both round-trip through a mock command layer that simulates `CB_AI_UNAVAILABLE` (5%), `CB_GH_RATE_LIMIT` (2%), and `CB_TRIAGE_EVIDENCE_MISMATCH` (3%).
- **Backend (Phase B):** implement projections, command services, controllers, GitHub ingestion (webhook receiver + outbox + dispatcher), backfill, resync, entities, and Flyway migrations V40–V47. Swap frontend from mock mode to live API by toggling `VITE_USE_BACKEND=true`. No `ddl-auto`.
- **Per-card isolation:** each card on each view is independently developable, mockable, and testable. Per-projection 500ms timeout degrades to `SectionResult<T>(data=null, error=...)` — never a page-level failure.
- **Net-new entities only:** Code & Build Management introduces 18 net-new tables per data-model §2; `Project`, `Workspace`, `Member`, and `Story` are reused read-only via existing cross-slice facades.
- **Observability-first:** no triggering, no config mutation, no secrets in V1. AI regenerate and Run re-run are the only write paths reachable from the UI. All state changes observed from GitHub arrive via webhook; backfill + nightly resync are the recovery paths.
- **Story-Id resolution via trailers only:** `Story-Id: STORY-…` or `Relates-to: STORY-…` in commit message trailers or PR body trailers; unknown ids are persisted with `StoryLinkStatus.UNKNOWN_STORY` and re-resolved by a nightly job rather than dropped (D10).
- **Shared primitives:** live under `src/shared/` / `com.sdlctower.shared` — Code & Build Management does not fork shell-level behavior.

## Traceability

- Requirements: [code-build-management-requirements.md](../01-requirements/code-build-management-requirements.md)
- Stories: [code-build-management-stories.md](../02-user-stories/code-build-management-stories.md)
- Spec: [code-build-management-spec.md](../03-spec/code-build-management-spec.md)
- Architecture: [code-build-management-architecture.md](../04-architecture/code-build-management-architecture.md)
- Data Flow: [code-build-management-data-flow.md](../04-architecture/code-build-management-data-flow.md)
- Data Model: [code-build-management-data-model.md](../04-architecture/code-build-management-data-model.md)
- Design: [code-build-management-design.md](../05-design/code-build-management-design.md)
- API Guide: [code-build-management-API_IMPLEMENTATION_GUIDE.md](../05-design/contracts/code-build-management-API_IMPLEMENTATION_GUIDE.md)

## Planning Assumptions

- `shared-app-shell`, `dashboard`, `requirement`, `incident`, `team-space`, `project-space`, `project-management`, `design-management` slices are merged and operational.
- Shared shell primitives (`SectionResult<T>`, `ApiResponse<T>`, `fetchJson<T>`, `LineageBadge.vue`, `shellUiStore.setBreadcrumbs`, `shellUiStore.setAiPanelContent`, route guard `requireWorkspaceMember(projectId)`) already exist and are reused as-is.
- The Requirement slice exposes a read-only `StoryLookup` facade returning `(storyId, title, projectId, state)` — if not yet available, stub in `com.sdlctower.shared.integration.requirement` and document as an upstream blocker.
- The Dashboard and Project Management slices expect a cross-slice facade `BuildHealthFacade` that returns per-project health badges sourced from Code & Build Management; ship the facade in Phase B.
- `WorkspaceAutonomyLookup` facade (from Design Management) is reused; AI regenerate gated by `aiAutonomyLevel >= SUPERVISED`.
- H2 is the local dev DB; Oracle is the deployed DB. Flyway migrations V40–V47 must run on both (split per-dialect if DDL diverges — notably `CLOB` for `log_excerpt.text`, `BLOB` for `workflow_raw_json`, and the `log_excerpt.byte_count <= 1048576` CHECK).
- Java 21 + Spring Boot 3.x; Vue 3 + Vite + Pinia + TypeScript.
- No localStorage / sessionStorage (per shared shell rules).
- Base path for all endpoints: `/api/v1/code-build-management` (per API guide §1).
- Mock toggle pattern follows repo convention: `import.meta.env.DEV && !import.meta.env.VITE_USE_BACKEND`.
- Access: all five read views are visible to any workspace member on the owning project. AI regenerate and Run re-run require PM or Tech Lead role; BLOCKER-severity AI PR review note **bodies** (not counts) are role-gated to PM + Tech Lead (REQ-CB-84).
- Log excerpt hard limit: **1 MiB per step**, enforced server-side via `CHECK (byte_count <= 1048576)` and via truncation at ingestion time.
- GitHub App auth: single org-scoped App; installation id per workspace (D2). Webhook secret rotated via platform ops, stored encrypted at rest.
- Webhook signature header: `X-Hub-Signature-256`; HMAC-SHA-256 against raw request body; constant-time compare (D3).
- Supported webhook events for V1: `push`, `pull_request`, `pull_request_review`, `workflow_run`, `workflow_job`, `check_run`, `installation`, `installation_repositories`.
- Ingestion is **outbox-backed** — webhook receiver persists to `ingestion_outbox` and returns 202 immediately; the dispatcher worker is the one that parses and projects (D4).
- No polling fallback for build status; webhook + backfill + nightly resync only (D5). Install-time backfill covers the trailing 14 days; nightly resync covers the trailing 72 hours.
- AI outputs persist keyed by `(entityId, entityType, skillVersion)` — the natural cache key rotates on skill bump and on head advance (for PR reviews) or new run id (for triage).
- AI triage evidence integrity check: AI response must only reference `(runId, jobId, stepId)` tuples present in the request prompt; mismatches are rejected as `CB_TRIAGE_EVIDENCE_MISMATCH` and persisted with `AiRowStatus.FAILED_EVIDENCE`.
- BLOCKER-severity PR review notes: all workspace members see the **count**; the note **body** is visible only to PM + Tech Lead on that project.
- Log redaction patterns (applied at ingestion, never to pre-redacted content): `AKIA[0-9A-Z]{16}` (AWS access key id), `ghp_[A-Za-z0-9]{36}` / `gho_[A-Za-z0-9]{36}` / `ghs_[A-Za-z0-9]{36}` (GitHub tokens), generic `Bearer [A-Za-z0-9._\-]{20,}` (tokens in auth headers).

---

## Phase A: Frontend (Codex)

### A0: Shared Infrastructure Preparation

- [ ] Reuse `src/shared/types/section.ts` for `SectionResult<T>`
- [ ] Reuse `src/shared/api/client.ts` for `fetchJson<T>`
- [ ] Reuse `src/shared/components/LineageBadge.vue` for provenance badges
- [ ] Reuse `shellUiStore.setBreadcrumbs(...)` + `shellUiStore.setAiPanelContent(...)`
- [ ] Confirm Tactical Command tokens already exported from shared design tokens; add only if absent: `--cb-status-success`, `--cb-status-failure`, `--cb-status-running`, `--cb-status-queued`, `--cb-status-cancelled`, `--cb-status-neutral`, `--cb-severity-blocker`, `--cb-severity-major`, `--cb-severity-minor`, `--cb-severity-info`, `--cb-link-ok`, `--cb-link-stale`, `--cb-link-unknown`
- [ ] Confirm route guard helper `requireWorkspaceMember(projectId)` exists; if not, coordinate with shell owners before merge

### A1: Define Code & Build Management Types

- [ ] `src/features/code-build-management/types/enums.ts` — all enums per data-model §2.2 (`RepoVisibility`, `PrState`, `RunStatus`, `RunTrigger`, `StepConclusion`, `StoryLinkStatus`, `AiNoteSeverity`, `AiRowStatus`, `ChangeLogEntryType`)
- [ ] `src/features/code-build-management/types/catalog.ts` — `CatalogSummary`, `CatalogRepoRow`, `CatalogFilter`, `CatalogSection` (grouped by project)
- [ ] `src/features/code-build-management/types/repo.ts` — `RepoDetail`, `RepoHeader`, `BranchRow`, `RecentCommitRow`, `RecentPrRow`, `RecentRunRow`, `RepoHealthSummary`
- [ ] `src/features/code-build-management/types/pr.ts` — `PullRequestDetail`, `PrHeader`, `PrCheckRow`, `PrReviewRow`, `PrCommitRow`, `AiPrReviewPayload`, `AiPrReviewNote`, `AiPrReviewNoteCounts`
- [ ] `src/features/code-build-management/types/run.ts` — `PipelineRunDetail`, `RunHeader`, `RunJobRow`, `RunStepRow`, `LogExcerptBlock`, `AiTriagePayload`, `AiTriageRow`
- [ ] `src/features/code-build-management/types/traceability.ts` — `TraceabilityStoryRow`, `TraceabilityCommitRef`, `TraceabilityBuildRef`, `TraceabilityStatus`
- [ ] `src/features/code-build-management/types/requests.ts` — mutation request/response records for `regenerateAiPrReview`, `regenerateAiTriage`, `rerunRun`
- [ ] `src/features/code-build-management/types/aggregate.ts` — `CatalogAggregate`, `RepoDetailAggregate`, `PrDetailAggregate`, `RunDetailAggregate`, `TraceabilityAggregate`, `CodeBuildManagementState`

### A2: Build Mock Data

- [ ] `src/features/code-build-management/mock/catalog.mock.ts` — seeds ~12 repos across 4 projects: mix of PUBLIC / PRIVATE; default branch success / failure / running distribution; at least one repo with no runs yet; at least one with a running workflow
- [ ] `src/features/code-build-management/mock/catalogSummary.mock.ts` — totals + status bucket counts + repos-without-runs count
- [ ] `src/features/code-build-management/mock/repoDetail.mock.ts` — full detail for 3 canonical repos: one healthy (recent green), one flaky (mixed recent runs), one stale (no activity 30+ days)
- [ ] `src/features/code-build-management/mock/prDetail.mock.ts` — 4 PRs: one OPEN with AI review returning 2 BLOCKER + 3 MAJOR + 4 MINOR notes, one OPEN clean, one MERGED, one CLOSED without merge
- [ ] `src/features/code-build-management/mock/runDetail.mock.ts` — 4 runs: one SUCCESS, one FAILURE with triage returning 3 rows (2 SUCCESS, 1 FAILED_EVIDENCE), one IN_PROGRESS, one CANCELLED
- [ ] `src/features/code-build-management/mock/logExcerpts.mock.ts` — canned log excerpt payloads per step, including at least one with an AWS key, one with `ghp_…`, and one with a `Bearer …` header — all pre-redacted to `***REDACTED:AWS_KEY***` / `***REDACTED:GH_TOKEN***` / `***REDACTED:BEARER***`
- [ ] `src/features/code-build-management/mock/aiPrReview.mock.ts` — SUCCESS for 2 PRs, PENDING for 1, FAILED with retry for 1; each SUCCESS payload includes note counts + bodies; one payload includes BLOCKER bodies that must be hidden from non-PM/Tech-Lead via UI gating
- [ ] `src/features/code-build-management/mock/aiTriage.mock.ts` — SUCCESS (all rows reference valid step ids), PENDING, FAILED_EVIDENCE (one row references a non-existent step id — renders as `FAILED_EVIDENCE`)
- [ ] `src/features/code-build-management/mock/traceability.mock.ts` — ~15 stories with known story ids, ~5 stories with UNKNOWN_STORY rows (commits that referenced a story id that does not resolve), covering all 4 `StoryLinkStatus` states
- [ ] `src/features/code-build-management/mock/webhookFixtures/` — canned webhook bodies (push, pull_request opened + synchronize, pull_request_review, workflow_run queued + in_progress + completed, workflow_job, check_run, installation created, installation_repositories added) keyed by event type — used by the mock command loop for round-trip ingestion simulation
- [ ] Mock command layer: `src/features/code-build-management/mock/commandLoop.ts` — simulates:
  - `CB_AI_UNAVAILABLE` (5% injected) on AI regenerate
  - `CB_GH_RATE_LIMIT` (2% injected) on rerun
  - `CB_TRIAGE_EVIDENCE_MISMATCH` (3% injected) on triage regeneration
  - `CB_AI_AUTONOMY_INSUFFICIENT` when workspace autonomy < SUPERVISED
  - `CB_ROLE_REQUIRED` for non-PM / non-Tech-Lead invokers
  - `CB_WORKSPACE_FORBIDDEN` for cross-workspace entity ids
  - `CB_UNKNOWN_STORY` surfaced only in Traceability view, never blocking
  - `CB_WEBHOOK_SIGNATURE_INVALID` when signature header is tampered in `__SIGNATURE_TRIGGER__` fixture
  - `CB_STALE_HEAD_SHA` on PR regenerate when `prevHeadSha` does not match current

### A3: Build Reusable Primitives

- [ ] `BuildStatusBadge.vue` — SUCCESS / FAILURE / RUNNING / QUEUED / CANCELLED / NEUTRAL with LED palette and live-animating running state
- [ ] `PrStateBadge.vue` — OPEN / MERGED / CLOSED / DRAFT
- [ ] `StoryLinkStatusChip.vue` — KNOWN / UNKNOWN_STORY / NO_STORY_ID / AMBIGUOUS with tooltip explaining derivation
- [ ] `ShaPill.vue` — short 7-char SHA monospace pill with copy-to-clipboard, deep-link chevron to GitHub commit
- [ ] `BranchPill.vue` — branch name with default-branch marker
- [ ] `RepoNavChip.vue` — `owner/repo` label with deep-link chevron to GitHub, `rel="noopener noreferrer"` enforced
- [ ] `DurationLabel.vue` — humanised duration (`3m 12s`) with absolute-timestamp tooltip
- [ ] `AiSeverityChip.vue` — BLOCKER / MAJOR / MINOR / INFO with color-coded LED styling
- [ ] `AiNoteCountsBar.vue` — visible-to-all summary of `(blocker, major, minor, info)` counts; never exposes BLOCKER bodies
- [ ] `AiTriageRowCard.vue` — row for one triage conclusion; SUCCESS / PENDING / FAILED_EVIDENCE rendering; shows `(runId, jobId, stepId)` evidence trail; Retry disabled unless admin
- [ ] `LogExcerptBlock.vue` — fixed-height monospace panel with line numbers, `REDACTED` segments rendered as muted tokens, open-in-GitHub chevron, horizontal scroll trap
- [ ] `RateLimitBanner.vue` — shown when `CB_GH_RATE_LIMIT` returns; amber with reset-at timestamp
- [ ] `WebhookStaleBanner.vue` — shown when viewed entity's `last_synced_at` is > 10 minutes stale AND the entity is in a non-terminal state
- [ ] `LineageBadge.vue` (reused) — surfaces provenance on AI outputs (skill version + generated-at)
- [ ] `BuildHealthSparkline.vue` — last 14 runs compressed as colored blocks (one per run); click → Run Detail

### A4: Build Catalog View Cards

- [ ] `CatalogSummaryBarCard.vue` — aggregate across visible projects: total repos, repos with recent SUCCESS, FAILURE, RUNNING, CANCELLED, NEUTRAL; repos with no recent runs
- [ ] `CatalogRepoGridCard.vue` — grouped by project, each group a row of repo tiles; sticky project header; supports filter by build status / visibility / project / search string
- [ ] `CatalogFilterBar.vue` — filter chips (build status, visibility, project), text search, sort toggle (recent activity / alphabetical / health-worst-first)
- [ ] `CatalogEmptyState.vue` — 4 distinct copies per design §8: "No repos in workspace yet", "No repos match your filters", "No GitHub App installation for this workspace", "No permission to view this workspace's repos"
- [ ] `CatalogErrorState.vue` — per-card retry UI; includes `<RateLimitBanner>` when applicable
- [ ] Each repo tile in the grid renders: `<RepoNavChip>`, last-default-branch `<BuildStatusBadge>`, `<BuildHealthSparkline>`, last-activity relative timestamp

### A5: Build Repo Detail View Cards

- [ ] `RepoHeaderCard.vue` — `<RepoNavChip>`, visibility badge, default branch, workspace / project badge, last synced indicator (shows `<WebhookStaleBanner>` when stale)
- [ ] `RepoRecentRunsCard.vue` — reverse-chron last 25 runs; each row: `<BuildStatusBadge>`, `<BranchPill>`, `<ShaPill>`, trigger, `<DurationLabel>`, actor; click → Run Detail
- [ ] `RepoRecentPrsCard.vue` — last 15 PRs; each row: `<PrStateBadge>`, title, number, author, base→head branches, `<AiNoteCountsBar>`; click → PR Detail
- [ ] `RepoRecentCommitsCard.vue` — last 20 commits on default branch; each row: `<ShaPill>`, short message, author, timestamp, `<StoryLinkStatusChip>`
- [ ] `RepoBranchesCard.vue` — protected-branch list with last-seen head SHA + last-run status
- [ ] `RepoHealthSummaryCard.vue` — 14-day rolling: success rate %, median duration, failure distribution (top 3 failing workflows)
- [ ] `RepoAiSummaryCard.vue` — workspace-level AI summary scoped to this repo (delta vs prior 14 days); Regenerate button gated by autonomy + role; reuses `<LineageBadge>`

### A6: Build PR Detail View Cards

- [ ] `PrHeaderCard.vue` — title, number, `<PrStateBadge>`, author, base → head with `<BranchPill>` + `<ShaPill>`, linked `<StoryLinkStatusChip>`, deep-link chevron to GitHub PR
- [ ] `PrChecksCard.vue` — rollup of check runs tied to head SHA; each row: check name, `<BuildStatusBadge>`, conclusion, `<DurationLabel>`, deep-link to the owning run
- [ ] `PrReviewsCard.vue` — human review list (APPROVED / CHANGES_REQUESTED / COMMENTED / DISMISSED) with reviewer, timestamp, and review body summary
- [ ] `PrCommitsCard.vue` — commits in this PR; each row: `<ShaPill>`, short message, author, `<StoryLinkStatusChip>`
- [ ] `PrAiReviewCard.vue` — `<AiNoteCountsBar>` always visible; note-body list visible only when `canSeeBlockerBodies` is true (PM + Tech Lead on this project); `<LineageBadge>` shows skill version + generated-at; Regenerate button gated by autonomy + role
- [ ] `PrEmptyStates/` — canonical empty-state copies per design §8 (no checks, no reviews, AI review PENDING, AI review FAILED, head advanced since last AI review)

### A7: Build Run Detail View Cards

- [ ] `RunHeaderCard.vue` — run number, workflow name, `<BuildStatusBadge>`, trigger, actor, `<BranchPill>` + `<ShaPill>`, `<DurationLabel>`, linked `<StoryLinkStatusChip>` when derivable; deep-link chevron to GitHub run
- [ ] `RunJobsCard.vue` — list of jobs for this run; each row expandable to show its steps; per-job `<BuildStatusBadge>` + `<DurationLabel>`
- [ ] `RunStepsCard.vue` — per-job step list; failing step expanded by default; `<LogExcerptBlock>` for the failing step
- [ ] `RunLogsCard.vue` — last 200 lines of combined run log (redacted); jump-to-failing-step control
- [ ] `RunAiTriageCard.vue` — list of `<AiTriageRowCard>`; Retry per-row gated by autonomy + role; one FAILED_EVIDENCE row never blocks others; `<LineageBadge>`
- [ ] `RunRerunCard.vue` — Re-run button gated by autonomy + role; shows last re-run attempt result + rate-limit guardrail; Re-run is **workflow-level** only in V1 (no single-job re-run)

### A8: Build Traceability View Cards

- [ ] `TraceabilityStoryRowCard.vue` — story id, title, project, state, linked commits + builds count, worst link status; expands to show each commit row
- [ ] `TraceabilityCommitRowCard.vue` — inside expanded story row: `<ShaPill>`, short message, author, repo, `<StoryLinkStatusChip>`, linked build chevron
- [ ] `TraceabilityUnknownStoryCard.vue` — commits that reference a story id that does not resolve; sortable by repo / age; provides a one-click "Open in Requirement slice to verify" deep-link
- [ ] `TraceabilityNoStoryIdCard.vue` — commits to protected branches with no `Story-Id:` trailer; surfaces a gentle nudge copy per design §8
- [ ] `TraceabilityFilterBar.vue` — filter by project / story state / link status / date range; sort by worst-first
- [ ] `TraceabilitySummaryCard.vue` — big-number counts across the 4 `StoryLinkStatus` buckets (KNOWN / UNKNOWN_STORY / NO_STORY_ID / AMBIGUOUS)

### A9: Build Pinia Store and API Client

- [ ] `frontend/src/features/code-build-management/api/codeBuildApi.ts` using `fetchJson<T>()` — one function per endpoint from API guide §2 (Catalog, Repo, PR, Run, Traceability aggregate + per-card GETs + `regenerateAiPrReview`, `regenerateAiTriage`, `rerunRun`)
- [ ] `frontend/src/features/code-build-management/stores/codeBuildStore.ts` with:
  - [ ] `initCatalog(filters?)`, `refreshCatalogCard(cardKey)`
  - [ ] `openRepo(repoId)`, `refreshRepoCard(cardKey)`, `closeRepo()`
  - [ ] `openPr(prId)`, `refreshPrCard(cardKey)`, `closePr()`
  - [ ] `openRun(runId)`, `refreshRunCard(cardKey)`, `closeRun()`
  - [ ] `initTraceability(filters?)`, `refreshTraceabilityCard(cardKey)`
  - [ ] Mutations: `regenerateAiPrReview(prId, {prevHeadSha, reason})`, `regenerateAiTriage(runId, {stepIds?, reason})`, `rerunRun(runId, {reason})`
  - [ ] `prevHeadSha` tracking for PR AI regenerate: stale → surface `CB_STALE_HEAD_SHA` as refresh-prompt toast that refetches PR aggregate
  - [ ] `reset()` on unmount
- [ ] Mock toggle: `import.meta.env.DEV && !import.meta.env.VITE_USE_BACKEND`
- [ ] Per-card retry preserves other cards' state (`SectionResult<T>` per card)
- [ ] Mutations update only the affected card's state (no full aggregate refetch); stale head advance forces aggregate refetch

### A10: Build Top-Level Views

- [ ] `CatalogView.vue` — top-level route view for `/code-build-management`; mounts 3 catalog cards in 12-column grid per design §3.1; breadcrumb `Dashboard / Code & Build`; AI panel scoped to catalog
- [ ] `RepoDetailView.vue` — top-level route view for `/code-build-management/repos/:repoId`; mounts 7 repo cards in 12-column grid per design §3.2; breadcrumb `Dashboard / Code & Build / {owner}/{repo}`
- [ ] `PrDetailView.vue` — top-level route view for `/code-build-management/prs/:prId`; mounts 5 PR cards in 12-column grid per design §3.3; breadcrumb `Dashboard / Code & Build / PR #{number} — {title}`
- [ ] `RunDetailView.vue` — top-level route view for `/code-build-management/runs/:runId`; mounts 6 run cards in 12-column grid per design §3.4; breadcrumb `Dashboard / Code & Build / Run #{number}`
- [ ] `TraceabilityView.vue` — top-level route view for `/code-build-management/traceability`; mounts story rows + summary + unknown-story + no-story-id cards per design §3.5; breadcrumb `Dashboard / Code & Build / Traceability`

### A11: Routing and Navigation Wiring

- [ ] Add `/code-build-management` → `CatalogView`
- [ ] Add `/code-build-management/traceability` → `TraceabilityView` (**must be registered before** the `:repoId` wildcard pattern per design §6)
- [ ] Add `/code-build-management/repos/:repoId` → `RepoDetailView`
- [ ] Add `/code-build-management/prs/:prId` → `PrDetailView`
- [ ] Add `/code-build-management/runs/:runId` → `RunDetailView`
- [ ] Remove Code & Build Management `comingSoon` flag from shared shell nav config
- [ ] Deep-link: Catalog tile → Repo Detail
- [ ] Deep-link: Repo recent runs row → Run Detail; Repo recent PRs row → PR Detail; Repo recent commits row → GitHub commit (external)
- [ ] Deep-link: PR checks row → Run Detail; PR commit row → GitHub commit
- [ ] Deep-link: Run header → GitHub run (external); Run jobs → GitHub job (external)
- [ ] Deep-link: Traceability KNOWN row → Requirement slice story; UNKNOWN_STORY row → Requirement slice search
- [ ] All external chevrons use `rel="noopener noreferrer"` per REQ-CB-79
- [ ] Ensure keyboard navigation and ARIA labels (tab order per card, Enter to drill, Esc to close panels, tooltip-exposed status derivation)

### A12: States and Error Isolation

- [ ] Per-card skeletons during load (all five views)
- [ ] Per-card error with retry; preserves other cards' state
- [ ] Per-card empty states (distinct copy per card)
- [ ] Page-level error for access denial (`CB_WORKSPACE_FORBIDDEN`), unresolvable entity id, or 404
- [ ] Mutation errors surfaced via toast + inline cues:
  - `CB_AI_UNAVAILABLE` → retry CTA in Triage / PR Review cards
  - `CB_GH_RATE_LIMIT` → `<RateLimitBanner>` with reset-at
  - `CB_TRIAGE_EVIDENCE_MISMATCH` → row marked `FAILED_EVIDENCE`; other rows unaffected
  - `CB_AI_AUTONOMY_INSUFFICIENT` → Regenerate disabled with tooltip copy
  - `CB_ROLE_REQUIRED` → Regenerate / Re-run hidden entirely (never disabled-but-visible — prevents role discovery)
  - `CB_STALE_HEAD_SHA` → refresh-prompt toast that refetches PR aggregate
- [ ] AI PR review BLOCKER bodies redacted for non-PM / non-Tech-Lead: `<AiNoteCountsBar>` always shows counts; bodies replaced with `Visible to PM + Tech Lead on this project`
- [ ] Webhook-stale banner: when viewed entity is in a non-terminal state and `last_synced_at > 10 min ago`, show `<WebhookStaleBanner>` with a manual refresh action (triggers GET on the per-card endpoint; never a rerun)

### A13: Validate Code & Build Management Frontend

- [ ] `npm run dev` — browses all five routes (`/code-build-management`, `/repos/repo-1`, `/prs/pr-42`, `/runs/run-991`, `/traceability`)
- [ ] `npm run build` passes
- [ ] Visual spot-check against design §3 layouts
- [ ] Catalog → Repo → Run and Catalog → Repo → PR drill work; back-navigation preserves filters
- [ ] Every mutation round-trips through mock `commandLoop.ts`; all 9 error codes (`CB_AI_UNAVAILABLE`, `CB_GH_RATE_LIMIT`, `CB_TRIAGE_EVIDENCE_MISMATCH`, `CB_AI_AUTONOMY_INSUFFICIENT`, `CB_ROLE_REQUIRED`, `CB_WORKSPACE_FORBIDDEN`, `CB_UNKNOWN_STORY`, `CB_WEBHOOK_SIGNATURE_INVALID`, `CB_STALE_HEAD_SHA`) are reachable and surfaced correctly
- [ ] BLOCKER PR review body redaction works: non-PM / non-Tech-Lead account sees only counts
- [ ] Log excerpt pre-redacted fixtures never show raw secrets in the DOM or console
- [ ] Traceability view: KNOWN / UNKNOWN_STORY / NO_STORY_ID / AMBIGUOUS all reachable; UNKNOWN_STORY row deep-links to Requirement slice search
- [ ] Vitest unit tests pass for: primitives, store actions (including `prevHeadSha` fencing), mock commandLoop, `StoryLinkStatus` derivation, role-gating for BLOCKER bodies

### Phase A Definition of Done

- All 5 views (Catalog, Repo Detail, PR Detail, Run Detail, Traceability) render with realistic mock data covering all status / severity / link-status states
- Per-card loading / error / empty states behave correctly; stale-webhook banner surfaces when applicable
- All 3 mutations (PR AI regenerate, Run triage regenerate, Run rerun) round-trip through mock `commandLoop.ts`; all 9 error codes reachable and surfaced correctly
- BLOCKER PR review body role-gating verified: `AiNoteCountsBar` always shown; bodies only rendered for PM + Tech Lead
- Log excerpt rendering never exposes raw secrets (only redacted markers)
- Build passes; no console errors; mock webhook fixtures round-trip cleanly

---

## Phase B: Backend (Claude Code)

### B0: Shared Infrastructure (Prerequisite)

- [x] Verify `com.sdlctower.shared.dto.ApiResponse` exists; reuse for all endpoints
- [x] Verify `com.sdlctower.shared.dto.SectionResultDto` exists; reuse for aggregate section envelopes
- [x] Add `CODE_BUILD_MANAGEMENT` constant to `com.sdlctower.shared.ApiConstants` with base path `/api/v1/code-build-management`
- [ ] Confirm `StoryLookup` facade in `com.sdlctower.shared.integration.requirement` returns `(storyId, title, projectId, state)`; if missing, stub and flag as upstream blocker
- [x] Confirm `WorkspaceAutonomyLookup` facade exists (reused from Design Management); if missing, coordinate before merge
- [x] Confirm `LineageEmitter`, `AuditLogEmitter`, `SkillInvocationEmitter` available in shared
- [x] Confirm `ProjectRoleLookup` facade exists and returns `(memberId, projectId) -> Optional<ProjectRole>`; BLOCKER PR review body gating depends on it

### B1: Create Code & Build Management Package Structure

- [x] Create `com.sdlctower.domain.codebuildmanagement` with sub-packages:
  - `controller/` — `CodeBuildController` (5 read + 3 mutate endpoints) and `GithubWebhookController` (one `POST /webhooks/github` endpoint)
  - `service/` — `CatalogService`, `RepoDetailService`, `PrDetailService`, `RunDetailService`, `TraceabilityService`, `AiPrReviewCommandService`, `AiTriageCommandService`, `RunRerunCommandService`
  - `projection/` — all read-path projections
  - `ingestion/` — `GithubWebhookSignatureVerifier`, `GithubEventParser`, `IngestionDispatcher`, `IngestionOutboxWorker`, `EventPayloadNormalizer`
  - `integration/` — `GithubAppAuthProvider`, `GithubRestClient`, `StoryLookupAdapter`, `WorkspaceAutonomyAdapter`, `ProjectRoleAdapter`, `AiSkillClient`
  - `policy/` — `CodeBuildAccessGuard`, `CodeBuildAiAutonomyPolicy`, `RunRerunRatePolicy`, `StoryIdExtractor`, `LogRedactor`, `AiTriageEvidenceIntegrityPolicy`, `BlockerVisibilityPolicy`, `HeadShaFencingPolicy`
  - `dto/` — records per data-model §3
  - `events/` — `CodeBuildChangeLogPublisher`, lineage / audit / skill-invocation emitters
  - `persistence/` — JPA entities + repositories
  - `resync/` — `InstallBackfillService`, `NightlyResyncJob`
  - `facade/` — `BuildHealthFacade` (cross-slice projection facade consumed by Dashboard + Project Management)
- [x] No cross-cutting logic leaks into `controller/`; controller calls service, never projection/repository directly
- [x] No direct JPA association from Code & Build Management to Requirement / Project tables — all cross-slice reads via facades

### B2: Create Code & Build Management DTOs

- [x] All DTOs per data-model §4 as Java 21 records
- [x] All enums per data-model §2.2 mirrored as Java enums with JSON codecs (`RepoVisibility`, `PrState`, `RunStatus`, `RunTrigger`, `StepConclusion`, `StoryLinkStatus`, `AiNoteSeverity`, `AiRowStatus`, `ChangeLogEntryType`)
- [x] Command request records: `RegenerateAiPrReviewRequest(prevHeadSha, reason)`, `RegenerateAiTriageRequest(stepIds?, reason)`, `RerunRunRequest(reason)`
- [x] Command response records include updated `skillVersion`, `generatedAt`, plus the affected aggregate card's updated state
- [x] Reuse `MemberRefDto`, `ProjectRefDto`, `WorkspaceRefDto` from shared
- [x] `SectionResultDto<T>` wrapper per aggregate card
- [ ] `GithubWebhookEnvelope` value type carrying `(deliveryId, eventType, signatureHeader, rawBody, receivedAt)`

### B3: Implement Projections

- [x] `CatalogSummaryProjection` — totals + status-bucket counts across repos the caller can read
- [x] `CatalogRepoGridProjection` — repo rows grouped by project; per-repo last-default-branch-run computed via join with `pipeline_run`
- [x] `RepoHeaderProjection` — identity + workspace + project + last-synced-at + visibility
- [x] `RepoRecentRunsProjection` — last 25 runs on this repo, any branch, reverse-chron
- [x] `RepoRecentPrsProjection` — last 15 PRs on this repo; joins to `ai_pr_review` for note counts
- [x] `RepoRecentCommitsProjection` — last 20 commits on default branch; joins to `commit_story_link` for `StoryLinkStatus`
- [x] `RepoBranchesProjection` — branches tracked (protected + default) with last-seen head
- [x] `RepoHealthSummaryProjection` — 14-day success rate %, median duration, top 3 failing workflows (pure aggregation; no persistence)
- [x] `RepoAiSummaryProjection` — latest `ai_workspace_summary` row scoped to `(repoId, skillVersion)`
- [x] `PrHeaderProjection` — identity + head sha + base / head refs + author + linked story
- [x] `PrChecksProjection` — `check_run` rows tied to the PR's head sha
- [x] `PrReviewsProjection` — human review list
- [x] `PrCommitsProjection` — commits in the PR with `StoryLinkStatus`
- [x] `PrAiReviewProjection` — latest `ai_pr_review` row for `(prId, headSha, skillVersion)`; applies `BlockerVisibilityPolicy` before returning bodies
- [x] `RunHeaderProjection` — identity + workflow + trigger + actor + head sha + branch + `StoryLinkStatus`
- [x] `RunJobsProjection` — jobs for this run
- [x] `RunStepsProjection` — steps per job; failing step eager-loads `LogExcerpt`
- [x] `RunLogsProjection` — combined tail (last 200 lines) derived from step log excerpts
- [x] `RunAiTriageProjection` — latest triage rows for `(runId, skillVersion)`; filters out rows that failed evidence integrity unless explicitly requested
- [x] `TraceabilityStoryRowsProjection` — stories in scope with joined commits + builds; batched `StoryLookup` fan-out
- [x] `TraceabilityUnknownStoryProjection` — `commit_story_link` rows with `UNKNOWN_STORY` status
- [x] `TraceabilityNoStoryIdProjection` — default-branch commits with no extracted story id
- [x] `TraceabilitySummaryProjection` — per-bucket counts across the 4 `StoryLinkStatus` buckets
- [x] All projections honour workspace isolation at the query level (filter by `workspace_id IN (…memberOf)`)

### B4: Implement Read Services

- [x] `CatalogService.loadAggregate(filters)` with parallel fan-out over catalog projections; per-projection 500ms timeout → `SectionResultDto(data=null, error=...)` on failure; never throws page-level
- [x] `CatalogService.load{Summary,Grid,Filters}` for per-card endpoints
- [x] `RepoDetailService.loadAggregate(repoId)` with parallel fan-out over 7 repo projections
- [x] `RepoDetailService.load{Header,RecentRuns,RecentPrs,RecentCommits,Branches,HealthSummary,AiSummary}` for per-card endpoints
- [x] `PrDetailService.loadAggregate(prId)` with parallel fan-out over 5 PR projections; applies `BlockerVisibilityPolicy.filter(reviewPayload, principal)` at service boundary
- [x] `PrDetailService.load{Header,Checks,Reviews,Commits,AiReview}` for per-card endpoints
- [x] `RunDetailService.loadAggregate(runId)` with parallel fan-out over 6 run projections
- [x] `RunDetailService.load{Header,Jobs,Steps,Logs,AiTriage,Rerun}` for per-card endpoints
- [x] `TraceabilityService.loadAggregate(filters)` with parallel fan-out over 4 traceability projections
- [x] All read methods emit zero writes; safe to retry; idempotent under repeat calls

### B5: Implement Command Services

- [x] `AiPrReviewCommandService.regenerate(prId, request, principal)`:
  - Resolves PR → workspaceId / projectId via join; `CodeBuildAccessGuard.requireRead` then `.requireAdmin(Role.PM | Role.TECH_LEAD)` → `CB_ROLE_REQUIRED`
  - `HeadShaFencingPolicy.check(prId, request.prevHeadSha)` → `CB_STALE_HEAD_SHA` (409) if mismatch
  - `CodeBuildAiAutonomyPolicy.requireAtLeast(workspaceId, SUPERVISED)` → `CB_AI_AUTONOMY_INSUFFICIENT`
  - Invokes `AiSkillClient.reviewPullRequest(...)`; on upstream 5xx → `CB_AI_UNAVAILABLE`; persists `AiPrReview` keyed by `(prId, headSha, skillVersion)`
  - Publishes `CodeBuildChangeLogEntry(AI_PR_REVIEW_REGENERATED)` + `LineageEvent` + `SkillInvocationEvent`
- [x] `AiTriageCommandService.regenerate(runId, request, principal)`:
  - `CodeBuildAccessGuard.requireRead` then `.requireAdmin(Role.PM | Role.TECH_LEAD)` → `CB_ROLE_REQUIRED`
  - `CodeBuildAiAutonomyPolicy.requireAtLeast(workspaceId, SUPERVISED)`
  - Builds prompt evidence set = `{(runId, jobId, stepId)}` for the run (or filtered by `request.stepIds`)
  - Invokes `AiSkillClient.triageRun(...)`; on upstream 5xx → `CB_AI_UNAVAILABLE`
  - `AiTriageEvidenceIntegrityPolicy.verify(response, evidence)` — every row must reference an id in the evidence set; rows that don't get `AiRowStatus.FAILED_EVIDENCE` and surface `CB_TRIAGE_EVIDENCE_MISMATCH` in the aggregate (never in the overall command result — mismatch is per-row, not command-level failure)
  - Persists `AiTriageRow` per row keyed by `(runId, skillVersion, stepId)`
  - Publishes `CodeBuildChangeLogEntry(AI_TRIAGE_REGENERATED)` + `LineageEvent` + `SkillInvocationEvent`
- [x] `RunRerunCommandService.rerun(runId, request, principal)`:
  - `CodeBuildAccessGuard.requireAdmin(Role.PM | Role.TECH_LEAD)`
  - `RunRerunRatePolicy.check(repoId)` — protects against GitHub rate limit; returns `CB_GH_RATE_LIMIT` on hit
  - Calls `GithubRestClient.rerunWorkflow(runId)`; persists a `PipelineRun` lineage entry linking original → new run
  - Publishes `CodeBuildChangeLogEntry(RUN_RERUN_REQUESTED)` + `LineageEvent`
- [x] All command services enforce workspace isolation → `CB_WORKSPACE_FORBIDDEN` on cross-workspace ids
- [x] Every successful mutation publishes `CodeBuildChangeLogEntry` + `LineageEvent` via shared emitters

### B6: Implement Policies, Security Utilities, and Extractors

- [x] `CodeBuildAccessGuard.requireRead(projectId, principal)` — workspace membership check
- [x] `CodeBuildAccessGuard.requireAdmin(projectId, principal, allowedRoles)` — role is PM or Tech Lead on that project
- [x] `CodeBuildAiAutonomyPolicy.requireAtLeast(workspaceId, level)` — reuses `WorkspaceAutonomyLookup`
- [x] `RunRerunRatePolicy` — per-repo sliding window; default 5 re-runs / hour / repo; rate limit returns `CB_GH_RATE_LIMIT`
- [x] `HeadShaFencingPolicy.check(prId, clientPrevHeadSha)` — compares to `pull_request.head_sha`
- [x] `AiTriageEvidenceIntegrityPolicy.verify(aiResponseRows, evidenceIds)` — set membership check per row
- [x] `BlockerVisibilityPolicy.filter(aiPrReviewPayload, principal, projectId)` — if role ∉ {PM, TECH_LEAD}: preserve counts (`blocker`, `major`, `minor`, `info`), strip BLOCKER bodies entirely, leave MAJOR / MINOR / INFO bodies intact
- [x] `StoryIdExtractor` — regex:
  ```java
  private static final Pattern TRAILER = Pattern.compile(
      "^\\s*(?:Story-Id|Relates-to)\\s*:\\s*(STORY-[A-Z0-9\\-]+)\\s*$",
      Pattern.MULTILINE);
  ```

  - Applied to commit message body AND PR body; first match wins; returns `Optional<String>`
  - If no match → `StoryLinkStatus.NO_STORY_ID`
  - If match but `StoryLookup` returns empty → `StoryLinkStatus.UNKNOWN_STORY` (persists the candidate id for nightly re-resolve)
  - If multiple distinct matches → `StoryLinkStatus.AMBIGUOUS`
  - Otherwise → `StoryLinkStatus.KNOWN`
- [x] `LogRedactor` — applied at ingestion time (never to already-redacted content); patterns:
  - `AKIA[0-9A-Z]{16}` → `***REDACTED:AWS_KEY***`
  - `ghp_[A-Za-z0-9]{36}` → `***REDACTED:GH_TOKEN***`
  - `gho_[A-Za-z0-9]{36}` → `***REDACTED:GH_TOKEN***`
  - `ghs_[A-Za-z0-9]{36}` → `***REDACTED:GH_TOKEN***`
  - `(?i)Bearer\\s+[A-Za-z0-9._\\-]{20,}` → `Bearer ***REDACTED:BEARER***`
  - Redactor runs BEFORE truncation; truncation enforces ≤1 MiB per step via `CHECK (byte_count <= 1048576)`
- [x] Exception classes: `CodeBuildAccessDeniedException`, `StaleHeadShaException`, `AiAutonomyInsufficientException`, `AiUnavailableException`, `RateLimitExceededException`, `TriageEvidenceMismatchException`, `WebhookSignatureInvalidException`, `UnknownStoryException` (non-blocking), `RepoNotFoundException`, `PullRequestNotFoundException`, `PipelineRunNotFoundException`
- [x] `@ExceptionHandler` entries in `GlobalExceptionHandler` mapping each to its error code + HTTP status per API guide §4

### B7: Implement GitHub Integration Layer

- [ ] `GithubAppAuthProvider` — generates App JWT via RSA private key (from `CB_GH_APP_PRIVATE_KEY_PEM` env); exchanges for installation token per workspace installation id; caches token with expiry – 60s safety margin
- [ ] `GithubRestClient` — minimal REST client covering `POST /repos/{owner}/{repo}/actions/runs/{run_id}/rerun`, `GET /repos/{owner}/{repo}/pulls/{number}`, `GET /repos/{owner}/{repo}/actions/runs/{run_id}`, `GET /repos/{owner}/{repo}/actions/runs/{run_id}/jobs`, `GET /repos/{owner}/{repo}/actions/jobs/{job_id}/logs` — used by backfill, resync, rerun, and log fetch on-demand
- [ ] Rate-limit observer — reads `X-RateLimit-Remaining` / `X-RateLimit-Reset` on every response; stores in `RateLimitRegistry` used by `RunRerunRatePolicy`
- [ ] `StoryLookupAdapter` / `WorkspaceAutonomyAdapter` / `ProjectRoleAdapter` — thin wrappers around cross-slice facades
- [ ] `AiSkillClient` — invokes `reviewPullRequest`, `triageRun`, `summarizeRepo` via the shared AI gateway; returns skill version + generation timestamp; 60s timeout; surfaces 5xx as `AiUnavailableException`

### B8: Implement Ingestion Pipeline

- [x] `GithubWebhookController.POST /webhooks/github`:
  - Reads raw body (must preserve original bytes — use `HttpServletRequest.getInputStream()` via a `ContentCachingRequestWrapper`)
  - Extracts `X-Hub-Signature-256`, `X-GitHub-Event`, `X-GitHub-Delivery`
  - `GithubWebhookSignatureVerifier.verify(rawBody, signatureHeader, installationSecret)` using HMAC-SHA-256 with constant-time compare → `CB_WEBHOOK_SIGNATURE_INVALID` (401) on fail
  - Persists an `IngestionOutbox` row `(deliveryId UNIQUE, eventType, rawBody, receivedAt, status=PENDING)` — uniqueness on `deliveryId` provides idempotency
  - Returns `202 Accepted` immediately (webhook SLO is <2s per GitHub retry policy)
- [ ] `IngestionOutboxWorker` — Spring `@Scheduled` (or drained via worker executor) — picks `PENDING` rows, marks `IN_PROGRESS`, dispatches to `IngestionDispatcher`, marks `DONE` or `FAILED` with error payload; at-least-once delivery + row uniqueness = effectively-once processing
- [ ] `IngestionDispatcher` — routes by event type to a handler per data-flow §7:
  - `push` → `PushEventHandler` (upserts commits, applies `StoryIdExtractor`, writes `commit_story_link`)
  - `pull_request` (opened / synchronize / closed / reopened) → `PullRequestEventHandler` (upserts `pull_request`; synchronize bumps `head_sha`, invalidates AI PR review)
  - `pull_request_review` → `PullRequestReviewEventHandler` (upserts `pull_request_review`)
  - `workflow_run` → `WorkflowRunEventHandler` (upserts `pipeline_run`)
  - `workflow_job` → `WorkflowJobEventHandler` (upserts `pipeline_job`)
  - `check_run` → `CheckRunEventHandler`
  - `installation` / `installation_repositories` → `InstallationEventHandler` (triggers `InstallBackfillService` on create/add)
  - Unknown event type → log + mark `DONE` (forward-compatible; never `FAILED`)
- [ ] `EventPayloadNormalizer` — converts GitHub JSON into internal DTOs consumed by each handler
- [ ] Every handler applies `LogRedactor` when persisting log excerpts (workflow_job completion fetches logs on-demand via `GithubRestClient` and redacts before persist)
- [ ] Every handler updates `repo.last_synced_at` at completion
- [ ] Handlers emit `CodeBuildChangeLogEntry` via `CodeBuildChangeLogPublisher` for auditable state transitions (run completed, PR head advance, installation linked)
- [ ] Head-advance cascade: on `pull_request.synchronize` event, mark existing `AiPrReview` rows for that PR as stale (`invalidated_at` = now); next aggregate read shows stale hint; admin regenerate repopulates

### B9: Implement Resync + Backfill

- [ ] `InstallBackfillService.backfill(installationId)`:
  - Triggered on `installation.created` or `installation_repositories.added`
  - Enumerates repos in the installation
  - For each repo: fetch last 14 days of commits on default branch, open PRs, recent workflow runs (up to 14 days)
  - Batches upserts; applies `StoryIdExtractor`, `LogRedactor`
  - Idempotent under repeat via primary-key upserts + `deliveryId`-equivalent synthetic key per batch
- [ ] `NightlyResyncJob` — Spring `@Scheduled(cron = "0 30 2 * * *")` (02:30 workspace-local TZ; UTC for V1):
  - Walks installations; for each: fetches recent runs / PRs modified in last 72h
  - Reconciles delta against DB; any row observed on GitHub but missing locally → triggers a targeted handler path (no duplicate change-log entry if already present)
  - Emits a `ResyncRun` record to the change log for auditability

### B10: Implement Controllers

- [x] `CodeBuildController` with endpoints per API guide §2–§3:
  - GET `/catalog` — aggregate
  - GET `/catalog/summary`, `/catalog/grid` — per-card
  - GET `/repos/{repoId}` — repo detail aggregate
  - GET `/repos/{repoId}/header`, `/recent-runs`, `/recent-prs`, `/recent-commits`, `/branches`, `/health-summary`, `/ai-summary` — per-card
  - GET `/prs/{prId}` — PR detail aggregate (applies `BlockerVisibilityPolicy` at service boundary)
  - GET `/prs/{prId}/header`, `/checks`, `/reviews`, `/commits`, `/ai-review` — per-card
  - GET `/runs/{runId}` — run detail aggregate
  - GET `/runs/{runId}/header`, `/jobs`, `/steps`, `/logs`, `/ai-triage`, `/rerun` — per-card
  - GET `/traceability` — aggregate
  - GET `/traceability/story-rows`, `/unknown-story`, `/no-story-id`, `/summary` — per-card
  - POST `/prs/{prId}/ai-review/regenerate` — regen
  - POST `/runs/{runId}/ai-triage/regenerate` — regen
  - POST `/runs/{runId}/rerun` — rerun
- [x] `@Pattern` path validation on `repoId` (`^repo-[a-z0-9\-]+$`), `prId` (`^pr-[a-z0-9\-]+$`), `runId` (`^run-[a-z0-9\-]+$`)
- [x] `@RequestBody @Valid` on all command endpoints
- [x] Standard `ApiResponse<T>` envelope wrap via shared `ResponseBodyAdvice` or explicit `ApiResponse.ok(...)` calls
- [x] Mutation endpoints return `200 OK` with updated entity state; never `204`
- [x] `GithubWebhookController.POST /webhooks/github` returns `202 Accepted`; never returns the envelope — GitHub expects an empty body

### B11: Create Flyway Migrations

- [x] `V40__create_code_build_core.sql` — `repo`, `branch`, `commit`, `pull_request`; indexes on `(workspace_id, project_id)`, `(repo_id, created_at DESC)`, `(pull_request.head_sha)`, `(repo_id, default_branch)`
- [x] `V41__create_pipeline_entities.sql` — `pipeline_run`, `pipeline_job`, `pipeline_step`, `check_run`; indexes on `(repo_id, head_sha)`, `(repo_id, created_at DESC)`, `(run_id, job_number)`, `(job_id, step_number)`
- [x] `V42__create_log_excerpt.sql` — `log_excerpt` with `CHECK (byte_count <= 1048576)`; `text` as `CLOB` (H2) / `CLOB` (Oracle); `step_id` FK; index on `(step_id)`
- [x] `V43__create_pull_request_review.sql` — `pull_request_review` rows with reviewer, state, body, created_at
- [x] `V44__create_story_link.sql` — `commit_story_link`, `pr_story_link`; unique `(commit_sha, story_id)` and `(pr_id, story_id)`; index on `(story_id)`, `(link_status)`
- [x] `V45__create_ai_outputs.sql` — `ai_pr_review` (unique `(pr_id, head_sha, skill_version)`), `ai_triage_row` (unique `(run_id, step_id, skill_version)`), `ai_workspace_summary` (unique `(workspace_id, repo_id, skill_version)`); indexes on `(skill_version)`, `(generated_at DESC)`
- [x] `V46__create_ingestion_outbox_and_changelog.sql` — `ingestion_outbox` with UNIQUE `delivery_id`, index on `(status, received_at)`; `code_build_change_log` with index on `(entity_type, entity_id, created_at DESC)`
- [x] `V47__seed_code_build_local.sql` — local-only seed: ~4 repos across 2 workspaces, ~12 branches, ~80 commits (3 without `Story-Id`, 2 with UNKNOWN_STORY candidate), ~20 PRs (4 with AI reviews covering all severity mixes), ~40 pipeline runs (mix of SUCCESS / FAILURE / IN_PROGRESS / CANCELLED), ~6 triage rows (1 FAILED_EVIDENCE), ~4 AI workspace summaries
- [x] Verify all migrations run on H2 (local) without error
- [ ] Produce Oracle variant if DDL diverges (notably `BLOB` / `CLOB`, CHECK constraint syntax, identity columns) under `db/migration/oracle/`

### B12: Implement Entities and Repositories

- [x] JPA entities (18 net-new per data-model §3): `Repo`, `Branch`, `Commit`, `PullRequest`, `PullRequestReview`, `PipelineRun`, `PipelineJob`, `PipelineStep`, `CheckRun`, `LogExcerpt`, `CommitStoryLink`, `PrStoryLink`, `AiPrReview`, `AiPrReviewNote`, `AiTriageRow`, `AiWorkspaceSummary`, `IngestionOutbox`, `CodeBuildChangeLogEntry`
- [x] Map `LogExcerpt.text` as `@Lob String` (CLOB) with `FetchType.LAZY` — never eagerly loaded via catalog/repo/PR projections (only Run Detail)
- [x] Map `IngestionOutbox.rawBody` as `@Lob byte[]` with `FetchType.LAZY`
- [x] Repositories: one per entity; representative query methods:
  - `RepoRepository.findByWorkspaceIdIn(workspaceIds, Pageable)`
  - `CommitRepository.findTop20ByRepoIdAndBranchOrderByCommittedAtDesc(repoId, branch)`
  - `PullRequestRepository.findTop15ByRepoIdOrderByUpdatedAtDesc(repoId)`
  - `PipelineRunRepository.findTop25ByRepoIdOrderByCreatedAtDesc(repoId)`
  - `PipelineRunRepository.findByRepoIdAndHeadSha(repoId, headSha)` — used by PR checks rollup
  - `PipelineJobRepository.findByRunIdOrderByJobNumberAsc(runId)`
  - `PipelineStepRepository.findByJobIdOrderByStepNumberAsc(jobId)`
  - `LogExcerptRepository.findByStepId(stepId)`
  - `CommitStoryLinkRepository.findByStoryIdIn(storyIds)` — for traceability
  - `CommitStoryLinkRepository.findByLinkStatus(UNKNOWN_STORY, Pageable)` — nightly re-resolver
  - `AiPrReviewRepository.findTopByPrIdAndHeadShaOrderByGeneratedAtDesc(prId, headSha)`
  - `AiTriageRowRepository.findByRunIdAndSkillVersion(runId, skillVersion)`
  - `IngestionOutboxRepository.findByDeliveryId(deliveryId)` — idempotency check
  - `IngestionOutboxRepository.lockAndClaimOldest(batch)` — worker drain with row lock
- [x] Ensure all cross-slice reads (`StoryLookup`, `WorkspaceAutonomyLookup`, `ProjectRoleLookup`) go through facades — no direct JPA association from Code & Build Management to Requirement / Project tables

### B13: Governance, Lineage, and Cross-Slice Facade

- [ ] Every write path writes a `LineageEvent` identifying Code & Build Management as the authoring domain (reuse shared `LineageEmitter`)
- [ ] Every mutation endpoint writes an `AuditLogEntry` via shared `AuditLogEmitter`
- [ ] AI regenerate paths emit `SkillInvocation` events (invoker, autonomy level at time of call, skill version, latency, outcome, request token count, evidence integrity outcome)
- [x] `BuildHealthFacade` — cross-slice read facade exposed to Dashboard + Project Management:
  - `healthByProject(projectId)` → `{successRate14d, medianDuration, lastRunStatus, failingWorkflows}`
  - `healthByWorkspace(workspaceId)` → workspace-level rollup
  - `recentFailuresByWorkspace(workspaceId, since)` → list for dashboard failure feed
- [ ] Confirm Dashboard's "recent build activity" tile (if present) reads via `BuildHealthFacade` rather than the tables directly
- [ ] Confirm Project Management's project health badge reads via `BuildHealthFacade`

### B14: Backend Tests

- [ ] `CodeBuildControllerTest` (MockMvc):
  - Catalog aggregate happy + per-projection error isolation
  - Repo detail aggregate happy + 404
  - PR detail aggregate happy + BLOCKER body redaction for non-PM/TL principal + full bodies for PM principal
  - Run detail aggregate happy + triage evidence-mismatch row present + mismatch doesn't fail aggregate
  - Traceability aggregate happy + UNKNOWN_STORY bucket populated
  - Every mutation endpoint: happy + role denied + stale head sha + AI unavailable + autonomy insufficient + rate limit + validation failure
  - Workspace isolation: cross-workspace id → 403 `CB_WORKSPACE_FORBIDDEN`
- [ ] `GithubWebhookControllerTest` (MockMvc):
  - Valid signature → 202 + outbox row created
  - Invalid signature → 401 `CB_WEBHOOK_SIGNATURE_INVALID`
  - Duplicate `delivery_id` → 202 but no second outbox row (idempotent)
  - Missing `X-Hub-Signature-256` → 401
  - Unsupported event type → 202 + dispatcher marks DONE without touching domain tables
- [ ] `IngestionDispatcherTest`: one test per event type asserting handler invocation + correct upsert behavior
- [ ] `AiPrReviewCommandServiceTest`: happy + head-sha stale + autonomy + role + AI 5xx retry
- [ ] `AiTriageCommandServiceTest`: happy + evidence mismatch row marked + AI 5xx + autonomy
- [ ] `RunRerunCommandServiceTest`: happy + rate limit + role + GitHub 4xx passthrough
- [x] `StoryIdExtractorTest`: trailer present / absent / multiple distinct / UNKNOWN candidate → persists with UNKNOWN_STORY
- [x] `LogRedactorTest`: each pattern (AKIA, ghp_, gho_, ghs_, Bearer) / false positives (literal `bearer` in log text without token) / nested redaction idempotency
- [ ] `BlockerVisibilityPolicyTest`: PM sees bodies, Tech Lead sees bodies, Developer sees counts only
- [ ] `AiTriageEvidenceIntegrityPolicyTest`: row with valid ids passes; row referencing unknown `stepId` is marked `FAILED_EVIDENCE`; mixed set produces partial output
- [ ] `HeadShaFencingPolicyTest`: stale → `CB_STALE_HEAD_SHA`
- [ ] `CodeBuildAccessGuardTest`: read vs admin gating; cross-workspace blocked
- [ ] `CodeBuildAiAutonomyPolicyTest`: DISABLED / OBSERVATION / SUPERVISED / AUTONOMOUS gating
- [x] `RunRerunRatePolicyTest`: sliding window; limit hit → `CB_GH_RATE_LIMIT`
- [ ] `InstallBackfillServiceTest`: idempotent under repeat; honours 14-day window; all events go through handlers (no direct DB writes)
- [ ] `NightlyResyncJobTest`: delta reconciliation; no duplicate change log for already-seen state
- [ ] Repository tests against H2 for every entity + edge queries
- [ ] `FlywayMigrationIntegrationTest`: applies V40–V47 on a clean H2; verifies target schema (including the `log_excerpt.byte_count` CHECK rejects >1 MiB inserts; `ingestion_outbox.delivery_id` UNIQUE rejects duplicate) + seed counts
- [ ] Golden-file test for API envelopes — compare response JSON to canonical examples in API guide §3 (including the PR review role-gated shape for a non-PM principal)
- [ ] Webhook signature fixture test — canned body + known secret + expected HMAC → verifier passes; flipped byte → verifier rejects with constant-time compare

### B15: Connect Frontend to Backend

- [ ] Set `VITE_USE_BACKEND=true` in dev config
- [ ] Verify Vite proxy routes `/api/v1/code-build-management/*` to backend
- [ ] Smoke: browse `/code-build-management` against live backend — 3 catalog cards hydrated from seed data
- [ ] Smoke: browse `/code-build-management/repos/{seedRepoId}` — 7 repo cards hydrated
- [ ] Smoke: browse `/code-build-management/prs/{seedPrId}` — 5 PR cards hydrated; BLOCKER bodies visible only for PM / Tech Lead principal
- [ ] Smoke: browse `/code-build-management/runs/{seedRunId}` — 6 run cards hydrated; log excerpts show pre-redacted secrets only
- [ ] Smoke: browse `/code-build-management/traceability` — story rows populated; UNKNOWN_STORY bucket includes seeded unresolved ids
- [ ] Smoke: execute one mutation of each kind:
  - AI PR review regenerate (PM principal) → new row keyed by current head sha; change log entry
  - AI triage regenerate (Tech Lead principal) → rows refreshed; one row exercises `FAILED_EVIDENCE`
  - Run rerun (PM principal) → new `PipelineRun` with lineage link to original
- [ ] Smoke: force stale head sha (simulate PR sync in a second tab) → verify refresh-prompt toast + aggregate refetch
- [ ] Smoke: attempt each error path:
  - Non-PM user tries regenerate → controls hidden (not merely disabled)
  - Cross-workspace PR id → `CB_WORKSPACE_FORBIDDEN`
  - Rate limit injected on rerun → `<RateLimitBanner>`
  - AI skill 5xx → `CB_AI_UNAVAILABLE` toast + Retry in card
- [ ] Verify deep-links: Catalog → Repo → Run; Catalog → Repo → PR; Traceability → Requirement story (KNOWN) and Requirement search (UNKNOWN_STORY); external chevrons open with `rel="noopener noreferrer"`

### B16: Validate Full Stack

- [ ] `./mvnw verify` passes (all new tests + existing suites)
- [ ] `npm run build` passes
- [ ] End-to-end flow on seed data:
  - Simulate webhook: `pull_request opened` → ingestion outbox → dispatcher → PR row appears on Catalog + Repo + PR detail
  - Simulate webhook: `pull_request synchronize` with new head sha → existing AI PR review marked stale; UI shows regenerate CTA
  - Regenerate AI PR review → row keyed by new `(prId, headSha, skillVersion)`
  - Simulate webhook: `workflow_run completed` with FAILURE conclusion + `workflow_job` with logs → run + steps + log excerpt appear redacted
  - Regenerate AI triage → 3 rows produced; one FAILED_EVIDENCE doesn't block the others
  - Rerun run (PM principal) → new run persisted with lineage link
  - Traceability view: story that matches one of the seeded `Story-Id` trailers appears under KNOWN; unresolved id appears under UNKNOWN_STORY
- [ ] Webhook idempotency: replay the same `delivery_id` → no second outbox row
- [ ] BLOCKER body role-gating: Developer principal sees counts only; PM principal sees bodies
- [ ] Install backfill on new installation: fetches 14 days; all data arrives via handlers (no raw DB writes); change log populated
- [ ] Nightly resync: simulate drift (delete a local run row) → next resync re-hydrates without duplicate change log
- [ ] Audit log contains entries for every mutation; Lineage graph shows Code & Build Management as authoring domain; SkillInvocation records include autonomy level
- [ ] CSP + `rel="noopener noreferrer"` verified on every external GitHub deep-link
- [ ] Per-projection 500ms timeout: inject a projection that sleeps 1s → the containing card returns `SectionResultDto(error=...)`; other cards unaffected; page-level status remains 200
- [ ] Dashboard / Project Management build health badges populate via `BuildHealthFacade` (no regressions in those slices)

### Phase B Definition of Done

- All 30+ read endpoints return correct envelopes for happy path
- All 3 mutation endpoints enforce role + head-sha fencing + autonomy + rate limit + evidence integrity
- Webhook receiver is signature-verified, idempotent on `delivery_id`, outbox-backed, and returns 202 within SLO
- Dispatcher routes every supported event type to the correct handler; unknown events are forward-compatible
- Log redaction runs at ingestion; raw secrets never persisted to `log_excerpt.text`
- Evidence integrity check surfaces mismatched rows as `FAILED_EVIDENCE` without failing the command
- BLOCKER review body visibility correctly gated to PM + Tech Lead via `BlockerVisibilityPolicy`
- Per-projection 500ms timeout degrades to section-level error only
- Flyway V40–V47 migrations run cleanly on H2; Oracle variants prepared under `db/migration/oracle/` if divergent
- Frontend works in live mode (`VITE_USE_BACKEND=true`)
- `BuildHealthFacade` serves Dashboard + Project Management cross-slice reads without direct table access
- Write authority for all 18 Code & Build entities is demonstrably owned by Code & Build Management (no other slice mutates these tables)

---

## Dependency Plan

### Critical Path

```
A0 (shared infra) → A1 (types) → A2 (mocks) → A3 (primitives)
   → A4 (catalog) + A5 (repo) + A6 (pr) + A7 (run) + A8 (traceability)  [parallel]
   → A9 (store/API) → A10 (views) → A11 (routing) → A12 (states) → A13 (validate)
        │
        ↓
B0 (shared backend) → B1 (package) → B2 (DTOs)
   → B3 (projections in parallel) → B4 (read services)
   → B6 (policies + extractors + redactor) → B7 (GitHub integration) → B5 (command services)
   → B8 (ingestion pipeline) → B9 (backfill / resync)
   → B10 (controllers) → B11 (Flyway V40-V47) → B12 (entities/repos)
   → B13 (governance / lineage / BuildHealthFacade) → B14 (tests)
   → B15 (connect FE) → B16 (validate full stack)
```

### Parallel Workstreams

- A4 (catalog), A5 (repo), A6 (pr), A7 (run), A8 (traceability) can be built in parallel by multiple contributors once A3 primitives land
- Mock commandLoop + webhook fixtures (A2) can be authored in parallel with A3 primitives
- B3 projections can be built in parallel once DTOs are defined
- B5 command services can be split per capability (PrReview / Triage / Rerun)
- B7 GitHub integration and B8 ingestion can proceed in parallel with B3/B4 once DTOs are defined
- Frontend Phase A can proceed independently of backend work up through A13
- Flyway migrations (B11) can be drafted in parallel with entity work (B12) but must land before entity tests pass
- `StoryIdExtractor` / `LogRedactor` / `BlockerVisibilityPolicy` / `AiTriageEvidenceIntegrityPolicy` (B6) can be built in parallel with everything else — pure utilities with self-contained tests

---

## Risks / Blockers

| Risk                                                                                   | Mitigation                                                                                                                                                                                                              |
| -------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `StoryLookup` facade not yet implemented in Requirement slice                        | Stub in `com.sdlctower.shared.integration.requirement` returning a static map for seed data; flag as upstream blocker; document cutover plan                                                                          |
| GitHub webhook secret rotation breaks signature verification mid-flight                | Support verification against both current and previous secret for a 24h rotation window; platform ops rotates during low-traffic window                                                                                 |
| Webhook payload size > configured body limit                                           | Raise Spring body limit to 25 MiB specifically for `/webhooks/github`; GitHub hard cap is 25 MiB per payload per their docs; reject larger with 413                                                                   |
| GitHub rate-limit exhausted by backfill                                                | Backfill is throttled to respect `X-RateLimit-Remaining`; exhausted → persist progress, resume after `X-RateLimit-Reset`; emit `BackfillPaused` change log entry                                                 |
| `LogRedactor` false positive strips legitimate tokens in docs                        | Redactor runs only on captured workflow job logs, never on arbitrary text; patterns tuned to GitHub + AWS key shapes; regex changes require a reviewed changelog entry                                                  |
| AI triage evidence set bypassed if skill returns hallucinated step ids                 | `AiTriageEvidenceIntegrityPolicy` verifies set membership per row; mismatched rows render as `FAILED_EVIDENCE` and never populate `triage.conclusion`; command-level contract unchanged                           |
| BLOCKER bodies leaked via per-card endpoint bypassing aggregate                        | `BlockerVisibilityPolicy.filter` invoked inside the service boundary — applies to both aggregate and per-card endpoints; golden-file test for non-PM principal verifies counts-only shape on `/prs/{id}/ai-review` |
| Stale PR AI review not invalidated when head advances                                  | `pull_request.synchronize` handler marks existing `AiPrReview` rows stale (`invalidated_at` column); UI surfaces a regenerate CTA; aggregate read filters stale rows as `SUCCESS_STALE`                         |
| Outbox backlog grows unbounded if dispatcher is down                                   | Alert when `outbox.status='PENDING' AND received_at < now() - 5 min` grows beyond threshold; manual drain runbook documented; retention keeps PENDING indefinitely until drained, DONE rows pruned after 30 days      |
| Oracle CLOB handling differs from H2 for `log_excerpt.text`                          | Oracle-variant migration planned under `db/migration/oracle/`; verified on Oracle-in-Docker before deployment                                                                                                         |
| Cross-slice churn when Requirement slice changes its story state enum                  | Story resolution isolated in `StoryLookupAdapter`; any semantic change requires updating one adapter + re-running `commit_story_link` re-resolver                                                                   |
| Shell-level nav for Code & Build Management still feature-flagged in some environments | A11 removes `comingSoon`; coordinate with shell owners before merge                                                                                                                                                   |
| Re-run abuse exhausts GitHub Actions minutes                                           | `RunRerunRatePolicy` per-repo sliding window; default 5 re-runs / hour / repo; additionally gated by PM / Tech Lead role; emits a governance event on every rerun                                                     |
| Admin mutations bypass workspace isolation when entity id is spoofed                   | `CodeBuildAccessGuard.requireAdmin` always resolves the workspace from the entity row itself; never trusts the request body's workspace id                                                                            |

---

## Open Questions

- Should a CANCELLED run count toward `health-summary` failure rate? Default: NO — cancels exclude from both numerator and denominator.
- AI PR review auto-regeneration on head advance — synchronous (block synchronize processing until AI ready) or fire-and-forget? Default: fire-and-forget; aggregate read shows stale hint until regenerate completes.
- Should UNKNOWN_STORY rows expire after N days? Default: NO in V1; nightly re-resolver keeps trying indefinitely; V2 can add expiry policy.
- Log excerpt retention — how long? Default: 90 days for SUCCESS runs, 365 days for FAILURE runs, retained forever for runs linked to an incident (cross-slice signal).
- Re-run single job vs whole workflow — V1 supports workflow only? Default: YES, workflow-level only in V1; single-job rerun deferred to V2 pending GitHub API stability.
- Should `BuildHealthFacade` cache results? Default: request-scoped only in V1; no cross-request caching to avoid staleness cascades.
- What happens when an installation is uninstalled? Default: mark installation row `DEACTIVATED`; retain historical data (runs, PRs, commits) read-only; block new webhook delivery; backfill + rerun paths return `CB_INSTALLATION_DEACTIVATED`.
- Support GitHub Enterprise Server in V1? Default: NO — GitHub.com only in V1; GHES deferred pending platform ops support.
- Should the webhook endpoint enforce IP allowlist on top of signature verification? Default: signature verification alone is sufficient per GitHub's own security guidance; IP allowlist adds brittleness with GitHub's rotating hook origins.
- How do we surface webhook stale (last_synced_at old) for non-terminal entities? Default: `<WebhookStaleBanner>` on Repo / PR / Run detail views when viewed entity is in a non-terminal state AND `last_synced_at > 10 min ago`; never on Catalog / Traceability.
