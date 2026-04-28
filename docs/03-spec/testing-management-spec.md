# Testing Management — Spec

## Source

| Field | Value |
| ----- | ----- |
| Source Requirements | [testing-management-requirements.md](../01-requirements/testing-management-requirements.md) |
| Source Stories | [testing-management-stories.md](../02-user-stories/testing-management-stories.md) |
| PRD clause | §11.8, §10, §6.4, §6.5, §7.4, §7.5, §7.7, §8 |
| Status | Draft v1 (2026-04-18) |

## 1. Overview

The Testing Management slice is a **read-write observability and governance hub** for test plans, test cases, test runs, and AI-assisted test-case drafting, with full traceability to requirements (REQ-IDs / Story-Ids). The slice ingests read-only test results from external runners (JUnit, TestNG, Playwright, Cypress), manages test plans and cases, publishes AI-drafted candidate cases for human approval, and serves coverage views to downstream slices (Requirements, Dashboard, Incident). Test execution happens externally; the Control Tower does not trigger runs.

## 2. Routing

| Path | View | Notes |
| ---- | ---- | ----- |
| `/testing` | `CatalogView` | Workspace-scoped test plan catalog + aggregate summary |
| `/testing/plans/:planId` | `PlanDetailView` | 6-card plan detail: Header, Cases, Coverage, Recent Runs, AI Draft Inbox, AI Insights |
| `/testing/plans/:planId/cases/:caseId` | `CaseDetailView` | Case metadata + linked REQ chips + run history sparkline + 20 most-recent outcomes |
| `/testing/plans/:planId/runs/:runId` | `RunDetailView` | Run metadata + per-case outcomes + environment context + story coverage rollup |
| `/testing/traceability/:reqId` | `TraceabilityView` | Inverse lookup: all ACTIVE cases linked to a REQ-ID, their plans, and most-recent run outcomes |

All paths guarded by `requireWorkspaceMember(workspaceId)`; workspace resolved from path parameter's owning entity (plan→project→workspace, case→plan→project→workspace, run→plan→project→workspace).

No SPA route for `/api/v1/testing-management/**` — backend-only.

## 3. Functional Scope

Numbered subsystems. Each maps to one or more projections / services in the architecture doc.

1. **Catalog Summary** — aggregate counts + 7-day rollups across visible plans.
2. **Catalog Grid** — grouped list of plan tiles with filter, search, and coverage LED.
3. **Plan Header** — identity, state, owner, release target, timestamps, owning project lineage.
4. **Plan Cases** — list of ACTIVE + DRAFT + DEPRECATED cases with ID, title, type, priority, state, linked REQ chips, and last-run status.
5. **Plan Coverage** — inverse map REQ-ID → linked case count → latest-run status aggregation (GREEN/AMBER/RED/GREY).
6. **Plan Recent Runs** — 20 most-recent runs with ID, environment, trigger source, state, duration, pass/fail/skip counts, actor.
7. **Plan AI Draft Inbox** — all DRAFT cases (origin=AI_DRAFT) with source REQ-ID, skill version, timestamp, and approve/reject/edit affordances.
8. **Plan AI Insights** — narrative summary of plan health: cases trending red, REQs losing coverage, changes in last 7 days.
9. **Case Metadata** — title, type, priority, state, owner, preconditions, steps, expected result (all markdown-rendered, sanitized).
10. **Case REQ Linkage** — zero-or-many linked REQ-IDs with verification status (GREEN / AMBER / RED / UNVERIFIED).
11. **Case Defect Linkage** — zero-or-many linked Incident IDs.
12. **Case Run History** — sparkline + table of 20 most-recent outcomes with status (PASS/FAIL/SKIP/ERROR), duration, run ID, environment.
13. **Case Revision Log** — inline drawer listing all edits (actor, timestamp, field diff).
14. **Run Metadata** — plan, environment, trigger source (manual upload / CI webhook), actor, start/end timestamps, duration, overall state (RUNNING/PASSED/FAILED/ABORTED).
15. **Run Outcomes** — per-case status (PASS/FAIL/SKIP/ERROR) + failure excerpt (capped 4 KB, redacted).
16. **Run Failure Context** — "last-good" timestamp, failing assertion, one-click "Open Incident" action.
17. **Run Environment Metadata** — environment name, description, kind, URL, not credentials.
18. **Run Story Coverage** — union of REQ-IDs from all ACTIVE cases in run, capped at 200 REQs.
19. **Run Ingestion Webhook** — verify HMAC-SHA256, parse JUnit/TestNG/Playwright/Cypress, persist outcomes, emit events.
20. **Run Ingestion Upload** — manual file upload via Control Tower UI by QA Lead, same persistence path as webhook.
21. **AI Test Case Drafting** — invoke `test-case-drafter` skill from REQ-ID, persist as DRAFT, rate-limit 20/REQ/day/workspace.
22. **AI Draft Approval** — QA Lead / Tech Lead transitions DRAFT → ACTIVE, records audit.
23. **AI Draft Rejection** — transitions DRAFT → DEPRECATED, records audit.
24. **Traceability Inverse View** — REQ-ID → all ACTIVE linked cases, their plans, most-recent outcomes across all visible plans.
25. **Unresolved REQ-ID Resolver** — hourly job retries UNKNOWN_REQ status rows, updates to valid when Requirement slice resolves them.
26. **Environment Registry** — per-workspace CRUD for DEV/STAGING/PROD/EPHEMERAL/OTHER environments with metadata.
27. **AI Autonomy Gating** — observes workspace `aiAutonomyLevel` for every AI-invoking action.
28. **Workspace + Role Isolation** — enforces membership checks and role-based CRUD authority.

## 4. Behavior Rules

### Plan & Case Management

- **B1** — Test Plan state machine: `DRAFT → ACTIVE → ARCHIVED`. Only ACTIVE plans' cases count toward workspace-level coverage. Only users with QA Lead / Tech Lead / PM role may create/archive plans (REQ-TM-83).
- **B2** — Test Case state machine: `DRAFT (AI-drafted) → ACTIVE → DEPRECATED`. Only ACTIVE cases count toward plan coverage, pass-rate, and traceability views. DRAFT and DEPRECATED cases remain visible in history and Plan Detail Case list, but not in public coverage.
- **B3** — A DRAFT case shall NOT be executable in a run, shall NOT appear in Coverage card, shall NOT appear in Traceability inverse view. Only visible in Plan Detail "AI Draft Inbox" card and in full Case list with visual distinction (REQ-TM-23, REQ-TM-52).
- **B4** — A DEPRECATED case carries a deprecation badge, does not count toward coverage, remains visible for historical audit and run linkage (REQ-TM-24, REQ-TM-54).
- **B5** — Editing a case captures revision entry (actor, timestamp, field diff). Revision list accessible in a side drawer. Both ACTIVE and DRAFT cases support edits (REQ-TM-25).
- **B6** — When a case's linked REQ-IDs change, the Plan's Coverage card and the Requirement slice's inverse-view "Tests" tab shall reflect the change within 30s at P95 via event emission (REQ-TM-26).
- **B7** — Case type is one of: FUNCTIONAL, REGRESSION, SMOKE, PERF, SECURITY. Priority is one of: P0, P1, P2, P3. Both are immutable after case creation; changes only via deprecation + new case.
- **B8** — Case markdown (preconditions, steps, expected) shall be rendered through a sanitized renderer: no raw HTML, code fences permitted (REQ-TM-21).

### Test Run Ingestion

- **B9** — Run state machine: `INGESTING → PASSED | FAILED | ABORTED`. Once a run reaches terminal state, it is immutable (REQ-TM-33). Re-ingestion with the same `externalRunId` is a 409 conflict unless `force=true` is passed by an admin, which supersedes and audits the override.
- **B10** — Run ingestion shall support JUnit XML, TestNG XML, Playwright JSON, and Cypress Mochawesome JSON formats. Parser failures persist the run with state `INGEST_FAILED`, capturing first 2 KB of error (REQ-TM-74).
- **B11** — Run ingestion accepts either: (a) manual upload via Control Tower UI by QA Lead with form upload, or (b) signed webhook from external CI with HMAC-SHA256 signature (REQ-TM-35, REQ-TM-85).
- **B12** — Webhook signatures verified using per-workspace shared secret (issued by Platform Center); invalid signature → 401 + audit + no mutation (REQ-TM-85).
- **B13** — Run environment name is required; if unknown, auto-create with `kind=OTHER` and surface in Platform diagnostics for operator reclassification (REQ-TM-61).
- **B14** — Failure excerpts captured from run ingestion are redacted for common secret patterns (AWS keys, GitHub tokens, generic bearer tokens) before storage (REQ-TM-80).
- **B15** — Run per-case outcomes are: PASS, FAIL, SKIP, ERROR. Failure excerpts are capped at 4 KB per case and redacted before storage.
- **B16** — A FAILED run's case outcomes compute "last-good" timestamp by looking back to the most recent prior run where that case passed. Used in Run Detail and Open-Incident context (REQ-TM-31).

### REQ-ID ↔ Case ↔ Run Traceability

- **B17** — Every TestCase persists zero-or-many TestCaseReqLink rows. Each row names a REQ-ID / Story-Id and carries a status: `VERIFIED` (REQ found in Requirement slice), `UNKNOWN_REQ` (REQ not yet resolved), or `UNVERIFIED` (lookup failed). Links are author-maintained in UI; ingestion does not infer links (REQ-TM-40, REQ-TM-43).
- **B18** — A Validated REQ-ID is one that exists in the Requirement slice and is visible to the same workspace. Unknown REQ-IDs are persisted with `status=UNKNOWN_REQ` and surfaced on Plan Detail as unresolved-links diagnostics. An hourly resolver job retries them (REQ-TM-43).
- **B19** — A REQ-ID's coverage status is computed as:
  - `GREEN`: at least one ACTIVE linked case PASSED in its most-recent run in the last 7d.
  - `AMBER`: most-recent linked run is older than 7d OR mixed PASS/FAIL in the most-recent run.
  - `RED`: most-recent linked run FAILED or ERROR.
  - `GREY`: no runs yet.
  (REQ-TM-44)
- **B20** — Traceability Inverse View for a given REQ-ID lists all ACTIVE cases linked to it, their owning plans, and each case's most-recent run outcome across all visible plans. Scoped to caller's workspace and visible projects (REQ-TM-41).
- **B21** — When a case's linked REQ-IDs change, Traceability inverse views and Requirement slice's "Tests" tab shall be invalidated and regenerated within 30s at P95 (REQ-TM-26, REQ-TM-42).

### AI Test Case Drafting (Human-Approval Gated)

- **B22** — From a REQ-ID, only users with QA Lead or Tech Lead role in the owning project may invoke `POST /ai/test-cases/draft`, which returns a list of candidate cases produced by the `test-case-drafter` skill (REQ-TM-50, REQ-TM-112).
- **B23** — Candidate cases are persisted as TestCase rows with `state=DRAFT` and `origin=AI_DRAFT`. They appear in Plan Detail "AI Draft Inbox" card and nowhere else in active views (REQ-TM-51).
- **B24** — Only QA Lead or Tech Lead on the owning project may approve a DRAFT. Approval transitions `state=ACTIVE` and records an AuditLog entry (REQ-TM-53). Edits before approval are allowed and recorded as revisions.
- **B25** — Rejecting a DRAFT transitions `state=DEPRECATED` with `deprecationReason=REJECTED_AT_INBOX`; rejected drafts remain visible in history but are excluded from all active views (REQ-TM-54).
- **B26** — AI drafts are keyed by (`reqId`, `skillVersion`, `planId`). When the skill version advances, prior drafts for the same key render with a STALE badge. An admin may trigger a re-draft (REQ-TM-55).
- **B27** — AI drafts honor the workspace's `aiAutonomyLevel`:
  - `DISABLED` → draft endpoint returns 403.
  - `OBSERVATION` → draft runs and results visible, Approve button disabled (read-only preview).
  - `SUPERVISED` / `AUTONOMOUS` → normal flow.
  - AI never has authority to transition a case to ACTIVE directly, regardless of autonomy level (REQ-TM-56).
- **B28** — AI drafts cite the REQ text they consumed; the Draft Inbox surfaces the source excerpt so the human reviewer can verify grounding (REQ-TM-57).
- **B29** — Draft endpoint is rate-limited: at most 20 drafts per REQ per day per workspace. UI surfaces remaining quota (REQ-TM-58).
- **B30** — AI test-case-drafter prompts shall not include failure-excerpt secrets or environment credentials; only REQ text + prior case titles are in-scope (REQ-TM-81).

### Environment Registry

- **B31** — Per-workspace Environment registry with `name` (unique within workspace), `description`, `kind` (DEV / STAGING / PROD / EPHEMERAL / OTHER), `url` (optional, for deep-link), and `archived` flag. Archived environments reject new runs (REQ-TM-60, REQ-TM-62).
- **B32** — Run ingestion must name a registered environment; unknown environment names auto-create with `kind=OTHER` and surface in Platform diagnostics for operator reclassification (REQ-TM-61).

### Empty States and Degradation

- **B33** — Empty state text must be distinguishable: "no plans yet in this workspace", "no cases in this plan", "no runs yet for this plan", "no AI drafts pending", "AI drafts disabled for this workspace", "AI draft generation failed" (REQ-TM-71).
- **B34** — Per-card section error with retry on Catalog, Plan Detail, Case Detail, and Run Detail pages. No page-level wipeout for a single card failure (REQ-TM-70, REQ-TM-18).
- **B35** — Stale AI drafts carry a visible STALE badge and a "re-draft" action for QA Leads (REQ-TM-72).
- **B36** — When Requirement slice lookup fails, REQ-ID chips on cases render as `UNVERIFIED` (grey) rather than hiding — absence of verification must never look the same as absence of linkage (REQ-TM-73).
- **B37** — When run ingestion parse fails, run is persisted in state `INGEST_FAILED` with first 2 KB of parser output captured. Operator may re-upload after fixing; a FAILED ingestion never silently partial-writes case outcomes (REQ-TM-74).

### Governance and Audit

- **B38** — Every run ingestion, AI invocation, and case state transition emitted via shared `AuditLogEmitter` with full context (actor, workspace, plan, case, change). Admin may override run immutability with `force=true`, audited as ADMIN_OVERRIDE (REQ-TM-82).
- **B39** — All write paths (plan CRUD, case CRUD, run ingestion, AI draft persistence, approvals, rejections) emit `LineageEvent` naming Testing Management as authoring domain (REQ-TM-84).
- **B40** — Only QA Lead and Tech Lead roles may approve/reject AI drafts. Only QA Lead, Tech Lead, PM may create/archive plans. Engineers and other readers have read-only access (REQ-TM-83).
- **B41** — Workspace isolation enforced at plan level; every read projection resolves `plan → project → workspace` and filters against caller's workspace membership (REQ-TM-06).

## 5. State Machines

### Plan State

```
DRAFT ──(activate)──> ACTIVE ──(archive)──> ARCHIVED
  |                     |                      |
  └─────────(delete)────┴──────────────────────┘
```

- DRAFT: editable, not visible in public catalog summary, cases not counted.
- ACTIVE: visible, cases counted, can transition to ARCHIVED.
- ARCHIVED: read-only, no new cases, no new runs, visible in history views.

### Case State

```
DRAFT (AI-drafted) ──(approve)──> ACTIVE ──(deprecate)──> DEPRECATED
        |                           |          (auto)          |
        ├─(reject)───────────────────────────────────────────>|
        |
        └─(edit)──> DRAFT (revised)
```

- DRAFT: not counted, not executable, only in AI Draft Inbox.
- ACTIVE: counted in coverage, executable in runs, visible in all views.
- DEPRECATED: not counted, not executable, visible in history for audit.

### Run State

```
INGESTING ──(complete)──> PASSED
    |                        |
    ├─(complete)──> FAILED   |
    |                  |      |
    ├─(abort)──> ABORTED     |
    |                        |
    └─(parse error)──> INGEST_FAILED
```

- INGESTING: temporary, results incomplete.
- PASSED / FAILED / ABORTED: terminal, immutable unless admin `force=true`.
- INGEST_FAILED: terminal but permits re-upload after fixing parser error.

## 6. Error Codes

All errors follow shared envelope: `{ error: { code, message, details? }, meta }`.

| Code | HTTP | When | REQ |
| ---- | ---- | ---- | --- |
| `TM_WORKSPACE_FORBIDDEN` | 403 | Caller lacks membership in workspace resolved from path | REQ-TM-06 |
| `TM_ROLE_REQUIRED` | 403 | Action requires QA Lead / Tech Lead / PM role | REQ-TM-83 |
| `TM_PLAN_NOT_FOUND` | 404 | `planId` does not exist or outside workspace scope | — |
| `TM_CASE_NOT_FOUND` | 404 | `caseId` does not exist or case outside plan scope | — |
| `TM_RUN_NOT_FOUND` | 404 | `runId` does not exist or outside plan scope | — |
| `TM_UNKNOWN_REQ` | 404 | Traceability lookup REQ-ID does not resolve | REQ-TM-41 |
| `TM_ENVIRONMENT_NOT_FOUND` | 404 | Environment name not registered in workspace | REQ-TM-61 |
| `TM_RUN_CONFLICT` | 409 | `externalRunId` already ingested; use `force=true` to override | REQ-TM-33 |
| `TM_AI_AUTONOMY_INSUFFICIENT` | 409 | Workspace autonomy level below required floor (DISABLED) | REQ-TM-56 |
| `TM_DRAFT_RATE_LIMIT_EXCEEDED` | 429 | 20 drafts per REQ per day per workspace quota exceeded | REQ-TM-58 |
| `TM_AI_UNAVAILABLE` | 503 | AI skill client failure / timeout (retryable) | — |
| `TM_INGEST_SIGNATURE_INVALID` | 401 | Webhook signature verification failed | REQ-TM-85 |
| `TM_INGEST_PAYLOAD_INVALID` | 400 | Webhook payload fails parsing or schema validation | — |
| `TM_INGEST_PARSE_ERROR` | 422 | Run file parsing (JUnit/TestNG/Playwright/Cypress) failed | REQ-TM-74 |
| `TM_INGEST_PARTIAL_FAILURE` | 207 | Some cases parsed, others failed; full details in `details.failures` | — |
| `TM_ENVIRONMENT_ARCHIVED` | 410 | Archived environment cannot accept new runs | REQ-TM-62 |

## 7. Decisions

- **D1** — Observability-only in V1; the Control Tower does not trigger, re-run, or cancel external tests. Rationale: scope matches §11.8 without creating execution-authority ambiguity against external runners.
- **D2** — Run format support limited to JUnit, TestNG, Playwright, Cypress in V1. Other formats deferred. Rationale: covers 80% of common CI ecosystems; adapter interface designed for future extensibility.
- **D3** — AI scope: test-case generation from REQ-ID only, human-approval gated. No auto-healing, no risk-prioritization, no coverage-gap AI in V1. Rationale: matches autonomy-level governance; keeps approval surface small.
- **D4** — Primary traceability is REQ-ID ↔ Case ↔ Run. Incident linkage is manual "Open Incident" action only (not auto-create). Rationale: preserves human judgment in defect lifecycle; incident slice owns incident creation authority.
- **D5** — Webhook-driven run ingestion; manual upload via UI as fallback. No polling mode. Rationale: freshness SLO requires push; UI fallback covers edge cases and ad-hoc testing.
- **D6** — Failure excerpts never include credentials (redacted before storage). AI prompts never receive redacted excerpts. Rationale: minimize security blast radius; protect secret patterns from AI models.
- **D7** — DRAFT cases (AI-drafted, pre-approval) are first-class but excluded from all coverage, pass-rate, and traceability views. Rationale: prevents half-baked cases from biasing metrics; clear visual distinction in UI.
- **D8** — Unresolved REQ-ID links (status=UNKNOWN_REQ) persist and are surfaced as diagnostics. Hourly resolver job auto-heals when Requirement slice later resolves them. Rationale: eventual consistency without losing linkage intent.
- **D9** — AI drafts keyed by (reqId, skillVersion, planId) and marked STALE on skill-version advance. Prior drafts kept for audit. Rationale: explainability and reproducibility; avoids silent degradation.
- **D10** — Workspace AI autonomy level gates draft endpoint and Approve button (OBSERVATION → read-only). AI never has authority to approve/activate cases directly. Rationale: governance boundary clarity; human always signs off on case activation.

## 8. Dependencies

| Dependency | Type | Use | REQ |
| ---------- | ---- | --- | --- |
| Requirement slice StoryLookup facade | read | REQ-ID / Story-Id verification, inverse lookup | REQ-TM-40, REQ-TM-43 |
| Platform Center secret vault | read | Per-workspace webhook HMAC-SHA256 shared secret | REQ-TM-85 |
| Platform Center AI autonomy config | read | Workspace `aiAutonomyLevel` (DISABLED/OBSERVATION/SUPERVISED/AUTONOMOUS) | REQ-TM-56 |
| Workspace service | read | Workspace membership check | REQ-TM-06 |
| Project service | read | plan→project→workspace resolution | REQ-TM-06 |
| Member service | read | Owner derivation for cases / plans | — |
| Shared `AuditLogEmitter`, `LineageEmitter` | write | Audit + lineage for every write path | REQ-TM-82, REQ-TM-84 |
| Shared AI skill runtime (`AiSkillClient`) | call | test-case-drafter skill invocation | REQ-TM-50 |
| Shared shell (`AppShell`, `shellUiStore`) | UI | Breadcrumbs + card error boundaries | REQ-TM-18 |

## 9. Non-Functional Requirements

### Performance (NFR) — REQ-TM-92

- Catalog aggregate P95 load budget: **1200ms** (includes summary bar + grid load).
- Plan Detail P95: **1500ms** (parallel card load with timeout fallback; per-projection timeout 500ms with `SectionResult` fallback).
- Run Detail P95: **1500ms**.
- Case Detail P95: **800ms** (smaller scope).
- No card shall block page render; use async `SectionResult` + error boundary fallback.

### Size & Bundle (NFR) — REQ-TM-93

- No card shall ship more than **300 KB** of gzipped JS on critical path.
- Run failure-excerpt rendering (if large) must be virtualized; do not render all 20 outcomes upfront.
- Markdown sanitization must be performed server-side before transit; client never parses untrusted HTML.

### Accessibility (NFR) — REQ-TM-90

- All cards shall meet shared shell's a11y bar: keyboard navigation (Tab, Arrow, Enter), ARIA roles (`role=table`, `role=button`), color-blind-safe LEDs with shape/text redundancy (not color-only).
- Coverage LED: green + checkmark, amber + dash, red + X, grey + dash.
- All buttons must be focusable and labeled.

### Internationalization (NFR) — REQ-TM-91

- All text shall use shared i18n token set; no hard-coded English in components.
- Date formatting deferred to i18n library; timestamps always include timezone.
- Coverage LED labels must be i18n tokens: `coverage.green`, `coverage.amber`, `coverage.red`, `coverage.grey`.

## 10. Validation Rules

### TestPlan Validation

- `name`: required, 1–255 chars, unique within project.
- `state`: one of DRAFT, ACTIVE, ARCHIVED.
- `releaseTarget`: optional, string, max 128 chars.
- `owner`: required, must be a workspace member.
- `projectId`: required, must be accessible to caller.
- Before state transition to ARCHIVED, any DRAFT cases must be rejected or promoted.

### TestCase Validation

- `title`: required, 1–255 chars.
- `type`: one of FUNCTIONAL, REGRESSION, SMOKE, PERF, SECURITY.
- `priority`: one of P0, P1, P2, P3.
- `preconditions`: optional, markdown, max 8 KB.
- `steps`: required (ordered markdown list), max 16 KB.
- `expectedResult`: required, markdown, max 8 KB.
- `owner`: required, must be a workspace member.
- `state`: one of DRAFT, ACTIVE, DEPRECATED.
- `linkedReqs`: zero-or-many REQ-ID / Story-Id strings, validated at link time.
- `planId`: required, must be accessible to caller.
- On DRAFT→ACTIVE promotion, at least one REQ-ID link is recommended (warning if zero).

### TestRun Validation

- `planId`: required.
- `externalRunId`: required, unique within plan.
- `environment`: required, must be registered in workspace; auto-create with kind=OTHER if unknown.
- `triggeredBy`: one of MANUAL_UPLOAD, CI_WEBHOOK.
- `actor`: required, must be workspace member.
- `caseOutcomes`: zero-or-many per-case outcomes (case ID + status + duration + excerpt).
- `excerpt`: per-case, max 4 KB, redacted before storage.
- All REQ-ID chips derived from case linkage (not inferred from run content).

### TestCaseReqLink Validation

- `caseId`: required.
- `reqId`: required, validated against Requirement slice.
- `status`: one of VERIFIED, UNKNOWN_REQ, UNVERIFIED.
- On creation, status=UNKNOWN_REQ pending hourly resolver job.
- Duplicates (same caseId+reqId) are idempotent; update if exists, insert if new.

### Environment Validation

- `name`: required, 1–128 chars, unique within workspace.
- `kind`: one of DEV, STAGING, PROD, EPHEMERAL, OTHER.
- `description`: optional, max 512 chars.
- `url`: optional, must be valid URL if provided.
- `archived`: boolean, default false.

## 11. Out of Scope / Deferred to V1.1+

- Running tests from the Control Tower (no execution authority).
- Editing CI pipeline configuration.
- Auto-triaging failures into incidents (manual "Open Incident" only).
- AI-driven coverage-gap analysis and risk-based test prioritization.
- AI test-case auto-healing and flaky-case auto-detection.
- Cross-workspace analytics and reporting.
- Per-test-plan ACL beyond workspace membership.
- Rich step editors with screenshot/attachment inline editing (markdown body only in V1).
- Test plan approval workflows beyond simple ACTIVE/ARCHIVED state.
- "My tests" personalized lens (deferred to V1.1).
- Streaming log viewer (excerpt-only in V1).
- Custom test format adapters beyond JUnit/TestNG/Playwright/Cypress.

## 12. Glossary

- **TestPlan** — a logical grouping of test cases scoped to a project and release target; first-class object with state (DRAFT/ACTIVE/ARCHIVED).
- **TestCase** — an executable test procedure with preconditions, steps, expected result, linked REQ-IDs, and state (DRAFT/ACTIVE/DEPRECATED).
- **TestRun** — a single execution outcome captured from external runner; immutable once ingested; carries per-case status, duration, failure excerpts.
- **Case Outcome** — per-case result in a run: PASS, FAIL, SKIP, ERROR.
- **DRAFT Case** — AI-generated candidate case, pre-approval, excluded from coverage and pass-rate.
- **ACTIVE Case** — human-approved case, counted in coverage and pass-rate, executable in runs.
- **DEPRECATED Case** — rejected AI draft or archived case, not counted, visible in history for audit.
- **Coverage** — REQ-ID → count of linked ACTIVE cases → most-recent run status aggregation (GREEN/AMBER/RED/GREY).
- **REQ-ID** — stable identifier from Requirement slice (e.g., STORY-4211).
- **Verified / Unknown / Unverified** — link status: Verified (REQ found), Unknown (lookup pending), Unverified (lookup failed).
- **Skill Version** — AI skill semantic version; advance triggers re-draft opportunity.
- **AI Autonomy Level** — workspace governance setting: DISABLED (no AI), OBSERVATION (read-only preview), SUPERVISED (QA approves), AUTONOMOUS (QA approves, same flow).
- **Traceability** — bidirectional linkage: REQ-ID ↔ TestCase ↔ TestRun; inverse view shows "all cases testing a given REQ-ID".
- **Failure Excerpt** — first 4 KB of case failure log, redacted for secrets before storage.
- **Last-Good Timestamp** — most-recent prior run where the case passed; shown in failed case context.
- **External Run ID** — CI system's run identifier (Jenkins build number, GitHub Actions run ID, etc.); unique within plan.
- **Webhook Signature** — HMAC-SHA256(payload, workspace_secret); verified on every CI webhook ingestion.
