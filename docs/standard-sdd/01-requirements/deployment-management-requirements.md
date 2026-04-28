# Deployment Management Requirements

## 1. Context and Source

Requirements for the **Deployment Management** slice of the Agentic SDLC Control Tower. Derived from:

- PRD §11.9 (Deployment Management page definition — why can we ship, who approved, what did AI do, how is risk controlled)
- PRD §18.1 (V1 must-have scope — Deployment is included in the V1 must-have bucket)
- PRD §6.4 / §6.5 / §7 (traceability chain, AI platform, autonomy levels)
- Upstream slice: [code-build-management-requirements.md](./code-build-management-requirements.md) — Deployment consumes Build artifacts and extends the Story→Commit→Build chain forward to Deploy
- Scope decisions captured via the slice-kickoff questionnaire on 2026-04-18:
  - V1 is an **observability-first viewer** (read-only, no triggering, no approvals, no rollback execution)
  - Integration surface is limited to **Jenkins** (GitHub Actions CD, Argo CD, Spinnaker, Harness, Octopus deferred)
  - Primary traceability is **Story → Commit → Build → Deploy** (extends the Code & Build chain)
  - AI scope is **release-notes / deploy summary only** (no pre-deploy risk assessment, no failure triage, no auto-approve/rollback in V1)

## 2. Goal

Give PMs, Tech Leads, Release Managers, SREs, and Engineers a single page to understand, at a glance:

1. **What was released** — release versions per application, with their composition (build artifact, commits, stories), their lifecycle, and who approved them
2. **Where it ran** — per-environment deployment history (dev / test / staging / prod), current environment state, last-good revision, and last-failed revision
3. **How that deploy traces back** — every Deployment traces to a BuildArtifact, which traces through commits to user stories; the page makes the full Story → Commit → Build → Deploy chain visible and inverse-searchable
4. **Who approved it and why the gate passed** — the approval record for each deploy (approver, timestamp, evidence, gate policy version)
5. **What AI observed** — a generated, human-readable release-notes / deploy summary narrating what is in a deploy and how it compares to the previous revision in the same environment
6. **Where to dig deeper** — deep-links into the Code & Build slice (for the originating build), the Requirement slice (for each linked story), the Incident slice (for incidents correlated with this deploy window), and the upstream Jenkins console page

Non-goals for V1: triggering Jenkins jobs, approving or rejecting deployment gates, executing or reverting rollbacks, editing Jenkinsfiles, configuring environments or credentials, running blue/green or canary cutovers, mutating the deployment target (K8s, VM, serverless) in any way.

## 3. Scope Boundaries

| In scope (V1)                                                                                     | Out of scope (V1; revisit in V1.1+)                                         |
| ------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------- |
| Read-only Catalog: applications + recent release activity per project/workspace                   | Creating applications or environments from the Control Tower                |
| Read-only Application Detail: environments, recent releases, recent deploys, AI insights          | Triggering, rerunning, or cancelling Jenkins CD jobs                        |
| Read-only Release Detail: release metadata, composition (build + commits + stories), deploys      | Approving / rejecting release gates from Control Tower                      |
| Read-only Deploy Detail: stage timeline, step outcomes, approvals, artifact reference             | Executing rollbacks or blue/green / canary cutovers                         |
| Read-only Environment Detail: current revision, history, last-good / last-failed revision         | Editing Jenkinsfiles, environment config, credentials, quality gates        |
| Story ↔ Commit ↔ Build ↔ Deploy traceability via the upstream BuildArtifact                       | Inferring deploy→story links from deploy description heuristics             |
| AI release-notes / deploy summary (advisory narrative — human consumption only)                   | AI pre-deploy risk assessment (deferred to V1.1)                            |
| AI release-notes / deploy summary (advisory narrative — human consumption only)                   | AI failure triage / rollback recommendation (deferred to V1.1)              |
| AI release-notes / deploy summary (advisory narrative — human consumption only)                   | AI auto-approve / auto-rollback (explicitly out-of-scope for V1)            |
| Jenkins integration via job/stage webhooks + REST pull                                            | GitHub Actions CD, Argo CD, Spinnaker, Harness, Octopus                     |
| Webhook-driven ingestion with resync/backfill job                                                 | Polling-only mode                                                           |
| Deep-links out to Code & Build (build), Requirement (story), Incident (case), and Jenkins console | Log-streaming from Jenkins into the Control Tower                           |
| Workspace isolation and role-based read access                                                    | Fine-grained per-environment ACL beyond workspace membership                |

## 4. User Personas

- **PM** — wants a workspace-level summary of what shipped this week, to which environments, and whether prod is healthy.
- **Release Manager** — wants per-application release history: which versions exist, which is currently in each environment, which was rolled back, which is pending approval upstream.
- **Tech Lead** — wants per-deploy drill-in: did the deploy succeed, which stage failed, which build did it use, which stories did it carry.
- **SRE / On-call** — wants to correlate an incident time window to the most recent prod deploy of the same application, and reach the AI deploy summary for context.
- **Engineer** — wants to confirm that the commit they care about has reached an environment and see the owning release version.

## 5. Upstream / Downstream Dependencies

- **Upstream:** Code & Build Management (BuildArtifact as the join key into the trace chain), Requirement slice (Story lookup for linked stories), Platform Center (Jenkins credentials vault + webhook secret), Team / Project Space (workspace + project membership, application ownership).
- **Downstream:** Incident slice (deep-linking a failing deploy or a prod incident back to the specific deploy), Dashboard ("recent deployments" tile, "prod change failure rate" metric), future Testing slice (consumes the deploy reference for post-deploy test linkage — V1.1+), future Report Center (release velocity, change failure rate, deployment frequency).

## 6. Requirements

### 6.1 Catalog (Workspace-level view)

- **REQ-DP-01** — Catalog page lists every application visible to the user, grouped by project, with application name, primary runtime label (e.g., `jvm`, `node`, `k8s-service`), last deploy timestamp, the current revision in each tracked environment, and an aggregate health LED.
- **REQ-DP-02** — Catalog shall show an aggregate summary bar: total applications, total releases produced in the last 7 days, total deploys in the last 7 days, deploy success rate (%) over the last 7 days, deployment frequency per application (median), change failure rate (%) over the last 30 days.
- **REQ-DP-03** — Catalog must be filterable by project, environment (dev/test/staging/prod), deploy-status (GREEN/AMBER/RED — where AMBER is "last deploy partially succeeded or was rolled back"), and time window (24h / 7d / 30d).
- **REQ-DP-04** — Catalog must be searchable by application name or full path.
- **REQ-DP-05** — Each application tile shall deep-link to its Application Detail page.
- **REQ-DP-06** — Catalog must respect workspace isolation: an application attached to a project outside the user's workspace membership must not be returned.
- **REQ-DP-07** — Each application tile shall show a lineage badge indicating the upstream Project it belongs to (reusing `<LineageBadge>`).
- **REQ-DP-08** — Each application tile shall show a "current prod revision" chip that is empty (with a hint) when the application has no production environment configured.

### 6.2 Application Detail

- **REQ-DP-10** — Application Detail shall render 6 cards: Header, Environments, Recent Releases, Recent Deploys, Trace Summary, AI Insights.
- **REQ-DP-11** — Header shall show application name, description, owning Project, runtime label, Jenkins folder path, external-link-out to the Jenkins folder, and the last-deploy timestamp across all environments.
- **REQ-DP-12** — Environments card shall list every configured environment for this application (typical set: `dev`, `test`, `staging`, `prod`) with: current revision, current revision's release version, deploy status of current revision, timestamp of current revision, the most recent last-good revision fallback, and a chip indicating whether the environment is currently in a rolled-back state.
- **REQ-DP-13** — Recent Releases card shall list up to 20 most-recent releases with release version, source build reference, created-at timestamp, release state (`PREPARED` / `DEPLOYED` / `SUPERSEDED` / `ABANDONED`), and a count of stories this release carries.
- **REQ-DP-14** — Recent Deploys card shall list up to 30 most-recent deploys across all environments with: release version, environment, state (`PENDING` / `IN_PROGRESS` / `SUCCEEDED` / `FAILED` / `ROLLED_BACK` / `CANCELLED`), started-at, duration, approver (if the deploy passed an approval gate), and a chip indicating whether this deploy resulted in the environment's current revision.
- **REQ-DP-15** — Trace Summary card shall surface the aggregate story-count delivered to each environment in the last 30 days and a link to the Traceability Inverse View pre-filtered to this application.
- **REQ-DP-16** — AI Insights card shall render the latest narrative AI summary for this application (recently-shipped stories, deploy-success-rate trend, notable environment drift between staging and prod).
- **REQ-DP-17** — Every release / deploy row shall include a deep-link out to Jenkins (new tab, `rel="noopener noreferrer"`), resolved against the configured Jenkins base URL for the owning project.
- **REQ-DP-18** — Each card shall independently degrade on error (per-card error state + retry) without failing the page.

### 6.3 Release Detail

- **REQ-DP-20** — Release Detail shall show release metadata (release version, source build artifact reference with deep-link into the Code & Build slice, release state, created-at, created-by, release-notes — AI-generated or Jenkins-provided), a commit list (expanded from the BuildArtifact), a linked-stories list (the union of story IDs extracted from those commits and validated against the Requirement slice), and a deploys-of-this-release list.
- **REQ-DP-21** — Every linked story chip shall deep-link to that story in the Requirement slice; unverified story IDs render as `UNVERIFIED` (grey chip), never hidden.
- **REQ-DP-22** — Release Detail shall render an "AI Release Notes" card — the AI-generated, human-readable narrative describing what is in this release; evidence (commit SHAs, story IDs) shall be chip-linked below the narrative.
- **REQ-DP-23** — AI release notes are **advisory only** — they are never posted upstream to Jenkins, Git, or any external tracker automatically; they render inside the Control Tower's Release Detail page.
- **REQ-DP-24** — AI release notes shall be keyed by (releaseId, skillVersion); regenerating on the same release version with the same skill version must be a cache-hit. When the release's source build SHA changes (rebuild with same version — treated as corruption), the prior note is marked `STALE`.
- **REQ-DP-25** — Release Detail shall include a "Deploys" sub-section listing every deploy that carried this release, with environment, state, timestamp, approver, and deploy-detail deep-link. If zero deploys have happened yet, surface the "Not yet deployed" empty state.

### 6.4 Deploy Detail

- **REQ-DP-30** — Deploy Detail shall show deploy metadata (release version, environment, Jenkins job + build number, trigger — `push-to-main` / `manual` / `scheduled` / `promote-from-{env}`, actor, started-at, duration, conclusion), a Jenkins stage/step timeline (read-only — stage names, outcomes, durations only; no log streaming), and the artifact reference that was deployed.
- **REQ-DP-31** — The stage timeline shall render each Jenkins stage with a state (`SUCCESS` / `FAILURE` / `ABORTED` / `UNSTABLE` / `SKIPPED` / `IN_PROGRESS`), a duration, and — when the stage corresponds to a manual approval gate — the approver identity, the approval timestamp, and the approval decision.
- **REQ-DP-32** — Deploy Detail shall surface a one-click action to "open an Incident from this deploy" that deep-links to the Incident slice with pre-filled context (application id, environment, deploy id, release version). This is a navigation action only; the Control Tower does not itself mutate Jenkins or the target environment.
- **REQ-DP-33** — Deploy Detail shall include an Approvals card listing every approval that gated this deploy. Each entry carries: approver identity, approver role, timestamp, decision (`APPROVED` / `REJECTED`), the gate policy version that was active at approval time, and — if available — free-text rationale captured from the Jenkins input step.
- **REQ-DP-34** — Failed deploys shall surface the failing Jenkins stage name, the failing step reference (if available), and a visible chip indicating whether a rollback deploy immediately followed (linked to that rollback's Deploy Detail).
- **REQ-DP-35** — Deploy Detail shall surface the "resulted-in-current-revision" chip consistent with Application Detail's Recent Deploys card.

### 6.5 Environment Detail

- **REQ-DP-40** — Environment Detail (reachable from Application Detail → Environments card) shall show environment metadata (name, kind — `dev` / `test` / `staging` / `prod` / custom), the current revision, the prior revision, a 90-day deploy history timeline, last-good revision, last-failed revision, change failure rate over 30 days, and mean time to recover (MTTR) for the environment over 30 days.
- **REQ-DP-41** — The deploy history timeline shall mark rollback deploys visually distinct from forward deploys.
- **REQ-DP-42** — A rollback (as a deploy concept) shall be identified by: `trigger=rollback` on the deploy record OR the deploy's release version being chronologically earlier than the environment's prior revision.

### 6.6 Story ↔ Commit ↔ Build ↔ Deploy Traceability

- **REQ-DP-50** — Every Release carries a `buildArtifactRef` pointing at a BuildArtifact row owned by the Code & Build slice; the Release Detail page resolves this reference to display commits and stories.
- **REQ-DP-51** — When the upstream BuildArtifact is not yet ingested (webhook race), the Release row shall render with a `BUILD_PENDING` badge instead of silently hiding commits; the page retries resolution with backoff.
- **REQ-DP-52** — A Traceability Inverse View shall let users pick a Story-Id and see every Release and Deploy that carried it, grouped by environment, including timestamps and deploy state.
- **REQ-DP-53** — From the Requirement slice's Story Detail, a "Deploys" tab (added in a downstream PR) shall call the Deployment aggregate endpoint to render the same inverse view scoped to that story.
- **REQ-DP-54** — From the Code & Build slice's Run Detail, a "Deploys of this build" panel (added in a downstream PR) shall call the Deployment aggregate endpoint to render every deploy that used the given build artifact.
- **REQ-DP-55** — The Story ↔ Deploy derivation chain (Story → StoryCommitLink → Commit → BuildArtifact → Release → Deployment) is always computed read-side; the slice does NOT persist a direct Story ↔ Deploy link table. This preserves a single source of truth in Code & Build for Story↔Commit linkage.

### 6.7 AI Release Notes / Deploy Summary

- **REQ-DP-60** — A workspace-scoped AI summary card on the Catalog shall narrate the last 7 days of deploy activity: notable releases shipped to prod, applications trending red, environment drift hotspots, rollback events.
- **REQ-DP-61** — Workspace summary shall regenerate at most once every 15 minutes per workspace; manual Regenerate is admin-only and further rate-limited to 1/minute.
- **REQ-DP-62** — The summary shall explicitly list its evidence (application ids, release ids, deploy ids) so the user can verify claims.
- **REQ-DP-63** — Per-release AI notes (REQ-DP-22) shall include: a 1-paragraph narrative ("what is in this release"), a "since previous prod revision" diff narrative, and a risk hint that is explicitly described as advisory (not a go/no-go verdict).
- **REQ-DP-64** — AI release notes shall honor the workspace's `aiAutonomyLevel`: `DISABLED` → no generation; `OBSERVATION` → generated but labeled "AI (observation)"; `SUPERVISED` / `AUTONOMOUS` → generated normally. AI never has approval or rollback authority regardless of level.
- **REQ-DP-65** — AI release notes shall not be invoked for releases whose source build artifact does not resolve (REQ-DP-51), to avoid hallucinating commit/story content.
- **REQ-DP-66** — AI-generated text shall be rejected at service time if it references a story ID that is not in the release's resolved story set (evidence-integrity check); placeholder is surfaced in UI until regeneration.

### 6.8 Ingestion and Data Freshness

- **REQ-DP-70** — The system shall expose a Jenkins webhook receiver and accept events emitted by the Jenkins Notification Plugin (or equivalent): `job.started`, `job.completed`, `stage.completed`, `input.approved`, `input.rejected`, `build.promoted`, `build.deleted`.
- **REQ-DP-71** — Webhook signatures shall be verified using the shared webhook secret configured per-Jenkins-instance; invalid signatures are rejected with 401 and audited.
- **REQ-DP-72** — The system shall backfill the last 30 days of deploy history for a newly-integrated Jenkins instance via Jenkins REST pagination (`/job/{folder}/api/json?tree=...`), respecting Jenkins throttling.
- **REQ-DP-73** — A nightly resync job shall reconcile the last 72 hours to heal dropped webhooks; discrepancies produce audit events, not silent overwrites of user-visible state.
- **REQ-DP-74** — Data freshness SLO: a Jenkins `job.completed` event visible in the UI within 45s of Jenkins delivery at P95 under nominal load.
- **REQ-DP-75** — When Jenkins is unreachable or actively throttling, the system shall surface a banner on the affected workspace's Catalog ("Some deploy data may be stale due to Jenkins unavailability — next sync attempt at {time}") and retry with exponential backoff.
- **REQ-DP-76** — Jenkins Release version discovery: the system derives `releaseVersion` from the Jenkins build's declared parameter `RELEASE_VERSION` OR the Jenkinsfile-annotated stage name pattern `Release {version}`. If neither is present, the deploy is still ingested but flagged `RELEASE_UNRESOLVED` and shown under a diagnostics view.

### 6.9 Error Handling and Degraded States

- **REQ-DP-80** — Per-card section error with retry on the Catalog, Application Detail, Release Detail, Deploy Detail, and Environment Detail pages (no page-level whitelash for a single card failure).
- **REQ-DP-81** — Empty states must be distinguishable: "no applications configured for this workspace", "no releases yet", "no deploys to show", "environment has no deploys", "AI summary pending", "AI summary failed", "release not yet deployed", "build artifact pending ingestion".
- **REQ-DP-82** — Stale AI release notes (REQ-DP-24) must carry a visible STALE badge and a "re-run" action for admins.
- **REQ-DP-83** — When Code & Build slice lookup fails (BuildArtifact unresolved), Deploy Detail shall render with a `BUILD_PENDING` badge rather than hiding commit/story information — absence of upstream data must never look the same as absence of linkage.
- **REQ-DP-84** — When Jenkins reports `UNSTABLE` for a stage, the UI shall render it as AMBER (not GREEN, not RED) with a tooltip explaining the Jenkins semantics.

### 6.10 Security, Privacy, and Governance

- **REQ-DP-90** — Jenkins API tokens are stored in the shared secrets vault (Platform Center), never in Control Tower DB tables or logs.
- **REQ-DP-91** — Approver identities are surfaced with their display name only; email addresses are never rendered in the UI (reachable via hover-card for PM/Tech Lead if needed — V1.1).
- **REQ-DP-92** — AI prompts shall not include raw Jenkins credentials, environment secrets, or approval-step free-text rationale written by approvers under a "private note" convention.
- **REQ-DP-93** — Every ingestion event, AI invocation, and user view of sensitive detail (approval record, rollback history) shall be audited via the shared `AuditLogEmitter`.
- **REQ-DP-94** — Only Release Manager + Tech Lead + PM roles see the approver identity free-text rationale; other readers see the decision (`APPROVED` / `REJECTED`) and timestamp only.
- **REQ-DP-95** — All write paths (ingestion, AI release-notes persistence) emit `LineageEvent` naming Deployment Management as the authoring domain.

### 6.11 Accessibility, i18n, Performance

- **REQ-DP-100** — All cards shall meet the shared shell's a11y bar (keyboard navigation, ARIA roles, color-blind-safe LEDs with shape/text redundancy).
- **REQ-DP-101** — All text shall use the shared i18n token set; no hard-coded English in components.
- **REQ-DP-102** — Catalog aggregate P95 load budget: 1200ms; Application Detail P95 budget: 1500ms; Release Detail P95 budget: 1500ms; Deploy Detail P95 budget: 1500ms (per-projection timeout 500ms with `SectionResult` fallback).
- **REQ-DP-103** — No card shall ship more than 300 KB of gzipped JS on the critical path; stage-timeline rendering must be virtualized when stage count exceeds 40.

## 7. Open Questions (tracked, not blocking)

- Should V1 surface a "my releases" personalized lens (releases whose commit set includes commits authored by the current user)? Default: no — deferred to V1.1 because it requires mapping Jenkins actor/Git author → Member.
- Should AI release notes ever be mirrored back to Jenkins build description or Git as a tag annotation? Default: no in V1; would require additional governance around automated commentary in upstream systems.
- Should the Catalog support pinning applications? Default: no — list is driven by recent-activity ordering.
- Should unresolved build-artifact references (REQ-DP-51) auto-resolve when the build is later ingested? Default: yes, with a nightly resolver job re-computing Release composition on the affected rows.
- Should V1.1 include pre-deploy AI risk assessment? Default: yes — scoped as a strictly-advisory verdict with explicit evidence, with no authority to block the Jenkins pipeline.

## 8. Traceability to PRD

| PRD clause | Requirements covered |
| ---------- | -------------------- |
| §11.9 (Deployment page definition — versions, environments, approvals, rollback, risk, AI) | REQ-DP-10…18, REQ-DP-20…25, REQ-DP-30…35, REQ-DP-40…42 |
| §18.1 (V1 must-have bucket) | REQ-DP-01…08, REQ-DP-70…76 |
| §6.4 (full traceability chain Story → Deploy) | REQ-DP-50…55 |
| §7 (AI autonomy levels — AI never approves or rolls back) | REQ-DP-64, REQ-DP-60…66 |
| §6.5 (AI-native governance — evidence-integrity, audit) | REQ-DP-23, REQ-DP-66, REQ-DP-90…95 |
