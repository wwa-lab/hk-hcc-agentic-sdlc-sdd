# SDD Knowledge Graph Requirements

## Purpose

This document defines the requirements for turning SDD Markdown documents into
a structured, auditable knowledge graph. The capability supports Requirement
Management decision-making by showing relationships between requirements,
source references, SDD documents, IBM i objects, reviews, and downstream
artifacts.

## Source

This requirement package is derived from the Requirement Control Plane
direction and team alignment on:

- SDD documents remain the canonical source of truth in GitHub.
- A second structured sync repository stores deterministic graph artifacts.
- Neo4j is deployed as a query projection, not as the canonical document store.
- Requirement Management renders graph insight for user and IT decisions.

---

## 1. Scope

In scope:

- Standard SDD document metadata schema using Markdown front matter.
- Deterministic sync from SDD document repo to structured sync repo.
- Structured graph artifacts under `_graph/`.
- Validation of document identity, type, dependencies, ownership, and entity
  references.
- Neo4j import/upsert from structured artifacts.
- Backend graph API for Requirement Management.
- Requirement Management graph view backed by API data.
- Support for Application and SNOW Group ownership filtering.

Out of scope:

- Direct editing of SDD Markdown from Control Tower.
- Frontend direct connection to Neo4j.
- Using AI-inferred edges as confirmed graph truth without review.
- Replacing the existing Requirement Control Plane source/document/review
  tables.
- Neo4j cluster or enterprise HA design for the first local/dev phase.

---

## 2. Product Requirements

### SKG-REQ-01: Canonical source boundary

The SDD Markdown repository must remain the canonical source for human-readable
SDD content. The structured sync repository must be generated from the SDD repo
and may be rebuilt at any time.

### SKG-REQ-02: Structured sync repository

The system must maintain a structured sync repository that contains normalized
graph artifacts generated from SDD Markdown. This repository is the source used
for Neo4j ingestion and audit review.

### SKG-REQ-03: Required document front matter

Each graph-participating SDD Markdown document must expose stable front matter:

- `sdd_version`
- `doc_id`
- `doc_type`
- `requirement_id`
- `profile`
- `application_id`
- `snow_group`
- `source_refs`
- `depends_on`
- `entities`

### SKG-REQ-04: Stable document identity

Every SDD document must have a globally stable `doc_id` within the workspace or
application scope. The sync job must reject duplicate `doc_id` values.

### SKG-REQ-05: Upstream-only dependency authoring

Authors must declare upstream dependencies only through `depends_on`. The sync
job derives downstream relationships automatically.

### SKG-REQ-06: Profile-aware validation

The sync job must validate dependencies against the active pipeline profile.
For example, IBM i Program Spec may depend on Technical Design, Functional Spec,
and Impact Analysis, while DDS Review may depend on File Spec and DDS Source.

### SKG-REQ-07: Confirmed and suggested edges

The graph must distinguish confirmed edges from suggested edges. Confirmed
edges come from front matter, structured tables, or system records. Suggested
edges come from heuristics or AI and require review before becoming confirmed.

### SKG-REQ-08: Graph artifact output

The structured sync repo must publish:

- `_graph/manifest.json`
- `_graph/nodes.jsonl`
- `_graph/edges.jsonl`
- `_graph/issues.jsonl`
- `_graph/suggestions.jsonl`

### SKG-REQ-09: Sync issues as first-class output

Validation errors and warnings must be written to `_graph/issues.jsonl`.
Requirement Management must be able to show graph health without reading logs.

### SKG-REQ-10: Neo4j as projection

Neo4j must be treated as a rebuildable projection of the structured graph
artifacts. If Neo4j data is lost, it can be recreated from the structured sync
repo.

### SKG-REQ-11: Backend-owned graph access

The Control Tower frontend must never connect directly to Neo4j. All graph data
must be served through backend APIs.

### SKG-REQ-12: Local Docker support

Developers must be able to run Neo4j locally through Docker for graph model and
API testing before deploying to a company server.

### SKG-REQ-13: Ownership filtering

Graph queries must support filtering by Workspace, Application, SNOW Group,
Project, Requirement, profile, branch, and freshness status.

### SKG-REQ-14: Requirement Management graph view

Requirement Management must provide a graph view that displays SDD document
relationships and decision metrics such as missing documents, stale reviews,
impacted documents, and graph issues.

### SKG-REQ-15: Impact analysis

The graph API must support upstream and downstream traversal for a selected
Requirement, Document, Program, File, or Source.

### SKG-REQ-16: IBM i object awareness

The graph must support IBM i entities such as RPGLE/CLLE programs, service
programs, DDS files, PF, LF, DSPF, PRTF, library, member, and object type.

### SKG-REQ-17: Branch-aware graph projection

The graph must distinguish released baseline (`main`) from project preview
branches. Decision support must make clear whether a node comes from baseline
or preview work.

### SKG-REQ-18: Auditable sync runs

Each sync run must record source commit, structured repo commit, branch,
timestamp, issue counts, node count, edge count, and Neo4j import status.

### SKG-REQ-19: Idempotent import

Neo4j ingestion must be idempotent. Re-importing the same graph artifacts must
not create duplicate nodes or relationships.

### SKG-REQ-20: Failure isolation

Graph sync or Neo4j outage must not block normal Requirement Management list,
detail, source evidence, or SDD document viewing.

---

## 3. Non-Functional Requirements

### SKG-NFR-01: Accuracy

Confirmed graph relationships must be deterministic and traceable to a source
field, table, or system record. AI suggestions must never be silently promoted
to confirmed graph truth.

### SKG-NFR-02: Performance

Graph API responses for the Requirement Management graph view should return in
under 2 seconds for a workspace slice of up to 2,000 nodes and 10,000 edges.

### SKG-NFR-03: Rebuildability

A clean Neo4j database must be rebuildable from structured sync artifacts
without reading original Markdown again.

### SKG-NFR-04: Security

Neo4j credentials must be environment or secret managed. The Neo4j HTTP browser
port should be restricted to administrators in shared environments.

### SKG-NFR-05: Observability

Sync and import jobs must expose counts, duration, warnings, errors, and last
successful run metadata.

### SKG-NFR-06: Compatibility

The backend must continue to support local H2/Oracle persistence for core
Control Tower data. Neo4j is an optional graph provider that can be disabled.

---

## 4. Acceptance Criteria

- A developer can add front matter to SDD Markdown and run a sync job.
- The sync job writes `_graph/nodes.jsonl`, `_graph/edges.jsonl`,
  `_graph/issues.jsonl`, and `_graph/manifest.json`.
- Duplicate document IDs and missing dependency targets are reported.
- Confirmed and suggested edges are represented separately.
- Neo4j can be started locally by Docker and populated from structured graph
  artifacts.
- Requirement Management can display graph data through backend API calls.
- Disabling Neo4j falls back to manifest/profile graph data without breaking
  Requirement Management.
