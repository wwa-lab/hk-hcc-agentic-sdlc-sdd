# Testing Management — API Implementation Guide

## Source

- Spec: [../../03-spec/testing-management-spec.md](../../03-spec/testing-management-spec.md)
- Architecture: [../../04-architecture/testing-management-architecture.md](../../04-architecture/testing-management-architecture.md)
- Data flow: [../../04-architecture/testing-management-data-flow.md](../../04-architecture/testing-management-data-flow.md)
- Data model: [../../04-architecture/testing-management-data-model.md](../../04-architecture/testing-management-data-model.md)
- Design: [../testing-management-design.md](../testing-management-design.md)
- Requirements: [../../01-requirements/testing-management-requirements.md](../../01-requirements/testing-management-requirements.md)

## 1. Conventions

- **Base path:** `/api/v1/testing`
- **Envelope:** every response wrapped in `ApiResponse<T>`:
  ```json
  { "data": { /* T */ }, "meta": { "correlationId": "tm-…", "generatedAt": "2026-04-17T10:00:00Z" }, "error": null }
  ```
- **Section results:** aggregate endpoints return `SectionResult<T>` per card: `{ "data": T | null, "error": { "code", "message" } | null, "loadedAt": "…" }`
- **Timestamps:** ISO-8601 UTC with trailing `Z`.
- **Pagination:** cursor-based (`?cursor=…&limit=…`, default 20, max 100).
- **Correlation ID:** client-supplied `x-correlation-id` header is honored; server generates if absent. Always echoed in `meta.correlationId`.
- **Workspace isolation:** all endpoints require bearer token + `X-Workspace-Id` header; requests without a visible workspace are rejected with `TM_WORKSPACE_FORBIDDEN`.
- **Webhook base path:** `/api/v1/testing/webhooks/ingest` (skipped by standard auth filter; signature-verified).

## 2. Endpoint Overview

### 2.1 Catalog

| Method | Path | Purpose | Auth |
| ------ | ---- | ------- | ---- |
| GET | `/catalog` | Aggregate (list plans + summary) | Read |
| GET | `/catalog/summary` | Summary card only | Read |
| GET | `/catalog/grid` | Plan grid card only | Read |

Query params for `/catalog*`: `projectId`, `planState` (DRAFT/ACTIVE/ARCHIVED), `coverageLed` (GREEN/AMBER/RED/GREY), `releaseTarget`, `search`.

### 2.2 Plan Detail

| Method | Path | Purpose | Auth |
| ------ | ---- | ------- | ---- |
| GET | `/plans/{planId}` | 6-card aggregate | Read |
| GET | `/plans/{planId}/header` | Header card only | Read |
| GET | `/plans/{planId}/cases` | Cases card only | Read |
| GET | `/plans/{planId}/coverage` | Coverage card only | Read |
| GET | `/plans/{planId}/recent-runs` | Recent Runs card only | Read |
| GET | `/plans/{planId}/draft-inbox` | AI Draft Inbox card only | Read |
| GET | `/plans/{planId}/ai-insights` | AI Insights card only | Read |

### 2.3 Plan CRUD

| Method | Path | Purpose | Auth |
| ------ | ---- | ------- | ---- |
| GET | `/plans` | List all plans in workspace | Read |
| POST | `/plans` | Create new plan | QA Lead / PM |
| GET | `/plans/{planId}` | (see above aggregate) | Read |
| PATCH | `/plans/{planId}` | Update plan metadata | QA Lead / PM / Owner |
| DELETE | `/plans/{planId}` | Soft delete (→ ARCHIVED) | QA Lead / PM / Owner |

### 2.4 Case CRUD & State

| Method | Path | Purpose | Auth |
| ------ | ---- | ------- | ---- |
| POST | `/cases` | Create new test case | QA Lead / Tech Lead |
| GET | `/cases/{caseId}` | Case detail + revisions | Read |
| PATCH | `/cases/{caseId}` | Update case content | QA Lead / Tech Lead / Owner |
| POST | `/cases/{caseId}/state-transitions` | Approve DRAFT→ACTIVE or ACTIVE→DEPRECATED | QA Lead / Tech Lead |
| GET | `/cases/{caseId}/revisions` | Revision history | Read |

### 2.5 Run Ingestion & Detail

| Method | Path | Purpose | Auth |
| ------ | ---- | ------- | ---- |
| GET | `/runs` | List recent runs (workspace-scoped) | Read |
| GET | `/runs/{runId}` | Run detail + case outcomes | Read |
| POST | `/runs/ingest` | Upload JUnit/TestNG/Playwright/Cypress | QA Lead |
| POST | `/runs/webhook` | Signed CI webhook ingest | (signature-verified) |

### 2.6 Traceability & REQ Links

| Method | Path | Purpose | Auth |
| ------ | ---- | ------- | ---- |
| GET | `/traceability?reqId=REQ-...` | Inverse: REQ → cases → plans → runs | Read |
| GET | `/cases/{caseId}/req-links` | REQ-IDs linked to this case | Read |
| POST | `/cases/{caseId}/req-links` | Add REQ-ID link | QA Lead / Tech Lead / Owner |
| DELETE | `/cases/{caseId}/req-links/{linkId}` | Remove REQ-ID link | QA Lead / Tech Lead / Owner |

### 2.7 AI Test Case Generation

| Method | Path | Purpose | Auth |
| ------ | ---- | ------- | ---- |
| POST | `/ai/draft-cases` | Generate candidate cases from REQ-ID | QA Lead / Tech Lead |
| POST | `/cases/{caseId}/approve` | Approve DRAFT → ACTIVE | QA Lead / Tech Lead |
| POST | `/cases/{caseId}/reject` | Reject DRAFT → DEPRECATED | QA Lead / Tech Lead |

### 2.8 Environment Registry

| Method | Path | Purpose | Auth |
| ------ | ---- | ------- | ---- |
| GET | `/environments` | List workspace environments | Read |
| POST | `/environments` | Create environment | QA Lead / PM |
| GET | `/environments/{envId}` | Environment detail | Read |
| PATCH | `/environments/{envId}` | Update environment metadata | QA Lead / PM |

## 3. Example Payloads

### 3.1 `GET /catalog` (200)

```json
{
  "data": {
    "filtersEcho": { "projectId": "proj-billing", "planState": "ACTIVE", "releaseTarget": "v2.1", "search": null },
    "summary": {
      "data": {
        "totalPlans": 24,
        "totalActiveCases": 187,
        "runsLast7d": 156,
        "passRateLast7d": 0.91,
        "meanRunDurationSec": 342,
        "byCoverageLed": { "GREEN": 18, "AMBER": 4, "RED": 1, "GREY": 1 }
      },
      "error": null,
      "loadedAt": "2026-04-17T10:00:01Z"
    },
    "grid": {
      "data": [
        {
          "planId": "plan-001",
          "projectId": "proj-billing",
          "projectName": "Billing Platform",
          "name": "Payment Processing (Q2)",
          "releaseTarget": "v2.1",
          "owner": "qa-lead@acme.example",
          "state": "ACTIVE",
          "linkedCaseCount": 42,
          "coverageLed": "GREEN",
          "description": "Full QA coverage for payment processing module"
        }
      ],
      "error": null,
      "loadedAt": "2026-04-17T10:00:01Z"
    }
  },
  "meta": { "correlationId": "tm-abc123", "generatedAt": "2026-04-17T10:00:01Z" },
  "error": null
}
```

### 3.2 `POST /plans` (201)

**Request:**
```json
{
  "projectId": "proj-billing",
  "name": "Payment Processing (Q2)",
  "description": "Full QA coverage for payment processing module",
  "releaseTarget": "v2.1",
  "owner": "qa-lead@acme.example"
}
```

**Response (201 Created):**
```json
{
  "data": {
    "planId": "plan-001",
    "projectId": "proj-billing",
    "name": "Payment Processing (Q2)",
    "description": "Full QA coverage for payment processing module",
    "releaseTarget": "v2.1",
    "owner": "qa-lead@acme.example",
    "state": "DRAFT",
    "createdAt": "2026-04-17T10:00:05Z",
    "createdBy": "kim@acme.example",
    "updatedAt": "2026-04-17T10:00:05Z",
    "updatedBy": "kim@acme.example"
  },
  "meta": { "correlationId": "tm-post-001", "generatedAt": "2026-04-17T10:00:05Z" },
  "error": null
}
```

### 3.3 `GET /plans/{planId}` (200) — Full Plan Detail Aggregate

```json
{
  "data": {
    "header": {
      "data": {
        "planId": "plan-001",
        "projectId": "proj-billing",
        "name": "Payment Processing (Q2)",
        "description": "Full QA coverage for payment processing module",
        "releaseTarget": "v2.1",
        "owner": "qa-lead@acme.example",
        "state": "ACTIVE",
        "createdAt": "2026-04-15T09:00:00Z",
        "createdBy": "qa-lead@acme.example",
        "updatedAt": "2026-04-17T09:30:00Z",
        "updatedBy": "kim@acme.example"
      },
      "error": null,
      "loadedAt": "2026-04-17T10:00:02Z"
    },
    "cases": {
      "data": [
        {
          "caseId": "case-0001",
          "planId": "plan-001",
          "title": "User initiates payment via card",
          "type": "FUNCTIONAL",
          "priority": "P0",
          "state": "ACTIVE",
          "linkedReqs": [
            { "reqId": "REQ-BILLING-001", "status": "VERIFIED", "title": "Payment initiation" }
          ],
          "linkedDefects": [
            { "incidentId": "INC-2401", "title": "Payment timeout on 3DS", "severity": "P1" }
          ],
          "lastRunStatus": "PASSED",
          "lastRunAt": "2026-04-17T09:45:00Z"
        }
      ],
      "error": null,
      "loadedAt": "2026-04-17T10:00:02Z"
    },
    "coverage": {
      "data": [
        {
          "reqId": "REQ-BILLING-001",
          "reqTitle": "Payment initiation",
          "linkedCaseCount": 3,
          "mostRecentStatus": "PASSED",
          "mostRecentAt": "2026-04-17T09:45:00Z"
        }
      ],
      "error": null,
      "loadedAt": "2026-04-17T10:00:02Z"
    },
    "recentRuns": {
      "data": [
        {
          "runId": "run-001",
          "planId": "plan-001",
          "environment": "staging",
          "trigger": "MANUAL",
          "actor": "kim@acme.example",
          "state": "PASSED",
          "duration": 342,
          "passCount": 38,
          "failCount": 0,
          "skipCount": 4,
          "createdAt": "2026-04-17T09:45:00Z"
        }
      ],
      "error": null,
      "loadedAt": "2026-04-17T10:00:02Z"
    },
    "draftInbox": {
      "data": [
        {
          "caseId": "case-draft-001",
          "planId": "plan-001",
          "sourceReqId": "REQ-BILLING-002",
          "title": "Partial refund workflow (AI-drafted)",
          "preconditions": "User logged in, transaction completed",
          "steps": "1. Navigate to order details\n2. Click Refund\n3. Verify confirmation",
          "expected": "Refund completes within 30s",
          "type": "FUNCTIONAL",
          "priority": "P1",
          "state": "DRAFT",
          "origin": "AI_DRAFT",
          "skillVersion": "test-case-drafter-v1.2",
          "draftedAt": "2026-04-17T08:15:00Z",
          "citedExcerpt": "Refunds shall process within 30 seconds via automated workflow"
        }
      ],
      "error": null,
      "loadedAt": "2026-04-17T10:00:02Z"
    },
    "aiInsights": {
      "data": {
        "status": "SUCCESS",
        "generatedAt": "2026-04-17T09:55:00Z",
        "skillVersion": "tm-insights-v2",
        "narrative": "In the last 7 days, payment plan tests show 91% pass rate. REQ-BILLING-001 has full coverage with 3 active cases, all passed. REQ-BILLING-003 (3DS challenges) lost coverage on 2026-04-16 when the staging environment became unavailable.",
        "evidence": [
          { "kind": "plan", "id": "plan-001", "label": "Payment Processing" },
          { "kind": "req", "id": "REQ-BILLING-003", "label": "3DS challenge handling" }
        ]
      },
      "error": null,
      "loadedAt": "2026-04-17T10:00:02Z"
    }
  },
  "meta": { "correlationId": "tm-detail-001", "generatedAt": "2026-04-17T10:00:02Z" },
  "error": null
}
```

### 3.4 `POST /cases` (201) — Create Test Case

**Request:**
```json
{
  "planId": "plan-001",
  "title": "User initiates payment via card",
  "type": "FUNCTIONAL",
  "priority": "P0",
  "preconditions": "User logged in, cart populated",
  "steps": "1. Click Checkout\n2. Select Card payment\n3. Enter card details\n4. Click Pay",
  "expected": "Payment form submitted successfully",
  "linkedReqIds": ["REQ-BILLING-001"]
}
```

**Response (201 Created):**
```json
{
  "data": {
    "caseId": "case-0001",
    "planId": "plan-001",
    "title": "User initiates payment via card",
    "type": "FUNCTIONAL",
    "priority": "P0",
    "state": "ACTIVE",
    "preconditions": "User logged in, cart populated",
    "steps": "1. Click Checkout\n2. Select Card payment\n3. Enter card details\n4. Click Pay",
    "expected": "Payment form submitted successfully",
    "linkedReqs": [
      { "reqId": "REQ-BILLING-001", "status": "VERIFIED", "title": "Payment initiation" }
    ],
    "owner": "qa-lead@acme.example",
    "createdAt": "2026-04-17T10:00:10Z",
    "createdBy": "kim@acme.example"
  },
  "meta": { "correlationId": "tm-case-create", "generatedAt": "2026-04-17T10:00:10Z" },
  "error": null
}
```

### 3.5 `POST /ai/draft-cases` (200) — AI Draft Generation

**Request:**
```json
{
  "planId": "plan-001",
  "reqId": "REQ-BILLING-002",
  "skillVersion": "test-case-drafter-v1.2"
}
```

**Response (200):**
```json
{
  "data": {
    "reqId": "REQ-BILLING-002",
    "reqTitle": "Partial refund workflow",
    "citedExcerpt": "Refunds shall process within 30 seconds via automated workflow",
    "candidates": [
      {
        "caseId": "case-draft-001",
        "title": "Partial refund workflow (AI-drafted)",
        "preconditions": "User logged in, transaction completed",
        "steps": "1. Navigate to order details\n2. Click Refund\n3. Verify confirmation",
        "expected": "Refund completes within 30s",
        "type": "FUNCTIONAL",
        "priority": "P1",
        "state": "DRAFT",
        "origin": "AI_DRAFT",
        "skillVersion": "test-case-drafter-v1.2",
        "draftedAt": "2026-04-17T08:15:00Z"
      }
    ],
    "quotaRemaining": 19
  },
  "meta": { "correlationId": "tm-ai-draft", "generatedAt": "2026-04-17T08:15:00Z" },
  "error": null
}
```

### 3.6 `POST /cases/{caseId}/state-transitions` (200) — Approve DRAFT→ACTIVE

**Request:**
```json
{
  "targetState": "ACTIVE",
  "comment": "Reviewed and aligned with REQ-BILLING-002"
}
```

**Response (200):**
```json
{
  "data": {
    "caseId": "case-draft-001",
    "state": "ACTIVE",
    "updatedAt": "2026-04-17T08:20:00Z",
    "updatedBy": "qa-lead@acme.example"
  },
  "meta": { "correlationId": "tm-state-transition", "generatedAt": "2026-04-17T08:20:00Z" },
  "error": null
}
```

### 3.7 `POST /runs/ingest` (202) — Manual Run Upload

**Request (multipart/form-data):**
- Field `metadata`: `{ "planId": "plan-001", "environment": "staging", "externalRunId": "gha-run-99012" }`
- Field `file`: `results.xml` (JUnit format)

**Response (202 Accepted):**
```json
{
  "data": {
    "runId": "run-001",
    "planId": "plan-001",
    "environment": "staging",
    "externalRunId": "gha-run-99012",
    "trigger": "MANUAL",
    "actor": "kim@acme.example",
    "state": "RUNNING",
    "ingestStartedAt": "2026-04-17T10:05:00Z"
  },
  "meta": { "correlationId": "tm-ingest-001", "generatedAt": "2026-04-17T10:05:00Z" },
  "error": null
}
```

### 3.8 `GET /runs/{runId}` (200) — Run Detail with Case Outcomes

```json
{
  "data": {
    "runId": "run-001",
    "planId": "plan-001",
    "environment": "staging",
    "externalRunId": "gha-run-99012",
    "trigger": "MANUAL",
    "actor": "kim@acme.example",
    "state": "PASSED",
    "duration": 342,
    "passCount": 38,
    "failCount": 0,
    "skipCount": 4,
    "createdAt": "2026-04-17T09:45:00Z",
    "completedAt": "2026-04-17T10:10:42Z",
    "caseOutcomes": [
      {
        "caseId": "case-0001",
        "status": "PASSED",
        "duration": 4,
        "passedAt": "2026-04-17T09:45:15Z"
      },
      {
        "caseId": "case-0002",
        "status": "FAILED",
        "duration": 8,
        "failedAt": "2026-04-17T09:46:20Z",
        "failureMessage": "AssertionError: expected 'Success' but got 'Timeout'",
        "failureExcerpt": "[ERROR] Payment timeout after 30s",
        "lastPassedAt": "2026-04-15T10:30:00Z",
        "linkedDefects": [
          { "incidentId": "INC-2401", "title": "Payment timeout on 3DS", "severity": "P1" }
        ]
      }
    ],
    "environmentMetadata": {
      "name": "staging",
      "description": "Staging environment for testing",
      "kind": "STAGING",
      "url": "https://staging.acme.example"
    },
    "coveredReqs": [
      { "reqId": "REQ-BILLING-001", "title": "Payment initiation" },
      { "reqId": "REQ-BILLING-003", "title": "3DS challenge handling" }
    ]
  },
  "meta": { "correlationId": "tm-run-detail", "generatedAt": "2026-04-17T10:10:43Z" },
  "error": null
}
```

### 3.9 `GET /traceability?reqId=REQ-BILLING-001` (200) — Inverse Lookup

```json
{
  "data": {
    "reqChip": {
      "reqId": "REQ-BILLING-001",
      "status": "VERIFIED",
      "title": "Payment initiation",
      "projectId": "proj-billing"
    },
    "cases": {
      "data": [
        {
          "caseId": "case-0001",
          "planId": "plan-001",
          "title": "User initiates payment via card",
          "state": "ACTIVE",
          "lastRunStatus": "PASSED",
          "lastRunAt": "2026-04-17T09:45:00Z"
        }
      ],
      "error": null,
      "loadedAt": "2026-04-17T10:00:03Z"
    },
    "plans": {
      "data": [
        {
          "planId": "plan-001",
          "name": "Payment Processing (Q2)",
          "releaseTarget": "v2.1",
          "coverageLed": "GREEN"
        }
      ],
      "error": null,
      "loadedAt": "2026-04-17T10:00:03Z"
    }
  },
  "meta": { "correlationId": "tm-trace-req-001", "generatedAt": "2026-04-17T10:00:03Z" },
  "error": null
}
```

### 3.10 `GET /environments` (200) — Environment List

```json
{
  "data": [
    {
      "envId": "env-001",
      "name": "staging",
      "description": "Staging environment for testing",
      "kind": "STAGING",
      "url": "https://staging.acme.example",
      "archived": false,
      "createdAt": "2026-04-01T08:00:00Z"
    },
    {
      "envId": "env-002",
      "name": "prod-shadow",
      "description": "Production-like shadow for load testing",
      "kind": "OTHER",
      "url": null,
      "archived": false,
      "createdAt": "2026-04-05T10:30:00Z"
    }
  ],
  "meta": { "correlationId": "tm-envs", "generatedAt": "2026-04-17T10:00:04Z" },
  "error": null
}
```

### 3.11 Error Response Example

**GET /plans/plan-invalid (404)**
```json
{
  "data": null,
  "meta": { "correlationId": "tm-err-404", "generatedAt": "2026-04-17T10:00:05Z" },
  "error": { "code": "TM_PLAN_NOT_FOUND", "message": "Plan 'plan-invalid' not found or not accessible in this workspace" }
}
```

**POST /ai/draft-cases (429) — Rate Limited**
```json
{
  "data": null,
  "meta": { "correlationId": "tm-err-429", "generatedAt": "2026-04-17T10:05:00Z" },
  "error": { "code": "TM_RATE_LIMITED", "message": "Draft quota exceeded: 20 drafts per REQ per day" },
  "rateLimitRemaining": 0,
  "rateLimitReset": "2026-04-18T00:00:00Z"
}
```

## 4. Error Code Catalog

| Code | HTTP | Retryable | Notes |
| ---- | ---- | --------- | ----- |
| `TM_WORKSPACE_FORBIDDEN` | 403 | No | Caller not a workspace member or header missing |
| `TM_ROLE_REQUIRED` | 403 | No | QA Lead / Tech Lead / PM role required for this action |
| `TM_PLAN_NOT_FOUND` | 404 | No | `planId` unknown or out of scope |
| `TM_CASE_NOT_FOUND` | 404 | No | `caseId` unknown or out of scope |
| `TM_RUN_NOT_FOUND` | 404 | No | `runId` unknown or out of scope |
| `TM_ENV_NOT_FOUND` | 404 | No | Environment not found |
| `TM_REQ_NOT_FOUND` | 404 | No | REQ-ID not found in Requirement slice |
| `TM_REQ_UNVERIFIED` | 422 | No | REQ-ID persisted but not yet verified (unknown/not visible) |
| `TM_DUPLICATE_RUN_INGESTION` | 409 | No | `externalRunId` already ingested; pass `force=true` to override (admin-only) |
| `TM_INGEST_PARSE_FAILED` | 422 | No | JUnit/TestNG/Playwright/Cypress parser error; see `parseError` detail |
| `TM_INGEST_FORMAT_UNSUPPORTED` | 400 | No | File format not recognized (not JUnit/TestNG/Playwright/Cypress) |
| `TM_ENV_ARCHIVED` | 409 | No | Environment is archived; cannot ingest new runs |
| `TM_RUN_IMMUTABLE` | 409 | No | Run already ingested; cannot modify outcomes |
| `TM_STATE_TRANSITION_INVALID` | 422 | No | Invalid state transition (e.g., DEPRECATED→ACTIVE disallowed) |
| `TM_CASE_IN_ACTIVE_RUN` | 409 | No | Cannot delete/archive case; it appears in active runs |
| `TM_AI_AUTONOMY_INSUFFICIENT` | 403 | No | Workspace `aiAutonomyLevel=DISABLED` or `OBSERVATION` disallows action |
| `TM_AI_UNAVAILABLE` | 503 | Yes | AI skill client failure (transient) |
| `TM_RATE_LIMITED` | 429 | Yes | Draft quota exceeded or generic rate limit; includes `Retry-After` |
| `TM_WEBHOOK_SIGNATURE_INVALID` | 401 | No | HMAC-SHA256 signature verification failed |
| `TM_WEBHOOK_PAYLOAD_INVALID` | 400 | No | Webhook payload parse or schema validation failed |
| `TM_SECRET_REDACTION_FAILED` | 500 | Yes | Redactor crashed; run marked `INGEST_FAILED` for manual retry |

## 5. Backend Controller Skeleton (Spring Boot)

```java
package com.agentic.sdlc.testing.api;

import org.springframework.web.bind.annotation.*;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.validation.annotation.Validated;
import java.security.Principal;
import java.util.*;

@RestController
@RequestMapping("/api/v1/testing")
@Validated
public class TestingController {

    private final CatalogService catalogService;
    private final PlanService planService;
    private final CaseService caseService;
    private final RunService runService;
    private final TraceabilityService traceabilityService;
    private final EnvironmentService environmentService;
    private final AiDraftService aiDraftService;
    private final TestingAccessGuard accessGuard;

    // ---- Catalog ----
    @GetMapping("/catalog")
    public ApiResponse<CatalogAggregateDto> getCatalog(
            @RequestParam(required = false) String projectId,
            @RequestParam(required = false) String planState,
            @RequestParam(required = false) String coverageLed,
            @RequestParam(required = false) String releaseTarget,
            @RequestParam(required = false) String search,
            @RequestHeader("X-Workspace-Id") String workspaceId,
            Principal principal) {
        accessGuard.requireWorkspaceMember(workspaceId, principal);
        var filters = CatalogFilters.of(projectId, planState, coverageLed, releaseTarget, search);
        return ApiResponse.ok(catalogService.loadAggregate(workspaceId, filters, principal));
    }

    // ---- Plan CRUD ----
    @GetMapping("/plans")
    public ApiResponse<List<PlanRowDto>> listPlans(
            @RequestHeader("X-Workspace-Id") String workspaceId,
            @RequestParam(defaultValue = "0") String cursor,
            @RequestParam(defaultValue = "20") int limit,
            Principal principal) {
        accessGuard.requireWorkspaceMember(workspaceId, principal);
        return ApiResponse.ok(planService.list(workspaceId, cursor, limit, principal));
    }

    @PostMapping("/plans")
    public ResponseEntity<ApiResponse<PlanDto>> createPlan(
            @RequestBody CreatePlanRequest req,
            @RequestHeader("X-Workspace-Id") String workspaceId,
            Principal principal) {
        accessGuard.requireRole(workspaceId, principal, Role.QA_LEAD, Role.PM);
        var plan = planService.create(workspaceId, req, principal);
        return ResponseEntity.status(HttpStatus.CREATED).body(ApiResponse.ok(plan));
    }

    @GetMapping("/plans/{planId}")
    public ApiResponse<PlanDetailAggregateDto> getPlan(
            @PathVariable String planId,
            @RequestHeader("X-Workspace-Id") String workspaceId,
            Principal principal) {
        accessGuard.requireWorkspaceMember(workspaceId, principal);
        return ApiResponse.ok(planService.loadAggregate(planId, workspaceId, principal));
    }

    @PatchMapping("/plans/{planId}")
    public ApiResponse<PlanDto> updatePlan(
            @PathVariable String planId,
            @RequestBody UpdatePlanRequest req,
            @RequestHeader("X-Workspace-Id") String workspaceId,
            Principal principal) {
        accessGuard.requirePlanOwnerOrLead(planId, workspaceId, principal);
        return ApiResponse.ok(planService.update(planId, req, principal));
    }

    @DeleteMapping("/plans/{planId}")
    public ResponseEntity<Void> deletePlan(
            @PathVariable String planId,
            @RequestHeader("X-Workspace-Id") String workspaceId,
            Principal principal) {
        accessGuard.requirePlanOwnerOrLead(planId, workspaceId, principal);
        planService.softDelete(planId, principal);
        return ResponseEntity.noContent().build();
    }

    // ---- Case CRUD & State ----
    @PostMapping("/cases")
    public ResponseEntity<ApiResponse<CaseDto>> createCase(
            @RequestBody CreateCaseRequest req,
            @RequestHeader("X-Workspace-Id") String workspaceId,
            Principal principal) {
        accessGuard.requireRole(workspaceId, principal, Role.QA_LEAD, Role.TECH_LEAD);
        var testCase = caseService.create(workspaceId, req, principal);
        return ResponseEntity.status(HttpStatus.CREATED).body(ApiResponse.ok(testCase));
    }

    @GetMapping("/cases/{caseId}")
    public ApiResponse<CaseDetailDto> getCase(
            @PathVariable String caseId,
            @RequestHeader("X-Workspace-Id") String workspaceId,
            Principal principal) {
        accessGuard.requireWorkspaceMember(workspaceId, principal);
        return ApiResponse.ok(caseService.getDetail(caseId, workspaceId, principal));
    }

    @PatchMapping("/cases/{caseId}")
    public ApiResponse<CaseDto> updateCase(
            @PathVariable String caseId,
            @RequestBody UpdateCaseRequest req,
            @RequestHeader("X-Workspace-Id") String workspaceId,
            Principal principal) {
        accessGuard.requireRole(workspaceId, principal, Role.QA_LEAD, Role.TECH_LEAD);
        return ApiResponse.ok(caseService.update(caseId, req, workspaceId, principal));
    }

    @PostMapping("/cases/{caseId}/state-transitions")
    public ApiResponse<CaseDto> transitionState(
            @PathVariable String caseId,
            @RequestBody StateTransitionRequest req,
            @RequestHeader("X-Workspace-Id") String workspaceId,
            Principal principal) {
        accessGuard.requireRole(workspaceId, principal, Role.QA_LEAD, Role.TECH_LEAD);
        return ApiResponse.ok(caseService.transitionState(caseId, req, workspaceId, principal));
    }

    @GetMapping("/cases/{caseId}/revisions")
    public ApiResponse<List<RevisionRowDto>> getCaseRevisions(
            @PathVariable String caseId,
            @RequestHeader("X-Workspace-Id") String workspaceId,
            Principal principal) {
        accessGuard.requireWorkspaceMember(workspaceId, principal);
        return ApiResponse.ok(caseService.getRevisions(caseId, workspaceId));
    }

    // ---- Run Ingestion & Detail ----
    @GetMapping("/runs")
    public ApiResponse<List<RunRowDto>> listRuns(
            @RequestHeader("X-Workspace-Id") String workspaceId,
            @RequestParam(defaultValue = "0") String cursor,
            @RequestParam(defaultValue = "20") int limit,
            Principal principal) {
        accessGuard.requireWorkspaceMember(workspaceId, principal);
        return ApiResponse.ok(runService.listRecent(workspaceId, cursor, limit));
    }

    @GetMapping("/runs/{runId}")
    public ApiResponse<RunDetailDto> getRun(
            @PathVariable String runId,
            @RequestHeader("X-Workspace-Id") String workspaceId,
            Principal principal) {
        accessGuard.requireWorkspaceMember(workspaceId, principal);
        return ApiResponse.ok(runService.getDetail(runId, workspaceId));
    }

    @PostMapping("/runs/ingest")
    public ResponseEntity<ApiResponse<RunDto>> ingestRun(
            @RequestParam String planId,
            @RequestParam String environment,
            @RequestParam String externalRunId,
            @RequestParam(required = false, defaultValue = "false") boolean force,
            @RequestPart("metadata") IngestMetadataDto metadata,
            @RequestPart("file") org.springframework.web.multipart.MultipartFile file,
            @RequestHeader("X-Workspace-Id") String workspaceId,
            Principal principal) {
        accessGuard.requireRole(workspaceId, principal, Role.QA_LEAD);
        var run = runService.ingestRun(planId, environment, externalRunId, force, file, metadata, workspaceId, principal);
        return ResponseEntity.status(HttpStatus.ACCEPTED).body(ApiResponse.ok(run));
    }

    // ---- AI Draft ----
    @PostMapping("/ai/draft-cases")
    public ApiResponse<AiDraftResponseDto> draftCases(
            @RequestBody AiDraftRequestDto req,
            @RequestHeader("X-Workspace-Id") String workspaceId,
            Principal principal) {
        accessGuard.requireRole(workspaceId, principal, Role.QA_LEAD, Role.TECH_LEAD);
        return ApiResponse.ok(aiDraftService.draft(req.planId(), req.reqId(), req.skillVersion(), workspaceId, principal));
    }

    // ---- Traceability ----
    @GetMapping("/traceability")
    public ApiResponse<TraceabilityAggregateDto> getTraceability(
            @RequestParam String reqId,
            @RequestHeader("X-Workspace-Id") String workspaceId,
            Principal principal) {
        accessGuard.requireWorkspaceMember(workspaceId, principal);
        return ApiResponse.ok(traceabilityService.inverseLookup(reqId, workspaceId, principal));
    }

    // ---- Environment Registry ----
    @GetMapping("/environments")
    public ApiResponse<List<EnvironmentDto>> listEnvironments(
            @RequestHeader("X-Workspace-Id") String workspaceId,
            Principal principal) {
        accessGuard.requireWorkspaceMember(workspaceId, principal);
        return ApiResponse.ok(environmentService.list(workspaceId));
    }

    @PostMapping("/environments")
    public ResponseEntity<ApiResponse<EnvironmentDto>> createEnvironment(
            @RequestBody CreateEnvironmentRequest req,
            @RequestHeader("X-Workspace-Id") String workspaceId,
            Principal principal) {
        accessGuard.requireRole(workspaceId, principal, Role.QA_LEAD, Role.PM);
        var env = environmentService.create(workspaceId, req, principal);
        return ResponseEntity.status(HttpStatus.CREATED).body(ApiResponse.ok(env));
    }

    @PatchMapping("/environments/{envId}")
    public ApiResponse<EnvironmentDto> updateEnvironment(
            @PathVariable String envId,
            @RequestBody UpdateEnvironmentRequest req,
            @RequestHeader("X-Workspace-Id") String workspaceId,
            Principal principal) {
        accessGuard.requireRole(workspaceId, principal, Role.QA_LEAD, Role.PM);
        return ApiResponse.ok(environmentService.update(envId, req, workspaceId, principal));
    }
}
```

### 5.1 Webhook Controller

```java
@RestController
@RequestMapping("/api/v1/testing/webhooks")
public class TestingWebhookController {

    private final WebhookSignatureVerifier verifier;
    private final RunIngestionProcessor processor;
    private final AuditLogEmitter audit;

    @PostMapping(path = "/ingest", consumes = "application/json")
    public ResponseEntity<Void> receiveRunIngest(
            @RequestHeader("X-Webhook-Signature") String signature,
            @RequestHeader(value = "X-Delivery-Id", required = false) String deliveryId,
            @RequestBody byte[] body) {

        if (!verifier.verify(signature, body)) {
            audit.emit("TM_WEBHOOK_SIGNATURE_INVALID", Map.of("deliveryId", String.valueOf(deliveryId)));
            return ResponseEntity.status(HttpStatus.UNAUTHORIZED).build();
        }

        try {
            processor.processAsync(body, deliveryId);
            return ResponseEntity.accepted().build();
        } catch (InvalidPayloadException e) {
            audit.emit("TM_WEBHOOK_PAYLOAD_INVALID", Map.of("deliveryId", String.valueOf(deliveryId)));
            return ResponseEntity.badRequest().build();
        }
    }
}
```

### 5.2 RunResultParser

```java
public class RunResultParser {
    private static final Map<String, Parser> PARSERS = Map.of(
        "junit", new JunitXmlParser(),
        "testng", new TestngXmlParser(),
        "playwright", new PlaywrightJsonParser(),
        "cypress", new CypressJsonParser()
    );

    public RunResultsDto parse(String format, byte[] payload) throws ParseException {
        var parser = PARSERS.get(format.toLowerCase());
        if (parser == null) throw new ParseException("Unsupported format: " + format);
        return parser.parse(payload);
    }
}
```

### 5.3 SecretRedactor

```java
public class SecretRedactor {
    private static final List<Pattern> PATTERNS = List.of(
        Pattern.compile("AKIA[0-9A-Z]{16}"),  // AWS access key
        Pattern.compile("ghp_[A-Za-z0-9]{36,}"),  // GitHub personal token
        Pattern.compile("gho_[A-Za-z0-9]{36,}"),  // GitHub OAuth token
        Pattern.compile("ghs_[A-Za-z0-9]{36,}"),  // GitHub app token
        Pattern.compile("(?i)bearer\\s+[A-Za-z0-9._\\-]+")  // generic bearer
    );

    public String redact(String raw) {
        String out = raw;
        for (var p : PATTERNS) {
            out = p.matcher(out).replaceAll("REDACTED");
        }
        return out;
    }
}
```

### 5.4 CaseRevisionRecorder

```java
public class CaseRevisionRecorder {
    public CaseRevisionDto recordUpdate(TestCase old, TestCase updated, String actor) {
        var diff = FieldDiffer.diff(old, updated);
        return new CaseRevisionDto(
            UUID.randomUUID().toString(),
            old.id(),
            actor,
            Instant.now(),
            diff.changedFields()
        );
    }
}
```

## 6. Frontend Integration Guide

### 6.1 API Client Module

`frontend/src/features/testing-management/api/testingApi.ts`

```ts
import { fetchJson } from '@/shared/api/client';
import type { ApiResponse } from '@/shared/api/types';
import type {
  CatalogAggregate, PlanDetailAggregate, CaseDetailDto, RunDetailDto,
  TraceabilityAggregate, AiDraftResponseDto, EnvironmentDto
} from '../types/dto';

const BASE = '/api/v1/testing';

export const testingApi = {
  // Catalog
  getCatalog: (filters: CatalogFilters) =>
    fetchJson<ApiResponse<CatalogAggregate>>(`${BASE}/catalog?${qs(filters)}`),

  // Plans
  listPlans: (cursor?: string, limit = 20) =>
    fetchJson<ApiResponse<PlanRowDto[]>>(`${BASE}/plans?cursor=${cursor||''}&limit=${limit}`),
  createPlan: (req: CreatePlanRequest) =>
    fetchJson<ApiResponse<PlanDto>>(`${BASE}/plans`, { method: 'POST', body: JSON.stringify(req) }),
  getPlan: (planId: string) =>
    fetchJson<ApiResponse<PlanDetailAggregate>>(`${BASE}/plans/${planId}`),
  updatePlan: (planId: string, req: UpdatePlanRequest) =>
    fetchJson<ApiResponse<PlanDto>>(`${BASE}/plans/${planId}`, { method: 'PATCH', body: JSON.stringify(req) }),
  deletePlan: (planId: string) =>
    fetchJson<ApiResponse<void>>(`${BASE}/plans/${planId}`, { method: 'DELETE' }),

  // Cases
  createCase: (req: CreateCaseRequest) =>
    fetchJson<ApiResponse<CaseDto>>(`${BASE}/cases`, { method: 'POST', body: JSON.stringify(req) }),
  getCase: (caseId: string) =>
    fetchJson<ApiResponse<CaseDetailDto>>(`${BASE}/cases/${caseId}`),
  updateCase: (caseId: string, req: UpdateCaseRequest) =>
    fetchJson<ApiResponse<CaseDto>>(`${BASE}/cases/${caseId}`, { method: 'PATCH', body: JSON.stringify(req) }),
  transitionCaseState: (caseId: string, req: StateTransitionRequest) =>
    fetchJson<ApiResponse<CaseDto>>(`${BASE}/cases/${caseId}/state-transitions`, { method: 'POST', body: JSON.stringify(req) }),
  getCaseRevisions: (caseId: string) =>
    fetchJson<ApiResponse<RevisionRowDto[]>>(`${BASE}/cases/${caseId}/revisions`),

  // Runs
  listRuns: (cursor?: string, limit = 20) =>
    fetchJson<ApiResponse<RunRowDto[]>>(`${BASE}/runs?cursor=${cursor||''}&limit=${limit}`),
  getRun: (runId: string) =>
    fetchJson<ApiResponse<RunDetailDto>>(`${BASE}/runs/${runId}`),
  ingestRun: (formData: FormData) =>
    fetchJson<ApiResponse<RunDto>>(`${BASE}/runs/ingest`, { method: 'POST', body: formData }),

  // AI
  draftCases: (req: AiDraftRequestDto) =>
    fetchJson<ApiResponse<AiDraftResponseDto>>(`${BASE}/ai/draft-cases`, { method: 'POST', body: JSON.stringify(req) }),

  // Traceability
  getTraceability: (reqId: string) =>
    fetchJson<ApiResponse<TraceabilityAggregate>>(`${BASE}/traceability?reqId=${encodeURIComponent(reqId)}`),

  // Environments
  listEnvironments: () =>
    fetchJson<ApiResponse<EnvironmentDto[]>>(`${BASE}/environments`),
  createEnvironment: (req: CreateEnvironmentRequest) =>
    fetchJson<ApiResponse<EnvironmentDto>>(`${BASE}/environments`, { method: 'POST', body: JSON.stringify(req) }),
  updateEnvironment: (envId: string, req: UpdateEnvironmentRequest) =>
    fetchJson<ApiResponse<EnvironmentDto>>(`${BASE}/environments/${envId}`, { method: 'PATCH', body: JSON.stringify(req) }),
};

function qs(f: CatalogFilters): string { /* URLSearchParams builder */ return '' }
```

### 6.2 Pinia Store Pattern

`frontend/src/features/testing-management/stores/testingStore.ts`

```ts
import { defineStore } from 'pinia';
import { ref, computed } from 'vue';
import { testingApi } from '../api/testingApi';

export const useTestingStore = defineStore('testing', () => {
  const currentPlan = ref<PlanDetailAggregate | null>(null);
  const catalogFilters = ref<CatalogFilters>({});
  const loading = ref(false);
  const error = ref<ApiError | null>(null);

  const loadPlan = async (planId: string) => {
    loading.value = true;
    try {
      const response = await testingApi.getPlan(planId);
      if (response.error) {
        error.value = response.error;
        return;
      }
      currentPlan.value = response.data;
      error.value = null;
    } catch (e) {
      error.value = { code: 'NETWORK_ERROR', message: String(e) };
    } finally {
      loading.value = false;
    }
  };

  return {
    currentPlan,
    catalogFilters,
    loading,
    error,
    loadPlan,
  };
});
```

### 6.3 Vite Proxy Configuration

`frontend/vite.config.ts` — add to `server.proxy`:

```ts
'/api/v1/testing': {
  target: 'http://localhost:8080',
  changeOrigin: true,
  secure: false,
}
```

### 6.4 Composable Hook Example

`frontend/src/features/testing-management/composables/usePlanDetail.ts`

```ts
import { useTestingStore } from '../stores/testingStore';
import { useRoute } from 'vue-router';
import { onMounted, computed } from 'vue';

export function usePlanDetail() {
  const store = useTestingStore();
  const route = useRoute();
  const planId = computed(() => route.params.planId as string);

  onMounted(async () => {
    await store.loadPlan(planId.value);
  });

  return {
    plan: computed(() => store.currentPlan),
    loading: computed(() => store.loading),
    error: computed(() => store.error),
  };
}
```

## 7. Testing Contracts

### Golden-file API tests

For each endpoint, a JSON fixture lives under `backend/src/test/resources/tm/fixtures/{endpoint}-{statusCode}.json`. Integration tests compare the serialized response to the fixture.

**Example test:** `CatalogControllerIntegrationTest.java`
```java
@Test
void testGetCatalogSuccess() throws Exception {
    var response = mockMvc.perform(get("/api/v1/testing/catalog")
            .header("X-Workspace-Id", "ws-001")
            .header("Authorization", "Bearer token-user-1"))
        .andExpect(status().isOk())
        .andReturn().getResponse().getContentAsString();
    
    var expected = Files.readString(Path.of("src/test/resources/tm/fixtures/catalog-200.json"));
    assertJsonEquals(expected, response);
}
```

### Webhook signature fixture

`backend/src/test/resources/tm/webhooks/ingest.json` carries a realistic CI payload + pre-computed HMAC-SHA256 signature. Tests verify:
- 202 on valid signature
- 401 on signature mutation
- Idempotent no-op on duplicate delivery

### AI stubs

`AiDraftSkillClient` is a Spring `@Component` injected via interface; a `FakeAiDraftSkillClient` returns deterministic draft responses (title, preconditions, steps, expected, type, priority, citedExcerpt).

**Stub contract tests:**
- SUCCESS: draft returns candidates
- FAILED: transient error (retryable)
- AUTONOMY_DISABLED: returns 403
- AUTONOMY_OBSERVATION: returns drafts but with `approvable=false`

### Parser contract tests

JUnit, TestNG, Playwright, and Cypress parsers have golden fixtures:
- `backend/src/test/resources/tm/parsers/junit-sample.xml` → `RunResultsDto`
- `backend/src/test/resources/tm/parsers/playwright-sample.json` → `RunResultsDto`

### State transition validation

- DRAFT → ACTIVE: allowed (approval)
- DRAFT → DEPRECATED: allowed (rejection)
- ACTIVE → DEPRECATED: allowed (deprecation)
- ACTIVE → DRAFT: disallowed (422)
- DEPRECATED → *: disallowed (422)

## 8. Versioning Policy

- **API version:** path-embedded (`/v1/`). Breaking changes require `/v2/` plus deprecation window.
- **Skill versions:** `skillVersion` string is persisted in AI draft rows; rotating skill invalidates prior drafts naturally.
- **Schema versions:** Flyway monotonic — V50..V59 for Testing Management V1; V60+ reserved for V1.1.
- **Additive-first:** new optional fields are fine; removing or renaming fields triggers major version.
- **AI autonomy levels:** persisted per workspace; `DISABLED` → draft endpoint 403; `OBSERVATION` → drafts visible but read-only; `SUPERVISED`/`AUTONOMOUS` → normal flow (AI never auto-approves).

## 9. Auth Model & Headers

- **Bearer token:** supplied in `Authorization: Bearer <token>` header; decoded to extract user principal.
- **Workspace isolation:** `X-Workspace-Id` header is **required** on all endpoints; user must be a member of the workspace.
- **Role checks:** `Role.QA_LEAD`, `Role.TECH_LEAD`, `Role.PM`, `Role.ENGINEER`, `Role.READER`.
  - `POST /plans`: requires QA_LEAD, PM
  - `POST /cases`: requires QA_LEAD, TECH_LEAD
  - `POST /ai/draft-cases`: requires QA_LEAD, TECH_LEAD
  - `POST /runs/ingest`: requires QA_LEAD
  - `GET *`: READER and above
  - State transitions (approve/reject): requires QA_LEAD, TECH_LEAD
- **Webhook authentication:** HMAC-SHA256 signature in `X-Webhook-Signature` header; no bearer token required.

## 10. Example Implementation Sequence

### Step 1: Create a Plan

```bash
POST /api/v1/testing/plans
Authorization: Bearer <token>
X-Workspace-Id: ws-001
Content-Type: application/json

{
  "projectId": "proj-billing",
  "name": "Payment Processing Q2",
  "releaseTarget": "v2.1",
  "owner": "qa-lead@acme.example"
}

# Response (201)
{
  "data": { "planId": "plan-001", "state": "DRAFT", ... },
  "meta": { "correlationId": "tm-post-001", ... }
}
```

### Step 2: Create Test Cases

```bash
POST /api/v1/testing/cases
Authorization: Bearer <token>
X-Workspace-Id: ws-001
Content-Type: application/json

{
  "planId": "plan-001",
  "title": "User initiates payment",
  "type": "FUNCTIONAL",
  "priority": "P0",
  "steps": "1. Click Checkout\n2. Select Card\n3. Enter details\n4. Click Pay",
  "expected": "Form submitted",
  "linkedReqIds": ["REQ-BILLING-001"]
}
```

### Step 3: (Optional) Request AI Drafts

```bash
POST /api/v1/testing/ai/draft-cases
Authorization: Bearer <token>
X-Workspace-Id: ws-001
Content-Type: application/json

{
  "planId": "plan-001",
  "reqId": "REQ-BILLING-002",
  "skillVersion": "test-case-drafter-v1.2"
}

# Response (200)
{
  "data": {
    "reqId": "REQ-BILLING-002",
    "candidates": [ { "caseId": "case-draft-001", "title": "...", "state": "DRAFT" } ],
    "quotaRemaining": 19
  }
}
```

### Step 4: Ingest a Test Run

```bash
POST /api/v1/testing/runs/ingest
Authorization: Bearer <token>
X-Workspace-Id: ws-001
Content-Type: multipart/form-data

metadata: { "planId": "plan-001", "environment": "staging", "externalRunId": "gha-run-99012" }
file: <JUnit XML file>

# Response (202 Accepted)
{
  "data": { "runId": "run-001", "state": "RUNNING", ... }
}
```

### Step 5: View Results

```bash
GET /api/v1/testing/runs/run-001
Authorization: Bearer <token>
X-Workspace-Id: ws-001

# Response (200)
{
  "data": {
    "runId": "run-001",
    "state": "PASSED",
    "passCount": 38, "failCount": 0, "skipCount": 4,
    "caseOutcomes": [ { "caseId": "case-0001", "status": "PASSED", ... } ],
    "coveredReqs": [ { "reqId": "REQ-BILLING-001", ... } ]
  }
}
```
