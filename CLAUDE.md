# CLAUDE.md

## Project

Agentic SDLC Control Tower — AI-native enterprise software delivery control tower.

- Frontend: Vue 3 / Vite / Vue Router / Pinia / TypeScript
- Backend: Spring Boot 3.x / Java 21 / JPA / H2 (local) / Oracle (prod)
- Dev model: SDD (Spec Driven Development) — documents before code
- Frontend dev tool: Gemini
- Backend dev tool: Codex
- Claude Code: SDD pipeline, doc quality gates, review, orchestration

## Current State (2026-04-18)

All 13 SDLC slices are implemented. The app is a fully navigable control tower with frontend views, backend APIs, Flyway migrations (V1-V77), and seeded local data for every domain. Platform Center backend is the only remaining planned work.

### Implemented Slices

| Slice | Frontend Feature | Backend Domain | Migrations |
|-------|-----------------|----------------|------------|
| Shared App Shell | `shell/` | `shared/` | V1-V2 |
| Dashboard | `features/dashboard` | `domain/dashboard` | V3 |
| Requirement Management | `features/requirement` | `domain/requirement` | V5-V6 |
| Incident Management | `features/incident` | `domain/incident` | V4 |
| Team Space | `features/team-space` | `domain/teamspace` | V7-V8 |
| Project Space | `features/project-space` | `domain/projectspace` | V9-V11 |
| Project Management | `features/project-management` | `domain/projectmanagement` | V20-V26 |
| Design Management | `features/design-management` | `domain/designmanagement` | V30-V36 |
| Code & Build | `features/code-build-management` | `domain/codebuildmanagement` | V40-V47 |
| Testing Management | `features/testing-management` | `domain/testingmanagement` | V50-V53 |
| AI Center | `features/ai-center` | `domain/aicenter` | V60-V61 |
| Deployment Management | `features/deployment-management` | `domain/deploymentmanagement` | V70-V77 |
| Report Center | `features/reportcenter` | `domain/reportcenter` | V37-V39 |
| Platform Center | `features/platform` | -- (planned) | -- |

### Backend Conventions

- Package-by-feature: `com.sdlctower.domain.{slice}/{controller,service,persistence,dto,policy,...}`
- Bean name conflicts: when class names collide across slices (e.g. `CatalogService`), use `@Service("slicePrefixClassName")`
- All endpoints return `ApiResponse<T>` envelope; card-based views use `SectionResultDto<T>`
- Flyway migration numbering: V1-V11 (foundation), V20-V26 (project mgmt), V30-V36 (design), V37-V39 (reports), V40-V47 (code & build), V50-V53 (testing), V60-V61 (AI center), V70-V77 (deployment)
- Table name prefix: `dp_` for deployment management tables (avoids reserved word conflicts with `release`, `deploy`, etc.)

## General Execution Discipline

These rules adapt general LLM coding guidance to this repository. They are meant to reduce over-building, hidden assumptions, and noisy diffs. They should be applied with judgment: trivial tasks do not need ceremony, but non-trivial changes must be explicit and verifiable.

### 1. Think before coding

Before implementing, identify assumptions, unclear scope, and tradeoffs. If a request has multiple plausible interpretations, surface them instead of silently choosing one. If the simpler path is enough, say so. If the requested direction conflicts with the platform design principles or SDD contract, push back respectfully before coding.

### 2. Prefer the minimum sufficient change

Implement the smallest change that satisfies the request and the relevant SDD contract. Do not add speculative features, one-off abstractions, broad configurability, or defensive handling for scenarios that cannot occur in the current design. If a solution starts growing much larger than the problem, simplify before committing it.

### 3. Keep changes surgical

Touch only the files needed for the requested outcome. Do not refactor adjacent code, restyle unrelated files, or clean up pre-existing dead code unless explicitly asked. Match the local style even when a different style would be personally preferable. If a change creates unused imports, variables, or helpers, remove only the orphaned code created by that change.

### 4. Make success criteria verifiable

For multi-step tasks, define what "done" means and how each step will be checked. Prefer tests or concrete verification over visual inspection alone. For bug fixes, reproduce the failure or identify the failing contract before fixing it. For refactors, verify behavior before and after when practical.

### 5. Keep every changed line accountable

Every changed line should trace to one of: the user's request, an SDD requirement, an API/data contract, a failing test, or a necessary integration boundary. If a useful unrelated issue is discovered, mention it separately instead of folding it into the current diff.

## Platform Design Principles

These principles define the long-term operating model for Control Tower. They apply across Requirement, Design, Code, Test, Deploy, Incident, and future domains.

### 1. Control Tower is the control plane, not the content repository

Control Tower must not replace Jira, Confluence, GitHub, CI/CD systems, testing tools, deployment tools, or incident platforms. Its job is to index, display, review, approve, trace, and report freshness across those systems.

Control Tower should store metadata, references, comments, approvals, execution state, and traceability. It should not become a second canonical storage location for SDD Markdown document bodies.

### 2. External systems keep their source-of-truth roles

Jira and Confluence remain BAU business input systems. GitHub remains the source of truth for engineering artifacts and SDD documents. CI/CD, testing, deployment, and incident systems remain the source of truth for their operational facts.

Control Tower connects these sources through stable references instead of copying them into a new monolithic store.

### 3. GitHub `docs/` is the SDD artifact source of truth

Engineering-ready SDD artifacts are stored in the project repository under the root `docs/` folder. Control Tower reads document content from GitHub when rendering documents in the UI.

The platform may index document metadata such as repo, branch, path, commit SHA, blob SHA, document type, profile, status, and GitHub URL, but GitHub owns the Markdown body and version history.

### 4. Business review happens in Control Tower; engineering review happens in GitHub

BA, Business, and Product users should review, comment, approve, or request changes inside Control Tower. Developers and technical leads should review diffs, code, and document PRs in GitHub.

Both review surfaces refer to the same GitHub-backed document versions.

### 5. Comments and approvals must be version-bound

Every business comment, approval, or change request must be tied to the Git version that was reviewed, using commit SHA and blob SHA where available. Users may read the latest document from GitHub, but their decision is recorded against the exact version they saw.

This prevents comment drift and keeps reviews auditable when documents change later.

### 6. CLI agents are the execution plane

Repo-aware, code-aware, and long-running work should be executed by CLI agents, not by synchronous UI flows. Examples include reading a whole repo, generating or updating SDD docs, modifying code, running tests, producing PRs, and uploading execution outputs.

Control Tower should create requests, provide execution context, display status, and show artifacts. The agent performs the heavy work and writes results back through GitHub and platform callbacks.

### 7. MCP is an agent tool layer, not a UI dependency

Jira, Confluence, GitHub, and other BAU systems may be accessed through MCP connectors, but those connectors should normally live in the CLI agent runtime. Control Tower should not depend on MCP implementation details for its core UI.

Control Tower records source references and produces execution manifests. Agents use MCP, REST APIs, or internal connectors to resolve those references during execution.

### 8. Agents consume manifests, not guesses

Agents must not infer the latest sources, documents, or repo context on their own. Control Tower should generate an execution manifest that includes workspace/project identity, repo, branch/ref, SDD profile, source references, relevant GitHub documents, output expectations, and constraints.

The recommended model is "latest resolved, then pinned execution": resolve the latest approved or requested inputs at execution start, pin their versions in the manifest, then execute against those pinned versions.

### 9. SDD is profile-driven

Do not hard-code one Java-oriented SDD chain as the universal workflow. Different system types can define different SDD profiles, document chains, directory conventions, skill bindings, review gates, traceability rules, and tiering rules.

Examples:

- `standard-java-sdd`: Requirement, User Story, Spec, Architecture, Design, Tasks
- `ibm-i-sdd`: Requirement Normalizer, Functional Spec, Technical Design, Program Spec, File Spec, UT Plan, Test Scaffold, Review Reports

Profile metadata should let Control Tower render the right document stages and let CLI agents pick the right skill workflow.

### 10. Platform models should stay generic and lightweight

Day 1 platform primitives should be reusable across domains without becoming a full workflow engine:

- Source Reference
- Document Index
- Review / Comment
- Execution Manifest
- Artifact Link
- Freshness Status

Avoid building a complex DAG engine, custom document editor, GitHub replacement, broad permission DSL, or fully automated orchestration layer before the core operating model is proven.

### 11. Freshness is a first-class platform capability

Control Tower should make staleness visible across systems:

- Jira or Confluence changed after an SDD document was generated
- A GitHub document changed after business approval
- Code changed after a spec or test plan was approved
- Tests do not cover the latest requirement/spec version
- Deployment or incident evidence points to stale upstream artifacts

Freshness is one of the main reasons Control Tower exists; it should be modeled explicitly rather than treated as a UI detail.

### 12. Keep the system extensible without over-designing

The platform should use stable boundaries and minimal abstractions that can grow:

- BAU inputs: Jira, Confluence, uploads, KB
- SDD artifacts: GitHub `docs/`
- Execution: CLI agents and short LLM-backed tasks
- Review: Control Tower for business, GitHub for engineering
- Observability: AI Center / execution history / artifact and evidence links

Prefer small, composable contracts over a large orchestration framework. Build the common operating model first; automate more only when the real workflow requires it.

## Lessons Learned (Session 2026-04-18)

### 10. Bean name conflicts require explicit naming when class names collide across slices

**What happened:** Multiple slices define classes with the same simple name (`CatalogService`, `TraceabilityService`, `AiAutonomyPolicy`, `LogRedactor`). Spring's default bean naming (`catalogService`) causes `ConflictingBeanDefinitionException` at startup because the component scan finds two beans with the same name in different packages.

**Rule:** When adding a `@Service`, `@Component`, or similar annotation to a class whose simple name already exists in another domain package, always provide an explicit bean name prefixed with the slice: `@Service("deploymentCatalogService")`. Check for conflicts by searching `public class {ClassName}` across `src/main/java/com/sdlctower/domain/` before creating new beans.

### 11. Flyway migration version numbers must not collide with existing migrations

**What happened:** The tasks doc specified V60-V67 for Deployment Management, but V60-V61 were already taken by AI Center. Using duplicate version numbers causes Flyway to fail at startup.

**Rule:** Before assigning migration version numbers, always `ls backend/src/main/resources/db/migration/` to check existing versions. Use the next available range. Current allocation: V1-V11 (foundation), V20-V26 (project mgmt), V30-V36 (design), V37-V39 (reports), V40-V47 (code & build), V50-V53 (testing), V60-V61 (AI center), V70-V77 (deployment).

### 12. Use `dp_` table prefix for Deployment Management to avoid SQL reserved words

**What happened:** Table names `release`, `deploy`, and `application` are reserved words or common names in H2/Oracle. Using them directly causes SQL parsing errors.

**Rule:** Prefix all Deployment Management tables with `dp_` (e.g., `dp_release`, `dp_deploy`, `dp_application`). Other slices can use their own prefix if needed. Always test migrations on H2 before committing.

## Lessons Learned (Session 2026-04-15)

### 1. Search the full project before assuming file locations

**What happened:** When creating the Gemini prompt, I only referenced `docs/standard-sdd/projects/control-tower/05-design/design.md` (product module design) and missed `docs/standard-sdd/projects/control-tower/05-design/visual-design-system.md` (visual design system). The user had to correct me twice.

**Rule:** Before referencing any document, always `Glob` for all matching files across the entire project. Do not assume a file only exists in one location. Ask if ambiguous.

### 2. Do not assume tech versions — use what the user specifies

**What happened:** I defaulted to Java 17 in the backend prompt. The user's actual version is Java 21.

**Rule:** Never assume language/runtime versions. If the user has stated a version, use it exactly. If not stated, ask before proceeding.

### 3. Use package-by-feature for multi-domain systems, not package-by-layer

**What happened:** The initial backend structure used flat `controller/`, `service/`, `model/` packages. The user immediately identified this won't scale for a system with 13+ SDLC domains plus shared platform capabilities.

**Rule:** When the PRD or architecture describes multiple business domains with shared platform capabilities, always propose package-by-feature (`platform/`, `domain/`, `shared/`) from the start. Package-by-layer only works for trivially small projects.

### 4. All database schema changes must use Flyway migrations

**Rule:** Never rely on `ddl-auto: create-drop` or `update` for schema changes. All DB changes (tables, columns, indexes, seeds) must be recorded as Flyway migration scripts under `src/main/resources/db/migration/` following the naming convention `V{version}__{description}.sql`. `ddl-auto` is only acceptable for local H2 throwaway dev; production and shared environments must use Flyway exclusively.

### 5. Prompts for external tools should be lightweight pointers, not content duplication

**What happened:** The first Gemini prompt was a massive document that duplicated content already in the repo's design docs. The user wanted a short prompt that points to the existing files.

**Rule:** When creating prompts for external tools (Gemini, Codex, etc.), prefer short prompts that reference existing repo documents rather than duplicating their content. The external tool can read the files directly. Only inline content that cannot be found in the repo (e.g., acceptance criteria, scope boundaries, constraints).

## Lessons Learned (Session 2026-04-16)

### 6. Every SDD slice must produce a complete 6-doc set

**What happened:** The shared-app-shell slice was missing a per-slice requirements doc (`standard-sdd/projects/control-tower/01-requirements/shared-app-shell-requirements.md`) and a per-slice design doc (`standard-sdd/projects/control-tower/05-design/shared-app-shell-design.md`). The product-level `design.md` existed but it covers all modules, not just the shell. The user had to ask why these were missing.

**Rule:** When executing the SDD pipeline for any slice, always produce or verify **all 9 documents** (6 core + 3 supplementary):

**Core Documents (6):**

| # | Stage | File pattern | Content |
|---|-------|-------------|---------|
| 1 | Requirements | `standard-sdd/projects/control-tower/01-requirements/{slice}-requirements.md` | Requirements extracted from PRD with REQ-IDs and PRD section refs |
| 2 | User Stories | `standard-sdd/projects/control-tower/02-user-stories/{slice}-stories.md` | Agile stories with acceptance criteria |
| 3 | Spec | `standard-sdd/projects/control-tower/03-spec/{slice}-spec.md` | Implementation-facing contracts |
| 4 | Architecture | `standard-sdd/projects/control-tower/04-architecture/{slice}-architecture.md` | System context, components, data flow, state, integration |
| 5 | Design | `standard-sdd/projects/control-tower/05-design/{slice}-design.md` | Concrete APIs, file structure, data model, visual decisions, DB schema |
| 6 | Tasks | `standard-sdd/projects/control-tower/06-tasks/{slice}-tasks.md` | Phased implementation breakdown |

**Supplementary Artifacts (3):**

| # | Stage | File pattern | Content |
|---|-------|-------------|---------|
| 7 | Data Flow | `standard-sdd/projects/control-tower/04-architecture/{slice}-data-flow.md` | Runtime data flows, sequence diagrams, state machines, error cascade, refresh strategy |
| 8 | Data Model | `standard-sdd/projects/control-tower/04-architecture/{slice}-data-model.md` | Domain model ER diagram, frontend types, backend DTOs/entities, DB schema DDL, type mapping |
| 9 | API Guide | `standard-sdd/projects/control-tower/05-design/contracts/{slice}-API_IMPLEMENTATION_GUIDE.md` | Full endpoint contracts with JSON examples, backend/frontend implementation guide, testing contracts |

Before starting any implementation work, check that all 9 docs exist for the current slice. If any are missing, create them first. Do not start coding with an incomplete doc set.

### 7. Architecture docs must include Mermaid flow diagrams

**What happened:** The architecture doc was pure prose — no visual diagrams. The user asked why there were no flows. Architecture without diagrams is incomplete.

**Rule:** Every architecture doc must include at minimum these Mermaid diagrams:

1. **System context** — actors, systems, databases, and their relationships
2. **Component breakdown** — shell/module components and how they compose
3. **Data flow** — sequence diagram showing the runtime flow
4. **State boundaries** — what state lives where
5. **Integration** — how frontend and backend connect

Use only Mermaid 8.x-compatible syntax: `graph LR/TD/TB`, `sequenceDiagram`, `flowchart`. Do NOT use `C4Context`, `direction` inside subgraphs, `[( )]` cylinder notation, or `→` in node text — these require Mermaid 10+ and break in GitHub/IDE renderers.

### 8. Design docs must include concrete implementation details

**Rule:** A per-slice design doc is not a copy of the architecture. It must include:

- File structure map (actual paths)
- Component API contracts (props, inputs, sources)
- Data model (frontend types AND backend entity/table mapping)
- API contracts (endpoints, request/response JSON, status codes)
- Visual design decisions (tokens, typography, animation specifics)
- Database schema (DDL with column types)
- Error and empty state design
- Integration boundary diagram

### 9. Every slice must have data-flow, data-model, and API implementation guide

**What happened:** The dashboard and shared-app-shell slices had all 6 core SDD docs but were missing the 3 supplementary artifacts: `data-flow.md`, `data-model.md`, and `API_IMPLEMENTATION_GUIDE.md`. The user had to point this out after reviewing the folder structure.

**Rule:** Beyond the 6 core docs, every slice must also produce these 3 supplementary artifacts:

1. **`04-architecture/{slice}-data-flow.md`** — Runtime data flows with Mermaid sequence diagrams covering: page load (Phase A + B), error isolation, navigation, state machine, refresh strategy, API client chain
2. **`04-architecture/{slice}-data-model.md`** — Domain model ER diagram, complete frontend type catalog, backend DTO definitions, DB schema DDL (current + future), frontend-to-backend type mapping table
3. **`05-design/contracts/{slice}-API_IMPLEMENTATION_GUIDE.md`** — Full endpoint contract with complete JSON request/response examples, error handling, backend implementation skeleton, frontend integration guide (API client + store + proxy config), testing contracts, versioning policy

These are not optional — they are the bridge between design docs and implementation. Without them, Gemini and Codex have to guess at contracts.
