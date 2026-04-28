# Team Space Requirements

## Purpose

This document extracts the requirements relevant to the **Team Space** page from the full PRD (V0.9). It defines **what** the Team Space page must deliver, serving as the upstream input for the spec, architecture, design, and task documents of this slice.

Team Space is the **Workspace-level operating home** — the contextual layer between Dashboard (cross-team global view) and Project Space (single-project execution view). It answers "who is this team, what do they own, what is their governance model, and how healthy are they?" for a single Workspace.

## Source

All requirements below are derived from:

- [agentic_sdlc_control_tower_prd_v0.9.md](agentic_sdlc_control_tower_prd_v0.9.md)

Section references use the format `PRD §N.N`.

---

## 1. Scope

The Team Space page is described in the PRD as the single-Workspace operating home — the primary entry point of the multi-team isolation model (PRD §11.2).

It covers:

- Workspace Summary card (workspace identity, application, primary SNOW group, health)
- Team Operating Model card (operating mode, approval mode, AI autonomy level, oncall strategy, accountable owners)
- Member & Role Matrix card (members, roles, oncall duty, key permissions, recent activity)
- Team Default Templates card (templates, policies, workflows, skill packs, AI defaults — inherited / overridden / effective)
- Requirement & Spec Pipeline card (team-level inflow, story decomposition, Spec generation / review / block state)
- Team Metrics card (delivery efficiency, quality, stability, governance maturity, AI participation)
- Team Risk Radar card (high-risk projects, dependency blocks, approval backlog, config drift, incident hotspots)
- Project Distribution card (project list, status stratification, entry points into Project Space)
- SDLC 11-node main-chain health strip (Workspace-scoped)
- Workspace context switcher integration with the shared context bar
- Navigation entry points to Requirement Management, Project Space, Platform Center config inspection, Incident Management, AI Center

It does **not** cover:

- Editing templates, policies, or AI configuration (Platform Center owns editing; Team Space shows **effective / inherited / overridden** state, read-only)
- Single-project execution detail (Project Space owns that)
- Cross-workspace or cross-team aggregation (Dashboard owns that)
- Workspace creation, deletion, or admin (Platform Center)
- Member onboarding / permission granting (Access Management inside Platform Center)
- Requirement detail (Requirement Management)
- Incident detail (Incident Management)
- Historical exports and structured reports (Report Center)

---

## 2. Product Context Requirements

### REQ-TS-01: Single-Workspace operating home

Team Space is the operating home for **exactly one Workspace at a time**. It is not a cross-team aggregator. All data, metrics, and navigation points are scoped to the currently-selected Workspace.

> Source: PRD §11.2 — "Team Space 应被定义为单一 Workspace 的运营主页"

### REQ-TS-02: Context bridge between Dashboard and Project Space

Team Space is the contextual bridge between Dashboard (cross-team, cross-project global view) and Project Space (single-project execution view). Navigation between these three pages must preserve Workspace context and feel continuous.

> Source: PRD §11.2 — "Dashboard 与 Project Space 之间的上下文承接层"

### REQ-TS-03: Three audience layers

The page must serve:

- **Team Lead / Delivery Manager / Application Owner**: Workspace-level health, risk, approval backlog, governance posture
- **Team members (PM, Architect, Dev, QA, SRE)**: understanding the team's operating model, defaults, and how their project fits in
- **Platform governance / AI adoption reviewers**: observing how platform-level templates and AI configuration materialize at the team level

> Source: PRD §6.2, §6.3, §6.5

### REQ-TS-04: Configuration-driven layout

Card order, card visibility, and card-level default filters must be configuration-driven so a Workspace can tune its Team Space without code changes.

> Source: PRD §7.8 — "配置驱动"

### REQ-TS-05: AI is a participant, not a widget

Each Team Space card that surfaces an AI decision or signal (e.g., risk radar, operating model summary, requirement pipeline) must explicitly identify AI involvement — what the AI did, what it recommends, and whether human governance is required.

> Source: PRD §7.4 — "AI 是参与者，不是旁观者"

### REQ-TS-06: Read-only surface for platform-owned config

Team Space must display inherited platform/application/SNOW group configuration as **read-only**. Editing lives in Platform Center. Team Space reveals the effective state and the override lineage, not the edit surface.

> Source: PRD §9.3 — 四层继承机制; PRD §11.12 — Platform Center

---

## 3. Workspace Summary Card Requirements

### REQ-TS-10: Workspace identity

The Workspace Summary card must display:

- Workspace name and identifier
- Application name
- Primary SNOW Group (may be empty in compatibility mode per PRD §8.1)
- Active project count
- Active environment count
- Current aggregate health indicator (green / yellow / red)
- Accountable owner (Team Lead / Application Owner)

> Source: PRD §8.1, §11.2

### REQ-TS-11: Responsibility boundary

The card must display the **responsibility boundary** of this Workspace — the Applications, SNOW Groups, and Projects it owns or co-owns — to answer "what does this team own?".

> Source: PRD §11.2 — "当前团队是谁，负责哪些 Application / SNOW Group / Project 边界"

### REQ-TS-12: Compatibility mode

When the Workspace is configured in compatibility mode (no Primary SNOW Group), the card must clearly surface that state and not render the SNOW Group field as blank or broken.

> Source: PRD §8.1

---

## 4. Team Operating Model Card Requirements

### REQ-TS-20: Operating mode disclosure

The Team Operating Model card must display the currently-effective:

- **Operating Mode** (e.g., Standard / High-Governance / Fast-Path)
- **Default approval mode** (Auto / Reviewer-required / Multi-approver)
- **AI Autonomy Level** (Suggest-only / Human-in-loop / Autonomous-with-audit)
- **Oncall / duty strategy** (current oncall owner, rotation reference)
- **Accountable owners** (who owns what governance decision)

> Source: PRD §11.2

### REQ-TS-21: Inheritance lineage

Each value must show the **inheritance lineage** — where it was set (Platform Default / Application Default / SNOW Group Override / Project Override), so the viewer understands why the effective value is what it is.

> Source: PRD §9.3

### REQ-TS-22: Link to Platform Center

The card must provide a navigable entry point into Platform Center's configuration view for this Workspace, for users with the required permission.

> Source: PRD §11.12

---

## 5. Member & Role Matrix Card Requirements

### REQ-TS-30: Member list

The card must display the members of this Workspace, each showing:

- Display name and identifier
- Role(s) within this Workspace
- Oncall duty status (On / Off / Upcoming)
- Key permissions summary (e.g., Approve / Deploy / Configure)
- Recent activity indicator (last active date)

### REQ-TS-31: Role coverage gaps

The card must highlight **coverage gaps** — roles that are unfilled, oncall gaps (e.g., no primary oncall for upcoming window), and key positions without a backup.

> Source: PRD §11.2 — "识别值班与责任覆盖空洞"

### REQ-TS-32: Link to Access Management

The card must provide a navigable entry to Access Management (inside Platform Center) for permission changes — Team Space is read-only for permissions.

> Source: PRD §11.12

---

## 6. Team Default Templates Card Requirements

### REQ-TS-40: Effective defaults

The card must display the **effective defaults** for this Workspace across:

- Page templates
- Policy templates
- Workflow templates
- AI skill packs
- AI default configuration (model, autonomy, logging)

### REQ-TS-41: Override indicators

Each entry must clearly indicate whether it is:

- **Inherited** from Platform / Application / SNOW Group level (shown with lineage badge)
- **Overridden** at Workspace level (shown with override badge and link to the override)

> Source: PRD §9.3, §11.2

### REQ-TS-42: Exception inspection

The card must list any **exception overrides** currently in effect — e.g., a Project-level override of a Workspace-level default — so the Team Lead can see where exceptions exist.

> Source: PRD §11.2 — "例外覆盖点"

---

## 7. Requirement & Spec Pipeline Card Requirements

### REQ-TS-50: Pipeline flow counters

The card must display team-level counters along the primary chain:

- Requirements flowing in (last N days)
- User Stories being decomposed (in progress)
- Specs being generated / in review / blocked
- Approved Specs awaiting downstream work

> Source: PRD §11.2 — "Requirement & Spec Pipeline"

### REQ-TS-51: Blocker highlights

The card must highlight:

- Specs blocked awaiting approval (> N days)
- Requirements with no decomposed stories (> N days)
- Stories with no linked Spec (> N days)

### REQ-TS-52: Navigation into Requirement Management

The card must provide deep-link navigation into Requirement Management, pre-filtered by the clicked pipeline signal (e.g., "blocked specs" → Requirement Management with filter = spec.status:blocked).

> Source: PRD §11.2, §11.4

### REQ-TS-53: 11-node main-chain vocabulary

When the pipeline card (or any card) renders the SDLC main chain, it must use the unified 11-node vocabulary and visibly emphasize the Spec node as the execution hub.

> Source: PRD §13.1, §7.6

---

## 8. Team Metrics Card Requirements

### REQ-TS-60: Core metrics

The card must display Workspace-scoped versions of:

- **Delivery efficiency** (e.g., cycle time, throughput)
- **Quality** (e.g., defect rate, test pass rate)
- **Stability** (e.g., incident frequency, MTTR)
- **Governance maturity** (e.g., approval coverage, audit completeness)
- **AI participation** (e.g., % of skill-driven actions, AI decision acceptance rate)

> Source: PRD §4.4, §11.2

### REQ-TS-61: Trend indicators

Each metric must show a trend indicator (up / down / flat) over the previous period.

### REQ-TS-62: Drill into Report Center

Metric drill-downs must link into Report Center for historical analysis, not embed full historical charts.

> Source: PRD §10 — Dashboard vs Report Center separation

---

## 9. Team Risk Radar Card Requirements

### REQ-TS-70: Risk categories

The card must surface risks in at least these categories:

- **High-risk projects** (risk tier, primary risk reason)
- **Dependency blocks** (blocked task, blocking party, age)
- **Approval backlog** (pending approvals by age, owner)
- **Config drift** (deviations from template/policy defaults)
- **Incident hotspots** (recurring incidents, open incidents in this Workspace)

> Source: PRD §11.2 — "Team Risk Radar"

### REQ-TS-71: Severity ordering

Risks must be ordered by severity, not recency. Critical risks must use the crimson accent (consistent with the broader design system).

### REQ-TS-72: Action entry points

Each risk item must offer at least one direct action entry point — jump to the related Project Space, open the related Incident, or navigate to the approval.

---

## 10. Project Distribution Card Requirements

### REQ-TS-80: Project list / grid

The card must display the projects in this Workspace as cards or a compact grid showing:

- Project name and identifier
- Lifecycle stage (e.g., Discovery / Delivery / Steady-State / Retiring)
- Health indicator
- Primary risk (if any)
- Active Specs count
- Open Incidents count

### REQ-TS-81: Status stratification

Projects must be stratified by status (e.g., Healthy / At-risk / Critical / Archived tabs or groups) with counts per stratum.

### REQ-TS-82: Entry into Project Space

Each project card must offer a direct entry into the Project Space of that project, preserving Workspace context in navigation state.

> Source: PRD §11.3

---

## 11. Navigation & Context Requirements

### REQ-TS-90: Workspace switcher integration

Team Space must consume the shared app shell's Workspace context bar. Switching Workspace in the context bar must re-fetch Team Space data for the new Workspace without a full page reload.

> Source: PRD §15.2 — fixed context bar

### REQ-TS-91: Deep links

All card navigation points must produce deep-linkable URLs (e.g., `/team?workspaceId=ws-default-001&filter=...`), so users can share and bookmark Workspace-scoped views.

### REQ-TS-92: Breadcrumb and back navigation

Navigating from Team Space to Project Space or Requirement Management must register a breadcrumb trail so users can return with one click.

---

## 12. AI Command Panel Requirements

### REQ-TS-100: Page context projection

Team Space must project its context (current Workspace id, visible risk set, pipeline counters) to the right-side AI Command Panel.

> Source: PRD §15.4 — 右侧 AI 指令面板

### REQ-TS-101: Suggested actions

The AI Command Panel must surface Team-Space-appropriate suggestions, e.g.:

- "Summarize this week's governance posture"
- "Identify the top 3 risks blocking delivery"
- "Explain why Spec coverage dropped in Project X"

### REQ-TS-102: Skill execution records

All AI-assisted answers rendered on Team Space cards must be recorded as Skill Executions with input/output summary, timestamp, evidence references.

> Source: PRD §14

---

## 13. Governance & Audit Requirements

### REQ-TS-110: Read-only governance surface

All governance actions initiated from Team Space (approve, escalate) must pass through the platform governance pipeline with full audit capture. Team Space is not a back-door governance channel.

> Source: PRD §16.2

### REQ-TS-111: Audit trail on navigation-triggered skill executions

Any AI skill executed via Team Space navigation (e.g., "explain why") must record who / when / which skill / input / output / evidence.

> Source: PRD §14, §16.2

---

## 14. Workspace Isolation Requirements

### REQ-TS-120: Strict Workspace scoping

Team Space must hard-scope all data, metrics, counts, and navigation to the currently-active Workspace. Cross-workspace leakage is a defect.

> Source: PRD §9.2

### REQ-TS-121: Compatibility-mode handling

Workspaces running in compatibility mode (no Primary SNOW Group) must render all cards correctly, substituting null-safe displays where SNOW Group context is missing.

> Source: PRD §8.1

---

## 15. Visual and Experience Requirements

### REQ-TS-130: Card-based modular layout

Team Space must follow the shared design system's Widget/Card pattern with 8-card default layout.

> Source: PRD §15.3

### REQ-TS-131: Tactical Command aesthetic

Visual density and enterprise-grade styling must follow the "Tactical Command" aesthetic. Critical risks use the crimson accent. Health states use the standard green / yellow / red LED palette.

> Source: PRD §15.5, design.md §2

### REQ-TS-132: States

The page must support:

- Normal (Workspace loaded, cards populated)
- Loading (per-card skeletons)
- Empty (new Workspace with no projects / no members)
- Error (API failure per card — isolated, not page-wide)
- Partial (some cards loaded, others failed — per-card retry)

### REQ-TS-133: Per-card error isolation

A failure loading one card must not block the other cards. Each card manages its own loading / error / empty / ready state.

> Source: consistency with `requirement` and `dashboard` slice standards

### REQ-TS-134: SDLC chain strip

The top of the Team Space page must display a compact 11-node SDLC main-chain strip scoped to the Workspace, visibly emphasizing Spec.

> Source: PRD §11.2, §13.1, §7.6

---

## 16. Non-Functional Requirements

### REQ-TS-140: Frontend Phase A uses mocked data

Phase A (frontend) must render with static or mocked Workspace data. Backend integration arrives in Phase B.

### REQ-TS-141: Phase B backend API

Phase B provides real backend REST API endpoints for the Team Space aggregate view, per-card slices, and navigation deep-link resolution.

### REQ-TS-142: Performance

Team Space must first-paint within 300ms of receiving the Workspace id. Individual cards must hydrate within 500ms under typical Workspace volumes (< 20 members, < 30 projects, < 50 active risks).

### REQ-TS-143: Build must succeed

`npm run dev`, `npm run build`, and backend `./mvnw verify` must all succeed after implementation.

### REQ-TS-144: Flyway migrations

Any Team Space-specific schema addition (e.g., workspace-snapshot materialized views, metric caches) must land as a Flyway migration following the `V{n}__{description}.sql` convention. No `ddl-auto: update`.

> Source: CLAUDE.md rule #4

---

## 17. Out of Scope

- Editing templates, policies, AI config (Platform Center owns editing)
- Workspace admin (create / delete / rename)
- Member invitation / permission grants (Access Management)
- Project creation / archiving (Project Space / Project Management)
- Requirement detail editing (Requirement Management)
- Incident detail editing (Incident Management)
- Historical reporting / export (Report Center)
- Cross-workspace aggregation (Dashboard)
- Real-time WebSocket push for member activity (V1 uses on-load + refresh button)
- JIRA / ServiceNow member / project sync (future integration)
- Mobile-optimized layout (desktop-first per PRD §1)

---

## 18. Traceability

| Requirement | PRD Section |
|-------------|-------------|
| REQ-TS-01 | §11.2 |
| REQ-TS-02 | §11.2 |
| REQ-TS-03 | §6.2, §6.3, §6.5 |
| REQ-TS-04 | §7.8 |
| REQ-TS-05 | §7.4 |
| REQ-TS-06 | §9.3, §11.12 |
| REQ-TS-10 | §8.1, §11.2 |
| REQ-TS-11 | §11.2 |
| REQ-TS-12 | §8.1 |
| REQ-TS-20 | §11.2 |
| REQ-TS-21 | §9.3 |
| REQ-TS-22 | §11.12 |
| REQ-TS-30 | §11.2 |
| REQ-TS-31 | §11.2 |
| REQ-TS-32 | §11.12 |
| REQ-TS-40 | §11.2 |
| REQ-TS-41 | §9.3, §11.2 |
| REQ-TS-42 | §11.2 |
| REQ-TS-50 | §11.2 |
| REQ-TS-51 | §11.2 |
| REQ-TS-52 | §11.2, §11.4 |
| REQ-TS-53 | §13.1, §7.6 |
| REQ-TS-60 | §4.4, §11.2 |
| REQ-TS-61 | §11.2 |
| REQ-TS-62 | §10 |
| REQ-TS-70 | §11.2 |
| REQ-TS-71 | §15.5 |
| REQ-TS-72 | §11.2 |
| REQ-TS-80 | §11.2 |
| REQ-TS-81 | §11.2 |
| REQ-TS-82 | §11.3 |
| REQ-TS-90 | §15.2 |
| REQ-TS-91 | — |
| REQ-TS-92 | — |
| REQ-TS-100 | §15.4 |
| REQ-TS-101 | §15.4 |
| REQ-TS-102 | §14 |
| REQ-TS-110 | §16.2 |
| REQ-TS-111 | §14, §16.2 |
| REQ-TS-120 | §9.2 |
| REQ-TS-121 | §8.1 |
| REQ-TS-130 | §15.3 |
| REQ-TS-131 | §15.5 |
| REQ-TS-132 | — |
| REQ-TS-133 | — |
| REQ-TS-134 | §11.2, §13.1 |
| REQ-TS-140 | — |
| REQ-TS-141 | — |
| REQ-TS-142 | — |
| REQ-TS-143 | — |
| REQ-TS-144 | CLAUDE.md #4 |
