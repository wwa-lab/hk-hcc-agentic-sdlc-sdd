# SDD Knowledge Graph Design

## Purpose

This design document provides concrete implementation guidance for the SDD
Knowledge Graph: file layout, sync job behavior, backend services, frontend
components, Neo4j import, configuration, and error states.

## Traceability

- Requirements: [sdd-knowledge-graph-requirements.md](../01-requirements/sdd-knowledge-graph-requirements.md)
- Stories: [sdd-knowledge-graph-stories.md](../02-user-stories/sdd-knowledge-graph-stories.md)
- Spec: [sdd-knowledge-graph-spec.md](../03-spec/sdd-knowledge-graph-spec.md)
- Architecture: [sdd-knowledge-graph-architecture.md](../04-architecture/sdd-knowledge-graph-architecture.md)
- Data Model: [sdd-knowledge-graph-data-model.md](../04-architecture/sdd-knowledge-graph-data-model.md)

---

## 1. Repository Layout

### Control Tower repo

```text
backend/src/main/java/com/sdlctower/domain/requirement/graph/
  KnowledgeGraphController.java
  KnowledgeGraphService.java
  KnowledgeGraphProvider.java
  ProfileKnowledgeGraphProvider.java
  ManifestKnowledgeGraphProvider.java
  Neo4jKnowledgeGraphProvider.java
  GraphImportService.java
  dto/KnowledgeGraphDtos.java
  manifest/GraphManifestReader.java
  neo4j/Neo4jClientConfig.java

frontend/src/features/requirement/components/
  SddKnowledgeGraph.vue
  GraphHealthStrip.vue
  GraphNodeDetailPanel.vue

frontend/src/features/requirement/api/
  requirementApi.ts

docs/
  01-requirements/sdd-knowledge-graph-requirements.md
  ...
```

### Structured sync repo

```text
_graph/
  manifest.json
  nodes.jsonl
  edges.jsonl
  issues.jsonl
  suggestions.jsonl
  imports/
    neo4j-import-result.json
```

### SDD document repo

```text
docs/
  01-requirements/
  02-functional-spec/
  03-technical-design/
  04-program-spec/
  05-file-spec/
  06-ut-plan/
  07-test-scaffold/
  08-reviews/
```

---

## 2. Front Matter Authoring Design

Every graph-participating Markdown document starts with YAML front matter.

```yaml
---
sdd_version: 1
doc_id: BR-TD-AUTH-123
doc_type: technical-design
title: Authentication Technical Design
requirement_id: AUTH-123
profile: ibm-i
workspace_id: ws-default
project_id: PAY-2026
application_id: app-payment-gateway-pro
application_name: Payment-Gateway-Pro
snow_group: FIN-TECH-OPS
source_refs:
  - type: jira
    id: AUTH-123
    uri: jira://AUTH-123
depends_on:
  - BR-FS-AUTH-123
entities:
  programs:
    - name: AUTHR001
      library: PAYLIB
      source_type: RPGLE
      object_type: PGM
  files:
    - name: AUTHPF
      library: PAYLIB
      type: PF
---
```

Authoring rules:

- `doc_id` is immutable after first publication.
- `depends_on` contains upstream document IDs only.
- `source_refs` should use internal pointer URIs when direct link is not
  configured, for example `jira://AUTH-123`.
- `entities` is optional but strongly recommended for IBM i.

---

## 3. Sync Job Design

Command shape:

```bash
sdd-graph-sync \
  --source-repo /path/to/sdd-repo \
  --source-branch project/AUTH-123 \
  --structured-repo /path/to/structured-repo \
  --structured-branch project/AUTH-123 \
  --workspace-id ws-default \
  --application-id app-payment-gateway-pro \
  --snow-group FIN-TECH-OPS \
  --profile ibm-i
```

Implementation steps:

1. Resolve Git commits and branch metadata.
2. Scan Markdown files under configured docs root.
3. Parse front matter.
4. Normalize node IDs and edge IDs.
5. Extract entities and sources.
6. Validate required fields.
7. Validate dependency type transitions.
8. Write JSONL artifacts.
9. Write manifest.
10. Commit structured repo changes if configured.

Parsing guidance:

- Use a YAML parser for front matter.
- Use Markdown AST or conservative section parsing for structured tables.
- Avoid regular-expression-only parsing for full document structure.

---

## 4. Validation Design

Validation levels:

| Severity | Behavior |
|----------|----------|
| `ERROR` | Sync fails or graph publication blocked |
| `WARN` | Sync succeeds but issue appears in UI |
| `INFO` | Sync succeeds with informational note |

Hard failures:

- invalid YAML front matter
- missing `doc_id`
- duplicate `doc_id`
- missing `doc_type`
- missing ownership scope
- invalid JSONL write

Warnings:

- missing dependency target when branch baseline fallback may satisfy it
- invalid profile dependency type
- unknown IBM i entity reference
- no source reference

---

## 5. Neo4j Import Design

Local Docker:

```bash
docker compose -f docker-compose.neo4j.yml up -d
```

Backend env:

```bash
GRAPH_PROVIDER=neo4j
NEO4J_URI=bolt://localhost:7687
NEO4J_USERNAME=neo4j
NEO4J_PASSWORD=local-dev-password
NEO4J_DATABASE=neo4j
```

Import algorithm:

1. Read `manifest.json`.
2. Apply Neo4j constraints and indexes.
3. Stream `nodes.jsonl`.
4. `MERGE` node by `id`.
5. Add common label `GraphNode`.
6. Add domain label based on `kind`.
7. Stream `edges.jsonl`.
8. `MERGE` relationship by `from`, `to`, `type`, `scope`, and `branch`.
9. Write import run metadata.

Cypher sketch:

```cypher
MERGE (n:GraphNode {id: $id})
SET n += $properties,
    n.kind = $kind,
    n.label = $label,
    n.updatedAt = datetime()
```

Relationship sketch:

```cypher
MATCH (a:GraphNode {id: $from})
MATCH (b:GraphNode {id: $to})
MERGE (a)-[r:DEPENDS_ON {id: $id}]->(b)
SET r += $properties,
    r.source = $source,
    r.confidence = $confidence,
    r.updatedAt = datetime()
```

---

## 6. Backend Design

Configuration:

```yaml
app:
  graph:
    provider: ${GRAPH_PROVIDER:profile}
    manifest:
      root: ${GRAPH_MANIFEST_ROOT:}
    neo4j:
      uri: ${NEO4J_URI:bolt://localhost:7687}
      username: ${NEO4J_USERNAME:neo4j}
      password: ${NEO4J_PASSWORD:}
      database: ${NEO4J_DATABASE:neo4j}
```

Provider interface:

```java
public interface KnowledgeGraphProvider {
    KnowledgeGraphDto loadGraph(GraphQuery query);
    GraphNodeDetailDto loadNode(String nodeId, GraphQuery query);
    GraphImpactDto loadImpact(GraphImpactQuery query);
    GraphHealthDto loadHealth(GraphQuery query);
}
```

Recommended first implementation order:

1. `ProfileKnowledgeGraphProvider`
2. `ManifestKnowledgeGraphProvider`
3. `Neo4jKnowledgeGraphProvider`

This lets UI move to API-driven graph before Neo4j is required.

---

## 7. Frontend Design

Existing first component:

- `SddKnowledgeGraph.vue`

Next components:

- `GraphHealthStrip.vue`: metrics, stale state, last sync.
- `GraphNodeDetailPanel.vue`: selected node relationships and metadata.

Store additions:

```ts
const graph = ref<KnowledgeGraph | null>(null);
const graphLoading = ref(false);
const graphError = ref<string | null>(null);

async function fetchKnowledgeGraph(filters?: GraphFilters) {}
async function fetchGraphImpact(nodeId: string, direction: GraphDirection) {}
```

Graph view behavior:

- Graph is list-level, not requirement-detail-only.
- It respects active profile and current filters where possible.
- It shows unavailable state if graph provider fails.
- It can fall back to profile graph when backend graph is disabled.

---

## 8. API Design Summary

Endpoints:

```text
GET  /api/v1/requirements/knowledge-graph
GET  /api/v1/requirements/knowledge-graph/nodes/{nodeId}
GET  /api/v1/requirements/knowledge-graph/impact
GET  /api/v1/requirements/knowledge-graph/health
POST /api/v1/requirements/knowledge-graph/sync
POST /api/v1/requirements/knowledge-graph/import
```

Detailed API contracts are in:

- [sdd-knowledge-graph-API_IMPLEMENTATION_GUIDE.md](contracts/sdd-knowledge-graph-API_IMPLEMENTATION_GUIDE.md)

---

## 9. UI States

| State | UI behavior |
|-------|-------------|
| Loading | Graph canvas skeleton or dimmed graph |
| Ready | Nodes, edges, metrics, selected node details |
| Empty | Explain no graph artifacts exist for current scope |
| Stale | Show last sync timestamp and stale warning |
| Issues | Show issue count and link/filter to issue list |
| Neo4j unavailable | Show graph unavailable section without breaking page |

---

## 10. Testing Strategy

Sync job tests:

- front matter parsing
- duplicate doc ID detection
- missing dependency issue
- profile dependency validation
- JSONL output shape

Backend tests:

- provider selection
- profile provider fallback
- manifest reader
- Neo4j provider query mapping
- API section-level error behavior

Frontend tests:

- Graph view toggle
- graph render with nodes/edges
- selected node details
- unavailable state
- stale/issue metric display

---

## 11. Rollout Plan

1. Author front matter schema and examples.
2. Add sync job producing `_graph` artifacts.
3. Add profile/manifest backend provider.
4. Move UI graph from local profile data to API data.
5. Add local Neo4j import.
6. Add Neo4j provider.
7. Add impact traversal and ownership filters.
8. Add scheduled sync and company Docker deployment.
