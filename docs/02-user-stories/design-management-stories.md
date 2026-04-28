# Design Management Stories

## Scope

This document defines the user stories for the **Design Management** page — the design traceability plane of the Agentic SDLC Control Tower. V1 is scoped as a **lightweight, read-only viewer** over internal Stitch-authored HTML mocks, with one admin write path (register + link). The primary flow is Spec → Design traceability.

Stories are grouped by view:

- Catalog view — browse registered design artifacts in a Workspace
- Viewer view — inspect a single design artifact with embedded preview, linked Specs, AI summary, change history
- Traceability view — Spec-centric coverage view (what Specs have no design)
- Cross-cutting — role enforcement, shell integration, empty/error states, AI Command Panel

## Traceability

- Requirements: [design-management-requirements.md](../01-requirements/design-management-requirements.md)
- PRD: [agentic_sdlc_control_tower_prd_v0.9.md](../01-requirements/agentic_sdlc_control_tower_prd_v0.9.md) §11.6, §11.4, §13, §15, §16, §18.2
- Cross-slice refs: [requirement-stories.md](requirement-stories.md), [project-space-stories.md](project-space-stories.md), [project-management-stories.md](project-management-stories.md), [shared-app-shell-stories.md](shared-app-shell-stories.md)
- Visual reference: registered HTML mocks in `docs/05-design/` (Control Tower.html, Incident Command Center.html, Platform Center.html, Project Space.html); design system at [../05-design/design.md](../05-design/design.md)

---

## Catalog View Stories

### Story S1: Browse Design Artifact Catalog

As an **Architect** or **Developer**,
I want to browse all registered design artifacts in my current Workspace,
so that I can find the mock I need without going through external tools.

#### Acceptance Criteria

- The `/design-management` route renders a Catalog view scoped to the Workspace selected in the shared context bar
- Each row shows: preview thumbnail (or neutral placeholder), title, artifact kind chip, lifecycle stage chip, owning Project, authors, last-updated, linked-Spec count, worst-coverage status, AI-summary availability chip
- Rows sort by last-updated descending by default
- If the Workspace has zero artifacts, a first-class empty state renders with a "Register your first artifact" CTA (admin only; non-admins see a "No artifacts registered yet" message)
- All rows strictly belong to the current Workspace (cross-Workspace leakage is a defect)

#### Edge Notes

- Phase A uses mocked seed data (8 artifacts across 3 Workspaces, mixed kinds and coverage)
- Phase B fetches from `GET /api/v1/design-management/catalog?workspaceId=...`
- Preview thumbnail is a server-rendered static snapshot; if unavailable, show the neutral `PAGE_MOCK` / `COMPONENT_MOCK` / `FLOW_MOCK` / `STATE_MOCK` kind glyph

#### Requirement Refs

REQ-DM-01, REQ-DM-10, REQ-DM-11, REQ-DM-12, REQ-DM-70, REQ-DM-71

---

### Story S2: Read Catalog Summary Counters

As a **Workspace Admin** or **Architect**,
I want a summary strip at the top of the Catalog showing total artifacts, orphan designs, missing designs, and stale coverage,
so that I can judge the health of design coverage at a glance.

#### Acceptance Criteria

- A summary bar renders above the catalog list with: total artifacts, artifacts with at least one linked Spec, orphan artifacts (zero links), Specs with zero linked designs ("Missing designs"), artifacts with STALE coverage, last-refreshed timestamp
- Each counter is clickable and applies the corresponding filter to the catalog list
- An AI linkage-gap advisory string (≤ 200 chars) renders when available, clearly labeled as AI-generated
- All counters respect the current Workspace

#### Edge Notes

- "Missing designs" requires joining with Requirement Management — backend provides this via a projection
- If the AI linkage-gap advisory is unavailable, the advisory slot renders a neutral skeleton; the counters remain functional

#### Requirement Refs

REQ-DM-10, REQ-DM-11, REQ-DM-15, REQ-DM-73

---

### Story S3: Filter and Search the Catalog

As a **Developer**,
I want to filter artifacts by owning Project, lifecycle stage, artifact kind, coverage status, and author, and to search by title,
so that I can narrow in on the mock that covers the Spec I'm working on.

#### Acceptance Criteria

- Filters are available in a compact filter bar: Project, lifecycle stage, artifact kind, coverage status, author, text search
- Filters combine with AND semantics
- Selected filters are reflected in the URL as query params (`?project=...&kind=...`) and survive refresh
- Filters can be cleared individually and via a "Clear all filters" link
- Sort controls are separate from filters: title, last-updated, linked-Spec count, coverage status

#### Edge Notes

- Query-param contract is the source of truth; the UI reads from and writes to the URL
- Text search is client-side in Phase A; Phase B pushes it to the backend with `?q=...`

#### Requirement Refs

REQ-DM-13, REQ-DM-06

---

### Story S4: Drill Into an Artifact from the Catalog

As an **Architect**,
I want to click an artifact row to open its dedicated Viewer,
so that I can inspect the mock and its Spec coverage in depth.

#### Acceptance Criteria

- Clicking a row navigates to `/design-management/artifacts/:artifactId`
- The Viewer route preserves the calling filter context in query params (optional; used for back navigation)
- The Catalog view state (filters, sort) is preserved when the user navigates back

#### Edge Notes

- Back navigation uses the shell's breadcrumb pipeline; breadcrumbs must include Catalog → Viewer

#### Requirement Refs

REQ-DM-14, REQ-DM-83

---

## Viewer View Stories

### Story S5: Inspect an Artifact with Embedded Preview

As a **Developer**,
I want to open the Viewer and see the registered HTML mock rendered inside a sandboxed preview frame,
so that I can review the design without switching tools.

#### Acceptance Criteria

- The Viewer renders the artifact's current version inside a sandboxed `<iframe>` that cannot navigate the top-level window or script the parent
- Above the frame, the artifact header renders: title, kind chip, lifecycle stage chip, owning Project link, authors, version tag, registered-at + registered-by, "Open in new tab" action, "Regenerate AI summary" action (admin only)
- Below the frame (or in a side panel on wide viewports), the Linked-Spec strip, AI Summary panel, and Change History render
- First-paint (excluding iframe content) completes in under 800 ms P95
- Iframe content for artifacts ≤ 2 MB renders within 3 s
- Artifacts larger than 2 MB render an "Open in new tab" CTA in place of the inline iframe

#### Edge Notes

- `sandbox` attribute: omit `allow-top-navigation`, `allow-popups-to-escape-sandbox`; include `allow-scripts allow-same-origin` only if required for the mock to render (default: no `allow-same-origin`)
- On narrow viewports (<1024px), side panels collapse into a tabbed layout under the preview

#### Requirement Refs

REQ-DM-20, REQ-DM-21, REQ-DM-26, REQ-DM-27, REQ-DM-60, REQ-DM-61, REQ-DM-70

---

### Story S6: See Which Specs This Artifact Covers

As an **Architect** or **PM**,
I want a strip of linked-Spec chips on the Viewer, each with a coverage badge,
so that I can verify this design actually covers the Specs it claims.

#### Acceptance Criteria

- A Linked-Spec strip renders below the Viewer header
- Each chip shows: Spec ID, truncated Spec title, coverage status badge (`OK` / `PARTIAL` / `STALE` / `MISSING`), jump-in link to Requirement Management
- Coverage status is computed server-side at request time and is never stale-read from cache
- If the artifact has zero links, the strip renders a crimson "No Spec links" empty state with a "Link a Spec" admin CTA
- Hovering a chip reveals a tooltip with the Spec's latest revision number and the artifact's `coversRevision`

#### Edge Notes

- Coverage semantics per REQ-DM-25:
  - `OK` — artifact covers the Spec's latest revision
  - `PARTIAL` — author-declared partial coverage
  - `STALE` — artifact predates the Spec's latest revision
  - `MISSING` — the referenced Spec is archived or deleted (data-integrity flag)
- Jump-in link target: `/requirement/specs/{specId}` (contract owned by Requirement Management)

#### Requirement Refs

REQ-DM-02, REQ-DM-22, REQ-DM-25, REQ-DM-80

---

### Story S7: Read the AI Summary of an Artifact

As a **Reviewer** (Architect / PM / QA),
I want an AI-generated one-paragraph summary and a bulleted list of key UI elements,
so that I can orient myself without reading the entire HTML mock.

#### Acceptance Criteria

- The AI Summary panel shows: one-paragraph summary (≤ 500 chars), bulleted list of extracted key UI elements (≤ 10 items), generated-at timestamp, source artifact version, skill name + version, AI attribution glyph
- If the summary is unavailable, the panel shows a skeleton with a "Generate summary" CTA (admin only)
- Admin can click "Regenerate" to trigger a fresh AI run; during the run, the panel shows a loading state but the rest of the Viewer remains fully interactive
- AI output never silently replaces human-authored metadata (title, authors, etc.)

#### Edge Notes

- AI summaries are cached server-side keyed by `(artifactId, versionId, skillVersion)`; regeneration invalidates the cache for that key
- If the skill runtime returns an error, the panel renders the error inline via `SectionResult.ERROR` with a retry action; the rest of the Viewer is unaffected
- If the Workspace's AI autonomy level is below the skill's required level, the panel renders in "advisory-only" mode with no "Apply" action (there is no apply action in V1 regardless, but the badge must still be shown)

#### Requirement Refs

REQ-DM-05, REQ-DM-23, REQ-DM-40, REQ-DM-41, REQ-DM-42, REQ-DM-72, REQ-DM-73

---

### Story S8: Review Change History for an Artifact

As an **Auditor** or **Architect**,
I want a read-only timeline of the artifact's version events,
so that I can understand how the design evolved and who changed what.

#### Acceptance Criteria

- The Change History panel lists events in reverse chronological order
- Each entry shows: timestamp, actor (Member or AI skill), event type (`REGISTERED` / `VERSION_PUBLISHED` / `LINKED_SPEC` / `UNLINKED_SPEC` / `AI_SUMMARY_GENERATED` / `LIFECYCLE_CHANGED`), and a short human-readable description
- Pagination: 25 per page; older pages loaded on demand
- `LINKED_SPEC` / `UNLINKED_SPEC` entries link to the affected Spec via jump-in link
- `AI_SUMMARY_GENERATED` entries show the skill name + version

#### Edge Notes

- The history view is strictly read-only in V1; there is no inline edit/undo
- Auditors see the same view as Architects in V1; permission-level differences apply only to write actions

#### Requirement Refs

REQ-DM-24, REQ-DM-51

---

### Story S9: Open the Artifact in a New Tab

As a **Reviewer**,
I want an "Open in new tab" action on the Viewer,
so that I can see the full-screen rendering for detailed inspection.

#### Acceptance Criteria

- The "Open in new tab" action is always available from the Viewer header
- Clicking opens `/design-management/artifacts/:artifactId/raw` in a new tab; this route serves the raw artifact HTML with the same sandbox headers applied
- The raw-view route requires the same authorization as the Viewer route

#### Edge Notes

- The raw-view route must set `Content-Security-Policy` and `X-Frame-Options: SAMEORIGIN` consistent with the shell's security posture
- No chrome is rendered around the raw artifact — it is the artifact only

#### Requirement Refs

REQ-DM-20, REQ-DM-50, REQ-DM-52

---

## Traceability View Stories

### Story S10: See Which Specs Have No Design

As a **PM** or **Architect**,
I want a Spec-centric traceability page that lists all Specs in the Workspace and flags the ones with zero linked designs,
so that I can identify design-coverage gaps before they become slippage risks.

#### Acceptance Criteria

- The `/design-management/traceability?workspaceId=...` route renders a Spec-centric list
- Each row shows: Spec ID, Spec title, owning Project, linked-artifact count, coverage summary (count by `OK` / `PARTIAL` / `STALE`), and an inline linker CTA (admin only) for Specs with zero links
- Specs with zero linked designs are rendered in crimson and float to the top of the list by default
- Filter controls: coverage status, owning Project
- Sort: default is "missing first"; users can sort by Spec ID, Project, or coverage status

#### Edge Notes

- The list joins data from Requirement Management; Design Management does not own Spec identity
- Only Specs visible to the caller's role are shown; Requirement Management's visibility rules apply

#### Requirement Refs

REQ-DM-02, REQ-DM-30, REQ-DM-32, REQ-DM-80

---

### Story S11: Deep-Link into Traceability from Requirement Management

As an **Architect** reviewing a Spec in Requirement Management,
I want a "See linked designs" link that jumps directly to the traceability page filtered to my Spec,
so that I can go from Spec to coverage in one click.

#### Acceptance Criteria

- `/design-management/traceability?specId=...` is a supported route
- When opened with a `specId` param, the traceability page pre-filters to that single Spec row (expanded) and shows its full list of linked artifacts with coverage status
- If the Spec is not visible to the caller (workspace or role mismatch), the page renders a 403 empty state rather than a blank list
- The query-param contract is stable; Requirement Management may construct this URL freely

#### Edge Notes

- Breadcrumb trail: Requirement Management → Spec detail → Design Management / Traceability / {specId}
- Back navigation returns to the originating Spec detail in Requirement Management

#### Requirement Refs

REQ-DM-31, REQ-DM-80, REQ-DM-83

---

### Story S12: Link Existing Artifacts to a Spec (Admin)

As a **Workspace Admin**,
I want to open a compact linker modal from any Spec row and select one or more existing registered artifacts to link,
so that I can fill Spec → Design coverage gaps without leaving the page.

#### Acceptance Criteria

- The linker modal is available only from Spec rows with at least one missing or partial link (or zero links) — and only to users with `WORKSPACE_ADMIN`
- The modal lists artifacts in the Workspace that are not yet linked to this Spec; each row shows title, kind, last-updated
- Admin can multi-select and click "Link" to create `DesignSpecLink` records
- On success, the traceability row refreshes to reflect the new coverage status
- The action is audited (REQ-DM-51) and rolled up into the Change History of each affected artifact

#### Edge Notes

- The modal does NOT allow registering a new artifact — that is a separate admin flow
- Rate-limit: no more than 50 links per admin per minute (server-enforced)
- If the caller loses admin authority mid-session, the backend returns 403; the UI surfaces a "Your role changed" toast

#### Requirement Refs

REQ-DM-33, REQ-DM-50, REQ-DM-51

---

## AI Command Panel Stories

### Story S13: Use the AI Command Panel in Design Management Context

As an **Architect**,
I want the shared AI Command Panel to offer context-appropriate Design Management actions,
so that I can summarize an artifact, extract key UI elements, or enumerate linkage gaps without leaving the page.

#### Acceptance Criteria

- When the Design Management route is active, the AI Command Panel offers exactly three actions: "Summarize this artifact", "Extract key UI elements", "Enumerate linkage gaps in this Workspace"
- No generation, no critique, no drafting actions appear (per V1 scope)
- Each action's output is rendered in the panel with AI attribution, generated-at, skill name + version, and a "Regenerate" affordance (admin only)
- Actions that require an artifact context (summarize, extract) are disabled on Catalog / Traceability routes where no single artifact is focused
- Panel state persists across route changes within Design Management

#### Edge Notes

- The AI Command Panel UI is owned by the shared shell (REQ-SAS-*); Design Management only contributes the action registry and per-action handlers
- If the Workspace's AI autonomy level is below the skill's required level, actions render disabled with a tooltip explaining why

#### Requirement Refs

REQ-DM-40, REQ-DM-41, REQ-DM-42, REQ-DM-05, REQ-DM-83

---

## Cross-Cutting Stories

### Story S14: Enforce Role-Based Access

As a **Platform Admin**,
I want Design Management to enforce the declared role rules on both read and write paths,
so that a caller cannot see or mutate artifacts outside their authority.

#### Acceptance Criteria

- A `WORKSPACE_READER` sees Catalog, Viewer, Traceability for their Workspace and nothing else
- A `PROJECT_READER` only sees artifacts whose `projectId` is in their authorized Project list
- Only `WORKSPACE_ADMIN` can register, publish new versions, link/unlink Specs, regenerate AI summaries, or open the bulk linker modal
- Any write action not available to the caller renders disabled with a tooltip explaining the required role
- Cross-Workspace direct-by-id fetch returns 403 and the artifact never appears in any list response for that caller

#### Edge Notes

- Role resolution happens in `DesignAccessGuard` on the backend; the UI mirrors the same rules but never acts as the source of truth
- Auditors have full read access including the Change History

#### Requirement Refs

REQ-DM-50, REQ-DM-52

---

### Story S15: Render Loading / Empty / Error States Gracefully

As **any user**,
I want each card in Design Management to handle loading, empty, and error states independently,
so that a single failing backend projection doesn't take down the whole page.

#### Acceptance Criteria

- Every card wraps its content in a `SectionResult<T>` envelope with explicit LOADING / EMPTY / OK / ERROR states
- LOADING renders a skeleton that preserves layout dimensions (no layout shift on resolve)
- EMPTY renders a first-class empty state with an explanation and a single obvious next action
- ERROR renders a human-readable reason, a correlationId chip, and a "Retry" action; retry re-issues only the failing card's fetch, not the whole page
- A failing AI Summary never prevents the Viewer or Linked-Spec strip from rendering

#### Edge Notes

- Skeletons must use the Tactical Command skeleton primitive from `src/shared/components/`
- Error banner background must never dominate the layout; errors are per-card, not whole-page

#### Requirement Refs

REQ-DM-62, REQ-DM-71, REQ-DM-72

---

### Story S16: Register a New Design Artifact (Admin)

As a **Workspace Admin**,
I want to register a new Stitch/HTML mock as a design artifact, providing title, kind, owning Project, authors, and an HTML payload or path,
so that the artifact becomes browsable in the catalog.

#### Acceptance Criteria

- A "Register artifact" CTA is available to `WORKSPACE_ADMIN` from the Catalog summary bar
- The registration form collects: title, kind (`PAGE_MOCK` / `COMPONENT_MOCK` / `FLOW_MOCK` / `STATE_MOCK`), owning Project, authors, lifecycle stage (default `DRAFT`), HTML payload (inline or via reference to a `docs/05-design/*.html` path), initial linked Spec IDs (optional)
- On submit, the backend validates: HTML size ≤ 2 MB, no PII patterns (REQ-DM-53), caller role, Project ownership, Spec visibility
- On success, a new `DesignArtifact` + initial `DesignArtifactVersion` (v1) are created, a `REGISTERED` change-log entry is written, and the catalog refreshes
- AI summary generation is triggered asynchronously; the catalog row shows "AI Summary pending" until ready

#### Edge Notes

- PII rejection is hard (blocked), not soft (sanitized) — the admin must remove PII from the HTML before retrying
- V1 supports inline payload and `docs/05-design/*.html` path references; arbitrary URL upload is not V1
- Attempting to register an artifact that already exists (same title + Project + kind + authors) prompts "This looks like a duplicate" with an option to publish a new version instead

#### Requirement Refs

REQ-DM-07, REQ-DM-50, REQ-DM-51, REQ-DM-53, REQ-DM-90

---

### Story S17: Publish a New Version of an Artifact (Admin)

As a **Workspace Admin**,
I want to publish a new version of an existing artifact,
so that the catalog and viewer reflect the latest design while preserving history.

#### Acceptance Criteria

- From the Viewer header, `WORKSPACE_ADMIN` can invoke "Publish new version" and supply an updated HTML payload + optional change note
- A new `DesignArtifactVersion` (v{n+1}) is created; the prior version is preserved
- A `VERSION_PUBLISHED` change-log entry is written
- All linked Specs are re-evaluated for `STALE` status against their latest revisions
- AI summary cache for the artifact is invalidated; generation is re-triggered asynchronously

#### Edge Notes

- If the new payload fails the PII or size checks, the existing version remains the current version — no partial write
- Stale-token fencing: if another admin updated the artifact concurrently, the publish action returns `DM_STALE_REVISION` (409) with the current version

#### Requirement Refs

REQ-DM-07, REQ-DM-50, REQ-DM-51, REQ-DM-93

---

### Story S18: Deep-Link into Design Management from Other Slices

As **any cross-slice consumer**,
I want Design Management to accept deep-links from Requirement Management, Project Space, and Project Management,
so that contextual hand-offs feel seamless.

#### Acceptance Criteria

- `/design-management/traceability?specId=...` (from Requirement Management)
- `/design-management?project=...` (from Project Space)
- `/design-management?coverage=MISSING` (from Project Management "missing design" risk chip)
- `/design-management/artifacts/:artifactId` (from any slice with an artifact ID)
- Each deep-link preserves the shell's context bar selection (Workspace, Project)
- Breadcrumbs reflect the origin slice

#### Edge Notes

- If a deep-link refers to an artifact or Spec outside the caller's visibility, the target page renders a 403 empty state rather than a blank list
- Deep-link URLs are stable contracts; changes require a versioned path

#### Requirement Refs

REQ-DM-80, REQ-DM-81, REQ-DM-82, REQ-DM-83

---

### Story S19: Preserve Shell Chrome and Context

As **any user**,
I want Design Management to render inside the shared app shell with consistent context bar, primary nav, breadcrumbs, and AI Command Panel,
so that my cross-page navigation feels coherent.

#### Acceptance Criteria

- Every Design Management route renders inside the shared `AppShell.vue`
- The context bar shows: current Workspace, Application (derived), optional Project chip when on a project-scoped route
- Primary nav highlights "Design" when any Design Management route is active
- Breadcrumbs reflect Catalog → Viewer → linked Spec (when applicable)
- The AI Command Panel's Design Management handlers are registered on route activation and torn down on deactivation

#### Edge Notes

- Context bar selection changes (e.g., switching Workspace) reset the catalog to the newly selected Workspace
- No Design Management route may bypass the shell chrome

#### Requirement Refs

REQ-DM-83, REQ-DM-06, REQ-DM-08

---

### Story S20: Observability and Correlation Tracking

As a **Platform Operator**,
I want every Design Management request to carry a correlationId and to emit structured logs and metrics,
so that I can debug incidents and monitor performance.

#### Acceptance Criteria

- Every HTTP request emits a correlationId (generated or propagated from `X-Correlation-Id` header)
- Backend logs include: correlationId, workspaceId, artifactId (if applicable), userId, endpoint, latency, cache-hit-flag for AI summary, response status
- Page-level metrics: Catalog first-paint P95, Viewer first-paint P95, AI-summary cache-hit rate, per-card error rate
- Error banners in the UI show the correlationId as a small chip that users can copy for support

#### Edge Notes

- Logs must redact any user-provided HTML payload; log only size, hash, and version
- Metric names follow the platform convention `design_management.{surface}.{metric}`

#### Requirement Refs

REQ-DM-63, REQ-DM-72

---

## Story Index

| Story | Title | View |
|-------|-------|------|
| S1 | Browse Design Artifact Catalog | Catalog |
| S2 | Read Catalog Summary Counters | Catalog |
| S3 | Filter and Search the Catalog | Catalog |
| S4 | Drill Into an Artifact from the Catalog | Catalog |
| S5 | Inspect an Artifact with Embedded Preview | Viewer |
| S6 | See Which Specs This Artifact Covers | Viewer |
| S7 | Read the AI Summary of an Artifact | Viewer |
| S8 | Review Change History for an Artifact | Viewer |
| S9 | Open the Artifact in a New Tab | Viewer |
| S10 | See Which Specs Have No Design | Traceability |
| S11 | Deep-Link into Traceability from Requirement Management | Traceability |
| S12 | Link Existing Artifacts to a Spec (Admin) | Traceability |
| S13 | Use the AI Command Panel in Design Management Context | AI Panel |
| S14 | Enforce Role-Based Access | Cross-cutting |
| S15 | Render Loading / Empty / Error States Gracefully | Cross-cutting |
| S16 | Register a New Design Artifact (Admin) | Cross-cutting |
| S17 | Publish a New Version of an Artifact (Admin) | Cross-cutting |
| S18 | Deep-Link into Design Management from Other Slices | Cross-cutting |
| S19 | Preserve Shell Chrome and Context | Cross-cutting |
| S20 | Observability and Correlation Tracking | Cross-cutting |
