# Requirement Control Plane API Implementation Guide

## Purpose

This guide defines API contracts for the Requirement Control Plane adjustment.
All responses use the shared `ApiResponse<T>` envelope unless stated otherwise.

## Base Path

```text
/api/v1/requirements
```

## 1. Source References

### GET `/api/v1/requirements/{requirementId}/sources`

Returns source references linked to a requirement.

```json
{
  "data": [
    {
      "id": "src-001",
      "requirementId": "REQ-1024",
      "sourceType": "JIRA",
      "externalId": "PAY-123",
      "title": "Payment reconciliation enhancement",
      "url": "jira://PAY-123",
      "sourceUpdatedAt": "2026-04-27T08:12:00Z",
      "fetchedAt": "2026-04-27T09:00:00Z",
      "freshnessStatus": "FRESH",
      "errorMessage": null
    }
  ],
  "error": null
}
```

### POST `/api/v1/requirements/{requirementId}/sources`

Request:

```json
{
  "sourceType": "JIRA",
  "url": "jira://PAY-123"
}
```

Response: `SourceReferenceDto`

### POST `/api/v1/requirements/sources/{sourceId}/refresh`

Refreshes metadata from the source provider. Response: `SourceReferenceDto`.

## 2. SDD Documents

### GET `/api/v1/requirements/{requirementId}/sdd-documents?profileId={profileId}`

Returns profile stages merged with indexed GitHub documents.

`profileId` is optional. When present, the backend must render expected stages
from that profile, even when the indexed documents for that requirement were
created under another profile. This keeps the SDD document panel aligned with
the profile selected by the user, such as `ibm-i`.

`workspace.applicationId`, `workspace.applicationName`, and
`workspace.snowGroup` are resolved from platform/workspace configuration. The
Requirement UI uses them to filter and label the active SDD workspace; ownership
configuration remains outside this endpoint.

`stageLabel` is the expected profile stage name. `title` is the document display
name and must come from the indexed Markdown title, normally the first Markdown
H1 at the pinned commit/blob. If no H1 is available, the indexer may fall back
to a normalized path basename. The UI must display `title` as the clickable
document name and may show `stageLabel` as secondary classification.

All profile stage path patterns returned by this endpoint must resolve under
`workspace.docsRoot`, which is `docs/` for the central SDD repo. CLI skills may
use temporary working folders, but reviewable SDD Markdown must be indexed under
`docs/`.

The project boundary is `workspace.sddRepoFullName + workspace.workingBranch`.
The final file name is not a project boundary. A document instance is identified
by repo, branch/ref, path, commit SHA, and blob SHA.

For missing documents, `path` should contain the resolved expected path whenever
the required token values are known. `pathPattern` keeps the original profile
template, and `unresolvedTokens` lists tokens that could not yet be resolved.

Profiles may produce multiple documents for one stage. The target response
shape is `stageGroups[].documents[]`. The flattened `stages[]` array may remain
temporarily for backward-compatible UI rendering.

```json
{
  "data": {
    "requirementId": "REQ-1024",
    "profileId": "ibm-i",
    "workspace": {
      "id": "SDDW-PAY-2026-SSO",
      "applicationId": "payment-gateway",
      "applicationName": "Payment Gateway",
      "snowGroup": "APAC-PAYMENTS-L2",
      "sourceRepoFullName": "wwa-lab/payment-gateway-service",
      "sddRepoFullName": "wwa-lab/payment-gateway-sdd",
      "baseBranch": "main",
      "workingBranch": "project/PAY-2026-sso-upgrade",
      "lifecycleStatus": "IN_DEVELOPMENT",
      "docsRoot": "docs/",
      "releasePrUrl": null,
      "kbRepoFullName": "wwa-lab/payment-gateway-knowledge-base",
      "kbMainBranch": "main",
      "kbPreviewBranch": "project/PAY-2026-sso-upgrade",
      "graphManifestPath": "_graph/manifest.json"
    },
    "stages": [
      {
        "id": "doc-001",
        "sddType": "functional-spec",
        "stageLabel": "Functional Spec",
        "documentInstanceKey": "BR-20260406-001",
        "title": "Payment Reconciliation Functional Spec",
        "titleSource": "H1",
        "repoFullName": "wwa-lab/payment-gateway-sdd",
        "branchOrRef": "project/PAY-2026-sso-upgrade",
        "path": "docs/02-functional-spec/payment-reconciliation.md",
        "pathPattern": "docs/02-functional-spec/{br-id}.md",
        "pathVariables": { "br-id": "BR-20260406-001" },
        "unresolvedTokens": [],
        "latestCommitSha": "abc123",
        "latestBlobSha": "def456",
        "githubUrl": "https://github.com/wwa-lab/payment-gateway-sdd/blob/project/PAY-2026-sso-upgrade/docs/02-functional-spec/payment-reconciliation.md",
        "status": "IN_REVIEW",
        "freshnessStatus": "FRESH",
        "missing": false
      },
      {
        "id": null,
        "sddType": "program-spec",
        "stageLabel": "Program Spec",
        "documentInstanceKey": "PAYPGM01",
        "title": "Program Spec - PAYPGM01",
        "titleSource": "TOKEN_CONTEXT",
        "repoFullName": null,
        "branchOrRef": null,
        "path": "docs/04-program-spec/PAYPGM01.md",
        "pathPattern": "docs/04-program-spec/{program}.md",
        "pathVariables": { "program": "PAYPGM01" },
        "unresolvedTokens": [],
        "latestCommitSha": null,
        "latestBlobSha": null,
        "githubUrl": null,
        "status": "MISSING",
        "freshnessStatus": "MISSING_DOCUMENT",
        "missing": true
      }
    ],
    "stageGroups": [
      {
        "sddType": "program-spec",
        "stageLabel": "Program Spec",
        "expectedTier": "L2",
        "required": true,
        "documents": [
          {
            "id": null,
            "documentInstanceKey": "PAYPGM01",
            "title": "Program Spec - PAYPGM01",
            "path": "docs/04-program-spec/PAYPGM01.md",
            "pathPattern": "docs/04-program-spec/{program}.md",
            "missing": true
          },
          {
            "id": null,
            "documentInstanceKey": "PAYPGM02",
            "title": "Program Spec - PAYPGM02",
            "path": "docs/04-program-spec/PAYPGM02.md",
            "pathPattern": "docs/04-program-spec/{program}.md",
            "missing": true
          }
        ]
      }
    ]
  },
  "error": null
}
```

### GET `/api/v1/requirements/documents/{documentId}`

Fetches latest Markdown content from GitHub.

```json
{
  "data": {
    "document": {
      "id": "doc-001",
      "sddType": "functional-spec",
      "stageLabel": "Functional Spec",
      "title": "Payment Reconciliation Functional Spec",
      "repoFullName": "wwa-lab/payment-gateway-sdd",
      "branchOrRef": "project/PAY-2026-sso-upgrade",
      "path": "docs/02-functional-spec/payment-reconciliation.md",
      "latestCommitSha": "abc123",
      "latestBlobSha": "def456",
      "githubUrl": "https://github.com/wwa-lab/payment-gateway-sdd/blob/project/PAY-2026-sso-upgrade/docs/02-functional-spec/payment-reconciliation.md",
      "status": "IN_REVIEW",
      "freshnessStatus": "FRESH",
      "missing": false
    },
    "markdown": "# Functional Spec\n\n...",
    "commitSha": "abc123",
    "blobSha": "def456",
    "githubUrl": "https://github.com/wwa-lab/payment-gateway-sdd/blob/project/PAY-2026-sso-upgrade/docs/02-functional-spec/payment-reconciliation.md",
    "fetchedAt": "2026-04-27T09:05:00Z"
  },
  "error": null
}
```

## 3. Reviews

### POST `/api/v1/requirements/documents/{documentId}/reviews`

When `decision` is `REJECTED`, `comment` is required and must contain the
business reason for rejection. The backend returns a validation error for empty
rejection reasons.

Request:

```json
{
  "decision": "APPROVED",
  "comment": "Business scope is approved for the reviewed version.",
  "commitSha": "abc123",
  "blobSha": "def456",
  "anchorType": "DOCUMENT",
  "anchorValue": null
}
```

Response:

```json
{
  "data": {
    "id": "review-001",
    "documentId": "doc-001",
    "requirementId": "REQ-1024",
    "decision": "APPROVED",
    "comment": "Business scope is approved for the reviewed version.",
    "reviewerId": "u-123",
    "reviewerType": "BUSINESS",
    "commitSha": "abc123",
    "blobSha": "def456",
    "createdAt": "2026-04-27T09:10:00Z"
  },
  "error": null
}
```

### GET `/api/v1/requirements/{requirementId}/reviews`

Returns review history for all documents linked to a requirement.

## 4. Agent Runs

### POST `/api/v1/requirements/{requirementId}/agent-runs`

Creates a manifest for CLI-agent execution.

Request:

```json
{
  "skillKey": "ibm-i-workflow-orchestrator",
  "targetStage": "program-spec",
  "profileId": "ibm-i",
  "notes": "Generate the next IBM i artifact from current Jira and Functional Spec context."
}
```

Response:

```json
{
  "data": {
    "executionId": "exec-1024",
    "requirementId": "REQ-1024",
    "profileId": "ibm-i",
    "skillKey": "ibm-i-workflow-orchestrator",
    "status": "MANIFEST_READY",
    "manifest": {
      "executionId": "exec-1024",
      "requirementId": "REQ-1024",
      "profileId": "ibm-i",
      "repo": {
        "fullName": "wwa-lab/payment-app",
        "baseRef": "main"
      },
      "sources": [
        {
          "id": "src-001",
          "type": "JIRA",
          "url": "jira://PAY-123",
          "externalId": "PAY-123",
          "versionKey": "updated-2026-04-27T08:12:00Z"
        }
      ],
      "documents": [
        {
          "id": "doc-001",
          "type": "functional-spec",
          "path": "docs/02-functional-spec/payment-reconciliation.md",
          "commitSha": "abc123",
          "blobSha": "def456"
        }
      ],
      "output": {
        "docsRoot": "docs/",
        "targetStage": "program-spec"
      }
    },
    "artifactLinks": [],
    "createdAt": "2026-04-27T09:12:00Z"
  },
  "error": null
}
```

### GET `/api/v1/requirements/agent-runs/{executionId}`

Returns current run status.

### POST `/api/v1/requirements/agent-runs/{executionId}/callback`

Agent callback request:

```json
{
  "status": "COMPLETED",
  "outputSummary": {
    "message": "Program Spec generated and PR opened."
  },
  "artifactLinks": [
    {
      "artifactType": "GITHUB_PR",
      "storageType": "GITHUB",
      "title": "Generate IBM i Program Spec",
      "uri": "https://github.com/wwa-lab/payment-app/pull/42",
      "repoFullName": "wwa-lab/payment-app",
      "path": null,
      "commitSha": "789abc",
      "blobSha": null,
      "status": "IN_REVIEW"
    }
  ],
  "errorMessage": null
}
```

## 5. Traceability

### GET `/api/v1/requirements/{requirementId}/traceability`

Returns sources, docs, reviews, agent runs, artifact links, and freshness states.

## Error Handling

| Status | Case | Error |
|---|---|---|
| 400 | Missing URL or invalid review version | Human-readable validation message |
| 404 | Requirement, source, document, or run not found | Resource not found message |
| 409 | Review against stale content when strict mode enabled | Version conflict |
| 502 | External source or GitHub unavailable | Provider error |
| 500 | Unexpected failure | Generic failure message |

## Testing Contracts

- Source reference creation handles provider success and provider error.
- GitHub document fetch returns markdown and SHA metadata.
- Review creation rejects missing commit/blob.
- Agent manifest includes sources, profile, documents, and output target.
- Callback updates run status and artifact links.
- Traceability includes stale review when blob changes.
