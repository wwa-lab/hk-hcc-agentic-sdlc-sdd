# Dashboard / Control Tower Spec

## Purpose

This spec defines the implementation-facing contract for the Dashboard / Control
Tower page — the main landing page of the product.

## 1. Scope

In scope:

- SDLC chain health visualization component
- delivery metrics card
- AI participation summary card
- quality and testing card
- stability and incident card
- governance trust card
- recent activity stream
- value proof narrative card
- navigation links to downstream pages
- mocked data contracts for Phase A
- backend API contracts for Phase B

Out of scope:

- drill-down detail views
- report export
- real-time WebSocket push
- role-based widget visibility
- widget reordering

## 2. Page Layout Contract

The dashboard renders inside the shared app shell. The content area uses a
card-based grid layout with the following regions:

```
+----------------------------------------------+
| SDLC Chain Health (full width)               |
+----------------------------------------------+
| Delivery Metrics  | AI Participation         |
+-------------------+--------------------------+
| Quality & Testing | Stability & Incidents    |
+-------------------+--------------------------+
| Governance Trust  | Value Story              |
+-------------------+--------------------------+
| Recent Activity Stream (full width)          |
+----------------------------------------------+
```

All cards must support loading, empty, error, and normal states.

## 3. SDLC Chain Contract

### 3.1 Data Shape

```ts
interface SdlcStageHealth {
  readonly key: string;           // e.g., 'requirement', 'spec', 'code'
  readonly label: string;         // display label
  readonly status: 'healthy' | 'warning' | 'critical' | 'inactive';
  readonly itemCount: number;     // items currently in this stage
  readonly isHub: boolean;        // true for Spec node
  readonly routePath: string;     // navigation target
}
```

### 3.2 Display Rules

- All 11 stages rendered as a horizontal pipeline
- Each stage shows status via color-coded indicator
- Spec node is visually distinguished (larger, highlighted border, "hub" badge)
- Item count shown below each stage
- Each stage is clickable and navigates to the corresponding module page
- Stages for unimplemented modules navigate to placeholder page

### 3.3 Stage Keys

```
requirement | user-story | spec | architecture | design | tasks | code | test | deploy | incident | learning
```

## 4. Delivery Metrics Contract

### 4.1 Data Shape

```ts
interface DeliveryMetrics {
  readonly leadTime: MetricValue;
  readonly deployFrequency: MetricValue;
  readonly iterationCompletion: MetricValue;
  readonly bottleneckStage: string | null;   // stage key or null
}

interface MetricValue {
  readonly label: string;
  readonly value: string;                     // formatted display value
  readonly trend: 'up' | 'down' | 'stable';
  readonly trendIsPositive: boolean;          // true if trend direction is good
}
```

### 4.2 Display Rules

- Each metric shows: label, value, trend arrow with color
- Trend arrow is green when `trendIsPositive`, red when not
- Bottleneck stage highlighted in the SDLC chain (if applicable)

## 5. AI Participation Contract

### 5.1 Data Shape

```ts
interface AiParticipation {
  readonly usageRate: MetricValue;
  readonly adoptionRate: MetricValue;
  readonly autoExecSuccess: MetricValue;
  readonly timeSaved: MetricValue;
  readonly stageInvolvement: ReadonlyArray<{
    readonly stageKey: string;
    readonly involved: boolean;
    readonly actionsCount: number;
  }>;
}
```

### 5.2 Display Rules

- Headline metrics shown as metric cards with trend
- Stage involvement shown as small indicators per SDLC stage
- AI sections use a distinct accent color (e.g., AI teal/cyan)

## 6. Quality & Testing Contract

### 6.1 Data Shape

```ts
interface QualityMetrics {
  readonly buildSuccessRate: MetricValue;
  readonly testPassRate: MetricValue;
  readonly defectDensity: MetricValue;
  readonly specCoverage: MetricValue;
}
```

## 7. Stability & Incident Contract

### 7.1 Data Shape

```ts
interface StabilityMetrics {
  readonly activeIncidents: number;
  readonly criticalIncidents: number;
  readonly changeFailureRate: MetricValue;
  readonly mttr: MetricValue;
  readonly rollbackRate: MetricValue;
}
```

### 7.2 Display Rules

- Active incidents badge with severity color
- Critical incidents flagged prominently
- Incident count is clickable and navigates to Incident Management

## 8. Governance Trust Contract

### 8.1 Data Shape

```ts
interface GovernanceMetrics {
  readonly templateReuse: MetricValue;
  readonly configDrift: MetricValue;
  readonly auditCoverage: MetricValue;
  readonly policyHitRate: MetricValue;
}
```

### 8.2 Display Rules

- Non-compliant values flagged with warning color
- Clickable link to Platform Center for each indicator

## 9. Recent Activity Contract

### 9.1 Data Shape

```ts
interface ActivityEntry {
  readonly id: string;
  readonly actor: string;
  readonly actorType: 'ai' | 'human';
  readonly action: string;
  readonly stageKey: string;
  readonly timestamp: string;              // ISO 8601
}

interface RecentActivity {
  readonly entries: ReadonlyArray<ActivityEntry>;
  readonly total: number;
}
```

### 9.2 Display Rules

- Show most recent 10 entries, chronological (newest first)
- AI actions distinguished with an AI icon/badge
- Each entry shows: actor, action, stage, relative timestamp
- "View All" link at bottom navigates to `/platform` (audit section)

## 10. Value Story Contract

### 10.1 Data Shape

```ts
interface ValueStory {
  readonly headline: string;
  readonly metrics: ReadonlyArray<{
    readonly label: string;
    readonly value: string;
    readonly description: string;
  }>;
}
```

### 10.2 Display Rules

- Headline text summarizing AI-SDLC value
- 3-4 key proof metrics with descriptions
- Integrated into the dashboard grid, not a separate promotional banner

## 11. Dashboard Aggregate API (Phase B)

### 11.1 Endpoint

```
GET /api/v1/dashboard/summary
```

### 11.2 Response Shape

Each section is wrapped in a `SectionResult` envelope so that individual
sections can fail independently while others succeed. This reconciles the
single-API design with the requirement that cards fail in isolation.

```json
{
  "data": {
    "sdlcHealth":        { "data": [ ... ], "error": null },
    "deliveryMetrics":   { "data": { ... }, "error": null },
    "aiParticipation":   { "data": { ... }, "error": null },
    "qualityMetrics":    { "data": { ... }, "error": null },
    "stabilityMetrics":  { "data": { ... }, "error": null },
    "governanceMetrics": { "data": { ... }, "error": null },
    "recentActivity":    { "data": { ... }, "error": null },
    "valueStory":        { "data": { ... }, "error": null }
  },
  "error": null
}
```

```ts
interface SectionResult<T> {
  readonly data: T | null;
  readonly error: string | null;
}
```

If the top-level `error` is non-null, the entire request failed (e.g.,
auth failure, network error). If a section-level `error` is non-null,
only that section failed and other sections remain usable.

### 11.3 Behavior

- Response is scoped to current workspace (workspace ID from context)
- Returns mocked seed data in V1
- Uses the standard `ApiResponse<T>` envelope at the top level
- Each section uses `SectionResult<T>` for independent error reporting

## 12. Error and Empty State Rules

| State | Behavior |
|-------|----------|
| Loading | Skeleton cards with pulse animation |
| Normal | All cards render with data |
| Empty | Card shows "No data available" with contextual message |
| Section error | Failed card shows error message; other cards still render |
| Top-level error | All cards show error state (e.g., auth failure) |

Each card reads its own `SectionResult`. A section-level failure does not
cascade to other cards.

## 13. Navigation Links

| Dashboard Element | Navigation Target |
|-------------------|-------------------|
| SDLC chain stage node | Corresponding module page (see route map below) |
| Incident count badge | `/incidents` |
| Governance indicator | `/platform` |
| Activity "View All" | `/platform` |

### 13.1 SDLC Stage Route Map

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

## 14. Exit Criteria

This spec is ready for implementation when:

- SDLC chain stage keys and labels are accepted
- Metric card data shape is accepted
- Dashboard layout grid is accepted
- Phase A mock data structure is accepted
- Phase B API endpoint path is accepted
