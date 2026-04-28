# Code & Build Management — Data Flow

Runtime sequences, state machines, error cascade, and refresh strategy for the Code & Build Management slice. Operationalizes [code-build-management-architecture.md](code-build-management-architecture.md) and the contracts in [../05-design/contracts/code-build-management-API_IMPLEMENTATION_GUIDE.md](../05-design/contracts/code-build-management-API_IMPLEMENTATION_GUIDE.md).

## 1. Catalog Page Load (Phase A — mocks)

```mermaid
sequenceDiagram
  autonumber
  participant User
  participant Router
  participant View as CatalogView
  participant Store as codeBuildStore
  participant Mock as codeBuildMock

  User->>Router: /code-build
  Router->>View: mount
  View->>Store: initCatalog(filters)
  Store->>Mock: getCatalogAggregate(filters)
  Mock-->>Store: aggregate with 3 SectionResult cards
  Store-->>View: state.catalog.* hydrated
  View-->>User: render CatalogSummary, Grid, AiSummary, Filters
```

## 2. Catalog Page Load (Phase B — backend)

```mermaid
sequenceDiagram
  autonumber
  participant View as CatalogView
  participant Store as codeBuildStore
  participant Api as codeBuildApi
  participant Ctrl as CodeBuildController
  participant Svc as CatalogService
  participant DB
  participant Req as Requirement Slice

  View->>Store: initCatalog(filters)
  Store->>Api: GET /catalog?filters
  Api->>Ctrl: HTTP GET
  Ctrl->>Svc: loadAggregate(filters, principal)
  par per-projection fan-out (500ms each)
    Svc->>DB: CatalogSummaryProjection
  and
    Svc->>DB: CatalogGridProjection
  and
    Svc->>DB: CatalogAiSummaryProjection
  end
  Svc->>Req: (batched) verify Story-Ids referenced in summary evidence
  Svc-->>Ctrl: CatalogAggregate{SectionResult}
  Ctrl-->>Api: 200 ApiResponse<CatalogAggregate>
  Api-->>Store: aggregate
  Store-->>View: state hydrated
```

## 3. Repo Detail Page Load

```mermaid
sequenceDiagram
  autonumber
  participant View as RepoDetailView
  participant Store
  participant Api
  participant Ctrl
  participant Svc as RepoDetailService
  participant DB
  participant Req

  View->>Store: openRepo(repoId)
  Store->>Api: GET /repos/{repoId}
  Api->>Ctrl: HTTP GET
  Ctrl->>Svc: loadAggregate(repoId, principal)
  par 6 projections
    Svc->>DB: Header
  and
    Svc->>DB: Branches
  and
    Svc->>DB: OpenPrs
  and
    Svc->>DB: RecentCommits
  and
    Svc->>DB: RecentRuns
  and
    Svc->>DB: AiInsights
  end
  Svc->>Req: batched verify for Story-Ids in commit / PR rows
  Svc-->>Ctrl: RepoDetailAggregate
  Ctrl-->>Api: 200
  Api-->>Store: aggregate
  Store-->>View: render 6 cards
```

## 4. Run Detail with AI Triage

```mermaid
sequenceDiagram
  autonumber
  participant View as RunDetailView
  participant Store
  participant Api
  participant Ctrl
  participant Svc as RunDetailService
  participant TriageSvc as AiTriageService
  participant DB
  participant AI as Skill Runtime

  View->>Store: openRun(runId)
  Store->>Api: GET /runs/{runId}
  Api->>Ctrl: HTTP GET
  Ctrl->>Svc: loadAggregate(runId, principal)
  par projections
    Svc->>DB: RunHeader
  and
    Svc->>DB: RunTimeline
  and
    Svc->>DB: RunArtifacts
  and
    Svc->>DB: AiTriage (latest current row)
  end
  alt run is FAILED and no current triage exists
    Svc->>TriageSvc: enqueueTriage(runId)
    TriageSvc->>DB: insert triage row (status=PENDING)
  end
  Svc-->>Ctrl: RunDetailAggregate
  Ctrl-->>Api: 200
  Api-->>Store: aggregate
  Store-->>View: render

  Note over TriageSvc,AI: async background
  TriageSvc->>DB: load redacted log excerpts
  TriageSvc->>AI: generateTriage(context)
  AI-->>TriageSvc: TriageDraft
  TriageSvc->>TriageSvc: evidence integrity check
  alt evidence ok
    TriageSvc->>DB: update triage row (status=SUCCESS, payload)
  else mismatch
    TriageSvc->>DB: update triage row (status=EVIDENCE_MISMATCH)
  end
  Note over View: polls /runs/{runId}/ai-triage every 3s until non-PENDING
```

## 5. PR Review Re-run on Head Commit Advance

```mermaid
sequenceDiagram
  autonumber
  participant GH as GitHub
  participant Hook as WebhookController
  participant Dispatcher
  participant DB
  participant PrSvc as AiPrReviewService
  participant AI as Skill Runtime

  GH->>Hook: push webhook (new head commit on PR branch)
  Hook->>Dispatcher: dispatch
  Dispatcher->>DB: mark existing ai_pr_review rows for that PR as STALE
  Dispatcher->>PrSvc: scheduleReviewRun(prId, headSha) debounced 20s
  Note over PrSvc: debounce window
  PrSvc->>DB: insert ai_pr_review row (status=PENDING, keyed by (prId, headSha, skillVersion))
  PrSvc->>AI: generatePrReview(diff, linkedStories)
  AI-->>PrSvc: notes[]
  PrSvc->>DB: update row to SUCCESS with notes
```

## 6. Story → Commit → Build Inverse Lookup

```mermaid
sequenceDiagram
  autonumber
  participant View as TraceabilityView
  participant Store
  participant Api
  participant Ctrl
  participant Svc as TraceabilityService
  participant DB
  participant Req

  View->>Store: lookupStory(STORY-4211)
  Store->>Api: GET /traceability?storyId=STORY-4211
  Api->>Ctrl: HTTP GET
  Ctrl->>Req: verify Story-Id visibility
  alt story not visible
    Ctrl-->>Api: 404 CB_STORY_NOT_FOUND
  else story visible
    Ctrl->>Svc: inverseLookup(STORY-4211, principal)
    par three sections
      Svc->>DB: commits with story link
    and
      Svc->>DB: PRs with story link
    and
      Svc->>DB: runs containing those commits (capped at 100)
    end
    Svc-->>Ctrl: TraceabilityAggregate
    Ctrl-->>Api: 200
  end
  Api-->>Store: aggregate
  Store-->>View: render 3 sections
```

## 7. Webhook Ingestion

```mermaid
sequenceDiagram
  autonumber
  participant GH
  participant WH as WebhookController
  participant SV as SignatureVerifier
  participant Parser
  participant Outbox as IngestionOutbox
  participant Async as AsyncDispatcher
  participant DB
  participant Red as LogRedactor
  participant Audit

  GH->>WH: POST /webhooks/github
  WH->>SV: verify(signature, body, secret)
  alt signature invalid
    SV-->>WH: false
    WH->>Audit: CB_INGEST_SIGNATURE_INVALID
    WH-->>GH: 401
  else signature valid
    SV-->>WH: true
    WH->>Parser: parse(event, body)
    Parser-->>WH: TypedEvent
    WH->>Outbox: enqueue(TypedEvent)
    WH-->>GH: 202
  end

  loop async worker
    Async->>Outbox: claim oldest
    alt push event
      Async->>DB: upsert commits
      Async->>DB: extract and upsert story_commit_link
    else pull_request event
      Async->>DB: upsert pull_request
      Async->>DB: extract story links from PR body
    else workflow_run event
      Async->>DB: upsert pipeline_run
    else workflow_job event
      Async->>DB: upsert job, steps
      alt step conclusion FAILED
        Async->>GH: fetch step log (capped)
        Async->>Red: redact(log)
        Red-->>Async: redacted excerpt
        Async->>DB: upsert log_excerpt
      end
    end
    Async->>Audit: event + lineage
    Async->>Outbox: mark done
  end
```

## 8. Install Backfill and Nightly Resync

```mermaid
sequenceDiagram
  autonumber
  participant GH
  participant WH as WebhookController
  participant BF as InstallBackfillService
  participant Resync as ResyncScheduler
  participant Gh as GitHubRestClient
  participant DB

  GH->>WH: installation webhook (installed)
  WH->>BF: startBackfill(installationId, repos)
  BF->>Gh: list commits / PRs / runs for last 14 days per repo
  Gh->>GH: REST with retry-after
  GH-->>Gh: paginated data
  Gh-->>BF: normalized entities
  BF->>DB: upsert entities + story links
  BF->>DB: record InstallBackfillCheckpoint

  Note over Resync: every 24h
  Resync->>Gh: list commits / runs for last 72h per installed repo
  Resync->>DB: compare with ingested rows
  alt drift detected
    Resync->>DB: upsert missing rows
    Resync->>DB: write DriftAuditEntry
  end
```

## 9. State Machines

### 9.1 PipelineRun

```mermaid
stateDiagram-v2
  [*] --> QUEUED
  QUEUED --> IN_PROGRESS
  IN_PROGRESS --> COMPLETED_SUCCESS
  IN_PROGRESS --> COMPLETED_FAILURE
  IN_PROGRESS --> COMPLETED_CANCELLED
  IN_PROGRESS --> COMPLETED_TIMED_OUT
  COMPLETED_FAILURE --> RERUN_IN_PROGRESS
  COMPLETED_TIMED_OUT --> RERUN_IN_PROGRESS
  RERUN_IN_PROGRESS --> COMPLETED_SUCCESS
  RERUN_IN_PROGRESS --> COMPLETED_FAILURE
```

Triage rows follow the run's most-recent conclusion: when `RERUN_IN_PROGRESS → COMPLETED_*` changes, prior triage is marked SUPERSEDED and a fresh one is enqueued (only if final state is FAILURE / TIMED_OUT).

### 9.2 PullRequest

```mermaid
stateDiagram-v2
  [*] --> OPEN
  OPEN --> DRAFT
  DRAFT --> OPEN
  OPEN --> MERGED
  OPEN --> CLOSED
  DRAFT --> CLOSED
```

AI PR review runs on OPEN and DRAFT (if the workspace autonomy level permits). MERGED and CLOSED suppress further runs.

### 9.3 AI PR Review Note

```mermaid
stateDiagram-v2
  [*] --> PENDING
  PENDING --> SUCCESS
  PENDING --> FAILED
  SUCCESS --> STALE
  FAILED --> STALE
  STALE --> [*]
```

STALE = head commit advanced past this row's `headCommitSha`. Row stays in DB for audit but is hidden from default UI view.

### 9.4 AI Triage Row

```mermaid
stateDiagram-v2
  [*] --> PENDING
  PENDING --> SUCCESS
  PENDING --> FAILED
  PENDING --> EVIDENCE_MISMATCH
  SUCCESS --> SUPERSEDED
  FAILED --> SUPERSEDED
  EVIDENCE_MISMATCH --> SUPERSEDED
  SUPERSEDED --> [*]
```

## 10. Error Cascade and Per-Card Isolation

```mermaid
flowchart TD
  A[Catalog request] --> B[Fan-out 3 projections]
  B --> C1[Summary]
  B --> C2[Grid]
  B --> C3[AiSummary]
  C1 --> D1{timeout or error?}
  C2 --> D2{timeout or error?}
  C3 --> D3{timeout or error?}
  D1 -- no --> E1[SectionResult data]
  D1 -- yes --> F1[SectionResult error]
  D2 -- no --> E2[SectionResult data]
  D2 -- yes --> F2[SectionResult error]
  D3 -- no --> E3[SectionResult data]
  D3 -- yes --> F3[SectionResult error]
  E1 & E2 & E3 & F1 & F2 & F3 --> G[Merge into aggregate]
  G --> H[Return 200]
```

Page-level errors are reserved for `CB_WORKSPACE_FORBIDDEN` and `CB_REPO_NOT_FOUND` / `CB_RUN_NOT_FOUND` / `CB_STORY_NOT_FOUND`. Everything else degrades per card.

## 11. Refresh Strategy

- **Page focus regain** — refresh "stale" cards (AI summary, Open PRs, Recent Runs) if last load >60s ago.
- **Websocket / SSE (V1.1 candidate)** — out of scope for V1; webhook-driven updates reach the DB but the UI refreshes on navigation or manual refresh.
- **Manual refresh** — each card exposes a refresh icon that re-requests only that card.
- **AI Triage / Review polling** — when a row is PENDING, the view polls the specific endpoint every 3s with jitter; back-off to 10s after 30s; cap at 2 minutes (then surface FAILED).

## 12. Phase A / Phase B Toggle

Frontend toggles between mocks and backend via `import.meta.env.DEV && !import.meta.env.VITE_USE_BACKEND`. All mock latencies match the per-projection timeout (300–500ms) so Phase A UX reflects Phase B behavior. Mock `commandLoop` injects `CB_AI_UNAVAILABLE` at 5%, `CB_GH_RATE_LIMIT` at 2%, and an evidence-mismatch triage at 3% to exercise the UI states.

## 13. Observability

Every backend call is traced with a correlation id propagated from the frontend (`x-correlation-id`). Webhook ingestion generates its own correlation id and tags it with installation id + delivery id from the `X-GitHub-Delivery` header so webhook-to-UI latency can be measured end-to-end. Metrics (P95 latency, error rate, projection timeout rate, AI success rate, GH rate-limit hits) are exported via the shared metrics facade.
