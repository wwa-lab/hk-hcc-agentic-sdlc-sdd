---
sdd_version: "1.0"
doc_id: "hk-hcc:BR-001:technical-design"
doc_type: "technical-design"
title: "BR-001 Member Eligibility Technical Design"
owner: "hk-hcc"
profile: "ibm-i"
application_id: "hk-hcc"
project_id: "hk-hcc-core"
snow_group: "HK-HCC"
requirement_id: "BR-001"
source_refs: []
depends_on:
  - doc_id: "hk-hcc:BR-001:functional-spec"
    reason: "Technical design implements the functional eligibility behavior."
entities:
  programs:
    - name: "HCCELIGR"
      library: "HCCLIB"
      language: "RPGLE"
      change_type: "add"
  files:
    - name: "HCCMBRPF"
      library: "HCCLIB"
      object_type: "PF"
      change_type: "read"
---

# BR-001 Member Eligibility Technical Design

Add RPGLE program `HCCELIGR` to evaluate member eligibility against file
`HCCMBRPF`.

## Design Notes

- Read `HCCMBRPF` by member ID.
- Compare the claim service date to member effective and termination dates.
- Return an eligibility status and rejection code to the caller.
