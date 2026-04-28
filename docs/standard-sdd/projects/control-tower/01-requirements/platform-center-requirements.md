# Platform Center Requirements

## Purpose

This document extracts the requirements relevant to the **Platform Center** page from the full PRD (V0.9). Platform Center is the privileged **platform-administration surface** that carries the reusable, governance-critical foundations of the product: templates, configurations, audit, access, policy, and external integrations. It defines **what** the Platform Center must deliver in V1, serving as the upstream input for the spec, architecture, design, and task documents for this slice.

## Source

All requirements below are derived from:

- [agentic_sdlc_control_tower_prd_v0.9.md](agentic_sdlc_control_tower_prd_v0.9.md) — primary PRD
- [visual-design-system.md](../05-design/visual-design-system.md) — visual design system
- [design.md](../05-design/design.md) — product-module design system

Section references use the format `PRD §N.N`.

---

## 1. Scope

Platform Center is the **home of the platform base layer** — the reusable and governance-critical capabilities that every SDLC domain depends on. Per PRD §11.13, it carries the platform's templates, configuration, audit, permission, policy, skill registration, and integration adapter capabilities, and is "the key foundation for the product's reusability and maintainability."

> Source: PRD §11.13 — "Platform Center 是系统可复用与可维护的关键基础。"

### 1.1 V1 Scope

This slice delivers **all six platform capabilities** defined in PRD §12.1 through §12.6, each surfaced in Platform Center as a dedicated section:

1. **Template Management** (PRD §12.1) — Page, flow, project, policy, metric, and AI templates with inheritance, versioning, and override semantics
2. **Configuration Management** (PRD §12.2) — Page, field, component, flow, view, notification, and AI configurations with team/project override support
3. **Audit Management** (PRD §12.3) — Read-only browse of audit records covering configuration changes, permission changes, approvals, AI actions, skill executions, and policy hits
4. **Access Management** (PRD §12.4) — RBAC-first role/permission model with platform, application, workspace, and project scope bindings
5. **Policy & Governance** (PRD §12.5) — Action policies, approval rules, autonomy level defaults, risk thresholds, and exception handling
6. **Integration Framework** (PRD §12.6) — External-system adapter registry (Jira, Confluence, GitLab, Jenkins, ServiceNow) with workspace-scoped credentials and sync/push modes

All six capabilities are exposed as browse + CRUD views in V1, with the permission model restricted to a single **Platform Administrator** role (see REQ-PC-40).

> Confirmation: scope matches the "Platform Center 的四项基础能力" line in PRD §18.1 MVP list, expanded to all 6 capabilities per the product-line roadmap decision for this slice.

### 1.2 Explicitly Out of Scope in V1

Platform Center does **not** deliver:

- Low-code / no-code page composer UI (deferred per PRD §18.3 "复杂自定义低代码平台")
- Template marketplace or cross-tenant template federation
- Attribute-based access control (ABAC) editor UI — PRD §12.4 calls out ABAC as secondary to RBAC for V1
- Authentication / SSO / identity provider connection (assumed pre-existing via enterprise SSO)
- Skill registry editor UI (Skill objects are surfaced in AI Center, not re-edited here)
- Report-builder UI (Report Center handles deep reporting)
- Dynamic risk-threshold evaluation engine — V1 stores threshold configuration; runtime enforcement is handled by the per-domain Skill execution pipeline
- Automated remediation of configuration drift — V1 detects and displays drift; remediation is a human action via the domain pages
- Full-text search across all platform objects — V1 supports per-section filter/search only

### 1.3 Relationship to Other Slices

Platform Center owns the **authoritative models** for platform-base objects. Other slices consume these models read-only:

| Consumer slice | What it consumes from Platform Center |
|---|---|
| `shared-app-shell` | Navigation permissions (via Access Management role→nav-entry map) |
| `dashboard` | Metric template bindings; governance indicators |
| `requirement`, `project-management`, etc. | Page & field configuration for their detail views; flow templates |
| `ai-center` | Policy & Governance records per Skill; Audit records for Skill executions |
| `incident` | Notification configuration; approval policies for runbook actions |

This slice therefore **defines and owns** the platform-base domain (templates, configurations, audit, access, policies, integrations). Consumer slices must not re-define or duplicate these objects.

---

## 2. Product Context Requirements

### REQ-PC-01: Platform Center is a first-class navigation entry

Platform Center must appear as a top-level navigation item (`key = "platform"`) in the shared app shell, matching the entry already declared in the navigation seed (`codex-backend-phase-b.md`).

> Source: PRD §10, shared-app-shell nav contract

### REQ-PC-02: Platform Center is the home of the platform base layer

Platform Center must present the six PRD §12 capabilities (Template, Configuration, Audit, Access, Policy, Integration) as the central reusable foundation, not as scattered widgets in other pages.

> Source: PRD §11.13, §12.1–§12.6

### REQ-PC-03: Privileged admin surface with a single V1 role

Platform Center is restricted to users holding the **Platform Administrator** role in V1. Non-admin users navigating to `/platform` must see an explicit `403 Forbidden` state with guidance ("Contact your platform administrator") rather than a silent empty state.

> Source: PRD §16.1 role hierarchy, decision made for this slice

### REQ-PC-04: Audience and job-to-be-done

V1 Platform Center serves:

- **Platform Administrators** — curate templates, manage access, review audit, configure integrations, define policy
- **Auditors / Compliance Owners** — read-only browse of audit records and policy posture (via Platform Admin sub-role in V1; full Auditor role deferred)

It does not serve end-users, team leads, or contributors directly; those audiences interact with Platform Center outputs indirectly (via rendered pages, enforced policies, applied configurations).

> Source: PRD §6.1, §16.1

---

## 3. Template Management Requirements

### REQ-PC-10: Template catalog

Platform Center must display a browsable catalog of all templates across six kinds matching PRD §12.1:

- `page` — page template (layout + widget composition)
- `flow` — flow/workflow template (state machine + transitions)
- `project` — project-creation template (default settings for new projects)
- `policy` — policy template (reusable governance rule bundle)
- `metric` — metric template (reusable measurement definition)
- `ai` — AI template (Skill default prompts and parameters)

Each list entry must show at minimum: template key, human-readable name, kind, version, status (`draft` / `published` / `deprecated`), owner, last-modified timestamp, inheritance chain summary.

> Source: PRD §12.1

### REQ-PC-11: Template inheritance, versioning, overrides

Templates must model and display the four-layer inheritance chain defined in PRD §9.3:

**Platform Default → Application Default → SNOW Group Override → Project Override**

The detail view of any template must render the inheritance chain and highlight which layer currently supplies each field value (inherited / overridden / explicit).

> Source: PRD §9.3, §12.1

### REQ-PC-12: Template detail & edit

Selecting a template must open a detail view showing:

- Full metadata (key, name, kind, version, status, owner, description)
- Structured body (layout JSON, state machine JSON, rule set, etc. depending on kind)
- Version history (list of versions with author + timestamp; diff between adjacent versions V1 can be text-only)
- Usage list (which workspaces, applications, or projects currently reference this template)
- Create-new-version action (creates a draft version from the currently-published one)

Inline editing of template bodies in V1 is limited to metadata, status, and simple rule fields. Complex layout authoring is deferred.

> Source: PRD §12.1, product decision

### REQ-PC-13: Template status lifecycle

Each template carries a status that follows the transition rules:

- `draft` → `published`
- `published` → `deprecated`
- `deprecated` → `published` (reactivation)
- `draft` → (deleted; allowed only if no usages)

Transitions must be recorded in the audit log (REQ-PC-30).

> Source: PRD §12.1

---

## 4. Configuration Management Requirements

### REQ-PC-20: Configuration catalog by kind

Platform Center must display configurations grouped by the seven kinds in PRD §12.2:

- `page` — page-level config (which widgets show, in what order)
- `field` — field config (visibility, required, default, validation)
- `component` — component-arrangement config (form layout, column order)
- `flow-rule` — flow-rule config (transition guards, approvals per step)
- `view-rule` — view-rule config (filters, grouping, sort defaults)
- `notification` — notification rules (who is notified on which event)
- `ai-config` — AI configuration (per-skill autonomy defaults, allowed context, retry policy)

> Source: PRD §12.2

### REQ-PC-21: Team and project scope overrides

Configurations must support scope overrides at three levels beyond platform default: `application`, `snow_group`, and `project`. Each configuration entry records its `scope_type` and `scope_id`, enabling a resolve-time lookup that returns the narrowest applicable override.

> Source: PRD §9.3, §12.2

### REQ-PC-22: Configuration detail & edit

Selecting a configuration must expose:

- Key, kind, scope, value (structured JSON body), inherited-from reference (if override), status
- Edit action — JSON editor with schema validation (schemas bundled per kind in V1)
- "Revert to parent" action — deletes the override; the effective configuration reverts to the parent-scope value
- Usage preview — shows which pages / workflows resolve to this configuration at runtime

> Source: PRD §12.2, product decision

### REQ-PC-23: Configuration drift detection

Platform Center must surface configuration **drift indicators** when an override diverges materially from the platform default. V1 displays a drift badge in the catalog and a side-by-side compare in the detail view. Automatic drift remediation is out of scope (see §1.2).

> Source: PRD §17.4 "配置漂移情况" metric

---

## 5. Audit Management Requirements

### REQ-PC-30: Audit log viewer

Platform Center must provide a read-only audit log viewer showing records for all event categories defined in PRD §16.2:

- `config_change` — configuration / template modified
- `permission_change` — role assigned / revoked / permission edited
- `ai_suggestion` — AI produced a suggestion that was surfaced to a user
- `ai_execution` — AI executed an action (with autonomy level and policy decision)
- `approval_decision` — human approved or rejected a queued action
- `skill_execution` — skill run (link-through to AI Center run detail)
- `deployment_event` — release / rollback
- `incident_event` — incident created / state-changed / resolved
- `policy_hit` — policy rule matched and influenced an outcome

Each record displays: timestamp, actor (user or AI/Skill), action kind, affected object, workspace/project scope, outcome, evidence link.

> Source: PRD §12.3, §16.2

### REQ-PC-31: Audit filters

Audit log must support filtering by:

- Event category (multi-select)
- Actor (user or Skill key)
- Object type / object id
- Time range (24h / 7d / 30d / custom)
- Workspace / application / project scope
- Outcome (`success`, `failure`, `rejected`, `rolled_back`)

### REQ-PC-32: Audit retention policy (V1)

V1 retention: audit records are retained for at least **365 days**. Longer retention and export tooling are deferred to a Report Center / compliance slice.

> Source: PRD §16.3

### REQ-PC-33: Audit is append-only

Audit records must be immutable. V1 enforces this at the application layer (no update/delete controller paths); stronger database-level enforcement (triggers, WORM storage) is deferred.

> Source: PRD §16.3, security decision

---

## 6. Access Management Requirements

### REQ-PC-40: RBAC-first model

Access Management must implement role-based access control as the primary mechanism, per PRD §12.4. V1 supports the following roles:

- `PLATFORM_ADMIN` — full Platform Center access; can manage any workspace
- `WORKSPACE_ADMIN` — workspace-level administration (included for future expansion; not enforced in V1 Platform Center since the whole page is `PLATFORM_ADMIN`-gated)
- `WORKSPACE_MEMBER` — standard contributor
- `WORKSPACE_VIEWER` — read-only
- `AUDITOR` — read-only access to audit records (included in the data model; UI wiring deferred)

V1 enforcement scope for Platform Center itself: the entire page is gated behind `PLATFORM_ADMIN`. Other roles are stored and manageable from Platform Center but enforced by consumer slices.

> Source: PRD §12.4, §16.1

### REQ-PC-41: Role assignments catalog

Platform Center must show a browsable list of role assignments with columns: user identifier, role, scope kind (`platform` / `application` / `workspace` / `project`), scope id, granted-by, granted-at.

### REQ-PC-42: Role assignment CRUD

V1 supports: create a role assignment (pick user + role + scope), revoke an assignment, view assignment history. Bulk import and directory sync are deferred.

> Source: PRD §12.4

### REQ-PC-43: Scope model

Every permission / assignment is bound to a scope represented as a pair `(scope_type, scope_id)` where `scope_type ∈ { 'platform', 'application', 'workspace', 'project' }`. Scope `platform` uses a sentinel id (e.g., `"*"`).

> Source: PRD §12.4

### REQ-PC-44: ABAC is a V2 extension

V1 does not provide an ABAC rule editor. The data model must reserve placeholders (e.g., `attributes_json` column on role-assignment) so V2 can add attribute predicates without schema churn.

> Source: PRD §12.4 "以 RBAC 为主、ABAC 为辅"

---

## 7. Policy & Governance Requirements

### REQ-PC-50: Policy catalog

Platform Center must present a catalog of governance policies. Each policy has: key, human-readable name, category (`action`, `approval`, `autonomy`, `risk-threshold`, `exception`), scope (platform / application / workspace / project), status (`active` / `inactive` / `draft`), bound-to (Skill key or action kind), version.

> Source: PRD §12.5

### REQ-PC-51: Policy kinds and fields

The five kinds must be supported in V1 with the following fields:

- `action` — what actions the policy governs (object kind, operation), allowed-by (roles), blocked-by (roles), audit-only flag
- `approval` — which actions require approval, approver role(s), parallel vs. sequential, timeout behavior
- `autonomy` — per-Skill default autonomy level (`L0` / `L1` / `L2` / `L3`), override-allowed flag, escalation role
- `risk-threshold` — metric key, comparison operator, threshold value, action-on-breach
- `exception` — an explicit carve-out from an active policy, including reason, requester, approver, expiry

> Source: PRD §7.5, §12.5

### REQ-PC-52: Policy detail & edit

V1 supports: create a policy (all fields), edit an existing policy (creates a new version; previous version becomes inactive), activate/deactivate, add exception entries to an active policy.

### REQ-PC-53: Policy evaluation is a runtime concern

V1 Platform Center **does not evaluate policies at runtime**. It stores and serves policy configuration; the runtime evaluation (in Skill execution pipeline and in each domain service) is implemented in the consumer slices and in `ai-center`. Platform Center is responsible only for the source-of-truth records and for logging `policy_hit` audit events when consumer slices report them.

> Source: PRD §12.5, product decision

---

## 8. Integration Framework Requirements

### REQ-PC-60: Adapter registry

Platform Center must present a catalog of integration adapters covering at least the four V1 adapter kinds from PRD §12.6:

- `jira` — Requirement / priority / status synchronization
- `confluence` — Requirement source evidence and business-context synchronization
- `gitlab` — Code changes, MRs, branches, review status
- `jenkins` — Builds, pipelines, quality gates
- `servicenow` — Incidents, changes, on-call, approval context

Plus one extension slot: `custom-webhook` (generic webhook-based inbound adapter, deferred behavior but registry entry supported).

Integration connections must expose team ownership scope using Workspace, Application, and SNOW Group fields so multiple Jira, Confluence, Jenkins, and related systems can be configured without ambiguity across product teams.

> Source: PRD §12.6

### REQ-PC-61: Adapter instance = connection

An adapter instance (a "connection") is the unit of configuration: it binds an adapter kind to a workspace, carries credentials (stored referenced, never displayed), sync mode (`pull` / `push` / `both`), sync schedule, status (`enabled` / `disabled` / `error`), and last-sync timestamp.

> Source: PRD §12.6

### REQ-PC-62: Workspace-scoped credentials

Per PRD §12.6, every adapter instance's credentials must be scoped to a specific workspace; platform administrators cannot read the credential values across workspace boundaries. V1 stores credential references (e.g., pointer to secret store); actual secrets never surface in the UI or API responses.

> Source: PRD §12.6 — "集成凭据必须按 Workspace 隔离管理"

### REQ-PC-63: Connection test

Each adapter instance must support a "Test connection" action that performs a lightweight synthetic call (e.g., fetch current user, list projects) and returns success/failure. V1 logs the test outcome as an audit record.

> Source: PRD §12.6, product decision

### REQ-PC-64: Sync mode configuration

V1 supports per-instance configuration of:

- `pull` — scheduled pull from external system (cron-like expression)
- `push` — webhook endpoint URL for external-to-platform push
- `both` — both modes active on the same adapter instance

V1 does not implement the actual sync workers; it stores the configuration. Consumer slices (requirement, code-build, incident) implement and run the workers in later slices.

> Source: PRD §12.6

---

## 9. Visual and Experience Requirements

### REQ-PC-70: Left-nav-with-sections layout

Platform Center uses a **two-pane layout**:

- Left rail: sub-section list (Templates / Configurations / Audit / Access / Policies / Integrations)
- Main area: the selected section's catalog + detail; detail opens in-page as a two-column layout (list narrowed + detail to the right), or full-screen for edit flows

This differs from the single-scroll card layouts used by Dashboard or AI Center because the admin use case is "find and edit a specific record" rather than "scan a story."

> Source: PRD §15.3 (modular) adapted to admin UX; visual-design-system.md token system

### REQ-PC-71: Shared design tokens

All sub-sections must use the tokens, typography, and status colors from the shared design system (`visual-design-system.md`). Status badges (`active` / `draft` / `deprecated` / `error`) reuse the shared OK / Warning / Critical palette.

> Source: `visual-design-system.md` §2, `docs/standard-sdd/projects/control-tower/05-design/design.md`

### REQ-PC-72: States

Every catalog view must support:

- Normal (records present)
- Loading (skeleton rows)
- Empty (zero records — instructional state with the top-right "Create" action highlighted where applicable)
- Error (API failure — error state at catalog level, with retry)
- Forbidden (user is not `PLATFORM_ADMIN` — 403 state with explanation, per REQ-PC-03)

### REQ-PC-73: Danger-path confirmation

Destructive actions (deactivate policy, revoke role, delete template draft, disable integration) require a confirm dialog that explicitly names the target record and warns about downstream impact.

> Source: PRD §16.3 governance principles

---

## 10. Non-Functional Requirements

### REQ-PC-80: Frontend and backend via Codex

Per decision for this slice, **both the frontend and the backend** for Platform Center are generated via **Codex** (not Gemini). The implementation prompt artifacts were used during initial app generation and are not part of this profile-first SDD source repo.

This deviates from the Dashboard slice (which used Gemini for FE, Codex for BE) and is explicitly noted here.

> Source: product decision, CLAUDE.md Lesson #2 "use what the user specifies"

### REQ-PC-81: Workspace and platform scoping

Platform Center objects fall into two scope classes:

- **Platform-scoped** — templates, platform-default configurations, platform-default policies, adapter kinds (registry) — visible to all workspace admins
- **Workspace-scoped** — adapter instances (connections), role assignments for workspace scope, configuration overrides, policy overrides, audit records — scoped per PRD §9.2

The UI must show the active scope chip for any selected object and warn when an action crosses scope boundaries.

> Source: PRD §9.2, §16.3

### REQ-PC-82: Package-by-feature backend

All Platform Center entities must live under `backend/src/main/java/com/sdlctower/platform/` per the existing package-by-feature convention (CLAUDE.md Lesson #3), with sub-packages: `platform/template/`, `platform/configuration/`, `platform/audit/`, `platform/access/`, `platform/policy/`, `platform/integration/`.

> Source: CLAUDE.md Lesson #3; existing backend structure

### REQ-PC-83: Flyway migrations for all schema

All database schema for Platform Center must be introduced via Flyway migrations starting at the next available version number (V40+). No `ddl-auto: update`. Seed data uses `V{N+1}__seed_*.sql` migrations following repo convention.

> Source: CLAUDE.md Lesson #4; existing migration numbering

### REQ-PC-84: Builds must succeed

`npm run dev` and `npm run build` must both succeed from `frontend/`. `./mvnw test` must pass from `backend/`. New tests must cover at least one controller per sub-section (six controller tests minimum) plus one repository integration test per entity.

### REQ-PC-85: Performance

- Catalog list endpoints return within p95 ≤ 500ms for ≤ 500 records per list
- Audit log endpoint must paginate; page size default 50, max 200
- Policy resolve endpoint (if exposed) must return in p95 ≤ 100ms — deferred to runtime concern
- Any endpoint that joins across more than three tables must use a single aggregated SQL query (no N+1)

### REQ-PC-86: Observability

All mutating Platform Center endpoints must emit an audit record **as part of the same transaction** as the mutation. If audit insertion fails, the entire mutation must roll back. This is a correctness requirement for a governance surface.

> Source: PRD §16.2, §16.3

---

## 11. Out of Scope

- Low-code / no-code template editor with WYSIWYG layout
- Cross-tenant template marketplace
- Attribute-based access control (ABAC) editor UI
- Identity provider / SSO connection
- Skill registry editor (lives in AI Center)
- Real-time notifications panel (Platform Center itself)
- CSV / PDF export of audit logs (belongs to Report Center)
- Dynamic policy runtime evaluator (policies are stored, not evaluated here)
- Automated configuration-drift remediation
- Time-bounded ("just-in-time") elevated-role grants
- Full-text search across all platform objects (per-section filters only)

---

## 12. Traceability

| Requirement | PRD Section |
|-------------|-------------|
| REQ-PC-01 | §10, shared-app-shell nav |
| REQ-PC-02 | §11.13, §12.1–§12.6 |
| REQ-PC-03 | §16.1; slice decision |
| REQ-PC-04 | §6.1, §16.1 |
| REQ-PC-10 | §12.1 |
| REQ-PC-11 | §9.3, §12.1 |
| REQ-PC-12 | §12.1 |
| REQ-PC-13 | §12.1 |
| REQ-PC-20 | §12.2 |
| REQ-PC-21 | §9.3, §12.2 |
| REQ-PC-22 | §12.2 |
| REQ-PC-23 | §17.4 |
| REQ-PC-30 | §12.3, §16.2 |
| REQ-PC-31 | — (UX-derived) |
| REQ-PC-32 | §16.3 |
| REQ-PC-33 | §16.3 |
| REQ-PC-40 | §12.4, §16.1 |
| REQ-PC-41 | §12.4 |
| REQ-PC-42 | §12.4 |
| REQ-PC-43 | §12.4 |
| REQ-PC-44 | §12.4 |
| REQ-PC-50 | §12.5 |
| REQ-PC-51 | §7.5, §12.5 |
| REQ-PC-52 | §12.5 |
| REQ-PC-53 | §12.5 |
| REQ-PC-60 | §12.6 |
| REQ-PC-61 | §12.6 |
| REQ-PC-62 | §12.6 |
| REQ-PC-63 | §12.6 |
| REQ-PC-64 | §12.6 |
| REQ-PC-70 | §15.3 |
| REQ-PC-71 | `visual-design-system.md` §2, `docs/standard-sdd/projects/control-tower/05-design/design.md` |
| REQ-PC-72 | — (UX-derived) |
| REQ-PC-73 | §16.3 |
| REQ-PC-80 | CLAUDE.md Lesson #2; slice decision |
| REQ-PC-81 | §9.2, §16.3 |
| REQ-PC-82 | CLAUDE.md Lesson #3 |
| REQ-PC-83 | CLAUDE.md Lesson #4 |
| REQ-PC-84 | — |
| REQ-PC-85 | §17 (NFR category) |
| REQ-PC-86 | §16.2, §16.3 |
