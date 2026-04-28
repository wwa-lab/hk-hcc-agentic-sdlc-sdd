# SDD Knowledge Graph Data Model

## Purpose

This document defines the graph artifact schema, Neo4j projection model,
backend DTOs, and frontend types for the SDD Knowledge Graph.

## Traceability

- Requirements: [sdd-knowledge-graph-requirements.md](../01-requirements/sdd-knowledge-graph-requirements.md)
- Spec: [sdd-knowledge-graph-spec.md](../03-spec/sdd-knowledge-graph-spec.md)

---

## 1. Markdown Front Matter Schema

```yaml
---
sdd_version: 1
doc_id: BR-FS-AUTH-123
doc_type: functional-spec
title: Authentication Functional Spec
requirement_id: AUTH-123
profile: ibm-i
application_id: app-payment-gateway-pro
application_name: Payment-Gateway-Pro
snow_group: FIN-TECH-OPS
workspace_id: ws-default
project_id: PAY-2026
source_refs:
  - type: jira
    id: AUTH-123
    uri: jira://AUTH-123
depends_on:
  - BR-REQ-AUTH-123
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

Required fields:

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| `sdd_version` | number | yes | Schema version |
| `doc_id` | string | yes | Stable document ID |
| `doc_type` | string | yes | Profile document type |
| `requirement_id` | string | yes | Business requirement ID |
| `profile` | string | yes | Pipeline profile ID |
| `application_id` | string | yes | Application ownership |
| `snow_group` | string | yes | SNOW support group |
| `depends_on` | string array | no | Upstream doc IDs |
| `source_refs` | object array | no | Jira, Confluence, URL, upload |
| `entities` | object | no | Program/file/API/entity references |

---

## 2. Graph Artifact Files

Structured sync repo output:

```text
_graph/
  manifest.json
  nodes.jsonl
  edges.jsonl
  issues.jsonl
  suggestions.jsonl
```

### manifest.json

```json
{
  "schemaVersion": 1,
  "graphId": "graph:ws-default:app-payment-gateway-pro:project/AUTH-123",
  "workspaceId": "ws-default",
  "applicationId": "app-payment-gateway-pro",
  "snowGroup": "FIN-TECH-OPS",
  "profile": "ibm-i",
  "sourceRepoFullName": "wwa-lab/payment-gateway-sdd",
  "sourceBranch": "project/AUTH-123",
  "sourceCommitSha": "abc123",
  "structuredRepoFullName": "wwa-lab/payment-gateway-knowledge-base",
  "structuredBranch": "project/AUTH-123",
  "generatedAt": "2026-04-28T06:00:00Z",
  "nodeCount": 42,
  "edgeCount": 71,
  "issueCount": 3,
  "suggestionCount": 4
}
```

### nodes.jsonl

One JSON object per line:

```json
{"id":"doc:BR-FS-AUTH-123","kind":"DOCUMENT","label":"Functional Spec","properties":{"docId":"BR-FS-AUTH-123","docType":"functional-spec","requirementId":"AUTH-123","profile":"ibm-i","workspaceId":"ws-default","applicationId":"app-payment-gateway-pro","snowGroup":"FIN-TECH-OPS","repoFullName":"wwa-lab/payment-gateway-sdd","branch":"project/AUTH-123","path":"docs/02-functional-spec/AUTH-123.md","commitSha":"abc123","freshnessStatus":"FRESH"}}
```

### edges.jsonl

```json
{"id":"edge:doc:BR-FS-AUTH-123:DEPENDS_ON:doc:BR-REQ-AUTH-123","type":"DEPENDS_ON","from":"doc:BR-FS-AUTH-123","to":"doc:BR-REQ-AUTH-123","source":"frontmatter","confidence":1.0,"properties":{"profile":"ibm-i","branch":"project/AUTH-123"}}
```

### issues.jsonl

```json
{"severity":"ERROR","code":"MISSING_DEPENDENCY","subjectId":"doc:BR-FS-AUTH-123","message":"depends_on target BR-REQ-AUTH-123 was not found","sourcePath":"docs/02-functional-spec/AUTH-123.md","hint":"Add the Requirement Normalizer document or remove the dependency"}
```

### suggestions.jsonl

```json
{"id":"suggestion:doc:BR-PS-AUTH-123:MENTIONS:program:AUTHR001","type":"MENTIONS","from":"doc:BR-PS-AUTH-123","to":"program:AUTHR001","source":"heuristic","confidence":0.72,"reason":"Program name appears in Program Spec body"}
```

---

## 3. Node Kinds

| Kind | ID prefix | Description |
|------|-----------|-------------|
| `WORKSPACE` | `workspace:` | Control Tower workspace |
| `APPLICATION` | `application:` | Company application scope |
| `SNOW_GROUP` | `snow-group:` | Support group ownership |
| `PROJECT` | `project:` | Project branch or delivery scope |
| `REQUIREMENT` | `requirement:` | Business requirement |
| `SOURCE` | `source:` | Jira, Confluence, GitHub, URL, upload |
| `DOCUMENT` | `doc:` | SDD document |
| `REVIEW` | `review:` | Business or technical review |
| `ARTIFACT` | `artifact:` | Code, build, test, deploy artifact |
| `PROGRAM` | `program:` | IBM i RPGLE/CLLE program/member |
| `FILE` | `file:` | IBM i DDS or data file |

---

## 4. Relationship Types

| Type | From | To | Meaning |
|------|------|----|---------|
| `BELONGS_TO` | Project | Application | Project ownership |
| `OWNED_BY` | Application | SNOW Group | Support ownership |
| `HAS_SOURCE` | Requirement | Source | Requirement evidence |
| `HAS_SDD` | Requirement | Document | Requirement document set |
| `DEPENDS_ON` | Document | Document | Upstream SDD dependency |
| `MENTIONS` | Document | Program/File | Reference in document |
| `DEFINES` | Document | Program/File | Document defines target object |
| `REVIEWS` | Review | Document | Review bound to doc version |
| `IMPLEMENTS` | Artifact | Document | Artifact implements doc |
| `GENERATED_FROM` | Document | Source/Document | Generation lineage |

---

## 5. Neo4j Projection

Labels:

```text
:Workspace
:Application
:SnowGroup
:Project
:Requirement
:Source
:Document
:Review
:Artifact
:Program
:File
```

Indexes and constraints:

```cypher
CREATE CONSTRAINT graph_node_id IF NOT EXISTS
FOR (n:GraphNode) REQUIRE n.id IS UNIQUE;

CREATE INDEX document_doc_id IF NOT EXISTS
FOR (d:Document) ON (d.docId);

CREATE INDEX document_requirement_id IF NOT EXISTS
FOR (d:Document) ON (d.requirementId);

CREATE INDEX document_scope IF NOT EXISTS
FOR (d:Document) ON (d.workspaceId, d.applicationId, d.snowGroup, d.branch);

CREATE INDEX program_name IF NOT EXISTS
FOR (p:Program) ON (p.name);
```

Implementation note: every imported node should also carry common label
`GraphNode` so a single uniqueness constraint can protect all graph node IDs.

---

## 6. Backend DTOs

```java
public record KnowledgeGraphDto(
        GraphScopeDto scope,
        GraphHealthDto health,
        List<GraphNodeDto> nodes,
        List<GraphEdgeDto> edges,
        List<GraphIssueDto> issues,
        GraphSyncRunDto lastSync
) {}

public record GraphNodeDto(
        String id,
        String kind,
        String label,
        Map<String, Object> properties
) {}

public record GraphEdgeDto(
        String id,
        String type,
        String from,
        String to,
        String source,
        double confidence,
        Map<String, Object> properties
) {}

public record GraphIssueDto(
        String severity,
        String code,
        String subjectId,
        String message,
        String sourcePath,
        String hint
) {}
```

---

## 7. Frontend Types

```ts
export interface KnowledgeGraph {
  readonly scope: GraphScope;
  readonly health: GraphHealth;
  readonly nodes: ReadonlyArray<GraphNode>;
  readonly edges: ReadonlyArray<GraphEdge>;
  readonly issues: ReadonlyArray<GraphIssue>;
  readonly lastSync: GraphSyncRun | null;
}

export interface GraphNode {
  readonly id: string;
  readonly kind: string;
  readonly label: string;
  readonly properties: Record<string, unknown>;
}

export interface GraphEdge {
  readonly id: string;
  readonly type: string;
  readonly from: string;
  readonly to: string;
  readonly source: string;
  readonly confidence: number;
  readonly properties: Record<string, unknown>;
}
```

---

## 8. Sync Run Metadata

```json
{
  "runId": "skg-sync-20260428-060000",
  "status": "COMPLETED",
  "sourceRepoFullName": "wwa-lab/payment-gateway-sdd",
  "sourceBranch": "project/AUTH-123",
  "sourceCommitSha": "abc123",
  "structuredRepoFullName": "wwa-lab/payment-gateway-knowledge-base",
  "structuredBranch": "project/AUTH-123",
  "structuredCommitSha": "def456",
  "graphProvider": "neo4j",
  "nodeCount": 42,
  "edgeCount": 71,
  "issueCount": 3,
  "startedAt": "2026-04-28T06:00:00Z",
  "completedAt": "2026-04-28T06:00:12Z"
}
```

---

## 9. Validation Codes

| Code | Severity | Meaning |
|------|----------|---------|
| `INVALID_FRONT_MATTER` | ERROR | YAML cannot be parsed |
| `MISSING_REQUIRED_FIELD` | ERROR | Required graph metadata missing |
| `DUPLICATE_DOC_ID` | ERROR | More than one document has same `doc_id` |
| `UNKNOWN_DOC_TYPE` | ERROR | `doc_type` not in active profile |
| `MISSING_DEPENDENCY` | ERROR/WARN | `depends_on` target not found |
| `INVALID_DEPENDENCY_TYPE` | WARN | Type transition violates profile rules |
| `MISSING_OWNERSHIP` | ERROR | Application or SNOW Group missing |
| `UNKNOWN_ENTITY_REFERENCE` | WARN | Program/file reference cannot be resolved |
| `SUGGESTED_EDGE_REVIEW_REQUIRED` | INFO | Suggested edge needs review |

---

## 10. Rebuild Rule

Neo4j projection must be rebuildable from:

1. `_graph/manifest.json`
2. `_graph/nodes.jsonl`
3. `_graph/edges.jsonl`

No raw Markdown parsing should be required during Neo4j rebuild.
