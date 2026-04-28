# AI Center — Task Breakdown

## Purpose

Phased implementation plan for the AI Center slice. Both phases are delivered via **Codex** (FE and BE). Tasks are grouped by phase and ordered so that each task can be executed independently with a clean PR.

## Source

- [ai-center-spec.md](../03-spec/ai-center-spec.md)
- [ai-center-design.md](../05-design/ai-center-design.md)
- [ai-center-data-model.md](../04-architecture/ai-center-data-model.md)
- [ai-center-API_IMPLEMENTATION_GUIDE.md](../05-design/contracts/ai-center-API_IMPLEMENTATION_GUIDE.md)

---

## Phase Overview

| Phase       | Scope                     | Tool  | Deliverable                                                                                        |
| ----------- | ------------------------- | ----- | -------------------------------------------------------------------------------------------------- |
| **A** | Frontend with mocked data | Codex | `/ai-center` renders 4 cards + 2 detail panels; all 5 states covered; tests pass; build succeeds |
| **B** | Backend + live API wiring | Codex | Spring Boot endpoints, Flyway migrations, seed data, FE switched from mocks to live                |

**Dependency**: Phase B begins after Phase A is merged. Phase B must NOT require component re-work beyond swapping the API client from mock to live (per Dashboard / Incident precedent).

---

## Phase A — Frontend (Codex)

### A.1 Scaffold feature folder

- **Path**: `frontend/src/features/ai-center/`
- **Create**: folder structure per design doc §1.1
- **Exit**: `index.ts` barrel exports `AiCenterView`; folder tree matches spec
- **Verification**: `npm run build` succeeds with empty feature (view returns "AI Center placeholder")

### A.2 Types + constants

- **Files**: `types.ts`, `constants.ts`
- **Content**: all TS types from data-model doc §2 verbatim; 11-stage label map; autonomy tooltips; status color token lookups
- **Verification**: `tsc --noEmit` clean

### A.3 Mocks

- **File**: `api/mocks.ts`
- **Content**:
  - ≥ 6 mock skills (mix delivery/runtime) with populated stages, owner, success rates
  - ≥ 20 mock runs across 30 days, covering every status value
  - One mock `SkillDetail` per skill with `recentRuns` + policy + aggregate metrics
  - One mock `RunDetail` per run with steps, policy trail, evidence
  - Metrics mock returns realistic numbers with per-section data
  - Stage coverage mock returns all 11 entries
- **Verification**: mock data passes JSON shape check against API guide examples

### A.4 API client

- **File**: `api/aiCenterApi.ts`
- **Content**: exact signatures from API guide §4.1, gated by `VITE_USE_MOCK_API` env var
- **Verification**: unit tests cover happy path and error mapping (`throw new Error(res.error)`)

### A.5 Store

- **File**: `stores/aiCenterStore.ts`
- **Content**: Pinia store per API guide §4.2 + data-flow doc §9 behavior (workspace switch clears cache)
- **Verification**: Vitest unit tests for `init`, workspace switch, filter setters, refetch actions

### A.6 Composables

- **Files**: `composables/useSkillFilters.ts`, `composables/useRunFilters.ts`, `composables/useCardState.ts`
- **Content**:
  - `useSkillFilters` — client-side filter + debounced search; reactive `filteredSkills`
  - `useRunFilters` — form state + change emitter; NOT reactive to list (server-side re-fetch)
  - `useCardState` — helper that maps `{data, loading, error}` to UI state literal `"normal"|"loading"|"empty"|"error"`
- **Verification**: Vitest unit tests

### A.7 Adoption Metrics + Stage Coverage

- **Files**: `AdoptionMetricsCard.vue`, `MetricTile.vue`, `StageCoverageCard.vue`, `StageCoverageChain.vue`
- **Content**: per design doc §2.2 – §2.4
- **States**: loading skeleton; per-tile error isolation; page-level error fallback
- **Verification**: component tests in all 5 states

### A.8 Skill Catalog + Detail

- **Files**: `SkillCatalogCard.vue`, `SkillRow.vue`, `SkillFilters.vue`, `SkillDetailPanel.vue`
- **Content**: per design doc §2.5 – §2.7
- **States**: loading skeleton; empty state with helpful message; error with retry; detail 404 state
- **Verification**: component tests; router deep-link test for `/ai-center/skills/:skillKey`

### A.9 Run History + Detail

- **Files**: `RunHistoryCard.vue`, `RunRow.vue`, `RunFilters.vue`, `RunDetailPanel.vue`
- **Content**: per design doc §2.8 – §2.10
- **States**: loading; empty (no runs / filtered no match); error with retry; "Load more"; 404 detail
- **Verification**: component tests; router deep-link test; filter change triggers refetch with reset page

### A.10 View assembly + routing

- **Files**: `AiCenterView.vue`; router entry in `frontend/src/router/index.ts`
- **Content**: vertical stack + nested `<router-view>` slide-over slot; deep-link entry from Dashboard Learning node already in place
- **Verification**: navigating `/ai-center`, `/ai-center/skills/:skillKey`, `/ai-center/runs/:executionId` all render the correct state

### A.11 A11y pass

- Keyboard nav across all interactive controls
- Chips have aria-labels / tooltips
- Slide-over panels have focus trap + focus restore
- Status chips use `role="status"`
- **Verification**: manual tab-through, axe-core or equivalent linter pass

### A.12 Shared `SectionResult` promotion (conditional)

- If `SectionResult` / `SectionResultDto` is still in the dashboard feature folder, migrate it to `frontend/src/shared/api/` (FE) and `shared/dto/` (BE, in Phase B). Non-breaking move + re-export.
- **Verification**: dashboard slice still builds and tests pass

### A.13 Phase A acceptance

- [ ] `/ai-center` renders with mocks in all 5 states
- [ ] Deep-links to skill detail and run detail work
- [ ] Workspace switch re-fetches and re-renders
- [ ] `npm run dev` and `npm run build` both succeed
- [ ] `npm run test` passes for all new tests
- [ ] No ESLint/tsc warnings introduced

---

## Phase B — Backend (Claude Code)

### B.1 Flyway migration — schema

- **File**: `backend/src/main/resources/db/migration/V{n}__ai_center_schema.sql`
- **Content**: DDL per data-model doc §5; use `skill_key_code` column (not reserved word)
- **Verification**: new `MigrationTest` applies the migration from empty schema; `./mvnw test` green

### B.2 Flyway migration — seed (dev/local only)

- **Path**: `backend/src/main/resources/db/seed/V{n+1}__ai_center_seed.sql`
- **Content**: realistic seed matching Phase A mocks — 6+ skills, 20+ runs, stage coverage, policies, evidence links
- **Flyway config**: update `application-local.yml` / `application-dev.yml` to include `db/seed` in `spring.flyway.locations`. `application-prod.yml` must NOT.
- **Verification**: running with `local` profile populates data; running with `prod` profile does NOT attempt the seed migration

### B.3 JPA entities

- **Files**: `domain/aicenter/entity/Skill.java`, `SkillStage.java`, `Policy.java`, `SkillExecution.java`, `EvidenceLink.java`
- **Content**: per data-model doc §4. No Lombok; standard getters/setters.
- **Verification**: entity fields map cleanly to DDL in B.1

### B.4 Repositories

- **Files**: `SkillRepository.java`, `SkillExecutionRepository.java`, `PolicyRepository.java`
- **Content**:
  - Workspace-scoped query methods (`findByWorkspaceId...`)
  - Specification-based `findRuns(Specification, Pageable)` or JPQL with dynamic predicate
  - Aggregation queries for metrics (success rate, success-rate-30d, avg duration)
- **Verification**: `@DataJpaTest` with H2; workspace isolation assertion

### B.5 DTOs

- **Files**: per data-model doc §3 (Java records)
- **Verification**: record field names and shape match FE TS types

### B.6 Services

- **Files**: `MetricsService.java`, `SkillService.java`, `SkillExecutionService.java`
- **Content**: responsibilities per API guide §3.2; per-section error wrapping with `SectionResult`
- **Verification**: unit tests using mocked repos covering happy paths and edge cases

### B.7 Controller

- **File**: `AiCenterController.java`
- **Content**: per API guide §3.1
- **Verification**: MockMvc tests per API guide §3.4

### B.8 Exceptions

- **Files**: `SkillNotFoundException`, `SkillExecutionNotFoundException`; handlers added to `GlobalExceptionHandler`
- **Verification**: 404 MockMvc tests return the correct envelope

### B.9 Pagination validation

- `size` outside [1, 200] → `IllegalArgumentException` → 400 response
- **Verification**: MockMvc test for `size=201` and `size=0`

### B.10 Frontend wire-up (still Phase B, done by Codex)

- Set `VITE_USE_MOCK_API=false` (or remove) for default dev
- Verify `vite.config.ts` proxy covers `/api/v1/ai-center`; add if missing
- Run end-to-end: FE + BE both up, page renders BE-served data
- **Verification**: clicking a run row opens detail populated from BE

### B.11 Cross-domain service exposure

- Expose a small public Java API for other domains:
  - `SkillService.findByKey(workspaceId, key)` — used by Dashboard "active skill count" and Incident timeline
  - `SkillExecutionService.record(record)` — used by Incident when a skill execution occurs
  - `SkillExecutionService.findByTriggerSource(workspaceId, sourcePage)` — used by Incident for per-incident timeline card
- **Verification**: compile-only verification; actual consumption is the responsibility of consumer domain slices

### B.12 Phase B acceptance

- [ ] `./mvnw spring-boot:run -Dspring.profiles.active=local` starts cleanly
- [ ] All 6 endpoints return the correct envelope shapes per API guide §2
- [ ] `./mvnw test` passes (existing + new)
- [ ] FE switched from mocks; both `npm run dev` and `npm run build` succeed
- [ ] Workspace isolation verified via MockMvc test
- [ ] Pagination bounds enforced (400 on out-of-range)
- [ ] Seed data renders a populated, realistic `/ai-center` view
- [ ] No existing tests broken

---

## Risks and Mitigations

| Risk                                                            | Mitigation                                                                                                            |
| --------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------- |
| `SectionResult` scattered between dashboard + incident slices | Promote to `shared/` in Phase A (A.12); all slices re-export                                                        |
| Oracle reserved-word `key` collision                          | Use `skill_key_code` column + `@Column(name = ...)` (design doc §4)                                              |
| JSON columns not portable                                       | Store CLOB with app-side Jackson; never vendor JSON type                                                              |
| Consumer domains bypass service layer                           | Code review rule: domain-to-domain imports only from `*Service` or `*Dto`, never from `*Repository` or entities |
| Seed migration applied in prod                                  | Split Flyway `locations` by profile; verify CI deploys don't include `db/seed` folder classpath                   |
| FE breaks when BE adds enum values                              | Enum extensibility rule in API guide §6; FE renders unknown status as neutral chip                                   |

---

## Out of Scope (tracked for later slices)

- Policy editor UI + write endpoints
- Agent registry (abstraction above skills)
- Skill versioning UI
- Cost / token accounting
- Real-time push via SSE/WebSocket
- Right-side AI Command Panel UI (consumes these same backend models)
- Cross-workspace aggregation for platform admins
- Export to CSV/PDF (belongs to Report Center)
