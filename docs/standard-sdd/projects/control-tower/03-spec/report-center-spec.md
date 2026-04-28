# Report Center Spec

## Purpose

This is the **implementation-facing contract** for the Report Center slice.
It binds the upstream requirements and user stories to concrete behaviors,
API contracts, state machines, and constraints that frontend (Codex) and
backend (Codex) implementations must satisfy.

## Upstream Inputs

- [`report-center-requirements.md`](../01-requirements/report-center-requirements.md)
- [`report-center-stories.md`](../02-user-stories/report-center-stories.md)

## Downstream Artifacts

- [`report-center-architecture.md`](../04-architecture/report-center-architecture.md)
- [`report-center-design.md`](../05-design/report-center-design.md)
- [`report-center-API_IMPLEMENTATION_GUIDE.md`](../05-design/contracts/report-center-API_IMPLEMENTATION_GUIDE.md)
- [`report-center-tasks.md`](../06-tasks/report-center-tasks.md)

---

## 1. Surface Overview

### 1.1 Routes

| Path | Purpose |
|------|---------|
| `/reports` | Catalog + History + Exports tabs |
| `/reports/{reportKey}` | Report detail with filter form and rendered result |

`reportKey` is one of the V1 keys in §3.1.

### 1.2 Nav Integration

- Navigation label: `Reports`
- Sub-label: `History` (distinguishes from Dashboard which shows `Live`)
- Icon: `mdi-chart-timeline-variant` (or equivalent in design system)
- Order: after `Incidents`, before `Team Space` in `PrimaryNav`

---

## 2. Core Concepts

### 2.1 Report Definition (compile-time)

A **report definition** is a server-owned, code-authored record that fixes:

- a stable `reportKey`
- the `category` (V1: always `efficiency`)
- supported scopes (subset of `[org, workspace, project]`)
- supported filters and their allowed values
- supported groupings
- the chart type to render
- the drilldown column spec

Report definitions are **not** user-editable in V1. Adding a report = a
code change + migration.

### 2.2 Report Run (runtime)

A **report run** is one user's invocation of a report with a specific
`ReportRunRequest` (scope + filters + grouping). It produces a
`ReportRunResult` containing headline metrics, a chart series, a drilldown
table, and a `snapshotAt` timestamp.

### 2.3 Export Artifact

An **export artifact** is a generated file (CSV or PDF) bound to a single
report run. Artifacts are:

- stored for **7 days** then purged
- audited permanently (the audit record outlives the artifact)

---

## 3. Report Catalog (V1)

### 3.1 Report Keys

| reportKey | Category | Name | Scopes | Groupings | Chart |
|-----------|----------|------|--------|-----------|-------|
| `eff.lead-time` | efficiency | Delivery Lead Time | org, workspace, project | team, project, requirementType | histogram |
| `eff.cycle-time` | efficiency | Cycle Time by Stage | org, workspace, project | stage, team, project | stacked-bar |
| `eff.throughput` | efficiency | Throughput | org, workspace, project | week-team, week-project | grouped-bar |
| `eff.wip` | efficiency | Work In Progress | org, workspace, project | stage-team, owner-team | heatmap |
| `eff.flow-efficiency` | efficiency | Flow Efficiency | org, workspace, project | stage, team | horizontal-bar |

### 3.2 Common Filter Shape

Every report accepts a `ReportRunRequest`:

```jsonc
{
  "scope": "org" | "workspace" | "project",
  "scopeIds": ["..."],       // 1+ for workspace/project; exactly [orgId] for org
  "timeRange": {
    "preset": "last7d" | "last30d" | "last90d" | "qtd" | "ytd" | "custom",
    "startAt": "2026-01-01T00:00:00Z",  // required if preset = custom
    "endAt":   "2026-04-18T00:00:00Z"   // required if preset = custom
  },
  "grouping": "team" | "project" | "stage" | "week-team" | ...,
  "extraFilters": { /* per-report opt-in */ }
}
```

### 3.3 Validation Rules

- `scopeIds` non-empty; size ≤ 20
- For preset `custom`: `startAt` < `endAt`, range ≤ 366 days
- `grouping` must be in the report definition's allowed list
- Extra filters must match the report definition's schema

---

## 4. API Contracts (summary)

Full JSON examples in
[`report-center-API_IMPLEMENTATION_GUIDE.md`](../05-design/contracts/report-center-API_IMPLEMENTATION_GUIDE.md).

| Method | Path | Purpose |
|--------|------|---------|
| GET | `/api/v1/reports/catalog` | List report definitions the caller can access |
| POST | `/api/v1/reports/{reportKey}/run` | Run a report; returns `ReportRunResult` |
| POST | `/api/v1/reports/{reportKey}/export` | Start an export job for the same request |
| GET | `/api/v1/reports/exports/{exportId}` | Poll export status / download |
| GET | `/api/v1/reports/history` | Recent report runs for the caller (≤ 50) |
| GET | `/api/v1/reports/exports` | Recent exports for the caller (7-day window) |

All responses use the existing envelope:

```jsonc
{ "data": { ... }, "error": null }
```

Error envelope:

```jsonc
{ "data": null, "error": { "code": "VALIDATION_ERROR", "message": "..." } }
```

### 4.1 Section Isolation

The `ReportRunResult` uses **`SectionResult<T>`** (pattern reused from Dashboard) so a
partial failure in one section (e.g. chart) does not fail the full response.

```jsonc
{
  "snapshotAt": "2026-04-18T10:00:00Z",
  "headline":  { "data": [...], "error": null },
  "series":    { "data": [...], "error": null },
  "drilldown": { "data": [...], "error": null }
}
```

### 4.2 Status Codes

| Status | When |
|--------|------|
| 200 | Success (including section-level error inside envelope) |
| 202 | Export job accepted (async) |
| 400 | Validation failure |
| 403 | Scope not accessible by caller |
| 404 | Unknown `reportKey` or `exportId` |
| 413 | CSV export exceeds 100k rows |
| 429 | Rate limit (≥ 10 concurrent runs per user) |
| 500 | Unhandled server error |

---

## 5. State Machines

### 5.1 Report Run State (frontend)

```
idle → validating → running → rendered | error
rendered → running (when filters change)
error → running (retry)
```

### 5.2 Export Job State (backend + frontend)

```
queued → generating → ready | failed
ready → downloaded  (no state change, just audit record)
ready → expired     (after 7 days, nightly sweep)
```

---

## 6. Behavior Contracts

### 6.1 Catalog

- Returns only reports the caller's permissions can satisfy for at least one scope.
- If the caller has no access to any scope, catalog returns an empty list
  (not a 403).

### 6.2 Run

- Validates request; rejects with 400 on violation.
- Resolves scope → dataset filter:
  - `org` → `orgId IN (caller's org)`; requires `org-viewer` role
  - `workspace` → `workspaceId IN scopeIds ∩ caller's workspaces`
  - `project` → `projectId IN scopeIds ∩ caller's projects`
- Executes the aggregate query against `reporting_fact_*` tables.
- Stamps `snapshotAt` = `now()` UTC.
- Records a row in `report_run` for history.

### 6.3 Export

- `POST /export` returns 202 with an `exportId` and initial status `queued`.
- Worker generates the artifact:
  - CSV — stream rows to object storage; fail with 413 if > 100k rows.
  - PDF — render headline + chart image + drilldown; p95 ≤ 20s.
- On completion, status = `ready`; `downloadUrl` becomes valid.
- On failure, status = `failed` with `error.message`.
- Nightly sweep: artifacts > 7 days moved to `expired`; files purged.
- Every successful generation emits an audit record (§6.5).

### 6.4 History

- Returns up to 50 most recent `report_run` rows for the caller.
- Sorted desc by `runAt`.
- Supports `?reportKey=` filter.

### 6.5 Audit

Every export emits an `audit_event` row:

```
event = "report.export"
actor = caller userId
subject = reportKey
attributes = { scope, scopeIds, timeRange, grouping, format, exportId }
at = now()
```

Routed through the existing `shared/audit/` module (same module used by
platform audit events).

---

## 7. Non-Functional Contracts

### 7.1 Performance

| Operation | Target | Budget |
|-----------|--------|--------|
| GET catalog | p95 ≤ 200ms | in-memory registry |
| POST run | p95 ≤ 2.5s | indexed aggregate query |
| POST export (CSV) | p95 ≤ 10s | streaming write |
| POST export (PDF) | p95 ≤ 20s | server-side render |
| GET history | p95 ≤ 300ms | indexed by userId |

### 7.2 Graceful Degradation

- Slow render (> 2.5s) — succeed with warning header `X-Report-Slow: true`;
  UI shows banner.
- Timeout (> 30s client-side) — UI shows section error + Retry.
- Dataset > 100k rows for CSV — 413 with guidance.

### 7.3 Security

- All endpoints require authenticated caller (session cookie or bearer token).
- Scope enforcement via `@PreAuthorize`-style guard on every endpoint.
- Export download URLs are signed and expire in 15 minutes.

### 7.4 Audit

- All exports audited (REQ-RPT-42).
- Audit records are immutable and live in the platform audit store.

---

## 8. Data Contracts (summary)

Full types in
[`report-center-data-model.md`](../04-architecture/report-center-data-model.md).

Key shapes:

| Type | Purpose |
|------|---------|
| `ReportDefinition` | compile-time definition (key, category, scopes, etc.) |
| `ReportRunRequest` | caller's request |
| `ReportRunResult` | rendered result with headline / series / drilldown |
| `SectionResult<T>` | `{ data, error }` wrapper for per-section isolation |
| `ExportJob` | `{ id, status, format, downloadUrl?, error? }` |
| `ReportRunHistoryEntry` | summary row for history list |

---

## 9. Integration with Existing Platform

### 9.1 Reuse

- **Envelope** — `shared/dto/ApiResponse.java`
- **Section isolation** — `SectionResult<T>` pattern from Dashboard
- **Workspace context** — `platform/workspace/WorkspaceContext`
- **Nav** — add nav entry via `platform/navigation/NavigationService`
- **Audit** — existing `shared/audit/AuditEventService`
- **Error handling** — `shared/exception/GlobalExceptionHandler`
- **CORS + proxy** — reuse existing `/api/v1/**` rules

### 9.2 New Additions

- Domain module `domain/reportcenter/`
- Flyway migration series `V{next}__report_center_*.sql` — see data-model
- Front-end feature module `frontend/src/features/reportcenter/`
- Router entries under `frontend/src/router/index.ts`

### 9.3 Mock → Live Toggle

Follow the same pattern as Dashboard: a `useMockData` flag on the store so
frontend can develop without backend ready. Default OFF once backend is
deployed.

---

## 10. Out-of-Scope Contracts (V1)

- **NOT** a custom report builder — no `PUT /reports/definitions` endpoint
- **NOT** scheduled delivery — no scheduler, no subscription tables
- **NOT** Excel (.xlsx) — only CSV and PDF
- **NOT** individual-level drilldown
- **NOT** real-time push — snapshots only
- **NOT** a new audit infrastructure — reuse existing `shared/audit/`

---

## 11. Traceability

| Spec section | Requirement IDs | Story IDs |
|--------------|-----------------|-----------|
| §1, §3 | REQ-RPT-10..15 | RPT-S01, RPT-S02 |
| §3.2, §4 (run) | REQ-RPT-20..24 | RPT-S10..S14 |
| §4.1, §6.2 | REQ-RPT-30..32 | RPT-S20..S22 |
| §4 (export), §6.3, §6.5 | REQ-RPT-40..43 | RPT-S30..S32 |
| §4 (history), §6.4 | REQ-RPT-50..51 | RPT-S40, RPT-S41 |
| §6.2 (snapshotAt) | REQ-RPT-60..62 | RPT-S50 |
| §6.2 (scope resolve) | REQ-RPT-70..72 | RPT-S10 |
| §7 | REQ-RPT-80..83 | RPT-S51 |
