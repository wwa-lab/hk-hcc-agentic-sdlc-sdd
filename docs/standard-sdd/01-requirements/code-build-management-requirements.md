# Code & Build Management Requirements

## 1. Context and Source

Requirements for the **Code & Build Management** slice of the Agentic SDLC Control Tower. Derived from:

- PRD §11.7 (Code & Build Management page definition)
- PRD §18.1 (V1 must-have scope — Code & Build is included in the V1 must-have bucket)
- PRD §6.4 / §6.5 / §7 (traceability chain, AI platform, autonomy levels)
- Scope decisions captured via the slice-kickoff questionnaire on 2026-04-17:
  - V1 is an **observability-first viewer** (read-only, no triggering, no config)
  - Integration surface is limited to **GitHub + GitHub Actions** (other SCM/CI systems deferred)
  - Primary traceability is **Story → Commit → Build**
  - AI scope is **summary + failure triage + PR review assist** (no code generation, no auto-fix)

## 2. Goal

Give PMs, Tech Leads, Designers, and Engineers a single page to understand, at a glance:

1. **What code changed** — recent commits, open PRs, merged PRs — scoped to the workspaces the user can see
2. **What was built** — pipeline runs per repo, with state, duration, failing step, and artifact produced
3. **How that code maps back to stories** — every commit and PR traces to one or more user stories from the Requirement slice; every build traces back through its commits
4. **What AI observed** — narrative summary of the workspace's code/build health, triage notes on failed builds, and AI review notes on open PRs
5. **Where to dig deeper** — deep-links into Requirement stories, Incident cases, and the upstream GitHub page for a commit / PR / run

Non-goals for V1: triggering builds, re-running pipelines, editing pipeline YAML, configuring branch protections, merging PRs, opening PRs, writing code.

## 3. Scope Boundaries

| In scope (V1)                                                                         | Out of scope (V1; revisit in V1.1+)                               |
| ------------------------------------------------------------------------------------- | ----------------------------------------------------------------- |
| Read-only Catalog: repos + recent activity per project/workspace                      | Creating repos from the Control Tower                             |
| Read-only Repo Detail: branches, open PRs, recent commits, recent runs                | Merging, closing, opening PRs from Control Tower                  |
| Read-only Pipeline Run Detail: job timeline, step logs (capped), artifact links       | Re-running / cancelling runs                                      |
| Story ↔ Commit linkage via conventional trailer `Story-Id: STORY-…`                   | Inferring story IDs from branch name heuristics                   |
| AI build-failure triage (likely cause, suggested owner)                               | AI auto-fix or patch generation                                   |
| AI PR review assist (readable suggestions, NOT merge-gating)                          | AI-authored PRs, AI-driven merges                                 |
| AI narrative summary of recent code/build activity                                    | Predictive SLO/quality gate recommendations                       |
| GitHub + GitHub Actions via App-level installation                                    | GitLab, Bitbucket, Jenkins, Azure DevOps                          |
| Webhook-driven ingestion with resync/backfill job                                     | Polling-only mode                                                 |
| Deep-links out to requirement stories, incidents, and GitHub                          | Test results (owned by Testing slice), Deploys (Deployment slice) |
| Workspace isolation and role-based read access                                        | Fine-grained per-repo ACL beyond workspace membership             |

## 4. User Personas

- **PM** — wants a workspace-level summary of recent code activity and whether builds are healthy, plus "which stories shipped this week".
- **Tech Lead** — wants repo-level drill-in: what's on each branch, failing builds, failing tests, AI PR review notes.
- **Engineer** — wants to see their own commits/PRs and AI triage notes when a build they touched fails.
- **Designer / Analyst** — wants to confirm that a given story's commits landed and that the build was green.

## 5. Upstream / Downstream Dependencies

- **Upstream:** Requirement slice (for story lookup and Story→Commit backlink), Platform Center (for GitHub App installation + token vault), Team/Project Space (for workspace + project membership).
- **Downstream:** Incident slice (deep-linking a failing build into a new incident), Testing slice (consumes build artifact refs in V1.1), Deployment slice (consumes build artifacts in V1.1), Dashboard ("recent code activity" tile).

## 6. Requirements

### 6.1 Catalog (Workspace-level view)

- **REQ-CB-01** — Catalog page lists every repository visible to the user, grouped by project, with repo name, default branch, last activity timestamp, open PR count, and latest build status LED.
- **REQ-CB-02** — Catalog shall show an aggregate summary bar: total repos, total open PRs, recent builds count, success rate (%) over the last 7 days, mean build duration.
- **REQ-CB-03** — Catalog must be filterable by project, branch-is-default, build-status (GREEN/AMBER/RED), and time window (24h / 7d / 30d).
- **REQ-CB-04** — Catalog must be searchable by repo name or full path.
- **REQ-CB-05** — Each repo tile shall deep-link to its Repo Detail page.
- **REQ-CB-06** — Catalog must respect workspace isolation: a repo attached to a workspace the user does not belong to must not be returned.
- **REQ-CB-07** — Each repo tile shall show a lineage badge indicating the upstream Project it belongs to (reusing `<LineageBadge>`).

### 6.2 Repo Detail

- **REQ-CB-10** — Repo Detail shall render 6 cards: Header, Branches, Open PRs, Recent Commits, Recent Pipeline Runs, AI Insights.
- **REQ-CB-11** — Header shall show repo name, description, default branch, primary language, last push timestamp, an external-link-out to the GitHub page, and the owning Project.
- **REQ-CB-12** — Branches card shall list up to 20 most-recently-updated branches with name, ahead/behind-default counts, last commit SHA+message+author, and whether a PR exists.
- **REQ-CB-13** — Open PRs card shall list every open PR with number, title, author, source→target branch, state (OPEN / DRAFT), latest CI status, AI review note count, and a last-updated timestamp.
- **REQ-CB-14** — Recent Commits card shall list up to 30 most-recent commits on the default branch with SHA (short), author, message, timestamp, and linked Story-Id(s).
- **REQ-CB-15** — Recent Pipeline Runs card shall list up to 20 most-recent GH Actions runs with pipeline name, trigger (push/PR/schedule/manual), branch, state, duration, and commit SHA.
- **REQ-CB-16** — AI Insights card shall render the latest narrative AI summary for this repo (health, notable commits/PRs, build-pass-rate trend).
- **REQ-CB-17** — Every commit / PR / run row shall include a deep-link out to GitHub (new tab, `rel="noopener noreferrer"`).
- **REQ-CB-18** — Each card shall independently degrade on error (per-card error state + retry) without failing the page.

### 6.3 PR Detail and AI PR Review Assist

- **REQ-CB-20** — PR Detail shall show PR metadata (title, author, source/target, state, reviewers requested, labels, linked stories), commit list, CI status per run, and AI review notes.
- **REQ-CB-21** — AI PR review notes are **advisory only** — they never block merge and they are never posted as comments on GitHub automatically; they render inside the Control Tower's PR Detail page.
- **REQ-CB-22** — AI PR review notes shall be keyed by (prId, headCommitSha, skillVersion); when the head commit advances, stale notes must be visually marked `STALE` until a re-run completes.
- **REQ-CB-23** — AI PR review shall group notes by severity (BLOCKER / MAJOR / MINOR / NIT) and by file path.
- **REQ-CB-24** — Each note shall carry an explanation and, where possible, a line/range anchor back to the diff.
- **REQ-CB-25** — AI PR review shall honor the workspace's `aiAutonomyLevel`: `DISABLED` → no review runs; `OBSERVATION` → runs but reviewer identity is shown as "AI (observation)"; `SUPERVISED`/`AUTONOMOUS` → runs normally. AI never has merge authority regardless of level.
- **REQ-CB-26** — When a PR's head commit is pushed, the system shall enqueue an AI review re-run if autonomy ≥ OBSERVATION; debounced at 20s per PR to avoid storms during rebases.
- **REQ-CB-27** — AI PR review shall not be invoked for PRs authored by a bot account (configurable bot list) to avoid feedback loops.

### 6.4 Pipeline Run Detail and AI Failure Triage

- **REQ-CB-30** — Run Detail shall show run metadata (pipeline, trigger, branch, commit, actor, duration, conclusion), a job/step timeline, per-step outcome, and artifact links (metadata only — the page does not stream logs in V1).
- **REQ-CB-31** — Failed runs shall show an AI Failure Triage card with the likely root cause, the failing step reference, candidate owner(s) (derived from recent committers on the failing paths), and a confidence indicator.
- **REQ-CB-32** — AI Failure Triage must reference evidence (log excerpt, failing step id) and never hallucinate file/line numbers that do not exist in the run's logs.
- **REQ-CB-33** — AI Failure Triage shall rerun when the run's conclusion changes (e.g. a flaky-test re-run) and the most recent triage row is marked current.
- **REQ-CB-34** — Triage rows are keyed by (runId, skillVersion); cache rotates on version advance.
- **REQ-CB-35** — Run Detail shall offer a one-click action to "open an Incident from this failure" that deep-links to the Incident slice with pre-filled context (commit SHA, run URL, triage summary). This is a navigation action only; the Control Tower does not itself mutate GitHub.

### 6.5 Story ↔ Commit ↔ Build Traceability

- **REQ-CB-40** — The system shall extract `Story-Id: STORY-…` trailers from every ingested commit message (zero or more per commit).
- **REQ-CB-41** — The system shall also accept Story-Id references in PR body text under the form `Relates-to: STORY-…`.
- **REQ-CB-42** — For every ingested commit, the system shall persist a StoryCommitLink row per extracted Story-Id.
- **REQ-CB-43** — Validated story IDs are those that exist in the Requirement slice and are visible to the same workspace; unknown story IDs are persisted as link rows with `status=UNKNOWN_STORY` and surfaced on a dedicated "unresolved links" diagnostics view.
- **REQ-CB-44** — A Traceability view shall let users pick a Story-Id and see the inverse: all commits, PRs, and pipeline runs that reference it, across all repos the user can see.
- **REQ-CB-45** — From the Requirement slice's Story Detail, a "Code" tab (added in a downstream PR) shall call the Code & Build aggregate endpoint to render the same inverse view scoped to that story.
- **REQ-CB-46** — Build traceability: every PipelineRun carries `headCommitSha`; from a run, a user shall be able to expand "Stories this build contains" — the union of Story-Ids reachable from every commit in the run's commit range (capped at 100 commits in V1 to avoid pathological ranges).

### 6.6 AI Workspace Summary

- **REQ-CB-50** — A workspace-scoped AI summary card on the Catalog shall narrate the last 24 hours of code+build activity: notable merged stories, top failing pipeline, repos trending red, PR review hot spots.
- **REQ-CB-51** — Workspace summary shall regenerate at most once every 10 minutes per workspace; manual Regenerate is admin-only and further rate-limited (1/minute).
- **REQ-CB-52** — The summary shall explicitly list its evidence (repo ids, run ids, PR ids) so the user can verify claims.

### 6.7 Ingestion and Data Freshness

- **REQ-CB-60** — The system shall register a GitHub App and subscribe to the following webhook events: `push`, `pull_request`, `pull_request_review`, `workflow_run`, `workflow_job`, `check_run`, `installation`, `installation_repositories`.
- **REQ-CB-61** — Webhook signatures shall be verified using the App secret; invalid signatures are rejected with 401 and audited.
- **REQ-CB-62** — The system shall backfill the last 14 days of history for a newly-installed repo (commits, PRs, workflow runs) via REST pagination, respecting GitHub's rate limit (primary + secondary).
- **REQ-CB-63** — A nightly resync job shall reconcile the last 72 hours to heal dropped webhooks; discrepancies produce audit events, not silent overwrites of user-visible state.
- **REQ-CB-64** — Data freshness SLO: a push event visible in the UI within 30s of GitHub delivery at P95 under nominal load.
- **REQ-CB-65** — When GitHub API rate limit is hit, the system shall surface a banner on the affected workspace's Catalog ("Some data may be stale due to GitHub rate limits — next sync attempt at {time}") and retry with exponential backoff.

### 6.8 Error Handling and Degraded States

- **REQ-CB-70** — Per-card section error with retry on the Catalog, Repo Detail, PR Detail, and Run Detail pages (no page-level whitelash for a single card failure).
- **REQ-CB-71** — Empty states must be distinguishable: "no repos installed for this workspace", "no PRs open", "no recent runs", "no commits yet on default branch", "AI summary pending", "AI summary failed".
- **REQ-CB-72** — Stale AI notes (REQ-CB-22) must carry a visible STALE badge and a "re-run" action for admins.
- **REQ-CB-73** — When Requirement slice lookup fails, Story-Id chips render as `UNVERIFIED` (grey) rather than hiding — absence of verification must never look the same as absence of linkage.

### 6.9 Security, Privacy, and Governance

- **REQ-CB-80** — GitHub tokens are stored in the shared secrets vault (Platform Center), never in Control Tower DB tables or logs.
- **REQ-CB-81** — Log excerpts surfaced to the UI shall be redacted for common secret patterns (AWS keys, GitHub tokens, generic bearer tokens) before storage.
- **REQ-CB-82** — AI PR review prompts shall not include raw repository credentials or environment variables from the workflow run.
- **REQ-CB-83** — Every ingestion event, AI invocation, and user view of sensitive detail (artifact download URL, log excerpt) shall be audited via the shared `AuditLogEmitter`.
- **REQ-CB-84** — Only Tech Lead + PM roles see AI PR review notes marked `BLOCKER`; other readers see aggregated note counts only (privacy boundary for WIP critique).
- **REQ-CB-85** — All write paths (ingestion, AI triage persistence) emit `LineageEvent` naming Code & Build Management as the authoring domain.

### 6.10 Accessibility, i18n, Performance

- **REQ-CB-90** — All cards shall meet the shared shell's a11y bar (keyboard navigation, ARIA roles, color-blind-safe LEDs with shape/text redundancy).
- **REQ-CB-91** — All text shall use the shared i18n token set; no hard-coded English in components.
- **REQ-CB-92** — Catalog aggregate P95 load budget: 1200ms; Repo Detail P95 budget: 1500ms; Run Detail P95 budget: 1500ms (per-projection timeout 500ms with `SectionResult` fallback).
- **REQ-CB-93** — No card shall ship more than 300 KB of gzipped JS on the critical path; pipeline-run log excerpt rendering (if large) must be virtualized.

## 7. Open Questions (tracked, not blocking)

- Should V1 support a "my code" personalized lens (commits/PRs authored by the current user)? Default: no — deferred to V1.1 because it requires mapping GitHub login → Member.
- Should AI PR review notes ever be mirrored back to GitHub as comments? Default: no in V1; would require additional governance around automated commentary.
- Should the Catalog support pinning repos? Default: no — list is driven by recent-activity ordering.
- Should unresolved Story-Id links (REQ-CB-43) auto-resolve when the story later becomes visible? Default: yes, with a nightly resolver job writing back to the link rows.

## 8. Traceability to PRD

| PRD clause | Requirements covered |
| ---------- | -------------------- |
| §11.7 (Code & Build page definition, code↔Task/Spec/Design/Build chain) | REQ-CB-10…17, REQ-CB-40…46 |
| §18.1 (V1 must-have bucket) | REQ-CB-01…07, REQ-CB-60…65 |
| §6.4 (full traceability chain) | REQ-CB-40…46 |
| §7 (AI autonomy levels) | REQ-CB-25, REQ-CB-50…52 |
| §6.5 (AI-native governance) | REQ-CB-21, REQ-CB-80…85 |
