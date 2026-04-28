# Project Management Requirements

## Purpose

This document extracts the requirements relevant to the **Project Management** page from the full PRD (V0.9). It defines **what** the Project Management page must deliver, serving as the upstream input for the spec, architecture, design, and task documents of this slice.

Project Management is the **delivery operating plane**. Where Project Space (PRD §11.3) answers *"what is the state of this project?"*, Project Management (PRD §11.5) answers *"how do we keep the delivery rhythm across the portfolio and inside each project?"* — plans, milestones, capacity, resources, risks, dependencies, delivery progress.

This slice covers **two coupled views**:

- **Portfolio view** — a cross-project aggregation page that PMOs, delivery leads, and application owners use to monitor all projects within a Workspace (capacity, slippage, risk concentration, dependency bottlenecks).
- **Per-project Plan view** — a single-project execution workspace that the Project Manager uses to edit milestones, allocate capacity, acknowledge risks, resolve dependencies, and drive delivery rhythm.

Both views live in the same slice because they share the same underlying entities (Milestone, Risk, Dependency, Capacity Allocation) and the same governance pipeline, and users flip between them many times per day.

## Source

All requirements below are derived from:

- [agentic_sdlc_control_tower_prd_v0.9.md](agentic_sdlc_control_tower_prd_v0.9.md)
- Visual reference: no dedicated Project Management HTML mockup exists in `docs/standard-sdd/05-design/` as of 2026-04-17; the MilestoneExecutionHub pattern in [Project Space.html](../05-design/Project%20Space.html) and the general Tactical Command aesthetic in [design.md](../05-design/design.md) are the reuse anchors.
- Cross-slice references: [project-space-requirements.md](project-space-requirements.md), [team-space-requirements.md](team-space-requirements.md), [dashboard-requirements.md](dashboard-requirements.md).

Section references use the format `PRD §N.N`.

---

## 1. Scope

The Project Management page is described in the PRD as follows:

> "用于管理计划、里程碑、容量、资源、风险、依赖和交付进度。该页面偏项目运营与执行总览，重点不在文档，而在节奏控制与交付透明度。"
> — PRD §11.5

Translation: manages plans, milestones, capacity, resources, risks, dependencies, and delivery progress. The page leans toward project operations and execution overview; its focus is not on documents but on **rhythm control and delivery transparency**.

### 1.1 In Scope

**Portfolio view** (cross-project, Workspace-scoped):

- Portfolio Summary Bar (active projects, at-risk count, critical-risk count, slippage count, blocked dependencies)
- Portfolio Milestone Heatmap (projects × time, status-colored)
- Portfolio Capacity & Allocation (members × projects, % allocation, over-allocation warning)
- Portfolio Risk Concentration (top risks across projects, severity × category)
- Portfolio Dependency Bottlenecks (most-blocked / most-blocking edges across projects)
- Portfolio Delivery Cadence metrics (throughput, cycle-time, milestone hit-rate)
- Drill-in navigation into per-project Plan view

**Per-project Plan view** (single-project):

- Plan Header (project identity, lifecycle stage, plan health, next milestone, accountable PM)
- Milestone Planner (CRUD milestones, target dates, status, ordering, slippage reason, owner)
- Capacity Allocation (members × milestones, % allocation, buffer, over-allocation warning)
- Risk Registry Editor (acknowledge, mitigate, escalate, assign owner, link to Incident)
- Dependency Resolver (approve / reject / escalate dependency requests, link to upstream/downstream Project Space, contract status)
- Delivery Progress Strip (11-node SDLC chain health, scoped to this project, with throughput/slippage markers)
- Plan Change Log (audit of milestone / capacity / risk / dependency changes)
- AI Command Panel projection scoped to the project
- Governance hooks (approvals, escalations, AI autonomy level indicator)

### 1.2 Out of Scope

- Project creation / renaming / archiving (owned by Platform Center per PRD §11.12)
- Workspace / Application / SNOW Group configuration (Platform Center)
- Requirement / User-Story / Spec content editing (Requirement Management §11.4)
- Architecture / Design artifacts (Design Management §11.6)
- Code / Branch / Merge / Build actions (Code & Build §11.7)
- Test case / execution editing (Testing Management §11.8)
- Release gate, approval, rollout actions (Deployment Management §11.9)
- Incident triage and runtime action execution (Incident Management §11.10)
- Historical reporting / export / cross-Workspace rollup (Report Center §11.12)
- Resource **hiring / off-boarding / permission grants** (Access Management inside Platform Center)
- Time-tracking / timesheet ingestion (not in V1)
- Gantt-style dependency rescheduling across projects (V1 shows bottleneck list; multi-project critical-path is V2)

---

## 2. Product Context Requirements

### REQ-PM-01: Delivery operating plane

Project Management is the system's **delivery operating plane** — the primary page where a PM actively edits plan, capacity, risk ownership, and dependency resolution. It is write-heavy, unlike Project Space which is read-heavy.

> Source: PRD §11.5

### REQ-PM-02: Two coupled views in one slice

The slice must provide both a **Portfolio view** (Workspace-scoped cross-project aggregation) and a **Per-project Plan view** (single-project editor). Navigation between them must preserve filters where applicable (e.g., a risk filter applied on portfolio carries into project drill-in).

> Source: PRD §11.5 + product decision 2026-04-17

### REQ-PM-03: Four audience layers

The page must serve:

- **Project Manager / Delivery Lead** — edits milestones, allocates capacity, resolves dependencies, owns plan health. Primary editor.
- **Delivery Manager / PMO** — reads portfolio view, identifies slippage hot-spots, drives escalations.
- **Application Owner** — reads portfolio view and cross-project dependency bottlenecks within their Application.
- **Architect / Tech Lead** — reads the plan to align spec / design cadence with milestones; contributes to risks and dependencies.

> Source: PRD §6.2, §6.3, §6.4, §11.5

### REQ-PM-04: Reuse-and-extend domain model

The page must reuse existing domain entities (Project, Milestone, Risk, Dependency, Environment, Member/Role) defined by the Project Space and Team Space slices. Where PM needs additional fields (e.g., allocation %, risk mitigation status, dependency approval state), those must be introduced as **extensions** on existing entities or new entities that reference them — not duplicate entities.

> Source: CLAUDE.md Lesson #3 (package-by-feature), project-space-data-model.md

### REQ-PM-05: AI is a participant, not a widget

Every AI-surfaced item on Project Management (slippage prediction, capacity rebalance suggestion, dependency resolution proposal, risk mitigation draft) must explicitly identify AI involvement — what the AI did, what it recommends, and whether human governance is required.

> Source: PRD §7.4 — "AI 是参与者，不是旁观者"

### REQ-PM-06: Configuration-driven layout

Card order, card visibility, and card-level default filters for both the Portfolio and Plan views must be configuration-driven so a Workspace or Project can tune Project Management without code changes.

> Source: PRD §7.8

### REQ-PM-07: Spec-centric vocabulary preserved

The SDLC chain representation on Project Management must continue to emphasize Spec as the execution hub (11-node main chain, Spec highlighted).

> Source: PRD §7.6, §13.1

---

## 3. Portfolio View — Summary Bar Requirements

### REQ-PM-10: Portfolio summary counters

The Portfolio Summary Bar must display, scoped to the active Workspace:

- Total active projects count
- Projects with health = Red count
- Projects with At-Risk or Slipped milestones count
- Critical risks count (severity = Critical) across all projects
- Blocked dependencies count (dependency health = Red or blocker reason present)
- Pending approvals count (milestone approval, dependency approval, capacity approval)
- Last refreshed timestamp

> Source: PRD §11.5

### REQ-PM-11: Workspace-scoping

All counters must strictly honor the current Workspace context from the shared context bar. Cross-Workspace leakage is a defect.

> Source: PRD §9.2, §15.2

### REQ-PM-12: AI participation indicator

The Summary Bar must expose the count of milestones, risks, and dependencies that currently have an active AI recommendation pending human review.

> Source: PRD §7.4

---

## 4. Portfolio View — Milestone Heatmap Requirements

### REQ-PM-20: Project × time heatmap

The card must render a 2D heatmap where rows are projects and columns are time windows (weeks or milestones), with each cell colored by the dominant milestone status in that window (Not-Started / In-Progress / At-Risk / Completed / Slipped).

> Source: PRD §11.5 — "交付进度"

### REQ-PM-21: Slippage highlight

Any cell corresponding to a milestone in `At-Risk` or `Slipped` status must use the crimson / warning accent consistent with the design system.

> Source: PRD §15.5

### REQ-PM-22: Drill-in

Clicking a cell must open the Plan view for that Project, scrolled to the milestone in question.

> Source: REQ-PM-02

### REQ-PM-23: Time window configurability

The card must support at least three time granularities (weeks, months, milestone index). Default is configuration-driven (REQ-PM-06).

---

## 5. Portfolio View — Capacity & Allocation Requirements

### REQ-PM-30: Member × project allocation matrix

The card must display a matrix of members (rows) × projects (columns), each cell showing the member's % allocation to that project for the current planning window.

> Source: PRD §11.5 — "容量、资源"

### REQ-PM-31: Over-allocation flag

Any member whose total allocation across all projects exceeds 100% for the current window must render with a crimson flag and a short reason (e.g., "112% allocated — over by 12%").

> Source: PRD §11.5, §15.5

### REQ-PM-32: Under-utilization flag

Any member whose total allocation across all active projects is under a configured threshold (default 50%) must render with an amber flag. The threshold is configurable per Workspace.

### REQ-PM-33: Allocation source of truth

Allocations are recorded by the PM in the Plan view and flow up to Portfolio. Portfolio view is read-only; edits happen in the Plan view.

> Source: PRD §16.2 (single source of truth)

---

## 6. Portfolio View — Risk Concentration Requirements

### REQ-PM-40: Risk table aggregation

The card must list the top N risks across the Workspace (default N = 20) ordered by severity × age, each showing:

- Risk title
- Severity
- Category
- Project name (with drill-in link)
- Owner (or "Unassigned")
- Age in days
- Latest mitigation note
- AI recommendation (if any)

> Source: PRD §11.5 — "风险"

### REQ-PM-41: Severity × category heatmap

The card must also provide a severity × category heatmap for quick hot-spot identification.

### REQ-PM-42: Filter integration

Category and severity filters applied in the Portfolio view must carry into the Plan view drill-in.

---

## 7. Portfolio View — Dependency Bottlenecks Requirements

### REQ-PM-50: Bottleneck list

The card must list the top N dependency edges (default N = 15) ordered by blocker severity, each showing:

- Source project → target project (or external target)
- Relationship type (API / Data / Schedule / SLA)
- Blocker reason
- Owner team
- Days blocked
- AI recommendation (if any)

> Source: PRD §11.5 — "依赖"

### REQ-PM-51: Resolution entry point

Each bottleneck row must offer an "Open Resolver" entry that deep-links to the Plan view's Dependency Resolver for either the source or target project.

---

## 8. Portfolio View — Cadence Metrics Requirements

### REQ-PM-60: Cadence metrics

The card must show, for the Workspace, rolling windows (4, 12 weeks) of:

- Delivery throughput (milestones completed per week)
- Cycle time (milestone start → complete, median and p90)
- Milestone hit-rate (milestones completed on or before target date ÷ total completed)
- Plan stability (fraction of milestones whose target date moved in the last window)

> Source: PRD §11.5 — "交付进度"

### REQ-PM-61: Trendline

Each metric must include a trendline against the previous comparable window, with a ▲ / ▼ indicator and absolute delta.

---

## 9. Plan View — Plan Header Requirements

### REQ-PM-70: Plan identity strip

The Plan Header must display, scoped to the single active project:

- Project name and identifier
- Parent Workspace and Application (breadcrumb links)
- Lifecycle stage (Discovery / Delivery / Steady-State / Retiring)
- Plan health indicator (green / yellow / red / unknown)
- Next milestone short-label and target date
- Accountable PM and backup
- Active Autonomy Level (indicator only; edited in Platform Center)
- Last plan update timestamp

> Source: PRD §11.5, §11.12, §15.2

### REQ-PM-71: Plan health derivation

Plan health must be server-derived from: milestone slippage, open critical risks, blocked dependencies, over-allocated capacity. Hover / click reveals contributing factors.

> Source: PRD §11.5

---

## 10. Plan View — Milestone Planner Requirements

### REQ-PM-80: Milestone CRUD

The Milestone Planner must support:

- Create milestone (label, target date, owner, ordering, description)
- Edit milestone fields (all of the above, plus status and slippage reason)
- Reorder milestones (drag-and-drop or ordering field)
- Archive milestone (soft delete; audit preserved)

> Source: PRD §11.5

### REQ-PM-81: Milestone status state machine

Allowed transitions:

- `NOT_STARTED → IN_PROGRESS`
- `IN_PROGRESS → AT_RISK`
- `IN_PROGRESS → COMPLETED`
- `AT_RISK → IN_PROGRESS` (recovered)
- `AT_RISK → SLIPPED`
- `SLIPPED → IN_PROGRESS` (re-planned, new target)
- Any state → `ARCHIVED`

Transitions outside this set must be rejected by the backend.

> Source: PRD §11.5, REQ-PM-04

### REQ-PM-82: Slippage reason required for AT_RISK / SLIPPED

When a milestone transitions to `AT_RISK` or `SLIPPED`, the user must supply a slippage reason (min 10 chars). The reason is audited and surfaced on Project Space.

### REQ-PM-83: AI slippage prediction

The Planner must display, for each `IN_PROGRESS` milestone, an AI slippage-risk score (Low / Medium / High) and the contributing factors (e.g., "dependency blocked 4 days", "capacity under-allocated", "scope grew 18% in last 7 days"). The PM can accept or dismiss the prediction.

> Source: PRD §7.4, §11.5, §14

### REQ-PM-84: Link to Project Space

The Planner must offer a persistent "View in Project Space" entry that preserves `projectId` so the PM can cross-check the read-only operational view.

> Source: REQ-PS-42

---

## 11. Plan View — Capacity Allocation Requirements

### REQ-PM-90: Member × milestone matrix

The Capacity card must render members (rows) × milestones (columns), each cell is the member's % allocation to that milestone. Totals per member and per milestone are shown.

> Source: PRD §11.5 — "容量、资源"

### REQ-PM-91: Allocation edit

Cells must be editable inline (number input 0–100). Edits produce an audit event and update the Portfolio view on next refresh.

### REQ-PM-92: Over-allocation rejection

A row total exceeding 100% must be visually flagged but not hard-blocked (a PM may intentionally over-allocate short-term). The card must display a crimson warning and require a justification string to persist the state.

> Source: PRD §15.5, §16.2

### REQ-PM-93: Backup coverage check

For members marked as oncall or sole-owner of a milestone, the card must flag when no backup is assigned. The flag links to Platform Center Access Management for assignment.

> Source: PRD §11.12

### REQ-PM-94: AI rebalance suggestion

The card must surface, when applicable, an AI-generated capacity rebalance suggestion (e.g., "Shift 20% of Dev-A from Milestone-3 to Milestone-2 to reduce slippage risk"). The suggestion is applied only after PM accepts.

> Source: PRD §7.4

---

## 12. Plan View — Risk Registry Editor Requirements

### REQ-PM-100: Risk CRUD and workflow

The Risk Registry Editor must support:

- Create, edit, archive risks
- Transition through the risk state machine: `IDENTIFIED → ACKNOWLEDGED → MITIGATING → RESOLVED` (or → `ESCALATED` at any state)
- Assign owner, due date, mitigation note
- Link to an Incident, Approval, or mitigation task

> Source: PRD §11.5 — "风险"

### REQ-PM-101: Escalation path

`ESCALATED` transitions must notify the Application Owner and create a pending Approval. The approver list is derived from Platform Center role assignments.

> Source: PRD §11.12, §16.2

### REQ-PM-102: Severity ordering

Risks in the editor must be ordered by severity then age, matching Project Space ordering (REQ-PS-61).

### REQ-PM-103: AI mitigation draft

For each `ACKNOWLEDGED` risk, the AI Command Panel must offer a mitigation draft on demand. Accepted drafts become the `mitigation_note` field; audit captures the skill execution.

> Source: PRD §7.4, §14

---

## 13. Plan View — Dependency Resolver Requirements

### REQ-PM-110: Dependency list with resolution state

The Dependency Resolver must list upstream and downstream dependencies with a resolution state field: `PROPOSED / NEGOTIATING / APPROVED / REJECTED / AT_RISK / RESOLVED`.

> Source: PRD §11.5 — "依赖"

### REQ-PM-111: Approval workflow

Transitioning a dependency from `NEGOTIATING` to `APPROVED` requires a counter-signature from the target-side owner (if internal) or a logged contract commitment (if external). The backend must enforce this.

> Source: PRD §16.2

### REQ-PM-112: Deep-link to related Project Space

Internal dependencies must deep-link to the target's Project Space for counter-party review.

> Source: REQ-PS-50

### REQ-PM-113: AI dependency resolution proposal

The Resolver must surface, when applicable, an AI proposal to resolve a blocked dependency (e.g., "Route around via shared-event bus", "Defer milestone M3 by 5 days to align with upstream delivery"). Proposals are advisory and do not auto-apply.

> Source: PRD §7.4

---

## 14. Plan View — Delivery Progress Strip Requirements

### REQ-PM-120: 11-node SDLC chain health

The strip must render the 11-node SDLC main chain, scoped to the current Project, each node showing:

- Node label (Requirement → User Story → Spec → Architecture → Design → Tasks → Code → Test → Deploy → Incident → Learning)
- Throughput indicator (items flowed through in the last window)
- Health indicator (green / yellow / red)
- Deep-link into the corresponding lifecycle page

> Source: PRD §13.1, §11.5, §11.4–§11.10

### REQ-PM-121: Spec emphasized

Spec node must be visually emphasized as the execution hub.

> Source: PRD §7.6, §13.1

### REQ-PM-122: Slippage markers

Nodes whose throughput dropped more than a configurable threshold (default 30%) vs the prior comparable window must render a ▼ marker.

---

## 15. Plan View — Plan Change Log Requirements

### REQ-PM-130: Audit trail

The Plan Change Log must display, chronologically (most-recent first), all plan-affecting changes: milestone CRUD, capacity edits, risk state transitions, dependency approvals, AI recommendations accepted or dismissed.

> Source: PRD §16.2 — audit requirements

### REQ-PM-131: Change provenance

Each entry must show actor (user, or AI skill), action, target (milestone/risk/dep), before → after state, timestamp, and a link to the audit record.

### REQ-PM-132: Filterable

The log must support filtering by actor type (Human / AI), target type, and date range.

---

## 16. Cross-View Navigation & Context Requirements

### REQ-PM-140: Shared app shell

Project Management must render inside the shared app shell (REQ-SHELL-\* in shared-app-shell slice). It must respect the context bar (Workspace / Application / SNOW Group / Project).

> Source: PRD §15.2

### REQ-PM-141: Deep-linkable URLs

- `/project-management` — Portfolio view (scoped to current Workspace)
- `/project-management/:projectId` — Plan view for a specific project
- Query parameters preserve filters (e.g., `?riskSeverity=CRITICAL&ownerMemberId=m-42`)

### REQ-PM-142: Breadcrumb trail

Navigating from Portfolio → Plan → drill-in to Project Space / Requirement / Incident must register breadcrumbs: `Dashboard / Team Space (ws-...) / Project Management / Plan (proj-...) / Requirement Management`.

### REQ-PM-143: Entry points

Project Management must be reachable from:

- Shell primary navigation
- Dashboard slippage and risk cards (deep-link into Portfolio with filter applied)
- Team Space Project Distribution card (deep-link into Portfolio scoped to the Workspace's active projects)
- Project Space Milestone Execution Hub card (deep-link into Plan view for the project)

> Source: PRD §11.2, §11.3, §11.5

### REQ-PM-144: Plan context preservation on drill-out

When a PM drills from the Plan view into a linked page (Incident, Requirement, Deployment) and returns, the Plan view state (expanded sections, filters, scroll position) must be preserved.

---

## 17. AI Command Panel Requirements

### REQ-PM-150: Page context projection

Project Management must project its context to the AI Command Panel:

- Portfolio view: `workspaceId`, top-5 slipping milestones, top-5 over-allocated members, top-5 blocked dependencies
- Plan view: `projectId`, current-milestone id, top-3 risks, top-3 dependencies, capacity utilization

> Source: PRD §15.4

### REQ-PM-151: Suggested actions

The AI Command Panel must surface PM-appropriate suggestions:

Portfolio view:
- "Identify top 3 slippage hot-spots and likely causes"
- "Summarize capacity imbalance across the Workspace"
- "Which dependencies are most likely to block delivery in the next 2 weeks?"

Plan view:
- "Explain why Milestone M3 is predicted At-Risk"
- "Draft a mitigation plan for the top critical risk"
- "Suggest a capacity rebalance across the next milestone"
- "Summarize recent plan changes for stand-up"

### REQ-PM-152: Skill execution audit

All AI-assisted answers that reach Project Management must be recorded as Skill Executions with input / output summary, timestamp, evidence references, and whether the output was accepted, dismissed, or auto-applied.

> Source: PRD §14, §16.2

---

## 18. Governance & Audit Requirements

### REQ-PM-160: Governance on all mutations

Every mutation on Project Management (milestone, capacity, risk, dependency, AI accept) must pass through the platform governance pipeline with full audit capture. Project Management is not a back-door write channel.

> Source: PRD §16.2

### REQ-PM-161: Role-based write authority

Per the role model:

- Project Admin (incl. PM role): full milestone / capacity / risk / dependency CRUD for the project
- Project Contributor (Architect / Tech Lead / QA Lead / SRE): read; can comment and propose risks and dependencies; cannot directly move milestone status or approve dependencies
- Workspace Admin: all of the above across the Workspace; can escalate and approve at Workspace level
- Application Owner: read across the Application; approves escalations
- Auditor: read-only, including audit trail

Out-of-role writes must be rejected with a clear 403 and an audit entry.

> Source: PRD §16.1, §16.2

### REQ-PM-162: Autonomy level visibility

The Plan Header (REQ-PM-70) must show the project's current AI Autonomy Level. Changes to the level are made in Platform Center, not here, but the indicator must stay in sync within one refresh cycle.

> Source: PRD §11.10, §11.12

---

## 19. Isolation Requirements

### REQ-PM-170: Workspace scoping

Portfolio view data is strictly scoped to the current Workspace. Projects in other Workspaces must not appear.

> Source: PRD §9.2

### REQ-PM-171: Project scoping

Plan view data is strictly scoped to the single active Project. Cross-project reads in the Plan view (e.g., for a dependency target) must be via explicit linked-project drill-out, never via silent aggregation.

> Source: PRD §9.2

### REQ-PM-172: Parent-Workspace resolution

If the Plan-view Project's `workspaceId` does not match the context bar, the shell must auto-resolve Workspace selection before rendering.

> Source: REQ-PS-111

---

## 20. Visual & Experience Requirements

### REQ-PM-180: Card-based modular layout

Project Management must follow the shared design system's card / widget pattern; both views use the same grid primitives.

> Source: PRD §15.3

### REQ-PM-181: Tactical Command aesthetic

Visual density and enterprise-grade styling must follow "Tactical Command". Critical risks / blockers / over-allocation use the crimson accent. Health states use the standard green / yellow / red LED palette.

> Source: PRD §15.5, design.md

### REQ-PM-182: States

Both views must support: Normal / Loading (per-card skeletons) / Empty / Error (per-card, not page-wide) / Partial.

### REQ-PM-183: Per-card error isolation

A failure loading one card must not block the other cards. Each card manages its own loading / error / empty / ready state.

### REQ-PM-184: Inline edit affordance

All editable cells (milestone fields, capacity %, risk status, dep state) must expose an inline edit affordance consistent with the design system; a save action commits; optimistic UI updates with rollback on server error.

### REQ-PM-185: Unsaved-change protection

Navigating away from a Plan view with unsaved edits must prompt for confirmation.

---

## 21. Non-Functional Requirements

### REQ-PM-190: Frontend Phase A uses mocked data

Phase A renders with static or mocked Portfolio and Plan data. Backend integration arrives in Phase B.

### REQ-PM-191: Phase B backend API

Phase B provides real backend REST API endpoints for Portfolio aggregate, per-section reads, and Plan mutations.

### REQ-PM-192: Performance

- Portfolio view first-paint within 400ms of Workspace id receipt; individual cards hydrate within 800ms for up to 50 active projects.
- Plan view first-paint within 300ms of Project id receipt; cards hydrate within 500ms for typical project volumes (< 30 milestones, < 50 deps, < 100 risks).

### REQ-PM-193: Mutation latency

A single-field edit (milestone field, capacity cell, risk state transition) must round-trip within 500ms p95 on local dev and 1s p95 on staging.

### REQ-PM-194: Build must succeed

`npm run dev`, `npm run build`, and backend `./mvnw verify` must succeed after implementation.

### REQ-PM-195: Flyway migrations

All Project-Management-specific schema additions (capacity_allocation, risk_mitigation fields, dependency_resolution fields, plan_change_log, ai_suggestion) must land as Flyway migrations following `V{n}__{description}.sql`. No `ddl-auto: update`.

> Source: CLAUDE.md rule #4

### REQ-PM-196: Java & backend stack

Backend uses Java 21 and Spring Boot 3.x per project stack.

> Source: CLAUDE.md Lesson #2

---

## 22. Out of Scope (Explicit)

- Project creation / renaming / archiving (Platform Center)
- Role assignment / permission grants (Access Management)
- Workspace or Application configuration
- Requirement / Spec editing
- Design artifact editing
- Code / branch / merge / build
- Test case / execution editing
- Release gate, approval, rollout
- Incident detail editing
- Historical reporting / export
- Real-time WebSocket push (V1 uses on-load + manual refresh)
- Mobile-optimized layout (desktop-first)
- Gantt-style multi-project critical-path scheduling (V2)
- Time-tracking / timesheet ingestion
- Resource cost modeling / budget tracking (V2)

---

## 23. Traceability

| Requirement | PRD Section |
|-------------|-------------|
| REQ-PM-01 | §11.5 |
| REQ-PM-02 | §11.5 |
| REQ-PM-03 | §6.2, §6.3, §6.4, §11.5 |
| REQ-PM-04 | — (reuse rule; CLAUDE.md #3) |
| REQ-PM-05 | §7.4 |
| REQ-PM-06 | §7.8 |
| REQ-PM-07 | §7.6, §13.1 |
| REQ-PM-10 | §11.5 |
| REQ-PM-11 | §9.2, §15.2 |
| REQ-PM-12 | §7.4 |
| REQ-PM-20 | §11.5 |
| REQ-PM-21 | §15.5 |
| REQ-PM-22 | §11.5 |
| REQ-PM-23 | §7.8 |
| REQ-PM-30 | §11.5 |
| REQ-PM-31 | §11.5, §15.5 |
| REQ-PM-32 | §11.5 |
| REQ-PM-33 | §16.2 |
| REQ-PM-40 | §11.5 |
| REQ-PM-41 | §11.5 |
| REQ-PM-42 | §11.5 |
| REQ-PM-50 | §11.5 |
| REQ-PM-51 | §11.5 |
| REQ-PM-60 | §11.5 |
| REQ-PM-61 | §11.5 |
| REQ-PM-70 | §11.5, §11.12, §15.2 |
| REQ-PM-71 | §11.5 |
| REQ-PM-80 | §11.5 |
| REQ-PM-81 | §11.5 |
| REQ-PM-82 | §11.5, §16.2 |
| REQ-PM-83 | §7.4, §11.5, §14 |
| REQ-PM-84 | §11.3 |
| REQ-PM-90 | §11.5 |
| REQ-PM-91 | §16.2 |
| REQ-PM-92 | §15.5, §16.2 |
| REQ-PM-93 | §11.12 |
| REQ-PM-94 | §7.4 |
| REQ-PM-100 | §11.5 |
| REQ-PM-101 | §11.12, §16.2 |
| REQ-PM-102 | §11.3 |
| REQ-PM-103 | §7.4, §14 |
| REQ-PM-110 | §11.5 |
| REQ-PM-111 | §16.2 |
| REQ-PM-112 | §11.3 |
| REQ-PM-113 | §7.4 |
| REQ-PM-120 | §13.1, §11.5, §11.4–§11.10 |
| REQ-PM-121 | §7.6, §13.1 |
| REQ-PM-122 | §11.5 |
| REQ-PM-130 | §16.2 |
| REQ-PM-131 | §16.2 |
| REQ-PM-132 | §16.2 |
| REQ-PM-140 | §15.2 |
| REQ-PM-141 | — |
| REQ-PM-142 | — |
| REQ-PM-143 | §11.2, §11.3, §11.5 |
| REQ-PM-144 | §15.2 |
| REQ-PM-150 | §15.4 |
| REQ-PM-151 | §15.4 |
| REQ-PM-152 | §14, §16.2 |
| REQ-PM-160 | §16.2 |
| REQ-PM-161 | §16.1, §16.2 |
| REQ-PM-162 | §11.10, §11.12 |
| REQ-PM-170 | §9.2 |
| REQ-PM-171 | §9.2 |
| REQ-PM-172 | §15.2 |
| REQ-PM-180 | §15.3 |
| REQ-PM-181 | §15.5 |
| REQ-PM-182 | — |
| REQ-PM-183 | — |
| REQ-PM-184 | §15.3 |
| REQ-PM-185 | §15.3 |
| REQ-PM-190 | — |
| REQ-PM-191 | — |
| REQ-PM-192 | — |
| REQ-PM-193 | — |
| REQ-PM-194 | — |
| REQ-PM-195 | CLAUDE.md #4 |
| REQ-PM-196 | CLAUDE.md Lesson #2 |
