# Report Center Requirements

## Purpose

This document extracts the requirements relevant to the **Report Center** page
from the full PRD (V0.9). It defines **what** the Report Center must deliver
in V1, serving as the upstream input for the spec, architecture, design, and
task documents.

## Source

All requirements below are derived from:

- [agentic_sdlc_control_tower_prd_v0.9.md](agentic_sdlc_control_tower_prd_v0.9.md)

Section references use the format `PRD §N.N`.

---

## 1. Scope

### 1.1 In Scope (V1)

The Report Center V1 covers:

- A catalog of **canned reports** — a predefined, curated set of report types
- **Efficiency** category reports only
  - lead time, cycle time, throughput, WIP, flow efficiency
- **Scope hierarchy** — org level, workspace level, project level
- **Filters** — scope, time range, team/project multi-select, grouping dimension
- **Export** — CSV (raw data) and PDF (formatted snapshot)
- **History listing** — a user's recent report runs and re-open capability
- **Clear differentiation from Dashboard** — history/filter/export vs. realtime

### 1.2 Out of Scope (V1, deferred to V2+)

- **Quality** category reports (defect density, escape rate, test coverage, review findings)
- **Stability** category reports (MTTR, MTTD, change failure rate, rollback rate, SLO burn)
- **Governance** category reports (policy compliance, approval SLA, audit coverage)
- **AI Contribution** category reports (skill execution count, AI acceptance rate, human override rate)
- **Custom report builder** (user-defined metrics, dimensions, visualizations)
- **Scheduled delivery** (email/IM digest, subscription management)
- **Excel (.xlsx) export** (CSV + PDF only for V1)
- **Individual-level drill-down** (org/workspace/project only for V1; privacy-sensitive)
- **Shareable report links / public URLs**
- **Real-time streaming metrics** (those belong to Dashboard)

---

## 2. Product Context Requirements

### REQ-RPT-01: Historical, filterable, exportable, reusable

Report Center serves **history, reflection, reporting, and outward communication**.
It is distinct from Dashboard (realtime, operational, command view).

> Source: PRD §11.12 — "Report Center：历史、可筛选、可导出、可复用，用于复盘、汇报和对外沟通"

### REQ-RPT-02: Organization / team / project level reports

Reports must be scopable to **org level**, **team (workspace) level**, and
**project level**. Users should be able to filter and aggregate at any of
these levels.

> Source: PRD §11.12 — "组织级、团队级和项目级报表"

### REQ-RPT-03: Distinct from Dashboard

Report Center must be **clearly differentiated from Dashboard**:

- Dashboard — realtime, in-operation, command-and-story oriented
- Report Center — historical, filterable, exportable, reusable, for reflection
  and stakeholder communication

> Source: PRD §11.12, §11.1 — "Dashboard 与 Report Center 明确区分"

### REQ-RPT-04: Agentic SDLC promotion surface

Report Center is **also a key vehicle for promoting Agentic SDLC**. Reports
should be packagable as narratives that can be shared with stakeholders
outside the direct delivery team.

> Source: PRD §11.12 — "同时也是推广 Agentic SDLC 的重要载体"

---

## 3. Report Catalog Requirements (V1 — Efficiency only)

### REQ-RPT-10: Canned report catalog

The Report Center must expose a **catalog** of predefined report templates.
Each template has a stable `reportKey`, a category, a human-readable name, a
description, the scopes it supports, and the filters it accepts.

> Source: PRD §11.12

### REQ-RPT-11: Efficiency report — Lead Time

Report: **Delivery Lead Time** — time from requirement ready to deploy.

- Group by: team, project, requirement type
- Shape: distribution histogram + median/p75/p95 table

### REQ-RPT-12: Efficiency report — Cycle Time

Report: **Cycle Time** — time from work start to work complete for each
SDLC stage.

- Group by: stage, team, project
- Shape: stacked bar per stage + per-team trend line

### REQ-RPT-13: Efficiency report — Throughput

Report: **Throughput** — items completed per week, by team or project.

- Group by: week × team, week × project
- Shape: grouped bar chart + summary table

### REQ-RPT-14: Efficiency report — Work In Progress

Report: **Work in Progress (WIP)** — active items per stage and per owner,
with WIP-age aging buckets.

- Group by: stage × team, owner × team
- Shape: matrix heatmap + aging-bucket table

### REQ-RPT-15: Efficiency report — Flow Efficiency

Report: **Flow Efficiency** — active work time ÷ total cycle time, per stage.

- Group by: stage, team
- Shape: per-stage horizontal bar + trend line

---

## 4. Filter and Scope Requirements

### REQ-RPT-20: Scope selector

Every report must expose a **scope selector**: `org | workspace | project`.
Selecting a scope narrows all downstream filter options to entities within
that scope.

### REQ-RPT-21: Time range filter

Reports must support a **time range** filter with presets
(`last 7 days`, `last 30 days`, `last 90 days`, `quarter-to-date`,
`year-to-date`, `custom`). Custom allows a start/end date picker.

### REQ-RPT-22: Entity multi-select

Within a chosen scope, users must be able to **multi-select entities** (e.g.
multiple teams at workspace scope, multiple projects at project scope).

### REQ-RPT-23: Grouping dimension

Each report must accept a **grouping dimension** consistent with its report
definition (e.g. by team, by project, by stage, by week).

### REQ-RPT-24: Filter persistence per session

Filter selections must persist during a user's session so that navigating
away and back does not reset the filter state. Session-scoped persistence
is sufficient in V1 (no saved-view feature).

---

## 5. Result Presentation Requirements

### REQ-RPT-30: Tabular + visual side-by-side

Each report detail page must show:

- A **headline metric strip** — 2 to 4 KPI values summarizing the report
- A **primary visualization** — chart type defined per report
- A **drilldown table** — the raw aggregated rows backing the chart

### REQ-RPT-31: Empty and no-data states

When a filter combination yields no data, the report must show an empty-state
message that explains the filter and offers a "reset filters" action.

### REQ-RPT-32: Loading and error states

Slow queries must show a loading skeleton for the chart and table sections
independently. Errors in a section must not break the full page — the section
must show an inline error with a retry action.

---

## 6. Export Requirements

### REQ-RPT-40: CSV export

Any rendered report must be exportable to **CSV**. The CSV reflects the
drilldown table's columns, respects the active filters, and uses ISO 8601
timestamps.

### REQ-RPT-41: PDF export

Any rendered report must be exportable to **PDF**. The PDF includes:

- a title line with the report name, scope, time range, and generation time
- the headline metric strip as text
- the primary visualization rendered as an embedded image
- the drilldown table as a rendered table

### REQ-RPT-42: Export audit

Every successful export must be recorded in an audit trail with fields:
user, report key, scope, filters, format, and timestamp. The audit record
is the authoritative history of external report distribution.

> Source: PRD §12.3 — "审计不是附加能力，而是信任基础"

### REQ-RPT-43: Export size guardrail

If the drilldown result exceeds **100,000 rows**, the CSV export must fail
with a clear message instructing the user to narrow the filter.

---

## 7. History Requirements

### REQ-RPT-50: Per-user report history

The Report Center must show a user **their recent report runs** (up to 50)
with fields: report name, scope summary, time range, run timestamp, and a
"re-open" action that restores the exact filter set.

### REQ-RPT-51: Exports separately listed

Export events must be separately listed, with download links retained for
**7 days**. After 7 days the artifact is purged; the audit record remains.

---

## 8. Data and Freshness Requirements

### REQ-RPT-60: Point-in-time snapshot

Each report run is a **point-in-time snapshot**. The response must include a
`snapshotAt` timestamp so users know the data freshness.

### REQ-RPT-61: No realtime guarantee

Report Center does **not** guarantee realtime freshness. A brief lag
(minutes to hours) from source systems is acceptable, as reports are for
retrospective analysis rather than operational command.

> Source: PRD §11.12 — contrasted with Dashboard which is realtime

### REQ-RPT-62: Source system decoupling

Reports read from the platform's aggregate store populated by adapters
(Jira, GitLab, Jenkins, ServiceNow). Report Center must not call source
systems directly at query time.

> Source: PRD §12.6 — Adapter Model

---

## 9. Permission and Tenancy Requirements

### REQ-RPT-70: Scope-based authorization

A user may only query reports for scopes they have access to. Accessing an
unauthorized scope must return `403 Forbidden` with a clear message; the
report list UI must not surface scopes the user cannot access.

> Source: PRD §12.4 — Access Management (RBAC + ABAC)

### REQ-RPT-71: Workspace isolation

Workspace-scoped queries must be isolated by workspace ID. Cross-workspace
leakage via filter manipulation must be impossible.

> Source: PRD §11.4, §12.4

### REQ-RPT-72: Org-level gating

Org-level reports require the org-viewer role or above. V1 defaults: platform
admins and org analysts receive this role; others see workspace/project only.

---

## 10. Non-Functional Requirements

### REQ-RPT-80: Performance target — p95

For the V1 dataset (≤ 5 workspaces, ≤ 50 projects, 12 months of data), the
report-render API p95 must be **≤ 2.5 seconds**. Export generation p95 must
be **≤ 10 seconds** for CSV and **≤ 20 seconds** for PDF.

### REQ-RPT-81: Graceful degradation

If a report exceeds the performance target, the UI must show a warning banner
("Result exceeded expected render time; consider narrowing the filter") but
must still render the result.

### REQ-RPT-82: Accessibility

All charts must have a table alternate form (the drilldown table already
meets this). Colors must not be the sole carrier of meaning — shape and
label must also distinguish series.

### REQ-RPT-83: Internationalization

UI strings must be routed through the shared i18n layer so report names and
categories can be localized without code changes. Report data (team names,
project names) is passed through unchanged.

---

## 11. Out-of-Scope (explicit non-goals for V1)

- **NOT** a custom report builder (V2)
- **NOT** a scheduled email/IM delivery system (V2)
- **NOT** a realtime dashboard (that is Dashboard)
- **NOT** a data warehouse replacement — we only support the metrics defined in §3
- **NOT** a per-individual surveillance tool — individual-level drilldown is deferred
- **NOT** a replacement for source systems' native reports (Jira dashboards, GitLab analytics)
- **NOT** an alerting or notification surface

---

## 12. Traceability

| Requirement ID | PRD Section | Downstream Artifact |
|----------------|-------------|---------------------|
| REQ-RPT-01..04 | §11.12, §11.1 | spec §2, architecture §2 |
| REQ-RPT-10..15 | §11.12 | spec §3, design §4 |
| REQ-RPT-20..24 | §11.12 | spec §4, design §5 |
| REQ-RPT-30..32 | §11.12 | design §6 |
| REQ-RPT-40..43 | §11.12, §12.3 | design §7, API guide §4 |
| REQ-RPT-50..51 | §11.12 | design §8 |
| REQ-RPT-60..62 | §11.12, §12.6 | architecture §3, §4 |
| REQ-RPT-70..72 | §12.4 | architecture §5 |
| REQ-RPT-80..83 | §7 (NFRs) | architecture §6 |
