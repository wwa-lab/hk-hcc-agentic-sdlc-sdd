# Deployment Management — Spec

## Source

| Field | Value |
| ----- | ----- |
| Source Requirements | [deployment-management-requirements.md](../01-requirements/deployment-management-requirements.md) |
| Source Stories | [deployment-management-stories.md](../02-user-stories/deployment-management-stories.md) |
| PRD clause | §11.9, §18.1 |
| Status | Draft v1 (2026-04-18) |

## 1. Overview

The Deployment Management slice is a **read-only observability viewer** for Jenkins-orchestrated deployments, with Story↔Commit↔Build↔Deploy traceability and advisory AI (workspace summary and per-release AI release-notes). The slice does not mutate Jenkins state, does not trigger or re-run Jenkins jobs, does not approve or reject release gates, and does not execute rollbacks. AI output is advisory; AI never has approval or rollback authority.

## 2. Routing

| Path | View | Notes |
| ---- | ---- | ----- |
| `/deployment` | `CatalogView` | Workspace-scoped application catalog + AI summary |
| `/deployment/applications/:applicationId` | `ApplicationDetailView` | 6-card application detail |
| `/deployment/releases/:releaseId` | `ReleaseDetailView` | Release metadata + composition + AI release notes |
| `/deployment/deploys/:deployId` | `DeployDetailView` | Deploy stage timeline + approvals |
| `/deployment/applications/:applicationId/environments/:environmentName` | `EnvironmentDetailView` | Per-environment history + metrics |
| `/deployment/traceability` | `TraceabilityView` | Inverse lookup by Story-Id (Story → Release → Deploy) |

All paths are guarded by `requireWorkspaceMember(workspaceId)`; workspace is resolved from the path parameter's owning entity (application→project→workspace, release→application→project→workspace, deploy→release→application→project→workspace).

No SPA route for `/api/v1/deployment-management/**` — those are backend-only.

## 3. Functional Scope

Numbered subsystems. Each maps to one or more projections / services in the architecture doc.

1. **Catalog Summary** — aggregate counts + 7-day rollups across visible applications.
2. **Catalog Grid** — grouped list of application tiles with per-environment "current revision" chips and filters + search.
3. **Catalog AI Summary** — workspace-scoped narrative + evidence chips.
4. **Application Header** — identity, owning project, runtime label, Jenkins folder deep-link.
5. **Application Environments** — per-environment current revision, state, rollback chip, env-detail deep-link.
6. **Application Recent Releases** — up to 20 most-recent releases with version, state, story count.
7. **Application Recent Deploys** — up to 30 most-recent deploys across all environments with state, approver, "current revision" chip.
8. **Application Trace Summary** — aggregate story-count per environment in the last 30 days.
9. **Application AI Insights** — latest narrative summary for this application.
10. **Release Metadata** — version, build ref, state, created-at, release-notes source.
11. **Release Composition** — commits (resolved via BuildArtifact) + linked stories (verified against Requirement slice).
12. **Release Deploys** — every deploy that carried this release, grouped by environment.
13. **Release AI Notes** — AI-generated narrative + since-previous-prod diff + evidence chips.
14. **Deploy Header** — release version, environment, Jenkins job + build number, trigger, actor, duration, conclusion.
15. **Deploy Stage Timeline** — read-only metadata-only timeline (stage, state, duration, approver).
16. **Deploy Approvals** — approval records (approver, timestamp, decision, gate policy version, rationale for privileged roles).
17. **Deploy Artifact Ref** — artifact-metadata-only reference pointing to Code & Build slice.
18. **Open Incident From Deploy** — navigation action to Incident slice with pre-filled context.
19. **Environment Detail** — current + prior + last-good + last-failed revisions, 30-day CFR, 30-day MTTR, 90-day deploy timeline.
20. **Traceability Inverse View** — Story-Id → Releases → Deploys, grouped by environment.
21. **Requirement Slice Deploys Tab** — the same inverse view embedded into Story Detail.
22. **Code & Build Slice Deploys Panel** — build-artifact → deploys, embedded into Run Detail.
23. **Jenkins Webhook Ingestion** — verify, parse, upsert, emit domain events.
24. **Jenkins Backfill / Resync** — 30-day install backfill + 72h nightly reconciliation.
25. **AI Autonomy Gating** — observes workspace `aiAutonomyLevel` for every AI-invoking action.

## 4. Behavior Rules

- **B1** — Release version discovery is parameter-based (`RELEASE_VERSION` Jenkins parameter) with a pattern-based fallback (`Release {version}` stage annotation). Neither present → `RELEASE_UNRESOLVED` flag on the deploy row and the deploy is surfaced in a diagnostics view rather than mapped to a Release.
- **B2** — Releases are always created server-side when the first deploy ingesting a given (applicationId, releaseVersion) lands; subsequent deploys of the same version attach to the existing Release. No explicit "create release" step.
- **B3** — `buildArtifactRef` on a Release is the join key into Code & Build. The Deployment slice holds only the reference (slice id + artifact id) and resolves the details on read.
- **B4** — Story ↔ Deploy linkage is always derived read-side (Story → StoryCommitLink → Commit → BuildArtifact → Release → Deploy). The slice does NOT persist a direct Story↔Deploy link table; this preserves Code & Build as the single source of truth for Story↔Commit mapping.
- **B5** — A deploy is classified as ROLLBACK when `trigger=rollback` OR when the target release version is chronologically earlier (by release-creation timestamp) than the environment's prior-revision release version. Rollback deploys participate in Change Failure Rate calculations.
- **B6** — AI release notes are keyed by (releaseId, skillVersion). Stale notes (build SHA advanced after generation — corruption case) render a STALE badge and the prior note is kept for audit.
- **B7** — AI release notes are rejected at service time if they reference a story ID not in the release's resolved story set (evidence-integrity check); the frontend receives `DP_RELEASE_NOTES_EVIDENCE_MISMATCH` and renders a placeholder with an admin-only Regenerate.
- **B8** — AI release notes are NOT invoked when the release's BuildArtifact is unresolved (REQ-DP-51 / REQ-DP-65). The AI client refuses the request and returns an empty `PENDING_UPSTREAM` result.
- **B9** — AI never writes to Jenkins, Git, or any external tracker. No write-scope Jenkins token is issued for this subsystem. Enforced at the network layer.
- **B10** — Workspace AI summary is regenerated at most every 15 minutes; manual regen is admin-only and rate-limited to 1/min.
- **B11** — Jenkins webhook signatures are verified with the per-Jenkins-instance shared secret; invalid signature → 401 + audit + no mutation.
- **B12** — Workspace isolation is enforced at the application level; every read projection resolves `deploy → release → application → project → workspace` (or the equivalent partial chain) and filters against the caller's workspace membership.
- **B13** — Email addresses are never rendered in the UI; only display names. Privileged roles (Release Manager / Tech Lead / PM) see approver rationale; other readers see decision + timestamp only.
- **B14** — AI-generated content (summaries, release notes) never carries approval or rollback authority; actions triggered from AI output (Open Incident) pass through explicit user consent.
- **B15** — `UNSTABLE` Jenkins stage status is rendered AMBER in the UI, distinct from both GREEN (`SUCCESS`) and RED (`FAILURE`).

## 5. Error Codes

All errors follow the shared envelope: `{ error: { code, message, details? }, meta }`.

| Code | HTTP | When |
| ---- | ---- | ---- |
| `DP_WORKSPACE_FORBIDDEN` | 403 | Caller lacks membership in the workspace resolved from the path |
| `DP_ROLE_REQUIRED` | 403 | Action requires PM / Tech Lead / Release Manager (e.g. approver rationale, AI regen) |
| `DP_APPLICATION_NOT_FOUND` | 404 | `applicationId` does not exist or is outside workspace scope |
| `DP_RELEASE_NOT_FOUND` | 404 | `releaseId` does not exist or outside scope |
| `DP_DEPLOY_NOT_FOUND` | 404 | `deployId` does not exist or outside scope |
| `DP_ENVIRONMENT_NOT_FOUND` | 404 | Environment name not configured for this application |
| `DP_STORY_NOT_FOUND` | 404 | Traceability lookup story does not resolve |
| `DP_AI_AUTONOMY_INSUFFICIENT` | 409 | Workspace autonomy level below required floor |
| `DP_AI_UNAVAILABLE` | 503 | AI skill client failure / timeout (retryable) |
| `DP_RATE_LIMITED` | 429 | Too many requests (especially AI regen or traceability scan) |
| `DP_JENKINS_UNREACHABLE` | 503 | Upstream Jenkins unreachable or throttling |
| `DP_INGEST_SIGNATURE_INVALID` | 401 | Webhook signature verification failed |
| `DP_INGEST_PAYLOAD_INVALID` | 400 | Webhook payload fails parsing or schema validation |
| `DP_JENKINS_INSTANCE_UNKNOWN` | 404 | Webhook references an unknown Jenkins instance id |
| `DP_RELEASE_NOTES_EVIDENCE_MISMATCH` | 422 | AI release notes referenced stories outside the release's resolved set |
| `DP_BUILD_ARTIFACT_PENDING` | 409 | Upstream BuildArtifact not yet resolvable (retryable) |
| `DP_RELEASE_UNRESOLVED` | 422 | Jenkins build did not carry a `RELEASE_VERSION` parameter or matching stage pattern |

## 6. Decisions

- **D1** — Observability-only in V1 (no triggering, no approvals, no rollback execution, no Jenkinsfile edits). Documented in REQ §3 and in architecture §Decisions. Rationale: scope matches PRD §18.1 without creating write-authority ambiguity against Jenkins or the deploy target.
- **D2** — Integration surface limited to Jenkins. Other CD systems (Argo CD, Spinnaker, Harness, Octopus, GitHub Actions CD) are deferred. Rationale: single adapter dramatically simplifies ingestion, auth, and webhook handling for V1; adapter interface is designed to permit future adapters.
- **D3** — Primary traceability is Story → Commit → Build → Deploy, extending the Code & Build chain forward. Rationale: continuity of the existing chain means no duplicated link tables and a single source of truth.
- **D4** — AI scope for V1 is release-notes / deploy summary only. Pre-deploy risk assessment, failure triage, rollback recommendation, auto-approve / auto-rollback are all deferred. Rationale: narrowest AI authority surface that is still clearly valuable to PMs and release managers; governance boundary is simple to reason about.
- **D5** — Story ↔ Deploy linkage is always derived read-side, never persisted. Rationale: Code & Build owns Story↔Commit; persisting a second table would create dual-writes and drift risk.
- **D6** — Webhook-driven ingestion with 30-day install backfill and 72h nightly resync. No polling-only mode. Rationale: freshness SLO requires webhooks; resync protects against dropped events without masking divergence.
- **D7** — Jenkins API tokens live in Platform Center vault; Control Tower DB never stores them. Rationale: security boundary; Platform Center handles rotation.
- **D8** — AI release notes never write to Jenkins, Git, or any external tracker. No write-scope Jenkins token for this subsystem. Rationale: governance simplicity and reversibility; humans can forward AI notes manually.
- **D9** — AI output keyed by (releaseId, skillVersion); rotated on build SHA advance or skill version bump. Prior rows kept as SUPERSEDED/STALE for audit. Rationale: explainability and reproducibility.
- **D10** — Release rows are created lazily on first deploy of a new `RELEASE_VERSION`; no standalone "create release" API. Rationale: Jenkins is the source of truth for release existence; standalone creation would bifurcate the workflow.
- **D11** — `UNSTABLE` Jenkins state is rendered AMBER, not GREEN. Rationale: Jenkins semantics (tests failed but build continued) are meaningful to engineers and must not be flattened.
- **D12** — Email addresses never rendered; display names only. Approver rationale surfaced only to PM/Tech Lead/Release Manager. Rationale: privacy-by-default + least-privileged visibility of approver attribution.

## 7. Dependencies

| Dependency | Type | Use |
| ---------- | ---- | --- |
| Code & Build slice `BuildArtifactLookup` facade | read | BuildArtifact resolution, commit range, inverse build-→-deploys lookup |
| Requirement slice `StoryLookup` facade | read | Story-Id verification, inverse story-→-deploys lookup |
| Platform Center secret vault | read | Jenkins API tokens, webhook secrets per instance |
| Workspace service | read | Workspace membership check |
| Project service | read | Application→project→workspace resolution |
| Member service | read | Approver display-name resolution |
| Shared `AuditLogEmitter`, `LineageEmitter` | write | Audit + lineage for every write path |
| Shared AI skill runtime (`AiSkillClient`) | call | Workspace summary, per-release release-notes generation |
| Shared shell (`AppShell`, `shellUiStore`) | UI | Breadcrumbs + AI panel |

## 8. Scope boundaries explicitly restated

- No write calls to Jenkins (the Control Tower never triggers, cancels, reruns, promotes, approves, rejects, rolls back, or parameter-edits a Jenkins job).
- No log streaming from Jenkins in V1 — stage timeline is metadata-only.
- No polling-only fallback — if webhooks are unavailable, the slice surfaces a banner and relies on nightly resync.
- No custom CD adapters in V1 beyond Jenkins.
- No AI-authored approvals, rollbacks, Jenkins-build-description posts, or Git tag annotations.
- No persisted Story↔Deploy link table; linkage is always derived.

## 9. Glossary

- **Application** — a deployable unit the Control Tower tracks; typically 1:1 with a Jenkins folder or top-level pipeline.
- **Environment** — a named deploy target for an application (`dev`, `test`, `staging`, `prod`, plus custom).
- **Release** — a versioned artifact bundle identified by `releaseVersion` and bound to a single `BuildArtifact`; one Release is deployed N times across environments.
- **Deploy** — a single execution of the Jenkins deploy pipeline delivering one Release to one Environment.
- **Stage** — a Jenkins Pipeline stage; the timeline is rendered stage-by-stage in Deploy Detail.
- **Approval Gate** — a Jenkins `input` step that requires a human decision; every approval event is captured with approver, timestamp, decision, and gate policy version.
- **Rollback** — a deploy whose `trigger=rollback` or whose target release version is older than the environment's prior revision.
- **Current Revision** — the most recent `SUCCEEDED` deploy of an environment (counting rollbacks).
- **Last-Good Revision** — the most recent `SUCCEEDED` deploy of an environment that is NOT currently reverted by a later rollback.
- **Last-Failed Revision** — the most recent `FAILED` deploy of an environment.
- **Change Failure Rate (CFR)** — over a 30-day window, (`FAILED` + `ROLLED_BACK` deploys) / total deploys for the environment.
- **MTTR** — mean time from a `FAILED` deploy to the next `SUCCEEDED` deploy in the same environment, over a 30-day window.
- **Build Artifact Ref** — `{ sliceId: "code-build", buildArtifactId: "BA-…" }`; the join key into Code & Build.
- **Release Version** — a human-readable version string (e.g. `v2.18.3`); resolved from Jenkins parameter or stage pattern.
