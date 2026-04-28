# Report Center — API Implementation Guide

## Purpose

Full endpoint contracts with complete JSON examples, backend skeleton,
frontend integration patterns, and testing contracts for the Report
Center slice. This is the bridge between design docs and implementation
so Codex (frontend and backend) has an unambiguous target.

## Related Docs

- [`report-center-spec.md`](../../03-spec/report-center-spec.md)
- [`report-center-architecture.md`](../../04-architecture/report-center-architecture.md)
- [`report-center-data-flow.md`](../../04-architecture/report-center-data-flow.md)
- [`report-center-data-model.md`](../../04-architecture/report-center-data-model.md)
- [`report-center-design.md`](../report-center-design.md)

---

## 1. Conventions

- All endpoints are under `/api/v1/reports/**`.
- All responses use the existing envelope `{ data, error }` from
  `shared/dto/ApiResponse.java`.
- All timestamps are ISO-8601 (UTC).
- JSON field names are **camelCase** to match frontend TypeScript types
  exactly.
- All endpoints require an authenticated caller (same mechanism used by
  existing slices).
- Scope enforcement is centralized in `ScopeAuthGuard`.

### 1.1 Error codes

| Code | Meaning |
|------|---------|
| `VALIDATION_ERROR` | Request body invalid; field details in `message` |
| `FORBIDDEN_SCOPE` | Caller cannot access the requested scope |
| `REPORT_NOT_FOUND` | Unknown `reportKey` |
| `EXPORT_NOT_FOUND` | Unknown `exportId` or not owned by caller |
| `EXPORT_TOO_LARGE` | CSV > 100k rows |
| `RATE_LIMITED` | Per-user concurrent run or export cap exceeded |
| `INTERNAL_ERROR` | Unhandled server exception |

### 1.2 Common headers

- `X-Report-Slow: true` — set on `POST /run` responses that exceeded the
  p95 budget (2.5s). Frontend uses this to show the slow banner.

---

## 2. GET /api/v1/reports/catalog

List enabled reports the caller can access.

### 2.1 Request

```
GET /api/v1/reports/catalog HTTP/1.1
Authorization: Bearer <token>
```

### 2.2 Response — 200

```json
{
  "data": {
    "categories": [
      {
        "category": "efficiency",
        "label": "Efficiency",
        "reports": [
          {
            "reportKey": "eff.lead-time",
            "category": "efficiency",
            "name": "Delivery Lead Time",
            "description": "Time from requirement ready to deploy.",
            "supportedScopes": ["org", "workspace", "project"],
            "supportedGroupings": ["team", "project", "requirementType"],
            "defaultGrouping": "team",
            "chartType": "histogram",
            "drilldownColumns": [
              { "key": "team",  "label": "Team",          "type": "string",   "format": null },
              { "key": "p50",   "label": "Median (min)",  "type": "duration", "format": "minutes" },
              { "key": "p75",   "label": "p75 (min)",     "type": "duration", "format": "minutes" },
              { "key": "p95",   "label": "p95 (min)",     "type": "duration", "format": "minutes" }
            ],
            "status": "enabled"
          },
          {
            "reportKey": "eff.cycle-time",
            "category": "efficiency",
            "name": "Cycle Time by Stage",
            "description": "Time in each SDLC stage.",
            "supportedScopes": ["org", "workspace", "project"],
            "supportedGroupings": ["stage", "team", "project"],
            "defaultGrouping": "stage",
            "chartType": "stacked-bar",
            "drilldownColumns": [
              { "key": "stage", "label": "Stage", "type": "string", "format": null },
              { "key": "team",  "label": "Team",  "type": "string", "format": null },
              { "key": "avgMin","label": "Avg (min)", "type": "duration", "format": "minutes" }
            ],
            "status": "enabled"
          },
          {
            "reportKey": "eff.throughput",
            "category": "efficiency",
            "name": "Throughput",
            "description": "Items completed per week.",
            "supportedScopes": ["org", "workspace", "project"],
            "supportedGroupings": ["week-team", "week-project"],
            "defaultGrouping": "week-team",
            "chartType": "grouped-bar",
            "drilldownColumns": [
              { "key": "weekStart", "label": "Week",   "type": "date",   "format": "iso8601" },
              { "key": "group",     "label": "Team",   "type": "string", "format": null },
              { "key": "count",     "label": "Count",  "type": "number", "format": null }
            ],
            "status": "enabled"
          },
          {
            "reportKey": "eff.wip",
            "category": "efficiency",
            "name": "Work In Progress",
            "description": "Active items per stage with aging buckets.",
            "supportedScopes": ["org", "workspace", "project"],
            "supportedGroupings": ["stage-team", "owner-team"],
            "defaultGrouping": "stage-team",
            "chartType": "heatmap",
            "drilldownColumns": [
              { "key": "stage",    "label": "Stage",     "type": "string", "format": null },
              { "key": "team",     "label": "Team",      "type": "string", "format": null },
              { "key": "ageBucket","label": "Age",       "type": "string", "format": null },
              { "key": "count",    "label": "Count",     "type": "number", "format": null }
            ],
            "status": "enabled"
          },
          {
            "reportKey": "eff.flow-efficiency",
            "category": "efficiency",
            "name": "Flow Efficiency",
            "description": "Active work time ÷ total cycle time per stage.",
            "supportedScopes": ["org", "workspace", "project"],
            "supportedGroupings": ["stage", "team"],
            "defaultGrouping": "stage",
            "chartType": "horizontal-bar",
            "drilldownColumns": [
              { "key": "stage",        "label": "Stage",       "type": "string", "format": null },
              { "key": "flowEff",      "label": "Flow Eff.",   "type": "number", "format": "percent-1" },
              { "key": "activeMin",    "label": "Active (min)","type": "duration","format": "minutes" },
              { "key": "totalMin",     "label": "Total (min)", "type": "duration","format": "minutes" }
            ],
            "status": "enabled"
          }
        ]
      },
      { "category": "quality",         "label": "Quality",         "reports": [] },
      { "category": "stability",       "label": "Stability",       "reports": [] },
      { "category": "governance",      "label": "Governance",      "reports": [] },
      { "category": "ai-contribution", "label": "AI Contribution", "reports": [] }
    ]
  },
  "error": null
}
```

V2 categories return empty `reports` arrays — the frontend renders them
as "Coming soon" placeholders (RPT-S01).

---

## 3. POST /api/v1/reports/{reportKey}/run

Run a report and return the rendered result.

### 3.1 Request

```http
POST /api/v1/reports/eff.lead-time/run HTTP/1.1
Content-Type: application/json
Authorization: Bearer <token>

{
  "scope": "workspace",
  "scopeIds": ["ws-alpha", "ws-beta"],
  "timeRange": { "preset": "last30d" },
  "grouping": "team",
  "extraFilters": {}
}
```

### 3.2 Response — 200 (happy)

```json
{
  "data": {
    "reportKey": "eff.lead-time",
    "snapshotAt": "2026-04-18T10:04:12Z",
    "scope": "workspace",
    "scopeIds": ["ws-alpha", "ws-beta"],
    "timeRange": { "preset": "last30d", "startAt": "2026-03-19T00:00:00Z", "endAt": "2026-04-18T00:00:00Z" },
    "grouping": "team",
    "headline": {
      "data": [
        { "key": "p50",  "label": "Median lead time", "value": "3d 4h",  "numericValue": 4560, "trend": -8.2, "trendIsPositive": true },
        { "key": "p95",  "label": "p95 lead time",    "value": "9d 2h",  "numericValue": 13200,"trend":  2.1, "trendIsPositive": false },
        { "key": "vol",  "label": "Items completed",  "value": "184",    "numericValue": 184,  "trend":  6.4, "trendIsPositive": true }
      ],
      "error": null
    },
    "series": {
      "data": [
        { "groupKey": "team-alpha", "groupLabel": "Team Alpha", "x": 0,     "y":  5 },
        { "groupKey": "team-alpha", "groupLabel": "Team Alpha", "x": 1440,  "y": 22 },
        { "groupKey": "team-alpha", "groupLabel": "Team Alpha", "x": 2880,  "y": 31 },
        { "groupKey": "team-beta",  "groupLabel": "Team Beta",  "x": 0,     "y":  2 },
        { "groupKey": "team-beta",  "groupLabel": "Team Beta",  "x": 1440,  "y": 17 }
      ],
      "error": null
    },
    "drilldown": {
      "data": {
        "columns": [
          { "key": "team",  "label": "Team",         "type": "string",   "format": null },
          { "key": "p50",   "label": "Median (min)", "type": "duration", "format": "minutes" },
          { "key": "p75",   "label": "p75 (min)",    "type": "duration", "format": "minutes" },
          { "key": "p95",   "label": "p95 (min)",    "type": "duration", "format": "minutes" }
        ],
        "rows": [
          { "team": "Team Alpha", "p50": 4200, "p75": 8100,  "p95": 13200 },
          { "team": "Team Beta",  "p50": 5100, "p75": 9300,  "p95": 15100 }
        ],
        "totalRows": 2
      },
      "error": null
    },
    "slow": false
  },
  "error": null
}
```

### 3.3 Response — 200 (section-level partial failure)

```json
{
  "data": {
    "reportKey": "eff.lead-time",
    "snapshotAt": "2026-04-18T10:04:12Z",
    "scope": "workspace",
    "scopeIds": ["ws-alpha"],
    "timeRange": { "preset": "last30d", "startAt": "2026-03-19T00:00:00Z", "endAt": "2026-04-18T00:00:00Z" },
    "grouping": "team",
    "headline":  { "data": [ /* ... */ ], "error": null },
    "series":    { "data": null,           "error": "Chart series builder failed: unsupported bucket granularity" },
    "drilldown": { "data": { /* ... */ }, "error": null },
    "slow": false
  },
  "error": null
}
```

### 3.4 Response — 400 (validation)

```json
{
  "data": null,
  "error": { "code": "VALIDATION_ERROR", "message": "grouping \"owner-team\" is not supported for reportKey=eff.lead-time" }
}
```

### 3.5 Response — 403 (scope forbidden)

```json
{
  "data": null,
  "error": { "code": "FORBIDDEN_SCOPE", "message": "caller cannot access workspace ws-unknown" }
}
```

---

## 4. POST /api/v1/reports/{reportKey}/export

Start an async export job. Returns 202.

### 4.1 Request

```http
POST /api/v1/reports/eff.lead-time/export?format=pdf HTTP/1.1
Content-Type: application/json

{
  "scope": "workspace",
  "scopeIds": ["ws-alpha"],
  "timeRange": { "preset": "last30d" },
  "grouping": "team",
  "extraFilters": {}
}
```

### 4.2 Response — 202

```json
{
  "data": {
    "id": "exp-01HXYZ1234",
    "reportKey": "eff.lead-time",
    "format": "pdf",
    "status": "queued",
    "downloadUrl": null,
    "errorMessage": null,
    "bytes": null,
    "createdAt": "2026-04-18T10:04:30Z",
    "readyAt": null,
    "expiresAt": "2026-04-25T10:04:30Z"
  },
  "error": null
}
```

### 4.3 Response — 413 (CSV too large)

```json
{
  "data": null,
  "error": { "code": "EXPORT_TOO_LARGE", "message": "Result set has 142003 rows; max allowed is 100000. Please narrow filters." }
}
```

---

## 5. GET /api/v1/reports/exports/{exportId}

Poll export status.

### 5.1 Response — 200 (ready)

```json
{
  "data": {
    "id": "exp-01HXYZ1234",
    "reportKey": "eff.lead-time",
    "format": "pdf",
    "status": "ready",
    "downloadUrl": "/api/v1/reports/exports/exp-01HXYZ1234/file?sig=...&exp=1745928900",
    "errorMessage": null,
    "bytes": 482113,
    "createdAt": "2026-04-18T10:04:30Z",
    "readyAt":   "2026-04-18T10:04:51Z",
    "expiresAt": "2026-04-25T10:04:30Z"
  },
  "error": null
}
```

### 5.2 Response — 200 (failed)

```json
{
  "data": {
    "id": "exp-01HXYZ1234",
    "reportKey": "eff.lead-time",
    "format": "pdf",
    "status": "failed",
    "downloadUrl": null,
    "errorMessage": "PDF rendering failed: out of memory",
    "bytes": null,
    "createdAt": "2026-04-18T10:04:30Z",
    "readyAt":   null,
    "expiresAt": "2026-04-25T10:04:30Z"
  },
  "error": null
}
```

---

## 6. GET /api/v1/reports/history

List the caller's recent 50 report runs.

### 6.1 Query params

- `reportKey` (optional) — filter to a single report

### 6.2 Response — 200

```json
{
  "data": [
    {
      "runId": "run-01HAA1",
      "reportKey": "eff.lead-time",
      "reportName": "Delivery Lead Time",
      "scope": "workspace",
      "scopeSummary": "Workspace: 2",
      "timeRangeLabel": "Last 30 days",
      "grouping": "team",
      "runAt": "2026-04-18T10:04:12Z",
      "durationMs": 1412
    },
    {
      "runId": "run-01HAA0",
      "reportKey": "eff.throughput",
      "reportName": "Throughput",
      "scope": "project",
      "scopeSummary": "Project: 4",
      "timeRangeLabel": "Last 7 days",
      "grouping": "week-team",
      "runAt": "2026-04-17T14:18:05Z",
      "durationMs": 883
    }
  ],
  "error": null
}
```

---

## 7. GET /api/v1/reports/exports

List the caller's exports in the last 7 days.

### 7.1 Response — 200

```json
{
  "data": [
    {
      "exportId": "exp-01HXYZ1234",
      "reportKey": "eff.lead-time",
      "reportName": "Delivery Lead Time",
      "format": "pdf",
      "status": "ready",
      "createdAt": "2026-04-18T10:04:30Z",
      "expiresAt": "2026-04-25T10:04:30Z",
      "downloadUrl": "/api/v1/reports/exports/exp-01HXYZ1234/file?sig=...",
      "bytes": 482113
    }
  ],
  "error": null
}
```

---

## 8. Backend Implementation Skeleton

```java
// ReportCenterController.java
@RestController
@RequestMapping(ApiConstants.REPORTS_BASE)  // "/api/v1/reports"
public class ReportCenterController {
  private final ReportCatalogService catalog;
  private final ReportRunService runs;
  private final ReportExportService exports;
  private final ReportHistoryService history;

  public ReportCenterController(ReportCatalogService c, ReportRunService r,
                                ReportExportService e, ReportHistoryService h) {
    this.catalog = c; this.runs = r; this.exports = e; this.history = h;
  }

  @GetMapping("/catalog")
  public ApiResponse<CatalogDto> getCatalog() {
    return ApiResponse.ok(catalog.getCatalogForCaller());
  }

  @PostMapping("/{reportKey}/run")
  public ApiResponse<ReportRunResultDto> run(
      @PathVariable String reportKey,
      @Valid @RequestBody ReportRunRequestDto req) {
    return ApiResponse.ok(runs.run(reportKey, req));
  }

  @PostMapping("/{reportKey}/export")
  public ResponseEntity<ApiResponse<ExportJobDto>> export(
      @PathVariable String reportKey,
      @Valid @RequestBody ReportRunRequestDto req,
      @RequestParam String format) {
    ExportJobDto job = exports.enqueue(reportKey, req, format);
    return ResponseEntity.accepted().body(ApiResponse.ok(job));
  }

  @GetMapping("/exports/{exportId}")
  public ApiResponse<ExportJobDto> getExport(@PathVariable String exportId) {
    return ApiResponse.ok(exports.getForCaller(exportId));
  }

  @GetMapping("/history")
  public ApiResponse<List<ReportRunHistoryEntryDto>> getHistory(
      @RequestParam(required = false) String reportKey) {
    return ApiResponse.ok(history.forCaller(reportKey));
  }

  @GetMapping("/exports")
  public ApiResponse<List<ReportExportHistoryEntryDto>> getExportHistory() {
    return ApiResponse.ok(history.exportsForCaller());
  }
}
```

---

## 9. Frontend API Client

Location: `frontend/src/features/reportcenter/api/reportCenterApi.ts`

```typescript
import { fetchJson } from "@/shared/api/fetchJson";
import type {
  CatalogDto, ReportRunRequest, ReportRunResult,
  ExportJobDto, ReportRunHistoryEntry, ReportExportHistoryEntry, ExportFormat
} from "../types";

const BASE = "/api/v1/reports";

export const reportCenterApi = {
  getCatalog: () =>
    fetchJson<CatalogDto>(`${BASE}/catalog`),

  runReport: (reportKey: string, request: ReportRunRequest) =>
    fetchJson<ReportRunResult>(`${BASE}/${reportKey}/run`, {
      method: "POST",
      body: JSON.stringify(request),
    }),

  exportReport: (reportKey: string, request: ReportRunRequest, format: ExportFormat) =>
    fetchJson<ExportJobDto>(`${BASE}/${reportKey}/export?format=${format}`, {
      method: "POST",
      body: JSON.stringify(request),
    }),

  getExport: (exportId: string) =>
    fetchJson<ExportJobDto>(`${BASE}/exports/${exportId}`),

  getHistory: (reportKey?: string) => {
    const qs = reportKey ? `?reportKey=${encodeURIComponent(reportKey)}` : "";
    return fetchJson<ReportRunHistoryEntry[]>(`${BASE}/history${qs}`);
  },

  getExportHistory: () =>
    fetchJson<ReportExportHistoryEntry[]>(`${BASE}/exports`),
};
```

### 9.1 Pinia store outline

```typescript
// reportCenterStore.ts
export const useReportCenterStore = defineStore("reportCenter", {
  state: () => ({
    useMockData: false as boolean,
    catalog: null as CatalogDto | null,
    activeRequest: null as ReportRunRequest | null,
    activeResult: null as ReportRunResult | null,
    runLoading: false,
    runError: null as string | null,
    history: [] as ReportRunHistoryEntry[],
    exports: [] as ReportExportHistoryEntry[],
    exportJob: null as ExportJobDto | null,
  }),
  actions: {
    async fetchCatalog() { /* ... */ },
    async runReport(key: string, req: ReportRunRequest) { /* ... */ },
    async requestExport(key: string, req: ReportRunRequest, fmt: ExportFormat) { /* ... */ },
    async pollExport(id: string) { /* 2s interval, 30s cap */ },
    async fetchHistory(key?: string) { /* ... */ },
    async fetchExportHistory() { /* ... */ },
  },
});
```

### 9.2 Vite proxy

`frontend/vite.config.ts` already proxies `/api/v1/**` to the Spring dev
server. No additional proxy rule needed.

---

## 10. Testing Contracts

### 10.1 Backend — `ReportCenterControllerTest` (MockMvc)

- `GET /catalog` returns 200 with all 5 enabled efficiency reports
- `POST /eff.lead-time/run` with valid payload returns 200 + envelope + 3 sections
- `POST /eff.lead-time/run` with unknown grouping → 400 `VALIDATION_ERROR`
- `POST /eff.lead-time/run` with inaccessible scope → 403 `FORBIDDEN_SCOPE`
- `POST /unknown/run` → 404 `REPORT_NOT_FOUND`
- `POST /eff.lead-time/export?format=csv` returns 202 + `ExportJobDto`
- CSV size > 100k → 413 `EXPORT_TOO_LARGE`
- `GET /exports/unknown` → 404 `EXPORT_NOT_FOUND`
- `GET /history` returns only caller's runs, ≤ 50

### 10.2 Backend — `ReportRunServiceTest`

- Each of the 5 report definitions with synthetic fact data
- Scope resolution narrows to caller's accessible IDs
- `snapshotAt` is within 5s of now

### 10.3 Backend — `ExportWorkerTest`

- CSV generation writes all drilldown rows in order
- PDF generation renders valid PDF (signature `%PDF-`)
- Worker exception → status = `failed`, no audit event
- Success → audit event emitted with correct `attrs`

### 10.4 Contract fixtures

Check in JSON fixtures at
`backend/src/test/resources/contracts/report-center/`:

- `catalog.schema.json`
- `run-lead-time.fixture.json`
- `export-job.fixture.json`
- `history.fixture.json`

Controller tests assert responses match these schemas, and the frontend
type definitions must stay in lockstep.

### 10.5 Frontend — unit

Per `report-center-design.md §10.1`.

### 10.6 Frontend — e2e (Playwright)

Deferred to V2; V1 relies on unit + MockMvc contract tests.

---

## 11. Versioning

- All endpoints are under `/api/v1/`. Breaking changes require `/api/v2/`.
- The `ReportRunResult` shape uses `SectionResult<T>` so adding sections
  is **backward-compatible**; removing sections is breaking.
- Adding new report definitions is non-breaking (new rows in catalog).
- Changing a report's `chartType` or removing a grouping is breaking for
  the frontend — coordinate with a frontend release.

---

## 12. Traceability

| Guide section | Requirement IDs | Spec section |
|---------------|-----------------|--------------|
| §2 Catalog | REQ-RPT-10..15 | §3, §4, §6.1 |
| §3 Run | REQ-RPT-20..32 | §4, §6.2 |
| §4 Export | REQ-RPT-40..43 | §6.3, §6.5 |
| §5 Export poll | REQ-RPT-41 | §5.2 |
| §6 History | REQ-RPT-50 | §4, §6.4 |
| §7 Exports list | REQ-RPT-51 | §4, §6.4 |
| §8 Backend skeleton | — | §1 |
| §9 Frontend client | — | §9.3 (spec) |
| §10 Tests | REQ-RPT-42, REQ-RPT-82 | §7 |
| §11 Versioning | — | §1 |
