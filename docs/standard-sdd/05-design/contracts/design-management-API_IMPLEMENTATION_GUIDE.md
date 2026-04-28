# Design Management API Implementation Guide

## Purpose

Full endpoint contracts for the Design Management slice — paths, request / response JSON examples, error codes, backend implementation skeleton, frontend integration guide, testing contracts, and versioning policy.

## Traceability

- Spec: [../../03-spec/design-management-spec.md](../../03-spec/design-management-spec.md)
- Architecture: [../../04-architecture/design-management-architecture.md](../../04-architecture/design-management-architecture.md)
- Data flow: [../../04-architecture/design-management-data-flow.md](../../04-architecture/design-management-data-flow.md)
- Data model: [../../04-architecture/design-management-data-model.md](../../04-architecture/design-management-data-model.md)
- Design: [../design-management-design.md](../design-management-design.md)

---

## 1. Conventions

- Base path: `/api/v1/design-management`
- All responses use the shared `ApiResponse<T>` envelope:

```json
{
  "success": true,
  "data": { ... },
  "error": null,
  "correlationId": "corr-2026-04-17-abc123",
  "timestamp": "2026-04-17T09:12:33.123Z"
}
```

- All aggregate responses use `SectionResult<T>` per section; partial failures return HTTP 200 with `ERROR` sections.
- Content type: `application/json; charset=utf-8` (except `/preview` which is `text/html; charset=utf-8`).
- Auth: bearer token; resolved to a platform user; role checked per endpoint.
- Dates are ISO 8601 with offset (e.g., `2026-04-17T09:12:33.123Z`).
- Sizes are bytes (integers).
- Pagination: `page` (0-based), `pageSize` (default 25 for change log, max 50); response includes `total`.
- Idempotency:
  - `POST /links` is idempotent per `(artifactId, specId)` pair.
  - `POST /versions` requires `prevVersionId` as a fencing token.
  - `POST /ai-summary/regenerate` is idempotent when the cache key is unchanged.
- Caller context: `X-Correlation-Id` is propagated if supplied; otherwise generated server-side.

---

## 2. Endpoint Overview

### Catalog (Workspace-scoped, read)

| Method | Path | Role required |
|--------|------|---------------|
| GET | `/catalog?workspaceId={ws}&project=&kind=&coverage=&author=&q=&sort=&page=&pageSize=` | Workspace reader |
| GET | `/catalog/summary?workspaceId={ws}` | Workspace reader |

### Viewer (artifact-scoped)

| Method | Path | Role required |
|--------|------|---------------|
| GET | `/artifacts/{id}` | Workspace or Project reader |
| GET | `/artifacts/{id}/header` | Workspace or Project reader |
| GET | `/artifacts/{id}/preview` | Workspace or Project reader |
| GET | `/artifacts/{id}/links` | Workspace or Project reader |
| GET | `/artifacts/{id}/ai-summary` | Workspace or Project reader |
| GET | `/artifacts/{id}/history?page=0&pageSize=25` | Workspace or Project reader / Auditor |
| GET | `/artifacts/{id}/thumbnail` | Workspace or Project reader |

### Traceability (Workspace- or Spec-scoped)

| Method | Path | Role required |
|--------|------|---------------|
| GET | `/traceability?workspaceId={ws}&coverage=&project=&sort=` | Workspace reader |
| GET | `/traceability?specId={specId}` | caller must be able to see the Spec |
| GET | `/coverage/summary?projectId={projectId}` | Project reader (consumer: Project Management) |

### Admin writes (Workspace-admin)

| Method | Path | Role required |
|--------|------|---------------|
| POST | `/artifacts` | Workspace admin |
| POST | `/artifacts/{id}/versions` | Workspace admin |
| PATCH | `/artifacts/{id}/lifecycle` | Workspace admin |
| POST | `/artifacts/{id}/links` | Workspace admin |
| DELETE | `/artifacts/{id}/links/{specId}` | Workspace admin |
| POST | `/artifacts/{id}/ai-summary/regenerate` | Workspace admin |

### Internal (AI runtime → Design Management)

| Method | Path | Auth |
|--------|------|------|
| POST | `/internal/ai-summaries` | Service token |

---

## 3. Catalog Endpoints — Full Contracts

### 3.1 `GET /catalog?workspaceId={ws}&filters...`

Response `200`:

```json
{
  "success": true,
  "data": {
    "status": "OK",
    "data": [
      {
        "artifactId": "da-2001",
        "workspaceId": "ws-42",
        "projectId": "proj-8821",
        "projectName": "Control Plane Migration",
        "title": "Control Tower Dashboard",
        "kind": "PAGE_MOCK",
        "lifecycleStage": "APPROVED",
        "authors": [
          { "memberId": "mem-101", "displayName": "Yaozi Xiao" }
        ],
        "currentVersionId": "dav-2001-v3",
        "currentVersionTag": "v3",
        "lastUpdatedAt": "2026-04-16T11:22:00Z",
        "linkedSpecCount": 4,
        "worstCoverageStatus": "STALE",
        "aiSummaryReady": true,
        "previewThumbnailUrl": "/api/v1/design-management/artifacts/da-2001/thumbnail"
      }
    ],
    "error": null,
    "fetchedAt": "2026-04-17T09:12:11Z"
  },
  "error": null,
  "correlationId": "corr-2026-04-17-abc123",
  "timestamp": "2026-04-17T09:12:11.321Z"
}
```

Query params:

| Param | Type | Default | Notes |
|-------|------|---------|-------|
| `workspaceId` | string | (required) | Must be in caller's authorized set |
| `project` | string | none | Filter by Project |
| `kind` | enum | none | `PAGE_MOCK \| COMPONENT_MOCK \| FLOW_MOCK \| STATE_MOCK` |
| `coverage` | enum | none | `OK \| PARTIAL \| STALE \| MISSING \| NO_LINKS` |
| `author` | string | none | Member ID |
| `q` | string | none | Text search on title |
| `sort` | enum | `last_updated_desc` | `title_asc \| last_updated_desc \| linked_count_desc \| coverage_desc` |
| `page` | int | 0 | 0-based |
| `pageSize` | int | 50 | max 100 |

Errors: `DM_WORKSPACE_FORBIDDEN` (403).

### 3.2 `GET /catalog/summary?workspaceId={ws}`

Response `200`:

```json
{
  "success": true,
  "data": {
    "status": "OK",
    "data": {
      "workspaceId": "ws-42",
      "totalArtifacts": 27,
      "artifactsWithLinks": 21,
      "orphanArtifacts": 6,
      "missingDesigns": 9,
      "staleArtifacts": 4,
      "lastRefreshedAt": "2026-04-17T09:12:11Z",
      "aiLinkageAdvisory": "9 Specs in this Workspace have no linked design. 4 approved artifacts have stale Spec links."
    },
    "error": null,
    "fetchedAt": "2026-04-17T09:12:11Z"
  },
  "error": null,
  "correlationId": "corr-...",
  "timestamp": "..."
}
```

Errors: `DM_WORKSPACE_FORBIDDEN` (403).

---

## 4. Viewer Endpoints — Full Contracts

### 4.1 `GET /artifacts/{id}` — aggregate first paint

Response `200`:

```json
{
  "success": true,
  "data": {
    "header": {
      "status": "OK",
      "data": {
        "artifactId": "da-2001",
        "workspaceId": "ws-42",
        "projectId": "proj-8821",
        "projectName": "Control Plane Migration",
        "title": "Control Tower Dashboard",
        "kind": "PAGE_MOCK",
        "lifecycleStage": "APPROVED",
        "authors": [
          { "memberId": "mem-101", "displayName": "Yaozi Xiao" }
        ],
        "currentVersionId": "dav-2001-v3",
        "currentVersionTag": "v3",
        "registeredAt": "2026-03-20T14:10:00Z",
        "registeredBy": { "memberId": "mem-101", "displayName": "Yaozi Xiao" },
        "lastUpdatedAt": "2026-04-16T11:22:00Z"
      },
      "error": null,
      "fetchedAt": "2026-04-17T09:12:11Z"
    },
    "links": {
      "status": "OK",
      "data": [
        {
          "linkId": "dsl-440",
          "specId": "SPEC-DASH-020",
          "specTitle": "Dashboard SDLC chain strip",
          "coverageStatus": "STALE",
          "declaredCoverage": "FULL",
          "coversRevision": 5,
          "specLatestRevision": 7,
          "linkedAt": "2026-04-10T09:00:00Z",
          "linkedBy": { "memberId": "mem-101", "displayName": "Yaozi Xiao" }
        },
        {
          "linkId": "dsl-441",
          "specId": "SPEC-DASH-030",
          "specTitle": "Incident counter group",
          "coverageStatus": "OK",
          "declaredCoverage": "FULL",
          "coversRevision": 3,
          "specLatestRevision": 3,
          "linkedAt": "2026-04-10T09:00:00Z",
          "linkedBy": { "memberId": "mem-101", "displayName": "Yaozi Xiao" }
        }
      ],
      "error": null,
      "fetchedAt": "2026-04-17T09:12:11Z"
    },
    "aiSummary": {
      "status": "OK",
      "data": {
        "summaryId": "dai-2001-v3-a",
        "artifactId": "da-2001",
        "versionId": "dav-2001-v3",
        "versionTag": "v3",
        "skillName": "artifact-summarizer",
        "skillVersion": "1.3.0",
        "summaryText": "A dark-mode dashboard mock surfacing the SDLC chain (11 nodes), incident counters, workspace switcher, and an AI Command Panel slot. Emphasizes high-density information over marketing polish.",
        "keyElements": [
          "SDLC chain strip (11 nodes)",
          "Incident counter group",
          "AI Command Panel slot",
          "Workspace switcher",
          "Top context bar"
        ],
        "generatedAt": "2026-04-16T11:23:15Z",
        "advisoryOnly": false
      },
      "error": null,
      "fetchedAt": "2026-04-17T09:12:11Z"
    },
    "history": {
      "status": "OK",
      "data": {
        "entries": [
          {
            "entryId": "dcl-9001",
            "artifactId": "da-2001",
            "entryType": "VERSION_PUBLISHED",
            "actor": { "kind": "HUMAN", "member": { "memberId": "mem-101", "displayName": "Yaozi Xiao" } },
            "description": "Published v3 — aligned header with updated Spec SPEC-DASH-020",
            "relatedSpecId": null,
            "occurredAt": "2026-04-16T11:22:00Z",
            "correlationId": "corr-2026-04-16-xyz"
          },
          {
            "entryId": "dcl-9000",
            "artifactId": "da-2001",
            "entryType": "AI_SUMMARY_GENERATED",
            "actor": { "kind": "AI", "skillName": "artifact-summarizer", "skillVersion": "1.3.0", "skillExecutionId": "sx-882" },
            "description": "Summary generated for v3",
            "relatedSpecId": null,
            "occurredAt": "2026-04-16T11:23:15Z",
            "correlationId": "corr-2026-04-16-xyz"
          }
        ],
        "page": 0,
        "pageSize": 25,
        "total": 14
      },
      "error": null,
      "fetchedAt": "2026-04-17T09:12:11Z"
    }
  },
  "error": null,
  "correlationId": "corr-...",
  "timestamp": "..."
}
```

Errors: `DM_ARTIFACT_NOT_FOUND` (404), `DM_WORKSPACE_FORBIDDEN` (403), `DM_PROJECT_FORBIDDEN` (403).

### 4.2 `GET /artifacts/{id}/preview`

Response `200`:

- Content-Type: `text/html; charset=utf-8`
- Headers:
  - `Content-Security-Policy: default-src 'none'; img-src data: blob:; style-src 'unsafe-inline' cdn.tailwindcss.com fonts.googleapis.com fonts.gstatic.com; script-src cdn.tailwindcss.com; font-src fonts.gstatic.com; frame-ancestors 'self'`
  - `X-Frame-Options: SAMEORIGIN`
  - `X-Content-Type-Options: nosniff`
  - `Referrer-Policy: no-referrer`
- Body: raw HTML payload of the artifact's current version

If the artifact exceeds 2 MB, response is `413 DM_ARTIFACT_TOO_LARGE` with guidance to use `/artifacts/{id}/raw` (the UI then presents "Open in new tab" instead of an inline embed).

### 4.3 `GET /artifacts/{id}/ai-summary`

Response `200` (cache hit):

```json
{
  "success": true,
  "data": {
    "status": "OK",
    "data": {
      "summaryId": "dai-2001-v3-a",
      "artifactId": "da-2001",
      "versionId": "dav-2001-v3",
      "versionTag": "v3",
      "skillName": "artifact-summarizer",
      "skillVersion": "1.3.0",
      "summaryText": "...",
      "keyElements": ["...", "..."],
      "generatedAt": "2026-04-16T11:23:15Z",
      "advisoryOnly": false
    },
    "error": null,
    "fetchedAt": "2026-04-17T09:12:11Z"
  },
  "error": null,
  "correlationId": "corr-...",
  "timestamp": "..."
}
```

Response `200` (cache empty):

```json
{
  "success": true,
  "data": {
    "status": "EMPTY",
    "data": null,
    "error": null,
    "fetchedAt": "2026-04-17T09:12:11Z"
  },
  "error": null,
  "correlationId": "corr-...",
  "timestamp": "..."
}
```

### 4.4 `GET /artifacts/{id}/history?page=0&pageSize=25`

Response `200`:

```json
{
  "success": true,
  "data": {
    "status": "OK",
    "data": {
      "entries": [ /* ChangeLogEntryDto[] */ ],
      "page": 0,
      "pageSize": 25,
      "total": 42
    },
    "error": null,
    "fetchedAt": "..."
  },
  "error": null,
  "correlationId": "...",
  "timestamp": "..."
}
```

---

## 5. Traceability Endpoints — Full Contracts

### 5.1 `GET /traceability?workspaceId={ws}`

Response `200`:

```json
{
  "success": true,
  "data": {
    "status": "OK",
    "data": [
      {
        "specId": "SPEC-DASH-020",
        "specTitle": "Dashboard SDLC chain strip",
        "projectId": "proj-8821",
        "projectName": "Control Plane Migration",
        "specLatestRevision": 7,
        "specState": "ACTIVE",
        "linkedArtifacts": [
          {
            "artifactId": "da-2001",
            "title": "Control Tower Dashboard",
            "currentVersionTag": "v3",
            "coverageStatus": "STALE"
          }
        ],
        "coverageSummary": { "OK": 0, "PARTIAL": 0, "STALE": 1, "MISSING": 0, "UNKNOWN": 0 }
      },
      {
        "specId": "SPEC-DASH-040",
        "specTitle": "Workspace switcher",
        "projectId": "proj-8821",
        "projectName": "Control Plane Migration",
        "specLatestRevision": 2,
        "specState": "ACTIVE",
        "linkedArtifacts": [],
        "coverageSummary": { "OK": 0, "PARTIAL": 0, "STALE": 0, "MISSING": 0, "UNKNOWN": 0 }
      }
    ],
    "error": null,
    "fetchedAt": "..."
  },
  "error": null,
  "correlationId": "...",
  "timestamp": "..."
}
```

Query params:

| Param | Type | Default | Notes |
|-------|------|---------|-------|
| `workspaceId` | string | (required if `specId` absent) | |
| `specId` | string | optional | Focused single-Spec response |
| `coverage` | enum | none | Filter rows by primary coverage bucket |
| `project` | string | none | Filter by Project |
| `sort` | enum | `missing_first` | `missing_first \| spec_id_asc \| project_asc` |

Errors: `DM_WORKSPACE_FORBIDDEN`, `DM_SPEC_NOT_FOUND`.

### 5.2 `GET /coverage/summary?projectId={projectId}`

Used by Project Management (REQ-DM-82).

Response `200`:

```json
{
  "success": true,
  "data": {
    "status": "OK",
    "data": {
      "projectId": "proj-8821",
      "specCount": 18,
      "specsWithDesign": 11,
      "specsMissingDesign": 7,
      "staleCoverage": 2,
      "lastRefreshedAt": "..."
    },
    "error": null,
    "fetchedAt": "..."
  },
  "error": null,
  "correlationId": "...",
  "timestamp": "..."
}
```

---

## 6. Admin Write Endpoints — Full Contracts

### 6.1 `POST /artifacts` — register a new artifact

Request:

```json
{
  "workspaceId": "ws-42",
  "projectId": "proj-8821",
  "title": "Control Tower Dashboard",
  "kind": "PAGE_MOCK",
  "lifecycleStage": "DRAFT",
  "authorMemberIds": ["mem-101"],
  "htmlPayload": "<!DOCTYPE html>...",
  "htmlPath": null,
  "initialSpecLinks": [
    { "specId": "SPEC-DASH-020", "declaredCoverage": "FULL" }
  ]
}
```

- Exactly one of `htmlPayload` or `htmlPath` must be present.
- `htmlPath` must reference a file under `docs/standard-sdd/05-design/*.html`.

Response `201`:

```json
{
  "success": true,
  "data": {
    "artifactId": "da-5002",
    "currentVersionId": "dav-5002-v1",
    "currentVersionTag": "v1"
  },
  "error": null,
  "correlationId": "...",
  "timestamp": "..."
}
```

Errors:

- `DM_ROLE_REQUIRED` (403)
- `DM_WORKSPACE_FORBIDDEN` (403)
- `DM_PROJECT_FORBIDDEN` (403)
- `DM_PII_DETECTED` (422):

```json
{
  "success": false,
  "data": null,
  "error": {
    "code": "DM_PII_DETECTED",
    "message": "Payload contains PII patterns and was rejected.",
    "correlationId": "corr-...",
    "details": { "matchCount": 3, "offsets": [1024, 2048, 4096] }
  },
  "correlationId": "corr-...",
  "timestamp": "..."
}
```

- `DM_ARTIFACT_TOO_LARGE` (413)
- `DM_DUPLICATE_ARTIFACT` (409) — soft warning; client may retry with `?acknowledgeDuplicate=true` to force

### 6.2 `POST /artifacts/{id}/versions` — publish new version

Request:

```json
{
  "prevVersionId": "dav-2001-v3",
  "htmlPayload": "<!DOCTYPE html>...",
  "changeNote": "Updated SDLC chain node for new Spec revision"
}
```

Response `201`:

```json
{
  "success": true,
  "data": {
    "versionId": "dav-2001-v4",
    "versionTag": "v4"
  },
  "error": null,
  "correlationId": "...",
  "timestamp": "..."
}
```

Errors:

- `DM_STALE_VERSION` (409):

```json
{
  "success": false,
  "data": null,
  "error": {
    "code": "DM_STALE_VERSION",
    "message": "Another version was published concurrently.",
    "correlationId": "corr-...",
    "details": { "currentVersionId": "dav-2001-v4" }
  },
  "correlationId": "corr-...",
  "timestamp": "..."
}
```

- `DM_PII_DETECTED`, `DM_ARTIFACT_TOO_LARGE`, `DM_ROLE_REQUIRED`

### 6.3 `PATCH /artifacts/{id}/lifecycle`

Request:

```json
{
  "toStage": "APPROVED",
  "reason": "Reviewed and approved by Architect"
}
```

Response `200`:

```json
{
  "success": true,
  "data": { "artifactId": "da-2001", "lifecycleStage": "APPROVED" },
  "error": null,
  "correlationId": "...",
  "timestamp": "..."
}
```

Errors: `DM_INVALID_LIFECYCLE_TRANSITION` (409), `DM_ROLE_REQUIRED`.

### 6.4 `POST /artifacts/{id}/links`

Request:

```json
{
  "links": [
    { "specId": "SPEC-DASH-030", "declaredCoverage": "FULL" },
    { "specId": "SPEC-DASH-040", "declaredCoverage": "PARTIAL" }
  ]
}
```

Response `200`:

```json
{
  "success": true,
  "data": {
    "links": [
      {
        "linkId": "dsl-500",
        "specId": "SPEC-DASH-030",
        "coversRevision": 3,
        "declaredCoverage": "FULL",
        "linkedAt": "2026-04-17T09:14:00Z"
      },
      {
        "linkId": "dsl-501",
        "specId": "SPEC-DASH-040",
        "coversRevision": 2,
        "declaredCoverage": "PARTIAL",
        "linkedAt": "2026-04-17T09:14:00Z"
      }
    ]
  },
  "error": null,
  "correlationId": "...",
  "timestamp": "..."
}
```

Idempotent: re-posting a Spec already linked returns the existing link unchanged.

Errors: `DM_SPEC_NOT_FOUND` (404 on any Spec not visible), `DM_ROLE_REQUIRED`, rate-limit `429` after 50 links / minute.

### 6.5 `DELETE /artifacts/{id}/links/{specId}`

Response `204 No Content` (idempotent — no error on unknown link).

Errors: `DM_ROLE_REQUIRED` (403).

### 6.6 `POST /artifacts/{id}/ai-summary/regenerate`

Response `202 Accepted`:

```json
{
  "success": true,
  "data": { "status": "PENDING", "artifactId": "da-2001", "versionId": "dav-2001-v4" },
  "error": null,
  "correlationId": "...",
  "timestamp": "..."
}
```

Errors:

- `DM_AI_AUTONOMY_INSUFFICIENT` (403)
- `DM_AI_SKILL_UNAVAILABLE` (503)
- `DM_ROLE_REQUIRED` (403)
- `429` after 10 regenerations / artifact / hour / admin

---

## 7. Internal Endpoint — AI Runtime Callback

### 7.1 `POST /internal/ai-summaries`

Auth: service token (not user bearer).

Request:

```json
{
  "artifactId": "da-2001",
  "versionId": "dav-2001-v4",
  "skillName": "artifact-summarizer",
  "skillVersion": "1.3.0",
  "skillExecutionId": "sx-9001",
  "summaryText": "...",
  "keyElements": ["...", "..."],
  "advisoryOnly": false
}
```

Response `200`:

```json
{ "success": true, "data": { "summaryId": "dai-2001-v4-a" }, "error": null, "correlationId": "...", "timestamp": "..." }
```

Errors: `401 UNAUTHORIZED_SERVICE_TOKEN`, `404 DM_ARTIFACT_NOT_FOUND`.

---

## 8. Error Code Catalog

| Code | HTTP | Meaning |
|------|------|---------|
| `DM_WORKSPACE_FORBIDDEN` | 403 | Caller lacks Workspace access |
| `DM_PROJECT_FORBIDDEN` | 403 | Caller lacks Project-scoped access |
| `DM_ROLE_REQUIRED` | 403 | Caller role insufficient for the action |
| `DM_ARTIFACT_NOT_FOUND` | 404 | Artifact not visible / does not exist |
| `DM_SPEC_NOT_FOUND` | 404 | Spec not visible / does not exist |
| `DM_PII_DETECTED` | 422 | Registration / version payload contains PII |
| `DM_ARTIFACT_TOO_LARGE` | 413 | Payload exceeds 2 MB |
| `DM_STALE_VERSION` | 409 | Concurrent version publish rejected |
| `DM_INVALID_LIFECYCLE_TRANSITION` | 409 | Disallowed stage transition |
| `DM_DUPLICATE_ARTIFACT` | 409 | Soft duplicate warning; client may override |
| `DM_AI_AUTONOMY_INSUFFICIENT` | 403 | Workspace autonomy below skill requirement |
| `DM_AI_SKILL_UNAVAILABLE` | 503 | Skill runtime unreachable or rate-limited |
| `DM_DB_UNAVAILABLE` | 503 | Database unavailable |
| `UNAUTHORIZED_SERVICE_TOKEN` | 401 | Internal endpoint called without valid service token |

---

## 9. Backend Implementation Skeleton (Spring Boot 3.x / Java 21)

```java
@RestController
@RequestMapping("/api/v1/design-management")
@RequiredArgsConstructor
public class DesignManagementController {
  private final DesignAccessGuard guard;
  private final CatalogService catalogService;
  private final ViewerService viewerService;
  private final TraceabilityService traceabilityService;
  private final ArtifactCommandService artifactCommandService;
  private final LinkCommandService linkCommandService;
  private final AiSummaryService aiSummaryService;
  private final CoverageService coverageService;
  private final DesignMapper mapper;

  @GetMapping("/catalog")
  public ApiResponse<SectionResult<List<CatalogRowDto>>> getCatalog(
      @RequestParam String workspaceId,
      @RequestParam(required = false) String project,
      @RequestParam(required = false) ArtifactKind kind,
      @RequestParam(required = false) CoverageStatus coverage,
      @RequestParam(required = false) String author,
      @RequestParam(required = false) String q,
      @RequestParam(defaultValue = "last_updated_desc") String sort,
      @RequestParam(defaultValue = "0") int page,
      @RequestParam(defaultValue = "50") int pageSize,
      @AuthenticationPrincipal CallerContext caller) {
    guard.requireWorkspaceAccess(caller, workspaceId);
    CatalogFilters filters = new CatalogFilters(project, kind, coverage, author, q, sort, page, pageSize);
    return ApiResponse.ok(catalogService.fetchRows(caller, workspaceId, filters));
  }

  @GetMapping("/catalog/summary")
  public ApiResponse<SectionResult<CatalogSummaryDto>> getCatalogSummary(
      @RequestParam String workspaceId,
      @AuthenticationPrincipal CallerContext caller) {
    guard.requireWorkspaceAccess(caller, workspaceId);
    return ApiResponse.ok(catalogService.fetchSummary(caller, workspaceId));
  }

  @GetMapping("/artifacts/{id}")
  public ApiResponse<ViewerAggregateDto> getArtifact(
      @PathVariable String id,
      @AuthenticationPrincipal CallerContext caller) {
    guard.requireArtifactAccess(caller, id);
    return ApiResponse.ok(viewerService.fetchAggregate(caller, id));
  }

  @GetMapping(value = "/artifacts/{id}/preview", produces = MediaType.TEXT_HTML_VALUE)
  public ResponseEntity<byte[]> getArtifactPreview(
      @PathVariable String id,
      @AuthenticationPrincipal CallerContext caller) {
    guard.requireArtifactAccess(caller, id);
    return viewerService.fetchPreview(caller, id); // sets CSP + XFO + XCTO headers
  }

  @GetMapping("/artifacts/{id}/ai-summary")
  public ApiResponse<SectionResult<AiSummaryDto>> getAiSummary(
      @PathVariable String id,
      @AuthenticationPrincipal CallerContext caller) {
    guard.requireArtifactAccess(caller, id);
    return ApiResponse.ok(aiSummaryService.fetchLatest(caller, id));
  }

  @GetMapping("/artifacts/{id}/history")
  public ApiResponse<SectionResult<ChangeLogPageDto>> getHistory(
      @PathVariable String id,
      @RequestParam(defaultValue = "0") int page,
      @RequestParam(defaultValue = "25") int pageSize,
      @AuthenticationPrincipal CallerContext caller) {
    guard.requireArtifactAccess(caller, id);
    return ApiResponse.ok(viewerService.fetchHistory(caller, id, page, pageSize));
  }

  @GetMapping("/traceability")
  public ApiResponse<SectionResult<List<TraceabilitySpecRowDto>>> getTraceability(
      @RequestParam(required = false) String workspaceId,
      @RequestParam(required = false) String specId,
      @RequestParam(required = false) CoverageStatus coverage,
      @RequestParam(required = false) String project,
      @RequestParam(defaultValue = "missing_first") String sort,
      @AuthenticationPrincipal CallerContext caller) {
    TraceabilityFilters filters = new TraceabilityFilters(workspaceId, specId, coverage, project, sort);
    return ApiResponse.ok(traceabilityService.fetch(caller, filters));
  }

  @PostMapping("/artifacts")
  public ResponseEntity<ApiResponse<RegisterArtifactResponse>> registerArtifact(
      @Valid @RequestBody RegisterArtifactRequest request,
      @AuthenticationPrincipal CallerContext caller) {
    guard.requireWorkspaceAdmin(caller, request.workspaceId());
    RegisterArtifactResponse response = artifactCommandService.register(caller, request);
    return ResponseEntity.status(HttpStatus.CREATED).body(ApiResponse.ok(response));
  }

  @PostMapping("/artifacts/{id}/versions")
  public ResponseEntity<ApiResponse<PublishVersionResponse>> publishVersion(
      @PathVariable String id,
      @Valid @RequestBody PublishVersionRequest request,
      @AuthenticationPrincipal CallerContext caller) {
    guard.requireWorkspaceAdminByArtifact(caller, id);
    PublishVersionResponse response = artifactCommandService.publishVersion(caller, id, request);
    return ResponseEntity.status(HttpStatus.CREATED).body(ApiResponse.ok(response));
  }

  @PatchMapping("/artifacts/{id}/lifecycle")
  public ApiResponse<LifecycleChangedResponse> changeLifecycle(
      @PathVariable String id,
      @Valid @RequestBody ChangeLifecycleRequest request,
      @AuthenticationPrincipal CallerContext caller) {
    guard.requireWorkspaceAdminByArtifact(caller, id);
    return ApiResponse.ok(artifactCommandService.changeLifecycle(caller, id, request));
  }

  @PostMapping("/artifacts/{id}/links")
  public ApiResponse<LinkSpecsResponse> linkSpecs(
      @PathVariable String id,
      @Valid @RequestBody LinkSpecsRequest request,
      @AuthenticationPrincipal CallerContext caller) {
    guard.requireWorkspaceAdminByArtifact(caller, id);
    return ApiResponse.ok(linkCommandService.link(caller, id, request));
  }

  @DeleteMapping("/artifacts/{id}/links/{specId}")
  public ResponseEntity<Void> unlinkSpec(
      @PathVariable String id,
      @PathVariable String specId,
      @AuthenticationPrincipal CallerContext caller) {
    guard.requireWorkspaceAdminByArtifact(caller, id);
    linkCommandService.unlink(caller, id, specId);
    return ResponseEntity.noContent().build();
  }

  @PostMapping("/artifacts/{id}/ai-summary/regenerate")
  public ResponseEntity<ApiResponse<RegenerateAiSummaryResponse>> regenerateAiSummary(
      @PathVariable String id,
      @AuthenticationPrincipal CallerContext caller) {
    guard.requireWorkspaceAdminByArtifact(caller, id);
    RegenerateAiSummaryResponse response = aiSummaryService.regenerate(caller, id);
    return ResponseEntity.accepted().body(ApiResponse.ok(response));
  }
}

@RestController
@RequestMapping("/api/v1/design-management/internal")
@RequiredArgsConstructor
public class DesignManagementInternalController {
  private final AiSummaryService aiSummaryService;
  private final ServiceTokenGuard serviceTokenGuard;

  @PostMapping("/ai-summaries")
  public ApiResponse<PersistAiSummaryResponse> persistAiSummary(
      @RequestHeader("X-Service-Token") String token,
      @Valid @RequestBody PersistAiSummaryRequest request) {
    serviceTokenGuard.requireAiRuntimeToken(token);
    return ApiResponse.ok(aiSummaryService.persist(request));
  }
}
```

### 9.1 Service orchestration (excerpt)

```java
@Service
@RequiredArgsConstructor
public class ViewerService {
  private final ArtifactHeaderProjection headerProjection;
  private final LinkedSpecProjection linkedSpecProjection;
  private final AiSummaryService aiSummaryService;
  private final ChangeLogProjection changeLogProjection;
  private final CoverageService coverageService;
  private final RequirementAdapter requirementAdapter;

  public ViewerAggregateDto fetchAggregate(CallerContext caller, String artifactId) {
    SectionResult<ArtifactHeaderDto> header = tryProjection(() -> headerProjection.fetch(artifactId));
    SectionResult<List<LinkedSpecDto>> links = tryProjection(() -> {
      List<LinkedSpecRaw> raw = linkedSpecProjection.fetch(artifactId);
      Map<String, SpecRevisionInfo> revisions = requirementAdapter.batchSpecs(
          raw.stream().map(LinkedSpecRaw::specId).toList(), caller);
      return raw.stream()
          .map(r -> new LinkedSpecDto(r, coverageService.compute(r, revisions.get(r.specId()))))
          .toList();
    });
    SectionResult<AiSummaryDto> aiSummary = aiSummaryService.fetchLatest(caller, artifactId);
    SectionResult<ChangeLogPageDto> history = tryProjection(() -> changeLogProjection.fetch(artifactId, 0, 25));
    return new ViewerAggregateDto(header, links, aiSummary, history);
  }

  private <T> SectionResult<T> tryProjection(Supplier<T> s) {
    try {
      T data = s.get();
      return data == null
          ? SectionResult.empty()
          : SectionResult.ok(data);
    } catch (ProjectionException e) {
      return SectionResult.error(e.getCode(), e.getMessage(), MDC.get("correlationId"));
    }
  }
}
```

### 9.2 Coverage computation (excerpt)

```java
@Service
public class CoverageService {
  public CoverageStatus compute(LinkedSpecRaw link, SpecRevisionInfo spec) {
    if (spec == null) return CoverageStatus.UNKNOWN;
    if (spec.state() == SpecState.ARCHIVED || spec.state() == SpecState.DELETED) {
      return CoverageStatus.MISSING;
    }
    if (link.declaredCoverage() == DeclaredCoverage.PARTIAL) {
      return CoverageStatus.PARTIAL;
    }
    if (link.coversRevision() < spec.latestRevision()) {
      return CoverageStatus.STALE;
    }
    return CoverageStatus.OK;
  }
}
```

### 9.3 PII scanner (excerpt)

```java
@Component
public class PiiScanner {
  private final List<Pattern> patterns = List.of(
      Pattern.compile("[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\\.[A-Za-z]{2,}"),     // email
      Pattern.compile("\\b\\d{3}-\\d{2}-\\d{4}\\b"),                           // US SSN
      Pattern.compile("\\b\\d{4}[ -]?\\d{4}[ -]?\\d{4}[ -]?\\d{4}\\b")        // credit card-like
  );

  public PiiScanResult scan(byte[] payload) {
    String s = new String(payload, StandardCharsets.UTF_8);
    List<Integer> offsets = new ArrayList<>();
    for (Pattern p : patterns) {
      Matcher m = p.matcher(s);
      while (m.find()) offsets.add(m.start());
    }
    return new PiiScanResult(offsets.isEmpty(), offsets.size(), offsets);
  }
}
```

---

## 10. Frontend Integration Guide (Vue 3 / Pinia / TypeScript)

### 10.1 API client (excerpt)

```typescript
// api/designManagementApi.ts
import { fetchJson } from '@/shared/api/client'
import type {
  CatalogAggregateDto,
  CatalogRow,
  CatalogSummary,
  ViewerAggregate,
  AiSummary,
  ChangeLogPage,
  TraceabilitySpecRow
} from '../types'

const BASE = '/api/v1/design-management'

export const designManagementApi = {
  getCatalog: (workspaceId: string, filters: CatalogFilters) =>
    fetchJson<SectionResult<CatalogRow[]>>(`${BASE}/catalog?${buildQs({ workspaceId, ...filters })}`),

  getCatalogSummary: (workspaceId: string) =>
    fetchJson<SectionResult<CatalogSummary>>(`${BASE}/catalog/summary?workspaceId=${workspaceId}`),

  getArtifactAggregate: (artifactId: string) =>
    fetchJson<ViewerAggregate>(`${BASE}/artifacts/${artifactId}`),

  getArtifactAiSummary: (artifactId: string) =>
    fetchJson<SectionResult<AiSummary | null>>(`${BASE}/artifacts/${artifactId}/ai-summary`),

  getArtifactHistory: (artifactId: string, page = 0, pageSize = 25) =>
    fetchJson<SectionResult<ChangeLogPage>>(`${BASE}/artifacts/${artifactId}/history?page=${page}&pageSize=${pageSize}`),

  getTraceability: (workspaceId: string, filters: TraceabilityFilters) =>
    fetchJson<SectionResult<TraceabilitySpecRow[]>>(`${BASE}/traceability?${buildQs({ workspaceId, ...filters })}`),

  getTraceabilityForSpec: (specId: string) =>
    fetchJson<SectionResult<TraceabilitySpecRow>>(`${BASE}/traceability?specId=${specId}`),

  registerArtifact: (dto: RegisterArtifactRequest) =>
    fetchJson<{ artifactId: string; currentVersionId: string; currentVersionTag: string }>(`${BASE}/artifacts`, {
      method: 'POST',
      body: JSON.stringify(dto)
    }),

  publishVersion: (artifactId: string, dto: PublishVersionRequest) =>
    fetchJson<{ versionId: string; versionTag: string }>(`${BASE}/artifacts/${artifactId}/versions`, {
      method: 'POST',
      body: JSON.stringify(dto)
    }),

  linkSpecs: (artifactId: string, dto: LinkSpecsRequest) =>
    fetchJson<{ links: Array<{ linkId: string; specId: string; coversRevision: number; declaredCoverage: DeclaredCoverage; linkedAt: string }> }>(`${BASE}/artifacts/${artifactId}/links`, {
      method: 'POST',
      body: JSON.stringify(dto)
    }),

  unlinkSpec: (artifactId: string, specId: string) =>
    fetchJson<void>(`${BASE}/artifacts/${artifactId}/links/${specId}`, { method: 'DELETE' }),

  regenerateAiSummary: (artifactId: string) =>
    fetchJson<{ status: 'PENDING'; artifactId: string; versionId: string }>(`${BASE}/artifacts/${artifactId}/ai-summary/regenerate`, {
      method: 'POST'
    }),

  changeLifecycle: (artifactId: string, dto: ChangeLifecycleRequest) =>
    fetchJson<{ artifactId: string; lifecycleStage: LifecycleStage }>(`${BASE}/artifacts/${artifactId}/lifecycle`, {
      method: 'PATCH',
      body: JSON.stringify(dto)
    })
}
```

### 10.2 Pinia store (excerpt)

Covered in [design-management-design.md §3](../design-management-design.md#3-state-management-pinia). The store exposes per-card refresh actions so error-state retries issue exactly one projection call.

### 10.3 Vite proxy (dev)

```ts
// vite.config.ts
server: {
  proxy: {
    '/api': {
      target: 'http://localhost:8080',
      changeOrigin: true
    }
  }
}
```

---

## 11. Testing Contracts

### 11.1 Backend integration tests

- `DesignAccessGuardTest` — workspace/project isolation (403 paths)
- `CatalogServiceIT` — summary counters, worstCoverageStatus derivation
- `ViewerServiceIT` — parallel projection orchestration; per-section error isolation
- `ArtifactCommandServiceIT` — register (PII reject, size reject, duplicate); publish version (stale token); lifecycle transitions
- `LinkCommandServiceIT` — idempotency per (artifactId, specId); rate limiting
- `AiSummaryServiceIT` — cache key rotation on version publish; autonomy gating; regenerate 202 flow
- `PiiScannerTest` — canonical matches + negative cases; no payload logging
- `CoverageServiceTest` — all 5 status branches (OK / PARTIAL / STALE / MISSING / UNKNOWN)

### 11.2 Frontend contract tests

- Mock / live DTO parity asserted via shared `.d.ts` declarations
- Storybook stories for `DmCard` loading / empty / error / ok
- Story for `LinkedSpecStrip` covering all coverage statuses
- Story for `AiSummaryPanel` covering pending / advisory-only / error / ok

### 11.3 Manual verification checklist

- Catalog renders within 1.2s P95 on seeded Workspace
- Viewer renders within 0.8s P95 excluding iframe
- Iframe preview renders within 3s for a 1.5 MB seeded artifact
- PII trigger rejects; error banner shows correlationId
- Stale token on publish produces the correct 409 toast with retry
- Regenerate AI summary → pending state → ready state within 15s
- Cross-workspace direct-by-id returns 403 with correct error banner
- AI attribution badge is visible on every AI-produced surface

---

## 12. Versioning and Deprecation Policy

- Base path is `/api/v1`. V1 is stable for the duration of this release.
- Additions (new endpoints, new optional fields) are non-breaking and ship under `/api/v1`.
- Removals, renames, and semantic changes require `/api/v2`.
- Deprecated endpoints must emit `Deprecation: true` and `Sunset: <date>` headers for at least one release before removal.
- DTO fields added to existing responses must be optional; clients must tolerate unknown fields.

---

## 13. Out of V1 Scope (Reminder)

The API surface intentionally excludes:

- Design editing, review thread, approval workflow endpoints
- Version diff endpoints
- AI critique or AI generation endpoints
- Figma / external-tool integration endpoints
- Arbitrary image / PDF upload endpoints
- Cross-Workspace rollup endpoints
- Component library / design token endpoints

Add these in a future release with explicit spec → architecture → design updates.
