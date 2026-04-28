# Project Management Stories

## Scope

This document defines the user stories for the **Project Management** page — the delivery operating plane of the Agentic SDLC Control Tower. It covers two coupled views (Portfolio and Plan) in a single slice, as decided 2026-04-17.

Portfolio view: Workspace-scoped cross-project aggregation for PMOs, Delivery Managers, and Application Owners.
Plan view: Per-project editor for the Project Manager — milestones, capacity, risks, dependencies, delivery progress, audit.

## Traceability

- Requirements: [project-management-requirements.md](../01-requirements/project-management-requirements.md)
- PRD: [agentic_sdlc_control_tower_prd_v0.9.md](../01-requirements/agentic_sdlc_control_tower_prd_v0.9.md) §11.5, §11.3, §13.1, §15.2, §15.4, §16.2
- Cross-slice refs: [project-space-stories.md](project-space-stories.md), [team-space-stories.md](team-space-stories.md), [shared-app-shell-stories.md](shared-app-shell-stories.md)
- Visual reference: no dedicated Project Management HTML mockup yet; reuse the MilestoneExecutionHub pattern from [../05-design/Project Space.html](../05-design/Project%20Space.html)

---

## Portfolio View Stories

### Story S1: View Portfolio Summary Bar

As a Delivery Manager or Application Owner,
I want to see a Workspace-scoped summary of active projects, health, at-risk milestones, critical risks, blocked dependencies, and pending approvals,
so that I can immediately spot whether the portfolio is on-track or needs my attention.

#### Acceptance Criteria

- The Portfolio Summary Bar renders at the top of the Portfolio view
- Counters shown: active projects, projects with Red health, projects with At-Risk or Slipped milestones, Critical-severity risks, blocked dependencies, pending approvals, last-refreshed timestamp
- A separate "AI pending review" counter shows the number of milestones/risks/deps with an unreviewed AI recommendation
- Clicking any counter filters the rest of the Portfolio view to that subset
- All counters strictly honor the current Workspace context

#### Edge Notes

- Phase A uses mocked aggregate data; Phase B fetches from `GET /api/v1/project-management/portfolio/summary?workspaceId=...`
- Cross-Workspace projects must never appear (isolation rule)
- If Workspace has zero active projects, the bar renders an explicit empty state, not a spinner

#### Requirement Refs

REQ-PM-10, REQ-PM-11, REQ-PM-12, REQ-PM-170

---

### Story S2: Scan the Portfolio Milestone Heatmap

As a Delivery Manager,
I want to see a heatmap of projects × time colored by milestone status,
so that I can spot slippage hot-spots across the Workspace at a glance.

#### Acceptance Criteria

- Rows are projects; columns are time windows (weeks by default; switchable to months or milestone index)
- Each cell is colored by the dominant milestone status in that window: Not-Started / In-Progress / At-Risk / Completed / Slipped
- At-Risk and Slipped cells use the crimson / warning accent
- Clicking a cell opens the Plan view for that project, scrolled to the milestone
- Empty time windows (no milestones) render as neutral cells, not blank

#### Edge Notes

- Time granularity default is configuration-driven per Workspace (REQ-PM-23)
- Projects with no milestones render as a thin neutral row with a "No plan" chip

#### Requirement Refs

REQ-PM-20, REQ-PM-21, REQ-PM-22, REQ-PM-23

---

### Story S3: Review Cross-Project Capacity & Allocation

As a Delivery Manager or PMO,
I want to see a member × project allocation matrix with over-allocation and under-utilization flagged,
so that I can rebalance capacity before it becomes a slippage problem.

#### Acceptance Criteria

- Rows are members; columns are projects; cells show % allocation for the current planning window
- Any member whose total across projects exceeds 100% is flagged crimson with the overage amount
- Any member whose total is below the configured under-utilization threshold (default 50%) is flagged amber
- The Portfolio view is read-only for capacity; a "Edit in Plan view" link appears for each project column
- The matrix is sortable by member name, member total, and project total

#### Edge Notes

- The under-utilization threshold is Workspace-configurable (REQ-PM-32)
- Members with zero allocation across all active projects render in a separate "Unallocated" section
- Historical windows are V2; V1 shows current window only

#### Requirement Refs

REQ-PM-30, REQ-PM-31, REQ-PM-32, REQ-PM-33

---

### Story S4: Identify Risk Concentration

As an Application Owner,
I want to see top risks across all projects in the Workspace ordered by severity × age, plus a severity × category heatmap,
so that I can drive escalations and allocate mitigation effort.

#### Acceptance Criteria

- The Risk Concentration card lists the top N risks (default 20), each with title, severity, category, project, owner, age-in-days, latest mitigation note, AI recommendation (if any)
- A companion severity × category heatmap shows aggregated counts
- Category and severity filters carry into Plan view drill-in
- Clicking a risk opens the Plan view for that project with the risk expanded

#### Edge Notes

- Severity ordering is Critical → High → Medium → Low; ties break by age descending
- AI recommendation column is empty if no active AI suggestion exists (not hidden)

#### Requirement Refs

REQ-PM-40, REQ-PM-41, REQ-PM-42

---

### Story S5: Detect Dependency Bottlenecks

As a Delivery Manager,
I want to see the top blocked dependencies across the Workspace with reason, owner, and days blocked,
so that I can drive cross-project unblocking conversations.

#### Acceptance Criteria

- The Dependency Bottlenecks card lists the top N edges (default 15) ordered by days blocked descending
- Each row shows: source project → target (external or internal), relationship type, blocker reason, owner team, days blocked, AI proposal (if any)
- An "Open Resolver" action deep-links to the Dependency Resolver in the source project's Plan view
- External targets (not a registered project) render with an "External" chip and no drill-in link

#### Edge Notes

- If multiple blockers exist between the same pair of projects, each edge is a separate row
- V1 does not compute multi-project critical path — that is flagged as V2

#### Requirement Refs

REQ-PM-50, REQ-PM-51

---

### Story S6: Inspect Portfolio Delivery Cadence

As a PMO or Delivery Manager,
I want to see throughput, cycle time, milestone hit-rate, and plan stability for the Workspace with trendlines,
so that I can report delivery health up the chain and spot regressions.

#### Acceptance Criteria

- The Cadence Metrics card shows four metrics (throughput, cycle time median + p90, milestone hit-rate, plan stability) for 4-week and 12-week rolling windows
- Each metric includes a trendline against the prior comparable window with ▲/▼ and absolute delta
- Hovering a metric reveals the raw numerator / denominator for auditability
- Metrics are not shown as "0" when no data exists in the window; they render as "—" with an explanatory tooltip

#### Edge Notes

- Phase A uses mocked cadence data; Phase B derives metrics from milestone history
- Plan stability = fraction of milestones whose target date changed in the window

#### Requirement Refs

REQ-PM-60, REQ-PM-61

---

## Plan View Stories

### Story S7: Open the Plan for a Single Project

As a Project Manager,
I want to open `/project-management/:projectId` and see the identity, stage, plan health, next milestone, and accountable PM,
so that I have a single view of my plan with all the controls I need.

#### Acceptance Criteria

- The Plan Header shows: project name + id, parent Workspace and Application breadcrumbs, lifecycle stage, plan health LED, next milestone short-label + target date, accountable PM + backup, active Autonomy Level, last plan update timestamp
- Plan health LED is server-derived and hover reveals contributing factors (milestone slippage, critical risks, blocked deps, over-allocated capacity)
- A persistent "View in Project Space" link preserves projectId

#### Edge Notes

- Phase B data comes from `GET /api/v1/project-management/plan/:projectId/header`
- Parent Workspace is auto-resolved if the context bar's workspaceId differs (REQ-PM-172)

#### Requirement Refs

REQ-PM-70, REQ-PM-71, REQ-PM-84, REQ-PM-140, REQ-PM-162, REQ-PM-172

---

### Story S8: Create, Edit, and Reorder Milestones

As a Project Manager,
I want to CRUD milestones with label, target date, owner, ordering, and description,
so that I can define and maintain the plan for my project.

#### Acceptance Criteria

- A "New milestone" action opens an inline creation row
- Fields: label (required), target date (required, ≥ today), owner (member select, required), description (optional, ≤ 500 chars)
- Drag-and-drop (or numeric ordering) changes milestone order
- Archiving a milestone preserves its audit record and hides it from the active list (a "Show archived" toggle is available)
- Optimistic UI updates on save; rollback + toast on server error

#### Edge Notes

- Target date past today is allowed (historical milestones may be re-added during plan recovery)
- Duplicate labels within the same project are allowed but generate a warning

#### Requirement Refs

REQ-PM-80, REQ-PM-184, REQ-PM-185

---

### Story S9: Move a Milestone Through Its Lifecycle

As a Project Manager,
I want to move a milestone through Not-Started → In-Progress → At-Risk / Completed / Slipped with the appropriate required fields,
so that the plan reflects real execution state and Project Space reads consistent data.

#### Acceptance Criteria

- Only transitions defined by REQ-PM-81 are offered in the UI; others are hidden, not just disabled
- Transitioning to At-Risk or Slipped requires a slippage reason (min 10 chars)
- Transitioning to Completed records the `completedAt` timestamp server-side
- Invalid transitions attempted via API are rejected with HTTP 409 and a clear error code
- The change appears in the Plan Change Log immediately

#### Edge Notes

- Re-opening a Completed milestone is not allowed in V1; archive-and-replace is the pattern
- Recovering from Slipped requires a new target date (date validation enforced)

#### Requirement Refs

REQ-PM-81, REQ-PM-82, REQ-PM-130

---

### Story S10: Review AI Slippage Predictions on Milestones

As a Project Manager,
I want each In-Progress milestone to show an AI slippage-risk score and contributing factors with accept / dismiss controls,
so that I can proactively mitigate slippage and track that AI is genuinely useful.

#### Acceptance Criteria

- Every In-Progress milestone has a slippage-risk chip: Low / Medium / High (or "No signal" when AI has no basis)
- Expanding the chip reveals up to five contributing factors with evidence links
- "Accept" records the prediction as acknowledged and prompts to draft a mitigation action (routed to Risk Registry or AI Command Panel)
- "Dismiss" records the dismissal with an optional reason; subsequent predictions for the same milestone are suppressed for 24 hours
- Every accept / dismiss action is audited as a Skill Execution outcome

#### Edge Notes

- Slippage score is computed asynchronously by a platform skill; the UI renders "Computing…" until the first result arrives
- Evidence links may point to Incident, Requirement, Dependency, Capacity — each as a deep-link

#### Requirement Refs

REQ-PM-83, REQ-PM-152

---

### Story S11: Edit Member × Milestone Capacity Allocation

As a Project Manager,
I want to edit a matrix of members × milestones with % allocation per cell,
so that I can plan capacity against the real milestone breakdown.

#### Acceptance Criteria

- Cells are inline-editable number inputs (0–100)
- Row totals (per member) and column totals (per milestone) update on input
- Row totals > 100% flag crimson and require a justification string before save
- Inline save per cell; optimistic UI with rollback on 409 / 422
- A "Copy row" / "Copy column" affordance for repeating allocation patterns

#### Edge Notes

- V1 uses a single planning window (current milestone horizon); rolling-window windows are V2
- Members are filtered to those assigned to the project; adding new members happens in Access Management

#### Requirement Refs

REQ-PM-90, REQ-PM-91, REQ-PM-92, REQ-PM-184

---

### Story S12: Surface Backup Coverage Gaps

As a Project Manager,
I want the Capacity card to flag members who are sole-owners or oncall for a milestone without a named backup,
so that I don't hit single-point-of-failure coverage during delivery.

#### Acceptance Criteria

- Any row flagged with "no backup" shows an amber chip with the affected milestone names
- A "Manage in Access Management" link appears for users with the required permission; absent otherwise
- Resolution (assigning a backup) happens in Platform Center Access Management; Project Management does not edit role assignments

#### Edge Notes

- Definition of "oncall" comes from the shared oncall schedule service used by Team Space

#### Requirement Refs

REQ-PM-93, REQ-PM-161

---

### Story S13: Accept or Reject AI Capacity Rebalance Suggestions

As a Project Manager,
I want AI to suggest capacity rebalances based on observed slippage signals and accept with one click,
so that I get genuine AI leverage on rebalancing without losing control.

#### Acceptance Criteria

- When conditions apply, the Capacity card surfaces a "Rebalance suggested" banner with a specific proposal (e.g., "Shift 20% of Dev-A from M3 to M2")
- Accept applies the proposal, produces the corresponding cell edits (auditable), and records a Skill Execution with evidence
- Reject dismisses the proposal and records the dismissal with an optional reason
- A proposal that would push a row total > 100% must still require the justification string before application

#### Edge Notes

- Skill input includes: milestone slippage scores, current allocations, member availability signals
- Only one active proposal at a time per project

#### Requirement Refs

REQ-PM-94, REQ-PM-152

---

### Story S14: Run the Risk Registry

As a Project Manager,
I want to create, edit, and move risks through a state machine (Identified → Acknowledged → Mitigating → Resolved, with Escalate),
so that I have an auditable risk trail and the Project Space risk card reflects the current state.

#### Acceptance Criteria

- Risk CRUD fields: title (required), severity, category, owner, description, linked artifact (Incident / Approval / Task)
- State transitions are enforced server-side; invalid transitions return 409
- Escalation creates a pending Approval routed to the Application Owner (via platform approval pipeline)
- Mitigation notes are required when transitioning to Mitigating; resolution notes required when transitioning to Resolved
- Risks are ordered by severity then age, identical to Project Space

#### Edge Notes

- Risks created here feed the Portfolio Risk Concentration card (REQ-PM-40)
- Project Space shows the same risks in read-only mode

#### Requirement Refs

REQ-PM-100, REQ-PM-101, REQ-PM-102

---

### Story S15: Accept or Edit an AI Mitigation Draft

As a Project Manager,
I want the AI Command Panel to draft a mitigation note for an Acknowledged risk on demand,
so that I can save time drafting and still maintain ownership.

#### Acceptance Criteria

- The AI Command Panel exposes a "Draft mitigation" action for Acknowledged risks
- The draft includes: summary, proposed mitigation steps, evidence references, confidence score
- Accept writes the draft into `mitigation_note` and records a Skill Execution
- Edit-before-accept is supported; the edit is diffed against the AI draft in the audit record

#### Edge Notes

- If the AI has insufficient evidence, the panel shows "Not enough context" and does not fabricate

#### Requirement Refs

REQ-PM-103, REQ-PM-152

---

### Story S16: Resolve Dependencies Through an Approval Workflow

As a Project Manager,
I want to move a dependency through Proposed → Negotiating → Approved / Rejected / At-Risk / Resolved with a counter-signature requirement for internal approvals,
so that cross-project agreements are explicit and auditable.

#### Acceptance Criteria

- Dependency CRUD fields: source project (this), target (internal project id or external descriptor), relationship type, owner team, target dates, contract notes
- Internal `Approved` transitions require counter-signature from a user with target-side approve authority; until received, state is `NEGOTIATING`
- External targets record a logged contract commitment (free-text commitment + source reference)
- Rejections require a reason (min 10 chars)
- Deep-link to the target's Project Space when internal

#### Edge Notes

- Counter-signature is enforced by the backend based on role assignments in the target project
- A dependency without a registered internal target cannot be marked `Approved` via counter-signature; it must use the external path

#### Requirement Refs

REQ-PM-110, REQ-PM-111, REQ-PM-112

---

### Story S17: Review AI Dependency Resolution Proposals

As a Project Manager,
I want AI to propose workarounds for blocked dependencies (e.g., "route via shared event bus", "defer M3 by 5 days"),
so that I get leverage on cross-team unblocking without losing control over the final decision.

#### Acceptance Criteria

- A blocked dependency shows an "AI proposal" chip when one is available
- Expanding shows the proposal, rationale, evidence, and confidence
- Accept records acceptance as a Skill Execution but does not auto-apply — the PM must still drive the actual change (e.g., open an approval, update a milestone)
- Dismiss records dismissal with an optional reason

#### Edge Notes

- AI proposals are advisory only; they do not mutate entities directly
- A dismissed proposal is suppressed for 24 hours for the same dep

#### Requirement Refs

REQ-PM-113, REQ-PM-152

---

### Story S18: Read the Project Delivery Progress Strip

As a Project Manager or Tech Lead,
I want to see the 11-node SDLC chain scoped to my project with throughput, health, and slippage markers,
so that I can see whether the plan is in rhythm across all stages and drill into any node.

#### Acceptance Criteria

- The strip renders all 11 nodes (Requirement → Story → Spec → Architecture → Design → Tasks → Code → Test → Deploy → Incident → Learning)
- Each node shows: throughput count in the current window, health LED, slippage ▼ marker when throughput dropped > threshold vs prior comparable window
- Spec node is visibly emphasized
- Clicking a node opens the corresponding lifecycle page filtered by projectId

#### Edge Notes

- Throughput window default is 2 weeks; configuration-driven
- Nodes for lifecycle pages that do not yet exist render in a disabled state, not broken

#### Requirement Refs

REQ-PM-120, REQ-PM-121, REQ-PM-122, REQ-PM-07

---

### Story S19: Review the Plan Change Log

As a Project Manager, Application Owner, or Auditor,
I want a chronological log of all plan-affecting changes (milestones, capacity, risks, dependencies, AI recommendations accepted / dismissed),
so that I can explain what happened and when, and satisfy audit.

#### Acceptance Criteria

- Log entries are reverse-chronological (most recent first) with actor (user or AI skill), action, target, before → after state, timestamp, audit link
- Filters: actor type (Human / AI), target type (Milestone / Capacity / Risk / Dependency / AI-suggestion), date range
- Each entry expands to reveal the full before / after delta
- Pagination: 50 entries per page; scrollable or keyset

#### Edge Notes

- V1 retains 180 days of log in the hot store; older entries are available via Report Center
- AI entries include skill id, invocation id, and confidence

#### Requirement Refs

REQ-PM-130, REQ-PM-131, REQ-PM-132

---

## Cross-View Stories

### Story S20: Deep-Link into Plan View from Multiple Entry Points

As any team member,
I want to reach the Plan view from Dashboard slippage cards, Team Space Project Distribution, Project Space Milestone card, or direct URL,
so that I can land in the right context without re-navigating.

#### Acceptance Criteria

- Dashboard slippage / risk cards link into the Portfolio view with the risk filter applied
- Team Space Project Distribution links into the Portfolio view scoped to the Workspace's active projects
- Project Space Milestone Execution Hub card links into the Plan view for the project, scrolled to the relevant milestone
- Direct URLs `/project-management`, `/project-management?riskSeverity=CRITICAL`, and `/project-management/:projectId?milestoneId=...` all work and preserve breadcrumbs

#### Edge Notes

- Breadcrumb trail example: `Dashboard / Team Space (ws-42) / Project Management / Plan (proj-8821) / Requirement Management`
- Query-param filters are preserved through drill-in navigation (REQ-PM-42)

#### Requirement Refs

REQ-PM-141, REQ-PM-142, REQ-PM-143, REQ-PM-144

---

### Story S21: Interact With the Project Management AI Command Panel

As a Project Manager,
I want the AI Command Panel to propose the right suggestions for Portfolio view versus Plan view,
so that AI matches the task I am doing without me steering every time.

#### Acceptance Criteria

- In Portfolio view, the panel shows suggestions like "Identify slippage hot-spots", "Summarize capacity imbalance", "Which dependencies are most likely to block next 2 weeks?"
- In Plan view, the panel shows suggestions like "Explain why M3 is At-Risk", "Draft mitigation for top risk", "Suggest a capacity rebalance", "Summarize recent plan changes for stand-up"
- Invoking any suggestion projects the current page context (workspaceId or projectId, top entities) into the skill input
- Every invocation records a Skill Execution with input / output / evidence / accepted-or-dismissed outcome

#### Edge Notes

- Panel respects the project's current Autonomy Level — high-autonomy projects may auto-apply low-risk recommendations (V2); V1 always requires accept

#### Requirement Refs

REQ-PM-150, REQ-PM-151, REQ-PM-152

---

### Story S22: Enforce Role-Based Write Authority

As the platform,
I want to enforce that only the right roles can edit milestones, capacity, risks, or approve dependencies,
so that governance is real, not decorative.

#### Acceptance Criteria

- Project Contributor reads the Plan view but cannot change milestone status, edit capacity, or approve dependencies; editor affordances are hidden or disabled
- Project Admin (incl. PM role) has full Plan-view edit authority for the project
- Workspace Admin has all Project-Admin powers across the Workspace
- Application Owner has read across the Application and can approve escalations
- Auditor has read-only access including the full change log
- Server-side rejections emit 403 with a clear error code and an audit entry

#### Edge Notes

- Role resolution uses the same role model as Team Space and Project Space
- Frontend hides unauthorized actions; the backend is the source of truth

#### Requirement Refs

REQ-PM-161

---

### Story S23: Preserve Plan State Across Drill-Outs

As a Project Manager,
I want to drill into Incident, Requirement, or Deployment from the Plan view and return with my expanded sections, filters, and scroll position intact,
so that I do not lose context when investigating.

#### Acceptance Criteria

- Expanded card sections are preserved across navigation
- Active filters (risk severity, dep state, actor type in change log) are preserved
- Scroll position is preserved when returning via browser back or breadcrumb
- Unsaved edits prompt a confirmation dialog before leaving

#### Edge Notes

- Preservation is per-session and per-tab; a reload resets state
- Confirmation dialog copy: "You have unsaved changes. Leave without saving?"

#### Requirement Refs

REQ-PM-144, REQ-PM-185

---

### Story S24: Render Loading / Empty / Error States Gracefully

As any user,
I want every card on either view to show a clear loading, empty, or error state independently,
so that one failed card does not break the whole page.

#### Acceptance Criteria

- Each card has its own loading skeleton
- Empty states are explicit (e.g., "No active risks", "No dependencies configured") with a helpful next-action link where meaningful
- Errors render per-card with a retry button; the rest of the page remains functional
- A page-wide "stale" banner appears when the last refresh is > 5 minutes old on the Portfolio view

#### Edge Notes

- Per-card error isolation is the shared convention (REQ-PS-123, dashboard and requirement slices)

#### Requirement Refs

REQ-PM-182, REQ-PM-183

---

## Notes / Assumptions

- V1 writes flow through the existing platform governance and audit pipeline (same pipeline used by Project Space, Team Space, Incident).
- AI recommendations are computed by platform skills outside this slice; the slice consumes and records their outcomes.
- Multi-project critical-path scheduling, Gantt view, timesheet ingestion, and cost modeling are explicitly V2.
- The slice assumes shared-app-shell routing and context bar are already in place per the shared-app-shell slice.

## Open Questions

- Should the Portfolio view support multi-Workspace rollup for users who belong to more than one Workspace? (Current answer: no — REQ-PM-170. Verify with PMO stakeholder.)
- Is there a hard limit on the number of active milestones per project where the UI should paginate? (Assumed: < 30 per REQ-PS-132; PM allows larger but paginates at > 50.)
- Which skill pack provides the slippage-prediction model? (Assumed: platform default "PlanHealth" skill; confirm with AI Center owner.)
- Counter-signature for dependency approvals — is email/async acceptable for V1, or must it be in-app only? (Assumed: in-app via Approval pipeline.)
