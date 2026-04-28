# Platform Center — Task Breakdown

## Purpose

Phased implementation plan for the **Platform Center** slice (PRD §11.13 + §12.1–§12.6). Covers all six platform capabilities in V1:

1. Template Management
2. Configuration Management
3. Audit & Compliance
4. Access Control (RBAC)
5. Policy & Governance
6. Integration Framework

Both phases are delivered via **Codex** (FE and BE) — unlike earlier slices where Gemini handled FE. Tasks are grouped by phase and ordered so each task is an independent PR.

## Source Documents

- [platform-center-requirements.md](../01-requirements/platform-center-requirements.md)
- [platform-center-stories.md](../02-user-stories/platform-center-stories.md)
- [platform-center-spec.md](../03-spec/platform-center-spec.md)
- [platform-center-architecture.md](../04-architecture/platform-center-architecture.md)
- [platform-center-data-flow.md](../04-architecture/platform-center-data-flow.md)
- [platform-center-data-model.md](../04-architecture/platform-center-data-model.md)
- [platform-center-design.md](../05-design/platform-center-design.md)
- [platform-center-API_IMPLEMENTATION_GUIDE.md](../05-design/contracts/platform-center-API_IMPLEMENTATION_GUIDE.md)

---

## Phase Overview

| Phase | Scope | Tool | Deliverable |
|---|---|---|---|
| **A** | Frontend with mocked data | Codex | `/platform` renders shell + 6 sub-sections with all states; tests pass; build succeeds |
| **B** | Backend + live API wiring | Codex | Spring Boot endpoints for all 6 capabilities, Flyway V80–V87 migrations, seed data, FE switched from mocks to live |

**Dependency**: Phase B begins after Phase A is merged. Phase B must NOT require component re-work beyond swapping the API client from mock to live (per AI Center / Dashboard / Incident precedent).

**Tool assignment**: Both phases use Codex per explicit user instruction for this slice.

---

## Traceability

| Epic | Stories | REQs | Phase A Tasks | Phase B Tasks |
|---|---|---|---|---|
| Shell & Navigation | S-PC-01 – S-PC-05 | REQ-PC-01 – REQ-PC-10 | A.1 – A.4 | — |
| Template Mgmt | S-PC-10 – S-PC-15 | REQ-PC-20 – REQ-PC-29 | A.5 | B.3, B.4 |
| Configuration Mgmt | S-PC-20 – S-PC-24 | REQ-PC-30 – REQ-PC-39 | A.6 | B.3, B.5 |
| Audit & Compliance | S-PC-30 – S-PC-35 | REQ-PC-40 – REQ-PC-49 | A.7 | B.3, B.6, B.12 |
| Access Control | S-PC-40 – S-PC-46 | REQ-PC-50 – REQ-PC-59 | A.8 | B.3, B.7, B.11 |
| Policy & Governance | S-PC-50 – S-PC-55 | REQ-PC-60 – REQ-PC-69 | A.9 | B.3, B.8 |
| Integration Framework | S-PC-60 – S-PC-65 | REQ-PC-70 – REQ-PC-79 | A.10 | B.3, B.9, B.10 |
| Cross-cutting (RBAC guard, shared, a11y) | S-PC-70 – S-PC-72 | REQ-PC-80 – REQ-PC-86 | A.11 – A.15 | B.1, B.2, B.11, B.13 |

---

## Phase A — Frontend (Codex)

### A.0 Route guard + workspace/platform scope primer
- **Files**: `frontend/src/shell/router/guards/platformAdminGuard.ts` (new), update `frontend/src/router/index.ts` to register `/platform` with the guard
- **Content**: Guard checks `sessionStore.currentUser.roles.includes('PLATFORM_ADMIN')`; on fail, redirect to `/403` (use the shell's existing forbidden view if present, else add minimal placeholder). For Phase A, `sessionStore` seeds `['PLATFORM_ADMIN']` via a mock initializer so the route is reachable.
- **Exit**: `/platform` resolves; `/403` exists and renders when role missing
- **Verification**: unit test on guard with both role states; Playwright/Vitest E2E stub: non-admin user redirected
- **Ref**: design §4.1, API guide §7 (auth), REQ-PC-80

### A.1 Scaffold feature folder
- **Path**: `frontend/src/features/platform/`
- **Create**: folder tree per design doc §1.1 — `shell/`, `shared/`, `template/`, `configuration/`, `audit/`, `access/`, `policy/`, `integration/`, `api/`, `stores/`, `composables/`, `types/`, `constants/`, `index.ts`
- **Exit**: `index.ts` barrel exports `PlatformCenterView`; route resolves to placeholder "Platform Center"
- **Verification**: `npm run build` succeeds with empty feature; tsc clean

### A.2 Types + constants
- **Files**:
  - `types/platform.ts` — shared envelope, scope, audit event types
  - `types/template.ts`, `types/configuration.ts`, `types/audit.ts`, `types/access.ts`, `types/policy.ts`, `types/integration.ts`
  - `constants/scope.ts` — `ScopeLevel = 'platform' | 'application' | 'workspace' | 'project'`, inheritance order
  - `constants/templateCategories.ts`, `constants/policyCategories.ts`, `constants/connectionTypes.ts`
- **Content**: all TS types from data-model doc §2 verbatim; lookups from design doc §6 (visual tokens)
- **Verification**: `tsc --noEmit` clean; no duplicate types with shared/

### A.3 Shell composition
- **Files**:
  - `shell/PlatformCenterView.vue` — left rail + `<router-view>`
  - `shell/PlatformSubNav.vue` — 6 sub-section chips + active state
  - `shell/PlatformBreadcrumb.vue` — derived from route
  - `shell/PlatformCenterEmpty.vue` — placeholder for deep-link 404 inside Platform
- **Content**: per design doc §2.1 – §2.3; respects "Tactical Command" tokens + no-line rule
- **Routing**: `/platform` → redirect to `/platform/templates`; six child routes for each sub-section; catalog `list` and `detail` as named child routes so detail can be a slide-over or full view per design
- **Verification**: clicking each sub-section chip navigates; breadcrumb updates; browser back/forward work

### A.4 Shared components + composables
- **Files**:
  - `shared/components/CatalogTable.vue` — server-paginated table used across capabilities
  - `shared/components/CatalogFilters.vue` — category + status + search
  - `shared/components/ScopeChip.vue`, `shared/components/AuditEventRow.vue`, `shared/components/InheritanceChain.vue`
  - `composables/useCatalog.ts`, `composables/usePlatformAudit.ts`, `composables/useInheritanceResolver.ts`, `composables/useCardState.ts`
- **Content**: per design doc §3; card-state helper maps `{data, loading, error}` → `"normal"|"loading"|"empty"|"error"|"forbidden"`
- **Verification**: Vitest unit tests for each composable; storybook-style visual checks not required

### A.5 Template Management sub-section
- **Files**: `template/TemplateListView.vue`, `template/TemplateDetailPanel.vue`, `template/TemplateVersionTimeline.vue`, `template/TemplateEditForm.vue`, `stores/templateStore.ts`, `api/templateApi.ts`
- **Content**: per design doc §4.2; list columns: name, category, scope, version, status, lastUpdated; detail shows metadata + version timeline + lifecycle actions (Draft → Published → Deprecated → Archived)
- **States**: loading skeleton; empty; error with retry; forbidden (non-admin); draft/published/deprecated/archived visual variants
- **Mocks**: ≥ 12 templates across all categories, ≥ 3 versions for at least 2 templates
- **Verification**: component tests in all 5 states; state machine test (invalid transition Draft → Archived is blocked)
- **Stories**: S-PC-10 – S-PC-15

### A.6 Configuration Management sub-section
- **Files**: `configuration/ConfigurationListView.vue`, `configuration/ConfigurationDetailPanel.vue`, `configuration/InheritancePreview.vue`, `configuration/ConfigurationEditForm.vue`, `stores/configurationStore.ts`, `api/configurationApi.ts`
- **Content**: per design doc §4.3; list columns: key, scope, value-preview, source (Default/Override), lastUpdated; detail shows 4-layer inheritance chain (Platform Default → Application Default → SNOW Group Override → Project Override) with effective-value resolution
- **States**: loading; empty; error; forbidden; override-indicator visual
- **Mocks**: ≥ 20 configuration keys with at least 4 demonstrating full inheritance chain
- **Verification**: component tests; effective-value resolver unit test
- **Stories**: S-PC-20 – S-PC-24

### A.7 Audit & Compliance sub-section
- **Files**: `audit/AuditListView.vue`, `audit/AuditDetailPanel.vue`, `audit/AuditFilters.vue`, `audit/AuditExportDialog.vue`, `stores/auditStore.ts`, `api/auditApi.ts`
- **Content**: per design doc §4.4; filters: date range, actor, entity type, action, scope; cursor-based pagination; event row shows `when / who / what / scope / resource / result`
- **States**: loading; empty; error; forbidden; "Load more" button (cursor); filter-active indicator
- **Mocks**: ≥ 50 audit events across all capabilities, at least one per action type (CREATE/UPDATE/DELETE/PUBLISH/DEPRECATE/ASSIGN/REVOKE/CONNECT/DISCONNECT/POLICY_EVAL/EXCEPTION)
- **Verification**: component tests; filter change triggers refetch with cursor reset; CSV export stub (client-side)
- **Stories**: S-PC-30 – S-PC-35

### A.8 Access Control sub-section
- **Files**: `access/AccessListView.vue`, `access/RoleAssignmentEditor.vue`, `access/AccessDetailPanel.vue`, `access/LastAdminGuardDialog.vue`, `stores/accessStore.ts`, `api/accessApi.ts`
- **Content**: per design doc §4.5; list: assignments by user / role / scope; editor allows assign/revoke; last-admin guard warning dialog blocks final admin revocation
- **States**: loading; empty; error; forbidden; "last-admin" blocked confirm dialog
- **Mocks**: ≥ 8 role assignments with at least 2 distinct users holding PLATFORM_ADMIN (so the last-admin path is testable)
- **Verification**: component tests; last-admin guard unit test (attempting to revoke final admin shows dialog and blocks)
- **Stories**: S-PC-40 – S-PC-46

### A.9 Policy & Governance sub-section
- **Files**: `policy/PolicyListView.vue`, `policy/PolicyDetailPanel.vue`, `policy/PolicyEditForm.vue`, `policy/PolicyExceptionList.vue`, `policy/PolicyExceptionEditor.vue`, `stores/policyStore.ts`, `api/policyApi.ts`
- **Content**: per design doc §4.6; policies table (category, scope, effect, status); detail: rule body + exceptions list + state history (Draft → Active → Retired); exception editor with justification + expiry
- **States**: loading; empty; error; forbidden; invalid-transition block (Draft → Retired directly)
- **Mocks**: ≥ 10 policies across all categories, ≥ 3 with active exceptions, ≥ 1 with expired exception for visual fallback
- **Verification**: component tests; state machine guards; exception expiry date validator test
- **Stories**: S-PC-50 – S-PC-55

### A.10 Integration Framework sub-section
- **Files**: `integration/ConnectionListView.vue`, `integration/ConnectionDetailPanel.vue`, `integration/ConnectionEditForm.vue`, `integration/CredentialRefPicker.vue`, `integration/ConnectionHealthBadge.vue`, `stores/integrationStore.ts`, `api/integrationApi.ts`
- **Content**: per design doc §4.7; list: name, type (Jira/ServiceNow/GitHub/S3/etc.), scope, health, lastCheckedAt; detail includes "Test connection" action; credential picker shows `credentialRef` only (never plain secrets)
- **States**: loading; empty; error; forbidden; connection-failing badge + retry
- **Mocks**: ≥ 6 connections covering 3+ connector types; at least one failing health
- **Verification**: component tests; "Test connection" mocked and animated; no plaintext credential fields anywhere in the UI tree (snapshot test asserts)
- **Stories**: S-PC-60 – S-PC-65

### A.11 Pinia root store composition + workspace reactivity
- **File**: `stores/platformCenterStore.ts`
- **Content**: per API guide §4.2 + data-flow doc §9 — root store registers the 6 sub-stores, exposes `init()`, handles `workspaceChanged` event by clearing caches (note: platform capabilities are mostly workspace-independent but audit/policy can be workspace-scoped, so invalidate only the scoped portions)
- **Verification**: Vitest: `init` fetches lazily per sub-section; workspace switch triggers selective refetch

### A.12 API client barrel + mock wiring
- **File**: `api/index.ts`, plus per-capability `mocks.ts` files
- **Content**: All API clients gated by `VITE_USE_MOCK_API`; barrel re-exports every capability client
- **Mocks spec**: each `mocks.ts` exports realistic fixtures matching API guide §2.x JSON examples verbatim (shape-check via unit test)
- **Verification**: mock fixtures pass JSON-shape contract tests against API guide examples

### A.13 A11y + keyboard pass
- Keyboard nav across sub-nav, tables, forms, slide-overs
- Focus trap + focus restore on slide-over and dialog opens
- Status chips have `role="status"` + aria-labels
- Inheritance chain has `aria-describedby` linking each layer to source
- Forbidden state has `role="alert"` and a link to "request access"
- **Verification**: manual tab-through, axe-core pass

### A.14 Shared promotion (conditional)
- If `fetchJson`, `Page<T>`, `SectionResult<T>`, or cursor helpers exist only in the dashboard/ai-center feature folders, migrate them to `frontend/src/shared/api/` (non-breaking move + re-export). Skip if already shared.
- **Verification**: affected feature slices still build and tests pass

### A.15 Phase A acceptance
- [ ] `/platform` renders with mocks across all 6 sub-sections
- [ ] Non-admin user hitting `/platform/*` is redirected to `/403`
- [ ] Each sub-section renders all 5 states (normal / loading / empty / error / forbidden)
- [ ] Deep-links work for each detail view (e.g., `/platform/templates/:id`, `/platform/policies/:id`)
- [ ] Workspace switch triggers only the scoped sub-sections to refetch
- [ ] `npm run dev` and `npm run build` both succeed
- [ ] `npm run test` passes for all new tests
- [ ] No ESLint/tsc warnings introduced
- [ ] Inheritance preview renders all 4 layers correctly with effective-value highlight
- [ ] No plaintext credentials rendered anywhere (snapshot test asserts)

---

## Phase B — Backend (Codex)

### B.0 Package-by-feature layout
- **Path**: `backend/src/main/java/com/sdlctower/platform/` (note: Platform Center lives under `platform/`, not `domain/` — per data-model doc §4 and CLAUDE.md Lesson #3)
- **Sub-packages**: `template/`, `configuration/`, `audit/`, `access/`, `policy/`, `integration/`, `shared/`
- **Each sub-package**: `controller/`, `service/`, `repository/`, `entity/`, `dto/`, `exception/`
- **Verification**: folder tree matches design doc §1.2

### B.1 Flyway migrations — schema
- **Files** (under `backend/src/main/resources/db/migration/`):
  - `V80__create_platform_template.sql` — PLATFORM_TEMPLATE, PLATFORM_TEMPLATE_VERSION
  - `V81__create_platform_configuration.sql` — PLATFORM_CONFIGURATION
  - `V82__create_platform_audit.sql` — PLATFORM_AUDIT (append-only, partitioned-ready)
  - `V83__create_platform_role_assignment.sql` — PLATFORM_ROLE_ASSIGNMENT
  - `V84__create_platform_policy.sql` — PLATFORM_POLICY, PLATFORM_POLICY_EXCEPTION
  - `V85__create_platform_connection.sql` — PLATFORM_CONNECTION, PLATFORM_CREDENTIAL_REF (stub)
  - `V86__reserve_platform_future_columns.sql` — forward-compat reserved columns (`deleted_at`, `tenant_id`, `rev`) on all platform tables
- **Content**: DDL per data-model doc §5; all string columns use Oracle-compatible types; JSON fields stored as CLOB (no vendor JSON type)
- **Note**: V40–V47 are taken by Code & Build Management; V80+ is the next available range per CLAUDE.md migration allocation
- **Verification**: `MigrationTest` applies V80–V86 from empty schema on H2; `./mvnw test` green; DDL rehearsal on Oracle XE docker image (manual)

### B.2 Flyway migration — seed
- **File**: `backend/src/main/resources/db/migration/V87__seed_platform_center_data.sql`
- **Content**: realistic seed matching Phase A mock counts — ≥ 12 templates with ≥ 15 versions; ≥ 20 configuration keys spanning all 4 inheritance layers; ≥ 50 audit events covering every action type; ≥ 8 role assignments with ≥ 2 PLATFORM_ADMIN holders; ≥ 10 policies + exceptions; ≥ 6 connections + credential refs
- **Note**: seed placed in `db/migration/` following the project convention (V2, V3, V8, V11, V26, V36, V39, V47, V53, V61, V77 are all seeds in the migration folder)
- **Verification**: `local` profile populates data; seed data uses workspace-scoped IDs to avoid collision with other slices

### B.3 JPA entities
- **Files** (one per sub-package under `entity/`):
  - Template: `PlatformTemplate`, `PlatformTemplateVersion`
  - Configuration: `PlatformConfiguration`
  - Audit: `PlatformAudit`
  - Access: `PlatformRoleAssignment`
  - Policy: `PlatformPolicy`, `PlatformPolicyException`
  - Integration: `PlatformConnection`, `PlatformCredentialRef`
- **Content**: per data-model doc §4; no Lombok; standard getters/setters; JSON fields as String/CLOB + Jackson in service layer
- **Verification**: entity fields map cleanly to DDL in B.1; `@DataJpaTest` validates mapping against H2 schema

### B.4 Repositories
- **Files** (one per aggregate root):
  - `TemplateRepository`, `TemplateVersionRepository`
  - `ConfigurationRepository` (with `findEffectiveValue(key, scope)` JPQL)
  - `AuditRepository` (cursor pagination + filter predicate; append-only, no update methods)
  - `RoleAssignmentRepository` (with `countByRoleAndScope` for last-admin guard)
  - `PolicyRepository`, `PolicyExceptionRepository`
  - `ConnectionRepository`, `CredentialRefRepository`
- **Content**: scope-aware query methods; cursor helpers via `CursorCodec` (per API guide §5); no raw SQL unless necessary
- **Verification**: `@DataJpaTest` with H2; scope isolation assertions

### B.5 DTOs + CursorCodec
- **Files**: `dto/` per sub-package (Java records); `shared/cursor/CursorCodec.java`
- **Content**: per data-model doc §3; record field names and shape match FE TS types; camelCase JSON via `@JsonProperty` or Jackson config
- **Verification**: shape contract test compares record JSON output against API guide §2.x examples

### B.6 AuditWriter + atomic audit pattern
- **Files**:
  - `platform/audit/service/AuditWriter.java` — `@Transactional(propagation = Propagation.MANDATORY)` single `write(AuditEvent)` method
  - `platform/audit/service/AuditService.java` — read-side (list, detail, export)
  - `platform/audit/controller/AuditController.java`
- **Content**: per API guide §3.2; `AuditWriter` is called from every write path in every other platform sub-service and MUST participate in the caller's transaction (propagation MANDATORY). If called without an active transaction, throws `TransactionRequiredException`.
- **Verification**:
  - Unit test: calling `AuditWriter.write` without @Transactional throws
  - Integration test: mutation in `TemplateService` rolls back the audit row together with the domain row if the mutation fails after write
  - Append-only test: attempt to update or delete an audit row via JPA fails or is blocked by repository design (no `save()` on existing, no `delete()`)
- **Stories**: S-PC-30 – S-PC-35; REQ-PC-40 – REQ-PC-44

### B.7 Template service + controller
- **Files**: `platform/template/service/TemplateService.java`, `TemplateVersionService.java`, `controller/TemplateController.java`, `exception/TemplateNotFoundException.java`, `exception/InvalidTransitionException.java`
- **Content**: per API guide §3.3; lifecycle transitions enforced via explicit state machine (Draft → Published → Deprecated → Archived; invalid transitions throw `InvalidTransitionException` → 409); every mutation calls `auditWriter.write(...)` inside the same `@Transactional`
- **Verification**: MockMvc for each endpoint; state-machine test; atomic-audit test (see B.6)
- **Stories**: S-PC-10 – S-PC-15

### B.8 Configuration service + controller + inheritance resolver
- **Files**: `platform/configuration/service/ConfigurationService.java`, `InheritanceResolver.java`, `controller/ConfigurationController.java`, `exception/ConfigurationNotFoundException.java`, `exception/DuplicateScopeOverrideException.java`
- **Content**: per API guide §3.4; `InheritanceResolver` walks Platform → Application → SNOW Group → Project and returns effective value + provenance trail; unique constraint on `(config_key, scope_level, scope_id)` with 409 `DUPLICATE_SCOPE_OVERRIDE` on conflict
- **Verification**: unit test for resolver across all 4 layers; MockMvc for endpoints
- **Stories**: S-PC-20 – S-PC-24

### B.9 Access service + controller + last-admin guard
- **Files**: `platform/access/service/AccessService.java`, `controller/AccessController.java`, `exception/LastPlatformAdminException.java`, `exception/AccessAssignmentNotFoundException.java`
- **Content**: per API guide §3.5; `revoke(assignmentId)` runs a guard — if the target is a PLATFORM_ADMIN and `countActivePlatformAdmins() <= 1`, throw `LastPlatformAdminException` → 409 with `LAST_PLATFORM_ADMIN` error code; guard runs inside the same transaction as the revoke write
- **Verification**: MockMvc: revoke final admin → 409; revoke one of two admins → 200; assign new admin → audit event written
- **Stories**: S-PC-40 – S-PC-46

### B.10 Policy service + controller + state machine
- **Files**: `platform/policy/service/PolicyService.java`, `PolicyExceptionService.java`, `controller/PolicyController.java`, `exception/PolicyNotFoundException.java`, `exception/PolicyInUseException.java`
- **Content**: per API guide §3.6; state machine Draft → Active → Retired (no skips); cannot delete a policy with active exceptions → 409 `IN_USE`; exception expiry validated server-side
- **Verification**: MockMvc; state machine; delete-with-exceptions blocked
- **Stories**: S-PC-50 – S-PC-55

### B.11 Integration service + controller + connection test
- **Files**: `platform/integration/service/ConnectionService.java`, `CredentialRefService.java`, `controller/ConnectionController.java`, `exception/ConnectionNotFoundException.java`, `exception/ConnectionTestFailedException.java`
- **Content**: per API guide §3.7; `test(connectionId)` dispatches to a per-connector-type probe (stubbed in V1 — returns simulated outcomes); persists `lastCheckedAt` + `healthStatus`; mutations never accept or return plaintext secrets — only `credentialRef`
- **Verification**: MockMvc; plaintext-secret guard (controller input validation rejects any `password` / `apiKey` / `token` body field); persisted row has no secret columns
- **Stories**: S-PC-60 – S-PC-65

### B.12 RBAC guard — @RequireAdmin + AdminAuthGuard interceptor
- **Files**: `platform/shared/auth/RequireAdmin.java` (annotation), `platform/shared/auth/AdminAuthGuard.java` (HandlerInterceptor), `config/WebMvcConfig.java` (register interceptor)
- **Content**: per API guide §7; every platform controller method annotated with `@RequireAdmin`; interceptor inspects session/JWT (reusing the project's existing mechanism) for `PLATFORM_ADMIN` role; rejects with 403 `FORBIDDEN` if absent
- **Verification**: MockMvc: non-admin → 403 on every platform endpoint; admin → passes through; unit test on the interceptor
- **Stories**: S-PC-70; REQ-PC-80

### B.13 CursorCodec + pagination validation
- **Files**: `platform/shared/cursor/CursorCodec.java`, applied in all list controllers
- **Content**: per API guide §5; opaque base64-encoded `(sortKey, lastId)`; `size` outside [1, 200] → 400
- **Verification**: round-trip encode/decode test; MockMvc for `size=201` and `size=0`

### B.14 Integration audit hooks across all sub-services
- **Rule**: every mutating endpoint in B.7 – B.11 MUST call `auditWriter.write` inside its transaction. Verified by:
  - Static check: a custom `ArchUnit` test asserts that every `@Transactional` method on a `*Service` class under `platform/` whose name starts with `create|update|delete|publish|deprecate|archive|assign|revoke|connect|disconnect|test|activate|retire` invokes `AuditWriter.write`
  - Integration test per sub-service: perform a mutation, then query audit repository and assert the event exists with correct `action` and `resource`
- **Verification**: ArchUnit test passes; per-service integration tests green

### B.15 Frontend wire-up (still Phase B, done by Codex)
- Set `VITE_USE_MOCK_API=false` (or remove) for default dev
- Verify `vite.config.ts` proxy covers `/api/v1/platform/*`; add if missing
- Run end-to-end: FE + BE both up; `/platform` renders live BE-served data across all 6 sub-sections
- **Verification**: manual QA script (link-by-link click through each sub-section, each detail view, each mutating action)

### B.16 Phase B acceptance
- [ ] `./mvnw clean test` passes all tests (existing + new); ArchUnit rule in B.14 passes
- [ ] `./mvnw spring-boot:run -Dspring.profiles.active=local` starts cleanly with V80–V87 applied + seed loaded
- [ ] All 32 endpoints return JSON matching API guide §2 examples (shapes verified by MockMvc tests)
- [ ] Every platform endpoint returns 403 for non-admin (B.12 verified)
- [ ] `GET /api/v1/platform/*?size=201` → 400 with `{ data: null, error: "size must be between 1 and 200" }`
- [ ] Revoking final PLATFORM_ADMIN → 409 with `LAST_PLATFORM_ADMIN`
- [ ] Invalid state transitions (template, policy) → 409 with `INVALID_TRANSITION`
- [ ] Mutation failure rolls back audit row (atomic audit verified)
- [ ] Seed loads under `local`, skipped under `prod`
- [ ] FE switched from mocks; `npm run dev` + `npm run build` both succeed; `/platform` renders live data
- [ ] No existing tests broken

---

## Dependency Plan

```
                      ┌───────── A.0 guard ─────────┐
                      │                             │
                   A.1 scaffold ─── A.2 types ── A.3 shell ── A.4 shared
                                                                  │
          ┌──────────┬──────────┬──────────┬──────────┬─────────┤
       A.5 tmpl   A.6 cfg   A.7 audit  A.8 access  A.9 policy  A.10 integ
          │          │          │          │          │          │
          └──────────┴──────────┴──── A.11 stores ────┴──────────┘
                                      │
                               A.12 API barrel + mocks
                                      │
                                A.13 a11y    A.14 shared promote
                                      │
                                A.15 Phase A acceptance
                                      │
              ──────── Phase A merged ─────────
                                      │
                          B.0 package layout
                                      │
                          B.1 migrations schema
                                      │
                          B.2 seed migration
                                      │
                               B.3 entities
                                      │
                             B.4 repositories
                                      │
                            B.5 DTOs + Cursor
                                      │
                            B.6 AuditWriter (BLOCKER)
                                      │
          ┌──────────┬──────────┬──────────┬──────────┬──────────┐
       B.7 tmpl   B.8 cfg    B.9 access  B.10 pol  B.11 integ  B.12 guard
          │          │          │          │          │          │
          └──────────┴──────────┴──── B.13 CursorCodec ───────────┘
                                      │
                          B.14 ArchUnit audit rule
                                      │
                          B.15 FE wire-up
                                      │
                          B.16 Phase B acceptance
```

**Critical path**: A.0 → A.1 → A.4 → (A.5–A.10 parallel) → A.11 → A.15, then B.1 (V80–V86) → B.2 (V87 seed) → B.3 → B.6 → (B.7–B.11 parallel) → B.14 → B.16.

**Key gating task**: **B.6 (AuditWriter)** must land before any of B.7–B.11 because their mutations invoke `auditWriter.write`.

---

## Risks and Mitigations

| Risk | Mitigation |
|---|---|
| Single PLATFORM_ADMIN role not granular enough | V1 ships single role per user decision; layer fine-grained roles in V2 (tracked in open questions) |
| Atomic audit pattern skipped in new sub-services | ArchUnit rule in B.14 enforces `AuditWriter.write` call in every mutating service method |
| Seed data in prod | Seed uses workspace-scoped IDs; harmless in prod but can be excluded via Flyway `ignoreMigrationPatterns` if needed |
| Plaintext credentials leak into UI or API | UI snapshot test rejects `password`/`apiKey`/`token` in DOM; controller input validation rejects same fields in JSON body |
| Inheritance resolver drift FE vs BE | FE resolver is display-only; BE `InheritanceResolver` is authoritative; contract test asserts FE/BE produce same effective value for the same scope/key |
| Cursor token forgery | Opaque base64 + HMAC signing in `CursorCodec` (documented in API guide §5); tampered cursor → 400 |
| Oracle reserved-word collisions | All platform DDL reviewed against Oracle reserved list; columns like `key` → `config_key`, `value` → `config_value` |
| Cross-domain audit noise | Only platform-scoped mutations write to PLATFORM_AUDIT; domain-scoped events continue to their own domain audit (no unification in V1) |
| Interceptor order conflicts with workspace context | Register `AdminAuthGuard` AFTER `WorkspaceContextFilter` so workspace is resolved first; documented in B.12 |
| Policy exception bypass of policy state | State machine in B.10 blocks creating exceptions for Retired policies; FK cascade blocks orphaned exceptions |

---

## Open Questions

1. **Group-based admin** — V1 single PLATFORM_ADMIN role; V2 may need admin groups (e.g., `PLATFORM_TEMPLATE_ADMIN`, `PLATFORM_POLICY_ADMIN`). Track as separate PRD item.
2. **Audit retention** — V1 keeps all events; V2 may archive after N months. Data-model doc §5 notes `deleted_at` reserved but not used.
3. **Real connector probes** — V1 simulates `Connection.test()` outcomes; V2 wires real probes per connector type (Jira REST ping, ServiceNow table API, GitHub API rate-limit check, S3 HeadBucket).
4. **Credential vault integration** — `PLATFORM_CREDENTIAL_REF` table stores a reference id; the actual vault integration is stubbed in V1 (credential lookup returns a placeholder). V2 wires HashiCorp Vault / AWS SSM / Azure Key Vault.
5. **Cross-workspace aggregation** — V1 platform scope is global; workspace/application/project scopes coexist. V2 may need cross-workspace dashboards for platform admins.
6. **SSO/SCIM for role provisioning** — V1 accepts role assignment via API; V2 may sync from IdP via SCIM. `PLATFORM_ROLE_ASSIGNMENT.source` column reserved.

---

## Out of Scope (tracked for later slices)

- Fine-grained RBAC (per-sub-section admin roles)
- Policy-as-code editor (V1 is JSON/form-based)
- Real-time push of audit events (V1 is poll-on-filter-change)
- Template marketplace / import from external registries
- SSO / SCIM provisioning
- Vault integration for credential storage
- Cross-workspace or cross-tenant aggregation views
- Export to PDF/audit bundle (CSV client-side export only in V1; full audit bundle belongs to Report Center)
- Policy simulation / what-if tooling
- Connection auto-discovery
