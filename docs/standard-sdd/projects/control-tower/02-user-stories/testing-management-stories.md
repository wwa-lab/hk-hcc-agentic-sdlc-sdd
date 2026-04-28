# Testing Management — User Stories

## Source

- Requirements: [testing-management-requirements.md](../01-requirements/testing-management-requirements.md)
- PRD §11.8, §10, §6.4, §6.5, §7.4, §7.5, §7.7, §8

## Conventions

Every story uses `As a … / I want … / So that …` form with Given/When/Then acceptance criteria, edge notes, and REQ-TM references. Story IDs `TM-S1…TM-Sn` are stable and referenced downstream.

---

## Test Plan Catalog (Workspace-level)

### TM-S1 — Browse my workspace's test plans at a glance

**As a** PM on a workspace with multiple projects
**I want** a Catalog page listing all test plans I can see, grouped by project, with coverage and health at a glance
**So that** I can answer "are we ready to ship?" without opening each plan individually

**Acceptance criteria:**

- Given I am a member of workspaces W1 and W2, when I visit `/testing`, then the Catalog shows only plans linked to projects in W1, grouped by project.
- Given a plan has 10 active cases linked to 8 REQ-IDs and 6 of them passed in last 7d, when I look at the tile, then the coverage LED is amber (50–79%) and pass-rate shows 60%.
- Given I filter by `state=ACTIVE`, then only ACTIVE plans remain visible; DRAFT and ARCHIVED are hidden.
- Given I type `payment-` in search, then only plans whose name contains `payment-` are shown across all projects.

**Edge notes:**

- When zero plans exist in the workspace, render the "no plans yet" empty state with a tip to create a new plan.
- Coverage LED color: green ≥80%, amber 50–79%, red <50%, grey if no linked REQs.

**Requirements:** REQ-TM-01, REQ-TM-03, REQ-TM-04, REQ-TM-06

---

### TM-S2 — See aggregate health and summary metrics in the Catalog

**As a** QA Lead
**I want** a summary bar showing total plans, active cases, recent runs, and overall pass rate
**So that** I understand workspace-level testing health at a glance

**Acceptance criteria:**

- Given I visit `/testing`, then the Catalog summary bar shows: total plans (ACTIVE only), total active cases, total runs in last 7d, overall pass rate % (last 7d), and mean run duration.
- Given zero runs exist in the last 7d, when I hover the pass-rate badge, then a tooltip explains "no runs in the last 7 days".
- Given the workspace has no plans yet, the summary bar shows 0/0/0 gracefully.

**Edge notes:**

- Mean run duration is computed across all completed runs (state=PASSED/FAILED) in the time window; RUNNING/ABORTED are excluded.
- Pass rate = (sum of passed cases / sum of passed + failed cases) × 100, excluding SKIP and ERROR outcomes.

**Requirements:** REQ-TM-02, REQ-TM-03

---

### TM-S3 — Filter and sort test plans

**As a** PM
**I want** to filter plans by project, state, coverage LED, and release target
**So that** I can isolate the plans I care about

**Acceptance criteria:**

- Given I select `project=Billing`, when filters apply, then only plans in the Billing project remain.
- Given I select `state=DRAFT`, then only DRAFT plans show; state is a multi-select facet.
- Given I select `coverage=RED`, then only plans with red LEDs show (coverage <50%).
- Given I select `release=Q2-2026`, then only plans with that release target show; release target is optional on plans.

**Edge notes:**

- Filters are cumulative (AND logic); clearing one filter does not clear others.
- Release target is a free-text field; filter must be exact match (case-insensitive).

**Requirements:** REQ-TM-03, REQ-TM-04

---

### TM-S4 — Navigate to plan details from the Catalog

**As a** QA Lead
**I want** each plan tile to link to its Plan Detail page
**So that** I can drill down into a specific plan

**Acceptance criteria:**

- Given I click a plan tile on the Catalog, when the page loads, then I see Plan Detail with all 6 cards rendered.
- Given I navigate to `/testing/plans/{planId}` directly, then Plan Detail loads as if I had clicked the tile.

**Edge notes:**

- Breadcrumb must show: Workspace → Testing → Plan Name.
- Back button returns to Catalog with filters preserved (via URL state).

**Requirements:** REQ-TM-05, REQ-TM-90

---

### TM-S5 — See lineage badge showing which project owns the plan

**As a** PM across multiple projects
**I want** each plan tile to show a lineage badge indicating its owning project
**So that** I know which team owns the testing work

**Acceptance criteria:**

- Given a plan belongs to project "Billing", when I view the Catalog, then the plan tile shows a lineage badge with the Billing project icon and name.
- Given a project is archived, the lineage badge renders with a disabled-state indicator.

**Edge notes:**

- Lineage badge reuses the shared shell's `<LineageBadge>` component.

**Requirements:** REQ-TM-07

---

## Plan Detail

### TM-S6 — See test plan header with metadata

**As a** QA Lead
**I want** a Plan Detail page that shows plan name, description, state, owner, release target, and timestamps
**So that** I have full context on the plan upfront

**Acceptance criteria:**

- Given I navigate to a plan's detail page, when the Header card loads, then it displays: plan name, description (markdown), state (DRAFT/ACTIVE/ARCHIVED), owner name, release target, created-at, updated-at, and owning project.
- Given the plan's description is empty, render "No description" in italics.
- Given the plan's owner is deactivated, the owner name renders with a "deactivated" badge.

**Edge notes:**

- All metadata fields are immutable in the UI (no edit affordances on the header itself; edits are a future enhancement).
- Project name must be clickable to link to the owning project in Team/Project Space.

**Requirements:** REQ-TM-11

---

### TM-S7 — View all test cases in a plan

**As a** QA Lead
**I want** a Cases card listing every case in the plan with full metadata
**So that** I can understand coverage and see which cases are active/draft/deprecated

**Acceptance criteria:**

- Given a plan has 15 cases, when the Cases card loads, then it shows all 15 rows (no pagination; virtualize if >300 rows per REQ-TM-93).
- Given a case has type FUNCTIONAL, priority P0, and state ACTIVE, when I view the Cases card, then those attributes render as labels.
- Given a case is in DRAFT state, a blue "DRAFT" badge appears and the row is slightly desaturated.
- Given a case is DEPRECATED, a grey "DEPRECATED" badge appears with a deprecation icon.
- Given a case has 3 linked REQ-IDs, the row shows 3 green chips; clicking a chip navigates to the REQ's story page.
- Given a case's most-recent run passed, the status LED is green; if failed, red; if not-run, grey.

**Edge notes:**

- Last-run timestamp is the timestamp of the run (not the case's updated-at).
- Case rows are sortable by: ID, title, type, priority, state, last-run status.
- Search within the card by case title (substring match).

**Requirements:** REQ-TM-12, REQ-TM-23, REQ-TM-24

---

### TM-S8 — View coverage rollup by requirement

**As a** Tech Lead
**I want** a Coverage card showing every linked REQ-ID and how many cases cover it
**So that** I can see which requirements have test coverage

**Acceptance criteria:**

- Given a plan has cases linked to 12 REQ-IDs, when the Coverage card loads, then it lists all 12 REQ-IDs with case counts.
- Given REQ-2010 is linked by 2 cases and both passed in the most-recent run, the REQ row shows a green LED and "2 cases".
- Given REQ-2011 is linked by 1 case that failed in the most-recent run, the REQ row shows a red LED.
- Given REQ-2012 is linked by 2 cases but the most-recent run is >7 days old, the REQ row shows an amber LED ("last run 10d ago").
- Given REQ-2013 is linked by cases but one case passed and one failed in the same run, the LED is amber ("mixed status").
- Given a REQ-ID does not resolve in the Requirement slice, the row renders as "UNKNOWN_REQ" (grey LED) with a "not found in workspace" tooltip.

**Edge notes:**

- Coverage status follows REQ-TM-44 logic: GREEN (≥1 ACTIVE case passed last 7d), AMBER (older run or mixed), RED (latest is FAILED/ERROR), GREY (no runs).
- Clicking a REQ chip navigates to the Requirement slice's story detail.
- Unresolved links are surfaced in a separate "Unresolved links" diagnostics row below the main coverage list.

**Requirements:** REQ-TM-13, REQ-TM-26, REQ-TM-40, REQ-TM-41, REQ-TM-43, REQ-TM-44

---

### TM-S9 — View recent test runs for this plan

**As a** QA Lead
**I want** a Recent Runs card showing the last 20 runs for this plan with status and metrics
**So that** I can see the plan's recent execution history

**Acceptance criteria:**

- Given a plan has 50 runs total, when the Recent Runs card loads, then the top 20 (newest first) are displayed with pagination to load more.
- Given a run is PASSED, the status LED is green; FAILED is red; RUNNING is yellow; ABORTED is grey.
- Given a run has 45 passed, 3 failed, 2 skipped cases, the row shows a breakdown: "45P / 3F / 2S".
- Given a run has a duration of 325s, the row shows "5m 25s".
- Given a run was uploaded by user alice@example.com from manual upload, the trigger source shows "Manual (alice)"; if from CI webhook, shows "CI".
- Given I click a run row, I navigate to Run Detail for that run.

**Edge notes:**

- Duration is computed as (end timestamp - start timestamp); RUNNING runs show elapsed time with "in progress" suffix.
- Runs are sorted by start timestamp (newest first); pagination uses cursor-based limit.

**Requirements:** REQ-TM-14, REQ-TM-90

---

### TM-S10 — Review and manage AI-drafted test cases

**As a** QA Lead
**I want** an AI Draft Inbox card showing candidate test cases generated by AI, with approve/reject/edit affordances
**So that** I can review and activate AI-drafted cases

**Acceptance criteria:**

- Given a plan has 3 AI-drafted cases in DRAFT state, when the AI Draft Inbox card loads, then all 3 are listed with source REQ-ID, skill version, draft timestamp, and the inline approve/reject/edit buttons.
- Given I click Approve, the case transitions to ACTIVE and an audit log entry is recorded; the case moves to the Cases card.
- Given I click Reject, the case transitions to DEPRECATED with deprecationReason=REJECTED_AT_INBOX.
- Given I click Edit, an inline editor appears (markdown for preconditions/steps/expected); I can save edits before approving.
- Given no AI drafts exist, the card shows "No pending AI drafts" empty state.
- Given the workspace's aiAutonomyLevel=DISABLED, the card shows "AI draft generation is disabled for this workspace".

**Edge notes:**

- Editing a draft records a revision entry (per REQ-TM-25) before approval.
- The "Edit before approve" flow allows users to refine AI output; approving the edited version still records the original source REQ-ID.

**Requirements:** REQ-TM-15, REQ-TM-50, REQ-TM-51, REQ-TM-52, REQ-TM-53, REQ-TM-54, REQ-TM-56

---

### TM-S11 — See AI narrative insights for the plan

**As a** PM
**I want** an AI Insights card showing a single-paragraph summary of testing trends in the last 7 days
**So that** I can understand what changed in testing without deep-diving

**Acceptance criteria:**

- Given the workspace has a recent AI summary for this plan, when the AI Insights card loads, then the summary renders as a single paragraph with a "Generated Xm ago" timestamp.
- Given the summary cites trending red cases or coverage gaps, it shows evidence chips with case IDs or REQ-IDs; clicking a chip navigates to that case or REQ.
- Given the workspace's aiAutonomyLevel=DISABLED, the card shows "AI insights are disabled for this workspace" and no summary is generated.
- Given no summary exists or is older than 10m, an admin-only "Regenerate" button appears; clicking it requests a fresh summary (rate-limited to 1/min).

**Edge notes:**

- Insights generation is async; if in progress, show a skeleton. If failed, show FAILED + admin-only Retry.
- Summary must cite the evidence (case/REQ IDs) it relied on for transparency.

**Requirements:** REQ-TM-16, REQ-TM-90

---

### TM-S12 — Per-card error isolation on Plan Detail

**As a** QA Lead
**I want** a failing card to show an error state with retry, not take down the entire page
**So that** I can keep working with the cards that loaded

**Acceptance criteria:**

- Given the Cases card times out at 500ms, when Plan Detail renders, then the Cases card shows error+retry; other cards remain interactive.
- Given I click retry on the Cases card, only the Cases card re-requests; other cards' state is not reset.
- Given the backend returns `PLAN_NOT_FOUND`, the entire page renders a 404 error screen (not per-card degradation).

**Edge notes:**

- Per-card timeout is 500ms per REQ-TM-92; SectionResult fallback is used for graceful degradation.
- Global access-denied (403) also renders at page level, not per-card.

**Requirements:** REQ-TM-18, REQ-TM-70, REQ-TM-92

---

## Case Detail

### TM-S13 — View full test case content and history

**As a** Engineer
**I want** a Case Detail page showing case title, type, priority, state, owner, preconditions, steps, expected result, and linked requirements
**So that** I understand exactly what the case tests

**Acceptance criteria:**

- Given I navigate to a case's detail page, when the page loads, then it shows: title, type (FUNCTIONAL/REGRESSION/SMOKE/PERF/SECURITY), priority (P0/P1/P2/P3), state (ACTIVE/DRAFT/DEPRECATED), owner, preconditions, steps (ordered list), expected result, and linked REQ chips.
- Given a case is in DRAFT state (AI-drafted, pre-approval), a blue "DRAFT" badge appears and a note explains "This case is pending approval".
- Given a case is DEPRECATED, a grey deprecation badge appears; the case is read-only.
- Given the case's body contains markdown including code fences, they render correctly via sanitized markdown (no raw HTML).

**Edge notes:**

- All markdown is sanitized; HTML tags are stripped.
- Code fences (triple backtick) are preserved and syntax-highlighted if a language is specified.

**Requirements:** REQ-TM-20, REQ-TM-21, REQ-TM-23, REQ-TM-24

---

### TM-S14 — See linked requirements with visibility status

**As a** QA Lead
**I want** each linked REQ-ID to show a color-coded chip indicating whether the REQ is visible in my workspace
**So that** I know if a requirement is still valid or may have been deleted

**Acceptance criteria:**

- Given a case links to REQ-2050 which exists in the Requirement slice and is visible to my workspace, when I view the case, the REQ chip is green.
- Given a case links to REQ-2051 which exists but is in a project I cannot see, the chip is amber with label "UNVERIFIED" and tooltip "Story not visible in your workspace".
- Given a case links to REQ-2052 which does not exist (was deleted), the chip is red with label "NOT_FOUND".
- Given a case has zero linked REQs, a note explains "This case is not linked to any requirements".

**Edge notes:**

- Chip behavior mirrors Code & Build's Story-Id chip contract (REQ-CB-73).
- RED status (NOT_FOUND) suggests the REQ was deleted; the user may unlink it.

**Requirements:** REQ-TM-22

---

### TM-S15 — View case revision history

**As a** QA Lead
**I want** to see a list of every edit to this case with actor, timestamp, and field diffs
**So that** I understand what changed and why

**Acceptance criteria:**

- Given a case was edited 3 times, when I open the revision drawer, then all 3 edits are listed (newest first) with actor name, timestamp, and a diff of changed fields.
- Given an edit changed the "steps" field, the diff shows the before/after markdown side-by-side.
- Given an AI draft was edited before approval, the revision records "User edit" and "System: approved" as separate entries.

**Edge notes:**

- Revision drawer is accessible via a side-drawer icon on the case header.
- Revisions are immutable once recorded.

**Requirements:** REQ-TM-25

---

### TM-S16 — View recent case run outcomes in a sparkline and table

**As a** Engineer
**I want** to see the last 20 run outcomes for this case (pass/fail/skip/error) in a sparkline and detailed table
**So that** I can spot trends (e.g., is this case flaky?)

**Acceptance criteria:**

- Given a case has 15 historical run outcomes, when Case Detail loads, then a sparkline shows a visual trend (green for pass, red for fail, yellow for skip, grey for error) and a detailed table lists the 15 outcomes (newest first) with: run ID, run date, environment, status, and failure excerpt (if failed).
- Given a failed outcome has a 4 KB failure excerpt, the table shows the excerpt capped at 4 KB with a "View full log" link if longer.
- Given no outcomes exist, the table shows "No runs yet for this case".

**Edge notes:**

- Sparkline is capped at the last 20 outcomes for performance; table can paginate for more.
- Clicking a run ID navigates to Run Detail.

**Requirements:** REQ-TM-20, REQ-TM-93

---

### TM-S17 — Track requirement coverage changes with 30s propagation

**As a** Tech Lead
**I want** plan coverage to update within 30s at P95 when I link or unlink a REQ-ID from a case
**So that** dashboards and reports reflect the change quickly

**Acceptance criteria:**

- Given I unlink REQ-2050 from a case, when I save, then the Plan Detail's Coverage card updates to remove that REQ-ID within 30s at P95.
- Given I link a new REQ-2051 to a case, when I save, then the Requirement slice's inverse "Tests" tab (on the story detail) reflects the new case within 30s.
- Given a case has linked REQs and I navigate away and back, the coverage reflects the latest state.

**Edge notes:**

- Propagation is via event streaming (LineageEvent); no polling.
- P95 constraint is a system-level SLA (REQ-TM-92).

**Requirements:** REQ-TM-26, REQ-TM-40

---

## Run Detail

### TM-S18 — View test run metadata and per-case outcomes

**As a** Engineer
**I want** a Run Detail page showing run metadata, environment, and per-case outcomes (pass/fail/skip/error) with failure excerpts
**So that** I understand which cases failed and why

**Acceptance criteria:**

- Given I navigate to a run's detail page, when it loads, then the header shows: run ID, plan name, environment, trigger source (manual upload / CI webhook), actor (who uploaded), duration, start/end timestamps, and per-case outcome table.
- Given the run has 100 cases (45 passed, 3 failed, 2 skipped, 50 error), the outcome table lists all rows with sortable columns: case ID, case title, status (LED + text), failure excerpt.
- Given a case failed with a 5 KB failure excerpt, the table shows the excerpt capped at 4 KB with a "View full (5 KB)" action.
- Given a case error'd (e.g., timeout), the excerpt shows the error message from the runner (e.g., "Timeout after 30s").

**Edge notes:**

- Large excerpt rendering must be virtualized (REQ-TM-93) to avoid shipping >300 KB gzipped JS.
- Failure excerpt is redacted for secrets before storage (REQ-TM-80).

**Requirements:** REQ-TM-30, REQ-TM-93

---

### TM-S19 — See environment details for the run

**As a** QA Lead
**I want** Run Detail to show the run's environment (name, description, URL label)
**So that** I know which environment the test ran in

**Acceptance criteria:**

- Given a run executed on environment "staging", when Run Detail loads, then the environment section shows: name "staging", description (if any), and a clickable URL label (if the environment has a url field).
- Given the environment does not exist (was deleted), the environment name renders as "UNKNOWN" with a grey badge.

**Edge notes:**

- Environment URL is shown as a clickable link but never includes credentials (REQ-TM-80).
- If the environment is archived, a note appears: "This environment is archived and no longer accepts new runs".

**Requirements:** REQ-TM-32, REQ-TM-60

---

### TM-S20 — Prevent silent partial-writes of run ingestion failures

**As a** operator
**I want** failed run ingestions to be atomic: if parsing fails, the run is persisted in INGEST_FAILED state and no case outcomes are written
**So that** the Control Tower does not silently have incomplete data

**Acceptance criteria:**

- Given I upload a JUnit XML file that is malformed (invalid XML), when ingestion fails, then the run is persisted with `state=INGEST_FAILED` and the first 2 KB of the parser error is captured in a `parseError` field.
- Given a run is in INGEST_FAILED state, when Run Detail loads, it shows "Ingestion failed — view error" and an admin can re-upload after fixing the file.
- Given the file parse succeeded but the run result writing failed, the entire transaction rolls back and the run is never created.

**Edge notes:**

- INGEST_FAILED runs do not appear in pass-rate or coverage calculations.
- Operator workflow: fix the file → re-upload with the same `externalRunId` (conflict handling via REQ-TM-33).

**Requirements:** REQ-TM-74

---

### TM-S21 — Link failed cases to incidents (deep-link to Incident slice)

**As a** Tech Lead
**I want** a one-click "Open Incident" action on failed case rows
**So that** I can escalate a recurring failure into the incident-tracking system

**Acceptance criteria:**

- Given a case failed in a run, when I click "Open Incident" on the case row, then I am navigated to the Incident slice's create-incident form with: planId, caseId, runId, environment, and failure excerpt prefilled via query params.
- Given the failure excerpt contains redacted secrets (AWS keys), the excerpt shown to me does not expose the raw keys.
- Given I do not have permission to create incidents in this workspace, the button is disabled with a tooltip.

**Edge notes:**

- This is a navigation action; it does not mutate the Control Tower's run data.
- The Incident slice receives the query-param context and auto-fills its form.

**Requirements:** REQ-TM-31

---

### TM-S22 — Show last-good timestamp for failed cases

**As a** Engineer investigating a failure
**I want** a failed case row to show "last passed at …" timestamp
**So that** I can determine how long a case has been red

**Acceptance criteria:**

- Given a case failed on 2026-04-18 but last passed on 2026-04-15 at 14:30 UTC, when I view the failed case row, it shows "Last passed 3d ago".
- Given a case has never passed, the row shows "Never passed".

**Edge notes:**

- "Last passed" is computed across all runs (all plans) for this case, not just the current plan's runs.

**Requirements:** REQ-TM-31

---

### TM-S23 — View stories covered by the run

**As a** PM
**I want** Run Detail to show "Stories covered by this run": the union of all REQ-IDs linked from active cases in the run
**So that** I can see which requirements this run validates

**Acceptance criteria:**

- Given a run exercises 3 cases linked to a total of 7 unique REQ-IDs, when Run Detail loads, then a "Stories covered" section lists all 7 REQ-IDs with a link to each story.
- Given >200 REQ-IDs would be in the union, the list is capped at 200 with a note "(capped at 200; see plan coverage for full list)".
- Given a case has no linked REQs, that case contributes zero REQs to the union.

**Edge notes:**

- Only ACTIVE cases count toward the union (DRAFT/DEPRECATED cases are excluded).
- The cap at 200 is a pathological-plan safeguard (REQ-TM-36).

**Requirements:** REQ-TM-36

---

### TM-S24 — Immutable runs with admin force-override conflict handling

**As a** operator
**I want** run results to be immutable once ingested; re-ingestion with the same externalRunId to conflict unless force=true
**So that** accidental re-uploads don't corrupt the run record

**Acceptance criteria:**

- Given a run with externalRunId="ci-abc-123" is already ingested, when I upload the same externalRunId again (without force=true), the ingestion returns 409 Conflict.
- Given I am an admin and pass `force=true`, the prior run is superseded and an audit log entry records the override with the reason.
- Given the new run's case outcomes differ from the prior run, the change is audited and historical queries can still access the prior run (via versioning or audit trail).

**Edge notes:**

- externalRunId is a string identifier (e.g., GitHub workflow run ID, Jenkins build ID) provided by the external CI system.
- Force override is admin-only; non-admins cannot override.

**Requirements:** REQ-TM-33

---

## REQ ↔ Case ↔ Run Traceability

### TM-S25 — Maintain explicit REQ-ID ↔ Case links

**As a** QA Lead
**I want** to link cases to REQ-IDs manually in the UI (not inferred from ingestion)
**So that** traceability is intentional and maintainable

**Acceptance criteria:**

- Given a case is created, I can add one or more REQ-IDs (or Story-Ids) to it via a chip-input field.
- Given I link REQ-2050 to a case, a TestCaseReqLink row is persisted with that REQ-ID.
- Given I unlink a REQ-ID, the link row is deleted and coverage is recalculated.
- Given I link an unknown REQ-ID (one that doesn't exist in the Requirement slice), the link is still persisted with `status=UNKNOWN_REQ` and surfaced in the Plan's unresolved-links diagnostics.

**Edge notes:**

- REQ-IDs are validated (if they resolve) at link time; unresolved links are retried hourly via a background job (REQ-TM-43).
- Linking/unlinking emits an AuditLogEmitter entry and a LineageEvent.

**Requirements:** REQ-TM-40, REQ-TM-43

---

### TM-S26 — Inverse-lookup: find all test cases for a requirement

**As a** PM
**I want** a Traceability view where I enter a REQ-ID and see all test cases that cover it, across all plans
**So that** I can validate a requirement is tested

**Acceptance criteria:**

- Given I navigate to `/testing/traceability?reqId=REQ-2050`, then I see: all active cases linked to that REQ-ID (across all visible plans), grouped by plan, with each case showing its latest-run status.
- Given no cases are linked to the REQ-ID, the view shows "No test cases linked" empty state.
- Given the REQ-ID does not exist in the Requirement slice, I see a warning "This requirement was not found" but the view still renders (in case the requirement is temporarily unavailable).

**Edge notes:**

- The Traceability view is read-only and shows only ACTIVE cases (DRAFT/DEPRECATED excluded).
- Clicking a case navigates to Case Detail; clicking a run navigates to Run Detail.

**Requirements:** REQ-TM-41

---

### TM-S27 — Story-detail Tests tab (inverse view embedded in Requirement slice)

**As a** PM viewing a story in the Requirement slice
**I want** a Tests tab that shows all active test cases linked to this story across all visible plans
**So that** I stay in one place

**Acceptance criteria:**

- Given I navigate to a story's detail page in the Requirement slice (e.g., `/requirement/stories/STORY-4211`), when I click the Tests tab, then the Testing Management aggregate endpoint returns all test cases linked to that story.
- Given 5 cases are linked to the story, the tab shows them grouped by plan with latest-run status.
- Given the Testing Management service is unavailable, the tab shows a per-tab error with retry.

**Edge notes:**

- This tab is added to the Requirement slice in a downstream PR; coordinate with the Requirement slice maintainer.
- The Tests tab uses the same inverse-lookup aggregate endpoint as the Traceability view (REQ-TM-41).

**Requirements:** REQ-TM-42

---

### TM-S28 — Coverage status computation (GREEN/AMBER/RED/GREY)

**As a** Tech Lead
**I want** each requirement's coverage status to reflect its testing health: GREEN (recent pass), AMBER (old/mixed), RED (failed), GREY (no runs)
**So that** I can triage release-readiness by coverage health

**Acceptance criteria:**

- Given REQ-2050 has ≥1 ACTIVE case that passed in its most-recent run ≤7 days ago, the coverage status is GREEN.
- Given REQ-2051 has its most-recent run >7 days ago or shows mixed PASS/FAIL, the status is AMBER.
- Given REQ-2052 has its most-recent run result as FAILED or ERROR, the status is RED.
- Given REQ-2053 has zero runs, the status is GREY with tooltip "No test runs yet".

**Edge notes:**

- Coverage status is computed per-case and aggregated per-REQ; a REQ with 1 passing and 1 failing case shows AMBER (mixed).
- The 7-day window is a staleness threshold; old runs are treated as unreliable.

**Requirements:** REQ-TM-44

---

## AI Test Case Generation (human-approval gated)

### TM-S29 — Request AI-drafted test cases for a requirement

**As a** QA Lead
**I want** to invoke AI test-case generation from a REQ-ID and receive candidate cases
**So that** I have a starting point for manual test design

**Acceptance criteria:**

- Given I click "Generate test cases" in the Plan Detail Coverage section for REQ-2050, when the `POST /ai/test-cases/draft` endpoint is invoked, then the backend calls the `test-case-drafter` skill with the REQ text and returns a list of candidate cases (title, preconditions, steps, expected, type, priority).
- Given I have role QA Lead on the owning project, the button is enabled.
- Given I have role Engineer or PM (not QA Lead), the button is disabled with tooltip "Only QA Leads can generate draft cases".

**Edge notes:**

- The endpoint returns 2–5 candidate cases per invocation.
- Candidates are not persisted until the user approves them individually.
- Rate limit: 20 drafts per REQ per day per workspace (REQ-TM-58); the UI shows remaining quota.

**Requirements:** REQ-TM-50, REQ-TM-58, REQ-TM-83

---

### TM-S30 — Persist and review AI-drafted cases in the Draft Inbox

**As a** QA Lead
**I want** AI-drafted candidates to appear in the AI Draft Inbox with source REQ-ID, skill version, and citation
**So that** I can review the AI's work before activating

**Acceptance criteria:**

- Given AI generates 3 candidate cases for REQ-2050, when they are returned, then they are persisted as TestCase rows with `state=DRAFT`, `origin=AI_DRAFT`, and `sourceReqId=REQ-2050`.
- Given a draft case is created, an AuditLog entry records the AI generation event.
- Given the `test-case-drafter` skill has version `v1.2`, the draft case stores `skillVersion=v1.2` for future comparison (REQ-TM-55).
- Given a draft case was generated from REQ-2050, the Draft Inbox shows a citation: "Based on REQ-2050: [excerpt of REQ text, 1–2 sentences]" so I can verify the AI grounded correctly.

**Edge notes:**

- Drafts are NOT counted in coverage, pass rates, or the public Cases card until approved.
- Drafts are only visible in the AI Draft Inbox.

**Requirements:** REQ-TM-51, REQ-TM-52, REQ-TM-57

---

### TM-S31 — Approve, reject, or edit AI-drafted cases before activation

**As a** QA Lead
**I want** to approve a DRAFT case (transition to ACTIVE), reject it (mark DEPRECATED), or edit before approval
**So that** I control the quality gate for AI-generated content

**Acceptance criteria:**

- Given a draft case is in the AI Draft Inbox, when I click Approve, the case transitions to ACTIVE and the AuditLog records "QA Lead alice approved AI draft REQ-2050 v1.2".
- Given I click Reject, the case transitions to DEPRECATED with `deprecationReason=REJECTED_AT_INBOX`.
- Given I click Edit, an inline markdown editor appears; I can refine preconditions, steps, or expected result; saving edits records a revision entry with actor=current user.
- Given I approve the edited draft, the case transitions to ACTIVE; the revision history shows both the "AI generated" and "User edit" entries.

**Edge notes:**

- Rejecting or approving a draft removes it from the AI Draft Inbox.
- Approved drafts appear in the Cases card and are included in coverage/pass-rate calculations on their next run.

**Requirements:** REQ-TM-53, REQ-TM-54, REQ-TM-25

---

### TM-S32 — Stale AI drafts and re-draft workflow

**As a** QA Lead
**I want** to see when a draft is stale (skill version has advanced) and re-draft with the new version
**So that** I can benefit from AI improvements

**Acceptance criteria:**

- Given a draft case was created with `skillVersion=v1.1` and the skill has advanced to v1.2, when I view the AI Draft Inbox, the draft shows a STALE badge and a "Re-draft with latest version" button.
- Given I click "Re-draft", the system invokes `POST /ai/test-cases/draft` for the same REQ-ID with the new skill version; the prior stale draft is marked `superseded` and new candidates appear.
- Given a draft has been approved to ACTIVE, it never shows as stale (the check only applies to DRAFT cases).

**Edge notes:**

- Stale badge appears only if a newer skill version is available AND the draft has not been approved.
- Re-drafting consumes the daily quota (REQ-TM-58).

**Requirements:** REQ-TM-55, REQ-TM-72

---

### TM-S33 — AI draft autonomy level enforcement

**As a** workspace admin
**I want** AI draft visibility and approval to be controlled by the workspace's aiAutonomyLevel setting
**So that** I control how much AI autonomy my team uses

**Acceptance criteria:**

- Given `aiAutonomyLevel=DISABLED`, when I click "Generate test cases", the endpoint returns 403 and a message "AI draft generation is disabled in this workspace".
- Given `aiAutonomyLevel=OBSERVATION`, drafts are generated and visible in the Draft Inbox, but the Approve button is disabled; the case is read-only for review only.
- Given `aiAutonomyLevel=SUPERVISED` or `AUTONOMOUS`, drafts are generated and fully manageable (approve/reject/edit).
- Given any autonomy level, AI NEVER transitions a case to ACTIVE directly; human approval is always required (REQ-TM-56).

**Edge notes:**

- Autonomy level is set in Platform Center and applies workspace-wide.
- The OBSERVATION mode allows users to see what AI would do without committing.

**Requirements:** REQ-TM-56

---

### TM-S34 — AI draft prompts are safe and grounded

**As a** Security reviewer
**I want** AI draft prompts to never include failure excerpts, environment secrets, or credentials
**So that** casual use of the AI feature does not leak sensitive data

**Acceptance criteria:**

- Given I invoke test-case generation for REQ-2050, when the `test-case-drafter` skill is called, the prompt includes only: REQ-2050's text + any prior case titles for the same REQ-ID (for consistency).
- Given REQ-2050's description contains "password=secret123" (an example in the spec), the prompt omits the example and redacts the reference.
- Given a prior case has a failure excerpt, that excerpt is NOT included in the prompt.

**Edge notes:**

- Prompt safety is enforced by the Testing Management service before invoking the skill.
- The `test-case-drafter` skill never has access to run failure excerpts or environment config.

**Requirements:** REQ-TM-81

---

## Environment Registry

### TM-S35 — Manage the environment registry

**As a** QA Lead
**I want** to create, edit, and archive environments (e.g., dev, staging, prod-shadow) with metadata
**So that** test runs can reference them and I can organize testing by environment

**Acceptance criteria:**

- Given I navigate to Platform Center's Environment Registry for this workspace, when I create an environment with name "staging", description "Staging cloud env", kind "STAGING", and url "https://staging.example.com", the environment is persisted and available for run ingestion.
- Given an environment is archived, the `archived` flag is set and new runs cannot be ingested for that environment; existing runs remain visible.
- Given I edit an environment's description, the change is persisted and audited.

**Edge notes:**

- Environment names are unique within a workspace (case-insensitive).
- Kind options: DEV, STAGING, PROD, EPHEMERAL, OTHER.
- The URL is optional and used for deep-linking; it never stores credentials (REQ-TM-80).

**Requirements:** REQ-TM-60

---

### TM-S36 — Auto-create unknown environments from run ingestion

**As a** operator
**I want** unknown environment names in run ingestion to be auto-created with kind=OTHER
**So that** tests can ingest without blocking on missing environment setup

**Acceptance criteria:**

- Given a run is ingested with environment="qa-branch-pr-123" (not previously registered), when the run is persisted, a new Environment row is created with `kind=OTHER` and `archived=false`.
- Given the auto-created environment is unknown, the Platform Center diagnostics dashboard flags it for the operator to review and reclassify (e.g., to EPHEMERAL).

**Edge notes:**

- Auto-created environments are surfaced in a "Unknown environments detected" diagnostics section.
- The operator can later re-classify them via the Environment Registry.

**Requirements:** REQ-TM-61

---

### TM-S37 — Enforce run ingestion on archived environments

**As a** QA Lead
**I want** archived environments to reject new runs
**So that** old testing infrastructure cannot accidentally receive new test results

**Acceptance criteria:**

- Given environment "legacy-qa" is archived, when I upload a run with environment="legacy-qa", the ingestion is rejected with 400 and a message "Environment legacy-qa is archived; please choose an active environment".
- Given the environment is unarchived, the run can be ingested again.

**Edge notes:**

- Archived environments remain visible for historical run queries.

**Requirements:** REQ-TM-62

---

## Error Handling and Degraded States

### TM-S38 — Per-card error isolation and recovery

**As a** user
**I want** a single failing card to show error+retry without taking down the page
**So that** I can keep working with the cards that loaded

**Acceptance criteria:**

- Given the Runs card times out at 500ms on Plan Detail, when the page renders, all other cards (Header, Cases, Coverage, etc.) load normally and the Runs card shows "Error loading runs" with a Retry button.
- Given I click Retry, only the Runs card re-requests; other cards' state is preserved.
- Given the backend returns a workspace-level 403 (access denied), the entire page shows a single "Access denied" error screen (not per-card errors).

**Edge notes:**

- Per-card timeout is 500ms per the performance budget (REQ-TM-92).
- Global access errors (403, 401) render at page level; transient errors (5xx, timeouts) render per-card.

**Requirements:** REQ-TM-70, REQ-TM-18

---

### TM-S39 — Distinguish empty states from errors

**As a** user
**I want** empty states to clearly differentiate "nothing yet", "no access", and "generation failed"
**So that** I know what action to take

**Acceptance criteria:**

- Given a workspace has zero plans, the Catalog shows: "No test plans yet in this workspace. Create a plan to get started."
- Given a plan has zero cases, the Cases card shows: "No test cases in this plan. Add a case or generate with AI."
- Given a plan has zero runs, the Recent Runs card shows: "No test runs yet. Upload your first run results to see execution history."
- Given no AI drafts exist, the Draft Inbox shows: "No pending AI drafts. Generate candidates from a requirement."
- Given the workspace's aiAutonomyLevel=DISABLED, the Draft Inbox shows: "AI draft generation is disabled for this workspace. Contact your workspace admin."
- Given AI generation failed (e.g., API error), the Draft Inbox shows: "Draft generation failed. Retry or contact support."

**Edge notes:**

- Empty states have an icon + short copy + optional action button (e.g., "Create plan").
- Error empty states show Retry (if retryable) or Contact Support.

**Requirements:** REQ-TM-71

---

### TM-S40 — Unverified vs. missing requirement links

**As a** user
**I want** REQ-ID chips to always render even if unverified, never silently hide
**So that** absence of a chip means "not linked", not "failed to load"

**Acceptance criteria:**

- Given a case links to REQ-2050 but the Requirement slice is temporarily unavailable, when Case Detail renders, the REQ chip appears with "UNVERIFIED" status (grey LED) and tooltip "Requirement status unknown (service unavailable)".
- Given the service recovers, the next page load re-verifies and the chip updates to GREEN/AMBER/RED accordingly.
- Given a case has zero linked REQs, the REQ section shows "Not linked to any requirements" (no chips at all).

**Edge notes:**

- UNVERIFIED (grey) is different from NOT_FOUND (red) or UNVISIBLE (amber).
- The distinction prevents confusing "missing link" with "missing REQ".

**Requirements:** REQ-TM-73

---

## Security, Privacy, and Governance

### TM-S41 — Redact secrets from run failure excerpts

**As a** Security reviewer
**I want** common secret patterns (AWS keys, GitHub tokens) to be redacted from failure excerpts before storage
**So that** casual viewing of a failure page does not leak credentials

**Acceptance criteria:**

- Given a run's failure excerpt contains "AKIAIOSFODNN7EXAMPLE" (AWS key), when the run is ingested, the stored excerpt shows "AKIA****REDACTED".
- Given a failure excerpt contains "ghp_1234567890abcdefghijklmnopqr" (GitHub token), it is redacted to "ghp_****REDACTED".
- Given a failure excerpt contains "Bearer eyJhbGc..." (JWT), it is redacted to "Bearer ****REDACTED".
- Given an excerpt is redacted, the redaction is permanent; the raw excerpt is never stored in the Control Tower.

**Edge notes:**

- Redaction patterns are applied at ingestion time (REQ-TM-80).
- The same redaction is applied to failure excerpts shown to users AND to any excerpts passed to AI (REQ-TM-81).

**Requirements:** REQ-TM-80

---

### TM-S42 — Audit all writes: plan CRUD, case edits, run ingestion, AI approvals

**As a** compliance officer
**I want** every write (plan create/update/delete, case link changes, run ingestion, AI draft approval) to emit an audit log
**So that** I can track who did what when

**Acceptance criteria:**

- Given I create a test plan, an AuditLog entry is emitted with actor=current user, action=PLAN_CREATE, timestamp, and the plan details.
- Given I edit a case's linked REQs, an entry records actor, action=CASE_UPDATE, field=linkedReqs, and before/after values.
- Given a run is ingested (manual or webhook), an entry records actor (uploader or CI system), action=RUN_INGESTED, environment, and case-count summary.
- Given a QA Lead approves an AI draft, an entry records actor, action=AI_DRAFT_APPROVED, sourceReqId, and skillVersion.

**Edge notes:**

- Audit entries are emitted via the shared AuditLogEmitter.
- Entries are immutable and queryable via a standard audit query API.

**Requirements:** REQ-TM-82

---

### TM-S43 — Role-based access control for writes

**As a** workspace admin
**I want** write operations to enforce role requirements: only QA Lead/Tech Lead can approve AI drafts, only QA Lead/Tech Lead/PM can create/archive plans
**So that** non-leads cannot accidentally change the test roadmap

**Acceptance criteria:**

- Given I am an Engineer, when I try to approve an AI draft, the button is disabled and a tooltip says "Only QA Leads or Tech Leads can approve drafts".
- Given I am a PM, when I try to create a plan, the button is enabled; PMs can create/archive plans (REQ-TM-83).
- Given I am a QA Lead, I can approve drafts, create/archive plans, and edit cases; I have full write access.
- Given I am an Engineer, I can view all plans/cases/runs but cannot edit; read-only access.

**Edge notes:**

- Roles are project-scoped; a user can be QA Lead on one project and Engineer on another.

**Requirements:** REQ-TM-83

---

### TM-S44 — Emit lineage events for cross-slice traceability

**As a** Platform Center operator
**I want** all Testing Management writes to emit a LineageEvent naming Testing Management as the authoring domain
**So that** the full traceability chain (Req → Case → Run) is visible across slices

**Acceptance criteria:**

- Given a plan is created, a LineageEvent is emitted with `domain=TESTING_MANAGEMENT`, `eventType=PLAN_CREATED`, and the plan ID.
- Given a case's REQ links change, a LineageEvent is emitted with `domain=TESTING_MANAGEMENT`, `eventType=CASE_REQ_LINKED`, the case ID, and old/new REQ-IDs.
- Given a run is ingested, a LineageEvent is emitted with `domain=TESTING_MANAGEMENT`, `eventType=RUN_INGESTED`, plan ID, and case-count summary.

**Edge notes:**

- LineageEvents are emitted synchronously during the write (or via an event queue if async processing is enabled).

**Requirements:** REQ-TM-84

---

### TM-S45 — Secure webhook ingestion with HMAC-SHA256 signature verification

**As a** operator
**I want** run-ingestion webhooks from external CI to be signed with HMAC-SHA256
**So that** only authorized CI systems can ingest runs

**Acceptance criteria:**

- Given a CI system sends a webhook with a signed payload (HMAC-SHA256 using a per-workspace shared secret), when the webhook endpoint receives it, the signature is verified.
- Given the signature is valid, the run is ingested normally.
- Given the signature is invalid or missing, the request is rejected with 401 and an AuditLog entry records the failed verification attempt.
- Given a new workspace is created, the shared secret is issued via Platform Center and made available to the operator to configure in their CI system.

**Edge notes:**

- Shared secret is rotated via Platform Center (out of scope for V1 but the architecture must support it).
- Webhook signature verification is analogous to REQ-CB-65 in Code & Build.

**Requirements:** REQ-TM-85

---

## Accessibility, i18n, Performance

### TM-S46 — Accessible design with keyboard navigation and ARIA

**As a** accessibility reviewer
**I want** all Testing Management pages to meet WCAG 2.1 AA and use semantic HTML + ARIA roles
**So that** users with assistive technologies can navigate and understand the interface

**Acceptance criteria:**

- Given I navigate using only the keyboard (Tab, Enter, Escape), all interactive elements (buttons, links, chips, cards) are reachable and activate correctly.
- Given a modal or drawer is opened, keyboard focus is trapped within it; Escape closes the modal.
- Given buttons and controls are semantic (not divs with click handlers), they have proper roles and ARIA labels.
- Given the Results table has many rows, column headers are marked with `scope="col"` and row headers with `scope="row"`.

**Edge notes:**

- All cards use `<section>` with `aria-labelledby` to the card title.
- LEDs and color-coded indicators use shape + text redundancy (e.g., "PASS" label + green LED, not just green).

**Requirements:** REQ-TM-90

---

### TM-S47 — Internationalization with shared token set

**As a** i18n lead
**I want** all user-facing text to use the shared shell's i18n token set with no hard-coded English strings
**So that** the UI can be translated without code changes

**Acceptance criteria:**

- Given I view any Testing Management page, all visible text (button labels, card titles, empty-state copy, error messages) uses i18n tokens (e.g., `i18n.t('testing.plan.title')`).
- Given a new language is added to the shared i18n config, Testing Management pages automatically render in that language without code changes.

**Edge notes:**

- Markdown content (case steps, expected results) is user-generated and not i18n'd.
- Dynamic values (plan names, REQ-IDs) are interpolated into i18n templates.

**Requirements:** REQ-TM-91

---

### TM-S48 — Performance budgets for page load and per-card rendering

**As a** performance engineer
**I want** Catalog to load in ≤1200ms P95, Plan Detail in ≤1500ms P95, Run Detail in ≤1500ms P95
**So that** the UI remains responsive for users

**Acceptance criteria:**

- Given the Catalog page is loaded, the time to interactive (TTI) is ≤1200ms at P95 (measured via Lighthouse or RUM).
- Given Plan Detail with 6 cards is loaded, TTI is ≤1500ms P95; each card has a per-card timeout of 500ms.
- Given Run Detail with 100 cases is loaded, TTI is ≤1500ms P95; large tables are virtualized.
- Given all cards load successfully, the total JS bundle for the Testing Management slice is ≤300 KB gzipped.

**Edge notes:**

- Per-card timeout is 500ms; if a card exceeds this, it shows error+retry and other cards continue rendering.
- Run case tables with >50 rows are virtualized (window-based rendering) to avoid DOM bloat.

**Requirements:** REQ-TM-92, REQ-TM-93

---

### TM-S49 — Virtual rendering for large case outcome tables

**As a** engineer reviewing a run with 500 cases
**I want** the case outcomes table to render only visible rows (virtual scrolling)
**So that** the page remains responsive even with many cases

**Acceptance criteria:**

- Given a run has 500 case outcomes, when Run Detail loads, only the first ~20 visible rows are in the DOM; as I scroll, new rows are rendered and old ones removed.
- Given the table has 500 rows, the total gzipped JS is still ≤300 KB and TTI remains <1500ms.
- Given I search within the table (e.g., filter by case title), the virtual list reflows to show matching rows only.

**Edge notes:**

- Virtual scrolling uses a standard library (e.g., vue-virtual-scroller).
- Search/filter within a large table still recomputes the visible set efficiently.

**Requirements:** REQ-TM-93

---

## Cross-cutting

### TM-S50 — Deep-link parity for all entities

**As a** user
**I want** every entity (plan, case, run, REQ traceability view) to be independently URL-addressable
**So that** I can share links in chat and teammates land exactly where I was

**Acceptance criteria:**

- Given I navigate to `/testing/plans/{planId}` directly, Plan Detail loads as if I had drilled in from the Catalog.
- Given I navigate to `/testing/cases/{caseId}` directly, Case Detail loads with breadcrumbs showing the owning plan.
- Given I navigate to `/testing/runs/{runId}` directly, Run Detail loads with breadcrumbs showing the owning plan.
- Given I navigate to `/testing/traceability?reqId=REQ-2050` directly, the Traceability view loads and shows all cases linked to that REQ.

**Edge notes:**

- All URLs are shareable and work whether accessed via navigation or direct entry.
- Breadcrumbs always show the full path: Workspace → Testing → [Entity].

**Requirements:** REQ-TM-90

---

### TM-S51 — Run result format support (JUnit, TestNG, Playwright, Cypress)

**As a** operator
**I want** run ingestion to support JUnit XML, TestNG XML, Playwright JSON, and Cypress Mochawesome JSON
**So that** diverse test frameworks can report to the Control Tower

**Acceptance criteria:**

- Given I upload a JUnit XML file from Maven, the XML is parsed and case outcomes (pass/fail/skip) are extracted correctly.
- Given I upload a TestNG XML file, it is parsed and outcomes are extracted.
- Given I upload a Playwright JSON report, the report structure is parsed and case outcomes are extracted.
- Given I upload a Cypress Mochawesome JSON file, the report is parsed and outcomes are extracted.
- Given the file format is unrecognized, the ingestion fails with a clear error message indicating which formats are supported.

**Edge notes:**

- Format detection can be based on file extension or content inspection.
- All formats are normalized to a common internal representation (case ID, status, failure excerpt).

**Requirements:** REQ-TM-34

---

### TM-S52 — Manual and webhook run ingestion paths

**As a** QA Lead
**I want** to upload run results manually via the UI OR allow external CI to send webhook-ingested runs
**So that** runs can flow in from any CI system

**Acceptance criteria:**

- Given I click "Upload run results" on a plan, a file picker opens; I select a JUnit XML file and upload it. When successful, the run appears in the Recent Runs list.
- Given a CI webhook delivers a signed payload with run results, the webhook endpoint processes it (signature verified via REQ-TM-85) and the run is ingested without manual intervention.
- Given both manual and webhook runs are ingested, both appear in the Recent Runs list with the correct trigger source (manual vs. CI).

**Edge notes:**

- Manual uploads require QA Lead role on the plan's owning project.
- Webhook ingestion requires the signature to be valid; invalid signatures are rejected (REQ-TM-85).

**Requirements:** REQ-TM-35

---

## Story coverage summary

| Requirement range | Covered by |
| ---|---|
| REQ-TM-01…07 (Catalog) | TM-S1, TM-S2, TM-S3, TM-S4, TM-S5 |
| REQ-TM-10…18 (Plan Detail) | TM-S6, TM-S7, TM-S8, TM-S9, TM-S10, TM-S11, TM-S12 |
| REQ-TM-20…26 (Case Detail) | TM-S13, TM-S14, TM-S15, TM-S16, TM-S17 |
| REQ-TM-30…36 (Run Detail) | TM-S18, TM-S19, TM-S20, TM-S21, TM-S22, TM-S23, TM-S24 |
| REQ-TM-40…44 (Traceability) | TM-S25, TM-S26, TM-S27, TM-S28 |
| REQ-TM-50…58 (AI draft generation) | TM-S29, TM-S30, TM-S31, TM-S32, TM-S33, TM-S34 |
| REQ-TM-60…62 (Environment registry) | TM-S35, TM-S36, TM-S37 |
| REQ-TM-70…74 (Error handling) | TM-S38, TM-S39, TM-S40 |
| REQ-TM-80…85 (Security/governance) | TM-S41, TM-S42, TM-S43, TM-S44, TM-S45 |
| REQ-TM-90…93 (a11y/i18n/perf) | TM-S46, TM-S47, TM-S48, TM-S49, TM-S50, TM-S51, TM-S52 |
