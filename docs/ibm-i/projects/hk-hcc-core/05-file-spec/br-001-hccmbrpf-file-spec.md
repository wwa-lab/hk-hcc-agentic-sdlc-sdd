---
sdd_version: "1.0"
doc_id: "hk-hcc:BR-001:file-spec:HCCMBRPF"
doc_type: "file-spec"
title: "BR-001 HCCMBRPF File Spec"
owner: "hk-hcc"
profile: "ibm-i"
application_id: "hk-hcc"
project_id: "hk-hcc-core"
snow_group: "HK-HCC"
requirement_id: "BR-001"
source_refs: []
depends_on:
  - doc_id: "hk-hcc:BR-001:technical-design"
    reason: "File spec documents the physical file read by the technical design."
entities:
  programs: []
  files:
    - name: "HCCMBRPF"
      library: "HCCLIB"
      object_type: "PF"
      change_type: "read"
---

# BR-001 HCCMBRPF File Spec

`HCCMBRPF` stores member eligibility dates used by `HCCELIGR`.

## Fields

- `MBRID`: member identifier.
- `EFFDAT`: eligibility effective date.
- `TRMDAT`: eligibility termination date.
