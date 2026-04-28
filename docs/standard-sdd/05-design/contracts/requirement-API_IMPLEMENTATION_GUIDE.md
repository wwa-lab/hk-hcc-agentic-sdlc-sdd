# Requirement Management — API Implementation Guide

## Purpose

This document is the single source of truth for implementing the Requirement Management API.
It defines endpoint contracts, request/response shapes, error handling, and integration
patterns for both frontend and backend developers.

## Traceability

- Requirements: [requirement-requirements.md](../../01-requirements/requirement-requirements.md)
- Spec: [requirement-spec.md](../../03-spec/requirement-spec.md)
- Architecture: [requirement-architecture.md](../../04-architecture/requirement-architecture.md)
- Design: [requirement-design.md](../requirement-design.md)
- Data Model: [requirement-data-model.md](../../04-architecture/requirement-data-model.md)
- Data Flow: [requirement-data-flow.md](../../04-architecture/requirement-data-flow.md)

---

## 1. API Overview

| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/requirements` | List requirements with optional filters |
| GET | `/api/v1/requirements/:id` | Full requirement detail with all 6 sections |
| GET | `/api/v1/requirements/:id/chain` | SDLC chain for a requirement |
| GET | `/api/v1/requirements/:id/analysis` | AI analysis results for a requirement |
| POST | `/api/v1/requirements/:id/generate-stories` | Trigger AI story generation from requirement |
| POST | `/api/v1/requirements/:id/generate-spec` | Trigger AI spec generation from stories |
| POST | `/api/v1/requirements/:id/analyze` | Trigger AI analysis for a requirement |
| PATCH | `/api/v1/requirements/:id/status` | Update requirement status (kanban drag-drop) |
| GET | `/api/v1/pipeline-profiles/active` | Get resolved pipeline profile for workspace |
| POST | `/api/v1/requirements/:id/invoke-skill` | Invoke a profile-specific skill |
| POST | `/api/v1/requirements/normalize` | Normalize pasted raw input into requirement draft |
| POST | `/api/v1/requirements/imports` | Submit KB-backed file import and receive async receipt |
| GET | `/api/v1/requirements/imports/:importId` | Poll KB-backed file import status and retrieve draft when ready |
| POST | `/api/v1/requirements` | Create requirement from confirmed draft |

### Base URL

- Local dev: `http://localhost:8080/api/v1`
- Frontend proxy (Vite): `/api/v1` -> `http://localhost:8080/api/v1`

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

### Date Format

All timestamps use ISO 8601 format: `2026-04-16T09:15:00Z`

### Backend Constants

Add to `ApiConstants.java`:

```java
public static final String REQUIREMENTS = API_V1 + "/requirements";
public static final String REQUIREMENT_DETAIL = REQUIREMENTS + "/{requirementId}";
public static final String REQUIREMENT_CHAIN = REQUIREMENT_DETAIL + "/chain";
public static final String REQUIREMENT_ANALYSIS = REQUIREMENT_DETAIL + "/analysis";
public static final String REQUIREMENT_GENERATE_STORIES = REQUIREMENT_DETAIL + "/generate-stories";
public static final String REQUIREMENT_GENERATE_SPEC = REQUIREMENT_DETAIL + "/generate-spec";
public static final String REQUIREMENT_ANALYZE = REQUIREMENT_DETAIL + "/analyze";
public static final String REQUIREMENT_STATUS = REQUIREMENT_DETAIL + "/status";
```

---

## 2. GET /api/v1/requirements

### 2.1 Request

```
GET /api/v1/requirements
Accept: application/json
```

**Query Parameters:**

| Param | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `status` | String | No | -- | Filter by status: DRAFT, IN_REVIEW, APPROVED, IN_PROGRESS, DELIVERED, ARCHIVED |
| `priority` | String | No | -- | Filter by priority: CRITICAL, HIGH, MEDIUM, LOW |
| `category` | String | No | -- | Filter by category: FUNCTIONAL, NON_FUNCTIONAL, TECHNICAL, BUSINESS |
| `search` | String | No | -- | Text search across title and description |
| `sortBy` | String | No | updatedAt | Sort field: priority, status, updatedAt, createdAt, title, completenessScore |
| `sortDirection` | String | No | desc | Sort direction: asc, desc |

### 2.2 Success Response (200 OK)

```json
{
  "data": {
    "items": [
      {
        "id": "REQ-001",
        "title": "Support multi-workspace requirement isolation",
        "priority": "HIGH",
        "status": "APPROVED",
        "category": "FUNCTIONAL",
        "storyCount": 3,
        "specCount": 1,
        "completenessScore": 85,
        "assignee": "Alice Chen",
        "updatedAt": "2026-04-15T10:30:00Z",
        "createdAt": "2026-04-10T08:00:00Z"
      },
      {
        "id": "REQ-002",
        "title": "AI-assisted requirement completeness validation",
        "priority": "CRITICAL",
        "status": "IN_PROGRESS",
        "category": "FUNCTIONAL",
        "storyCount": 5,
        "specCount": 2,
        "completenessScore": 72,
        "assignee": "Bob Wang",
        "updatedAt": "2026-04-16T09:15:00Z",
        "createdAt": "2026-04-11T14:00:00Z"
      },
      {
        "id": "REQ-003",
        "title": "Define SLA thresholds for P1 incident response",
        "priority": "MEDIUM",
        "status": "IN_REVIEW",
        "category": "NON_FUNCTIONAL",
        "storyCount": 2,
        "specCount": 0,
        "completenessScore": 45,
        "assignee": "Carol Li",
        "updatedAt": "2026-04-14T16:20:00Z",
        "createdAt": "2026-04-12T09:30:00Z"
      },
      {
        "id": "REQ-004",
        "title": "Implement role-based access control for requirement editing",
        "priority": "HIGH",
        "status": "DRAFT",
        "category": "TECHNICAL",
        "storyCount": 0,
        "specCount": 0,
        "completenessScore": 20,
        "assignee": "David Zhang",
        "updatedAt": "2026-04-16T07:45:00Z",
        "createdAt": "2026-04-15T11:00:00Z"
      },
      {
        "id": "REQ-005",
        "title": "Deliver executive dashboard with requirement pipeline metrics",
        "priority": "LOW",
        "status": "DELIVERED",
        "category": "BUSINESS",
        "storyCount": 4,
        "specCount": 3,
        "completenessScore": 100,
        "assignee": "Emily Xu",
        "updatedAt": "2026-04-13T14:00:00Z",
        "createdAt": "2026-04-05T10:00:00Z"
      }
    ],
    "statusDistribution": {
      "DRAFT": 3,
      "IN_REVIEW": 2,
      "APPROVED": 5,
      "IN_PROGRESS": 4,
      "DELIVERED": 8,
      "ARCHIVED": 1
    },
    "totalCount": 23
  },
  "error": null
}
```

### 2.3 Error Cases

| Status | When | Error Message |
|--------|------|---------------|
| 400 | Invalid filter parameter value | "Invalid status value: UNKNOWN" |
| 500 | Internal server error | "Failed to load requirements" |

---

## 3. GET /api/v1/requirements/:id

### 3.1 Request

```
GET /api/v1/requirements/REQ-001
Accept: application/json
```

**Path Parameters:**

| Param | Type | Required | Description |
|-------|------|----------|-------------|
| `id` | String | Yes | Requirement ID (e.g., REQ-001) |

### 3.2 Success Response (200 OK)

The response uses a two-level envelope:
1. **Top level**: `ApiResponse<RequirementDetailDto>`
2. **Section level**: `SectionResultDto<T>` per card for error isolation

```json
{
  "data": {
    "header": {
      "data": {
        "id": "REQ-001",
        "title": "Support multi-workspace requirement isolation",
        "priority": "HIGH",
        "status": "APPROVED",
        "category": "FUNCTIONAL",
        "assignee": "Alice Chen",
        "source": "MANUAL",
        "completenessScore": 85,
        "createdAt": "2026-04-10T08:00:00Z",
        "updatedAt": "2026-04-15T10:30:00Z"
      },
      "error": null
    },
    "description": {
      "data": {
        "body": "All requirement data, user stories, and specs must be scoped to the active workspace. Users must not see or interact with artifacts from other workspaces. Cross-workspace dependencies must be explicitly declared and governed.",
        "businessContext": "Enterprise customers operate multiple independent product lines within a single Control Tower instance. Data leakage between workspaces poses compliance risk and user confusion.",
        "businessValue": "Enables multi-tenant deployment model, reducing infrastructure cost by 60% compared to per-tenant instances.",
        "stakeholders": ["Product Management", "Enterprise Architecture", "Security"],
        "externalReferences": [
          {
            "system": "JIRA",
            "id": "SDLC-1042",
            "url": "https://jira.example.com/browse/SDLC-1042"
          }
        ],
        "acceptanceCriteria": [
          {
            "id": "AC-001",
            "description": "Switching workspace context filters all requirement lists to the selected workspace only",
            "status": "PASSED"
          },
          {
            "id": "AC-002",
            "description": "A user with access to Workspace A cannot see requirements from Workspace B via API or UI",
            "status": "PASSED"
          },
          {
            "id": "AC-003",
            "description": "Cross-workspace requirement dependencies display a governance warning before linking",
            "status": "NOT_TESTED"
          }
        ]
      },
      "error": null
    },
    "stories": {
      "data": {
        "items": [
          {
            "id": "S-001",
            "title": "As a PM, I can switch workspace context and see only that workspace's requirements",
            "status": "DONE",
            "specId": "SPEC-012",
            "specStatus": "APPROVED"
          },
          {
            "id": "S-002",
            "title": "As an admin, I can configure workspace isolation rules for requirement visibility",
            "status": "IN_PROGRESS",
            "specId": null,
            "specStatus": null
          },
          {
            "id": "S-003",
            "title": "As a security auditor, I can verify no cross-workspace data leakage in requirement queries",
            "status": "DRAFT",
            "specId": null,
            "specStatus": null
          }
        ],
        "totalStories": 3,
        "storiesWithSpec": 1,
        "coveragePercent": 33
      },
      "error": null
    },
    "specs": {
      "data": {
        "items": [
          {
            "id": "SPEC-012",
            "title": "Workspace-scoped requirement query specification",
            "status": "APPROVED",
            "version": "1.2",
            "linkedStoryIds": ["S-001"],
            "updatedAt": "2026-04-14T11:00:00Z"
          }
        ],
        "totalSpecs": 1,
        "approvedSpecs": 1
      },
      "error": null
    },
    "chain": {
      "data": {
        "links": [
          {
            "artifactType": "requirement",
            "artifactId": "REQ-001",
            "artifactTitle": "Support multi-workspace requirement isolation",
            "routePath": "/requirements/REQ-001",
            "status": "APPROVED"
          },
          {
            "artifactType": "story",
            "artifactId": "S-001",
            "artifactTitle": "Workspace context switching for requirements",
            "routePath": "/stories/S-001",
            "status": "DONE"
          },
          {
            "artifactType": "spec",
            "artifactId": "SPEC-012",
            "artifactTitle": "Workspace-scoped requirement query specification",
            "routePath": "/specs/SPEC-012",
            "status": "APPROVED"
          },
          {
            "artifactType": "code",
            "artifactId": "MR-2089",
            "artifactTitle": "feat: workspace-scoped requirement repository",
            "routePath": "/code/MR-2089",
            "status": "MERGED"
          },
          {
            "artifactType": "test",
            "artifactId": "TS-045",
            "artifactTitle": "Workspace isolation integration tests",
            "routePath": "/tests/TS-045",
            "status": "PASSED"
          },
          {
            "artifactType": "deploy",
            "artifactId": "DEP-038",
            "artifactTitle": "Release v2.3.0 with workspace isolation",
            "routePath": "/deployments/DEP-038",
            "status": "DEPLOYED"
          }
        ]
      },
      "error": null
    },
    "analysis": {
      "data": {
        "qualityScore": 82,
        "qualityBreakdown": {
          "completeness": 90,
          "clarity": 85,
          "testability": 80,
          "consistency": 75
        },
        "suggestions": [
          {
            "type": "AMBIGUITY",
            "severity": "MEDIUM",
            "message": "Acceptance criterion AC-003 references 'governance warning' without specifying the warning content or dismissal behavior",
            "field": "acceptanceCriteria[2]"
          },
          {
            "type": "GAP",
            "severity": "LOW",
            "message": "No acceptance criterion covers the scenario where a requirement is shared across workspaces by explicit declaration",
            "field": "acceptanceCriteria"
          }
        ],
        "duplicateCheck": {
          "hasPotentialDuplicates": false,
          "candidates": []
        },
        "impactAnalysis": {
          "downstreamArtifacts": 6,
          "affectedTeams": ["Platform", "Security"],
          "riskLevel": "LOW"
        },
        "analyzedAt": "2026-04-15T10:30:00Z"
      },
      "error": null
    }
  },
  "error": null
}
```

### 3.3 Error Cases

| Status | When | Error Message |
|--------|------|---------------|
| 404 | Requirement ID not found | "Requirement not found: REQ-9999" |
| 500 | Internal server error | "Failed to load requirement detail" |

---

## 4. GET /api/v1/requirements/:id/chain

### 4.1 Request

```
GET /api/v1/requirements/REQ-001/chain
Accept: application/json
```

**Path Parameters:**

| Param | Type | Required | Description |
|-------|------|----------|-------------|
| `id` | String | Yes | Requirement ID (e.g., REQ-001) |

### 4.2 Success Response (200 OK)

```json
{
  "data": [
    {
      "artifactType": "requirement",
      "artifactId": "REQ-001",
      "artifactTitle": "Support multi-workspace requirement isolation",
      "routePath": "/requirements/REQ-001",
      "status": "APPROVED"
    },
    {
      "artifactType": "story",
      "artifactId": "S-001",
      "artifactTitle": "Workspace context switching for requirements",
      "routePath": "/stories/S-001",
      "status": "DONE"
    },
    {
      "artifactType": "spec",
      "artifactId": "SPEC-012",
      "artifactTitle": "Workspace-scoped requirement query specification",
      "routePath": "/specs/SPEC-012",
      "status": "APPROVED"
    },
    {
      "artifactType": "code",
      "artifactId": "MR-2089",
      "artifactTitle": "feat: workspace-scoped requirement repository",
      "routePath": "/code/MR-2089",
      "status": "MERGED"
    },
    {
      "artifactType": "test",
      "artifactId": "TS-045",
      "artifactTitle": "Workspace isolation integration tests",
      "routePath": "/tests/TS-045",
      "status": "PASSED"
    },
    {
      "artifactType": "deploy",
      "artifactId": "DEP-038",
      "artifactTitle": "Release v2.3.0 with workspace isolation",
      "routePath": "/deployments/DEP-038",
      "status": "DEPLOYED"
    }
  ],
  "error": null
}
```

### 4.3 Error Cases

| Status | When | Error Message |
|--------|------|---------------|
| 404 | Requirement ID not found | "Requirement not found: REQ-9999" |
| 500 | Internal server error | "Failed to load SDLC chain" |

---

## 5. GET /api/v1/requirements/:id/analysis

### 5.1 Request

```
GET /api/v1/requirements/REQ-001/analysis
Accept: application/json
```

**Path Parameters:**

| Param | Type | Required | Description |
|-------|------|----------|-------------|
| `id` | String | Yes | Requirement ID (e.g., REQ-001) |

### 5.2 Success Response (200 OK)

```json
{
  "data": {
    "qualityScore": 82,
    "qualityBreakdown": {
      "completeness": 90,
      "clarity": 85,
      "testability": 80,
      "consistency": 75
    },
    "suggestions": [
      {
        "type": "AMBIGUITY",
        "severity": "MEDIUM",
        "message": "Acceptance criterion AC-003 references 'governance warning' without specifying the warning content or dismissal behavior",
        "field": "acceptanceCriteria[2]"
      },
      {
        "type": "GAP",
        "severity": "LOW",
        "message": "No acceptance criterion covers the scenario where a requirement is shared across workspaces by explicit declaration",
        "field": "acceptanceCriteria"
      }
    ],
    "duplicateCheck": {
      "hasPotentialDuplicates": false,
      "candidates": []
    },
    "impactAnalysis": {
      "downstreamArtifacts": 6,
      "affectedTeams": ["Platform", "Security"],
      "riskLevel": "LOW"
    },
    "analyzedAt": "2026-04-15T10:30:00Z"
  },
  "error": null
}
```

### 5.3 Error Cases

| Status | When | Error Message |
|--------|------|---------------|
| 404 | Requirement ID not found | "Requirement not found: REQ-9999" |
| 500 | Internal server error | "Failed to load analysis results" |

---

## 6. POST /api/v1/requirements/:id/generate-stories

### 6.1 Request

```
POST /api/v1/requirements/REQ-001/generate-stories
Content-Type: application/json
```

**Path Parameters:**

| Param | Type | Required | Description |
|-------|------|----------|-------------|
| `id` | String | Yes | Requirement ID (e.g., REQ-001) |

**Request Body (optional):**

```json
{
  "prompt": "Focus on security-related acceptance flows"
}
```

| Field | Type | Required | Validation |
|-------|------|----------|------------|
| `prompt` | String | No | Max 1000 characters. Optional context hint for AI generation |

### 6.2 Success Response (202 Accepted)

```json
{
  "data": {
    "executionId": "EXEC-20260416-001",
    "skillName": "req-to-user-story",
    "status": "RUNNING",
    "requirementId": "REQ-001",
    "startedAt": "2026-04-16T11:30:00Z",
    "estimatedCompletionSeconds": 15,
    "message": "Story generation initiated for REQ-001"
  },
  "error": null
}
```

The client can poll for completion using the execution ID (future polling endpoint).
In Phase A/B, this returns immediately with a completed result and generated stories.

### 6.3 Error Cases

| Status | When | Error Message |
|--------|------|---------------|
| 404 | Requirement ID not found | "Requirement not found: REQ-9999" |
| 409 | Generation already in progress | "Story generation already in progress for REQ-001" |
| 500 | Internal server error | "Failed to initiate story generation" |

---

## 7. POST /api/v1/requirements/:id/generate-spec

### 7.1 Request

```
POST /api/v1/requirements/REQ-001/generate-spec
Content-Type: application/json
```

**Path Parameters:**

| Param | Type | Required | Description |
|-------|------|----------|-------------|
| `id` | String | Yes | Requirement ID (e.g., REQ-001) |

**Request Body:**

```json
{
  "storyIds": ["S-001", "S-002"]
}
```

| Field | Type | Required | Validation |
|-------|------|----------|------------|
| `storyIds` | String[] | Yes | Non-empty array. Each ID must belong to the specified requirement |

### 7.2 Success Response (202 Accepted)

```json
{
  "data": {
    "executionId": "EXEC-20260416-002",
    "skillName": "user-story-to-spec",
    "status": "RUNNING",
    "requirementId": "REQ-001",
    "inputStoryIds": ["S-001", "S-002"],
    "startedAt": "2026-04-16T11:35:00Z",
    "estimatedCompletionSeconds": 30,
    "message": "Spec generation initiated for 2 stories under REQ-001"
  },
  "error": null
}
```

### 7.3 Error Cases

| Status | When | Error Message |
|--------|------|---------------|
| 400 | Empty or missing storyIds | "At least one story ID is required" |
| 400 | Story does not belong to requirement | "Story S-099 is not linked to requirement REQ-001" |
| 404 | Requirement ID not found | "Requirement not found: REQ-9999" |
| 409 | Generation already in progress | "Spec generation already in progress for REQ-001" |
| 500 | Internal server error | "Failed to initiate spec generation" |

---

## 8. POST /api/v1/requirements/:id/analyze

### 8.1 Request

```
POST /api/v1/requirements/REQ-001/analyze
Content-Type: application/json
```

**Path Parameters:**

| Param | Type | Required | Description |
|-------|------|----------|-------------|
| `id` | String | Yes | Requirement ID (e.g., REQ-001) |

**Request Body (optional):**

```json
{
  "forceRefresh": true
}
```

| Field | Type | Required | Validation |
|-------|------|----------|------------|
| `forceRefresh` | Boolean | No | Default false. If true, re-runs analysis even if recent results exist |

### 8.2 Success Response (202 Accepted)

```json
{
  "data": {
    "executionId": "EXEC-20260416-003",
    "skillName": "requirement-analysis",
    "status": "RUNNING",
    "requirementId": "REQ-001",
    "startedAt": "2026-04-16T12:00:00Z",
    "estimatedCompletionSeconds": 10,
    "message": "AI analysis initiated for REQ-001"
  },
  "error": null
}
```

In Phase A/B, this returns immediately with completed status. The analysis result is then available via `GET /api/v1/requirements/:id/analysis`.

### 8.3 Error Cases

| Status | When | Error Message |
|--------|------|---------------|
| 404 | Requirement ID not found | "Requirement not found: REQ-9999" |
| 409 | Analysis already in progress | "Analysis already in progress for REQ-001" |
| 500 | Internal server error | "Failed to initiate analysis" |

---

## 9. PATCH /api/v1/requirements/:id/status

### 9.1 Request

```
PATCH /api/v1/requirements/REQ-001/status
Content-Type: application/json
```

**Path Parameters:**

| Param | Type | Required | Description |
|-------|------|----------|-------------|
| `id` | String | Yes | Requirement ID (e.g., REQ-001) |

**Request Body:**

```json
{
  "status": "IN_REVIEW"
}
```

| Field | Type | Required | Validation |
|-------|------|----------|------------|
| `status` | String | Yes | Must be a valid `RequirementStatus` value. Transition must be allowed by the state machine. |

### 9.2 Success Response (200 OK)

```json
{
  "data": {
    "id": "REQ-001",
    "title": "Support multi-workspace requirement isolation",
    "priority": "HIGH",
    "status": "IN_REVIEW",
    "category": "FUNCTIONAL",
    "assignee": "Alice Chen",
    "updatedAt": "2026-04-16T12:05:00Z"
  },
  "error": null
}
```

### 9.3 Error Cases

| Status | When | Error Message |
|--------|------|---------------|
| 400 | Invalid status value | "Invalid status: UNKNOWN" |
| 400 | Invalid state transition | "Cannot transition from DRAFT to DELIVERED" |
| 404 | Requirement ID not found | "Requirement not found: REQ-9999" |
| 500 | Internal server error | "Failed to update status" |

---

### 2.7 GET /api/v1/pipeline-profiles/active — Get active pipeline profile

Returns the resolved pipeline profile for the current workspace context.

**Query Parameters:**

| Param | Type | Required | Description |
|-------|------|----------|-------------|
| workspaceId | string | Yes | Current workspace ID |
| projectId | string | No | Project override (if any) |

**Response:** `ApiResponse<PipelineProfileDto>`

```json
{
  "data": {
    "id": "standard-sdd",
    "name": "Standard SDD",
    "description": "Standard Spec-Driven Development pipeline for Java/open-source projects",
    "chainNodes": [
      { "id": "requirement", "label": "Requirement", "isExecutionHub": false, "artifactType": "REQUIREMENT", "order": 1 },
      { "id": "user-story", "label": "User Story", "isExecutionHub": false, "artifactType": "USER_STORY", "order": 2 },
      { "id": "spec", "label": "Spec", "isExecutionHub": true, "artifactType": "SPEC", "order": 3 },
      { "id": "architecture", "label": "Architecture", "isExecutionHub": false, "artifactType": "ARCHITECTURE", "order": 4 },
      { "id": "design", "label": "Design", "isExecutionHub": false, "artifactType": "DESIGN", "order": 5 },
      { "id": "tasks", "label": "Tasks", "isExecutionHub": false, "artifactType": "TASK", "order": 6 },
      { "id": "code", "label": "Code", "isExecutionHub": false, "artifactType": "CODE", "order": 7 },
      { "id": "test", "label": "Test", "isExecutionHub": false, "artifactType": "TEST", "order": 8 },
      { "id": "deploy", "label": "Deploy", "isExecutionHub": false, "artifactType": "DEPLOYMENT", "order": 9 },
      { "id": "incident", "label": "Incident", "isExecutionHub": false, "artifactType": "INCIDENT", "order": 10 },
      { "id": "learning", "label": "Learning", "isExecutionHub": false, "artifactType": "LEARNING", "order": 11 }
    ],
    "executionHubNodeId": "spec",
    "skills": [
      { "skillId": "req-to-user-story", "label": "Generate Stories", "order": 1, "requiresEntryPath": null },
      { "skillId": "user-story-to-spec", "label": "Generate Spec", "order": 2, "requiresEntryPath": null }
    ],
    "specTiering": null,
    "entryPaths": [
      { "id": "standard", "label": "Standard", "description": "Full SDD chain from requirement to delivery", "startSkillId": "req-to-user-story", "applicability": "All new features and enhancements" }
    ],
    "traceabilityModel": "per-layer",
    "isDefault": true
  },
  "error": null
}
```

**IBM i Profile Example:**

```json
{
  "data": {
    "id": "ibm-i",
    "name": "IBM i",
    "description": "Unified pipeline for IBM i (iSeries/AS400) BAU development — delegates to ibm-i-workflow-orchestrator for path and tier routing",
    "chainNodes": [
      { "id": "requirement-normalizer", "label": "Requirement Normalizer", "isExecutionHub": false, "artifactType": "REQUIREMENT", "order": 1 },
      { "id": "functional-spec", "label": "Functional Spec", "isExecutionHub": false, "artifactType": "FUNCTIONAL_SPEC", "order": 2 },
      { "id": "technical-design", "label": "Technical Design", "isExecutionHub": false, "artifactType": "TECHNICAL_DESIGN", "order": 3 },
      { "id": "program-spec", "label": "Program Spec", "isExecutionHub": true, "artifactType": "PROGRAM_SPEC", "order": 4 },
      { "id": "file-spec", "label": "File Spec", "isExecutionHub": false, "artifactType": "FILE_SPEC", "order": 5 },
      { "id": "ut-plan", "label": "UT Plan", "isExecutionHub": false, "artifactType": "TEST_PLAN", "order": 6 },
      { "id": "test-scaffold", "label": "Test Scaffold", "isExecutionHub": false, "artifactType": "TEST_SCAFFOLD", "order": 7 },
      { "id": "spec-review", "label": "Spec Review", "isExecutionHub": false, "artifactType": "REVIEW", "order": 8 },
      { "id": "dds-review", "label": "DDS Review", "isExecutionHub": false, "artifactType": "REVIEW", "order": 9 },
      { "id": "code-review", "label": "Code Review", "isExecutionHub": false, "artifactType": "REVIEW", "order": 10 }
    ],
    "executionHubNodeId": "program-spec",
    "skills": [
      { "skillId": "ibm-i-workflow-orchestrator", "label": "Send to Orchestrator", "order": 1, "requiresEntryPath": null }
    ],
    "specTiering": {
      "tiers": [
        { "id": "L1", "label": "Lite", "description": "Single-field change, flag toggle, validation threshold", "sectionCount": 10 },
        { "id": "L2", "label": "Standard", "description": "Moderate logic change, new subroutine, interface change", "sectionCount": 16 },
        { "id": "L3", "label": "Full", "description": "New program, major redesign, multi-file transaction", "sectionCount": 25 }
      ],
      "defaultTier": "L2"
    },
    "entryPaths": [
      { "id": "full-chain", "label": "Full Chain", "description": "New programs or large-scope changes", "startSkillId": "ibm-i-workflow-orchestrator", "applicability": "New programs, major redesign" },
      { "id": "enhancement", "label": "Enhancement", "description": "Existing source modification", "startSkillId": "ibm-i-workflow-orchestrator", "applicability": "Most common: modifying existing programs" },
      { "id": "fast-path", "label": "Fast-Path", "description": "Small, well-understood changes", "startSkillId": "ibm-i-workflow-orchestrator", "applicability": "Enhancement, bug fix, new logic path, error handling only" }
    ],
    "traceabilityModel": "shared-br",
    "isDefault": false,
    "usesOrchestrator": true
  },
  "error": null
}
```

> **Note:** All three entry paths point to the same `startSkillId` (`ibm-i-workflow-orchestrator`) because the orchestrator determines the path automatically based on input analysis. The `entryPaths` array is retained for documentation purposes and for displaying the orchestrator's decision in the UI.

### 2.8 POST /api/v1/requirements/:id/invoke-skill — Invoke a profile-specific skill

Triggers a skill from the active pipeline profile against a requirement.

**Request Body (Standard SDD):**
```json
{
  "skillId": "req-to-user-story"
}
```

**Request Body (IBM i — orchestrator):**
```json
{
  "skillId": "ibm-i-workflow-orchestrator"
}
```

> For IBM i, `entryPathId` and `tierId` are NOT sent by the frontend — the orchestrator determines them automatically.

**Response (Standard SDD):** `ApiResponse<SkillExecutionResultDto>`
```json
{
  "data": {
    "executionId": "EXEC-20260416-001",
    "skillId": "req-to-user-story",
    "status": "TRIGGERED",
    "message": "Skill 'Generate Stories' has been triggered for REQ-001",
    "orchestratorResult": null
  },
  "error": null
}
```

**Response (IBM i — orchestrator):** `ApiResponse<SkillExecutionResultDto>`
```json
{
  "data": {
    "executionId": "EXEC-20260416-002",
    "skillId": "ibm-i-workflow-orchestrator",
    "status": "TRIGGERED",
    "message": "Orchestrator has been triggered for REQ-001",
    "orchestratorResult": {
      "determinedPathId": "enhancement",
      "determinedPathLabel": "Enhancement",
      "determinedTierId": "L2",
      "determinedTierLabel": "Standard",
      "reasoning": "Existing RPG/COBOL source modification detected — source analysis required before Program Spec generation. Moderate complexity suggests L2 tier.",
      "confidence": "high"
    }
  },
  "error": null
}
```

### 2.9 POST /api/v1/requirements/normalize — Normalize raw input

Accepts raw business input and invokes the active profile's normalizer skill to produce a structured requirement draft.

This endpoint accepts `application/json` for pasted text, email bodies, or meeting summaries.

**JSON Request Body:**
```json
{
  "rawInput": {
    "sourceType": "TEXT",
    "text": "From: Alice Chen\nSubject: Order processing needs multi-currency support\n\nHi team,\n\nAfter our meeting with the APAC sales team, we need to support multi-currency in the order processing module. Key points:\n- Support USD, EUR, GBP, JPY, AUD at minimum\n- Real-time exchange rate lookup from treasury system\n- Orders should display both local currency and USD equivalent\n- Monthly reconciliation report needed\n\nDeadline: Q3 2026\nPriority: High\n\nThanks,\nAlice",
    "fileName": null,
    "fileSize": null
  },
  "profileId": "standard-sdd"
}
```

**Response:** `ApiResponse<RequirementDraftDto>`

```json
{
  "data": {
    "title": "Support multi-currency in order processing module",
    "prioritySuggestion": "HIGH",
    "categorySuggestion": "FUNCTIONAL",
    "description": "The order processing module must support multi-currency transactions to serve the APAC sales team. Orders should be processed in local currencies while maintaining USD equivalents for reporting.",
    "businessJustification": "APAC sales team requires local currency support to serve regional customers effectively. Current USD-only processing creates friction and reporting gaps.",
    "acceptanceCriteria": [
      "System supports USD, EUR, GBP, JPY, AUD currencies",
      "Real-time exchange rate lookup from treasury system API",
      "Order display shows both local currency amount and USD equivalent",
      "Monthly reconciliation report available with currency breakdown"
    ],
    "assumptions": [
      "Treasury system API is available and accessible",
      "Exchange rates are updated at least daily"
    ],
    "constraints": [
      "Must be delivered by Q3 2026"
    ],
    "missingInfo": [
      {
        "field": "stakeholder",
        "message": "APAC sales team lead not identified — who is the business owner?",
        "severity": "warning"
      },
      {
        "field": "compliance",
        "message": "Currency handling may have regulatory requirements — not addressed in source",
        "severity": "info"
      }
    ],
    "openQuestions": [
      {
        "id": "OQ-1",
        "question": "Should the system support additional currencies beyond the initial five?",
        "context": "Source says 'at minimum' — implies potential expansion"
      },
      {
        "id": "OQ-2",
        "question": "What is the fallback if the treasury system API is unavailable?",
        "context": "No error handling mentioned for exchange rate lookup"
      }
    ],
    "normalizedBy": "req-normalizer",
    "normalizedAt": "2026-04-16T08:30:00Z"
  },
  "error": null
}
```

### 2.9A POST /api/v1/requirements/imports — Submit KB-backed file import

Accepts KB-aligned file uploads, submits them to the configured knowledge-base provider, and returns an async receipt immediately.

**Multipart Contract:**

- `Content-Type`: `multipart/form-data`
- Form field `kb_name` (`string`, required): target knowledge base name
- Form field `file` (`file`, required, repeatable): one or many uploaded source files
- Form field `profileId` (`string`, optional): requirement profile override; defaults to active profile if omitted
- Total request size limit: `100 MB`
- Supported uploaded formats: `.txt`, `.md`, `.pdf`, `.html`, `.htm`, `.xlsx`, `.xls`, `.docx`, `.csv`, `.zip`
- ZIP archives are unpacked during intake; supported inner files are submitted individually to the KB provider

**Response:** `ApiResponse<RequirementImportStatusDto>`

```json
{
  "data": {
    "importId": "254d73cc-7adf-42b7-abcf-6bd84023d0a6",
    "taskId": "KB-TASK-20260416-001",
    "status": "PROCESSING",
    "message": "Files uploaded successfully for knowledge base 'example_kb'. Processing in background.",
    "knowledgeBaseName": "example_kb",
    "datasetId": "dataset-123",
    "totalNumberOfFiles": 3,
    "numberOfSuccesses": 2,
    "numberOfFailures": 1,
    "totalSize": 123456,
    "unsupportedFileTypes": [".exe"],
    "supportedFileTypes": [".txt", ".md", ".pdf", ".html", ".xlsx", ".xls", ".docx", ".csv", ".htm"],
    "files": [
      {
        "displayName": "summary.md",
        "sourceFileName": "summary.md",
        "sourceKind": "FILE",
        "fileExtension": ".md",
        "fileSize": 2048,
        "supported": true,
        "providerStatus": "COMPLETED",
        "providerDocumentId": "doc-001"
      },
      {
        "displayName": "payload.exe",
        "sourceFileName": "payload.exe",
        "sourceKind": "FILE",
        "fileExtension": ".exe",
        "fileSize": 1024,
        "supported": false,
        "providerStatus": "UNSUPPORTED",
        "errorMessage": "Unsupported file type for knowledge base ingestion"
      }
    ],
    "createdAt": "2026-04-16T08:30:00Z",
    "updatedAt": "2026-04-16T08:30:00Z"
  },
  "error": null
}
```

### 2.9B GET /api/v1/requirements/imports/:importId — Poll import status

Returns the latest import status. Once KB indexing and requirement normalization finish, the response includes the `draft`.

**Response:** `ApiResponse<RequirementImportStatusDto>`

```json
{
  "data": {
    "importId": "254d73cc-7adf-42b7-abcf-6bd84023d0a6",
    "taskId": "KB-TASK-20260416-001",
    "status": "DRAFT_READY",
    "message": "Draft ready for review.",
    "knowledgeBaseName": "example_kb",
    "datasetId": "dataset-123",
    "totalNumberOfFiles": 2,
    "numberOfSuccesses": 2,
    "numberOfFailures": 0,
    "totalSize": 123456,
    "unsupportedFileTypes": [],
    "supportedFileTypes": [".txt", ".md", ".pdf", ".html", ".xlsx", ".xls", ".docx", ".csv", ".htm"],
    "files": [
      {
        "displayName": "summary.md",
        "sourceFileName": "summary.md",
        "sourceKind": "FILE",
        "fileExtension": ".md",
        "fileSize": 2048,
        "supported": true,
        "providerStatus": "COMPLETED",
        "preview": "Need a batch monitor for overnight IBM i failures...",
        "providerDocumentId": "doc-001"
      }
    ],
    "draft": {
      "title": "Imported requirement package from 2 files",
      "prioritySuggestion": "Medium",
      "categorySuggestion": "Functional",
      "description": "Imported 2 source file(s) for knowledge base example_kb...",
      "businessJustification": "Business value to be confirmed during review.",
      "acceptanceCriteria": ["Alert support within five minutes"],
      "assumptions": [],
      "constraints": [],
      "missingInfo": ["Business owner not identified"],
      "openQuestions": ["Which uploaded file should be treated as the source of truth for this requirement?"],
      "normalizedBy": "standard-sdd-normalizer",
      "normalizedAt": "2026-04-16T08:31:00Z"
    },
    "createdAt": "2026-04-16T08:30:00Z",
    "updatedAt": "2026-04-16T08:31:00Z"
  },
  "error": null
}
```

### 2.10 POST /api/v1/requirements — Create requirement from draft

Creates a new requirement from a confirmed draft.

**Request Body:**
```json
{
  "title": "Support multi-currency in order processing module",
  "priority": "HIGH",
  "category": "FUNCTIONAL",
  "description": "The order processing module must support multi-currency transactions...",
  "businessJustification": "APAC sales team requires local currency support...",
  "acceptanceCriteria": [
    "System supports USD, EUR, GBP, JPY, AUD currencies",
    "Real-time exchange rate lookup from treasury system API",
    "Order display shows both local currency amount and USD equivalent",
    "Monthly reconciliation report available with currency breakdown"
  ],
  "assumptions": ["Treasury system API is available and accessible"],
  "constraints": ["Must be delivered by Q3 2026"],
  "sourceAttachment": {
    "sourceType": "TEXT",
    "text": "From: Alice Chen\nSubject: Order processing needs multi-currency support\n...",
    "fileName": null
  }
}
```

**Response:** `ApiResponse<RequirementListItemDto>`
```json
{
  "data": {
    "id": "REQ-024",
    "title": "Support multi-currency in order processing module",
    "priority": "HIGH",
    "status": "DRAFT",
    "category": "FUNCTIONAL",
    "storyCount": 0,
    "specCount": 0,
    "completenessScore": 0,
    "assignee": null,
    "updatedAt": "2026-04-16T08:35:00Z",
    "createdAt": "2026-04-16T08:35:00Z"
  },
  "error": null
}
```

---

## 10. Backend Implementation Notes

### 10.1 Controller Structure

```
RequirementController
  @GetMapping(REQUIREMENTS)                    -> listRequirements(filters)
  @GetMapping(REQUIREMENT_DETAIL)              -> getRequirementDetail(requirementId)
  @GetMapping(REQUIREMENT_CHAIN)               -> getRequirementChain(requirementId)
  @GetMapping(REQUIREMENT_ANALYSIS)            -> getRequirementAnalysis(requirementId)
  @PostMapping(REQUIREMENT_GENERATE_STORIES)   -> generateStories(requirementId, body)
  @PostMapping(REQUIREMENT_GENERATE_SPEC)      -> generateSpec(requirementId, body)
  @PostMapping(REQUIREMENT_ANALYZE)            -> analyze(requirementId, body)
  @PatchMapping(REQUIREMENT_STATUS)            -> updateStatus(requirementId, body)
```

### 10.2 Service Layer

`RequirementService` assembles responses:
- `getRequirementList(filters)` -> returns `RequirementListDto` with status distribution and items
- `getRequirementDetail(id)` -> returns `RequirementDetailDto` with 6 `SectionResultDto` sections
- `getRequirementChain(id)` -> returns `List<SdlcChainLinkDto>`
- `getRequirementAnalysis(id)` -> returns `AiAnalysisDto`
- `generateStories(id, prompt)` -> validates requirement exists, returns `SkillExecutionResultDto`
- `generateSpec(id, storyIds)` -> validates requirement + stories, returns `SkillExecutionResultDto`

Phase A: Service returns hard-coded seed data (same pattern as `DashboardService` and `IncidentService`).
Phase B: Service reads from JPA repositories.

### 10.3 DTO Structure

```
domain/requirement/
  RequirementController.java
  RequirementService.java
  dto/
    RequirementListDto.java          -- items[], statusDistribution, totalCount
    RequirementSummaryDto.java       -- id, title, priority, status, category, storyCount, specCount, completenessScore, assignee, updatedAt, createdAt
    RequirementDetailDto.java        -- header, description, stories, specs, chain, analysis (all SectionResultDto<T>)
    RequirementHeaderDto.java        -- id, title, priority, status, category, assignee, source, completenessScore, createdAt, updatedAt
    RequirementDescriptionDto.java   -- body, businessContext, businessValue, stakeholders, externalReferences, acceptanceCriteria
    StorySummaryDto.java             -- id, title, status, specId, specStatus
    StorySectionDto.java             -- items[], totalStories, storiesWithSpec, coveragePercent
    SpecSummaryDto.java              -- id, title, status, version, linkedStoryIds, updatedAt
    SpecSectionDto.java              -- items[], totalSpecs, approvedSpecs
    SdlcChainLinkDto.java            -- artifactType, artifactId, artifactTitle, routePath, status
    AiAnalysisDto.java               -- qualityScore, qualityBreakdown, suggestions, duplicateCheck, impactAnalysis, analyzedAt
    SkillExecutionResultDto.java     -- executionId, skillName, status, requirementId, startedAt, estimatedCompletionSeconds, message
    ExternalReferenceDto.java        -- system, id, url
    AcceptanceCriterionDto.java      -- id, description, status
    SuggestionDto.java               -- type, severity, message, field
```

### 8.4 Shared Infrastructure

`SectionResultDto<T>` is already in `shared/dto/SectionResultDto.java`. No move needed.

### 8.5 Package Location

```
com.sdlctower.domain.requirement/
  RequirementController.java
  RequirementService.java
  dto/
    (all DTOs listed above)
```

---

## 9. Frontend Implementation Notes

### 9.1 API Client

Create `features/incident/api/requirementApi.ts` at `features/requirement/api/requirementApi.ts`:

```typescript
// features/requirement/api/requirementApi.ts
import { fetchJson, postJson } from '@/shared/api/client';
import type {
  RequirementListDto,
  RequirementDetailDto,
  SdlcChainLink,
  AiAnalysisDto,
  SkillExecutionResultDto,
} from '../types/requirement';

export interface RequirementFilters {
  readonly status?: string;
  readonly priority?: string;
  readonly category?: string;
  readonly search?: string;
  readonly sortBy?: string;
  readonly sortDirection?: string;
}

function buildQueryString(filters: RequirementFilters): string {
  const params = new URLSearchParams();
  if (filters.status) params.set('status', filters.status);
  if (filters.priority) params.set('priority', filters.priority);
  if (filters.category) params.set('category', filters.category);
  if (filters.search) params.set('search', filters.search);
  if (filters.sortBy) params.set('sortBy', filters.sortBy);
  if (filters.sortDirection) params.set('sortDirection', filters.sortDirection);
  const qs = params.toString();
  return qs ? `?${qs}` : '';
}

export function getRequirementList(
  filters: RequirementFilters = {}
): Promise<RequirementListDto> {
  return fetchJson<RequirementListDto>(
    `/requirements${buildQueryString(filters)}`
  );
}

export function getRequirementDetail(
  id: string
): Promise<RequirementDetailDto> {
  return fetchJson<RequirementDetailDto>(`/requirements/${id}`);
}

export function getRequirementChain(
  id: string
): Promise<SdlcChainLink[]> {
  return fetchJson<SdlcChainLink[]>(`/requirements/${id}/chain`);
}

export function getRequirementAnalysis(
  id: string
): Promise<AiAnalysisDto> {
  return fetchJson<AiAnalysisDto>(`/requirements/${id}/analysis`);
}

export function generateStories(
  id: string,
  prompt?: string
): Promise<SkillExecutionResultDto> {
  return postJson<SkillExecutionResultDto>(
    `/requirements/${id}/generate-stories`,
    prompt ? { prompt } : undefined
  );
}

export function generateSpec(
  id: string,
  storyIds: readonly string[]
): Promise<SkillExecutionResultDto> {
  return postJson<SkillExecutionResultDto>(
    `/requirements/${id}/generate-spec`,
    { storyIds }
  );
}
```

GET endpoints use `fetchJson<T>`, POST endpoints use `postJson<T>` -- both from `shared/api/client.ts`.

### 9.2 Frontend Types

Create `features/requirement/types/requirement.ts`:

```typescript
// features/requirement/types/requirement.ts
import type { SectionResult } from '@/shared/types/section';

// --- List types ---

export interface RequirementSummary {
  readonly id: string;
  readonly title: string;
  readonly priority: 'CRITICAL' | 'HIGH' | 'MEDIUM' | 'LOW';
  readonly status: RequirementStatus;
  readonly category: 'FUNCTIONAL' | 'NON_FUNCTIONAL' | 'TECHNICAL' | 'BUSINESS';
  readonly storyCount: number;
  readonly specCount: number;
  readonly completenessScore: number;
  readonly assignee: string;
  readonly updatedAt: string;
  readonly createdAt: string;
}

export type RequirementStatus =
  | 'DRAFT'
  | 'IN_REVIEW'
  | 'APPROVED'
  | 'IN_PROGRESS'
  | 'DELIVERED'
  | 'ARCHIVED';

export interface StatusDistribution {
  readonly DRAFT: number;
  readonly IN_REVIEW: number;
  readonly APPROVED: number;
  readonly IN_PROGRESS: number;
  readonly DELIVERED: number;
  readonly ARCHIVED: number;
}

export interface RequirementListDto {
  readonly items: readonly RequirementSummary[];
  readonly statusDistribution: StatusDistribution;
  readonly totalCount: number;
}

// --- Detail types ---

export interface RequirementDetailDto {
  readonly header: SectionResult<RequirementHeader>;
  readonly description: SectionResult<RequirementDescription>;
  readonly stories: SectionResult<StorySectionDto>;
  readonly specs: SectionResult<SpecSectionDto>;
  readonly chain: SectionResult<ChainSectionDto>;
  readonly analysis: SectionResult<AiAnalysisDto>;
}

export interface RequirementHeader {
  readonly id: string;
  readonly title: string;
  readonly priority: 'CRITICAL' | 'HIGH' | 'MEDIUM' | 'LOW';
  readonly status: RequirementStatus;
  readonly category: string;
  readonly assignee: string;
  readonly source: 'MANUAL' | 'IMPORTED' | 'AI_GENERATED';
  readonly completenessScore: number;
  readonly createdAt: string;
  readonly updatedAt: string;
}

export interface RequirementDescription {
  readonly body: string;
  readonly businessContext: string;
  readonly businessValue: string;
  readonly stakeholders: readonly string[];
  readonly externalReferences: readonly ExternalReference[];
  readonly acceptanceCriteria: readonly AcceptanceCriterion[];
}

export interface ExternalReference {
  readonly system: string;
  readonly id: string;
  readonly url: string;
}

export interface AcceptanceCriterion {
  readonly id: string;
  readonly description: string;
  readonly status: 'PASSED' | 'FAILED' | 'NOT_TESTED';
}

// --- Story section ---

export interface StorySummary {
  readonly id: string;
  readonly title: string;
  readonly status: string;
  readonly specId: string | null;
  readonly specStatus: string | null;
}

export interface StorySectionDto {
  readonly items: readonly StorySummary[];
  readonly totalStories: number;
  readonly storiesWithSpec: number;
  readonly coveragePercent: number;
}

// --- Spec section ---

export interface SpecSummary {
  readonly id: string;
  readonly title: string;
  readonly status: string;
  readonly version: string;
  readonly linkedStoryIds: readonly string[];
  readonly updatedAt: string;
}

export interface SpecSectionDto {
  readonly items: readonly SpecSummary[];
  readonly totalSpecs: number;
  readonly approvedSpecs: number;
}

// --- SDLC Chain ---

export interface SdlcChainLink {
  readonly artifactType: string;
  readonly artifactId: string;
  readonly artifactTitle: string;
  readonly routePath: string;
  readonly status: string;
}

export interface ChainSectionDto {
  readonly links: readonly SdlcChainLink[];
}

// --- AI Analysis ---

export interface QualityBreakdown {
  readonly completeness: number;
  readonly clarity: number;
  readonly testability: number;
  readonly consistency: number;
}

export interface Suggestion {
  readonly type: 'AMBIGUITY' | 'GAP' | 'DUPLICATE' | 'INCONSISTENCY';
  readonly severity: 'HIGH' | 'MEDIUM' | 'LOW';
  readonly message: string;
  readonly field: string;
}

export interface DuplicateCheck {
  readonly hasPotentialDuplicates: boolean;
  readonly candidates: readonly string[];
}

export interface ImpactAnalysis {
  readonly downstreamArtifacts: number;
  readonly affectedTeams: readonly string[];
  readonly riskLevel: 'HIGH' | 'MEDIUM' | 'LOW';
}

export interface AiAnalysisDto {
  readonly qualityScore: number;
  readonly qualityBreakdown: QualityBreakdown;
  readonly suggestions: readonly Suggestion[];
  readonly duplicateCheck: DuplicateCheck;
  readonly impactAnalysis: ImpactAnalysis;
  readonly analyzedAt: string;
}

// --- Skill execution ---

export interface SkillExecutionResultDto {
  readonly executionId: string;
  readonly skillName: string;
  readonly status: 'RUNNING' | 'COMPLETED' | 'FAILED';
  readonly requirementId: string;
  readonly startedAt: string;
  readonly estimatedCompletionSeconds: number;
  readonly message: string;
}
```

### 9.3 Pinia Store

Create `features/requirement/stores/requirementStore.ts`:

```typescript
// features/requirement/stores/requirementStore.ts
import { defineStore } from 'pinia';
import { ref, readonly } from 'vue';
import type {
  RequirementListDto,
  RequirementDetailDto,
  SkillExecutionResultDto,
} from '../types/requirement';
import type { RequirementFilters } from '../api/requirementApi';
import * as requirementApi from '../api/requirementApi';
import { mockRequirementList, mockRequirementDetail } from '../mockData';

const useMock = import.meta.env.DEV && !import.meta.env.VITE_USE_BACKEND;

export const useRequirementStore = defineStore('requirement', () => {
  const list = ref<RequirementListDto | null>(null);
  const detail = ref<RequirementDetailDto | null>(null);
  const loading = ref(false);
  const error = ref<string | null>(null);

  async function fetchList(filters: RequirementFilters = {}): Promise<void> {
    loading.value = true;
    error.value = null;
    try {
      list.value = useMock
        ? mockRequirementList
        : await requirementApi.getRequirementList(filters);
    } catch (e: unknown) {
      error.value = e instanceof Error ? e.message : 'Failed to load requirements';
    } finally {
      loading.value = false;
    }
  }

  async function fetchDetail(id: string): Promise<void> {
    loading.value = true;
    error.value = null;
    try {
      detail.value = useMock
        ? mockRequirementDetail
        : await requirementApi.getRequirementDetail(id);
    } catch (e: unknown) {
      error.value = e instanceof Error ? e.message : 'Failed to load requirement detail';
    } finally {
      loading.value = false;
    }
  }

  async function triggerGenerateStories(
    id: string,
    prompt?: string
  ): Promise<SkillExecutionResultDto | null> {
    try {
      return await requirementApi.generateStories(id, prompt);
    } catch (e: unknown) {
      error.value = e instanceof Error ? e.message : 'Failed to generate stories';
      return null;
    }
  }

  async function triggerGenerateSpec(
    id: string,
    storyIds: readonly string[]
  ): Promise<SkillExecutionResultDto | null> {
    try {
      return await requirementApi.generateSpec(id, storyIds);
    } catch (e: unknown) {
      error.value = e instanceof Error ? e.message : 'Failed to generate spec';
      return null;
    }
  }

  return {
    list: readonly(list),
    detail: readonly(detail),
    loading: readonly(loading),
    error: readonly(error),
    fetchList,
    fetchDetail,
    triggerGenerateStories,
    triggerGenerateSpec,
  };
});
```

### 9.4 Vite Proxy

No proxy changes needed. The existing proxy in `vite.config.ts` already forwards
all `/api/*` traffic to `http://localhost:8080`, which covers `/api/v1/requirements/*`.

### 9.5 Mock Data Toggle

Store checks `import.meta.env.DEV && !import.meta.env.VITE_USE_BACKEND` to decide between
mock and live data. This matches the verified dashboard and incident store pattern.

---

## 10. State Reference

### Requirement Status

```
DRAFT -> IN_REVIEW -> APPROVED -> IN_PROGRESS -> DELIVERED

Override states: ARCHIVED (from any active state)
Rejection: IN_REVIEW -> DRAFT (with review feedback)
Archival: any active state -> ARCHIVED (authorized archival)
```

### Requirement Priority

```
CRITICAL | HIGH | MEDIUM | LOW
```

### Requirement Category

```
FUNCTIONAL | NON_FUNCTIONAL | TECHNICAL | BUSINESS
```

### Acceptance Criterion Status

```
NOT_TESTED | PASSED | FAILED
```

### Story Status (child objects)

```
DRAFT -> IN_PROGRESS -> DONE
```

### Spec Status (linked objects)

```
DRAFT -> REVIEW -> APPROVED -> IMPLEMENTED -> VERIFIED
```

### Skill Execution Status

```
RUNNING -> COMPLETED | FAILED
```

---

## 11. Testing Contracts

### Backend (MockMvc)

| Test | Method | Path | Expected |
|------|--------|------|----------|
| List returns 200 with items | GET | /api/v1/requirements | 200, body contains `items[]` and `statusDistribution` |
| List with priority filter | GET | /api/v1/requirements?priority=CRITICAL | 200, all items have priority CRITICAL |
| List with status filter | GET | /api/v1/requirements?status=APPROVED | 200, all items have status APPROVED |
| List with category filter | GET | /api/v1/requirements?category=FUNCTIONAL | 200, all items have category FUNCTIONAL |
| List with search | GET | /api/v1/requirements?search=workspace | 200, items match search term in title |
| Detail returns 200 with 6 sections | GET | /api/v1/requirements/REQ-001 | 200, body contains header, description, stories, specs, chain, analysis |
| Detail with invalid ID returns 404 | GET | /api/v1/requirements/REQ-9999 | 404, error = "Requirement not found: REQ-9999" |
| Chain returns 200 | GET | /api/v1/requirements/REQ-001/chain | 200, body contains array of chain links |
| Chain with invalid ID returns 404 | GET | /api/v1/requirements/REQ-9999/chain | 404 |
| Analysis returns 200 | GET | /api/v1/requirements/REQ-001/analysis | 200, body contains qualityScore |
| Analysis with invalid ID returns 404 | GET | /api/v1/requirements/REQ-9999/analysis | 404 |
| Generate stories returns 202 | POST | /api/v1/requirements/REQ-001/generate-stories | 202, body contains executionId and status = "RUNNING" |
| Generate stories with invalid ID returns 404 | POST | /api/v1/requirements/REQ-9999/generate-stories | 404 |
| Generate spec returns 202 | POST | /api/v1/requirements/REQ-001/generate-spec | 202, body contains executionId |
| Generate spec without storyIds returns 400 | POST | /api/v1/requirements/REQ-001/generate-spec | 400, body `{}` |
| Generate spec with invalid story returns 400 | POST | /api/v1/requirements/REQ-001/generate-spec | 400, body `{"storyIds":["S-999"]}` |

### Frontend (Vitest)

| Test | Scope | Expected |
|------|-------|----------|
| requirementApi.getRequirementList returns list | Unit | Calls fetchJson with /requirements, returns RequirementListDto |
| requirementApi.getRequirementList passes filters | Unit | Query string contains all provided filter params |
| requirementApi.getRequirementDetail returns detail | Unit | Calls fetchJson with /requirements/:id |
| requirementApi.generateStories calls postJson | Unit | Calls postJson with /requirements/:id/generate-stories |
| requirementApi.generateSpec validates storyIds | Unit | Calls postJson with body containing storyIds |
| requirementStore.fetchList sets list ref | Unit | After fetchList(), list.value is non-null |
| requirementStore.fetchList handles error | Unit | On API failure, error.value is set |
| requirementStore.fetchDetail sets detail ref | Unit | After fetchDetail(), detail.value has 6 sections |
| requirementStore uses mock data in dev mode | Unit | When VITE_USE_BACKEND is unset, returns mock data |

---

## 12. Versioning Policy

- All endpoints use the `/api/v1` prefix
- No breaking changes will be made without a version bump to `/api/v2`
- Additive changes (new optional fields, new endpoints) are allowed within v1
- Removed or renamed fields require a new version
- Deprecated fields will be documented and retained for at least one release cycle
