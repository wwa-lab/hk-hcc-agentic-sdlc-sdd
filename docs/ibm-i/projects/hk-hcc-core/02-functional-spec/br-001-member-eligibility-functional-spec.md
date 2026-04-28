---
sdd_version: "1.0"
doc_id: "hk-hcc:BR-001:functional-spec"
doc_type: "functional-spec"
title: "BR-001 Member Eligibility Functional Spec"
owner: "hk-hcc"
profile: "ibm-i"
application_id: "hk-hcc"
project_id: "hk-hcc-core"
snow_group: "HK-HCC"
requirement_id: "BR-001"
source_refs: []
depends_on:
  - doc_id: "hk-hcc:BR-001:requirement-normalizer"
    reason: "Functional behavior refines the normalized business rule."
entities:
  programs: []
  files: []
---

# BR-001 Member Eligibility Functional Spec

When a claim enters pricing, the application checks the member record using the
claim member ID and service date.

## Acceptance Criteria

- Active members continue to claim pricing.
- Inactive members receive rejection code `ELIG-001`.
- Missing member records receive rejection code `ELIG-404`.
