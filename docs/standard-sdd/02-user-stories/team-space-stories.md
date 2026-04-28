# Team Space Stories

## Scope

This document defines the user stories for the **Team Space** page — the Workspace-level operating home that sits between Dashboard (cross-team global) and Project Space (single-project execution) in the Agentic SDLC Control Tower.

## Traceability

- Requirements: [team-space-requirements.md](../01-requirements/team-space-requirements.md)
- PRD: [agentic_sdlc_control_tower_prd_v0.9.md](../01-requirements/agentic_sdlc_control_tower_prd_v0.9.md) §11.2, §9.3, §13.1, §15.2

---

## Story S1: View Workspace Summary

As a Team Lead,
I want to see a summary of the current Workspace (application, primary SNOW group, active projects, environments, health, owner),
so that I immediately know the identity and health of the team I am responsible for.

### Acceptance Criteria

- The Workspace Summary card appears at the top-left of Team Space
- The card shows: Workspace name, Application, Primary SNOW Group, active project count, active environment count, aggregate health (green/yellow/red), accountable owner
- If the Workspace is in compatibility mode (no Primary SNOW Group), the SNOW Group field renders as a neutral "Not configured" chip, not a blank
- Aggregate health LED uses the shared status LED palette
- The card renders inside the shared shell with the context bar showing the same Workspace

### Edge Notes

- V1 uses mocked Workspace data; Phase B fetches from `/api/v1/workspaces/:id/summary`
- Aggregate health is a derived field; computation lives in the backend service

### Requirement Refs

REQ-TS-10, REQ-TS-11, REQ-TS-12, REQ-TS-120, REQ-TS-121

---

## Story S2: Inspect Team Operating Model

As a Team Lead,
I want to see the currently-effective operating mode, approval mode, AI autonomy level, oncall strategy, and accountable owners,
so that I understand how my team is governed today and why.

### Acceptance Criteria

- The Team Operating Model card shows: Operating Mode, default approval mode, AI Autonomy Level, oncall owner, accountable owners for each governance area
- Each value displays a lineage badge showing where it was set: Platform Default / Application / SNOW Group / Workspace Override / Project Override
- Hovering the lineage badge reveals the override chain in a tooltip
- A "View in Platform Center" link is present for users with the required permission; absent otherwise
- Values marked as `Overridden` use the override badge styling consistent with the shared design system

### Edge Notes

- Lineage resolution is a read-only presentation of the platform configuration inheritance chain (§9.3)
- Team Space never allows editing these values

### Requirement Refs

REQ-TS-20, REQ-TS-21, REQ-TS-22, REQ-TS-06

---

## Story S3: Review Member & Role Matrix

As a Team Lead,
I want to see who is in my team, what role they hold, who is oncall, and whether we have coverage gaps,
so that I can identify risks in team composition before they become incidents.

### Acceptance Criteria

- The Member & Role Matrix card shows a list or matrix with: display name, role(s), oncall status (On / Off / Upcoming), key permissions summary, last-active date
- Roles are shown as chips; multiple roles per member are allowed
- Coverage gaps are surfaced as an explicit banner at the top of the card — e.g., "No primary oncall for 2026-04-19" or "Approver role unfilled"
- A "Manage in Access Management" link navigates into Platform Center for users with permission
- Member rows are sortable by last-active

### Edge Notes

- Team Space never grants permissions; it only surfaces the state
- In V1, oncall data may be populated from the Team Default Templates / Workflow config; if unavailable, the oncall column shows "Not configured"

### Requirement Refs

REQ-TS-30, REQ-TS-31, REQ-TS-32

---

## Story S4: Inspect Team Default Templates

As a Platform Reviewer or Team Lead,
I want to see the templates, policies, workflows, skill packs, and AI defaults currently effective for this Workspace,
so that I can assess governance posture and spot exception overrides.

### Acceptance Criteria

- The Team Default Templates card lists, at minimum, groups for: Page Templates, Policies, Workflows, Skill Packs, AI Defaults
- Each entry shows: name, version, lineage (inherited from Platform / Application / SNOW Group / Workspace), override status
- Inherited entries render with a neutral lineage chip; overridden entries use the override chip and link to the override detail
- A separate "Exception Overrides" section enumerates project-level overrides inside this Workspace
- A "View in Platform Center" link is present for authorized users

### Edge Notes

- Project-level overrides may not exist in a new Workspace — section renders empty-state with guidance, not error
- AI Defaults include model, autonomy level, logging policy

### Requirement Refs

REQ-TS-40, REQ-TS-41, REQ-TS-42, REQ-TS-06

---

## Story S5: Monitor Requirement & Spec Pipeline

As a Delivery Manager,
I want to see the team's Requirement → Story → Spec pipeline health at a glance with blockers highlighted,
so that I can intervene on stuck Specs before they delay delivery.

### Acceptance Criteria

- The Requirement & Spec Pipeline card shows counters for: Requirements inflow (last 7 days), Stories decomposing, Specs generating / in review / blocked, approved Specs awaiting downstream
- Blocker highlights section lists: Specs blocked > N days, Requirements without stories > N days, Stories without Spec > N days (N is configuration-driven)
- Clicking any counter or blocker item navigates to Requirement Management with the corresponding filter pre-applied
- The card visually renders a compressed 11-node SDLC main chain strip with Spec highlighted as the execution hub
- The main-chain strip shows workspace-scoped node health (green/yellow/red) per node

### Edge Notes

- Deep-link filters to Requirement Management use query parameters consistent with the Requirement slice contract
- If the team has < 5 requirements, blocker thresholds adjust down automatically (cold-start behavior)

### Requirement Refs

REQ-TS-50, REQ-TS-51, REQ-TS-52, REQ-TS-53, REQ-TS-134

---

## Story S6: Read Team Metrics

As a Team Lead or Application Owner,
I want to see delivery efficiency, quality, stability, governance maturity, and AI participation metrics scoped to this Workspace with trend indicators,
so that I can understand where we are strong and where we are drifting.

### Acceptance Criteria

- The Team Metrics card shows five metric groups: Delivery Efficiency, Quality, Stability, Governance Maturity, AI Participation
- Each metric displays: current value, previous-period comparison, trend indicator (up / down / flat)
- Trend indicators color-match the direction of improvement (green = improving, red = degrading; neutral for flat)
- Each metric has a "View history" link that deep-links into Report Center filtered by this Workspace and metric
- Tooltip on metric name explains computation

### Edge Notes

- V1 computes metrics from mocked / recent-window data; historical backfill belongs to Report Center
- Metric definitions are shared across Team Space and Dashboard (single source of truth)

### Requirement Refs

REQ-TS-60, REQ-TS-61, REQ-TS-62

---

## Story S7: Scan the Team Risk Radar

As a Team Lead,
I want a single card that surfaces the top risks across high-risk projects, dependency blocks, approval backlog, config drift, and incident hotspots,
so that I can triage what needs attention today.

### Acceptance Criteria

- The Team Risk Radar card groups risks by category: High-risk projects, Dependency blocks, Approval backlog, Config drift, Incident hotspots
- Risks are ordered by severity, not recency
- Critical-severity risks use the crimson accent
- Each risk item shows: title, category, severity, age, primary action (e.g., "Open Project Space", "View Incident", "Review Approval")
- Clicking the primary action navigates to the related surface with context preserved
- Empty state when no risks — renders a reassuring "All green" empty state, not an error

### Edge Notes

- Risk items are derived from lower-level signals (project health, approval queue, incident feed); computation happens backend-side
- AI-computed risks are tagged with a "Skill:" badge showing which skill produced the risk

### Requirement Refs

REQ-TS-70, REQ-TS-71, REQ-TS-72, REQ-TS-05

---

## Story S8: Browse Project Distribution

As a Team Lead,
I want to see the projects in this Workspace stratified by status with quick entry into any Project Space,
so that I can jump into the one that matters right now.

### Acceptance Criteria

- The Project Distribution card shows projects as compact cards or a grid
- Each project card displays: name, lifecycle stage, health indicator, primary risk (if any), active Spec count, open Incident count
- Projects are stratified into tabs or grouped sections: Healthy / At-Risk / Critical / Archived with counts per group
- Clicking a project card navigates to the Project Space of that project, preserving Workspace context in navigation state
- Keyboard navigation (arrow keys) between project cards is supported

### Edge Notes

- V1 Project Space may not yet exist as a full slice; Phase B Team Space still provides the navigation URL so Team Space does not block on Project Space delivery
- If Project Space is unimplemented, navigation fallback is a "Project details coming soon" page, not a 404

### Requirement Refs

REQ-TS-80, REQ-TS-81, REQ-TS-82

---

## Story S9: Switch Workspace Context

As a Team Lead who owns multiple Workspaces,
I want to switch Workspace context from the shared app shell and see Team Space reload with the new Workspace's data,
so that I can compare or manage multiple teams I am accountable for.

### Acceptance Criteria

- The shared shell's Workspace switcher is usable while on Team Space
- Switching Workspace updates the URL to `/team?workspaceId=:newWorkspaceId` and re-fetches all cards for the new Workspace
- No full page reload; state transitions are animated or use per-card skeletons
- Browser Back restores the previous Workspace's view
- If the user lacks access to the target Workspace, the switcher prevents selection with a clear message

### Edge Notes

- Re-fetch uses the same per-card fetch pattern; partial-failure isolation still applies
- A smooth transition avoids the appearance of a full navigation event

### Requirement Refs

REQ-TS-90, REQ-TS-91, REQ-TS-120

---

## Story S10: Handle Loading, Error, and Empty States

As any Team Space user,
I want loading, error, and empty states to render gracefully per card with retry options,
so that a partial failure does not lock me out of the rest of the page.

### Acceptance Criteria

- Each card shows its own skeleton while loading
- Each card shows its own error state with a retry affordance if its fetch fails
- Empty-state designs are distinct per card (e.g., "No projects yet", "No risks detected", "No oncall configured")
- A global page error (auth failure, missing Workspace access) renders a full-page message, not per-card errors
- Retry on a single card does not trigger a re-fetch on other cards

### Edge Notes

- Per-card isolation matches the Requirement and Dashboard slice patterns
- V1 retry is manual; automatic retry with backoff is V2

### Requirement Refs

REQ-TS-132, REQ-TS-133

---

## Story S11: Use the AI Command Panel for Team-Level Questions

As a Team Lead,
I want to ask the AI Command Panel Workspace-level questions ("Summarize this week's governance posture", "Why did Spec coverage drop?"),
so that I can get explanation without navigating through multiple pages.

### Acceptance Criteria

- When Team Space is active, the AI Command Panel receives Workspace context (workspaceId, visible risks, pipeline counters)
- The panel surfaces Team-Space-appropriate suggested prompts
- AI responses render with a Skill Execution record showing: skill name, inputs summary, output, timestamp, evidence links
- Evidence links navigate to the specific card or detail surface the answer is drawn from
- Users can pin an AI answer to the page for later reference (pin stored in UI state, not persisted in V1)

### Edge Notes

- AI responses are always tagged with the skill that produced them
- AI is a participant, not a chat widget — each answer must be auditable

### Requirement Refs

REQ-TS-100, REQ-TS-101, REQ-TS-102, REQ-TS-111

---

## Story S12: Navigate Back from Deep Links with Breadcrumb

As any Team Space user,
I want a breadcrumb trail that lets me return to Team Space after navigating to a Project Space, Requirement Management, or Incident detail,
so that I do not lose my place.

### Acceptance Criteria

- Navigating from Team Space to a downstream page registers the breadcrumb: `Dashboard / Team Space (Workspace X) / <downstream>`
- Clicking any crumb returns to that step with the original Workspace context preserved
- Browser Back also restores the prior Team Space view
- Deep-linked URLs (shared or bookmarked) that arrive without a breadcrumb history still render correctly — Team Space generates a default breadcrumb from URL segments

### Edge Notes

- Breadcrumb trail is stored in shell navigation state, not persisted

### Requirement Refs

REQ-TS-91, REQ-TS-92

---

## Summary

| # | Story | Primary Persona | Key Requirement Refs |
|---|-------|----------------|----------------------|
| S1 | View Workspace Summary | Team Lead | REQ-TS-10, 11, 12 |
| S2 | Inspect Team Operating Model | Team Lead | REQ-TS-20, 21, 22 |
| S3 | Review Member & Role Matrix | Team Lead | REQ-TS-30, 31, 32 |
| S4 | Inspect Team Default Templates | Platform Reviewer | REQ-TS-40, 41, 42 |
| S5 | Monitor Requirement & Spec Pipeline | Delivery Manager | REQ-TS-50, 51, 52, 53 |
| S6 | Read Team Metrics | Team Lead | REQ-TS-60, 61, 62 |
| S7 | Scan the Team Risk Radar | Team Lead | REQ-TS-70, 71, 72 |
| S8 | Browse Project Distribution | Team Lead | REQ-TS-80, 81, 82 |
| S9 | Switch Workspace Context | Multi-workspace Lead | REQ-TS-90, 91, 120 |
| S10 | Handle Loading / Error / Empty States | Any | REQ-TS-132, 133 |
| S11 | Use AI Command Panel for Team Questions | Team Lead | REQ-TS-100, 101, 102 |
| S12 | Breadcrumb Back Navigation | Any | REQ-TS-91, 92 |
