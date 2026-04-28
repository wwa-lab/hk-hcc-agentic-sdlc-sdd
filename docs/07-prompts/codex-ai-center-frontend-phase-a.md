# Codex Prompt: AI Center Phase A — Frontend Implementation

## Task

Implement the AI Center frontend slice (`/ai-center`) with mocked data. This slice uses **Codex for both frontend and backend** (unlike earlier slices where Gemini handled FE). No backend dependency in Phase A.

## Read First (do not skip)

Read these docs fully before writing any code:

1. **Feature contract** — [`docs/03-spec/ai-center-spec.md`](../03-spec/ai-center-spec.md)
2. **Design decisions** — [`docs/05-design/ai-center-design.md`](../05-design/ai-center-design.md)
3. **API contract** (shapes your mocks must satisfy) — [`docs/05-design/contracts/ai-center-API_IMPLEMENTATION_GUIDE.md`](../05-design/contracts/ai-center-API_IMPLEMENTATION_GUIDE.md)
4. **Data model** (TypeScript types verbatim) — [`docs/04-architecture/ai-center-data-model.md`](../04-architecture/ai-center-data-model.md)
5. **Data flow** (state transitions, filter/refresh behavior) — [`docs/04-architecture/ai-center-data-flow.md`](../04-architecture/ai-center-data-flow.md)
6. **Task breakdown** (order of operations) — [`docs/06-tasks/ai-center-tasks.md`](../06-tasks/ai-center-tasks.md) § Phase A
7. **Stories** (acceptance criteria) — [`docs/02-user-stories/ai-center-stories.md`](../02-user-stories/ai-center-stories.md)
8. **Visual design system** — `design.md` (project root) — "Tactical Command" tokens, no-line rule, Inter + JetBrains Mono
9. **Existing FE conventions** — examine `frontend/src/features/dashboard/` and `frontend/src/features/incident/` and match their structure exactly

## Existing Frontend (DO NOT break)

- Shell already exists under `frontend/src/shell/` (navigation, workspace context, right AI panel rail, routing)
- Dashboard slice already references `/ai-center` route path (for the Learning SDLC node) — your slice must actually implement that route
- `SectionResult<T>` may already exist in the dashboard feature; promote it to `frontend/src/shared/api/` if not already shared, with a non-breaking re-export
- Reuse `fetchJson<T>` from `frontend/src/shared/api/` (exact path TBC — check existing usage)

## Scope (Phase A)

Build the frontend for AI Center under `frontend/src/features/ai-center/` per the file structure in `ai-center-design.md` §1.1. Deliverables:

- **Four main cards**: Adoption Metrics (5 tiles), Stage Coverage (11-node chain), Skill Catalog (filterable table), Run History (server-paginated table)
- **Two slide-over detail panels**: Skill Detail (`/ai-center/skills/:skillKey`), Run Detail (`/ai-center/runs/:executionId`)
- **Five card states**: Normal, Loading (skeleton), Empty, Error (card-level retry), Partial (per-section errors for Metrics)
- **Client-side filter + search** for Skill Catalog; **server-backed filter + pagination** for Run History (in Phase A these are mock-backed but the wiring must be server-shaped)
- **Mocks in `api/mocks.ts`** that exactly match the JSON examples in the API guide
- **Unit + component tests** (Vitest) per the task doc

## What NOT To Do

- Do NOT call any real backend; everything via `api/mocks.ts` gated by `VITE_USE_MOCK_API=true` (convention inherited from Dashboard slice)
- Do NOT modify shell code (`frontend/src/shell/`) beyond adding the `/ai-center` route children if not already there
- Do NOT redefine types that already exist (`Page<T>`, `SectionResult<T>`, workspace context)
- Do NOT import from other feature folders (`features/dashboard`, `features/incident`, etc.) — only from `shared/` and `shell/`
- Do NOT introduce new UI libraries
- Do NOT build the AI Command Panel UI — it's out of scope for this slice (even though it consumes the same backend models)
- Do NOT add policy edit controls — V1 policy view is read-only

## Acceptance Criteria

- [ ] `npm run dev` shows a populated `/ai-center` with realistic mock data
- [ ] Deep-link `/ai-center/skills/incident-diagnosis` opens the skill detail slide-over
- [ ] Deep-link `/ai-center/runs/<any-mock-run-id>` opens the run detail slide-over
- [ ] Workspace switch event from the shell causes AI Center to refetch and re-render
- [ ] Each card renders all 5 states (demo by toggling a dev query flag or by crafting corresponding mocks)
- [ ] `npm run build` succeeds with zero TS errors
- [ ] `npm run test` passes; coverage for store, composables, and each card
- [ ] No ESLint warnings introduced
- [ ] a11y: Tab-navigable, focus trap on slide-over, status chips have aria-labels
- [ ] Mock response shapes match the API guide examples exactly (this is what makes Phase B trivial)
