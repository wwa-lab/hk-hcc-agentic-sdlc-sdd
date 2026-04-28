# Requirement Management Stories

## Scope

This document defines the user stories for the Requirement Management page —
the upstream entry point of the Spec Driven Development pipeline in the
Agentic SDLC Control Tower.

## Traceability

- Requirements: [requirement-requirements.md](../01-requirements/requirement-requirements.md)
- PRD: [agentic_sdlc_control_tower_prd_v0.9.md](../01-requirements/agentic_sdlc_control_tower_prd_v0.9.md) §11.4, §13.1, §14

---

## Story S1: View Requirement List with Priority and Status Context

As a PM or BA,
I want to see a list of requirements with priority, status, and category,
so that I can assess the current requirement landscape and decide where to focus.

### Acceptance Criteria

- The requirement page shows a list/table of requirements in the current workspace
- Each requirement row displays: ID, title, priority (Critical / High / Medium / Low), status (Draft / In Review / Approved / In Progress / Delivered / Archived), category
- Each row shows linked story count and linked spec count as badge indicators
- Requirements are sortable by priority, status, recency, and title
- Requirements are filterable by status, priority, and category
- Critical-priority requirements are visually flagged with the crimson accent
- A summary strip at the top shows distribution by status (e.g., "5 Draft, 3 In Review, 8 Approved")
- Clicking a requirement row opens the full detail view

### Edge Notes

- V1 uses mocked requirement data; real data comes in Phase B
- Pagination is not required for V1 (expect <100 requirements per workspace)
- Column-level sorting should feel instant with mock data

### Requirement Refs

REQ-REQ-10, REQ-REQ-80, REQ-REQ-81

---

## Story S2: View Requirement Detail

As a PM,
I want to see full requirement details including description, acceptance criteria,
business context, stakeholders, and linked artifacts,
so that I can understand the requirement completely before making decisions.

### Acceptance Criteria

- Requirement detail view shows a header section with: ID, title, priority, status, category
- Description section displays the full requirement narrative with rich text formatting
- Acceptance criteria are listed as a numbered checklist
- Business justification section explains the business value and context
- Stakeholders are listed with their roles (owner, reviewer, approver)
- Linked artifacts section shows: derived user stories, linked specs, related requirements
- Metadata displays: created date, last modified, author, version
- Detail view renders inside the shared shell with context bar and AI panel

### Edge Notes

- V1 uses mocked detail data
- Rich text editing is out of scope for V1; display-only with markdown rendering
- Attachment support (files, images) is out of scope for V1

### Requirement Refs

REQ-REQ-11, REQ-REQ-12, REQ-REQ-60

---

## Story S3: View Requirement Kanban Board

As a delivery manager,
I want a kanban view of requirements by status,
so that I can track flow and identify bottlenecks in the requirement pipeline.

### Acceptance Criteria

- Kanban view displays columns for each status: Draft, In Review, Approved, In Progress, Delivered, Archived
- Each requirement card in the kanban shows: ID, title, priority badge, linked story count
- Cards are color-coded by priority (crimson for Critical, amber for High)
- Column headers show the count of requirements in each column
- Empty columns are displayed with a subtle placeholder
- Kanban view is accessible from the same page via a view toggle (list / kanban)
- The view toggle state persists during the session

### Edge Notes

- V1 kanban is read-only; drag-and-drop status transitions are V2
- Card layout should be compact enough to show 5-8 cards per column without scrolling
- Archived column may be collapsed by default

### Requirement Refs

REQ-REQ-20, REQ-REQ-80

---

## Story S4: View Priority Matrix

As a PM,
I want to see requirements plotted on an impact vs effort matrix,
so that I can make prioritization decisions based on value and cost.

### Acceptance Criteria

- Priority matrix displays a 2x2 grid with axes: Impact (vertical) and Effort (horizontal)
- Quadrants are labeled: Quick Wins (High Impact / Low Effort), Strategic (High Impact / High Effort), Fill-ins (Low Impact / Low Effort), Avoid (Low Impact / High Effort)
- Each requirement appears as a dot or mini-card in the appropriate quadrant
- Hovering a requirement dot shows a tooltip with: ID, title, priority, status
- Clicking a requirement dot navigates to the detail view
- Quadrant counts are displayed (e.g., "Quick Wins: 4")
- The matrix uses the shared design system color palette for quadrant backgrounds

### Edge Notes

- V1 uses mocked impact and effort scores; real scoring comes in Phase B
- The matrix is read-only; repositioning requirements by drag is out of scope
- If no requirements have impact/effort scores, show an empty state with guidance

### Requirement Refs

REQ-REQ-30, REQ-REQ-90

---

## Story S5: Derive User Stories from Requirement

As a PM,
I want to see derived user stories for each requirement and trigger AI-assisted
story generation,
so that I can efficiently break down requirements into implementable work items.

### Acceptance Criteria

- Requirement detail includes a "User Stories" card listing all derived stories
- Each story entry shows: ID, title, status, acceptance criteria count
- A parent-child relationship is visually indicated between requirement and stories
- A "Generate Stories" action button is available on the card
- Clicking "Generate Stories" triggers the `req-to-user-story` skill (mocked in V1)
- Generated stories appear in the list with a "Generated" badge
- If no stories exist, the card shows an empty state: "No stories yet — generate from this requirement"

### Edge Notes

- V1 mocks the generation flow; the action button triggers a state change showing pre-defined stories
- Actual AI skill execution integration is Phase B
- Story editing and refinement are out of scope for V1

### Requirement Refs

REQ-REQ-40, REQ-REQ-41, REQ-REQ-70

---

## Story S6: Link and Track Specs

As an architect or PM,
I want to see which specs are linked to each requirement and trigger spec generation,
so that I can ensure every requirement has a corresponding execution contract.

### Acceptance Criteria

- Requirement detail includes a "Linked Specs" card listing all associated specs
- Each spec entry shows: ID, title, status, version number
- Spec status uses visual indicators: Draft (neutral), In Review (amber), Approved (green), Blocked (crimson)
- A "Generate Spec" action button is available on the card
- Clicking "Generate Spec" triggers the `user-story-to-spec` skill (mocked in V1)
- The Spec card is visually emphasized as the execution hub per PRD section 7.6
- If no specs exist, the card shows: "No specs linked — generate from user stories"
- Each spec entry has a navigation link to the spec detail page (or "Coming Soon" placeholder)

### Edge Notes

- V1 mocks the generation flow; actual skill execution is Phase B
- Spec is the execution hub — its card should have slightly elevated visual prominence (e.g., border accent or icon)
- Spec versioning display is read-only in V1

### Requirement Refs

REQ-REQ-42, REQ-REQ-43, REQ-REQ-70

---

## Story S7: Trace Full SDLC Chain from Requirement

As a team lead,
I want to trace from any requirement through the full SDLC chain,
so that I can understand delivery status and identify gaps or blockers.

### Acceptance Criteria

- Requirement detail includes an "SDLC Chain" section showing the full 11-node chain
- Chain visualization shows: Requirement, User Story, Spec, Architecture, Design, Tasks, Code, Test, Deploy, Incident, Learning
- Each node displays a count of linked objects and a health indicator (green / amber / red / grey)
- The Spec node is always visually prominent (highlighted border or accent) per PRD section 13.1
- Clicking a chain node navigates to the corresponding module page (or "Coming Soon" placeholder)
- If no downstream objects exist for a node, it displays count "0" with grey styling
- Chain can be displayed as a horizontal flow or compact summary bar

### Edge Notes

- V1 uses mocked chain data; actual cross-object linking requires Phase B backend
- Navigation targets for unimplemented module pages show "Coming Soon" placeholder
- The full 11-node chain is the default; compression is not needed for the requirement page (compression applies to incident page per PRD section 13.1)

### Requirement Refs

REQ-REQ-60, REQ-REQ-61, REQ-REQ-62

---

## Story S8: View AI Analysis and Recommendations

As a PM,
I want AI-powered analysis of requirements including completeness scoring,
gap detection, and impact assessment,
so that I can improve requirement quality before they enter the delivery pipeline.

### Acceptance Criteria

- Requirement detail includes an "AI Analysis" card with AI-generated insights
- Card shows a completeness score (0-100%) with a visual progress indicator
- Missing elements are listed (e.g., "Missing acceptance criteria", "No business justification")
- Similar or duplicate requirement detection shows matches with similarity percentage
- Impact assessment describes which downstream artifacts and teams are affected
- AI recommendations are listed as actionable suggestions (e.g., "Add measurable acceptance criteria")
- Each recommendation has an "Apply" action (mocked in V1)
- Analysis card shows a "Last analyzed" timestamp

### Edge Notes

- V1 uses mocked AI analysis data; real AI integration is Phase B
- The completeness scoring algorithm is deterministic in V1 (based on field presence)
- Duplicate detection uses mocked similarity scores

### Requirement Refs

REQ-REQ-50, REQ-REQ-51, REQ-REQ-52

---

## Story S9: Use AI Command Panel for Requirement Operations

As a user,
I want the right-side AI Command Panel to provide context-aware assistance
on the current requirement page,
so that I can get intelligent help without leaving the page.

### Acceptance Criteria

- AI Command Panel is available on the requirement page via the shared shell
- Panel context automatically reflects the current page: requirement list or requirement detail
- On the list view, the panel offers: requirement landscape summarization, priority distribution analysis
- On the detail view, the panel offers: requirement refinement suggestions, acceptance criteria generation, impact analysis
- Panel actions are contextual — suggestions reference the currently viewed requirement
- Panel follows the shared design system treatment (dark panel, monospace output)
- Panel state resets when navigating between requirements

### Edge Notes

- V1 uses mocked panel responses; actual AI integration is Phase B
- The panel is a projection of AI Center capabilities, not an independent data source
- Panel interaction history is not persisted in V1

### Requirement Refs

REQ-REQ-53, REQ-REQ-90, REQ-REQ-91

---

## Story S10: Handle Loading, Error, and Empty States

As a user,
I want clear feedback when the requirement page is loading, encounters an error,
or has no data,
so that I always understand the current state and am never confused by a blank screen.

### Acceptance Criteria

- Loading state: skeleton placeholders shown while data loads, matching the card layout
- Error state: error message with retry option when API call fails
- Empty state: "No requirements yet" message with positive framing and guidance (e.g., "Create your first requirement to start the SDD pipeline")
- Partial state: if some cards load but others fail, show per-card error states (not full page error)
- All states follow the SectionResult pattern used across the platform
- All states use the shared design system visual treatment

### Edge Notes

- V1 uses mocked data so error states are only demonstrated, not triggered by real failures
- Empty state design should feel intentional and guide users toward action
- The SectionResult pattern (loading / loaded / error / empty) is shared with dashboard and incident

### Requirement Refs

REQ-REQ-92

---

## Story S11: Navigate Between List, Kanban, and Detail Views

As a user,
I want seamless navigation between list view, kanban view, and detail view,
so that I can efficiently switch perspectives without losing context.

### Acceptance Criteria

- List view is the default when navigating to the requirement page (`/requirements`)
- Kanban view is accessible via a view toggle or query parameter (`/requirements?view=kanban`)
- Clicking a requirement in either list or kanban view transitions to the detail view
- Detail view is accessible via URL with requirement ID (`/requirements/:id`)
- Detail view has a back-navigation to return to the previous view (list or kanban)
- Current workspace context is preserved across navigation
- View toggle state (list vs kanban) is preserved when returning from detail view
- Deep-linking to a specific requirement via URL is supported
- Page renders inside the shared shell with context bar and AI panel

### Edge Notes

- V1 uses Vue Router params for requirement detail routing
- Mobile-optimized navigation is out of scope
- Browser back/forward navigation should work correctly

### Requirement Refs

REQ-REQ-80, REQ-REQ-81, REQ-REQ-82

---

## Story S12: Filter and Search Requirements

As a user,
I want to filter by priority, status, category, and search by keyword,
so that I can quickly find specific requirements in a large list.

### Acceptance Criteria

- A filter bar is displayed above the requirement list with dropdown filters for: priority, status, category
- A text search input allows keyword search across requirement title and description
- Active filters are shown as removable chips/tags
- Filters are combinable (e.g., priority=Critical AND status=In Review)
- Filter state is synced to URL query parameters for shareability and bookmark support
- Clearing all filters resets to the default unfiltered view
- Filter results update in real-time as filters are applied (no submit button needed)
- Result count is displayed (e.g., "Showing 12 of 45 requirements")
- Filters apply to both list and kanban views

### Edge Notes

- V1 filters operate on mocked client-side data; server-side filtering is Phase B
- Advanced search (boolean operators, field-specific queries) is out of scope for V1
- Filter performance should be instant with <100 mocked requirements

### Requirement Refs

REQ-REQ-83, REQ-REQ-84

---

## Story S13: Select and View Active Pipeline Profile

As a team lead or PM,
I want to see which pipeline profile is active for the current workspace and understand how it shapes the requirement workflow,
so that I know which document chain, skills, and spec tiers apply to my team's requirements.

### Acceptance Criteria

- The requirement page header displays the active pipeline profile name (e.g., "Standard SDD", "IBM i")
- A profile info tooltip or expandable section shows: chain node summary, available skills, spec tiering (if applicable), traceability model
- The profile is inherited from Workspace → Project config; the page does not allow inline profile switching (that is a Platform Center action)
- When no custom profile is configured, the "Standard SDD" profile is used as the default
- The profile name is visible in the context bar or page header area

### Edge Notes

- V1 ships with two built-in profiles: "Standard SDD" and "IBM i" (read-only)
- The IBM i profile delegates to `ibm-i-workflow-orchestrator` as a single skill entry point
- Profile creation and editing is out of scope (Platform Center responsibility)
- Profile switching requires navigating to Workspace/Project settings

### Requirement Refs

REQ-REQ-120, REQ-REQ-121, REQ-REQ-123

---

## Story S14: View Profile-Adapted SDLC Chain and Skill Actions

As a PM or architect,
I want the SDLC chain visualization and skill action buttons to adapt to the active pipeline profile,
so that I see the correct chain shape and can trigger the right skills for my technology stack.

### Acceptance Criteria

- The SDLC chain card renders the chain nodes defined by the active profile (e.g., 11 nodes for Standard SDD, 10 nodes for IBM i)
- The execution hub node (Spec for Standard SDD, Program Spec for IBM i) is visually highlighted regardless of profile
- For Standard SDD: "Generate" action buttons show profile-specific skill names (e.g., "Generate Stories", "Generate Spec")
- For IBM i: a single "Send to Orchestrator" button is shown, which delegates to `ibm-i-workflow-orchestrator`
- Clicking a profile-specific action triggers the corresponding skill from the profile's skill binding
- Chain nodes that don't exist in the active profile are not shown (no empty placeholders)
- For the IBM i profile, the orchestrator's routing decision (path and tier) is displayed as a read-only indicator after invocation — users do not manually select the path

### Edge Notes

- V1 frontend renders chain nodes from mock profile data; backend profile API is Phase B
- Skill execution is stubbed in V1 (shows "Skill triggered" confirmation)
- The IBM i profile has a single skill binding (`ibm-i-workflow-orchestrator`); the orchestrator determines path and tier internally

### Requirement Refs

REQ-REQ-124, REQ-REQ-125, REQ-REQ-127

---

## Story S15: Use Spec Tiering for Legacy Pipeline Profiles

As a developer working on an IBM i enhancement,
I want to view the orchestrator-determined spec tier (L1 Lite / L2 Standard / L3 Full) for my change,
so that I can see how the documentation depth has been matched to the complexity of my change.

### Acceptance Criteria

- When the active profile defines spec tiering (e.g., IBM i profile), the orchestrator-determined tier is displayed as a read-only badge on the LinkedSpecsCard — users do not manually select the tier
- Tier values are: L1 (Lite — single-field change, ~10 sections), L2 (Standard — moderate logic change, ~16 sections), L3 (Full — new program or major redesign, all sections)
- The determined tier is shown on each linked spec card alongside the spec status
- When the active profile does NOT define spec tiering (e.g., Standard SDD), the tier badge is hidden
- Tier descriptions are available as tooltip text for reference

### Edge Notes

- V1 tier display is cosmetic (shown from mock data); actual tier determination requires IBM i orchestrator integration
- The `ibm-i-workflow-orchestrator` determines the tier based on input analysis (change scope, complexity, affected programs)
- Only profiles with `specTiering` configured show the tier UI

### Requirement Refs

REQ-REQ-126, REQ-REQ-128

---

## Story S16: Import Raw Business Input as Requirement

As a PM or business analyst,
I want to import raw business input (text, Excel, PDF, email, meeting transcript) into the Requirement Management page,
so that I can convert unstructured business needs into structured requirements without manual data entry.

### Acceptance Criteria

- The requirement list view shows a prominent "Import Requirement" action button in the page header
- Clicking "Import Requirement" opens an import panel (modal or right drawer)
- The import panel offers four input modes: Paste Text, Upload File, Email Import, Meeting Summary
- "Paste Text" provides a textarea for direct text input (meeting notes, Slack messages, verbal requirements)
- "Upload File" provides a drag-and-drop zone accepting: .xlsx, .csv, .pdf, .eml, .msg, .vtt, .txt, .png, .jpg, .jpeg, .webp
- File size limit: 10MB per file (V1)
- Only one file or text input can be processed at a time (no multi-file upload in V1)
- After providing input, the user clicks "Normalize with AI" to trigger the normalizer skill
- The import panel shows a loading state while the AI normalizer processes the input
- Error states are shown if file parsing fails or the normalizer returns an error

### Edge Notes

- V1 normalizer is stubbed (returns a hardcoded draft based on input type); actual AI normalization requires skill engine integration
- Email parsing (.eml/.msg) extracts subject as title candidate and body as description candidate
- Excel parsing: V1 treats the first sheet as input; multi-sheet selection is V2
- File upload uses browser File API; no server-side file storage in Phase A

### Requirement Refs

REQ-REQ-130, REQ-REQ-131, REQ-REQ-136

---

## Story S17: Review AI-Normalized Requirement Draft

As a PM,
I want to review the AI-extracted requirement draft and edit it before confirming,
so that I can correct any AI mistakes and ensure the requirement is complete and accurate.

### Acceptance Criteria

- After AI normalization, the import panel transitions to a "Review Draft" view
- The review view shows all extracted fields: title, priority (AI-suggested), category, description, acceptance criteria, business justification, assumptions, constraints
- AI-suggested values are visually marked (e.g., "AI suggested" badge) to distinguish from user-entered values
- Missing information flags are highlighted with amber warning indicators (e.g., "⚠ Stakeholder not identified", "⚠ Deadline not specified")
- Open questions extracted by the AI are listed (e.g., "Which currencies should be supported?")
- All fields are editable — the user can override any AI suggestion
- Three actions are available: "Edit Draft" (inline editing), "Confirm & Create" (creates the requirement), "Discard" (cancels import)
- On "Confirm & Create": the requirement is added to the list with status "Draft" and the original source is attached
- On "Discard": the draft is discarded, import panel returns to initial state
- For IBM i profiles: the review view shows Requirement Package format (CF-nn, CBR-nn, CE-nn candidate items) instead of standard format

### Edge Notes

- V1 review is frontend-only (draft stored in Pinia state, not persisted until confirmed)
- "Edit Draft" enables inline editing of all fields; no separate edit page
- AI confidence scores per field are V2 (V1 shows all suggestions equally)
- For batch import (Excel with multiple rows), each row gets its own review card

### Requirement Refs

REQ-REQ-132, REQ-REQ-133, REQ-REQ-134

---

## Story S18: Batch Import Requirements from Excel

As a PM,
I want to import multiple requirements at once from an Excel spreadsheet,
so that I can efficiently onboard a backlog of requirements without importing them one by one.

### Acceptance Criteria

- When an Excel file (.xlsx/.csv) is uploaded, the system parses rows and presents a preview table
- The preview table shows each row as a candidate requirement with extracted columns: title, priority, category, description
- The user can select/deselect individual rows for import using checkboxes
- A "Select All" / "Deselect All" toggle is provided
- Clicking "Normalize Selected" triggers AI normalization for each selected row
- After normalization, each row becomes a draft requirement in a batch review list
- The user can expand each draft to review/edit details (same review fields as S17)
- "Confirm All" creates all reviewed drafts as requirements with status "Draft"
- "Confirm Individual" allows confirming drafts one at a time
- Column mapping: the system attempts auto-detection of column headers (Title, Priority, Description, etc.); the user can manually remap columns if auto-detection fails
- Maximum 50 rows per batch import in V1

### Edge Notes

- V1 column auto-detection is heuristic (matches common header names); ML-based detection is V2
- Rows with parsing errors are flagged but do not block import of other rows
- Progress indicator shows "3 of 12 normalized..." during batch processing

### Requirement Refs

REQ-REQ-131, REQ-REQ-135

---

## Summary

| Story | Capability Domain | Primary Actor |
|-------|------------------|---------------|
| S1 | Requirement list & triage | PM / BA |
| S2 | Requirement detail | PM |
| S3 | Kanban board | Delivery manager |
| S4 | Priority matrix | PM |
| S5 | Story derivation & AI generation | PM |
| S6 | Spec linkage & generation | Architect / PM |
| S7 | SDLC chain traceability | Team lead |
| S8 | AI analysis & recommendations | PM |
| S9 | AI Command Panel | All users |
| S10 | States (loading/error/empty) | All users |
| S11 | Navigation & routing | All users |
| S12 | Filter & search | All users |
| S13 | Pipeline profile display | Team lead / PM |
| S14 | Profile-adapted chain & skills | PM / Architect |
| S15 | Spec tiering for legacy profiles | Developer |
| S16 | Import raw business input | PM / BA |
| S17 | Review AI-normalized draft | PM |
| S18 | Batch import from Excel | PM |
