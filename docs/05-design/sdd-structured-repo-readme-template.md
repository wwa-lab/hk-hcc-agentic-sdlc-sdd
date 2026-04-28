# SDD Knowledge Graph Structured Repo

This repository stores generated graph artifacts for one SDD source repository.
The Markdown SDD repository remains the canonical source of document content.

## Branch Policy

- `main`: released/baseline graph artifacts
- `project/<id-or-slug>`: preview artifacts for in-flight SDD work
- Branch names should mirror the source SDD repository branch names

## Artifact Layout

```text
_graph/
  manifest.json
  nodes.jsonl
  edges.jsonl
  issues.jsonl
  suggestions.jsonl
```

## Rebuild

```bash
node scripts/sdd-graph-sync.mjs \
  --source /path/to/source-sdd-repo \
  --out /path/to/structured-repo/_graph \
  --profile ibm-i \
  --workspace ws-default \
  --application app-payment-gateway-pro \
  --snow-group FIN-TECH-OPS \
  --project PAY-2026 \
  --commit=true
```

## Provider Configuration

Manifest provider:

```bash
GRAPH_PROVIDER=manifest
GRAPH_MANIFEST_ROOT=/path/to/structured-repo
```

Neo4j provider:

```bash
GRAPH_PROVIDER=neo4j
NEO4J_URI=bolt://localhost:7687
NEO4J_USERNAME=neo4j
NEO4J_PASSWORD=local-dev-password
NEO4J_DATABASE=neo4j
```

## Review Checklist

- `manifest.json` counts match expected document count.
- `issues.jsonl` has no `ERROR` severity rows before promotion.
- `edges.jsonl` only contains accepted dependency pairs or reviewed warnings.
- Branch metadata matches the source SDD branch.
