# Requirement Control Plane Tasks

## Objective

Implement the Requirement Control Plane adjustment as an additive evolution of
Requirement Management. The goal is to align the slice with the platform
principles: source references, GitHub-backed SDD docs, business review, SDD
profiles, agent manifests, and freshness.

## Traceability

- Requirements: [requirement-control-plane-requirements.md](../01-requirements/requirement-control-plane-requirements.md)
- Stories: [requirement-control-plane-stories.md](../02-user-stories/requirement-control-plane-stories.md)
- Spec: [requirement-control-plane-spec.md](../03-spec/requirement-control-plane-spec.md)
- Architecture: [requirement-control-plane-architecture.md](../04-architecture/requirement-control-plane-architecture.md)
- Data Flow: [requirement-control-plane-data-flow.md](../04-architecture/requirement-control-plane-data-flow.md)
- Data Model: [requirement-control-plane-data-model.md](../04-architecture/requirement-control-plane-data-model.md)
- Design: [requirement-control-plane-design.md](../05-design/requirement-control-plane-design.md)
- API Guide: [requirement-control-plane-API_IMPLEMENTATION_GUIDE.md](../05-design/contracts/requirement-control-plane-API_IMPLEMENTATION_GUIDE.md)

## Phase 0: Alignment and Contracts

- [x] Confirm this SDD package is accepted as the next-stage Requirement
      Management adjustment scope
- [x] Keep existing Requirement Management behavior during transition
- [x] Decide initial source providers for Day 1: manual URL metadata first,
      provider fetch second
- [x] Decide initial GitHub access path: GitHub app connector, PAT-backed
      backend gateway, or existing internal gateway
- [x] Decide default repo/branch resolution for a requirement/project
- [x] Define central SDD repo branch model: `main` as released baseline,
      project branch as in-flight review workspace
- [x] Define Central Knowledge Base repo as generated graph output, not a
      second SDD source of truth

## Phase 1: Data Model and Backend Foundations

- [x] Add Flyway migration for `project_sdd_workspace`
- [x] Add Flyway migration for `requirement_source_reference`
- [x] Add Flyway migration for `requirement_sdd_document_index`
- [x] Add Flyway migration for `requirement_document_review`
- [x] Add Flyway migration for `requirement_agent_run`
- [x] Add Flyway migration for `requirement_artifact_link`
- [x] Create source reference entity/repository/service/DTO
- [x] Create SDD document index entity/repository/service/DTO
- [x] Create document review entity/repository/service/DTO
- [x] Create agent run and artifact link entities/repositories/services/DTOs
- [x] Create SDD workspace entity/repository/DTO
- [x] Add freshness service projection

## Phase 2: Source References

- [x] Add `GET /requirements/{id}/sources`
- [x] Add `POST /requirements/{id}/sources`
- [x] Add `POST /requirements/sources/{sourceId}/refresh`
- [x] Add `SourceReferencesPanel.vue`
- [x] Add `SourceReferenceCard.vue`
- [x] Support Jira, Confluence, GitHub, KB, upload, and generic URL source
      types in DTOs
- [x] Add mocked source refs for DEV mode
- [x] Add section-level error and retry states

## Phase 3: GitHub SDD Document Index and Viewer

- [x] Add GitHub document gateway abstraction
- [x] Add `GET /requirements/{id}/sdd-documents`
- [x] Add `GET /requirements/documents/{documentId}`
- [x] Add `SddDocumentsPanel.vue`
- [x] Add `SddDocumentStageRow.vue`
- [x] Add `GitHubMarkdownViewer.vue`
- [x] Render profile expected stages with present/missing document state
- [x] Show GitHub URL, repo, path, commit SHA, and blob SHA
- [x] Show Application, SNOW Group, source repo, SDD repo, working branch, and
      Knowledge Base repo context in the document panel
- [x] Ensure Markdown content is fetched from GitHub and not treated as DB
      source of truth

## Phase 4: Business Review

- [x] Add `POST /requirements/documents/{documentId}/reviews`
- [x] Add `GET /requirements/{id}/reviews`
- [x] Validate commit SHA and blob SHA for review creation
- [x] Add `BusinessReviewPanel.vue`
- [x] Add `ReviewHistoryList.vue`
- [x] Support COMMENT, APPROVED, CHANGES_REQUESTED, REJECTED decisions
- [x] Mark approval stale when latest blob differs from reviewed blob
- [x] Add tests for version-bound review behavior

## Phase 5: Profile-Driven Stage Rendering

- [x] Extend frontend profile model with document stage definitions
- [x] Update Standard Java profile stages and path patterns
- [x] Rewrite IBM i profile based on `build-agent-skill` workflow:
      Requirement Normalizer, Functional Spec, Technical Design, Program Spec,
      File Spec, UT Plan, Test Scaffold, Spec Review, DDS Review, Code Review
- [x] Include IBM i BR-xx traceability and L1/L2/L3 tiering metadata
- [x] Make Requirement detail document stages entirely profile-driven
- [x] Remove hardcoded Java labels from new control-plane sections
- [x] Add `RequirementSkillFlowView.vue` for profile-owned skill inputs,
      skill outputs, skill dependencies, and document dependencies
- [x] Expand IBM i Skill & Document Flow from orchestrator-only to all 16
      `build-agent-skill` skills with flow-only source/code/report artifacts
- [x] Add Skill & Document Flow entry points from Requirement list and detail

## Phase 6: Agent Run Manifest

- [x] Add `POST /requirements/{id}/agent-runs`
- [x] Add `GET /requirements/agent-runs/{executionId}`
- [x] Add `POST /requirements/agent-runs/{executionId}/callback`
- [x] Generate manifest with requirement, profile, repo, branch/ref, source
      references, document references, output expectations, and constraints
- [x] Include source repo, central SDD workspace branch, and Knowledge Base
      graph context in the manifest
- [x] Pin source/document versions at manifest creation time
- [x] Keep agent run status available for backend audit and developer
      diagnostics instead of showing a standalone CLI Runs card in the
      BA-facing Requirement Detail page
- [x] Store artifact links from callbacks

## Phase 7: Freshness and Traceability

- [x] Add `GET /requirements/{id}/traceability`
- [x] Compute source-to-document freshness
- [x] Compute review-to-document freshness
- [x] Add `RequirementTraceabilityPanel.vue`
- [x] Use common freshness states:
      FRESH, SOURCE_CHANGED, DOCUMENT_CHANGED_AFTER_REVIEW, MISSING_DOCUMENT,
      MISSING_SOURCE, UNKNOWN, ERROR
- [x] Add visual freshness chips in source and document panels
- [x] Add tests for stale source and stale review scenarios

## Phase 8: Integration With Existing Requirement Features

- [x] Keep existing import/normalize flow operational
- [x] Reposition Normalize with AI as assisted intake
- [x] Add source reference creation from confirmed import draft
- [x] Transition Generate Stories / Generate Spec actions toward Request Agent
      Run in the new document panel
- [x] Keep Jira user story links visible while removing the legacy Linked Specs
      card from Requirement Detail; SDD/spec documents are represented by the
      GitHub SDD Documents panel
- [x] Render Jira stories as read-only Jira projections with source freshness,
      last synced metadata, Open in Jira, and Refresh from Jira actions
- [x] Remove the legacy SDLC Chain card from Requirement Detail; traceability is
      represented by Source References, GitHub SDD Documents, Business Review,
      and Review Readiness until a branch-aware delivery trace is introduced
- [x] Remove the legacy Analysis Snapshot card from Requirement Detail; AI
      analysis belongs in intake/normalize or agent output flows, not the
      document review surface
- [x] Update Requirement detail ordering to emphasize sources and GitHub docs

## Phase 9: Verification

- [x] Backend unit tests for services and freshness calculations
- [x] Backend controller tests for all new endpoints
- [x] Frontend store tests for source refs, document viewer, reviews, and agent
      runs
- [x] UI smoke test for Requirement detail with Standard Java profile
- [x] UI smoke test for Requirement detail with IBM i profile
- [x] Verify GitHub fetch section-level failure does not break page
- [x] Verify review decision is tied to commit/blob metadata
- [x] Verify agent callback creates artifact links and updates run status

## Phase 10: Knowledge Base Graph Publishing

- [x] Define Central Knowledge Base repo context in workspace metadata
- [x] Expose Knowledge Base repo, main branch, preview branch, and graph
      manifest path in SDD document index DTOs
- [ ] Add deterministic sync job that reads central SDD `main` and publishes
      Obsidian-compatible Markdown nodes plus `_graph/nodes.jsonl` and
      `_graph/edges.jsonl`
- [ ] Add project preview graph generation from SDD project branch to matching
      Knowledge Base preview branch
- [x] Add Requirement Management SDD Knowledge Graph view for profile document
      nodes, dependencies, and control-plane health metrics
- [ ] Wire Control Tower Knowledge Graph view to Neo4j-backed document nodes
      and cross-requirement graph navigation

## Phase 11: Document Instance Resolution

- [ ] Add `documentInstanceKey`, `titleSource`, `pathPattern`,
      `pathVariables`, and `unresolvedTokens` to SDD document DTOs and index
      persistence.
- [ ] Resolve profile path tokens from requirement metadata, source references,
      project metadata, and latest agent manifest before rendering missing
      documents.
- [ ] Add `stageGroups[].documents[]` response shape while preserving flattened
      `stages[]` during UI transition.
- [ ] Support one-to-many profile stages, especially IBM i Program Spec and
      File Spec instances.
- [ ] Update the GitHub indexer to derive display title from frontmatter
      `title`, then first H1, then normalized path basename.
- [ ] Add controller tests for branch-based document identity and resolved
      missing document paths.

## Rollout Notes

- Roll out as additive sections first.
- Do not remove existing Requirement cards until the new source/doc/review flow
  is stable.
- Start with manual source metadata and mocked GitHub documents if integration
  credentials are not ready.
- Use feature flags if demo environments must keep the current simplified flow.
