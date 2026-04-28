# Platform Center Design

## Purpose

This document is the implementation-facing design for the Platform Center slice. It gives concrete file paths, component contracts, routing, state management, visual decisions, DB schema summary, and error/empty states. It is distinct from the architecture doc (which defines "what and why") — this doc defines "where, how, and with what".

## Source

- [platform-center-spec.md](../03-spec/platform-center-spec.md)
- [platform-center-architecture.md](../04-architecture/platform-center-architecture.md)
- [platform-center-data-flow.md](../04-architecture/platform-center-data-flow.md)
- [platform-center-data-model.md](../04-architecture/platform-center-data-model.md)
- [platform-center-API_IMPLEMENTATION_GUIDE.md](contracts/platform-center-API_IMPLEMENTATION_GUIDE.md)
- [design.md (root)](../../design.md) — visual design system
- [design.md (product)](design.md) — product-module design
- Existing patterns: `frontend/src/features/incident/`, `frontend/src/features/dashboard/`, `frontend/src/features/project-space/`

---

## 1. File Structure

### 1.1 Frontend (Vue 3 + Vite + Pinia + TypeScript)

The `frontend/src/features/platform/` directory already exists as an empty placeholder. This slice fills it:

```
frontend/src/features/platform/
├── index.ts                          # feature registration, exports routes
├── routes.ts                         # Vue Router child routes for /platform
├── guard.ts                          # route-level PLATFORM_ADMIN guard
│
├── shell/
│   ├── PlatformShell.vue             # two-pane layout: rail + outlet
│   ├── SubSectionRail.vue            # left-rail nav between 6 sub-sections
│   └── ForbiddenView.vue             # 403 screen
│
├── shared/
│   ├── api.ts                        # shared axios/api base + mock toggle
│   ├── CatalogTable.vue              # shared catalog table primitive
│   ├── DetailPanel.vue               # shared right/overlay detail panel
│   ├── DestructiveConfirm.vue        # shared destructive-confirm modal
│   ├── EmptyState.vue                # empty-state primitive
│   ├── ErrorPanel.vue                # error primitive
│   ├── ScopeChip.vue                 # {type:id} chip with tooltip
│   ├── InheritanceChip.vue           # layer chip + popover
│   ├── StatusBadge.vue               # active / draft / deprecated / error
│   ├── DriftBadge.vue                # drift indicator
│   ├── CursorPager.vue               # cursor-based pagination UI
│   ├── types.ts                      # shared TS types (ScopeType, Role, etc.)
│   └── useCurrentUser.ts             # composable reading /access/me
│
├── templates/
│   ├── TemplatesView.vue             # catalog + detail
│   ├── TemplateDetail.vue            # body + versions + inheritance
│   ├── TemplateVersionList.vue
│   ├── TemplatePublishDialog.vue
│   ├── store.ts                      # Pinia store: templatesStore
│   ├── api.ts                        # typed API functions
│   └── mocks.ts                      # Phase A fixtures
│
├── configurations/
│   ├── ConfigurationsView.vue
│   ├── ConfigurationDetail.vue
│   ├── OverrideForm.vue
│   ├── DriftCompare.vue
│   ├── store.ts
│   ├── api.ts
│   └── mocks.ts
│
├── audit/
│   ├── AuditView.vue
│   ├── AuditDetail.vue
│   ├── AuditFilters.vue
│   ├── store.ts
│   ├── api.ts
│   └── mocks.ts
│
├── access/
│   ├── AccessView.vue
│   ├── AssignRoleDialog.vue
│   ├── store.ts
│   ├── api.ts
│   └── mocks.ts
│
├── policies/
│   ├── PoliciesView.vue
│   ├── PolicyDetail.vue
│   ├── PolicyForm.vue
│   ├── PolicyExceptionList.vue
│   ├── AddExceptionDialog.vue
│   ├── store.ts
│   ├── api.ts
│   └── mocks.ts
│
└── integrations/
    ├── IntegrationsView.vue
    ├── AdapterRegistryHeader.vue
    ├── ConnectionDetail.vue
    ├── ConnectionForm.vue
    ├── TestConnectionButton.vue
    ├── store.ts
    ├── api.ts
    └── mocks.ts
```

**Router wiring** (added in `frontend/src/router/index.ts`):

```ts
import platformRoutes from "@/features/platform/routes";
// ...
{
  path: "/platform",
  component: () => import("@/features/platform/shell/PlatformShell.vue"),
  beforeEnter: platformGuard,
  children: platformRoutes,
}
```

### 1.2 Backend (Spring Boot 3.x + Java 21)

Under `backend/src/main/java/com/sdlctower/platform/`, the slice adds six sub-packages (plus a `shared` package for the cross-cutting beans):

```
backend/src/main/java/com/sdlctower/platform/
├── workspace/                         # existing
├── navigation/                        # existing
│
├── shared/                            # NEW — cross-cutting platform beans
│   ├── AdminAuthGuard.java            # HandlerInterceptor + @RequireAdmin
│   ├── AuditWriter.java               # single-row insert bean
│   ├── ScopeResolver.java             # parses/validates scope query params
│   ├── InheritanceResolver.java
│   ├── CursorCodec.java               # base64 encode/decode cursor
│   └── dto/
│       ├── CursorPage.java
│       ├── ApiError.java
│       └── ScopeDto.java
│
├── template/
│   ├── Template.java                  # @Entity
│   ├── TemplateVersion.java
│   ├── TemplateRepository.java
│   ├── TemplateVersionRepository.java
│   ├── TemplateService.java
│   ├── TemplateController.java
│   ├── TemplateSummaryDto.java
│   ├── TemplateDetailDto.java
│   ├── TemplateVersionDto.java
│   └── InheritanceFieldDto.java
│
├── configuration/
│   ├── Configuration.java
│   ├── ConfigurationRepository.java
│   ├── ConfigurationService.java
│   ├── ConfigurationController.java
│   ├── ConfigurationSummaryDto.java
│   └── ConfigurationDetailDto.java
│
├── audit/
│   ├── AuditRecord.java
│   ├── AuditRepository.java
│   ├── AuditQueryService.java
│   ├── AuditController.java
│   └── AuditRecordDto.java
│
├── access/
│   ├── RoleAssignment.java
│   ├── RoleAssignmentRepository.java
│   ├── AccessService.java
│   ├── AccessController.java          # includes /me + assignments
│   ├── RoleAssignmentDto.java
│   ├── AssignRoleRequest.java
│   └── CurrentUserDto.java
│
├── policy/
│   ├── Policy.java
│   ├── PolicyException.java
│   ├── PolicyRepository.java
│   ├── PolicyExceptionRepository.java
│   ├── PolicyService.java
│   ├── PolicyController.java
│   ├── PolicyDto.java
│   ├── PolicyExceptionDto.java
│   └── CreatePolicyRequest.java
│
└── integration/
    ├── Connection.java
    ├── ConnectionRepository.java
    ├── CredentialRef.java
    ├── CredentialRefRepository.java
    ├── SecretStoreStub.java           # @Service
    ├── IntegrationService.java
    ├── IntegrationController.java
    ├── ConnectionDto.java
    ├── AdapterDescriptorDto.java
    ├── ConnectionTestResultDto.java
    ├── CreateConnectionRequest.java
    └── adapter/
        ├── AdapterTester.java         # interface
        ├── JiraAdapterTester.java
        ├── GitlabAdapterTester.java
        ├── JenkinsAdapterTester.java
        ├── ServicenowAdapterTester.java
        └── WebhookAdapterTester.java
```

Test structure mirrors main (package-by-feature per CLAUDE.md Lesson #3).

### 1.3 Flyway migrations

Under `backend/src/main/resources/db/migration/`, add:

```
V40__create_platform_template.sql
V41__create_platform_configuration.sql
V42__create_platform_audit.sql
V43__create_platform_role_assignment.sql
V44__create_platform_policy.sql
V45__create_platform_connection.sql
V46__seed_platform_center_data.sql
V47__reserve_platform_future_columns.sql
```

Full DDL in [platform-center-data-model.md §6](../04-architecture/platform-center-data-model.md).

---

## 2. Layout Composition

### 2.1 Platform Center shell (all sub-sections)

```
┌──────────────────────────────────────────────────────────────────────────┐
│ (shared app shell: top nav, workspace chip, AI panel toggle)            │
├────────────────┬─────────────────────────────────────────────────────────┤
│                │  ┌────────────────────────────────────────────────────┐ │
│ Sub-section    │  │ Sub-section page header                            │ │
│ rail           │  │  Title · filters · "Create" action (right)         │ │
│                │  ├────────────────────────────────────────────────────┤ │
│  ○ Templates   │  │                                                    │ │
│  ● Configs     │  │  CatalogTable (sortable, paginated)                │ │
│  ○ Audit       │  │  ┌─ row ────────────────────────────────────────┐ │ │
│  ○ Access      │  │  │ name · kind · scope · status · last-mod · … │ │ │
│  ○ Policies    │  │  └───────────────────────────────────────────── ┘ │ │
│  ○ Integrations│  │  (empty / loading / error states replace rows)    │ │
│                │  │                                                    │ │
│                │  │  CursorPager                                       │ │
│                │  └────────────────────────────────────────────────────┘ │
└────────────────┴─────────────────────────────────────────────────────────┘
Rail width: 224px (collapsible to 56px icon-only)
Main area: fluid, max-width 1440px centered
```

Detail panel opens as an overlay from the right, width 480–720px depending on content density. Dialogs (create, destructive-confirm) are centered modals.

### 2.2 Template detail panel

```
┌─ TemplateDetail (overlay) ───────────────────────────────────────┐
│ Header: <Name>  · <status chip> · <kind chip>        [X close]   │
│ ─────────────────────────────────────────────────────────────────│
│ Tabs: [Overview] [Inheritance] [Versions] [Usage]                │
│ ─────────────────────────────────────────────────────────────────│
│  Overview body                                                   │
│    Key:            project-default-v1                            │
│    Owner:          admin@sdlctower.local                         │
│    Current version: 3 (published)                                │
│    Description:    …                                             │
│    Actions:  [Publish] [Deprecate] [New version] [Delete (draft)]│
│ ─────────────────────────────────────────────────────────────────│
│  Inheritance tab                                                 │
│    Field: defaultMilestones                                      │
│      Effective: [Kickoff, Design, Build, Rollout]                │
│      Winning layer: ● Project  (others: Platform set, App -, SG -)│
│    …                                                             │
│ ─────────────────────────────────────────────────────────────────│
│  Versions tab                                                    │
│    v3 · 2026-04-10 · admin@… · [View body]                       │
│    v2 · 2026-03-02 · admin@… · [View body]                       │
└──────────────────────────────────────────────────────────────────┘
```

---

## 3. Component API Contracts

### 3.1 Shared primitives

#### `CatalogTable.vue`

| Prop | Type | Required | Notes |
|------|------|----------|-------|
| `rows` | `Row[]` | yes | generic row type |
| `columns` | `ColumnDef[]` | yes | `{key, label, width, render?}` |
| `state` | `'loading' \| 'error' \| 'empty' \| 'normal'` | yes | drives the fallback view |
| `errorMessage` | `string?` | no | shown inside ErrorPanel |
| `emptyMessage` | `string` | yes | shown inside EmptyState |
| `emptyActionLabel` | `string?` | no | optional primary CTA |
| `selectedId` | `string?` | no | highlights the selected row |

Emits: `@rowClick(row)`, `@retry()`, `@emptyAction()`.

#### `DetailPanel.vue`

| Prop | Type | Required |
|------|------|----------|
| `open` | `boolean` | yes |
| `title` | `string` | yes |
| `width` | `'sm' \| 'md' \| 'lg'` | no (default `md`) |

Emits: `@close`, slot `header-actions`, default slot body.

#### `DestructiveConfirm.vue`

| Prop | Type |
|------|------|
| `open` | `boolean` |
| `title` | `string` |
| `message` | `string` |
| `targetLabel` | `string` | (verbatim target name — bold in body) |
| `confirmLabel` | `string` (default `"Confirm"`) |
| `disabled` | `boolean` (when `true`, primary is grey + tooltip provided) |
| `disabledReason` | `string?` |

Emits: `@confirm`, `@cancel`.

#### `InheritanceChip.vue`

| Prop | Type |
|------|------|
| `winningLayer` | `'platform' \| 'application' \| 'snowGroup' \| 'project'` |
| `layers` | `{ platform: unknown; application: unknown\|null; snowGroup: unknown\|null; project: unknown\|null }` |

Renders a colored dot + label + tooltip showing all four layer values side-by-side.

#### `ScopeChip.vue`

| Prop | Type |
|------|------|
| `scopeType` | `ScopeType` |
| `scopeId` | `string` |

Renders `platform:*` / `workspace:WS-42` / etc. with a tooltip for the full-id.

### 3.2 Per-sub-section view contracts

All six `*View.vue` components share the same overall shape:

| Input | Source |
|-------|--------|
| Filters | URL query string (via `useRoute`) |
| Catalog rows | sub-section Pinia store |
| Selected row id | URL path `:id?` |
| Current-user roles | `useCurrentUser()` composable |

Emits-to-store actions: `fetchCatalog()`, `applyFilter()`, `selectRow(id)`, `mutateThenRefetch()`.

### 3.3 Route-guard contract

```ts
// guard.ts
export const platformGuard: NavigationGuard = async (to, from, next) => {
  const user = await fetchCurrentUser();
  if (user.roles.includes("PLATFORM_ADMIN")) return next();
  return next({ name: "platform-forbidden" });
};
```

`fetchCurrentUser()` calls `GET /api/v1/platform/access/me`. Failure → treat as non-admin.

---

## 4. State Management

### 4.1 Pinia stores — shape

Each sub-section has an identically-shaped store (illustrated with `templatesStore`):

```ts
interface TemplatesStoreState {
  status: 'idle' | 'loading' | 'error' | 'ready';
  error: string | null;
  items: TemplateSummary[];
  selectedId: string | null;
  detail: TemplateDetail | null;
  filters: { kind?: TemplateKind; status?: TemplateStatus; q?: string };
  cursor: string | null;       // next-page cursor
  total: number | null;
  lastFetchedAt: number | null;
}
```

Actions: `fetchCatalog`, `fetchNextPage`, `applyFilter`, `selectRow`, `fetchDetail`, `publish`, `deprecate`, `reactivate`, `delete`, `createNewVersion`.

The store is the ONLY consumer of `api.ts`. Components never call `api.ts` directly.

### 4.2 Cache policy

- On mount: if `lastFetchedAt` is within 60 seconds and filters unchanged, use cache; otherwise fetch.
- After any successful mutation within the sub-section: refetch catalog page 1 and reload current detail if any.
- Across sub-sections: no shared cache. Each sub-section is isolated.

### 4.3 URL state

The active sub-section is encoded in the path (`/platform/{sub}/:id?`). Catalog filters are encoded in the query string. This keeps navigation, refresh, and shareable links consistent.

---

## 5. Routing

```
/platform                            → redirect to /platform/templates
/platform/forbidden                  → ForbiddenView
/platform/templates                  → TemplatesView (catalog)
/platform/templates/:id              → TemplatesView + open DetailPanel
/platform/configurations             → ConfigurationsView
/platform/configurations/:id         → ConfigurationsView + DetailPanel
/platform/audit                      → AuditView
/platform/audit/:id                  → AuditView + DetailPanel
/platform/access                     → AccessView
/platform/policies                   → PoliciesView
/platform/policies/:id               → PoliciesView + DetailPanel
/platform/integrations               → IntegrationsView
/platform/integrations/:id           → IntegrationsView + DetailPanel
```

All routes are children of the `/platform` parent route, which runs the admin guard.

---

## 6. Visual Design Decisions

### 6.1 Color tokens (shared, per `design.md`)

| Token | Usage |
|-------|-------|
| `--color-ok` (`#4CAF50` equivalent) | `published`, `active`, `enabled` status; connection test success |
| `--color-warning` (`#FFC107` equivalent) | `draft`, `has_drift`, `pending` |
| `--color-critical` (`#F44336` equivalent) | `deprecated` indicator border, destructive action primary button, connection test failure |
| `--color-neutral` | `inactive`, `disabled` |
| `--color-accent-primary` | Active sub-section rail indicator |

Exact hex values come from `design.md`; Platform Center references tokens by name.

### 6.2 Typography

- Page header: `font-size: 20px; font-weight: 600;`
- Sub-section title: `font-size: 16px; font-weight: 600;`
- Table header: `font-size: 12px; text-transform: uppercase; letter-spacing: 0.06em;`
- Row text: `font-size: 14px;` with monospace for keys
- Chip text: `font-size: 11px; font-weight: 500; letter-spacing: 0.02em;`

### 6.3 Status badge styles

| Status | Background | Text | Border |
|--------|------------|------|--------|
| `published` / `active` / `enabled` | `--color-ok-surface` | `--color-ok` | 1px `--color-ok` |
| `draft` | `--color-warning-surface` | `--color-warning` | 1px dashed `--color-warning` |
| `deprecated` / `inactive` / `disabled` | `--color-neutral-surface` | `--color-neutral` | 1px `--color-neutral` |
| `error` | `--color-critical-surface` | `--color-critical` | 1px `--color-critical` |

### 6.4 Destructive confirm button

Destructive primary button uses `--color-critical` background with `#fff` text; focus ring is 2px `--color-critical`; disabled state fades to 60% opacity with cursor `not-allowed`.

### 6.5 Animation

- Catalog row hover: 80ms ease-out background-color transition
- Detail panel open/close: 160ms ease-out transform slide
- Sub-section rail switch: no animation (instant)
- Status badge: none — badges are intentionally static to keep the admin surface calm

---

## 7. Error, Empty, and Forbidden States

| State | When | What renders |
|-------|------|--------------|
| Loading | Catalog fetch in-flight | Skeleton rows (6 rows, pulse animation) |
| Normal | `items.length > 0` | CatalogTable with rows |
| Empty | 200 OK with zero rows | EmptyState: illustration + message + optional primary action (see FR-70 table) |
| Error | 5xx or network error | ErrorPanel: message + Retry button (preserves filters) |
| Forbidden | Route-guard redirected to `/platform/forbidden` | ForbiddenView: icon + "Platform Center is restricted to platform administrators" + "Back to Dashboard" |
| Detail load error | `GET /id` failed | ErrorPanel inside DetailPanel; close button still works |
| Mutation validation (422/400) | Bad input | Inline field-level messages in the form; primary Save button stays disabled until valid |
| Mutation conflict (409) | e.g., last-admin, in-use delete | Toast + destructive-confirm primary button disabled with tooltip (for known cases) |
| Mutation unknown (500) | Unexpected error | Red toast: "Something went wrong. Try again." + retry hint in logs |

---

## 8. Validation

All forms use client-side validation mirroring server rules:

| Field | Rule | Error message |
|-------|------|---------------|
| `template.key` | `^[a-z][a-z0-9-]{2,63}$` | "Key must be lowercase letters, numbers, and dashes (3–64 chars)" |
| `configuration.body` | Valid JSON; schema for the kind | "Body is not valid for kind {kind}" |
| `roleAssignment.role` | Must be one of the five roles | "Invalid role" |
| `roleAssignment.scopeId` | Required; `platform` scope must be `*` | "Scope id required" or "Platform scope id must be *" |
| `policy.body` | Kind-shaped JSON (see data-model §8.3) | "Policy body does not match {category} schema" |
| `connection.credentialRef` | Starts with `cred-` + 3–64 lowercase-dash | "Invalid credential reference" |
| `connection.pullSchedule` | Valid 6-field cron when `syncMode` ∈ `pull|both` | "Invalid cron expression" |
| `connection.pushUrl` | Valid https URL when `syncMode` ∈ `push|both` | "Invalid https URL" |
| `policyException.expiresAt` | Must be a future timestamp | "Expiry must be in the future" |

Server-side validation returns 400/422 with `error.code` and `error.details.fields` so the form can highlight the right inputs.

---

## 9. Testing

### 9.1 Frontend

| Test | Lives in | Covers |
|------|----------|--------|
| `TemplatesView.spec.ts` | `frontend/src/features/platform/templates/` | Catalog fetch, filters, row select, detail open |
| `store.spec.ts` per sub-section | same | Store transitions (idle → loading → ready/error) |
| `guard.spec.ts` | `frontend/src/features/platform/` | Non-admin routed to `/platform/forbidden` |
| `DestructiveConfirm.spec.ts` | `shared/` | Primary disabled when `disabled=true`; confirm emits |
| `InheritanceChip.spec.ts` | `shared/` | Winning layer rendered correctly |

Use Vitest + Vue Test Utils. Mock API responses via `msw` or direct module stubbing.

### 9.2 Backend

Required tests (one per listed class at minimum):

| Test | Class |
|------|-------|
| `TemplateControllerTest` | GET list / detail / publish / deprecate / reactivate / delete |
| `ConfigurationControllerTest` | GET list / create / revert / drift |
| `AuditControllerTest` | GET list (paginated, filtered) / GET by id |
| `AccessControllerTest` | GET /me / assignments CRUD / last-admin guard |
| `PolicyControllerTest` | CRUD + version-on-edit + exception add/revoke |
| `IntegrationControllerTest` | CRUD + test-connection + enable/disable |
| `AuditWriterIntegrationTest` | Audit written in same tx; rollback on failure |
| `InheritanceResolverTest` | Layer merging + winning-layer selection |
| `ScopeResolverTest` | Parse / validate scope query params |

Use Spring Boot Test + MockMvc; H2 for test profile. Controller tests use `@WebMvcTest` where possible.

### 9.3 Acceptance criteria for the slice

- `./mvnw test` passes with all new tests green
- `npm run build` succeeds from `frontend/`
- `npm run dev` starts and `/platform` renders the Templates catalog (Phase A: mocks; Phase B: live API)
- Route to `/platform` as a non-admin shows the Forbidden view
- Publishing a draft template produces (a) updated status in catalog and (b) a new `PLATFORM_AUDIT` row of action `template.publish`
- Revoking the last `PLATFORM_ADMIN` returns 409 with code `LAST_PLATFORM_ADMIN`
- Connection test on a seeded connection returns `{ok: true, latencyMs, message}`

---

## 10. Integration Boundary Diagram

```mermaid
graph LR
    subgraph Platform Center slice
        FE[Vue feature: features/platform/]
        BE[Spring package: platform.{sub}]
        DB[PLATFORM_* tables]
    end

    subgraph Upstream it consumes
        SHELL[shared-app-shell]
        WORKSPACE[platform.workspace]
    end

    subgraph Downstream consumers (later slices)
        AICENTER[ai-center]
        DOMAINS[domain slices]
        DASHBOARD[dashboard]
        REPORT[report-center]
    end

    FE --> SHELL
    FE --> BE
    BE --> WORKSPACE
    BE --> DB

    DOMAINS -. read policies/config .-> BE
    AICENTER -. write skill-exec audit via AuditWriter .-> BE
    DASHBOARD -. read metric templates .-> BE
    REPORT -. read audit .-> BE
```

Key rules:

- Platform Center NEVER imports from `domain.*` packages (per CLAUDE.md Lesson #3)
- Downstream slices reach Platform Center only through REST or through the shared `AuditWriter` bean
- Platform Center treats `platform.workspace` and `platform.navigation` as read-only collaborators

---

## 11. Risks & Tradeoffs

| Design choice | Alternative | Why |
|---------------|-------------|-----|
| Per-sub-section store | Shared store | Isolation, testability, smaller surface per test |
| JSON-in-CLOB for bodies | Strongly-typed table per kind | V1 flexibility; schemas evolve without migrations |
| Inline audit write in tx | Async audit queue | Correctness over throughput; V1 scale low |
| `AUDITOR` role in data model but not UI | Defer entirely | Reserves schema so V2 doesn't need migration |
| Single route guard at `/platform` | Per-route guard | Simpler; whole module is one admin surface |
| Cursor pagination everywhere | Offset pagination | Consistent, safe under concurrent inserts |
| `SecretStoreStub` in this slice | Defer secret store to a different slice | Keeps slice self-contained and deployable |

---

## 12. Open Questions

| Q | Default |
|---|---|
| Do we expose search-by-user-id in Access? | Yes — substring match on `user_id` |
| Do we allow cross-scope audit queries? | Yes — no scope filter defaults to all scopes |
| Does the UI render the JSON body with a schema-aware form or raw editor? | V1 raw JSON textarea with validation; schema-aware form is V2 |
| Does "Test connection" require admin lock (only one at a time per connection)? | No — V1 allows parallel test calls; backend handles concurrency |
| How is `currentVersion` computed when draft and published coexist? | `currentVersion` always points to the latest non-deprecated version (draft wins over published if newer) |
