# AI Center — Design

## Purpose

Concrete design decisions that Codex (FE + BE) will implement. Covers file structure, component contracts, API endpoint signatures, database schema decisions, visual tokens, and error/empty state design.

## Source

- [ai-center-requirements.md](../01-requirements/ai-center-requirements.md)
- [ai-center-spec.md](../03-spec/ai-center-spec.md)
- [ai-center-architecture.md](../04-architecture/ai-center-architecture.md)
- [ai-center-data-model.md](../04-architecture/ai-center-data-model.md)
- [ai-center-API_IMPLEMENTATION_GUIDE.md](contracts/ai-center-API_IMPLEMENTATION_GUIDE.md)
- Visual design system: `../../design.md` (project root)

---

## 1. File Structure

### 1.1 Frontend

```
frontend/src/features/ai-center/
├── AiCenterView.vue
├── components/
│   ├── AdoptionMetricsCard.vue
│   ├── MetricTile.vue
│   ├── StageCoverageCard.vue
│   ├── StageCoverageChain.vue
│   ├── SkillCatalogCard.vue
│   ├── SkillRow.vue
│   ├── SkillFilters.vue
│   ├── SkillDetailPanel.vue
│   ├── RunHistoryCard.vue
│   ├── RunRow.vue
│   ├── RunFilters.vue
│   └── RunDetailPanel.vue
├── composables/
│   ├── useSkillFilters.ts
│   ├── useRunFilters.ts
│   └── useCardState.ts          # shared Normal/Loading/Empty/Error helper
├── stores/
│   └── aiCenterStore.ts
├── api/
│   ├── aiCenterApi.ts
│   └── mocks.ts
├── types.ts
├── constants.ts                  # stage labels, autonomy tooltips
└── index.ts                      # barrel + router entry
```

Router addition (`frontend/src/router/index.ts` — verify if already there; Dashboard slice already references `/ai-center`):

```ts
{
  path: "/ai-center",
  component: () => import("@/features/ai-center/AiCenterView.vue"),
  children: [
    { path: "skills/:skillKey", component: () => import("@/features/ai-center/components/SkillDetailPanel.vue") },
    { path: "runs/:executionId", component: () => import("@/features/ai-center/components/RunDetailPanel.vue") }
  ]
}
```

### 1.2 Backend

```
backend/src/main/java/com/sdlctower/domain/aicenter/
├── AiCenterController.java
├── service/
│   ├── SkillService.java
│   ├── SkillExecutionService.java
│   └── MetricsService.java
├── repository/
│   ├── SkillRepository.java
│   ├── SkillExecutionRepository.java
│   └── PolicyRepository.java
├── entity/
│   ├── Skill.java
│   ├── SkillStage.java
│   ├── Policy.java
│   ├── SkillExecution.java
│   └── EvidenceLink.java
├── dto/                          # per data-model doc §3
└── exception/
    ├── SkillNotFoundException.java
    └── SkillExecutionNotFoundException.java

backend/src/main/resources/db/migration/
├── V{n}__ai_center_schema.sql
└── V{n+1}__ai_center_seed.sql    # profile-gated via Flyway locations
```

> Java package name uses `aicenter` (no hyphen) even though the directory is written as `ai-center` elsewhere — Java packages do not allow hyphens. All documentation prose uses `ai-center` for readability.

---

## 2. Component API Contracts

### 2.1 `AiCenterView.vue`

- **Props**: none (route-level view)
- **Reads**: `workspaceId` from shell's `workspaceContextStore`
- **Dispatches on mount**: `aiCenterStore.init(workspaceId)`
- **Layout**: vertical stack — `AdoptionMetricsCard` (row of 5 tiles + delta) → `StageCoverageCard` (11-node chain visual) → `SkillCatalogCard` → `RunHistoryCard`. Right side reserves space for the shell's AI Command Panel.
- **Child outlets**: `<router-view />` in a slide-over panel slot for skill / run detail routes

### 2.2 `AdoptionMetricsCard.vue`

```ts
interface Props {
  metrics: SectionResult<MetricsSummary>
  loading: boolean
}
```

Renders 5 `MetricTile`s in a grid. Per-section `SectionResult` is unwrapped per tile, so a single metric failing does not blank the whole card.

### 2.3 `MetricTile.vue`

```ts
interface Props {
  label: string
  value: SectionResult<MetricValue>
  icon?: string
}
```

### 2.4 `StageCoverageCard.vue`

```ts
interface Props {
  coverage: StageCoverage | null
  loading: boolean
  error: string | null
}
```

Uses the canonical 11-node chain labels from `constants.ts`. Covered stages render filled; uncovered render muted. Hover reveals skill count and names.

### 2.5 `SkillCatalogCard.vue`

```ts
interface Props {
  skills: Skill[]
  loading: boolean
  error: string | null
}
```

Owns the filter bar and table. Filters are controlled via `useSkillFilters` composable. Search is debounced 150ms (client-side only).

### 2.6 `SkillRow.vue`

```ts
interface Props {
  skill: Skill
  selected: boolean
}

// Emits
(e: "select", skillKey: string): void
```

Visual: key + name (monospace key, regular name), category chip, status chip, autonomy chip, owner, last-run relative time, 30-day success rate bar.

### 2.7 `SkillDetailPanel.vue`

Route-mounted at `/ai-center/skills/:skillKey`. Slide-over panel, 640px wide.

- Fetches `SkillDetail` from store on mount / route change
- Sections: Header (key/name/status/autonomy), Description, Input/Output Contract, Current Policy, Recent Runs (up to 10), Aggregate Metrics

### 2.8 `RunHistoryCard.vue`

```ts
interface Props {
  runs: Page<Run>
  loading: boolean
  error: string | null
}
```

Owns run-level filters (server-driven). Shows "Refresh" button, total count, current page indicator. "Load more" at bottom when `hasMore: true`.

### 2.9 `RunRow.vue`

```ts
interface Props {
  run: Run
  selected: boolean
}
(e: "select", executionId: string): void
```

Visual: relative start time, skill name, status chip, triggered-by icon (AI bot vs user), trigger-source page (as link chip), duration, outcome summary (single line, ellipsis).

### 2.10 `RunDetailPanel.vue`

Route-mounted at `/ai-center/runs/:executionId`. Slide-over panel, 720px wide.

- Sections: Header (skill + status + duration), Trigger Source (with "Go to source" button if `triggerSourceUrl`), Input Summary, Step Breakdown (timeline), Output Summary, Policy Trail, Evidence Links, Audit Record link

---

## 3. API Endpoint Signatures (summary — full in API guide)

| Method | Path | Query / Path Params | Response |
|---|---|---|---|
| GET | `/api/v1/ai-center/metrics` | `window=30d\|7d\|24h` | `ApiResponse<MetricsSummaryDto>` |
| GET | `/api/v1/ai-center/stage-coverage` | — | `ApiResponse<StageCoverageDto>` |
| GET | `/api/v1/ai-center/skills` | — | `ApiResponse<List<SkillDto>>` |
| GET | `/api/v1/ai-center/skills/{skillKey}` | path | `ApiResponse<SkillDetailDto>` or 404 |
| GET | `/api/v1/ai-center/runs` | `skillKey[]`, `status[]`, `triggerSourcePage`, `startedAfter`, `startedBefore`, `triggeredByType`, `page`, `size` | `ApiResponse<PageDto<RunDto>>` |
| GET | `/api/v1/ai-center/runs/{executionId}` | path | `ApiResponse<RunDetailDto>` or 404 |

All endpoints accept workspace context via header (reuse shell mechanism). No request bodies in V1.

---

## 4. Database Schema Decisions

See `ai-center-data-model.md` §5 for DDL. Additional design decisions:

- **Column name**: `skill.key` renamed to `skill_key_code VARCHAR(128)` to avoid Oracle reserved-word quoting. Entity field remains `key` (Java field, JSON field) via `@Column(name = "skill_key_code")`. JSON wire format continues to use `key`. This affects only the DDL, not TS/DTO contracts.
- **JSON storage**: CLOB with application-side Jackson (de)serialization. No `CLOB → JSON` columns — portable H2/Oracle.
- **Seed gating**: Flyway `locations: classpath:db/migration` + `classpath:db/seed` with `spring.flyway.locations` configured per profile. `seed` folder only included for `local` and `dev` profiles.
- **Cascade deletes**: `evidence_link` cascades with `skill_execution`. `skill_stage` and `policy` cascade with `skill`. `skill_execution` does NOT cascade with `skill` (historical runs kept for audit).

---

## 5. Visual Design Decisions

Per project `design.md` §2 (Tactical Command aesthetic).

### 5.1 Color Tokens (reused)

| Usage | Token |
|---|---|
| Success / active | `--color-status-ok` |
| Warning / beta | `--color-status-warning` |
| Critical / failed / rejected | `--color-status-critical` |
| Neutral / deprecated | `--color-text-muted` |
| Accent (primary CTAs, chip selection) | `--color-accent-crimson` |

Status chip → color mapping:

| Status | Token |
|---|---|
| `active` / `succeeded` | ok |
| `beta` / `pending_approval` / `running` | warning |
| `failed` / `rejected` / `rolled_back` | critical |
| `deprecated` / — | muted |

Autonomy chip:

| Level | Token | Style |
|---|---|---|
| L0-Manual | muted | outline |
| L1-Assist | warning | outline |
| L2-Auto-with-approval | ok | outline |
| L3-Auto | ok | filled |

### 5.2 Typography

- Skill keys: `font-mono`, `font-size-sm`, `letter-spacing: 0.02em`
- Skill names, run outcome summaries: `font-sans`, `font-size-base`
- Metric values: `font-size-2xl`, `font-weight-bold`
- Use the shared design tokens — do not introduce new fonts or sizes.

### 5.3 Density

- Table row height: 48px (matches Dashboard activity feed)
- Card padding: 16px; card gap: 16px
- Detail panel: 640–720px wide, full-height, slide-over from the right

### 5.4 Animation

- Card skeleton pulse: 1.2s ease-in-out, reused shared class
- Slide-over panel: 180ms cubic-bezier(.2,.8,.2,1) — reuse existing transition from shell if present
- Filter chip toggle: 120ms opacity + transform
- NO bespoke Lottie or heavy animations

### 5.5 Iconography

Use Lucide icons already available in the project:

| Purpose | Icon |
|---|---|
| Skill catalog | `library` |
| Run history | `activity` |
| Metrics | `gauge` |
| Policy | `shield` |
| Evidence | `paperclip` |
| Audit | `scroll-text` |
| AI trigger | `bot` |
| Human trigger | `user` |
| System trigger | `cog` |

---

## 6. State Design

### 6.1 Normal
All cards render with data; filter bars functional; refresh control enabled.

### 6.2 Loading
Each card independently renders a skeleton. Skeleton shape should roughly match final content (strip of tiles, table rows, etc.).

### 6.3 Empty

| Card | Empty message |
|---|---|
| Skill Catalog | "No skills registered in this workspace yet. Skills are registered via Platform Center → Skill Registry." (V1: Platform Center link may be non-clickable if that slice is not live.) |
| Run History | "No skill executions recorded in the selected time range." Shows filter reset CTA if filters are applied. |
| Adoption Metrics | "Not enough data yet — run some AI skills to see adoption." |
| Stage Coverage | Always shows all 11 stages; "uncovered" visual is itself the empty state indicator. |

### 6.4 Error

Card-level error shows:
- Inline banner with red accent
- One-line error message (from backend's `ApiResponse.error`)
- "Retry" button which re-calls only that card's endpoint

Page-level global error (e.g., network outage for all endpoints) shown as a banner at the top of the view with a single "Retry all" button.

### 6.5 Partial

Metrics card uses per-section `SectionResult` — individual tiles may show their own mini error with per-tile retry. Other cards are monolithic (one endpoint = one card state).

---

## 7. Accessibility

- All chips have `role="status"` and `aria-label` describing the value
- Autonomy level chips have tooltips accessible via keyboard (focus + aria-describedby)
- Filter chips are `<button type="button" aria-pressed="...">`
- Tables use proper `<thead>` / `<tbody>` / `<th scope="col">`
- Slide-over panels use `role="dialog"` with focus trap on open, focus restore on close
- All interactive controls reachable by Tab; visible focus ring

---

## 8. Integration Boundary Diagram

```mermaid
graph LR
  subgraph Shell [Shell owns]
    Layout[App Layout + top nav + right AI panel slot]
    WSStore[WorkspaceContext store]
    Router[Vue Router]
  end

  subgraph AICenter [ai-center feature owns]
    View[AiCenterView]
    Store[aiCenterStore]
    Api[aiCenterApi]
    Types[types.ts]
  end

  subgraph SharedBackend [ai-center backend owns]
    Endpoints[/api/v1/ai-center/*/]
    Entities[Skill / SkillExecution / Policy entities]
    Migrations[V{n}__ai_center_*.sql]
  end

  Layout --> View
  Router --> View
  WSStore -->|workspaceId prop| Store
  View --> Store
  Store --> Api
  Api --> Endpoints
  Endpoints --> Entities
  Entities --> Migrations
```

**Non-boundary (do not cross):**

- AI Center feature MUST NOT directly import from other feature folders (no cross-feature FE coupling).
- Consumer backend domains MUST NOT import AI Center JPA entities; go through service interfaces.

---

## 9. Route Map Summary

| URL | Component | State loaded |
|---|---|---|
| `/ai-center` | `AiCenterView` | all 4 card endpoints |
| `/ai-center/skills/:skillKey` | `AiCenterView` + `SkillDetailPanel` slide-over | 4 cards + skill detail |
| `/ai-center/runs/:executionId` | `AiCenterView` + `RunDetailPanel` slide-over | 4 cards + run detail |

Deep-linking any of these URLs from outside (e.g., Dashboard) renders the full page state.

---

## 10. Open Questions (resolved)

| Question | Resolution |
|---|---|
| Detail as new route or in-page panel? | **Slide-over panel mounted at a nested route** — preserves deep-linkability while keeping main page loaded behind. |
| Pagination vs infinite scroll on runs? | **"Load more" button** — simpler, cheaper, more predictable for an enterprise audience. |
| Stage coverage as chart or chain? | **11-node chain** — reuses Dashboard convention (PRD §10). |
| Policy view entry point? | **Inside Skill Detail** (no top-level "Policies" tab in V1). |
| Evidence rendering? | **List of links in V1**, no inline viewer (that's a later Report Center slice). |
| Java package hyphen? | **`aicenter` (no hyphen)** — Java reality; docs keep `ai-center`. |
