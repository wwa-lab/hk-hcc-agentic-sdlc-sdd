# Agentic SDLC SDD Repository

This repository is the canonical source of truth for human-readable Spec Driven
Development documents for Agentic SDLC Control Tower.

## Repository Role

- Owns Markdown SDD source documents.
- Keeps requirements, stories, specs, architecture, design, API guides, tasks,
  and prompts reviewable in Git.
- Provides stable document metadata for downstream structured graph generation.
- Does not store generated graph artifacts.

## Layout

```text
docs/
├── 00-context/
├── 01-requirements/
├── 02-user-stories/
├── 03-spec/
├── 04-architecture/
├── 05-design/
│   └── contracts/
├── 06-tasks/
└── 07-prompts/
```

## Graph Metadata

Graph-participating documents should start with YAML front matter:

```yaml
---
sdd_version: "1.0"
doc_id: "agentic-sdlc-control-tower:sdd-knowledge-graph:requirement"
doc_type: "requirement"
title: "SDD Knowledge Graph Requirements"
owner: "wwa-lab"
profile: "standard-sdd"
application_id: "agentic-sdlc-control-tower"
snow_group: null
requirement_id: "SKG"
source_refs: []
depends_on: []
entities:
  programs: []
  files: []
---
```

Required fields for graph sync are:

- `doc_id`
- `doc_type`
- `title`
- `owner`
- `profile`

Recommended fields are:

- `sdd_version`
- `application_id`
- `snow_group`
- `requirement_id`
- `source_refs`
- `depends_on`
- `entities`

## Downstream Sync

The structured graph repository is generated from this repo. A typical local
sync command is run from the structured data repository:

```bash
npm run sync -- --source ../agentic-sdlc-sdd --workspace ws-default-001 --application agentic-sdlc-control-tower
```

The generated repository owns `_graph/` artifacts and Neo4j ingestion inputs.
