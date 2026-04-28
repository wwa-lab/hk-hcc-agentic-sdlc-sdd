# Project Management API Implementation Guide

## Purpose

Full endpoint contracts for the Project Management slice — paths, request / response JSON examples, error codes, backend implementation skeleton, frontend integration guide, testing contracts, and versioning policy.

## Traceability

- Spec: [../../03-spec/project-management-spec.md](../../03-spec/project-management-spec.md)
- Architecture: [../../04-architecture/project-management-architecture.md](../../04-architecture/project-management-architecture.md)
- Data flow: [../../04-architecture/project-management-data-flow.md](../../04-architecture/project-management-data-flow.md)
- Data model: [../../04-architecture/project-management-data-model.md](../../04-architecture/project-management-data-model.md)
- Design: [../project-management-design.md](../project-management-design.md)

---

## 1. Conventions

- Base path: `/api/v1/project-management`
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
- Content type: `application/json; charset=utf-8`.
- Auth: bearer token; resolved to a platform user; role checked per endpoint.
- Dates are ISO 8601 (`YYYY-MM-DD` for LocalDate; full ISO with offset for Instant).
- Monetary / numeric precision: percentages are integers 0–500 where applicable.
- Pagination: `page` (0-based), `pageSize` (default 50, max 200), response includes `total`.
- Idempotency: all `PATCH` / `POST-transition` / `POST-countersign` require a `planRevision` fencing token (see §6.4).

---

## 2. Endpoint Overview

### Portfolio (Workspace-scoped, read)

| Method | Path | Role required |
|--------|------|---------------|
| GET | `/portfolio?workspaceId={ws}` | Workspace reader |
| GET | `/portfolio/summary?workspaceId={ws}` | Workspace reader |
| GET | `/portfolio/heatmap?workspaceId={ws}&window={WEEK\|MONTH\|MILESTONE}` | Workspace reader |
| GET | `/portfolio/capacity?workspaceId={ws}` | Workspace reader |
| GET | `/portfolio/risks?workspaceId={ws}&limit=20&severity=&category=` | Workspace reader |
| GET | `/portfolio/dependencies?workspaceId={ws}&limit=15` | Workspace reader |
| GET | `/portfolio/cadence?workspaceId={ws}` | Workspace reader |

### Plan read (project-scoped)

| Method | Path | Role required |
|--------|------|---------------|
| GET | `/plan/{projectId}` | Project reader |
| GET | `/plan/{projectId}/header` | Project reader |
| GET | `/plan/{projectId}/milestones?includeArchived=false` | Project reader |
| GET | `/plan/{projectId}/capacity` | Project reader |
| GET | `/plan/{projectId}/risks?state=&severity=` | Project reader |
| GET | `/plan/{projectId}/dependencies?state=` | Project reader |
| GET | `/plan/{projectId}/progress` | Project reader |
| GET | `/plan/{projectId}/change-log?actorType=&targetType=&from=&to=&page=0&pageSize=50` | Project reader / Auditor |
| GET | `/plan/{projectId}/ai-suggestions?state=PENDING` | Project reader |

### Plan mutations (project-scoped)

| Method | Path | Role required |
|--------|------|---------------|
| POST | `/plan/{projectId}/milestones` | Project Admin (PM) / Workspace Admin |
| PATCH | `/plan/{projectId}/milestones/{id}` | Project Admin / Workspace Admin |
| POST | `/plan/{projectId}/milestones/{id}/transition` | Project Admin / Workspace Admin |
| POST | `/plan/{projectId}/milestones/{id}/archive` | Project Admin / Workspace Admin |
| PATCH | `/plan/{projectId}/capacity` | Project Admin / Workspace Admin |
| POST | `/plan/{projectId}/risks` | Project Admin / Contributor |
| PATCH | `/plan/{projectId}/risks/{id}` | Project Admin / Contributor |
| POST | `/plan/{projectId}/risks/{id}/transition` | Project Admin / Workspace Admin (ESCALATE: any write role) |
| POST | `/plan/{projectId}/dependencies` | Project Admin / Contributor |
| PATCH | `/plan/{projectId}/dependencies/{id}` | Project Admin |
| POST | `/plan/{projectId}/dependencies/{id}/transition` | Project Admin / Workspace Admin |
| POST | `/plan/{projectId}/dependencies/{id}/countersign` | Target-project Admin / Target Workspace Admin |
| POST | `/plan/{projectId}/ai-suggestions/{id}/accept` | Project Admin |
| POST | `/plan/{projectId}/ai-suggestions/{id}/dismiss` | Project Admin |

### Internal (AI runtime → Project Management)

| Method | Path | Auth |
|--------|------|------|
| POST | `/internal/ai-suggestions` | Service token |

---

## 3. Portfolio Endpoints — Full Contracts

### 3.1 `GET /portfolio?workspaceId={ws}` — aggregate first paint

Response `200` (truncated for readability):

```json
{
  "success": true,
  "data": {
    "summary": {
      "status": "OK",
      "data": {
        "workspaceId": "ws-42",
        "activeProjects": 17,
        "redProjects": 3,
        "atRiskOrSlippedMilestones": 9,
        "criticalRisks": 4,
        "blockedDependencies": 5,
        "pendingApprovals": 7,
        "aiPendingReview": 12,
        "lastRefreshedAt": "2026-04-17T09:12:11Z"
      },
      "error": null,
      "fetchedAt": "2026-04-17T09:12:11Z"
    },
    "heatmap": {
      "status": "OK",
      "data": {
        "window": "WEEK",
        "columns": ["2026-W15", "2026-W16", "2026-W17", "2026-W18"],
        "rows": [
          {
            "projectId": "proj-8821",
            "projectName": "Control Plane Migration",
            "cells": [
              { "windowLabel": "2026-W15", "dominantStatus": "IN_PROGRESS" },
              { "windowLabel": "2026-W16", "dominantStatus": "AT_RISK" },
              { "windowLabel": "2026-W17", "dominantStatus": "SLIPPED" },
              { "windowLabel": "2026-W18", "dominantStatus": "NONE" }
            ]
          }
        ]
      },
      "error": null,
      "fetchedAt": "2026-04-17T09:12:11Z"
    },
    "capacity": { "status": "OK", "data": { "...": "..." }, "error": null, "fetchedAt": "..." },
    "risks": { "status": "OK", "data": { "...": "..." }, "error": null, "fetchedAt": "..." },
    "bottlenecks": { "status": "OK", "data": [], "error": null, "fetchedAt": "..." },
    "cadence": {
      "status": "ERROR",
      "data": null,
      "error": { "code": "PM_CADENCE_INSUFFICIENT_DATA", "message": "Need at least 4 weeks of history", "correlationId": "corr-..." },
      "fetchedAt": "2026-04-17T09:12:11Z"
    }
  },
  "error": null,
  "correlationId": "corr-2026-04-17-aggregate",
  "timestamp": "2026-04-17T09:12:11.987Z"
}
```

Errors on the whole endpoint: `400 PM_VALIDATION_ERROR` when `workspaceId` missing; `403 PM_AUTH_FORBIDDEN` if caller has no Workspace role.

### 3.2 `GET /portfolio/summary?workspaceId={ws}`

Payload: the `summary` block above, unwrapped from the aggregate:

```json
{
  "success": true,
  "data": {
    "status": "OK",
    "data": { "workspaceId": "ws-42", "activeProjects": 17, "...": "..." },
    "error": null,
    "fetchedAt": "2026-04-17T09:12:11Z"
  },
  "error": null,
  "correlationId": "corr-...",
  "timestamp": "..."
}
```

### 3.3 `GET /portfolio/heatmap?workspaceId=...&window=WEEK|MONTH|MILESTONE`

- `window` default `WEEK`.
- `PM_HEATMAP_UNSUPPORTED_WINDOW` (422) if unknown value.

### 3.4 `GET /portfolio/capacity?workspaceId=...`

```json
{
  "status": "OK",
  "data": {
    "projects": [
      { "projectId": "proj-8821", "projectName": "Control Plane Migration" },
      { "projectId": "proj-8822", "projectName": "Data Platform Refresh" }
    ],
    "rows": [
      {
        "memberId": "mem-42",
        "displayName": "Asha Nair",
        "totalPercent": 112,
        "flag": "OVER",
        "cells": [
          { "projectId": "proj-8821", "percent": 60 },
          { "projectId": "proj-8822", "percent": 52 }
        ]
      }
    ],
    "underThreshold": 50
  },
  "error": null,
  "fetchedAt": "..."
}
```

### 3.5 `GET /portfolio/risks?workspaceId=...&limit=20&severity=CRITICAL&category=DEPENDENCY`

```json
{
  "status": "OK",
  "data": {
    "topRisks": [
      {
        "riskId": "risk-901",
        "projectId": "proj-8821",
        "projectName": "Control Plane Migration",
        "title": "Schema diverges from upstream",
        "severity": "CRITICAL",
        "category": "DEPENDENCY",
        "ownerMemberId": "mem-17",
        "ageDays": 6,
        "mitigationNote": "Pairing with data team Wednesday",
        "aiRecommendationId": "sug-77"
      }
    ],
    "severityCategoryHeatmap": [
      { "severity": "CRITICAL", "category": "DEPENDENCY", "count": 2 },
      { "severity": "HIGH", "category": "DELIVERY", "count": 5 }
    ]
  },
  "error": null,
  "fetchedAt": "..."
}
```

### 3.6 `GET /portfolio/dependencies?workspaceId=...&limit=15`

```json
{
  "status": "OK",
  "data": [
    {
      "dependencyId": "dep-331",
      "sourceProjectId": "proj-8821",
      "sourceProjectName": "Control Plane Migration",
      "targetProjectId": "proj-8833",
      "targetDescriptor": "Data Lake Stream v2",
      "external": false,
      "relationship": "API",
      "blockerReason": "Contract undecided on auth scheme",
      "ownerTeam": "Data Platform",
      "daysBlocked": 11,
      "aiProposalId": "sug-88"
    }
  ],
  "error": null,
  "fetchedAt": "..."
}
```

### 3.7 `GET /portfolio/cadence?workspaceId=...`

```json
{
  "status": "OK",
  "data": [
    { "key": "THROUGHPUT", "window": "4W", "value": 6.5, "deltaAbs": 1.2, "trend": "UP" },
    { "key": "THROUGHPUT", "window": "12W", "value": 5.1, "deltaAbs": -0.4, "trend": "DOWN" },
    { "key": "CYCLE_TIME_MEDIAN", "window": "4W", "value": 9.0, "deltaAbs": -1.0, "trend": "DOWN" },
    { "key": "CYCLE_TIME_P90", "window": "4W", "value": 24.0, "deltaAbs": 2.0, "trend": "UP" },
    { "key": "HIT_RATE", "window": "4W", "value": 0.78, "deltaAbs": 0.05, "trend": "UP" },
    { "key": "PLAN_STABILITY", "window": "4W", "value": 0.18, "deltaAbs": -0.02, "trend": "DOWN" }
  ],
  "error": null,
  "fetchedAt": "..."
}
```

---

## 4. Plan Read Endpoints — Full Contracts

### 4.1 `GET /plan/{projectId}` — aggregate first paint

Same envelope pattern as `/portfolio`. Sections: `header`, `milestones`, `capacity`, `risks`, `dependencies`, `progress`, `changeLog` (page=0), `aiSuggestions`.

### 4.2 `GET /plan/{projectId}/header`

```json
{
  "status": "OK",
  "data": {
    "projectId": "proj-8821",
    "projectName": "Control Plane Migration",
    "workspaceId": "ws-42",
    "workspaceName": "Platform Workspace",
    "applicationId": "app-12",
    "applicationName": "Platform Application",
    "lifecycleStage": "DELIVERY",
    "planHealth": "YELLOW",
    "planHealthFactors": ["2 AT_RISK milestones", "1 blocked dependency"],
    "nextMilestone": { "id": "ms-3", "label": "Alpha Cut", "targetDate": "2026-05-02" },
    "pmMemberId": "mem-7",
    "pmDisplayName": "Rafi Patel",
    "pmBackupMemberId": "mem-12",
    "autonomyLevel": "L2",
    "lastUpdatedAt": "2026-04-17T09:00:00Z"
  },
  "error": null,
  "fetchedAt": "..."
}
```

### 4.3 `GET /plan/{projectId}/milestones`

```json
{
  "status": "OK",
  "data": [
    {
      "id": "ms-1", "projectId": "proj-8821", "label": "Discovery",
      "description": null, "targetDate": "2026-03-15",
      "status": "COMPLETED", "percentComplete": 100,
      "ownerMemberId": "mem-7", "ownerDisplayName": "Rafi Patel",
      "slippageReason": null, "ordering": 0,
      "slippage": null, "planRevision": 4,
      "createdAt": "2026-02-10T09:00:00Z",
      "completedAt": "2026-03-14T17:00:00Z",
      "archivedAt": null
    },
    {
      "id": "ms-3", "projectId": "proj-8821", "label": "Alpha Cut",
      "description": "Cut first alpha release", "targetDate": "2026-05-02",
      "status": "AT_RISK", "percentComplete": 42,
      "ownerMemberId": "mem-7", "ownerDisplayName": "Rafi Patel",
      "slippageReason": "Upstream schema dependency slipping by 7 days",
      "ordering": 2,
      "slippage": {
        "score": "HIGH",
        "factors": [
          { "label": "Dependency blocked 7 days", "evidence": "/project-management/proj-8821?depId=dep-331" },
          { "label": "Scope grew 18% in last 7 days", "evidence": "/requirements?projectId=proj-8821" }
        ],
        "computedAt": "2026-04-17T06:00:00Z"
      },
      "planRevision": 11,
      "createdAt": "2026-03-01T09:00:00Z",
      "completedAt": null,
      "archivedAt": null
    }
  ],
  "error": null,
  "fetchedAt": "..."
}
```

### 4.4 `GET /plan/{projectId}/capacity`

```json
{
  "status": "OK",
  "data": {
    "milestones": [
      { "id": "ms-2", "label": "Spec", "ordering": 1 },
      { "id": "ms-3", "label": "Alpha Cut", "ordering": 2 }
    ],
    "members": [
      { "id": "mem-42", "displayName": "Asha Nair", "hasBackup": true, "onCall": false },
      { "id": "mem-7", "displayName": "Rafi Patel", "hasBackup": false, "onCall": true }
    ],
    "cells": [
      { "memberId": "mem-42", "milestoneId": "ms-2", "percent": 40, "justification": null,
        "windowStart": "2026-04-01", "windowEnd": "2026-05-01" },
      { "memberId": "mem-42", "milestoneId": "ms-3", "percent": 60, "justification": null,
        "windowStart": "2026-04-15", "windowEnd": "2026-05-15" },
      { "memberId": "mem-7",  "milestoneId": "ms-3", "percent": 80, "justification": "PM doubles as engineer this sprint",
        "windowStart": "2026-04-15", "windowEnd": "2026-05-15" }
    ],
    "rowTotals": { "mem-42": 100, "mem-7": 80 },
    "columnTotals": { "ms-2": 40, "ms-3": 140 },
    "underThreshold": 50
  },
  "error": null,
  "fetchedAt": "..."
}
```

### 4.5 `GET /plan/{projectId}/risks?state=&severity=`

Shape mirrors data model §3.2. Example omitted for brevity.

### 4.6 `GET /plan/{projectId}/dependencies?state=`

Shape mirrors data model §3.2. Example omitted for brevity.

### 4.7 `GET /plan/{projectId}/progress`

```json
{
  "status": "OK",
  "data": [
    { "node": "REQUIREMENT", "throughput": 12, "priorThroughput": 10, "health": "GREEN", "slipped": false, "deepLink": "/requirements?projectId=proj-8821" },
    { "node": "STORY",       "throughput": 24, "priorThroughput": 20, "health": "GREEN", "slipped": false, "deepLink": "/stories?projectId=proj-8821" },
    { "node": "SPEC",        "throughput": 8,  "priorThroughput": 14, "health": "YELLOW","slipped": true,  "deepLink": "/specs?projectId=proj-8821" },
    { "node": "ARCHITECTURE","throughput": 3,  "priorThroughput": 3,  "health": "GREEN", "slipped": false, "deepLink": "/architecture?projectId=proj-8821" },
    { "node": "DESIGN",      "throughput": 4,  "priorThroughput": 5,  "health": "GREEN", "slipped": false, "deepLink": "/design?projectId=proj-8821" },
    { "node": "TASKS",       "throughput": 22, "priorThroughput": 24, "health": "GREEN", "slipped": false, "deepLink": "/tasks?projectId=proj-8821" },
    { "node": "CODE",        "throughput": 18, "priorThroughput": 21, "health": "GREEN", "slipped": false, "deepLink": "/code?projectId=proj-8821" },
    { "node": "TEST",        "throughput": 17, "priorThroughput": 20, "health": "GREEN", "slipped": false, "deepLink": "/test?projectId=proj-8821" },
    { "node": "DEPLOY",      "throughput": 3,  "priorThroughput": 2,  "health": "GREEN", "slipped": false, "deepLink": "/deploy?projectId=proj-8821" },
    { "node": "INCIDENT",    "throughput": 1,  "priorThroughput": 0,  "health": "YELLOW","slipped": false, "deepLink": "/incidents?projectId=proj-8821" },
    { "node": "LEARNING",    "throughput": 2,  "priorThroughput": 3,  "health": "GREEN", "slipped": false, "deepLink": "/learning?projectId=proj-8821" }
  ],
  "error": null,
  "fetchedAt": "..."
}
```

### 4.8 `GET /plan/{projectId}/change-log?...`

```json
{
  "status": "OK",
  "data": {
    "entries": [
      {
        "id": "log-2001", "projectId": "proj-8821",
        "actorType": "HUMAN", "actorMemberId": "mem-7", "actorDisplayName": "Rafi Patel",
        "skillExecutionId": null,
        "action": "TRANSITION", "targetType": "MILESTONE", "targetId": "ms-3",
        "beforeJson": { "status": "IN_PROGRESS" },
        "afterJson": { "status": "AT_RISK", "slippageReason": "Upstream schema dependency slipping by 7 days" },
        "correlationId": "corr-abc",
        "auditLinkId": "audit-998",
        "at": "2026-04-17T08:45:00Z"
      }
    ],
    "page": 0,
    "total": 143
  },
  "error": null,
  "fetchedAt": "..."
}
```

### 4.9 `GET /plan/{projectId}/ai-suggestions?state=PENDING`

```json
{
  "status": "OK",
  "data": [
    {
      "id": "sug-77", "projectId": "proj-8821",
      "kind": "MITIGATION", "targetType": "RISK", "targetId": "risk-901",
      "payload": {
        "summary": "Engage upstream data team to co-author schema contract this week.",
        "details": "Historical signal: schema contracts unresolved for 7+ days typically slip by 14 days.",
        "evidence": [
          { "label": "History of similar risks", "href": "/report-center/history?category=DEPENDENCY" }
        ]
      },
      "confidence": 0.78,
      "state": "PENDING",
      "skillExecutionId": "skill-exec-2211",
      "suppressionUntil": null,
      "createdAt": "2026-04-17T06:15:00Z",
      "resolvedAt": null
    }
  ],
  "error": null,
  "fetchedAt": "..."
}
```

---

## 5. Plan Mutation Endpoints — Full Contracts

All mutations require `Authorization: Bearer <token>` and the caller's role gating (see §2). All return the updated entity on success.

### 5.1 `POST /plan/{projectId}/milestones` — create

Request:

```json
{
  "label": "Beta Cut",
  "description": "Broader rollout to 3 tenants",
  "targetDate": "2026-06-10",
  "ownerMemberId": "mem-7",
  "ordering": 3
}
```

Response `201`: `MilestoneDto` (as in §4.3).

Errors:
- `422 PM_VALIDATION_ERROR` — missing / invalid fields.
- `403 PM_AUTH_FORBIDDEN` — caller lacks write role.

### 5.2 `PATCH /plan/{projectId}/milestones/{id}` — update fields

Request:

```json
{
  "label": "Beta Cut v2",
  "description": null,
  "targetDate": "2026-06-17",
  "ownerMemberId": "mem-7",
  "ordering": 3,
  "planRevision": 5
}
```

- Any of the fields may be provided; server applies non-null ones.
- `planRevision` is required.

Errors:
- `409 PM_STALE_REVISION` — revision mismatch; response includes current revision.
- `422 PM_VALIDATION_ERROR` — field violations.

### 5.3 `POST /plan/{projectId}/milestones/{id}/transition`

Request (AT_RISK example):

```json
{
  "to": "AT_RISK",
  "slippageReason": "Upstream schema dependency slipping by 7 days",
  "newTargetDate": null,
  "planRevision": 11
}
```

Request (SLIPPED → IN_PROGRESS recovery):

```json
{
  "to": "IN_PROGRESS",
  "slippageReason": null,
  "newTargetDate": "2026-05-20",
  "planRevision": 12
}
```

Errors:
- `409 PM_INVALID_TRANSITION` — not allowed by state machine.
- `422 PM_SLIPPAGE_REASON_REQUIRED` — missing reason for AT_RISK/SLIPPED.
- `422 PM_VALIDATION_ERROR` — newTargetDate not ≥ today when required.

### 5.4 `POST /plan/{projectId}/milestones/{id}/archive`

Request: empty body. Response `200`: archived MilestoneDto.

### 5.5 `PATCH /plan/{projectId}/capacity` — batch update

Request:

```json
{
  "edits": [
    { "memberId": "mem-42", "milestoneId": "ms-2", "percent": 50, "justification": null, "planRevision": 3 },
    { "memberId": "mem-42", "milestoneId": "ms-3", "percent": 70, "justification": null, "planRevision": 4 },
    { "memberId": "mem-7",  "milestoneId": "ms-3", "percent": 80,
      "justification": "PM doubles as engineer this sprint", "planRevision": 2 }
  ]
}
```

Response `200`: `PlanCapacityMatrixDto` (entire matrix re-served after recompute).

Errors:
- `422 PM_OVERALLOCATION_JUSTIFICATION_REQUIRED` — a row total > 100 without justification on at least one of the contributing cells. Response includes the offending memberIds and current totals.
- `409 PM_STALE_REVISION` — any cell's `planRevision` is stale; response includes offending cells and their latest revisions.
- Batch is atomic: on failure, no cell is persisted.

### 5.6 Risk mutations

`POST /plan/{projectId}/risks`:

```json
{
  "title": "Schema diverges from upstream",
  "severity": "CRITICAL",
  "category": "DEPENDENCY",
  "ownerMemberId": "mem-17",
  "linkedIncidentId": null,
  "linkedTaskId": null
}
```

Response `201`: `RiskDto`.

`PATCH /plan/{projectId}/risks/{id}`:

```json
{
  "title": "Schema diverges from upstream (scope: auth and sizing)",
  "severity": "CRITICAL",
  "category": "DEPENDENCY",
  "ownerMemberId": "mem-17",
  "planRevision": 3
}
```

`POST /plan/{projectId}/risks/{id}/transition`:

```json
{
  "to": "MITIGATING",
  "mitigationNote": "Paired with data team Wed 14:00; contract review Thu.",
  "resolutionNote": null,
  "linkedIncidentId": null,
  "planRevision": 4
}
```

Escalation example:

```json
{
  "to": "ESCALATED",
  "mitigationNote": null,
  "resolutionNote": null,
  "linkedIncidentId": "incd-22",
  "planRevision": 5
}
```

Errors: `PM_MITIGATION_NOTE_REQUIRED`, `PM_RESOLUTION_NOTE_REQUIRED`, `PM_INVALID_TRANSITION`, `PM_STALE_REVISION`.

### 5.7 Dependency mutations

`POST /plan/{projectId}/dependencies`:

```json
{
  "targetRef": "Data Lake Stream v2",
  "targetProjectId": "proj-8833",
  "direction": "UPSTREAM",
  "relationship": "API",
  "ownerTeam": "Data Platform",
  "blockerReason": "Contract undecided on auth scheme"
}
```

Response `201`: `DependencyDto`.

`PATCH /plan/{projectId}/dependencies/{id}`:

```json
{
  "ownerTeam": "Data Platform",
  "blockerReason": "Contract decided; stub pending",
  "planRevision": 2
}
```

`POST /plan/{projectId}/dependencies/{id}/transition`:

```json
{
  "to": "NEGOTIATING",
  "rejectionReason": null,
  "contractCommitment": null,
  "planRevision": 3
}
```

APPROVED (external):

```json
{
  "to": "APPROVED",
  "rejectionReason": null,
  "contractCommitment": "Agreement #7742 signed 2026-04-16 by Data Platform lead.",
  "planRevision": 4
}
```

REJECTED:

```json
{
  "to": "REJECTED",
  "rejectionReason": "Target team cannot deliver in this window; re-scoping.",
  "contractCommitment": null,
  "planRevision": 5
}
```

`POST /plan/{projectId}/dependencies/{id}/countersign` (internal APPROVED path):

```json
{ "planRevision": 4 }
```

- Caller must be Project Admin / Workspace Admin for the **target** project.
- Response includes `DependencyDto` with `resolutionState=APPROVED`, `counterSignatureMemberId=<caller>`.

Errors: `PM_DEP_COUNTERSIGN_REQUIRED`, `PM_DEP_CONTRACT_REQUIRED`, `PM_DEP_REJECTION_REASON_REQUIRED`, `PM_INVALID_TRANSITION`, `PM_STALE_REVISION`, `PM_AUTH_FORBIDDEN`.

### 5.8 AI suggestion accept / dismiss

`POST /plan/{projectId}/ai-suggestions/{id}/accept`:

Request: `{}`

Response:

```json
{
  "success": true,
  "data": {
    "suggestion": { "id": "sug-77", "...": "...", "state": "ACCEPTED", "resolvedAt": "2026-04-17T09:15:00Z" },
    "auditLinkId": "audit-1012"
  },
  "error": null,
  "correlationId": "corr-...",
  "timestamp": "..."
}
```

`POST /plan/{projectId}/ai-suggestions/{id}/dismiss`:

```json
{ "reason": "Already mitigated offline; AI has stale context." }
```

Response: `{ "state": "DISMISSED", "suppressionUntil": "2026-04-18T09:15:00Z", "resolvedAt": "..." }`.

Errors: `PM_NOT_FOUND`, `PM_AI_SUGGESTION_ALREADY_RESOLVED` (409).

---

## 6. Policy Details

### 6.1 Authorization matrix

| Action | Project Contributor | Project Admin (PM) | Workspace Admin | Application Owner | Auditor |
|--------|:--:|:--:|:--:|:--:|:--:|
| Read Plan | ✅ | ✅ | ✅ | ✅ | ✅ |
| Read Portfolio | ✅ | ✅ | ✅ | ✅ | ✅ |
| Create / update milestones | ❌ | ✅ | ✅ | ❌ | ❌ |
| Transition milestones | ❌ | ✅ | ✅ | ❌ | ❌ |
| Archive milestones | ❌ | ✅ | ✅ | ❌ | ❌ |
| Edit capacity allocation | ❌ | ✅ | ✅ | ❌ | ❌ |
| Create risks | ✅ | ✅ | ✅ | ❌ | ❌ |
| Update / transition risks (non-escalate) | ❌ | ✅ | ✅ | ❌ | ❌ |
| Escalate risks | ✅ | ✅ | ✅ | ❌ | ❌ |
| Create / propose dependencies | ✅ | ✅ | ✅ | ❌ | ❌ |
| Update / transition dependencies | ❌ | ✅ | ✅ | ❌ | ❌ |
| Counter-sign dependencies (target side) | ❌ | ✅ (target-side) | ✅ (target-side) | ❌ | ❌ |
| Accept / dismiss AI suggestions | ❌ | ✅ | ✅ | ❌ | ❌ |

### 6.2 Isolation

- Portfolio endpoints: 403 if caller has no role in the requested `workspaceId`.
- Plan endpoints: 403 if caller has no role in the `projectId` or its parent Workspace.

### 6.3 Autonomy policy

V1 reads the project's Autonomy Level but does not auto-apply. The Plan Header endpoint returns the current level; Platform Center is authoritative.

### 6.4 Concurrency / fencing

- Every mutable entity returns a `planRevision` that increments on each mutation.
- All `PATCH` / transition / countersign / archive requests require `planRevision`.
- Stale: `409 PM_STALE_REVISION` with `{ current: 12, provided: 11 }`.

### 6.5 Audit

Every mutation emits:
- A `plan_change_log` row (hot window visible in UI).
- A platform audit event (long-term compliance).

Both are produced atomically with the DB commit via outbox pattern.

---

## 7. Backend Implementation Skeleton

```java
// ProjectManagementController.java
@RestController
@RequestMapping(ProjectManagementConstants.BASE_PATH) // "/api/v1/project-management"
public class ProjectManagementController {

    private final PlanAccessGuard guard;
    private final PortfolioService portfolio;
    private final PlanService plan;
    private final MilestoneService milestones;
    private final CapacityService capacity;
    private final RiskService risks;
    private final DependencyService dependencies;
    private final AiSuggestionService aiSuggestions;

    // portfolio
    @GetMapping("/portfolio")
    public ApiResponse<PortfolioAggregateDto> portfolioAggregate(
        @RequestParam String workspaceId, @AuthenticationPrincipal Caller caller) {
      guard.workspaceRead(caller, workspaceId);
      return ApiResponse.ok(portfolio.aggregate(workspaceId));
    }
    // ... other portfolio section endpoints

    // plan read
    @GetMapping("/plan/{projectId}")
    public ApiResponse<PlanAggregateDto> planAggregate(
        @PathVariable String projectId, @AuthenticationPrincipal Caller caller) {
      guard.projectRead(caller, projectId);
      return ApiResponse.ok(plan.aggregate(projectId));
    }
    // ... other plan section endpoints

    // milestone mutations
    @PostMapping("/plan/{projectId}/milestones")
    public ApiResponse<MilestoneDto> createMilestone(
        @PathVariable String projectId, @Valid @RequestBody CreateMilestoneRequest req,
        @AuthenticationPrincipal Caller caller) {
      guard.projectWrite(caller, projectId);
      return ApiResponse.ok(milestones.create(projectId, req, caller));
    }

    @PostMapping("/plan/{projectId}/milestones/{id}/transition")
    public ApiResponse<MilestoneDto> transition(
        @PathVariable String projectId, @PathVariable String id,
        @Valid @RequestBody TransitionMilestoneRequest req,
        @AuthenticationPrincipal Caller caller) {
      guard.projectWrite(caller, projectId);
      return ApiResponse.ok(milestones.transition(projectId, id, req, caller));
    }

    // ... analogous shapes for capacity / risks / dependencies / ai-suggestions
}
```

```java
// PlanPolicy.java
@Component
public class PlanPolicy {
    private static final Map<MilestoneStatus, Set<MilestoneStatus>> M_ALLOWED = Map.of(
        MilestoneStatus.NOT_STARTED, Set.of(MilestoneStatus.IN_PROGRESS, MilestoneStatus.ARCHIVED),
        MilestoneStatus.IN_PROGRESS, Set.of(MilestoneStatus.COMPLETED, MilestoneStatus.AT_RISK, MilestoneStatus.ARCHIVED),
        MilestoneStatus.AT_RISK,     Set.of(MilestoneStatus.IN_PROGRESS, MilestoneStatus.SLIPPED, MilestoneStatus.ARCHIVED),
        MilestoneStatus.SLIPPED,     Set.of(MilestoneStatus.IN_PROGRESS, MilestoneStatus.ARCHIVED),
        MilestoneStatus.COMPLETED,   Set.of(MilestoneStatus.ARCHIVED)
    );

    public void assertAllowed(MilestoneStatus from, MilestoneStatus to) {
        if (!M_ALLOWED.getOrDefault(from, Set.of()).contains(to)) {
            throw new PmException(ErrorCodes.PM_INVALID_TRANSITION, "From " + from + " to " + to + " is not allowed");
        }
    }

    public void requireSlippageReason(MilestoneStatus to, String reason) {
        if ((to == MilestoneStatus.AT_RISK || to == MilestoneStatus.SLIPPED) &&
            (reason == null || reason.trim().length() < 10)) {
            throw new PmException(ErrorCodes.PM_SLIPPAGE_REASON_REQUIRED, "Slippage reason must be ≥ 10 chars");
        }
    }
    // ... other policy rules
}
```

```java
// MilestoneService.java (transition excerpt)
@Service @RequiredArgsConstructor
public class MilestoneService {
    private final MilestoneRepository repo;
    private final PlanPolicy policy;
    private final PlanChangeEventPublisher events;

    @Transactional
    public MilestoneDto transition(String projectId, String id, TransitionMilestoneRequest req, Caller caller) {
        var m = repo.findByProjectIdAndId(projectId, id).orElseThrow(() -> new PmException(ErrorCodes.PM_NOT_FOUND));
        if (m.getPlanRevision() != req.planRevision())
            throw new PmException(ErrorCodes.PM_STALE_REVISION, "current=" + m.getPlanRevision());
        var to = MilestoneStatus.valueOf(req.to());
        policy.assertAllowed(m.getStatus(), to);
        policy.requireSlippageReason(to, req.slippageReason());

        var before = snapshot(m);
        m.setStatus(to);
        if (to == MilestoneStatus.AT_RISK || to == MilestoneStatus.SLIPPED) m.setSlippageReason(req.slippageReason());
        if (to == MilestoneStatus.COMPLETED) m.setCompletedAt(Instant.now());
        if (to == MilestoneStatus.IN_PROGRESS && m.getStatus() == MilestoneStatus.SLIPPED && req.newTargetDate() != null)
            m.setTargetDate(req.newTargetDate());
        m.setPlanRevision(m.getPlanRevision() + 1);
        repo.save(m);

        events.publish(PlanChangeEvent.transition(projectId, "MILESTONE", id, before, snapshot(m), caller));
        return MilestoneMapper.toDto(m);
    }
}
```

---

## 8. Frontend Integration Guide

### 8.1 API client

```typescript
// frontend/src/features/project-management/api/projectManagementApi.ts
import { fetchJson } from '@/shared/api/client';
import type {
  PortfolioAggregate, PlanAggregate,
  Milestone, CreateMilestoneRequest, UpdateMilestoneRequest, TransitionMilestoneRequest,
  Risk, Dependency, AiSuggestion, PlanCapacityMatrix, CapacityBatchUpdateRequest,
} from '../types';

const BASE = '/api/v1/project-management';

export const projectManagementApi = {
  // portfolio
  async getPortfolioAggregate(workspaceId: string): Promise<PortfolioAggregate> {
    return fetchJson<PortfolioAggregate>(`${BASE}/portfolio?workspaceId=${encodeURIComponent(workspaceId)}`);
  },
  async getPortfolioSection(workspaceId: string, section: string, params?: Record<string, string>): Promise<any> {
    const qs = new URLSearchParams({ workspaceId, ...(params ?? {}) }).toString();
    return fetchJson(`${BASE}/portfolio/${section}?${qs}`);
  },

  // plan
  async getPlanAggregate(projectId: string): Promise<PlanAggregate> {
    return fetchJson<PlanAggregate>(`${BASE}/plan/${projectId}`);
  },
  async getPlanSection(projectId: string, section: string, params?: Record<string, string>): Promise<any> {
    const qs = params ? `?${new URLSearchParams(params)}` : '';
    return fetchJson(`${BASE}/plan/${projectId}/${section}${qs}`);
  },

  // milestone mutations
  async createMilestone(projectId: string, req: CreateMilestoneRequest): Promise<Milestone> {
    return fetchJson<Milestone>(`${BASE}/plan/${projectId}/milestones`, { method: 'POST', body: JSON.stringify(req) });
  },
  async updateMilestone(projectId: string, id: string, req: UpdateMilestoneRequest): Promise<Milestone> {
    return fetchJson<Milestone>(`${BASE}/plan/${projectId}/milestones/${id}`, { method: 'PATCH', body: JSON.stringify(req) });
  },
  async transitionMilestone(projectId: string, id: string, req: TransitionMilestoneRequest): Promise<Milestone> {
    return fetchJson<Milestone>(`${BASE}/plan/${projectId}/milestones/${id}/transition`, { method: 'POST', body: JSON.stringify(req) });
  },
  async archiveMilestone(projectId: string, id: string): Promise<void> {
    return fetchJson<void>(`${BASE}/plan/${projectId}/milestones/${id}/archive`, { method: 'POST' });
  },

  // capacity
  async batchUpdateCapacity(projectId: string, req: CapacityBatchUpdateRequest): Promise<PlanCapacityMatrix> {
    return fetchJson<PlanCapacityMatrix>(`${BASE}/plan/${projectId}/capacity`, { method: 'PATCH', body: JSON.stringify(req) });
  },

  // risks / dependencies / AI suggestions follow the same shape
  // ...
};
```

### 8.2 Mock toggle

`fetchJson` honors `VITE_USE_MOCK=true` to swap to a mock provider that returns fixtures from `mocks/`.

### 8.3 Vite proxy

```ts
// frontend/vite.config.ts excerpt
server: {
  proxy: {
    '/api': { target: 'http://localhost:8080', changeOrigin: true, secure: false }
  }
}
```

---

## 9. Testing Contracts

### 9.1 Backend

- `MilestoneServiceTest` — exhaustive state machine tests + reason validation.
- `CapacityServiceTest` — batch atomicity + over-allocation gate.
- `RiskServiceTest` — state machine + escalation emits approval.
- `DependencyServiceTest` — counter-signature path (internal) and contract commitment path (external).
- `AiSuggestionServiceTest` — accept / dismiss / suppression window.
- `PlanAccessGuardTest` — role and isolation matrix (§6.1 / §6.2).
- `FlywayMigrationTest` — apply V20–V25 against the project-space baseline; assert indexes and constraints.
- `PortfolioServiceIT` — seed dataset, assert projections and budgets (< 800ms at 50 projects).

### 9.2 Frontend

- `usePlanStateMachine.spec.ts` — valid and invalid transitions, guard functions.
- `MilestonePlanner.spec.ts` — create / edit / transition / archive + optimistic rollback on 409.
- `CapacityAllocationEditor.spec.ts` — batch debounce + justification dialog + over-allocation handling.
- `RiskRegistryEditor.spec.ts` — state transitions including escalation.
- `DependencyResolver.spec.ts` — counter-signature path + external commitment path.
- `PlanAiSuggestionsPanel.spec.ts` — accept / dismiss + 24h suppression indicator.

### 9.3 Contract tests (cross)

A JSON-schema fixture set under `contracts/fixtures/` validates that server responses match frontend types. CI runs contract tests on both ends.

---

## 10. Versioning Policy

- Paths are `/api/v1/project-management`. Breaking changes → `/api/v2`.
- Non-breaking additions: new optional fields, new endpoints. Allowed on V1.
- DTO removals: two-phase migration (deprecate → remove in the next major).
- Enum additions: allowed; frontend must treat unknown values as a neutral fallback.
- Schema: additive only; renames are two-phase (add-new → backfill → flip → drop-old) via Flyway.

---

## 11. Examples by Story

| Story | Primary endpoint(s) |
|-------|---------------------|
| S1 | `GET /portfolio/summary` |
| S2 | `GET /portfolio/heatmap` |
| S3 | `GET /portfolio/capacity` |
| S4 | `GET /portfolio/risks` |
| S5 | `GET /portfolio/dependencies` |
| S6 | `GET /portfolio/cadence` |
| S7 | `GET /plan/{projectId}/header` |
| S8 | `POST/PATCH /plan/{projectId}/milestones*` |
| S9 | `POST /plan/{projectId}/milestones/{id}/transition` |
| S10 | `GET /plan/{projectId}/ai-suggestions`, accept / dismiss |
| S11 | `PATCH /plan/{projectId}/capacity` |
| S12 | derived from `GET /plan/{projectId}/capacity` (hasBackup, onCall) |
| S13 | `POST .../ai-suggestions/{id}/accept` (REBALANCE) + batch capacity |
| S14 | Risk CRUD + transition |
| S15 | `POST .../ai-suggestions/{id}/accept` (MITIGATION) |
| S16 | Dependency CRUD + transition + countersign |
| S17 | AI suggestion accept (DEP_RESOLUTION) |
| S18 | `GET /plan/{projectId}/progress` |
| S19 | `GET /plan/{projectId}/change-log` |
| S20 | Deep-link variants of Portfolio and Plan endpoints with query params |
| S21 | AI Command Panel invokes suggestion endpoints |
| S22 | Authorization matrix enforced server-side |
| S23 | Plan state preserved via query params + store |
| S24 | `SectionResult<T>` + per-card retry |

---

## 12. Quick Reference — Error Codes

(Repeated for ease of use during implementation; canonical list is in [data-model §8](../../04-architecture/project-management-data-model.md).)

| Code | HTTP | Meaning |
|------|------|---------|
| `PM_AUTH_FORBIDDEN` | 403 | Insufficient role |
| `PM_WORKSPACE_MISMATCH` | 409 | Plan-view project outside context Workspace |
| `PM_INVALID_TRANSITION` | 409 | State-machine violation |
| `PM_SLIPPAGE_REASON_REQUIRED` | 422 | — |
| `PM_OVERALLOCATION_JUSTIFICATION_REQUIRED` | 422 | — |
| `PM_MITIGATION_NOTE_REQUIRED` | 422 | — |
| `PM_RESOLUTION_NOTE_REQUIRED` | 422 | — |
| `PM_DEP_COUNTERSIGN_REQUIRED` | 422 | — |
| `PM_DEP_CONTRACT_REQUIRED` | 422 | — |
| `PM_DEP_REJECTION_REASON_REQUIRED` | 422 | — |
| `PM_STALE_REVISION` | 409 | planRevision mismatch |
| `PM_AI_SUGGESTION_ALREADY_RESOLVED` | 409 | — |
| `PM_AI_SUGGESTION_SUPPRESSED` | 409 | — |
| `PM_HEATMAP_UNSUPPORTED_WINDOW` | 422 | — |
| `PM_CADENCE_INSUFFICIENT_DATA` | 200 / section error | not enough history |
| `PM_NOT_FOUND` | 404 | — |
| `PM_VALIDATION_ERROR` | 422 | generic validation |
