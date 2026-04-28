# Codex Prompt: Platform Center Phase A — Frontend Implementation

## Task

Implement the **Platform Center** frontend slice (`/platform`) with mocked data. This slice uses **Codex for both frontend and backend** (explicit user decision for platform-center, deviating from Gemini-for-FE default). No backend dependency in Phase A.

Platform Center is the admin console for six platform capabilities (PRD §11.13 + §12.1–§12.6):

1. Template Management
2. Configuration Management
3. Audit & Compliance
4. Access Control (RBAC)
5. Policy & Governance
6. Integration Framework

Access is restricted to users holding the `PLATFORM_ADMIN` role; non-admins are redirected to `/403`.

## Read First (do not skip)

Read these docs fully before writing any code:

1. **Feature contract** — [`docs/03-spec/platform-center-spec.md`](../03-spec/platform-center-spec.md)
2. **Design decisions** — [`docs/05-design/platform-center-design.md`](../05-design/platform-center-design.md)
3. **API contract** (shapes your mocks must satisfy) — [`docs/05-design/contracts/platform-center-API_IMPLEMENTATION_GUIDE.md`](../05-design/contracts/platform-center-API_IMPLEMENTATION_GUIDE.md)
4. **Data model** (TypeScript types verbatim) — [`docs/04-architecture/platform-center-data-model.md`](../04-architecture/platform-center-data-model.md)
5. **Data flow** (state transitions, inheritance, refresh) — [`docs/04-architecture/platform-center-data-flow.md`](../04-architecture/platform-center-data-flow.md)
6. **Architecture** (component boundaries, state ownership) — [`docs/04-architecture/platform-center-architecture.md`](../04-architecture/platform-center-architecture.md)
7. **Task breakdown** (order of operations) — [`docs/06-tasks/platform-center-tasks.md`](../06-tasks/platform-center-tasks.md) § Phase A
8. **Stories** (acceptance criteria) — [`docs/02-user-stories/platform-center-stories.md`](../02-user-stories/platform-center-stories.md)
9. **Requirements** — [`docs/01-requirements/platform-center-requirements.md`](../01-requirements/platform-center-requirements.md)
10. **Visual design system** — `design.md` (project root) — "Tactical Command" tokens, no-line rule, Inter + JetBrains Mono
11. **Project conventions** — [`CLAUDE.md`](../../CLAUDE.md) — especially Lessons #5 (point to repo, don't duplicate) and #8 (design doc completeness)
12. **Existing FE conventions** — examine `frontend/src/features/dashboard/`, `frontend/src/features/incident/`, and `frontend/src/features/ai-center/` and match their structure exactly

## Existing Frontend (DO NOT break)

- Shell already exists under `frontend/src/shell/` (navigation, workspace context, right AI panel rail, routing)
- Navigation entry `{ key: "platform", label: "Platform Center", path: "/platform" }` already seeded in the shell's nav config — your slice implements the route destination
- `features/platform/` directory already exists as an empty placeholder — populate it
- `SectionResult<T>`, `Page<T>`, and cursor helpers may already exist in `shared/api/` (from earlier slices) — reuse; do not redefine. If they still live only in a feature folder, promote non-breakingly.
- Reuse `fetchJson<T>` from `frontend/src/shared/api/`
- Reuse the forbidden / 403 view if the shell already exposes one; otherwise add a minimal `views/ForbiddenView.vue` under `shell/views/`

## Scope (Phase A)

Build the frontend for Platform Center under `frontend/src/features/platform/` per the file structure in `platform-center-design.md` §1.1. Deliverables:

- **Admin-only route guard** — `/platform/*` requires `PLATFORM_ADMIN`; non-admin → `/403`
- **Shell composition** — left sub-nav with 6 chips (Templates / Configuration / Audit / Access / Policy / Integration), breadcrumb, `<router-view>` for active sub-section
- **Six sub-sections**, each with: list view (paginated, filterable) + detail view (slide-over or panel per design) + edit form where applicable
- **Shared components**: `CatalogTable`, `CatalogFilters`, `ScopeChip`, `AuditEventRow`, `InheritanceChain`
- **Six states per card/view**: Normal, Loading (skeleton), Empty, Error (retry), Forbidden (non-admin fallback), plus state-machine-invalid feedback (Templates, Policies)
- **Client-side filter + search** for short catalogs (Templates, Policies, Connections); **server-backed cursor pagination** for Audit (which can be very large)
- **Mocks in `api/*/mocks.ts`** that exactly match JSON examples in the API guide §2.x — ≥ 12 templates, ≥ 20 configurations, ≥ 50 audit events, ≥ 8 role assignments, ≥ 10 policies, ≥ 6 connections
- **Plaintext-credential guard**: credential picker shows `credentialRef` only — any plaintext `password`/`apiKey`/`token` in the component tree must fail a snapshot test
- **Unit + component tests** (Vitest) per the task doc

## What NOT To Do

- Do NOT call any real backend; everything via feature mock files gated by `VITE_USE_MOCK_API=true` (convention inherited from Dashboard / AI Center)
- Do NOT modify shell code (`frontend/src/shell/`) beyond: (a) registering the `/platform` route tree, (b) adding the guard, (c) adding a `/403` view if missing
- Do NOT redefine types that already exist (`Page<T>`, `SectionResult<T>`, workspace context, cursor helpers)
- Do NOT import from other feature folders (`features/dashboard`, `features/incident`, `features/ai-center`) — only from `shared/` and `shell/`
- Do NOT introduce new UI libraries
- Do NOT render or store plaintext credentials anywhere — credential fields accept/display `credentialRef` only
- Do NOT wire a live connection-test call — Phase A simulates outcomes via mock
- Do NOT build cross-workspace aggregation views (out of scope for V1)
- Do NOT hand-roll new design tokens — reuse the "Tactical Command" token set

## Acceptance Criteria

- [ ] `npm run dev` shows a populated `/platform` with realistic mock data across all 6 sub-sections
- [ ] Non-admin user hitting `/platform/*` is redirected to `/403`
- [ ] Deep-links for each sub-section work (e.g., `/platform/templates/:id`, `/platform/configurations/:id`, `/platform/audit/:id`, `/platform/access/:id`, `/platform/policies/:id`, `/platform/connections/:id`)
- [ ] Each sub-section renders all defined states (normal / loading / empty / error / forbidden)
- [ ] Template lifecycle: attempting Draft → Archived (invalid skip) surfaces a user-visible blocking feedback before any API call
- [ ] Policy lifecycle: same invalid-transition guard
- [ ] Configuration detail renders all 4 inheritance layers with effective-value highlight (Platform Default → Application Default → SNOW Group Override → Project Override)
- [ ] Access editor: attempting to revoke the final PLATFORM_ADMIN opens a blocking confirm dialog and does not dispatch the revoke
- [ ] Audit filter changes trigger refetch with cursor reset; "Load more" cursor works
- [ ] Connection "Test" action animates and surfaces simulated success/failure state
- [ ] No plaintext credential fields (`password`, `apiKey`, `token`) appear in the rendered DOM (snapshot test asserts)
- [ ] Workspace switch event from the shell selectively refetches the workspace-scoped sub-sections (Audit, Policy) without reloading the others
- [ ] `npm run build` succeeds with zero TS errors
- [ ] `npm run test` passes; coverage for stores, composables, and each sub-section's list/detail
- [ ] No ESLint warnings introduced
- [ ] a11y: tab-navigable, focus trap on slide-over and confirm dialogs, status chips have aria-labels, inheritance chain layers have `aria-describedby`, forbidden state has `role="alert"`
- [ ] Mock response shapes match the API guide examples exactly (this is what makes Phase B trivial)
