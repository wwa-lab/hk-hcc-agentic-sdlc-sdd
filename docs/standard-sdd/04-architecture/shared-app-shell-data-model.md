# Shared App Shell Data Model

## Purpose

This document defines the domain and persistent data model for the Shared App Shell,
covering frontend types, backend DTOs/entities, and the database schema.

## Traceability

- Architecture: [shared-app-shell-architecture.md](shared-app-shell-architecture.md)
- Design: [shared-app-shell-design.md](../05-design/shared-app-shell-design.md)
- Spec: [shared-app-shell-spec.md](../03-spec/shared-app-shell-spec.md)
- Types source: `frontend/src/shared/types/shell.ts`

---

## 1. Domain Model Overview

```mermaid
graph TD
    subgraph Shell["Shell Domain"]
        WC["WorkspaceContext"]
        NI["NavItem"]
        SPC["ShellPageConfig"]
        APC["AiPanelContent"]
    end

    subgraph ValueObjects["Value Objects"]
        SA["ShellAction"]
        RB["ReasoningBullet"]
        SS["SystemStatus"]
    end

    SPC -->|"0:N"| SA
    APC -->|"0:N"| RB
    WC -.->|"drives"| SPC
    NI -.->|"drives"| SPC
```

---

## 2. Frontend Type Model

All types are defined in `frontend/src/shared/types/shell.ts`.

### 2.1 WorkspaceContext

The global workspace context displayed in the TopContextBar.

```typescript
interface WorkspaceContext {
  workspace: string;              // Required — workspace name
  application: string;            // Required — application name
  snowGroup?: string | null;      // Optional — ServiceNow group
  project?: string | null;        // Optional — project name
  environment?: string | null;    // Optional — environment name
}
```

| Field | Required | Display Label | Empty Fallback |
|-------|----------|---------------|----------------|
| `workspace` | Yes | "Workspace" | N/A (always present) |
| `application` | Yes | "Application" | N/A (always present) |
| `snowGroup` | No | "SNOW Group" | `---` |
| `project` | No | "Project" | `---` |
| `environment` | No | "Environment" | `---` |

### 2.2 ShellPageConfig

Delivered via Vue Router route `meta` field. Drives PageHeader rendering.

```typescript
interface ShellPageConfig {
  navKey: string;                                   // Maps to NavItem.key for active state
  title: string;                                    // Page title
  subtitle?: string;                                // Optional subtitle
  actions?: ReadonlyArray<ShellAction>;              // Optional action buttons
}

interface ShellAction {
  key: string;                                      // Unique action identifier
  label: string;                                    // Button label
  variant?: 'default' | 'ai';                       // Visual variant
}
```

### 2.3 NavItem

Navigation entry rendered in PrimaryNav.

```typescript
interface NavItem {
  key: string;              // Unique route key (e.g., "dashboard", "incidents")
  label: string;            // Display label (e.g., "Incident Management")
  path: string;             // Route path (e.g., "/incidents")
  icon: string;             // Lucide icon name (e.g., "LayoutDashboard")
  comingSoon?: boolean;     // If true, item renders dimmed
}
```

### 2.4 AiPanelContent

Content for the AI Command Panel zones.

```typescript
interface AiPanelContent {
  summary: string;
  reasoning: ReadonlyArray<{
    text: string;
    status: 'ok' | 'warning' | 'error';
  }>;
  evidence: string;
}
```

### 2.5 SystemStatus

Status LED in the nav footer.

```typescript
type SystemStatus = 'ready' | 'degraded' | 'offline';
```

| Status | LED Color | Token |
|--------|-----------|-------|
| `ready` | Green | `--color-health-emerald` |
| `degraded` | Amber | `--color-approval-amber` |
| `offline` | Red | `--color-incident-crimson` |

---

## 3. Backend DTO/Entity Model

### 3.1 WorkspaceContext Entity (JPA)

```java
package com.sdlctower.platform.workspace;

@Entity
@Table(name = "workspace_context")
public class WorkspaceContext {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "workspace_name", nullable = false)
    private String workspace;

    @Column(name = "application_name", nullable = false)
    private String application;

    @Column(name = "snow_group")
    private String snowGroup;

    @Column(name = "project_name")
    private String project;

    @Column(name = "environment_name")
    private String environment;

    // getters (no setters — managed by JPA lifecycle)
}
```

### 3.2 WorkspaceContextDto (API-facing)

```java
package com.sdlctower.platform.workspace;

public record WorkspaceContextDto(
    String workspace,
    String application,
    String snowGroup,      // nullable
    String project,        // nullable
    String environment     // nullable
) {
    public static WorkspaceContextDto fromEntity(WorkspaceContext entity) {
        return new WorkspaceContextDto(
            entity.getWorkspace(),
            entity.getApplication(),
            entity.getSnowGroup(),
            entity.getProject(),
            entity.getEnvironment()
        );
    }
}
```

### 3.3 NavItem Record (No Entity — Static in V1)

```java
package com.sdlctower.platform.navigation;

public record NavItem(
    String key,
    String label,
    String path,
    boolean comingSoon,
    String icon,
    int order
) {}
```

V1: Hardcoded in `NavigationService`. No database table.
V2+: May be driven by a `navigation_item` table for dynamic configuration.

### 3.4 ApiResponse Envelope (Shared)

```java
package com.sdlctower.shared.dto;

public record ApiResponse<T>(T data, String error) {
    public static <T> ApiResponse<T> ok(T data) {
        return new ApiResponse<>(data, null);
    }
    public static <T> ApiResponse<T> fail(String error) {
        return new ApiResponse<>(null, error);
    }
}
```

---

## 4. Type Mapping — Frontend to Backend

| Frontend (TypeScript) | Backend (Java) | JSON Key | Nullable |
|-----------------------|----------------|----------|----------|
| `WorkspaceContext.workspace` | `WorkspaceContextDto.workspace` | `workspace` | No |
| `WorkspaceContext.application` | `WorkspaceContextDto.application` | `application` | No |
| `WorkspaceContext.snowGroup` | `WorkspaceContextDto.snowGroup` | `snowGroup` | Yes |
| `WorkspaceContext.project` | `WorkspaceContextDto.project` | `project` | Yes |
| `WorkspaceContext.environment` | `WorkspaceContextDto.environment` | `environment` | Yes |
| `NavItem.key` | `NavItem.key` | `key` | No |
| `NavItem.label` | `NavItem.label` | `label` | No |
| `NavItem.path` | `NavItem.path` | `path` | No |
| `NavItem.icon` | `NavItem.icon` | `icon` | No |
| `NavItem.comingSoon` | `NavItem.comingSoon` | `comingSoon` | No (default false) |
| — | `NavItem.order` | `order` | No (backend only) |

---

## 5. Database Schema

Managed by Flyway migrations. JPA `ddl-auto` is only used for local H2 throwaway dev.

### 5.1 Entity Relationship Diagram

```mermaid
graph TD
    subgraph Shell["Shell Tables"]
        WCT["workspace_context"]
    end

    subgraph Future["Future Tables"]
        NIT["navigation_item (V2+)"]
        TP["theme_preference (V2+)"]
    end

    WCT -.->|"future: FK"| NIT
```

### 5.2 workspace_context Table

```sql
-- V1__create_workspace_context.sql
CREATE TABLE workspace_context (
    id                BIGINT GENERATED BY DEFAULT AS IDENTITY PRIMARY KEY,
    workspace_name    VARCHAR(255) NOT NULL,
    application_name  VARCHAR(255) NOT NULL,
    snow_group        VARCHAR(255),
    project_name      VARCHAR(255),
    environment_name  VARCHAR(255)
);
```

### 5.3 Seed Data

```sql
-- V2__seed_workspace_context.sql
INSERT INTO workspace_context (
    workspace_name, application_name, snow_group, project_name, environment_name
) VALUES (
    'Global SDLC Tower',
    'Payment-Gateway-Pro',
    'FIN-TECH-OPS',
    'Q2-Cloud-Migration',
    'Production'
);
```

### 5.4 Column-to-Field Mapping

| DB Column | Java Entity Field | DTO Field | TypeScript Field |
|-----------|-------------------|-----------|------------------|
| `id` | `id` | (excluded) | (not exposed) |
| `workspace_name` | `workspace` | `workspace` | `workspace` |
| `application_name` | `application` | `application` | `application` |
| `snow_group` | `snowGroup` | `snowGroup` | `snowGroup` |
| `project_name` | `project` | `project` | `project` |
| `environment_name` | `environment` | `environment` | `environment` |

---

## 6. State Model

### 6.1 Pinia Store State

```mermaid
graph TD
    subgraph WorkspaceStore["workspaceStore (Pinia)"]
        CTX["context: WorkspaceContext"]
        LOAD["loading: boolean"]
        ERR["error: string | null"]
    end

    subgraph Consumers["Consumers"]
        TCB["TopContextBar"]
        DR["DataRibbon"]
        PAGES["Page views (future)"]
    end

    CTX -->|readonly| TCB
    CTX -->|readonly| DR
    CTX -->|readonly| PAGES
    LOAD -->|readonly| TCB
    ERR -->|readonly| TCB
```

### 6.2 Store State Machine

```mermaid
graph LR
    INIT["init<br/>context: empty<br/>loading: false<br/>error: null"]
    LOADING["loading<br/>context: empty<br/>loading: true<br/>error: null"]
    LOADED["loaded<br/>context: data<br/>loading: false<br/>error: null"]
    ERROR["error<br/>context: empty<br/>loading: false<br/>error: string"]

    INIT -->|"load()"| LOADING
    LOADING -->|"success"| LOADED
    LOADING -->|"failure"| ERROR
    ERROR -->|"retry / load()"| LOADING
```

### 6.3 Theme State (localStorage)

| Key | Type | Default | Persisted |
|-----|------|---------|-----------|
| `sdlc-tower-theme` | `'dark' \| 'light'` | `'dark'` | localStorage |

---

## 7. Navigation Items (V1 Static Data)

| # | Key | Label | Path | Coming Soon | Icon |
|---|-----|-------|------|-------------|------|
| 1 | `dashboard` | Dashboard | `/` | false | LayoutDashboard |
| 2 | `team` | Team Space | `/team` | true | Users |
| 3 | `project-space` | Project Space | `/project-space` | false | FolderKanban |
| 4 | `requirements` | Requirement Mgmt | `/requirements` | true | FileText |
| 5 | `project-management` | Project Mgmt | `/project-management` | true | Kanban |
| 6 | `design` | Design Mgmt | `/design` | true | Palette |
| 7 | `code` | Code & Build | `/code` | true | Code |
| 8 | `testing` | Testing | `/testing` | true | TestTube |
| 9 | `deployment` | Deployment | `/deployment` | true | Rocket |
| 10 | `incidents` | Incident Mgmt | `/incidents` | false | AlertTriangle |
| 11 | `ai-center` | AI Center | `/ai-center` | true | Bot |
| 12 | `reports` | Report Center | `/reports` | true | BarChart3 |
| 13 | `platform` | Platform Center | `/platform` | false | Settings |
