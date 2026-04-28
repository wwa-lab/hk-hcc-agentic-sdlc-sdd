# SDD Front Matter Authoring Guide

This guide defines the metadata contract for SDD documents that participate in
the SDD Knowledge Graph.

## Scope Defaults

- Workspace: `workspaceId`
- Application: `applicationId`
- SNOW Group: `snowGroup`
- Project: `projectId`
- Branch: source SDD branch, mirrored to the structured graph branch

Structured sync repo naming policy:

- Repository: `<application>-sdd-knowledge-graph`
- Main branch: `main`
- Preview/project branch: same name as the source SDD working branch
- Generated artifacts live under `_graph/`

First pilot profile: `ibm-i`. Standard SDD remains supported for non IBM i
workspaces and for local development.

## Required Fields

```yaml
---
doc_id: BR-FS-AUTH-123
doc_type: functional-spec
title: Authentication Functional Spec
profile: ibm-i
owner: payments-platform
requirement_id: AUTH-123
workspace_id: ws-default
application_id: app-payment-gateway-pro
snow_group: FIN-TECH-OPS
project_id: PAY-2026
depends_on:
  - BR-REQ-AUTH-123
source_refs:
  - type: jira
    external_id: PAY-123
    url: https://jira.example/browse/PAY-123
    title: SSO enhancement
---
```

Required ownership fields are `owner`, `workspace_id`, `application_id`,
`snow_group`, and `project_id`. `requirement_id` is required when the document is
part of a requirement chain.

## Allowed `doc_type`

Standard SDD:

- `requirement`
- `user-story`
- `spec`
- `architecture`
- `data-flow`
- `data-model`
- `design`
- `api-guide`
- `tasks`

IBM i:

- `requirement-normalizer`
- `functional-spec`
- `technical-design`
- `program-spec`
- `file-spec`
- `ut-plan`
- `test-scaffold`
- `spec-review`
- `dds-review`
- `code-review`

## `source_refs`

Each source reference should include:

- `type`: `jira`, `confluence`, `github`, `servicenow`, `upload`, or `url`
- `external_id`: optional source system ID
- `url`: canonical source URL
- `title`: human-readable source title

## IBM i Entities

IBM i program references:

```yaml
entities:
  programs:
    - name: PAYAUTHR
      library: PAYLIB
      language: RPGLE
      change_type: modify
```

IBM i file references:

```yaml
entities:
  files:
    - name: PAYUSRPF
      library: PAYLIB
      object_type: PF
      change_type: read-write
```

## Dependency Rules

Use `depends_on` to point from the current document to upstream source documents.
Targets must be existing `doc_id` values in the active branch.

Standard SDD accepted pairs include:

- `user-story -> requirement`
- `spec -> user-story`
- `architecture -> spec`
- `data-flow -> architecture`
- `data-model -> architecture`
- `design -> data-flow`
- `design -> data-model`
- `api-guide -> design`
- `tasks -> api-guide`
- `tasks -> design`

IBM i accepted pairs include:

- `functional-spec -> requirement-normalizer`
- `technical-design -> functional-spec`
- `program-spec -> technical-design`
- `file-spec -> technical-design`
- `ut-plan -> program-spec`
- `test-scaffold -> ut-plan`
- `spec-review -> technical-design`
- `dds-review -> file-spec`
- `code-review -> program-spec`

## Structured Output

Run the local sync job:

```bash
node scripts/sdd-graph-sync.mjs --source . --out _graph --profile ibm-i --application app-payment-gateway-pro --snow-group FIN-TECH-OPS --project PAY-2026
```

The job emits:

- `_graph/manifest.json`
- `_graph/nodes.jsonl`
- `_graph/edges.jsonl`
- `_graph/issues.jsonl`
- `_graph/suggestions.jsonl`

Use `--commit=true` only in the structured sync repository or CI job.

## Local Neo4j Runbook

Start local Neo4j with:

```bash
docker compose -f docker-compose.neo4j.yml up -d
```

Default settings:

- URI: `bolt://localhost:7687`
- Username: `neo4j`
- Password: `local-dev-password`
- Database: `neo4j`

To clear local graph data, stop Neo4j and remove the named Docker volume for
the compose project, then start it again. In shared company environments, use a
separate database or graph namespace per workspace/application/project.

## Troubleshooting

- `MISSING_REQUIRED_FIELD`: add the required front matter field.
- `DUPLICATE_DOC_ID`: make `doc_id` unique within the active branch.
- `INVALID_DOC_TYPE`: use a profile-approved `doc_type`.
- `MISSING_DEPENDENCY_TARGET`: fix the `depends_on` target or add the missing
  upstream document.
- `UNUSUAL_DEPENDENCY_PAIR`: confirm the edge is intentional or adjust the
  document types.
