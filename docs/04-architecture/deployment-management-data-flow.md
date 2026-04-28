# Deployment Management — Data Flow

Runtime sequences, state machines, error cascade, and refresh strategy for the Deployment Management slice. Operationalizes [deployment-management-architecture.md](deployment-management-architecture.md) and the contracts in [../05-design/contracts/deployment-management-API_IMPLEMENTATION_GUIDE.md](../05-design/contracts/deployment-management-API_IMPLEMENTATION_GUIDE.md).

## 1. Catalog Page Load (Phase A — mocks)

```mermaid
sequenceDiagram
  autonumber
  participant User
  participant Router
  participant View as CatalogView
  participant Store as deploymentStore
  participant Mock as deploymentMock

  User->>Router: /deployment
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
  participant Store as deploymentStore
  participant Api as deploymentApi
  participant Ctrl as DeploymentController
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

## 3. Application Detail Page Load

```mermaid
sequenceDiagram
  autonumber
  participant View as ApplicationDetailView
  participant Store
  participant Api
  participant Ctrl
  participant Svc as ApplicationDetailService
  participant DB
  participant CB as Code and Build Facade
  participant Req as Requirement Facade

  View->>Store: openApplication(applicationId)
  Store->>Api: GET /applications/{applicationId}
  Api->>Ctrl: HTTP GET
  Ctrl->>Svc: loadAggregate(applicationId, principal)
  par 6 projections
    Svc->>DB: Header
  and
    Svc->>DB: Environments
  and
    Svc->>DB: RecentReleases
  and
    Svc->>DB: RecentDeploys
  and
    Svc->>DB: TraceSummary
  and
    Svc->>DB: AiInsights
  end
  Svc->>CB: batched resolve buildArtifactRef -> commitCount per release row
  Svc->>Req: batched verify for Story-Ids referenced in TraceSummary
  Svc-->>Ctrl: ApplicationDetailAggregate
  Ctrl-->>Api: 200
  Api-->>Store: aggregate
  Store-->>View: render 6 cards
```

## 4. Release Detail Page Load with AI Release Notes

```mermaid
sequenceDiagram
  autonumber
  participant View as ReleaseDetailView
  participant Store
  participant Api
  participant Ctrl
  participant Svc as ReleaseDetailService
  participant NotesSvc as AiReleaseNotesService
  participant DB
  participant CB as Code and Build Facade
  participant Req as Requirement Facade
  participant AI as Skill Runtime

  View->>Store: openRelease(releaseId)
  Store->>Api: GET /releases/{releaseId}
  Api->>Ctrl: HTTP GET
  Ctrl->>Svc: loadAggregate(releaseId, principal)
  par projections
    Svc->>DB: ReleaseHeader
  and
    Svc->>DB: ReleaseDeploys
  and
    Svc->>DB: AiReleaseNotes (latest current row)
  end
  Svc->>CB: resolve BuildArtifact (commits[], commitRange)
  alt BuildArtifact unresolved
    CB-->>Svc: PENDING
    Svc-->>Ctrl: aggregate with ReleaseCommits.error=DP_BUILD_ARTIFACT_PENDING
  else resolved
    CB-->>Svc: artifact with commits
    Svc->>Req: batched verify story IDs extracted from commits
    Svc-->>Ctrl: aggregate
  end

  alt release is DEPLOYED and no current AI notes exist and BuildArtifact resolved and autonomy>=OBSERVATION
    Svc->>NotesSvc: enqueueNotes(releaseId)
    NotesSvc->>DB: insert ai_release_notes row (status=PENDING)
  end
  Ctrl-->>Api: 200
  Api-->>Store: aggregate
  Store-->>View: render

  Note over NotesSvc,AI: async background
  NotesSvc->>CB: resolve BuildArtifact (commits + range + prior-prod-revision diff base)
  NotesSvc->>Req: resolve stories (union from commits)
  NotesSvc->>AI: generateReleaseNotes(release, commits, stories, priorProdRelease)
  AI-->>NotesSvc: NotesDraft
  NotesSvc->>NotesSvc: evidence integrity check (every referenced storyId must be in resolved set)
  alt evidence ok
    NotesSvc->>DB: update ai_release_notes row (status=SUCCESS, payload)
  else mismatch
    NotesSvc->>DB: update row (status=EVIDENCE_MISMATCH)
  end
  Note over View: polls /releases/{releaseId}/ai-notes every 3s until non-PENDING
```

## 5. Deploy Detail Page Load

```mermaid
sequenceDiagram
  autonumber
  participant View as DeployDetailView
  participant Store
  participant Api
  participant Ctrl
  participant Svc as DeployDetailService
  participant DB
  participant CB as Code and Build Facade

  View->>Store: openDeploy(deployId)
  Store->>Api: GET /deploys/{deployId}
  Api->>Ctrl: HTTP GET
  Ctrl->>Svc: loadAggregate(deployId, principal)
  par 4 projections
    Svc->>DB: DeployHeader
  and
    Svc->>DB: StageTimeline
  and
    Svc->>DB: Approvals (role-gated rationale)
  and
    Svc->>DB: ArtifactRef
  end
  Svc->>CB: (optional) enrich ArtifactRef with build summary
  Svc-->>Ctrl: DeployDetailAggregate
  Ctrl-->>Api: 200
  Api-->>Store: aggregate
  Store-->>View: render 4 cards plus OpenIncidentAction
```

## 6. Environment Detail Page Load

```mermaid
sequenceDiagram
  autonumber
  participant View as EnvironmentDetailView
  participant Store
  participant Api
  participant Ctrl
  participant Svc as EnvironmentDetailService
  participant DB

  View->>Store: openEnvironment(applicationId, environmentName)
  Store->>Api: GET /applications/{applicationId}/environments/{environmentName}
  Api->>Ctrl: HTTP GET
  Ctrl->>Svc: loadAggregate(applicationId, environmentName, principal)
  par 4 projections
    Svc->>DB: EnvironmentHeader
  and
    Svc->>DB: CurrentRevision + PriorRevision + LastGood + LastFailed
  and
    Svc->>DB: 90-day deploy timeline
  and
    Svc->>DB: 30-day metrics (CFR, MTTR)
  end
  Svc-->>Ctrl: EnvironmentDetailAggregate
  Ctrl-->>Api: 200
  Api-->>Store: aggregate
  Store-->>View: render
```

## 7. Story → Release → Deploy Inverse Lookup

```mermaid
sequenceDiagram
  autonumber
  participant View as TraceabilityView
  participant Store
  participant Api
  participant Ctrl
  participant Svc as TraceabilityService
  participant Req
  participant CB as Code and Build Facade
  participant DB

  View->>Store: lookupStory(STORY-4211)
  Store->>Api: GET /traceability?storyId=STORY-4211
  Api->>Ctrl: HTTP GET
  Ctrl->>Req: verify Story-Id visibility
  alt story not visible
    Ctrl-->>Api: 404 DP_STORY_NOT_FOUND
  else story visible
    Ctrl->>Svc: inverseLookup(STORY-4211, principal)
    Svc->>CB: inverse(storyId) -> buildArtifactIds[]
    CB-->>Svc: [BA-77182, BA-77190, ...]
    Svc->>DB: releases WHERE buildArtifactRef IN (...)
    Svc->>DB: deploys WHERE releaseId IN (...)
    Svc-->>Ctrl: TraceabilityAggregate(releases[], deploys[] grouped by environment)
    Ctrl-->>Api: 200
  end
  Api-->>Store: aggregate
  Store-->>View: render releases + deploys grouped by env
```

## 8. Webhook Ingestion from Jenkins

```mermaid
sequenceDiagram
  autonumber
  participant J as Jenkins
  participant WH as WebhookController
  participant SV as SignatureVerifier
  participant Parser
  participant Outbox as IngestionOutbox
  participant Async as AsyncDispatcher
  participant DB
  participant RR as ReleaseVersionResolver
  participant RD as RollbackDetector
  participant Audit

  J->>WH: POST /webhooks/jenkins
  WH->>SV: verify(signature, body, perInstanceSecret)
  alt signature invalid
    SV-->>WH: false
    WH->>Audit: DP_INGEST_SIGNATURE_INVALID
    WH-->>J: 401
  else signature valid
    SV-->>WH: true
    WH->>Parser: parse(event, body)
    Parser-->>WH: TypedEvent
    WH->>Outbox: enqueue(TypedEvent)
    WH-->>J: 202
  end

  loop async worker
    Async->>Outbox: claim oldest
    alt job.started
      Async->>DB: upsert deploy(state=IN_PROGRESS)
    else stage.completed
      Async->>DB: upsert deploy_stage with outcome + duration
    else input.approved / input.rejected
      Async->>DB: upsert approval_event(approver, decision, gatePolicyVersion, rationale)
    else job.completed
      Async->>RR: resolveReleaseVersion(build)
      alt unresolved
        Async->>DB: update deploy(unresolvedFlag=true, state=conclusion)
      else resolved
        Async->>DB: upsert Release(applicationId, releaseVersion) if absent
        Async->>DB: update deploy(releaseId, state=conclusion, conclusionAt)
      end
      Async->>RD: classify(deploy, priorEnvironmentDeploys)
      RD-->>Async: isRollback
      Async->>DB: set deploy.isRollback
    else build.promoted
      Async->>DB: upsert deploy_promotion_link
    else build.deleted
      Async->>DB: mark deploy tombstoned (keep for audit)
    end
    Async->>Audit: event + lineage
    Async->>Outbox: mark done
  end
```

## 9. Install Backfill and Nightly Resync

```mermaid
sequenceDiagram
  autonumber
  participant Op as Operator
  participant WH as Platform Center
  participant BF as JenkinsBackfillService
  participant Resync as ResyncScheduler
  participant Jc as JenkinsRestClient
  participant J as Jenkins
  participant DB

  Op->>WH: register Jenkins instance (baseUrl, credentialId)
  WH->>BF: startBackfill(jenkinsInstanceId, folders)
  BF->>Jc: list jobs / builds for last 30 days per folder
  Jc->>J: REST with retry-after
  J-->>Jc: paginated data
  Jc-->>BF: normalized entities
  BF->>DB: upsert applications / releases / deploys / stages
  BF->>DB: record JenkinsBackfillCheckpoint

  Note over Resync: every 24h
  Resync->>Jc: list builds for last 72h per Jenkins instance
  Resync->>DB: compare with ingested rows
  alt drift detected
    Resync->>DB: upsert missing rows
    Resync->>DB: write DriftAuditEntry
  end
```

## 10. State Machines

### 10.1 Deploy

```mermaid
stateDiagram-v2
  [*] --> PENDING
  PENDING --> IN_PROGRESS
  IN_PROGRESS --> SUCCEEDED
  IN_PROGRESS --> FAILED
  IN_PROGRESS --> CANCELLED
  FAILED --> ROLLED_BACK
  SUCCEEDED --> ROLLED_BACK
  ROLLED_BACK --> [*]
  SUCCEEDED --> [*]
  CANCELLED --> [*]
```

- `PENDING` ← Jenkins `job.queued`
- `IN_PROGRESS` ← Jenkins `job.started`
- `SUCCEEDED` ← Jenkins `job.completed` with conclusion `SUCCESS`
- `FAILED` ← Jenkins `job.completed` with conclusion `FAILURE`
- `CANCELLED` ← Jenkins `job.completed` with conclusion `ABORTED`
- `ROLLED_BACK` is a logical label: the Deploy row keeps its terminal state (`SUCCEEDED` or `FAILED`) and a later deploy with `isRollback=true` for the same environment flips the current-revision pointer.

### 10.2 Release

```mermaid
stateDiagram-v2
  [*] --> PREPARED
  PREPARED --> DEPLOYED
  DEPLOYED --> SUPERSEDED
  PREPARED --> ABANDONED
  SUPERSEDED --> [*]
  ABANDONED --> [*]
```

- `PREPARED` — Release row exists but no `SUCCEEDED` deploy yet.
- `DEPLOYED` — at least one `SUCCEEDED` deploy to any environment.
- `SUPERSEDED` — a later Release has replaced this one as current in every environment that had reached it.
- `ABANDONED` — operator-marked (V1.1).

### 10.3 AI Release Notes Row

```mermaid
stateDiagram-v2
  [*] --> PENDING
  PENDING --> SUCCESS
  PENDING --> FAILED
  PENDING --> EVIDENCE_MISMATCH
  SUCCESS --> STALE
  FAILED --> STALE
  EVIDENCE_MISMATCH --> STALE
  STALE --> [*]
```

STALE = the release's source `buildArtifactSha` advanced past this row's snapshot (rebuild-with-same-version corruption case). The row stays in DB for audit but is hidden from default UI view; admin can Regenerate.

### 10.4 Approval Event

```mermaid
stateDiagram-v2
  [*] --> PROMPTED
  PROMPTED --> APPROVED
  PROMPTED --> REJECTED
  PROMPTED --> TIMED_OUT
  APPROVED --> [*]
  REJECTED --> [*]
  TIMED_OUT --> [*]
```

- `PROMPTED` ← Jenkins `input.prompted` (pipeline is waiting for a human decision).
- `APPROVED` ← Jenkins `input.approved`.
- `REJECTED` ← Jenkins `input.rejected` or Jenkins `input.aborted`.
- `TIMED_OUT` ← Jenkins `input.timeout` (if configured).

## 11. Error Cascade and Per-Card Isolation

```mermaid
flowchart TD
  A[Application Detail request] --> B[Fan-out 6 projections]
  B --> C1[Header]
  B --> C2[Environments]
  B --> C3[RecentReleases]
  B --> C4[RecentDeploys]
  B --> C5[TraceSummary]
  B --> C6[AiInsights]
  C1 --> D1{timeout or error?}
  C2 --> D2{timeout or error?}
  C3 --> D3{timeout or error?}
  C4 --> D4{timeout or error?}
  C5 --> D5{timeout or error?}
  C6 --> D6{timeout or error?}
  D1 -- no --> E1[SectionResult data]
  D1 -- yes --> F1[SectionResult error]
  D2 -- no --> E2[SectionResult data]
  D2 -- yes --> F2[SectionResult error]
  D3 -- no --> E3[SectionResult data]
  D3 -- yes --> F3[SectionResult error]
  D4 -- no --> E4[SectionResult data]
  D4 -- yes --> F4[SectionResult error]
  D5 -- no --> E5[SectionResult data]
  D5 -- yes --> F5[SectionResult error]
  D6 -- no --> E6[SectionResult data]
  D6 -- yes --> F6[SectionResult error]
  E1 & E2 & E3 & E4 & E5 & E6 & F1 & F2 & F3 & F4 & F5 & F6 --> G[Merge into aggregate]
  G --> H[Return 200]
```

Page-level errors are reserved for `DP_WORKSPACE_FORBIDDEN`, `DP_APPLICATION_NOT_FOUND`, `DP_RELEASE_NOT_FOUND`, `DP_DEPLOY_NOT_FOUND`, `DP_ENVIRONMENT_NOT_FOUND`, and `DP_STORY_NOT_FOUND`. Everything else degrades per card.

## 12. Refresh Strategy

- **Page focus regain** — refresh "stale" cards (AI summary, Recent Deploys, Environments current revision) if last load >60s ago.
- **Websocket / SSE (V1.1 candidate)** — out of scope for V1; webhook-driven updates reach the DB but the UI refreshes on navigation or manual refresh.
- **Manual refresh** — each card exposes a refresh icon that re-requests only that card.
- **AI Release Notes polling** — when a row is PENDING, the view polls `/releases/{releaseId}/ai-notes` every 3s with jitter; back-off to 10s after 30s; cap at 2 minutes (then surface FAILED).
- **BuildArtifact pending polling** — when a Release's `buildArtifactRef` is PENDING, the Release Detail page polls the Build card every 5s with jitter for up to 5 minutes, then falls back to admin Retry.

## 13. Phase A / Phase B Toggle

Frontend toggles between mocks and backend via `import.meta.env.DEV && !import.meta.env.VITE_USE_BACKEND`. All mock latencies match the per-projection timeout (300–500ms) so Phase A UX reflects Phase B behavior. Mock `commandLoop` injects `DP_AI_UNAVAILABLE` at 5%, `DP_JENKINS_UNREACHABLE` at 2%, `DP_BUILD_ARTIFACT_PENDING` at 3%, and an evidence-mismatch release-notes row at 3% to exercise the UI states.

## 14. Observability

Every backend call is traced with a correlation id propagated from the frontend (`x-correlation-id`). Webhook ingestion generates its own correlation id and tags it with Jenkins instance id + build full-name + build number so webhook-to-UI latency can be measured end-to-end. Metrics (P95 latency, error rate, projection timeout rate, AI success rate, Jenkins unavailability hits, rollback rate per environment, change-failure-rate, MTTR) are exported via the shared metrics facade.
