# Incident Management — API Implementation Guide

## Purpose

This document is the single source of truth for implementing the Incident Management API.
It defines endpoint contracts, request/response shapes, error handling, and integration
patterns for both frontend and backend developers.

## Traceability

- Spec: [incident-spec.md](../../03-spec/incident-spec.md)
- Architecture: [incident-architecture.md](../../04-architecture/incident-architecture.md)
- Design: [incident-design.md](../incident-design.md)
- Data Model: [incident-data-model.md](../../04-architecture/incident-data-model.md)
- Data Flow: [incident-data-flow.md](../../04-architecture/incident-data-flow.md)

---

## 1. API Overview

| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/incidents` | List incidents with optional filters |
| GET | `/api/v1/incidents/:id` | Full incident detail with all 7 sections |
| POST | `/api/v1/incidents/:id/actions/:actionId/approve` | Approve a pending action |
| POST | `/api/v1/incidents/:id/actions/:actionId/reject` | Reject a pending action with reason |

### Base URL

- Local dev: `http://localhost:8080/api/v1`
- Frontend proxy (Vite): `/api/v1` → `http://localhost:8080/api/v1`

### Response Envelope

All responses use the shared `ApiResponse<T>` envelope:

```json
{
  "data": { ... },
  "error": null
}
```

Error responses:

```json
{
  "data": null,
  "error": "Human-readable error message"
}
```

### Backend Constants

Add to `ApiConstants.java`:

```java
public static final String INCIDENTS = API_V1 + "/incidents";
public static final String INCIDENT_DETAIL = INCIDENTS + "/{incidentId}";
public static final String INCIDENT_ACTION_APPROVE = INCIDENT_DETAIL + "/actions/{actionId}/approve";
public static final String INCIDENT_ACTION_REJECT = INCIDENT_DETAIL + "/actions/{actionId}/reject";
```

---

## 2. GET /api/v1/incidents

### 2.1 Request

```
GET /api/v1/incidents
Accept: application/json
```

**Query Parameters:**

| Param | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `priority` | String | No | — | Filter by priority: P1, P2, P3, P4 |
| `status` | String | No | — | Filter by status (comma-separated) |
| `handlerType` | String | No | — | Filter: AI, Human, Hybrid |
| `showResolved` | Boolean | No | false | Include resolved/closed incidents |

### 2.2 Success Response (200 OK)

```json
{
  "data": {
    "severityDistribution": {
      "p1": 2,
      "p2": 3,
      "p3": 1,
      "p4": 0
    },
    "incidents": [
      {
        "id": "INC-0422",
        "title": "API Gateway Latency Spike (>500ms)",
        "priority": "P1",
        "status": "AI_INVESTIGATING",
        "handlerType": "AI",
        "controlMode": "Approval",
        "detectedAt": "2026-04-16T09:40:00Z",
        "duration": "PT1H23M"
      },
      {
        "id": "INC-0421",
        "title": "Database Connection Pool Exhaustion",
        "priority": "P2",
        "status": "PENDING_APPROVAL",
        "handlerType": "AI",
        "controlMode": "Approval",
        "detectedAt": "2026-04-16T08:15:00Z",
        "duration": "PT2H48M"
      },
      {
        "id": "INC-0420",
        "title": "Cache Miss Rate Spike in Product Service",
        "priority": "P3",
        "status": "RESOLVED",
        "handlerType": "Hybrid",
        "controlMode": "Manual",
        "detectedAt": "2026-04-15T14:30:00Z",
        "duration": "PT4H12M"
      },
      {
        "id": "INC-0419",
        "title": "Certificate Expiry Warning (7 days)",
        "priority": "P3",
        "status": "CLOSED",
        "handlerType": "AI",
        "controlMode": "Auto",
        "detectedAt": "2026-04-14T10:00:00Z",
        "duration": "PT0H45M"
      },
      {
        "id": "INC-0418",
        "title": "Memory Leak in Notification Worker",
        "priority": "P2",
        "status": "LEARNING",
        "handlerType": "AI",
        "controlMode": "Approval",
        "detectedAt": "2026-04-13T16:20:00Z",
        "duration": "PT6H30M"
      }
    ]
  },
  "error": null
}
```

### 2.3 Error Cases

| Status | When | Error Message |
|--------|------|---------------|
| 500 | Internal server error | "Failed to load incidents" |

---

## 3. GET /api/v1/incidents/:id

### 3.1 Request

```
GET /api/v1/incidents/INC-0422
Accept: application/json
```

**Path Parameters:**

| Param | Type | Required | Description |
|-------|------|----------|-------------|
| `id` | String | Yes | Incident ID (e.g., INC-0422) |

### 3.2 Success Response (200 OK)

The response uses a two-level envelope:
1. **Top level**: `ApiResponse<IncidentDetailDto>`
2. **Section level**: `SectionResultDto<T>` per card for error isolation

```json
{
  "data": {
    "header": {
      "data": {
        "id": "INC-0422",
        "title": "API Gateway Latency Spike (>500ms)",
        "priority": "P1",
        "status": "PENDING_APPROVAL",
        "handlerType": "AI",
        "controlMode": "Approval",
        "autonomyLevel": "Level2_SuggestApprove",
        "detectedAt": "2026-04-16T09:40:00Z",
        "acknowledgedAt": "2026-04-16T09:40:02Z",
        "resolvedAt": null,
        "duration": "PT1H23M"
      },
      "error": null
    },
    "diagnosis": {
      "data": {
        "entries": [
          {
            "timestamp": "09:41:02",
            "text": "Analyzing k8s ingress logs for anomalous patterns...",
            "entryType": "analysis"
          },
          {
            "timestamp": "09:41:05",
            "text": "Detected 3x increase in p99 latency starting at 09:38",
            "entryType": "finding"
          },
          {
            "timestamp": "09:41:08",
            "text": "Correlating with recent deployment DEP-041 (v2.4.0) at 09:35",
            "entryType": "analysis"
          },
          {
            "timestamp": "09:41:12",
            "text": "Pattern identified: SSL handshake timeout causing connection queuing",
            "entryType": "finding"
          },
          {
            "timestamp": "09:41:15",
            "text": "Root cause: New TLS config in v2.4.0 increases handshake time by 300ms under load",
            "entryType": "conclusion"
          },
          {
            "timestamp": "09:41:18",
            "text": "SUGGESTION: Scale pod replicas from 3 to 5 to absorb load while TLS config is patched",
            "entryType": "suggestion"
          }
        ],
        "rootCause": {
          "hypothesis": "TLS configuration change in deployment v2.4.0 causes SSL handshake timeout under load, queuing connections at the ingress layer",
          "confidence": "High"
        },
        "affectedComponents": [
          "api-gateway",
          "ingress-controller",
          "product-service"
        ]
      },
      "error": null
    },
    "skillTimeline": {
      "data": {
        "executions": [
          {
            "skillName": "incident-detection",
            "startTime": "2026-04-16T09:40:00Z",
            "endTime": "2026-04-16T09:40:02Z",
            "status": "completed",
            "inputSummary": "Anomaly signal: p99 latency > 500ms on api-gateway",
            "outputSummary": "Incident INC-0422 created, priority P1"
          },
          {
            "skillName": "incident-correlation",
            "startTime": "2026-04-16T09:40:03Z",
            "endTime": "2026-04-16T09:40:08Z",
            "status": "completed",
            "inputSummary": "INC-0422 signals + recent change log",
            "outputSummary": "Correlated with DEP-041 (v2.4.0) deployed at 09:35"
          },
          {
            "skillName": "incident-diagnosis",
            "startTime": "2026-04-16T09:41:00Z",
            "endTime": "2026-04-16T09:41:15Z",
            "status": "completed",
            "inputSummary": "INC-0422 + correlation data + ingress logs",
            "outputSummary": "Root cause: TLS config change causing handshake timeout"
          },
          {
            "skillName": "incident-remediation",
            "startTime": "2026-04-16T09:41:16Z",
            "endTime": "2026-04-16T09:41:18Z",
            "status": "pending_approval",
            "inputSummary": "Root cause + affected components",
            "outputSummary": "Proposed: Scale replicas 3→5 (requires approval per policy)"
          }
        ]
      },
      "error": null
    },
    "actions": {
      "data": {
        "actions": [
          {
            "id": "ACT-001",
            "description": "Scale api-gateway pod replicas from 3 to 5",
            "actionType": "requires_approval",
            "executionStatus": "pending",
            "timestamp": "2026-04-16T09:41:18Z",
            "impactAssessment": "Increases resource allocation by 67%; absorbs current load spike; no downtime",
            "isRollbackable": true,
            "policyRef": "POL-003: Infrastructure scaling requires approval for >50% capacity change"
          }
        ]
      },
      "error": null
    },
    "governance": {
      "data": {
        "entries": []
      },
      "error": null
    },
    "sdlcChain": {
      "data": {
        "links": [
          {
            "artifactType": "spec",
            "artifactId": "SPEC-089",
            "artifactTitle": "API Gateway TLS Configuration Spec",
            "routePath": "/requirements"
          },
          {
            "artifactType": "code",
            "artifactId": "MR-1234",
            "artifactTitle": "feat: update TLS config for mTLS support",
            "routePath": "/code"
          },
          {
            "artifactType": "deploy",
            "artifactId": "DEP-041",
            "artifactTitle": "Release v2.4.0 to production",
            "routePath": "/deployments"
          }
        ]
      },
      "error": null
    },
    "learning": {
      "data": null,
      "error": null
    }
  },
  "error": null
}
```

### 3.3 Error Cases

| Status | When | Error Message |
|--------|------|---------------|
| 404 | Incident ID not found | "Incident not found: INC-9999" |
| 500 | Internal server error | "Failed to load incident detail" |

---

## 4. POST /api/v1/incidents/:id/actions/:actionId/approve

### 4.1 Request

```
POST /api/v1/incidents/INC-0422/actions/ACT-001/approve
Content-Type: application/json
```

**Request body:** None (empty body or `{}`).

### 4.2 Success Response (200 OK)

```json
{
  "data": {
    "actionId": "ACT-001",
    "newStatus": "approved",
    "governanceEntry": {
      "actor": "leo.chen",
      "timestamp": "2026-04-16T11:03:45Z",
      "actionTaken": "approve",
      "reason": null,
      "policyRef": null
    }
  },
  "error": null
}
```

### 4.3 Error Cases

| Status | When | Error Message |
|--------|------|---------------|
| 400 | Action not in PENDING state | "Action ACT-001 is not in PENDING state" |
| 404 | Incident or action not found | "Action not found: ACT-001" |

---

## 5. POST /api/v1/incidents/:id/actions/:actionId/reject

### 5.1 Request

```
POST /api/v1/incidents/INC-0422/actions/ACT-001/reject
Content-Type: application/json
```

**Request body:**

```json
{
  "reason": "Prefer to roll back deployment v2.4.0 instead of scaling"
}
```

| Field | Type | Required | Validation |
|-------|------|----------|-----------|
| `reason` | String | Yes | Non-empty, max 500 characters |

### 5.2 Success Response (200 OK)

```json
{
  "data": {
    "actionId": "ACT-001",
    "newStatus": "rejected",
    "governanceEntry": {
      "actor": "leo.chen",
      "timestamp": "2026-04-16T11:03:45Z",
      "actionTaken": "reject",
      "reason": "Prefer to roll back deployment v2.4.0 instead of scaling",
      "policyRef": null
    }
  },
  "error": null
}
```

### 5.3 Error Cases

| Status | When | Error Message |
|--------|------|---------------|
| 400 | Action not in PENDING state | "Action ACT-001 is not in PENDING state" |
| 400 | Missing or empty reason | "Rejection reason is required" |
| 404 | Incident or action not found | "Action not found: ACT-001" |

---

## 6. Backend Implementation Notes

### 6.1 Controller Structure

```
IncidentController
  @GetMapping(INCIDENTS)          → listIncidents(filters)
  @GetMapping(INCIDENT_DETAIL)    → getIncidentDetail(incidentId)
  @PostMapping(ACTION_APPROVE)    → approveAction(incidentId, actionId)
  @PostMapping(ACTION_REJECT)     → rejectAction(incidentId, actionId, body)
```

### 6.2 Service Layer

`IncidentService` assembles responses:
- `getIncidentList(filters)` → returns `IncidentListDto` with severity distribution
- `getIncidentDetail(id)` → returns `IncidentDetailDto` with 7 `SectionResultDto` sections
- `approveAction(incidentId, actionId)` → validates state, transitions, creates governance entry
- `rejectAction(incidentId, actionId, reason)` → validates state + reason, transitions, creates entry

Phase A: Service returns hard-coded seed data (same pattern as `DashboardService`).
Phase B: Service reads from JPA repositories.

### 6.3 Shared Infrastructure

`SectionResultDto<T>` is currently in `domain/dashboard/dto/`. For the incident module to use it:
- **Option A**: Move `SectionResultDto` to `shared/dto/` (recommended)
- **Option B**: Duplicate in `domain/incident/dto/` (not recommended)

---

## 7. Frontend Implementation Notes

### 7.1 API Client

The existing `fetchJson<T>` in `shared/api/client.ts` only supports GET requests.
Approve/reject require POST. Add a `postJson<T>` function to the shared client in task A0:

```typescript
// shared/api/client.ts — new function
export async function postJson<T>(path: string, body?: unknown): Promise<T> {
  const response = await fetch(`${API_BASE}${path}`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: body != null ? JSON.stringify(body) : undefined,
  });
  if (!response.ok) {
    throw new ApiError(response.status, response.statusText);
  }
  const envelope: ApiEnvelope<T> = await response.json();
  if (envelope.error) {
    throw new ApiError(response.status, envelope.error, envelope.error);
  }
  return envelope.data as T;
}
```

```
incidentApi.ts
  getIncidentList(filters?)  → fetchJson<IncidentListDto>('/incidents')
  getIncidentDetail(id)      → fetchJson<IncidentDetailDto>(`/incidents/${id}`)
  approveAction(incidentId, actionId)  → postJson<ActionApprovalResult>(`/incidents/${incidentId}/actions/${actionId}/approve`)
  rejectAction(incidentId, actionId, reason)  → postJson<ActionApprovalResult>(`/incidents/${incidentId}/actions/${actionId}/reject`, { reason })
```

GET endpoints use `fetchJson<T>`, POST endpoints use `postJson<T>` — both from `shared/api/client.ts`.

### 7.2 Vite Proxy

No proxy changes needed. The existing proxy in `vite.config.ts:13` already forwards
all `/api/*` traffic to `http://localhost:8080`, which covers `/api/v1/incidents/*`.

### 7.3 Mock Data Toggle

Store checks `import.meta.env.DEV && !import.meta.env.VITE_USE_BACKEND` to decide between
mock and live data. This matches the verified dashboard store pattern (`dashboardStore.ts:7`).

---

## 8. State Reference

### Incident Status

```
DETECTED → AI_INVESTIGATING → AI_DIAGNOSED → ACTION_PROPOSED
  → PENDING_APPROVAL → EXECUTING → RESOLVED → LEARNING → CLOSED

Overrides: ESCALATED, MANUAL_OVERRIDE → RESOLVED
```

### Action Status

```
PENDING → APPROVED → EXECUTING → EXECUTED | ROLLED_BACK
PENDING → REJECTED
```

### Skill Status

```
RUNNING → COMPLETED | FAILED | PENDING_APPROVAL
```

---

## 9. Testing Contracts

### Backend (MockMvc)

| Test | Method | Path | Expected |
|------|--------|------|----------|
| List returns 200 with incidents | GET | /api/v1/incidents | 200, body contains `incidents[]` and `severityDistribution` |
| List with priority filter | GET | /api/v1/incidents?priority=P1 | 200, all incidents are P1 |
| Detail returns 200 with 7 sections | GET | /api/v1/incidents/INC-0422 | 200, body contains header, diagnosis, skillTimeline, actions, governance, sdlcChain, learning |
| Detail with invalid ID returns 404 | GET | /api/v1/incidents/INC-9999 | 404 |
| Approve returns 200 | POST | /api/v1/incidents/INC-0422/actions/ACT-001/approve | 200, newStatus = "approved" |
| Reject returns 200 | POST | /api/v1/incidents/INC-0422/actions/ACT-001/reject | 200, newStatus = "rejected" |
| Reject without reason returns 400 | POST | /api/v1/incidents/INC-0422/actions/ACT-001/reject | 400, body `{}` |
