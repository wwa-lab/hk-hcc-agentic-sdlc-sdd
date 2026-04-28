---
sdd_version: "1.0"
doc_id: "hk-hcc:BR-001:program-spec:HCCELIGR"
doc_type: "program-spec"
title: "BR-001 HCCELIGR Program Spec"
owner: "hk-hcc"
profile: "ibm-i"
application_id: "hk-hcc"
project_id: "hk-hcc-core"
snow_group: "HK-HCC"
requirement_id: "BR-001"
source_refs: []
depends_on:
  - doc_id: "hk-hcc:BR-001:technical-design"
    reason: "Program spec details the RPGLE implementation from the technical design."
entities:
  programs:
    - name: "HCCELIGR"
      library: "HCCLIB"
      language: "RPGLE"
      change_type: "add"
  files: []
---

# BR-001 HCCELIGR Program Spec

`HCCELIGR` receives member ID and service date, reads eligibility attributes,
and returns an eligibility decision.

## Interface

- Input: member ID, service date.
- Output: eligible flag, rejection code, rejection message.
