# Testing Management — Data Flow

Runtime sequences, state machines, error cascade, and refresh strategy for the Testing Management slice. Operationalizes [testing-management-architecture.md](testing-management-architecture.md) and the contracts in [../05-design/contracts/testing-management-API_IMPLEMENTATION_GUIDE.md](../05-design/contracts/testing-management-API_IMPLEMENTATION_GUIDE.md).

## 1. Catalog Page Load (Phase A — mocks)

```mermaid
sequenceDiagram
  autonumber
  participant User
  participant Router
  participant View as CatalogView
  participant Store as testingStore
  participant Mock as testingMock

  User->>Router: /testing
  Router->>View: mount
  View->>Store: initCatalog(filters)
  Store->>Mock: getCatalogAggregate(filters)
  Mock-->>Store: aggregate with 3 SectionResult cards
  Store-->>View: state.catalog.* hydrated
  View-->>User: render Summary, Grid, Filters
```

## 2. Catalog Page Load (Phase B — backend)

```mermaid
sequenceDiagram
  autonumber
  participant View as CatalogView
  participant Store as testingStore
  participant Api as testingApi
  participant Ctrl as TestingCatalogController
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
    Svc->>DB: CatalogGridProjection (test plans with case counts, state, coverage LED)
  end
  Svc->>Req: (batched) verify linked REQ-IDs visible in workspace
  Svc-->>Ctrl: CatalogAggregate{SectionResult}
  Ctrl-->>Api: 200 ApiResponse<CatalogAggregate>
  Api-->>Store: aggregate
  Store-->>View: state hydrated
```

## 3. Plan Detail Page Load

```mermaid
sequenceDiagram
  autonumber
  participant View as PlanDetailView
  participant Store
  participant Api
  participant Ctrl
  participant Svc as PlanDetailService
  participant DB
  participant Req

  View->>Store: openPlan(planId)
  Store->>Api: GET /plans/{planId}
  Api->>Ctrl: HTTP GET
  Ctrl->>Svc: loadAggregate(planId, principal)
  par 6 card projections (500ms each)
    Svc->>DB: Header (plan metadata)
  and
    Svc->>DB: Cases (case catalog with type, priority, state, linked-REQs, last-run status)
  and
    Svc->>DB: Coverage (REQ-ID rollup with case counts and aggregate status)
  and
    Svc->>DB: RecentRuns (run catalog with environment, trigger, actor, pass/fail counts)
  and
    Svc->>DB: AiDraftInbox (DRAFT state cases with source REQ, skill version, inline approve/reject)
  and
    Svc->>DB: AiInsights (narrative summary of 7-day changes, red-trending cases, coverage gaps)
  end
  Svc->>Req: batched verify REQ-IDs from cases and coverage
  Svc-->>Ctrl: PlanDetailAggregate{SectionResult}
  Ctrl-->>Api: 200
  Api-->>Store: aggregate
  Store-->>View: render 6 cards
```

## 4. Run Detail Page Load

```mermaid
sequenceDiagram
  autonumber
  participant View as RunDetailView
  participant Store
  participant Api
  participant Ctrl
  participant Svc as RunDetailService
  participant DB
  participant Req

  View->>Store: openRun(runId)
  Store->>Api: GET /runs/{runId}
  Api->>Ctrl: HTTP GET
  Ctrl->>Svc: loadAggregate(runId, principal)
  par run projections
    Svc->>DB: RunHeader (run metadata, plan, environment, trigger, actor, duration)
  and
    Svc->>DB: RunCaseOutcomes (per-case PASS/FAIL/SKIP/ERROR, failure excerpts capped 4KB, last-good timestamps)
  and
    Svc->>DB: RunCoverage (stories covered, REQ-ID rollup)
  end
  Svc->>Req: verify referenced REQ-IDs visible
  Svc-->>Ctrl: RunDetailAggregate
  Ctrl-->>Api: 200
  Api-->>Store: aggregate
  Store-->>View: render cards
```

## 5. Run Ingestion via Webhook

```mermaid
sequenceDiagram
  autonumber
  participant CI as CI / Webhook Source
  participant Ctrl as RunIngestController
  participant SV as SignatureVerifier
  participant Parser as IngestionParser
  participant Norm as Normalizer
  participant DB
  participant Red as LogRedactor
  participant Emit as LineageEmitter
  participant Ui as UI refresh
  participant Audit

  CI->>Ctrl: POST /webhooks/ingest (JUnit/TestNG/Playwright/Cypress)
  Ctrl->>SV: verify(signature, body, workspace secret)
  alt signature invalid
    SV-->>Ctrl: false
    Ctrl->>Audit: TM_INGEST_SIGNATURE_INVALID
    Ctrl-->>CI: 401
  else signature valid
    SV-->>Ctrl: true
    Ctrl->>Parser: parse(file format, body)
    alt parse fails
      Parser-->>Ctrl: ParseError
      Ctrl->>DB: persist Run(status=INGEST_FAILED, errorMsg=first 2KB)
      Ctrl->>Audit: TM_INGEST_FAILED (format)
      Ctrl-->>CI: 400 TM_INGEST_PARSE_ERROR
    else parse succeeds
      Parser-->>Ctrl: ParsedRunBundle
      Ctrl->>Norm: normalize(bundle, planId, environmentId, actor)
      Norm->>Red: redact failure excerpts for secrets
      Red-->>Norm: redacted excerpts
      Norm-->>Ctrl: NormalizedRunEntity
      Ctrl->>DB: persist Run + RunCaseOutcome rows
      Ctrl->>Audit: TM_RUN_INGESTED
      Ctrl->>Emit: emitLineageEvent (TM authoring domain)
      Emit-->>Ctrl: ack
      Ctrl-->>CI: 200 runId
      Ctrl->>Ui: broadcast refresh (plan view, case view if in viewport)
    end
  end
```

## 6. AI Draft Generation + Approval Workflow

```mermaid
sequenceDiagram
  autonumber
  participant User
  participant View as DraftInboxView
  participant Store
  participant Api
  participant Ctrl as DraftController
  participant AuthZ as AuthZ Gate
  participant Svc as AiDraftService
  participant Skill as Skill Runtime
  participant DB
  participant Audit

  User->>View: click "Draft from REQ-123"
  View->>Store: requestDraft(reqId, planId)
  Store->>Api: POST /ai/test-cases/draft (reqId, planId)
  Api->>Ctrl: HTTP POST
  Ctrl->>AuthZ: canDraft(user, planId) - must be QA Lead or Tech Lead
  alt authz fails
    AuthZ-->>Ctrl: false
    Ctrl-->>Api: 403 TM_DRAFT_UNAUTHORIZED
  else authz succeeds
    Ctrl->>AuthZ: checkAutonomy(workspace)
    alt autonomy=DISABLED
      AuthZ-->>Ctrl: DISABLED
      Ctrl-->>Api: 403 TM_AI_DISABLED
    else autonomy=OBSERVATION
      Ctrl->>Svc: enqueueDraft(reqId, planId, skillVersion)
      Svc->>DB: load REQ text from Requirement slice
      Svc->>Skill: generateTestCases(reqText, priorCaseTitles)
      Skill-->>Svc: CandidateCaseList
      Svc->>DB: persist TestCase rows (state=DRAFT, origin=AI_DRAFT)
      Svc-->>Ctrl: draft candidates
      Ctrl-->>Api: 200 with candidates marked OBSERVATION (readonly preview)
      Api-->>Store: draft list (approve button disabled)
      Store-->>View: render inbox with disabled approve
    else autonomy=SUPERVISED or AUTONOMOUS
      Ctrl->>Svc: enqueueDraft(reqId, planId, skillVersion)
      Svc->>DB: load REQ text
      Svc->>Skill: generateTestCases(reqText, priorCaseTitles)
      Skill-->>Svc: CandidateCaseList
      Svc->>DB: persist TestCase rows (state=DRAFT, origin=AI_DRAFT, cited excerpt)
      Svc-->>Ctrl: draft candidates
      Ctrl-->>Api: 200 CandidateCaseList
      Api-->>Store: draft list
      Store-->>View: render inbox with approve/reject/edit buttons
      Note over User,View: User reviews AI drafts
      User->>View: click "Approve" on a draft case
      View->>Store: approveDraft(caseId)
      Store->>Api: PATCH /cases/{caseId} (state=ACTIVE)
      Api->>Ctrl: HTTP PATCH
      Ctrl->>AuthZ: canApproveDraft(user, caseId)
      alt authz fails
        AuthZ-->>Ctrl: false
        Ctrl-->>Api: 403
      else authz succeeds
        Ctrl->>Svc: transitionDraft(caseId, DRAFT->ACTIVE)
        Svc->>DB: update TestCase (state=ACTIVE, approvedBy, approvedAt)
        Svc->>Audit: TM_DRAFT_APPROVED (actor, caseId, reqId, skillVersion)
        Svc->>Emit: emitLineageEvent
        Svc-->>Ctrl: ack
        Ctrl-->>Api: 200 updated case
        Api-->>Store: updated
        Store-->>View: refresh inbox (case removed from DRAFT list)
      end
    end
  end
```

## 7. REQ Linkage Verification

```mermaid
sequenceDiagram
  autonumber
  participant User
  participant View as CaseDetailView
  participant Store
  participant Api
  participant Ctrl as CaseController
  participant Svc as CaseService
  participant DB
  participant ReqCache as Requirement Cache
  participant ReqSlice as Requirement Slice API
  participant Audit

  User->>View: edit case, add linked REQ "REQ-456"
  View->>Store: saveCase(caseId, linkedReqs=[REQ-456])
  Store->>Api: PATCH /cases/{caseId} (linkedReqs)
  Api->>Ctrl: HTTP PATCH
  Ctrl->>Svc: updateCase(caseId, linkedReqs, principal)
  par validate and persist
    Svc->>DB: begin txn
    Svc->>DB: update TestCase linkedReqs
    Svc->>ReqCache: verify(REQ-456, workspaceId)
    alt in cache
      ReqCache-->>Svc: found, status=VERIFIED
    else not in cache
      ReqCache->>ReqSlice: GET /requirements/{REQ-456} ?workspaceId
      alt REQ visible and found
        ReqSlice-->>ReqCache: Requirement entity
        ReqCache-->>Svc: status=VERIFIED
      else REQ not found or not visible
        ReqSlice-->>ReqCache: 404
        ReqCache-->>Svc: status=UNKNOWN_REQ
      end
    end
    Svc->>DB: upsert TestCaseReqLink(caseId, reqId, status)
    Svc->>Audit: TM_CASE_REQ_LINKED
    Svc-->>Ctrl: ack
  end
  Ctrl-->>Api: 200 updated case
  Api-->>Store: case with req chips (GREEN=verified, AMBER=not visible, RED=not found)
  Store-->>View: render chips
  Note over Store: invalidate plan coverage + requirement inverse view
```

## 8. Case Detail Deep-Link with REQ Traceability

```mermaid
sequenceDiagram
  autonumber
  participant User
  participant Router
  participant View as CaseDetailView
  participant Store
  participant Api
  participant Ctrl as CaseDetailController
  participant DB
  participant Req

  User->>Router: navigate /cases/{caseId}?from=REQ-456
  Router->>View: mount (caseId, fromReqId)
  View->>Store: loadCase(caseId)
  Store->>Api: GET /cases/{caseId}
  Api->>Ctrl: HTTP GET
  Ctrl->>DB: load Case, linked REQs, last 20 run outcomes
  Ctrl->>Req: verify linked REQ-IDs visible
  Ctrl-->>Api: 200 CaseDetailAggregate
  Api-->>Store: case + outcomes + req chips
  Store-->>View: render case header, steps, linked REQ chips (highlight REQ-456 if it matches fromReqId)
  Note over View: user can click REQ-456 chip to navigate to Requirement story
  User->>Router: click REQ-456 chip
  Router->>Router: navigate /requirements?storyId=REQ-456 (or deep-link to Requirement slice)
```

## 9. Requirement Slice Tests Tab Callback

```mermaid
sequenceDiagram
  autonumber
  participant User
  participant ReqView as Requirement StoryDetailView
  participant ReqRouter as Requirement Router
  participant Api as TestingApi
  participant Ctrl as TraceabilityController
  participant Svc as TraceabilityService
  participant DB
  participant Req as Requirement Slice Verify

  User->>ReqRouter: navigate /requirements/REQ-456
  ReqRouter->>ReqView: mount (storyId=REQ-456)
  ReqView->>ReqView: render Tests tab (empty, ready for callback)
  User->>ReqView: click Tests tab
  ReqView->>Api: GET /traceability?storyId=REQ-456
  Api->>Ctrl: HTTP GET
  Ctrl->>Req: verify(REQ-456, workspaceId)
  alt REQ not visible
    Req-->>Ctrl: 404
    Ctrl-->>Api: 404 TM_STORY_NOT_FOUND
  else REQ visible
    Ctrl->>Svc: inverseLookup(REQ-456, principal)
    par three sections
      Svc->>DB: load active test cases linked to REQ-456
    and
      Svc->>DB: load plans containing those cases
    and
      Svc->>DB: load most-recent run outcome per case (capped 200 REQs)
    end
    Svc-->>Ctrl: TraceabilityAggregate(cases, plans, run outcomes)
    Ctrl-->>Api: 200
  end
  Api-->>ReqView: aggregate (cases table with plan, status, last-run timestamp)
  ReqView-->>User: render Tests tab with case grid
  Note over User: each case row is a deep-link back to /testing/cases/{caseId}?from=REQ-456
```

## 10. State Machines

### 10.1 TestPlan

```mermaid
stateDiagram-v2
  [*] --> DRAFT
  DRAFT --> ACTIVE
  ACTIVE --> ARCHIVED
  DRAFT --> ARCHIVED
  ARCHIVED --> [*]
```

DRAFT plans do not accept run ingestion. ARCHIVED plans are read-only and do not accept new runs or case edits.

### 10.2 TestCase

```mermaid
stateDiagram-v2
  [*] --> DRAFT
  DRAFT --> ACTIVE
  DRAFT --> DEPRECATED
  ACTIVE --> DEPRECATED
  DEPRECATED --> [*]
```

DRAFT = AI-drafted, pending approval. ACTIVE = executable and counted in coverage. DEPRECATED = hidden from active views but retained for audit. State transitions are gated by role (QA Lead / Tech Lead for approval).

### 10.3 TestRun

```mermaid
stateDiagram-v2
  [*] --> INGESTING
  INGESTING --> PASSED
  INGESTING --> FAILED
  INGESTING --> ABORTED
  INGESTING --> INGEST_FAILED
  PASSED --> [*]
  FAILED --> [*]
  ABORTED --> [*]
  INGEST_FAILED --> [*]
```

INGESTING = webhook/upload in progress. PASSED = all cases passed. FAILED = at least one case failed. ABORTED = run was cancelled. INGEST_FAILED = parse/validation failure (immutable; operator must re-upload). Runs are immutable once terminal; re-ingestion with same `externalRunId` is a 409 conflict unless `force=true` (admin override).

### 10.4 AI Draft Lifecycle

```mermaid
stateDiagram-v2
  [*] --> CREATED
  CREATED --> EDITED
  EDITED --> APPROVED
  CREATED --> APPROVED
  APPROVED --> ACTIVE_CASE
  CREATED --> REJECTED
  EDITED --> REJECTED
  REJECTED --> DEPRECATED_CASE
  ACTIVE_CASE --> [*]
  DEPRECATED_CASE --> [*]
```

CREATED = AI skill generated candidates. EDITED = human reviewer made changes before approval. APPROVED = reviewer approved; transitions TestCase.state to ACTIVE. REJECTED = reviewer rejected; transitions TestCase.state to DEPRECATED with reason=REJECTED_AT_INBOX. STALE drafts (skill version bump) retain their state but render with STALE badge.

## 11. Error Cascade and Per-Card Isolation

```mermaid
flowchart TD
  A[Plan Detail request] --> B[Fan-out 6 card projections]
  B --> C1[Header]
  B --> C2[Cases]
  B --> C3[Coverage]
  B --> C4[RecentRuns]
  B --> C5[AiDraftInbox]
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
  G --> H[Return 200 with per-card retry affordance]
```

Page-level errors are reserved for `TM_PLAN_NOT_FOUND`, `TM_RUN_NOT_FOUND`, `TM_CASE_NOT_FOUND`. Everything else (projection timeout, requirement slice unavailable, AI unavailable) degrades per card with per-section retry button. Each card independently re-requests only its projection.

### 11.1 Per-Section Retry Flow

When a card error occurs (e.g., Coverage card timeout), the user clicks "Retry" on that card. The frontend re-requests only that card's endpoint, passing the card identifier. The backend re-executes only that projection (e.g., `CoverageProjection`) without re-executing Header, Cases, etc. This minimizes latency and avoids thundering herd on Requirement slice lookup.

```mermaid
sequenceDiagram
  autonumber
  participant User
  participant View
  participant Store
  participant Api
  participant Ctrl

  User->>View: click Retry on Coverage card (failed)
  View->>Store: retryCardProjection(planId, cardType=COVERAGE)
  Store->>Api: GET /plans/{planId}?card=COVERAGE
  Api->>Ctrl: HTTP GET (card param)
  Ctrl->>Ctrl: skip Header, Cases, RecentRuns, etc. - only run Coverage projection
  Ctrl-->>Api: 200 {coverage: SectionResult}
  Api-->>Store: merged partial aggregate
  Store-->>View: update coverage card only
```

## 12. Refresh Strategy

- **Page focus regain** — refresh "stale" cards (AI Insights, Recent Runs) if last load >60s ago (REQ-TM-92 P95 budget).
- **Run ingestion webhook** — emits LineageEvent that propagates to open Plan Detail / Run Detail views via Websocket (V1.1 candidate). V1 relies on manual refresh or navigation.
- **Manual refresh** — each card exposes a refresh icon that re-requests only that card (per-section retry flow in §11.1).
- **Case state change** — when a case transitions DRAFT→ACTIVE (approval) or ACTIVE→DEPRECATED (rejection), invalidate Plan Coverage and Requirement inverse views; refresh within 30s at P95 (REQ-TM-26).
- **REQ linkage change** — when TestCaseReqLink rows are added/removed, invalidate Plan Coverage and Requirement inverse views.
- **AI skill version bump** — when a skill version advances, mark prior drafts with `skillVersion < current` as STALE; render STALE badge and "re-draft" action.

## 13. API Client Chain

The frontend API client for Testing Management (`testingApi`) applies:

1. **Authorization token** — bearer token from auth store (via shared auth interceptor)
2. **Workspace header** — `X-Workspace-Id` from shared context
3. **Projection timeout** — per-projection 500ms timeout with `SectionResult` fallback; if a projection exceeds 500ms, the section returns `{status: TIMEOUT, data: null, error: "Projection timed out"}` and the UI renders the timeout state with retry
4. **Error codec** — errors are wrapped in `ApiResponse<T>` with error code (e.g., `TM_PLAN_NOT_FOUND`, `TM_INGEST_PARSE_ERROR`) and user-facing message
5. **Correlation id** — every request propagates `X-Correlation-Id` from frontend (or generated by backend) for end-to-end traceability
6. **Retry policy** — non-idempotent writes (POST, PATCH) do not auto-retry; read operations (GET) retry once on network error with exponential backoff; per-card retries are user-driven (Retry button)

## 14. Observability

Every backend call is traced with a correlation id. Run ingestion and AI draft generation generate their own correlation ids and tag them with:
- Run ingestion: workspace id, external run id, format type (JUnit/Playwright/etc.), ingestion trigger (webhook vs. manual upload), parse success/failure
- AI draft generation: workspace id, req id, skill version, autonomy level, draft success/failure, candidate count
- Case state transitions: workspace id, plan id, case id, actor, from/to state, audit trail

Metrics exported: P95 load latency by page and card, error rate by error code (per-card vs. page-level), REQ lookup cache hit rate, AI draft generation rate and success rate, run ingestion parse failure rate.

## 15. Open Questions (tracked, not blocking)

- Should REQ linkage verification cache be distributed (Redis) or local (Caffeine)? Default: Caffeine with 15-min TTL per workspace + 5-min lazy refresh (V1.1 refinement).
- Should per-section retry auto-trigger on specific transient errors (e.g., Requirement slice 503)? Default: no in V1; user-driven retry button. V1.1 candidate: auto-retry with exponential backoff, capped at 2 retries.
- Should run ingestion webhook support async/eventual consistency (enqueue → process later) vs. sync processing? Default: sync within 30s (REQ-TM-92 budget); V1.1 candidate: async with polling for long-running formats.
