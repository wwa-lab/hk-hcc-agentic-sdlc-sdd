# Project Space Stories

## Scope

This document defines the user stories for the **Project Space** page — the single-Project execution home that sits between Team Space (Workspace-level operating home) and the lifecycle pages (Requirement / Design / Code / Test / Deploy / Incident) in the Agentic SDLC Control Tower.

## Traceability

- Requirements: [project-space-requirements.md](../01-requirements/project-space-requirements.md)
- PRD: [agentic_sdlc_control_tower_prd_v0.9.md](../01-requirements/agentic_sdlc_control_tower_prd_v0.9.md) §11.3, §13.1, §15.2
- Visual reference: [../05-design/Project Space.html](../05-design/Project%20Space.html)

---

## Story S1: View Project Summary Bar

As a Project Manager,
I want to see a summary of the current Project (name, parent Workspace, lifecycle stage, aggregate health, active milestone, owners, key counters),
so that I immediately understand the identity and health of the project I am responsible for.

### Acceptance Criteria

- The Project Summary Bar appears at the top of Project Space
- The bar shows: Project name + id, parent Workspace (with link back to Team Space), Application, lifecycle stage, aggregate health (green/yellow/red/unknown), PM, Tech Lead, active milestone label + target date, last update timestamp
- Compact counters show active Specs, open Incidents, pending Approvals, active Critical+High risks
- Aggregate health LED uses the shared status LED palette
- Hovering the health LED reveals the contributing factors (risks, incidents, slippage)

### Edge Notes

- V1 uses mocked Project data; Phase B fetches from `/api/v1/project-space/:id/summary`
- Aggregate health is derived server-side
- Last update timestamp reflects the most recent projection refresh

### Requirement Refs

REQ-PS-10, REQ-PS-11, REQ-PS-12, REQ-PS-110, REQ-PS-111

---

## Story S2: Inspect Leadership & Ownership

As a Project Manager or Team Lead,
I want to see who owns this project across PM / Architect / Tech Lead / QA / SRE / AI Adoption,
so that I know who to escalate to and whether we have coverage gaps.

### Acceptance Criteria

- The Leadership & Ownership card shows one row per accountable role
- Each row shows: role label, person name + id, oncall status (On / Off / Upcoming), backup presence indicator
- Roles without a named backup are flagged with a "no backup" chip
- A "Manage in Access Management" link is present for users with the required permission; absent otherwise
- Empty states are explicit (e.g., "AI Adoption owner not assigned") rather than blank rows

### Edge Notes

- Role assignments come from Access Management; Project Space is read-only
- Oncall resolution uses the existing oncall schedule service

### Requirement Refs

REQ-PS-20, REQ-PS-21, REQ-PS-22

---

## Story S3: Navigate the Project SDLC Chain

As any team member,
I want to see the 11-node SDLC main chain scoped to this Project with live counts/health and one-click entries,
so that I can jump straight into the Requirement / Spec / Design / Code / Test / Deploy / Incident view filtered by this Project.

### Acceptance Criteria

- The SDLC Deep Links card renders all 11 nodes in canonical order (Requirement → Story → Spec → Architecture → Design → Tasks → Code → Test → Deploy → Incident → Learning)
- Each node shows a compact count or health indicator
- The Spec node is visibly emphasized
- Clicking any node opens the corresponding lifecycle page pre-filtered by `projectId`
- Nodes targeting lifecycle pages that do not yet exist render in a disabled/"coming soon" state, not a broken link

### Edge Notes

- Filter vocabulary aligns with each target page's query params
- Navigation carries both `projectId` and `workspaceId` so upstream breadcrumb is coherent

### Requirement Refs

REQ-PS-30, REQ-PS-31, REQ-PS-32

---

## Story S4: Track the Milestone Execution Hub

As a Project Manager,
I want to see the project's milestone timeline with status, completion %, owner, and slippage indicator,
so that I can see where we stand and what is at risk.

### Acceptance Criteria

- The Milestone Execution Hub card renders milestones in chronological order
- Each milestone shows: label, target date, status (Not-Started / In-Progress / At-Risk / Completed / Slipped), % complete (when computable), owner, slippage reason (if At-Risk / Slipped)
- The current milestone is marked prominently
- At-Risk / Slipped milestones use the crimson accent per design system
- A "Manage in Project Management" link navigates out for editing

### Edge Notes

- Milestone data is read-only in Project Space
- When no milestones are defined, the card shows an "No milestones yet" empty state with a create-in-Project-Management CTA

### Requirement Refs

REQ-PS-40, REQ-PS-41, REQ-PS-42

---

## Story S5: Read the Operational Dependency Map

As an Architect or Tech Lead,
I want to see upstream dependencies and downstream dependents of this project with health and any active blocker,
so that I can identify integration risks before they become incidents.

### Acceptance Criteria

- The Operational Dependency Map card separates Upstream and Downstream lists
- Each entry shows: name, relationship type (API / Data / Schedule / SLA), owner team, current health (green/yellow/red), blocker reason (if any)
- Entries currently blocking delivery render with crimson accent
- Each entry offers a primary action: open the dependency's Project Space (if registered) or open a related Incident

### Edge Notes

- Dependencies that are external (not a registered project) show an "External" badge and cannot be navigated to a Project Space
- Health defaults to `unknown` when the dependency's own health service is unreachable

### Requirement Refs

REQ-PS-50, REQ-PS-51, REQ-PS-52

---

## Story S6: Scan the Risk & Vulnerability Registry

As any stakeholder,
I want to see a list of active project-scoped risks ordered by severity with owner, age, and action entry,
so that I can focus on what actually matters.

### Acceptance Criteria

- The Risk & Vulnerability Registry card lists active risks ordered by severity (Critical → High → Medium → Low), then by age desc
- Each row shows: title, severity chip, category, owner, age in days, latest mitigation note, primary action
- Critical risks use the crimson accent
- Empty state ("All green") appears when no active risks exist

### Edge Notes

- Risk age counts days since detection, not days since last update
- Primary action resolves per risk: open Incident, open Approval, or open mitigation task

### Requirement Refs

REQ-PS-60, REQ-PS-61, REQ-PS-62

---

## Story S7: Read the Environment & Version Matrix

As an SRE / Deploy Lead,
I want to see the DEV / STAGING / PROD environments with their deployed version, last deploy time, health, and gate status,
so that I can see where we are in the release pipeline and whether there is dangerous drift.

### Acceptance Criteria

- The Environment & Version Matrix card renders one tile per environment
- Each tile shows: env label, version / build id, last-deploy timestamp, health (green/yellow/red), gate status (Auto / Approval Required / Blocked), approver (if pending)
- If PROD and STAGING versions diverge beyond threshold, a "drift" indicator is shown with the commit delta
- Each tile offers a "View in Deployment Management" deep link

### Edge Notes

- Gate status mapping: Auto → green accent, Approval Required → amber, Blocked → crimson
- Environment list is inventory-driven; custom environments render alongside DEV/STAGING/PROD

### Requirement Refs

REQ-PS-70, REQ-PS-71, REQ-PS-72

---

## Story S8: Switch Project Context

As any user,
I want to switch the active Project via the shared shell context or by navigating directly to another Project Space URL and have Project Space re-fetch,
so that I can move between projects I own without a full page reload.

### Acceptance Criteria

- Selecting a new Project updates the route to `/project-space/:projectId` (optionally preserving `workspaceId` in the query string)
- Project Space clears card state to loading, then rehydrates per-card
- If the new Project belongs to a different Workspace, the shell updates Workspace / Application automatically
- URL is deep-linkable and shareable

### Edge Notes

- Switching to a Project with access denied renders a page-level error, not card-level errors
- Switching preserves scroll position per card category

### Requirement Refs

REQ-PS-80, REQ-PS-81, REQ-PS-111

---

## Story S9: Handle Loading, Error, Empty States

As any user,
I want each card to manage its own skeleton / error / empty state with retry,
so that a single card failure does not break the whole page.

### Acceptance Criteria

- Each card shows a skeleton shimmer while loading
- Failed cards show an error banner with a retry button; other cards remain usable
- Empty cards show a distinct, informative empty message (not a skeleton forever)
- Auth / access denial renders as a page-level error with a link back to Team Space

### Requirement Refs

REQ-PS-122, REQ-PS-123

---

## Story S10: Use AI Command Panel for Project Questions

As any user,
I want the AI Command Panel to know what Project I am on and offer project-scoped prompts,
so that I can ask "why is milestone M3 at risk" without repeating context.

### Acceptance Criteria

- On mount and on Project switch, Project Space pushes `ProjectSpaceContext { projectId, workspaceId, activeMilestone, topRisks, envPosture }` to the AI Command Panel
- Panel displays at least three project-appropriate suggested prompts
- Skill executions initiated from the panel are recorded via the existing skill-execution audit path

### Requirement Refs

REQ-PS-90, REQ-PS-91, REQ-PS-92, REQ-PS-101

---

## Story S11: Breadcrumb Back Navigation

As any user,
I want navigating from Project Space into a lifecycle page to register a breadcrumb so I can return with one click,
so that I don't lose context when drilling down.

### Acceptance Criteria

- Navigating from Project Space to Requirement / Design / Code / Test / Deploy / Incident registers a breadcrumb entry
- Browser Back returns to Project Space at the same scroll position
- Breadcrumb label reflects the originating Project

### Requirement Refs

REQ-PS-82

---

## Story S12: Governed Navigation Auditing

As a Platform governance reviewer,
I want every AI skill execution and access-denial on Project Space to be audited,
so that we maintain the audit baseline expected of AI-native systems.

### Acceptance Criteria

- Skill executions initiated on Project Space are logged with: who / when / which skill / input / output / evidence
- Access denial events (e.g., cross-project leakage attempt) emit an audit event
- No governance action bypasses the platform governance pipeline

### Requirement Refs

REQ-PS-100, REQ-PS-101, REQ-PS-110
