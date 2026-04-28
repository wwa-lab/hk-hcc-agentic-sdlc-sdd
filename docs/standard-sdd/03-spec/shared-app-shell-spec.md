# Shared App Shell Spec

## Purpose

This spec defines the implementation-facing contract for the shared shell used
by Round 1 pages.

## 1. Scope

In scope:

- shared shell layout
- primary navigation region
- top context bar region
- page title and action region contract
- global utility entries
- persistent AI Command Panel region
- route-to-active-nav mapping

Out of scope:

- page-specific business modules
- backend APIs
- final visual token system
- responsive mobile behavior
- full AI execution logic

## 2. Layout Contract

The shell must expose these structural regions:

- `PrimaryNav`
- `TopContextBar`
- `PageHeader`
- `PageContent`
- `AiCommandPanel`

The shell is the outer frame for all Round 1 business pages.

## 3. Primary Navigation Contract

### 3.1 Required Entries

The navigation must render the following entries in fixed order:

1. Dashboard
2. Team Space
3. Project Space
4. Requirement Management
5. Project Management
6. Design Management
7. Code & Build
8. Testing
9. Deployment
10. Incident Management
11. AI Center
12. Report Center
13. Platform Center

### 3.2 Active State Rules

- Exactly one item may be active at a time
- Active state is determined by current route identity
- Active state uses the same visual pattern across pages
- V1 baseline pattern is a left-edge active indicator

### 3.3 Behavior Rules

- Navigation labels use PRD vocabulary
- Navigation items are shell-owned, not page-owned
- Navigation may later support permission filtering, but V1 foundation renders the full set

## 4. Context Bar Contract

### 4.1 Required Fields

The top context bar must support display of:

- `workspace`
- `application`
- `snowGroup`
- `project`
- `environment`

### 4.2 Display Rules

- Fields are displayed in a fixed order
- Each field has a stable label and value region
- Missing optional values render as empty-safe placeholders rather than breaking layout
- The context bar is visible on all Round 1 business pages

### 4.3 Data Contract

V1 contract:

```ts
type WorkspaceContext = {
  workspace: string;
  application: string;
  snowGroup?: string | null;
  project?: string | null;
  environment?: string | null;
};
```

## 5. Global Utility Contract

The shell must expose entry points for:

- global search
- notifications
- audit/history

These utilities are shell-level controls and are not page-specific actions.

## 6. AI Command Panel Contract

### 6.1 Structural Requirement

The shell must render a persistent right-side AI panel region in desktop layouts.

### 6.2 Content Zones

The panel should support these zones:

- summary
- reasoning
- evidence
- action or command input area

### 6.3 Ownership Rules

- The shell owns the panel container
- Pages provide panel content or content configuration
- AI Center is the global management page
- AI Command Panel is the current-page projection of AI state
- Pages with no page-scoped AI projection may suppress the panel; the main page content remains the source of truth for review, freshness, and execution status.

## 7. Page Integration Contract

Each page using the shell must be able to provide:

- page title
- page subtitle or context hint
- optional page-level actions
- main body content
- page-scoped AI panel content
- optional AI panel visibility

V1 contract:

```ts
type ShellPageConfig = {
  navKey: string;
  title: string;
  subtitle?: string;
  actions?: Array<{ key: string; label: string }>;
  showAiPanel?: boolean;
};
```

## 8. Error And Empty State Rules

The shell itself must support:

- loading context
- partial context data
- permission-restricted utilities
- missing AI panel content

Fallback behavior:

- shell structure still renders
- unavailable regions degrade with placeholder or disabled state
- page content failures do not remove the surrounding shell

## 9. Audit And Traceability Notes

- Shell-level utility actions should be auditable in later slices
- Route changes themselves do not require audit for V1 foundation
- Context display must remain consistent with workspace isolation rules

## 10. Risks And Open Questions

- Navigation label vocabulary: the PRD uses full forms (e.g., "Code & Build Management") while some downstream docs use shortened forms. The final nav labels should be confirmed before implementation.
- The AI Command Panel container is defined here but its content delivery contract (how pages provide content to zones) will need resolution before the first page-with-AI slice.
- WorkspaceContext in V1 uses static or mocked data. The backend API contract must be defined before the first data-connected slice.

## 11. Exit Criteria

This spec is ready for implementation when:

- the route list is accepted
- context field contract is accepted
- shell regions are accepted
- ownership boundary between shell and page content is accepted
