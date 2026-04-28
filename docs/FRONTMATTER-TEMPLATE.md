# SDD Front Matter Template

Use this template at the top of any SDD document that should participate in the
knowledge graph.

```yaml
---
sdd_version: "1.0"
doc_id: "agentic-sdlc-control-tower:{slice}:{doc-type}"
doc_type: "requirement"
title: "{Document Title}"
owner: "wwa-lab"
profile: "standard-sdd"
application_id: "agentic-sdlc-control-tower"
snow_group: null
requirement_id: null
source_refs: []
depends_on: []
entities:
  programs: []
  files: []
---
```

## Standard `doc_type` Values

- `requirement`
- `user-story`
- `spec`
- `architecture`
- `data-flow`
- `data-model`
- `design`
- `api-guide`
- `tasks`

## Dependency Pattern

```yaml
depends_on:
  - doc_id: "agentic-sdlc-control-tower:{slice}:requirement"
    reason: "Stories refine approved requirements."
```

## Source Reference Pattern

```yaml
source_refs:
  - type: "prd"
    title: "Agentic SDLC Control Tower PRD"
    external_id: "PRD-SECTION-10"
    url: null
```
