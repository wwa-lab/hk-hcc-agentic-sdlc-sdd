# Code & Build Management — User Stories

## Source

- Requirements: [code-build-management-requirements.md](../01-requirements/code-build-management-requirements.md)
- PRD §11.7, §18.1

## Conventions

Every story uses `As a … / I want … / So that …` form with Given/When/Then acceptance criteria, edge notes, and REQ-CB references. Story IDs `CB-S1…CB-Sn` are stable and referenced downstream.

---

## Catalog (Workspace-level)

### CB-S1 — Browse my workspace's code & build activity at a glance

**As a** PM on a workspace with 6 projects
**I want** a Catalog page listing all repos I can see, grouped by project, with health at a glance
**So that** I can answer "is the code side healthy?" in 10 seconds without opening GitHub

**Acceptance criteria:**

- Given I am a member of workspaces W1 and W2, when I visit `/code-build`, then the Catalog shows only repos linked to projects in W1 and W2, grouped by project.
- Given a repo has an open PR that is red in CI, when I look at the tile, then the LED is red and the tile shows the open-PR count as amber.
- Given I filter by `build-status=RED`, then only tiles whose most-recent default-branch build is RED remain visible.
- Given I type `billing-` in search, then only repos whose name contains `billing-` are shown across all projects.

**Edge notes:**

- When zero repos are installed for the workspace, render the "no repos" empty state with a Platform Center deep-link to install the GitHub App.
- If the GitHub rate-limit banner is active, show it above the aggregate summary bar.

**Requirements:** REQ-CB-01, REQ-CB-02, REQ-CB-03, REQ-CB-04, REQ-CB-06, REQ-CB-07, REQ-CB-71, REQ-CB-90, REQ-CB-92

---

### CB-S2 — See an AI narrative of the last 24 hours across my workspace

**As a** PM
**I want** a single-paragraph AI summary of "what happened in code & build in the last 24h" with clickable evidence
**So that** I can brief my stand-up in one minute

**Acceptance criteria:**

- Given the summary exists and is ≤10 minutes old, when I open the Catalog, then the AI Insights card renders the summary inline with a "Generated Xm ago" timestamp.
- Given I click an evidence chip (e.g. "3 red runs on `api-gateway`"), then I deep-link to the filtered Repo Detail page for that repo with the time window preserved.
- Given I am an admin and the summary is older than 10m, when I click Regenerate, then a new summary is requested and the button is rate-limited to 1/min.
- Given the workspace has `aiAutonomyLevel=DISABLED`, then the AI Insights card is replaced with a static "AI summary disabled for this workspace" notice.

**Edge notes:**

- Summary must include evidence IDs it relied on; the card surfaces them as chips.
- If summary generation failed, surface FAILED + admin-only Retry.

**Requirements:** REQ-CB-50, REQ-CB-51, REQ-CB-52, REQ-CB-25, REQ-CB-71

---

## Repo Detail

### CB-S3 — Drill into a repo's branches, PRs, commits, and runs

**As a** Tech Lead
**I want** a single page per repo with branches, open PRs, recent commits, recent runs, and AI insights
**So that** I do not have to bounce between GitHub tabs

**Acceptance criteria:**

- Given I click a tile on the Catalog, when Repo Detail loads, then Header / Branches / Open PRs / Recent Commits / Recent Pipeline Runs / AI Insights cards render with independent skeletons.
- Given the Branches card loads successfully but the Runs card times out at 500ms, then only the Runs card shows an error+retry; other cards remain interactive.
- Given a branch has an open PR, the Branches card shall mark it with a PR chip linking to that PR on the same page.
- Given I click any commit SHA, then a new tab opens on GitHub's commit page with `rel="noopener noreferrer"`.

**Edge notes:**

- If the repo has zero open PRs, show the "No PRs open" empty state (not an error).
- If the default branch is empty (first commit not yet pushed), Branches shows the default branch with `0 commits` and a hint.

**Requirements:** REQ-CB-10, REQ-CB-11, REQ-CB-12, REQ-CB-13, REQ-CB-14, REQ-CB-15, REQ-CB-17, REQ-CB-18, REQ-CB-70, REQ-CB-71, REQ-CB-92

---

### CB-S4 — See stories associated with each commit

**As a** PM
**I want** each commit row to show the Story-Id(s) it links to
**So that** I know what product work landed

**Acceptance criteria:**

- Given a commit message contains `Story-Id: STORY-4211`, when the commit is ingested, then a StoryCommitLink row is persisted and the commit row renders a clickable `STORY-4211` chip.
- Given the Story-Id exists in the Requirement slice within my workspace, then the chip is green and links to the story's detail page.
- Given the Story-Id does not resolve, then the chip is amber and labeled `UNVERIFIED`; hover shows "Story not found or not visible".
- Given a commit has two Story-Ids, both chips render; ordering is the order they appear in the message.

**Edge notes:**

- PR body `Relates-to: STORY-…` counts too (REQ-CB-41).
- Branch-name heuristics (e.g. `feat/story-4211-foo`) are NOT used.

**Requirements:** REQ-CB-40, REQ-CB-41, REQ-CB-42, REQ-CB-43, REQ-CB-73

---

## PR Detail + AI Review Assist

### CB-S5 — Read AI PR review notes inside the Control Tower

**As an** Engineer reviewing a teammate's PR
**I want** AI-authored review notes grouped by severity and file
**So that** I can focus my human review on the hot spots

**Acceptance criteria:**

- Given the PR's head commit has a successful AI review run, when I open PR Detail, then notes are grouped by severity (BLOCKER/MAJOR/MINOR/NIT) and by file path.
- Given the head commit advances while I am looking at the page, then the prior AI notes render with a STALE badge and a "re-run triggered" indicator.
- Given I am not a PM or Tech Lead on the repo's project, then BLOCKER-severity notes are counted but not shown by body; other severities render fully.
- Given the workspace `aiAutonomyLevel=OBSERVATION`, then the AI reviewer identity renders as "AI (observation)"; all notes are otherwise visible.

**Edge notes:**

- PRs authored by bots are skipped (REQ-CB-27) — render "AI review skipped (bot author)".
- AI review notes never appear as comments on GitHub.

**Requirements:** REQ-CB-20, REQ-CB-21, REQ-CB-22, REQ-CB-23, REQ-CB-24, REQ-CB-25, REQ-CB-26, REQ-CB-27, REQ-CB-84

---

### CB-S6 — See which stories a PR covers

**As a** PM
**I want** the PR's linked stories list on PR Detail
**So that** I know what product intent the PR carries

**Acceptance criteria:**

- Given the PR body includes `Relates-to: STORY-…` or any of its commits includes `Story-Id: STORY-…`, then the PR Detail page shows a union-of-all linked stories chip row.
- Given a linked story is VERIFIED, the chip is green; unverified is amber.
- Given I click a story chip, then I navigate to the Requirement slice's story page.

**Edge notes:** Duplicates collapsed; ordering: verified first, then unverified, then alphabetical.

**Requirements:** REQ-CB-40, REQ-CB-41, REQ-CB-42, REQ-CB-43

---

## Pipeline Run Detail + AI Failure Triage

### CB-S7 — Understand why a build failed without leaving the Control Tower

**As an** Engineer
**I want** the failed run to show an AI triage card with likely cause, failing step, and a candidate owner
**So that** I do not have to read a 10k-line workflow log before I know what to investigate

**Acceptance criteria:**

- Given a run concludes as FAILED, when I open Run Detail, then an AI Failure Triage card renders with `likelyCause`, `failingStepRef`, `candidateOwners[]`, and a `confidence` indicator.
- Given the triage cites a log excerpt, then the excerpt is rendered with byte offsets and a link to the full workflow log on GitHub.
- Given the run's conclusion changes (e.g. a re-run passes), then the prior triage row is marked `SUPERSEDED` and a fresh one is generated.
- Given the triage's `failingStepRef` is not one of the run's actual steps, then the card is rejected at service time and a placeholder "Triage unavailable — evidence mismatch" is shown instead.

**Edge notes:**

- Successful runs do not generate triage.
- If triage generation fails, show FAILED + admin-only Retry, per REQ-CB-34.

**Requirements:** REQ-CB-30, REQ-CB-31, REQ-CB-32, REQ-CB-33, REQ-CB-34, REQ-CB-71, REQ-CB-72, REQ-CB-82

---

### CB-S8 — Open an Incident from a failing build

**As a** Tech Lead
**I want** a one-click "Open Incident" action on Run Detail
**So that** I can escalate a recurring failure without retyping the context

**Acceptance criteria:**

- Given a run is FAILED, when I click Open Incident, then I am navigated to the Incident slice's create-incident form with `commitSha`, `runUrl`, `pipelineName`, and `triageSummary` prefilled via query params.
- Given I do not have permission to create incidents in this workspace, then the button is disabled with a tooltip; nothing mutates in GitHub.
- Given the incident is created, the Control Tower does not itself write a comment back to GitHub in V1.

**Edge notes:** The action is a navigation; it never mutates GitHub state.

**Requirements:** REQ-CB-35, REQ-CB-85

---

## Traceability View

### CB-S9 — Inverse lookup: find every commit/PR/run for a story

**As a** PM
**I want** to enter a Story-Id and see all code activity that references it across my workspace
**So that** I can answer "has STORY-4211 actually shipped?"

**Acceptance criteria:**

- Given I navigate to `/code-build/traceability?storyId=STORY-4211`, then I see three sections: Commits, PRs, Pipeline Runs (that contain at least one of the linked commits).
- Given a Pipeline Run contains >100 commits in its range, then the "Stories this build contains" list is capped at 100 and a banner explains the cap.
- Given the Story-Id does not resolve in the Requirement slice, then I see an `UNKNOWN_STORY` warning but the link rows still render.
- Given no activity exists, empty-state copy distinguishes "no activity yet" from "no visible activity — you may lack access".

**Edge notes:** This view is invoked both directly (via URL) and embedded (as a Requirement-slice story-detail Code tab).

**Requirements:** REQ-CB-44, REQ-CB-45, REQ-CB-46, REQ-CB-71

---

### CB-S10 — Story-detail Code tab

**As an** Engineer viewing a story in the Requirement slice
**I want** a Code tab that shows the commits, PRs, and builds linked to this story
**So that** I stay in one place

**Acceptance criteria:**

- Given I am on `requirement/stories/STORY-4211`, when I click the Code tab, then Code & Build's inverse-lookup aggregate is rendered inline.
- Given the Code & Build service is down, then the tab shows a per-tab error with retry; the rest of the story page is unaffected.
- Given no code references this story yet, render an explanatory empty state.

**Edge notes:** This requires a cross-slice aggregate endpoint; coordinate with the Requirement slice maintainer.

**Requirements:** REQ-CB-45

---

## Ingestion and Freshness

### CB-S11 — Webhook-driven freshness with backfill safety net

**As an** operator
**I want** push events to appear in the UI within 30s at P95, with a resync job that heals dropped webhooks
**So that** the Control Tower does not silently diverge from GitHub

**Acceptance criteria:**

- Given GitHub delivers a `push` webhook, when the signature verifies and the payload parses, then affected Catalog/Repo Detail cards reflect the new commit within 30s at P95.
- Given a webhook fails signature verification, then the request is rejected with 401 and an audit log entry is written; no data mutation occurs.
- Given a scheduled 72h resync detects a missing commit, then the commit is backfilled and an audit event describes the delta.
- Given GitHub primary+secondary rate limits are hit, then affected workspaces show a rate-limit banner and retries are exponential.

**Edge notes:** A new App installation triggers a 14-day backfill (REQ-CB-62).

**Requirements:** REQ-CB-60, REQ-CB-61, REQ-CB-62, REQ-CB-63, REQ-CB-64, REQ-CB-65

---

## Security, Privacy, Governance

### CB-S12 — Log excerpts are redacted before I see them

**As a** Security reviewer
**I want** log excerpts in the UI to have known secret patterns redacted
**So that** casual viewing of a failure page does not leak AWS keys

**Acceptance criteria:**

- Given a log excerpt contains `AKIA…` or `ghp_…`, when it is stored from ingestion, then it is redacted to `AKIA******REDACTED` in the stored excerpt and in any downstream AI prompt.
- Given an artifact URL is surfaced to the UI, viewing it emits an audit entry.
- Given AI is invoked for PR review or triage, the prompt shall not include repository secrets or workflow environment variables.

**Edge notes:** Redaction is purely additive — the original raw log is not stored in Control Tower at all; we keep only redacted excerpts.

**Requirements:** REQ-CB-80, REQ-CB-81, REQ-CB-82, REQ-CB-83, REQ-CB-85

---

### CB-S13 — Non-lead readers do not see BLOCKER-severity WIP critique

**As a** non-Lead Engineer
**I want** BLOCKER-severity AI review notes to be aggregated (counted) but not shown by body
**So that** WIP critique is not casually readable across teams

**Acceptance criteria:**

- Given my role on the repo's project is not PM/Tech Lead, when I view PR Detail, then BLOCKERs render as a count pill only (`3 BLOCKERs — visible to Leads`); MAJOR/MINOR/NIT render fully.
- Given I am a Tech Lead on a different project, then the same redaction applies — role is project-scoped, not workspace-scoped.

**Edge notes:** This is a default; workspace admins may widen via Platform Center (out of scope for V1).

**Requirements:** REQ-CB-84

---

## Cross-cutting

### CB-S14 — Per-card error isolation across every page

**As a** user on any Code & Build page
**I want** a single failing card to not take down the page
**So that** I can keep working with the cards that loaded

**Acceptance criteria:**

- Given one projection times out at 500ms, then only that card shows error+retry; the rest render normally.
- Given I click retry on that card, then only that card re-requests and no other card's state is reset.
- Given the backend returns `CB_WORKSPACE_FORBIDDEN`, then the entire page renders a single access-denied screen, not per-card errors.

**Requirements:** REQ-CB-18, REQ-CB-70, REQ-CB-92

---

### CB-S15 — Deep-link parity

**As a** user
**I want** every identity (repo, PR, commit, run, story) to be independently URL-addressable and shareable
**So that** I can drop a link in chat and the teammate lands exactly where I was

**Acceptance criteria:**

- Given I navigate to `/code-build/repos/{repoId}` directly, then the page loads as if I had drilled in from the Catalog.
- Given I navigate to `/code-build/runs/{runId}` directly, then Run Detail loads and breadcrumbs are correct.
- Given I navigate to `/code-build/traceability?storyId=STORY-4211` directly, then the inverse view renders and deep-links out are correct.

**Requirements:** REQ-CB-05, REQ-CB-44, REQ-CB-90

---

## Story coverage summary

| Requirement range | Covered by |
| ----------------- | ---------- |
| REQ-CB-01…07 (Catalog) | CB-S1, CB-S2 |
| REQ-CB-10…18 (Repo Detail) | CB-S3, CB-S4, CB-S14 |
| REQ-CB-20…27 (PR Detail + AI review) | CB-S5, CB-S6, CB-S13 |
| REQ-CB-30…35 (Run Detail + AI triage) | CB-S7, CB-S8 |
| REQ-CB-40…46 (Story↔Commit↔Build traceability) | CB-S4, CB-S6, CB-S9, CB-S10 |
| REQ-CB-50…52 (AI workspace summary) | CB-S2 |
| REQ-CB-60…65 (Ingestion) | CB-S11 |
| REQ-CB-70…73 (Error/empty) | CB-S1, CB-S3, CB-S7, CB-S14 |
| REQ-CB-80…85 (Security/privacy) | CB-S12, CB-S13 |
| REQ-CB-90…93 (NFRs) | CB-S1, CB-S3, CB-S14, CB-S15 |
