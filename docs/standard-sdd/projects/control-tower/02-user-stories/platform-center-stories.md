# Platform Center User Stories

## Purpose

This document converts the requirements in [platform-center-requirements.md](../01-requirements/platform-center-requirements.md) into Agile user stories with Given/When/Then acceptance criteria. Each story traces back to one or more REQ-PC-XX IDs.

## Source

- [platform-center-requirements.md](../01-requirements/platform-center-requirements.md)
- [agentic_sdlc_control_tower_prd_v0.9.md](../01-requirements/agentic_sdlc_control_tower_prd_v0.9.md)

---

## Epics

| Epic | Scope | Stories |
|------|-------|---------|
| E1. Access & Entry | Platform Center gating, navigation, forbidden state | S-PC-01 through S-PC-03 |
| E2. Template Management | Browse, view, version, lifecycle templates | S-PC-10 through S-PC-14 |
| E3. Configuration Management | Browse, edit, override, drift | S-PC-20 through S-PC-23 |
| E4. Audit Management | Browse, filter, drill-down audit records | S-PC-30 through S-PC-32 |
| E5. Access Management | Browse, assign, revoke roles | S-PC-40 through S-PC-43 |
| E6. Policy & Governance | Browse, create, edit, exception-handle policies | S-PC-50 through S-PC-53 |
| E7. Integration Framework | Browse adapters, configure connections, test, set sync mode | S-PC-60 through S-PC-63 |
| E8. Cross-cutting | Error / empty / destructive-action UX | S-PC-70 through S-PC-72 |

---

## Epic 1 — Access & Entry

### S-PC-01: Navigate to Platform Center

> **As a** Platform Administrator  
> **I want** to click the Platform Center nav entry in the shared app shell  
> **So that** I can reach the platform-administration surface.

**Traces to:** REQ-PC-01, REQ-PC-02, REQ-PC-03

**Acceptance Criteria**

- **Given** the user is authenticated as `PLATFORM_ADMIN`  
  **When** they click the "Platform Center" navigation entry  
  **Then** the app routes to `/platform` and renders the Platform Center shell with the six sub-section rail visible.

- **Given** the user is on `/platform`  
  **When** the page loads  
  **Then** the default sub-section is "Templates" and its catalog fetch is initiated.

### S-PC-02: Non-admin is blocked with guidance

> **As a** non-admin user who navigated to `/platform`  
> **I want** a clear forbidden screen rather than a blank page  
> **So that** I know I am in the right place but lack authority.

**Traces to:** REQ-PC-03, REQ-PC-72

**Acceptance Criteria**

- **Given** the user does not hold `PLATFORM_ADMIN`  
  **When** they navigate to `/platform`  
  **Then** a 403 state is rendered with the message "Platform Center is restricted to platform administrators. Contact your platform administrator for access."

- **Given** the 403 state is showing  
  **When** the user clicks the "Back to Dashboard" action  
  **Then** they are routed to `/`.

### S-PC-03: Sub-section navigation within Platform Center

> **As a** Platform Administrator  
> **I want** to switch between the six sub-sections from the left rail  
> **So that** I can reach a specific capability without leaving Platform Center.

**Traces to:** REQ-PC-70

**Acceptance Criteria**

- **Given** the user is on `/platform/templates`  
  **When** they click "Audit" in the left rail  
  **Then** the route becomes `/platform/audit` and the audit catalog loads.

- **Given** the user is viewing a detail record in any sub-section  
  **When** they switch sub-sections from the rail  
  **Then** the previous detail panel closes and the newly selected sub-section loads from its catalog default.

---

## Epic 2 — Template Management

### S-PC-10: Browse templates

> **As a** Platform Administrator  
> **I want** a catalog of all platform templates across six kinds  
> **So that** I can see what reusable assets exist.

**Traces to:** REQ-PC-10

**Acceptance Criteria**

- **Given** the Templates sub-section is active  
  **When** the catalog loads  
  **Then** each row shows: template key, name, kind (page/flow/project/policy/metric/ai), version, status, owner, last-modified.

- **Given** the catalog is loaded  
  **When** the admin filters by kind = "flow"  
  **Then** only flow templates are displayed and the selected kind filter chip is highlighted.

### S-PC-11: View template inheritance chain

> **As a** Platform Administrator  
> **I want** to see the four-layer inheritance chain on a template detail  
> **So that** I can tell which layer actually supplies each value.

**Traces to:** REQ-PC-11

**Acceptance Criteria**

- **Given** a template with overrides at Application and Project layers  
  **When** the admin opens detail view  
  **Then** each field row shows the effective value plus a chip indicating the winning layer (Platform / Application / SNOW Group / Project) and a secondary column showing the platform-default value for reference.

- **Given** no override exists at any layer for a given field  
  **When** detail renders  
  **Then** the layer chip shows "Platform" and no override markers appear.

### S-PC-12: View template versions

> **As a** Platform Administrator  
> **I want** to see the version history of a template  
> **So that** I can understand what has changed over time.

**Traces to:** REQ-PC-12

**Acceptance Criteria**

- **Given** a template has three published versions  
  **When** the admin opens the "Versions" tab of detail view  
  **Then** three rows appear in reverse-chronological order, each showing version id, author, timestamp, and a "View body" action.

### S-PC-13: Publish / deprecate / reactivate template

> **As a** Platform Administrator  
> **I want** to transition a template between `draft`, `published`, and `deprecated`  
> **So that** consumers only see templates that are safe to use.

**Traces to:** REQ-PC-13, REQ-PC-73

**Acceptance Criteria**

- **Given** a template is in `draft` status  
  **When** the admin clicks "Publish" and confirms in the dialog  
  **Then** the template status becomes `published`, an audit record of kind `config_change` is created, and the status chip on the catalog row updates to "Published".

- **Given** a template is `published`  
  **When** the admin clicks "Deprecate" and confirms  
  **Then** status becomes `deprecated`, audit record is created, and the status chip updates. Templates in `deprecated` status remain resolvable by existing consumers but new usages must display a deprecation warning (UX concern for consumer slices).

- **Given** a template is `deprecated`  
  **When** the admin clicks "Reactivate" and confirms  
  **Then** status returns to `published` and the transition is audited.

### S-PC-14: Cannot delete a referenced template

> **As a** Platform Administrator  
> **I want** the system to block deletion of a draft template that has usages  
> **So that** I cannot break consumers accidentally.

**Traces to:** REQ-PC-12

**Acceptance Criteria**

- **Given** a draft template that has at least one recorded usage  
  **When** the admin clicks "Delete"  
  **Then** the destructive confirm dialog displays the list of referencing scopes and the primary action is disabled with the tooltip "Cannot delete — template is referenced."

- **Given** a draft template with no usages  
  **When** the admin confirms deletion  
  **Then** the record is removed, an audit `config_change` record is created with action `template.delete`.

---

## Epic 3 — Configuration Management

### S-PC-20: Browse configurations by kind

> **As a** Platform Administrator  
> **I want** to browse configurations grouped by kind  
> **So that** I can find page/field/flow/ai configurations quickly.

**Traces to:** REQ-PC-20

**Acceptance Criteria**

- **Given** the Configuration sub-section is active  
  **When** the catalog loads  
  **Then** a kind selector is present with the seven kinds (page/field/component/flow-rule/view-rule/notification/ai-config) and the default shows all.

### S-PC-21: Create a scope-specific override

> **As a** Platform Administrator  
> **I want** to create a configuration override at the application, SNOW group, or project scope  
> **So that** a team can deviate from the platform default without forking the template.

**Traces to:** REQ-PC-21, REQ-PC-22

**Acceptance Criteria**

- **Given** a platform-default configuration `X`  
  **When** the admin clicks "Add override", picks scope `workspace = WS-42`, and saves a JSON body  
  **Then** a new configuration row appears linked to `X` as parent, scope shown as `workspace:WS-42`, status `active`.

- **Given** the JSON body fails schema validation  
  **When** the admin clicks Save  
  **Then** the form displays inline validation errors and does not submit.

### S-PC-22: Revert an override to parent

> **As a** Platform Administrator  
> **I want** to revert a configuration override back to the parent  
> **So that** the workspace resumes inheriting the default.

**Traces to:** REQ-PC-22, REQ-PC-73

**Acceptance Criteria**

- **Given** an override configuration is being viewed  
  **When** the admin clicks "Revert to parent" and confirms in the destructive dialog  
  **Then** the override record is deleted, the catalog row disappears, and an audit `config_change` entry is recorded.

### S-PC-23: See configuration drift indicator

> **As a** Platform Administrator  
> **I want** to see which overrides have drifted from the platform default  
> **So that** I can review whether the divergence is still intentional.

**Traces to:** REQ-PC-23

**Acceptance Criteria**

- **Given** an override whose body differs materially from the platform default  
  **When** the catalog renders  
  **Then** a "Drift" badge appears on the row and the detail view shows a side-by-side compare highlighting changed fields.

---

## Epic 4 — Audit Management

### S-PC-30: Browse recent audit records

> **As a** Platform Administrator  
> **I want** a reverse-chronological audit log  
> **So that** I can see what has happened recently on the platform.

**Traces to:** REQ-PC-30

**Acceptance Criteria**

- **Given** the Audit sub-section is active  
  **When** the log loads  
  **Then** rows show timestamp, actor, action kind, object, scope, outcome, and an "Evidence" link when available, ordered newest first, with pagination default 50.

### S-PC-31: Filter audit records

> **As a** Platform Administrator  
> **I want** to filter audit records by category, actor, object, time range, and outcome  
> **So that** I can investigate a specific event.

**Traces to:** REQ-PC-31

**Acceptance Criteria**

- **Given** the audit log is open  
  **When** the admin sets filter `category = permission_change AND outcome = success AND time range = last 24h`  
  **Then** only matching rows are returned and the URL query string reflects the active filters so the view is shareable.

### S-PC-32: Drill into an audit record

> **As a** Platform Administrator  
> **I want** to drill into a single audit record  
> **So that** I can see the full payload, actor context, and linked evidence.

**Traces to:** REQ-PC-30

**Acceptance Criteria**

- **Given** an audit record row  
  **When** the admin clicks it  
  **Then** a side panel opens showing full payload, before/after values if applicable, actor details, and link-outs for evidence (AI Center for `skill_execution`, Incident detail for `incident_event`, etc.).

- **Given** a record of kind `skill_execution`  
  **When** the admin clicks "View run in AI Center"  
  **Then** the app routes to `/ai-center/runs/{executionId}`.

---

## Epic 5 — Access Management

### S-PC-40: Browse role assignments

> **As a** Platform Administrator  
> **I want** a list of role assignments across all scopes  
> **So that** I can audit who has what.

**Traces to:** REQ-PC-41

**Acceptance Criteria**

- **Given** the Access sub-section is active  
  **When** the catalog loads  
  **Then** each row shows user, role, scope kind, scope id, granted-by, granted-at; default sort is by granted-at descending.

### S-PC-41: Assign a role

> **As a** Platform Administrator  
> **I want** to assign a user to a role at a specific scope  
> **So that** they gain the corresponding permissions.

**Traces to:** REQ-PC-42, REQ-PC-43

**Acceptance Criteria**

- **Given** the admin clicks "Assign role"  
  **When** they pick user, role, and scope `(scope_type = workspace, scope_id = WS-42)` and confirm  
  **Then** a new assignment is created, the catalog refreshes, an audit `permission_change` record with action `role.grant` is written.

- **Given** the admin selects `scope_type = platform`  
  **When** they open the scope id dropdown  
  **Then** only the sentinel `"*"` value is available (since platform scope has a single id).

### S-PC-42: Revoke a role

> **As a** Platform Administrator  
> **I want** to revoke an existing role assignment  
> **So that** I can remove access that is no longer needed.

**Traces to:** REQ-PC-42, REQ-PC-73

**Acceptance Criteria**

- **Given** an existing assignment row  
  **When** the admin clicks "Revoke" and confirms the destructive dialog  
  **Then** the assignment is deleted, the row disappears, and an audit `permission_change` record with action `role.revoke` is written.

### S-PC-43: Cannot revoke the last platform admin

> **As a** Platform Administrator  
> **I want** the system to prevent revoking the last remaining `PLATFORM_ADMIN` assignment  
> **So that** the platform can never be locked out.

**Traces to:** REQ-PC-42, REQ-PC-86

**Acceptance Criteria**

- **Given** only one active `PLATFORM_ADMIN` assignment exists  
  **When** the admin clicks Revoke on it  
  **Then** the confirm dialog disables the primary action with the tooltip "Cannot revoke the last platform administrator" and no mutation is submitted.

---

## Epic 6 — Policy & Governance

### S-PC-50: Browse policies

> **As a** Platform Administrator  
> **I want** a catalog of governance policies  
> **So that** I can audit posture and find policies to edit.

**Traces to:** REQ-PC-50

**Acceptance Criteria**

- **Given** the Policy sub-section is active  
  **When** the catalog loads  
  **Then** rows show policy key, name, category, scope, status, bound-to, version; the kind filter offers the five categories (action/approval/autonomy/risk-threshold/exception).

### S-PC-51: Create a new policy

> **As a** Platform Administrator  
> **I want** to create a policy of any of the five kinds  
> **So that** I can encode new governance rules.

**Traces to:** REQ-PC-51, REQ-PC-52

**Acceptance Criteria**

- **Given** the admin clicks "New policy"  
  **When** they pick kind = `autonomy`, bind to Skill key `incident-diagnosis`, set default level = `L1-Assist`, save  
  **Then** a policy record with version `v1`, status `active` is created; an audit `config_change` record with action `policy.create` is written.

- **Given** required fields are missing  
  **When** the admin clicks Save  
  **Then** inline validation errors appear and submission is blocked.

### S-PC-52: Edit policy (new version)

> **As a** Platform Administrator  
> **I want** editing a policy to produce a new version instead of overwriting  
> **So that** history is preserved.

**Traces to:** REQ-PC-52, REQ-PC-33

**Acceptance Criteria**

- **Given** policy `P` at version `v2`, status `active`  
  **When** the admin edits a field and saves  
  **Then** a new record at version `v3` is created with status `active`; version `v2` is transitioned to status `inactive`; an audit `config_change` record with action `policy.update` is written with before/after summary.

### S-PC-53: Add an exception

> **As a** Platform Administrator  
> **I want** to add a time-boxed exception to an active policy  
> **So that** I can unblock a specific case without disabling the whole policy.

**Traces to:** REQ-PC-51, REQ-PC-52

**Acceptance Criteria**

- **Given** an active policy `P`  
  **When** the admin opens detail, clicks "Add exception", enters reason, requester, approver, expiry, and saves  
  **Then** an exception row appears under the policy, expires at the given date, and an audit `config_change` record with action `policy.exception.add` is written.

---

## Epic 7 — Integration Framework

### S-PC-60: Browse adapter registry and instances

> **As a** Platform Administrator  
> **I want** to see available adapter kinds and existing connection instances  
> **So that** I understand what's integrated.

**Traces to:** REQ-PC-60, REQ-PC-61

**Acceptance Criteria**

- **Given** the Integration sub-section is active  
  **When** the catalog loads  
  **Then** the header shows the adapter kinds (Jira / Confluence / GitLab / Jenkins / ServiceNow / custom-webhook) and the body lists all configured connection instances with columns: kind, workspace scope, sync mode, status, last-sync.

### S-PC-61: Configure a connection

> **As a** Platform Administrator  
> **I want** to create a Jira connection for a workspace  
> **So that** the requirement slice can sync with Jira.

**Traces to:** REQ-PC-61, REQ-PC-62, REQ-PC-64

**Acceptance Criteria**

- **Given** the admin clicks "New connection"  
  **When** they pick kind = Jira, scope = workspace `WS-42`, paste a credential id (referenced from secret store), pick sync mode = `both`, and save  
  **Then** a connection record appears with status `enabled` and an audit `config_change` record with action `integration.create` is written.

- **Given** the admin is viewing an existing connection's detail  
  **When** they attempt to read the credential value  
  **Then** the UI shows a masked placeholder (e.g., `••••••••`) and no plain-text value is ever returned from the API.

### S-PC-62: Test a connection

> **As a** Platform Administrator  
> **I want** to click a "Test connection" button on a configured connection  
> **So that** I can confirm credentials and endpoint work before enabling sync.

**Traces to:** REQ-PC-63

**Acceptance Criteria**

- **Given** a connection is configured  
  **When** the admin clicks "Test connection"  
  **Then** the backend performs a synthetic call and returns success or failure within 10 seconds; the result is displayed as a toast and recorded as an audit event of kind `integration.test`.

### S-PC-63: Disable a connection

> **As a** Platform Administrator  
> **I want** to disable an existing connection  
> **So that** I can pause sync without losing the configuration.

**Traces to:** REQ-PC-61, REQ-PC-73

**Acceptance Criteria**

- **Given** a connection with status `enabled`  
  **When** the admin clicks "Disable" and confirms  
  **Then** status changes to `disabled`, the audit record is written, and future sync workers will skip this connection.

---

## Epic 8 — Cross-Cutting UX

### S-PC-70: Catalog empty state

> **As a** Platform Administrator  
> **I want** a helpful empty state in each catalog  
> **So that** I know what to do on an empty platform.

**Traces to:** REQ-PC-72

**Acceptance Criteria**

- **Given** a catalog endpoint returns zero records  
  **When** the catalog view renders  
  **Then** an empty state is shown with: an illustration, a one-sentence explanation ("No templates have been created yet"), and a primary action button (e.g., "Create template") — except for Audit, where the empty state reads "No audit events match your filters" with no primary action.

### S-PC-71: Catalog error state

> **As a** Platform Administrator  
> **I want** an actionable error state when a catalog fetch fails  
> **So that** I can retry without reloading the page.

**Traces to:** REQ-PC-72

**Acceptance Criteria**

- **Given** a catalog endpoint returns HTTP 500 or times out  
  **When** the view processes the error  
  **Then** the catalog area renders an inline error with a Retry button; the rest of the Platform Center shell (left rail, workspace chip) remains interactive.

### S-PC-72: Destructive action confirmation

> **As a** Platform Administrator  
> **I want** every destructive action to force an explicit confirmation  
> **So that** I cannot break production by misclick.

**Traces to:** REQ-PC-73

**Acceptance Criteria**

- **Given** any of (deactivate policy, revoke role, delete template, disable integration, revert override)  
  **When** the admin triggers the action  
  **Then** a modal appears naming the target record (e.g., "Revoke role `WORKSPACE_ADMIN` from user `alice@corp` in scope `workspace:WS-42`?") with a destructive (red) primary button and a Cancel button; the primary button is the only way to proceed.

---

## Traceability Matrix

| Story | REQ-PC-XX |
|-------|-----------|
| S-PC-01 | REQ-PC-01, REQ-PC-02, REQ-PC-03 |
| S-PC-02 | REQ-PC-03, REQ-PC-72 |
| S-PC-03 | REQ-PC-70 |
| S-PC-10 | REQ-PC-10 |
| S-PC-11 | REQ-PC-11 |
| S-PC-12 | REQ-PC-12 |
| S-PC-13 | REQ-PC-13, REQ-PC-73 |
| S-PC-14 | REQ-PC-12 |
| S-PC-20 | REQ-PC-20 |
| S-PC-21 | REQ-PC-21, REQ-PC-22 |
| S-PC-22 | REQ-PC-22, REQ-PC-73 |
| S-PC-23 | REQ-PC-23 |
| S-PC-30 | REQ-PC-30 |
| S-PC-31 | REQ-PC-31 |
| S-PC-32 | REQ-PC-30 |
| S-PC-40 | REQ-PC-41 |
| S-PC-41 | REQ-PC-42, REQ-PC-43 |
| S-PC-42 | REQ-PC-42, REQ-PC-73 |
| S-PC-43 | REQ-PC-42, REQ-PC-86 |
| S-PC-50 | REQ-PC-50 |
| S-PC-51 | REQ-PC-51, REQ-PC-52 |
| S-PC-52 | REQ-PC-52, REQ-PC-33 |
| S-PC-53 | REQ-PC-51, REQ-PC-52 |
| S-PC-60 | REQ-PC-60, REQ-PC-61 |
| S-PC-61 | REQ-PC-61, REQ-PC-62, REQ-PC-64 |
| S-PC-62 | REQ-PC-63 |
| S-PC-63 | REQ-PC-61, REQ-PC-73 |
| S-PC-70 | REQ-PC-72 |
| S-PC-71 | REQ-PC-72 |
| S-PC-72 | REQ-PC-73 |
