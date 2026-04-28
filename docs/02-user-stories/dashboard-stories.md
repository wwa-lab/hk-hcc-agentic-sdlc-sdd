# Dashboard / Control Tower Stories

## Scope

This document defines the user stories for the Dashboard / Control Tower page,
the main landing page of the Agentic SDLC Control Tower product.

## Story S1: See End-to-End SDLC Chain Health

As a team lead or delivery manager
I want to see the full 11-node SDLC chain with per-stage health indicators
So that I can identify bottleneck stages and understand where work is flowing

### Acceptance Criteria

- The dashboard displays all 11 SDLC stages: Requirement, User Story, Spec, Architecture, Design, Tasks, Code, Test, Deploy, Incident, Learning
- Each stage shows a health status: healthy, warning, critical, or inactive
- Spec is visually emphasized as the execution hub (distinct styling, larger node, or highlight)
- The chain tells a flow story — users can see where items concentrate and where blockages exist
- Stage nodes link to the corresponding module page (e.g., clicking "Code" navigates to Code & Build)

### Edge Notes

- V1 uses mocked health data; real aggregation comes in Phase B
- Folded/collapsed display is not required for the dashboard (full 11 nodes)

## Story S2: Review Delivery Efficiency Metrics

As a delivery manager
I want to see key delivery metrics at a glance
So that I can assess team velocity and identify delivery slowdowns

### Acceptance Criteria

- Dashboard shows: Lead Time, Deployment Frequency, Iteration Completion Rate
- Each metric displays a current value and a trend indicator (up/down/stable)
- Bottleneck stage is highlighted when delivery slows
- Metrics are scoped to the current workspace context

### Edge Notes

- Trend data requires at least two data points; V1 mocked data includes sample trends

## Story S3: Understand AI Participation Across Stages

As a platform admin or management viewer
I want to see how AI participates across SDLC stages
So that I can demonstrate AI value and justify platform investment

### Acceptance Criteria

- Dashboard shows AI involvement per SDLC stage (visual indicator per stage)
- Key AI metrics visible: AI Usage Rate, Suggestion Adoption Rate, Auto-Execution Success Rate, AI Time Saved
- AI is presented as an active participant, not a passive advisor
- The AI participation section clearly answers: "What has AI done, and what impact did it have?"

### Edge Notes

- V1 uses mocked AI participation data
- AI Center provides deeper drill-down; dashboard shows summary only

## Story S4: Monitor Quality and Stability Signals

As a QA lead or SRE
I want to see quality and stability indicators
So that I can assess release readiness and operational health

### Acceptance Criteria

- Quality metrics: Build Success Rate, Test Pass Rate, Defect Density, Spec Coverage
- Stability metrics: Active Incidents (with severity), Change Failure Rate, MTTR, Rollback Rate
- Critical values are visually flagged (color or icon)
- Clicking an incident indicator navigates to Incident Management

### Edge Notes

- Quality and stability may be displayed as separate cards or combined
- V1 uses mocked data

## Story S5: See Recent Key Actions

As any user
I want to see a stream of recent key actions across the workspace
So that I know what happened since my last visit

### Acceptance Criteria

- Dashboard shows a chronological activity stream (most recent first)
- Each entry shows: actor (AI or human name), action description, SDLC stage, timestamp
- AI actions are visually distinguished from human actions (icon or label)
- Stream shows at most 10 recent entries with a "View All" link
- Activity is scoped to the current workspace

### Edge Notes

- V1 uses mocked activity data
- Full activity history belongs to Audit Management in Platform Center

## Story S6: Read the Value Proof Narrative

As a management viewer or visitor
I want to see a value story that demonstrates AI-native SDLC impact
So that I can understand the platform's tangible benefits

### Acceptance Criteria

- Dashboard includes a value narrative section with headline metrics: time saved, quality improved, incidents prevented
- The narrative uses real (or mocked) evidence, not generic marketing copy
- The section integrates naturally with operational metrics rather than being a separate promotional block

### Edge Notes

- V1 value narrative uses mocked data
- This section should evolve into a dynamic, data-driven story in future versions

## Story S7: Check Governance Trust Indicators

As a platform admin
I want to see governance health indicators
So that I can confirm policy compliance and audit readiness

### Acceptance Criteria

- Dashboard shows: Template Reuse Rate, Configuration Drift status, Audit Coverage, Policy Hit Rate
- Non-compliant indicators are flagged visually
- Clicking a governance indicator navigates to Platform Center

### Edge Notes

- V1 uses mocked governance data
- Governance detail belongs to Platform Center; dashboard shows summary only

## Story S8: Navigate from Dashboard to Detail Pages

As any user
I want to click through from dashboard signals to the relevant detail page
So that I can investigate further without losing context

### Acceptance Criteria

- SDLC chain nodes link to corresponding module pages
- Incident indicators link to Incident Management
- Governance indicators link to Platform Center
- Navigation preserves the current workspace context
- Shell remains stable during navigation

### Edge Notes

- "Coming Soon" pages show a placeholder when the target module is not yet implemented

## Assumptions

- Dashboard renders within the shared app shell with context bar, nav, and AI panel
- Desktop-first; mobile layout is out of scope
- Phase A uses mocked data; Phase B connects to backend APIs
- All dashboard widgets are visible to all users; role-based visibility is deferred
- Dashboard data is scoped to the current workspace

## Dependencies

- Shared app shell must be implemented (Slice 1 — done)
- Router, Pinia store, and API client infrastructure exist (Slice 1 — done)
- WorkspaceContext is available from workspace store
- Design HTML prototype ([Control Tower.html](../05-design/Control%20Tower.html)) provides visual reference

## Open Questions (Resolved)

- ~~Should the SDLC chain be a horizontal pipeline or a more creative visualization?~~ **Resolved:** Horizontal pipeline with arrow connectors (see spec §3.2, design §7.4).
- ~~Should delivery metrics show sparkline charts or just numbers with trend arrows?~~ **Resolved:** Numbers with trend arrows; charts deferred to V2+ (see architecture §4, spec §4.2).
- ~~How many recent activity items should be visible by default?~~ **Resolved:** 10 entries with "View All" link (see spec §9.2).

## Out of Scope

- Report generation or export
- Real-time WebSocket updates
- Widget drag-and-drop reordering
- Role-based widget filtering
- Drill-down detail views within the dashboard page
