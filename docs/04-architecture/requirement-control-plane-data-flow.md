# Requirement Control Plane Data Flow

## Purpose

This document describes runtime data flows for Requirement Control Plane:
source intake, GitHub document rendering, business review, agent manifest
creation, agent callback, and freshness refresh.

## 1. Source Reference Intake

```mermaid
sequenceDiagram
    participant User
    participant UI as SourceReferencesPanel
    participant Store as Requirement Store
    participant API as Requirement API
    participant Source as SourceReferenceService
    participant Gateway as SourceMetadataGateway
    participant DB as DB

    User->>UI: Paste Jira or Confluence URL
    UI->>Store: addSourceReference(url)
    Store->>API: POST /requirements/{id}/sources
    API->>Source: create(reference)
    Source->>Gateway: Resolve metadata
    Gateway-->>Source: title, externalId, sourceUpdatedAt
    Source->>DB: Save source_reference
    Source-->>API: SourceReferenceDto
    API-->>Store: SourceReferenceDto
    Store-->>UI: Render source card
```

Error behavior:

- If metadata cannot be fetched, store the URL with `status=ERROR`.
- UI shows retry action.
- Requirement detail continues rendering.

## 2. Open Latest GitHub SDD Document

```mermaid
sequenceDiagram
    participant User
    participant UI as SddDocumentsPanel
    participant API as Requirement API
    participant Docs as SddDocumentService
    participant GH as GitHubDocumentGateway
    participant DB as DB

    User->>UI: Open document
    UI->>API: GET /requirements/documents/{documentId}
    API->>Docs: fetchLatest(documentId)
    Docs->>DB: Load repo/ref/path
    Docs->>GH: Fetch Markdown and blob metadata
    GH-->>Docs: markdown, commitSha, blobSha, githubUrl
    Docs->>DB: Update latest metadata
    Docs-->>API: DocumentContentDto
    API-->>UI: Markdown + version metadata
    UI->>UI: Render Markdown
```

Rules:

- Markdown body is fetched from GitHub.
- Control Tower may update index metadata after fetch.
- The fetched content is not canonical storage in Control Tower.

## 3. Business Review

```mermaid
sequenceDiagram
    participant User
    participant UI as BusinessReviewPanel
    participant API as Requirement API
    participant Review as DocumentReviewService
    participant DB as DB

    User->>UI: Comment or approve
    UI->>API: POST /requirements/documents/{documentId}/reviews
    API->>Review: createReview(request)
    Review->>Review: Validate commitSha and blobSha
    Review->>DB: Save document_review
    Review-->>API: ReviewDto
    API-->>UI: ReviewDto
    UI->>UI: Append review history
```

Rules:

- Review requests must include commit SHA and blob SHA.
- Approval becomes stale if the document blob changes later.

## 4. Agent Run Request

```mermaid
sequenceDiagram
    participant Developer
    participant Tool as CLI / Developer Action
    participant API as Requirement API
    participant AgentSvc as RequirementAgentRunService
    participant Docs as SddDocumentService
    participant Source as SourceReferenceService
    participant Profile as PipelineProfileService
    participant DB as DB

    Developer->>Tool: Request agent run
    Tool->>API: POST /requirements/{id}/agent-runs
    API->>AgentSvc: createRun(request)
    AgentSvc->>Profile: Resolve active profile
    AgentSvc->>Source: Resolve latest source refs
    AgentSvc->>Docs: Resolve latest doc refs
    AgentSvc->>DB: Save execution + manifest
    AgentSvc-->>API: AgentRunDto
    API-->>UI: AgentRunDto
```

The manifest pins resolved versions at creation time.

## 5. CLI Agent Callback

```mermaid
sequenceDiagram
    participant Agent as CLI Agent
    participant API as Requirement API
    participant AgentSvc as RequirementAgentRunService
    participant Docs as SddDocumentService
    participant DB as DB

    Agent->>API: POST /requirements/agent-runs/{executionId}/callback
    API->>AgentSvc: updateStatus(callback)
    AgentSvc->>DB: Save status, output summary, errors
    alt Artifacts included
        AgentSvc->>Docs: Upsert document indexes / artifact links
        Docs->>DB: Save GitHub PR and doc metadata
    end
    AgentSvc-->>API: AgentRunDto
```

Supported callback statuses:

- RUNNING
- COMPLETED
- FAILED
- STALE_CONTEXT
- CANCELED

## 6. Freshness Refresh

```mermaid
sequenceDiagram
    participant UI as Requirement Detail
    participant API as Requirement API
    participant Fresh as FreshnessService
    participant Source as SourceReferenceService
    participant Docs as SddDocumentService
    participant Review as DocumentReviewService

    UI->>API: GET /requirements/{id}/traceability
    API->>Fresh: buildTraceability(id)
    Fresh->>Source: Load source refs
    Fresh->>Docs: Load indexed docs
    Fresh->>Review: Load reviews
    Fresh->>Fresh: Compute freshness states
    Fresh-->>API: TraceabilityDto
    API-->>UI: TraceabilityDto
```

Freshness states:

- FRESH
- SOURCE_CHANGED
- DOCUMENT_CHANGED_AFTER_REVIEW
- MISSING_DOCUMENT
- MISSING_SOURCE
- UNKNOWN
- ERROR

## 7. Profile-Driven Document Rendering

```mermaid
sequenceDiagram
    participant UI as Requirement Detail
    participant API as Requirement API
    participant Profile as PipelineProfileService
    participant Docs as SddDocumentService

    UI->>API: GET /requirements/{id}/sdd-documents
    API->>Profile: Resolve active profile
    API->>Docs: Load indexed documents
    Docs-->>API: Existing document refs
    API->>API: Merge profile expected stages with docs
    API-->>UI: Stage list with present/missing docs
```

The UI renders missing expected documents rather than hiding gaps.
