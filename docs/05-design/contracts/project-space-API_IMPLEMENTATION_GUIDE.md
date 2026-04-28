# Project Space — API Implementation Guide

## Purpose

This is the executable API contract for the Project Space slice. Both the backend (Codex-driven Spring Boot / Java 21) and the frontend (Gemini-driven Vue 3 / TypeScript) must conform to this contract.

## Traceability

- Spec: [project-space-spec.md](../../03-spec/project-space-spec.md)
- Architecture: [project-space-architecture.md](../../04-architecture/project-space-architecture.md)
- Data Model: [project-space-data-model.md](../../04-architecture/project-space-data-model.md)
- Design: [project-space-design.md](../project-space-design.md)

---

## 1. API Overview

### Base URL

```
/api/v1/project-space
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
  "error": "Project proj-999 not found"
}
```

### Date Format

ISO-8601 UTC for `Instant` fields (e.g., `lastUpdatedAt`), ISO date (`YYYY-MM-DD`) for `LocalDate` fields (e.g., `targetDate`). Java Jackson defaults are used.

### Backend Constants

```java
// shared/ApiConstants.java
public final class ApiConstants {
    public static final String PROJECT_SPACE = API_V1 + "/project-space";
}

// domain/projectspace/ProjectSpaceConstants.java
public final class ProjectSpaceConstants {
    public static final Duration PROJECTION_BUDGET = Duration.ofMillis(500);
    public static final Duration AGGREGATE_TOTAL_BUDGET = Duration.ofMillis(2_000);
    public static final int VERSION_DRIFT_MINOR_MAX = 10;
}
```

### Authorization

- All endpoints require a valid authenticated principal.
- `ProjectAccessGuard` enforces read access on `:projectId`.
- Denial responses use the shared `ApiResponse.fail(...)` envelope with HTTP 403.

---

## 2. GET /api/v1/project-space/{projectId}

Aggregate first-paint endpoint.

### 2.1 Request

```
GET /api/v1/project-space/proj-8821
Authorization: Bearer <token>
```

Path parameter:

- `projectId`: string, pattern `^proj-[a-z0-9\-]+$`

### 2.2 Success Response (200 OK)

```json
{
  "data": {
    "projectId": "proj-8821",
    "workspaceId": "ws-default-001",
    "summary":      { "data": { /* see §3 shape */ }, "error": null },
    "leadership":   { "data": { /* see §4 shape */ }, "error": null },
    "chain":        { "data": { /* see §5 shape */ }, "error": null },
    "milestones":   { "data": { /* see §6 shape */ }, "error": null },
    "dependencies": { "data": { /* see §7 shape */ }, "error": null },
    "risks":        { "data": { /* see §8 shape */ }, "error": null },
    "environments": { "data": { /* see §9 shape */ }, "error": null }
  },
  "error": null
}
```

Each inner section is a `SectionResultDto` — if one projection fails, only that section carries `data: null` and a non-null `error`.

### 2.3 Error Cases

| Status | Example error message | When |
|--------|------------------------|------|
| 400 | `Invalid projectId: INVALID` | Path param does not match pattern |
| 403 | `Project access denied: proj-other` | User lacks read access |
| 404 | `Project proj-999 not found` | Project id does not exist |
| 500 | `Internal server error` | Both aggregate and all projections failed |

---

## 3. GET /api/v1/project-space/{projectId}/summary

### 3.1 Request

```
GET /api/v1/project-space/proj-8821/summary
```

### 3.2 Success Response (200 OK)

```json
{
  "data": {
    "id": "proj-8821",
    "name": "Quantum Mesh Gateway",
    "workspaceId": "ws-default-001",
    "workspaceName": "Payments Platform",
    "applicationId": "app-payments",
    "applicationName": "Payments",
    "lifecycleStage": "DELIVERY",
    "healthAggregate": "YELLOW",
    "healthFactors": [
      { "label": "1 critical dependency risk", "severity": "CRIT" },
      { "label": "Milestone GA Launch at risk", "severity": "WARN" }
    ],
    "pm":       { "memberId": "u-007", "displayName": "Grace Hopper" },
    "techLead": { "memberId": "u-011", "displayName": "Alan Turing" },
    "activeMilestone": {
      "id": "MS-PRJ8821-2",
      "label": "Alpha Release",
      "targetDate": "2026-05-01"
    },
    "counters": {
      "activeSpecs": 7,
      "openIncidents": 1,
      "pendingApprovals": 3,
      "criticalHighRisks": 2
    },
    "lastUpdatedAt": "2026-04-17T10:05:00Z",
    "teamSpaceLink": "/team?workspaceId=ws-default-001"
  },
  "error": null
}
```

---

## 4. GET /api/v1/project-space/{projectId}/leadership

### 4.1 Success Response (200 OK)

```json
{
  "data": {
    "assignments": [
      {
        "role": "PM",
        "memberId": "u-007",
        "displayName": "Grace Hopper",
        "oncallStatus": "OFF",
        "backupPresent": true,
        "backupMemberId": "u-012",
        "backupDisplayName": "Katherine Johnson"
      },
      {
        "role": "ARCHITECT",
        "memberId": "u-003",
        "displayName": "Ada Lovelace",
        "oncallStatus": "OFF",
        "backupPresent": false,
        "backupMemberId": null,
        "backupDisplayName": null
      },
      {
        "role": "TECH_LEAD",
        "memberId": "u-011",
        "displayName": "Alan Turing",
        "oncallStatus": "ON",
        "backupPresent": true,
        "backupMemberId": "u-015",
        "backupDisplayName": "Barbara Liskov"
      },
      {
        "role": "QA_LEAD",
        "memberId": "u-020",
        "displayName": "Margaret Hamilton",
        "oncallStatus": "OFF",
        "backupPresent": true,
        "backupMemberId": "u-021",
        "backupDisplayName": "Carol Shaw"
      },
      {
        "role": "SRE",
        "memberId": "u-030",
        "displayName": "Linus Torvalds",
        "oncallStatus": "UPCOMING",
        "backupPresent": false,
        "backupMemberId": null,
        "backupDisplayName": null
      },
      {
        "role": "AI_ADOPTION",
        "memberId": null,
        "displayName": null,
        "oncallStatus": "NONE",
        "backupPresent": false,
        "backupMemberId": null,
        "backupDisplayName": null
      }
    ],
    "accessManagementLink": {
      "url": "/platform?view=access&projectId=proj-8821",
      "enabled": false
    }
  },
  "error": null
}
```

---

## 5. GET /api/v1/project-space/{projectId}/chain

### 5.1 Success Response (200 OK)

```json
{
  "data": {
    "nodes": [
      { "nodeKey": "REQUIREMENT",  "label": "Requirement",  "count": 12, "health": "GREEN",  "isExecutionHub": false, "deepLink": "/requirements?projectId=proj-8821", "enabled": true },
      { "nodeKey": "USER_STORY",   "label": "User Story",   "count": 18, "health": "GREEN",  "isExecutionHub": false, "deepLink": "/requirements?projectId=proj-8821&node=story", "enabled": true },
      { "nodeKey": "SPEC",         "label": "Spec",         "count":  7, "health": "YELLOW", "isExecutionHub": true,  "deepLink": "/requirements?projectId=proj-8821&node=spec",  "enabled": true },
      { "nodeKey": "ARCHITECTURE", "label": "Architecture", "count":  3, "health": "GREEN",  "isExecutionHub": false, "deepLink": "/design?projectId=proj-8821&node=arch",        "enabled": false },
      { "nodeKey": "DESIGN",       "label": "Design",       "count":  5, "health": "GREEN",  "isExecutionHub": false, "deepLink": "/design?projectId=proj-8821",                  "enabled": false },
      { "nodeKey": "TASKS",        "label": "Tasks",        "count": 42, "health": "GREEN",  "isExecutionHub": false, "deepLink": "/code?projectId=proj-8821&node=tasks",         "enabled": false },
      { "nodeKey": "CODE",         "label": "Code & Build", "count": null,"health": "GREEN",  "isExecutionHub": false, "deepLink": "/code?projectId=proj-8821",                    "enabled": false },
      { "nodeKey": "TEST",         "label": "Test",         "count": null,"health": "YELLOW", "isExecutionHub": false, "deepLink": "/testing?projectId=proj-8821",                 "enabled": false },
      { "nodeKey": "DEPLOY",       "label": "Deploy",       "count":  3, "health": "GREEN",  "isExecutionHub": false, "deepLink": "/deployment?projectId=proj-8821",              "enabled": false },
      { "nodeKey": "INCIDENT",     "label": "Incident",     "count":  1, "health": "RED",    "isExecutionHub": false, "deepLink": "/incidents?projectId=proj-8821",               "enabled": true },
      { "nodeKey": "LEARNING",     "label": "Learning",     "count": null,"health": "GREEN",  "isExecutionHub": false, "deepLink": "/reports/learning?projectId=proj-8821",        "enabled": false }
    ]
  },
  "error": null
}
```

Response must always contain 11 nodes in canonical order. The `enabled` field reflects feature flags for downstream pages.

---

## 6. GET /api/v1/project-space/{projectId}/milestones

### 6.1 Success Response (200 OK)

```json
{
  "data": {
    "milestones": [
      {
        "id": "MS-PRJ8821-1",
        "label": "Discovery Complete",
        "targetDate": "2026-03-15",
        "status": "COMPLETED",
        "percentComplete": 100,
        "owner": { "memberId": "u-007", "displayName": "Grace Hopper" },
        "isCurrent": false,
        "slippageReason": null
      },
      {
        "id": "MS-PRJ8821-2",
        "label": "Alpha Release",
        "targetDate": "2026-05-01",
        "status": "IN_PROGRESS",
        "percentComplete": 60,
        "owner": { "memberId": "u-007", "displayName": "Grace Hopper" },
        "isCurrent": true,
        "slippageReason": null
      },
      {
        "id": "MS-PRJ8821-3",
        "label": "GA Launch",
        "targetDate": "2026-06-30",
        "status": "AT_RISK",
        "percentComplete": 20,
        "owner": { "memberId": "u-007", "displayName": "Grace Hopper" },
        "isCurrent": false,
        "slippageReason": "Upstream identity-service-v2 slipping"
      },
      {
        "id": "MS-PRJ8821-4",
        "label": "Steady-State Handover",
        "targetDate": "2026-09-15",
        "status": "NOT_STARTED",
        "percentComplete": null,
        "owner": null,
        "isCurrent": false,
        "slippageReason": null
      }
    ],
    "projectManagementLink": {
      "url": "/project-management?projectId=proj-8821",
      "enabled": false
    }
  },
  "error": null
}
```

---

## 7. GET /api/v1/project-space/{projectId}/dependencies

### 7.1 Success Response (200 OK)

```json
{
  "data": {
    "upstream": [
      {
        "id": "DEP-1",
        "targetName": "Identity-Service-V2",
        "targetRef": "proj-identity-v2",
        "targetProjectId": "proj-identity-v2",
        "external": false,
        "direction": "UPSTREAM",
        "relationship": "API",
        "ownerTeam": "Identity Platform",
        "health": "YELLOW",
        "blockerReason": "Slipping on SSO contract",
        "primaryAction": {
          "url": "/project-space/proj-identity-v2",
          "enabled": true
        }
      },
      {
        "id": "DEP-3",
        "targetName": "Salesforce CPQ",
        "targetRef": "external:salesforce-cpq",
        "targetProjectId": null,
        "external": true,
        "direction": "UPSTREAM",
        "relationship": "DATA",
        "ownerTeam": "External",
        "health": "UNKNOWN",
        "blockerReason": null,
        "primaryAction": null
      }
    ],
    "downstream": [
      {
        "id": "DEP-2",
        "targetName": "Mobile-Gateway-Alpha",
        "targetRef": "external:mobile-gw-alpha",
        "targetProjectId": null,
        "external": true,
        "direction": "DOWNSTREAM",
        "relationship": "API",
        "ownerTeam": "Mobile Platform",
        "health": "GREEN",
        "blockerReason": null,
        "primaryAction": null
      }
    ]
  },
  "error": null
}
```

---

## 8. GET /api/v1/project-space/{projectId}/risks

### 8.1 Success Response (200 OK)

```json
{
  "data": {
    "items": [
      {
        "id": "RISK-PRJ8821-2",
        "title": "Dependency circularity",
        "severity": "CRITICAL",
        "category": "DEPENDENCY",
        "owner": { "memberId": "u-003", "displayName": "Ada Lovelace" },
        "ageDays": 4,
        "latestNote": "identity-v2 <-> payments-core circular in build graph",
        "primaryAction": {
          "url": "/incidents?projectId=proj-8821",
          "enabled": true
        }
      },
      {
        "id": "RISK-PRJ8821-1",
        "title": "Latency spike in STAGING",
        "severity": "HIGH",
        "category": "TECHNICAL",
        "owner": { "memberId": "u-030", "displayName": "Linus Torvalds" },
        "ageDays": 1,
        "latestNote": "p99 2.3x baseline after v2.3.7",
        "primaryAction": {
          "url": "/incidents?projectId=proj-8821&kind=perf",
          "enabled": true
        }
      },
      {
        "id": "RISK-PRJ8821-3",
        "title": "Auth token expiry policy drift",
        "severity": "MEDIUM",
        "category": "SECURITY",
        "owner": null,
        "ageDays": 12,
        "latestNote": "Refresh policy drifted from platform default",
        "primaryAction": {
          "url": "/platform?view=config&projectId=proj-8821",
          "enabled": false
        }
      }
    ],
    "total": 3,
    "lastRefreshed": "2026-04-17T10:00:00Z"
  },
  "error": null
}
```

Items are pre-ordered server-side: `severity DESC` then `ageDays DESC`.

### 8.2 Empty State Example

```json
{
  "data": {
    "items": [],
    "total": 0,
    "lastRefreshed": "2026-04-17T10:00:00Z"
  },
  "error": null
}
```

---

## 9. GET /api/v1/project-space/{projectId}/environments

### 9.1 Success Response (200 OK)

```json
{
  "data": {
    "environments": [
      {
        "id": "ENV-DEV-PRJ8821",
        "label": "DEV",
        "kind": "DEV",
        "versionRef": "v2.4.0-SNAPSHOT",
        "buildId": "b-1921",
        "health": "GREEN",
        "gateStatus": "AUTO",
        "approver": null,
        "lastDeployedAt": "2026-04-17T08:12:00Z",
        "drift": {
          "band": "MAJOR",
          "commitDelta": 24,
          "sinceVersion": "v2.3.4",
          "description": "24 commits ahead of PROD"
        },
        "deploymentLink": {
          "url": "/deployment?projectId=proj-8821&envId=ENV-DEV-PRJ8821",
          "enabled": false
        }
      },
      {
        "id": "ENV-STAGING-PRJ8821",
        "label": "STAGING",
        "kind": "STAGING",
        "versionRef": "v2.3.7",
        "buildId": "b-1910",
        "health": "YELLOW",
        "gateStatus": "APPROVAL_REQUIRED",
        "approver": { "memberId": "u-008", "displayName": "Donald Knuth" },
        "lastDeployedAt": "2026-04-16T18:00:00Z",
        "drift": {
          "band": "MINOR",
          "commitDelta": 12,
          "sinceVersion": "v2.3.4",
          "description": "12 commits ahead of PROD"
        },
        "deploymentLink": {
          "url": "/deployment?projectId=proj-8821&envId=ENV-STAGING-PRJ8821",
          "enabled": false
        }
      },
      {
        "id": "ENV-PROD-PRJ8821",
        "label": "PROD",
        "kind": "PROD",
        "versionRef": "v2.3.4",
        "buildId": "b-1889",
        "health": "GREEN",
        "gateStatus": "APPROVAL_REQUIRED",
        "approver": { "memberId": "u-007", "displayName": "Grace Hopper" },
        "lastDeployedAt": "2026-04-10T09:30:00Z",
        "drift": null,
        "deploymentLink": {
          "url": "/deployment?projectId=proj-8821&envId=ENV-PROD-PRJ8821",
          "enabled": false
        }
      }
    ]
  },
  "error": null
}
```

---

## 10. Backend Implementation Notes

### 10.1 Controller Skeleton

```java
@RestController
@RequestMapping(ApiConstants.PROJECT_SPACE)
public class ProjectSpaceController {

    private final ProjectSpaceService service;
    private final ProjectAccessGuard accessGuard;

    public ProjectSpaceController(ProjectSpaceService service, ProjectAccessGuard accessGuard) {
        this.service = service;
        this.accessGuard = accessGuard;
    }

    @GetMapping("/{projectId}")
    public ApiResponse<ProjectSpaceAggregateDto> aggregate(
            @PathVariable @Pattern(regexp = "^proj-[a-z0-9\\-]+$") String projectId
    ) {
        accessGuard.check(projectId);
        return ApiResponse.ok(service.loadAggregate(projectId));
    }

    @GetMapping("/{projectId}/summary")
    public ApiResponse<ProjectSummaryDto> summary(@PathVariable String projectId) {
        accessGuard.check(projectId);
        return ApiResponse.ok(service.loadSummary(projectId));
    }

    // ... leadership, chain, milestones, dependencies, risks, environments
}
```

### 10.2 Service Layer

```java
@Service
public class ProjectSpaceService {

    private final List<ProjectSpaceProjection<?>> projections;
    private final Executor projectionExecutor;

    public ProjectSpaceAggregateDto loadAggregate(String projectId) {
        // Parallel fan-out with per-projection timeout (500ms)
        // Collect into ProjectSpaceAggregateDto with per-section SectionResultDto
    }
}
```

Each projection implements:

```java
public interface ProjectSpaceProjection<T> {
    String key();
    SectionResultDto<T> load(String projectId);
}
```

### 10.3 Access Guard

```java
@Component
public class ProjectAccessGuard {
    public void check(String projectId) {
        // Resolve current authenticated principal
        // Verify user has read access to projectId (usually via workspaceId membership)
        // Throw ProjectAccessDeniedException on failure.
        // Add a matching @ExceptionHandler in GlobalExceptionHandler so 403
        // responses stay inside the shared ApiResponse envelope.
    }
}
```

### 10.4 Package Location

Package-by-feature under `com.sdlctower.domain.projectspace`. Shared primitives reuse the existing `com.sdlctower.shared.dto.ApiResponse`, `com.sdlctower.shared.dto.SectionResultDto`, and `com.sdlctower.shared.ApiConstants`.

### 10.5 Migration

- `V9__create_project_space_tables.sql` — `milestones`, `project_dependencies`, `environments`, `deployments` tables
- `V10__add_project_id_to_risk_signals.sql` — extends `risk_signals` with a nullable `project_id` column for project-scoped filtering
- `V11__seed_project_space_data.sql` — seed data for local dev

Run via Flyway on app startup. No `ddl-auto: update`.

---

## 11. Frontend Implementation Notes

### 11.1 API Client

```typescript
// frontend/src/features/project-space/api/projectSpaceApi.ts

import { fetchJson } from '@/shared/api/client';
import type {
  ProjectSpaceAggregate,
  ProjectSummary,
  LeadershipOwnership,
  SdlcChainState,
  MilestoneHub,
  DependencyMap,
  RiskRegistry,
  EnvironmentMatrix,
} from '../types/aggregate';

const USE_MOCK = import.meta.env.DEV && !import.meta.env.VITE_USE_BACKEND;

export type ProjectSpaceCardKey =
  | 'summary' | 'leadership' | 'chain' | 'milestones'
  | 'dependencies' | 'risks' | 'environments';

const SECTION_PATHS: Record<ProjectSpaceCardKey, string> = {
  summary: 'summary',
  leadership: 'leadership',
  chain: 'chain',
  milestones: 'milestones',
  dependencies: 'dependencies',
  risks: 'risks',
  environments: 'environments',
};

export const projectSpaceApi = {
  async getAggregate(projectId: string): Promise<ProjectSpaceAggregate> {
    if (USE_MOCK) return (await import('../mock/aggregate.mock')).aggregate(projectId);
    return fetchJson<ProjectSpaceAggregate>(`/project-space/${projectId}`);
  },

  async getSummary(projectId: string): Promise<ProjectSummary> {
    if (USE_MOCK) return (await import('../mock/summary.mock')).summary(projectId);
    return fetchJson<ProjectSummary>(`/project-space/${projectId}/summary`);
  },

  async getSection<K extends ProjectSpaceCardKey>(
    cardKey: K,
    projectId: string,
  ): Promise<unknown> {
    return fetchJson(`/project-space/${projectId}/${SECTION_PATHS[cardKey]}`);
  },
};
```

### 11.2 Pinia Store

```typescript
// frontend/src/features/project-space/stores/projectSpaceStore.ts

import { defineStore } from 'pinia';
import { ref } from 'vue';
import { projectSpaceApi, type ProjectSpaceCardKey } from '../api/projectSpaceApi';
import type { ProjectSpaceAggregate } from '../types/aggregate';

export const useProjectSpaceStore = defineStore('projectSpace', () => {
  const projectId = ref<string | null>(null);
  const workspaceId = ref<string | null>(null);
  const aggregate = ref<ProjectSpaceAggregate | null>(null);
  const isLoading = ref(false);
  const error = ref<string | null>(null);
  const loadingCards = ref<Record<ProjectSpaceCardKey, boolean>>({
    summary: false, leadership: false, chain: false, milestones: false,
    dependencies: false, risks: false, environments: false,
  });

  async function initProject(nextProjectId: string) {
    projectId.value = nextProjectId;
    isLoading.value = true;
    error.value = null;
    try {
      const result = await projectSpaceApi.getAggregate(nextProjectId);
      aggregate.value = result;
      workspaceId.value = result.workspaceId;
    } catch (err) {
      error.value = err instanceof Error ? err.message : 'Failed to load Project Space';
    } finally {
      isLoading.value = false;
    }
  }

  async function retryCard(cardKey: ProjectSpaceCardKey) {
    if (!projectId.value || !aggregate.value) return;
    loadingCards.value[cardKey] = true;
    try {
      const section = await projectSpaceApi.getSection(cardKey, projectId.value);
      (aggregate.value[cardKey] as unknown) = { data: section, error: null };
    } finally {
      loadingCards.value[cardKey] = false;
    }
  }

  return { projectId, workspaceId, aggregate, isLoading, error, loadingCards, initProject, retryCard };
});
```

### 11.3 Vite Proxy

```typescript
// vite.config.ts (reuses existing proxy block)
server: {
  proxy: {
    '/api': {
      target: 'http://localhost:8080',
      changeOrigin: true,
    },
  },
}
```

### 11.4 Mock Aggregate Skeleton

```typescript
// src/features/project-space/mock/aggregate.mock.ts

import type { ProjectSpaceAggregate } from '../types/aggregate';

export function aggregate(projectId: string): ProjectSpaceAggregate {
  return {
    projectId,
    workspaceId: 'ws-default-001',
    summary:      { data: { /* ... see §3 */ }, error: null },
    leadership:   { data: { /* ... see §4 */ }, error: null },
    chain:        { data: { /* ... see §5 */ }, error: null },
    milestones:   { data: { /* ... see §6 */ }, error: null },
    dependencies: { data: { /* ... see §7 */ }, error: null },
    risks:        { data: { /* ... see §8 */ }, error: null },
    environments: { data: { /* ... see §9 */ }, error: null },
  };
}
```

---

## 12. Testing Contracts

### 12.1 Backend (MockMvc)

```java
@WebMvcTest(ProjectSpaceController.class)
class ProjectSpaceControllerTest {

    @Autowired MockMvc mvc;
    @MockBean ProjectSpaceService service;
    @MockBean ProjectAccessGuard accessGuard;

    @Test void aggregate_returns200_withValidProject() throws Exception {
        when(service.loadAggregate("proj-8821")).thenReturn(sampleAggregate());
        mvc.perform(get("/api/v1/project-space/proj-8821"))
           .andExpect(status().isOk())
           .andExpect(jsonPath("$.data.projectId").value("proj-8821"))
           .andExpect(jsonPath("$.data.summary.data.id").value("proj-8821"))
           .andExpect(jsonPath("$.data.summary.error").isEmpty());
    }

    @Test void aggregate_returns400_withInvalidId() throws Exception {
        mvc.perform(get("/api/v1/project-space/INVALID"))
           .andExpect(status().isBadRequest())
           .andExpect(jsonPath("$.error").value(org.hamcrest.Matchers.containsString("Invalid projectId")));
    }

    @Test void aggregate_returns403_whenAccessDenied() throws Exception {
        doThrow(new ProjectAccessDeniedException("proj-other"))
            .when(accessGuard).check("proj-other");
        mvc.perform(get("/api/v1/project-space/proj-other"))
           .andExpect(status().isForbidden())
           .andExpect(jsonPath("$.error").value(org.hamcrest.Matchers.containsString("Project access denied")));
    }

    @Test void chain_alwaysReturns11Nodes() throws Exception {
        when(service.loadChain("proj-8821")).thenReturn(sampleChain());
        mvc.perform(get("/api/v1/project-space/proj-8821/chain"))
           .andExpect(status().isOk())
           .andExpect(jsonPath("$.data.nodes.length()").value(11));
    }
}
```

Also verify:

- Per-projection timeout yields `SectionResultDto.data == null` and a non-null `error` for that section only.
- Parallel fan-out: service completes within the aggregate total budget.
- Flyway V9 / V10 / V11 migrations run cleanly on H2.
- `risks.items` returns already-sorted by severity DESC, ageDays DESC.
- Version drift band computed correctly at thresholds 0 / 10 / 11.

### 12.2 Frontend (Vitest)

```typescript
describe('projectSpaceStore', () => {
  it('initProject populates all cards from aggregate', async () => {
    const store = useProjectSpaceStore();
    await store.initProject('proj-8821');
    expect(store.aggregate?.summary.data).not.toBeNull();
    expect(store.aggregate?.risks.error).toBeNull();
    expect(store.workspaceId).toBe('ws-default-001');
    expect(store.error).toBeNull();
  });

  it('switchProject resets and reloads', async () => {
    const store = useProjectSpaceStore();
    await store.initProject('proj-a');
    await store.initProject('proj-b');
    expect(store.projectId).toBe('proj-b');
  });

  it('retryCard recovers a single card without affecting others', async () => {
    // ...
  });
});

describe('SdlcDeepLinksCard', () => {
  it('always renders 11 nodes in canonical order', () => {
    // mount with shorter chain → still 11 rendered (missing default to UNKNOWN, disabled)
  });

  it('highlights Spec as execution hub', () => {
    // ...
  });

  it('renders disabled tiles with coming-soon tooltip when enabled=false', () => {
    // ...
  });
});
```

---

## 13. Versioning Policy

- Path-versioned at `/api/v1`. Breaking changes require a new version path.
- Backward-compatible additions: new optional fields, new enum values (frontend must gracefully handle unknown enum values).
- Frontend fails-safe: unknown enum values render as neutral chips; unknown card keys in aggregate are ignored.
- All breaking changes announced in repo release notes plus CLAUDE.md lessons learned section.
