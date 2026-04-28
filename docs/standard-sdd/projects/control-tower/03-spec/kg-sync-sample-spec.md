---
sdd_version: "1.0"
doc_id: "agentic-sdlc-control-tower:KG-SAMPLE:spec"
doc_type: "spec"
title: "KG Sample Spec"
owner: "wwa-lab"
profile: "standard-sdd"
application_id: "agentic-sdlc-control-tower"
project_id: "control-tower"
snow_group: "SDLC Platform"
requirement_id: "KG-SAMPLE"
source_refs: []
depends_on:
  - doc_id: "agentic-sdlc-control-tower:KG-SAMPLE:user-story"
    reason: "Spec defines behavior for the user story."
entities:
  programs: []
  files: []
---

# KG Sample Spec

The graph sync command scans Markdown files with SDD front matter and writes
JSONL graph artifacts.

## Behavior

- Include documents whose `profile` matches the active sync profile.
- Include documents whose `project_id` matches the active project.
- Write nodes, edges, issues, suggestions, and a manifest.
