# Codex Prompt: AI Center Phase B — Backend Implementation

## Task

Add the **AI Center API** to the existing Spring Boot backend. Frontend already renders `/ai-center` with mocked data (Phase A). Your job: stand up the backend endpoints, persistence, migrations, and seed so the frontend can switch from mocks to live with zero component changes.

## Read First (do not skip)

1. **Feature contract** — [`docs/03-spec/ai-center-spec.md`](../03-spec/ai-center-spec.md)
2. **API contract** (endpoint shapes, controller skeleton, tests) — [`docs/05-design/contracts/ai-center-API_IMPLEMENTATION_GUIDE.md`](../05-design/contracts/ai-center-API_IMPLEMENTATION_GUIDE.md)
3. **Data model** (entities, DTOs, DDL) — [`docs/04-architecture/ai-center-data-model.md`](../04-architecture/ai-center-data-model.md)
4. **Architecture** (package structure, ownership rules) — [`docs/04-architecture/ai-center-architecture.md`](../04-architecture/ai-center-architecture.md)
5. **Design** (file layout, DB naming decisions) — [`docs/05-design/ai-center-design.md`](../05-design/ai-center-design.md)
6. **Task breakdown** — [`docs/06-tasks/ai-center-tasks.md`](../06-tasks/ai-center-tasks.md) § Phase B
7. **Project conventions** — [`CLAUDE.md`](../../CLAUDE.md) — especially Lessons #3 (package-by-feature), #4 (Flyway only)
8. **Existing BE slices to mirror** — look at `backend/src/main/java/com/sdlctower/domain/dashboard/` and `.../incident/` for structure, test style, and existing shared infra (`shared/dto/ApiResponse.java`, `shared/exception/GlobalExceptionHandler.java`, `platform/workspace/`)

## Existing Backend (DO NOT break)

- `shared/dto/ApiResponse.java` — response envelope `{ data, error }` (reuse)
- `shared/exception/GlobalExceptionHandler`, `ResourceNotFoundException` (extend)
- `platform/workspace/` — WorkspaceContext, plus the `X-Workspace-Id` propagation mechanism (reuse exactly)
- `config/CorsConfig.java` — CORS for Vite (do not modify)
- Flyway migrations in `src/main/resources/db/migration/` — your new files use the **next available version number**
- Existing tests in `src/test/` must continue to pass
- `dashboard/dto/SectionResultDto.java` — if it exists only in the dashboard package, promote it to `shared/dto/` as a non-breaking move and re-export. If already in `shared/`, import from there.

## Tech Stack (unchanged)

| Layer | Technology |
|---|---|
| Framework | Spring Boot 3.x (Java 21) |
| Build | Maven |
| ORM | JPA / Hibernate |
| DB (local) | H2 in-memory (profile `local`) |
| DB (prod) | Oracle (profile `prod`) |
| Migrations | Flyway (no `ddl-auto: update`) |

## Scope (Phase B)

Build the backend for AI Center under `backend/src/main/java/com/sdlctower/domain/aicenter/`:

1. **Flyway migrations**:
   - `V{n}__ai_center_schema.sql` (DDL per data-model doc §5)
   - `V{n+1}__ai_center_seed.sql` under `db/seed/` — included via `spring.flyway.locations` only for `local` and `dev` profiles, NEVER `prod`
2. **Entities** (Skill, SkillStage, Policy, SkillExecution, EvidenceLink) — package `entity/`, no Lombok
3. **Repositories** with workspace-scoped query methods + dynamic filter predicate for runs
4. **DTOs** as Java records matching TS types exactly (camelCase JSON)
5. **Services** — `MetricsService`, `SkillService`, `SkillExecutionService`
6. **Controller** — six GET endpoints per API guide §2
7. **Exceptions** — `SkillNotFoundException`, `SkillExecutionNotFoundException`, plus handlers in `GlobalExceptionHandler`
8. **Tests** — MockMvc for every endpoint + workspace isolation + pagination bounds + 404s; `@DataJpaTest` for repos; `MigrationTest` for Flyway
9. **Frontend wire-up** — switch `VITE_USE_MOCK_API` to `false` (or remove), verify `vite.config.ts` proxy covers `/api/v1/ai-center`, rerun FE against live BE
10. **Cross-domain service surface** — expose `SkillService.findByKey(...)`, `SkillExecutionService.record(...)`, `SkillExecutionService.findByTriggerSource(...)` as public APIs that other domains (Incident, Dashboard, Requirement) can depend on

## Package Structure

```
backend/src/main/java/com/sdlctower/domain/aicenter/
├── AiCenterController.java
├── service/
│   ├── MetricsService.java
│   ├── SkillService.java
│   └── SkillExecutionService.java
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
├── dto/              (per data-model doc §3)
└── exception/
    ├── SkillNotFoundException.java
    └── SkillExecutionNotFoundException.java
```

Java package is `aicenter` (no hyphen). Docs and URLs keep `ai-center`.

## DB Naming Rule (important)

- `skill.key` column must be named `skill_key_code` in DDL to avoid Oracle reserved word (per design doc §4). Entity field stays `key` via `@Column(name = "skill_key_code")`. JSON wire format stays `key`.
- Use CLOB for JSON-shaped fields; serialize via Jackson in the service layer. No vendor JSON types.

## What NOT To Do

- Do NOT use `ddl-auto: update` or `create-drop` for anything other than throwaway local H2 exploration — all schema lives in Flyway migrations (per CLAUDE.md Lesson #4)
- Do NOT put any entities outside `domain/aicenter/` (per CLAUDE.md Lesson #3 — package-by-feature)
- Do NOT have other domains directly access AI Center's repositories or entities — they go through `SkillService` / `SkillExecutionService`
- Do NOT add write endpoints (V1 is read-only from AI Center's perspective; only the cross-domain service method `SkillExecutionService.record` is a write, and it is called from OTHER domains)
- Do NOT add Bean Validation (V1 is GET-only; validate query params inline with `IllegalArgumentException` → 400)
- Do NOT use Lombok — standard records / getters/setters
- Do NOT add authentication or role-based authorization (V1 relies on workspace membership only)
- Do NOT include the `db/seed` folder in `prod` Flyway locations
- Do NOT modify existing shell / dashboard / incident / requirement domain code except: (a) promote `SectionResultDto` to `shared/dto/` if needed (non-breaking), (b) add exception handler methods to the existing `GlobalExceptionHandler`, (c) wire frontend to live API
- Do NOT break existing tests

## Acceptance Criteria

- [ ] `./mvnw clean test` passes all tests (existing + new)
- [ ] `./mvnw spring-boot:run -Dspring.profiles.active=local` starts cleanly
- [ ] All six endpoints return JSON matching the examples in the API guide §2 (shapes verified by tests)
- [ ] `GET /api/v1/ai-center/runs?size=201` → 400 with `{ data: null, error: "size must be between 1 and 200" }`
- [ ] `GET /api/v1/ai-center/skills/not-a-real-key` → 404 with `{ data: null, error: "Skill not found: not-a-real-key" }`
- [ ] Workspace isolation test: seeding `WS-A` and `WS-B` and requesting `WS-A` returns no `WS-B` rows on any endpoint
- [ ] Flyway migration applies cleanly from empty schema (via `MigrationTest`)
- [ ] Seed migration populates realistic demo data matching Phase A mock counts (≥ 6 skills, ≥ 20 executions, stage coverage ≥ 8/11) when run under `local` profile; does NOT run under `prod`
- [ ] Frontend `npm run dev` + `npm run build` still succeed; `/ai-center` now renders live backend data end-to-end
- [ ] No ESLint, tsc, or Spotless / Checkstyle warnings introduced
- [ ] Cross-domain service methods exposed (compile-only); documented in JavaDoc
