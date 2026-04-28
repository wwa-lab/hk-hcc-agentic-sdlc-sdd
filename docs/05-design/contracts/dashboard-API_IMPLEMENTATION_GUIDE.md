# Dashboard API Implementation Guide

## Purpose

This document is the single source of truth for implementing the Dashboard API.
It defines endpoint contracts, request/response shapes, error handling, and
integration patterns for both frontend and backend developers.

## Traceability

- Spec: [dashboard-spec.md](../../03-spec/dashboard-spec.md) (§11)
- Architecture: [dashboard-architecture.md](../../04-architecture/dashboard-architecture.md) (§7)
- Design: [dashboard-design.md](../dashboard-design.md) (§6)
- Data Model: [dashboard-data-model.md](../../04-architecture/dashboard-data-model.md)
- Data Flow: [dashboard-data-flow.md](../../04-architecture/dashboard-data-flow.md)

---

## 1. API Overview

| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/dashboard/summary` | Fetch complete dashboard summary for current workspace |

### Base URL

- Local dev: `http://localhost:8080/api/v1`
- Frontend proxy (Vite): `/api/v1` → `http://localhost:8080/api/v1`

---

## 2. GET /api/v1/dashboard/summary

### 2.1 Request

```
GET /api/v1/dashboard/summary
Accept: application/json
```

**Query Parameters**: None (V1). Future: `workspaceId` will be derived from authentication context.

**Headers**: Standard HTTP headers. No authentication required in V1.

### 2.2 Success Response (200 OK)

The response uses a two-level envelope:
1. **Top level**: `ApiResponse<T>` — shared envelope from `com.sdlctower.shared.dto`
2. **Section level**: `SectionResult<T>` — per-card error isolation

```json
{
  "data": {
    "sdlcHealth": {
      "data": [
        {
          "key": "requirement",
          "label": "Requirement",
          "status": "healthy",
          "itemCount": 24,
          "isHub": false,
          "routePath": "/requirements"
        },
        {
          "key": "user-story",
          "label": "User Story",
          "status": "healthy",
          "itemCount": 67,
          "isHub": false,
          "routePath": "/requirements"
        },
        {
          "key": "spec",
          "label": "Spec",
          "status": "warning",
          "itemCount": 12,
          "isHub": true,
          "routePath": "/requirements"
        },
        {
          "key": "architecture",
          "label": "Architecture",
          "status": "healthy",
          "itemCount": 8,
          "isHub": false,
          "routePath": "/design"
        },
        {
          "key": "design",
          "label": "Design",
          "status": "healthy",
          "itemCount": 15,
          "isHub": false,
          "routePath": "/design"
        },
        {
          "key": "tasks",
          "label": "Tasks",
          "status": "healthy",
          "itemCount": 143,
          "isHub": false,
          "routePath": "/project-management"
        },
        {
          "key": "code",
          "label": "Code",
          "status": "healthy",
          "itemCount": 89,
          "isHub": false,
          "routePath": "/code"
        },
        {
          "key": "test",
          "label": "Test",
          "status": "warning",
          "itemCount": 34,
          "isHub": false,
          "routePath": "/testing"
        },
        {
          "key": "deploy",
          "label": "Deploy",
          "status": "healthy",
          "itemCount": 7,
          "isHub": false,
          "routePath": "/deployment"
        },
        {
          "key": "incident",
          "label": "Incident",
          "status": "critical",
          "itemCount": 3,
          "isHub": false,
          "routePath": "/incidents"
        },
        {
          "key": "learning",
          "label": "Learning",
          "status": "inactive",
          "itemCount": 0,
          "isHub": false,
          "routePath": "/ai-center"
        }
      ],
      "error": null
    },
    "deliveryMetrics": {
      "data": {
        "leadTime": {
          "label": "Lead Time",
          "value": "4.2d",
          "trend": "down",
          "trendIsPositive": true
        },
        "deployFrequency": {
          "label": "Deploy Frequency",
          "value": "3.1/wk",
          "trend": "up",
          "trendIsPositive": true
        },
        "iterationCompletion": {
          "label": "Iteration Completion",
          "value": "87%",
          "trend": "up",
          "trendIsPositive": true
        },
        "bottleneckStage": "spec"
      },
      "error": null
    },
    "aiParticipation": {
      "data": {
        "usageRate": {
          "label": "Usage Rate",
          "value": "78%",
          "trend": "up",
          "trendIsPositive": true
        },
        "adoptionRate": {
          "label": "Adoption Rate",
          "value": "64%",
          "trend": "up",
          "trendIsPositive": true
        },
        "autoExecSuccess": {
          "label": "Auto-Exec Success",
          "value": "92%",
          "trend": "stable",
          "trendIsPositive": true
        },
        "timeSaved": {
          "label": "Time Saved",
          "value": "124h",
          "trend": "up",
          "trendIsPositive": true
        },
        "stageInvolvement": [
          { "stageKey": "requirement", "involved": true, "actionsCount": 45 },
          { "stageKey": "user-story", "involved": true, "actionsCount": 128 },
          { "stageKey": "spec", "involved": true, "actionsCount": 342 },
          { "stageKey": "architecture", "involved": true, "actionsCount": 12 },
          { "stageKey": "design", "involved": true, "actionsCount": 56 },
          { "stageKey": "tasks", "involved": true, "actionsCount": 89 },
          { "stageKey": "code", "involved": true, "actionsCount": 567 },
          { "stageKey": "test", "involved": true, "actionsCount": 234 },
          { "stageKey": "deploy", "involved": true, "actionsCount": 45 },
          { "stageKey": "incident", "involved": false, "actionsCount": 0 },
          { "stageKey": "learning", "involved": true, "actionsCount": 23 }
        ]
      },
      "error": null
    },
    "qualityMetrics": {
      "data": {
        "buildSuccessRate": {
          "label": "Build Success",
          "value": "98.4%",
          "trend": "up",
          "trendIsPositive": true
        },
        "testPassRate": {
          "label": "Test Pass Rate",
          "value": "99.1%",
          "trend": "stable",
          "trendIsPositive": true
        },
        "defectDensity": {
          "label": "Defect Density",
          "value": "0.42/kloc",
          "trend": "down",
          "trendIsPositive": true
        },
        "specCoverage": {
          "label": "Spec Coverage",
          "value": "84%",
          "trend": "up",
          "trendIsPositive": true
        }
      },
      "error": null
    },
    "stabilityMetrics": {
      "data": {
        "activeIncidents": 3,
        "criticalIncidents": 1,
        "changeFailureRate": {
          "label": "Change Failure",
          "value": "2.1%",
          "trend": "down",
          "trendIsPositive": true
        },
        "mttr": {
          "label": "MTTR",
          "value": "45m",
          "trend": "down",
          "trendIsPositive": true
        },
        "rollbackRate": {
          "label": "Rollback Rate",
          "value": "0.5%",
          "trend": "stable",
          "trendIsPositive": true
        }
      },
      "error": null
    },
    "governanceMetrics": {
      "data": {
        "templateReuse": {
          "label": "Template Reuse",
          "value": "82%",
          "trend": "up",
          "trendIsPositive": true
        },
        "configDrift": {
          "label": "Config Drift",
          "value": "3.1%",
          "trend": "down",
          "trendIsPositive": true
        },
        "auditCoverage": {
          "label": "Audit Coverage",
          "value": "91%",
          "trend": "up",
          "trendIsPositive": true
        },
        "policyHitRate": {
          "label": "Policy Hit Rate",
          "value": "97%",
          "trend": "stable",
          "trendIsPositive": true
        }
      },
      "error": null
    },
    "recentActivity": {
      "data": {
        "entries": [
          {
            "id": "1",
            "actor": "Gemini Agent",
            "actorType": "ai",
            "action": "Generated Spec for Feature #452",
            "stageKey": "spec",
            "timestamp": "2026-04-16T09:55:00Z"
          },
          {
            "id": "2",
            "actor": "Sarah Chen",
            "actorType": "human",
            "action": "Reviewed Architecture Design",
            "stageKey": "architecture",
            "timestamp": "2026-04-16T09:45:00Z"
          }
        ],
        "total": 1542
      },
      "error": null
    },
    "valueStory": {
      "data": {
        "headline": "AI agents have reduced lead time by 35% this month while maintaining 100% audit compliance.",
        "metrics": [
          {
            "label": "Efficiency Gain",
            "value": "+42%",
            "description": "Increase in developer throughput vs last quarter"
          },
          {
            "label": "Quality Lift",
            "value": "-28%",
            "description": "Reduction in post-release defects"
          },
          {
            "label": "Risk Reduction",
            "value": "High",
            "description": "Automated policy enforcement on all specs"
          }
        ]
      },
      "error": null
    }
  },
  "error": null
}
```

### 2.3 Section-Level Partial Failure (200 OK)

When individual sections fail, the top-level response still returns 200.
Only the failed section has `error` set; other sections remain valid.

```json
{
  "data": {
    "sdlcHealth": { "data": [...], "error": null },
    "deliveryMetrics": { "data": {...}, "error": null },
    "qualityMetrics": { "data": null, "error": "Timeout querying quality metrics" },
    "stabilityMetrics": { "data": {...}, "error": null }
  },
  "error": null
}
```

### 2.4 Top-Level Error Response

When the entire request fails (auth, network, server crash), the top-level
`error` is set and `data` is null.

```json
{
  "data": null,
  "error": "Failed to load dashboard summary"
}
```

| HTTP Status | Meaning | Frontend Handling |
|-------------|---------|-------------------|
| 200 | Success (may include partial section failures) | Render available sections; show error state for failed sections |
| 401 | Unauthorized (future) | Redirect to login |
| 403 | Forbidden (future) | Show access denied |
| 500 | Server error | Show full-page error state |

---

## 3. Response Schema Reference

### 3.1 SectionResult Envelope

Every section in `DashboardSummary` is wrapped in this envelope:

```typescript
interface SectionResult<T> {
  data: T | null;     // Present on success
  error: string | null; // Present on failure
}
```

**Invariant**: Exactly one of `data` or `error` is non-null.

### 3.2 MetricValue (Reused Across Sections)

```typescript
interface MetricValue {
  label: string;            // Human-readable label
  value: string;            // Formatted display value (e.g., "4.2d", "87%")
  trend: 'up' | 'down' | 'stable';
  trendIsPositive: boolean; // Whether trend direction is good for this metric
}
```

**Note**: `value` is a pre-formatted string, not a raw number. The backend formats
the value for display. The frontend renders it as-is.

### 3.3 SDLC Stage Keys (Enumeration)

```
requirement | user-story | spec | architecture | design | tasks | code | test | deploy | incident | learning
```

Always exactly 11 stages in this order. The `spec` stage has `isHub: true`.

### 3.4 Route Map

| Stage Key | Route Path |
|-----------|-----------|
| requirement | `/requirements` |
| user-story | `/requirements` |
| spec | `/requirements` |
| architecture | `/design` |
| design | `/design` |
| tasks | `/project-management` |
| code | `/code` |
| test | `/testing` |
| deploy | `/deployment` |
| incident | `/incidents` |
| learning | `/ai-center` |

---

## 4. Backend Implementation Guide

### 4.1 Controller

```java
package com.sdlctower.domain.dashboard;

import com.sdlctower.shared.dto.ApiResponse;
import com.sdlctower.domain.dashboard.dto.DashboardSummaryDto;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
@RequestMapping("/api/v1/dashboard")
public class DashboardController {

    private final DashboardService dashboardService;

    public DashboardController(DashboardService dashboardService) {
        this.dashboardService = dashboardService;
    }

    @GetMapping("/summary")
    public ApiResponse<DashboardSummaryDto> getSummary() {
        DashboardSummaryDto summary = dashboardService.getDashboardSummary();
        return ApiResponse.ok(summary);
    }
}
```

### 4.2 Service

```java
package com.sdlctower.domain.dashboard;

import com.sdlctower.domain.dashboard.dto.*;
import org.springframework.stereotype.Service;
import java.util.List;

@Service
public class DashboardService {

    /**
     * Returns a complete dashboard summary.
     * V1: Returns hardcoded seed data.
     * V2+: Will aggregate from domain tables.
     */
    public DashboardSummaryDto getDashboardSummary() {
        return new DashboardSummaryDto(
            SectionResultDto.ok(buildSdlcHealth()),
            SectionResultDto.ok(buildDeliveryMetrics()),
            SectionResultDto.ok(buildAiParticipation()),
            SectionResultDto.ok(buildQualityMetrics()),
            SectionResultDto.ok(buildStabilityMetrics()),
            SectionResultDto.ok(buildGovernanceMetrics()),
            SectionResultDto.ok(buildRecentActivity()),
            SectionResultDto.ok(buildValueStory())
        );
    }

    // Each build*() method constructs seed data DTO.
    // See dashboard-data-model.md §3 for complete DTO definitions.
}
```

### 4.3 Package Structure

```
backend/src/main/java/com/sdlctower/
├── domain/
│   └── dashboard/
│       ├── DashboardController.java
│       ├── DashboardService.java
│       └── dto/
│           ├── DashboardSummaryDto.java
│           ├── SectionResultDto.java
│           ├── SdlcStageHealthDto.java
│           ├── MetricValueDto.java
│           ├── DeliveryMetricsDto.java
│           ├── AiParticipationDto.java
│           ├── AiInvolvementDto.java
│           ├── QualityMetricsDto.java
│           ├── StabilityMetricsDto.java
│           ├── GovernanceMetricsDto.java
│           ├── ActivityEntryDto.java
│           ├── RecentActivityDto.java
│           ├── ValueStoryDto.java
│           └── ValueStoryMetricDto.java
```

---

## 5. Frontend Integration Guide

### 5.1 API Client

```typescript
// frontend/src/features/dashboard/api/dashboardApi.ts
import { fetchJson } from '@/shared/api/client';
import type { DashboardSummary } from '../types/dashboard';

export const dashboardApi = {
  async getSummary(): Promise<DashboardSummary> {
    return fetchJson<DashboardSummary>('/dashboard/summary');
  }
};
```

### 5.2 Store Integration

```typescript
// frontend/src/features/dashboard/stores/dashboardStore.ts
import { defineStore } from 'pinia';
import { ref } from 'vue';
import type { DashboardSummary } from '../types/dashboard';
import { dashboardApi } from '../api/dashboardApi';
import { MOCK_DASHBOARD_DATA } from '../mockData';

const USE_MOCK = import.meta.env.DEV && !import.meta.env.VITE_USE_BACKEND;

export const useDashboardStore = defineStore('dashboard', () => {
  const summary = ref<DashboardSummary | null>(null);
  const isLoading = ref(false);
  const error = ref<string | null>(null);

  async function fetchSummary() {
    isLoading.value = true;
    error.value = null;
    try {
      summary.value = USE_MOCK
        ? MOCK_DASHBOARD_DATA
        : await dashboardApi.getSummary();
    } catch (err) {
      console.error('Failed to fetch dashboard summary:', err);
      error.value = 'Failed to load dashboard data. Please try again later.';
    } finally {
      isLoading.value = false;
    }
  }

  return { summary, isLoading, error, fetchSummary };
});
```

### 5.3 Vite Proxy Configuration

```typescript
// vite.config.ts (add to existing config)
export default defineConfig({
  server: {
    proxy: {
      '/api': {
        target: 'http://localhost:8080',
        changeOrigin: true
      }
    }
  }
});
```

---

## 6. Testing Contracts

### 6.1 Backend Tests (MockMvc)

```java
@SpringBootTest
@AutoConfigureMockMvc
@ActiveProfiles("local")
class DashboardControllerTest {

    @Autowired MockMvc mockMvc;

    @Test
    void getDashboardSummaryReturnsWrappedSummary() throws Exception {
        mockMvc.perform(get(ApiConstants.DASHBOARD_SUMMARY))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.data").exists())
            .andExpect(jsonPath("$.data.sdlcHealth.data.length()").value(11))
            .andExpect(jsonPath("$.data.sdlcHealth.data[2].key").value("spec"))
            .andExpect(jsonPath("$.data.sdlcHealth.data[2].isHub").value(true))
            .andExpect(jsonPath("$.data.sdlcHealth.data[9].status").value("critical"))
            .andExpect(jsonPath("$.data.deliveryMetrics.data.leadTime.label").value("Lead Time"))
            .andExpect(jsonPath("$.data.recentActivity.data.entries[0].actorType").value("ai"))
            .andExpect(jsonPath("$.data.valueStory.data.headline").exists());
    }
}
```

### 6.2 Frontend Assertions

After loading the dashboard:
- `summary.sdlcHealth.data` has exactly 11 items
- `summary.sdlcHealth.data` contains one item with `isHub === true` and `key === 'spec'`
- All `MetricValue` fields have non-empty `label`, `value`, and valid `trend`
- `summary.recentActivity.data.entries` has at most 10 items
- All timestamps are valid ISO 8601 strings

---

## 7. Versioning and Evolution

| Version | Changes |
|---------|---------|
| V1 (current) | Single endpoint, seed data, no auth |
| V2 | Real data aggregation from domain tables |
| V3 | Workspace-scoped with auth; query param for time range |
| V4 | WebSocket push for real-time updates |

### Breaking Change Policy

- New fields can be added to response without version bump
- Removing or renaming fields requires a new API version
- Section-level changes within `SectionResult` are backwards-compatible
