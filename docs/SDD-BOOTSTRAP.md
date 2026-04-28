# SDD Bootstrap

## Goal

Use the repo in a simple, repeatable `Spec Driven Development` loop before writing code. Every slice produces a complete 9-document set before implementation begins.

## Recommended Order

Follow this sequence for each Standard SDD slice under `standard-sdd/`:

1. `standard-sdd/01-requirements` — `{slice}-requirements.md`
2. `standard-sdd/02-user-stories` — `{slice}-stories.md`
3. `standard-sdd/03-spec` — `{slice}-spec.md`
4. `standard-sdd/04-architecture` — `{slice}-architecture.md` + `{slice}-data-flow.md` + `{slice}-data-model.md`
5. `standard-sdd/05-design` — `{slice}-design.md` + `contracts/{slice}-API_IMPLEMENTATION_GUIDE.md`
6. `standard-sdd/06-tasks` — `{slice}-tasks.md`
7. code

## Minimum Gate Before Code

Do not start implementation until the slice can answer all of these clearly:

- What problem is this slice solving
- Who is the user or actor
- What is in scope and out of scope
- What must happen in the happy path
- What must happen in error or empty states
- What data shape or state contract is needed
- Which module owns which responsibility
- How acceptance will be checked

## The 9-Document Set Per Slice

Every slice must produce all 9 documents before coding starts. This is enforced by CLAUDE.md rules #6 and #9.

### Core Documents (6)

| # | Stage | File pattern | Content |
|---|-------|-------------|---------|
| 1 | Requirements | `standard-sdd/01-requirements/{slice}-requirements.md` | Requirements extracted from PRD with REQ-IDs and PRD section refs |
| 2 | User Stories | `standard-sdd/02-user-stories/{slice}-stories.md` | Agile stories with acceptance criteria |
| 3 | Spec | `standard-sdd/03-spec/{slice}-spec.md` | Implementation-facing contracts |
| 4 | Architecture | `standard-sdd/04-architecture/{slice}-architecture.md` | System context, components, data flow, state, integration (with Mermaid diagrams) |
| 5 | Design | `standard-sdd/05-design/{slice}-design.md` | Concrete APIs, file structure, data model, visual decisions, DB schema |
| 6 | Tasks | `standard-sdd/06-tasks/{slice}-tasks.md` | Phased implementation breakdown |

### Supplementary Artifacts (3)

| # | Stage | File pattern | Content |
|---|-------|-------------|---------|
| 7 | Data Flow | `standard-sdd/04-architecture/{slice}-data-flow.md` | Runtime data flows, sequence diagrams, state machines, error cascade, refresh strategy |
| 8 | Data Model | `standard-sdd/04-architecture/{slice}-data-model.md` | Domain model ER diagram, frontend types, backend DTOs/entities, DB schema DDL, type mapping |
| 9 | API Guide | `standard-sdd/05-design/contracts/{slice}-API_IMPLEMENTATION_GUIDE.md` | Full endpoint contracts with JSON examples, backend/frontend implementation guide, testing contracts |

## Slice Roadmap

The roadmap follows the PRD §10 information architecture. Each slice produces the complete 9-document set before implementation.

| Order | Slice | Status | Docs |
|-------|-------|--------|------|
| 0 | `shared-app-shell` | Completed (docs + code) | 9/9 |
| 1 | `dashboard` | Completed (docs + code) | 9/9 |
| 2 | `requirement` | Completed (docs + code) | 9/9 |
| 3 | `incident` | Completed (docs + code) | 9/9 |
| 4 | `team-space` | Docs complete; implementation pending | 9/9 |
| 5 | **`project-space`** | **Docs complete; implementation pending** | 9/9 |
| 6 | `project-management` | Not started | 0/9 |
| 7 | `design-management` | Not started | 0/9 |
| 8 | `code-build-management` | Not started | 0/9 |
| 9 | `testing-management` | Not started | 0/9 |
| 10 | `deployment-management` | Not started | 0/9 |
| 11 | `ai-center` | Not started | 0/9 |
| 12 | `report-center` | Not started | 0/9 |
| 13 | `platform-center` | Not started | 0/9 |

## Current Slice: Project Space

Project Space is the single-project execution home — the contextual bridge between Team Space (Workspace-level operating home) and the lifecycle pages (Requirement / Design / Code / Test / Deploy / Incident). See PRD §11.3.

Documents for this slice:

- Requirements: [project-space-requirements.md](standard-sdd/01-requirements/project-space-requirements.md)
- Stories: [project-space-stories.md](standard-sdd/02-user-stories/project-space-stories.md)
- Spec: [project-space-spec.md](standard-sdd/03-spec/project-space-spec.md)
- Architecture: [project-space-architecture.md](standard-sdd/04-architecture/project-space-architecture.md)
- Data Flow: [project-space-data-flow.md](standard-sdd/04-architecture/project-space-data-flow.md)
- Data Model: [project-space-data-model.md](standard-sdd/04-architecture/project-space-data-model.md)
- Design: [project-space-design.md](standard-sdd/05-design/project-space-design.md)
- API Guide: [project-space-API_IMPLEMENTATION_GUIDE.md](standard-sdd/05-design/contracts/project-space-API_IMPLEMENTATION_GUIDE.md)
- Tasks: [project-space-tasks.md](standard-sdd/06-tasks/project-space-tasks.md)

## Active Add-On: SDD Knowledge Graph

SDD Knowledge Graph is an additive Requirement Control Plane capability. It
defines the document metadata, structured sync repository, graph artifacts,
Neo4j projection, backend graph API, and Requirement Management graph view for
decision support.

Documents for this add-on:

- Requirements: [sdd-knowledge-graph-requirements.md](standard-sdd/01-requirements/sdd-knowledge-graph-requirements.md)
- Stories: [sdd-knowledge-graph-stories.md](standard-sdd/02-user-stories/sdd-knowledge-graph-stories.md)
- Spec: [sdd-knowledge-graph-spec.md](standard-sdd/03-spec/sdd-knowledge-graph-spec.md)
- Architecture: [sdd-knowledge-graph-architecture.md](standard-sdd/04-architecture/sdd-knowledge-graph-architecture.md)
- Data Flow: [sdd-knowledge-graph-data-flow.md](standard-sdd/04-architecture/sdd-knowledge-graph-data-flow.md)
- Data Model: [sdd-knowledge-graph-data-model.md](standard-sdd/04-architecture/sdd-knowledge-graph-data-model.md)
- Design: [sdd-knowledge-graph-design.md](standard-sdd/05-design/sdd-knowledge-graph-design.md)
- API Guide: [sdd-knowledge-graph-API_IMPLEMENTATION_GUIDE.md](standard-sdd/05-design/contracts/sdd-knowledge-graph-API_IMPLEMENTATION_GUIDE.md)
- Tasks: [sdd-knowledge-graph-tasks.md](standard-sdd/06-tasks/sdd-knowledge-graph-tasks.md)

## Previous Slice: Team Space

Team Space is the Workspace-level operating home — the contextual bridge between Dashboard (cross-team, cross-project global) and Project Space (single-project execution). See PRD §11.2.

Documents for this slice:

- Requirements: [team-space-requirements.md](standard-sdd/01-requirements/team-space-requirements.md)
- Stories: [team-space-stories.md](standard-sdd/02-user-stories/team-space-stories.md)
- Spec: [team-space-spec.md](standard-sdd/03-spec/team-space-spec.md)
- Architecture: [team-space-architecture.md](standard-sdd/04-architecture/team-space-architecture.md)
- Data Flow: [team-space-data-flow.md](standard-sdd/04-architecture/team-space-data-flow.md)
- Data Model: [team-space-data-model.md](standard-sdd/04-architecture/team-space-data-model.md)
- Design: [team-space-design.md](standard-sdd/05-design/team-space-design.md)
- API Guide: [team-space-API_IMPLEMENTATION_GUIDE.md](standard-sdd/05-design/contracts/team-space-API_IMPLEMENTATION_GUIDE.md)
- Tasks: [team-space-tasks.md](standard-sdd/06-tasks/team-space-tasks.md)

## First Slice In This Repo

The first foundation slice was:

- `shared-app-shell`

Reference documents for this slice:

- Requirements: [shared-app-shell-requirements.md](standard-sdd/01-requirements/shared-app-shell-requirements.md)
- Stories: [shared-app-shell-stories.md](standard-sdd/02-user-stories/shared-app-shell-stories.md)
- Spec: [shared-app-shell-spec.md](standard-sdd/03-spec/shared-app-shell-spec.md)
- Architecture: [shared-app-shell-architecture.md](standard-sdd/04-architecture/shared-app-shell-architecture.md)
- Design: [shared-app-shell-design.md](standard-sdd/05-design/shared-app-shell-design.md)
- Tasks: [shared-app-shell-tasks.md](standard-sdd/06-tasks/shared-app-shell-tasks.md)

## How To Use This Starter

When starting a new slice:

1. Duplicate the structure of a completed slice (e.g., `requirement` or `team-space`) as a template
2. Replace scope, stories, and contracts with the new slice content
3. Make sure `03-spec` is the source of truth for implementation behavior
4. Produce all 9 documents before breaking work into tasks
5. Only then break work into tasks and start coding

## Simple Rule Of Thumb

If a coding decision cannot be traced back to a story or spec rule yet, the document alignment is not finished.

## Quality Gates (per CLAUDE.md Lessons Learned)

- **All 9 docs exist** before starting implementation (rule #6, #9)
- **Architecture docs include Mermaid diagrams** for system context, component breakdown, data flow, state boundaries, integration (rule #7)
- **Design docs include concrete implementation details** — file paths, component API contracts, data model, API contracts, DB schema DDL, error/empty states, integration boundary diagram (rule #8)
- **Mermaid 8.x-compatible syntax only** — no `C4Context`, no `direction` inside subgraphs, no `[( )]` cylinder notation, no `→` in node text (rule #7)
- **Package-by-feature** for backend domain modules (rule #3)
- **Flyway migrations** for all schema changes — no `ddl-auto: update` (rule #4)
- **Use what the user specifies** for tech versions (Java 21, not Java 17) (rule #2)
- **Search the full project** before assuming file locations (rule #1)
- **External-tool prompts are pointers**, not content duplication (rule #5)
