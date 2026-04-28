# Codex Prompt: Report Center — Phase A (Frontend, Mocked)

## Task

Implement the **Report Center** frontend feature module in the existing Vue 3 / Vite / Pinia / TypeScript app. This is **Phase A** — all network effects are stubbed via a mock client so the UI can be built and reviewed without the backend.

## Authoritative references (read these first)

Do not duplicate their contents. Read them and implement to them.

- Requirements: `docs/01-requirements/report-center-requirements.md`
- User stories (acceptance criteria): `docs/02-user-stories/report-center-stories.md`
- Spec (contracts): `docs/03-spec/report-center-spec.md`
- Architecture: `docs/04-architecture/report-center-architecture.md`
- Data flow: `docs/04-architecture/report-center-data-flow.md`
- Data model (types + mapping): `docs/04-architecture/report-center-data-model.md`
- Design (file layout, component APIs, visual states): `docs/05-design/report-center-design.md`
- API contracts (JSON shapes for mocks): `docs/05-design/contracts/report-center-API_IMPLEMENTATION_GUIDE.md`
- Tasks: `docs/06-tasks/report-center-tasks.md` — follow **Phase A** (A1..A8)

## Scope for this prompt

Execute **Phase A** exit criteria in `06-tasks` §A. In particular:

1. Create the feature module at `frontend/src/features/reportcenter/` using the file structure in design §2.
2. Implement `types.ts` exactly matching data-model §2.
3. Register routes and nav entry per design §1 and tasks A2 (do NOT modify shell components).
4. Implement `ReportCatalogView.vue` and `ReportDetailView.vue` with tabs (Catalog / History / Exports — only Catalog wired in Phase A).
5. Implement `FilterForm`, `HeadlineStrip`, `ChartSection`, `DrilldownSection`, `SectionError`, `SectionSkeleton`.
6. Implement the 5 chart components (Histogram, StackedBar, GroupedBar, Heatmap, HorizontalBar) using ECharts 5.x + `vue-echarts`.
7. Implement `reportCenterStore.ts` with `useMockData = true` by default and a state machine per data-flow §4.1.
8. Implement `api/reportCenterApi.ts` — the real API client (with `fetchJson`). It should be called when `useMockData = false`.
9. Implement `api/mockReports.ts` — returns data matching JSON examples in API guide §2 and §3. Include a toggle to simulate a section-level failure for the chart section (RPT-S22).
10. Add tests per design §10.1 and tasks A8.

## Constraints

- Vue 3 Composition API with `<script setup>` only.
- TypeScript strict mode; no `any` in public types.
- Do NOT use `localStorage` or `sessionStorage`. Filter persistence is in-memory per tab session.
- Use the existing `fetchJson` helper; do not create a new HTTP utility.
- Follow the existing project's lint and formatting rules.
- No color-only signals in charts (REQ-RPT-82).
- V2 categories (Quality / Stability / Governance / AI Contribution) must render as disabled "Coming soon" placeholders — no click-through.
- Do NOT create JPA entities, backend DTOs, or touch `backend/`.

## What NOT To Do

- Do NOT modify shell components (`frontend/src/shell/`).
- Do NOT modify shared types, stores, or the router aside from adding the two report routes.
- Do NOT add new global dependencies beyond those listed in architecture §3.
- Do NOT wire backend endpoints yet — leave `useMockData = true` on commit.

## Acceptance Criteria

- [ ] `/reports` renders the catalog with 5 Efficiency reports + 4 disabled "Coming soon" categories.
- [ ] `/reports/eff.lead-time` renders a mock report with headline, chart, drilldown.
- [ ] Each of the 5 report keys renders its correct chart type.
- [ ] Filter changes re-render the report; URL mirrors filter state.
- [ ] Section error simulation breaks only the affected section.
- [ ] `npm run lint && npm run build` passes.
- [ ] Unit tests pass.

## Handoff

On completion, flip `useMockData = false` only after Phase B1 backend is verified running locally — that step is covered by tasks B12.
