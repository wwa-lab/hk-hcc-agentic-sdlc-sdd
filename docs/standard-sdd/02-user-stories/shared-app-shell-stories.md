# Shared App Shell Stories

## Scope

This document defines the first simple SDD slice for implementation:

- the shared application shell used by Round 1 pages

It is the minimum foundation needed before deeper page modules are built.

## Story S1: Enter Any Business Page Through A Consistent Shell

As a user  
I want every major page to use the same shell structure  
So that I can navigate the product without relearning the frame each time

### Acceptance Criteria

- Every Round 1 page renders the same left navigation structure
- Every Round 1 page renders the same top context bar structure
- Every Round 1 page reserves a persistent right-side AI Command Panel region
- The active page is visually distinguishable in the left navigation
- The shell remains usable for both overview pages and detail-heavy pages

### Edge Notes

- Content inside the page body may differ by route
- The shell should not force each page to duplicate layout markup

## Story S2: Understand Current Context At A Glance

As a user  
I want the current business context to be visible at the top of the page  
So that I know which workspace and delivery scope I am looking at

### Acceptance Criteria

- The top context bar shows `Workspace / Application / SNOW Group / Project / Environment`
- Missing optional values are rendered safely without breaking layout
- Context display works consistently across Dashboard, Project Space, Incident Management, and Platform Center

### Edge Notes

- Organizations without SNOW Group usage may show an empty or fallback value
- The shell should distinguish context display from page-specific content

## Story S3: Reach Major Product Areas From Primary Navigation

As a user  
I want a stable primary navigation  
So that I can move across the major SDLC areas quickly

### Acceptance Criteria

- The navigation exposes the 13 PRD page entries
- The current route maps to exactly one active navigation item
- Navigation labels use the agreed product vocabulary from the PRD
- The shared shell allows later route linking without structural rewrites

### Navigation Set

- Dashboard
- Team Space
- Project Space
- Requirement Management
- Project Management
- Design Management
- Code & Build
- Testing
- Deployment
- Incident Management
- AI Center
- Report Center
- Platform Center

## Story S4: See AI As A Native Operating Surface

As a user  
I want a persistent AI Command Panel region in the shell  
So that AI feels like part of the product surface rather than a separate tool

### Acceptance Criteria

- The shell reserves a right-side panel region on supported desktop layouts
- The panel supports summary, reasoning, evidence, and action content areas
- The panel is layout-stable even when page body content changes
- The shell can render page-scoped AI content without redefining the panel structure on each page

### Edge Notes

- This foundation slice does not define full AI behavior
- This slice only defines the container, regions, and integration contract

## Story S5: Access Global Utility Actions

As a user  
I want consistent global utility entry points  
So that search, notifications, and audit are always reachable

### Acceptance Criteria

- The shell includes global search entry
- The shell includes notifications entry
- The shell includes audit/history entry
- These utilities appear in a consistent area of the shell

## Assumptions

- Desktop-first; mobile layout is out of scope for V1 shell
- Shell renders with static or mocked `WorkspaceContext`; no live backend dependency
- All 13 navigation entries are rendered for every user; permission filtering is deferred
- Vue 3 with Composition API and `<script setup>` is the frontend implementation target
- AI Command Panel is a layout container only; behavior and content logic are deferred

## Dependencies

- PRD V0.9 navigation vocabulary (§10 information architecture) is accepted
- `WorkspaceContext` field contract is accepted before context bar implementation
- Vue Router setup must exist before route-to-nav mapping can be wired
- Design HTML prototypes (Round 1) provide visual reference but are not binding

## Open Questions

- Should the nav labels use the full PRD vocabulary (e.g., "Code & Build Management") or shortened forms (e.g., "Code & Build")?
- Should the shell support a collapsed/minimized navigation state in V1?
- What is the fallback behavior when the AI Command Panel has no page-scoped content?

## Out Of Scope For This Slice

- page-specific business widgets
- live data integration
- permission-driven menu filtering
- mobile layout optimization
- full AI action workflow behavior
- design token finalization
