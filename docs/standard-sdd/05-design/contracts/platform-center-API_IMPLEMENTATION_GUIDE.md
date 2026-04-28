# Platform Center — API Implementation Guide

## Purpose

This document is the complete REST contract for the Platform Center slice: every endpoint, request and response JSON examples, error codes, backend implementation notes, frontend integration notes, and testing contracts.

It is the primary source of truth for Codex when implementing both the frontend and the backend for this slice. All endpoints are mounted under `/api/v1/platform/*`.

## Source

- [platform-center-spec.md](../../03-spec/platform-center-spec.md)
- [platform-center-data-model.md](../../04-architecture/platform-center-data-model.md)
- [platform-center-design.md](../platform-center-design.md)

## Traceability

| Spec FR | Endpoint |
|---------|----------|
| FR-02, S-PC-02 | `GET /access/me` |
| FR-10 | `GET /templates` |
| FR-11 | `GET /templates/{id}` |
| FR-12 | `GET /templates/{id}/versions` |
| FR-13 | `POST /templates/{id}/publish`, `/deprecate`, `/reactivate`; `DELETE /templates/{id}` |
| FR-14 | `POST /templates/{id}/versions` |
| FR-20 | `GET /configurations` |
| FR-21 | `POST /configurations` |
| FR-22 | `DELETE /configurations/{id}` |
| FR-23 | `GET /configurations/{id}/drift` |
| FR-30 | `GET /audit` |
| FR-31 | `GET /audit/{id}` |
| FR-40 | `GET /access/assignments` |
| FR-41 | `POST /access/assignments` |
| FR-42, FR-43 | `DELETE /access/assignments/{id}` |
| FR-50 | `GET /policies` |
| FR-51 | `POST /policies` |
| FR-52 | `PUT /policies/{id}` |
| FR-53 | `POST /policies/{id}/exceptions`, `DELETE .../{exceptionId}` |
| FR-54 | `POST /policies/{id}/activate`, `/deactivate` |
| FR-60 | `GET /integrations/adapters` |
| FR-61 | `GET /integrations/connections` |
| FR-62 | `POST /integrations/connections` |
| FR-63 | `POST /integrations/connections/{id}/test` |
| FR-64 | `POST /integrations/connections/{id}/enable`, `/disable`; `DELETE .../{id}` |

---

## API Overview

All endpoints are prefixed `/api/v1/platform`. Paths below are written relative to that prefix for brevity.

| # | Method | Path | Purpose |
|---|--------|------|---------|
| 1 | GET | `/access/me` | Current user + roles (used by route guard) |
| 2 | GET | `/templates` | Template catalog (filtered, paginated) |
| 3 | GET | `/templates/{id}` | Template detail + inheritance |
| 4 | GET | `/templates/{id}/versions` | Version list |
| 5 | POST | `/templates/{id}/versions` | Create new version from draft body |
| 6 | POST | `/templates/{id}/publish` | Transition draft → published |
| 7 | POST | `/templates/{id}/deprecate` | Transition published → deprecated |
| 8 | POST | `/templates/{id}/reactivate` | Transition deprecated → published |
| 9 | DELETE | `/templates/{id}` | Delete draft with 0 usages |
| 10 | GET | `/configurations` | Configuration catalog |
| 11 | POST | `/configurations` | Create override |
| 12 | DELETE | `/configurations/{id}` | Revert override to parent |
| 13 | GET | `/configurations/{id}/drift` | Drift report |
| 14 | GET | `/audit` | Audit log (paginated, filtered) |
| 15 | GET | `/audit/{id}` | Audit record detail |
| 16 | GET | `/access/assignments` | Role assignment catalog |
| 17 | POST | `/access/assignments` | Assign role |
| 18 | DELETE | `/access/assignments/{id}` | Revoke role (last-admin guarded) |
| 19 | GET | `/policies` | Policy catalog |
| 20 | POST | `/policies` | Create policy (v1) |
| 21 | PUT | `/policies/{id}` | Edit policy (new version) |
| 22 | POST | `/policies/{id}/activate` | Activate |
| 23 | POST | `/policies/{id}/deactivate` | Deactivate |
| 24 | POST | `/policies/{id}/exceptions` | Add exception |
| 25 | DELETE | `/policies/{id}/exceptions/{exceptionId}` | Revoke exception |
| 26 | GET | `/integrations/adapters` | Adapter registry (static) |
| 27 | GET | `/integrations/connections` | Connection catalog |
| 28 | POST | `/integrations/connections` | Create connection |
| 29 | POST | `/integrations/connections/{id}/test` | Test connection |
| 30 | POST | `/integrations/connections/{id}/enable` | Enable |
| 31 | POST | `/integrations/connections/{id}/disable` | Disable |
| 32 | DELETE | `/integrations/connections/{id}` | Delete connection |

---

## Envelope Conventions

**Single-entity response:**

```json
{ "data": { /* entity */ } }
```

**List response:**

```json
{
  "data": [ /* entities */ ],
  "pagination": {
    "nextCursor": "eyJsYXN0SWQiOiJ0bXBsLTAwMDUiLCJsYXN0VHMiOiIyMDI2LTA0LTE4VDAyOjE1OjAwWiJ9",
    "total": 127
  }
}
```

**Error response:**

```json
{
  "error": {
    "code": "INVALID_TRANSITION",
    "message": "Template is not in draft status",
    "details": { "currentStatus": "published" }
  }
}
```

**HTTP codes:**

| Code | Meaning |
|------|---------|
| 200 | OK |
| 201 | Created |
| 204 | No Content |
| 400 | Bad request (malformed) |
| 403 | Forbidden (not `PLATFORM_ADMIN`) |
| 404 | Not found |
| 409 | Conflict (e.g., `INVALID_TRANSITION`, `IN_USE`, `LAST_PLATFORM_ADMIN`) |
| 422 | Unprocessable entity (schema / validation) |
| 500 | Server error |

---

## 1. `GET /access/me`

Returns the current user's identity and resolved roles. Called by the frontend route guard.

**Request:** none (session / auth resolved server-side; V1 reads a dev header or default seed admin).

**Response 200:**

```json
{
  "data": {
    "userId": "admin@sdlctower.local",
    "displayName": "Platform Admin",
    "roles": ["PLATFORM_ADMIN"],
    "scopes": [
      { "scopeType": "platform", "scopeId": "*" }
    ]
  }
}
```

**Response 200 for non-admin:**

```json
{
  "data": {
    "userId": "alice@corp",
    "displayName": "Alice",
    "roles": ["WORKSPACE_MEMBER"],
    "scopes": [
      { "scopeType": "workspace", "scopeId": "ws-default" }
    ]
  }
}
```

No 401/403 from this endpoint — the route guard decides UI behavior.

---

## 2. `GET /templates`

List templates with filters.

**Query parameters:**

| Param | Type | Default | Notes |
|-------|------|---------|-------|
| `kind` | `string` | — | one of `page,flow,project,policy,metric,ai` |
| `status` | `string` | — | `draft,published,deprecated` |
| `ownerId` | `string` | — | exact match |
| `q` | `string` | — | substring match on `name` or `key` |
| `limit` | `int` | `50` | max `200` |
| `cursor` | `string` | — | opaque pagination cursor |

**Response 200:**

```json
{
  "data": [
    {
      "id": "tmpl-proj-001",
      "key": "project-default-v1",
      "name": "Project Default (V1)",
      "kind": "project",
      "status": "published",
      "version": 1,
      "ownerId": "admin@sdlctower.local",
      "lastModifiedAt": "2026-04-18T02:15:00Z"
    },
    {
      "id": "tmpl-flow-001",
      "key": "requirement-flow-default",
      "name": "Requirement Flow Default",
      "kind": "flow",
      "status": "published",
      "version": 1,
      "ownerId": "admin@sdlctower.local",
      "lastModifiedAt": "2026-04-18T02:15:00Z"
    }
  ],
  "pagination": {
    "nextCursor": null,
    "total": 6
  }
}
```

---

## 3. `GET /templates/{id}`

Template detail with inheritance resolution.

**Path:** `id` — template id.

**Query:** `scope` — optional `<scopeType>:<scopeId>` used as the child scope for resolution (default `platform:*`).

**Response 200:**

```json
{
  "data": {
    "template": {
      "id": "tmpl-proj-001",
      "key": "project-default-v1",
      "name": "Project Default (V1)",
      "kind": "project",
      "status": "published",
      "version": 1,
      "ownerId": "admin@sdlctower.local",
      "description": "Default project template",
      "lastModifiedAt": "2026-04-18T02:15:00Z"
    },
    "version": {
      "id": "tver-proj-001-v1",
      "templateId": "tmpl-proj-001",
      "version": 1,
      "body": { "defaultMilestones": ["Kickoff","Design","Build"] },
      "createdAt": "2026-04-18T02:15:00Z",
      "createdBy": "admin@sdlctower.local"
    },
    "inheritance": {
      "defaultMilestones": {
        "effectiveValue": ["Kickoff","Design","Build"],
        "winningLayer": "platform",
        "layers": {
          "platform": ["Kickoff","Design","Build"],
          "application": null,
          "snowGroup": null,
          "project": null
        }
      }
    }
  }
}
```

**Errors:** 404 if template does not exist.

---

## 4. `GET /templates/{id}/versions`

**Response 200:**

```json
{
  "data": [
    {
      "id": "tver-proj-001-v3",
      "templateId": "tmpl-proj-001",
      "version": 3,
      "body": { "defaultMilestones": ["Kickoff","Design","Build","Rollout"] },
      "createdAt": "2026-04-15T10:00:00Z",
      "createdBy": "admin@sdlctower.local"
    },
    {
      "id": "tver-proj-001-v2",
      "templateId": "tmpl-proj-001",
      "version": 2,
      "body": { "defaultMilestones": ["Kickoff","Design","Build"] },
      "createdAt": "2026-03-02T10:00:00Z",
      "createdBy": "admin@sdlctower.local"
    }
  ]
}
```

---

## 5. `POST /templates/{id}/versions`

Create a new version from a body.

**Request:**

```json
{
  "body": { "defaultMilestones": ["Kickoff","Design","Build","QA","Rollout"] }
}
```

**Response 201:**

```json
{
  "data": {
    "id": "tver-proj-001-v4",
    "templateId": "tmpl-proj-001",
    "version": 4,
    "body": { "defaultMilestones": ["Kickoff","Design","Build","QA","Rollout"] },
    "createdAt": "2026-04-18T03:00:00Z",
    "createdBy": "admin@sdlctower.local"
  }
}
```

Side effect: audit insert with `category=config_change`, `action=template.new_version`.

**Errors:** 404 if template missing; 422 if body fails kind-schema validation.

---

## 6–8. Template Lifecycle

### `POST /templates/{id}/publish`

**Request:** empty body.

**Response 200:** updated `TemplateSummaryDto` wrapped in `data`.

**Errors:**

- 409 `INVALID_TRANSITION` if current status is not `draft`:
  ```json
  { "error": { "code": "INVALID_TRANSITION", "message": "Template is not in draft status", "details": { "currentStatus": "published" } } }
  ```

### `POST /templates/{id}/deprecate`

Same shape; requires current status `published`. 409 `INVALID_TRANSITION` otherwise.

### `POST /templates/{id}/reactivate`

Same shape; requires current status `deprecated`. 409 `INVALID_TRANSITION` otherwise.

Side effect for all three: audit insert with `category=config_change`, `action=template.publish|deprecate|reactivate`.

---

## 9. `DELETE /templates/{id}`

Delete a draft template with zero usages.

**Response 204** on success.

**Errors:**

- 404 if template missing
- 409 `NOT_DELETABLE` if status != `draft`:
  ```json
  { "error": { "code": "NOT_DELETABLE", "message": "Only draft templates can be deleted", "details": { "currentStatus": "published" } } }
  ```
- 409 `IN_USE` if `usage_count > 0`:
  ```json
  { "error": { "code": "IN_USE", "message": "Template is referenced by 3 scopes", "details": { "usageCount": 3 } } }
  ```

---

## 10. `GET /configurations`

**Query params:** `kind`, `scopeType`, `scopeId`, `status`, `q`, `limit`, `cursor`.

**Response 200:**

```json
{
  "data": [
    {
      "id": "cfg-001",
      "key": "req.page.layout",
      "kind": "page",
      "scopeType": "platform",
      "scopeId": "*",
      "parentId": null,
      "status": "active",
      "hasDrift": false,
      "lastModifiedAt": "2026-04-18T02:15:00Z"
    },
    {
      "id": "cfg-008",
      "key": "req.page.layout",
      "kind": "page",
      "scopeType": "workspace",
      "scopeId": "ws-default",
      "parentId": "cfg-001",
      "status": "active",
      "hasDrift": true,
      "lastModifiedAt": "2026-04-18T02:16:00Z"
    }
  ],
  "pagination": { "nextCursor": null, "total": 8 }
}
```

---

## 11. `POST /configurations`

Create a scope override.

**Request:**

```json
{
  "parentId": "cfg-001",
  "scopeType": "workspace",
  "scopeId": "ws-default",
  "body": { "columns": 3 }
}
```

**Response 201:** created `ConfigurationDetailDto`.

**Errors:**

- 404 if `parentId` missing
- 422 `INVALID_BODY` if `body` fails validation for the configuration's kind
- 409 `DUPLICATE_SCOPE_OVERRIDE` if an override already exists for the same `(key, kind, scopeType, scopeId)` combination

Side effect: audit `category=config_change`, `action=configuration.create`.

---

## 12. `DELETE /configurations/{id}`

Revert override.

**Response 204** on success.

**Errors:**

- 404 if missing
- 409 `CANNOT_REVERT_PLATFORM_DEFAULT` if the target is a platform-default row (`scopeType=platform` with no parent):
  ```json
  { "error": { "code": "CANNOT_REVERT_PLATFORM_DEFAULT", "message": "Platform defaults cannot be reverted" } }
  ```

Side effect: audit `category=config_change`, `action=configuration.revert`.

---

## 13. `GET /configurations/{id}/drift`

**Response 200:**

```json
{
  "data": {
    "hasDrift": true,
    "driftFields": ["columns"],
    "platformDefault": { "columns": 2 },
    "override": { "columns": 3 }
  }
}
```

For the platform-default configuration itself, returns `hasDrift: false` and `platformDefault: null`.

---

## 14. `GET /audit`

**Query params:**

| Param | Type | Notes |
|-------|------|-------|
| `category` | `string[]` (repeated) | multi-value |
| `actor` | `string` | exact match |
| `objectType` | `string` | exact match |
| `objectId` | `string` | exact match |
| `outcome` | `string` | `success,failure,rejected,rolled_back` |
| `scopeType` | `string` | |
| `scopeId` | `string` | |
| `timeRange` | `string` | `24h,7d,30d` or explicit ISO range `2026-04-01T00:00:00Z/2026-04-18T00:00:00Z` |
| `limit` | `int` | default 50, max 200 |
| `cursor` | `string` | |

**Response 200:**

```json
{
  "data": [
    {
      "id": "aud-0002",
      "timestamp": "2026-04-18T04:12:03Z",
      "actor": "admin@sdlctower.local",
      "actorType": "user",
      "category": "config_change",
      "action": "template.publish",
      "objectType": "template",
      "objectId": "tmpl-proj-001",
      "scopeType": "platform",
      "scopeId": "*",
      "outcome": "success",
      "evidenceRef": null,
      "payload": { "before": { "status": "draft" }, "after": { "status": "published" } }
    }
  ],
  "pagination": { "nextCursor": null, "total": 2 }
}
```

---

## 15. `GET /audit/{id}`

Returns a single record with the same shape as above (wrapped in `data`). 404 if missing.

---

## 16. `GET /access/assignments`

**Query params:** `userId`, `role`, `scopeType`, `scopeId`, `limit`, `cursor`.

**Response 200:**

```json
{
  "data": [
    {
      "id": "ra-default-admin",
      "userId": "admin@sdlctower.local",
      "userDisplayName": "Platform Admin",
      "role": "PLATFORM_ADMIN",
      "scopeType": "platform",
      "scopeId": "*",
      "grantedBy": "system",
      "grantedAt": "2026-04-18T02:15:00Z"
    }
  ],
  "pagination": { "nextCursor": null, "total": 1 }
}
```

---

## 17. `POST /access/assignments`

**Request:**

```json
{
  "userId": "alice@corp",
  "role": "WORKSPACE_ADMIN",
  "scopeType": "workspace",
  "scopeId": "ws-default"
}
```

**Response 201:** created `RoleAssignmentDto`.

**Errors:**

- 422 `INVALID_ROLE` if role is not one of the five
- 422 `INVALID_SCOPE_ID` if `scopeType=platform` and `scopeId != "*"`
- 409 `DUPLICATE_ASSIGNMENT` if an identical assignment already exists

Side effect: audit `category=permission_change`, `action=role.grant`.

---

## 18. `DELETE /access/assignments/{id}`

**Response 204** on success.

**Errors:**

- 404 if missing
- 409 `LAST_PLATFORM_ADMIN` if deleting would leave zero active `PLATFORM_ADMIN` assignments:
  ```json
  { "error": { "code": "LAST_PLATFORM_ADMIN", "message": "Cannot revoke the last platform administrator" } }
  ```

Side effect: audit `category=permission_change`, `action=role.revoke`.

---

## 19. `GET /policies`

**Query params:** `category`, `status`, `scopeType`, `scopeId`, `boundTo`, `q`, `limit`, `cursor`.

**Response 200:**

```json
{
  "data": [
    {
      "id": "pol-aut-001",
      "key": "skill.req-to-user-story.aut",
      "name": "req-to-user-story autonomy",
      "category": "autonomy",
      "scopeType": "platform",
      "scopeId": "*",
      "boundTo": "req-to-user-story",
      "version": 1,
      "status": "active",
      "body": { "defaultLevel": "L1-Assist", "escalation": "PLATFORM_ADMIN" },
      "createdAt": "2026-04-18T02:15:00Z",
      "createdBy": "admin@sdlctower.local"
    }
  ],
  "pagination": { "nextCursor": null, "total": 5 }
}
```

---

## 20. `POST /policies`

**Request:**

```json
{
  "key": "skill.incident-diagnosis.aut",
  "name": "incident-diagnosis autonomy",
  "category": "autonomy",
  "scopeType": "platform",
  "scopeId": "*",
  "boundTo": "incident-diagnosis",
  "body": { "defaultLevel": "L1-Assist" }
}
```

**Response 201:** created `PolicyDto` with `version=1`, `status="active"`.

**Errors:** 422 `INVALID_BODY` for body shape mismatch; 409 `DUPLICATE_KEY` if an active policy with same `(key, scopeType, scopeId)` already exists.

Side effect: audit `category=config_change`, `action=policy.create`.

---

## 21. `PUT /policies/{id}`

Edits a policy by creating a new version; previous version goes `inactive`.

**Request:**

```json
{
  "name": "incident-diagnosis autonomy (tightened)",
  "body": { "defaultLevel": "L0-Manual" }
}
```

**Response 200:** new `PolicyDto` with `version+1`, `status="active"`.

**Errors:** 404 if missing; 422 `INVALID_BODY`.

Side effect: audit `category=config_change`, `action=policy.update` with payload `{ "fromVersion": n, "toVersion": n+1, "diff": {...} }`.

---

## 22–23. `POST /policies/{id}/activate` and `/deactivate`

**Response 200:** updated `PolicyDto`.

Activate: requires `status != active`. Deactivate: requires `status = active`.

**Errors:** 404; 409 `INVALID_TRANSITION` on wrong current status.

Side effect: audit `action=policy.activate|deactivate`.

---

## 24. `POST /policies/{id}/exceptions`

**Request:**

```json
{
  "reason": "weekend hotfix exception requested by team lead",
  "requesterId": "bob@corp",
  "approverId": "admin@sdlctower.local",
  "expiresAt": "2026-05-01T00:00:00Z"
}
```

**Response 201:**

```json
{
  "data": {
    "id": "pex-0001",
    "policyId": "pol-app-001",
    "reason": "weekend hotfix exception requested by team lead",
    "requesterId": "bob@corp",
    "approverId": "admin@sdlctower.local",
    "createdAt": "2026-04-18T05:00:00Z",
    "expiresAt": "2026-05-01T00:00:00Z",
    "revokedAt": null
  }
}
```

**Errors:** 404 if policy missing; 422 `INVALID_EXPIRY` if `expiresAt <= now`.

Side effect: audit `action=policy.exception.add`.

---

## 25. `DELETE /policies/{id}/exceptions/{exceptionId}`

Revokes an exception early by setting `revoked_at = now`.

**Response 204.**

Side effect: audit `action=policy.exception.revoke`.

---

## 26. `GET /integrations/adapters`

Static registry.

**Response 200:**

```json
{
  "data": [
    { "kind": "jira",          "label": "Jira",           "supportedModes": ["pull","push","both"], "capabilities": ["requirement","issue-status"] },
    { "kind": "confluence",    "label": "Confluence",     "supportedModes": ["pull"],               "capabilities": ["requirement-source","business-context"] },
    { "kind": "gitlab",        "label": "GitLab",         "supportedModes": ["pull","both"],        "capabilities": ["code-change","merge-request","branch"] },
    { "kind": "jenkins",       "label": "Jenkins",        "supportedModes": ["push","both"],        "capabilities": ["build","pipeline","quality-gate"] },
    { "kind": "servicenow",    "label": "ServiceNow",     "supportedModes": ["pull","push","both"], "capabilities": ["incident","change","oncall"] },
    { "kind": "custom-webhook","label": "Custom Webhook", "supportedModes": ["push"],               "capabilities": ["generic"] }
  ]
}
```

---

## 27. `GET /integrations/connections`

**Query:** `kind`, `scopeWorkspaceId`, `applicationId`, `snowGroupId`, `status`, `syncMode`, `limit`, `cursor`.

**Response 200:**

```json
{
  "data": [
    {
      "id": "conn-jira-ws1",
      "kind": "jira",
      "scopeWorkspaceId": "ws-default",
      "applicationId": "app-payment-gateway-pro",
      "applicationName": "Payment-Gateway-Pro",
      "snowGroupId": "snow-fin-tech-ops",
      "snowGroupName": "FIN-TECH-OPS",
      "baseUrl": "https://jira.company.com",
      "credentialRef": "cred-jira-demo",
      "syncMode": "both",
      "pullSchedule": "0 */15 * * * *",
      "pushUrl": "https://webhook.local/jira",
      "status": "disabled",
      "lastSyncAt": null,
      "lastTestAt": null,
      "lastTestOk": null
    }
  ],
  "pagination": { "nextCursor": null, "total": 4 }
}
```

Credentials are NEVER returned in plain text.

---

## 28. `POST /integrations/connections`

**Request:**

```json
{
  "kind": "jira",
  "scopeWorkspaceId": "ws-default",
  "applicationId": "app-payment-gateway-pro",
  "applicationName": "Payment-Gateway-Pro",
  "snowGroupId": "snow-fin-tech-ops",
  "snowGroupName": "FIN-TECH-OPS",
  "baseUrl": "https://jira.company.com",
  "credentialRef": "cred-jira-demo",
  "syncMode": "both",
  "pullSchedule": "0 */15 * * * *",
  "pushUrl": "https://webhook.local/jira"
}
```

**Response 201:** created `ConnectionDto` with `status="enabled"`.

**Errors:** 422 `UNKNOWN_ADAPTER_KIND`, `UNKNOWN_CREDENTIAL_REF`, `INVALID_SYNC_CONFIG` (when schedule/URL not consistent with mode).

Side effect: audit `action=integration.create`.

---

## 29. `POST /integrations/connections/{id}/test`

**Response 200:**

```json
{
  "data": {
    "ok": true,
    "latencyMs": 182,
    "message": "Authenticated as demo-bot"
  }
}
```

**Failure 200 (business-logic failure surfaces as 200 with `ok=false`):**

```json
{
  "data": {
    "ok": false,
    "latencyMs": 10000,
    "message": "Timeout after 10s"
  }
}
```

Side effect: audit `category=integration.test`, `action=integration.test`, `outcome=success|failure`.

---

## 30–31. `POST /integrations/connections/{id}/enable` and `/disable`

**Response 200:** updated `ConnectionDto`.

**Errors:** 404; 409 `INVALID_TRANSITION` if already in the target state.

Side effect: audit `action=integration.enable|disable`.

---

## 32. `DELETE /integrations/connections/{id}`

**Response 204.**

Side effect: audit `action=integration.delete`.

---

## Backend Implementation Notes

### Package-by-feature layout

Refer to [platform-center-design.md §1.2](../platform-center-design.md) for the full Java file tree. Key rules:

- Each sub-package owns its controller, service, repository, DTOs, and entity
- Cross-cutting beans (`AuditWriter`, `AdminAuthGuard`, `InheritanceResolver`, `ScopeResolver`, `CursorCodec`) live in `platform.shared`
- No sub-package imports another (e.g., `policy` does not import `access`); they communicate via REST or the shared bean layer

### Controller skeleton (illustrative — Templates)

```java
@RestController
@RequestMapping("/api/v1/platform/templates")
@RequireAdmin
public class TemplateController {

    private final TemplateService service;
    public TemplateController(TemplateService service) { this.service = service; }

    @GetMapping
    public CursorPage<TemplateSummaryDto> list(@RequestParam(required = false) String kind,
                                               @RequestParam(required = false) String status,
                                               @RequestParam(required = false) String ownerId,
                                               @RequestParam(required = false) String q,
                                               @RequestParam(defaultValue = "50") int limit,
                                               @RequestParam(required = false) String cursor) {
        return service.list(new TemplateFilter(kind, status, ownerId, q, limit, cursor));
    }

    @GetMapping("/{id}")
    public Map<String,Object> detail(@PathVariable String id,
                                     @RequestParam(defaultValue = "platform:*") String scope) {
        return Map.of("data", service.detail(id, scope));
    }

    @PostMapping("/{id}/publish")
    public Map<String,Object> publish(@PathVariable String id, @AuthenticationPrincipal String actor) {
        return Map.of("data", service.publish(id, actor));
    }
    // ... other endpoints
}
```

### Service skeleton — atomic audit pattern

```java
@Service
public class TemplateService {

    private final TemplateRepository templates;
    private final AuditWriter audit;
    public TemplateService(TemplateRepository templates, AuditWriter audit) {
        this.templates = templates;
        this.audit = audit;
    }

    @Transactional
    public TemplateSummaryDto publish(String id, String actor) {
        Template t = templates.findById(id).orElseThrow(() -> new ResourceNotFoundException("template", id));
        if (!"draft".equals(t.getStatus())) {
            throw new InvalidTransitionException("Template is not in draft status", t.getStatus());
        }
        String before = t.getStatus();
        t.setStatus("published");
        templates.save(t);
        audit.write(AuditEvent.builder()
            .actor(actor).actorType("user")
            .category("config_change").action("template.publish")
            .objectType("template").objectId(id)
            .scopeType("platform").scopeId("*")
            .outcome("success")
            .payload(Map.of("before", Map.of("status", before), "after", Map.of("status", "published")))
            .build());
        return TemplateMapper.toSummaryDto(t);
    }
}
```

### AuditWriter

```java
@Service
public class AuditWriter {
    private final AuditRepository repo;
    private final Clock clock;
    public AuditWriter(AuditRepository repo, Clock clock) { this.repo = repo; this.clock = clock; }

    @Transactional(propagation = Propagation.MANDATORY)   // must run in caller tx
    public void write(AuditEvent event) {
        AuditRecord rec = AuditMapper.fromEvent(event, clock.instant(), UUID.randomUUID().toString());
        repo.save(rec);
    }
}
```

`Propagation.MANDATORY` ensures audit writes can never create their own transaction; if the caller forgets to open one, the insert fails — forcing the correct pattern at compile/test time.

### AdminAuthGuard

```java
@Target({ElementType.TYPE, ElementType.METHOD}) @Retention(RetentionPolicy.RUNTIME)
public @interface RequireAdmin {}

@Component
public class AdminAuthGuard implements HandlerInterceptor {
    private final AccessService access;
    public AdminAuthGuard(AccessService access) { this.access = access; }

    @Override
    public boolean preHandle(HttpServletRequest req, HttpServletResponse res, Object handler) throws Exception {
        if (!(handler instanceof HandlerMethod hm)) return true;
        boolean required = hm.getBeanType().isAnnotationPresent(RequireAdmin.class)
                           || hm.hasMethodAnnotation(RequireAdmin.class);
        if (!required) return true;
        if (access.currentUserHasRole("PLATFORM_ADMIN")) return true;
        res.setStatus(HttpStatus.FORBIDDEN.value());
        res.getWriter().write("{\"error\":{\"code\":\"FORBIDDEN\",\"message\":\"Platform admin required\"}}");
        return false;
    }
}
```

Register via `WebMvcConfigurer.addInterceptors(...)`.

### V1 auth stub

`AccessService.currentUserHasRole(role)` reads a request header `X-User-Id` (local profile only) or a seeded default (`admin@sdlctower.local`) when the header is absent. Production auth is wired in a future slice.

### CursorCodec

```java
public record Cursor(String lastId, Instant lastTs) {
    public String encode() {
        return Base64.getUrlEncoder().withoutPadding().encodeToString(
            ("{\"lastId\":\""+lastId+"\",\"lastTs\":\""+lastTs+"\"}").getBytes(UTF_8));
    }
    public static Cursor decode(String s) {
        if (s == null || s.isBlank()) return null;
        // parse base64 -> json -> fields
        ...
    }
}
```

---

## Frontend Implementation Notes

### API client (per sub-section)

```ts
// frontend/src/features/platform/templates/api.ts
import { http } from "@/shared/http";
import type { TemplateSummary, TemplateDetail, TemplateVersion } from "../shared/types";

export const templatesApi = {
  list: (filters: ListFilters) =>
    http.get<{ data: TemplateSummary[]; pagination: Pagination }>("/api/v1/platform/templates", { params: filters }),
  detail: (id: string, scope: string = "platform:*") =>
    http.get<{ data: TemplateDetail }>(`/api/v1/platform/templates/${id}`, { params: { scope } }),
  versions: (id: string) =>
    http.get<{ data: TemplateVersion[] }>(`/api/v1/platform/templates/${id}/versions`),
  publish: (id: string) =>
    http.post<{ data: TemplateSummary }>(`/api/v1/platform/templates/${id}/publish`),
  deprecate: (id: string) =>
    http.post<{ data: TemplateSummary }>(`/api/v1/platform/templates/${id}/deprecate`),
  reactivate: (id: string) =>
    http.post<{ data: TemplateSummary }>(`/api/v1/platform/templates/${id}/reactivate`),
  newVersion: (id: string, body: Record<string, unknown>) =>
    http.post<{ data: TemplateVersion }>(`/api/v1/platform/templates/${id}/versions`, { body }),
  delete: (id: string) => http.delete<void>(`/api/v1/platform/templates/${id}`),
};
```

### Pinia store (per sub-section)

```ts
// frontend/src/features/platform/templates/store.ts
export const useTemplatesStore = defineStore("platform-templates", {
  state: () => ({
    status: "idle" as "idle"|"loading"|"ready"|"error",
    error: null as string|null,
    items: [] as TemplateSummary[],
    filters: {} as Record<string, string>,
    cursor: null as string|null,
    total: null as number|null,
    selectedId: null as string|null,
    detail: null as TemplateDetail|null,
    lastFetchedAt: null as number|null,
  }),
  actions: {
    async fetchCatalog(force = false) { /* ... */ },
    async fetchDetail(id: string) { /* ... */ },
    async publish(id: string) { await templatesApi.publish(id); await this.fetchCatalog(true); },
    // ...
  }
});
```

### Mock toggle

`frontend/src/features/platform/shared/api.ts` exports a single flag:

```ts
export const USE_MOCK = import.meta.env.VITE_PLATFORM_USE_MOCK === "true";
```

Each sub-section's `api.ts` checks `USE_MOCK` and falls back to its `mocks.ts` fixtures when true. Phase A ships with `VITE_PLATFORM_USE_MOCK=true`; Phase B flips the flag.

### Vite proxy

Ensure `vite.config.ts` proxies `/api` to `http://localhost:8080` (existing pattern; verify on wire-up):

```ts
server: {
  proxy: {
    "/api": { target: "http://localhost:8080", changeOrigin: true }
  }
}
```

### Route guard

```ts
// frontend/src/features/platform/guard.ts
import { currentUserApi } from "./shared/api";

export const platformGuard: NavigationGuard = async (to, _from, next) => {
  try {
    const { data } = await currentUserApi.me();
    if (data.roles.includes("PLATFORM_ADMIN")) return next();
  } catch {
    // fall through
  }
  return next({ path: "/platform/forbidden" });
};
```

---

## Testing Contracts

### Backend — endpoint tests

| Test | Expectation |
|------|-------------|
| `GET /access/me` | 200; JSON shape matches; `roles` is a non-null array |
| `GET /templates` | 200; seeded rows (6) returned; filters narrow rows |
| `GET /templates/{id}` | 200 with inheritance; 404 for unknown id |
| `POST /templates/{id}/publish` draft→published | 200; audit row inserted; catalog reflects new status |
| `POST /templates/{id}/publish` published→ | 409 `INVALID_TRANSITION`; no audit row inserted |
| `DELETE /templates/{id}` draft with usage | 409 `IN_USE` |
| `GET /configurations?scopeType=workspace&scopeId=ws-default` | Returns only workspace-scoped rows |
| `POST /configurations` duplicate | 409 `DUPLICATE_SCOPE_OVERRIDE` |
| `DELETE /access/assignments/{last admin}` | 409 `LAST_PLATFORM_ADMIN` |
| `PUT /policies/{id}` | New version row; previous row flipped to `inactive` |
| `POST /integrations/connections/{id}/test` | `ok: true` with `latencyMs` > 0 (stubbed adapter) |

### Backend — integration test for atomic audit

- Mock `AuditRepository.save` to throw after any mutation
- Verify the mutation is rolled back (DB unchanged)
- Verify HTTP response is 500 with `INTERNAL_ERROR`

### Frontend — feature tests

| Test | Expectation |
|------|-------------|
| Route guard redirects non-admin to `/platform/forbidden` | `next({path: "/platform/forbidden"})` called |
| Templates catalog renders 6 seeded rows | `items.length === 6` after `fetchCatalog` |
| Publish button opens destructive-confirm with target name | Modal body includes template name verbatim |
| Publish success refreshes catalog | `fetchCatalog` called twice (initial + after mutation) |
| Audit filters serialize to URL | Query string reflects filters |
| Forbidden view renders "Back to Dashboard" button | `/` navigation on click |

---

## Versioning Policy

- API version is hard-coded `/api/v1/` for this slice
- Adding fields to a response is non-breaking; clients must ignore unknown fields
- Removing or renaming fields requires `/api/v2/` and a migration window
- Error `code` values are stable; messages may change
- Enum string values for `kind`, `category`, `role`, etc., are stable; new values may be added (clients should tolerate unknown values rather than crash)
