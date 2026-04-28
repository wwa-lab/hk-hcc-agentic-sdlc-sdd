# AI Center — Data Flow

## Purpose

This document captures the **runtime** data flows of AI Center: how data moves through the system on page load, during navigation, under failure, and during refresh. It complements the architecture doc (structure) with sequence and state diagrams.

## Source

- [ai-center-architecture.md](ai-center-architecture.md)
- [ai-center-spec.md](../03-spec/ai-center-spec.md)

---

## 1. Flow Inventory

| # | Flow | Trigger | Phase A | Phase B |
|---|---|---|---|---|
| 1 | Page initial load | Navigate to `/ai-center` | Mocks | Live API |
| 2 | Skill detail drill-down | Click skill row / deep-link | Mocks | Live API |
| 3 | Run detail drill-down | Click run row / deep-link | Mocks | Live API |
| 4 | Filter / search (catalog) | Filter chip / search input | Client-only | Client-only |
| 5 | Filter (run history) | Run filter chip | Mock refetch | Server refetch |
| 6 | Manual refresh | Click "Refresh" on runs card | Mock refetch | Server refetch |
| 7 | Workspace switch | Shell workspace change event | Mocks re-init | API re-init |
| 8 | Error / retry (card) | Retry click on errored card | Mock refetch | Live refetch |
| 9 | Cross-page deep-link | Dashboard Learning → `/ai-center` | Route-level handoff | Route-level handoff |
| 10 | Drill-back to source | Click "Go to source" on run | Route transition | Route transition |

---

## 2. Page Load (Phase A — Mocks)

```mermaid
sequenceDiagram
  participant U as User
  participant Shell as Shell Router
  participant View as AiCenterView
  participant Store as aiCenterStore
  participant Api as aiCenterApi (mock mode)
  participant Mocks as mocks.ts

  U->>Shell: Navigate /ai-center
  Shell->>View: mount
  View->>Store: init(workspaceId)
  Store->>Store: set loading=true on all 4 sections
  par Fetch metrics
    Store->>Api: getMetrics(ws, 30d)
    Api->>Mocks: seedMetrics(ws)
    Mocks-->>Api: MetricsDto
    Api-->>Store: ok
  and Fetch skills
    Store->>Api: getSkills(ws)
    Api->>Mocks: seedSkills(ws)
    Mocks-->>Api: SkillDto[]
    Api-->>Store: ok
  and Fetch runs
    Store->>Api: getRuns(ws, {page:1,size:50})
    Api->>Mocks: seedRuns(ws, 1, 50)
    Mocks-->>Api: Page<RunDto>
    Api-->>Store: ok
  and Fetch coverage
    Store->>Api: getStageCoverage(ws)
    Api->>Mocks: seedStageCoverage(ws)
    Mocks-->>Api: StageCoverageDto
    Api-->>Store: ok
  end
  Store-->>View: state ready (per-card)
  View-->>U: render 4 cards
```

---

## 3. Page Load (Phase B — Live API)

```mermaid
sequenceDiagram
  participant U as User
  participant View as AiCenterView
  participant Store as aiCenterStore
  participant Api as aiCenterApi (live)
  participant BE as AiCenterController
  participant Svc as Services
  participant DB as DB

  U->>View: Navigate /ai-center
  View->>Store: init(workspaceId)
  par
    Store->>Api: GET /api/v1/ai-center/metrics?window=30d
    Api->>BE: HTTP GET
    BE->>Svc: MetricsService.aggregate(ws, 30d)
    Svc->>DB: SELECT ... WHERE workspace_id=ws
    DB-->>Svc: rows
    Svc-->>BE: MetricsDto with SectionResults
    BE-->>Api: ApiResponse<MetricsDto>
    Api-->>Store: typed result
  and
    Store->>Api: GET /api/v1/ai-center/skills
    Api->>BE: HTTP GET
    BE->>Svc: SkillService.list(ws)
    Svc->>DB: SELECT * FROM skill WHERE workspace_id=ws
    DB-->>Svc: rows
    Svc-->>BE: SkillDto[]
    BE-->>Api: ApiResponse<SkillDto[]>
  and
    Store->>Api: GET /api/v1/ai-center/runs?page=1&size=50
    Api->>BE: HTTP GET
    BE->>Svc: SkillExecutionService.page(ws, filters, page, size)
    Svc->>DB: SELECT ... ORDER BY started_at DESC LIMIT/OFFSET
    DB-->>Svc: rows + total
    Svc-->>BE: PageDto<RunDto>
    BE-->>Api: ApiResponse<PageDto<RunDto>>
  and
    Store->>Api: GET /api/v1/ai-center/stage-coverage
    Api->>BE: HTTP GET
    BE->>Svc: MetricsService.stageCoverage(ws)
    Svc->>DB: JOIN skill × skill_stage
    DB-->>Svc: rows
    Svc-->>BE: StageCoverageDto
    BE-->>Api: ApiResponse<StageCoverageDto>
  end
  Store-->>View: state ready
  View-->>U: render 4 cards
```

---

## 4. Skill Detail Flow

```mermaid
sequenceDiagram
  participant U as User
  participant View as SkillCatalogCard
  participant Router as Vue Router
  participant Panel as SkillDetailPanel
  participant Store as aiCenterStore
  participant Api as aiCenterApi
  participant BE as Backend

  U->>View: click skill row
  View->>Router: push(/ai-center/skills/:skillKey)
  Router->>Panel: mount with route param
  Panel->>Store: selectSkill(skillKey)
  alt skill already in cache with full detail
    Store-->>Panel: cached detail
  else detail not cached or stale
    Store->>Api: getSkillDetail(ws, skillKey)
    Api->>BE: GET /api/v1/ai-center/skills/{skillKey}
    BE-->>Api: ApiResponse<SkillDetailDto>
    Api-->>Store: ok
    Store-->>Panel: detail
  end
  Panel-->>U: render description, policy, recent runs, metrics
```

**404 sub-flow:**

```mermaid
sequenceDiagram
  participant Panel as SkillDetailPanel
  participant Api as aiCenterApi
  participant BE as Backend

  Panel->>Api: getSkillDetail(ws, unknownKey)
  Api->>BE: GET /api/v1/ai-center/skills/unknownKey
  BE-->>Api: 404 ApiResponse{error: "Not found"}
  Api-->>Panel: throw NotFoundError
  Panel-->>Panel: show "Skill not found" state with back link
```

---

## 5. Run Detail Flow

```mermaid
sequenceDiagram
  participant U as User
  participant Card as RunHistoryCard
  participant Router as Vue Router
  participant Panel as RunDetailPanel
  participant Store as aiCenterStore
  participant Api as aiCenterApi
  participant BE as Backend

  U->>Card: click run row
  Card->>Router: push(/ai-center/runs/:executionId)
  Router->>Panel: mount
  Panel->>Store: selectRun(executionId)
  Store->>Api: getRunDetail(ws, executionId)
  Api->>BE: GET /api/v1/ai-center/runs/{executionId}
  BE-->>Api: ApiResponse<RunDetailDto>
  Api-->>Store: ok
  Store-->>Panel: detail {inputSummary, steps, output, policyTrail, evidenceLinks, auditRecordId, triggerSource}
  Panel-->>U: render detail + Evidence/Audit/Source buttons
```

---

## 6. Filter and Refresh Behaviors

### 6.1 Catalog — Client-side Filter

```mermaid
sequenceDiagram
  participant U as User
  participant View as SkillCatalogCard
  participant Comp as useSkillFilters
  participant Store as aiCenterStore

  U->>View: toggle category=runtime chip
  View->>Comp: setCategory("runtime")
  Comp->>Store: update filter state
  Store-->>View: derive filteredSkills getter
  View-->>U: re-render list (no network I/O)
```

### 6.2 Run History — Server-side Filter

```mermaid
sequenceDiagram
  participant U as User
  participant View as RunHistoryCard
  participant Store as aiCenterStore
  participant Api as aiCenterApi
  participant BE as Backend

  U->>View: set status=failed + skill=incident-diagnosis
  View->>Store: setRunFilters({status:["failed"], skillKey:["incident-diagnosis"]})
  Store->>Store: reset page to 1
  Store->>Api: getRuns(ws, filters, page=1, size=50)
  Api->>BE: GET /api/v1/ai-center/runs?status=failed&skillKey=incident-diagnosis&page=1&size=50
  BE-->>Api: ApiResponse<PageDto<RunDto>>
  Api-->>Store: page
  Store-->>View: re-render with filtered page
```

### 6.3 Manual Refresh

```mermaid
sequenceDiagram
  participant U as User
  participant View as RunHistoryCard
  participant Store as aiCenterStore

  U->>View: click Refresh
  View->>Store: refetchRuns()
  Note over Store: reuses current filter + page state
  Store->>Store: fetch (same as 6.2 but not resetting page)
```

---

## 7. Error Cascade

```mermaid
sequenceDiagram
  participant Store as aiCenterStore
  participant Api as aiCenterApi
  participant BE as Backend

  Store->>Api: getRuns(ws, filters, 1, 50)
  Api->>BE: GET /runs
  BE-->>Api: 500 Internal Server Error with ApiResponse{error: "..."}
  Api-->>Store: throw ApiError
  Store-->>Store: set runs.section.error = msg
  Store-->>Store: leave other sections untouched
```

**Card rendering behavior:**

| Section error state | Other cards | Card content |
|---|---|---|
| `error` set, `data` null | untouched | error banner + "Retry" button |
| `error` set, `data` prev cached | untouched | stale data banner + "Retry" button |
| `loading` true | untouched | skeleton |
| Both null | untouched | empty state |

---

## 8. State Machine — Run Card (per card)

```mermaid
stateDiagram-v2
  [*] --> Idle
  Idle --> Loading: init() or refetch()
  Loading --> Normal: success (data present)
  Loading --> Empty: success (data empty)
  Loading --> Error: failure
  Normal --> Loading: refetch()
  Empty --> Loading: refetch()
  Error --> Loading: retry()
  Normal --> Normal: filter change (client only)
  Normal --> Loading: filter change (server-side)
```

---

## 9. Workspace Switch

```mermaid
sequenceDiagram
  participant Shell as Shell WorkspaceStore
  participant Store as aiCenterStore
  participant Api as aiCenterApi
  participant BE as Backend

  Shell-->>Store: workspaceId changed (WS-A -> WS-B)
  Store->>Store: clear cached data
  Store->>Store: reset filters + page + selections
  Store->>Api: refetch all 4 (metrics, skills, runs, stage-coverage) with WS-B
  Api->>BE: GET with new workspaceId
  BE-->>Api: WS-B data
  Api-->>Store: ok
  Store-->>Store: state refreshed
```

---

## 10. API Client Chain

```mermaid
graph LR
  Comp[Card component] -->|await store action| Store[aiCenterStore]
  Store -->|call typed fn| Api[aiCenterApi.ts]
  Api -->|fetchJson path, opts| Shared[shared/fetchJson]
  Shared -->|HTTP GET| Proxy[Vite dev proxy]
  Proxy --> BE[AiCenterController]
  BE -->|service call| Svc[Services]
  Svc -->|repo| Repo[JPA Repository]
  Repo --> DB[DB]
```

**Mock mode (Phase A):** `aiCenterApi` short-circuits to `mocks.ts` when `import.meta.env.VITE_USE_MOCK_API === "true"` (convention reused from dashboard slice). Phase B removes this flag or leaves it as a developer toggle.

---

## 11. Refresh Strategy

| Trigger | Refresh | Cache TTL |
|---|---|---|
| Page mount | all 4 sections | — |
| Manual refresh (runs card) | runs only | bypasses cache |
| Workspace switch | all 4 sections | invalidates all cached |
| Filter change on runs | runs only | bypasses cache |
| Navigate to skill detail | skill detail only (if not cached or >5min old) | 5 min TTL |
| Navigate to run detail | run detail only | no cache (always fresh) |

---

## 12. Telemetry Hooks (V1)

Minimal, but reserved:

- Emit `aic:view-loaded` event on first successful render
- Emit `aic:card-error` on any card-level error with `{card, workspaceId, statusCode}`
- Emit `aic:drill-into-skill` and `aic:drill-into-run` for UX analytics (no PII)
- V1 uses console.debug; future slice can wire to a real analytics sink.
