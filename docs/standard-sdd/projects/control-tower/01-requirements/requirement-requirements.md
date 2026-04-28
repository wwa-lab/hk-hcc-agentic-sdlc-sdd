# Requirement Management Requirements

## Purpose

This document extracts the requirements relevant to the Requirement Management
page from the full PRD (V0.9). It defines **what** the requirement management
page must deliver, serving as the upstream input for the spec, architecture,
design, and task documents.

## Source

All requirements below are derived from:

- [agentic_sdlc_control_tower_prd_v0.9.md](agentic_sdlc_control_tower_prd_v0.9.md)

Section references use the format `PRD §N.N`.

---

## 1. Scope

The Requirement Management page is described in the PRD as the entry point for
the Spec Driven Development (SDD) chain — the place where requirements are
captured, classified, prioritized, and decomposed into User Stories and Specs.

It covers:

- Requirement list with priority, status, category, and business context
- Requirement detail with acceptance criteria and business background
- Kanban view for requirement lifecycle management
- Priority matrix visualization
- User Story derivation from requirements (parent-child relationship)
- Spec generation entry and linkage (Spec as first-class citizen)
- Full chain traceability: Requirement → User Story → Spec → downstream objects
- Reverse traceability from Incident back to originating Requirement
- AI-assisted requirement analysis, impact analysis, and gap detection
- AI Command Panel (page-context projection)

It does **not** cover:

- Spec detail editing and versioning (that belongs to Spec Management)
- User Story backlog management and sprint planning (that belongs to User Story Management)
- Global AI skill management (that belongs to AI Center)
- Historical requirement reports and export (that belongs to Report Center)
- Platform-level policy and template editing (that belongs to Platform Center)
- Architecture and design document management (those belong to their respective pages)

---

## 2. Product Context Requirements

### REQ-REQ-01: SDD chain entry point

The Requirement Management page is the starting point of the Spec Driven
Development chain. Requirements are the origin from which User Stories and
Specs are derived. The page must make this chain relationship explicit and
navigable.

> Source: PRD §13 — "Requirement → User Story → Spec → Architecture → Design → Tasks → Code → Test → Deploy → Incident → Learning"

### REQ-REQ-02: Spec as execution hub

Spec is the execution hub of the SDD model. The Requirement page must
prominently surface Spec status and provide direct entry points for Spec
generation from requirements or user stories. Spec must never be buried
or treated as a secondary artifact.

> Source: PRD §7.6 — "Spec 作为执行枢纽", §13 — "Spec 应被高亮标识"

### REQ-REQ-03: Three audience layers

The page must serve:

- **Product managers / Business analysts**: requirement capture, prioritization,
  acceptance criteria, business context
- **Team leads / Delivery managers**: requirement status, decomposition progress,
  spec coverage, chain completeness
- **Management**: requirement pipeline health, AI-assisted analysis value,
  traceability posture

> Source: PRD §6.2, §6.5, §6.6

### REQ-REQ-04: Configuration-driven behavior

The requirement page behavior (available statuses, priority levels, categories,
workflow transitions) must be configuration-driven, not hardcoded.

> Source: PRD §7.8 — "配置驱动"

---

## 3. Requirement List Requirements

### REQ-REQ-10: Requirement list/table view

The page must support a list/table view of all requirements in the current
workspace, with columns including at minimum:

- Requirement ID (e.g., REQ-0042)
- Title / summary
- Priority (Critical / High / Medium / Low)
- Status (Draft / In Review / Approved / In Progress / Delivered / Archived)
- Category / type (Functional / Non-Functional / Technical / Business)
- Assignee
- Derived User Story count
- Linked Spec count and status
- Created date
- Last updated date

### REQ-REQ-11: Filtering and sorting

The list view must support:

- Filtering by priority, status, category, assignee, date range
- Sorting by priority, status, recency, title
- Text search across title and description
- Active vs. completed tabs or filter

### REQ-REQ-12: Bulk operations

The list view should support bulk selection and bulk status transitions
for authorized users.

---

## 4. Requirement Detail Requirements

### REQ-REQ-20: Requirement detail header

Each requirement must display a header containing at minimum:

- Requirement ID
- Title / summary
- Priority with visual indicator (Critical uses crimson accent)
- Current status (with state machine transitions)
- Category / type
- Assignee
- Created and last modified timestamps
- Source (manual entry / imported / AI-generated)

### REQ-REQ-21: Requirement description and business context

The detail view must include:

- Full requirement description (rich text)
- Business context and rationale
- Business value justification
- Stakeholder references
- Related external references (JIRA, Confluence, etc.)

### REQ-REQ-22: Acceptance criteria

Each requirement must support structured acceptance criteria:

- List of testable acceptance criteria items
- Each criterion has a pass/fail status
- Criteria can be linked to test cases downstream

### REQ-REQ-23: Priority matrix positioning

The detail view must show where this requirement sits in the priority
matrix (impact vs. effort or similar 2x2 classification).

### REQ-REQ-24: Requirement status state machine

Requirement status must follow a defined state machine:

**Draft → In Review → Approved → In Progress → Delivered → Archived**

---

## 5. User Story Derivation Requirements

### REQ-REQ-30: Derive User Stories from requirements

The page must provide an explicit action to derive one or more User Stories
from a requirement. This derivation should:

- Create User Story objects linked to the parent requirement
- Pre-populate the story with context from the requirement
- Support AI-assisted story generation (via `req-to-user-story` skill)
- Allow manual story creation with parent linkage

> Source: PRD §14 — skill: `req-to-user-story`

### REQ-REQ-31: Parent-child relationship display

The requirement detail must show all derived User Stories as child objects,
displaying:

- Story ID and title
- Story status
- Linked Spec status (if any)
- Navigation link to the User Story detail

### REQ-REQ-32: Decomposition completeness indicator

Each requirement must show a decomposition completeness indicator:

- How many stories have been derived
- Whether all stories have linked Specs
- Whether any gaps exist (requirement coverage)

---

## 6. Spec Generation & Linkage Requirements

### REQ-REQ-40: Spec generation entry point

The page must provide a prominent entry point for generating a Spec from
a requirement or from a derived User Story. This is a primary workflow,
not a secondary action.

> Source: PRD §7.6, §13 — Spec as execution hub

### REQ-REQ-41: Spec generation via skill

Spec generation should invoke the `user-story-to-spec` skill, producing
a structured Spec with:

- Source tracking (which requirement/story originated it)
- Version number
- State definitions
- Object models
- Interface contracts
- Permission rules
- Exception flows

> Source: PRD §14 — skill: `user-story-to-spec`, §13 — Spec structured contracts

### REQ-REQ-42: Spec status tracking on requirement

Each requirement must surface the status of all linked Specs:

- Spec ID and title
- Spec status (Draft / Review / Approved / Implemented / Verified)
- Spec version
- Navigation link to the Spec detail

### REQ-REQ-43: Spec coverage indicator

The requirement list and detail views must show a Spec coverage indicator —
how many requirements have fully-specified Specs vs. those that are
unspecified or partially specified.

---

## 7. Chain Traceability Requirements

### REQ-REQ-50: Forward chain traceability

Each requirement must show its downstream chain:

**Requirement → User Story → Spec → Architecture → Design → Tasks → Code → Test → Deploy**

With navigation links to each object in the chain.

> Source: PRD §7.7 — "全链路可追溯性", §13

### REQ-REQ-51: Reverse chain traceability

Each requirement must support reverse traceability:

- Which Incidents trace back to this requirement
- Which test failures relate to this requirement
- Which deployments include changes to this requirement

> Source: PRD §11.10 — incident-to-requirement traceability, §13

### REQ-REQ-52: Compressed chain with Spec visible

When displaying the SDD chain in requirement context, the chain may be
compressed (not all 11 nodes), but:

- Spec node must always be visible
- Collapsed nodes must be indicated
- User must be able to expand to the full chain

> Source: PRD §13.1 — "可以折叠部分节点，但必须显示 Spec 节点"

### REQ-REQ-53: Chain completeness health

The page must show an aggregate chain completeness metric:

- How many requirements have complete chains (Requirement → Story → Spec minimum)
- Which requirements have broken or incomplete chains
- Visual indicator (green/yellow/red) for chain health

---

## 8. Kanban View Requirements

### REQ-REQ-60: Kanban board view

The page must support a Kanban board view showing requirements organized
by their lifecycle status columns:

- Draft
- In Review
- Approved
- In Progress
- Delivered
- Archived

### REQ-REQ-61: Kanban interactions

The Kanban board must support:

- Drag-and-drop to change status (subject to state machine rules)
- Priority-based ordering within columns
- Visual priority indicators (color-coded cards)
- Quick-glance counts per column
- Filtering consistent with the list view filters

### REQ-REQ-62: Priority matrix view

The page must support a priority matrix (2x2 grid) view where requirements
are plotted by impact vs. effort (or similar configurable axes), enabling
prioritization decisions.

### REQ-REQ-63: SDD knowledge graph view

The page must support a knowledge graph view that visualizes SDD document
relationships for the active pipeline profile. The first version may derive
nodes and edges from the structured SDD profile/document dependency manifest
and overlay control-plane health metrics such as indexed documents, missing
documents, stale reviews, and aligned requirements. The target production data
source is the structured SDD index repository and Neo4j graph projection.

---

## 9. AI Assistance Requirements

### REQ-REQ-70: AI-powered requirement analysis

The page must support AI-assisted requirement analysis including:

- Requirement quality assessment (completeness, clarity, testability)
- Ambiguity detection and improvement suggestions
- Duplicate and overlap detection across requirements
- Gap analysis against business objectives

> Source: PRD §11.4 — "AI 辅助分析"

### REQ-REQ-71: AI impact analysis

The page must support AI-powered impact analysis:

- What downstream objects are affected if a requirement changes
- Blast radius visualization across the SDD chain
- Dependency conflict detection

### REQ-REQ-72: AI Command Panel integration

The page must integrate with the right-side AI Command Panel, projecting
page-specific context including:

- Current requirement context
- Available AI skills: `req-to-user-story`, `user-story-to-spec`
- Suggested actions based on requirement state
- Natural language query support for requirement-related questions

> Source: PRD §15.4 — "右侧 AI 指令面板"

### REQ-REQ-73: AI skill execution records

All AI skill executions (story generation, spec generation, analysis) must
be recorded with:

- Skill name and version
- Input summary
- Output summary
- Execution status
- Timestamp
- Evidence references

> Source: PRD §14 — "所有关键活动都可以被登记为 Skill，并具有执行记录"

---

## 10. Governance & Audit Requirements

### REQ-REQ-80: Change tracking

All requirement changes must be tracked with:

- Who made the change
- When the change was made
- What was changed (field-level diff)
- Why it was changed (change reason)

> Source: PRD §16.2 — audit and governance

### REQ-REQ-81: Approval flow

Requirement status transitions that cross governance boundaries (e.g.,
Draft → Approved) must support configurable approval flows.

### REQ-REQ-82: Audit trail

All governance actions (approve, reject, cancel, re-open) must be recorded
with who, when, and why — feeding into the platform audit system.

> Source: PRD §16.2

---

## 11. Workspace Isolation Requirements

### REQ-REQ-90: Workspace-scoped data

All requirement data must be scoped to the current workspace context.
Users must not see requirements from other workspaces.

> Source: PRD §9.2

### REQ-REQ-91: Context bar integration

The requirement page must respect the fixed context bar showing
Workspace / Application / SNOW Group / Project / Environment, and filter
data accordingly.

> Source: PRD §15.2

---

## 12. Visual and Experience Requirements

### REQ-REQ-100: Card-based modular layout

Requirement page content must be organized as modular cards following the
shared design system (Widget/Card pattern).

> Source: PRD §15.3 — "卡片化与模块化"

### REQ-REQ-101: High-density, control-tower style

Visual style must be high-density and enterprise-grade, with the "Tactical
Command" aesthetic. Critical priority requirements should use the crimson
accent.

> Source: PRD §15.5, visual-design-system.md §2

### REQ-REQ-102: States

The requirement page must support:

- Normal (data loaded, requirements present)
- Loading (data fetching)
- Empty (no requirements — first-use guidance)
- Error (API failure)
- Partial (some sections loaded, others failed)

### REQ-REQ-103: Responsive card density

Cards should adapt density based on viewport, maintaining readability at
both standard and high-density layouts.

---

## 13. Non-Functional Requirements

### REQ-REQ-110: Frontend Phase A uses mocked data

Phase A (frontend) must render with static or mocked data. Backend
integration is added in Phase B.

### REQ-REQ-111: Phase B backend API

Phase B provides real backend REST API endpoints for requirement CRUD,
story derivation, spec linkage, and chain queries.

### REQ-REQ-112: Build must succeed

`npm run dev` and `npm run build` must both succeed after implementation.

### REQ-REQ-113: Performance

The requirement list must render within 200ms for up to 500 requirements.
Kanban and priority matrix views must remain responsive with the same
data volume.

---

## 14. Pipeline Profile Requirements

### REQ-REQ-120: Pluggable pipeline profiles

The Requirement Management page must support multiple SDD pipeline profiles that define the document chain, available skills, spec tiering rules, entry paths, and traceability model. The page adapts its chain visualization, skill actions, and artifact rendering based on the active profile.

> Source: PRD §7.8 (Configuration-driven), §12.1 (Template Management), §14 (Skill system)

### REQ-REQ-121: Standard SDD profile (default)

The system must ship with a built-in "Standard SDD" profile that implements the 11-node chain: Requirement → User Story → Spec → Architecture → Design → Tasks → Code → Test → Deploy → Incident → Learning. This is the default profile for Java/open-source projects.

> Source: PRD §13.1

### REQ-REQ-122: Legacy system profile support

The system must support registering custom pipeline profiles for legacy technology stacks. For IBM i / iSeries, a single "IBM i" profile is registered. This profile delegates all skill execution to a single entry point — `ibm-i-workflow-orchestrator` — which automatically determines the workflow path (Full Chain / Enhancement / Fast-Path) and spec tier (L1/L2/L3) based on input analysis. The profile defines its own chain nodes, traceability model, and the orchestrator as the sole skill binding. Example: IBM i profile uses Requirement Normalizer → Functional Spec → Technical Design → Program Spec → File Spec → UT Plan → Test Scaffold → Spec Review → DDS Review → Code Review with shared BR-xx traceability; the orchestrator determines path and tier without user selection.

> Source: PRD §7.8, §14

### REQ-REQ-123: Profile inheritance from workspace

Pipeline profiles are assigned at the Workspace or Project level and inherit via the platform configuration hierarchy: Platform Default → Application Default → SNOW Group Override → Project Override.

> Source: PRD §9.3

### REQ-REQ-124: Profile-specific skill binding

Each pipeline profile defines which AI skills are available for requirement operations. The "Generate Stories" and "Generate Spec" actions are replaced by profile-specific skill invocations (e.g., `req-to-user-story` for Standard SDD vs `ibm-i-workflow-orchestrator` for IBM i, which delegates internally to the appropriate IBM i skills).

> Source: PRD §14

### REQ-REQ-125: Profile-specific chain rendering

The SDLC chain visualization card must render the chain nodes defined by the active pipeline profile, not a hardcoded 11-node chain. Spec or its equivalent must remain visually highlighted as the execution hub regardless of profile.

> Source: PRD §13.1, §7.6

### REQ-REQ-126: Spec tiering support

For profiles that define spec tiering (e.g., IBM i L1/L2/L3), the tier is determined automatically by the `ibm-i-workflow-orchestrator` based on input analysis. The UI displays the orchestrator-determined tier as a read-only badge — users do not manually select the tier. Profiles without tiering (e.g., Standard SDD) do not show the tier indicator.

> Source: build-agent-skill tier-guide

### REQ-REQ-127: Orchestrator-determined entry paths

Pipeline profiles may define multiple entry paths into the requirement pipeline. For the IBM i profile, three paths exist: Full Chain (new programs), Enhancement Path (existing source analysis first), and Fast-Path (mini requirement for small changes). The `ibm-i-workflow-orchestrator` automatically determines the appropriate path based on input analysis — users do not manually select the path. The UI displays the orchestrator's routing decision as a read-only indicator.

> Source: build-agent-skill workflow-orchestrator

### REQ-REQ-128: Cross-layer traceability model

Pipeline profiles define their traceability model. Standard SDD uses per-layer IDs (REQ-xx, SPEC-xx). IBM i profiles use shared BR-xx numbering that persists across all layers (Functional Spec → Technical Design → Program Spec). The traceability visualization must adapt to the active model.

> Source: build-agent-skill BR-xx continuity model

---

## 15. Requirement Intake Requirements

### REQ-REQ-130: Multi-source requirement intake

The Requirement Management page must provide a primary action ("Import Requirement") that accepts raw business input from multiple source formats and uses AI to normalize it into structured requirement drafts. This is the primary entry point for creating new requirements.

> Source: PRD §11.4 — "需求列表、看板、优先级矩阵、需求拆解"; PRD §7.6 — Spec is execution hub (intake feeds the chain)

### REQ-REQ-131: Supported input formats

The intake flow must support the following raw input formats:
- **Plain text**: paste directly into the import panel
- **Excel (.xlsx/.csv)**: upload file, parse rows or sheets as candidate requirements
- **PDF (.pdf)**: upload file, extract text for AI normalization
- **Outlook email (.eml/.msg)**: upload file, parse subject/body/attachments
- **Meeting transcript (.vtt/.txt)**: upload file (e.g., Zoom summary), extract action items and requirements
- **Image (.png/.jpg/.jpeg/.webp)**: upload photo or screenshot (whiteboard, sticky notes, Slack/Teams screenshot, scanned document), AI vision extracts text and requirements

> Source: Enterprise BAU workflow requirements

### REQ-REQ-132: AI-powered normalization

When raw input is submitted, the system must invoke the active pipeline profile's normalizer skill to produce a structured requirement draft. The normalizer extracts: title, priority suggestion, category, description, acceptance criteria, business justification, assumptions, constraints, missing information flags, and open questions.

> Source: build-agent-skill `ibm-i-requirement-normalizer` pattern; PRD §14 (Skill system)

### REQ-REQ-133: Profile-specific normalization

The normalizer skill invoked depends on the active pipeline profile:
- Standard SDD: `req-normalizer` → produces Requirement with acceptance criteria
- IBM i: `ibm-i-workflow-orchestrator` → the orchestrator determines the appropriate normalization mode (full, enhancement, or mini) and produces the corresponding output (Requirement Package with CF-nn, CBR-nn, CE-nn candidate items, or Mini Requirement)

> Source: Pipeline profile architecture; build-agent-skill workflow-orchestrator

### REQ-REQ-134: Draft review before confirmation

After AI normalization, the user must review the extracted requirement draft before it becomes an official requirement. The review step shows all extracted fields, highlights missing information and open questions, and allows the user to edit, confirm, or discard the draft.

> Source: PRD §7.5 — "人是治理者，不是默认执行者"

### REQ-REQ-135: Batch import from structured files

When an Excel file contains multiple rows, the system should allow the user to preview all extracted candidate requirements and select which ones to import. Each selected row becomes a separate requirement draft subject to individual review.

> Source: Enterprise workflow — requirements often arrive as spreadsheet lists

### REQ-REQ-136: Source attachment preservation

The original raw input (file or text) must be preserved as an attachment on the created requirement for audit traceability. Users can view the original source alongside the structured requirement.

> Source: PRD §16.2 (Audit requirements)

### REQ-REQ-137: Import history and audit trail

All import operations must be logged with: source format, file name (if applicable), normalizer skill used, user who initiated, timestamp, and outcome (confirmed/discarded).

> Source: PRD §16.2 (Audit requirements); PRD §14 (Skill execution records)

---

## 16. Out of Scope

- Spec detail editing and full Spec lifecycle management (belongs to Spec Management page)
- User Story sprint planning and velocity tracking (belongs to User Story Management page)
- Real-time WebSocket push (V1 uses polling or on-load fetch)
- JIRA / Azure DevOps bi-directional sync (future integration)
- AI model training or fine-tuning from requirement data
- Cross-workspace requirement dependency analysis
- Mobile-optimized requirement view
- Pipeline profile editor / designer (Platform Center responsibility)
- IBM i skill execution (handled by build-agent-skill, not this module)
- Real-time email/calendar integration (Outlook plugin, Zoom webhook) — V2
- OCR for handwritten notes with complex layout recognition — V2 (basic image-to-text via AI vision is in V1)

---

## 17. Traceability

| Requirement | PRD Section |
|-------------|-------------|
| REQ-REQ-01 | §13 |
| REQ-REQ-02 | §7.6, §13 |
| REQ-REQ-03 | §6.2, §6.5, §6.6 |
| REQ-REQ-04 | §7.8 |
| REQ-REQ-10 | §11.4 |
| REQ-REQ-11 | §11.4 |
| REQ-REQ-12 | — |
| REQ-REQ-20 | §11.4 |
| REQ-REQ-21 | §11.4 |
| REQ-REQ-22 | §11.4 |
| REQ-REQ-23 | §11.4 |
| REQ-REQ-24 | §11.4 |
| REQ-REQ-30 | §14 |
| REQ-REQ-31 | §13 |
| REQ-REQ-32 | §13 |
| REQ-REQ-40 | §7.6, §13 |
| REQ-REQ-41 | §14, §13 |
| REQ-REQ-42 | §13 |
| REQ-REQ-43 | §13 |
| REQ-REQ-50 | §7.7, §13 |
| REQ-REQ-51 | §11.10, §13 |
| REQ-REQ-52 | §13.1 |
| REQ-REQ-53 | §7.7 |
| REQ-REQ-60 | §11.4 |
| REQ-REQ-61 | §11.4 |
| REQ-REQ-62 | §11.4 |
| REQ-REQ-70 | §11.4 |
| REQ-REQ-71 | §11.4 |
| REQ-REQ-72 | §15.4 |
| REQ-REQ-73 | §14 |
| REQ-REQ-80 | §16.2 |
| REQ-REQ-81 | §16.2 |
| REQ-REQ-82 | §16.2 |
| REQ-REQ-90 | §9.2 |
| REQ-REQ-91 | §15.2 |
| REQ-REQ-100 | §15.3 |
| REQ-REQ-101 | §15.5 |
| REQ-REQ-102 | — |
| REQ-REQ-103 | — |
| REQ-REQ-110 | — |
| REQ-REQ-111 | — |
| REQ-REQ-112 | — |
| REQ-REQ-113 | — |
| REQ-REQ-120 | §7.8, §12.1, §14 |
| REQ-REQ-121 | §13.1 |
| REQ-REQ-122 | §7.8, §14 |
| REQ-REQ-123 | §9.3 |
| REQ-REQ-124 | §14 |
| REQ-REQ-125 | §13.1, §7.6 |
| REQ-REQ-126 | build-agent-skill |
| REQ-REQ-127 | build-agent-skill |
| REQ-REQ-128 | build-agent-skill |
| REQ-REQ-130 | §11.4, §7.6 |
| REQ-REQ-131 | Enterprise BAU workflow |
| REQ-REQ-132 | build-agent-skill, §14 |
| REQ-REQ-133 | Pipeline profile, build-agent-skill |
| REQ-REQ-134 | §7.5 |
| REQ-REQ-135 | Enterprise workflow |
| REQ-REQ-136 | §16.2 |
| REQ-REQ-137 | §16.2, §14 |
