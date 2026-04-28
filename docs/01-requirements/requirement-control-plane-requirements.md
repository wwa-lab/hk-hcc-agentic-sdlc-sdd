# Requirement Control Plane Requirements

## Purpose

This document defines the next-stage requirements for evolving Requirement
Management from a requirement CRUD and AI-assisted drafting surface into a
platform control plane for business source intake, GitHub-backed SDD document
review, profile-driven SDD chains, CLI-agent execution requests, and freshness
tracking.

This is an additive alignment document. It does not replace the existing
Requirement Management SDD set. It defines the change set needed to align that
slice with the platform design principles in `CLAUDE.md`.

## Sources

- `CLAUDE.md` — Platform Design Principles
- `docs/01-requirements/requirement-requirements.md`
- `docs/03-spec/requirement-spec.md`
- `docs/04-architecture/requirement-architecture.md`
- `docs/04-architecture/requirement-data-model.md`
- `docs/05-design/requirement-design.md`
- IBM i skill family: `wwa-lab/build-agent-skill`

## 1. Scope

Requirement Control Plane covers:

- Application and SNOW Group ownership context for multi-team support
- Central SDD repository workspace resolution for each project
- Project SDD branches created from the central SDD repository `main` branch
- Central Knowledge Base repository references for generated knowledge graph
  outputs
- Business source intake from Jira, Confluence, uploads, and KB references
- Source reference registration, metadata refresh, and freshness checks
- GitHub `docs/` document discovery and rendering
- Business comments, approvals, and change requests bound to Git commit/blob
  versions
- Profile-driven SDD document chain rendering for Standard Java and IBM i
- CLI-agent execution manifest creation for long-running SDD production work
- Artifact links to GitHub documents, PRs, review reports, and generated outputs
- Requirement-to-source-to-document traceability

It does not cover:

- Replacing Jira or Confluence as BAU source systems
- Editing Markdown documents inside Control Tower
- Replacing GitHub PR review
- Running repo-aware generation, code changes, or tests synchronously from the UI
- Building a full workflow or DAG orchestration engine
- Building a universal document authoring system

## 2. Product Principles

### REQ-RCP-01: Control Tower as control plane

Requirement Management must treat Control Tower as the control plane and review
surface. It must not become the canonical store for SDD Markdown document bodies.

### REQ-RCP-02: External source systems retain authority

Jira, Confluence, uploads, and KB systems must remain source systems for business
input. GitHub must remain the source of truth for engineering-ready SDD
artifacts.

### REQ-RCP-03: GitHub `docs/` is the SDD artifact source

The UI must display SDD documents by reading current content from GitHub. The
platform may index metadata, but the Markdown body and version history belong to
GitHub.

### REQ-RCP-04: Business review is separate from engineering review

Business users must be able to review, comment, approve, or request changes in
Control Tower. Developers and technical leads continue to review GitHub PRs and
diffs in GitHub.

### REQ-RCP-05: Reviews are version-bound

Every comment, approval, and change request must be associated with the Git
commit SHA and blob SHA of the document version being reviewed.

### REQ-RCP-06: Project branches are the development SDD workspace

The central SDD repository `main` branch represents the released baseline.
In-flight projects must use a project branch created from `main`. Control Tower
must display and review documents using the selected project branch, not always
the central repository `main` branch.

### REQ-RCP-07: Knowledge Base is a generated layer

The Central Knowledge Base repository must be treated as a generated knowledge
graph layer derived from released SDD content. It must not become a second
manual source of truth for SDD Markdown.

## 3. Source Reference Requirements

### REQ-RCP-10: Source reference registration

Users must be able to register business sources against a requirement. Supported
source types for Day 1 are:

- Jira issue or epic URL
- Confluence page URL
- Uploaded source bundle or KB document reference
- Existing GitHub document or PR URL

### REQ-RCP-11: Source metadata

Each source reference must capture source type, external ID, title, URL, source
updated time when available, fetched time, version/checksum when available, and
linked requirement ID.

### REQ-RCP-12: Source freshness

The system must indicate when a source changed after an SDD document was
generated or reviewed. Freshness may be computed from `sourceUpdatedAt`,
checksum, external version, or provider-specific metadata.

### REQ-RCP-13: Source cards

Requirement detail must include a Source References section showing source
system, title, link, last fetched time, source updated time, freshness state, and
refresh action.

### REQ-RCP-14: Jira story projection

Requirement detail may show linked Jira stories, but Jira remains the source of
truth for story title, status, assignee, sprint, and workflow state. Control
Tower must treat Jira stories as a read-only projection with cached sync
metadata, Open in Jira, and Refresh from Jira actions.

## 4. GitHub SDD Document Requirements

### REQ-RCP-20: Document index

The platform must index GitHub SDD documents associated with a requirement. The
index must store repo, branch/ref, path, SDD type, profile ID, latest commit SHA,
latest blob SHA, status, and GitHub URL.

### REQ-RCP-20A: SDD workspace context

The document index must be associated with an SDD workspace context containing
application ID, application name, SNOW Group, source repo, central SDD repo,
base branch, working branch, docs root, lifecycle status, release PR URL when
available, and Knowledge Base repo metadata.

Application and SNOW Group ownership must be configurable through Platform
Center or equivalent workspace administration surfaces. Requirement detail uses
these values as selected filter context only; it must not become the edit
surface for ownership mapping.

### REQ-RCP-20B: Profile-selected document chain

Requirement detail must resolve expected SDD document stages from the profile
selected by the user. If the user selects the IBM i profile, the document panel
must render the IBM i chain and mark missing IBM i documents explicitly instead
of falling back to Standard SDD stages.

### REQ-RCP-20C: Project document identity

The platform must identify a reviewable SDD document by central SDD repository,
project working branch, path, commit SHA, and blob SHA. File name alone is not a
project boundary. Multiple projects may use similar document names safely
because each project is isolated by its SDD workspace branch until release.

### REQ-RCP-20D: Resolved expected paths

Profile path patterns may contain tokens such as `{br-id}`, `{program}`,
`{file}`, or `{slug}`. Before showing missing documents, the platform should
resolve tokens from requirement metadata, source references, project metadata,
or the latest agent manifest. If a token is not yet known, the UI may show the
template path, but known tokens must be substituted so users see the expected
future document path.

### REQ-RCP-20E: Stage groups and document instances

A profile stage may produce more than one document. IBM i examples include
multiple Program Specs and File Specs for one BR. The document model must
support stage groups containing zero, one, or many document instances. Each
instance must have a stable instance key, resolved path, title, repo/ref
metadata when present, and missing state when absent.

### REQ-RCP-21: Latest document rendering

When a user opens a document in Control Tower, the backend must fetch the latest
Markdown content from GitHub for the indexed repo/path/ref and return the content
with version metadata.

### REQ-RCP-22: No canonical Markdown body in Control Tower

Control Tower must not treat cached Markdown content as authoritative. If content
is cached for performance later, it must be explicitly marked as cache and
validated against GitHub metadata.

### REQ-RCP-23: GitHub deep link

Every rendered document must provide a link to open the same document in GitHub.

### REQ-RCP-24: Document version display

The UI must show the current commit SHA, document path, last updated time when
available, and whether the user is viewing latest or a pinned reviewed version.

The clickable document name must use the indexed Markdown title. The profile
stage label may be shown as secondary metadata so users can distinguish the
document's role without confusing it with the actual document name.

Document titles should be auto-populated by the indexer from Markdown
frontmatter `title` first, then the first Markdown H1, then a normalized file
name fallback. Manual title edits in Control Tower are not the normal workflow.

### REQ-RCP-25: Branch-aware document rendering

Requirement detail must show which SDD workspace branch is being reviewed.
Document links and Markdown fetches must use that branch or the pinned commit
from the document index.

### REQ-RCP-26: Knowledge graph handoff metadata

The platform must expose the configured Knowledge Base repository, main branch,
project preview branch, and graph manifest path so downstream sync jobs and UI
surfaces can connect SDD documents to knowledge graph nodes.

## 5. Business Review Requirements

### REQ-RCP-30: Business comment

Business users must be able to add document-level comments in Control Tower.
Day 1 comments may be document-level or heading-level; inline line comments are
deferred.

### REQ-RCP-31: Approval decision

Business users must be able to mark a document version as Approved, Changes
Requested, or Rejected.

Rejected decisions must require a non-empty business reason so delivery teams
know what must change before the document can be approved.

### REQ-RCP-32: Version-bound decision

Approval decisions must store document ID, repo, path, commit SHA, blob SHA,
reviewer, decision, comment, and timestamp.

### REQ-RCP-33: Review history

Requirement detail must show review history grouped by document and version.

## 6. Profile-Driven SDD Requirements

### REQ-RCP-40: Active SDD profile

Requirement Management must render document stages from the active SDD profile,
not from a hardcoded Java-only chain.

### REQ-RCP-41: Standard Java profile

The Standard Java profile must support Requirement, User Story, Spec,
Architecture, Design, Tasks, Code, and Test document stages with the existing SDD
folder conventions.

### REQ-RCP-42: IBM i profile

The IBM i profile must support Requirement Normalizer, Functional Spec,
Technical Design, Program Spec, File Spec, UT Plan, Test Scaffold, Spec Review,
DDS Review, and Code Review stages. It must support BR-xx continuity, L1/L2/L3
tiering, fast-path enhancement work, program chain, and file chain concepts.
Its Skill & Document Flow must expose the full `wwa-lab/build-agent-skill`
family as individual skills, including analyzer, generator, reviewer, precheck,
and workflow-orchestrator skills.

### REQ-RCP-43: Profile-owned document paths

Each profile must define expected document stages and default path patterns under
`docs/`. The UI must use these definitions to render expected and missing
documents.

### REQ-RCP-44: Profile-owned skill bindings

Each profile may define CLI-agent skill IDs and review gates. The UI may request
an agent run, but must not execute repo-aware skills synchronously.

### REQ-RCP-45: Skill and document dependency map

Requirement Management must provide a profile-driven page that shows each
skill's input documents, output documents, upstream skill dependencies, and
document-to-document dependencies. The page must derive this map from the active
SDD profile so Standard Java, IBM i, and future legacy profiles can expose their
own workflow without hardcoded UI chains.

The dependency map may include source and generated artifacts that are not
Requirement Detail SDD stages, such as raw Jira input, existing RPGLE/CLLE
source, DDS source, generated code, compile precheck reports, and workflow
routing manifests. These flow artifacts must not pollute the GitHub SDD
Documents panel.

## 7. Agent Execution Requirements

### REQ-RCP-50: Agent run request

Users with permission must be able to request an agent run from Requirement
detail. The request creates an execution manifest, not an immediate synchronous
generation inside the UI.

### REQ-RCP-51: Execution manifest

The manifest must include execution ID, project ID, requirement ID, active SDD
profile, source repo, central SDD workspace, project working branch, Knowledge
Base repo context, source references, known document references, output
expectations, and constraints.

### REQ-RCP-52: Latest resolved, then pinned execution

When creating a manifest, the platform must resolve the latest requested sources
and documents, pin versions in the manifest, and make the agent execute against
those versions.

### REQ-RCP-53: Agent callback

Agents must be able to report execution status, output summary, GitHub PR URL,
artifact links, and errors back to Control Tower.

### REQ-RCP-54: Execution visibility

Requirement detail must show recent agent runs, their statuses, requested skill,
source context, output artifacts, and failure reason when applicable.

## 8. Freshness and Traceability Requirements

### REQ-RCP-60: Requirement source freshness

The page must show whether business sources changed after the currently indexed
SDD documents were generated.

### REQ-RCP-61: Review freshness

The page must show whether a document changed after business approval.

### REQ-RCP-62: Traceability map

The page must connect requirement, source references, SDD documents, reviews,
agent executions, and GitHub artifacts in one traceability view.

### REQ-RCP-63: Stale state vocabulary

Freshness states must use a common vocabulary: Fresh, Source Changed, Document
Changed After Review, Missing Document, Missing Source, Unknown.

## 9. Non-Functional Requirements

### REQ-RCP-70: Permission boundaries

Business users may comment and approve in Control Tower. Engineering review
continues in GitHub. Agent run requests require a higher permission than read or
comment.

### REQ-RCP-71: Auditability

Source reference creation, document review decisions, manifest creation, and
agent callback updates must be auditable.

### REQ-RCP-72: Provider isolation

Jira, Confluence, GitHub, KB, and MCP-specific details must be isolated behind
integration gateways or agent manifests. UI components should use platform DTOs.

### REQ-RCP-73: Performance

Opening a Requirement detail page must not block on all external systems.
Document content fetches and source refreshes may be lazy or user-triggered.

### REQ-RCP-74: Extensibility

The model must support additional SDD profiles and source systems without
schema changes for every new profile-specific stage.
