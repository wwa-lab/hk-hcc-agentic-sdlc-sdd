# Feature Specification: Design Management

| Attribute | Value |
|-----------|-------|
| Slice | design-management |
| Source Requirements | [design-management-requirements.md](../01-requirements/design-management-requirements.md) |
| Source Stories | [design-management-stories.md](../02-user-stories/design-management-stories.md) |
| PRD | [agentic_sdlc_control_tower_prd_v0.9.md](../01-requirements/agentic_sdlc_control_tower_prd_v0.9.md) §11.6 |

## Overview

Design Management is the **design traceability plane** of the Agentic SDLC Control Tower. V1 is a **lightweight, read-only viewer** over internal Stitch-authored HTML mocks, with one narrow admin write path (register + link). The primary flow is Spec → Design traceability: a reader starts from a Spec and verifies which design artifacts cover it.

The slice is scoped to:

- **Catalog** — browsable list of design artifacts registered for the current Workspace
- **Viewer** — per-artifact view with embedded sandboxed preview, linked-Spec strip, AI summary, change history
- **Traceability** — Spec-centric view that exposes coverage gaps
- **AI Command Panel projection** — summarize, extract key elements, flag linkage gaps (no critique, no generation)

It is **not** scoped to design authoring/editing, review workflow (comments, approvals), version diff, design token management, design-to-code handoff tracking, AI critique, AI generation, Figma integration, arbitrary image/PDF upload, or cross-Workspace rollup (Report Center).

## Source Stories

| Story | Description |
|-------|-------------|
| S1 | Browse Design Artifact Catalog |
| S2 | Read Catalog Summary Counters |
| S3 | Filter and Search the Catalog |
| S4 | Drill Into an Artifact from the Catalog |
| S5 | Inspect an Artifact with Embedded Preview |
| S6 | See Which Specs This Artifact Covers |
| S7 | Read the AI Summary of an Artifact |
| S8 | Review Change History for an Artifact |
| S9 | Open the Artifact in a New Tab |
| S10 | See Which Specs Have No Design |
| S11 | Deep-Link into Traceability from Requirement Management |
| S12 | Link Existing Artifacts to a Spec (Admin) |
| S13 | Use the AI Command Panel in Design Management Context |
| S14 | Enforce Role-Based Access |
| S15 | Render Loading / Empty / Error States Gracefully |
| S16 | Register a New Design Artifact (Admin) |
| S17 | Publish a New Version of an Artifact (Admin) |
| S18 | Deep-Link into Design Management from Other Slices |
| S19 | Preserve Shell Chrome and Context |
| S20 | Observability and Correlation Tracking |

## Actors / Users

| Actor | Primary Role | Access |
|-------|--------------|--------|
| Architect / Tech Lead | Viewer consumer; primary reviewer of artifacts | Read within Workspace |
| Developer | Opens artifacts while implementing against a Spec | Read within Workspace or Project |
| Project Manager | Checks coverage during milestone planning | Read within Workspace |
| QA Lead / UX Reviewer | Prepares test plans and heuristic reviews | Read within Workspace |
| Workspace Admin | Registers artifacts, publishes versions, links/unlinks Specs, triggers AI regeneration | Read + Write within Workspace |
| Auditor | Consumes full Change History for compliance | Read-only including history |
| AI Skill Runtime | Produces summaries, key-element extractions, linkage-gap audits | System actor |
| Platform Governance Pipeline | Audit + policy enforcement | System actor |

## Functional Scope

### Catalog View (Workspace-scoped, read-heavy)

1. **Catalog Summary Bar** (REQ-DM-10, REQ-DM-11, REQ-DM-15)
2. **Catalog Filters & Search** (REQ-DM-13, REQ-DM-06)
3. **Catalog Rows** (REQ-DM-12, REQ-DM-14)

### Viewer View (Artifact-scoped, read-heavy)

4. **Artifact Header** (REQ-DM-21)
5. **Sandboxed Preview** (REQ-DM-20, REQ-DM-27, REQ-DM-61)
6. **Linked-Spec Strip** (REQ-DM-22, REQ-DM-25)
7. **AI Summary Panel** (REQ-DM-23, REQ-DM-40, REQ-DM-41, REQ-DM-42)
8. **Change History Timeline** (REQ-DM-24)
9. **Context Chips** (REQ-DM-26)

### Traceability View (Workspace-scoped, read-heavy with one admin write)

10. **Spec-centric list** (REQ-DM-30, REQ-DM-32)
11. **Deep-link from Requirement Management** (REQ-DM-31)
12. **Bulk Link-design modal** (REQ-DM-33, admin only)

### Admin Write Paths (narrow, write-light)

13. **Register Artifact** (Story S16, REQ-DM-50, REQ-DM-53)
14. **Publish New Version** (Story S17, REQ-DM-93)
15. **Link / Unlink Spec** (Story S12, REQ-DM-33, REQ-DM-51)
16. **Regenerate AI Summary** (Story S7, REQ-DM-41)

### Cross-Cutting

17. **Role-based access** (REQ-DM-50, REQ-DM-52)
18. **Audit trail** (REQ-DM-51)
19. **Section envelope and error isolation** (REQ-DM-62, REQ-DM-72)
20. **Shell integration** (REQ-DM-83)
21. **Observability** (REQ-DM-63)

## Routing

| Path | View | Purpose |
|------|------|---------|
| `/design-management` | Catalog | Workspace-scoped catalog; reads `workspaceId` from context bar |
| `/design-management?project=...&kind=...&coverage=...&q=...` | Catalog (filtered) | Query params preserve filters across deep-links and refresh |
| `/design-management/artifacts/:artifactId` | Viewer | Single-artifact viewer |
| `/design-management/artifacts/:artifactId/raw` | Raw preview | Full-screen HTML render (no chrome) |
| `/design-management/traceability` | Traceability | Spec-centric coverage list |
| `/design-management/traceability?specId=...` | Traceability (focused) | Single-Spec traceability from deep-link |

All primary paths render inside the shared app shell and participate in the breadcrumb pipeline (REQ-DM-83). The `/raw` route does not render shell chrome.

## Contracts

The full API contract lives in [../05-design/contracts/design-management-API_IMPLEMENTATION_GUIDE.md](../05-design/contracts/design-management-API_IMPLEMENTATION_GUIDE.md). This section summarizes:

### Catalog view endpoints (all read)

| Method | Path | Purpose |
|--------|------|---------|
| GET | `/api/v1/design-management/catalog?workspaceId=...&project=...&kind=...&coverage=...&q=...&sort=...` | Catalog rows |
| GET | `/api/v1/design-management/catalog/summary?workspaceId=...` | Summary Bar counters |

### Viewer view endpoints (all read except admin writes)

| Method | Path | Purpose |
|--------|------|---------|
| GET | `/api/v1/design-management/artifacts/{id}` | Aggregate viewer payload |
| GET | `/api/v1/design-management/artifacts/{id}/header` | Artifact header |
| GET | `/api/v1/design-management/artifacts/{id}/preview` | Sandbox preview HTML (text/html with CSP) |
| GET | `/api/v1/design-management/artifacts/{id}/links` | Linked-Spec strip with coverage |
| GET | `/api/v1/design-management/artifacts/{id}/ai-summary` | Cached AI summary |
| GET | `/api/v1/design-management/artifacts/{id}/history?page=0&pageSize=25` | Change history |

### Traceability view endpoints

| Method | Path | Purpose |
|--------|------|---------|
| GET | `/api/v1/design-management/traceability?workspaceId=...&coverage=...&project=...` | Spec-centric coverage list |
| GET | `/api/v1/design-management/traceability?specId=...` | Single-Spec focused traceability |
| GET | `/api/v1/design-management/coverage/summary?projectId=...` | (cross-slice) coverage summary for Project Management |

### Admin write endpoints

| Method | Path | Purpose |
|--------|------|---------|
| POST | `/api/v1/design-management/artifacts` | Register a new artifact (v1 published on creation) |
| POST | `/api/v1/design-management/artifacts/{id}/versions` | Publish a new version |
| PATCH | `/api/v1/design-management/artifacts/{id}/lifecycle` | Change lifecycle stage (DRAFT → READY_FOR_REVIEW → APPROVED → DEPRECATED) |
| POST | `/api/v1/design-management/artifacts/{id}/links` | Link one or more Specs |
| DELETE | `/api/v1/design-management/artifacts/{id}/links/{specId}` | Unlink a Spec |
| POST | `/api/v1/design-management/artifacts/{id}/ai-summary/regenerate` | Force regenerate |

### Internal endpoints

| Method | Path | Purpose |
|--------|------|---------|
| POST | `/api/v1/design-management/internal/ai-summaries` | AI runtime writes a completed summary |

## Data Contracts

### Artifact Kind (enum)

`PAGE_MOCK` | `COMPONENT_MOCK` | `FLOW_MOCK` | `STATE_MOCK`

### Lifecycle Stage (enum)

`DRAFT` | `READY_FOR_REVIEW` | `APPROVED` | `DEPRECATED`

Lifecycle transitions are linear with a single backward transition permitted (`APPROVED` → `DEPRECATED`). Other transitions are blocked by `DM_INVALID_LIFECYCLE_TRANSITION`.

### Coverage Status (enum)

`OK` | `PARTIAL` | `STALE` | `MISSING`

Computed server-side at request time — never stale-read from cache. See REQ-DM-25 for semantics.

### DesignArtifact (catalog row shape, excerpt)

```json
{
  "artifactId": "da-2001",
  "workspaceId": "ws-42",
  "projectId": "proj-8821",
  "title": "Control Tower Dashboard",
  "kind": "PAGE_MOCK",
  "lifecycleStage": "APPROVED",
  "authors": [{ "memberId": "mem-101", "name": "Yaozi Xiao" }],
  "currentVersionId": "dav-2001-v3",
  "currentVersionTag": "v3",
  "lastUpdatedAt": "2026-04-16T11:22:00Z",
  "linkedSpecCount": 4,
  "worstCoverageStatus": "STALE",
  "aiSummaryReady": true,
  "previewThumbnailUrl": "/api/v1/design-management/artifacts/da-2001/thumbnail"
}
```

### DesignArtifactVersion (immutable)

```json
{
  "versionId": "dav-2001-v3",
  "artifactId": "da-2001",
  "versionTag": "v3",
  "htmlHash": "sha256:...",
  "htmlSizeBytes": 712450,
  "publishedAt": "2026-04-16T11:22:00Z",
  "publishedByMemberId": "mem-101",
  "changeNote": "Aligned header with updated Spec REQ-DASH-020"
}
```

### DesignSpecLink

```json
{
  "linkId": "dsl-440",
  "artifactId": "da-2001",
  "specId": "SPEC-DASH-020",
  "coversRevision": 5,
  "declaredCoverage": "FULL",
  "currentCoverageStatus": "STALE",
  "linkedAt": "2026-04-10T09:00:00Z",
  "linkedByMemberId": "mem-101"
}
```

### DesignAiSummary

```json
{
  "summaryId": "dai-2001-v3-a",
  "artifactId": "da-2001",
  "versionId": "dav-2001-v3",
  "skillName": "artifact-summarizer",
  "skillVersion": "1.3.0",
  "summary": "A dashboard mock surfacing SDLC chain health, incident counters, and AI-pending review queue...",
  "keyElements": [
    "SDLC chain strip (11 nodes)",
    "Incident counter group",
    "AI Command Panel slot",
    "Workspace switcher"
  ],
  "generatedAt": "2026-04-16T11:23:15Z",
  "advisoryOnly": false
}
```

## Behavior Rules

### B1: Workspace isolation

Every read and write resolves the caller's authorized Workspaces via `DesignAccessGuard`. Any attempt to access an artifact outside the caller's authorized set returns `403` with code `DM_WORKSPACE_FORBIDDEN`. The artifact must not appear in any list response to that caller.

> REQ-DM-11, REQ-DM-52

### B2: Role gating

| Action | Required Role |
|--------|--------------|
| Read catalog, viewer, traceability | `WORKSPACE_READER` (or `PROJECT_READER` scoped to the artifact's Project) |
| Register artifact | `WORKSPACE_ADMIN` |
| Publish new version | `WORKSPACE_ADMIN` |
| Link / unlink Spec | `WORKSPACE_ADMIN` |
| Regenerate AI summary | `WORKSPACE_ADMIN` |
| Change lifecycle stage | `WORKSPACE_ADMIN` |
| Read full Change History | `WORKSPACE_READER` + `AUDITOR` (auditor sees all fields; reader sees public fields) |

> REQ-DM-50

### B3: Lifecycle state machine

Allowed transitions:

- `DRAFT` → `READY_FOR_REVIEW`
- `READY_FOR_REVIEW` → `APPROVED`
- `READY_FOR_REVIEW` → `DRAFT` (return-for-rework)
- `APPROVED` → `DEPRECATED`

Blocked transitions return `DM_INVALID_LIFECYCLE_TRANSITION`. V1 does not model review threads or approvals; `READY_FOR_REVIEW` → `APPROVED` is a direct admin action with an audit entry.

> REQ-DM-07

### B4: Coverage computation

Coverage status per `DesignSpecLink` is computed at read time from `(link.coversRevision, spec.latestRevision, spec.state, link.declaredCoverage)`:

- If `spec.state` is `ARCHIVED` or `DELETED` → `MISSING`
- Else if `link.declaredCoverage` is `PARTIAL` → `PARTIAL`
- Else if `link.coversRevision < spec.latestRevision` → `STALE`
- Else → `OK`

Computation is cheap; no server-side cache.

> REQ-DM-25

### B5: PII rejection on registration / version publish

Server validates the HTML payload against platform-defined PII regexes (email, national-ID). Any match blocks the write with `DM_PII_DETECTED` and a list of match offsets (no payload content logged). Admin must remove PII and retry.

> REQ-DM-53

### B6: Size enforcement

Artifacts > 2 MB uncompressed are rejected on write with `DM_ARTIFACT_TOO_LARGE`. On read, if the stored artifact exceeds 2 MB (e.g., grandfathered), the Viewer renders "Open in new tab" in place of the inline iframe.

> REQ-DM-90, REQ-DM-61

### B7: Sandbox headers

The preview endpoint (`GET /api/v1/design-management/artifacts/{id}/preview`) responds with:

```
Content-Type: text/html; charset=utf-8
Content-Security-Policy: default-src 'none'; img-src data: blob:; style-src 'unsafe-inline' cdn.tailwindcss.com fonts.googleapis.com fonts.gstatic.com; script-src cdn.tailwindcss.com; font-src fonts.gstatic.com; frame-ancestors 'self'
X-Frame-Options: SAMEORIGIN
X-Content-Type-Options: nosniff
Referrer-Policy: no-referrer
```

CSP is conservative by default; admins cannot relax CSP per-artifact in V1.

> REQ-DM-20

### B8: AI summary cache keying

The summary cache is keyed by `(artifactId, versionId, skillName, skillVersion)`. Cache invalidates when a new version is published (the key naturally rotates). The regenerate action forces a new skill run with the current `skillVersion` and supersedes the prior cache entry for display.

> REQ-DM-93, REQ-DM-41

### B9: Section envelope

Every aggregate read response uses the shared `SectionResult<T>` envelope per section, so partial failures return HTTP 200 with per-section `ERROR` entries and a correlationId.

```json
{
  "status": "OK | LOADING | EMPTY | ERROR",
  "data": { ... } | null,
  "error": { "code": "...", "message": "...", "correlationId": "..." } | null,
  "fetchedAt": "ISO-8601"
}
```

> REQ-DM-62, REQ-DM-72

### B10: AI autonomy gating

AI actions (summarize, extract, linkage-gap audit) check the Workspace's AI autonomy level against the skill's required level at invocation time. If below, the action returns `DM_AI_AUTONOMY_INSUFFICIENT` and the UI renders the panel in "advisory-only" mode.

> REQ-DM-42

### B11: Idempotency

- `POST /artifacts` — not idempotent; dedup is advisory via title+project+kind (see Story S16)
- `POST /artifacts/{id}/versions` — requires `prevVersionId` as a fencing token; stale token → `DM_STALE_VERSION`
- `POST /artifacts/{id}/links` — idempotent per `(artifactId, specId)`; re-posting the same Spec link returns the existing link unchanged
- `DELETE /artifacts/{id}/links/{specId}` — idempotent
- `POST /artifacts/{id}/ai-summary/regenerate` — idempotent by design; repeated calls produce the same cache entry if the underlying version and skill version haven't changed

## Non-Functional Targets

| Metric | Target |
|--------|--------|
| Catalog first-paint P95 | ≤ 1,200 ms (Workspace ≤ 200 artifacts) |
| Viewer first-paint P95 (excluding iframe) | ≤ 800 ms |
| Iframe render for artifact ≤ 2 MB | ≤ 3 s from Viewer first-paint |
| AI Summary cached response P95 | ≤ 200 ms |
| AI Summary cold (first generation) P95 | ≤ 15 s |
| Availability (read) | 99.5% monthly |
| Max artifact size | 2 MB uncompressed (hard reject on write) |
| Max catalog size per Workspace | 200 artifacts (V1 design target) |

> REQ-DM-60, REQ-DM-61, REQ-DM-62, REQ-DM-90, REQ-DM-91

## Error Code Registry

| Code | HTTP | Meaning |
|------|------|---------|
| `DM_WORKSPACE_FORBIDDEN` | 403 | Caller lacks Workspace access |
| `DM_PROJECT_FORBIDDEN` | 403 | Caller lacks Project-scoped access |
| `DM_ROLE_REQUIRED` | 403 | Caller role insufficient for the requested action |
| `DM_ARTIFACT_NOT_FOUND` | 404 | Artifact does not exist or is not visible |
| `DM_SPEC_NOT_FOUND` | 404 | Spec does not exist or is not visible |
| `DM_PII_DETECTED` | 422 | Registration / version payload contains PII |
| `DM_ARTIFACT_TOO_LARGE` | 413 | Payload exceeds 2 MB |
| `DM_STALE_VERSION` | 409 | Concurrent version publish rejected |
| `DM_INVALID_LIFECYCLE_TRANSITION` | 409 | Requested lifecycle transition not allowed |
| `DM_DUPLICATE_ARTIFACT` | 409 | Duplicate registration advisory (soft warning, not hard block) |
| `DM_AI_AUTONOMY_INSUFFICIENT` | 403 | AI skill requires higher Workspace autonomy level |
| `DM_AI_SKILL_UNAVAILABLE` | 503 | Skill runtime unreachable or rate-limited |

## Decisions Recorded

### D1: V1 is read-heavy by design

V1 ships as a lightweight viewer. The only write paths are administrative: register, publish new version, link/unlink Spec, regenerate AI summary, change lifecycle stage. Design editing, review threads, version diff, and AI critique / generation are explicitly deferred.

> REQ-DM-07, product decision 2026-04-17

### D2: Stitch / internal HTML is the only artifact source in V1

V1 does not integrate with Figma or other external design tools and does not accept arbitrary image/PDF uploads. Registration accepts either an inline HTML payload or a reference to a `docs/05-design/*.html` path. Any other source is out of scope.

> REQ-DM-07, product decision 2026-04-17

### D3: Spec-centric traceability is the primary flow

The Traceability view and the Linked-Spec strip on the Viewer are the primary UX surfaces. Catalog browsing is secondary — valuable, but the page is organized around the Spec → Design question first.

> REQ-DM-02, product decision 2026-04-17

### D4: AI is strictly advisory; no apply actions in V1

AI produces summaries, key-element extractions, and linkage-gap audits. There is no "apply AI suggestion" action. AI never mutates artifact metadata or links on its own.

> REQ-DM-05, REQ-DM-40, product decision 2026-04-17

### D5: New entities are minimal

Only four net-new entities: `DesignArtifact`, `DesignArtifactVersion`, `DesignSpecLink`, `DesignAiSummary`. No duplication of Project, Spec, Member, or Workspace. See [design-management-data-model.md](../04-architecture/design-management-data-model.md).

> REQ-DM-04, CLAUDE.md Lesson #3

### D6: Sandbox is conservative

Preview iframes use a default CSP that blocks inline scripts and restricts network origins. Per-artifact CSP relaxation is not V1. The Stitch mocks in `docs/05-design/` currently reference `cdn.tailwindcss.com` and `fonts.googleapis.com`; the default CSP explicitly allows these.

> REQ-DM-20, product decision 2026-04-17

## Dependencies

| Upstream | Contract |
|----------|----------|
| `shared-app-shell` | AppShell.vue, context bar, AI Command Panel, `SectionResult<T>`, `fetchJson<T>`, shell stores |
| `requirement` | Spec identity, `/requirement/specs/{specId}` route, Spec visibility rules |
| `project-space` | Project entity (read), Project visibility rules |
| Platform (future stub) | Artifact-kind taxonomy, lifecycle vocabulary, AI autonomy level |
| AI Skill Runtime | `artifact-summarizer`, `key-element-extractor`, `linkage-gap-auditor` skills |

## Out of V1 Scope (again, for emphasis)

- Design authoring or editing inside the product
- Review workflow (comments, threads, approvals, sign-offs)
- Version diff
- AI design critique (a11y, consistency, pattern)
- AI design generation
- Figma or external design-tool integration
- Arbitrary image/PDF upload
- Design token management
- Component library catalog
- Design-to-code drift detection
- Cross-Workspace rollup
