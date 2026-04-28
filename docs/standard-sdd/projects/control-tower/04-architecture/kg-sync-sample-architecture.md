---
sdd_version: "1.0"
doc_id: "agentic-sdlc-control-tower:KG-SAMPLE:architecture"
doc_type: "architecture"
title: "KG Sample Architecture"
owner: "wwa-lab"
profile: "standard-sdd"
application_id: "agentic-sdlc-control-tower"
project_id: "control-tower"
snow_group: "SDLC Platform"
requirement_id: "KG-SAMPLE"
source_refs: []
depends_on:
  - doc_id: "agentic-sdlc-control-tower:KG-SAMPLE:spec"
    reason: "Architecture implements the sync behavior described in the spec."
entities:
  programs: []
  files: []
---

# KG Sample Architecture

SDD source documents stay in the SDD repo. The structured data repo stores
generated graph artifacts under `_graph/{profile}/{project}`.

## Components

- SDD repository: human-authored Markdown.
- KG data repository: generated graph artifacts.
- GitHub Actions: sync orchestration.
