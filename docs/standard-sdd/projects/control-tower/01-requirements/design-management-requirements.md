# Design Management Requirements

## Purpose

This document extracts the requirements relevant to the **Design Management** page from the full PRD (V0.9). It defines **what** the Design Management page must deliver in V1, serving as the upstream input for the spec, architecture, design, and task documents of this slice.

Design Management is the **design traceability plane**. Where Requirement Management (PRD §11.4) owns "what should we build", Design Management (PRD §11.6) owns "how did we visualize and structure it, and does the design actually cover the spec it claims to cover?"

Per the 2026-04-17 scope decision, V1 is intentionally **lightweight** (PRD §18.2 — "可轻量化上线"):

- Read-only design artifact viewer
- Integrates with internal Stitch-authored HTML mocks already present under `docs/standard-sdd/projects/control-tower/05-design/*.html`
- Primary user flow: **Spec → Design traceability** (start from a Requirement/Spec, see the designs that claim to cover it, verify coverage)
- AI scope: summary + traceability only (no critique, no generation)

Editing, review workflow, design tokens management, component library catalog, AI critique, and design-to-code handoff tracking are explicitly out of V1 scope and deferred to later releases.

## Source

All requirements below are derived from:

- [agentic_sdlc_control_tower_prd_v0.9.md](agentic_sdlc_control_tower_prd_v0.9.md) — §11.6, §11.4, §13, §15, §16, §18.2
- Visual reference: the internal Stitch-authored HTML mocks in `docs/standard-sdd/projects/control-tower/05-design/` — [Control Tower.html](../05-design/Control%20Tower.html), [Incident Command Center.html](../05-design/Incident%20Command%20Center.html), [Platform Center.html](../05-design/Platform%20Center.html), [Project Space.html](../05-design/Project%20Space.html), plus [design.md](../05-design/design.md) (product-level visual system) and [stitch-brief.md](../05-design/stitch-brief.md)
- Cross-slice references: [requirement-requirements.md](requirement-requirements.md), [project-space-requirements.md](project-space-requirements.md), [project-management-requirements.md](project-management-requirements.md), [shared-app-shell-requirements.md](shared-app-shell-requirements.md)

Section references use the format `PRD §N.N`.

---

## 1. Scope

The Design Management page is described in the PRD as follows:

> "用于管理 Architecture 和 Design 资产，包括设计文档、评审记录、接口设计、影响分析和设计与 Spec 的对应关系。该页面应支持从上游 Spec 追踪到设计输出，并支持设计与实现的一致性校验。"
> — PRD §11.6

Translation: manages Architecture and Design assets including design documents, review records, interface designs, impact analysis, and design-to-Spec correspondence. The page should support tracing from upstream Spec to design output, and support consistency validation between design and implementation.

### 1.1 In Scope (V1 — lightweight viewer)

**Artifact Catalog view** (Workspace-scoped):

- Design Artifact Catalog — browsable list of internal Stitch/HTML mock artifacts registered for the Workspace
- Artifact metadata (title, authors, last-updated, lifecycle stage, owning Project, linked Spec IDs, preview thumbnail)
- Filter by Project, status, owner, linked Spec, artifact kind
- Deep link into the per-artifact Viewer

**Artifact Viewer view** (single artifact):

- Embedded preview of the Stitch/HTML artifact (rendered inside a sandboxed iframe)
- Artifact header (title, kind, lifecycle stage, authors, last-updated, version tag)
- Linked Spec strip — Spec IDs this artifact claims to cover, each with a jump-in link to Requirement Management
- Coverage indicator per linked Spec (OK / PARTIAL / MISSING / STALE), derived server-side
- Linked Project / Milestone / Requirement context
- AI Summary panel (AI-generated one-paragraph summary + bulleted key elements, pulled from AI Skill Runtime)
- Change history (read-only timeline of `artifact_version` entries)
- Export link (open artifact HTML in a new tab for full-screen review)

**Spec → Design traceability view** (from a Spec):

- Given a Spec ID, list all design artifacts that claim to cover it
- Show coverage status per artifact
- "Missing design" indicator — Spec rows with zero linked artifacts rendered in crimson
- Entry from Requirement Management via deep-link

**AI Command Panel projection** (Design Management context):

- Summarize current artifact
- Extract key UI elements from current artifact
- Flag Spec → Design linkage gaps for the current Workspace
- No critique, no generation, no editing

**Cross-cutting:**

- Workspace isolation (PRD §9.2)
- Role-based visibility (Workspace reader, Project reader)
- Audit log read-only view (registration / linkage / AI summary events)
- Consistent shell chrome (context bar, nav, AI Command Panel) per REQ-SAS-*

### 1.2 Out of Scope (V1)

- Design artifact authoring, editing, or in-place mutation (V1 is read-only)
- Design review / approval workflow (comments, sign-offs, review threads) — deferred
- Version comparison / diff between artifact versions — deferred
- Design token management and component library catalog — deferred (lives in Platform Center or a future Design Ops slice)
- AI design critique (a11y, consistency, pattern alignment) — deferred
- AI design generation (generate variant mocks, generate HTML from a Spec) — explicitly not V1
- Figma or external design-tool integrations — not V1 (V1 is internal Stitch / HTML mocks only)
- Design-to-code handoff tracking and drift detection — deferred to Code & Build slice
- Cross-Workspace artifact rollup — Report Center responsibility
- Raw upload of arbitrary images / PDFs as artifacts — not V1 (artifacts must be registered HTML mocks)
- Live collaborative editing — not applicable to V1 viewer
- Impact analysis graph across design artifacts — deferred
- Permission grants / user management — Platform Center

---

## 2. Product Context Requirements

### REQ-DM-01: Design traceability plane

Design Management is the system's **design traceability plane** — the primary page where a reader inspects a Stitch/HTML design artifact, sees which Specs it claims to cover, and verifies the coverage is current. It is read-heavy; mutations in V1 are limited to registering an artifact and linking it to Specs.

> Source: PRD §11.6

### REQ-DM-02: Spec-centric traceability (primary flow)

The primary V1 user flow is **Spec → Design**: a reader starts on a Requirement or Spec and navigates to the designs that cover it. The inverse flow (Design → Spec) is supported but is not the primary optimization target.

> Source: PRD §11.6, §7.6 (Spec-as-hub principle), product decision 2026-04-17

### REQ-DM-03: V1 audience

The page must serve the following V1 audiences:

- **Architect / Tech Lead** — primary reader, inspects design artifacts to align spec cadence with design cadence.
- **Project Manager** — checks "does this Spec have a design?" during milestone planning.
- **Developer** — opens a design artifact while implementing against a Spec.
- **QA Lead / UX Reviewer** — reads design artifacts to prepare test plans or heuristic review.
- **Workspace Admin** — registers artifacts and links them to Specs.

Designer-as-editor is intentionally not a V1 audience (V1 is read-only).

> Source: PRD §6.1, §6.2, §6.4, §11.6

### REQ-DM-04: Reuse-and-extend domain model

The page must reuse existing domain entities (Workspace, Project, Requirement/Spec, Member/Role) defined upstream. New entities introduced by this slice are limited to:

- `DesignArtifact` — catalog entry pointing at a Stitch/HTML mock
- `DesignArtifactVersion` — immutable version snapshot
- `DesignSpecLink` — linkage between a `DesignArtifact` and a Spec ID, with a coverage field
- `DesignAiSummary` — cached AI summary keyed by artifact version

No duplicate Project, Spec, or Member entities.

> Source: CLAUDE.md Lesson #3 (package-by-feature), requirement-data-model.md

### REQ-DM-05: AI participates visibly

Any AI-surfaced content on Design Management (artifact summary, key-element extraction, linkage-gap flag) must explicitly identify AI involvement — what the AI produced, when it was produced, the source artifact version, and a "regenerate" action. AI output never silently overwrites human-authored metadata.

> Source: PRD §7.4 — "AI 是参与者，不是旁观者"

### REQ-DM-06: Configuration-driven layout

Card order, card visibility, and default filter values on both the Catalog view and the Viewer view must be configuration-driven so a Workspace can tune Design Management without code changes.

> Source: PRD §7.8

### REQ-DM-07: Lightweight V1 posture

V1 intentionally ships as a read-only viewer. Any feature that implies editing, review workflow, version diff, or AI critique is deferred. The page must render usefully even if the only thing it shows is "here are the registered HTML mocks for this Workspace, and here is their Spec coverage."

> Source: PRD §18.2, product decision 2026-04-17

### REQ-DM-08: Spec-centric vocabulary preserved

The SDLC chain representation on Design Management breadcrumbs and the Delivery Progress context strip (if shown) must continue to emphasize Spec as the execution hub.

> Source: PRD §7.6, §13.1

---

## 3. Artifact Catalog View Requirements

### REQ-DM-10: Catalog summary counters

The Catalog view header must display, scoped to the active Workspace:

- Total registered design artifacts
- Artifacts with at least one linked Spec
- Artifacts with zero linked Specs ("Orphan designs")
- Specs with zero linked designs in this Workspace ("Missing designs")
- Artifacts with STALE coverage (design version predates linked Spec's latest revision)
- Last catalog refresh timestamp

> Source: PRD §11.6

### REQ-DM-11: Workspace-scoping

All counters and rows must strictly honor the current Workspace context from the shared context bar. Cross-Workspace leakage is a defect.

> Source: PRD §9.2, §15.2

### REQ-DM-12: Artifact row shape

Each catalog row must show:

- Preview thumbnail (server-rendered static snapshot of the HTML mock, if available; otherwise a neutral placeholder)
- Artifact title
- Artifact kind (one of: `PAGE_MOCK`, `COMPONENT_MOCK`, `FLOW_MOCK`, `STATE_MOCK`)
- Lifecycle stage (`DRAFT` / `READY_FOR_REVIEW` / `APPROVED` / `DEPRECATED`)
- Owning Project (link)
- Authors (list of Members)
- Last-updated timestamp
- Linked Spec count + worst coverage status across those links
- AI-summary availability chip ("AI Summary ready" / "AI Summary pending")

> Source: PRD §11.6, REQ-DM-05

### REQ-DM-13: Filter and sort

The catalog must support:

- Filter by owning Project
- Filter by lifecycle stage
- Filter by artifact kind
- Filter by coverage status (OK / PARTIAL / MISSING / STALE / NO_LINKS)
- Filter by author
- Text search on title
- Sort by title, last-updated, linked Spec count, coverage status

> Source: PRD §11.6, §7.8

### REQ-DM-14: Drill-in

Clicking a catalog row must open the Artifact Viewer at `/design-management/artifacts/:artifactId`.

### REQ-DM-15: AI linkage-gap flag

The catalog header must surface an AI-generated "linkage gap" summary when available — a short natural-language statement (≤ 200 chars) such as *"3 Specs in this Workspace have no linked design. 2 approved artifacts have stale Spec links."* The AI text must never replace the structured counters in REQ-DM-10; it is advisory.

> Source: PRD §7.4, REQ-DM-05

---

## 4. Artifact Viewer Requirements

### REQ-DM-20: Sandboxed embed

The artifact preview must render the registered Stitch/HTML mock inside a sandboxed `<iframe>` with `sandbox="allow-scripts allow-same-origin"` removed where possible; the frame must not be able to script the parent window or navigate the top-level location.

> Source: Security posture, PRD §16.4 (audit & isolation principles)

### REQ-DM-21: Artifact header

The viewer header must show:

- Title + artifact kind chip
- Lifecycle stage chip
- Owning Project (link)
- Authors
- Version tag (e.g., `v3.2`) and "version as-of" timestamp
- Registration timestamp + registered-by
- "Open in new tab" action (full-screen preview)
- "Regenerate AI summary" action (admin only)

### REQ-DM-22: Linked-Spec strip

Immediately below the header, the viewer must render a strip of linked Spec chips. Each chip shows:

- Spec ID (e.g., `SPEC-REQ-042`)
- Spec title (truncated)
- Coverage status badge (OK / PARTIAL / MISSING / STALE)
- Jump-in link to the Spec in Requirement Management

If the artifact has zero linked Specs, the strip renders a crimson "No Spec links" empty state with a "Link a Spec" admin action.

> Source: PRD §11.6, REQ-DM-02

### REQ-DM-23: AI Summary panel

The viewer must render an AI Summary panel containing:

- One-paragraph natural-language summary of the artifact (≤ 500 chars)
- Bulleted list of extracted key UI elements (≤ 10 items)
- Generated-at timestamp and source artifact version
- "Regenerate" action (admin only)
- Explicit AI attribution badge

If the AI summary is unavailable or pending, the panel renders a skeleton with a "Generate summary" CTA for admins.

> Source: PRD §7.4, REQ-DM-05

### REQ-DM-24: Change history

The viewer must show a read-only timeline of version events for the artifact:

- Timestamp, actor (human Member or AI skill execution), event type (`REGISTERED` / `VERSION_PUBLISHED` / `LINKED_SPEC` / `UNLINKED_SPEC` / `AI_SUMMARY_GENERATED` / `LIFECYCLE_CHANGED`), and a short human-readable description.

Pagination: 25 entries per page.

> Source: PRD §16.3 (audit principles)

### REQ-DM-25: Coverage indicator semantics

Per linked Spec, the viewer must compute and display a coverage status:

- `OK` — artifact version is at or after the Spec's latest revision AND the artifact's `coversRevision` matches
- `PARTIAL` — artifact's self-declared coverage is "partial" (author-declared)
- `STALE` — artifact version predates the Spec's latest revision
- `MISSING` — the Spec was deleted or archived but the link still exists (data integrity flag)

Coverage is computed server-side at request time (not cached beyond the `SectionResult` envelope).

> Source: PRD §11.6 — "设计与实现的一致性校验"

### REQ-DM-26: Context chips

The viewer must render context chips linking out to:

- Owning Project (Project Space)
- Primary linked Milestone (if any)
- Linked Requirements (Requirement Management)
- AI Command Panel (scoped to this artifact)

### REQ-DM-27: Responsive behavior

The viewer must render usefully at ≥ 1280px width. Below 1024px, the Linked-Spec strip and AI Summary panel collapse into a tabbed presentation under the preview. Mobile (<768px) is out of V1 scope.

> Source: design.md — desktop-first constraint

---

## 5. Spec → Design Traceability Requirements

### REQ-DM-30: Spec-centric traceability page

The slice must expose a route `/design-management/traceability?workspaceId=...` that lists, for the active Workspace:

- All Specs in scope (joined from Requirement Management)
- For each Spec: the list of linked Design Artifacts, each with coverage status
- Specs with zero linked Designs are rendered in crimson and float to the top by default

> Source: PRD §11.6, REQ-DM-02

### REQ-DM-31: Deep-link from Requirement Management

Requirement Management must be able to deep-link into this page pre-filtered to a single Spec ID via `/design-management/traceability?specId=...`. This deep-link is owned by Requirement Management; Design Management only guarantees the URL contract.

> Source: product decision 2026-04-17

### REQ-DM-32: Filters

The traceability page must support filter by coverage status (OK / PARTIAL / MISSING / STALE / NO_LINKS) and by owning Project.

### REQ-DM-33: Bulk "Link design" action (admin only)

From a Spec row with zero linked artifacts, an admin may open a compact linker modal that allows selecting one or more existing registered artifacts in the Workspace and linking them to the Spec. The modal is V1 (admin-only write path); general design editing remains out of V1.

> Source: product decision 2026-04-17 — allow minimal linkage writes so the catalog has a way to get populated

---

## 6. AI Command Panel Requirements

### REQ-DM-40: Contextual AI scope

The shared AI Command Panel, when focused on Design Management, must support the following capabilities only:

- Summarize the current artifact
- Extract key UI elements from the current artifact
- Enumerate Spec-linkage gaps for the current Workspace

No generation, no critique, no drafting, no editing-style suggestions in V1.

> Source: REQ-DM-05, PRD §15.4, product decision 2026-04-17

### REQ-DM-41: Provenance and regeneration

Every AI-produced output must show: generating skill name, skill version, source artifact version, generated-at timestamp, and a "regenerate" affordance (admin only). Outputs are cached per artifact version; regeneration creates a new cache entry and supersedes the prior one for display.

> Source: REQ-DM-05, PRD §7.4

### REQ-DM-42: Skill governance

AI skills used by Design Management (artifact-summarizer, key-element-extractor, linkage-gap-auditor) must be registered in Platform Center's Skill Catalog and honor the Workspace's AI autonomy level. If autonomy is below the required threshold, AI output renders in "advisory-only" mode (no "apply" action).

> Source: PRD §11.13, §7.4

---

## 7. Governance, Audit, Isolation Requirements

### REQ-DM-50: Role-based access

Access rules for V1:

- `WORKSPACE_READER` — view Catalog, Viewer, Traceability
- `WORKSPACE_ADMIN` — all reader actions plus register artifact, publish new version, link / unlink Spec, trigger AI regeneration, run bulk link modal
- `PROJECT_READER` — scoped view of artifacts whose `projectId` is in their authorized Project list
- `AUDITOR` — read all, including full change history

Any write path unavailable to the caller must be rendered disabled with a tooltip explaining the required role.

> Source: PRD §10, §16.3

### REQ-DM-51: Audit trail

Every mutation (register, version publish, link, unlink, AI regeneration request, lifecycle change) must be recorded in the Design Management change log with: actor, timestamp, action, artifact-id, version, before/after value, correlationId.

> Source: PRD §16.3

### REQ-DM-52: Workspace isolation

Cross-Workspace reads must fail closed. Artifacts are keyed by Workspace; a caller without access to the owning Workspace must receive `403` on any direct-by-id fetch and the artifact must not appear in any list response.

> Source: PRD §9.2

### REQ-DM-53: PII posture

V1 registered artifacts are HTML mocks; they may contain placeholder names and sample data but should not carry real PII. Backend must reject registration payloads that match platform-defined PII patterns (email, national-ID regexes). Registration is blocked, not sanitized.

> Source: PRD §16.4

---

## 8. Non-Functional Requirements

### REQ-DM-60: First-paint performance

Catalog view first-paint P95 ≤ 1,200 ms on a Workspace with ≤ 200 registered artifacts. Artifact Viewer first-paint P95 ≤ 800 ms excluding iframe render time.

> Source: PRD §16 (performance posture)

### REQ-DM-61: Iframe sandbox performance

Embedded HTML mock must render within 3 s of Viewer first-paint for an artifact ≤ 2 MB. Artifacts > 2 MB render with an intermediary "Open in new tab" CTA instead of inline embed.

### REQ-DM-62: Availability

Design Management page must honor the platform's read-availability target (99.5% monthly), and must degrade gracefully: the page renders with per-card `SectionResult.ERROR` envelopes rather than a whole-page crash when any backend projection fails.

> Source: PRD §16, shared-app-shell-requirements.md REQ-SAS-*

### REQ-DM-63: Observability

Backend must emit structured logs and metrics per request (correlationId, workspaceId, artifactId, latency, cache-hit-flag for AI summary) consistent with the platform observability schema.

> Source: PRD §16.5

---

## 9. Visual & Experience Requirements

### REQ-DM-70: Design system conformance

All Design Management UI must use Tactical Command tokens and primitives defined in [design.md](../05-design/design.md) — crimson / amber / LED palette, `PmCard`-style skeleton / empty / error envelope, JetBrains Mono for IDs, Inter for copy. No bespoke palette.

> Source: design.md, CLAUDE.md Lesson #7

### REQ-DM-71: Empty-state dignity

Empty states (no artifacts registered, no Spec links, no AI summary, no Workspace selected) must be first-class designs, not afterthoughts. Each empty state must explain what the user is looking at and offer a single obvious next action.

> Source: design.md

### REQ-DM-72: Error isolation

Per-card errors must render inside the card's `SectionResult.ERROR` envelope with a human-readable reason and a "retry" action. A failing AI Summary must not prevent the rest of the Viewer from rendering.

> Source: REQ-SAS-* error isolation posture

### REQ-DM-73: AI attribution glyph

Every AI-produced item renders with the platform AI attribution glyph (sparkle + "AI" text) and must never appear indistinguishable from human-authored content.

> Source: PRD §7.4

---

## 10. Cross-Slice Interaction Requirements

### REQ-DM-80: Requirement Management interop

- Each linked Spec chip must deep-link into Requirement Management at the Spec detail route.
- Requirement Management may deep-link into `/design-management/traceability?specId=...`.
- Requirement Management owns Spec identity; Design Management treats Spec IDs as external references.

> Source: requirement-requirements.md, PRD §11.4

### REQ-DM-81: Project Space interop

- Each artifact row's Project chip deep-links to Project Space.
- Project Space's "Design" tab (if/when added) may list artifacts by calling `GET /api/v1/design-management/catalog?projectId=...`.

> Source: project-space-requirements.md

### REQ-DM-82: Project Management interop

- Project Management Plan view may consult `GET /api/v1/design-management/coverage/summary?projectId=...` to surface "missing design" risk signals. V1 of PM does not depend on this call; it is an optional enrichment.

> Source: project-management-requirements.md

### REQ-DM-83: Shared app shell

- Design Management renders inside the shared app shell (REQ-SAS-*): context bar, primary nav, right-side AI Command Panel, breadcrumbs.
- All routes participate in the shell's breadcrumb pipeline.

> Source: shared-app-shell-requirements.md

### REQ-DM-84: Platform Center

- Design Management does not own artifact-kind taxonomy or lifecycle stage vocabulary. These come from Platform Center configuration.
- Design Management must read the current taxonomy at load time; changes in Platform Center must take effect on next page load.

> Source: PRD §11.13

---

## 11. Data Volume & Boundaries

### REQ-DM-90: Artifact size

V1 supports registered HTML mocks up to 2 MB per artifact (uncompressed). Artifacts > 2 MB render via external-open CTA only (REQ-DM-61).

### REQ-DM-91: Catalog size

V1 is designed for Workspaces with up to 200 active artifacts. Larger Workspaces are a V2 pagination concern.

### REQ-DM-92: Version retention

All versions of an artifact are retained indefinitely in V1. Pruning is a V2 concern.

### REQ-DM-93: AI summary cache

AI summary output is cached server-side keyed by `(artifactId, versionId, skillVersion)`. Cache is invalidated on new version publication of the underlying artifact. Regenerate action forces a cache miss.

---

## 12. Requirement Index

| REQ-ID | Area | Summary |
|--------|------|---------|
| REQ-DM-01 | Product Context | Design traceability plane |
| REQ-DM-02 | Product Context | Spec-centric traceability as primary flow |
| REQ-DM-03 | Product Context | V1 audience |
| REQ-DM-04 | Product Context | Reuse-and-extend domain model |
| REQ-DM-05 | Product Context | AI participates visibly |
| REQ-DM-06 | Product Context | Configuration-driven layout |
| REQ-DM-07 | Product Context | Lightweight V1 posture |
| REQ-DM-08 | Product Context | Spec-centric vocabulary |
| REQ-DM-10 | Catalog | Summary counters |
| REQ-DM-11 | Catalog | Workspace scoping |
| REQ-DM-12 | Catalog | Artifact row shape |
| REQ-DM-13 | Catalog | Filter and sort |
| REQ-DM-14 | Catalog | Drill-in |
| REQ-DM-15 | Catalog | AI linkage-gap flag |
| REQ-DM-20 | Viewer | Sandboxed embed |
| REQ-DM-21 | Viewer | Artifact header |
| REQ-DM-22 | Viewer | Linked-Spec strip |
| REQ-DM-23 | Viewer | AI Summary panel |
| REQ-DM-24 | Viewer | Change history |
| REQ-DM-25 | Viewer | Coverage indicator semantics |
| REQ-DM-26 | Viewer | Context chips |
| REQ-DM-27 | Viewer | Responsive behavior |
| REQ-DM-30 | Traceability | Spec-centric traceability page |
| REQ-DM-31 | Traceability | Deep-link from Requirement Management |
| REQ-DM-32 | Traceability | Filters |
| REQ-DM-33 | Traceability | Bulk Link-design modal (admin) |
| REQ-DM-40 | AI | Contextual AI scope |
| REQ-DM-41 | AI | Provenance and regeneration |
| REQ-DM-42 | AI | Skill governance |
| REQ-DM-50 | Governance | Role-based access |
| REQ-DM-51 | Governance | Audit trail |
| REQ-DM-52 | Governance | Workspace isolation |
| REQ-DM-53 | Governance | PII posture |
| REQ-DM-60 | Non-functional | First-paint performance |
| REQ-DM-61 | Non-functional | Iframe sandbox performance |
| REQ-DM-62 | Non-functional | Availability |
| REQ-DM-63 | Non-functional | Observability |
| REQ-DM-70 | Visual | Design system conformance |
| REQ-DM-71 | Visual | Empty-state dignity |
| REQ-DM-72 | Visual | Error isolation |
| REQ-DM-73 | Visual | AI attribution glyph |
| REQ-DM-80 | Interop | Requirement Management |
| REQ-DM-81 | Interop | Project Space |
| REQ-DM-82 | Interop | Project Management |
| REQ-DM-83 | Interop | Shared app shell |
| REQ-DM-84 | Interop | Platform Center |
| REQ-DM-90 | Volume | Artifact size |
| REQ-DM-91 | Volume | Catalog size |
| REQ-DM-92 | Volume | Version retention |
| REQ-DM-93 | Volume | AI summary cache |
