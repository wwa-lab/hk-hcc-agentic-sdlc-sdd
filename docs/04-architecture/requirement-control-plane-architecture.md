# Requirement Control Plane Architecture

## Overview

Requirement Control Plane extends the Requirement Management slice with a
platform-oriented architecture:

- External BAU systems provide business source references
- Central SDD repositories provide SDD document content and version history
- Project branches provide in-flight SDD review workspaces
- Central Knowledge Base repositories publish generated knowledge graph outputs
- Control Tower indexes metadata, renders documents, records reviews, creates
  agent manifests, and reports freshness
- CLI agents perform repo-aware and long-running work

## System Context

```mermaid
graph TD
    BA["BA / Business User"]
    DEV["Developer / Tech Lead"]
    PM["Delivery Manager"]

    CT["Control Tower Requirement Page"]
    API["Control Tower Backend"]
    DB["Control Tower DB<br/>metadata and reviews"]
    SRCREPO["Source Repo<br/>code truth"]
    SDDREPO["Central SDD Repo<br/>main and project branches"]
    KBREPO["Central Knowledge Base Repo<br/>graph nodes and edges"]
    GH["GitHub<br/>PRs and reviews"]
    JIRA["Jira"]
    CONF["Confluence"]
    KB["KB / Upload Store"]
    AGENT["CLI Agent Runtime"]
    MCP["MCP / Connectors"]

    BA --> CT
    PM --> CT
    DEV --> GH
    DEV --> AGENT

    CT --> API
    API --> DB
    API --> SDDREPO
    API --> KBREPO
    API --> JIRA
    API --> CONF
    API --> KB

    AGENT --> MCP
    MCP --> JIRA
    MCP --> CONF
    MCP --> SRCREPO
    MCP --> SDDREPO
    AGENT --> SRCREPO
    AGENT --> SDDREPO
    AGENT --> KBREPO
    DEV --> GH
    AGENT --> API
```

## Component Architecture

```mermaid
graph TD
    subgraph Frontend["frontend/src/features/requirement"]
        Detail["RequirementDetailView"]
        SourcePanel["SourceReferencesPanel"]
        DocPanel["SddDocumentsPanel"]
        Viewer["GitHubMarkdownViewer"]
        ReviewPanel["BusinessReviewPanel"]
        TracePanel["RequirementTraceabilityPanel"]
        Store["requirementStore"]
        ApiClient["requirementApi"]
    end

    subgraph Backend["backend/domain/requirement"]
        Controller["RequirementController"]
        SourceSvc["SourceReferenceService"]
        DocSvc["SddDocumentService"]
        ReviewSvc["DocumentReviewService"]
        AgentSvc["RequirementAgentRunService"]
        FreshSvc["FreshnessService"]
    end

    subgraph Platform["backend/platform"]
        ProfileSvc["PipelineProfileService"]
        GitHubGateway["GitHubDocumentGateway"]
        SourceGateway["SourceMetadataGateway"]
        WorkspaceSvc["SddWorkspaceResolver"]
    end

    Detail --> SourcePanel
    Detail --> DocPanel
    Detail --> ReviewPanel
    Detail --> AgentPanel
    Detail --> TracePanel
    DocPanel --> Viewer
    Store --> ApiClient
    ApiClient --> Controller

    Controller --> SourceSvc
    Controller --> DocSvc
    Controller --> ReviewSvc
    Controller --> AgentSvc
    Controller --> FreshSvc
    DocSvc --> GitHubGateway
    DocSvc --> WorkspaceSvc
    SourceSvc --> SourceGateway
    DocSvc --> ProfileSvc
```

## Backend Package Boundaries

Requirement-specific services own requirement-linked source references,
document indexes, reviews, manifests, and freshness projections. Provider-specific
logic should live behind platform gateways or agent runtime connectors.

Suggested packages:

```text
com.sdlctower.domain.requirement.source
com.sdlctower.domain.requirement.document
com.sdlctower.domain.requirement.review
com.sdlctower.domain.requirement.agent
com.sdlctower.domain.requirement.freshness
com.sdlctower.platform.github
com.sdlctower.platform.source
```

## Data Ownership

| Data | Owner | Notes |
|---|---|---|
| Jira / Confluence body | Source system | Referenced and optionally summarized, not copied as source of truth |
| Source code | Source repository | Read by CLI agents, not copied into Control Tower |
| SDD Markdown body | Central SDD repo project branch or main | Fetched on open |
| Released SDD baseline | Central SDD repo `main` | Post-release PR merge target |
| In-flight SDD workspace | Central SDD repo project branch | Business review and CLI output during project |
| Knowledge graph nodes and edges | Central Knowledge Base repo | Generated from released SDD main or project preview branch |
| Source reference metadata | Control Tower DB | URL, external ID, source updated time, fetched time |
| SDD workspace metadata | Control Tower DB | application, SNOW Group, source repo, SDD repo, branches, KB repo |
| Document index metadata | Control Tower DB | repo/path/ref/SHA/status/profile/workspace/document instance key |
| Business comments and approvals | Control Tower DB | version-bound |
| Engineering review | GitHub | PR review and diff |
| Agent execution manifest | Control Tower DB | Context handoff |
| Agent outputs | GitHub plus Control Tower artifact links | PRs, docs, reports |

## Profile-Driven Rendering

```mermaid
graph TD
    Profile["Active SDD Profile"]
    Stages["Document Stage Definitions"]
    Paths["Default Path Patterns"]
    Skills["Agent Skill Bindings"]
    Gates["Review Gates"]
    UI["Requirement Detail UI"]
    Manifest["Agent Manifest"]
    Tokens["Resolved Path Tokens"]
    Instances["Document Instances"]

    Profile --> Stages
    Profile --> Paths
    Profile --> Skills
    Profile --> Gates
    Stages --> Instances
    Paths --> Tokens
    Tokens --> Instances
    Instances --> UI
    Skills --> Manifest
    Manifest --> Tokens
    Gates --> UI
```

Profiles define stage templates; runtime context resolves them into document
instances. The same stage can produce multiple instances, such as IBM i Program
Specs for several programs. Missing document rows should use resolved expected
paths whenever token values are known.

## Agent Boundary

Control Tower creates the manifest and records status. Agents execute outside
the web app.

```mermaid
sequenceDiagram
    participant UI as Requirement UI
    participant API as Control Tower API
    participant DB as Control Tower DB
    participant Agent as CLI Agent
    participant SRC as Source Repo
    participant SDD as Central SDD Repo
    participant KB as Central KB Repo

    UI->>API: Request agent run
    API->>DB: Create execution + manifest
    API-->>UI: Manifest ready
    Agent->>API: Fetch manifest
    Agent->>SRC: Read source repo context
    Agent->>SDD: Write docs to project branch
    Agent->>SDD: Open post-release PR when requested
    Agent->>KB: Optionally publish project graph preview
    Agent->>API: Callback with PR and artifacts
    API->>DB: Update execution and artifact links
    UI->>API: Refresh run status
```

## Branch and Knowledge Strategy

Central SDD repositories use `main` as the released baseline. Each in-flight
project creates a working branch from `main`, for example
`project/PAY-2026-sso-upgrade`. Business users review that project branch in
Control Tower. After release, engineering opens a GitHub PR from the project
branch back to `main`; once merged, `main` becomes the new SDD baseline.

Document identity is branch-aware: `central SDD repo + project branch + path +
commit/blob`. File names are generated for readability and traceability, but do
not define the project boundary.

Central Knowledge Base repositories are generated from SDD content. The released
knowledge graph is generated from central SDD `main`. Project preview graphs may
be generated from the SDD project branch into a matching Knowledge Base preview
branch. The Knowledge Base repo is not a second manual document source.

## Freshness Strategy

Freshness is computed as a projection, not as a replacement for external
systems. The first implementation can compare timestamps and Git blob versions:

- `sourceUpdatedAt > documentCommitTime` means Source Changed
- `documentBlobSha != reviewedBlobSha` means Document Changed After Review
- Missing indexed document for expected profile stage means Missing Document
- Missing source for requirement means Missing Source

## Integration Risks

| Risk | Mitigation |
|---|---|
| Provider metadata differs across Jira/Confluence | Store generic metadata plus provider payload summary |
| GitHub fetch latency | Lazy fetch document content; keep metadata list fast |
| Business comments drift after doc change | Bind comments to commit/blob |
| IBM i stages do not match Java SDD | Use profile-defined stages |
| Agent executes against wrong context | Manifest pinning and stale-context callback |
