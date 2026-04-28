# Testing Management Requirements

## 1. Context and Source

Requirements for the **Testing Management** slice of the Agentic SDLC Control Tower. Derived from:

- PRD §11.8 (Testing Management page definition)
- PRD §10 (V1 module list — Testing Management is in scope)
- PRD §6.4 / §6.5 / §7.4 / §7.5 / §7.7 (traceability chain, AI-native governance, autonomy levels)
- PRD §8 (core object model: Test Plan / Test Run as first-class objects)
- Scope decisions captured via the slice-kickoff questionnaire on 2026-04-18:
  - **Full QA lifecycle** scope — test plans, test cases, test runs, defects, coverage, environments
  - AI-native capability scope is **AI test-case generation from requirements only** (human-approval gated); other AI capabilities (risk prioritization, auto-triage, coverage-gap AI) are deferred to V1.1+
  - Primary integration boundary is **Requirements (traceability)** — every test case links back to at least one REQ-ID / Story-Id
  - CI/Incident/PM integrations are **extension points**, not first-class — surfaced via deep-links and read-only hooks only
  - Dev tooling: **Codex** for both Frontend and Backend (overrides the CLAUDE.md default of Gemini for FE, per the 2026-04-18 slice-kickoff decision)

## 2. Goal

Give PMs, QA Leads, Engineers, and Tech Leads a single page to understand, at a glance:

1. **What is being tested** — test plans scoped to a workspace/project/release, their cases, and coverage against requirements
2. **What executed** — recent test runs with pass/fail/skip counts, duration, environment, and the plan/case breakdown
3. **What failed and why** — failed cases with their last-good history, defect links, owner, and environment context
4. **How testing maps back to requirements** — every test case traces to one or more Story-Ids / REQ-IDs; every REQ has an on-page coverage indicator
5. **Where AI helped** — AI-drafted candidate test cases from a selected REQ/Story, always gated by human approval before becoming "active"
6. **Where to dig deeper** — deep-links into Requirement stories, Incident cases, and (V1.1+) Code & Build runs and Deployment slices

Non-goals for V1: running tests from the Control Tower, editing CI pipeline configuration, auto-triaging failures into incidents, AI-driven coverage-gap analysis, AI test-case auto-healing, cross-workspace analytics.

## 3. Scope Boundaries

| In scope (V1)                                                                                 | Out of scope (V1; revisit in V1.1+)                                                |
| --------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------- |
| Read/write Test Plan CRUD (name, scope, owner, release target, state)                         | Test plan approval workflows beyond a simple ACTIVE/ARCHIVED state                 |
| Read/write Test Case CRUD (title, preconditions, steps, expected, priority, type, linkedReqs) | Rich step editors with screenshot/attachments beyond a single markdown body        |
| Read-only Test Run ingestion from external runners (JUnit/TestNG XML, Playwright, Cypress)    | Triggering or re-running tests from the Control Tower                              |
| Defect linkage: a failed case may link to zero-or-many Incident IDs                           | Opening defects as incidents automatically (one-click "Open Incident" is manual)   |
| Coverage view: REQ-ID → linked test cases → most-recent case status                           | AI-driven coverage-gap analysis (deferred)                                         |
| AI test-case generation from a REQ-ID / Story-Id, human-approval gated (DRAFT → ACTIVE)       | AI auto-healing of broken cases, AI-driven flakiness classification (deferred)     |
| Environment registry: named environments (e.g. `dev`, `staging`, `prod-shadow`) + metadata    | Per-environment entitlement / approval gates                                       |
| Workspace isolation and role-based read/write access                                          | Per-test-plan ACL beyond workspace membership                                      |
| Deep-links to Requirement stories and Incident cases                                          | Cross-slice aggregate dashboards beyond the Requirement → Tests inverse view       |
| Per-card section error with retry; empty-state differentiation                                | Global page-level error page for card failures                                     |

## 4. User Personas

- **PM** — wants a workspace-level read on "are we ready to ship?" via coverage and recent-run health, plus story-level "which tests covered STORY-4211 and did they pass?"
- **QA Lead** — wants to build test plans, review candidate AI-drafted cases before activation, triage failed runs, and surface defect clusters.
- **Engineer** — wants to see test results for their changes, read failure context, and link back to the failing case's history.
- **Tech Lead** — wants coverage-by-requirement and failure clustering to inform release-readiness conversations.

## 5. Upstream / Downstream Dependencies

- **Upstream:** Requirement slice (for REQ-ID / Story-Id lookup and Story → TestCase backlink), Team/Project Space (for workspace + project membership + role), Platform Center (for AI skill registry and autonomy level config), Shared App Shell (for layout, lineage badge, card error boundary).
- **Downstream:** Incident slice (deep-linking a failed run into a new incident), Dashboard ("coverage and recent runs" tile), Report Center (V1.1+ historical coverage trend), Code & Build slice (V1.1+ will consume the Build → TestRun linkage), Deployment slice (V1.1+ release-readiness gate).

## 6. Requirements

### 6.1 Test Plan Catalog (Workspace-level view)

- **REQ-TM-01** — Catalog page lists every test plan visible to the user, grouped by project, with plan name, release target, owner, state (DRAFT / ACTIVE / ARCHIVED), linked-case count, and a coverage LED (green ≥80%, amber 50–79%, red <50%, grey if no linked REQs).
- **REQ-TM-02** — Catalog shall show an aggregate summary bar: total plans, total active cases, total runs in last 7d, overall pass rate (%) in last 7d, mean run duration in last 7d.
- **REQ-TM-03** — Catalog must be filterable by project, plan state (DRAFT/ACTIVE/ARCHIVED), coverage LED, and release target.
- **REQ-TM-04** — Catalog must be searchable by plan name.
- **REQ-TM-05** — Each plan tile shall deep-link to its Plan Detail page.
- **REQ-TM-06** — Catalog must respect workspace isolation: a plan attached to a project the user does not belong to must not be returned.
- **REQ-TM-07** — Each plan tile shall show a lineage badge indicating the upstream Project it belongs to (reusing `<LineageBadge>`).

### 6.2 Plan Detail

- **REQ-TM-10** — Plan Detail shall render 6 cards: Header, Cases, Coverage, Recent Runs, AI Draft Inbox, AI Insights.
- **REQ-TM-11** — Header shall show plan name, description, state, owner, release target, created/updated timestamps, and the owning Project.
- **REQ-TM-12** — Cases card shall list every case in the plan with ID, title, type (FUNCTIONAL / REGRESSION / SMOKE / PERF / SECURITY), priority (P0/P1/P2/P3), state (ACTIVE / DRAFT / DEPRECATED), linked-REQ chips, last-run status for this plan's most-recent run, and last-run timestamp.
- **REQ-TM-13** — Coverage card shall list every linked REQ-ID reachable from the plan's cases, with the case count per REQ and the aggregate latest-run status (PASS / FAIL / MIXED / NOT_RUN).
- **REQ-TM-14** — Recent Runs card shall list up to 20 most-recent runs for this plan with run ID, environment, trigger source (manual upload / CI webhook), state (RUNNING / PASSED / FAILED / ABORTED), duration, pass/fail/skip counts, and the actor who uploaded the results.
- **REQ-TM-15** — AI Draft Inbox card shall list every AI-drafted candidate case for this plan that is still in DRAFT state, with source REQ-ID, skill version, draft-timestamp, and the inline approve / reject / edit-before-approve affordances.
- **REQ-TM-16** — AI Insights card shall render an AI narrative summary for this plan: "what changed in the last 7 days, which cases are trending red, which REQs lost coverage".
- **REQ-TM-17** — Every case / run / REQ row shall include a deep-link (case → Case Detail, run → Run Detail, REQ → Requirement slice Story Detail).
- **REQ-TM-18** — Each card shall independently degrade on error (per-card error state + retry) without failing the page.

### 6.3 Case Detail

- **REQ-TM-20** — Case Detail shall show title, type, priority, state, owner, preconditions (markdown), steps (ordered markdown list), expected result (markdown), linked REQ chips, linked defect (incident) chips, and last 20 run outcomes (sparkline + table).
- **REQ-TM-21** — Case body content shall be rendered through a sanitized markdown renderer; no raw HTML; code fences are permitted.
- **REQ-TM-22** — Each linked REQ chip shall render GREEN if the REQ is visible in the user's workspace, AMBER if not visible, RED if not found — analogous to the Code & Build Story-Id chip contract (REQ-CB-73).
- **REQ-TM-23** — A case in DRAFT state (AI-drafted, pre-approval) shall be visually distinguished and shall not be counted in the plan's coverage or pass-rate figures.
- **REQ-TM-24** — A case in DEPRECATED state shall render with a deprecation badge, shall not be counted in coverage, but shall remain visible for historical runs.
- **REQ-TM-25** — Editing a case shall capture a revision entry (actor, timestamp, field diff); the revision list renders in a side drawer.
- **REQ-TM-26** — When a case's linked REQs change, the downstream Plan Coverage and Requirement-Slice inverse view shall reflect the change within 30s at P95.

### 6.4 Run Detail

- **REQ-TM-30** — Run Detail shall show run metadata (plan, environment, trigger, actor, duration, start/end timestamps), per-case outcomes (PASS / FAIL / SKIP / ERROR), and per-case failure excerpts (capped at 4 KB per case).
- **REQ-TM-31** — Failed case rows shall show the last-good timestamp ("last passed at …"), the failing assertion message, and a one-click "Open Incident from this failure" action that deep-links to the Incident slice with pre-filled context (plan ID, case ID, run ID, environment, failure excerpt).
- **REQ-TM-32** — Run Detail shall show the run's environment metadata (name, description, URL label, not credentials).
- **REQ-TM-33** — Run results are immutable once ingested; re-ingestion with the same `externalRunId` is a conflict (409) unless `force=true` is passed by an admin, which supersedes the prior run and audits the override.
- **REQ-TM-34** — Run ingestion shall support uploading JUnit XML, TestNG XML, Playwright JSON, and Cypress Mochawesome JSON formats in V1.
- **REQ-TM-35** — Run ingestion shall accept either (a) a manual upload via the Control Tower UI by a QA Lead, or (b) a signed webhook from an external CI system carrying the same file payload.
- **REQ-TM-36** — Run Detail shall expose a "Stories covered by this run" rollup: the union of REQ-IDs linked from every ACTIVE case in the run, capped at 200 REQs in V1 to avoid pathological plans.

### 6.5 REQ ↔ Case ↔ Run Traceability

- **REQ-TM-40** — Every TestCase shall persist zero-or-many TestCaseReqLink rows, each naming a REQ-ID / Story-Id. Links are author-maintained in the UI; ingestion does not infer links.
- **REQ-TM-41** — A Traceability view shall let users pick a REQ-ID / Story-Id and see the inverse: all active test cases, the plans they belong to, and the most recent run outcome per case, across all plans the user can see.
- **REQ-TM-42** — From the Requirement slice's Story Detail, a "Tests" tab (added in a downstream PR) shall call the Testing Management aggregate endpoint to render the same inverse view scoped to that story.
- **REQ-TM-43** — Validated REQ-IDs are those that exist in the Requirement slice and are visible to the same workspace. Unknown REQ-IDs are persisted as link rows with `status=UNKNOWN_REQ` and surfaced on the Plan Detail "unresolved links" diagnostics row; an hourly resolver job retries them.
- **REQ-TM-44** — A REQ-ID's coverage status is computed as: `GREEN` if at least one ACTIVE linked case passed in its most-recent run in the last 7d; `AMBER` if the most-recent linked run is older than 7d or mixed PASS/FAIL; `RED` if the most-recent linked run FAILED or ERROR; `GREY` if no runs yet.

### 6.6 AI Test Case Generation (human-approval gated)

- **REQ-TM-50** — From a REQ-ID, a user with role `QA Lead` or `Tech Lead` in the owning project may invoke `POST /ai/test-cases/draft` which returns a list of candidate cases (title, preconditions, steps, expected, type, priority) produced by the `test-case-drafter` skill.
- **REQ-TM-51** — Candidate cases are persisted as TestCase rows with `state=DRAFT` and `origin=AI_DRAFT`; they appear in the Plan Detail "AI Draft Inbox".
- **REQ-TM-52** — A DRAFT case shall NOT be executable in a run, shall NOT count toward coverage, and shall NOT appear in the public Cases card as active — only in the AI Draft Inbox.
- **REQ-TM-53** — Only a QA Lead or Tech Lead on the owning project may approve a DRAFT. Approval transitions `state=ACTIVE` and records an AuditLog entry; edits before approval are allowed and recorded as revisions (REQ-TM-25).
- **REQ-TM-54** — Rejecting a DRAFT transitions `state=DEPRECATED` with `deprecationReason=REJECTED_AT_INBOX`; rejected drafts remain visible in history but are excluded from all active views.
- **REQ-TM-55** — AI drafts are keyed by (`reqId`, `skillVersion`, `planId`); when the skill version advances, prior drafts for the same key render with a STALE badge and an admin may trigger a re-draft.
- **REQ-TM-56** — AI drafts shall honor the workspace's `aiAutonomyLevel`: `DISABLED` → draft endpoint returns 403; `OBSERVATION` → draft runs and results are visible but the Approve button is disabled (read-only preview); `SUPERVISED`/`AUTONOMOUS` → normal flow. AI never has authority to transition a case to ACTIVE directly, regardless of autonomy level.
- **REQ-TM-57** — AI drafts shall cite the REQ text they consumed; the Draft Inbox surfaces the source excerpt so the human reviewer can verify the model grounded on the right REQ.
- **REQ-TM-58** — The draft endpoint is rate-limited: at most 20 drafts per REQ per day per workspace; the UI surfaces the remaining quota.

### 6.7 Environment Registry

- **REQ-TM-60** — The system shall maintain a per-workspace Environment registry with `name` (unique within workspace), `description`, `kind` (DEV / STAGING / PROD / EPHEMERAL / OTHER), `url` (optional, for deep-link), and `archived` flag.
- **REQ-TM-61** — Run ingestion must name a registered environment; unknown environment names are auto-created with `kind=OTHER` and surfaced in the Platform diagnostics (the operator should later reclassify them).
- **REQ-TM-62** — Archived environments shall not accept new runs; existing runs remain visible for historical viewing.

### 6.8 Error Handling and Degraded States

- **REQ-TM-70** — Per-card section error with retry on the Catalog, Plan Detail, Case Detail, and Run Detail pages (no page-level wipeout for a single card failure).
- **REQ-TM-71** — Empty states must be distinguishable: "no plans yet in this workspace", "no cases in this plan", "no runs yet for this plan", "no AI drafts pending", "AI drafts disabled for this workspace", "AI draft generation failed".
- **REQ-TM-72** — Stale AI drafts (REQ-TM-55) must carry a visible STALE badge and a "re-draft" action for QA Leads.
- **REQ-TM-73** — When Requirement slice lookup fails, REQ-ID chips on cases render as `UNVERIFIED` (grey) rather than hiding — absence of verification must never look the same as absence of linkage.
- **REQ-TM-74** — When a run ingestion parse fails, the run is persisted in state `INGEST_FAILED` with the first 2 KB of parser output captured, and the operator may re-upload after fixing; a FAILED ingestion never silently partial-writes case outcomes.

### 6.9 Security, Privacy, and Governance

- **REQ-TM-80** — Failure excerpts captured from run ingestion shall be redacted for common secret patterns (AWS keys, GitHub tokens, generic bearer tokens) before storage, analogous to REQ-CB-81.
- **REQ-TM-81** — AI test-case-drafter prompts shall not include any failure-excerpt secrets or environment credentials; only REQ text + prior case titles are in-scope for the prompt.
- **REQ-TM-82** — Every run ingestion, AI invocation, and case state transition shall be audited via the shared `AuditLogEmitter`.
- **REQ-TM-83** — Only QA Lead and Tech Lead roles may approve AI drafts; only QA Lead, Tech Lead, and PM may create / archive plans; Engineers and other readers have read-only access to plans/cases/runs.
- **REQ-TM-84** — All write paths (plan CRUD, case CRUD, run ingestion, AI draft persistence, approvals, rejections) emit `LineageEvent` naming Testing Management as the authoring domain.
- **REQ-TM-85** — Run-ingestion webhooks shall verify an HMAC-SHA256 signature using a per-workspace shared secret issued via Platform Center; invalid signatures are rejected with 401 and audited.

### 6.10 Accessibility, i18n, Performance

- **REQ-TM-90** — All cards shall meet the shared shell's a11y bar (keyboard navigation, ARIA roles, color-blind-safe LEDs with shape/text redundancy).
- **REQ-TM-91** — All text shall use the shared i18n token set; no hard-coded English in components.
- **REQ-TM-92** — Catalog aggregate P95 load budget: 1200ms; Plan Detail P95: 1500ms; Run Detail P95: 1500ms (per-projection timeout 500ms with `SectionResult` fallback).
- **REQ-TM-93** — No card shall ship more than 300 KB of gzipped JS on the critical path; run failure-excerpt rendering (if large) must be virtualized.

## 7. Open Questions (tracked, not blocking)

- Should V1 support a "my tests" personalized lens (cases owned by the current user)? Default: no — deferred to V1.1.
- Should AI drafts ever auto-approve at `AUTONOMOUS`? Default: no in V1; `AUTONOMOUS` still requires a human approval for state transition to ACTIVE — AI never has authoring authority in this slice.
- Should flaky-case detection ship in V1? Default: no — deferred to V1.1 as part of the "risk-based prioritization" capability.
- Should unresolved REQ-ID links (REQ-TM-43) auto-resolve when the story later becomes visible? Default: yes, via an hourly resolver job writing back to the link rows.

## 8. Traceability to PRD

| PRD clause                                                                 | Requirements covered                               |
| -------------------------------------------------------------------------- | -------------------------------------------------- |
| §11.8 (Testing page definition, coverage of Spec / design constraints)     | REQ-TM-10…18, REQ-TM-40…44                         |
| §10 (V1 module list — Testing Management in scope)                         | REQ-TM-01…07                                       |
| §6.4 (full traceability chain)                                             | REQ-TM-40…44                                       |
| §7.4 / §7.5 (AI participant, human governor)                               | REQ-TM-50…58, REQ-TM-83                            |
| §7.7 (full-chain traceability)                                             | REQ-TM-40…44, REQ-TM-84                            |
| §8 (Test Plan / Test Run as first-class objects)                           | REQ-TM-10, REQ-TM-30                               |
| §6.5 (AI-native governance, audit)                                         | REQ-TM-56, REQ-TM-80…85                            |
