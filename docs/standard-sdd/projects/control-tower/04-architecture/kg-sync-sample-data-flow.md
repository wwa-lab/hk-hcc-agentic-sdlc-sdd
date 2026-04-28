---
sdd_version: "1.0"
doc_id: "agentic-sdlc-control-tower:KG-SAMPLE:data-flow"
doc_type: "data-flow"
title: "KG Sample Data Flow"
owner: "wwa-lab"
profile: "standard-sdd"
application_id: "agentic-sdlc-control-tower"
project_id: "control-tower"
snow_group: "SDLC Platform"
requirement_id: "KG-SAMPLE"
source_refs: []
depends_on:
  - doc_id: "agentic-sdlc-control-tower:KG-SAMPLE:architecture"
    reason: "Data flow explains how the architecture moves SDD content into graph artifacts."
entities:
  programs: []
  files: []
---

# KG Sample Data Flow

An SDD commit triggers a notification workflow. The KG workflow checks out the
SDD repo, generates graph artifacts, commits the output, and pushes it back.

## Flow

1. SDD document changes on `main`.
2. SDD workflow sends a repository dispatch event.
3. KG workflow regenerates project-scoped artifacts.
