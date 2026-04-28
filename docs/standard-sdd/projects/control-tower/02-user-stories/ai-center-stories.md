# AI Center — User Stories

## Purpose

Translate the requirements in `01-requirements/ai-center-requirements.md` into Jira-ready Agile user stories with Given/When/Then acceptance criteria. Stories are grouped by epic and map back to REQ-AIC-* IDs.

## Source

- Upstream: [ai-center-requirements.md](../01-requirements/ai-center-requirements.md)
- PRD: [agentic_sdlc_control_tower_prd_v0.9.md](../01-requirements/agentic_sdlc_control_tower_prd_v0.9.md)

---

## Epics

| Epic | Scope | Stories |
|------|-------|---------|
| **E1. Skill Catalog** | Browse and search registered skills | US-AIC-01 … US-AIC-04 |
| **E2. Run History** | Global timeline of skill executions | US-AIC-10 … US-AIC-13 |
| **E3. Adoption Metrics** | Top-level effectiveness summary | US-AIC-20 … US-AIC-22 |
| **E4. Policy View** | Read-only policy + autonomy view | US-AIC-30 … US-AIC-31 |
| **E5. Evidence & Audit** | Link-through to evidence and audit | US-AIC-40 … US-AIC-41 |
| **E6. Cross-page Navigation** | Deep-link in, drill-back out | US-AIC-50 … US-AIC-51 |
| **E7. Shell Integration & States** | Workspace scoping, loading/error/empty | US-AIC-60 … US-AIC-63 |

---

## Epic E1 — Skill Catalog

### US-AIC-01 — Browse the skill catalog
**As a** Team Lead
**I want** to see a list of all AI skills registered in my workspace
**So that** I can understand what AI capabilities my team has at its disposal.

**Acceptance Criteria**
- **Given** I navigate to `/ai-center`
  **When** the page loads with data
  **Then** I see a catalog card listing skills with: skill key, name, category, status, default autonomy, owner, last execution timestamp, and 30-day success rate.
- **Given** the catalog has ≤100 skills
  **When** I scroll the catalog
  **Then** all skills are rendered without pagination (V1 scale).

**Covers:** REQ-AIC-10

---

### US-AIC-02 — Filter and search the catalog
**As a** Platform Admin
**I want** to filter skills by category, status, autonomy level, and owner, and search by key or name
**So that** I can find the skills I need to review quickly.

**Acceptance Criteria**
- **Given** I am on the catalog
  **When** I apply a filter (e.g., category = `runtime`)
  **Then** only matching skills remain in the list and the filter chip is visible.
- **Given** filters are applied
  **When** I clear all filters
  **Then** the full catalog is restored.

**Covers:** REQ-AIC-13

---

### US-AIC-03 — Open skill detail view
**As a** Delivery Manager
**I want** to click a skill to see its description, current policy bindings, recent runs, and aggregate metrics
**So that** I can decide whether to recommend or restrict its use on my team.

**Acceptance Criteria**
- **Given** I click a skill row
  **When** the detail view opens
  **Then** I see description, input/output contract summary, current policy, last N runs, and aggregate metrics (success rate, avg duration, adoption trend).
- **Given** the skill has no runs yet
  **When** I open its detail
  **Then** the runs section shows an empty state instead of an error.

**Covers:** REQ-AIC-12

---

### US-AIC-04 — Categories match PRD taxonomy
**As a** product stakeholder
**I want** the catalog to organize skills by the PRD §14 categories (delivery-phase vs runtime-phase)
**So that** the UI matches the product vocabulary and is easy to map onto the 11-node SDLC chain.

**Acceptance Criteria**
- **Given** the catalog is rendered
  **When** I toggle to the "by category" view
  **Then** skills are grouped under the two category families with the PRD-specified sub-categories.

**Covers:** REQ-AIC-11

---

## Epic E2 — Run History

### US-AIC-10 — View global run history
**As an** SRE
**I want** to see a reverse-chronological timeline of all skill executions in my workspace
**So that** I can see what AI has been doing recently.

**Acceptance Criteria**
- **Given** I am on the Run History card
  **When** data loads
  **Then** each row shows: execution ID, skill key + name, trigger source, start/end timestamps, duration, status, outcome summary, and links to evidence + audit.
- **Given** more than 50 runs exist
  **When** I reach the bottom of the first page
  **Then** a "Load more" or pagination control fetches the next page.

**Covers:** REQ-AIC-20

---

### US-AIC-11 — Filter run history
**As an** auditor
**I want** to filter runs by skill, status, trigger source, time range, and AI-vs-human trigger
**So that** I can narrow in on the set of executions relevant to a question.

**Acceptance Criteria**
- **Given** I apply multiple filters simultaneously
  **When** results load
  **Then** only runs matching all filters are shown and the total count reflects the filtered set.

**Covers:** REQ-AIC-21

---

### US-AIC-12 — Drill into a run
**As a** governance owner
**I want** to open a run and see input summary, step breakdown, output, policy trail, evidence links, and audit link
**So that** I can audit what the AI actually did.

**Acceptance Criteria**
- **Given** I click a run row
  **When** the detail opens
  **Then** I see all six fields above, with evidence and audit links opening in appropriate target views (new tab or modal).

**Covers:** REQ-AIC-22

---

### US-AIC-13 — Manual refresh
**As a** user monitoring an in-progress run
**I want** a refresh control on the run history card
**So that** I can see updated statuses without reloading the whole page.

**Acceptance Criteria**
- **Given** the card is in the Normal state
  **When** I click "Refresh"
  **Then** a new fetch is issued and the list updates with any newly-arrived or status-changed runs.

**Covers:** REQ-AIC-73

---

## Epic E3 — Adoption Metrics

### US-AIC-20 — See the AI adoption summary
**As a** Platform Admin
**I want** a top-of-page metrics strip covering usage rate, adoption rate, auto-exec success rate, time saved, and stage coverage
**So that** I have an at-a-glance read on AI health in this workspace.

**Acceptance Criteria**
- **Given** I load the page
  **When** metrics data returns
  **Then** five metric cards are visible with values, units, and trend indicators for a 30-day default window.

**Covers:** REQ-AIC-30, REQ-AIC-31

---

### US-AIC-21 — Trends vs prior window
**As a** Delivery Manager
**I want** each metric to show a ↑/↓/→ indicator and delta vs. the previous equivalent window
**So that** I can tell if AI usage is improving or regressing.

**Acceptance Criteria**
- **Given** metric data includes both current and previous window values
  **When** cards render
  **Then** each card shows the delta and an arrow; when delta is zero the indicator is →.

**Covers:** REQ-AIC-31

---

### US-AIC-22 — Stage coverage visualization
**As a** product stakeholder
**I want** to see which of the 11 SDLC stages currently have active AI skills
**So that** I can identify coverage gaps.

**Acceptance Criteria**
- **Given** the page is rendered
  **When** I look at the stage coverage widget
  **Then** the canonical 11-node chain is displayed with each node marked as covered / uncovered, and covered nodes show the count of active skills.

**Covers:** REQ-AIC-32

---

## Epic E4 — Policy View

### US-AIC-30 — View skill policy read-only
**As an** auditor
**I want** to view the current policy configuration for each skill (autonomy level, approval gates, risk thresholds)
**So that** I can confirm what governance is in force.

**Acceptance Criteria**
- **Given** I open a skill detail
  **When** the Policy section renders
  **Then** I see autonomy level, approval-required action types, authorized approver roles, and last-changed timestamp — and no edit controls are present in V1.

**Covers:** REQ-AIC-40, REQ-AIC-41, REQ-AIC-42

---

### US-AIC-31 — Understand autonomy levels
**As a** first-time user
**I want** autonomy levels (L0–L3) to be visually distinguishable and have tooltips explaining each
**So that** I can understand what AI can do on its own vs. under approval.

**Acceptance Criteria**
- **Given** autonomy chips render on any skill row
  **When** I hover/focus one
  **Then** a tooltip shows the level name and a one-line description matching the REQ-AIC-41 definitions.

**Covers:** REQ-AIC-41

---

## Epic E5 — Evidence & Audit

### US-AIC-40 — Open evidence for a run
**As an** auditor
**I want** to click an evidence link on a run detail and see the referenced artifacts
**So that** I can verify what the AI consulted and produced.

**Acceptance Criteria**
- **Given** a run has evidence references
  **When** I click "Evidence"
  **Then** I see a list of evidence links (artifact title, type, source system, link). V1 does not require an inline evidence viewer.

**Covers:** REQ-AIC-50

---

### US-AIC-41 — Jump to audit record
**As an** auditor
**I want** every run detail to have a link to its audit record
**So that** I can cross-reference against the platform audit log.

**Acceptance Criteria**
- **Given** a run record exists
  **When** I click "Audit record"
  **Then** I navigate to (or open a modal for) the matching audit entry.

**Covers:** REQ-AIC-51

---

## Epic E6 — Cross-page Navigation

### US-AIC-50 — Deep-link into AI Center
**As a** Dashboard user clicking the "Learning" stage
**I want** to land on AI Center
**So that** the Dashboard SDLC chain terminates at the AI governance surface.

**Acceptance Criteria**
- **Given** I am on `/` and click the Learning SDLC node (route path `/ai-center`)
  **When** the route resolves
  **Then** AI Center loads with its default view (catalog + run history + metrics).

**Covers:** REQ-AIC-60

---

### US-AIC-51 — Drill back to source page
**As a** user on a run detail
**I want** a "go to source" link when the run was triggered from a specific context (incident, requirement, etc.)
**So that** I can jump back to where the skill ran.

**Acceptance Criteria**
- **Given** a run record has a `triggerSource` with a known `sourceUrl`
  **When** I click "Go to source"
  **Then** I navigate to that URL.
- **Given** a run has no sourceUrl
  **When** the detail renders
  **Then** the link is hidden (not disabled).

**Covers:** REQ-AIC-61

---

## Epic E7 — Shell Integration & States

### US-AIC-60 — Workspace scoping
**As a** multi-workspace user
**I want** AI Center to show data only for my current workspace
**So that** data from other workspaces does not leak in.

**Acceptance Criteria**
- **Given** the shared app shell reports workspace `WS-A`
  **When** AI Center fetches data
  **Then** all API calls include the workspace context and responses include only `WS-A` data.
- **Given** I switch workspaces
  **When** the shell context changes
  **Then** AI Center re-fetches and re-renders for the new workspace.

**Covers:** REQ-AIC-81

---

### US-AIC-61 — Loading state
**As a** user
**I want** skeleton loaders on each card while data loads
**So that** the page doesn't feel broken.

**Acceptance Criteria**
- **Given** I open the page
  **When** any card's API call is in-flight
  **Then** that card shows a skeleton; other cards load independently.

**Covers:** REQ-AIC-72

---

### US-AIC-62 — Empty state
**As a** brand-new workspace owner
**I want** a helpful empty state when no skills are registered
**So that** I know what to do next.

**Acceptance Criteria**
- **Given** the catalog returns zero skills
  **When** the card renders
  **Then** an empty state shows a short message and a link (where applicable) to Platform Center skill registration.

**Covers:** REQ-AIC-72

---

### US-AIC-63 — Partial failure isolation
**As a** user
**I want** one card's failure not to break the whole page
**So that** I can still see the parts that succeeded.

**Acceptance Criteria**
- **Given** the catalog API returns 200 but run history API returns 500
  **When** the page renders
  **Then** the catalog and metrics cards render normally and the run history card shows a per-card error with retry.

**Covers:** REQ-AIC-72

---

## Traceability Matrix

| Story | Requirement(s) |
|-------|----------------|
| US-AIC-01 | REQ-AIC-10 |
| US-AIC-02 | REQ-AIC-13 |
| US-AIC-03 | REQ-AIC-12 |
| US-AIC-04 | REQ-AIC-11 |
| US-AIC-10 | REQ-AIC-20 |
| US-AIC-11 | REQ-AIC-21 |
| US-AIC-12 | REQ-AIC-22 |
| US-AIC-13 | REQ-AIC-73 |
| US-AIC-20 | REQ-AIC-30, REQ-AIC-31 |
| US-AIC-21 | REQ-AIC-31 |
| US-AIC-22 | REQ-AIC-32 |
| US-AIC-30 | REQ-AIC-40, REQ-AIC-41, REQ-AIC-42 |
| US-AIC-31 | REQ-AIC-41 |
| US-AIC-40 | REQ-AIC-50 |
| US-AIC-41 | REQ-AIC-51 |
| US-AIC-50 | REQ-AIC-60 |
| US-AIC-51 | REQ-AIC-61 |
| US-AIC-60 | REQ-AIC-81 |
| US-AIC-61 | REQ-AIC-72 |
| US-AIC-62 | REQ-AIC-72 |
| US-AIC-63 | REQ-AIC-72 |
