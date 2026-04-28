# SDD Knowledge Graph Data Flow

## Purpose

This document defines runtime flows for generating, validating, importing, and
querying the SDD Knowledge Graph.

## Traceability

- Requirements: [sdd-knowledge-graph-requirements.md](../01-requirements/sdd-knowledge-graph-requirements.md)
- Spec: [sdd-knowledge-graph-spec.md](../03-spec/sdd-knowledge-graph-spec.md)
- Architecture: [sdd-knowledge-graph-architecture.md](sdd-knowledge-graph-architecture.md)

---

## 1. End-to-End Flow

```mermaid
sequenceDiagram
    participant Author as Author or CLI Agent
    participant SDD as SDD Repo
    participant Sync as Structured Sync Job
    participant Structured as Structured Repo
    participant Neo4j as Neo4j
    participant Backend as Control Tower Backend
    participant UI as Requirement Management

    Author->>SDD: Commit Markdown with front matter
    Sync->>SDD: Fetch branch and changed Markdown
    Sync->>Sync: Parse front matter and sections
    Sync->>Sync: Validate profile dependencies
    Sync->>Structured: Commit _graph artifacts
    Sync->>Neo4j: Upsert nodes and edges
    UI->>Backend: GET knowledge graph
    Backend->>Neo4j: Query graph projection
    Neo4j-->>Backend: Nodes, edges, metrics
    Backend-->>UI: Graph DTO
```

---

## 2. Structured Sync Flow

```mermaid
flowchart TD
    Start["Start sync"]
    Resolve["Resolve workspace, application, SNOW Group, profile, branch"]
    Fetch["Fetch SDD Markdown files"]
    Parse["Parse front matter"]
    Extract["Extract structured sections"]
    Normalize["Normalize nodes and edges"]
    Validate["Validate identity and dependencies"]
    Write["Write _graph artifacts"]
    Commit["Commit structured repo changes"]
    End["Sync complete"]

    Start --> Resolve
    Resolve --> Fetch
    Fetch --> Parse
    Parse --> Extract
    Extract --> Normalize
    Normalize --> Validate
    Validate --> Write
    Write --> Commit
    Commit --> End
```

Validation output is always written to `_graph/issues.jsonl` when possible.
Hard failures prevent artifact publication if identity or format integrity is
compromised.

---

## 3. Neo4j Import Flow

```mermaid
sequenceDiagram
    participant Importer as Neo4j Importer
    participant Repo as Structured Repo
    participant Neo4j as Neo4j

    Importer->>Repo: Read manifest
    Importer->>Repo: Stream nodes.jsonl
    Importer->>Neo4j: MERGE nodes by id
    Importer->>Repo: Stream edges.jsonl
    Importer->>Neo4j: MERGE relationships by from, to, type
    Importer->>Neo4j: Store import run metadata
    Importer->>Repo: Optionally write import result
```

Import must be idempotent:

- Node key: `id`
- Relationship identity: `from`, `to`, `type`, `scope`, `branch`
- Properties are updated on re-import.

---

## 4. Requirement Management Graph Query Flow

```mermaid
sequenceDiagram
    participant UI as Requirement Graph View
    participant Store as Requirement Store
    participant API as requirementApi
    participant Backend as Backend Graph Service
    participant Provider as Graph Provider

    UI->>Store: User switches to Graph view
    Store->>API: GET /requirements/knowledge-graph
    API->>Backend: HTTP request with filters
    Backend->>Provider: loadGraph(filter)
    Provider-->>Backend: Graph DTO
    Backend-->>API: ApiResponse<KnowledgeGraphDto>
    API-->>Store: Graph data
    Store-->>UI: Render nodes, edges, metrics
```

Provider behavior:

- `profile`: return profile document stages/dependencies.
- `manifest`: read graph artifacts.
- `neo4j`: query Neo4j.

---

## 5. Impact Query Flow

```mermaid
sequenceDiagram
    participant UI as Graph Detail Panel
    participant Backend as Backend Graph Service
    participant Neo4j as Neo4j

    UI->>Backend: GET /requirements/knowledge-graph/impact?nodeId=doc:BR-FS-AUTH-123&direction=downstream
    Backend->>Neo4j: MATCH variable-length paths
    Neo4j-->>Backend: Impact paths
    Backend-->>UI: Impact result with depth and relationship types
```

Impact query modes:

- `upstream`: what the selected node depends on.
- `downstream`: what can be affected by selected node changes.
- `bidirectional`: both directions for investigation.

---

## 6. Failure Flows

### Invalid front matter

```mermaid
flowchart TD
    Parse["Parse front matter"]
    Invalid["Invalid or missing required fields"]
    Issue["Write issue"]
    Stop["Stop publication for hard failure"]

    Parse --> Invalid
    Invalid --> Issue
    Issue --> Stop
```

### Neo4j unavailable

```mermaid
flowchart TD
    Query["Graph API request"]
    Neo4j["Neo4j provider unavailable"]
    Fallback["Try manifest or profile provider"]
    Response["Return graph or unavailable section"]

    Query --> Neo4j
    Neo4j --> Fallback
    Fallback --> Response
```

### Stale graph

Stale graph is not a hard failure. The response includes:

- last sync timestamp
- source commit
- structured repo commit
- stale flag
- issue count

The UI renders stale state in the graph toolbar.

---

## 7. Refresh Strategy

Trigger options:

- Scheduled sync job.
- Manual backend trigger.
- GitHub webhook for SDD repo changes.
- CLI command after SDD generation.

Initial recommended sequence:

1. Manual CLI sync against local SDD branch.
2. Manual import into local Docker Neo4j.
3. Backend reads local Neo4j.
4. Later add scheduled or webhook-based automation.

---

## 8. Data Retention

- Structured repo keeps graph artifacts by commit history.
- Neo4j keeps latest imported graph by scope and branch.
- Import run metadata keeps recent import status.
- Old branch graph data may be pruned when project branches close.

---

## 9. Observability

Every sync/import run should emit:

- run ID
- start and end timestamp
- source repo and branch
- source commit
- structured repo commit
- graph provider
- node count
- edge count
- issue count by severity
- import duration
- failure message when present
