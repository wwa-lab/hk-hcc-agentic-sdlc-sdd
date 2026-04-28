# Code & Build Management — Architecture

## 1. Purpose

This document describes the architecture of the **Code & Build Management** slice. V1 is a read-only observability viewer for GitHub + GitHub Actions with Story↔Commit↔Build traceability and three AI capabilities: workspace summary, failure triage, and PR review assist.

### Upstream references

- Requirements: [../01-requirements/code-build-management-requirements.md](../01-requirements/code-build-management-requirements.md)
- Stories: [../02-user-stories/code-build-management-stories.md](../02-user-stories/code-build-management-stories.md)
- Spec: [../03-spec/code-build-management-spec.md](../03-spec/code-build-management-spec.md)

## 2. System Context

Actors, systems, and stores relevant to the slice:

```mermaid
graph LR
  PM[PM]
  Lead[Tech Lead]
  Eng[Engineer]

  subgraph ControlTower[Agentic SDLC Control Tower]
    Shell[App Shell]
    CBFE[Code and Build Frontend]
    CBBE[Code and Build Backend]
    ReqSlice[Requirement Slice]
    IncSlice[Incident Slice]
    Platform[Platform Center]
    AI[AI Skill Runtime]
  end

  GH[GitHub REST and Webhooks]
  DB[Oracle / H2]
  Vault[Secret Vault]
  Audit[Audit / Lineage]

  PM --> Shell
  Lead --> Shell
  Eng --> Shell
  Shell --> CBFE
  CBFE --> CBBE
  CBBE --> DB
  CBBE --> ReqSlice
  CBBE --> IncSlice
  CBBE --> AI
  CBBE --> Platform
  Platform --> Vault
  GH --> CBBE
  CBBE --> GH
  CBBE --> Audit
```

## 3. Component Breakdown — Frontend

```mermaid
graph TB
  Shell[App Shell]

  subgraph FE[Code and Build Frontend]
    CatalogView
    RepoDetailView
    PrDetailView
    RunDetailView
    TraceabilityView
    Store[Pinia Store]
    ApiClient[codeBuildApi]
  end

  Shell --> CatalogView
  Shell --> RepoDetailView
  Shell --> PrDetailView
  Shell --> RunDetailView
  Shell --> TraceabilityView

  CatalogView --> Store
  RepoDetailView --> Store
  PrDetailView --> Store
  RunDetailView --> Store
  TraceabilityView --> Store

  Store --> ApiClient
  ApiClient --> BE[Backend REST]
```

Catalog View mounts: `CatalogSummaryBarCard`, `CatalogGridCard`, `CatalogFilterBar`, `CatalogAiSummaryCard`.

Repo Detail mounts: `RepoHeaderCard`, `RepoBranchesCard`, `RepoOpenPrsCard`, `RepoRecentCommitsCard`, `RepoRecentRunsCard`, `RepoAiInsightsCard`.

PR Detail mounts: `PrHeaderCard`, `PrLinkedStoriesCard`, `PrCiStatusCard`, `PrAiReviewCard`.

Run Detail mounts: `RunHeaderCard`, `RunTimelineCard`, `RunArtifactsCard`, `RunAiTriageCard`, `OpenIncidentAction`.

Traceability View mounts: `TraceabilityInputCard`, `TraceabilityCommitsCard`, `TraceabilityPrsCard`, `TraceabilityRunsCard`.

## 4. Component Breakdown — Backend

```mermaid
graph TB
  subgraph BE[com.sdlctower.domain.codebuildmanagement]
    Ctrl[CodeBuildController]
    Webhook[GithubWebhookController]

    subgraph Services
      CatSvc[CatalogService]
      RepoSvc[RepoDetailService]
      PrSvc[PrDetailService]
      RunSvc[RunDetailService]
      TraceSvc[TraceabilityService]
      AiSumSvc[AiSummaryService]
      AiTriSvc[AiTriageService]
      AiPrSvc[AiPrReviewService]
    end

    subgraph Ingestion
      SigVerify[WebhookSignatureVerifier]
      Parser[WebhookPayloadParser]
      Dispatcher[IngestionDispatcher]
      Resync[ResyncScheduler]
      Backfill[InstallBackfillService]
    end

    subgraph Projections
      Proj[All Projections]
    end

    subgraph Policy
      AccessGuard[CodeBuildAccessGuard]
      Redactor[LogRedactor]
      StoryExtract[StoryIdExtractor]
      Autonomy[AiAutonomyPolicy]
    end

    subgraph Persistence
      Repos[JPA Repositories]
    end

    Integration[GitHubRestClient]
  end

  Ctrl --> Services
  Services --> Proj
  Services --> Policy
  Proj --> Repos
  Repos --> DB[Database]

  Webhook --> SigVerify
  Webhook --> Parser
  Parser --> Dispatcher
  Dispatcher --> Repos
  Dispatcher --> StoryExtract
  Dispatcher --> Redactor
  Resync --> Integration
  Backfill --> Integration
  Integration --> GH[GitHub API]

  AiSumSvc --> SkillRuntime[AI Skill Runtime]
  AiTriSvc --> SkillRuntime
  AiPrSvc --> SkillRuntime
```

## 5. Data Flow (High-level)

```mermaid
sequenceDiagram
  autonumber
  participant User
  participant FE as Code and Build FE
  participant BE as Code and Build BE
  participant DB
  participant Req as Requirement Slice
  participant AI as AI Skill Runtime

  User->>FE: Navigate to /code-build
  FE->>BE: GET /catalog
  par per-projection fan-out
    BE->>DB: CatalogSummaryProjection
  and
    BE->>DB: CatalogGridProjection
  and
    BE->>DB: AiSummaryProjection
  end
  BE-->>FE: Aggregate with SectionResult per card
  FE-->>User: Render cards

  User->>FE: Drill into repo
  FE->>BE: GET /repos/{repoId}
  par 6 projections
    BE->>DB: Header / Branches / PRs / Commits / Runs / AiInsights
  end
  BE-->>FE: Aggregate
  FE-->>User: Render 6 cards

  User->>FE: Open PR detail
  FE->>BE: GET /repos/{repoId}/prs/{prNumber}
  BE->>Req: verify Story-Id chips
  BE->>DB: AiPrReviewProjection keyed by (prId, headSha, skillVersion)
  BE-->>FE: Aggregate
  FE-->>User: Render PR + stories + AI notes

  User->>FE: Open failed run
  FE->>BE: GET /runs/{runId}
  BE->>DB: RunHeader / Timeline / Artifacts / AiTriage
  alt triage missing for this run
    BE->>AI: generateTriage(runId, logExcerpts)
    AI-->>BE: TriageDraft
    BE->>BE: evidence integrity check
    BE->>DB: persist triage or placeholder
  end
  BE-->>FE: Aggregate
  FE-->>User: Render run + triage
```

## 6. Ingestion Flow

```mermaid
sequenceDiagram
  autonumber
  participant GH as GitHub
  participant WH as WebhookController
  participant SV as SignatureVerifier
  participant PR as PayloadParser
  participant D as Dispatcher
  participant DB
  participant Audit

  GH->>WH: POST webhook (event, X-Hub-Signature-256)
  WH->>SV: verify(signature, body, secret)
  alt signature invalid
    SV-->>WH: invalid
    WH-->>GH: 401
    WH->>Audit: CB_INGEST_SIGNATURE_INVALID
  else signature valid
    SV-->>WH: ok
    WH->>PR: parse(event, body)
    PR-->>WH: typed payload
    WH->>D: dispatch(typedPayload)
    D->>DB: upsert Repository / PR / Commit / Run / Job / Step
    D->>DB: extract and upsert StoryCommitLink
    D->>DB: redact and persist log excerpts (for failed steps)
    D->>Audit: write audit + lineage events
    WH-->>GH: 202
  end
```

Resync (every 24h) and install backfill (on `installation` + `installation_repositories`) walk the GitHub REST API and reconcile against ingested rows.

## 7. State Boundaries

- **Frontend state (Pinia):** derived view aggregates per page, per-card status, active filters. No raw webhook payloads, no tokens.
- **Backend state (DB):** canonical record of repositories, branches, pull requests, commits, pipeline runs, jobs, steps, story links, AI review notes, triage rows, AI summary rows, change log, redacted log excerpts.
- **Secret Vault (Platform Center):** GitHub App private key and webhook secret. Backend holds short-lived installation access tokens in memory; never persisted.
- **GitHub (external):** source of truth for code / PR / pipeline state. Control Tower is a read-side consumer.
- **Audit / Lineage stores:** cross-slice shared, append-only.

## 8. Integration Boundary

```mermaid
graph LR
  subgraph FE[Frontend]
    Views
  end

  subgraph BE[Backend]
    Rest[REST]
    Hooks[Webhook]
    Jobs[Scheduled Jobs]
  end

  subgraph External
    GH[GitHub]
    AI[AI Runtime]
    Req[Requirement Slice]
    Plat[Platform Center]
  end

  Views -->|/api/v1/code-build/*| Rest
  GH -->|webhooks| Hooks
  Rest -->|read| GH
  Jobs -->|resync and backfill| GH
  Rest -->|verify Story-Id| Req
  Rest -->|invoke skill| AI
  Rest -->|install tokens| Plat
```

## 9. Non-functional Constraints

- P95 aggregate latency: Catalog ≤1200ms; Repo Detail ≤1500ms; Run Detail ≤1500ms.
- Per-projection timeout: 500ms; failing projection returns `SectionResult(data=null, error=...)`.
- Webhook freshness SLO: push visible in UI within 30s at P95.
- Webhook receiver handles 50 req/s burst per installation with backpressure to an async dispatcher queue.
- GitHub rate-limit aware: primary (5000/hr/install) and secondary (abuse) both handled with retry-after and a per-installation token bucket.
- Every mutation path emits `AuditLogEntry` + `LineageEvent`.
- No direct table access from other slices; downstream consumers use facade endpoints.

## 10. Security Posture

- Webhook signature HMAC-SHA-256 verified on every request; invalid → 401 + audit.
- GitHub tokens never leave Platform Center vault; installation tokens are minted per request and held in memory, never logged.
- Log excerpts are redacted before storage (AWS keys `AKIA…`, GitHub PATs `ghp_…`, generic `Bearer …`); regex + deny-list per shared `SecretsRedactor` utility.
- AI prompts exclude environment variables and secret-shaped tokens (redactor runs on prompts too).
- BLOCKER-severity AI review note bodies are role-gated (PM/Tech Lead on the repo's project); others see counts only.
- All ingestion and AI invocation is audited with correlation ID propagation.

## 11. Risks and Mitigations

| Risk | Mitigation |
| ---- | ---------- |
| Webhook floods during CI storm | Async dispatcher queue with per-installation backpressure + bounded concurrency |
| GitHub rate limits hurt backfill / resync | Per-installation token bucket; exponential backoff; UI banner when sustained |
| AI triage hallucinates file/step references | Evidence integrity check at service time; placeholder shown if mismatch (REQ-CB-32) |
| AI review note visible to wrong audience | BLOCKER role gate + project-scoped access guard |
| Log excerpt leaks secrets | Redactor runs on ingestion AND on AI prompt construction; no raw logs stored |
| Story-Id typos pollute link rows | `UNKNOWN_STORY` status + nightly resolver + visible `UNVERIFIED` chip |
| GitHub App uninstall orphans data | `installation` webhook handler soft-archives repos; audit trail preserved |
| Oracle BLOB handling differs from H2 | Migration authored with dialect-aware column types; verified on Oracle-in-Docker |
| Re-runs create duplicate triage rows | Unique `(runId, skillVersion)` index; SUPERSEDED marker on conclusion change |
| Cross-slice Requirement lookup cascades latency | Batch verify calls; cache within a single request scope only |

## 12. Decisions

Mirrors the spec's decision list (D1–D10) plus architecture-specific choices:

- **D11** — Webhook receiver is synchronous for signature verification + parsing only; heavy work is handed off to `IngestionDispatcher` via an in-process queue (Spring `@Async` + bounded thread pool) backed by a persistent outbox table for durability across restarts.
- **D12** — Projections read from DB; they never call GitHub directly. All GitHub reads happen in ingestion and resync paths. Rationale: predictable latency for user-facing aggregates.
- **D13** — AI runs (summary, triage, PR review) are invoked asynchronously from the request path; user-facing requests read the latest persisted AI row and show PENDING if none exists. Rationale: keeps P95 latency within budget even when the AI runtime is slow.
- **D14** — Story link rows are computed at ingestion time, not at query time. Rationale: avoids re-parsing commit messages on every Catalog load.

## 13. Glossary

See the spec's glossary (§9). Architecture adds:

- **Dispatcher** — in-process component that consumes parsed webhook events and upserts DB rows.
- **Outbox** — DB-backed persistent queue used by the async dispatcher to survive restarts.
- **Projection** — a read-side query function that builds a view DTO from DB rows.
- **Installation Token** — short-lived token minted per API call via GitHub App JWT.
