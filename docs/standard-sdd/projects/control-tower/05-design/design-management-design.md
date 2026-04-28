# Design Management Design

## Purpose

Concrete implementation design for the Design Management slice — file structure, component API contracts, visual decisions, data contracts with the backend, state / error / empty design, and database schema references. This doc is the bridge between [design-management-spec.md](../03-spec/design-management-spec.md), the architecture trio ([architecture](../04-architecture/design-management-architecture.md), [data flow](../04-architecture/design-management-data-flow.md), [data model](../04-architecture/design-management-data-model.md)), and the API contract in [contracts/design-management-API_IMPLEMENTATION_GUIDE.md](contracts/design-management-API_IMPLEMENTATION_GUIDE.md).

V1 is a **read-heavy lightweight viewer** over internal Stitch / HTML mocks. Write paths are narrow admin operations. Spec → Design traceability is the primary flow.

## Traceability

- Spec: [../03-spec/design-management-spec.md](../03-spec/design-management-spec.md)
- Architecture: [../04-architecture/design-management-architecture.md](../04-architecture/design-management-architecture.md)
- Data flow: [../04-architecture/design-management-data-flow.md](../04-architecture/design-management-data-flow.md)
- Data model: [../04-architecture/design-management-data-model.md](../04-architecture/design-management-data-model.md)
- Visual design system: [visual-design-system.md](visual-design-system.md)
- Sibling slice conventions: [project-management-design.md](project-management-design.md), [requirement-design.md](requirement-design.md)
- Stitch mocks to reference: [Control Tower.html](Control%20Tower.html), [Incident Command Center.html](Incident%20Command%20Center.html), [Platform Center.html](Platform%20Center.html), [Project Space.html](Project%20Space.html)

---

## 1. Frontend File Structure

```
frontend/src/features/design-management/
├── CatalogView.vue                             // route: /design-management
├── ArtifactViewerView.vue                      // route: /design-management/artifacts/:artifactId
├── ArtifactRawView.vue                         // route: /design-management/artifacts/:artifactId/raw (no shell chrome)
├── TraceabilityView.vue                        // route: /design-management/traceability
├── components/
│   ├── catalog/
│   │   ├── CatalogSummaryBar.vue
│   │   ├── CatalogFilterBar.vue
│   │   ├── CatalogList.vue
│   │   ├── CatalogRow.vue
│   │   ├── CatalogRowThumbnail.vue
│   │   └── CoverageBadge.vue
│   ├── viewer/
│   │   ├── ArtifactHeader.vue
│   │   ├── ArtifactPreviewFrame.vue             // sandboxed iframe wrapper
│   │   ├── LinkedSpecStrip.vue
│   │   ├── LinkedSpecChip.vue
│   │   ├── AiSummaryPanel.vue
│   │   ├── AiSummaryBullets.vue
│   │   ├── ChangeHistoryTimeline.vue
│   │   ├── ChangeHistoryRow.vue
│   │   └── ContextChipStrip.vue
│   ├── traceability/
│   │   ├── TraceabilitySummary.vue
│   │   ├── TraceabilityFilterBar.vue
│   │   ├── TraceabilitySpecList.vue
│   │   ├── TraceabilitySpecRow.vue
│   │   └── LinkerModal.vue                       // admin-only multi-select modal
│   ├── admin/
│   │   ├── RegisterArtifactModal.vue
│   │   ├── PublishVersionModal.vue
│   │   └── LifecycleStageDialog.vue
│   └── shared/
│       ├── DmCard.vue                           // wrapper with loading / error / empty states
│       ├── LifecycleChip.vue
│       ├── KindChip.vue
│       ├── AiAttributionBadge.vue
│       ├── AuthorsInline.vue
│       ├── VersionTag.vue
│       └── UnsavedChangesGuard.vue
├── composables/
│   ├── useLifecycleStateMachine.ts              // allowed-transitions helper
│   ├── useArtifactPolling.ts                    // AI summary 3s x 5 polling
│   ├── useCatalogFilters.ts                     // query-param filters
│   └── useDeepLinks.ts                          // cross-slice URL builders
├── stores/
│   └── designManagementStore.ts                 // Pinia store
├── api/
│   └── designManagementApi.ts                   // API client, mock-aware
├── types/
│   ├── enums.ts
│   ├── catalog.ts
│   ├── viewer.ts
│   ├── traceability.ts
│   └── requests.ts
├── mocks/
│   ├── catalogAggregate.mock.ts
│   ├── catalogSummary.mock.ts
│   ├── viewerAggregate.mock.ts
│   ├── aiSummary.mock.ts
│   ├── changeHistory.mock.ts
│   ├── traceability.mock.ts
│   └── commandLoop.ts                           // simulates DM_STALE_VERSION, PII triggers
└── __tests__/
    ├── useLifecycleStateMachine.spec.ts
    ├── CatalogFilterBar.spec.ts
    ├── LinkedSpecStrip.spec.ts
    ├── AiSummaryPanel.spec.ts
    └── ArtifactPreviewFrame.spec.ts
```

### Router wiring (excerpt)

```typescript
// frontend/src/router/index.ts
{
  path: '/design-management',
  component: () => import('@/features/design-management/CatalogView.vue'),
  meta: { title: 'Design Management', role: 'WORKSPACE_READ' }
},
{
  path: '/design-management/artifacts/:artifactId',
  component: () => import('@/features/design-management/ArtifactViewerView.vue'),
  meta: { title: 'Artifact Viewer', role: 'WORKSPACE_READ' },
  props: true
},
{
  path: '/design-management/artifacts/:artifactId/raw',
  component: () => import('@/features/design-management/ArtifactRawView.vue'),
  meta: { title: 'Artifact (Raw)', role: 'WORKSPACE_READ', bypassShell: true },
  props: true
},
{
  path: '/design-management/traceability',
  component: () => import('@/features/design-management/TraceabilityView.vue'),
  meta: { title: 'Design Traceability', role: 'WORKSPACE_READ' }
}
```

---

## 2. Component API Contracts

### 2.1 Catalog view

| Component | Props | Emits | Source |
|-----------|-------|-------|--------|
| `CatalogView` | none (reads `workspaceId` from shell store) | — | `store.catalogAggregate` |
| `CatalogSummaryBar` | `summary: SectionResult<CatalogSummary>` | `filter(field, value)` | `store.catalogAggregate.summary` |
| `CatalogFilterBar` | `filters: CatalogFilters` | `change(partial)` | url query params |
| `CatalogList` | `rows: SectionResult<CatalogRow[]>` | `open(artifactId)` | store |
| `CatalogRow` | `row: CatalogRow` | `open`, `contextmenu` | — |
| `CatalogRowThumbnail` | `url: string \| null`, `kind: ArtifactKind` | — | — |
| `CoverageBadge` | `status: CoverageStatus`, `tooltip?: string` | — | — |

### 2.2 Viewer view

| Component | Props | Emits | Source |
|-----------|-------|-------|--------|
| `ArtifactViewerView` | `artifactId: string` (route prop) | — | `store.viewerAggregate[artifactId]` |
| `ArtifactHeader` | `header: SectionResult<ArtifactHeader>`, `isAdmin: boolean` | `publishVersion`, `regenerateSummary`, `changeLifecycle`, `openRaw` | store |
| `ArtifactPreviewFrame` | `artifactId: string`, `sizeBytes: number`, `allowEmbed: boolean` | `embedBlocked` | direct API |
| `LinkedSpecStrip` | `links: SectionResult<LinkedSpec[]>`, `isAdmin: boolean` | `openSpec(specId)`, `unlink(specId)`, `linkNew` | store |
| `LinkedSpecChip` | `link: LinkedSpec` | `open`, `unlink` | — |
| `AiSummaryPanel` | `summary: SectionResult<AiSummary \| null>`, `isAdmin: boolean`, `pending: boolean` | `regenerate` | store |
| `AiSummaryBullets` | `items: string[]` | — | — |
| `ChangeHistoryTimeline` | `page: SectionResult<ChangeLogPage>` | `pageChange(n)` | store |
| `ChangeHistoryRow` | `entry: ChangeLogEntry` | `openSpec(specId)` | — |
| `ContextChipStrip` | `header: ArtifactHeader` | — | — |

### 2.3 Traceability view

| Component | Props | Emits | Source |
|-----------|-------|-------|--------|
| `TraceabilityView` | none | — | `store.traceability` |
| `TraceabilitySummary` | `stats: { total: number; missing: number; stale: number }` | — | store |
| `TraceabilityFilterBar` | `filters: TraceabilityFilters` | `change(partial)` | url query params |
| `TraceabilitySpecList` | `rows: SectionResult<TraceabilitySpecRow[]>` | `openLinker(specId)`, `openArtifact(artifactId)` | store |
| `TraceabilitySpecRow` | `row: TraceabilitySpecRow`, `isAdmin: boolean` | `openLinker`, `openArtifact` | — |
| `LinkerModal` | `specId: string`, `workspaceId: string`, `open: boolean` | `close`, `linked(specId, artifactIds)` | store |

### 2.4 Admin panels

| Component | Props | Emits | Source |
|-----------|-------|-------|--------|
| `RegisterArtifactModal` | `workspaceId: string`, `open: boolean` | `close`, `registered(artifactId)` | store |
| `PublishVersionModal` | `artifactId: string`, `prevVersionId: string`, `open: boolean` | `close`, `published(versionId)` | store |
| `LifecycleStageDialog` | `artifactId: string`, `fromStage: LifecycleStage`, `allowedTransitions: LifecycleStage[]` | `close`, `changed(toStage, reason)` | store |

### 2.5 Shared components

`DmCard.vue` — wraps a section and renders a canonical skeleton / empty / error state based on the `SectionResult<T>` envelope.

```vue
<template>
  <section class="dm-card" :aria-busy="status === 'LOADING'">
    <header class="dm-card__header">
      <slot name="title" />
      <slot name="actions" />
    </header>
    <div class="dm-card__body">
      <template v-if="status === 'LOADING'">
        <slot name="skeleton">
          <DmCardSkeleton />
        </slot>
      </template>
      <template v-else-if="status === 'EMPTY'">
        <slot name="empty">
          <DmCardEmpty :message="emptyMessage" :action="emptyAction" />
        </slot>
      </template>
      <template v-else-if="status === 'ERROR'">
        <DmCardError :error="error" @retry="$emit('retry')" />
      </template>
      <template v-else>
        <slot />
      </template>
    </div>
  </section>
</template>
```

`LifecycleChip.vue` — renders a colored chip for `DRAFT` (neutral grey), `READY_FOR_REVIEW` (amber), `APPROVED` (LED green), `DEPRECATED` (muted crimson).

`KindChip.vue` — renders a small icon-labeled chip for the artifact kind: page, component, flow, state.

`AiAttributionBadge.vue` — the platform AI sparkle glyph + "AI" text label required by REQ-DM-73.

---

## 3. State Management (Pinia)

### 3.1 Store shape

```typescript
// stores/designManagementStore.ts
export interface DesignManagementState {
  catalogAggregate: {
    rows: SectionResult<CatalogRow[]>
    summary: SectionResult<CatalogSummary>
    filters: CatalogFilters
  } | null

  viewerAggregate: Record<string, {
    header: SectionResult<ArtifactHeader>
    links: SectionResult<LinkedSpec[]>
    aiSummary: SectionResult<AiSummary | null>
    history: SectionResult<ChangeLogPage>
    aiPending: boolean
  }>

  traceability: {
    rows: SectionResult<TraceabilitySpecRow[]>
    filters: TraceabilityFilters
  } | null

  adminAction: {
    inFlight: boolean
    lastError: SectionError | null
  }
}

export const useDesignManagementStore = defineStore('design-management', {
  state: (): DesignManagementState => ({
    catalogAggregate: null,
    viewerAggregate: {},
    traceability: null,
    adminAction: { inFlight: false, lastError: null }
  }),
  actions: {
    async initCatalog(workspaceId: string, filters?: CatalogFilters) { /* ... */ },
    async refreshCatalogRows() { /* ... */ },
    async refreshCatalogSummary() { /* ... */ },
    async openArtifact(artifactId: string) { /* ... */ },
    async refreshViewer(artifactId: string) { /* ... */ },
    async refreshViewerCard(artifactId: string, card: 'header' | 'links' | 'aiSummary' | 'history') { /* ... */ },
    async initTraceability(workspaceId: string, filters?: TraceabilityFilters) { /* ... */ },
    async initTraceabilityForSpec(specId: string) { /* ... */ },
    async registerArtifact(request: RegisterArtifactRequest) { /* ... */ },
    async publishVersion(artifactId: string, request: PublishVersionRequest) { /* ... */ },
    async linkSpecs(artifactId: string, request: LinkSpecsRequest) { /* ... */ },
    async unlinkSpec(artifactId: string, specId: string) { /* ... */ },
    async regenerateAiSummary(artifactId: string) { /* triggers + starts polling */ },
    async changeLifecycle(artifactId: string, request: ChangeLifecycleRequest) { /* ... */ }
  }
})
```

### 3.2 Cross-store subscriptions

- Subscribes to `shellContextStore.workspaceId` — resets Catalog and Traceability on change
- Subscribes to `shellContextStore.projectId` — refreshes Catalog filter when set
- Registers three AI Command Panel actions on route activation via `shellUiStore.setAiPanelContent({ actions: [...] })`; tears down on deactivation

### 3.3 URL as source of truth

- `CatalogFilters` is reflected into the URL via `useCatalogFilters()` composable
- `TraceabilityFilters` is reflected via query params
- `?specId=...` on Traceability expands that Spec's row
- URL changes debounce at 200–300 ms before triggering backend calls

---

## 4. Visual Design Decisions

### 4.1 Tokens

Design Management inherits the Tactical Command token set from [visual-design-system.md](visual-design-system.md). No slice-specific tokens are introduced. Specifically:

- Crimson accents — `STALE` / `MISSING` coverage; orphan artifacts; over-limit warnings
- Amber accents — `PARTIAL` coverage; `READY_FOR_REVIEW` lifecycle chip; "AI Summary pending" chip
- LED green — `OK` coverage; `APPROVED` lifecycle chip
- Neutral grey — `DRAFT` lifecycle chip; empty-state placeholders
- JetBrains Mono — artifact IDs, Spec IDs, version tags, correlationId chips
- Inter — everything else

### 4.2 Card anatomy

Every surface uses the `DmCard` shell: header (title + optional actions) → body (content | skeleton | empty | error). Cards never render their own loading spinner outside the `DmCard` envelope.

### 4.3 Catalog row layout

- 48 × 48 preview thumbnail (left) — falls back to kind glyph
- Title block (center-left) — title, kind chip, lifecycle chip inline
- Project + authors block — small-muted row
- Right rail — linked-Spec count, worst-coverage badge, AI-summary chip, `lastUpdatedAt` relative time
- Entire row is clickable; no nested clickable controls inside the row except the overflow menu

### 4.4 Viewer layout (≥ 1280px)

```
+-------------------------------------------------------------+
| ArtifactHeader (title, chips, version, authors, actions)    |
+--------------------------------------+----------------------+
|                                      | LinkedSpecStrip       |
| ArtifactPreviewFrame                 |                       |
| (sandboxed iframe; ratio-aware)      +----------------------+
|                                      | AiSummaryPanel        |
|                                      |                       |
|                                      +----------------------+
|                                      | ChangeHistoryTimeline |
+--------------------------------------+----------------------+
| ContextChipStrip (bottom, breadcrumb-like)                   |
+-------------------------------------------------------------+
```

### 4.5 Viewer layout (1024–1279px)

Linked-Spec strip, AI Summary, Change History collapse into a right-side tabbed panel.

### 4.6 Viewer layout (<1024px)

Preview frame above; strip / AI / history become tabs under the preview. `<1024` is a best-effort degrade; `<768` is out of V1 scope.

### 4.7 Animation / motion

- Skeletons pulse at 1.2s — the shell-provided `DmCardSkeleton` is used as-is.
- Coverage badge color transitions fade over 200 ms on re-fetch.
- Admin modals (Register, Publish, Lifecycle) slide-up 160 ms with a 80 ms fade.
- No page-level transitions; route changes are immediate for enterprise feel.

### 4.8 Empty states

Each surface has a first-class empty state (per REQ-DM-71):

| Surface | Empty message | CTA |
|---------|--------------|-----|
| Catalog (0 artifacts, admin) | "No design artifacts registered yet." | "Register your first artifact" |
| Catalog (0 artifacts, non-admin) | "No design artifacts registered in this Workspace." | (none — passive) |
| Viewer LinkedSpecStrip (0 links, admin) | "This artifact doesn't cover any Spec yet." | "Link a Spec" |
| Viewer LinkedSpecStrip (0 links, non-admin) | "This artifact has no Spec links." | (none — passive) |
| Viewer AiSummaryPanel (no cache, admin) | "No summary yet." | "Generate summary" |
| Viewer AiSummaryPanel (no cache, non-admin) | "Summary not yet available." | (passive refresh) |
| Traceability (no missing) | "Every visible Spec has at least one linked design." | (none) |
| Traceability (Spec-scoped, 0 links) | "No design covers this Spec yet." | "Link a design" (admin) |

### 4.9 Error presentation

Errors render inside the `DmCard` envelope: crimson left-border (4 px), inline error code chip (monospace, small), human-readable message, correlationId chip, "Retry" primary action. Errors never hide the card's title/header.

### 4.10 AI attribution

Every AI-produced string is preceded by the `AiAttributionBadge`. No AI output may render without attribution.

---

## 5. Data Contracts (summary)

Full contracts in [contracts/design-management-API_IMPLEMENTATION_GUIDE.md](contracts/design-management-API_IMPLEMENTATION_GUIDE.md). Summary of the most frequently used:

### 5.1 Catalog

```
GET /api/v1/design-management/catalog?workspaceId={ws}&project=...&kind=...&coverage=...&q=...&sort=last_updated_desc
→ SectionResult<CatalogRow[]>

GET /api/v1/design-management/catalog/summary?workspaceId={ws}
→ SectionResult<CatalogSummary>
```

### 5.2 Viewer

```
GET /api/v1/design-management/artifacts/{id}
→ ViewerAggregateDto { header, links, aiSummary, history }

GET /api/v1/design-management/artifacts/{id}/preview
→ text/html (with conservative CSP)

GET /api/v1/design-management/artifacts/{id}/ai-summary
→ SectionResult<AiSummary | null>
```

### 5.3 Traceability

```
GET /api/v1/design-management/traceability?workspaceId={ws}&coverage=MISSING&project=...
→ SectionResult<TraceabilitySpecRow[]>

GET /api/v1/design-management/traceability?specId=SPEC-DASH-020
→ SectionResult<TraceabilitySpecRow>   (single-row focused response)
```

### 5.4 Admin

```
POST /api/v1/design-management/artifacts
POST /api/v1/design-management/artifacts/{id}/versions   (body includes prevVersionId)
POST /api/v1/design-management/artifacts/{id}/links
DELETE /api/v1/design-management/artifacts/{id}/links/{specId}
POST /api/v1/design-management/artifacts/{id}/ai-summary/regenerate
PATCH /api/v1/design-management/artifacts/{id}/lifecycle
```

---

## 6. Database Schema Reference

Schema DDL is defined in [../04-architecture/design-management-data-model.md](../04-architecture/design-management-data-model.md) §5 and shipped as Flyway migrations:

- `V30__design_management_artifact.sql`
- `V31__design_management_author.sql`
- `V32__design_management_version.sql`
- `V33__design_management_link.sql`
- `V34__design_management_ai_summary.sql`
- `V35__design_management_change_log.sql`
- `V36__design_management_seed.sql` (dev/local profile only)

Per CLAUDE.md Lesson #4 — Flyway only, no `ddl-auto`. Per-dialect folders for H2 vs Oracle as needed.

---

## 7. Integration Boundary

```mermaid
graph LR
  subgraph Shared App Shell
    AppShell[AppShell.vue]
    ContextBar[TopContextBar]
    AiPanel[AiCommandPanel]
    Nav[PrimaryNav]
  end

  subgraph Design Management
    CatalogView[CatalogView.vue]
    ViewerView[ArtifactViewerView.vue]
    TraceView[TraceabilityView.vue]
    Store[useDesignManagementStore]
    Api[designManagementApi]
  end

  subgraph Other Slices
    REQ[Requirement Management<br/>/requirement/specs/:id]
    PS[Project Space<br/>/project-space/:projectId]
    PM[Project Management<br/>/project-management/:projectId]
  end

  subgraph Backend
    DMApi[/api/v1/design-management/*]
    Platform[Platform Governance]
    Skill[AI Skill Runtime]
  end

  AppShell --> CatalogView
  AppShell --> ViewerView
  AppShell --> TraceView
  CatalogView --> Store
  ViewerView --> Store
  TraceView --> Store
  Store --> Api
  Api --> DMApi
  DMApi --> Platform
  DMApi --> Skill
  ViewerView --> REQ
  CatalogView --> PS
  TraceView --> REQ
  PM --> DMApi
  ContextBar --> Store
  AiPanel --> Store
```

---

## 8. Accessibility

- All interactive elements meet WCAG 2.1 AA contrast via Tactical Command tokens.
- `ArtifactPreviewFrame` exposes `title` attribute from the artifact's title.
- Coverage badges include both color and textual status (never color-only).
- Skeletons set `aria-busy="true"`; empty states set `role="status"`.
- Admin modals trap focus; Esc closes; primary action has visible focus ring.
- AI attribution badge includes `aria-label="AI-generated content"`.

---

## 9. Testing Strategy

### 9.1 Unit tests (Vitest)

- `useLifecycleStateMachine.spec.ts` — validates allowed-transition matrix
- `useCatalogFilters.spec.ts` — URL ↔ filter synchronization
- `CatalogRow.spec.ts` — renders each kind, lifecycle, coverage combination
- `LinkedSpecStrip.spec.ts` — OK / PARTIAL / STALE / MISSING / empty paths
- `AiSummaryPanel.spec.ts` — pending, error, ready, autonomy-gated states
- `ArtifactPreviewFrame.spec.ts` — size-cap branch, sandbox attrs assertion

### 9.2 Component tests

- `RegisterArtifactModal` — form validation, PII trigger mock path
- `PublishVersionModal` — stale-token branch
- `LinkerModal` — multi-select, 50-per-minute rate limit respected

### 9.3 Integration tests (backend)

- Projections return consistent `SectionResult<T>` envelopes
- `DesignAccessGuard` rejects cross-Workspace access
- `PiiScanner` hard-rejects canonical PII payloads
- Lifecycle state machine blocks invalid transitions
- Stale `prevVersionId` → `DM_STALE_VERSION`

### 9.4 Contract tests

- Phase A mock fixtures share TypeScript types with backend DTOs
- A shared `.d.ts` file asserts DTO parity; drift is a P1 defect

---

## 10. Phase A / Phase B Toggle

- `VITE_USE_BACKEND=true` switches the API client from mocks to live backend; no call-site changes.
- Phase A mocks simulate `DM_STALE_VERSION` (5% injection), `DM_PII_DETECTED` (via `__PII_TRIGGER__` string), and AI summary pending → ready transitions (3s simulated delay).
- Phase A Viewer renders the actual HTML files in `docs/standard-sdd/projects/control-tower/05-design/*.html` as preview content by mapping seed artifact IDs to file paths. This gives the UI a realistic first impression before the backend lands.
