# Deployment Management — User Stories

## Source

- Requirements: [deployment-management-requirements.md](../01-requirements/deployment-management-requirements.md)
- PRD §11.9, §18.1

## Conventions

Every story uses `As a … / I want … / So that …` form with Given/When/Then acceptance criteria, edge notes, and REQ-DP references. Story IDs `DP-S1…DP-Sn` are stable and referenced downstream.

---

## Catalog (Workspace-level)

### DP-S1 — Browse my workspace's release & deploy activity at a glance

**As a** PM on a workspace with 6 projects
**I want** a Catalog page listing all applications I can see, grouped by project, with per-environment health at a glance
**So that** I can answer "what shipped this week and is prod healthy?" in 10 seconds without opening Jenkins

**Acceptance criteria:**

- Given I am a member of workspaces W1 and W2, when I visit `/deployment`, then the Catalog shows only applications linked to projects in W1 and W2, grouped by project.
- Given an application's most recent prod deploy is RED, when I look at the tile, then the aggregate LED is red and the "current prod revision" chip renders with a red border.
- Given I filter by `environment=prod` + `deploy-status=RED`, then only tiles whose most-recent prod deploy is FAILED or ROLLED_BACK remain visible.
- Given I type `checkout-` in search, then only applications whose name contains `checkout-` are shown across all projects.

**Edge notes:**

- When zero applications are configured for the workspace, render the "no applications" empty state with a Platform Center deep-link to configure Jenkins integration.
- If the Jenkins unavailability banner is active, show it above the aggregate summary bar.

**Requirements:** REQ-DP-01, REQ-DP-02, REQ-DP-03, REQ-DP-04, REQ-DP-06, REQ-DP-07, REQ-DP-08, REQ-DP-81, REQ-DP-100, REQ-DP-102

---

### DP-S2 — See an AI narrative of the last 7 days across my workspace

**As a** PM
**I want** a single-paragraph AI summary of "what deployed in the last 7 days" with clickable evidence
**So that** I can brief my release review in one minute

**Acceptance criteria:**

- Given the summary exists and is ≤15 minutes old, when I open the Catalog, then the AI Insights card renders the summary inline with a "Generated Xm ago" timestamp.
- Given I click an evidence chip (e.g. "2 rollbacks on `api-gateway`"), then I deep-link to the filtered Application Detail page for that application with the time window preserved.
- Given I am an admin and the summary is older than 15m, when I click Regenerate, then a new summary is requested and the button is rate-limited to 1/min.
- Given the workspace has `aiAutonomyLevel=DISABLED`, then the AI Insights card is replaced with a static "AI summary disabled for this workspace" notice.

**Edge notes:**

- Summary must include evidence IDs it relied on; the card surfaces them as chips.
- If summary generation failed, surface FAILED + admin-only Retry.

**Requirements:** REQ-DP-60, REQ-DP-61, REQ-DP-62, REQ-DP-64, REQ-DP-81

---

## Application Detail

### DP-S3 — Drill into an application's environments, releases, and deploys

**As a** Release Manager
**I want** a single page per application with environments, releases, deploys, trace summary, and AI insights
**So that** I do not have to hop between Jenkins folders and spreadsheets to reason about a release

**Acceptance criteria:**

- Given I click a tile on the Catalog, when Application Detail loads, then Header / Environments / Recent Releases / Recent Deploys / Trace Summary / AI Insights cards render with independent skeletons.
- Given the Environments card loads successfully but the Recent Deploys card times out at 500ms, then only the Deploys card shows an error+retry; other cards remain interactive.
- Given an environment is currently in a rolled-back state, the Environments card shall mark it with a ROLLED-BACK chip and show the target revision the rollback landed on.
- Given I click any Jenkins build number, then a new tab opens on the Jenkins build page with `rel="noopener noreferrer"`.

**Edge notes:**

- If the application has zero releases, show the "no releases yet" empty state (not an error).
- If an environment has zero deploys, show a hint chip under its label ("No deploys yet").

**Requirements:** REQ-DP-10, REQ-DP-11, REQ-DP-12, REQ-DP-13, REQ-DP-14, REQ-DP-15, REQ-DP-17, REQ-DP-18, REQ-DP-80, REQ-DP-81, REQ-DP-102

---

### DP-S4 — Understand the current state of each environment for an application

**As a** SRE
**I want** the Environments card to tell me which revision is currently live in each environment and when it got there
**So that** I can correlate an incident timeline to the last deploy that changed prod

**Acceptance criteria:**

- Given `prod` currently runs release `v2.18.3` deployed at `2026-04-17 09:41 UTC`, when I open Application Detail, then the Environments card shows `prod: v2.18.3 (succeeded, 9h ago)` with a "resulted-in-current-revision" chip on the matching deploy row in Recent Deploys.
- Given `prod` was rolled back at `2026-04-17 23:10 UTC` to `v2.18.1`, when I open Application Detail, then the Environments card shows `prod: v2.18.1 (rolled-back)` with a link to the rollback deploy.
- Given I click the environment name, then I navigate to the Environment Detail page for that application + environment.

**Edge notes:**

- "Current revision" may be stale by up to the REQ-DP-74 freshness SLO (45s P95); no banner required unless stale by >5 minutes.
- If current revision is unresolved (upstream Jenkins returned a `releaseVersion=null`), render as `UNKNOWN` with a diagnostics tooltip.

**Requirements:** REQ-DP-12, REQ-DP-40, REQ-DP-41, REQ-DP-42, REQ-DP-74, REQ-DP-76, REQ-DP-84

---

## Release Detail

### DP-S5 — Inspect what is in a release

**As a** PM
**I want** Release Detail to show the build, the commits, and the linked stories for a given release version
**So that** I know exactly what customer-visible work is carried by that release

**Acceptance criteria:**

- Given release `v2.18.3` was built from BuildArtifact `BA-77182`, when I open Release Detail, then the Build card shows `BA-77182` with a deep-link into the Code & Build slice, and the Commits list shows every commit from the artifact's commit range.
- Given a commit's message carries `Story-Id: STORY-4211`, then the Stories list includes `STORY-4211` as a chip that deep-links to the Requirement slice.
- Given a story ID does not resolve in the Requirement slice, then the chip renders as `UNVERIFIED` (grey) rather than being hidden.
- Given the upstream BuildArtifact has not yet been ingested, then the Release row on Application Detail is badged `BUILD_PENDING` and opening Release Detail surfaces the pending state on the Build card only; other cards remain interactive.

**Edge notes:**

- Release Detail recomputes composition on each page load; no stale cache on commits/stories.
- If the release has an unusually large commit range (>100 commits), show a capped-by-policy banner on the Commits card.

**Requirements:** REQ-DP-20, REQ-DP-21, REQ-DP-50, REQ-DP-51, REQ-DP-55, REQ-DP-83

---

### DP-S6 — Read AI release notes before I ship an announcement

**As a** PM about to draft a release announcement
**I want** an AI-generated narrative describing what's in this release plus its diff vs the prior prod revision
**So that** I can turn it into a customer-facing note in minutes, not hours

**Acceptance criteria:**

- Given the release's build artifact has resolved and `aiAutonomyLevel >= OBSERVATION`, when I open Release Detail, then the AI Release Notes card shows a one-paragraph narrative, a "since previous prod revision" paragraph, and evidence chips (commit SHAs, story IDs).
- Given I hover over a story chip in the AI narrative, then the underlying story title and status are surfaced in a popover.
- Given the AI note references a story ID that is not in the release's resolved story set, then the card renders as `EVIDENCE_MISMATCH` with a placeholder and an admin-only Regenerate button.
- Given `aiAutonomyLevel=DISABLED`, then the AI Release Notes card is replaced with a static "AI release notes disabled for this workspace" notice.

**Edge notes:**

- AI notes are never posted to Jenkins, Git, or any external tracker automatically.
- A stale note (build SHA advanced after generation — corruption case) renders a STALE badge.

**Requirements:** REQ-DP-22, REQ-DP-23, REQ-DP-24, REQ-DP-63, REQ-DP-64, REQ-DP-65, REQ-DP-66, REQ-DP-82

---

## Deploy Detail

### DP-S7 — Understand why a deploy succeeded or failed

**As a** Tech Lead whose team owns `checkout-service`
**I want** Deploy Detail to show the stage timeline with state per stage, plus the failing stage call-out when applicable
**So that** I can reason about the deploy without opening the Jenkins console

**Acceptance criteria:**

- Given deploy `DEP-99023` executed 6 stages, when I open Deploy Detail, then the timeline renders each stage with its state (`SUCCESS` / `FAILURE` / `ABORTED` / `UNSTABLE` / `SKIPPED` / `IN_PROGRESS`) and duration.
- Given a stage is `FAILURE`, then the Deploy Detail shows a "Failing stage" call-out at the top with the stage name and — if present — the failing step reference.
- Given the deploy failed and a subsequent rollback deploy exists for the same environment within 1 hour, then a "Followed by rollback: DEP-99025" chip is shown, linking to that rollback's Deploy Detail.
- Given a stage state is `UNSTABLE`, then it renders AMBER with a tooltip explaining Jenkins' "unstable" semantics (tests failed but build continued).

**Edge notes:**

- The timeline is metadata-only; no log streaming in V1.
- For deploys with >40 stages, the timeline virtualizes.

**Requirements:** REQ-DP-30, REQ-DP-31, REQ-DP-34, REQ-DP-84, REQ-DP-103

---

### DP-S8 — Confirm who approved a prod deploy and under what policy

**As a** PM auditing a prod change
**I want** the Approvals card on Deploy Detail to show every approval gate that the deploy passed, who approved, when, and under which gate-policy version
**So that** I can answer audit questions about who authorized the change

**Acceptance criteria:**

- Given deploy `DEP-99023` passed an approval gate at stage `Approve Prod Release`, when I open Deploy Detail, then the Approvals card lists the approver name, timestamp, decision (`APPROVED` / `REJECTED`), and the gate policy version active at that time.
- Given I am a Release Manager / Tech Lead / PM, then the Approvals card additionally shows any free-text rationale captured from the Jenkins input step.
- Given I am an Engineer (role without rationale visibility), then the rationale field is omitted and only decision + timestamp are shown.
- Given the deploy has zero approval gates (e.g. `dev` environment), then the Approvals card renders an "No approvals required for this deploy" empty state.

**Edge notes:**

- If the approver identity cannot be resolved to a platform Member, show the Jenkins user id verbatim with a "(unresolved)" suffix.
- Email addresses are never rendered; only display names.

**Requirements:** REQ-DP-33, REQ-DP-91, REQ-DP-93, REQ-DP-94

---

### DP-S9 — Open an incident straight from a failing deploy

**As a** SRE on-call
**I want** a one-click "Open Incident from this deploy" action
**So that** I can start incident intake with the right pre-filled context instead of typing ids by hand

**Acceptance criteria:**

- Given deploy `DEP-99023` is `FAILED`, when I click "Open Incident from this deploy", then I navigate into the Incident slice's new-incident page with pre-filled: application id, environment, deploy id, release version, and a link back to this Deploy Detail.
- Given the Incident slice is unreachable, then the button surfaces an inline error "Incident slice unavailable — try again or file manually" without mutating any state.
- Given the deploy is `SUCCEEDED`, the button is still available (an SRE may want to open an incident for downstream symptoms) but shows a confirmation dialog clarifying "this deploy succeeded — are you sure?".

**Edge notes:**

- This action is navigation-only; it does not call Jenkins or mutate any deploy state.

**Requirements:** REQ-DP-32, REQ-DP-93

---

## Environment Detail

### DP-S10 — Read environment-level deploy history and rollback pattern

**As a** Release Manager
**I want** Environment Detail to show the last 90 days of deploys, last-good and last-failed revisions, change failure rate, and MTTR
**So that** I can spot environment drift and argue for engineering investment

**Acceptance criteria:**

- Given I navigate into `prod` environment of `checkout-service`, when Environment Detail loads, then I see: current revision, prior revision, last-good revision, last-failed revision, 30-day change failure rate (%), 30-day MTTR, and a 90-day deploy timeline.
- Given 2 rollback deploys happened in the last 30 days out of 40 total deploys, then change failure rate renders as `5.0%`.
- Given `staging` has no deploys in the last 30 days, then change failure rate and MTTR render as `—` with a "No deploys in window" hint.
- Given the deploy timeline contains >200 entries, it virtualizes.

**Edge notes:**

- "Rollback" detection is per REQ-DP-42 (trigger=rollback OR release version older than prior revision).
- MTTR is measured from first `FAILED` deploy to the next `SUCCEEDED` deploy in the same environment.

**Requirements:** REQ-DP-40, REQ-DP-41, REQ-DP-42, REQ-DP-103

---

## Traceability

### DP-S11 — Find every release and deploy that carried a given story

**As a** PM fielding a "is STORY-4211 in prod?" question
**I want** a Traceability Inverse View that accepts a Story-Id and returns every Release and Deploy that carried it, grouped by environment
**So that** I can answer without asking engineering

**Acceptance criteria:**

- Given I enter `STORY-4211` on `/deployment/traceability`, when results load, then I see every Release whose composition includes that story, and beneath each release, every Deploy that carried it (grouped by environment).
- Given `STORY-4211` is in no Release yet, then the view renders a "Not yet in any release" empty state.
- Given the story ID does not exist in the Requirement slice, then the view renders `UNKNOWN STORY` without issuing a trace query.
- Given I am on the Requirement slice's Story Detail and open the "Deploys" tab, then the same inverse view is embedded (REQ-DP-53).

**Edge notes:**

- The trace chain is always read-side derived (Story → StoryCommitLink → Commit → BuildArtifact → Release → Deploy). No direct Story ↔ Deploy link table (REQ-DP-55).
- The derivation depends on the Code & Build slice being available; if unavailable, the view shows a degraded-mode banner.

**Requirements:** REQ-DP-50, REQ-DP-52, REQ-DP-53, REQ-DP-55

---

### DP-S12 — See every deploy that used a given build artifact

**As a** Tech Lead on the Code & Build slice's Run Detail page
**I want** a "Deploys of this build" panel that tells me which environments this build artifact has reached
**So that** I know whether the green build has already shipped or is still waiting

**Acceptance criteria:**

- Given Build `BA-77182` has been deployed to `dev` and `staging` but not `prod`, when the panel renders, then it lists the two deploys with environment, state, timestamp, and deploy-detail deep-link.
- Given the build has not been deployed anywhere, then the panel renders "Not yet deployed" empty state with a hint about expected promotion time.
- Given the Deployment aggregate endpoint is unavailable, then the panel shows an error+retry that does NOT fail the Run Detail page.

**Requirements:** REQ-DP-54, REQ-DP-55, REQ-DP-80

---

## Ingestion and Degraded States

### DP-S13 — Keep the data fresh when webhooks drop

**As a** platform operator
**I want** a nightly resync job to reconcile the last 72h of Jenkins history
**So that** silently-dropped webhooks cannot leave the UI showing stale "current revision" forever

**Acceptance criteria:**

- Given the nightly resync discovers a Jenkins build completion that was not ingested, when resync runs, then the missing deploy is created with its original timestamps and an `INGEST_SOURCE=resync` audit event is emitted.
- Given resync discovers a discrepancy on a user-visible field (e.g. `conclusion`), when it resolves, then the field is updated AND an audit event is emitted — no silent overwrite.
- Given Jenkins is unreachable during resync, the job fails gracefully and surfaces a banner on affected workspaces' Catalog.

**Edge notes:**

- Resync window is fixed at 72h; older discrepancies must be handled by a separate manual backfill.

**Requirements:** REQ-DP-72, REQ-DP-73, REQ-DP-75, REQ-DP-95

---

### DP-S14 — Cope with a BuildArtifact that hasn't been ingested yet

**As a** PM opening a freshly-minted release
**I want** the Release Detail page to handle the webhook race where the upstream build hasn't yet been ingested
**So that** I don't see a scary-looking broken page during the 10-second race window

**Acceptance criteria:**

- Given the Release's `buildArtifactRef` does not yet resolve in the Code & Build slice, when Release Detail loads, then the Build card renders a `BUILD_PENDING` badge and the Commits / Stories lists render skeletons with a "resolving upstream build…" hint.
- Given the build artifact resolves within 30 seconds, then the cards auto-refresh without a full page reload.
- Given the build artifact does not resolve within 5 minutes, then the page switches to a terminal error state on the Build card with an admin-only Retry.

**Edge notes:**

- AI release notes (REQ-DP-65) are not invoked while BuildArtifact is pending — absence of upstream data would otherwise cause hallucination risk.

**Requirements:** REQ-DP-51, REQ-DP-65, REQ-DP-83

---

## AI Governance

### DP-S15 — Trust that AI is never approving, rolling back, or posting upstream

**As a** governance lead
**I want** the AI autonomy contract to be visibly and enforceably advisory
**So that** AI output can never substitute for a human approver on a prod change

**Acceptance criteria:**

- Given `aiAutonomyLevel=AUTONOMOUS`, when I view any AI release-notes card, then the narrative is clearly labeled "AI-generated" with an evidence footnote, and no button on the card triggers an approval, a rollback, a Jenkins job, or an outbound post.
- Given an AI narrative would reference a story ID not in the release's resolved set, the server rejects the output with `DP_RELEASE_NOTES_EVIDENCE_MISMATCH` and surfaces a placeholder — never renders the bogus claim.
- Given the workspace's autonomy level changes from `AUTONOMOUS` to `DISABLED`, any existing AI release-notes card is hidden on the next page load (not deleted from audit).

**Edge notes:**

- AI output changes never cascade into Jenkins, Git, or any external tracker in V1.

**Requirements:** REQ-DP-23, REQ-DP-60, REQ-DP-64, REQ-DP-66, REQ-DP-92, REQ-DP-93, REQ-DP-95
