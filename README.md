# HK-HCC Agentic SDLC SDD Repository

This repository is the canonical source of truth for human-readable Spec Driven
Development documents for Agentic SDLC Control Tower.

## Repository Role

- Owns Markdown SDD source documents.
- Keeps requirements, stories, specs, architecture, design, API guides, tasks,
  and prompts reviewable in Git.
- Provides stable document metadata for downstream structured graph generation.
- Does not store generated graph artifacts.

## Profiles

This repo is profile-driven. A profile defines document types, dependency
rules, traceability fields, and downstream graph validation behavior.

Supported profiles:

- `standard-sdd`: generic product/application SDD chain.
- `ibm-i`: IBM i delivery chain with RPG/CL/DDS-oriented artifacts and review
  gates.

The existing top-level numbered folders are the `standard-sdd` baseline. IBM i
work should use `docs/ibm-i/` and `profile: "ibm-i"` in front matter.

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
├── 07-prompts/
└── ibm-i/
    ├── 01-requirement-normalizer/
    ├── 02-functional-spec/
    ├── 03-technical-design/
    ├── 04-program-spec/
    ├── 05-file-spec/
    ├── 06-ut-plan/
    ├── 07-test-scaffold/
    ├── 08-spec-review/
    ├── 09-dds-review/
    └── 10-code-review/
```

## Graph Metadata

Graph-participating documents should start with YAML front matter. Standard SDD
example:

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

IBM i example:

```yaml
---
sdd_version: "1.0"
doc_id: "hk-hcc:BR-001:functional-spec"
doc_type: "functional-spec"
title: "BR-001 Functional Spec"
owner: "hk-hcc"
profile: "ibm-i"
application_id: "hk-hcc"
snow_group: "HK-HCC"
requirement_id: "BR-001"
source_refs: []
depends_on:
  - doc_id: "hk-hcc:BR-001:requirement-normalizer"
    reason: "Functional spec refines normalized business requirement."
entities:
  programs:
    - name: "HCCPGM01"
      library: "HCCLIB"
      language: "RPGLE"
      change_type: "modify"
  files:
    - name: "HCCPF01"
      library: "HCCLIB"
      object_type: "PF"
      change_type: "read-write"
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

IBM i `doc_type` values are:

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

## Downstream Sync

The structured graph repository is generated from this repo. A typical local
sync command is run from the structured data repository:

```bash
npm run sync -- --source ../hk-hcc-agentic-sdlc-sdd --workspace ws-default-001 --application agentic-sdlc-control-tower
```

For IBM i graph validation, pass the IBM i profile:

```bash
npm run sync -- --source ../hk-hcc-agentic-sdlc-sdd --profile ibm-i --workspace ws-default-001 --application hk-hcc --snow-group HK-HCC
```

The generated repository owns `_graph/` artifacts and Neo4j ingestion inputs.
