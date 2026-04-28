# Report Center — Data Flow

## Purpose

Runtime data flows for the **Report Center** slice. Sequence diagrams,
state machines, error cascades, and refresh strategy. This document
supplements [`report-center-architecture.md`](./report-center-architecture.md).

---

## 1. Phase A: Frontend with Mock Data

```mermaid
sequenceDiagram
    autonumber
    participant U as User
    participant R as Router
    participant V as ReportCenterView
    participant S as reportCenterStore (Pinia)
    participant M as mockReports.ts

    U->>R: GET /reports
    R->>V: mount CatalogView
    V->>S: fetchCatalog()
    S->>M: getMockCatalog()
    M-->>S: CatalogDefinition[]
    S-->>V: catalog state
    V-->>U: render catalog

    U->>V: click "Lead Time"
    V->>R: push /reports/eff.lead-time
    R->>V: mount ReportDetailView
    V->>S: initDetail("eff.lead-time")
    S-->>V: defaults + empty result

    U->>V: apply filters
    V->>S: runReport(request)
    S->>M: getMockRunResult(key, request)
    M-->>S: ReportRunResult
    S-->>V: result state
    V-->>U: render headline/chart/drilldown
```

**Phase A contract:** All network effects replaced by `mockReports.ts`.
`useMockData` flag = `true` during FE development before backend is live.

---

## 2. Phase B: Frontend → Backend

### 2.1 Catalog load

```mermaid
sequenceDiagram
    autonumber
    participant V as CatalogView
    participant S as reportCenterStore
    participant CLI as reportCenterApi.ts
    participant HTTP as fetchJson
    participant CTRL as ReportCenterController
    participant CAT as ReportCatalogService
    participant REG as DefinitionRegistry
    participant WS as WorkspaceContextService

    V->>S: fetchCatalog()
    S->>CLI: getCatalog()
    CLI->>HTTP: GET /api/v1/reports/catalog
    HTTP->>CTRL: HTTP GET
    CTRL->>CAT: getCatalogForCaller()
    CAT->>WS: listAccessibleScopes(caller)
    WS-->>CAT: {org?, workspaces[], projects[]}
    CAT->>REG: allDefinitions()
    REG-->>CAT: ReportDefinition[]
    CAT-->>CTRL: filtered catalog
    CTRL-->>HTTP: 200 { data: CatalogDto }
    HTTP-->>CLI: parsed dto
    CLI-->>S: catalog
    S-->>V: catalog state
```

### 2.2 Report run

```mermaid
sequenceDiagram
    autonumber
    participant V as ReportDetailView
    participant S as reportCenterStore
    participant CLI as reportCenterApi.ts
    participant CTRL as ReportCenterController
    participant RUN as ReportRunService
    participant GUARD as ScopeAuthGuard
    participant REPO as ReportingFactRepository
    participant DB as DB

    V->>S: runReport(request)
    S->>CLI: runReport(key, request)
    CLI->>CTRL: POST /reports/{key}/run
    CTRL->>GUARD: check(caller, scope, scopeIds)
    GUARD-->>CTRL: ok
    CTRL->>RUN: run(request, def)
    RUN->>REPO: aggregate(def, request)
    REPO->>DB: indexed SELECT
    DB-->>REPO: rows
    REPO-->>RUN: aggregates
    RUN->>RUN: buildHeadline() / buildSeries() / buildDrilldown()
    RUN-->>CTRL: ReportRunResult
    CTRL->>REPO: insert report_run
    CTRL-->>CLI: 200 { data: result }
    CLI-->>S: result
    S-->>V: render
```

### 2.3 Export (async)

```mermaid
sequenceDiagram
    autonumber
    participant V as ExportActions
    participant S as reportCenterStore
    participant CLI as reportCenterApi.ts
    participant CTRL as ReportCenterController
    participant EXP as ReportExportService
    participant W as ExportWorker
    participant RENDER as PdfRenderer / CsvWriter
    participant BLOB as ArtifactStore
    participant AUD as AuditEventService

    V->>S: requestExport(format)
    S->>CLI: export(key, request, format)
    CLI->>CTRL: POST /reports/{key}/export
    CTRL->>EXP: enqueue(request, format, caller)
    EXP->>EXP: insert report_export (status=queued)
    EXP-->>CTRL: exportId
    CTRL-->>CLI: 202 { exportId, status: queued }
    CLI-->>S: job ref
    S-->>V: show "preparing" toast
    S->>S: startPoll(exportId)

    Note right of W: async worker picks up
    W->>W: status=generating
    W->>RENDER: renderCsv() or renderPdf()
    RENDER-->>W: bytes
    W->>BLOB: store(bytes)
    W->>W: status=ready, downloadUrl=signed
    W->>AUD: emit("report.export", ...)

    loop poll every 2s, up to 30s
        S->>CLI: getExport(exportId)
        CLI->>CTRL: GET /reports/exports/{id}
        CTRL-->>CLI: { status, downloadUrl? }
    end

    alt status=ready
        S-->>V: show "Download" CTA
        V->>CLI: window.open(downloadUrl)
    else status=failed
        S-->>V: show error + retry
    end
```

---

## 3. Error Cascade

Report Center uses **section-isolated** responses so a failure in one
section does not fail the full page.

```mermaid
flowchart TD
    A[POST /reports/:key/run] --> B{Validate request}
    B -->|400| E1[HTTP 400 top-level error]
    B --> C{Scope auth}
    C -->|403| E2[HTTP 403 top-level error]
    C --> D[Run aggregate query]
    D -->|DB error| E3[HTTP 500 top-level error]
    D --> H{Assemble sections}
    H --> H1[Headline section]
    H --> H2[Series section]
    H --> H3[Drilldown section]
    H1 -->|ok| R[200 with section.data]
    H1 -->|build err| RE1[200 with section.error]
    H2 -->|ok| R
    H2 -->|build err| RE2[200 with section.error]
    H3 -->|ok| R
    H3 -->|build err| RE3[200 with section.error]
```

Rules:

- **Top-level failure** (validation, auth, DB connection, unhandled
  exception) → 4xx/5xx with `{ error: { code, message } }`.
- **Section-level failure** (chart builder threw, drilldown exceeded
  soft limit) → 200 with `SectionResult.error` set for that section and
  `SectionResult.data = null`.
- Frontend renders each section independently; one section's error cannot
  crash the view.

---

## 4. State Machines

### 4.1 Frontend report run state

```mermaid
stateDiagram-v2
    [*] --> idle
    idle --> validating: applyFilters()
    validating --> idle: validation fail
    validating --> running: ok
    running --> rendered: 200
    running --> error: 4xx/5xx
    rendered --> running: filters changed
    error --> running: retry
```

### 4.2 Export job state

```mermaid
stateDiagram-v2
    [*] --> queued: POST /export
    queued --> generating: worker picks up
    generating --> ready: file written
    generating --> failed: worker exception
    ready --> expired: nightly sweep > 7 days
    ready --> [*]: downloaded (audit only)
    failed --> [*]
    expired --> [*]
```

---

## 5. Refresh Strategy

- **Catalog** — fetched once per session; refreshed if user navigates away
  and back after > 5 minutes.
- **Report run** — on-demand only; no auto-refresh in V1 (reports are
  historical snapshots, not live).
- **History / Exports** — fetched on tab activation; soft refresh every
  30 seconds while the tab is visible, using the page-visibility API.
- **Export polling** — every 2 seconds for 30 seconds after POST; then
  stop and show "still generating — come back later" hint.

---

## 6. Deep-linking and URL State

Filter state is mirrored to the URL query string so reports are linkable:

```
/reports/eff.lead-time?scope=workspace&scopeIds=ws-1,ws-2&preset=last30d&grouping=team
```

On mount, `ReportDetailView` parses URL → populates filter form → kicks
off a run. On filter change, the URL is `replaceState`-d (no new history
entry for every keystroke).

---

## 7. Audit Flow

Every successful export emits:

```
AuditEvent {
  event:   "report.export",
  actor:   callerUserId,
  subject: reportKey,
  at:      now(),
  attrs: {
    scope: "workspace",
    scopeIds: ["ws-1"],
    timeRange: { preset: "last30d" },
    grouping: "team",
    format: "pdf",
    exportId: "exp-abc",
    rowCount: 812,
    bytes: 482113
  }
}
```

Routed through `shared/audit/AuditEventService.record(event)`. The call is
**synchronous from the worker's perspective** — if audit write fails, the
export is marked `failed`. "No audit, no export" is a hard invariant
(REQ-RPT-42 is load-bearing).

---

## 8. Frontend API Client Chain

```mermaid
graph LR
    V["Components (e.g. ReportDetailView)"]
    S["reportCenterStore.ts (Pinia)"]
    API["reportCenterApi.ts"]
    HTTP["shared/fetchJson<T>"]
    CTRL["Spring Controller"]

    V --> S
    S --> API
    API --> HTTP
    HTTP --> CTRL
```

- `useMockData=true` in dev → `API` returns mock instead of calling `HTTP`.
- `useMockData=false` → normal flow through `fetchJson` which applies the
  same envelope parsing as other slices.

---

## 9. Traceability

| Flow | Spec ref | Requirement IDs |
|------|----------|-----------------|
| §1 Phase A | §9.3 | (FE decoupling) |
| §2.1 Catalog | §6.1 | REQ-RPT-10, REQ-RPT-70 |
| §2.2 Run | §6.2 | REQ-RPT-20..32, REQ-RPT-60 |
| §2.3 Export | §6.3, §6.5 | REQ-RPT-40..43 |
| §3 Error cascade | §4.1, §4.2 | REQ-RPT-32, REQ-RPT-81 |
| §4 State machines | §5 | REQ-RPT-22 |
| §5 Refresh | §6 | REQ-RPT-61 |
| §7 Audit | §6.5 | REQ-RPT-42 |
