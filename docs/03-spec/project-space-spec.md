# Feature Specification: Project Space

| Attribute | Value |
|-----------|-------|
| Slice | project-space |
| Source Requirements | [project-space-requirements.md](../01-requirements/project-space-requirements.md) |
| Source Stories | [project-space-stories.md](../02-user-stories/project-space-stories.md) |
| PRD | [agentic_sdlc_control_tower_prd_v0.9.md](../01-requirements/agentic_sdlc_control_tower_prd_v0.9.md) §11.3 |

## Overview

Project Space is the **single-project execution home** of the Agentic SDLC Control Tower. It is the contextual bridge between Team Space (Workspace-level operating home) and the lifecycle pages (Requirement / Design / Code / Test / Deploy / Incident).

For a selected Project, Project Space answers:

- What is this project and what is its current health?
- Who owns each accountable role and where are the coverage gaps?
- Where are we on the 11-node SDLC main chain for this project?
- What milestone are we on, and is anything slipping?
- What dependencies (upstream / downstream) are blocking us?
- What are the top active risks?
- Where are our environments and is there version drift?

All data is strictly scoped to a single Project. Cross-project aggregation lives on Team Space / Dashboard; execution detail lives in the lifecycle pages.

## Source Stories

| Story | Description |
|-------|-------------|
| S1 | View Project Summary Bar |
| S2 | Inspect Leadership & Ownership |
| S3 | Navigate the Project SDLC Chain |
| S4 | Track the Milestone Execution Hub |
| S5 | Read the Operational Dependency Map |
| S6 | Scan the Risk & Vulnerability Registry |
| S7 | Read the Environment & Version Matrix |
| S8 | Switch Project Context |
| S9 | Handle Loading, Error, Empty States |
| S10 | Use AI Command Panel for Project Questions |
| S11 | Breadcrumb Back Navigation |
| S12 | Governed Navigation Auditing |

## Actors / Users

| Actor | Goal |
|-------|------|
| Project Manager / Delivery Lead | Milestone, slippage, approval backlog awareness |
| Architect | Spec coverage, dependency shape, drift detection |
| Tech Lead | Engineering readiness, environment posture |
| QA Lead / SRE | Quality + stability posture per project |
| Team Lead / Application Owner | Drilling from Team Space into a specific project |
| AI Command Panel | Consumes Project-Space context for assistive actions |

## Functional Scope

The Project Space page consists of the following top-level UI elements:

1. **Project Summary Bar** (REQ-PS-10, REQ-PS-11, REQ-PS-12)
2. **Leadership & Ownership card** (REQ-PS-20, REQ-PS-21, REQ-PS-22)
3. **SDLC Deep Links card** (REQ-PS-30, REQ-PS-31, REQ-PS-32)
4. **Milestone Execution Hub card** (REQ-PS-40, REQ-PS-41, REQ-PS-42)
5. **Operational Dependency Map card** (REQ-PS-50, REQ-PS-51, REQ-PS-52)
6. **Risk & Vulnerability Registry card** (REQ-PS-60, REQ-PS-61, REQ-PS-62)
7. **Environment & Version Matrix card** (REQ-PS-70, REQ-PS-71, REQ-PS-72)
8. **SDLC 11-node chain strip** (REQ-PS-124, integrated into #3)

Rendered inside the shared app shell, consuming shared Project context from the route and shell state, and projecting context to the right-side AI Command Panel.

## Functional Requirements

### F-PS-SUMMARY: Project Summary Bar

- Display Project name + id, parent Workspace (with back link to Team Space), Application, lifecycle stage, aggregate health, PM, Tech Lead, active milestone label + target date, last update timestamp.
- Compact counters: active Specs, open Incidents, pending Approvals, active Critical+High risks.
- Aggregate health is derived server-side from risks + incidents + approvals + milestone slippage.
- Hovering the health LED reveals the contributing factors.

### F-PS-LEADERSHIP: Leadership & Ownership

- Display one row per accountable role: PM, Architect, Tech Lead, QA Lead, SRE, AI Adoption.
- Each row shows display name, oncall status (On / Off / Upcoming), backup presence indicator.
- Roles without a named backup show a "no backup" chip.
- "Manage in Access Management" link is gated by permission `access.manage`.
- Empty state per role is explicit ("Not assigned") rather than blank.

### F-PS-CHAIN: SDLC Deep Links

- Render 11 nodes in canonical order (Requirement → Story → Spec → Architecture → Design → Tasks → Code → Test → Deploy → Incident → Learning).
- Each node shows a compact count / health indicator.
- Spec node visibly emphasized.
- Each node links to the corresponding lifecycle page pre-filtered by `projectId`.
- Nodes whose target page is not yet implemented render as disabled with a "coming soon" state (via feature flag).

### F-PS-MILESTONES: Milestone Execution Hub

- Render milestones in chronological order.
- Each milestone shows label, target date, status (`NOT_STARTED` / `IN_PROGRESS` / `AT_RISK` / `COMPLETED` / `SLIPPED`), completion %, owner, slippage reason.
- Current milestone marked prominently.
- At-Risk / Slipped use crimson accent.
- "Manage in Project Management" link; card itself read-only.
- Empty state with create-in-Project-Management CTA when no milestones exist.

### F-PS-DEPENDENCIES: Operational Dependency Map

- Two groups: Upstream dependencies, Downstream dependents.
- Each entry: name, relationship type (`API` / `DATA` / `SCHEDULE` / `SLA`), owner team, current health, blocker reason.
- External dependencies show "External" badge; no Project Space link.
- Blockers use crimson accent.
- Primary action per entry: open dependency's Project Space or related Incident.

### F-PS-RISKS: Risk & Vulnerability Registry

- List active risks ordered by severity (Critical → High → Medium → Low), then age desc.
- Each entry: title, severity chip, category, owner, age in days, latest mitigation note, primary action.
- Critical risks use crimson accent.
- Empty state "All green" when no risks.

### F-PS-ENVIRONMENTS: Environment & Version Matrix

- List environments (DEV / STAGING / PROD + custom).
- Each tile: env label, version / build id, last-deploy timestamp, health, gate status (`AUTO` / `APPROVAL_REQUIRED` / `BLOCKED`), approver when pending.
- Version drift threshold: if PROD and STAGING diverge by > 10 commits (configurable), show "drift" indicator.
- Each tile links to Deployment Management for the env.

### F-PS-CONTEXT: Context Bar Integration

- Consumes `currentProject` from the shell context bar.
- Switching Project re-fetches without full reload.
- If `currentProject.workspaceId !== shell.currentWorkspace.id`, the shell auto-selects the Project's Workspace and Application first.

### F-PS-AI-PANEL: AI Command Panel Integration

- Push `ProjectSpaceContext` (`{ projectId, workspaceId, activeMilestone, topRisks, envPosture }`) on mount and on switch.
- Render at least 3 Project-appropriate suggested prompts.
- Every skill execution recorded via existing skill-execution audit pipeline.

### F-PS-STATES: State Handling

- Per-card loading / error / empty / ready.
- Per-card retry leaves other cards intact.
- Aggregate-endpoint failure falls back to per-card endpoints; double failure triggers page-level error.
- Auth / access denial renders page-level error with a link back to Team Space.

## API Contract (Summary)

### Aggregate

```
GET /api/v1/project-space/{projectId}
→ 200: { data: ProjectSpaceAggregateDto, error: null }
```

Response contains eight `SectionResult<T>` slots: `summary`, `leadership`, `chain`, `milestones`, `dependencies`, `risks`, `environments`, `activity`.

### Per-Card Refresh

```
GET /api/v1/project-space/{projectId}/summary
GET /api/v1/project-space/{projectId}/leadership
GET /api/v1/project-space/{projectId}/chain
GET /api/v1/project-space/{projectId}/milestones
GET /api/v1/project-space/{projectId}/dependencies
GET /api/v1/project-space/{projectId}/risks
GET /api/v1/project-space/{projectId}/environments
```

Authorization: `projectId` must resolve to a Project the authenticated principal can read. `ProjectAccessGuard` enforces this.

Error envelope: `{ data: null, error: "<message>" }` with HTTP 403 / 404 as appropriate.

Full contract detail lives in [../05-design/contracts/project-space-API_IMPLEMENTATION_GUIDE.md](../05-design/contracts/project-space-API_IMPLEMENTATION_GUIDE.md).

## State Model (Frontend Store)

```
ProjectSpaceState = {
  projectId: string | null
  workspaceId: string | null
  cards: {
    summary:      CardState<ProjectSummary>
    leadership:   CardState<LeadershipOwnership>
    chain:        CardState<SdlcChainState>
    milestones:   CardState<MilestoneHub>
    dependencies: CardState<DependencyMap>
    risks:        CardState<RiskRegistry>
    environments: CardState<EnvironmentMatrix>
  }
}

CardState<T> = { state: 'idle'|'loading'|'ready'|'error'|'empty', data: T|null, error: string|null }
```

## Navigation / Deep-Link Vocabulary

| From | To | Query |
|------|----|-------|
| Project Summary Bar → Team Space | `/team` | `workspaceId` |
| SDLC node (Requirement) | `/requirements` | `projectId`, `workspaceId` |
| SDLC node (Design) | `/design` | `projectId`, `workspaceId` (feature-flagged) |
| SDLC node (Code) | `/code` | `projectId`, `workspaceId` (feature-flagged) |
| SDLC node (Test) | `/testing` | `projectId`, `workspaceId` (feature-flagged) |
| SDLC node (Deploy) | `/deployment` | `projectId`, `workspaceId` (feature-flagged) |
| SDLC node (Incident) | `/incidents` | `projectId`, `workspaceId` |
| Milestone card → Project Management | `/project-management` | `projectId` (feature-flagged) |
| Dependency → Project Space | `/project-space/:projectId` | `workspaceId` (optional) |
| Dependency → Incident | `/incidents/:id` | — |
| Risk → Action entry | depends on action | — |
| Environment → Deployment Management | `/deployment` | `projectId`, `envId` (feature-flagged) |

Feature flags: `project-space.enabled`, `project-management-link.enabled`, `design-link.enabled`, `code-link.enabled`, `testing-link.enabled`, `deployment-link.enabled`.

## Non-Functional Requirements

- First paint < 300ms after receiving `projectId`.
- Per-card hydration < 500ms under typical volumes.
- Per-card timeouts isolate failures.
- Accessibility: keyboard navigation across cards; ARIA roles on interactive entries.
- Responsive down to 1280 wide (desktop-first per PRD §1).

## Testing Contract

- Unit: per-card render + state transitions (Vitest).
- Integration: store interactions, route param changes, context-bar driven switch.
- Backend unit: per-projection logic with stub facades.
- Backend integration: MockMvc with in-memory DB, full aggregate + per-card endpoints, access denial path.
- Contract: shared fixtures verify frontend types match backend DTO JSON (snapshot tests).

## Open Questions

- Milestone completion % source: derived from Tasks coverage vs manual? Confirm with Project Management owners before B3.
- Dependency data source in V1: static registry in `project_dependencies` table or real-time probe of upstream health? Start with static registry; real-time probe deferred.
- Version drift threshold default: proposed 10 commits; confirm per environment.
- Whether `AI Adoption owner` is a first-class role in V1 or derived from Workspace default; confirm before S2 implementation.
- AI Command Panel: is there a Project-Space-specific skill pack, or do we reuse Team-Space skills with re-scoping? Confirm before A13.
- Permission-gated link strategy (hide vs disable) — reuse Team Space precedent unless overridden.
