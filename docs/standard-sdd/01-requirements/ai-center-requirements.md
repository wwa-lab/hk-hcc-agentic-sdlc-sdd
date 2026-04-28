# AI Center Requirements

## Purpose

This document extracts the requirements relevant to the **AI Center** page from the full PRD (V0.9). It defines **what** the AI Center must deliver in V1 (MVP lightweight), serving as the upstream input for the spec, architecture, design, and task documents.

## Source

All requirements below are derived from:

- [agentic_sdlc_control_tower_prd_v0.9.md](agentic_sdlc_control_tower_prd_v0.9.md)

Section references use the format `PRD §N.N`.

---

## 1. Scope

The AI Center is described in the PRD as **the primary surface that treats AI as a platform capability**, not a per-page sidebar component. It is the global home for the AI capability layer — where Skills, Agents, Policies, and run history are registered, governed, and evaluated.

> Source: PRD §11.11 — "AI Center 是'把 AI 作为平台能力'而不是'页面附属组件'的主要体现"

### 1.1 V1 Scope (MVP Lightweight per PRD §18.1)

It covers:

- **Skill Catalog** — browsable directory of registered AI skills (metadata, status, owner, autonomy level)
- **Run History** — global timeline of skill executions across the workspace, with filters and drill-down
- **Adoption Metrics Summary** — usage rate, adoption rate, auto-exec success rate, stage coverage
- **Policy View (read-only)** — current autonomy levels and approval policies per skill
- **Evidence & Audit Entry Points** — pointers to evidence and audit records linked to each execution

### 1.2 Explicitly Out of Scope in V1

It does **not** cover:

- Prompt / Policy **editor** (V1 is read-only view; editing deferred to a later slice)
- Agent registry UI (beyond Skill level; Agents are a deferred concept)
- Training / fine-tuning flows
- Per-page AI Command Panel UI (that is the page-scoped projection; this slice defines the **shared models** it consumes but not its UI)
- Historical effectiveness analytics beyond the V1 summary metrics (belongs to Report Center)
- Cross-workspace skill federation

> Confirmation: scope matches the "AI Center（轻量版）" line in PRD §18.1 MVP list.

### 1.3 Relationship to the Right-Side AI Command Panel

Per PRD §11.11 and §15.4, AI Center and the AI Command Panel share **the same backend models** (Skill, SkillExecution, Policy, Evidence, Audit) but have **separate UIs**:

- **AI Center** — global management, catalog, policy, run history, metrics
- **AI Command Panel** — page-scoped projection of the same models, constrained to the current page's context

This slice **defines and owns** the shared Skill / SkillExecution / Policy / Evidence / Audit-link domain models in the backend. The AI Command Panel (out of scope here) will consume them.

---

## 2. Product Context Requirements

### REQ-AIC-01: AI as platform capability, not per-page widget

AI Center must present AI capabilities as a first-class, centrally-governed platform surface — not a loose collection of per-page chat components.

> Source: PRD §11.11

### REQ-AIC-02: Shared backend model with AI Command Panel

AI Center and the right-side AI Command Panel share the same Skill Execution, Policy, Evidence, and Audit models. AI Center owns the global management UI; the Command Panel owns the page-context projection.

> Source: PRD §11.11, §15.4 — "它应复用全局 AI 能力、Skill Execution、Policy 与 Evidence 模型"

### REQ-AIC-03: Three audience layers

The page must serve:

- **Platform Admin / AI Governance Owner** — registry curation, policy oversight, adoption posture
- **Team Lead / Delivery Manager** — skill effectiveness in their workspace, autonomy posture
- **Individual Contributor / Auditor** — skill lookup, run history for a specific skill or trigger

> Source: PRD §6.x personas, §16.1 roles

---

## 3. Skill Catalog Requirements

### REQ-AIC-10: Skill catalog list view

AI Center must display a browsable catalog of all registered AI skills in the current workspace. Each list entry must show at minimum:

- Skill key (e.g., `req-to-user-story`, `incident-diagnosis`)
- Human-readable name and category (SDLC stage or runtime stage)
- Status: `active` / `beta` / `deprecated`
- Default autonomy level (see REQ-AIC-41)
- Owner / maintaining team
- Last execution timestamp
- 30-day success rate

> Source: PRD §14 — "所有关键活动都可以被登记为 Skill，并具有执行记录、步骤、状态、触发来源、输入输出、证据和审计轨迹"

### REQ-AIC-11: Skill categories

Skills must be organized by category matching the PRD §14 taxonomy:

- **Delivery-phase skills**: `req-to-user-story`, `user-story-to-spec`, `spec-to-architecture`, `architecture-to-design`, `design-to-tasks`, `tasks-to-code`, `tasks-to-implementation`, `review-code-against-design`, `review-doc-quality`
- **Runtime-phase skills**: `incident-detection`, `incident-correlation`, `incident-diagnosis`, `incident-remediation`, `incident-learning`

> Source: PRD §14

### REQ-AIC-12: Skill detail view

Selecting a skill from the catalog must open a detail view showing:

- Full description and input/output contract summary
- Current policy bindings (autonomy, approval requirements, risk thresholds)
- Recent execution history (last N executions, filterable)
- Aggregate metrics (success rate, avg duration, adoption trend)
- Owning team and change history (linked to audit)

> Source: PRD §11.11, §14

### REQ-AIC-13: Filter and search

Users must be able to filter skills by category, status, autonomy level, and owner, and search by skill key or name.

---

## 4. Run History Requirements

### REQ-AIC-20: Global run history timeline

AI Center must display a global, reverse-chronological run history of skill executions in the current workspace. Each row must show:

- Execution ID
- Skill key and name
- Trigger source (which page / which action / which user or automation)
- Start and end timestamps
- Duration
- Status (`running`, `succeeded`, `failed`, `pending_approval`, `rejected`, `rolled_back`)
- Outcome summary (one-line)
- Links to evidence and audit record

> Source: PRD §11.11, §14 — "运行历史"; "执行记录、步骤、状态、触发来源、输入输出、证据和审计轨迹"

### REQ-AIC-21: Filters

Run history must support filtering by:

- Skill (single or multiple)
- Status
- Trigger source page (e.g., Incident, Requirement)
- Time range (last 24h / 7d / 30d / custom)
- Triggered by AI vs. by human

### REQ-AIC-22: Run detail view

Selecting a run must expose the full execution record:

- Input summary (redacted if sensitive)
- Step breakdown (for multi-step skills)
- Output / result
- Policy decision trail (which policy rules evaluated, which gate it passed/was held at)
- Evidence attachments (links to referenced artifacts)
- Audit record link

> Source: PRD §14, §16.2

---

## 5. Adoption & Effectiveness Metrics Requirements

### REQ-AIC-30: Adoption summary cards

AI Center must display a top-level metrics strip covering the following (workspace-scoped, 30-day window by default):

- **AI usage rate** — % of eligible SDLC events where an AI skill was invoked
- **Adoption rate** — % of AI suggestions that were accepted / executed
- **Auto-execution success rate** — % of auto-mode executions that succeeded without human intervention
- **Time saved estimate** — aggregate time saved via AI (hours)
- **Stage coverage** — number of SDLC stages covered by at least one active skill

> Source: PRD §17.5 — "AI 使用率、自治级别、建议采纳率、自动执行成功率、AI 节省工时、AI 参与阶段覆盖率"

### REQ-AIC-31: Trend indicators

Each metric must show a trend arrow (↑ / ↓ / →) and delta vs. the previous equivalent window.

### REQ-AIC-32: Stage coverage visualization

Stage coverage must be visualized against the canonical 11-node SDLC chain (Requirement → User Story → Spec → Architecture → Design → Tasks → Code → Test → Deploy → Incident → Learning), indicating which stages have active AI skills.

> Source: PRD §10 — 11-node chain

---

## 6. Policy View Requirements

### REQ-AIC-40: Read-only policy view

AI Center must display the current policy configuration per skill in a read-only view. V1 does **not** include a policy editor.

> Source: PRD §11.11, §12.5 — "Policy & Governance 必须作为一等能力"; V1 scoping per §18.1

### REQ-AIC-41: Autonomy level display

Each skill must display its current autonomy level from the set:

- `L0-Manual` — AI provides suggestions only
- `L1-Assist` — AI proposes; human approves before execution
- `L2-Auto-with-approval` — AI executes; high-risk actions require approval
- `L3-Auto` — AI executes without approval; human governs via audit

> Source: PRD §7.5, §12.5

### REQ-AIC-42: Approval requirement indicators

For skills in `L1` or `L2`, the view must indicate which action types within that skill require approval, and which role(s) are authorized to approve.

> Source: PRD §16.1

---

## 7. Evidence and Audit Integration Requirements

### REQ-AIC-50: Evidence link-through

Each skill execution record must link to its evidence set (inputs consulted, artifacts produced). V1 renders a list of links; the evidence browser UI is deferred.

> Source: PRD §11.11, §16.2

### REQ-AIC-51: Audit trail integration

All AI Center actions (viewing policy, filtering runs) are **read-only** in V1 so do not themselves generate audit records. However, every skill execution already creates an audit record; AI Center must provide a link-through to that record.

> Source: PRD §16.2 — "Skill 触发与执行" as mandatory audit event

---

## 8. Integration with Other Pages

### REQ-AIC-60: Navigation in from other pages

Other pages (Dashboard "Learning" stage, Incident execution timeline, Requirement AI-assist entries) must be able to deep-link to AI Center and land on:

- A specific skill detail view, or
- A specific run detail view

> Source: Dashboard slice already routes its `learning` stage to `/ai-center` — confirmed in `codex-dashboard-phase-b.md`.

### REQ-AIC-61: Navigation out to source pages

From a run detail in AI Center, users must be able to navigate back to the source page where the skill was triggered (e.g., an incident page, a requirement detail) when such a source exists.

---

## 9. Visual and Experience Requirements

### REQ-AIC-70: Card-based modular layout

AI Center content must be organized as modular cards following the shared design system (metrics strip → skill catalog card → run history card, with detail views in-page or as drill-down routes).

> Source: PRD §15.3

### REQ-AIC-71: High-density, control-tower style

Visual style must be high-density and enterprise-grade, matching the "Tactical Command" aesthetic used by Dashboard and Incident. Skill status and autonomy level use the shared status color tokens (operational/warning/critical/ok).

> Source: PRD §15.5, `visual-design-system.md` §2

### REQ-AIC-72: States

AI Center must support:

- Normal (skills + runs present)
- Loading (data fetching)
- Empty (no skills registered yet — instructional state pointing to Platform Center registration)
- Error (API failure — at card level and page level)
- Partial (metrics loaded, catalog failed, etc. — per-card error isolation)

### REQ-AIC-73: Real-time feel

Run history should feel live/updating. V1 uses on-load fetch plus a manual refresh control; polling or server-sent events are out of scope for V1.

---

## 10. Non-Functional Requirements

### REQ-AIC-80: Frontend Phase A uses mocked data

Phase A (frontend) must render with static or mocked data. Backend integration is added in Phase B. Both phases are delivered via **Codex** for this slice (not Gemini).

### REQ-AIC-81: AI Center data is workspace-scoped

All skill catalog entries, run history, policies, and metrics must be scoped to the current workspace context propagated by the shared app shell.

> Source: PRD §9.2, §16.3

### REQ-AIC-82: Shared backend models are first-class

The `Skill`, `SkillExecution`, `Policy`, and `Evidence` entities must live under `backend/src/main/java/com/sdlctower/domain/ai-center/` (package-by-feature per CLAUDE.md Lesson #3) and be persisted via Flyway migrations (per Lesson #4).

### REQ-AIC-83: Build must succeed

`npm run dev` and `npm run build` must both succeed after implementation. `./mvnw test` must pass for all new backend tests.

### REQ-AIC-84: Performance

- Catalog list must render within 500ms of API response (≤100 skills expected in V1)
- Run history must paginate; page size default 50, max 200
- Metrics summary must return in a single aggregated API call (no N+1)

---

## 11. Out of Scope

- Prompt editor, Policy editor, Agent registry UI
- Skill versioning UI (data model supports it; UI deferred)
- Skill cost / token accounting display
- AI training data management
- Cross-workspace skill federation
- Real-time WebSocket push for run status
- Export of run history to CSV / PDF (belongs to Report Center)
- Role-based permissions UI (V1 uses workspace-scoped read access for all members)

---

## 12. Traceability

| Requirement | PRD Section |
|-------------|-------------|
| REQ-AIC-01 | §11.11 |
| REQ-AIC-02 | §11.11, §15.4 |
| REQ-AIC-03 | §6.x, §16.1 |
| REQ-AIC-10 | §11.11, §14 |
| REQ-AIC-11 | §14 |
| REQ-AIC-12 | §11.11, §14 |
| REQ-AIC-13 | — (UX-derived) |
| REQ-AIC-20 | §11.11, §14 |
| REQ-AIC-21 | — (UX-derived) |
| REQ-AIC-22 | §14, §16.2 |
| REQ-AIC-30 | §17.5 |
| REQ-AIC-31 | §17.5 |
| REQ-AIC-32 | §10, §17.5 |
| REQ-AIC-40 | §11.11, §12.5, §18.1 |
| REQ-AIC-41 | §7.5, §12.5 |
| REQ-AIC-42 | §16.1 |
| REQ-AIC-50 | §11.11, §16.2 |
| REQ-AIC-51 | §16.2 |
| REQ-AIC-60 | Dashboard slice |
| REQ-AIC-61 | — (UX-derived) |
| REQ-AIC-70 | §15.3 |
| REQ-AIC-71 | §15.5, `visual-design-system.md` §2 |
| REQ-AIC-72 | — |
| REQ-AIC-73 | — |
| REQ-AIC-80 | — |
| REQ-AIC-81 | §9.2, §16.3 |
| REQ-AIC-82 | CLAUDE.md Lessons #3, #4 |
| REQ-AIC-83 | — |
| REQ-AIC-84 | — |
