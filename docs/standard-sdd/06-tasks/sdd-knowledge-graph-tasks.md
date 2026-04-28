# SDD Knowledge Graph Tasks

## Objective

Implement the SDD Knowledge Graph capability as an additive evolution of
Requirement Control Plane. The goal is to create a deterministic structured
graph artifact pipeline, optional Neo4j projection, and Requirement Management
graph view for decision support.

## Traceability

- Requirements: [sdd-knowledge-graph-requirements.md](../01-requirements/sdd-knowledge-graph-requirements.md)
- Stories: [sdd-knowledge-graph-stories.md](../02-user-stories/sdd-knowledge-graph-stories.md)
- Spec: [sdd-knowledge-graph-spec.md](../03-spec/sdd-knowledge-graph-spec.md)
- Architecture: [sdd-knowledge-graph-architecture.md](../04-architecture/sdd-knowledge-graph-architecture.md)
- Data Flow: [sdd-knowledge-graph-data-flow.md](../04-architecture/sdd-knowledge-graph-data-flow.md)
- Data Model: [sdd-knowledge-graph-data-model.md](../04-architecture/sdd-knowledge-graph-data-model.md)
- Design: [sdd-knowledge-graph-design.md](../05-design/sdd-knowledge-graph-design.md)
- API Guide: [sdd-knowledge-graph-API_IMPLEMENTATION_GUIDE.md](../05-design/contracts/sdd-knowledge-graph-API_IMPLEMENTATION_GUIDE.md)

---

## Phase 0: Alignment and Local Environment

- [x] Confirm architecture: SDD repo -> structured sync repo -> Neo4j -> backend -> UI
- [x] Confirm Neo4j is a separately deployed Docker service
- [x] Add local Neo4j Docker Compose file
- [x] Document local Neo4j connection settings
- [x] Confirm graph scope naming: Workspace, Application, SNOW Group, Project
- [x] Confirm structured sync repo naming and branch policy
- [x] Confirm first pilot profile: IBM i or Standard SDD

## Phase 1: SDD Metadata Standard

- [x] Create front matter schema for graph-participating SDD documents
- [x] Add examples for Standard SDD documents
- [x] Add examples for IBM i documents
- [x] Define allowed `doc_type` values per profile
- [x] Define required ownership fields
- [x] Define `source_refs` format
- [x] Define `entities.programs` and `entities.files` format for IBM i
- [x] Define `depends_on` authoring rules
- [x] Add documentation to SDD authoring guide or README

## Phase 2: Structured Sync Job

- [x] Choose implementation home for sync job:
      backend command, standalone script, or external CI job
- [x] Implement Git source branch checkout/read
- [x] Implement Markdown file discovery
- [x] Implement YAML front matter parser
- [x] Implement structured section extraction hooks
- [x] Implement node normalization
- [x] Implement edge normalization
- [x] Implement issue normalization
- [x] Emit `_graph/manifest.json`
- [x] Emit `_graph/nodes.jsonl`
- [x] Emit `_graph/edges.jsonl`
- [x] Emit `_graph/issues.jsonl`
- [x] Emit `_graph/suggestions.jsonl`
- [x] Commit generated graph artifacts to structured sync repo

## Phase 3: Validation

- [x] Validate required front matter fields
- [x] Validate duplicate `doc_id`
- [x] Validate `doc_type` exists in active profile
- [x] Validate `depends_on` targets exist
- [x] Validate dependency type pairs by profile
- [x] Validate ownership metadata
- [x] Validate source reference shape
- [x] Validate IBM i program/file entity shape
- [ ] Write validation issue fixtures
- [ ] Add tests for hard failures and warnings

## Phase 4: Backend Graph API Foundation

- [x] Add graph DTOs
- [x] Add `KnowledgeGraphProvider` interface
- [x] Add `ProfileKnowledgeGraphProvider`
- [x] Add `KnowledgeGraphService`
- [x] Add `KnowledgeGraphController`
- [x] Add `GET /requirements/knowledge-graph`
- [x] Add `GET /requirements/knowledge-graph/nodes/{nodeId}`
- [x] Add `GET /requirements/knowledge-graph/impact`
- [x] Add `GET /requirements/knowledge-graph/health`
- [x] Add section-level error mapping
- [x] Add MockMvc tests for graph endpoints

## Phase 5: Manifest Provider

- [x] Add graph manifest root configuration
- [x] Add manifest reader
- [x] Add JSONL streaming parser
- [x] Map structured artifacts to graph DTOs
- [x] Support filters for workspace, application, SNOW Group, project, branch
- [x] Support issue inclusion toggle
- [ ] Add tests with fixture graph artifacts

## Phase 6: Frontend Graph API Integration

- [x] Add graph DTO TypeScript types
- [x] Add `requirementApi.getKnowledgeGraph`
- [x] Add `requirementApi.getGraphNode`
- [x] Add `requirementApi.getGraphImpact`
- [x] Add `requirementApi.getGraphHealth`
- [x] Add graph state to requirement store
- [x] Fetch graph when user switches to Graph view
- [x] Show graph loading, error, empty, stale, and issue states
- [x] Keep current profile-based graph as dev fallback
- [ ] Add component tests for graph view

## Phase 7: Neo4j Provider

- [x] Add Neo4j Java driver dependency
- [x] Add Neo4j configuration properties
- [x] Add Neo4j driver bean
- [ ] Add constraints/index initialization
- [ ] Implement node import with `MERGE`
- [ ] Implement edge import with `MERGE`
- [ ] Implement import run metadata
- [ ] Implement graph query from Neo4j
- [ ] Implement node detail query
- [ ] Implement impact traversal query
- [ ] Add local integration test plan

## Phase 8: Neo4j Import and Rebuild

- [x] Add `POST /requirements/knowledge-graph/import`
- [x] Add import result DTO
- [ ] Read structured repo manifest
- [ ] Stream nodes and edges into Neo4j
- [ ] Record skipped rows and errors
- [x] Add command or script for local rebuild
- [x] Document how to clear local Neo4j graph volume
- [ ] Add smoke test against local Docker Neo4j

## Phase 9: Decision Support Queries

- [x] Downstream impact from document
- [x] Upstream dependency chain for document
- [ ] Missing dependency report by Application/SNOW Group
- [ ] Stale review report by Application/SNOW Group
- [ ] IBM i program coverage query
- [ ] IBM i file coverage query
- [ ] Requirements with broken SDD chain query
- [ ] Documents without source references query

## Phase 10: Operationalization

- [ ] Add scheduled sync option
- [ ] Add GitHub webhook option
- [ ] Add manual admin sync trigger policy
- [ ] Add graph health to Platform Center or Requirement control strip
- [ ] Add sync/import audit events
- [ ] Add company Docker deployment guide
- [ ] Add secret management guidance
- [ ] Add production network policy guidance

## Phase 11: Documentation and Rollout

- [x] Add SDD authoring guide for front matter
- [x] Add structured repo README template
- [x] Add Neo4j local runbook
- [x] Add Neo4j company server runbook
- [x] Add graph troubleshooting guide
- [ ] Run pilot on one IBM i requirement chain
- [ ] Review graph issues with IT/user stakeholders
- [ ] Promote pilot rules to team standard

---

## Acceptance Gate Before Coding Beyond UI Prototype

- [ ] 9 SDD documents exist and are reviewed
- [ ] Front matter schema accepted
- [ ] Structured repo output format accepted
- [ ] Graph provider API accepted
- [ ] Local Neo4j Docker path verified
- [ ] Pilot profile and pilot project selected
- [ ] Confirmed/suggested edge policy accepted
