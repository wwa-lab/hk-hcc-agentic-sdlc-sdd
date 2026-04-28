# Design Management Data Flow

## Purpose

Runtime sequences, state machines, error cascade, and refresh strategy for the Design Management slice. This document operationalizes the architecture in [design-management-architecture.md](design-management-architecture.md) and the contracts in [../05-design/contracts/design-management-API_IMPLEMENTATION_GUIDE.md](../05-design/contracts/design-management-API_IMPLEMENTATION_GUIDE.md).

All Mermaid diagrams use Mermaid 8.x-compatible syntax.

---

## 1. Load Lifecycle

### 1.1 Catalog view — Phase A (frontend-only, mocked)

```mermaid
sequenceDiagram
    autonumber
    participant U as User
    participant Router as Vue Router
    participant View as CatalogView
    participant Store as DM Store
    participant Api as DM API Client
    participant Mock as Mock Fixtures

    U->>Router: Navigate /design-management
    Router->>View: Mount
    View->>Store: initCatalog(workspaceId)
    Store->>Api: getCatalog + getCatalogSummary
    Api->>Mock: load catalog.mock.ts + summary.mock.ts
    Mock-->>Api: CatalogAggregateDto
    Api-->>Store: rows + summary populated
    Store-->>View: reactive render
    View-->>U: Catalog paint (summary bar, filter bar, rows)
```

### 1.2 Catalog view — Phase B (backend integration)

```mermaid
sequenceDiagram
    autonumber
    participant U as User
    participant View as CatalogView
    participant Store as DM Store
    participant Api as DM API Client
    participant BE as DesignManagementController
    participant Svc as CatalogService
    participant Guard as DesignAccessGuard
    participant Proj as Projections
    participant DB as DB
    participant REQ as Requirement Adapter

    U->>View: /design-management?workspaceId=ws-42
    View->>Store: initCatalog(ws-42)
    Store->>Api: getCatalog(ws-42) + getCatalogSummary(ws-42)
    Api->>BE: GET /catalog?workspaceId=ws-42
    BE->>Guard: resolve(caller)
    Guard-->>BE: role, authorized workspaces
    BE->>Svc: fetchCatalog(caller, ws-42, filters)
    par parallel projections
      Svc->>Proj: catalogRowProjection(ws-42, filters)
      Proj->>DB: SELECT artifacts JOIN latest version
      DB-->>Proj: rows
      Proj->>REQ: batch spec revisions (for worstCoverageStatus)
      REQ-->>Proj: revision map
      Proj-->>Svc: rows with worst coverage precomputed
    and
      Svc->>Proj: catalogSummaryProjection(ws-42)
      Proj->>DB: SELECT counters
      DB-->>Proj: counters
      Proj-->>Svc: summary data
    end
    Svc-->>BE: CatalogAggregateDto (SectionResult<T> x2)
    BE-->>Api: 200 JSON
    Api-->>Store: cards populated
    Store-->>View: reactive render
    View-->>U: Catalog paint
```

### 1.3 Viewer view — Phase B initial load

```mermaid
sequenceDiagram
    autonumber
    participant U as User
    participant View as ViewerView
    participant Store as DM Store
    participant Api as DM API Client
    participant BE as Controller
    participant Svc as ViewerService
    participant DB as DB
    participant REQ as Requirement Adapter
    participant Skill as Skill Runtime

    U->>View: /design-management/artifacts/da-2001
    View->>Store: openArtifact(da-2001)
    Store->>Api: getArtifactAggregate(da-2001)
    Api->>BE: GET /artifacts/da-2001
    BE->>Svc: fetchViewer(caller, da-2001)
    par header + links + ai + history
      Svc->>DB: SELECT header
      DB-->>Svc: header
    and
      Svc->>DB: SELECT links
      DB-->>Svc: link rows
      Svc->>REQ: batch spec revisions
      REQ-->>Svc: revision map
      Svc->>Svc: compute coverage per link
    and
      Svc->>DB: SELECT latest AI summary (cache lookup)
      DB-->>Svc: summary or null
    and
      Svc->>DB: SELECT change log page 0 (25 rows)
      DB-->>Svc: history rows
    end
    Svc-->>BE: ViewerAggregateDto
    BE-->>Api: 200 JSON
    Api-->>Store: viewer populated
    Store-->>View: reactive render
    View-->>U: Viewer paint (header, preview frame, links, AI panel, history)

    Note over Svc,Skill: Async AI summary (if cache empty)

    Svc->>Skill: invoke artifact-summarizer async
    Skill-->>BE: POST /internal/ai-summaries (later)
    BE->>Svc: persistSummary(...)
    Svc->>DB: UPSERT DesignAiSummary
    Svc-->>BE: ack
    BE-->>Api: (UI polls /ai-summary or subscribes to shell event bus)
    Api-->>Store: summary populated
    Store-->>View: AI panel re-renders (no full reload)
```

### 1.4 Preview iframe content load

```mermaid
sequenceDiagram
    autonumber
    participant View as ViewerView
    participant Iframe as ArtifactPreviewFrame
    participant Api as DM API Client
    participant BE as Controller
    participant Svc as ViewerService
    participant DB as DB

    View->>Iframe: <iframe sandbox="allow-scripts" src="/api/v1/design-management/artifacts/da-2001/preview">
    Iframe->>Api: GET /artifacts/da-2001/preview
    Api->>BE: GET /artifacts/da-2001/preview
    BE->>Svc: fetchPreview(caller, da-2001)
    Svc->>DB: SELECT current version HTML
    DB-->>Svc: html bytes + metadata
    Svc-->>BE: html + CSP + XFO + XCTO headers
    BE-->>Iframe: 200 text/html (with conservative CSP)
    Iframe-->>View: rendered mock
```

### 1.5 Traceability view — Phase B

```mermaid
sequenceDiagram
    autonumber
    participant U as User
    participant View as TraceabilityView
    participant Store as DM Store
    participant Api as DM API Client
    participant BE as Controller
    participant Svc as TraceabilityService
    participant REQ as Requirement Adapter
    participant DB as DB

    U->>View: /design-management/traceability?workspaceId=ws-42
    View->>Store: initTraceability(ws-42)
    Store->>Api: getTraceability(ws-42, filters)
    Api->>BE: GET /traceability?workspaceId=ws-42
    BE->>Svc: fetchTraceability(caller, ws-42, filters)
    Svc->>REQ: listSpecs(ws-42, callerRole)
    REQ-->>Svc: Spec index (id, title, project, revision, state)
    Svc->>DB: SELECT links WHERE workspaceId=ws-42
    DB-->>Svc: link rows
    Svc->>Svc: join spec x links, compute coverage per pair
    Svc-->>BE: TraceabilityDto (specs with linked artifacts + coverage)
    BE-->>Api: 200 JSON
    Api-->>Store: trace populated
    Store-->>View: reactive render
    View-->>U: Traceability paint (missing first, sorted)
```

---

## 2. Admin Write Paths

### 2.1 Register a new artifact

```mermaid
sequenceDiagram
    autonumber
    participant Admin as Workspace Admin
    participant Modal as RegisterArtifactModal
    participant Store as DM Store
    participant Api as DM API Client
    participant BE as Controller
    participant Guard as DesignAccessGuard
    participant Pii as PiiScanner
    participant San as HtmlSanitizer
    participant Cmd as ArtifactCommandService
    participant DB as DB
    participant Event as DesignChangeEventPublisher
    participant Skill as Skill Runtime

    Admin->>Modal: Submit form
    Modal->>Store: registerArtifact(dto)
    Store->>Api: POST /artifacts
    Api->>BE: POST /artifacts {title, kind, projectId, authors, lifecycle, htmlPayload, linkedSpecIds}
    BE->>Guard: requireRole(WORKSPACE_ADMIN, workspaceId)
    alt not admin
      Guard-->>BE: 403 DM_ROLE_REQUIRED
      BE-->>Api: 403
    else authorized
      BE->>Pii: scan(htmlPayload)
      alt PII detected
        Pii-->>BE: matches (offsets only)
        BE-->>Api: 422 DM_PII_DETECTED
      else clean
        BE->>San: validate(htmlPayload)
        alt > 2 MB
          San-->>BE: reject
          BE-->>Api: 413 DM_ARTIFACT_TOO_LARGE
        else ok
          BE->>Cmd: register(dto)
          Cmd->>DB: INSERT DesignArtifact (v=1, current)
          Cmd->>DB: INSERT DesignArtifactVersion (v1)
          opt linkedSpecIds present
            Cmd->>DB: INSERT DesignSpecLink rows (idempotent)
          end
          Cmd->>Event: publish(ARTIFACT_REGISTERED)
          Event->>Event: audit pipeline
          Cmd->>Skill: invoke artifact-summarizer async
          Cmd-->>BE: artifactId + versionId
          BE-->>Api: 201 Created {artifactId, versionId}
          Api-->>Store: refresh catalog
        end
      end
    end
```

### 2.2 Publish a new version

```mermaid
sequenceDiagram
    autonumber
    participant Admin as Workspace Admin
    participant Modal as PublishVersionModal
    participant Store as DM Store
    participant Api as DM API Client
    participant BE as Controller
    participant Guard as DesignAccessGuard
    participant Pii as PiiScanner
    participant San as HtmlSanitizer
    participant Cmd as ArtifactCommandService
    participant DB as DB
    participant Event as DesignChangeEventPublisher
    participant Skill as Skill Runtime

    Admin->>Modal: Submit (prevVersionId, htmlPayload, changeNote)
    Modal->>Store: publishVersion(id, dto)
    Store->>Api: POST /artifacts/{id}/versions
    Api->>BE: POST /artifacts/{id}/versions
    BE->>Guard: requireRole(WORKSPACE_ADMIN)
    BE->>Cmd: publishVersion(id, dto)
    Cmd->>DB: SELECT current artifact + version
    alt current.versionId != dto.prevVersionId
      Cmd-->>BE: 409 DM_STALE_VERSION {currentVersionId}
      BE-->>Api: 409
    else fencing ok
      Cmd->>Pii: scan(htmlPayload)
      Cmd->>San: validate size
      Cmd->>DB: INSERT DesignArtifactVersion (v=n+1)
      Cmd->>DB: UPDATE DesignArtifact SET currentVersionId=..., lastUpdatedAt=now()
      Cmd->>DB: (AI summary cache invalidated naturally by key rotation)
      Cmd->>Event: publish(VERSION_PUBLISHED)
      Cmd->>Skill: invoke artifact-summarizer async
      Cmd-->>BE: new versionId
      BE-->>Api: 201 Created
      Api-->>Store: refresh viewer
    end
```

### 2.3 Link a Spec (idempotent)

```mermaid
sequenceDiagram
    autonumber
    participant Admin as Workspace Admin
    participant Modal as LinkerModal
    participant Store as DM Store
    participant Api as DM API Client
    participant BE as Controller
    participant Guard as DesignAccessGuard
    participant Cmd as LinkCommandService
    participant REQ as Requirement Adapter
    participant DB as DB
    participant Event as DesignChangeEventPublisher

    Admin->>Modal: Select specIds[]
    Modal->>Store: linkSpecs(artifactId, specIds, declaredCoverage)
    Store->>Api: POST /artifacts/{id}/links
    Api->>BE: POST /artifacts/{id}/links
    BE->>Guard: requireRole(WORKSPACE_ADMIN)
    BE->>Cmd: link(artifactId, specIds, coverage)
    Cmd->>REQ: batch specs resolve (visibility + latest revision)
    alt any spec invisible
      REQ-->>Cmd: filtered set
      Cmd-->>BE: 403 DM_SPEC_NOT_FOUND (for the forbidden subset)
    else all visible
      loop for each specId
        Cmd->>DB: SELECT existing link by (artifactId, specId)
        alt exists
          Cmd->>DB: (no-op; idempotent)
        else missing
          Cmd->>DB: INSERT DesignSpecLink {artifactId, specId, coversRevision, declaredCoverage}
          Cmd->>Event: publish(SPEC_LINKED)
        end
      end
      Cmd-->>BE: linked set
      BE-->>Api: 200 OK {links: [...]}
      Api-->>Store: refresh viewer/traceability
    end
```

### 2.4 Regenerate AI summary

```mermaid
sequenceDiagram
    autonumber
    participant Admin as Workspace Admin
    participant UI as AiSummaryPanel
    participant Store as DM Store
    participant Api as DM API Client
    participant BE as Controller
    participant Guard as DesignAccessGuard
    participant Svc as AiSummaryService
    participant Policy as DesignPolicy
    participant DB as DB
    participant Skill as Skill Runtime

    Admin->>UI: Click Regenerate
    UI->>Store: regenerateAiSummary(artifactId)
    Store->>Api: POST /artifacts/{id}/ai-summary/regenerate
    Api->>BE: POST /artifacts/{id}/ai-summary/regenerate
    BE->>Guard: requireRole(WORKSPACE_ADMIN)
    BE->>Policy: checkAutonomy(workspace, 'artifact-summarizer')
    alt autonomy insufficient
      Policy-->>BE: deny
      BE-->>Api: 403 DM_AI_AUTONOMY_INSUFFICIENT
    else allowed
      BE->>Svc: regenerate(artifactId)
      Svc->>DB: SELECT current version
      Svc->>Skill: invoke artifact-summarizer (versionId, skillVersion)
      Skill-->>Svc: accepted (202)
      Svc-->>BE: 202 Accepted {pending}
      BE-->>Api: 202
      Api-->>Store: mark pending, start short poll / subscribe
      Skill->>BE: POST /internal/ai-summaries {artifactId, versionId, summary, keyElements}
      BE->>Svc: persistSummary(dto)
      Svc->>DB: UPSERT DesignAiSummary
      Svc->>Event: publish(AI_SUMMARY_GENERATED)
      BE-->>Skill: 200 OK
    end
```

Note: In V1, the frontend uses short-interval polling (every 3s, max 5 attempts) to fetch the completed summary. A later iteration may move to a shell-owned websocket channel; Design Management does not introduce its own transport.

### 2.5 Lifecycle stage transition

```mermaid
stateDiagram-v2
    [*] --> DRAFT
    DRAFT --> READY_FOR_REVIEW: submit for review
    READY_FOR_REVIEW --> DRAFT: return for rework
    READY_FOR_REVIEW --> APPROVED: approve
    APPROVED --> DEPRECATED: deprecate
    DEPRECATED --> [*]
```

All transitions are admin-invoked, audited, and subject to `DesignPolicy.assertAllowed(from, to)`; disallowed transitions return `DM_INVALID_LIFECYCLE_TRANSITION`. V1 does not model review threads or multi-approver flows.

### 2.6 Coverage status derivation

```mermaid
flowchart TB
  Start[Compute coverage for link L]
  S1{spec.state == ARCHIVED or DELETED?}
  S2{link.declaredCoverage == PARTIAL?}
  S3{link.coversRevision < spec.latestRevision?}
  M[MISSING]
  P[PARTIAL]
  St[STALE]
  OK[OK]

  Start --> S1
  S1 -- yes --> M
  S1 -- no --> S2
  S2 -- yes --> P
  S2 -- no --> S3
  S3 -- yes --> St
  S3 -- no --> OK
```

Computed at request time inside `CoverageService`; never persisted. The Requirement adapter batches Spec lookups so a page with N links costs one batch call, not N.

---

## 3. Error Cascade & Isolation

### 3.1 Per-card error isolation

```mermaid
graph TB
  Req[Incoming request]
  Svc[Service orchestrator]
  P1[Projection 1<br/>header]
  P2[Projection 2<br/>links + coverage]
  P3[Projection 3<br/>AI summary]
  P4[Projection 4<br/>change history]
  E1[SectionResult.OK]
  E2[SectionResult.OK or ERROR]
  E3[SectionResult.OK or ERROR]
  E4[SectionResult.OK or ERROR]
  Dto[ViewerAggregateDto]
  UI[Viewer UI renders<br/>per-card ERROR where applicable]

  Req --> Svc
  Svc --> P1
  Svc --> P2
  Svc --> P3
  Svc --> P4
  P1 --> E1
  P2 --> E2
  P3 --> E3
  P4 --> E4
  E1 --> Dto
  E2 --> Dto
  E3 --> Dto
  E4 --> Dto
  Dto --> UI
```

A failure in any one projection is captured as a `SectionResult.ERROR` with a `correlationId`. The HTTP response is still `200` overall unless ALL sections fail (in which case the controller returns `500` with a top-level error). The UI renders each card independently.

### 3.2 AI failure cascade

- Skill runtime timeout → `SectionResult.ERROR` on the AI Summary section only; retry button visible; rest of Viewer unaffected
- Autonomy gating denial → `SectionResult.OK` with `advisoryOnly=true` and no action buttons
- Cache hit stale (artifact published a new version but summary is still v{n-1}) → UI shows "Summary is for v{n-1}, current is v{n}" chip; background regeneration kicks off if admin opens the panel

### 3.3 Requirement unavailable

If the Requirement adapter times out:

- Catalog's `worstCoverageStatus` rendered as `UNKNOWN` with a neutral chip (not crimson); no cards fail
- Viewer's Linked-Spec strip renders each chip's coverage as `UNKNOWN`; tooltip explains "Spec revision unavailable"
- Traceability view renders `SectionResult.ERROR` for the section since the Spec index cannot be computed

### 3.4 Database unavailable

Whole-page failure mode. Controller returns `503 DM_DB_UNAVAILABLE`. The UI shows the shared shell's outage banner. No DM-specific handling.

---

## 4. Refresh Strategy

| Trigger | Affected cards | Mechanism |
|---------|----------------|-----------|
| User changes Workspace in shell context bar | Catalog (all), Traceability | Store subscribes to `shellContextStore.workspaceId`; re-inits on change |
| User changes filter | Catalog rows | URL query update → store re-fetch of rows projection only (summary unchanged) |
| User clicks retry in an ERROR card | That card only | Store exposes `refreshCard(key)` that re-issues the single projection |
| Admin registers new artifact | Catalog rows + summary | Optimistic insert; then real refresh |
| Admin publishes new version | Viewer header, preview, AI summary panel (pending), change history | Store exposes `refreshViewer(artifactId)` after the write succeeds |
| Admin links / unlinks Spec | Viewer Linked-Spec strip, Traceability row for affected Spec | Store refreshes both surfaces if user is on them |
| AI summary callback | AI Summary panel | Poll (V1) or shell event bus (future); store updates summary for that `(artifactId, versionId)` |
| User leaves Viewer | Everything viewer-specific | Store clears viewer cache for that artifact to avoid stale read on next open |

### 4.1 Polling window for AI summary

When an admin triggers regeneration, the frontend polls `GET /artifacts/{id}/ai-summary` at 3-second intervals, max 5 attempts (15s budget). If still pending after 5 attempts, the panel renders a "Summary still generating — check back soon" state with a manual refresh button. The polling is scoped to the Viewer route; navigating away cancels it.

### 4.2 Catalog refresh debouncing

Filter changes debounce at 200 ms before triggering a fetch. Text search debounces at 300 ms. The summary bar projection is NOT re-fetched on filter change — only the rows projection.

---

## 5. Phase A → Phase B Migration

### 5.1 Mock toggle

The frontend runs in Phase A (mocked) by default in development; Phase B (live backend) is enabled via `VITE_USE_BACKEND=true`. The API client switches between mock fixtures and real `fetchJson<T>` calls at module boundary; no call-site changes are needed.

### 5.2 Compatibility guarantees

- Mock fixtures must return the same DTO shape as the real backend
- `SectionResult<T>` envelope is preserved even in mocks
- Mock mutations simulate `DM_STALE_VERSION` (5% injection rate) so the UI stale-token handling is testable in Phase A
- Mock PII detection is simulated via a fixed trigger string (`__PII_TRIGGER__`) so admins can test the PII rejection flow

### 5.3 Contract validation

Integration tests in Phase B assert DTO compatibility with Phase A mocks using a shared TypeScript type declaration. Drift between mock and live is a P1 defect.

---

## 6. Observability

### 6.1 Correlation ID propagation

```mermaid
sequenceDiagram
    autonumber
    participant UI
    participant Api as DM API Client
    participant BE as Controller
    participant Svc as Service
    participant Skill as Skill Runtime

    UI->>Api: user action
    Api->>Api: generate or inherit X-Correlation-Id
    Api->>BE: HTTP with X-Correlation-Id: corr-...
    BE->>Svc: propagate via MDC
    Svc->>Skill: HTTP with same X-Correlation-Id
    Skill-->>Svc: response
    Svc-->>BE: response (correlationId in SectionResult.error if any)
    BE-->>Api: response + X-Correlation-Id
    Api-->>UI: rendered (correlationId available for error chips)
```

### 6.2 Metrics

| Metric | Surface | Purpose |
|--------|---------|---------|
| `design_management.catalog.first_paint_ms` | Frontend | P95 dashboard |
| `design_management.viewer.first_paint_ms` | Frontend | P95 dashboard |
| `design_management.ai_summary.cache_hit_rate` | Backend | Cache effectiveness |
| `design_management.ai_summary.cold_duration_ms` | Backend | P95 dashboard |
| `design_management.{endpoint}.error_rate` | Backend | Error tracking |
| `design_management.pii_rejections` | Backend | Compliance counter |
| `design_management.stale_version_rejections` | Backend | Concurrency signal |

### 6.3 Log redaction

HTML payloads are NEVER logged. Only size (bytes), hash (sha256), and versionTag are logged. PII match offsets are logged but match content is not.

---

## 7. Summary

- Reads: Catalog, Viewer, Traceability assemble parallel projections into per-section `SectionResult<T>`; partial failures isolate per card.
- Writes: admin-only, narrow, audited; version writes use `prevVersionId` fencing; link writes are idempotent.
- Coverage: computed at request time, never persisted; Requirement adapter batches Spec lookups.
- AI: async via Skill Runtime; cached by `(artifactId, versionId, skillVersion)`; regenerate rotates cache; polling 3s × 5.
- Sandbox: conservative CSP on preview; no per-artifact relaxation; size-cap 2 MB hard-reject on write.
- State lives in backend DB; frontend mirrors URL and shell context; local UI state never outlives the route.
