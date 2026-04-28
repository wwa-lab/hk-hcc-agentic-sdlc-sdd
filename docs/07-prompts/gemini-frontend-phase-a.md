# Gemini Prompt: Phase A — Frontend Shell Implementation

## Task

Implement the frontend for **Agentic SDLC Control Tower** based on the design documents and HTML prototypes in this repo. This is a greenfield project with no existing code.

## Design Sources (read all before coding)

### 1. Visual Design System (PRIMARY — follow strictly)

- **`design.md`** (project root)

This is the "Tactical Command" design system. Key rules:
- Dark palette: `surface` (#0b1326), `surface-container-low` (#131b2e), `surface-container-high` (#222a3d)
- **No-Line Rule**: no 1px borders for layout — use background tonal shifts
- Typography: Inter for UI, JetBrains Mono for technical data/IDs
- 4px border radius, no large radii
- Status LEDs: Emerald (#4edea3), Crimson (#ffb4ab), Amber (#F59E0B)
- Glassmorphism for floating overlays
- "Machined" button style with gradient accents

### 2. Product & Module Design (structure and layout)

- **`docs/05-design/design.md`**

Pay attention to:
- §Implementation Scope Note — what is in scope for this slice
- §Module Design > Global App Shell — shell requirements and Vue 3 mapping
- §Vue 3 Component Seed — component and composable candidates

### 3. HTML Prototypes (visual reference, not pixel-binding)

- `docs/05-design/Control Tower.html`
- `docs/05-design/Project Space.html`
- `docs/05-design/Incident Command Center.html`
- `docs/05-design/Platform Center.html`

## Tech Stack

- Vue 3 (Composition API, `<script setup>`)
- Vite + TypeScript
- Vue Router + Pinia
- Custom CSS following `design.md` design system — no heavy UI library (Vuetify, Element Plus, etc.)

## Scope

Build **Phase A only** — frontend shell with mocked data, no backend:

1. Scaffold Vue 3 project under `frontend/`
2. Build the 5 shell components: `AppShell.vue`, `PrimaryNav.vue`, `TopContextBar.vue`, `GlobalActionBar.vue`, `AiCommandPanel.vue`
3. Wire all 13 navigation routes (4 Round 1 pages + 9 "Coming Soon" placeholders)
4. Implement `WorkspaceContext` and `ShellPageConfig` TypeScript types (contracts defined in `docs/03-spec/shared-app-shell-spec.md` §4.3 and §7)
5. Create `useWorkspaceContext()` composable with static mocked data
6. Apply the "Tactical Command" design system from root `design.md` — dark palette, no-line rule, Inter + JetBrains Mono, 4px radius, tonal depth

## What NOT To Do

- No backend, API calls, or database
- No mobile responsive layout
- No real AI behavior or chat logic
- No authentication or permission filtering
- No React, Angular, or Svelte

## Acceptance Criteria

- `npm run dev` starts the app
- All 13 routes navigable; exactly one nav item highlighted per route
- Top context bar shows 5 fields with graceful fallback for null optional values
- AI Command Panel visible as persistent right-side 320px rail
- `npm run build` succeeds with no TypeScript errors
