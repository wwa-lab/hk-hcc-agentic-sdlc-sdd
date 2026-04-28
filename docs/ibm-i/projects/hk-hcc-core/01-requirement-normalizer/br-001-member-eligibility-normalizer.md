---
sdd_version: "1.0"
doc_id: "hk-hcc:BR-001:requirement-normalizer"
doc_type: "requirement-normalizer"
title: "BR-001 Member Eligibility Requirement Normalizer"
owner: "hk-hcc"
profile: "ibm-i"
application_id: "hk-hcc"
project_id: "hk-hcc-core"
snow_group: "HK-HCC"
requirement_id: "BR-001"
source_refs:
  - type: "business-rule"
    external_id: "BR-001"
    title: "Member eligibility must be validated before claim pricing."
depends_on: []
entities:
  programs: []
  files: []
---

# BR-001 Member Eligibility Requirement Normalizer

Validate member eligibility before claim pricing starts.

## Normalized Rule

- The system must confirm that a member is active on the claim service date.
- The validation result must be available to downstream pricing logic.
- Inactive members should stop the pricing flow with a clear rejection reason.
