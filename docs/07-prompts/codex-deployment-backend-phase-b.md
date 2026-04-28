# Codex Prompt: Deployment Management Phase B — Backend Implementation

## Task

Add the **Deployment Management API** to the existing Spring Boot backend. Frontend already renders `/deployment` with mocked data (Phase A). Your job: stand up the backend endpoints, Jenkins ingestion pipeline, persistence, Flyway migrations V60–V67, and seed so the frontend can switch from mocks to live with zero component changes.

V1 scope is a **read-only observability viewer** over Jenkins. The Control Tower writes NOTHING to Jenkins. The backend's only write paths are:

1. AI regenerate (release notes / deploy summary / workspace summary) — user-initiated
2. Jenkins webhook ingestion — sync-verify signature, persist to outbox, return 202; async dispatcher materializes rows
3. Backfill + nightly resync — reads from Jenkins, routes through the same event handlers (never direct DB writes)

## Read First (do not skip)

1. **Feature contract** — [`docs/03-spec/deployment-management-spec.md`](../03-spec/deployment-management-spec.md)
2. **API contract** (endpoint shapes, controller skeleton, error codes, golden fixtures) — [`docs/05-design/contracts/deployment-management-API_IMPLEMENTATION_GUIDE.md`](../05-design/contracts/deployment-management-API_IMPLEMENTATION_GUIDE.md)
3. **Data model** (entities, DTOs, Flyway DDL V60–V67) — [`docs/04-architecture/deployment-management-data-model.md`](../04-architecture/deployment-management-data-model.md)
4. **Architecture** (component breakdown, Decisions D1–D19, ownership rules, non-functional constraints) — [`docs/04-architecture/deployment-management-architecture.md`](../04-architecture/deployment-management-architecture.md)
5. **Data flow** (sequence diagrams, state machines, error cascade, refresh strategy) — [`docs/04-architecture/deployment-management-data-flow.md`](../04-architecture/deployment-management-data-flow.md)
6. **Design** (package layout, DB naming decisions, projection boundaries) — [`docs/05-design/deployment-management-design.md`](../05-design/deployment-management-design.md)
7. **Task breakdown** — [`docs/06-tasks/deployment-management-tasks.md`](../06-tasks/deployment-management-tasks.md) § Phase B
8. **Requirements** (REQ-DP-*) — [`docs/01-requirements/deployment-management-requirements.md`](../01-requirements/deployment-management-requirements.md)
9. **Project conventions** — [`CLAUDE.md`](../../CLAUDE.md) — especially Lesson #2 (Java 21), #3 (package-by-feature), #4 (Flyway only)
10. **Existing BE slices to mirror** — study `backend/src/main/java/com/sdlctower/domain/codebuildmanagement/` first (closest analog: read-only observability, CI/CD adapter, signed webhook receiver, outbox pattern, Story-Id trace chain, AI autonomy gating, per-card section results). Also look at `.../dashboard/`, `.../incident/`, `.../aicenter/` for shared infra (`shared/dto/ApiResponse.java`, `shared/exception/GlobalExceptionHandler.java`, `platform/workspace/`, `shared/audit/`, `shared/lineage/`, `shared/ai/AiSkillClient.java`).

## Existing Backend (DO NOT break)

- `shared/dto/ApiResponse.java`, `shared/dto/SectionResultDto.java` — response envelopes (reuse as-is)
- `shared/exception/GlobalExceptionHandler` — extend with Deployment error code handlers
- `platform/workspace/` — `WorkspaceContext`, `X-Workspace-Id` propagation (reuse exactly)
- `shared/audit/AuditLogEmitter`, `shared/lineage/LineageEmitter`, `shared/ai/AiSkillClient` — reuse
- `shared/integration/requirement/StoryLookup` — already in place (returns story chips)
- `shared/integration/designmanagement/WorkspaceAutonomyLookup` — already in place
- `shared/integration/codebuild/CodeBuildFacade` — **critical upstream dependency**: Deployment Management consumes `resolveCommits(applicationId, releaseVersion)`, `resolveLinkedStories(releaseId)`, `commitRangeBetween(priorHeadSha, headSha)`, `listCommitsForRelease(releaseId)`, `listStoryIdsForRelease(releaseId)`, and the new internal endpoint on Code & Build `/internal/release/{releaseId}/commit-slice`. If the facade interface does not yet have all these methods, add them on the Code & Build side first (non-breaking additions) before this slice depends on them.
- `config/CorsConfig.java` — CORS for Vite (do not modify)
- Flyway migrations in `src/main/resources/db/migration/` — your new files use versions **V60–V67** exactly as specified in the data model
- Existing tests in `src/test/` must continue to pass

## Tech Stack (unchanged)

| Layer | Technology |
|---|---|
| Framework | Spring Boot 3.x (Java 21) |
| Build | Maven |
| ORM | JPA / Hibernate |
| DB (local) | H2 in-memory (profile `local`) |
| DB (prod) | Oracle (profile `prod`) |
| Migrations | Flyway (no `ddl-auto: update`) |

## Scope (Phase B)

Build the backend for Deployment Management under `backend/src/main/java/com/sdlctower/domain/deploymentmanagement/` per the package layout in `deployment-management-design.md` §2. Deliverables:

### 1. Flyway migrations (exact versions V60–V67)

Per data-model §5 DDL verbatim:

- `V60__create_jenkins_instance_and_application.sql`
- `V61__create_application_environment.sql`
- `V62__create_release.sql`
- `V63__create_deploy_and_deploy_stage.sql` — includes `uq_deploy_jenkins UNIQUE (jenkins_instance_id, jenkins_job_full_name, jenkins_build_number)` as the idempotency key
- `V64__create_approval_event.sql` — `rationale_cipher CLOB` (encrypted at rest)
- `V65__create_ai_release_notes_and_summary.sql` — evidence hash columns
- `V66__create_deployment_change_log_and_outbox.sql` — change log + `deployment_ingestion_outbox` (unique `delivery_id`) + `jenkins_backfill_checkpoint`
- `V67__seed_deployment_local.sql` — under `db/seed/` only for `local` and `dev` profiles, NEVER `prod`. Seed: 1 Jenkins instance, 3 applications across 2 projects, 4 envs each, 10 releases, 30 deploys across states, 6 approval events, 4 AI release notes rows (SUCCESS / PENDING / FAILED / EVIDENCE_MISMATCH), 4 AI deploy summaries

### 2. Entities (12 net-new, no Lombok)

Per data-model §3: `JenkinsInstanceEntity`, `ApplicationEntity`, `ApplicationEnvironmentEntity`, `ReleaseEntity`, `DeployEntity`, `DeployStageEntity`, `ApprovalEventEntity`, `AiReleaseNotesEntity`, `AiDeploymentSummaryEntity`, `DeploymentChangeLogEntity`, `DeploymentIngestionOutboxEntity`, `JenkinsBackfillCheckpointEntity`.

Map `@Lob`-heavy fields (`ai_release_notes.body_markdown`, `ai_deployment_summary.narrative`, `approval_event.rationale_cipher`, `deployment_ingestion_outbox.raw_body`) as `FetchType.LAZY`.

### 3. Repositories

One per entity with query methods listed in `deployment-management-tasks.md` § B12. Key ones:

- `DeployRepository.findByJenkinsInstanceIdAndJenkinsJobFullNameAndJenkinsBuildNumber(...)` — idempotency lookup (paired with `uq_deploy_jenkins`)
- `DeployRepository.findLastSuccessByApplicationAndEnvironment(...)` — change summary + rollback comparison
- `DeploymentIngestionOutboxRepository.findByDeliveryId(deliveryId)` + `lockAndClaimOldest(batch)` — outbox worker drain with row lock

All cross-slice reads go through facades — no direct JPA association from Deployment Management to `build_artifact`, `commit`, `story`, `member`, `project`.

### 4. DTOs (Java records)

Per data-model §4. camelCase JSON. Match the frontend `types/aggregate.ts` byte-for-byte (this is the hand-off point that makes the golden-file test pass).

### 5. Projections (per-card)

One projection class per card listed in design doc §2. Each returns `SectionResult<T>`. Each wraps data access in a **500ms timeout** (`CompletableFuture` + `orTimeout`) and converts timeout to `SectionResult.error("DP_TIMEOUT", "...")`. Per-card isolation is non-negotiable — a failing projection MUST NOT blank neighbors (REQ-DP-* and architecture §Non-functional).

### 6. Read services

- `CatalogService`, `ApplicationDetailService`, `ReleaseDetailService`, `DeployDetailService`, `EnvironmentDetailService`, `TraceabilityService`
- Each enforces `DeploymentAccessGuard` at entry (never inside projections)
- `TraceabilityService.inverseLookup` composes Release + Deploy rows from `CodeBuildFacade.storyCommitReleases(storyId)` + local deploy index; cap at 100 with `capNotice`

### 7. Command services (AI regenerate)

- `AiReleaseNotesService.enqueueRegeneration(releaseId, principal)` — role + autonomy + debounce 60s keyed on `releaseId`
- `AiReleaseNotesService.regenerateWorkspaceSummary(workspaceId, principal)` — token bucket 1/min, burst 1
- `AiDeploymentSummaryService.enqueueRegeneration(deployId, principal)` — role + autonomy + debounce 30s keyed on `deployId`; returns `status=SKIPPED` when change-summary.commitCount == 0
- AI calls are async fire-and-forget: PENDING row written immediately; completion writes SUCCESS / FAILED / EVIDENCE_MISMATCH
- All AI results routed through `ReleaseNotesEvidenceValidator` / `DeploymentSummaryEvidenceValidator`; mismatches persisted as `EVIDENCE_MISMATCH` and NEVER surface as user-facing errors

### 8. Policies / helpers

- `DeploymentAccessGuard` (workspace + project role)
- `AiAutonomyPolicy` (DISABLED / OBSERVATION / SUPERVISED / AUTONOMOUS — regenerate requires `>= SUPERVISED`)
- `ApprovalRedactionPolicy` (strips `rationale` and `approver` display name when principal lacks `deployment:view-approvals`)
- `ReleaseNotesEvidenceValidator` / `DeploymentSummaryEvidenceValidator`
- `LogRedactor` (AWS key id, GitHub tokens, `Bearer …`, `(?i)token[= ]+[A-Za-z0-9]{20,}` Jenkins tokens; applied at ingestion on `deploy_stage.log_excerpt.text`; constant-time; idempotent)
- `ReleaseVersionResolver` — lazy-creates `release` rows keyed on `(applicationId, releaseVersion)`; defers with `DP_RELEASE_UNRESOLVED` when `RELEASE_VERSION` absent on Jenkins payload (API guide §5.2)
- `RollbackDetector` — dual-signal: `trigger=ROLLBACK` OR `releaseVersion < priorSuccessfulDeployReleaseVersion` on same env; persists `rollback_detection_signal` on the deploy row (API guide §5.3)
- `ReleaseVersionComparator` — dot-segmented numeric with lexicographic fallback; ties break on `created_at`

### 9. Jenkins integration

- `JenkinsRestClient` (read-only; circuit breaker + retry with jitter; returns `DP_JENKINS_UNREACHABLE` on failure) — used only on backfill and IN_PROGRESS stage fill
- `JenkinsAuth` — resolves per-instance API token from encrypted storage; never logged
- `JenkinsSignatureVerifier` — HMAC-SHA-256; constant-time compare; secret resolved from `JenkinsInstance` row keyed on `X-Jenkins-Instance` header
- `JenkinsPayloadParser` — normalizes Notification-Plugin JSON to internal `TypedEvent` enum

### 10. Ingestion pipeline

- `JenkinsWebhookController.POST /webhooks/jenkins` — sync-verify, persist to outbox, return 202. Never returns the envelope — Jenkins expects an empty body (API guide §5.1)
- `IngestionDispatcher` — reads outbox, routes by `TypedEvent` per tasks doc § B8:
  - `job.started` → `ReleaseVersionResolver.resolveOrCreate` → upsert `deploy` row (state=IN_PROGRESS, trigger resolved from Jenkins metadata)
  - `job.completed` → update `deploy` state (`SUCCEEDED` / `FAILED` / `CANCELLED`); `UNSTABLE` → SUCCEEDED with led=AMBER; run `RollbackDetector`
  - `stage.completed` → append `deploy_stage` row with conclusion + duration; apply `LogRedactor` if log excerpt attached
  - `input.approved|rejected|timedOut` → append `approval_event` row with encrypted rationale
  - `build.promoted` → log-only (V1)
  - `build.deleted` → soft-hide (set `hidden_at`)
  - Unknown event type → mark DONE forward-compatible; never FAILED
- Idempotency: duplicate `deliveryId` → outbox no-op; duplicate `(instance, job, build)` → `uq_deploy_jenkins` rejects

### 11. Backfill + nightly resync

- `JenkinsBackfillService.backfill(jenkinsInstanceId)` — triggered on instance registration + webhook-drop detection; 14-day window; goes through the same event handlers (never direct DB writes); progress persisted in `jenkins_backfill_checkpoint`
- `NightlyResyncJob` — `@Scheduled(cron = "0 15 2 * * *")` UTC; 72h reconciliation window; delta routed through handlers; emits `ResyncRun` to change log

### 12. Controller

`DeploymentController` with all endpoints per API guide §2. Path validation via `@Pattern`. Standard `ApiResponse<T>` wrap. Mutation endpoints return 200 with updated state. Internal facade endpoints (`/internal/story/{storyId}/deploy-aggregate`, `/internal/release/{releaseId}/commit-slice`) behind `X-Service-Token` filter.

### 13. Governance / lineage / cross-slice facade

- `DeploymentHealthFacade` — cross-slice read facade for Dashboard + Project Management: `healthByProject`, `healthByWorkspace`, `recentRollbacksByWorkspace`
- Every regenerate writes `AuditLogEntry` + `LineageEvent`
- Every AI call emits `SkillInvocation` (invoker, autonomy level, skill version, latency, outcome, evidence integrity outcome)
- Incident deep-link handshake: verify `context` object `{ deployId, releaseVersion, environmentName, failingStageId, summary }` is accepted by Incident slice's create form

### 14. Tests (comprehensive — see tasks doc § B14)

- `DeploymentControllerTest` (MockMvc) — every endpoint happy + 403/404, per-projection isolation, approval role-redaction, traceability with no deploys
- `JenkinsWebhookControllerTest` (MockMvc) — valid/invalid signature, unknown instance, malformed payload, duplicate `deliveryId`, duplicate `(instance, job, build)`, unknown event type (202 + DONE)
- `IngestionDispatcherTest` — one test per event type
- `ReleaseVersionResolverTest`, `RollbackDetectorTest` (both signals), `ApprovalRedactionPolicyTest`, `ReleaseNotesEvidenceValidatorTest`, `DeploymentSummaryEvidenceValidatorTest`, `LogRedactorTest`
- `AiReleaseNotesServiceTest`, `AiDeploymentSummaryServiceTest` (including SKIPPED path)
- `DeploymentAccessGuardTest`, `JenkinsBackfillServiceTest`, `NightlyResyncJobTest`
- Repository tests for every entity + edge queries (idempotency, rollback comparator, metrics windowing)
- `FlywayMigrationIntegrationTest` — V60–V67 apply cleanly on H2; `uq_deploy_jenkins` rejects duplicates
- Golden-file test — compare API responses to canonical examples in API guide §3 byte-for-byte (including redacted-approvals shape for non-privileged principal)
- Webhook signature fixture test — HMAC pass/fail with constant-time compare

### 15. Frontend wire-up

- Set `VITE_USE_BACKEND=true` in dev config
- Verify Vite proxy routes `/api/v1/deployment-management/*` to backend
- Run the full smoke-test matrix per tasks doc § B15 + B16

## Package Structure

```
backend/src/main/java/com/sdlctower/domain/deploymentmanagement/
├── controller/
│   ├── DeploymentController.java
│   └── JenkinsWebhookController.java
├── service/                 (Catalog/App/Release/Deploy/Env/Traceability + AiReleaseNotes + AiDeploymentSummary)
├── ingestion/               (SignatureVerifier, PayloadParser, Dispatcher, ResyncScheduler, BackfillService, OutboxWorker, ReleaseVersionResolver, RollbackDetector)
├── projection/              (one per card)
├── policy/                  (AccessGuard, AiAutonomyPolicy, ApprovalRedactionPolicy, EvidenceValidators, LogRedactor)
├── integration/             (JenkinsRestClient, JenkinsAuth, CodeBuildFacade consumer, RequirementFacade consumer, AiSkillClient consumer)
├── persistence/{entity,repository,converter}
├── dto/                     (all records)
└── events/                  (DeploymentChangeLogPublisher)
```

Java package is `deploymentmanagement` (no hyphen). Docs and URLs keep `deployment-management`.

## DB Naming Rules

- `release` is an Oracle reserved word — the table is named `release` with quoted identifiers where required OR renamed to `app_release` if quoting is not viable on Oracle. Data-model doc uses `release`; if Oracle's strict mode breaks it, add a per-dialect variant under `db/migration/oracle/` that uses `app_release` with an alias view `CREATE VIEW release AS SELECT * FROM app_release;` OR stick with quoted identifiers. Pick ONE approach and hold it across the migration set.
- All `*_cipher` CLOB columns are populated via an application-layer `AesGcmCipher` component (do NOT push encryption into the DB).
- Use CLOB for markdown and long-text fields; serialize via Jackson in the service layer. No vendor JSON types.

## What NOT To Do

- Do NOT use `ddl-auto: update` or `create-drop` for anything other than throwaway local H2 exploration — all schema lives in Flyway migrations (CLAUDE.md Lesson #4)
- Do NOT put any entities outside `domain/deploymentmanagement/` (CLAUDE.md Lesson #3 — package-by-feature)
- Do NOT have other domains directly access Deployment Management's repositories or entities — they go through `DeploymentHealthFacade` or the public service methods
- Do NOT write to Jenkins from any code path. The `JenkinsRestClient` is READ-ONLY. No POSTs to `/job/.../build`, `/input/…/proceed`, `/promote`, `/doDelete`, `/stop`, or any other mutation endpoint.
- Do NOT add endpoints that trigger / approve / promote / cancel / roll back a Jenkins build. V1 is observability-only.
- Do NOT add pre-deploy AI risk assessment, failure triage, or auto-rollback suggestions — AI scope in V1 is release notes / deploy summary only
- Do NOT persist a Story↔Deploy link table. Traceability is resolved read-side via `CodeBuildFacade`. Code & Build is the single source of truth for Story↔Commit.
- Do NOT create a story registry or story-lookup table in this slice — reuse `StoryLookup` from Requirement.
- Do NOT use Lombok — standard records / getters/setters
- Do NOT skip HMAC signature verification on the webhook endpoint, even for local development. The verifier is constant-time.
- Do NOT log raw webhook bodies, Jenkins API tokens, approval rationale, or `rationale_cipher` contents. Any log line that might carry them goes through `LogRedactor`.
- Do NOT include `db/seed/V67__seed_deployment_local.sql` in `prod` Flyway locations. Guard via `spring.flyway.locations` profile-switching.
- Do NOT modify existing shell / dashboard / code-build / requirement / incident code except: (a) add exception handler methods to the existing `GlobalExceptionHandler`, (b) add the new `CodeBuildFacade` method signatures on the Code & Build side (non-breaking), (c) wire frontend to live API
- Do NOT break existing tests
- Do NOT raise `DP_RELEASE_NOTES_EVIDENCE_MISMATCH` as an HTTP error — it is persisted as an `AiRowStatus.EVIDENCE_MISMATCH` on the `ai_release_notes` row and surfaces to the user as a banner, not a 4xx
- Do NOT skip the 500ms per-projection timeout. It is the mechanism that makes per-card isolation real.

## Acceptance Criteria

- [ ] `./mvnw clean verify` passes all tests (existing + new)
- [ ] `./mvnw spring-boot:run -Dspring.profiles.active=local` starts cleanly
- [ ] All read endpoints return JSON matching API guide §3 examples byte-for-byte (verified by golden-file tests)
- [ ] `POST /api/v1/deployment-management/webhooks/jenkins` with invalid signature → 401 `DP_INGEST_SIGNATURE_INVALID`
- [ ] `POST /api/v1/deployment-management/webhooks/jenkins` with valid signature → 202 + outbox row created; duplicate `deliveryId` → 202 + no second outbox row; duplicate `(instance, job, build)` → handler no-op via `uq_deploy_jenkins`
- [ ] `GET /api/v1/deployment-management/applications/not-a-real-app` → 404 `DP_APPLICATION_NOT_FOUND`
- [ ] `GET /api/v1/deployment-management/releases/not-a-real-release` → 404 `DP_RELEASE_NOT_FOUND`
- [ ] `POST /api/v1/deployment-management/releases/{id}/ai-notes/regenerate` without role → 403 `DP_ROLE_REQUIRED`; with role but workspace autonomy=DISABLED → 409 `DP_AI_AUTONOMY_INSUFFICIENT`; twice within 60s → 429 `DP_RATE_LIMITED`
- [ ] Approval rationale + approver redaction: Developer principal sees `(redacted)` on both fields; PM / Tech Lead / Release Manager see actual values (verified by golden-file test with two principal variants)
- [ ] Dual-signal rollback detection: explicit `trigger=ROLLBACK` → positive with `rollback_detection_signal=trigger=rollback`; version-older-than-prior → positive with `rollback_detection_signal=version-older-than-prior`; same version re-deploy → negative
- [ ] Release row lazy creation: `job.started` with new `RELEASE_VERSION` → `release` row created (state=PREPARED); `job.started` without `RELEASE_VERSION` → outbox entry deferred with `DP_RELEASE_UNRESOLVED`, retried on backfill
- [ ] Evidence mismatch: AI skill returns a commit sha not in `CodeBuildFacade.listCommitsForRelease(releaseId)` → persisted as `AiRowStatus.EVIDENCE_MISMATCH`, banner rendered, no HTTP error
- [ ] Deploy summary SKIPPED: change-summary.commitCount == 0 → persisted as `AiRowStatus.SUCCESS` with `status=SKIPPED` in the payload
- [ ] Workspace isolation: seeding `WS-A` and `WS-B` then requesting `WS-A` returns no `WS-B` rows on any endpoint
- [ ] Flyway V60–V67 migrations apply cleanly from empty schema on H2 (via `FlywayMigrationIntegrationTest`); `uq_deploy_jenkins` rejects duplicate inserts; `release` unique constraint `(application_id, release_version)` holds
- [ ] Seed migration populates realistic demo data matching Phase A mock counts under `local` profile; does NOT run under `prod`
- [ ] Per-projection 500ms timeout verified: injected 1-second sleep in one projection returns `SectionResultDto(error=...)` without blanking neighbors
- [ ] Jenkins freshness SLO: happy-path webhook → `last_ingested_at` within 45s P95 of Jenkins timestamp (smoke test on seed data)
- [ ] Backfill idempotency: re-running `JenkinsBackfillService.backfill(instanceId)` produces zero new deploy rows
- [ ] Nightly resync: simulated drift (delete a local deploy row) → next resync re-hydrates without duplicate change-log entries
- [ ] `DeploymentHealthFacade` consumed by Dashboard + Project Management without regression to those slices' tests
- [ ] Incident deep-link handshake: `context` object carries `deployId`, `releaseVersion`, `environmentName`, `failingStageId`, `summary`; Incident slice pre-fills its create form; no write to Jenkins
- [ ] Frontend `npm run dev` + `npm run build` still succeed; `/deployment` now renders live backend data end-to-end with all six views
- [ ] No ESLint, `tsc`, Spotless / Checkstyle warnings introduced
- [ ] Cross-domain service methods and facade documented in JavaDoc
- [ ] Audit log contains entries for every regenerate; Lineage graph identifies Deployment Management as authoring domain; SkillInvocation records include autonomy level + evidence outcome
- [ ] Write authority for all 12 Deployment Management entities is demonstrably owned by this slice — no other slice writes to `jenkins_instance`, `application`, `application_environment`, `release`, `deploy`, `deploy_stage`, `approval_event`, `ai_release_notes`, `ai_deployment_summary`, `deployment_change_log`, `deployment_ingestion_outbox`, `jenkins_backfill_checkpoint`
