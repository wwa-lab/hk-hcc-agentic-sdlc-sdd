# Project Space Requirements

## Purpose

This document extracts the requirements relevant to the **Project Space** page from the full PRD (V0.9). It defines **what** the Project Space page must deliver, serving as the upstream input for the spec, architecture, design, and task documents of this slice.

Project Space is the **single-project execution home** — the contextual layer between Team Space (Workspace-level operating home) and the lifecycle management pages (Requirement / Design / Code & Build / Testing / Deployment / Incident). It answers "what is the state of this project, who owns it, where is it going next, and where are the risks?" for a single Project.

## Source

All requirements below are derived from:

- [agentic_sdlc_control_tower_prd_v0.9.md](agentic_sdlc_control_tower_prd_v0.9.md)
- Visual reference: [../05-design/Project Space.html](../05-design/Project%20Space.html)

Section references use the format `PRD §N.N`.

---

## 1. Scope

The Project Space page is described in the PRD as the single-project operating home — the primary bridge between organizational view (Workspace) and execution view (SDLC lifecycle pages) (PRD §11.3).

It covers:

- Project Summary Bar (project identity, lifecycle stage, aggregate health, key counters, last update)
- Leadership & Ownership card (PM, Architect, Tech Lead, QA Lead, SRE, accountable owners with oncall duty)
- SDLC Deep Links card (per-project 11-node main chain with per-node status and deep-link entry into each lifecycle page)
- Milestone Execution Hub card (milestones, target dates, completion, current milestone, slippage indicator)
- Operational Dependency Map card (upstream/downstream project dependencies, blocking edges, SLA commitments)
- Risk & Vulnerability Registry card (risks by severity, owner, age, action)
- Environment & Version Matrix card (DEV / STAGING / PROD environments with version, health, last deploy, window)
- Related SDLC Chain summary (project-scoped link vocabulary into Requirement / Spec / Design / Code / Test / Deploy / Incident)
- Project context switcher integration with the shared context bar
- AI Command Panel integration with project-scoped suggestions and skill executions

It does **not** cover:

- Workspace-level aggregation (Team Space owns that)
- Cross-project or cross-workspace aggregation (Dashboard owns that)
- Requirement / Spec content editing (Requirement Management owns editing)
- Architecture / Design artifact editing (Design Management owns editing)
- Code, Branch, Merge, Build detail (Code & Build Management owns that)
- Test case / execution / coverage detail (Testing Management owns that)
- Release gate / approval / rollout detail (Deployment Management owns that)
- Incident triage / execution detail (Incident Management owns that)
- Project creation, archiving, renaming (Project Management owns lifecycle admin)
- Historical reporting / export (Report Center)

---

## 2. Product Context Requirements

### REQ-PS-01: Single-Project operating home

Project Space is the operating home for **exactly one Project at a time**. It is not a cross-project aggregator. All data, metrics, counts, and navigation points are scoped to the currently-selected Project.

> Source: PRD §11.3 — "用于展示单项目级总览"

### REQ-PS-02: Context bridge between Team Space and lifecycle pages

Project Space is the contextual bridge between Team Space (Workspace-level operating home) and the execution pages (Requirement / Design / Code / Test / Deploy / Incident). Navigation between these pages must preserve Project context (and Workspace context upstream).

> Source: PRD §11.3 — "从组织视角切换到执行视角的主要桥梁"

### REQ-PS-03: Four audience layers

The page must serve:

- **Project Manager / Delivery Lead**: milestone progression, slippage risk, dependency blocks, approval backlog
- **Architect / Tech Lead**: spec coverage, design artifacts, dependency shape, environment posture
- **Team members (Dev / QA / SRE)**: what to work on next, environment state, open risks
- **Application Owner / Team Lead (drilling from Team Space)**: execution health, current milestone, top risks

> Source: PRD §6.2, §6.3, §6.5, §11.3

### REQ-PS-04: Configuration-driven layout

Card order, card visibility, and card-level default filters must be configuration-driven so a Project can tune its Project Space without code changes.

> Source: PRD §7.8 — "配置驱动"

### REQ-PS-05: AI is a participant, not a widget

Each Project Space card that surfaces an AI decision or signal (risk, dependency, milestone slippage prediction) must explicitly identify AI involvement — what the AI did, what it recommends, and whether human governance is required.

> Source: PRD §7.4 — "AI 是参与者，不是旁观者"

### REQ-PS-06: Read-only surface for Project inventory data

Project Space must display inherited Workspace / Application configuration, milestone plan, environment inventory, and dependency wiring as **read-only**. Edits happen in Project Management, Platform Center, or Deployment Management — not Project Space.

> Source: PRD §11.5, §11.12

---

## 3. Project Summary Bar Requirements

### REQ-PS-10: Project identity

The Project Summary Bar must display:

- Project name and identifier
- Parent Workspace name and identifier (with link back to Team Space)
- Parent Application name
- Lifecycle stage (Discovery / Delivery / Steady-State / Retiring)
- Aggregate health indicator (green / yellow / red / unknown)
- Accountable Project Manager and Tech Lead
- Active milestone short-label and target date
- Last update timestamp

> Source: PRD §11.3

### REQ-PS-11: Key execution counters

The Summary Bar must display compact counters:

- Active Specs count
- Open Incidents count
- Pending Approvals count
- Active Risks count (Critical + High only)

### REQ-PS-12: Aggregate health disclosure

The aggregate health indicator must be derived server-side from risks, open incidents, approval backlog, and milestone slippage. Hover / click reveals the contributing factors.

---

## 4. Leadership & Ownership Card Requirements

### REQ-PS-20: Role coverage

The Leadership & Ownership card must display, for this Project:

- **Project Manager** (accountable delivery owner)
- **Architect** (accountable architecture owner)
- **Tech Lead** (accountable engineering owner)
- **QA Lead** (accountable quality owner)
- **SRE / Operations owner** (accountable runtime owner)
- **AI Adoption owner** (if configured) — accountable for AI skill-pack configuration

> Source: PRD §11.3, visual reference mockup §2

### REQ-PS-21: Oncall duty and backup

Each role entry must show current oncall status (On / Off / Upcoming) and whether a named backup exists. Roles without a backup must display a "no backup" indicator.

### REQ-PS-22: Link to Access Management

The card must provide a navigable entry into Access Management (inside Platform Center) for role assignment changes — Project Space is read-only for role assignment.

> Source: PRD §11.12

---

## 5. SDLC Deep Links Card Requirements

### REQ-PS-30: 11-node main-chain deep links

The card must render the 11-node SDLC main chain with a compact per-node tile containing:

- Node label (Requirement / Story / Spec / Architecture / Design / Tasks / Code / Test / Deploy / Incident / Learning)
- Per-node count or health indicator (e.g., open count, coverage %, last activity)
- Navigable entry into the corresponding lifecycle page, scoped to the current Project

> Source: PRD §13.1, §11.3, §11.4–§11.10

### REQ-PS-31: Spec highlighted as execution hub

The Spec node must be visibly emphasized consistent with the rest of the product's Spec-centric vocabulary.

> Source: PRD §7.6, §13.1

### REQ-PS-32: Project-scoped query vocabulary

Each deep link must carry `projectId` (and `workspaceId`) query parameters so the target page opens pre-filtered by Project.

---

## 6. Milestone Execution Hub Card Requirements

### REQ-PS-40: Milestone timeline

The card must display the project's milestones as a timeline, each showing:

- Milestone label and target date
- Status (Not-Started / In-Progress / At-Risk / Completed / Slipped)
- Current milestone marker
- Completion percentage (if computable)
- Owner (if assigned)

> Source: PRD §11.3

### REQ-PS-41: Slippage indicator

Any milestone currently in `At-Risk` or `Slipped` status must render with a warning/crimson accent consistent with the design system, and show the reason (short string).

### REQ-PS-42: Link to Project Management

The card must provide a navigable entry into Project Management for milestone editing. Project Space itself is read-only for milestones.

> Source: PRD §11.5

---

## 7. Operational Dependency Map Card Requirements

### REQ-PS-50: Dependency list

The card must list:

- **Upstream dependencies** (services / projects this project depends on)
- **Downstream dependents** (services / projects that depend on this project)

Each entry shows name, relationship type (API / Data / Schedule / SLA), owner team, current health, and any active blocker.

> Source: PRD §11.3 — "依赖"

### REQ-PS-51: Blocking highlights

Dependencies that are currently blocking delivery (i.e., a dependency is down or a contract is unresolved) must render with a crimson accent and an explicit blocker reason.

### REQ-PS-52: Action entry points

Each dependency row must offer at least one action entry point — open the dependency's Project Space (if it is a registered project) or open the related Incident.

---

## 8. Risk & Vulnerability Registry Card Requirements

### REQ-PS-60: Risk list

The card must list active project-scoped risks, each showing:

- Risk title
- Severity (Critical / High / Medium / Low)
- Category (Technical / Security / Delivery / Dependency / Governance)
- Owner (if assigned)
- Age (days since detected)
- Latest status / mitigation note
- Action entry point (open Incident, open Approval, open mitigation task)

> Source: PRD §11.3 — "风险"

### REQ-PS-61: Severity ordering

Risks must be ordered by severity, not recency. Critical risks use the crimson accent.

### REQ-PS-62: Empty state

If no active risks exist, the card must display an "All green" empty state rather than a spinner or blank.

---

## 9. Environment & Version Matrix Card Requirements

### REQ-PS-70: Environment inventory

The card must list, per environment (DEV / STAGING / PROD and any custom environments):

- Environment label and identifier
- Deployed version / build id
- Deploy window (e.g., "last deployed 2026-04-16T08:00Z")
- Health indicator (Green / Yellow / Red)
- Gate status (Auto / Approval Required / Blocked)
- Owner / approver (if pending approval)

> Source: PRD §11.3 — "版本、环境"

### REQ-PS-71: Version drift highlight

If PROD and STAGING versions diverge beyond a threshold (e.g., > N commits behind), the card must surface the drift explicitly.

### REQ-PS-72: Deep link to Deployment Management

Each environment tile must offer a navigable entry into Deployment Management scoped to the environment — Project Space does not own deploy actions.

> Source: PRD §11.9

---

## 10. Navigation & Context Requirements

### REQ-PS-80: Shell context bar integration

Project Space must render inside the shared app shell, resolve the current Project from the active route while staying aligned with parent Workspace / Application context, and re-fetch data when the active Project changes without a full page reload.

> Source: PRD §15.2

### REQ-PS-81: Deep-linkable URLs

All card navigation points must produce deep-linkable URLs (e.g., `/project-space/proj-8821` and onward to `/requirements?projectId=...&workspaceId=...`) so users can share and bookmark Project-scoped views.

### REQ-PS-82: Breadcrumb and back navigation

Navigating from Project Space to any lifecycle page must register a breadcrumb trail so users can return with one click. Typical trail: `Dashboard / Team Space (ws-...) / Project Space (proj-...) / Requirement Management`.

### REQ-PS-83: Entry from Team Space

Project Space must be reachable from Team Space's Project Distribution card via `/project-space/:id` (optionally preserving `workspaceId` in the query string when available). Workspace context is preserved.

> Source: PRD §11.2, §11.3

---

## 11. AI Command Panel Requirements

### REQ-PS-90: Page context projection

Project Space must project its context (current projectId, workspaceId, current milestone, top risks, environment posture) to the right-side AI Command Panel.

> Source: PRD §15.4

### REQ-PS-91: Suggested actions

The AI Command Panel must surface Project-Space-appropriate suggestions, e.g.:

- "Summarize why milestone M3 is at risk"
- "Identify the top 3 dependency blockers"
- "Explain why STAGING lags PROD by 12 commits"
- "Draft a risk mitigation plan for the top critical risk"

### REQ-PS-92: Skill execution records

All AI-assisted answers rendered on Project Space cards must be recorded as Skill Executions with input/output summary, timestamp, evidence references.

> Source: PRD §14

---

## 12. Governance & Audit Requirements

### REQ-PS-100: Read-only governance surface

All governance actions initiated from Project Space (approvals, escalations) must pass through the platform governance pipeline with full audit capture. Project Space is not a back-door governance channel.

> Source: PRD §16.2

### REQ-PS-101: Audit trail on navigation-triggered skill executions

Any AI skill executed via Project Space navigation (e.g., "summarize milestone risk") must record who / when / which skill / input / output / evidence.

> Source: PRD §14, §16.2

---

## 13. Project Isolation Requirements

### REQ-PS-110: Strict Project scoping

Project Space must hard-scope all data, metrics, counts, and navigation to the currently-active Project. Cross-project leakage is a defect.

> Source: PRD §9.2

### REQ-PS-111: Parent-Workspace consistency

If the current Project's `workspaceId` does not match the context bar's current `workspaceId`, the shell must resolve the conflict by re-selecting the Project's Workspace (and Application) automatically before rendering.

> Source: PRD §15.2

---

## 14. Visual and Experience Requirements

### REQ-PS-120: Card-based modular layout

Project Space must follow the shared design system's Widget/Card pattern, reusing existing shell primitives.

> Source: PRD §15.3

### REQ-PS-121: Tactical Command aesthetic

Visual density and enterprise-grade styling must follow the "Tactical Command" aesthetic. Critical risks / blockers use the crimson accent. Health states use the standard green / yellow / red LED palette.

> Source: PRD §15.5, design.md §2

### REQ-PS-122: States

The page must support:

- Normal (Project loaded, cards populated)
- Loading (per-card skeletons)
- Empty (new Project with no milestones / no environments / no risks)
- Error (API failure per card — isolated, not page-wide)
- Partial (some cards loaded, others failed — per-card retry)

### REQ-PS-123: Per-card error isolation

A failure loading one card must not block the other cards. Each card manages its own loading / error / empty / ready state.

> Source: consistency with `dashboard`, `requirement`, and `team-space` slice standards

### REQ-PS-124: SDLC chain strip emphasis

The 11-node SDLC chain strip on Project Space must visibly emphasize Spec as the execution hub.

> Source: PRD §13.1, §7.6

---

## 15. Non-Functional Requirements

### REQ-PS-130: Frontend Phase A uses mocked data

Phase A (frontend) must render with static or mocked Project data. Backend integration arrives in Phase B.

### REQ-PS-131: Phase B backend API

Phase B provides real backend REST API endpoints for the Project Space aggregate view, per-card slices, and navigation deep-link resolution.

### REQ-PS-132: Performance

Project Space must first-paint within 300ms of receiving the Project id. Individual cards must hydrate within 500ms under typical Project volumes (< 30 milestones, < 50 dependencies, < 100 active risks, < 10 environments).

### REQ-PS-133: Build must succeed

`npm run dev`, `npm run build`, and backend `./mvnw verify` must all succeed after implementation.

### REQ-PS-134: Flyway migrations

Any Project Space-specific schema addition (dependency edges, milestone cache, version-drift projection) must land as a Flyway migration following the `V{n}__{description}.sql` convention. No `ddl-auto: update`.

> Source: CLAUDE.md rule #4

---

## 16. Out of Scope

- Milestone editing (Project Management owns editing)
- Environment creation / decommission (Deployment Management / Platform Center)
- Role / permission grants (Access Management)
- Requirement / Spec content editing (Requirement Management)
- Design artifact editing (Design Management)
- Code / branch / merge operations (Code & Build Management)
- Test case / execution detail (Testing Management)
- Release approvals (Deployment Management)
- Incident detail editing (Incident Management)
- Historical reporting / export (Report Center)
- Cross-project aggregation (Team Space / Dashboard)
- Real-time WebSocket push for deployment / milestone state (V1 uses on-load + refresh button)
- Mobile-optimized layout (desktop-first per PRD §1)

---

## 17. Traceability

| Requirement | PRD Section |
|-------------|-------------|
| REQ-PS-01 | §11.3 |
| REQ-PS-02 | §11.3 |
| REQ-PS-03 | §6.2, §6.3, §6.5, §11.3 |
| REQ-PS-04 | §7.8 |
| REQ-PS-05 | §7.4 |
| REQ-PS-06 | §11.5, §11.12 |
| REQ-PS-10 | §11.3 |
| REQ-PS-11 | §11.3 |
| REQ-PS-12 | §11.3 |
| REQ-PS-20 | §11.3 |
| REQ-PS-21 | §11.3 |
| REQ-PS-22 | §11.12 |
| REQ-PS-30 | §13.1, §11.3 |
| REQ-PS-31 | §7.6, §13.1 |
| REQ-PS-32 | — |
| REQ-PS-40 | §11.3 |
| REQ-PS-41 | §15.5 |
| REQ-PS-42 | §11.5 |
| REQ-PS-50 | §11.3 |
| REQ-PS-51 | §15.5 |
| REQ-PS-52 | — |
| REQ-PS-60 | §11.3 |
| REQ-PS-61 | §15.5 |
| REQ-PS-62 | — |
| REQ-PS-70 | §11.3 |
| REQ-PS-71 | §11.3, §11.9 |
| REQ-PS-72 | §11.9 |
| REQ-PS-80 | §15.2 |
| REQ-PS-81 | — |
| REQ-PS-82 | — |
| REQ-PS-83 | §11.2, §11.3 |
| REQ-PS-90 | §15.4 |
| REQ-PS-91 | §15.4 |
| REQ-PS-92 | §14 |
| REQ-PS-100 | §16.2 |
| REQ-PS-101 | §14, §16.2 |
| REQ-PS-110 | §9.2 |
| REQ-PS-111 | §15.2 |
| REQ-PS-120 | §15.3 |
| REQ-PS-121 | §15.5 |
| REQ-PS-122 | — |
| REQ-PS-123 | — |
| REQ-PS-124 | §13.1, §7.6 |
| REQ-PS-130 | — |
| REQ-PS-131 | — |
| REQ-PS-132 | — |
| REQ-PS-133 | — |
| REQ-PS-134 | CLAUDE.md #4 |
