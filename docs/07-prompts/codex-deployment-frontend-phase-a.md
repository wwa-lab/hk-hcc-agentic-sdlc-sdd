# Codex Prompt: Deployment Management Phase A — Frontend Implementation

## Task

Implement the **Deployment Management** frontend slice (`/deployment`) with mocked data. This slice uses **Codex for both frontend and backend** (the user has explicitly chosen Codex over Gemini for this slice). No backend dependency in Phase A — everything goes through a mock command layer that exercises every documented state, error code, and polling flow.

V1 scope: a **read-only observability viewer** over deployments executed by **Jenkins**. The Control Tower does NOT trigger, approve, promote, cancel, or roll back Jenkins builds. The frontend observes state and renders it. The only write paths are AI regenerate (release notes / deploy summary / workspace summary).

## Read First (do not skip)

Read these docs fully before writing any code:

1. **Feature contract** — [`docs/03-spec/deployment-management-spec.md`](../03-spec/deployment-management-spec.md)
2. **Design decisions** — [`docs/05-design/deployment-management-design.md`](../05-design/deployment-management-design.md) (file structure, routing, component API contracts, store shape, visual tokens, empty/error states, Phase A/B toggle)
3. **API contract** (shapes your mocks must satisfy byte-for-byte) — [`docs/05-design/contracts/deployment-management-API_IMPLEMENTATION_GUIDE.md`](../05-design/contracts/deployment-management-API_IMPLEMENTATION_GUIDE.md)
4. **Data model** (TypeScript types verbatim — mirror these in `types/`) — [`docs/04-architecture/deployment-management-data-model.md`](../04-architecture/deployment-management-data-model.md)
5. **Data flow** (state transitions, polling lifecycles, refresh strategy) — [`docs/04-architecture/deployment-management-data-flow.md`](../04-architecture/deployment-management-data-flow.md)
6. **Task breakdown** (order of operations) — [`docs/06-tasks/deployment-management-tasks.md`](../06-tasks/deployment-management-tasks.md) § Phase A
7. **Stories** (acceptance criteria) — [`docs/02-user-stories/deployment-management-stories.md`](../02-user-stories/deployment-management-stories.md)
8. **Requirements** (REQ-DP-*) — [`docs/01-requirements/deployment-management-requirements.md`](../01-requirements/deployment-management-requirements.md)
9. **Architecture** (component breakdown, state boundaries, integration) — [`docs/04-architecture/deployment-management-architecture.md`](../04-architecture/deployment-management-architecture.md)
10. **Visual design system** — `design.md` at project root — "Tactical Command" tokens, no-line rule, Inter + JetBrains Mono
11. **Existing FE conventions** — examine `frontend/src/features/code-build-management/` first (closest analog: observability-first, Story-Id trace chain, AI autonomy gating, per-card section results) and match its structure exactly. Also skim `features/dashboard/` and `features/incident/` for shell conventions.

## Existing Frontend (DO NOT break)

- Shell already exists under `frontend/src/shell/` (navigation, workspace context, right AI panel rail, routing)
- Dashboard / Code & Build / Requirement / Incident slices already exist and reference one another — your slice must integrate via their deep-link contracts (Requirement story page for traceability; Incident create form for failed-deploy deep-links), not by importing their code
- `SectionResult<T>`, `ApiResponse<T>`, `fetchJson<T>` already live under `frontend/src/shared/api/` — reuse as-is
- Shell nav config already has a `comingSoon: true` entry for Deployment — flip it to `false` as part of this slice
- `StoryChip.vue` / `HealthLed.vue` / `AiRowStatusBanner.vue` / `RedactedLogExcerpt.vue` / `RateLimitBanner.vue` / `LineageBadge.vue` / `DurationPill.vue` already exist from earlier slices in `src/shared/components/` or `features/code-build-management/components/primitives/`. Reuse them from the shared layer; if a primitive lives only in Code & Build, promote it to `src/shared/components/primitives/` as a non-breaking move and re-export.

## Scope (Phase A)

Build the frontend for Deployment Management under `frontend/src/features/deployment-management/` per the file structure in `deployment-management-design.md` §2. Deliverables:

### Six views with per-view card grids

1. **Catalog** (`/deployment`) — 4 cards: Summary bar, Filter bar, Grid (applications grouped by project with per-env pills), AI Summary
2. **Application Detail** (`/deployment/applications/:applicationId`) — 6 cards: Header, Environments strip, Recent Releases, Recent Deploys, Metrics (DF/CFR/MTTR), AI Insights
3. **Release Detail** (`/deployment/releases/:releaseId`) — 5 cards: Header, Composition (commits resolved via Code & Build facade, capped 100), Deploys, Linked Stories, AI Release Notes
4. **Deploy Detail** (`/deployment/deploys/:deployId`) — 5 cards: Header (with rollback detection signal), Stages (streaming for IN_PROGRESS), Approvals (role-gated redaction), Change Summary, AI Deploy Summary
5. **Environment Detail** (`/deployment/applications/:applicationId/environments/:name`) — 4 cards: Header, Current, Timeline, Stability
6. **Traceability** (`/deployment/traceability`) — 3 cards: Input (typeahead by story id), Releases, Deploys

### Primitives (all listed in design doc §5.1)

`DeployStateBadge`, `DeployTriggerChip`, `EnvironmentChip`, `ReleaseStateBadge`, `StageConclusionChip`, `StoryChip` (reused), `DurationPill` (reused), `FreshnessIndicator` (new — 3-tier Fresh/Degraded/Stale), `JenkinsLinkOut`, `ReleaseVersionPill`, `HealthLed` (reused), `AiRowStatusBanner` (reused), `RedactedLogExcerpt` (reused), `RateLimitBanner` (reused), `JenkinsStaleBanner` (new), `ReleaseNotesMarkdown` (sanitized markdown renderer).

### Pinia store

Shape per design doc §6: `catalog`, `applicationDetail`, `releaseDetail`, `deployDetail`, `environmentDetail`, `traceability`, `filters`, `activeIds`, `loading`, `errors`, `pollHandles`, `principal`. Actions per design doc §6 — including polling lifecycles:

- **AI PENDING polling**: 3s → 10s backoff, cap 5m
- **IN_PROGRESS deploy polling**: 5s tick while view is open; stop on terminal state or view unmount

### Mocks — must match API guide §3 exactly

Under `mock/`: `catalog.mock.ts`, `applicationDetail.mock.ts`, `releaseDetail.mock.ts`, `deployDetail.mock.ts`, `environmentDetail.mock.ts`, `traceability.mock.ts`, `approvals.mock.ts`, `aiReleaseNotes.mock.ts`, `aiDeploymentSummary.mock.ts`, `webhookFixtures/` (Jenkins Notification-Plugin bodies), `commandLoop.ts` (round-trip simulator).

The mock command loop MUST simulate every error code documented in API guide §4:

- `DP_AI_UNAVAILABLE` (5% injected), `DP_JENKINS_UNREACHABLE` (2%), `DP_RELEASE_NOTES_EVIDENCE_MISMATCH` (3%), `DP_BUILD_ARTIFACT_PENDING` (2%), `DP_RELEASE_UNRESOLVED` (1%), `DP_AI_AUTONOMY_INSUFFICIENT`, `DP_ROLE_REQUIRED`, `DP_WORKSPACE_FORBIDDEN`, `DP_STORY_NOT_FOUND` (only in Traceability), `DP_INGEST_SIGNATURE_INVALID`, `DP_RATE_LIMITED`
- IN_PROGRESS → SUCCEEDED / FAILED transitions on a 15-second mock clock for the in-progress deploy fixture
- AI PENDING → SUCCESS / FAILED / EVIDENCE_MISMATCH transitions on the polling clock

### Phase A/B toggle

```ts
const USE_BACKEND = import.meta.env.VITE_USE_BACKEND === 'true';
export const deploymentApi = USE_BACKEND ? liveClient : mockClient;
```

### Tests

Vitest for: every primitive, every store action, mock command loop happy + each error path, IN_PROGRESS polling lifecycle, AI PENDING polling lifecycle, role-gated approval redaction, evidence-mismatch banner rendering, autonomy gating branches. `@vue/test-utils` for every card's states (loading/success/error/empty/stale). axe-core smoke pass on each view.

## What NOT To Do

- Do NOT call any real backend; everything via the mock layer gated by `VITE_USE_BACKEND !== 'true'`
- Do NOT modify shell code (`frontend/src/shell/`) beyond flipping the Deployment nav entry from `comingSoon: true` to `false`
- Do NOT redefine types that already exist (`ApiResponse<T>`, `SectionResult<T>`, workspace context, `StoryChip` type shape from Code & Build)
- Do NOT import from other feature folders (`features/code-build-management`, `features/dashboard`, etc.) — go through `shared/` or promote the primitive to `shared/components/`
- Do NOT introduce new UI libraries, state libraries, or styling approaches
- Do NOT build any control that triggers / approves / promotes / cancels / rolls back a Jenkins build — the Control Tower is read-only toward Jenkins (REQ-DP-* non-goals)
- Do NOT persist approval rationale, approver display name, or any Jenkins credential in frontend state beyond what the current view renders
- Do NOT add pre-deploy AI risk assessment, failure triage, or auto-rollback suggestions — AI scope in V1 is release notes / deploy summary only
- Do NOT use localStorage / sessionStorage (per shell rules)
- Do NOT render raw commit messages without going through the existing `StoryIdExtractor` helper (reuse from Code & Build slice; do NOT re-implement regex in Deployment)
- Do NOT reimplement Story ↔ Commit resolution — that's owned by Code & Build. Deployment resolves composition + linked stories read-side by calling `/internal/release/{releaseId}/commit-slice` in Phase B; in Phase A, mocks return the pre-resolved shape

## Acceptance Criteria

- [ ] `npm run dev` renders a populated `/deployment` catalog with realistic mock data across 4 projects, 14 applications, 4 envs each, with at least one ROLLED_BACK on prod and one IN_PROGRESS on dev
- [ ] Deep-link `/deployment/applications/{anyMockAppId}` opens the Application Detail with 6 cards hydrated
- [ ] Deep-link `/deployment/releases/{anyMockReleaseId}` opens the Release Detail with 5 cards hydrated
- [ ] Deep-link `/deployment/deploys/{anyMockDeployId}` opens the Deploy Detail with 5 cards hydrated; rollback variant renders `rollback_detection_signal` tooltip; IN_PROGRESS variant streams stage updates on a 5s tick
- [ ] Deep-link `/deployment/applications/{anyMockAppId}/environments/prod` opens Environment Detail with 4 cards hydrated
- [ ] Deep-link `/deployment/traceability?storyId=STORY-…` returns releases + deploys for a known seed story; an unknown story id renders `DP_STORY_NOT_FOUND` empty-state without blocking the page
- [ ] AI regenerate paths work end-to-end in the mock loop: PENDING → SUCCESS, PENDING → FAILED (retry), PENDING → EVIDENCE_MISMATCH (banner + locked content)
- [ ] Workspace switch from the shell causes Deployment Management to refetch and re-render
- [ ] Every card renders every state (Loading / Success / Error / Empty / Stale) — demo by crafting corresponding mock fixtures
- [ ] Approval role-gated redaction verified: Developer principal sees `(redacted)` for both rationale and approver; PM / Tech Lead / Release Manager see actual values
- [ ] `npm run build` succeeds with zero TS errors
- [ ] `npm run test` passes; coverage for store, polling, and each card
- [ ] No ESLint warnings introduced
- [ ] a11y: tab-navigable, focus trap on slide-overs (if any), status chips have `aria-label`, `FreshnessIndicator` exposes tier via `aria-label`, redacted content reads as "redacted" to assistive tech
- [ ] Mock response shapes match API guide §3 examples byte-for-byte (this is what makes Phase B trivial — a golden-file test in Phase B will diff against the frontend mocks)
- [ ] Per-card isolation verified: a single card's mock throwing does NOT blank its neighbors on the same view
