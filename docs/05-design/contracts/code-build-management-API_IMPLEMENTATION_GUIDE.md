# Code & Build Management — API Implementation Guide

## Source

- Spec: [../../03-spec/code-build-management-spec.md](../../03-spec/code-build-management-spec.md)
- Architecture: [../../04-architecture/code-build-management-architecture.md](../../04-architecture/code-build-management-architecture.md)
- Data flow: [../../04-architecture/code-build-management-data-flow.md](../../04-architecture/code-build-management-data-flow.md)
- Data model: [../../04-architecture/code-build-management-data-model.md](../../04-architecture/code-build-management-data-model.md)
- Design: [../code-build-management-design.md](../code-build-management-design.md)

## 1. Conventions

- **Base path:** `/api/v1/code-build-management`
- **Envelope:** every response wrapped in `ApiResponse<T>`:
  ```json
  { "data": { /* T */ }, "meta": { "correlationId": "cbm-…", "generatedAt": "2026-04-17T10:00:00Z" }, "error": null }
  ```
- **Section results:** aggregate endpoints return `SectionResult<T>` per card: `{ "data": T | null, "error": { "code", "message" } | null, "loadedAt": "…" }`
- **Timestamps:** ISO-8601 UTC with trailing `Z`.
- **Pagination:** cursor-based (`?cursor=…&limit=…`, default 20, max 100).
- **Correlation ID:** client-supplied `x-correlation-id` header is honored; server generates if absent. Always echoed in `meta.correlationId`.
- **Webhook base path:** `/api/v1/code-build-management/webhooks/github` (separate path prefix, skipped by the standard auth filter because GitHub authenticates via signature).

## 2. Endpoint Overview

### 2.1 Catalog

| Method | Path | Purpose |
| ------ | ---- | ------- |
| GET | `/catalog` | Aggregate (Summary + Grid + AiSummary) |
| GET | `/catalog/summary` | Summary card only |
| GET | `/catalog/grid` | Grid card only |
| GET | `/catalog/ai-summary` | AI Summary card |
| POST | `/catalog/ai-summary/regenerate` | Admin-only manual regen (rate-limited 1/min) |

Query params for `/catalog*`: `workspaceIds`, `projectId`, `buildStatus`, `timeWindow` (`24h`/`7d`/`30d`), `search`.

### 2.2 Repo Detail

| Method | Path | Purpose |
| ------ | ---- | ------- |
| GET | `/repos/{repoId}` | 6-card aggregate |
| GET | `/repos/{repoId}/header` | — |
| GET | `/repos/{repoId}/branches` | — |
| GET | `/repos/{repoId}/open-prs` | — |
| GET | `/repos/{repoId}/recent-commits` | — |
| GET | `/repos/{repoId}/recent-runs` | — |
| GET | `/repos/{repoId}/ai-insights` | — |

### 2.3 PR Detail

| Method | Path | Purpose |
| ------ | ---- | ------- |
| GET | `/repos/{repoId}/prs/{prNumber}` | 4-card aggregate |
| GET | `/repos/{repoId}/prs/{prNumber}/header` | — |
| GET | `/repos/{repoId}/prs/{prNumber}/linked-stories` | — |
| GET | `/repos/{repoId}/prs/{prNumber}/ci-status` | — |
| GET | `/repos/{repoId}/prs/{prNumber}/ai-review` | — |
| POST | `/repos/{repoId}/prs/{prNumber}/ai-review/rerun` | Admin-only; debounced 20s server-side |

### 2.4 Run Detail

| Method | Path | Purpose |
| ------ | ---- | ------- |
| GET | `/runs/{runId}` | 4-card aggregate |
| GET | `/runs/{runId}/header` | — |
| GET | `/runs/{runId}/timeline` | — |
| GET | `/runs/{runId}/artifacts` | — |
| GET | `/runs/{runId}/ai-triage` | — |
| POST | `/runs/{runId}/ai-triage/rerun` | Admin-only |

### 2.5 Traceability

| Method | Path | Purpose |
| ------ | ---- | ------- |
| GET | `/traceability?storyId=…` | Inverse lookup aggregate |
| GET | `/traceability/commits?storyId=…` | — |
| GET | `/traceability/prs?storyId=…` | — |
| GET | `/traceability/runs?storyId=…` | — |

### 2.6 Webhooks

| Method | Path | Purpose |
| ------ | ---- | ------- |
| POST | `/webhooks/github` | GitHub App event receiver |

### 2.7 Cross-slice facade (for Requirement slice's Story Detail Code tab)

| Method | Path | Purpose |
| ------ | ---- | ------- |
| GET | `/internal/story/{storyId}/aggregate` | Same as `/traceability` but returns condensed cross-slice view |

Internal endpoints require `X-Service-Token` header; never exposed to browser.

## 3. Example Payloads

### 3.1 `GET /catalog` (200)

```json
{
  "data": {
    "filtersEcho": { "workspaceIds": ["ws-001"], "timeWindow": "7d" },
    "summary": {
      "data": {
        "visibleRepos": 18,
        "openPrs": 27,
        "runsLast7d": 412,
        "successRate7d": 0.87,
        "meanBuildDurationSec": 412,
        "byLed": { "GREEN": 12, "AMBER": 3, "RED": 2, "UNKNOWN": 1 }
      },
      "error": null,
      "loadedAt": "2026-04-17T10:00:01Z"
    },
    "grid": {
      "data": [
        {
          "projectId": "proj-billing",
          "projectName": "Billing Platform",
          "repos": [
            {
              "repoId": "repo-5521",
              "fullName": "acme/billing-api",
              "projectId": "proj-billing",
              "workspaceId": "ws-001",
              "defaultBranch": "main",
              "lastActivityAt": "2026-04-17T09:42:11Z",
              "openPrCount": 4,
              "latestBuildLed": "RED",
              "description": "Billing REST API"
            }
          ]
        }
      ],
      "error": null,
      "loadedAt": "2026-04-17T10:00:01Z"
    },
    "aiSummary": {
      "data": {
        "status": "SUCCESS",
        "generatedAt": "2026-04-17T09:55:00Z",
        "skillVersion": "cb-summary-v3",
        "narrative": "In the last 24h the billing workspace merged 12 PRs (7 stories) but the `billing-api` main build has failed three times …",
        "evidence": [
          { "kind": "pr", "id": "pr-8821", "label": "acme/billing-api#421" },
          { "kind": "run", "id": "run-99012", "label": "billing-api main #5521" }
        ]
      },
      "error": null,
      "loadedAt": "2026-04-17T10:00:01Z"
    }
  },
  "meta": { "correlationId": "cbm-abc123", "generatedAt": "2026-04-17T10:00:01Z" },
  "error": null
}
```

### 3.2 `GET /repos/{repoId}` (200)

```json
{
  "data": {
    "header": { "data": {
      "repoId": "repo-5521", "fullName": "acme/billing-api",
      "projectId": "proj-billing", "workspaceId": "ws-001",
      "defaultBranch": "main", "primaryLanguage": "Java",
      "lastPushAt": "2026-04-17T09:42:11Z",
      "externalUrl": "https://github.com/acme/billing-api",
      "description": "Billing REST API"
    }, "error": null, "loadedAt": "2026-04-17T10:00:02Z" },
    "branches": { "data": [
      { "name": "main", "aheadOfDefault": 0, "behindDefault": 0,
        "lastCommitSha": "a1b2c3d4e5f6a1b2c3d4e5f6a1b2c3d4e5f6a1b2",
        "lastCommitMessage": "feat: add partial refunds\n\nStory-Id: STORY-4211",
        "lastCommitAuthor": "jane@acme.example",
        "lastCommitAt": "2026-04-17T09:42:11Z", "openPrNumber": null }
    ], "error": null, "loadedAt": "…" },
    "openPrs": { "data": [
      { "prId": "pr-8821", "prNumber": 421, "title": "feat: adjust tax rounding",
        "author": "kim@acme.example", "sourceBranch": "feat/tax-rounding",
        "targetBranch": "main", "state": "OPEN",
        "ciLed": "AMBER", "aiReviewNoteCount": 5,
        "lastUpdatedAt": "2026-04-17T09:30:00Z" }
    ], "error": null, "loadedAt": "…" },
    "recentCommits": { "data": [
      { "sha": "a1b2c3d4e5f6a1b2c3d4e5f6a1b2c3d4e5f6a1b2",
        "shortSha": "a1b2c3d4",
        "author": "jane@acme.example",
        "message": "feat: add partial refunds\n\nStory-Id: STORY-4211",
        "committedAt": "2026-04-17T09:42:11Z",
        "storyChips": [ { "storyId": "STORY-4211", "status": "VERIFIED",
          "title": "Partial refund workflow", "projectId": "proj-billing" } ] }
    ], "error": null, "loadedAt": "…" },
    "recentRuns": { "data": [
      { "runId": "run-99012", "pipelineName": "main-build",
        "trigger": "PUSH", "branch": "main", "status": "COMPLETED_FAILURE",
        "durationSec": 412, "headSha": "a1b2c3d4e5f6a1b2c3d4e5f6a1b2c3d4e5f6a1b2" }
    ], "error": null, "loadedAt": "…" },
    "aiInsights": { "data": {
      "status": "SUCCESS",
      "generatedAt": "2026-04-17T09:55:00Z",
      "narrative": "Main build failed 3× in the last 24h; most recent failure in step `integration-tests.billing.TaxServiceTest` — candidate owner: @jane",
      "evidence": [ { "kind": "run", "id": "run-99012", "label": "main-build #5521" } ],
      "skillVersion": "cb-repo-insights-v2"
    }, "error": null, "loadedAt": "…" }
  },
  "meta": { "correlationId": "cbm-def456", "generatedAt": "2026-04-17T10:00:02Z" },
  "error": null
}
```

### 3.3 `GET /repos/{repoId}/prs/{prNumber}` — AI PR Review sample

```json
{
  "data": {
    "header": { "data": {
      "prId": "pr-8821", "prNumber": 421, "repoId": "repo-5521",
      "title": "feat: adjust tax rounding",
      "author": "kim@acme.example",
      "sourceBranch": "feat/tax-rounding", "targetBranch": "main",
      "state": "OPEN", "reviewersRequested": ["jane@acme.example"],
      "labels": ["billing","tax"],
      "createdAt": "2026-04-15T13:00:00Z",
      "updatedAt": "2026-04-17T09:30:00Z",
      "headSha": "b2c3d4e5f6b2c3d4e5f6b2c3d4e5f6b2c3d4e5f6",
      "isBotAuthored": false,
      "externalUrl": "https://github.com/acme/billing-api/pull/421"
    }, "error": null, "loadedAt": "…" },
    "linkedStories": { "data": [
      { "storyId": "STORY-4099", "status": "VERIFIED", "title": "Tax rounding cleanup", "projectId": "proj-billing" }
    ], "error": null, "loadedAt": "…" },
    "ciStatus": { "data": {
      "aggregateLed": "AMBER",
      "runs": [
        { "runId": "run-99020", "pipelineName": "pr-checks", "trigger": "PULL_REQUEST",
          "branch": "feat/tax-rounding", "status": "COMPLETED_SUCCESS",
          "durationSec": 204, "headSha": "b2c3d4e5f6b2c3d4e5f6b2c3d4e5f6b2c3d4e5f6" },
        { "runId": "run-99021", "pipelineName": "integration", "trigger": "PULL_REQUEST",
          "branch": "feat/tax-rounding", "status": "COMPLETED_FAILURE",
          "durationSec": 612, "headSha": "b2c3d4e5f6b2c3d4e5f6b2c3d4e5f6b2c3d4e5f6" }
      ]
    }, "error": null, "loadedAt": "…" },
    "aiReview": { "data": {
      "status": "SUCCESS",
      "keyedOnSha": "b2c3d4e5f6b2c3d4e5f6b2c3d4e5f6b2c3d4e5f6",
      "skillVersion": "cb-pr-review-v1",
      "generatedAt": "2026-04-17T09:32:00Z",
      "notesBySeverity": {
        "BLOCKER": [ { "id": "note-1", "severity": "BLOCKER",
          "filePath": "src/main/java/com/acme/billing/TaxService.java",
          "startLine": 142, "endLine": 155,
          "message": "Rounding uses HALF_UP but spec requires HALF_EVEN; will over-charge by $0.01 on boundary cases.",
          "evidenceUrl": "https://github.com/acme/billing-api/pull/421/files#diff-…L142-L155" } ],
        "MAJOR": [],
        "MINOR": [ { "id": "note-2", "severity": "MINOR", "filePath": "src/test/…",
          "message": "Consider property-based test for boundary cases." } ],
        "NIT": []
      }
    }, "error": null, "loadedAt": "…" }
  },
  "meta": { "correlationId": "cbm-ghi789", "generatedAt": "2026-04-17T10:00:03Z" },
  "error": null
}
```

For a non-Lead principal, the `BLOCKER` branch is returned as `{ "hiddenCount": 1 }` instead of the array.

### 3.4 `GET /runs/{runId}` — FAILED run with triage

```json
{
  "data": {
    "header": { "data": {
      "runId": "run-99012", "runNumber": 5521,
      "pipelineName": "main-build",
      "repoId": "repo-5521",
      "trigger": "PUSH",
      "branch": "main",
      "actor": "jane@acme.example",
      "headSha": "a1b2c3d4e5f6a1b2c3d4e5f6a1b2c3d4e5f6a1b2",
      "status": "COMPLETED_FAILURE",
      "durationSec": 412,
      "startedAt": "2026-04-17T09:35:00Z",
      "completedAt": "2026-04-17T09:41:52Z",
      "externalUrl": "https://github.com/acme/billing-api/actions/runs/99012"
    }, "error": null, "loadedAt": "…" },
    "timeline": { "data": [
      { "jobId": "job-1", "name": "build", "status": "COMPLETED_SUCCESS", "conclusion": "SUCCESS",
        "steps": [] },
      { "jobId": "job-2", "name": "integration-tests", "status": "COMPLETED_FAILURE", "conclusion": "FAILURE",
        "steps": [
          { "stepId": "step-2-3", "name": "TaxServiceTest", "order": 3, "conclusion": "FAILURE",
            "startedAt": "2026-04-17T09:39:00Z", "completedAt": "2026-04-17T09:41:40Z",
            "logExcerpt": { "redacted": true,
              "text": "[INFO] Running com.acme.billing.TaxServiceTest\n[ERROR] boundary_rounding FAILED ...",
              "bytes": 4012 } }
        ] }
    ], "error": null, "loadedAt": "…" },
    "artifacts": { "data": [], "error": null, "loadedAt": "…" },
    "triage": { "data": {
      "status": "SUCCESS",
      "keyedOnRunId": "run-99012",
      "skillVersion": "cb-triage-v2",
      "generatedAt": "2026-04-17T09:45:00Z",
      "likelyCause": "TaxServiceTest.boundary_rounding fails on HALF_UP vs HALF_EVEN inconsistency introduced by commit a1b2c3d4.",
      "failingStepRef": { "jobId": "job-2", "stepId": "step-2-3" },
      "candidateOwners": [
        { "memberId": "mem-jane", "displayName": "Jane Kim", "rationale": "Authored commit a1b2c3d4 on `TaxService.java`" }
      ],
      "confidence": "HIGH",
      "evidence": [
        { "kind": "log", "id": "step-2-3", "excerpt": "boundary_rounding FAILED", "bytes": 412 }
      ]
    }, "error": null, "loadedAt": "…" },
    "openIncidentContext": {
      "commitSha": "a1b2c3d4e5f6a1b2c3d4e5f6a1b2c3d4e5f6a1b2",
      "runUrl": "https://github.com/acme/billing-api/actions/runs/99012",
      "pipelineName": "main-build",
      "triageSummary": "TaxServiceTest.boundary_rounding fails on HALF_UP vs HALF_EVEN inconsistency."
    }
  },
  "meta": { "correlationId": "cbm-jkl012", "generatedAt": "2026-04-17T10:00:04Z" },
  "error": null
}
```

### 3.5 `GET /traceability?storyId=STORY-4211`

```json
{
  "data": {
    "storyChip": { "storyId": "STORY-4211", "status": "VERIFIED", "title": "Partial refund workflow", "projectId": "proj-billing" },
    "commits": { "data": [ /* RecentCommitRowDto[] */ ], "error": null, "loadedAt": "…" },
    "prs": { "data": [ /* OpenPrRowDto[] */ ], "error": null, "loadedAt": "…" },
    "runs": { "data": [ /* RecentRunRowDto[] */ ], "error": null, "loadedAt": "…" },
    "capNotice": null
  },
  "meta": { "correlationId": "cbm-mno345", "generatedAt": "…" },
  "error": null
}
```

### 3.6 Error envelope

```json
{
  "data": null,
  "meta": { "correlationId": "cbm-err001", "generatedAt": "…" },
  "error": { "code": "CB_WORKSPACE_FORBIDDEN", "message": "Workspace not accessible to caller" }
}
```

## 4. Error Code Catalog

| Code | HTTP | Retryable | Notes |
| ---- | ---- | --------- | ----- |
| `CB_WORKSPACE_FORBIDDEN` | 403 | No | Caller not a workspace member |
| `CB_ROLE_REQUIRED` | 403 | No | Admin-only action attempted |
| `CB_REPO_NOT_FOUND` | 404 | No | `repoId` unknown or out of scope |
| `CB_PR_NOT_FOUND` | 404 | No | |
| `CB_RUN_NOT_FOUND` | 404 | No | |
| `CB_STORY_NOT_FOUND` | 404 | No | Story not visible or unknown |
| `CB_AI_AUTONOMY_INSUFFICIENT` | 409 | No | Workspace autonomy below floor |
| `CB_AI_UNAVAILABLE` | 503 | Yes | AI skill client failure |
| `CB_RATE_LIMITED` | 429 | Yes | Includes `Retry-After` header |
| `CB_GH_RATE_LIMIT` | 503 | Yes | GitHub primary/secondary rate limit |
| `CB_INGEST_SIGNATURE_INVALID` | 401 | No | Webhook signature failed |
| `CB_INGEST_PAYLOAD_INVALID` | 400 | No | Webhook payload parse failed |
| `CB_INSTALL_UNKNOWN` | 404 | No | Unknown installation id on webhook |
| `CB_TRIAGE_EVIDENCE_MISMATCH` | 422 | No | Triage output rejected at service time |
| `CB_RANGE_TOO_LARGE` | 422 | No | Commit range exceeds 100-commit cap |

## 5. Backend Controller Skeleton (Spring Boot)

```java
@RestController
@RequestMapping("/api/v1/code-build-management")
@Validated
public class CodeBuildController {

    private final CatalogService catalogService;
    private final RepoDetailService repoDetailService;
    private final PrDetailService prDetailService;
    private final RunDetailService runDetailService;
    private final TraceabilityService traceabilityService;
    private final AiSummaryService aiSummaryService;
    private final AiPrReviewService aiPrReviewService;
    private final AiTriageService aiTriageService;
    private final CodeBuildAccessGuard accessGuard;

    // ---- Catalog ----
    @GetMapping("/catalog")
    public ApiResponse<CatalogAggregateDto> getCatalog(
            @RequestParam(required = false) List<String> workspaceIds,
            @RequestParam(required = false) String projectId,
            @RequestParam(required = false) String buildStatus,
            @RequestParam(required = false, defaultValue = "7d") String timeWindow,
            @RequestParam(required = false) String search,
            Principal principal) {
        var filters = CatalogFilters.of(workspaceIds, projectId, buildStatus, timeWindow, search);
        return ApiResponse.ok(catalogService.loadAggregate(filters, principal));
    }

    @PostMapping("/catalog/ai-summary/regenerate")
    public ApiResponse<AiWorkspaceSummaryDto> regenerateSummary(
            @RequestParam String workspaceId, Principal principal) {
        accessGuard.requireAdmin(workspaceId, principal);
        return ApiResponse.ok(aiSummaryService.regenerate(workspaceId, principal));
    }

    // ---- Repo Detail ----
    @GetMapping("/repos/{repoId}")
    public ApiResponse<RepoDetailAggregateDto> getRepo(
            @PathVariable @Pattern(regexp = "^repo-[a-z0-9\\-]+$") String repoId,
            Principal principal) {
        accessGuard.requireRead(repoId, principal);
        return ApiResponse.ok(repoDetailService.loadAggregate(repoId, principal));
    }

    // ---- PR Detail ----
    @GetMapping("/repos/{repoId}/prs/{prNumber}")
    public ApiResponse<PrDetailAggregateDto> getPr(
            @PathVariable @Pattern(regexp = "^repo-[a-z0-9\\-]+$") String repoId,
            @PathVariable @Min(1) int prNumber,
            Principal principal) {
        accessGuard.requireRead(repoId, principal);
        return ApiResponse.ok(prDetailService.loadAggregate(repoId, prNumber, principal));
    }

    @PostMapping("/repos/{repoId}/prs/{prNumber}/ai-review/rerun")
    public ApiResponse<AiPrReviewDto> rerunReview(
            @PathVariable String repoId, @PathVariable int prNumber, Principal principal) {
        accessGuard.requireAdminOnRepo(repoId, principal);
        return ApiResponse.ok(aiPrReviewService.enqueueRerun(repoId, prNumber, principal));
    }

    // ---- Run Detail ----
    @GetMapping("/runs/{runId}")
    public ApiResponse<RunDetailAggregateDto> getRun(
            @PathVariable @Pattern(regexp = "^run-[a-z0-9\\-]+$") String runId,
            Principal principal) {
        accessGuard.requireReadByRun(runId, principal);
        return ApiResponse.ok(runDetailService.loadAggregate(runId, principal));
    }

    @PostMapping("/runs/{runId}/ai-triage/rerun")
    public ApiResponse<AiBuildTriageDto> rerunTriage(@PathVariable String runId, Principal principal) {
        accessGuard.requireAdminOnRun(runId, principal);
        return ApiResponse.ok(aiTriageService.enqueueRerun(runId, principal));
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
@RequestMapping("/api/v1/code-build-management/webhooks")
public class GithubWebhookController {

    private final WebhookSignatureVerifier verifier;
    private final WebhookPayloadParser parser;
    private final IngestionOutboxWriter outboxWriter;
    private final AuditLogEmitter audit;

    @PostMapping(path = "/github", consumes = "application/json")
    public ResponseEntity<Void> receive(
            @RequestHeader("X-GitHub-Event") String event,
            @RequestHeader(value = "X-GitHub-Delivery", required = false) String deliveryId,
            @RequestHeader("X-Hub-Signature-256") String signature,
            @RequestBody byte[] body) {

        if (!verifier.verify(signature, body)) {
            audit.emit("CB_INGEST_SIGNATURE_INVALID", Map.of("deliveryId", String.valueOf(deliveryId)));
            return ResponseEntity.status(HttpStatus.UNAUTHORIZED).build();
        }

        TypedEvent typed;
        try {
            typed = parser.parse(event, body);
        } catch (InvalidPayloadException e) {
            audit.emit("CB_INGEST_PAYLOAD_INVALID", Map.of("event", event, "deliveryId", String.valueOf(deliveryId)));
            return ResponseEntity.badRequest().build();
        }

        outboxWriter.enqueue(deliveryId, typed, body);
        return ResponseEntity.accepted().build();
    }
}
```

### 5.2 StoryIdExtractor

```java
public class StoryIdExtractor {
    private static final Pattern TRAILER = Pattern.compile(
        "^\\s*(?:Story-Id|Relates-to)\\s*:\\s*(STORY-[A-Z0-9\\-]+)\\s*$",
        Pattern.MULTILINE);

    public List<String> extract(String text) {
        if (text == null || text.isBlank()) return List.of();
        var matcher = TRAILER.matcher(text);
        var out = new LinkedHashSet<String>();
        while (matcher.find()) out.add(matcher.group(1));
        return List.copyOf(out);
    }
}
```

### 5.3 LogRedactor

```java
public class LogRedactor {
    private static final List<Pattern> PATTERNS = List.of(
        Pattern.compile("AKIA[0-9A-Z]{16}"),
        Pattern.compile("ghp_[A-Za-z0-9]{36,}"),
        Pattern.compile("gho_[A-Za-z0-9]{36,}"),
        Pattern.compile("ghs_[A-Za-z0-9]{36,}"),
        Pattern.compile("(?i)bearer\\s+[A-Za-z0-9._\\-]+"),
        Pattern.compile("[A-Za-z0-9+/=]{40,}")  // generic long tokens — high false-positive, only in suspicious paths
    );

    public String redact(String raw) {
        String out = raw;
        for (var p : PATTERNS.subList(0, 5)) {
            out = p.matcher(out).replaceAll("REDACTED");
        }
        return out;
    }
}
```

### 5.4 AiTriageService evidence integrity check

```java
public AiBuildTriageDto persistOrReject(PipelineRun run, AiTriageDraft draft, String skillVersion) {
    var refJob = run.jobs().stream().filter(j -> j.id().equals(draft.failingStepRef().jobId())).findFirst();
    if (refJob.isEmpty() || refJob.get().steps().stream().noneMatch(s -> s.id().equals(draft.failingStepRef().stepId()))) {
        return repo.persistMismatch(run.id(), skillVersion);       // status=EVIDENCE_MISMATCH
    }
    return repo.persistSuccess(run.id(), skillVersion, draft);
}
```

### 5.5 Rate limiting and idempotency

- `POST /catalog/ai-summary/regenerate` uses a per-workspace token bucket (1/min burst 1).
- Webhook delivery id is unique; outbox insert on duplicate delivery id is a no-op (idempotency).
- `POST /ai-review/rerun` and `POST /ai-triage/rerun` debounce 20s server-side keyed by (prId, headSha) and (runId) respectively.

## 6. Frontend API Client

`frontend/src/features/code-build-management/api/codeBuildApi.ts`

```ts
import { fetchJson } from '@/shared/api/client';
import type { ApiResponse } from '@/shared/api/types';
import type {
  CatalogAggregate, RepoDetailAggregate, PrDetailAggregate, RunDetailAggregate,
  TraceabilityAggregate, AiWorkspaceSummary, AiPrReview, AiBuildTriage, CatalogFilters
} from '../types/aggregate';

const BASE = '/api/v1/code-build-management';

export const codeBuildApi = {
  // Catalog
  getCatalog: (f: CatalogFilters) =>
    fetchJson<ApiResponse<CatalogAggregate>>(`${BASE}/catalog?${qs(f)}`),
  regenerateWorkspaceSummary: (workspaceId: string) =>
    fetchJson<ApiResponse<AiWorkspaceSummary>>(
      `${BASE}/catalog/ai-summary/regenerate?workspaceId=${workspaceId}`,
      { method: 'POST' }),

  // Repo
  getRepo: (repoId: string) =>
    fetchJson<ApiResponse<RepoDetailAggregate>>(`${BASE}/repos/${repoId}`),

  // PR
  getPr: (repoId: string, prNumber: number) =>
    fetchJson<ApiResponse<PrDetailAggregate>>(`${BASE}/repos/${repoId}/prs/${prNumber}`),
  rerunReview: (repoId: string, prNumber: number) =>
    fetchJson<ApiResponse<AiPrReview>>(
      `${BASE}/repos/${repoId}/prs/${prNumber}/ai-review/rerun`, { method: 'POST' }),

  // Run
  getRun: (runId: string) =>
    fetchJson<ApiResponse<RunDetailAggregate>>(`${BASE}/runs/${runId}`),
  rerunTriage: (runId: string) =>
    fetchJson<ApiResponse<AiBuildTriage>>(`${BASE}/runs/${runId}/ai-triage/rerun`, { method: 'POST' }),

  // Traceability
  getTraceability: (storyId: string) =>
    fetchJson<ApiResponse<TraceabilityAggregate>>(`${BASE}/traceability?storyId=${encodeURIComponent(storyId)}`),
};

function qs(f: CatalogFilters): string { /* URLSearchParams builder */ return '' }
```

## 7. Testing Contracts

### Golden-file API tests

For each endpoint, a JSON fixture lives under `backend/src/test/resources/cbm/fixtures/{endpoint}-200.json`. Integration tests compare the serialized response to the fixture. Fixtures are updated intentionally when the contract changes and the PR description calls it out.

### Webhook signature fixture

`backend/src/test/resources/cbm/webhooks/push.json` carries a realistic GitHub `push` payload plus a pre-computed valid signature for a known test secret. Tests confirm 202 on valid, 401 on mutation, and idempotent no-op on duplicate delivery.

### AI stubs

The AI skill client is a Spring `@Component` injected via interface; a `FakeAiSkillClient` returns deterministic responses. Tests cover: SUCCESS, FAILED (transient), EVIDENCE_MISMATCH (triage), bot-author skip (PR review).

## 8. Versioning Policy

- **API version:** path-embedded (`/v1/`). Breaking changes require `/v2/` plus a deprecation window for `/v1/`.
- **Skill versions:** `skillVersion` string is persisted alongside AI output rows; rotating the skill invalidates prior cache naturally (unique index on `(…, skill_version)`).
- **Schema versions:** Flyway monotonic — V40..V47 for V1; V48+ reserved for V1.1.
- **Additive-first:** new optional fields are fine; removing or renaming fields triggers a new major version.
- **Cross-slice facade (`/internal/story/{storyId}/aggregate`):** stable contract, change requires coordinated update with Requirement slice maintainer.
