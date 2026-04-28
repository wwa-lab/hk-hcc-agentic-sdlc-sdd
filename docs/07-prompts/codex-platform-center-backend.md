# Codex Prompt: Platform Center Phase B — Backend Implementation

## Task

Add the **Platform Center API** to the existing Spring Boot backend. Frontend already renders `/platform` with mocked data (Phase A). Your job: stand up the backend endpoints, persistence, migrations, and seed for all six platform capabilities so the frontend can switch from mocks to live with zero component changes.

Capabilities (PRD §11.13 + §12.1–§12.6):

1. Template Management
2. Configuration Management
3. Audit & Compliance
4. Access Control (RBAC) — single role `PLATFORM_ADMIN` in V1
5. Policy & Governance
6. Integration Framework

## Read First (do not skip)

1. **Feature contract** — [`docs/03-spec/platform-center-spec.md`](../03-spec/platform-center-spec.md)
2. **API contract** (endpoint shapes, controller skeletons, tests) — [`docs/05-design/contracts/platform-center-API_IMPLEMENTATION_GUIDE.md`](../05-design/contracts/platform-center-API_IMPLEMENTATION_GUIDE.md)
3. **Data model** (entities, DTOs, full DDL for V40–V47) — [`docs/04-architecture/platform-center-data-model.md`](../04-architecture/platform-center-data-model.md)
4. **Architecture** (package structure, ownership rules) — [`docs/04-architecture/platform-center-architecture.md`](../04-architecture/platform-center-architecture.md)
5. **Data flow** (atomic audit pattern, inheritance resolver, state machines) — [`docs/04-architecture/platform-center-data-flow.md`](../04-architecture/platform-center-data-flow.md)
6. **Design** (file layout, DB naming decisions) — [`docs/05-design/platform-center-design.md`](../05-design/platform-center-design.md)
7. **Task breakdown** — [`docs/06-tasks/platform-center-tasks.md`](../06-tasks/platform-center-tasks.md) § Phase B
8. **Requirements** — [`docs/01-requirements/platform-center-requirements.md`](../01-requirements/platform-center-requirements.md)
9. **Project conventions** — [`CLAUDE.md`](../../CLAUDE.md) — especially Lessons #3 (package-by-feature), #4 (Flyway only), #5 (point to repo), #6 (9-doc set)
10. **Existing BE slices to mirror** — look at `backend/src/main/java/com/sdlctower/domain/aicenter/`, `.../dashboard/`, `.../incident/` for structure, test style, and shared infra (`shared/dto/ApiResponse.java`, `shared/exception/GlobalExceptionHandler.java`)

## Existing Backend (DO NOT break)

- `shared/dto/ApiResponse.java` — response envelope `{ data, error }` (reuse)
- `shared/exception/GlobalExceptionHandler`, `ResourceNotFoundException` (extend with platform-specific handlers)
- `platform/workspace/` — existing `WorkspaceContext` + `X-Workspace-Id` propagation (reuse; your new platform capabilities live in sibling packages under `platform/`, NOT `domain/`)
- `config/CorsConfig.java` — CORS for Vite (do not modify)
- Flyway migrations in `src/main/resources/db/migration/` — existing migrations run up to V36. Use **V40–V47** per the data-model doc, leaving V37–V39 reserved for intervening slices.
- Existing tests in `src/test/` must continue to pass
- Existing session / auth mechanism — reuse whatever supplies the current user's role claim; do not invent a new auth stack

## Tech Stack (unchanged)

| Layer | Technology |
|---|---|
| Framework | Spring Boot 3.x (**Java 21** — per CLAUDE.md Lesson #2) |
| Build | Maven |
| ORM | JPA / Hibernate |
| DB (local) | H2 in-memory (profile `local`) |
| DB (prod) | Oracle (profile `prod`) |
| Migrations | Flyway only — NO `ddl-auto: update` (per CLAUDE.md Lesson #4) |

## Scope (Phase B)

Build the backend for Platform Center under `backend/src/main/java/com/sdlctower/platform/` (note: under `platform/`, NOT `domain/` — Platform Center is a platform-level capability, not a business domain).

1. **Flyway migrations** (per data-model doc §5):
   - `V40__create_platform_template.sql`
   - `V41__create_platform_configuration.sql`
   - `V42__create_platform_audit.sql` (append-only)
   - `V43__create_platform_role_assignment.sql`
   - `V44__create_platform_policy.sql`
   - `V45__create_platform_connection.sql`
   - `V46__seed_platform_center_data.sql` — **placed under `db/seed/`**, included only in `local`/`dev` profiles' `spring.flyway.locations`; NEVER `prod`
   - `V47__reserve_platform_future_columns.sql` — forward-compat reserved columns under `db/migration/`
2. **JPA entities** (per data-model doc §4): `PlatformTemplate`, `PlatformTemplateVersion`, `PlatformConfiguration`, `PlatformAudit`, `PlatformRoleAssignment`, `PlatformPolicy`, `PlatformPolicyException`, `PlatformConnection`, `PlatformCredentialRef` — no Lombok
3. **Repositories** — scope-aware query methods; cursor-based pagination helpers; `AuditRepository` is append-only (no update/delete methods); `RoleAssignmentRepository.countActivePlatformAdmins()` for last-admin guard
4. **DTOs** as Java records matching FE TS types exactly (camelCase JSON)
5. **Services, one per capability**:
   - `TemplateService`, `TemplateVersionService` — lifecycle state machine (Draft → Published → Deprecated → Archived)
   - `ConfigurationService` + `InheritanceResolver` — 4-layer resolver (Platform → Application → SNOW Group → Project)
   - `AuditService` (read-side) + **`AuditWriter`** (write-side, `@Transactional(propagation = MANDATORY)`)
   - `AccessService` — last-admin guard
   - `PolicyService`, `PolicyExceptionService` — state machine (Draft → Active → Retired)
   - `ConnectionService`, `CredentialRefService` — connection test stub, credentialRef abstraction
6. **Controllers** — 32 endpoints per API guide §2; every method annotated `@RequireAdmin`
7. **RBAC**: `@RequireAdmin` annotation + `AdminAuthGuard` HandlerInterceptor (per API guide §7); registered AFTER `WorkspaceContextFilter`
8. **Atomic audit pattern**: every mutation calls `auditWriter.write(...)` inside the same transaction as the domain write; mutation failure rolls back audit row
9. **Pagination**: `CursorCodec` (opaque base64 + HMAC); `size` bounded to [1, 200] → 400 if out of range
10. **Exceptions + handlers**: `TemplateNotFoundException`, `InvalidTransitionException`, `ConfigurationNotFoundException`, `DuplicateScopeOverrideException`, `LastPlatformAdminException`, `PolicyNotFoundException`, `PolicyInUseException`, `ConnectionNotFoundException`, `ConnectionTestFailedException` — all wired into `GlobalExceptionHandler`
11. **ArchUnit rule**: assert that every mutating service method under `platform/*/service/` invokes `AuditWriter.write`
12. **Frontend wire-up**: switch `VITE_USE_MOCK_API` to `false`, verify `vite.config.ts` proxy covers `/api/v1/platform/*`, rerun FE against live BE

## Package Structure

```
backend/src/main/java/com/sdlctower/platform/
├── template/           (controller/, service/, repository/, entity/, dto/, exception/)
├── configuration/      (+ service/InheritanceResolver.java)
├── audit/              (+ service/AuditWriter.java — MANDATORY propagation)
├── access/             (+ service/LastAdminGuard inside AccessService)
├── policy/
├── integration/
└── shared/
    ├── auth/           (RequireAdmin.java, AdminAuthGuard.java)
    ├── cursor/         (CursorCodec.java)
    └── scope/          (ScopeLevel.java, ScopeContext.java)
```

Java package is `platform` (already exists; add sub-packages). Docs and URLs keep `platform-center` / `/platform`.

## DB Naming Rules (important)

- Oracle reserved words: use safer names — `config_key` (not `key`), `config_value` (not `value`), `event_action` (not `action`), `audit_when` (not `when`). Entity field names can stay idiomatic via `@Column(name = "...")`. JSON wire format follows API guide §2.x.
- JSON-shaped columns (policy rule body, template schema, audit payload): store as CLOB + Jackson in service layer. No vendor JSON types.
- Append-only tables: `PLATFORM_AUDIT` has no update/delete repository methods.
- Forward-compat reserved columns in V47: `deleted_at TIMESTAMP NULL`, `tenant_id VARCHAR(36) NULL`, `rev BIGINT DEFAULT 0` on all platform tables.

## Atomic Audit Pattern (must verify)

Every mutating service method must:

```java
@Transactional
public TemplateDto publish(UUID id) {
    PlatformTemplate t = repo.findById(id).orElseThrow(...);
    t.setStatus(TemplateStatus.PUBLISHED);
    t = repo.save(t);
    auditWriter.write(AuditEvent.of("TEMPLATE", id, "PUBLISH", ...));
    return TemplateDto.of(t);
}
```

`AuditWriter.write` is annotated `@Transactional(propagation = Propagation.MANDATORY)` — calling it without an active transaction throws `TransactionRequiredException`. If the outer `@Transactional` rolls back, the audit row rolls back with it.

## What NOT To Do

- Do NOT use `ddl-auto: update` or `create-drop` beyond local H2 exploration — all schema lives in Flyway migrations (CLAUDE.md Lesson #4)
- Do NOT put Platform Center entities under `domain/` — Platform Center is a platform capability (CLAUDE.md Lesson #3)
- Do NOT access platform repositories or entities from outside their sub-package — go through `*Service` or `*Dto`
- Do NOT allow writes to `PLATFORM_AUDIT` via any path other than `AuditWriter.write`
- Do NOT call `AuditWriter.write` outside an active `@Transactional` (enforced by `MANDATORY` propagation)
- Do NOT accept or return plaintext secrets in any connection endpoint — only `credentialRef`. Controller must reject any body field named `password`, `apiKey`, `token`, `secret`.
- Do NOT use Lombok — standard records / getters/setters
- Do NOT include the `db/seed` folder in `prod` Flyway `locations`
- Do NOT modify existing domain/shell code except: (a) promote shared DTOs to `shared/dto/` if needed (non-breaking), (b) add exception handler methods to the existing `GlobalExceptionHandler`, (c) register the new interceptor in `WebMvcConfig`, (d) wire frontend to live API
- Do NOT break existing tests
- Do NOT introduce a new auth framework — reuse the session/role mechanism already in the repo
- Do NOT ship cross-workspace aggregation endpoints (out of scope for V1)
- Do NOT implement real connector probes — V1 `Connection.test()` simulates outcomes (V2 wires real probes per connector type)

## Acceptance Criteria

- [ ] `./mvnw clean test` passes all tests (existing + new); ArchUnit rule asserting every mutating platform service method invokes `AuditWriter.write` passes
- [ ] `./mvnw spring-boot:run -Dspring.profiles.active=local` starts cleanly with V40–V47 applied and seed loaded
- [ ] All 32 endpoints return JSON matching API guide §2 examples (shapes verified by MockMvc tests)
- [ ] Every platform endpoint returns 403 with `FORBIDDEN` for a non-admin caller
- [ ] `GET /api/v1/platform/*?size=201` → 400 with `{ data: null, error: "size must be between 1 and 200" }`
- [ ] Revoking the final PLATFORM_ADMIN → 409 with `LAST_PLATFORM_ADMIN`
- [ ] Template lifecycle: Draft → Archived (skipping Published) → 409 `INVALID_TRANSITION`; Draft → Published → 200; Published → Deprecated → 200; Deprecated → Archived → 200
- [ ] Policy lifecycle: Draft → Retired (skipping Active) → 409 `INVALID_TRANSITION`
- [ ] Policy with active exceptions cannot be deleted → 409 `IN_USE`
- [ ] Creating a duplicate scope override → 409 `DUPLICATE_SCOPE_OVERRIDE`
- [ ] Configuration `InheritanceResolver` returns correct effective value across all 4 layers with provenance trail
- [ ] Atomic audit: a service mutation that throws after `auditWriter.write` leaves **no** audit row (verified by integration test)
- [ ] `PLATFORM_AUDIT` append-only: repository has no `save`-on-update or `delete` methods (compile + reflective test)
- [ ] Connection endpoint rejects any body containing `password`, `apiKey`, `token`, or `secret`; persisted rows contain no secret columns
- [ ] `Connection.test` returns a simulated outcome and updates `lastCheckedAt` + `healthStatus`
- [ ] Seed loads under `local`, skipped under `prod` (asserted by CI artifact lint)
- [ ] `MigrationTest` applies V40–V47 cleanly from empty schema on H2; DDL rehearsal on Oracle XE docker image documented
- [ ] Frontend `npm run dev` + `npm run build` still succeed; `/platform` now renders live backend data end-to-end across all 6 sub-sections
- [ ] No existing tests broken; no ESLint, tsc, or Spotless/Checkstyle warnings introduced
- [ ] Cross-capability services expose minimal public APIs documented in JavaDoc (e.g., `AuditWriter.write` consumable by future domain slices that want to emit platform-scoped audit rows)
