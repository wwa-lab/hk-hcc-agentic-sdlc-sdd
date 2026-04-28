# SDD Front Matter Templates

Use this template at the top of any SDD document that should participate in the
knowledge graph.

## Standard SDD

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

### Standard `doc_type` Values

- `requirement`
- `user-story`
- `spec`
- `architecture`
- `data-flow`
- `data-model`
- `design`
- `api-guide`
- `tasks`

## IBM i

```yaml
---
sdd_version: "1.0"
doc_id: "hk-hcc:{br-id}:{doc-type}"
doc_type: "functional-spec"
title: "{BR-ID} Functional Spec"
owner: "hk-hcc"
profile: "ibm-i"
application_id: "hk-hcc"
snow_group: "HK-HCC"
requirement_id: "{BR-ID}"
source_refs: []
depends_on:
  - doc_id: "hk-hcc:{br-id}:requirement-normalizer"
    reason: "Functional spec refines normalized business requirement."
entities:
  programs:
    - name: "{PROGRAM}"
      library: "{LIBRARY}"
      language: "RPGLE"
      change_type: "modify"
  files:
    - name: "{FILE}"
      library: "{LIBRARY}"
      object_type: "PF"
      change_type: "read-write"
---
```

### IBM i `doc_type` Values

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

### IBM i Dependency Pattern

```yaml
depends_on:
  - doc_id: "hk-hcc:{br-id}:functional-spec"
    reason: "Technical design elaborates the functional spec."
```

Accepted IBM i dependency pairs:

- `functional-spec -> requirement-normalizer`
- `technical-design -> functional-spec`
- `program-spec -> technical-design`
- `file-spec -> technical-design`
- `ut-plan -> program-spec`
- `test-scaffold -> ut-plan`
- `spec-review -> technical-design`
- `dds-review -> file-spec`
- `code-review -> program-spec`

## Standard Dependency Pattern

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
