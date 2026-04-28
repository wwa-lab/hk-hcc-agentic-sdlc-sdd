# AI Center — Spec

## Purpose

Implementation-facing contract for the AI Center slice. This spec is the bridge between requirements/stories and the architecture/design docs — it defines concrete behaviors, inputs, outputs, constraints, and NFRs that Codex (FE + BE) must satisfy.

## Source

- Upstream: [ai-center-requirements.md](../01-requirements/ai-center-requirements.md), [ai-center-stories.md](../02-user-stories/ai-center-stories.md)
- PRD: [agentic_sdlc_control_tower_prd_v0.9.md](../01-requirements/agentic_sdlc_control_tower_prd_v0.9.md)

---

## 1. Feature Summary

AI Center is a workspace-scoped page mounted at `/ai-center` that surfaces the platform's AI capability layer through four cards:

1. **Adoption Metrics Strip** — 5 summary metrics + stage coverage.
2. **Skill Catalog** — filterable, searchable list of registered skills.
3. **Run History** — paginated, filterable timeline of skill executions.
4. **Policy & Detail drill-downs** — read-only policy per skill; per-run detail.

Phase A (frontend) renders against mocked data. Phase B (backend) introduces the `domain/ai-center/` backend package, Flyway migrations, and live API wiring. Both phases are delivered via **Codex**.

---

## 2. Functional Contracts

### 2.1 Page Bootstrap

- Route: `/ai-center` (registered under authenticated app shell).
- On mount, the page MUST:
  1. Read `workspaceId` from the shared workspace context store.
  2. Dispatch four parallel requests: `GET /metrics`, `GET /skills`, `GET /runs?page=1&size=50`, `GET /stage-coverage`.
  3. Render skeletons per card while each is pending.
  4. Apply per-card `SectionResult<T>` error isolation (reuse existing `SectionResultDto` / `SectionResult` types from the dashboard slice — do NOT redefine).

### 2.2 Skill Catalog

| Behavior | Contract |
|---|---|
| Listing | Server returns complete catalog for workspace (V1 ≤100 skills; no server pagination). |
| Sort | Default: last-executed desc, nulls last. User can sort by name asc, success-rate desc. |
| Filter | Client-side for V1: category, status, autonomy level, owner. Multi-select chips. |
| Search | Client-side substring match on `key` and `name`, case-insensitive. |
| Detail | Route `/ai-center/skills/:skillKey` opens detail view (same page, side panel OR new route — design decides; spec requires addressable URL). |

### 2.3 Run History

| Behavior | Contract |
|---|---|
| Listing | Server-paginated. Default page size 50, max 200. |
| Sort | Default: `startedAt desc`. No other sorts in V1. |
| Filter | Server-side: `skillKey` (multi), `status` (multi), `triggerSourcePage`, `startedAfter`, `startedBefore`, `triggeredBy` (`ai`\|`human`). |
| Detail | Route `/ai-center/runs/:executionId` opens run detail. |
| Refresh | Manual button only in V1. No polling. |

### 2.4 Adoption Metrics

| Metric | Formula (V1) | Source |
|---|---|---|
| AI Usage Rate | `executions / eligibleSdlcEvents` over window | `skill_executions` + seed workspace activity counter |
| Adoption Rate | `acceptedSuggestions / totalSuggestions` over window | `skill_executions` where `status IN ('succeeded','pending_approval','rejected')`; acceptance = `succeeded`+approved |
| Auto-Exec Success Rate | `autoSucceeded / autoAttempted` over window | `skill_executions` where `autonomyLevel >= L2` |
| Time Saved | Sum of `timeSavedMinutes` from each execution (seeded per skill definition) | `skill_executions.time_saved_minutes` |
| Stage Coverage | Distinct `stageKey`s with ≥1 active skill | join `skills` × `skill_stages` |

- Default window: **30 days**. Previous window: **preceding 30 days** for delta.
- All five returned in a single aggregated call: `GET /api/v1/ai-center/metrics?window=30d`.

### 2.5 Policy View

- Read-only in V1.
- Per skill, exposes: `autonomyLevel` (enum L0–L3), `approvalRequiredActions` (string[]), `authorizedApproverRoles` (string[]), `riskThresholds` (free-form JSON blob), `lastChangedAt`, `lastChangedBy`.
- No write endpoints in V1.

### 2.6 Evidence & Audit Links

- `evidenceLinks`: array of `{ title, type, sourceSystem, url }` returned inline in run detail.
- `auditRecordId`: string; frontend constructs audit URL via shared audit URL helper (stub in V1 — opens a placeholder modal with the ID if audit service not yet implemented).

---

## 3. API Surface (summary; full contracts in API_IMPLEMENTATION_GUIDE)

| Method | Path | Purpose |
|---|---|---|
| GET | `/api/v1/ai-center/metrics` | Adoption metrics + stage coverage |
| GET | `/api/v1/ai-center/skills` | Skill catalog |
| GET | `/api/v1/ai-center/skills/{skillKey}` | Skill detail (incl. current policy) |
| GET | `/api/v1/ai-center/runs` | Paginated run history |
| GET | `/api/v1/ai-center/runs/{executionId}` | Run detail |

- All responses use the shared `ApiResponse<T>` envelope (`{ data, error }`).
- Top-level aggregate (`/metrics`) uses per-section `SectionResult<T>` for error isolation.
- All endpoints are GET-only in V1. No write endpoints.

---

## 4. Data Ownership

Per CLAUDE.md Lesson #3 and the user's confirmed scoping choice, the `domain/ai-center/` package OWNS the following entities. Other slices may READ via service methods or DTO translation, never direct JPA access.

| Entity | Owner | Read by |
|---|---|---|
| `Skill` | `domain/ai-center/` | Dashboard (count), Incident (skill timeline), Report Center (future) |
| `SkillStage` (bridge) | `domain/ai-center/` | Dashboard (stage coverage) |
| `SkillExecution` | `domain/ai-center/` | Incident (timeline card), Requirement (AI-assist log), Dashboard (activity feed) |
| `Policy` | `domain/ai-center/` | future policy editor slice |
| `EvidenceLink` | `domain/ai-center/` (value-object table) | Incident run detail |

`Audit` records remain owned by `shared/audit/` (future); AI Center only references `auditRecordId`.

---

## 5. Persistence

- Flyway migrations under `src/main/resources/db/migration/V{n}__ai_center_*.sql`.
- No use of `ddl-auto: update` for these tables (per Lesson #4).
- V1 seed data inserted via an `afterMigrate__ai_center_seed.sql` callback only for the `local` profile OR via a `@Profile("local")` DataLoader. See data-model doc.

---

## 6. Workspace Isolation

Every query MUST filter by `workspace_id`. Implementation:

- DB column `workspace_id VARCHAR(64) NOT NULL` on `skill`, `skill_execution`, `policy`.
- Repository methods accept `workspaceId` explicitly (no implicit thread-local in V1).
- Controller extracts `workspaceId` from the existing `WorkspaceContext` header/store mechanism established by the shell slice.

---

## 7. Non-Functional Requirements

| NFR | Target |
|---|---|
| API latency (p95) | `/metrics` ≤ 200 ms; `/skills` ≤ 150 ms; `/runs?size=50` ≤ 300 ms (seed dataset) |
| Page TTFMP | ≤ 1.2 s over localhost |
| Catalog render | ≤ 500 ms after API response for 100 skills |
| Run history page size | default 50, max 200 |
| Build | `./mvnw test` green; `npm run dev` + `npm run build` green |
| Accessibility | All interactive controls keyboard-reachable; status chips have aria-labels |
| Browser support | Evergreen Chromium/Firefox/Safari per project default |

---

## 8. Error Handling Contract

- **Top-level API failure** (network / 5xx) → `ApiResponse.error = "..."`, `data = null`. Page shows global error with retry.
- **Per-section failure** (inside `/metrics`) → `SectionResult.error`, sibling sections still render.
- **Per-card failure** (individual endpoint fails but others succeed) → only that card shows error, Retry control re-fetches just that endpoint.
- Error messages returned by the backend MUST NOT leak stack traces or internal identifiers; they are safe to display.
- Unknown `skillKey` or `executionId` → `404` with `ApiResponse.error = "Not found"`; FE shows dedicated not-found state.

---

## 9. Security and Audit

- V1 assumes authenticated workspace member has read access to all AI Center data (same model as Dashboard / Incident).
- Reads are NOT audited. Writes do not exist in V1. (The platform audit model still receives entries for every skill execution; AI Center simply references them.)
- No PII is stored in Skill / Execution records. Inputs/outputs are stored as JSON blobs; sensitive content is expected to be redacted by the skill itself before persistence (V1 seed data is clean).

---

## 10. Test Contract

### 10.1 Frontend (Codex Phase A)

- Unit tests with Vitest for:
  - Filter/search logic in catalog composable
  - `SectionResult` unwrapping helpers if new ones introduced
  - Mock API client happy-path and error-path
- Component tests for each card in the 5 states (Normal / Loading / Empty / Error / Partial).

### 10.2 Backend (Codex Phase B)

- MockMvc tests for every endpoint covering:
  - 200 happy path with seed data
  - 404 on unknown `skillKey` / `executionId`
  - Workspace isolation (request with workspace `WS-A` returns no `WS-B` rows)
  - Pagination bounds (`size=200` OK, `size=201` rejected with 400)
  - Filter combinations on `/runs`
- JPA repository tests using `@DataJpaTest` with H2.
- Flyway migration verified to apply cleanly from empty schema.

---

## 11. Phase Boundaries

| Phase | Scope | Tool |
|---|---|---|
| A | Frontend: routes, stores, API client (mock mode), components, styling, 5 states, tests | **Codex** |
| B | Backend: entities, migrations, DTOs, service, controllers, tests, FE switch from mock → live API | **Codex** |

Phase A MUST produce a working page with mocked data and no backend dependency. Phase B MUST NOT require FE component re-work beyond the API client + store swap (a-la the Dashboard slice pattern).

---

## 12. Explicit Constraints

- Do NOT use WebSocket / SSE in V1.
- Do NOT add role-based authz checks in V1 (relies on workspace membership only).
- Do NOT introduce a new response envelope; reuse `ApiResponse<T>` and `SectionResult<T>`.
- Do NOT use Lombok (project standard — Java 21 records preferred).
- Do NOT modify the shell (`frontend/src/shell/`) beyond adding the `/ai-center` route if it does not already exist (Dashboard slice already references `/ai-center`; verify).
- Do NOT redefine Skill/Execution types in other domains — import from `domain/ai-center/`.

---

## 13. Traceability

| Spec Section | Stories / Requirements |
|---|---|
| §2.1 Page Bootstrap | US-AIC-60, US-AIC-61, US-AIC-63 / REQ-AIC-72, REQ-AIC-81 |
| §2.2 Skill Catalog | US-AIC-01 – US-AIC-04 / REQ-AIC-10 – REQ-AIC-13 |
| §2.3 Run History | US-AIC-10 – US-AIC-13 / REQ-AIC-20 – REQ-AIC-22, REQ-AIC-73 |
| §2.4 Adoption Metrics | US-AIC-20 – US-AIC-22 / REQ-AIC-30 – REQ-AIC-32 |
| §2.5 Policy View | US-AIC-30, US-AIC-31 / REQ-AIC-40 – REQ-AIC-42 |
| §2.6 Evidence & Audit | US-AIC-40, US-AIC-41 / REQ-AIC-50, REQ-AIC-51 |
| §4 Data Ownership | REQ-AIC-02, REQ-AIC-82 |
| §6 Workspace Isolation | REQ-AIC-81 |
| §7 NFR | REQ-AIC-83, REQ-AIC-84 |
