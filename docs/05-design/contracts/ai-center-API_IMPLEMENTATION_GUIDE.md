# AI Center — API Implementation Guide

## Purpose

Full endpoint contract for AI Center. This is the single source of truth for request/response shapes that FE and BE implementations must match exactly.

- FE (Phase A): mocks must satisfy these shapes so Phase B can swap to live without component changes.
- BE (Phase B): controllers and DTOs must produce these shapes.

## Source

- [ai-center-spec.md](../../03-spec/ai-center-spec.md)
- [ai-center-data-model.md](../../04-architecture/ai-center-data-model.md)
- [ai-center-design.md](../ai-center-design.md)

---

## 1. Shared Envelope and Conventions

- All responses use `ApiResponse<T>`:
  ```json
  { "data": <T | null>, "error": "<string | null>" }
  ```
- Top-level `error` = overall failure; `data` is populated only when error is null.
- HTTP status codes:
  - 200 — success (even if a nested `SectionResult` has errors)
  - 400 — bad request (invalid query params, e.g. `size > 200`)
  - 404 — entity not found by id/key
  - 500 — uncaught server error

- camelCase JSON for all field names.
- `Instant` fields serialized as ISO-8601 UTC with `Z` suffix (e.g., `2026-04-17T12:34:56Z`).
- `null` is always explicit.
- Workspace context propagated via request header `X-Workspace-Id` (reuse the convention from the shell slice — confirm in implementation). Missing header → 400.

---

## 2. Endpoint Reference

### 2.1 `GET /api/v1/ai-center/metrics`

**Query Parameters**

| Param | Type | Required | Default | Notes |
|---|---|---|---|---|
| `window` | enum `24h`\|`7d`\|`30d` | no | `30d` | — |

**Response**

```json
{
  "data": {
    "window": "30d",
    "aiUsageRate": {
      "data": { "value": 62.4, "unit": "%", "delta": 4.1, "trend": "up", "isPositive": true },
      "error": null
    },
    "adoptionRate": {
      "data": { "value": 71.8, "unit": "%", "delta": -2.3, "trend": "down", "isPositive": false },
      "error": null
    },
    "autoExecSuccessRate": {
      "data": { "value": 88.5, "unit": "%", "delta": 0.0, "trend": "flat", "isPositive": true },
      "error": null
    },
    "timeSavedHours": {
      "data": { "value": 142.0, "unit": "hours", "delta": 18.0, "trend": "up", "isPositive": true },
      "error": null
    },
    "stageCoverageCount": {
      "data": { "value": 9, "unit": "count", "delta": 1, "trend": "up", "isPositive": true },
      "error": null
    }
  },
  "error": null
}
```

**Error response (partial — per-section)**

```json
{
  "data": {
    "window": "30d",
    "aiUsageRate": { "data": null, "error": "Aggregation timed out" },
    "adoptionRate":  { "data": {...}, "error": null },
    "autoExecSuccessRate": { "data": {...}, "error": null },
    "timeSavedHours": { "data": {...}, "error": null },
    "stageCoverageCount": { "data": {...}, "error": null }
  },
  "error": null
}
```

---

### 2.2 `GET /api/v1/ai-center/stage-coverage`

**Response**

```json
{
  "data": {
    "entries": [
      { "stageKey": "requirement",  "stageLabel": "Requirement",  "activeSkillCount": 2, "covered": true },
      { "stageKey": "user-story",   "stageLabel": "User Story",   "activeSkillCount": 1, "covered": true },
      { "stageKey": "spec",         "stageLabel": "Spec",         "activeSkillCount": 1, "covered": true },
      { "stageKey": "architecture", "stageLabel": "Architecture", "activeSkillCount": 1, "covered": true },
      { "stageKey": "design",       "stageLabel": "Design",       "activeSkillCount": 1, "covered": true },
      { "stageKey": "tasks",        "stageLabel": "Tasks",        "activeSkillCount": 1, "covered": true },
      { "stageKey": "code",         "stageLabel": "Code",         "activeSkillCount": 2, "covered": true },
      { "stageKey": "test",         "stageLabel": "Test",         "activeSkillCount": 0, "covered": false },
      { "stageKey": "deploy",       "stageLabel": "Deploy",       "activeSkillCount": 0, "covered": false },
      { "stageKey": "incident",     "stageLabel": "Incident",     "activeSkillCount": 3, "covered": true },
      { "stageKey": "learning",     "stageLabel": "Learning",     "activeSkillCount": 1, "covered": true }
    ]
  },
  "error": null
}
```

Entries MUST be returned in the canonical 11-stage order regardless of coverage.

---

### 2.3 `GET /api/v1/ai-center/skills`

**Response**

```json
{
  "data": [
    {
      "id": "sk_01HZZZ...",
      "key": "incident-diagnosis",
      "name": "Incident Diagnosis",
      "category": "runtime",
      "subCategory": "Incident",
      "status": "active",
      "defaultAutonomy": "L2-Auto-with-approval",
      "owner": "platform-sre",
      "description": "Performs root cause analysis correlating signals with recent changes.",
      "stages": ["incident"],
      "lastExecutedAt": "2026-04-17T14:02:11Z",
      "successRate30d": 0.91,
      "version": 3
    },
    {
      "id": "sk_01HZZY...",
      "key": "req-to-user-story",
      "name": "Requirement to User Story",
      "category": "delivery",
      "subCategory": "Requirement",
      "status": "active",
      "defaultAutonomy": "L1-Assist",
      "owner": "platform-ai",
      "description": "Transforms raw requirements into Jira-ready user stories.",
      "stages": ["requirement", "user-story"],
      "lastExecutedAt": "2026-04-18T09:11:03Z",
      "successRate30d": 0.88,
      "version": 2
    }
  ],
  "error": null
}
```

Returns all skills for the workspace; no server-side pagination in V1 (≤100 rows).

---

### 2.4 `GET /api/v1/ai-center/skills/{skillKey}`

**Path parameter**: `skillKey` — the skill's `key` (not its UUID).

**Response (200)**

```json
{
  "data": {
    "skill": {
      "id": "sk_01HZZZ...",
      "key": "incident-diagnosis",
      "name": "Incident Diagnosis",
      "category": "runtime",
      "subCategory": "Incident",
      "status": "active",
      "defaultAutonomy": "L2-Auto-with-approval",
      "owner": "platform-sre",
      "description": "Performs root cause analysis correlating signals with recent changes.",
      "stages": ["incident"],
      "lastExecutedAt": "2026-04-17T14:02:11Z",
      "successRate30d": 0.91,
      "version": 3
    },
    "inputContract": "IncidentId, SignalSet, RecentChangeWindow",
    "outputContract": "RootCauseHypothesis, EvidenceSet, ConfidenceScore, RecommendedActions",
    "policy": {
      "skillKey": "incident-diagnosis",
      "autonomyLevel": "L2-Auto-with-approval",
      "approvalRequiredActions": ["restart-service", "scale-up-resources"],
      "authorizedApproverRoles": ["Workspace Admin", "SRE Lead"],
      "riskThresholds": { "blastRadius": "namespace", "maxRevenueImpactUsd": 1000 },
      "lastChangedAt": "2026-04-02T12:00:00Z",
      "lastChangedBy": "platform-admin@corp.example"
    },
    "recentRuns": [
      {
        "id": "run_01HAAAA...",
        "skillKey": "incident-diagnosis",
        "skillName": "Incident Diagnosis",
        "status": "succeeded",
        "triggerSourceType": "page",
        "triggerSourcePage": "/incidents/INC-0422",
        "triggerSourceUrl": "/incidents/INC-0422",
        "triggeredBy": "auto-detector",
        "triggeredByType": "ai",
        "startedAt": "2026-04-17T14:01:42Z",
        "endedAt": "2026-04-17T14:02:11Z",
        "durationMs": 29120,
        "outcomeSummary": "Correlated to deploy DEP-9981 rollback recommended",
        "auditRecordId": "aud_01HBBB..."
      }
    ],
    "aggregateMetrics": {
      "successRate": 0.91,
      "avgDurationMs": 32450,
      "adoptionTrend": "up",
      "totalRuns30d": 44
    }
  },
  "error": null
}
```

**Response (404)**

```json
{ "data": null, "error": "Skill not found: unknown-key" }
```

---

### 2.5 `GET /api/v1/ai-center/runs`

**Query Parameters**

| Param | Type | Required | Default | Notes |
|---|---|---|---|---|
| `skillKey` | string[] (repeatable) | no | all | `?skillKey=a&skillKey=b` |
| `status` | string[] | no | all | from the 6 enum values |
| `triggerSourcePage` | string | no | — | exact match |
| `startedAfter` | ISO instant | no | — | inclusive |
| `startedBefore` | ISO instant | no | — | exclusive |
| `triggeredByType` | `ai`\|`human`\|`system` | no | — | — |
| `page` | int ≥ 1 | no | `1` | 1-indexed |
| `size` | int 1..200 | no | `50` | 400 if out of range |

**Response**

```json
{
  "data": {
    "items": [
      {
        "id": "run_01HAAAA...",
        "skillKey": "incident-diagnosis",
        "skillName": "Incident Diagnosis",
        "status": "succeeded",
        "triggerSourceType": "page",
        "triggerSourcePage": "/incidents/INC-0422",
        "triggerSourceUrl": "/incidents/INC-0422",
        "triggeredBy": "auto-detector",
        "triggeredByType": "ai",
        "startedAt": "2026-04-17T14:01:42Z",
        "endedAt": "2026-04-17T14:02:11Z",
        "durationMs": 29120,
        "outcomeSummary": "Correlated to deploy DEP-9981 rollback recommended",
        "auditRecordId": "aud_01HBBB..."
      }
    ],
    "page": 1,
    "size": 50,
    "total": 182,
    "hasMore": true
  },
  "error": null
}
```

**Bad request (400)**

```json
{ "data": null, "error": "size must be between 1 and 200" }
```

---

### 2.6 `GET /api/v1/ai-center/runs/{executionId}`

**Response (200)**

```json
{
  "data": {
    "run": {
      "id": "run_01HAAAA...",
      "skillKey": "incident-diagnosis",
      "skillName": "Incident Diagnosis",
      "status": "succeeded",
      "triggerSourceType": "page",
      "triggerSourcePage": "/incidents/INC-0422",
      "triggerSourceUrl": "/incidents/INC-0422",
      "triggeredBy": "auto-detector",
      "triggeredByType": "ai",
      "startedAt": "2026-04-17T14:01:42Z",
      "endedAt": "2026-04-17T14:02:11Z",
      "durationMs": 29120,
      "outcomeSummary": "Correlated to deploy DEP-9981 rollback recommended",
      "auditRecordId": "aud_01HBBB..."
    },
    "inputSummary": {
      "incidentId": "INC-0422",
      "signals": ["high-5xx-rate", "p99-latency-spike"],
      "recentChangeWindowMinutes": 60
    },
    "outputSummary": {
      "rootCauseHypothesis": "Deploy DEP-9981 introduced a null-check regression in OrderService",
      "confidence": 0.92,
      "recommendedActions": ["rollback:DEP-9981"]
    },
    "stepBreakdown": [
      { "ordinal": 1, "name": "gather-signals",       "status": "succeeded", "startedAt": "2026-04-17T14:01:42Z", "endedAt": "2026-04-17T14:01:48Z", "durationMs": 6000, "note": null },
      { "ordinal": 2, "name": "correlate-changes",    "status": "succeeded", "startedAt": "2026-04-17T14:01:48Z", "endedAt": "2026-04-17T14:01:58Z", "durationMs": 10000, "note": null },
      { "ordinal": 3, "name": "form-hypothesis",      "status": "succeeded", "startedAt": "2026-04-17T14:01:58Z", "endedAt": "2026-04-17T14:02:08Z", "durationMs": 10000, "note": null },
      { "ordinal": 4, "name": "propose-actions",      "status": "succeeded", "startedAt": "2026-04-17T14:02:08Z", "endedAt": "2026-04-17T14:02:11Z", "durationMs": 3000,  "note": null }
    ],
    "policyTrail": [
      { "rule": "autonomy<=L2",           "decision": "allowed",           "at": "2026-04-17T14:01:42Z", "note": null },
      { "rule": "action=rollback gated",  "decision": "held-for-approval", "at": "2026-04-17T14:02:11Z", "note": "Escalated to SRE Lead" }
    ],
    "evidenceLinks": [
      { "title": "Deploy DEP-9981",                 "type": "deploy",    "sourceSystem": "internal", "url": "/deployments/DEP-9981" },
      { "title": "Grafana latency dashboard (14:00-14:10)", "type": "metric", "sourceSystem": "grafana",  "url": "https://grafana.internal/d/api-latency" },
      { "title": "MR !4123 order-service",          "type": "code",      "sourceSystem": "gitlab",   "url": "https://gitlab.example/org/order-service/-/merge_requests/4123" }
    ],
    "autonomyLevel": "L2-Auto-with-approval",
    "timeSavedMinutes": 25
  },
  "error": null
}
```

**Response (404)**

```json
{ "data": null, "error": "Skill execution not found: unknown-id" }
```

---

## 3. Backend Implementation Guide

### 3.1 Controller skeleton

```java
@RestController
@RequestMapping("/api/v1/ai-center")
public class AiCenterController {

    private final MetricsService metricsService;
    private final SkillService skillService;
    private final SkillExecutionService executionService;

    public AiCenterController(MetricsService metricsService,
                              SkillService skillService,
                              SkillExecutionService executionService) {
        this.metricsService = metricsService;
        this.skillService = skillService;
        this.executionService = executionService;
    }

    @GetMapping("/metrics")
    public ApiResponse<MetricsSummaryDto> metrics(
            @RequestHeader("X-Workspace-Id") String workspaceId,
            @RequestParam(defaultValue = "30d") String window) {
        return ApiResponse.ok(metricsService.summarize(workspaceId, window));
    }

    @GetMapping("/stage-coverage")
    public ApiResponse<StageCoverageDto> stageCoverage(
            @RequestHeader("X-Workspace-Id") String workspaceId) {
        return ApiResponse.ok(metricsService.stageCoverage(workspaceId));
    }

    @GetMapping("/skills")
    public ApiResponse<List<SkillDto>> skills(
            @RequestHeader("X-Workspace-Id") String workspaceId) {
        return ApiResponse.ok(skillService.list(workspaceId));
    }

    @GetMapping("/skills/{skillKey}")
    public ApiResponse<SkillDetailDto> skillDetail(
            @RequestHeader("X-Workspace-Id") String workspaceId,
            @PathVariable String skillKey) {
        return ApiResponse.ok(skillService.detail(workspaceId, skillKey));
    }

    @GetMapping("/runs")
    public ApiResponse<PageDto<RunDto>> runs(
            @RequestHeader("X-Workspace-Id") String workspaceId,
            @RequestParam(required = false) List<String> skillKey,
            @RequestParam(required = false) List<String> status,
            @RequestParam(required = false) String triggerSourcePage,
            @RequestParam(required = false) Instant startedAfter,
            @RequestParam(required = false) Instant startedBefore,
            @RequestParam(required = false) String triggeredByType,
            @RequestParam(defaultValue = "1") int page,
            @RequestParam(defaultValue = "50") int size) {
        if (size < 1 || size > 200) {
            throw new IllegalArgumentException("size must be between 1 and 200");
        }
        RunFilter filter = new RunFilter(skillKey, status, triggerSourcePage,
                                         startedAfter, startedBefore, triggeredByType);
        return ApiResponse.ok(executionService.page(workspaceId, filter, page, size));
    }

    @GetMapping("/runs/{executionId}")
    public ApiResponse<RunDetailDto> runDetail(
            @RequestHeader("X-Workspace-Id") String workspaceId,
            @PathVariable String executionId) {
        return ApiResponse.ok(executionService.detail(workspaceId, executionId));
    }
}
```

### 3.2 Service responsibilities

- **`MetricsService`**
  - `summarize(workspaceId, window)` — returns `MetricsSummaryDto` with per-section `SectionResult`. A single section's computation failure must not bubble up; wrap in `SectionResult.fail(msg)`.
  - `stageCoverage(workspaceId)` — joins `skill` × `skill_stage`, returns all 11 canonical stages with counts.

- **`SkillService`**
  - `list(workspaceId)` — returns all skills with derived `lastExecutedAt` and `successRate30d`.
  - `detail(workspaceId, skillKey)` — 404s via `SkillNotFoundException` if missing; assembles policy, recent 10 runs, aggregate metrics.

- **`SkillExecutionService`**
  - `page(workspaceId, filter, page, size)` — builds dynamic predicate (Specification API or JPQL) for filters; returns `PageDto<RunDto>`.
  - `detail(workspaceId, executionId)` — 404s via `SkillExecutionNotFoundException`.
  - `record(...)` — exposed to OTHER domains (Incident, Requirement) to record new executions. Not called from AI Center controllers in V1.

### 3.3 Exception mapping

Add to the existing `GlobalExceptionHandler`:

```java
@ExceptionHandler(SkillNotFoundException.class)
public ResponseEntity<ApiResponse<Void>> handleSkillNotFound(SkillNotFoundException e) {
    return ResponseEntity.status(404).body(ApiResponse.fail(e.getMessage()));
}

@ExceptionHandler(SkillExecutionNotFoundException.class)
public ResponseEntity<ApiResponse<Void>> handleExecNotFound(SkillExecutionNotFoundException e) {
    return ResponseEntity.status(404).body(ApiResponse.fail(e.getMessage()));
}

@ExceptionHandler(IllegalArgumentException.class)
public ResponseEntity<ApiResponse<Void>> handleBadRequest(IllegalArgumentException e) {
    return ResponseEntity.status(400).body(ApiResponse.fail(e.getMessage()));
}
```

### 3.4 Tests (MockMvc)

Create `AiCenterControllerTest.java`:

- `metrics_returns200_withAllFiveSections()`
- `stageCoverage_returnsAll11Stages_inCanonicalOrder()`
- `skills_returnsListForWorkspace()`
- `skills_respectWorkspaceIsolation()` — seed WS-A + WS-B, request WS-A, assert no WS-B rows
- `skillDetail_returns200_forExistingKey()`
- `skillDetail_returns404_forUnknownKey()`
- `runs_returns200_withPage()`
- `runs_returns400_forSize201()`
- `runs_filtersByStatus()` / `runs_filtersBySkillKey()` / `runs_filtersByTimeRange()`
- `runDetail_returns200_withEvidence_and_steps()`
- `runDetail_returns404_forUnknownId()`

Plus a `@DataJpaTest` repository test for workspace isolation and a Flyway migration test (`MigrationTest`) verifying clean apply from empty schema.

### 3.5 Flyway migrations

Place schema in `db/migration/V{n}__ai_center_schema.sql` using the next available version number. Seed in `db/seed/V{n+1}__ai_center_seed.sql`; include the `seed` folder only for `local` and `dev` profiles via `spring.flyway.locations`.

---

## 4. Frontend Implementation Guide

### 4.1 API client (`aiCenterApi.ts`)

```ts
import { fetchJson } from "@/shared/api/fetchJson";
import type { ApiResponse } from "@/shared/api/types";
import type {
  MetricsSummary, StageCoverage, Skill, SkillDetail, Run, RunDetail, Page
} from "../types";

const BASE = "/api/v1/ai-center";

export async function getMetrics(workspaceId: string, window: "24h"|"7d"|"30d" = "30d")
  : Promise<MetricsSummary> {
  const res = await fetchJson<ApiResponse<MetricsSummary>>(
    `${BASE}/metrics?window=${window}`,
    { headers: { "X-Workspace-Id": workspaceId } }
  );
  if (res.error || !res.data) throw new Error(res.error ?? "No data");
  return res.data;
}

export async function getStageCoverage(workspaceId: string): Promise<StageCoverage> {
  const res = await fetchJson<ApiResponse<{ entries: StageCoverage }>>(
    `${BASE}/stage-coverage`,
    { headers: { "X-Workspace-Id": workspaceId } }
  );
  if (res.error || !res.data) throw new Error(res.error ?? "No data");
  return res.data.entries;
}

export async function getSkills(workspaceId: string): Promise<Skill[]> {
  const res = await fetchJson<ApiResponse<Skill[]>>(
    `${BASE}/skills`,
    { headers: { "X-Workspace-Id": workspaceId } }
  );
  if (res.error || !res.data) throw new Error(res.error ?? "No data");
  return res.data;
}

export async function getSkillDetail(workspaceId: string, skillKey: string): Promise<SkillDetail> {
  const res = await fetchJson<ApiResponse<{ skill: Skill } & Omit<SkillDetail, keyof Skill>>>(
    `${BASE}/skills/${encodeURIComponent(skillKey)}`,
    { headers: { "X-Workspace-Id": workspaceId } }
  );
  if (res.error || !res.data) throw new Error(res.error ?? "No data");
  const { skill, ...rest } = res.data;
  return { ...skill, ...rest };
}

export interface RunQuery {
  skillKey?: string[];
  status?: string[];
  triggerSourcePage?: string;
  startedAfter?: string;
  startedBefore?: string;
  triggeredByType?: "ai"|"human"|"system";
  page?: number;
  size?: number;
}

export async function getRuns(workspaceId: string, q: RunQuery = {}): Promise<Page<Run>> {
  const qs = new URLSearchParams();
  (q.skillKey ?? []).forEach(v => qs.append("skillKey", v));
  (q.status ?? []).forEach(v => qs.append("status", v));
  if (q.triggerSourcePage) qs.set("triggerSourcePage", q.triggerSourcePage);
  if (q.startedAfter) qs.set("startedAfter", q.startedAfter);
  if (q.startedBefore) qs.set("startedBefore", q.startedBefore);
  if (q.triggeredByType) qs.set("triggeredByType", q.triggeredByType);
  qs.set("page", String(q.page ?? 1));
  qs.set("size", String(q.size ?? 50));

  const res = await fetchJson<ApiResponse<Page<Run>>>(
    `${BASE}/runs?${qs}`,
    { headers: { "X-Workspace-Id": workspaceId } }
  );
  if (res.error || !res.data) throw new Error(res.error ?? "No data");
  return res.data;
}

export async function getRunDetail(workspaceId: string, executionId: string): Promise<RunDetail> {
  const res = await fetchJson<ApiResponse<{ run: Run } & Omit<RunDetail, keyof Run>>>(
    `${BASE}/runs/${encodeURIComponent(executionId)}`,
    { headers: { "X-Workspace-Id": workspaceId } }
  );
  if (res.error || !res.data) throw new Error(res.error ?? "No data");
  const { run, ...rest } = res.data;
  return { ...run, ...rest };
}
```

### 4.2 Store (`aiCenterStore.ts`)

Pinia store with:

- State: `metrics`, `stageCoverage`, `skills`, `runsPage`, `skillDetail`, `runDetail` — each with `{ data, loading, error }` shape.
- Actions: `init(workspaceId)`, `refetchMetrics`, `refetchSkills`, `refetchRuns`, `refetchStageCoverage`, `selectSkill`, `selectRun`, `setRunFilters`, `setRunPage`.
- Getters: `filteredSkills` (applies client-side filter + search), `runFiltersApplied`.

### 4.3 Mock mode

Mocks live in `api/mocks.ts`. `aiCenterApi.ts` gates via `import.meta.env.VITE_USE_MOCK_API === "true"` (reuse dashboard slice convention — verify key name). Mock responses MUST exactly match the JSON shapes in this document.

### 4.4 Vite proxy

Verify `vite.config.ts` proxy covers `/api/v1/ai-center` — the existing `/api` rule likely covers it. If not, add:

```ts
"/api/v1/ai-center": { target: "http://localhost:8080", changeOrigin: true }
```

### 4.5 FE tests (Vitest)

- `aiCenterStore.test.ts` — init spawns 4 fetches, workspace switch resets, filter state transitions
- `useSkillFilters.test.ts` — category/status/autonomy/owner filters + search
- Component tests: each card in all 5 states (Normal / Loading / Empty / Error / Partial)

---

## 5. Testing Contract

| Test | Frontend | Backend |
|---|---|---|
| Happy path render | component tests with mocks per endpoint shape | MockMvc 200 with seed data |
| Empty state | mock empty responses | seed with workspace that has no skills |
| Error state | mock reject | inject 500 via stubbed service in a slice test |
| Workspace isolation | — | MockMvc with `X-Workspace-Id: WS-A` ≠ `WS-B` |
| 404 on unknown key/id | mock rejected | actual JPA `Optional.empty()` path |
| Pagination bounds | — | `size=201` → 400 |
| Filter combinations | store tests | MockMvc for each filter |

---

## 6. Versioning Policy

- URL version `/api/v1/ai-center/...` is the stable V1 contract.
- Additive fields on existing endpoints are non-breaking (FE must tolerate unknown extra fields).
- Removing or renaming a field requires `v2` routing.
- Changing a field's type or semantics requires `v2` routing.
- Enum additions (e.g., new `status` value) must be listed here before backend returns them so FE can render them safely.

---

## 7. Non-Requirements

- No websocket/SSE endpoints in V1.
- No write endpoints in V1.
- No role-based authz beyond workspace membership.
- No request body validation framework (Bean Validation) in V1 — GET-only API.
