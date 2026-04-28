# Code & Build Management — Spec

## Source

| Field | Value |
| ----- | ----- |
| Source Requirements | [code-build-management-requirements.md](../01-requirements/code-build-management-requirements.md) |
| Source Stories | [code-build-management-stories.md](../02-user-stories/code-build-management-stories.md) |
| PRD clause | §11.7, §18.1 |
| Status | Draft v1 (2026-04-17) |

## 1. Overview

The Code & Build Management slice is a **read-only observability viewer** for integrated GitHub repositories and GitHub Actions pipelines, with Story↔Commit↔Build traceability and AI assistance (workspace summary, failure triage, PR review assist). The slice does not mutate GitHub state, does not trigger builds, and does not merge PRs. AI output is advisory.

## 2. Routing

| Path | View | Notes |
| ---- | ---- | ----- |
| `/code-build` | `CatalogView` | Workspace-scoped repo catalog + AI summary |
| `/code-build/repos/:repoId` | `RepoDetailView` | 6-card repo detail |
| `/code-build/repos/:repoId/prs/:prNumber` | `PrDetailView` | PR metadata + AI review notes |
| `/code-build/runs/:runId` | `RunDetailView` | Pipeline run detail + AI triage |
| `/code-build/traceability` | `TraceabilityView` | Inverse lookup by Story-Id |

All paths are guarded by `requireWorkspaceMember(workspaceId)`; workspace is resolved from the path parameter's owning entity (repo→project→workspace, run→repo→project→workspace).

No SPA route for `/api/v1/code-build-management/**` — those are backend-only.

## 3. Functional Scope

Numbered subsystems. Each maps to one or more projections / services in the architecture doc.

1. **Catalog Summary** — aggregate counts + 7-day rollups across visible repos.
2. **Catalog Grid** — grouped list of repo tiles with filter + search.
3. **Catalog AI Summary** — workspace-scoped narrative + evidence chips.
4. **Repo Header** — identity, default branch, push timestamp, GitHub deep-link.
5. **Repo Branches** — 20 most-recent branches with ahead/behind, PR presence.
6. **Repo Open PRs** — open PRs with CI status + AI review note counts.
7. **Repo Recent Commits** — 30 most-recent default-branch commits with Story chips.
8. **Repo Recent Runs** — 20 most-recent pipeline runs with state and duration.
9. **Repo AI Insights** — latest narrative summary for this repo.
10. **PR Metadata** — PR details, reviewers, linked stories (union of commit + body refs).
11. **PR AI Review** — severity-grouped advisory notes keyed by head commit.
12. **Run Header** — run metadata (pipeline, trigger, conclusion, duration).
13. **Run Timeline** — job/step timeline with per-step outcome.
14. **Run Artifacts** — artifact list (metadata + GitHub download deep-link).
15. **Run AI Triage** — failed-run root-cause card with evidence and candidate owners.
16. **Open Incident From Run** — navigation action to Incident slice with pre-filled context.
17. **Traceability Inverse View** — commits/PRs/runs for a given Story-Id.
18. **Requirement Slice Code Tab** — the same inverse view embedded into Story Detail.
19. **Webhook Ingestion** — verify, parse, upsert, emit domain events.
20. **Backfill / Resync** — 14-day install backfill + 72h nightly reconciliation.
21. **AI Autonomy Gating** — observes workspace `aiAutonomyLevel` for every AI-invoking action.

## 4. Behavior Rules

- **B1** — Story-Id extraction is trailer-based (`Story-Id: STORY-…`) with no branch-name fallback.
- **B2** — PR body `Relates-to: STORY-…` references count; multiple references allowed per PR or commit.
- **B3** — Story link rows are persisted even when the Story-Id does not resolve in the Requirement slice; the row carries `status=UNKNOWN_STORY` and is shown as `UNVERIFIED` in the UI.
- **B4** — AI PR review runs are keyed by (prId, headCommitSha, skillVersion); advancing the head commit does NOT delete stale rows — it marks them STALE and triggers a new run (debounced 20s).
- **B5** — AI PR review never posts comments to GitHub. Ever. Enforced at the network layer (no GitHub write-scope tokens for this subsystem).
- **B6** — AI failure triage is generated only for FAILED runs; SUCCEEDED runs never produce triage rows.
- **B7** — A re-run with a new conclusion supersedes the prior triage row; the prior row is marked `SUPERSEDED` and kept for audit.
- **B8** — AI triage output is rejected at service time if its `failingStepRef` does not match one of the actual step IDs in the run (evidence-integrity check); placeholder is surfaced in UI.
- **B9** — Workspace AI summary is regenerated at most every 10 minutes; manual regen admin-only and rate-limited to 1/min.
- **B10** — Webhook signatures are verified with the GitHub App secret; invalid signature → 401 + audit + no mutation.
- **B11** — Workspace isolation is enforced at the repository level; every read projection resolves `repo → project → workspace` and filters against the caller's workspace membership.
- **B12** — Secret patterns in log excerpts are redacted before storage and before AI prompting; AWS keys, GitHub tokens, generic bearer tokens.
- **B13** — BLOCKER-severity AI PR review notes are visible only to PM/Tech Lead on the repo's project; others see aggregated count.
- **B14** — AI-generated content (summaries, triage, PR review) never carries write authority; actions triggered from AI output (e.g., Open Incident) pass through explicit user consent.
- **B15** — The "Stories this build contains" range expansion is capped at 100 commits per run; exceeding the cap surfaces a banner.

## 5. Error Codes

All errors follow the shared envelope: `{ error: { code, message, details? }, meta }`.

| Code | HTTP | When |
| ---- | ---- | ---- |
| `CB_WORKSPACE_FORBIDDEN` | 403 | Caller lacks membership in the workspace resolved from the path |
| `CB_ROLE_REQUIRED` | 403 | Action requires PM/Tech Lead (e.g. BLOCKER note body, AI regen) |
| `CB_REPO_NOT_FOUND` | 404 | `repoId` does not exist or is outside workspace scope |
| `CB_RUN_NOT_FOUND` | 404 | `runId` does not exist or outside scope |
| `CB_PR_NOT_FOUND` | 404 | `prId`/`prNumber` does not exist or outside scope |
| `CB_STORY_NOT_FOUND` | 404 | Traceability lookup story does not resolve |
| `CB_AI_AUTONOMY_INSUFFICIENT` | 409 | Workspace autonomy level below required floor |
| `CB_AI_UNAVAILABLE` | 503 | AI skill client failure / timeout (retryable) |
| `CB_RATE_LIMITED` | 429 | Too many requests (especially AI regen or scan) |
| `CB_GH_RATE_LIMIT` | 503 | Upstream GitHub primary/secondary rate limit |
| `CB_INGEST_SIGNATURE_INVALID` | 401 | Webhook signature verification failed |
| `CB_INGEST_PAYLOAD_INVALID` | 400 | Webhook payload fails parsing or schema validation |
| `CB_INSTALL_UNKNOWN` | 404 | Webhook references an unknown installation id |
| `CB_TRIAGE_EVIDENCE_MISMATCH` | 422 | AI triage output did not pass evidence integrity check |
| `CB_RANGE_TOO_LARGE` | 422 | Build range exceeds 100-commit cap (REQ-CB-46) |

## 6. Decisions

- **D1** — Observability-only in V1 (no build triggering, no PR merging, no pipeline edit). Documented in REQ §3 and in architecture §Decisions. Rationale: scope matches PRD §18.1 without creating write-authority ambiguity against upstream SCM.
- **D2** — Integration surface limited to GitHub + GitHub Actions. Other SCM/CI systems deferred. Rationale: single adapter dramatically simplifies ingestion, auth, and rate-limit handling for V1; adapter interface is designed to permit future adapters.
- **D3** — Primary traceability is Story→Commit→Build. Commit→Incident traceability is deferred. Rationale: product asked for product-intent traceability first; incident linkage is provided via the Open-Incident navigation only.
- **D4** — AI scope: summary + triage + PR review assist. No code generation, no auto-fix, no auto-comment. AI never has merge authority. Rationale: matches scope answer; keeps governance surface small.
- **D5** — Webhook-driven ingestion with 14-day install backfill and 72h nightly resync. No polling-only mode. Rationale: freshness SLO requires webhooks; resync protects against dropped events without masking divergence.
- **D6** — GitHub tokens live in Platform Center vault; Control Tower DB never stores them. Rationale: security boundary; also lets Platform Center handle token rotation centrally.
- **D7** — Log excerpts surfaced to UI are redacted; original raw logs are never persisted in Control Tower. Rationale: minimize blast radius for any data breach.
- **D8** — AI PR review never writes to GitHub. No write-scope GitHub token is issued for this subsystem. Rationale: governance simplicity + reversibility; humans can forward AI notes manually if desired.
- **D9** — AI output keyed by skill version and rotated on head-commit advance (for PR review) or run conclusion change (for triage). Prior rows kept as SUPERSEDED/STALE for audit. Rationale: explainability and reproducibility.
- **D10** — Unresolved Story-Id links are persisted with `status=UNKNOWN_STORY`; a nightly resolver job auto-heals when the story becomes visible. Rationale: eventual consistency with the Requirement slice without losing the linkage intent.

## 7. Dependencies

| Dependency | Type | Use |
| ---------- | ---- | --- |
| Requirement slice `SpecRevisionLookup`-style `StoryLookup` facade | read | Story-Id verification, inverse lookup |
| Platform Center secret vault | read | GitHub App private key, webhook secret |
| Workspace service | read | Workspace membership check |
| Project service | read | repo→project→workspace resolution |
| Member service | read | Candidate-owner derivation for triage |
| Shared `AuditLogEmitter`, `LineageEmitter` | write | Audit + lineage for every write path |
| Shared AI skill runtime (`AiSkillClient`) | call | Summary, triage, PR review generation |
| Shared shell (`AppShell`, `shellUiStore`) | UI | Breadcrumbs + AI panel |

## 8. Scope boundaries explicitly restated

- No write calls to GitHub (other than webhook-ACKs, which are read-side).
- No streaming log viewer in V1 — excerpt-only.
- No polling mode fallback — if webhooks are unavailable, the slice surfaces a banner and relies on nightly resync only.
- No custom SCM adapters in V1.
- No AI-authored comments or commits anywhere.

## 9. Glossary

- **Run** — a GitHub Actions workflow run (one execution of one workflow file on one commit/branch/ref).
- **Job** — a parallel unit within a run.
- **Step** — an ordered unit within a job.
- **Check Run** — the check-suite representation of a run (used for CI status on PRs).
- **Installation** — a GitHub App installation on an organization, scoping which repos are visible.
- **Triage** — AI-generated root-cause analysis for a failed run.
- **PR Review Note** — AI-generated advisory comment on a PR diff.
- **Head Commit** — the tip commit of a PR's source branch.
- **Story-Id** — a stable identifier of a user story from the Requirement slice (e.g., `STORY-4211`).
