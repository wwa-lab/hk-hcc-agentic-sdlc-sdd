# IBM i SDD Profile

This folder is the IBM i profile workspace for HK-HCC SDD documents.

Use `profile: "ibm-i"` in every graph-participating document. The knowledge
graph sync validates IBM i documents with the IBM i document type chain and
dependency rules.

IBM i projects live under `projects/{project-id}/`.

## Document Chain

```text
projects/{project-id}/
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

## Document Types

| Folder | `doc_type` | Purpose |
| --- | --- | --- |
| `01-requirement-normalizer` | `requirement-normalizer` | Normalize source requirement into BR/CF/CBR/CE traceable items. |
| `02-functional-spec` | `functional-spec` | Define user-visible functional behavior and acceptance rules. |
| `03-technical-design` | `technical-design` | Define IBM i technical approach, interfaces, jobs, and data access. |
| `04-program-spec` | `program-spec` | Specify RPG/CL/program-level changes. |
| `05-file-spec` | `file-spec` | Specify PF/LF/DSPF/PRTF/DDS or DB2 file-level changes. |
| `06-ut-plan` | `ut-plan` | Define unit test plan and test data. |
| `07-test-scaffold` | `test-scaffold` | Capture generated or manual test scaffolding. |
| `08-spec-review` | `spec-review` | Review technical design/spec completeness. |
| `09-dds-review` | `dds-review` | Review DDS/file impact and compatibility. |
| `10-code-review` | `code-review` | Review RPG/CL/source implementation readiness. |

## Front Matter Example

```yaml
---
sdd_version: "1.0"
doc_id: "hk-hcc:BR-001:functional-spec"
doc_type: "functional-spec"
title: "BR-001 Functional Spec"
owner: "hk-hcc"
profile: "ibm-i"
application_id: "hk-hcc"
project_id: "hk-hcc-core"
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

## Graph Sync

Run the structured sync with the IBM i profile:

```bash
npm run sync:ibm-i
```

The current baseline IBM i project is `hk-hcc-core`.
