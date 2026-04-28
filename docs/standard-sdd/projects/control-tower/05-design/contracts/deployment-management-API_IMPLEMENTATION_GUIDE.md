# Deployment Management — API Implementation Guide

## Source

- Requirements: [../../01-requirements/deployment-management-requirements.md](../../01-requirements/deployment-management-requirements.md)
- Stories: [../../02-user-stories/deployment-management-stories.md](../../02-user-stories/deployment-management-stories.md)
- Spec: [../../03-spec/deployment-management-spec.md](../../03-spec/deployment-management-spec.md)
- Architecture: [../../04-architecture/deployment-management-architecture.md](../../04-architecture/deployment-management-architecture.md)
- Data flow: [../../04-architecture/deployment-management-data-flow.md](../../04-architecture/deployment-management-data-flow.md)
- Data model: [../../04-architecture/deployment-management-data-model.md](../../04-architecture/deployment-management-data-model.md)
- Design: [../deployment-management-design.md](../deployment-management-design.md)

## 1. Conventions

- **Base path:** `/api/v1/deployment-management`
- **Envelope:** every response wrapped in `ApiResponse<T>`:
  ```json
  { "data": { /* T */ }, "meta": { "correlationId": "dpm-…", "generatedAt": "2026-04-18T10:00:00Z" }, "error": null }
  ```
- **Section results:** aggregate endpoints return `SectionResult<T>` per card: `{ "data": T | null, "error": { "code", "message" } | null, "loadedAt": "…" }`
- **Timestamps:** ISO-8601 UTC with trailing `Z`.
- **Pagination:** cursor-based (`?cursor=…&limit=…`, default 20, max 100).
- **Correlation ID:** client-supplied `x-correlation-id` header is honored; server generates if absent. Always echoed in `meta.correlationId`.
- **Webhook base path:** `/api/v1/deployment-management/webhooks/jenkins` (separate path prefix, skipped by the standard auth filter; Jenkins authenticates via HMAC signature).
- **IDs:** `application-…`, `release-…`, `deploy-…`, `env-…`, `approval-…`, `release-notes-…`, `deploy-summary-…`, `STORY-…`.

## 2. Endpoint Overview

### 2.1 Catalog

| Method | Path | Purpose |
| ------ | ---- | ------- |
| GET | `/catalog` | Aggregate (Summary + Grid + AiSummary) |
| GET | `/catalog/summary` | Summary card only |
| GET | `/catalog/grid` | Grid card only |
| GET | `/catalog/ai-summary` | AI Summary card |
| POST | `/catalog/ai-summary/regenerate` | Admin-only manual regen (rate-limited 1/min) |

Query params for `/catalog*`: `workspaceIds`, `projectId`, `environmentKind` (`DEV`/`QA`/`STAGING`/`PROD`/`CANARY`), `deployState`, `trigger`, `timeWindow` (`24h`/`7d`/`30d`), `search`.

### 2.2 Application Detail

| Method | Path | Purpose |
| ------ | ---- | ------- |
| GET | `/applications/{applicationId}` | 6-card aggregate |
| GET | `/applications/{applicationId}/header` | — |
| GET | `/applications/{applicationId}/environments` | — |
| GET | `/applications/{applicationId}/recent-releases` | — |
| GET | `/applications/{applicationId}/recent-deploys` | — |
| GET | `/applications/{applicationId}/metrics` | — |
| GET | `/applications/{applicationId}/ai-insights` | — |

### 2.3 Release Detail

| Method | Path | Purpose |
| ------ | ---- | ------- |
| GET | `/releases/{releaseId}` | 5-card aggregate |
| GET | `/releases/{releaseId}/header` | — |
| GET | `/releases/{releaseId}/composition` | — |
| GET | `/releases/{releaseId}/deploys` | — |
| GET | `/releases/{releaseId}/linked-stories` | — |
| GET | `/releases/{releaseId}/ai-notes` | — |
| POST | `/releases/{releaseId}/ai-notes/regenerate` | Admin-only; debounced 60s server-side |

### 2.4 Deploy Detail

| Method | Path | Purpose |
| ------ | ---- | ------- |
| GET | `/deploys/{deployId}` | 5-card aggregate |
| GET | `/deploys/{deployId}/header` | — |
| GET | `/deploys/{deployId}/stages` | — |
| GET | `/deploys/{deployId}/approvals` | — |
| GET | `/deploys/{deployId}/change-summary` | — |
| GET | `/deploys/{deployId}/ai-summary` | — |
| POST | `/deploys/{deployId}/ai-summary/regenerate` | Admin-only |

### 2.5 Environment Detail

| Method | Path | Purpose |
| ------ | ---- | ------- |
| GET | `/applications/{applicationId}/environments/{name}` | 4-card aggregate |
| GET | `/applications/{applicationId}/environments/{name}/header` | — |
| GET | `/applications/{applicationId}/environments/{name}/current` | — |
| GET | `/applications/{applicationId}/environments/{name}/timeline` | — |
| GET | `/applications/{applicationId}/environments/{name}/stability` | — |

### 2.6 Traceability

| Method | Path | Purpose |
| ------ | ---- | ------- |
| GET | `/traceability?storyId=…` | Inverse lookup aggregate |
| GET | `/traceability/releases?storyId=…` | — |
| GET | `/traceability/deploys?storyId=…` | — |

### 2.7 Webhooks

| Method | Path | Purpose |
| ------ | ---- | ------- |
| POST | `/webhooks/jenkins` | Jenkins Notification Plugin event receiver |

### 2.8 Cross-slice facade

| Method | Path | Purpose |
| ------ | ---- | ------- |
| GET | `/internal/story/{storyId}/deploy-aggregate` | Condensed Release + Deploy view for Story Detail's Deployment tab |
| GET | `/internal/release/{releaseId}/commit-slice` | Returns `buildArtifactSliceId` / `buildArtifactId` for Code & Build to resolve commit range |

Internal endpoints require `X-Service-Token` header; never exposed to browser.

## 3. Example Payloads

### 3.1 `GET /catalog` (200)

```json
{
  "data": {
    "filtersEcho": { "workspaceIds": ["ws-001"], "timeWindow": "7d" },
    "summary": {
      "data": {
        "visibleApplications": 14,
        "deploysLast7d": 82,
        "successRate7d": 0.89,
        "meanDeployDurationSec": 612,
        "deploymentFrequencyPerDay": 11.7,
        "changeFailureRate7d": 0.09,
        "mttrMinutes7d": 22,
        "byState": { "SUCCEEDED": 71, "FAILED": 6, "ROLLED_BACK": 2, "IN_PROGRESS": 3 },
        "byEnvironment": { "DEV": 31, "QA": 22, "STAGING": 18, "PROD": 11 }
      },
      "error": null,
      "loadedAt": "2026-04-18T10:00:01Z"
    },
    "grid": {
      "data": [
        {
          "projectId": "proj-billing",
          "projectName": "Billing Platform",
          "applications": [
            {
              "applicationId": "application-5521",
              "slug": "billing-api",
              "displayName": "Billing API",
              "projectId": "proj-billing",
              "workspaceId": "ws-001",
              "jenkinsInstanceId": "jenkins-inst-1",
              "jenkinsJobFullName": "billing/billing-api/main",
              "environments": [
                { "name": "dev", "kind": "DEV",
                  "currentReleaseVersion": "2026.04.18-1247",
                  "currentDeployState": "SUCCEEDED",
                  "lastDeployedAt": "2026-04-18T09:42:11Z" },
                { "name": "qa", "kind": "QA",
                  "currentReleaseVersion": "2026.04.17-1102",
                  "currentDeployState": "SUCCEEDED",
                  "lastDeployedAt": "2026-04-17T17:30:00Z" },
                { "name": "staging", "kind": "STAGING",
                  "currentReleaseVersion": "2026.04.16-0942",
                  "currentDeployState": "SUCCEEDED",
                  "lastDeployedAt": "2026-04-16T09:42:11Z" },
                { "name": "prod", "kind": "PROD",
                  "currentReleaseVersion": "2026.04.15-0830",
                  "currentDeployState": "ROLLED_BACK",
                  "lastDeployedAt": "2026-04-15T11:05:00Z" }
              ],
              "overallLed": "AMBER",
              "lastActivityAt": "2026-04-18T09:42:11Z"
            }
          ]
        }
      ],
      "error": null,
      "loadedAt": "2026-04-18T10:00:01Z"
    },
    "aiSummary": {
      "data": {
        "status": "SUCCESS",
        "generatedAt": "2026-04-18T09:55:00Z",
        "skillVersion": "dp-summary-v2",
        "narrative": "Last 7d, `billing-api` deployed 11 times with 1 rollback on prod (release 2026.04.15-0830). The `checkout-service` PROD cadence has stabilized to once-per-day. Consider reviewing the rollback cause before the next prod push.",
        "evidence": [
          { "kind": "deploy", "id": "deploy-88121", "label": "billing-api prod #5521 ROLLED_BACK" },
          { "kind": "release", "id": "release-44019", "label": "billing-api 2026.04.15-0830" }
        ]
      },
      "error": null,
      "loadedAt": "2026-04-18T10:00:01Z"
    }
  },
  "meta": { "correlationId": "dpm-abc123", "generatedAt": "2026-04-18T10:00:01Z" },
  "error": null
}
```

### 3.2 `GET /applications/{applicationId}` (200)

```json
{
  "data": {
    "header": { "data": {
      "applicationId": "application-5521",
      "slug": "billing-api",
      "displayName": "Billing API",
      "projectId": "proj-billing",
      "workspaceId": "ws-001",
      "jenkinsInstanceId": "jenkins-inst-1",
      "jenkinsJobFullName": "billing/billing-api/main",
      "jenkinsJobUrl": "https://jenkins.acme.example/job/billing/job/billing-api/job/main/",
      "description": "Billing REST API deployment job"
    }, "error": null, "loadedAt": "2026-04-18T10:00:02Z" },
    "environments": { "data": [
      { "name": "dev", "kind": "DEV",
        "currentReleaseId": "release-44030", "currentReleaseVersion": "2026.04.18-1247",
        "currentDeployId": "deploy-88202", "currentDeployState": "SUCCEEDED",
        "lastDeployedAt": "2026-04-18T09:42:11Z", "lastDeployer": "jane@acme.example" },
      { "name": "prod", "kind": "PROD",
        "currentReleaseId": "release-44019", "currentReleaseVersion": "2026.04.15-0830",
        "currentDeployId": "deploy-88121", "currentDeployState": "ROLLED_BACK",
        "lastDeployedAt": "2026-04-15T11:05:00Z", "lastDeployer": "release-bot@acme.example" }
    ], "error": null, "loadedAt": "…" },
    "recentReleases": { "data": [
      { "releaseId": "release-44030", "releaseVersion": "2026.04.18-1247",
        "state": "DEPLOYED", "createdAt": "2026-04-18T09:15:00Z",
        "buildArtifactSliceId": "cb", "buildArtifactId": "artifact-99012",
        "deployCount": 2 }
    ], "error": null, "loadedAt": "…" },
    "recentDeploys": { "data": [
      { "deployId": "deploy-88202", "releaseId": "release-44030", "releaseVersion": "2026.04.18-1247",
        "environmentName": "dev", "environmentKind": "DEV",
        "state": "SUCCEEDED", "trigger": "WEBHOOK", "rolledBack": false,
        "startedAt": "2026-04-18T09:40:00Z", "completedAt": "2026-04-18T09:42:11Z",
        "durationSec": 131, "actor": "jane@acme.example",
        "jenkinsBuildUrl": "https://jenkins.acme.example/job/billing/job/billing-api/job/main/5521/" }
    ], "error": null, "loadedAt": "…" },
    "metrics": { "data": {
      "window": "30d",
      "deploymentFrequencyPerDay": 1.7,
      "changeFailureRate": 0.08,
      "mttrMinutes": 18,
      "meanLeadTimeMinutes": 142,
      "byEnvironment": {
        "DEV": { "deploys": 38, "failureRate": 0.05 },
        "PROD": { "deploys": 12, "failureRate": 0.17, "rollbacks": 1 }
      }
    }, "error": null, "loadedAt": "…" },
    "aiInsights": { "data": {
      "status": "SUCCESS",
      "generatedAt": "2026-04-18T09:55:00Z",
      "skillVersion": "dp-app-insights-v1",
      "narrative": "Prod rollback rate (8%) is above workspace median (3%). The most recent rollback (2026-04-15) was triggered 7 minutes after deploy.",
      "evidence": [ { "kind": "deploy", "id": "deploy-88121", "label": "prod #5521" } ]
    }, "error": null, "loadedAt": "…" }
  },
  "meta": { "correlationId": "dpm-def456", "generatedAt": "2026-04-18T10:00:02Z" },
  "error": null
}
```

### 3.3 `GET /releases/{releaseId}` — Release with AI Notes

```json
{
  "data": {
    "header": { "data": {
      "releaseId": "release-44030",
      "applicationId": "application-5521",
      "applicationSlug": "billing-api",
      "releaseVersion": "2026.04.18-1247",
      "state": "DEPLOYED",
      "buildArtifactSliceId": "cb",
      "buildArtifactId": "artifact-99012",
      "createdAt": "2026-04-18T09:15:00Z",
      "firstDeployedAt": "2026-04-18T09:42:11Z",
      "lastDeployedAt": "2026-04-18T09:42:11Z"
    }, "error": null, "loadedAt": "…" },
    "composition": { "data": {
      "buildArtifactId": "artifact-99012",
      "sourceRunId": "run-99012",
      "headSha": "a1b2c3d4e5f6a1b2c3d4e5f6a1b2c3d4e5f6a1b2",
      "priorReleaseVersion": "2026.04.17-1102",
      "priorHeadSha": "0f0e0d0c0b0a0f0e0d0c0b0a0f0e0d0c0b0a0f0e",
      "commits": [
        { "sha": "a1b2c3d4e5f6a1b2c3d4e5f6a1b2c3d4e5f6a1b2",
          "shortSha": "a1b2c3d4",
          "author": "jane@acme.example",
          "message": "feat: adjust tax rounding\n\nStory-Id: STORY-4211",
          "committedAt": "2026-04-18T09:00:00Z",
          "storyChips": [ { "storyId": "STORY-4211", "status": "VERIFIED",
            "title": "Tax rounding cleanup", "projectId": "proj-billing" } ] }
      ],
      "commitCount": 1,
      "capNotice": null
    }, "error": null, "loadedAt": "…" },
    "deploys": { "data": [
      { "deployId": "deploy-88202", "environmentName": "dev", "environmentKind": "DEV",
        "state": "SUCCEEDED", "trigger": "WEBHOOK", "rolledBack": false,
        "startedAt": "2026-04-18T09:40:00Z", "completedAt": "2026-04-18T09:42:11Z" }
    ], "error": null, "loadedAt": "…" },
    "linkedStories": { "data": [
      { "storyId": "STORY-4211", "status": "VERIFIED",
        "title": "Tax rounding cleanup", "projectId": "proj-billing",
        "commitCount": 1 }
    ], "error": null, "loadedAt": "…" },
    "aiNotes": { "data": {
      "status": "SUCCESS",
      "keyedOnReleaseId": "release-44030",
      "skillVersion": "dp-release-notes-v3",
      "generatedAt": "2026-04-18T09:20:00Z",
      "headline": "Tax rounding fix (HALF_EVEN) lands in billing-api 2026.04.18-1247",
      "highlights": [
        "Fixes boundary-case over-charge on refunds (STORY-4211)"
      ],
      "bodyMarkdown": "## What changed\n- Tax rounding now uses HALF_EVEN …\n\n## Risk\n- Low — isolated to TaxService.",
      "evidence": [
        { "kind": "commit", "id": "a1b2c3d4", "label": "feat: adjust tax rounding" },
        { "kind": "story", "id": "STORY-4211", "label": "Tax rounding cleanup" }
      ],
      "evidenceHash": "sha256:3f9a…"
    }, "error": null, "loadedAt": "…" }
  },
  "meta": { "correlationId": "dpm-ghi789", "generatedAt": "2026-04-18T10:00:03Z" },
  "error": null
}
```

### 3.4 `GET /deploys/{deployId}` — FAILED deploy with approvals

```json
{
  "data": {
    "header": { "data": {
      "deployId": "deploy-88121",
      "releaseId": "release-44019",
      "releaseVersion": "2026.04.15-0830",
      "applicationId": "application-5521",
      "applicationSlug": "billing-api",
      "environmentName": "prod", "environmentKind": "PROD",
      "state": "ROLLED_BACK",
      "trigger": "MANUAL",
      "rolledBack": true,
      "rollbackDetectionSignal": "trigger=rollback",
      "actor": "release-bot@acme.example",
      "startedAt": "2026-04-15T10:58:00Z",
      "completedAt": "2026-04-15T11:05:00Z",
      "durationSec": 420,
      "jenkinsInstanceId": "jenkins-inst-1",
      "jenkinsJobFullName": "billing/billing-api/main",
      "jenkinsBuildNumber": 5519,
      "jenkinsBuildUrl": "https://jenkins.acme.example/job/billing/job/billing-api/job/main/5519/"
    }, "error": null, "loadedAt": "…" },
    "stages": { "data": [
      { "stageId": "stage-1", "order": 1, "name": "Checkout", "state": "SUCCESS",
        "startedAt": "2026-04-15T10:58:05Z", "completedAt": "2026-04-15T10:58:20Z", "durationSec": 15 },
      { "stageId": "stage-2", "order": 2, "name": "Deploy to Prod", "state": "SUCCESS",
        "startedAt": "2026-04-15T10:58:20Z", "completedAt": "2026-04-15T11:01:00Z", "durationSec": 160 },
      { "stageId": "stage-3", "order": 3, "name": "Smoke", "state": "FAILURE",
        "startedAt": "2026-04-15T11:01:00Z", "completedAt": "2026-04-15T11:04:00Z", "durationSec": 180,
        "logExcerpt": { "redacted": true, "text": "[ERROR] /health returned 500 …", "bytes": 2048 } }
    ], "error": null, "loadedAt": "…" },
    "approvals": { "data": [
      { "approvalEventId": "approval-7701", "stageOrder": 2, "stepName": "Promote to Prod",
        "state": "APPROVED",
        "promptedAt": "2026-04-15T10:56:00Z", "resolvedAt": "2026-04-15T10:57:30Z",
        "approver": "kim@acme.example",
        "rationale": "scheduled weekly release — backout plan reviewed" }
    ], "error": null, "loadedAt": "…" },
    "changeSummary": { "data": {
      "priorSuccessfulDeployId": "deploy-88098",
      "priorReleaseVersion": "2026.04.08-0915",
      "priorHeadSha": "77665544aabbccddeeff77665544aabbccddeeff",
      "headSha": "0f0e0d0c0b0a0f0e0d0c0b0a0f0e0d0c0b0a0f0e",
      "commitCount": 12,
      "linkedStoryIds": ["STORY-4199","STORY-4202","STORY-4211"],
      "capNotice": null
    }, "error": null, "loadedAt": "…" },
    "aiSummary": { "data": {
      "status": "SUCCESS",
      "keyedOnDeployId": "deploy-88121",
      "skillVersion": "dp-deploy-summary-v1",
      "generatedAt": "2026-04-15T11:06:00Z",
      "headline": "Prod deploy rolled back after smoke failure",
      "narrative": "Stage 'Smoke' failed with HTTP 500 on /health 3 minutes after promotion. Triggered rollback at 11:05. Most likely cause: migration V48 not applied to prod DB.",
      "evidence": [
        { "kind": "stage", "id": "stage-3", "label": "Smoke (FAILURE)" },
        { "kind": "deploy", "id": "deploy-88122", "label": "rollback deploy" }
      ],
      "evidenceHash": "sha256:7a4b…"
    }, "error": null, "loadedAt": "…" }
  },
  "meta": { "correlationId": "dpm-jkl012", "generatedAt": "2026-04-18T10:00:04Z" },
  "error": null
}
```

Approvals for a principal without `deployment:view-approvals`: `rationale` is replaced with `"(redacted)"` and the approver display name is replaced with `"(redacted)"`; the event id and timestamps remain.

### 3.5 `GET /applications/{applicationId}/environments/{name}`

```json
{
  "data": {
    "header": { "data": {
      "applicationId": "application-5521", "applicationSlug": "billing-api",
      "environmentName": "prod", "environmentKind": "PROD",
      "currentReleaseId": "release-44019", "currentReleaseVersion": "2026.04.15-0830",
      "lastStableReleaseVersion": "2026.04.08-0915"
    }, "error": null, "loadedAt": "…" },
    "current": { "data": {
      "deployId": "deploy-88121", "state": "ROLLED_BACK",
      "deployedAt": "2026-04-15T11:05:00Z",
      "deployer": "release-bot@acme.example"
    }, "error": null, "loadedAt": "…" },
    "timeline": { "data": [
      { "deployId": "deploy-88121", "releaseVersion": "2026.04.15-0830",
        "state": "ROLLED_BACK", "trigger": "MANUAL", "rolledBack": true,
        "startedAt": "2026-04-15T10:58:00Z", "completedAt": "2026-04-15T11:05:00Z" }
    ], "error": null, "loadedAt": "…" },
    "stability": { "data": {
      "window": "30d",
      "deploys": 12,
      "changeFailureRate": 0.17,
      "mttrMinutes": 42,
      "lastFailureAt": "2026-04-15T11:05:00Z",
      "lastRollbackAt": "2026-04-15T11:05:00Z"
    }, "error": null, "loadedAt": "…" }
  },
  "meta": { "correlationId": "dpm-mno345", "generatedAt": "…" },
  "error": null
}
```

### 3.6 `GET /traceability?storyId=STORY-4211`

```json
{
  "data": {
    "storyChip": { "storyId": "STORY-4211", "status": "VERIFIED",
      "title": "Tax rounding cleanup", "projectId": "proj-billing" },
    "releases": { "data": [
      { "releaseId": "release-44030", "applicationSlug": "billing-api",
        "releaseVersion": "2026.04.18-1247", "state": "DEPLOYED",
        "createdAt": "2026-04-18T09:15:00Z" }
    ], "error": null, "loadedAt": "…" },
    "deploys": { "data": [
      { "deployId": "deploy-88202", "environmentKind": "DEV",
        "state": "SUCCEEDED", "trigger": "WEBHOOK",
        "completedAt": "2026-04-18T09:42:11Z" }
    ], "error": null, "loadedAt": "…" },
    "capNotice": null
  },
  "meta": { "correlationId": "dpm-pqr678", "generatedAt": "…" },
  "error": null
}
```

### 3.7 Error envelope

```json
{
  "data": null,
  "meta": { "correlationId": "dpm-err001", "generatedAt": "…" },
  "error": { "code": "DP_WORKSPACE_FORBIDDEN", "message": "Workspace not accessible to caller" }
}
```

## 4. Error Code Catalog

| Code | HTTP | Retryable | Notes |
| ---- | ---- | --------- | ----- |
| `DP_WORKSPACE_FORBIDDEN` | 403 | No | Caller not a workspace member |
| `DP_ROLE_REQUIRED` | 403 | No | Admin-only action or restricted field attempted |
| `DP_APPLICATION_NOT_FOUND` | 404 | No | `applicationId` unknown or out of scope |
| `DP_RELEASE_NOT_FOUND` | 404 | No | |
| `DP_DEPLOY_NOT_FOUND` | 404 | No | |
| `DP_ENVIRONMENT_NOT_FOUND` | 404 | No | |
| `DP_STORY_NOT_FOUND` | 404 | No | Story not visible or unknown in Requirement slice |
| `DP_AI_AUTONOMY_INSUFFICIENT` | 409 | No | Workspace autonomy below OBSERVATION floor |
| `DP_AI_UNAVAILABLE` | 503 | Yes | AI skill client transient failure |
| `DP_RATE_LIMITED` | 429 | Yes | Includes `Retry-After` header |
| `DP_JENKINS_UNREACHABLE` | 503 | Yes | Jenkins primary/secondary failure on backfill path |
| `DP_INGEST_SIGNATURE_INVALID` | 401 | No | Webhook signature failed |
| `DP_INGEST_PAYLOAD_INVALID` | 400 | No | Webhook payload parse failed |
| `DP_JENKINS_INSTANCE_UNKNOWN` | 404 | No | Unknown Jenkins instance id on webhook |
| `DP_RELEASE_NOTES_EVIDENCE_MISMATCH` | 422 | No | Release notes or deploy summary rejected at persist time |
| `DP_BUILD_ARTIFACT_PENDING` | 409 | Yes | Upstream Code & Build artifact not yet ingested — try later |
| `DP_RELEASE_UNRESOLVED` | 409 | Yes | `RELEASE_VERSION` param missing on Jenkins build — Release row lazy-creation deferred |
| `DP_RANGE_TOO_LARGE` | 422 | No | Commit range exceeds 100-commit cap |

## 5. Backend Controller Skeleton (Spring Boot)

```java
@RestController
@RequestMapping("/api/v1/deployment-management")
@Validated
public class DeploymentController {

    private final CatalogService catalogService;
    private final ApplicationDetailService applicationDetailService;
    private final ReleaseDetailService releaseDetailService;
    private final DeployDetailService deployDetailService;
    private final EnvironmentDetailService environmentDetailService;
    private final TraceabilityService traceabilityService;
    private final AiReleaseNotesService aiReleaseNotesService;
    private final AiDeploymentSummaryService aiDeploymentSummaryService;
    private final DeploymentAccessGuard accessGuard;

    // ---- Catalog ----
    @GetMapping("/catalog")
    public ApiResponse<CatalogAggregateDto> getCatalog(
            @RequestParam(required = false) List<String> workspaceIds,
            @RequestParam(required = false) String projectId,
            @RequestParam(required = false) String environmentKind,
            @RequestParam(required = false) String deployState,
            @RequestParam(required = false) String trigger,
            @RequestParam(required = false, defaultValue = "7d") String timeWindow,
            @RequestParam(required = false) String search,
            Principal principal) {
        var filters = CatalogFilters.of(workspaceIds, projectId, environmentKind,
                deployState, trigger, timeWindow, search);
        return ApiResponse.ok(catalogService.loadAggregate(filters, principal));
    }

    @PostMapping("/catalog/ai-summary/regenerate")
    public ApiResponse<AiWorkspaceSummaryDto> regenerateSummary(
            @RequestParam String workspaceId, Principal principal) {
        accessGuard.requireAdmin(workspaceId, principal);
        return ApiResponse.ok(aiReleaseNotesService.regenerateWorkspaceSummary(workspaceId, principal));
    }

    // ---- Application Detail ----
    @GetMapping("/applications/{applicationId}")
    public ApiResponse<ApplicationDetailAggregateDto> getApplication(
            @PathVariable @Pattern(regexp = "^application-[a-z0-9\\-]+$") String applicationId,
            Principal principal) {
        accessGuard.requireReadOnApplication(applicationId, principal);
        return ApiResponse.ok(applicationDetailService.loadAggregate(applicationId, principal));
    }

    // ---- Release Detail ----
    @GetMapping("/releases/{releaseId}")
    public ApiResponse<ReleaseDetailAggregateDto> getRelease(
            @PathVariable @Pattern(regexp = "^release-[a-z0-9\\-]+$") String releaseId,
            Principal principal) {
        accessGuard.requireReadOnRelease(releaseId, principal);
        return ApiResponse.ok(releaseDetailService.loadAggregate(releaseId, principal));
    }

    @PostMapping("/releases/{releaseId}/ai-notes/regenerate")
    public ApiResponse<AiReleaseNotesDto> regenerateNotes(
            @PathVariable String releaseId, Principal principal) {
        accessGuard.requireAdminOnRelease(releaseId, principal);
        return ApiResponse.ok(aiReleaseNotesService.enqueueRegeneration(releaseId, principal));
    }

    // ---- Deploy Detail ----
    @GetMapping("/deploys/{deployId}")
    public ApiResponse<DeployDetailAggregateDto> getDeploy(
            @PathVariable @Pattern(regexp = "^deploy-[a-z0-9\\-]+$") String deployId,
            Principal principal) {
        accessGuard.requireReadOnDeploy(deployId, principal);
        return ApiResponse.ok(deployDetailService.loadAggregate(deployId, principal));
    }

    @PostMapping("/deploys/{deployId}/ai-summary/regenerate")
    public ApiResponse<AiDeploymentSummaryDto> regenerateDeploySummary(
            @PathVariable String deployId, Principal principal) {
        accessGuard.requireAdminOnDeploy(deployId, principal);
        return ApiResponse.ok(aiDeploymentSummaryService.enqueueRegeneration(deployId, principal));
    }

    // ---- Environment Detail ----
    @GetMapping("/applications/{applicationId}/environments/{name}")
    public ApiResponse<EnvironmentDetailAggregateDto> getEnvironment(
            @PathVariable String applicationId,
            @PathVariable @Pattern(regexp = "^[a-z][a-z0-9_\\-]{0,62}$") String name,
            Principal principal) {
        accessGuard.requireReadOnApplication(applicationId, principal);
        return ApiResponse.ok(environmentDetailService.loadAggregate(applicationId, name, principal));
    }

    // ---- Traceability ----
    @GetMapping("/traceability")
    public ApiResponse<TraceabilityAggregateDto> getTraceability(
            @RequestParam @Pattern(regexp = "^STORY-[A-Z0-9\\-]+$") String storyId,
            Principal principal) {
        return ApiResponse.ok(traceabilityService.inverseLookup(storyId, principal));
    }
}
```

### 5.1 Webhook Controller

```java
@RestController
@RequestMapping("/api/v1/deployment-management/webhooks")
public class JenkinsWebhookController {

    private final JenkinsSignatureVerifier verifier;
    private final JenkinsPayloadParser parser;
    private final IngestionOutboxWriter outboxWriter;
    private final AuditLogEmitter audit;

    @PostMapping(path = "/jenkins", consumes = "application/json")
    public ResponseEntity<Void> receive(
            @RequestHeader("X-Jenkins-Instance") String instanceId,
            @RequestHeader(value = "X-Jenkins-Delivery", required = false) String deliveryId,
            @RequestHeader("X-Jenkins-Event") String event,
            @RequestHeader("X-Jenkins-Signature") String signature,
            @RequestBody byte[] body) {

        if (!verifier.verify(instanceId, signature, body)) {
            audit.emit("DP_INGEST_SIGNATURE_INVALID",
                Map.of("deliveryId", String.valueOf(deliveryId), "event", event));
            return ResponseEntity.status(HttpStatus.UNAUTHORIZED).build();
        }

        TypedEvent typed;
        try {
            typed = parser.parse(event, body);
        } catch (UnknownInstanceException e) {
            audit.emit("DP_JENKINS_INSTANCE_UNKNOWN",
                Map.of("instanceId", instanceId, "event", event));
            return ResponseEntity.status(HttpStatus.NOT_FOUND).build();
        } catch (InvalidPayloadException e) {
            audit.emit("DP_INGEST_PAYLOAD_INVALID",
                Map.of("event", event, "deliveryId", String.valueOf(deliveryId)));
            return ResponseEntity.badRequest().build();
        }

        outboxWriter.enqueue(deliveryId, typed, body);  // idempotent on deliveryId
        return ResponseEntity.accepted().build();       // 202 — sync verify, async dispatch
    }
}
```

Supported Jenkins event types (from Notification Plugin / `post { … }` hooks):

- `job.started` — create/update `deploy` row, state=IN_PROGRESS
- `stage.completed` — append `deploy_stage` row with conclusion
- `input.approved` / `input.rejected` / `input.timedOut` — append `approval_event`
- `job.completed` — transition `deploy` row to SUCCEEDED / FAILED / UNSTABLE→AMBER / CANCELLED
- `build.promoted` — optional future signal; currently logged only
- `build.deleted` — soft-hide the deploy row (logical delete)

### 5.2 ReleaseVersionResolver

```java
public class ReleaseVersionResolver {

    private final ReleaseRepository releaseRepo;
    private final CodeBuildFacade codeBuild;

    /**
     * On job.started we receive (applicationId, releaseVersion, buildArtifactId?).
     * Resolve-or-create Release row keyed on (applicationId, releaseVersion).
     */
    public ReleaseEntity resolveOrCreate(String applicationId,
                                         String releaseVersion,
                                         String buildArtifactIdOrNull) {
        return releaseRepo.findByApplicationAndVersion(applicationId, releaseVersion)
            .orElseGet(() -> {
                var buildArtifactId = buildArtifactIdOrNull != null
                    ? buildArtifactIdOrNull
                    : codeBuild.findArtifactByJenkinsBuild(applicationId, releaseVersion)
                        .orElse(null);
                return releaseRepo.save(new ReleaseEntity(
                    applicationId, releaseVersion,
                    ReleaseState.PREPARED,
                    "cb", buildArtifactId,
                    Instant.now()));
            });
    }
}
```

### 5.3 RollbackDetector

Dual-signal: explicit `trigger=ROLLBACK` OR deploying a release version older than the last successful deploy on the same environment.

```java
public class RollbackDetector {

    public boolean isRollback(DeployEntity candidate, Optional<DeployEntity> priorSuccessSameEnv) {
        if (candidate.getTrigger() == DeployTrigger.ROLLBACK) return true;
        return priorSuccessSameEnv
            .map(p -> ReleaseVersionComparator.compare(
                    candidate.getReleaseVersion(), p.getReleaseVersion()) < 0)
            .orElse(false);
    }
}
```

The signal that fired is persisted as `rollback_detection_signal` (`trigger=rollback` | `version-older-than-prior`) on the deploy row for audit.

### 5.4 ReleaseNotesEvidenceValidator

```java
public AiReleaseNotesDto persistOrReject(ReleaseEntity release, AiReleaseNotesDraft draft, String skillVersion) {
    var actualCommits = codeBuild.listCommitsForRelease(release.getId()).stream()
        .map(CommitRef::shortSha).collect(Collectors.toSet());
    var storyIds = codeBuild.listStoryIdsForRelease(release.getId());

    boolean commitsMatch = draft.evidence().stream()
        .filter(e -> "commit".equals(e.kind()))
        .allMatch(e -> actualCommits.contains(e.id()));
    boolean storiesMatch = draft.evidence().stream()
        .filter(e -> "story".equals(e.kind()))
        .allMatch(e -> storyIds.contains(e.id()));

    if (!commitsMatch || !storiesMatch) {
        return repo.persistMismatch(release.getId(), skillVersion);  // status=EVIDENCE_MISMATCH
    }
    return repo.persistSuccess(release.getId(), skillVersion, draft);
}
```

### 5.5 Rate limiting and idempotency

- `POST /catalog/ai-summary/regenerate` uses a per-workspace token bucket (1/min, burst 1).
- `POST /releases/{id}/ai-notes/regenerate` debounces 60s server-side keyed by `releaseId`.
- `POST /deploys/{id}/ai-summary/regenerate` debounces 30s server-side keyed by `deployId`.
- Webhook idempotency is enforced by the DB unique constraint `uq_deploy_jenkins UNIQUE (jenkins_instance_id, jenkins_job_full_name, jenkins_build_number)` on the `deploy` table. Outbox insert on duplicate `deliveryId` is a no-op.

## 6. Frontend API Client

`frontend/src/features/deployment-management/api/deploymentApi.ts`

```ts
import { fetchJson } from '@/shared/api/client';
import type { ApiResponse } from '@/shared/api/types';
import type {
  CatalogAggregate, ApplicationDetailAggregate, ReleaseDetailAggregate,
  DeployDetailAggregate, EnvironmentDetailAggregate, TraceabilityAggregate,
  AiWorkspaceSummary, AiReleaseNotes, AiDeploymentSummary, CatalogFilters
} from '../types/aggregate';

const BASE = '/api/v1/deployment-management';

export const deploymentApi = {
  // Catalog
  getCatalog: (f: CatalogFilters) =>
    fetchJson<ApiResponse<CatalogAggregate>>(`${BASE}/catalog?${qs(f)}`),
  regenerateWorkspaceSummary: (workspaceId: string) =>
    fetchJson<ApiResponse<AiWorkspaceSummary>>(
      `${BASE}/catalog/ai-summary/regenerate?workspaceId=${workspaceId}`,
      { method: 'POST' }),

  // Application
  getApplication: (applicationId: string) =>
    fetchJson<ApiResponse<ApplicationDetailAggregate>>(`${BASE}/applications/${applicationId}`),

  // Release
  getRelease: (releaseId: string) =>
    fetchJson<ApiResponse<ReleaseDetailAggregate>>(`${BASE}/releases/${releaseId}`),
  regenerateReleaseNotes: (releaseId: string) =>
    fetchJson<ApiResponse<AiReleaseNotes>>(
      `${BASE}/releases/${releaseId}/ai-notes/regenerate`, { method: 'POST' }),

  // Deploy
  getDeploy: (deployId: string) =>
    fetchJson<ApiResponse<DeployDetailAggregate>>(`${BASE}/deploys/${deployId}`),
  regenerateDeploySummary: (deployId: string) =>
    fetchJson<ApiResponse<AiDeploymentSummary>>(
      `${BASE}/deploys/${deployId}/ai-summary/regenerate`, { method: 'POST' }),

  // Environment
  getEnvironment: (applicationId: string, name: string) =>
    fetchJson<ApiResponse<EnvironmentDetailAggregate>>(
      `${BASE}/applications/${applicationId}/environments/${encodeURIComponent(name)}`),

  // Traceability
  getTraceability: (storyId: string) =>
    fetchJson<ApiResponse<TraceabilityAggregate>>(
      `${BASE}/traceability?storyId=${encodeURIComponent(storyId)}`),
};

function qs(f: CatalogFilters): string { /* URLSearchParams builder */ return '' }
```

### 6.1 Vite dev-server proxy config

```ts
// vite.config.ts
export default defineConfig({
  server: {
    proxy: {
      '/api/v1/deployment-management': {
        target: 'http://localhost:8080',
        changeOrigin: true,
      },
    },
  },
});
```

## 7. Testing Contracts

### Golden-file API tests

For each endpoint, a JSON fixture lives under `backend/src/test/resources/dpm/fixtures/{endpoint}-200.json`. Integration tests compare the serialized response to the fixture. Fixtures are updated intentionally when the contract changes and the PR description calls it out. At minimum:

- `catalog-200.json`
- `application-detail-200.json`
- `release-detail-200.json`
- `deploy-detail-rollback-200.json`
- `deploy-detail-in-progress-200.json`
- `environment-detail-200.json`
- `traceability-200.json`
- Error fixtures per code in §4.

### Webhook signature fixtures

`backend/src/test/resources/dpm/webhooks/` carries realistic Jenkins Notification Plugin payloads for each event type, plus a pre-computed valid HMAC signature for a known test secret. Tests confirm 202 on valid, 401 on mutation, 404 on unknown instance, 400 on malformed payload, and idempotent no-op on duplicate delivery id.

### AI stubs

The AI skill client is a Spring `@Component` injected via interface; a `FakeAiSkillClient` returns deterministic responses. Tests cover: SUCCESS, FAILED (transient), EVIDENCE_MISMATCH (release notes: fabricated commit sha), skip (no meaningful change on deploy summary).

### Contract drift guard

A lightweight test compares the published TypeScript aggregate types (generated from Java DTOs via a serialization dump) to the frontend's `types/aggregate.ts`. Drift breaks the build.

## 8. Versioning Policy

- **API version:** path-embedded (`/v1/`). Breaking changes require `/v2/` plus a deprecation window for `/v1/`.
- **Skill versions:** `skillVersion` string is persisted alongside AI output rows; rotating the skill invalidates prior cache naturally (unique index on `(…, skill_version)`).
- **Schema versions:** Flyway monotonic — V60..V67 for V1; V68+ reserved for V1.1.
- **Additive-first:** new optional fields are fine; removing or renaming fields triggers a new major version.
- **Cross-slice facades:**
  - `/internal/story/{storyId}/deploy-aggregate` — stable contract; change requires coordinated update with Requirement slice maintainer.
  - `/internal/release/{releaseId}/commit-slice` — stable contract; change requires coordinated update with Code & Build slice maintainer.
- **Event contracts:** The Jenkins event types listed in §5.1 are treated as a versioned contract. Adding a new event type is additive; removing or renaming requires a migration note in the backfill logic.
