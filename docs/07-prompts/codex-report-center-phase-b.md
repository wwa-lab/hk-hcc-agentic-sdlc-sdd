# Codex Prompt: Report Center — Phase B (Backend + FE wire-up)

## Task

Implement the **Report Center** backend in the existing Spring Boot 3.x / Java 21 / JPA app, and wire the already-built Phase A frontend to it.

## Authoritative references (read these first)

Do not duplicate their contents. Read them and implement to them.

- Requirements: `docs/01-requirements/report-center-requirements.md`
- Spec (contracts, error codes, status codes): `docs/03-spec/report-center-spec.md`
- Architecture (components, flows, NFRs): `docs/04-architecture/report-center-architecture.md`
- Data flow: `docs/04-architecture/report-center-data-flow.md`
- Data model (DTOs, entities, DDL, type mapping): `docs/04-architecture/report-center-data-model.md`
- Design (file layout, permissions, tests): `docs/05-design/report-center-design.md`
- API contracts (JSON, error codes, versioning): `docs/05-design/contracts/report-center-API_IMPLEMENTATION_GUIDE.md`
- Tasks: `docs/06-tasks/report-center-tasks.md` — follow **Phase B1, B2, B3** sequentially

## Scope for this prompt

Execute **tasks B1..B33** in order. The three milestones are:

- **M1 (B1..B12)** — Catalog + Run + Scope auth + History read path + FE flip to live
- **M2 (B20..B27)** — Export (CSV + PDF) + Artifact storage + Audit + Retention sweep + FE export actions
- **M3 (B30..B33)** — History and Exports tabs on the catalog page

## Existing Backend (DO NOT break)

The backend already contains:

- `platform/workspace/` — `WorkspaceContextService` (use for scope resolution)
- `platform/navigation/` — `NavigationService` (add Reports nav entry)
- `shared/dto/ApiResponse` — response envelope `{ data, error }` (reuse)
- `shared/audit/AuditEventService` — emit `report.export` events via this module
- `shared/exception/GlobalExceptionHandler` — reuse for 4xx mapping
- Flyway migrations in `src/main/resources/db/migration/` — use the next unclaimed version

Do NOT modify those packages beyond:

- adding one nav entry via `NavigationService`
- adding `REPORTS_BASE = "/api/v1/reports"` to `ApiConstants`

## Tech stack

- Spring Boot 3.x, Java 21, Maven
- JPA / Hibernate
- H2 (local) / Oracle (prod)
- Flyway for ALL schema changes (CLAUDE.md rule 4)
- Apache Commons CSV for CSV generation
- OpenHTMLtoPDF for PDF generation
- Spring `@Async` for export worker

## Constraints

- Package-by-feature: everything new lives under `com.sdlctower.domain.reportcenter.*`.
- No Lombok.
- All DB changes via Flyway. **Do NOT** rely on `ddl-auto`.
- Report definitions are code (`ReportDefinition` implementations registered by Spring DI) — not DB rows.
- Section isolation: failures in chart / headline / drilldown return 200 with `SectionResult.error` set; only top-level failures return 4xx/5xx.
- `SectionResult<T>` — reuse the pattern from Dashboard. If no shared location exists yet, duplicate the record in `domain/reportcenter/dto/` (do NOT modify the Dashboard class).
- Every successful export MUST emit an audit event via the existing `AuditEventService`. If audit emit fails, the export is marked `failed`. "No audit, no export" is a hard invariant.
- CSV export: reject > 100k rows with 413 (`EXPORT_TOO_LARGE`).
- Download URLs: signed, TTL 15 minutes.
- Retention sweep: artifacts older than 7 days → `expired` + file purged; audit rows remain.

## What NOT To Do

- Do NOT implement a custom report builder (V2).
- Do NOT implement scheduled delivery / subscriptions (V2).
- Do NOT implement Excel / .xlsx export (V2).
- Do NOT implement Quality / Stability / Governance / AI Contribution categories (V2).
- Do NOT implement per-individual drilldown (V2).
- Do NOT introduce a new audit infrastructure.
- Do NOT modify shell / shared / platform code beyond the two allowlisted touches above.
- Do NOT modify existing tests.

## Frontend wire-up (after B1..B11 pass)

- `frontend/src/features/reportcenter/stores/reportCenterStore.ts` — flip `useMockData = false`.
- Vite proxy for `/api/v1/**` should already exist; add only if missing.
- Smoke test every screen against the live backend before moving to M2.

## Acceptance Criteria

### M1

- [ ] `./mvnw spring-boot:run -Dspring.profiles.active=local` starts clean
- [ ] Flyway migrations applied; all report-center tables + fact tables present
- [ ] `GET /api/v1/reports/catalog` returns 200 with 5 enabled efficiency reports
- [ ] `POST /api/v1/reports/eff.lead-time/run` returns 200 with all 3 sections
- [ ] 400 on invalid grouping, 403 on unauthorized scope, 404 on unknown reportKey
- [ ] `./mvnw test` passes
- [ ] Frontend renders against live backend end-to-end

### M2

- [ ] `POST /export?format=csv` and `?format=pdf` return 202 + `ExportJobDto`
- [ ] CSV export completes in < 10s p95, PDF in < 20s p95
- [ ] 413 on oversized CSV (> 100k rows)
- [ ] Every completed export produces a matching `audit_event` via the shared audit module
- [ ] Download links work and expire at TTL
- [ ] `ExportActions.vue` + `ExportJobToast.vue` drive the end-to-end flow

### M3

- [ ] `GET /history` returns only caller's runs, sorted desc, limit 50
- [ ] Clicking a history row opens the report with exact filters
- [ ] `GET /exports` list respects 7-day window; older records absent

## Versioning

All endpoints under `/api/v1/reports/**`. Breaking changes are not allowed; use `/api/v2/` if ever needed.

## Handoff

When all three milestones pass, open a PR summarizing the endpoints added, migrations applied, and the results of the acceptance checks above.
