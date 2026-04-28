# Dashboard / Control Tower Requirements

## Purpose

This document extracts the requirements relevant to the Dashboard / Control Tower
page from the full PRD (V0.9). It defines **what** the dashboard must deliver,
serving as the upstream input for the spec, architecture, design, and task documents.

## Source

All requirements below are derived from:

- [agentic_sdlc_control_tower_prd_v0.9.md](agentic_sdlc_control_tower_prd_v0.9.md)

Section references use the format `PRD §N.N`.

---

## 1. Scope

The Dashboard / Control Tower is the main landing page for the product. It covers:

- SDLC chain health visualization
- delivery efficiency metrics
- quality and testing indicators
- release and incident status
- AI participation and value demonstration
- governance trust indicators
- recent key actions stream
- value proof narrative

It does **not** cover:

- detailed page-specific drill-downs (those belong to downstream pages)
- historical report generation and export (that belongs to Report Center)
- AI management and skill configuration (that belongs to AI Center)
- team-level operational details (that belongs to Team Space)

---

## 2. Product Context Requirements

### REQ-DASH-01: Main landing page

The Dashboard is the product's primary entry point and main landing page. It
must simultaneously serve management viewers, team leads, and daily operational
users.

> Source: PRD §11.1 — "首页是产品展示与运营指挥的主入口"

### REQ-DASH-02: Three audience layers

The dashboard must serve three distinct audience layers:

- **Management / visitors**: value demonstration and AI proof narrative
- **Team leads / delivery managers**: cross-stage health and risk signals
- **Operational users**: recent actions, current status, quick navigation

> Source: PRD §11.1 — "它应同时服务管理者、团队负责人和日常用户"

### REQ-DASH-03: Operational, not promotional

The dashboard must feel like a real-time operational control surface, not a
marketing page or a collection of disconnected charts.

> Source: PRD §11.1 — "首页不应只是图表集合，而要体现一条完整的软件交付故事线"

---

## 3. SDLC Chain Requirements

### REQ-DASH-10: Full 11-node SDLC chain

The dashboard must display the complete 11-node SDLC chain:

**Requirement → User Story → Spec → Architecture → Design → Tasks → Code → Test → Deploy → Incident → Learning**

> Source: PRD §11.1, §13.1

### REQ-DASH-11: Spec highlighted as execution hub

Within the SDLC chain, Spec must be visually highlighted as the execution hub.
It should not blend into the chain as an equal peer node.

> Source: PRD §13.1 — "突出 Spec"

### REQ-DASH-12: Per-stage health indication

Each node in the SDLC chain should convey a health or status signal (e.g.,
healthy, warning, critical, inactive) so the user can identify bottleneck
stages at a glance.

> Source: PRD §11.1 — "交付节奏", "需求流转"

### REQ-DASH-13: Chain as story, not decoration

The SDLC chain must tell a delivery story — it should show where work
currently concentrates, where bottlenecks are, and how work flows through
the pipeline.

> Source: PRD §11.1 — "体现一条完整的软件交付故事线"

---

## 4. Delivery Metrics Requirements

### REQ-DASH-20: Delivery rhythm metrics

The dashboard must display delivery efficiency metrics including:

- Lead Time (requirement to production)
- Deployment Frequency
- Iteration Completion Rate
- Bottleneck stage identification

> Source: PRD §11.1 — "交付节奏", §17.1

### REQ-DASH-21: Requirement flow status

The dashboard must show the current state of requirement flow: how many
requirements are in each stage, what is blocked, and what is progressing.

> Source: PRD §11.1 — "需求流转"

---

## 5. Quality and Testing Requirements

### REQ-DASH-30: Quality metrics

The dashboard must display quality indicators including:

- Build Success Rate
- Test Pass Rate
- Defect Density
- Spec Coverage

> Source: PRD §11.1 — "测试与质量", §17.2

---

## 6. Release and Incident Requirements

### REQ-DASH-40: Release and incident status

The dashboard must display release and incident indicators including:

- Active Incident Count and Severity
- Change Failure Rate
- MTTR (Mean Time to Resolve)
- Recent Deployments and their status

> Source: PRD §11.1 — "发布与事故", §17.3

---

## 7. AI Participation Requirements

### REQ-DASH-50: AI participation visibility

The dashboard must make AI participation visible across SDLC stages:

- AI Usage Rate per stage
- AI Autonomy Level distribution
- Suggestion Adoption Rate
- Auto-Execution Success Rate
- AI Time Saved

> Source: PRD §11.1 — "AI 参与度", §17.5

### REQ-DASH-51: AI as actor, not sidebar

AI participation must be presented as a core operational signal, not as a
sidebar metric. It should be clear that AI is an active participant in the
delivery process.

> Source: PRD §7.4, §21 — "AI 不是在旁边提建议，而是在系统里真正执行工作"

---

## 8. Governance Requirements

### REQ-DASH-60: Governance trust indicators

The dashboard must display governance health:

- Template Reuse Rate
- Configuration Drift status
- Audit Coverage
- Policy Hit Rate

> Source: PRD §11.1 — "治理状态", §17.4

---

## 9. Activity Requirements

### REQ-DASH-70: Recent key actions

The dashboard must show a stream of recent key actions across the workspace,
including who (AI or human) performed the action, when, and the SDLC stage
context.

> Source: PRD §11.1 — "最近关键动作"

### REQ-DASH-71: Value proof narrative

The dashboard must include a value story section that demonstrates the
tangible impact of AI-native SDLC adoption: time saved, quality improved,
incidents prevented.

> Source: PRD §11.1 — "价值证明故事线"

---

## 10. Differentiation Requirements

### REQ-DASH-80: Dashboard vs Report Center

The dashboard is real-time, operational, and story-driven. Report Center is
historical, filterable, exportable, and structured for stakeholder communication.
The dashboard must not attempt to be a report tool.

> Source: PRD §11.1 — "Dashboard 侧重实时运营态势、异常信号和跨阶段故事线"

### REQ-DASH-81: Dashboard vs Team Space

The dashboard is cross-team and cross-project. Team Space is scoped to a
single workspace. The dashboard provides the organizational overlay.

> Source: PRD §11.2 — "Dashboard 面向跨团队、跨项目的全局运营视角"

---

## 11. Visual and Experience Requirements

### REQ-DASH-90: Card-based modular layout

Dashboard content must be organized as modular cards/widgets that can
conceptually be rearranged by role and scenario.

> Source: PRD §15.3

### REQ-DASH-91: High-density, control-tower style

Visual style must be high-density and enterprise-grade, consistent with the
shared shell aesthetic.

> Source: PRD §15.5

### REQ-DASH-92: States

The dashboard must support at minimum:

- Normal (data loaded)
- Loading (data fetching)
- Empty (no data available)
- Error (API failure)
- Partial (some widgets loaded, others failed)

---

## 12. Non-Functional Requirements

### REQ-DASH-100: Frontend Phase A uses mocked data

Phase A (frontend) must render with static or mocked data. Backend
integration is added in Phase B.

### REQ-DASH-101: Dashboard data is workspace-scoped

All dashboard data must be scoped to the current workspace context. The
backend must enforce workspace isolation.

> Source: PRD §9.2

### REQ-DASH-102: Build must succeed

`npm run dev` and `npm run build` must both succeed after dashboard
implementation.

---

## 13. Out of Scope

- Drill-down into individual requirements, incidents, or deployments
- Report generation, filtering, or export
- AI skill configuration or management
- Real-time WebSocket push (V1 uses polling or on-load fetch)
- Role-based widget visibility (V1 shows all widgets to all users)
- Widget drag-and-drop reordering

---

## 14. Traceability

| Requirement | PRD Section |
|-------------|-------------|
| REQ-DASH-01 | §11.1 |
| REQ-DASH-02 | §11.1 |
| REQ-DASH-03 | §11.1 |
| REQ-DASH-10 | §11.1, §13.1 |
| REQ-DASH-11 | §13.1 |
| REQ-DASH-12 | §11.1 |
| REQ-DASH-13 | §11.1 |
| REQ-DASH-20 | §11.1, §17.1 |
| REQ-DASH-21 | §11.1 |
| REQ-DASH-30 | §11.1, §17.2 |
| REQ-DASH-40 | §11.1, §17.3 |
| REQ-DASH-50 | §11.1, §17.5 |
| REQ-DASH-51 | §7.4, §21 |
| REQ-DASH-60 | §11.1, §17.4 |
| REQ-DASH-70 | §11.1 |
| REQ-DASH-71 | §11.1 |
| REQ-DASH-80 | §11.1 |
| REQ-DASH-81 | §11.2 |
| REQ-DASH-90 | §15.3 |
| REQ-DASH-91 | §15.5 |
| REQ-DASH-92 | — |
| REQ-DASH-100 | — |
| REQ-DASH-101 | §9.2 |
| REQ-DASH-102 | — |
