# Team Space — API Implementation Guide

## Purpose

This is the executable API contract for the Team Space slice. Both the backend (Codex-driven Spring Boot / Java 21) and the frontend (Gemini-driven Vue 3 / TypeScript) must conform to this contract.

## Traceability

- Spec: [team-space-spec.md](../../03-spec/team-space-spec.md)
- Architecture: [team-space-architecture.md](../../04-architecture/team-space-architecture.md)
- Data Model: [team-space-data-model.md](../../04-architecture/team-space-data-model.md)
- Design: [team-space-design.md](../team-space-design.md)

---

## 1. API Overview

### Base URL

```
/api/v1/team-space
```

### Response Envelope

All responses use the shared envelope:

```json
{
  "data": { /* typed payload */ },
  "error": null
}
```

Error responses:

```json
{
  "data": null,
  "error": "Workspace ws-999 not found"
}
```

### Date Format

ISO-8601 UTC strings. Java `Instant` serializes via Jackson's default configuration.

### Backend Constants

```java
// shared/ApiConstants.java
public final class ApiConstants {
    public static final String TEAM_SPACE = API_V1 + "/team-space";
}

// domain/teamspace/TeamSpaceConstants.java
public final class TeamSpaceConstants {
    public static final Duration PROJECTION_BUDGET = Duration.ofMillis(500);
    public static final Duration AGGREGATE_TOTAL_BUDGET = Duration.ofMillis(2_000);
}
```

### Authorization

- All endpoints require a valid authenticated principal.
- `WorkspaceAccessGuard` enforces read access on `:workspaceId`.
- Denial responses use the shared `ApiResponse.fail(...)` envelope with HTTP 403.

---

## 2. GET /api/v1/team-space/{workspaceId}

Aggregate first-paint endpoint.

### 2.1 Request

```
GET /api/v1/team-space/ws-default-001
Authorization: Bearer <token>
```

Path parameter:

- `workspaceId`: string, pattern `^ws-[a-z0-9\-]+$`

### 2.2 Success Response (200 OK)

```json
{
  "data": {
    "workspaceId": "ws-default-001",
    "summary": {
      "data": {
        "id": "ws-default-001",
        "name": "Payments Platform",
        "applicationId": "app-payments",
        "applicationName": "Payments",
        "snowGroupId": "snow-payments-core",
        "snowGroupName": "Payments Core",
        "activeProjectCount": 7,
        "activeEnvironmentCount": 4,
        "healthAggregate": "YELLOW",
        "ownerId": "u-007",
        "ownerDisplayName": "Grace Hopper",
        "compatibilityMode": false,
        "responsibilityBoundary": {
          "applications": ["Payments"],
          "snowGroups": ["Payments Core"],
          "projectCount": 7
        }
      },
      "error": null
    },
    "operatingModel": { "data": { /* see §3 shape */ }, "error": null },
    "members": { "data": { /* see §4 shape */ }, "error": null },
    "templates": { "data": { /* see §5 shape */ }, "error": null },
    "pipeline": { "data": { /* see §6 shape */ }, "error": null },
    "metrics": { "data": { /* see §7 shape */ }, "error": null },
    "risks": { "data": { /* see §8 shape */ }, "error": null },
    "projects": { "data": { /* see §9 shape */ }, "error": null }
  },
  "error": null
}
```

Each inner section is a `SectionResultDto` — if one projection fails, only that section carries `data: null` and a non-null `error`.

### 2.3 Error Cases

| Status | Example error message | When |
|--------|------------------------|------|
| 400 | `Invalid workspaceId: INVALID` | Path param does not match pattern |
| 403 | `Workspace access denied: ws-other` | User lacks read access |
| 404 | `Workspace ws-999 not found` | Workspace id does not exist |
| 500 | `Internal server error` | Both aggregate and all projections failed |

---

## 3. GET /api/v1/team-space/{workspaceId}/summary

### 3.1 Request

```
GET /api/v1/team-space/ws-default-001/summary
```

### 3.2 Success Response (200 OK)

```json
{
  "data": {
    "id": "ws-default-001",
    "name": "Payments Platform",
    "applicationId": "app-payments",
    "applicationName": "Payments",
    "snowGroupId": "snow-payments-core",
    "snowGroupName": "Payments Core",
    "activeProjectCount": 7,
    "activeEnvironmentCount": 4,
    "healthAggregate": "YELLOW",
    "ownerId": "u-007",
    "ownerDisplayName": "Grace Hopper",
    "compatibilityMode": false,
    "responsibilityBoundary": {
      "applications": ["Payments"],
      "snowGroups": ["Payments Core"],
      "projectCount": 7
    }
  }
}
```

### 3.3 Compatibility Mode Example

```json
{
  "data": {
    "id": "ws-legacy-001",
    "name": "Legacy Mainframe",
    "snowGroupId": null,
    "snowGroupName": null,
    "compatibilityMode": true,
    "...": "..."
  }
}
```

---

## 4. GET /api/v1/team-space/{workspaceId}/operating-model

### 4.1 Success Response (200 OK)

```json
{
  "data": {
    "operatingMode": {
      "value": "STANDARD",
      "lineage": {
        "origin": "APPLICATION",
        "overridden": false,
        "chain": [
          { "origin": "PLATFORM", "value": "STANDARD", "setAt": "2026-01-10T00:00:00Z", "setBy": "platform-admin" },
          { "origin": "APPLICATION", "value": "STANDARD", "setAt": "2026-02-01T00:00:00Z", "setBy": "app-owner-payments" }
        ]
      }
    },
    "approvalMode": {
      "value": "REVIEWER_REQUIRED",
      "lineage": { "origin": "WORKSPACE", "overridden": true, "chain": [] }
    },
    "aiAutonomyLevel": {
      "value": "HUMAN_IN_LOOP",
      "lineage": { "origin": "PLATFORM", "overridden": false, "chain": [] }
    },
    "oncallOwner": {
      "value": {
        "memberId": "u-011",
        "displayName": "Alan Turing",
        "rotationRef": "rot-payments-primary"
      },
      "lineage": { "origin": "WORKSPACE", "overridden": false, "chain": [] }
    },
    "accountableOwners": [
      { "area": "DELIVERY", "memberId": "u-007", "displayName": "Grace Hopper" },
      { "area": "APPROVAL", "memberId": "u-003", "displayName": "Ada Lovelace" },
      { "area": "INCIDENT", "memberId": "u-011", "displayName": "Alan Turing" },
      { "area": "GOVERNANCE", "memberId": "u-007", "displayName": "Grace Hopper" }
    ],
    "platformCenterLink": {
      "url": "/platform?view=config&workspaceId=ws-default-001&section=operating-model",
      "enabled": false
    }
  }
}
```

---

## 5. GET /api/v1/team-space/{workspaceId}/members

### 5.1 Success Response (200 OK)

```json
{
  "data": {
    "members": [
      {
        "memberId": "u-007",
        "displayName": "Grace Hopper",
        "roles": ["TEAM_LEAD", "APPROVER"],
        "oncallStatus": "OFF",
        "keyPermissions": ["APPROVE", "CONFIGURE"],
        "lastActiveAt": "2026-04-17T09:42:00Z"
      },
      {
        "memberId": "u-011",
        "displayName": "Alan Turing",
        "roles": ["SRE", "ONCALL_PRIMARY"],
        "oncallStatus": "ON",
        "keyPermissions": ["DEPLOY"],
        "lastActiveAt": "2026-04-17T10:05:00Z"
      }
    ],
    "coverageGaps": [
      {
        "kind": "BACKUP_MISSING",
        "description": "No secondary oncall assigned for the 2026-04-19 – 2026-04-21 window",
        "window": "2026-04-19 – 2026-04-21"
      }
    ],
    "accessManagementLink": {
      "url": "/platform?view=access&workspaceId=ws-default-001",
      "enabled": false
    }
  }
}
```

---

## 6. GET /api/v1/team-space/{workspaceId}/templates

### 6.1 Success Response (200 OK)

```json
{
  "data": {
    "groups": {
      "PAGE": [
        {
          "id": "tpl-page-team-space",
          "name": "Team Space Layout",
          "version": "1.0.0",
          "kind": "PAGE",
          "lineage": { "origin": "PLATFORM", "overridden": false, "chain": [] }
        }
      ],
      "POLICY": [
        {
          "id": "tpl-policy-approval",
          "name": "Default Approval Policy",
          "version": "2.1.0",
          "kind": "POLICY",
          "lineage": { "origin": "APPLICATION", "overridden": false, "chain": [] }
        }
      ],
      "WORKFLOW": [],
      "SKILL_PACK": [
        {
          "id": "tpl-skill-standard-sdd",
          "name": "Standard SDD Skill Pack",
          "version": "1.2.0",
          "kind": "SKILL_PACK",
          "lineage": { "origin": "PLATFORM", "overridden": false, "chain": [] }
        }
      ],
      "AI_DEFAULT": [
        {
          "id": "tpl-ai-default",
          "name": "AI Default Config",
          "version": "0.3.0",
          "kind": "AI_DEFAULT",
          "lineage": { "origin": "WORKSPACE", "overridden": true, "chain": [] }
        }
      ]
    },
    "exceptionOverrides": [
      {
        "templateId": "tpl-policy-approval",
        "templateName": "Default Approval Policy",
        "overrideScope": "PROJECT",
        "overrideScopeId": "proj-42",
        "overrideScopeName": "Gateway Migration",
        "overriddenAt": "2026-03-12T00:00:00Z",
        "overriddenBy": "u-007"
      }
    ]
  }
}
```

---

## 7. GET /api/v1/team-space/{workspaceId}/pipeline

### 7.1 Success Response (200 OK)

```json
{
  "data": {
    "counters": {
      "requirementsInflow7d": 12,
      "storiesDecomposing": 18,
      "specsGenerating": 3,
      "specsInReview": 5,
      "specsBlocked": 2,
      "specsApprovedAwaitingDownstream": 7
    },
    "blockers": [
      {
        "kind": "SPEC_BLOCKED",
        "targetId": "SPEC-0088",
        "targetTitle": "Webhook retry strategy",
        "ageDays": 5,
        "filterDeeplink": "/requirements?filter=blocked-specs&workspaceId=ws-default-001"
      }
    ],
    "chain": [
      { "nodeKey": "REQUIREMENT",  "health": "GREEN",  "isExecutionHub": false },
      { "nodeKey": "USER_STORY",   "health": "GREEN",  "isExecutionHub": false },
      { "nodeKey": "SPEC",         "health": "YELLOW", "isExecutionHub": true  },
      { "nodeKey": "ARCHITECTURE", "health": "GREEN",  "isExecutionHub": false },
      { "nodeKey": "DESIGN",       "health": "GREEN",  "isExecutionHub": false },
      { "nodeKey": "TASKS",        "health": "GREEN",  "isExecutionHub": false },
      { "nodeKey": "CODE",         "health": "GREEN",  "isExecutionHub": false },
      { "nodeKey": "TEST",         "health": "YELLOW", "isExecutionHub": false },
      { "nodeKey": "DEPLOY",       "health": "GREEN",  "isExecutionHub": false },
      { "nodeKey": "INCIDENT",     "health": "RED",    "isExecutionHub": false },
      { "nodeKey": "LEARNING",     "health": "GREEN",  "isExecutionHub": false }
    ],
    "blockerThresholdDays": 3
  }
}
```

---

## 8. GET /api/v1/team-space/{workspaceId}/metrics

### 8.1 Success Response (200 OK)

```json
{
  "data": {
    "deliveryEfficiency": [
      {
        "key": "delivery.cycleTime",
        "label": "Cycle Time",
        "currentValue": 4.2,
        "previousValue": 5.1,
        "unit": "DAYS",
        "trend": "DOWN",
        "historyLink": "/reports/metric/delivery.cycleTime?workspaceId=ws-default-001",
        "tooltip": "Average days from Approved to Delivered"
      },
      {
        "key": "delivery.throughput",
        "label": "Throughput",
        "currentValue": 18.0,
        "previousValue": 15.0,
        "unit": "COUNT",
        "trend": "UP",
        "historyLink": "/reports/metric/delivery.throughput?workspaceId=ws-default-001",
        "tooltip": "Requirements delivered per week"
      }
    ],
    "quality": [ /* ... */ ],
    "stability": [ /* ... */ ],
    "governanceMaturity": [ /* ... */ ],
    "aiParticipation": [ /* ... */ ],
    "lastRefreshed": "2026-04-17T01:00:00Z"
  }
}
```

---

## 9. GET /api/v1/team-space/{workspaceId}/risks

### 9.1 Success Response (200 OK)

```json
{
  "data": {
    "groups": {
      "INCIDENT": [
        {
          "id": "RISK-1001",
          "category": "INCIDENT",
          "severity": "CRITICAL",
          "title": "P1: payment-service outage",
          "detail": "Payment throughput 0 rps for 12m",
          "ageDays": 0,
          "primaryAction": {
            "label": "Open Incident",
            "url": "/incidents/INC-0042"
          }
        }
      ],
      "APPROVAL": [
        {
          "id": "RISK-1002",
          "category": "APPROVAL",
          "severity": "HIGH",
          "title": "4 Spec approvals pending > 3d",
          "detail": "Oldest: SPEC-0088 (5 days)",
          "ageDays": 3,
          "primaryAction": {
            "label": "Review approvals",
            "url": "/platform?view=approvals&workspaceId=ws-default-001"
          }
        }
      ],
      "CONFIG_DRIFT": [
        {
          "id": "RISK-1003",
          "category": "CONFIG_DRIFT",
          "severity": "MEDIUM",
          "title": "Approval mode overridden at project level for 3 projects",
          "detail": "Projects: proj-42, proj-55, proj-88",
          "ageDays": 12,
          "primaryAction": {
            "label": "View in Platform Center",
            "url": "/platform?view=config&workspaceId=ws-default-001&section=approval"
          }
        }
      ],
      "PROJECT": [],
      "DEPENDENCY": []
    },
    "lastRefreshed": "2026-04-17T10:00:00Z",
    "total": 3
  }
}
```

---

## 10. GET /api/v1/team-space/{workspaceId}/projects

### 10.1 Success Response (200 OK)

```json
{
  "data": {
    "groups": {
      "HEALTHY": [
        {
          "id": "proj-11",
          "name": "Card Issuance",
          "lifecycleStage": "DELIVERY",
          "healthStratum": "HEALTHY",
          "primaryRisk": null,
          "activeSpecCount": 4,
          "openIncidentCount": 0,
          "projectSpaceUrl": "/project-space/proj-11"
        }
      ],
      "AT_RISK": [
        {
          "id": "proj-42",
          "name": "Gateway Migration",
          "lifecycleStage": "DELIVERY",
          "healthStratum": "AT_RISK",
          "primaryRisk": "2 blocked specs",
          "activeSpecCount": 7,
          "openIncidentCount": 1,
          "projectSpaceUrl": "/project-space/proj-42"
        }
      ],
      "CRITICAL": [],
      "ARCHIVED": []
    },
    "totals": {
      "HEALTHY": 4,
      "AT_RISK": 2,
      "CRITICAL": 0,
      "ARCHIVED": 1
    }
  }
}
```

---

## 11. Backend Implementation Notes

### 11.1 Controller Skeleton

```java
@RestController
@RequestMapping(ApiConstants.TEAM_SPACE)
public class TeamSpaceController {

    private final TeamSpaceService service;
    private final WorkspaceAccessGuard accessGuard;

    public TeamSpaceController(TeamSpaceService service, WorkspaceAccessGuard accessGuard) {
        this.service = service;
        this.accessGuard = accessGuard;
    }

    @GetMapping("/{workspaceId}")
    public ApiResponse<TeamSpaceAggregateDto> aggregate(
            @PathVariable @Pattern(regexp = "^ws-[a-z0-9\\-]+$") String workspaceId
    ) {
        accessGuard.check(workspaceId);
        return ApiResponse.ok(service.loadAggregate(workspaceId));
    }

    @GetMapping("/{workspaceId}/summary")
    public ApiResponse<WorkspaceSummaryDto> summary(@PathVariable String workspaceId) {
        accessGuard.check(workspaceId);
        return ApiResponse.ok(service.loadSummary(workspaceId));
    }

    // ... operating-model, members, templates, pipeline, metrics, risks, projects
}
```

### 11.2 Service Layer

```java
@Service
public class TeamSpaceService {

    private final List<TeamSpaceProjection<?>> projections;
    private final Executor projectionExecutor;

    public TeamSpaceAggregateDto loadAggregate(String workspaceId) {
        // Parallel fan-out with per-projection timeout (500ms)
        // Collect into TeamSpaceAggregateDto with per-section SectionResultDto
    }
}
```

Each projection implements:

```java
public interface TeamSpaceProjection<T> {
    String key();
    SectionResultDto<T> load(String workspaceId);
}
```

### 11.3 Access Guard

```java
@Component
public class WorkspaceAccessGuard {
    public void check(String workspaceId) {
        // Resolve current authenticated principal
        // Verify user has read access to workspaceId
        // Throw WorkspaceAccessDeniedException on failure
    }
}
```

### 11.4 Package Location

Package-by-feature under `com.sdlctower.domain.teamspace`. Shared primitives reuse the existing `com.sdlctower.shared.dto.ApiResponse`, `com.sdlctower.shared.dto.SectionResultDto`, and `com.sdlctower.shared.ApiConstants`.

### 11.5 Migration

- `V7__create_team_space_tables.sql` — `risk_signals`, `metric_snapshots` tables
- `V8__seed_team_space_data.sql` — seed data for local dev

Run via Flyway on app startup. No `ddl-auto: update`.

---

## 12. Frontend Implementation Notes

### 12.1 API Client

```typescript
// frontend/src/features/team-space/api/teamSpaceApi.ts

import { fetchJson } from '@/shared/api/client';
import type { TeamSpaceAggregate, TeamSpaceCardKey, WorkspaceSummary, /* ... */ } from '../types/aggregate';

const USE_MOCK = import.meta.env.DEV && !import.meta.env.VITE_USE_BACKEND;
const SECTION_PATHS: Record<TeamSpaceCardKey, string> = {
  summary: 'summary',
  operatingModel: 'operating-model',
  members: 'members',
  templates: 'templates',
  pipeline: 'pipeline',
  metrics: 'metrics',
  risks: 'risks',
  projects: 'projects',
};

export const teamSpaceApi = {
  async getAggregate(workspaceId: string): Promise<TeamSpaceAggregate> {
    if (USE_MOCK) return (await import('../mock/aggregate.mock')).aggregate(workspaceId);
    return fetchJson<TeamSpaceAggregate>(`/team-space/${workspaceId}`);
  },

  async getSummary(workspaceId: string): Promise<WorkspaceSummary> {
    if (USE_MOCK) return (await import('../mock/summary.mock')).summary(workspaceId);
    return fetchJson<WorkspaceSummary>(`/team-space/${workspaceId}/summary`);
  },

  async getSection(cardKey: TeamSpaceCardKey, workspaceId: string): Promise<TeamSpaceAggregate[TeamSpaceCardKey]> {
    return fetchJson<TeamSpaceAggregate[TeamSpaceCardKey]>(`/team-space/${workspaceId}/${SECTION_PATHS[cardKey]}`);
  },
};
```

### 12.2 Pinia Store

```typescript
// frontend/src/features/team-space/stores/teamSpaceStore.ts

import { defineStore } from 'pinia';
import { ref } from 'vue';
import { teamSpaceApi } from '../api/teamSpaceApi';
import type { TeamSpaceAggregate, TeamSpaceCardKey } from '../types/aggregate';

export const useTeamSpaceStore = defineStore('teamSpace', () => {
  const workspaceId = ref<string | null>(null);
  const aggregate = ref<TeamSpaceAggregate | null>(null);
  const isLoading = ref(false);
  const error = ref<string | null>(null);
  const loadingCards = ref<Record<TeamSpaceCardKey, boolean>>({
    summary: false,
    operatingModel: false,
    members: false,
    templates: false,
    pipeline: false,
    metrics: false,
    risks: false,
    projects: false,
  });

  async function initWorkspace(nextWorkspaceId: string) {
    workspaceId.value = nextWorkspaceId;
    isLoading.value = true;
    error.value = null;
    try {
      aggregate.value = await teamSpaceApi.getAggregate(nextWorkspaceId);
    } catch (err) {
      error.value = err instanceof Error ? err.message : 'Failed to load Team Space';
    } finally {
      isLoading.value = false;
    }
  }

  async function retryCard(cardKey: TeamSpaceCardKey) {
    if (!workspaceId.value || !aggregate.value) return;
    loadingCards.value[cardKey] = true;
    try {
      aggregate.value[cardKey] = await teamSpaceApi.getSection(cardKey, workspaceId.value);
    } finally {
      loadingCards.value[cardKey] = false;
    }
  }

  return { workspaceId, aggregate, isLoading, error, loadingCards, initWorkspace, retryCard };
});
```

### 12.3 Vite Proxy

```typescript
// vite.config.ts (additions)
server: {
  proxy: {
    '/api': {
      target: 'http://localhost:8080',
      changeOrigin: true,
    },
  },
}
```

### 12.4 Mock Aggregate Skeleton

```typescript
// src/features/team-space/mock/aggregate.mock.ts

import type { TeamSpaceAggregate } from '../types/aggregate';

export function aggregate(workspaceId: string): TeamSpaceAggregate {
  return {
    workspaceId,
    summary: { data: { /* ... */ }, error: null },
    operatingModel: { data: { /* ... */ }, error: null },
    // ...
  };
}
```

---

## 13. Testing Contracts

### 13.1 Backend (MockMvc)

```java
@WebMvcTest(TeamSpaceController.class)
class TeamSpaceControllerTest {

    @Autowired MockMvc mvc;
    @MockBean TeamSpaceService service;
    @MockBean WorkspaceAccessGuard accessGuard;

    @Test void aggregate_returns200_withValidWorkspace() throws Exception {
        when(service.loadAggregate("ws-default-001")).thenReturn(sampleAggregate());
        mvc.perform(get("/api/v1/team-space/ws-default-001"))
           .andExpect(status().isOk())
           .andExpect(jsonPath("$.data.workspaceId").value("ws-default-001"))
           .andExpect(jsonPath("$.data.summary.data.id").value("ws-default-001"))
           .andExpect(jsonPath("$.data.summary.error").isEmpty());
    }

    @Test void aggregate_returns400_withInvalidId() throws Exception {
        mvc.perform(get("/api/v1/team-space/INVALID"))
           .andExpect(status().isBadRequest())
           .andExpect(jsonPath("$.error").value(org.hamcrest.Matchers.containsString("Invalid workspaceId")));
    }

    @Test void aggregate_returns403_whenAccessDenied() throws Exception {
        doThrow(new WorkspaceAccessDeniedException("ws-other"))
            .when(accessGuard).check("ws-other");
        mvc.perform(get("/api/v1/team-space/ws-other"))
           .andExpect(status().isForbidden())
           .andExpect(jsonPath("$.error").value(org.hamcrest.Matchers.containsString("Workspace access denied")));
    }
}
```

Also verify:

- Per-projection timeout yields `SectionResultDto.data == null` and a non-null `error` for that section only.
- Parallel fan-out: service completes within the aggregate total budget.
- Flyway V7 / V8 migrations run cleanly on H2.

### 13.2 Frontend (Vitest)

```typescript
describe('teamSpaceStore', () => {
  it('initWorkspace populates all cards from aggregate', async () => {
    const store = useTeamSpaceStore();
    await store.initWorkspace('ws-default-001');
    expect(store.aggregate?.summary.data).not.toBeNull();
    expect(store.aggregate?.risks.error).toBeNull();
    expect(store.error).toBeNull();
  });

  it('switchWorkspace resets and reloads', async () => {
    const store = useTeamSpaceStore();
    await store.initWorkspace('ws-a');
    await store.switchWorkspace('ws-b');
    expect(store.workspaceId).toBe('ws-b');
  });

  it('retryCard recovers a single card without affecting others', async () => {
    // ...
  });
});

describe('SdlcChainStrip', () => {
  it('always renders 11 nodes', () => {
    // mount with shorter chain → still 11 rendered (missing default to UNKNOWN)
  });

  it('highlights Spec as execution hub', () => {
    // ...
  });
});
```

---

## 14. Versioning Policy

- Path-versioned at `/api/v1`. Breaking changes require a new version path.
- Backward-compatible additions: new optional fields, new enum values (frontend must gracefully handle unknown enum values).
- Frontend fails-safe: unknown enum values render as neutral chips; unknown card keys in aggregate are ignored.
- All breaking changes announced in repo release notes plus CLAUDE.md lessons learned section.
