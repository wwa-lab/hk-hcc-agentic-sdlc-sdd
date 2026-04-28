# SDD Knowledge Graph User Stories

## Traceability

- Requirements: [sdd-knowledge-graph-requirements.md](../01-requirements/sdd-knowledge-graph-requirements.md)

---

## Actors

- **Business Analyst**: needs confidence that requirements and SDD documents are
  complete before review.
- **IT Lead / Architect**: needs impact analysis across documents, programs,
  files, tests, and reviews.
- **IBM i Developer**: needs to understand which RPGLE/CLLE members and DDS
  files are affected by a requirement.
- **Platform Engineer**: operates sync jobs, graph artifacts, and Neo4j.
- **Control Tower Admin**: configures graph provider and monitors graph health.

---

## Stories

### SKG-S1: Declare graph metadata in SDD Markdown

As an SDD document author, I want to add standard front matter to each SDD
document so that relationships are explicit before the document enters the
knowledge graph.

Acceptance criteria:

- Required fields are documented and validated.
- `doc_id`, `doc_type`, `requirement_id`, `profile`, `application_id`, and
  `snow_group` are required for graph-participating documents.
- `depends_on` contains upstream document IDs only.
- Invalid YAML or missing required fields are reported as sync issues.

### SKG-S2: Run deterministic structured sync

As a platform engineer, I want a sync job to parse the SDD repo and publish
structured graph artifacts so Neo4j can be rebuilt from deterministic files.

Acceptance criteria:

- The job scans configured SDD repo branch and docs root.
- It emits `_graph/nodes.jsonl`, `_graph/edges.jsonl`,
  `_graph/issues.jsonl`, `_graph/suggestions.jsonl`, and
  `_graph/manifest.json`.
- The structured artifacts include source commit and sync timestamp.
- The job fails hard for duplicate `doc_id` and invalid artifact format.

### SKG-S3: Validate profile-specific dependencies

As an architect, I want document relationships validated against the active
profile so that graph edges reflect the expected SDD workflow.

Acceptance criteria:

- Standard SDD and IBM i profiles define allowed document type dependencies.
- Invalid dependency type pairs are written to `_graph/issues.jsonl`.
- Missing upstream documents are written as warnings or errors based on rule
  severity.
- The sync result includes issue counts by severity.

### SKG-S4: Separate confirmed and suggested graph edges

As an IT lead, I want to see which graph relationships are confirmed and which
are suggested so decisions are based on audited evidence.

Acceptance criteria:

- Confirmed edges have `confidence=1.0` and `source` equal to
  `frontmatter`, `system`, or `review`.
- Suggested edges are emitted to `_graph/suggestions.jsonl`.
- Suggested edges do not become Neo4j confirmed relationships until approved.
- The UI can hide or show suggestions independently.

### SKG-S5: Import graph artifacts into Neo4j

As a platform engineer, I want to import structured graph artifacts into Neo4j
idempotently so the graph can support fast traversal and decision queries.

Acceptance criteria:

- Node import uses stable IDs and `MERGE`.
- Relationship import uses stable `from`, `to`, and `type`.
- Re-running the same import does not create duplicates.
- Import result records node count, edge count, skipped rows, and errors.

### SKG-S6: View SDD document graph in Requirement Management

As a business analyst, I want to open a graph view in Requirement Management so
I can understand how SDD documents relate without reading every Markdown file.

Acceptance criteria:

- Graph view is available next to List, Kanban, and Matrix.
- Nodes show document label, type, traceability key, tier, and health.
- Edges show dependency direction.
- Selecting a node shows upstream and downstream relationships.

### SKG-S7: Analyze impact from a changed document

As an architect, I want to select a document and see downstream impacted
documents, reviews, programs, files, and artifacts so I can plan change risk.

Acceptance criteria:

- API supports downstream traversal from any document node.
- Results show path depth and relationship types.
- Stale reviews are called out separately.
- Results can be filtered by Application, SNOW Group, Project, and branch.

### SKG-S8: Analyze IBM i program and file coverage

As an IBM i developer, I want the graph to show which programs and DDS files are
mentioned, defined, reviewed, or impacted by SDD documents.

Acceptance criteria:

- Program nodes include name, library, member, source type, and object type when
  available.
- File nodes include file name and DDS type when available.
- Document-to-program and document-to-file edges are typed.
- Missing file/program references are emitted as graph issues.

### SKG-S9: Monitor graph health

As a Control Tower admin, I want to see sync and graph import health so I can
trust the graph view before users rely on it.

Acceptance criteria:

- Last sync run metadata is available from API.
- Last Neo4j import metadata is available from API.
- Issue counts by severity are visible.
- Neo4j unavailable state is shown without breaking Requirement Management.

### SKG-S10: Support local Docker graph experiments

As a developer, I want to run Neo4j locally through Docker so I can validate the
model and API before using a company server.

Acceptance criteria:

- Repo includes an optional Neo4j Docker Compose file.
- Local password and Bolt URI are documented.
- Backend graph provider can be configured by environment variables.
- Local Neo4j can be cleared and rebuilt from structured artifacts.

### SKG-S11: Query graph by ownership scope

As a team lead, I want to filter graph data by Application and SNOW Group so I
can focus on my team's systems.

Acceptance criteria:

- Graph API accepts `workspaceId`, `applicationId`, `snowGroup`, and `projectId`
  filters.
- Graph nodes carry ownership metadata.
- Query results include active filter echo metadata.
- Missing ownership metadata appears as a validation issue.

### SKG-S12: Keep normal Requirement Management usable during graph failures

As a business user, I want normal requirement list/detail pages to keep working
even if Neo4j or graph sync is down.

Acceptance criteria:

- Graph API failure is isolated to graph sections.
- List, detail, source evidence, and SDD document panels continue to render.
- UI shows graph unavailable state with last successful sync if known.
