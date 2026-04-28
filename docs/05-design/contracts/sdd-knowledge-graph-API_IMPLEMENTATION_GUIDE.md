# SDD Knowledge Graph API Implementation Guide

## Purpose

This guide defines backend API contracts for the SDD Knowledge Graph. The API
keeps Requirement Management independent from whether graph data comes from
profile fallback, structured manifest files, or Neo4j.

## Base Path

```text
/api/v1/requirements/knowledge-graph
```

All endpoints return `ApiResponse<T>`.

---

## 1. DTO Overview

### KnowledgeGraphDto

```json
{
  "scope": {},
  "health": {},
  "nodes": [],
  "edges": [],
  "issues": [],
  "lastSync": null
}
```

### GraphScopeDto

```json
{
  "workspaceId": "ws-default",
  "applicationId": "app-payment-gateway-pro",
  "snowGroup": "FIN-TECH-OPS",
  "projectId": "PAY-2026",
  "profileId": "ibm-i",
  "branch": "project/AUTH-123",
  "provider": "neo4j"
}
```

### GraphHealthDto

```json
{
  "nodeCount": 42,
  "edgeCount": 71,
  "issueCount": 3,
  "errorCount": 1,
  "warningCount": 2,
  "suggestionCount": 4,
  "stale": false,
  "lastGeneratedAt": "2026-04-28T06:00:00Z",
  "lastImportedAt": "2026-04-28T06:00:12Z"
}
```

### GraphNodeDto

```json
{
  "id": "doc:BR-FS-AUTH-123",
  "kind": "DOCUMENT",
  "label": "Functional Spec",
  "properties": {
    "docId": "BR-FS-AUTH-123",
    "docType": "functional-spec",
    "requirementId": "AUTH-123",
    "profile": "ibm-i",
    "applicationId": "app-payment-gateway-pro",
    "snowGroup": "FIN-TECH-OPS",
    "path": "docs/02-functional-spec/AUTH-123.md",
    "freshnessStatus": "FRESH"
  }
}
```

### GraphEdgeDto

```json
{
  "id": "edge:doc:BR-FS-AUTH-123:DEPENDS_ON:doc:BR-REQ-AUTH-123",
  "type": "DEPENDS_ON",
  "from": "doc:BR-FS-AUTH-123",
  "to": "doc:BR-REQ-AUTH-123",
  "source": "frontmatter",
  "confidence": 1.0,
  "properties": {
    "profile": "ibm-i",
    "branch": "project/AUTH-123"
  }
}
```

---

## 2. GET `/knowledge-graph`

Load graph for current filter scope.

### Query

| Name | Required | Description |
|------|----------|-------------|
| `workspaceId` | no | Workspace filter |
| `applicationId` | no | Application filter |
| `snowGroup` | no | SNOW Group filter |
| `projectId` | no | Project filter |
| `profileId` | no | Pipeline profile filter |
| `branch` | no | SDD branch or structured branch |
| `requirementId` | no | Requirement filter |
| `nodeKinds` | no | Comma-separated node kinds |
| `includeIssues` | no | Defaults to true |
| `includeSuggestions` | no | Defaults to false |
| `limit` | no | Max nodes; default 1000 |

### Response 200

```json
{
  "data": {
    "scope": {
      "workspaceId": "ws-default",
      "applicationId": "app-payment-gateway-pro",
      "snowGroup": "FIN-TECH-OPS",
      "projectId": "PAY-2026",
      "profileId": "ibm-i",
      "branch": "project/AUTH-123",
      "provider": "neo4j"
    },
    "health": {
      "nodeCount": 42,
      "edgeCount": 71,
      "issueCount": 3,
      "errorCount": 1,
      "warningCount": 2,
      "suggestionCount": 4,
      "stale": false,
      "lastGeneratedAt": "2026-04-28T06:00:00Z",
      "lastImportedAt": "2026-04-28T06:00:12Z"
    },
    "nodes": [
      {
        "id": "doc:BR-FS-AUTH-123",
        "kind": "DOCUMENT",
        "label": "Functional Spec",
        "properties": {
          "docType": "functional-spec",
          "requirementId": "AUTH-123",
          "path": "docs/02-functional-spec/AUTH-123.md"
        }
      }
    ],
    "edges": [
      {
        "id": "edge:doc:BR-FS-AUTH-123:DEPENDS_ON:doc:BR-REQ-AUTH-123",
        "type": "DEPENDS_ON",
        "from": "doc:BR-FS-AUTH-123",
        "to": "doc:BR-REQ-AUTH-123",
        "source": "frontmatter",
        "confidence": 1.0,
        "properties": {}
      }
    ],
    "issues": [],
    "lastSync": {
      "runId": "skg-sync-20260428-060000",
      "status": "COMPLETED",
      "sourceCommitSha": "abc123",
      "structuredCommitSha": "def456",
      "startedAt": "2026-04-28T06:00:00Z",
      "completedAt": "2026-04-28T06:00:12Z"
    }
  }
}
```

### Errors

- 400 `INVALID_GRAPH_FILTER`
- 503 `GRAPH_PROVIDER_UNAVAILABLE`

The backend should prefer returning a successful response with provider status
when fallback data is available.

---

## 3. GET `/knowledge-graph/nodes/{nodeId}`

Load one node with direct relationships.

### Query

Same scope filters as graph list.

### Response 200

```json
{
  "data": {
    "node": {
      "id": "doc:BR-FS-AUTH-123",
      "kind": "DOCUMENT",
      "label": "Functional Spec",
      "properties": {
        "path": "docs/02-functional-spec/AUTH-123.md"
      }
    },
    "incoming": [
      {
        "id": "edge:doc:BR-TD-AUTH-123:DEPENDS_ON:doc:BR-FS-AUTH-123",
        "type": "DEPENDS_ON",
        "from": "doc:BR-TD-AUTH-123",
        "to": "doc:BR-FS-AUTH-123",
        "source": "frontmatter",
        "confidence": 1.0,
        "properties": {}
      }
    ],
    "outgoing": [
      {
        "id": "edge:doc:BR-FS-AUTH-123:DEPENDS_ON:doc:BR-REQ-AUTH-123",
        "type": "DEPENDS_ON",
        "from": "doc:BR-FS-AUTH-123",
        "to": "doc:BR-REQ-AUTH-123",
        "source": "frontmatter",
        "confidence": 1.0,
        "properties": {}
      }
    ],
    "issues": []
  }
}
```

---

## 4. GET `/knowledge-graph/impact`

Traverse upstream or downstream impact from a node.

### Query

| Name | Required | Description |
|------|----------|-------------|
| `nodeId` | yes | Starting node ID |
| `direction` | no | `upstream`, `downstream`, or `both`; default `downstream` |
| `maxDepth` | no | Default 5 |
| `relationshipTypes` | no | Comma-separated relationship filter |
| `applicationId` | no | Ownership filter |
| `snowGroup` | no | Ownership filter |
| `branch` | no | Branch filter |

### Response 200

```json
{
  "data": {
    "startNodeId": "doc:BR-FS-AUTH-123",
    "direction": "downstream",
    "maxDepth": 5,
    "paths": [
      {
        "depth": 1,
        "nodes": [
          { "id": "doc:BR-FS-AUTH-123", "kind": "DOCUMENT", "label": "Functional Spec", "properties": {} },
          { "id": "doc:BR-TD-AUTH-123", "kind": "DOCUMENT", "label": "Technical Design", "properties": {} }
        ],
        "edges": [
          {
            "id": "edge:doc:BR-TD-AUTH-123:DEPENDS_ON:doc:BR-FS-AUTH-123",
            "type": "DEPENDS_ON",
            "from": "doc:BR-TD-AUTH-123",
            "to": "doc:BR-FS-AUTH-123",
            "source": "frontmatter",
            "confidence": 1.0,
            "properties": {}
          }
        ]
      }
    ],
    "summary": {
      "impactedDocuments": 7,
      "impactedPrograms": 2,
      "impactedFiles": 1,
      "staleReviews": 1
    }
  }
}
```

---

## 5. GET `/knowledge-graph/health`

Return graph health and last sync/import state.

### Response 200

```json
{
  "data": {
    "provider": "neo4j",
    "available": true,
    "stale": false,
    "nodeCount": 42,
    "edgeCount": 71,
    "issueCount": 3,
    "lastSync": {
      "runId": "skg-sync-20260428-060000",
      "status": "COMPLETED",
      "sourceCommitSha": "abc123",
      "structuredCommitSha": "def456",
      "startedAt": "2026-04-28T06:00:00Z",
      "completedAt": "2026-04-28T06:00:12Z"
    },
    "lastImport": {
      "runId": "skg-import-20260428-060012",
      "status": "COMPLETED",
      "nodeCount": 42,
      "edgeCount": 71,
      "completedAt": "2026-04-28T06:00:18Z"
    }
  }
}
```

---

## 6. POST `/knowledge-graph/sync`

Trigger sync job. This endpoint may be admin-only or disabled in production if
sync is run by external CI.

### Request

```json
{
  "workspaceId": "ws-default",
  "applicationId": "app-payment-gateway-pro",
  "snowGroup": "FIN-TECH-OPS",
  "projectId": "PAY-2026",
  "profileId": "ibm-i",
  "sourceRepoFullName": "wwa-lab/payment-gateway-sdd",
  "sourceBranch": "project/AUTH-123",
  "structuredRepoFullName": "wwa-lab/payment-gateway-knowledge-base",
  "structuredBranch": "project/AUTH-123"
}
```

### Response 202

```json
{
  "data": {
    "runId": "skg-sync-20260428-060000",
    "status": "QUEUED",
    "message": "Graph sync queued"
  }
}
```

---

## 7. POST `/knowledge-graph/import`

Trigger Neo4j import from structured graph artifacts.

### Request

```json
{
  "graphId": "graph:ws-default:app-payment-gateway-pro:project/AUTH-123",
  "structuredRepoFullName": "wwa-lab/payment-gateway-knowledge-base",
  "structuredBranch": "project/AUTH-123",
  "manifestPath": "_graph/manifest.json"
}
```

### Response 202

```json
{
  "data": {
    "runId": "skg-import-20260428-060012",
    "status": "QUEUED",
    "message": "Neo4j import queued"
  }
}
```

---

## 8. Backend Implementation Notes

Dependencies:

```xml
<dependency>
  <groupId>org.neo4j.driver</groupId>
  <artifactId>neo4j-java-driver</artifactId>
</dependency>
```

Provider selection:

```java
@Service
public class KnowledgeGraphService {
    private final KnowledgeGraphProvider provider;
}
```

Configuration keys:

```text
GRAPH_PROVIDER=profile|manifest|neo4j
GRAPH_MANIFEST_ROOT=/path/to/structured/repo
NEO4J_URI=bolt://localhost:7687
NEO4J_USERNAME=neo4j
NEO4J_PASSWORD=local-dev-password
NEO4J_DATABASE=neo4j
```

---

## 9. Frontend Implementation Notes

Add API functions:

```ts
async getKnowledgeGraph(filters?: GraphFilters): Promise<KnowledgeGraph>
async getGraphNode(nodeId: string, filters?: GraphFilters): Promise<GraphNodeDetail>
async getGraphImpact(request: GraphImpactRequest): Promise<GraphImpact>
async getGraphHealth(filters?: GraphFilters): Promise<GraphHealth>
```

Graph view should:

- call `getKnowledgeGraph` when user switches to Graph view.
- show profile fallback graph only when backend graph API is unavailable in dev
  mode.
- keep graph errors section-scoped.

---

## 10. Test Contracts

Backend:

- `KnowledgeGraphControllerTest`
- `ProfileKnowledgeGraphProviderTest`
- `ManifestKnowledgeGraphProviderTest`
- `Neo4jKnowledgeGraphProviderTest` with Testcontainers optional later

Frontend:

- `SddKnowledgeGraph.spec.ts`
- `requirementStore.graph.spec.ts`

Sync job:

- parsing fixtures
- validation fixtures
- JSONL snapshots
