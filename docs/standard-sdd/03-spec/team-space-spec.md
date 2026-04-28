# Feature Specification: Team Space

| Attribute | Value |
|-----------|-------|
| Slice | team-space |
| Source Requirements | [team-space-requirements.md](../01-requirements/team-space-requirements.md) |
| Source Stories | [team-space-stories.md](../02-user-stories/team-space-stories.md) |
| PRD | [agentic_sdlc_control_tower_prd_v0.9.md](../01-requirements/agentic_sdlc_control_tower_prd_v0.9.md) §11.2 |

## Overview

Team Space is the **Workspace-level operating home** of the Agentic SDLC Control Tower. It is the contextual bridge between Dashboard (cross-team, cross-project global view) and Project Space (single-project execution view).

For a selected Workspace, Team Space answers:

- Who is this team and what does it own?
- What governance model, templates, and AI operating model are currently in effect?
- What requirements, Specs, risks, approvals, and incidents need attention?
- How healthy is the team on the SDLC main chain?
- Which projects should the Team Lead jump into first?

All data is strictly scoped to a single Workspace. Cross-workspace aggregation lives on Dashboard; single-project execution lives in Project Space.

## Source Stories

| Story | Description |
|-------|-------------|
| S1 | View Workspace Summary |
| S2 | Inspect Team Operating Model |
| S3 | Review Member & Role Matrix |
| S4 | Inspect Team Default Templates |
| S5 | Monitor Requirement & Spec Pipeline |
| S6 | Read Team Metrics |
| S7 | Scan the Team Risk Radar |
| S8 | Browse Project Distribution |
| S9 | Switch Workspace Context |
| S10 | Handle Loading, Error, Empty States |
| S11 | Use AI Command Panel for Team Questions |
| S12 | Breadcrumb Back Navigation |

## Actors / Users

| Actor | Goal |
|-------|------|
| Team Lead / Delivery Manager | Daily operating awareness, triage of risks and approvals |
| Application Owner | Governance posture of one or more Workspaces |
| Platform Reviewer | Observing how platform-level defaults materialize in a Workspace |
| PM / Architect / Dev / QA / SRE | Orienting to team operating model and jumping into own Project Space |
| AI Command Panel | Consumes Team-Space context for assistive actions |

## Functional Scope

The Team Space page consists of the following top-level UI elements:

1. **Workspace Summary card** (REQ-TS-10, REQ-TS-11, REQ-TS-12)
2. **Team Operating Model card** (REQ-TS-20, REQ-TS-21, REQ-TS-22)
3. **Member & Role Matrix card** (REQ-TS-30, REQ-TS-31, REQ-TS-32)
4. **Team Default Templates card** (REQ-TS-40, REQ-TS-41, REQ-TS-42)
5. **Requirement & Spec Pipeline card** (REQ-TS-50, REQ-TS-51, REQ-TS-52, REQ-TS-53)
6. **Team Metrics card** (REQ-TS-60, REQ-TS-61, REQ-TS-62)
7. **Team Risk Radar card** (REQ-TS-70, REQ-TS-71, REQ-TS-72)
8. **Project Distribution card** (REQ-TS-80, REQ-TS-81, REQ-TS-82)
9. **SDLC 11-node main-chain strip** (REQ-TS-134)

Rendered inside the shared app shell, consuming the Workspace context bar, and projecting context to the right-side AI Command Panel.

## Functional Requirements

### F-TS-SUMMARY: Workspace Summary

- Display Workspace name, Application, Primary SNOW Group (nullable), active project count, active env count, aggregate health (green/yellow/red), accountable owner.
- When Primary SNOW Group is null (compatibility mode), render a neutral "Not configured" chip.
- Aggregate health is derived server-side from risks + incident open count + approval backlog.

### F-TS-OPMODEL: Team Operating Model

- Display Operating Mode, default approval mode, AI Autonomy Level, oncall owner, accountable owners.
- Each value displays inheritance lineage: Platform Default / Application / SNOW Group / Workspace Override / Project Override.
- Lineage resolution is read-only.
- "View in Platform Center" link is gated by permission `platform.config.view`.

### F-TS-MEMBERS: Member & Role Matrix

- List members with display name, role(s), oncall status, key permissions summary, last-active date.
- Banner at top enumerates coverage gaps: missing oncall windows, unfilled approver role, etc.
- "Manage in Access Management" link is gated by permission `access.manage`.

### F-TS-TEMPLATES: Team Default Templates

- List templates by group: Page Templates, Policies, Workflows, Skill Packs, AI Defaults.
- Each entry shows name, version, lineage, override status.
- Separate section enumerates project-level exception overrides.
- Empty state for a new Workspace with no project overrides.

### F-TS-PIPELINE: Requirement & Spec Pipeline

- Show counters: Requirements inflow (7-day), Stories decomposing, Specs generating / in review / blocked, approved Specs awaiting downstream.
- Blocker highlights: Specs blocked > N days, Requirements without stories > N days, Stories without Spec > N days. N is configurable per Workspace with a default.
- Counters and blockers deep-link into Requirement Management with pre-applied filters.
- Render compressed SDLC 11-node main-chain strip with Spec highlighted; show per-node health.

### F-TS-METRICS: Team Metrics

- Show groups: Delivery Efficiency, Quality, Stability, Governance Maturity, AI Participation.
- Each metric displays current value, previous-period delta, trend indicator.
- Each metric has a "View history" link deep-linking into Report Center.

### F-TS-RISK: Team Risk Radar

- Group risks by category: High-risk projects, Dependency blocks, Approval backlog, Config drift, Incident hotspots.
- Order by severity, not recency. Critical risks use crimson accent.
- Each item has a primary action entry point (open Project Space, open Incident, open Approval).
- Empty state "All green" when no risks exist.

### F-TS-PROJECTS: Project Distribution

- Show projects stratified by Healthy / At-Risk / Critical / Archived.
- Each project card shows name, lifecycle stage, health indicator, primary risk, active Spec count, open Incident count.
- Clicking a card navigates to the Project Space (or fallback stub page if not yet implemented).

### F-TS-NAV: Navigation & Context

- Stable shell route: `/team`.
- Deep-linkable Workspace URL: `/team?workspaceId=:workspaceId`.
- Workspace switch in the shell updates the URL and re-fetches cards without full reload.
- Breadcrumb trail registered: `Dashboard / Team Space (Workspace X) / <downstream>`.

### F-TS-STATE: Loading, Error, Empty States

- Per-card skeleton while loading.
- Per-card error with retry; retry does not affect other cards.
- Page-level error only for auth failure or Workspace access denial.
- Distinct per-card empty states.

### F-TS-AI: AI Command Panel Integration

- Project Workspace context (workspaceId, visible risks, pipeline counters) to the AI Command Panel.
- Surface Team-Space-appropriate suggested prompts.
- Each AI response is recorded as a Skill Execution with evidence links.

## Non-Functional Requirements

| Requirement | Target |
|-------------|--------|
| First-paint latency | < 300ms after Workspace id is received |
| Per-card hydration | < 500ms under typical Workspace volume (< 20 members, < 30 projects) |
| Workspace isolation | Hard server-side scoping; cross-workspace leakage is a defect |
| Build / test success | `npm run build` + `./mvnw verify` must both pass |
| Schema changes | Flyway migrations only; no `ddl-auto: update` |
| Visual style | Tactical Command aesthetic, card density consistent with design.md §2 |

## Workflow / System Flow

### Primary Flow: Page Load

1. User navigates to `/team` or `/team?workspaceId=:workspaceId` (directly or via Dashboard drill-down).
2. Shell resolves the active Workspace, updates the context bar.
3. Team Space resolves the target `workspaceId` from the route query and then loads the aggregate endpoint for first paint.
4. Each card renders its skeleton until its aggregate section resolves.
5. Cards hydrate independently from the aggregate response; later retries use the per-card endpoints.
6. The SDLC main-chain strip renders once pipeline + risk data is available.

### Secondary Flow: Workspace Switch

1. User selects a different Workspace in the shell switcher.
2. Router updates URL to `/team?workspaceId=:newWorkspaceId`.
3. Page reloads the aggregate Team Space payload scoped to the new Workspace.
4. Existing card state is replaced via per-card skeletons (no full-page flash).
5. Breadcrumb updates.

### Secondary Flow: Drill-Down Navigation

1. User clicks a pipeline counter, risk item, or project card.
2. Router navigates to target surface (Requirement Management / Incident / Project Space) with context preserved.
3. Breadcrumb trail records the Team Space step.
4. Browser Back restores Team Space at the same Workspace with prior scroll.

## Data / Configuration Requirements

### Workspace Context

- Every fetch must include `workspaceId` as the scoping key.
- Backend enforces `workspaceId`-based authorization on every endpoint.

### Configuration Inheritance

Resolution order (read-only on this page):

```
Platform Default
  → Application Default
    → SNOW Group Override
      → Workspace Override
        → Project Override (for template/policy inheritance)
```

### Deep-link Query Parameters

| Parameter | Purpose |
|-----------|---------|
| `filter=blocked-specs` | Pre-filter Requirement Management |
| `filter=pending-approvals` | Pre-filter Platform Center approvals stub |
| `projectId=...` | Jump into Project Space |
| `incidentId=...` | Jump into Incident Management |

## Integrations

| System | Role |
|--------|------|
| Shared App Shell | Hosts the page, provides context bar, AI Command Panel, breadcrumb |
| Dashboard | Drill-in entry point into Team Space |
| Requirement Management | Forward navigation from pipeline counters |
| Incident Management | Forward navigation from Risk Radar incident hotspots |
| Project Space | Forward navigation from Project Distribution |
| Platform Center | Read-only links into template / policy / AI config detail |
| Access Management | Read-only link into member / permission detail |
| Report Center | Metric history deep-link |
| AI Command Panel | Receives page context |

## Dependencies

| Dependency | Status | Notes |
|-----------|--------|-------|
| shared-app-shell slice | Completed | Provides context bar, shell, AI panel |
| dashboard slice | Completed | Provides entry links |
| requirement slice | Completed | Downstream navigation target with filter vocabulary |
| incident slice | Completed | Downstream navigation target |
| project-space slice | Not started | Fallback stub page until this slice lands |
| Platform Center | Not started | Team Space uses stub deep-links until Platform Center lands |

## Risks / Ambiguities

- **Risk:** Aggregate health computation overlaps with Dashboard's health card. Mitigation: single backend service owns the computation, both pages consume it.
- **Risk:** Project Space may not be implemented when Team Space ships. Mitigation: navigation resolves to a deferred-stub page rather than 404.
- **Ambiguity:** Exact metric definitions for "Governance Maturity" and "AI Participation" — open question for Platform Center alignment (see Open Questions).

## Out of Scope

- Editing any template, policy, workflow, or AI config
- Granting or revoking member permissions
- Creating projects or workspaces
- Full incident lifecycle UI
- Exporting reports
- Cross-workspace aggregation

## Data Contracts Summary

See [team-space-data-model.md](../04-architecture/team-space-data-model.md) for full type definitions. High-level summary:

- `WorkspaceSummaryDto` — identity, counts, health
- `TeamOperatingModelDto` — operating mode, approval mode, AI autonomy, oncall, with lineage
- `MemberMatrixDto` — members, roles, oncall, coverage gaps
- `TeamDefaultTemplatesDto` — templates grouped with lineage and override state
- `RequirementPipelineDto` — counters, blockers, chain health
- `TeamMetricsDto` — metric values with trend
- `TeamRiskRadarDto` — risks grouped by category and severity
- `ProjectDistributionDto` — project cards grouped by health stratum
- `TeamSpaceAggregateDto` — optional top-level aggregate wrapper

## API Contracts Summary

See [team-space-API_IMPLEMENTATION_GUIDE.md](../05-design/contracts/team-space-API_IMPLEMENTATION_GUIDE.md) for full endpoint contracts.

| Endpoint | Purpose |
|----------|---------|
| `GET /api/v1/team-space/:workspaceId` | Aggregate Team Space view (all cards in one call) |
| `GET /api/v1/team-space/:workspaceId/summary` | Workspace Summary only |
| `GET /api/v1/team-space/:workspaceId/operating-model` | Operating Model only |
| `GET /api/v1/team-space/:workspaceId/members` | Members matrix |
| `GET /api/v1/team-space/:workspaceId/templates` | Default Templates |
| `GET /api/v1/team-space/:workspaceId/pipeline` | Requirement/Spec pipeline |
| `GET /api/v1/team-space/:workspaceId/metrics` | Metrics |
| `GET /api/v1/team-space/:workspaceId/risks` | Risk Radar |
| `GET /api/v1/team-space/:workspaceId/projects` | Project Distribution |

Per-card endpoints support retry granularity; aggregate endpoint supports first-paint latency target.

## Acceptance Matrix

| Requirement | Story | Acceptance Check |
|-------------|-------|------------------|
| REQ-TS-10, 11, 12 | S1 | Workspace Summary card renders identity + health; compatibility mode handled |
| REQ-TS-20, 21, 22 | S2 | Operating Model card shows values + lineage + Platform link |
| REQ-TS-30, 31, 32 | S3 | Member matrix + coverage gap banner + Access link |
| REQ-TS-40, 41, 42 | S4 | Templates grouped with lineage; exception overrides section |
| REQ-TS-50, 51, 52, 53 | S5 | Pipeline counters + blockers + chain strip + deep-links |
| REQ-TS-60, 61, 62 | S6 | Metrics + trend + Report Center link |
| REQ-TS-70, 71, 72 | S7 | Risk Radar grouped + severity-ordered + actions |
| REQ-TS-80, 81, 82 | S8 | Projects stratified + navigate to Project Space |
| REQ-TS-90, 91, 120 | S9 | Workspace switch re-fetches without reload |
| REQ-TS-132, 133 | S10 | Per-card skeletons / errors / empty states / retry |
| REQ-TS-100, 101, 102 | S11 | AI Command Panel context + suggested prompts + skill records |
| REQ-TS-91, 92 | S12 | Breadcrumb trail on downstream navigation |

## Open Questions

- **OQ-1:** Final metric definitions for Governance Maturity and AI Participation — needs Platform Center alignment.
- **OQ-2:** Whether project lifecycle stage enum is sourced from Project Management slice (future) or Team Space defines its own.
- **OQ-3:** Thresholds (`N days`) for pipeline blockers — default values vs per-Workspace configuration. Proposed default: 3 days.
- **OQ-4:** Whether the AI Command Panel needs a dedicated Team-Space skill (e.g., `team-space-summarizer`) or reuses dashboard-level skills.
- **OQ-5:** Permission check strategy — whether "View in Platform Center" / "Manage in Access Management" links are hidden or disabled when permission is absent.
