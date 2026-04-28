# Detailed Design: Agentic SDLC Control Tower

## Overview

This document captures the product and interface design for Agentic SDLC Control Tower.
It translates the PRD into a design artifact set that can be used for:

- frontend design alignment
- Vue 3 implementation planning
- downstream task breakdown

Current focus:

- establish the shared product shell
- define Round 1 representative pages
- create a stable vocabulary for modules, cards, and user flows

### Implementation Scope Note

For the **Shared App Shell** slice (first implementation), only the following
sections are in scope for coding:

- §Module Design > Global App Shell
- §Vue 3 Component Seed (shell components only: `AppShell.vue`, `PrimaryNav.vue`,
  `TopContextBar.vue`, `GlobalActionBar.vue`, `AiCommandPanel.vue`)
- §Candidate Composables (`useWorkspaceContext()`, `useAiCommandPanel()`)

Page-specific content modules (Dashboard, Project Space, Incident Management,
Platform Center) are documented here for context and visual reference but will
be implemented in subsequent slices.

## Round 1 Prototype References

The current Stitch-generated exploration files are:

- [Control Tower.html](/Users/leo/wwa-lab/GitHub/Agentic-SDLC-Control-Tower/docs/standard-sdd/05-design/Control%20Tower.html:1)
- [Project Space.html](/Users/leo/wwa-lab/GitHub/Agentic-SDLC-Control-Tower/docs/standard-sdd/05-design/Project%20Space.html:1)
- [Incident Command Center.html](/Users/leo/wwa-lab/GitHub/Agentic-SDLC-Control-Tower/docs/standard-sdd/05-design/Incident%20Command%20Center.html:1)
- [Platform Center.html](/Users/leo/wwa-lab/GitHub/Agentic-SDLC-Control-Tower/docs/standard-sdd/05-design/Platform%20Center.html:1)

These files should be treated as exploratory UI artifacts, not yet as final design source of truth.

## Source Architecture

Primary source:

- [agentic_sdlc_control_tower_prd_v0.9.md](/Users/leo/wwa-lab/GitHub/Agentic-SDLC-Control-Tower/docs/standard-sdd/01-requirements/agentic_sdlc_control_tower_prd_v0.9.md:1)

Key source anchors:

- information architecture: [agentic_sdlc_control_tower_prd_v0.9.md](/Users/leo/wwa-lab/GitHub/Agentic-SDLC-Control-Tower/docs/standard-sdd/01-requirements/agentic_sdlc_control_tower_prd_v0.9.md:229)
- page definitions: [agentic_sdlc_control_tower_prd_v0.9.md](/Users/leo/wwa-lab/GitHub/Agentic-SDLC-Control-Tower/docs/standard-sdd/01-requirements/agentic_sdlc_control_tower_prd_v0.9.md:252)
- frontend experience principles: [agentic_sdlc_control_tower_prd_v0.9.md](/Users/leo/wwa-lab/GitHub/Agentic-SDLC-Control-Tower/docs/standard-sdd/01-requirements/agentic_sdlc_control_tower_prd_v0.9.md:457)
- workspace and context model: [agentic_sdlc_control_tower_prd_v0.9.md](/Users/leo/wwa-lab/GitHub/Agentic-SDLC-Control-Tower/docs/standard-sdd/01-requirements/agentic_sdlc_control_tower_prd_v0.9.md:149)
- skill execution model: [agentic_sdlc_control_tower_prd_v0.9.md](/Users/leo/wwa-lab/GitHub/Agentic-SDLC-Control-Tower/docs/standard-sdd/01-requirements/agentic_sdlc_control_tower_prd_v0.9.md:425)

Supporting brief:

- [stitch-brief.md](/Users/leo/wwa-lab/GitHub/Agentic-SDLC-Control-Tower/docs/standard-sdd/05-design/stitch-brief.md:1)

## Design Assumptions

- The first implementation target is a desktop-first enterprise web app.
- The frontend stack will use Vue 3 with Composition API and `<script setup>`.
- Route orchestration will likely use Vue Router. [Assumption]
- Shared client state will likely use Pinia. [Assumption]
- Initial design output will be produced in Stitch, then normalized into this document.
- Round 1 design focuses on app shell plus representative pages, not the full product depth.

## Design Scope

### In Scope

- global app shell
- global navigation and context model
- right-side AI Command Panel
- page-level information hierarchy
- Round 1 pages:
  - Dashboard / Control Tower
  - Project Space
  - Incident Management
  - Platform Center
- Vue 3-oriented component decomposition
- route and state-boundary recommendations

### Out of Scope For This Revision

- detailed backend API contracts
- final visual token values
- design system implementation code
- mobile-first adaptation
- page-complete high-fidelity treatment for every SDLC module

## Module Design

### Global App Shell

#### Purpose

Shared operating shell for all business pages.

#### Required Elements

- left navigation
- top fixed context bar
- page title and action region
- global search
- notifications entry
- audit entry
- right-side persistent AI Command Panel

#### Initial Design Notes

- Context bar must always show `Workspace / Application / SNOW Group / Project / Environment`.
- The shell should support both overview layouts and detail-heavy operational layouts.
- AI Command Panel should remain a core surface instead of an optional drawer.

#### Current Prototype Observations

- The generated shell establishes a consistent dark, high-density control-room style across all four pages.
- The left navigation now maps to the PRD's 13-page information architecture and uses a consistent left-edge active state.
- The top bar now renders the required `Workspace / Application / SNOW Group / Project / Environment` context model on the Round 1 pages.
- The product shell now uses `SDLC Tower` as the operational product name, with `Agentic SDLC Control Tower` retained as the formal long name.
- The right-side AI Command Panel is normalized to a persistent 320px rail in the representative shell patterns.

#### Vue 3 Mapping Notes

- candidate layout component: `AppShell.vue`
- candidate shell regions:
  - `PrimaryNav.vue`
  - `TopContextBar.vue`
  - `GlobalActionBar.vue`
  - `AiCommandPanel.vue`

### Dashboard / Control Tower

#### Purpose

Landing page for value demonstration and operational overview.

#### Required Content

- delivery rhythm
- requirement flow
- testing and quality
- release and incident status
- AI participation
- governance status
- recent key actions
- value story

#### Information Hierarchy

- executive summary
- cross-stage health
- recent operational signals
- evidence of AI contribution
- governance trust indicators

#### Open Design Slots

- accepted card set from Stitch
- approved chart vs narrative balance
- role-specific default arrangement

#### Current Prototype Reference

- [Control Tower.html](/Users/leo/wwa-lab/GitHub/Agentic-SDLC-Control-Tower/docs/standard-sdd/05-design/Control%20Tower.html:1)

### Project Space

#### Purpose

Bridge organizational visibility and project execution.

#### Required Content

- milestones
- status
- roles
- dependencies
- versions
- environments
- risks
- related SDLC entry points

#### Information Hierarchy

- project identity and health
- milestone and timeline snapshot
- environment and version overview
- execution risks and dependencies
- deep links into downstream work areas

#### Open Design Slots

- project hero/header structure
- role summary format
- dependency visualization approach

#### Current Prototype Reference

- [Project Space.html](/Users/leo/wwa-lab/GitHub/Agentic-SDLC-Control-Tower/docs/standard-sdd/05-design/Project%20Space.html:1)

### Incident Management

#### Purpose

AI-native operations command center.

#### Required Modules

- Incident Header
- AI / Skill Execution Timeline
- AI Diagnosis and Reasoning
- AI Actions Taken
- Human Governance
- Related SDLC Chain
- AI Learning and Prevention
- AI Command Panel

#### Required Explicit States

- handled by AI / Human / Hybrid
- autonomy level
- control mode: Auto / Approval / Manual
- executed AI actions
- pending approval actions
- requirement/spec/design/code/test/deploy relation

#### Information Hierarchy

- incident severity and operational posture
- AI reasoning and timeline
- actions taken and actions waiting
- governance and approvals
- SDLC traceability
- learning and prevention

#### Open Design Slots

- timeline density and evidence model
- approval interaction model
- incident-to-SDLC trace visualization

#### Current Prototype Reference

- [Incident Command Center.html](/Users/leo/wwa-lab/GitHub/Agentic-SDLC-Control-Tower/docs/standard-sdd/05-design/Incident%20Command%20Center.html:1)

### Platform Center

#### Purpose

Governance and platform capability hub.

#### Required Domains

- Template Management
- Configuration Management
- Audit Management
- Access Management
- Policy and Governance

#### Information Hierarchy

- governance overview
- policy and access controls
- template and configuration administration
- audit visibility
- change trust and traceability

#### Open Design Slots

- management index layout
- cross-domain navigation model
- policy and audit summary card patterns

#### Current Prototype Reference

- [Platform Center.html](/Users/leo/wwa-lab/GitHub/Agentic-SDLC-Control-Tower/docs/standard-sdd/05-design/Platform%20Center.html:1)

### Round 2 Candidate Modules

The following modules are intentionally deferred for a later pass:

- Team Space
- Requirement Management
- Project Management
- Design Management
- Code & Build Management
- Testing Management
- Deployment Management
- AI Center
- Report Center

## API / Interface Design

The shell V1 uses static or mocked data and has no backend API dependency.
Backend API contracts will be defined in the first data-connected slice (expected
to be Dashboard or Project Space).

Future API surface areas (out of scope for shell slice):

- page-level data dependencies
- query and mutation boundaries
- approval action interfaces
- audit and timeline retrieval interfaces

## Data Design

This revision defines UI-facing conceptual objects only.

### Core Context Objects

- Workspace
- Application
- SNOW Group
- Project
- Environment

### Core Business Objects

- Requirement
- User Story
- Spec
- Architecture Artifact
- Design Artifact
- Task
- Code Change / Build
- Test Plan / Test Run
- Deployment / Release
- Incident
- Learning Artifact
- Skill
- Skill Execution
- Policy
- Template

### UI Data Questions

- Which objects require cross-page persistent summaries?
- Which object states must be visible in badges vs detail drawers?
- Which chain links should be directly navigable in Round 1?

## UI / User Flow Design

### Global Flows

- switch workspace context
- navigate across SDLC stages
- inspect AI-generated evidence
- move from overview to detail
- approve or reject governance-gated actions

### Round 1 Primary Flows

#### Flow 1: Dashboard to Project

1. User lands on Dashboard / Control Tower.
2. User reviews cross-stage signals and recent key actions.
3. User selects a project or risk signal.
4. User enters Project Space for execution-level detail.

#### Flow 2: Incident Response Review

1. User opens an active incident.
2. User reviews incident header and current control mode.
3. User checks AI diagnosis and execution timeline.
4. User inspects actions already taken.
5. User approves, rejects, or escalates pending actions.
6. User reviews linked SDLC evidence and learning notes.

#### Flow 3: Governance Administration

1. Platform user opens Platform Center.
2. User navigates into templates, configuration, audit, access, or policy.
3. User inspects current state and scoped context.
4. User performs or reviews a governed administrative action.

## Workflow / Execution Design

### Shared UX Workflow Rules

- every page must preserve visible context
- AI actions must expose status and evidence
- governed actions must expose approval state
- traceability links should point upstream and downstream where available

### Initial Approval States

- draft
- suggested
- pending approval
- approved
- rejected
- executed

These states are placeholders and should be refined after the first design review. [Assumption]

## Integration Design

This revision remains UI-first, but the product direction now assumes an adapter-based integration model.

UI-level implications:

- deployment, testing, code, and incident data may originate from external systems through workspace-scoped adapters
- integrations must expose both connection state and auditability in the UI
- governance actions may require approvals and audit logging even when the source action originated in an external system
- skill execution history is a first-class cross-page input
- MVP integration surfaces should anticipate Jira, GitLab, Jenkins, and ServiceNow as the initial external systems

## Security / Audit / Reliability Design

### Security

- page visibility must align with workspace and role context
- action availability should be policy-aware
- privileged areas should clearly surface control boundaries

### Audit

- key administrative actions should show audit entry points
- AI recommendations and executions should be reviewable
- incident and governance actions should expose evidence where possible

### Reliability

- operational pages should support loading, empty, and degraded states
- critical panels should have graceful fallback states when evidence is partial

## Validation and Error Handling

Each Round 1 page should define at least these states:

- normal
- loading
- empty
- error
- approval pending

Additional states to refine after Stitch output:

- partial data
- stale data
- permission restricted

## Testing Considerations

### Design Review Checks

- Does the shell keep context visible at all times?
- Does the UI make Spec visible as a first-class object?
- Is Incident Management clearly different from a traditional ticket page?
- Is the AI Command Panel treated as a core operating surface?
- Does Platform Center communicate governance and traceability clearly?

### Vue 3 Implementation Checks

- layout boundaries map cleanly to Vue SFCs
- state ownership does not collapse into one oversized page component
- route structure supports overview and detail transitions
- composables are used for shared cross-page behaviors

## Risks / Design Tradeoffs

- If the dashboard leans too heavily on charts, the product story weakens.
- If the AI panel behaves like a generic chat sidebar, the AI-native positioning weakens.
- If Platform Center looks like plain admin CRUD, governance differentiation weakens.
- If too many pages are designed at once, component vocabulary may fragment.

## Current Prototype Gaps

- Team Space, Report Center, and AI Center still need first-pass page prototypes.
- The HTML files are exploratory design artifacts only; interaction rules, empty states, and responsive behavior are not yet fully normalized.
- Shared tokens, spacing contracts, and component APIs still need to be translated into implementation-ready Vue 3 design decisions.

## Open Questions

- Should Spec appear as a dedicated top-level nav item in V1, or remain embedded within Requirement flows?
- Which page should become the default landing page for non-admin operational users?
- How persistent should the right-side AI Command Panel be in high-density incident workflows?
- Which approval actions should be available inline versus inside drawers or modals?
- Which visual language should dominate the product: operational calm, control-room intensity, or executive clarity?

## Vue 3 Component Seed

These are starter candidates for downstream refinement:

- `AppShell.vue`
- `PrimaryNav.vue`
- `TopContextBar.vue`
- `AiCommandPanel.vue`
- `DashboardView.vue`
- `ProjectSpaceView.vue`
- `IncidentDetailView.vue`
- `PlatformCenterView.vue`
- `MetricStoryCard.vue`
- `ExecutionTimeline.vue`
- `GovernanceQueueCard.vue`
- `SdlcTracePanel.vue`

### Candidate Composables

- `useWorkspaceContext()`
- `useAiCommandPanel()`
- `useIncidentTimeline()`
- `useGovernanceApprovals()`
- `useAuditFilters()`

## Next Update Instructions

After Stitch returns a first design pass:

1. capture the accepted visual direction in this document
2. replace each "Open Design Slots" subsection with approved modules and layout choices
3. add route map and state-boundary decisions
4. convert approved sections into execution tasks
