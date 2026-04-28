# Incident Management Requirements

## Purpose

This document extracts the requirements relevant to the Incident Management
page from the full PRD (V0.9). It defines **what** the incident page must
deliver, serving as the upstream input for the spec, architecture, design,
and task documents.

## Source

All requirements below are derived from:

- [agentic_sdlc_control_tower_prd_v0.9.md](agentic_sdlc_control_tower_prd_v0.9.md)

Section references use the format `PRD §N.N`.

---

## 1. Scope

The Incident Management page is described in the PRD as **the product's most
differentiated page** — an AI-native operational command center, not a
traditional incident ticket view.

It covers:

- Incident list with severity, status, and handler (AI / Human / Hybrid)
- AI/Skill execution timeline per incident
- AI diagnosis and reasoning display
- AI actions taken with audit trail
- Human governance: pending approvals, overrides, manual escalations
- Related SDLC chain traceability (Requirement → ... → Incident → Learning)
- AI learning and prevention recommendations
- AI Command Panel (page-context projection)

It does **not** cover:

- Global AI skill management (that belongs to AI Center)
- Historical incident reports and export (that belongs to Report Center)
- Platform-level policy and template editing (that belongs to Platform Center)
- Deployment pipeline management (that belongs to Deployment Management)

---

## 2. Product Context Requirements

### REQ-INC-01: AI-native command center, not a ticket tracker

The Incident page is not a traditional ITSM ticket list. It is an AI-native
operational command center where AI is the primary executor and humans govern.

> Source: PRD §11.10 — "不是传统的 production incident ticket 页，而是一个 AI-native 的运行指挥中心"

### REQ-INC-02: Inverted execution model

The incident lifecycle follows an AI-first flow:

**Signal → AI Detection → AI Diagnosis → AI Action → Human Governance → Resolution → Learning**

This is the inverse of the traditional flow where humans do everything and AI
only suggests.

> Source: PRD §11.10 — "AI 是事故处理的主要执行者，人主要承担审批、覆盖、策略治理和风险确认职责"

### REQ-INC-03: Three audience layers

The page must serve:

- **SRE / DevOps**: real-time incident status, AI diagnosis, action approval
- **Team leads / Delivery managers**: incident impact, governance posture, risk
- **Management**: AI value in incident handling, resolution speed, prevention

> Source: PRD §6.5, §6.2, §6.6

---

## 3. Incident Header Requirements

### REQ-INC-10: Incident header with AI handling context

Each incident must display a header containing at minimum:

- Incident ID (e.g., INC-0422)
- Title / summary
- Priority / severity (P1–P4)
- Current status (with state machine transitions)
- Handler type: **AI / Human / Hybrid**
- Current autonomy level
- Current control mode: **Auto / Approval / Manual**
- Timestamps: detected, acknowledged, resolved
- Duration / age

> Source: PRD §11.10 — "事故由谁处理：AI / Human / Hybrid", "当前自治级别", "当前控制模式"

### REQ-INC-11: Incident status state machine

Incident status must follow a defined state machine:

**Detected → AI Investigating → AI Diagnosed → Action Proposed → Pending Approval → Executing → Resolved → Learning → Closed**

With override states: **Escalated (to human)**, **Manual Override**

> Source: PRD §11.10 flow definition

---

## 4. AI Execution Timeline Requirements

### REQ-INC-20: AI/Skill execution timeline

The page must show a chronological timeline of all AI skill executions
related to the current incident, including:

- Skill name (e.g., incident-detection, incident-diagnosis, incident-remediation)
- Execution start/end timestamps
- Input summary
- Output / result summary
- Status (running, completed, failed, pending approval)
- Evidence references

> Source: PRD §14 — "所有关键活动都可以被登记为 Skill，并具有执行记录、步骤、状态、触发来源、输入输出、证据和审计轨迹"

### REQ-INC-21: Skill types for incident lifecycle

The system should model incident handling as a series of executable skills:

- `incident-detection` — signal correlation and anomaly identification
- `incident-correlation` — linking related signals and changes
- `incident-diagnosis` — root cause analysis and evidence gathering
- `incident-remediation` — proposed or executed fix actions
- `incident-learning` — post-incident analysis and prevention recommendations

> Source: PRD §14

---

## 5. AI Diagnosis & Reasoning Requirements

### REQ-INC-30: AI diagnosis panel

The page must display AI diagnostic reasoning including:

- Detected signals and their sources
- Correlation with recent changes (deploys, config changes, code merges)
- Root cause hypothesis with confidence level
- Evidence chain supporting the diagnosis
- Affected components and blast radius

> Source: PRD §11.10 — "AI Diagnosis & Reasoning"

### REQ-INC-31: Real-time diagnosis feed

AI diagnosis should be displayed as a real-time feed (log-style) showing
the AI's investigation steps, findings, and intermediate conclusions.

> Source: existing placeholder — "AI_DIAGNOSIS_FEED" pattern

---

## 6. AI Actions Requirements

### REQ-INC-40: AI actions taken

The page must display all actions taken (or proposed) by AI, including:

- Action description
- Action type (automated / requires approval)
- Execution status (pending, approved, rejected, executed, rolled back)
- Timestamp
- Impact assessment
- Rollback capability

> Source: PRD §11.10 — "AI 已经做了哪些动作"

### REQ-INC-41: Actions pending human approval

The page must clearly highlight which actions are awaiting human approval,
with approve/reject controls for authorized users.

> Source: PRD §11.10 — "哪些动作仍待人工审批"

---

## 7. Human Governance Requirements

### REQ-INC-50: Human governance section

The page must have a dedicated governance section showing:

- Pending approvals with action details
- Approval/rejection history with reasons
- Manual overrides taken
- Escalation history
- Policy constraints that triggered approval requirements

> Source: PRD §11.10 — "Human Governance"

### REQ-INC-51: Autonomy level display

The current AI autonomy level must be visible and explain which actions
AI can take automatically vs. which require human approval.

> Source: PRD §7.5 — "人是治理者，不是默认执行者"

### REQ-INC-52: Audit trail

All governance actions (approve, reject, escalate, override) must be
recorded with who, when, and why — feeding into the platform audit system.

> Source: PRD §16.2 — "审批与拒绝", "事故处置与复盘"

---

## 8. SDLC Chain Traceability Requirements

### REQ-INC-60: Related SDLC chain

Each incident must show its relationship to the SDLC chain:

- Which **Requirement** / **Spec** / **Design** / **Code Change** / **Test** /
  **Deployment** is related to or caused the incident
- Navigation links to the related objects in their respective pages

> Source: PRD §11.10 — "事故与 Requirement / Spec / Design / Code / Test / Deploy 的关系"

### REQ-INC-61: Compressed chain with Spec visible

When displaying the SDLC chain in incident context, the chain may be
compressed (not all 11 nodes), but:

- Spec node must always be visible
- Collapsed nodes must be indicated
- User must be able to expand to the full chain

> Source: PRD §13.1 — "可以折叠部分节点，但必须显示 Spec 节点"

---

## 9. AI Learning & Prevention Requirements

### REQ-INC-70: AI learning section

After resolution, the page must show what the AI learned, including:

- Root cause confirmed
- Pattern identified
- Prevention recommendations
- Suggested policy/configuration changes
- Knowledge base entry created

> Source: PRD §11.10 — "AI 学到了什么、如何防止再次发生"

### REQ-INC-71: Learning feeds back into platform

AI learning should reference how it feeds back into the platform:
prevention rules, updated detection patterns, and recommended spec/design
changes.

> Source: PRD §14 — "incident-learning" skill

---

## 10. Incident List Requirements

### REQ-INC-80: Incident list view

The page must support a list/table view of all incidents in the current
workspace, with:

- Filtering by priority, status, handler type, date range
- Sorting by severity, recency, duration
- Quick-glance severity distribution
- Active vs. resolved tabs or filter

### REQ-INC-81: Incident detail view

Clicking an incident from the list opens the full incident detail view
with all sections described above (header, timeline, diagnosis, actions,
governance, chain, learning).

---

## 11. Visual and Experience Requirements

### REQ-INC-90: Card-based modular layout

Incident page content must be organized as modular cards following the
shared design system.

> Source: PRD §15.3

### REQ-INC-91: High-density, control-tower style

Visual style must be high-density and enterprise-grade, with the "Tactical
Command" aesthetic. Incident severity should use the crimson accent for
critical items.

> Source: PRD §15.5, design.md §2

### REQ-INC-92: States

The incident page must support:

- Normal (data loaded, incidents present)
- Loading (data fetching)
- Empty (no incidents — this is a good state)
- Error (API failure)
- Partial (some sections loaded, others failed)

### REQ-INC-93: Real-time feel

The AI diagnosis feed and action timeline should feel live/updating,
even if V1 uses polling rather than WebSocket.

---

## 12. Non-Functional Requirements

### REQ-INC-100: Frontend Phase A uses mocked data

Phase A (frontend) must render with static or mocked data. Backend
integration is added in Phase B.

### REQ-INC-101: Incident data is workspace-scoped

All incident data must be scoped to the current workspace context.

> Source: PRD §9.2

### REQ-INC-102: Build must succeed

`npm run dev` and `npm run build` must both succeed after implementation.

---

## 13. Out of Scope

- Real-time WebSocket push (V1 uses polling or on-load fetch)
- PagerDuty / OpsGenie / ServiceNow bi-directional sync
- Automated rollback execution without human approval (V1 requires approval)
- AI model training or fine-tuning from incidents
- Cross-workspace incident correlation
- Mobile-optimized incident view

---

## 14. Traceability

| Requirement | PRD Section |
|-------------|-------------|
| REQ-INC-01 | §11.10 |
| REQ-INC-02 | §11.10 |
| REQ-INC-03 | §6.2, §6.5, §6.6 |
| REQ-INC-10 | §11.10 |
| REQ-INC-11 | §11.10 |
| REQ-INC-20 | §14 |
| REQ-INC-21 | §14 |
| REQ-INC-30 | §11.10 |
| REQ-INC-31 | — (existing prototype) |
| REQ-INC-40 | §11.10 |
| REQ-INC-41 | §11.10 |
| REQ-INC-50 | §11.10 |
| REQ-INC-51 | §7.5 |
| REQ-INC-52 | §16.2 |
| REQ-INC-60 | §11.10 |
| REQ-INC-61 | §13.1 |
| REQ-INC-70 | §11.10 |
| REQ-INC-71 | §14 |
| REQ-INC-80 | — |
| REQ-INC-81 | — |
| REQ-INC-90 | §15.3 |
| REQ-INC-91 | §15.5 |
| REQ-INC-92 | — |
| REQ-INC-93 | — |
| REQ-INC-100 | — |
| REQ-INC-101 | §9.2 |
| REQ-INC-102 | — |
