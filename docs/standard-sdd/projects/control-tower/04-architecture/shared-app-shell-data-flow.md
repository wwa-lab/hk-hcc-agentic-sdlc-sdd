# Shared App Shell Data Flow

## Purpose

This document details the runtime data flows for the Shared App Shell,
covering workspace context loading, navigation rendering, theme switching,
route transitions, and frontend-backend integration.

## Traceability

- Architecture: [shared-app-shell-architecture.md](shared-app-shell-architecture.md)
- Design: [shared-app-shell-design.md](../05-design/shared-app-shell-design.md)
- Spec: [shared-app-shell-spec.md](../03-spec/shared-app-shell-spec.md)

---

## 1. Application Bootstrap Flow

```mermaid
sequenceDiagram
    participant Browser
    participant Vite as Vite Dev Server
    participant App as main.ts
    participant Router as Vue Router
    participant Shell as AppShell.vue
    participant WS as workspaceStore
    participant API as workspaceApi.ts
    participant Client as fetchJson (client.ts)
    participant Backend as Spring Boot

    Browser->>Vite: GET /
    Vite-->>Browser: index.html + JS bundle
    Browser->>App: Mount Vue app
    App->>Router: Install router
    App->>Router: Resolve initial route (/)
    Router->>Shell: Mount AppShell
    Shell->>WS: onMounted() calls load()
    WS->>API: getWorkspaceContext()
    API->>Client: fetchJson('/workspace-context')
    Client->>Backend: GET /api/v1/workspace-context
    Backend-->>Client: ApiResponse with WorkspaceContextDto
    Client->>Client: Unwrap envelope
    Client-->>API: WorkspaceContext
    API-->>WS: WorkspaceContext
    WS->>WS: Set context = data, loading = false
    WS-->>Shell: Reactive state update
    Shell->>Shell: Render TopContextBar with context
    Router->>Shell: Mount page view via router-view
```

---

## 2. Workspace Context Loading

```mermaid
sequenceDiagram
    participant Shell as AppShell
    participant WS as workspaceStore
    participant API as workspaceApi.ts
    participant Backend as WorkspaceContextController

    Shell->>WS: load()
    WS->>WS: loading = true, error = null
    WS->>API: getWorkspaceContext()
    API->>Backend: GET /api/v1/workspace-context

    alt Success
        Backend-->>API: ApiResponse ok(WorkspaceContextDto)
        API-->>WS: WorkspaceContext
        WS->>WS: context = data, loading = false
    else API Error (network, 500)
        Backend-->>API: Error
        API-->>WS: throw ApiError
        WS->>WS: error = message, loading = false
    else Empty (404)
        Backend-->>API: 404
        API-->>WS: throw ApiError(404)
        WS->>WS: error = "Not found", loading = false
    end

    WS-->>Shell: Reactive update triggers re-render
```

### Context Display States

| Store State | TopContextBar Renders |
|-------------|----------------------|
| `loading = true` | "Loading workspace context..." |
| `error != null` | "Context unavailable" + retry button |
| `context` loaded, optional field null | Field shows `---` placeholder |
| `context` loaded, all fields present | Full 5-field context chain |

---

## 3. Navigation Rendering Flow

```mermaid
sequenceDiagram
    participant Shell as AppShell
    participant Nav as PrimaryNav
    participant Router as Vue Router

    Shell->>Nav: Mount component
    Nav->>Nav: Import NAVIGATION_ITEMS (static)
    Nav->>Router: Watch route.meta.navKey
    Router-->>Nav: Current navKey
    Nav->>Nav: Render 13 items
    Nav->>Nav: Highlight item matching navKey
    Nav->>Nav: Dim items with comingSoon = true
    Nav->>Nav: Render system status LED in footer
```

### Navigation Item Sources

V1 uses a static `NAVIGATION_ITEMS` array imported from `router/index.ts`.
The backend `GET /api/v1/nav/entries` endpoint exists for future dynamic navigation.

```mermaid
graph LR
    subgraph V1["V1 - Static"]
        ROUTER["router/index.ts<br/>NAVIGATION_ITEMS[]"]
        NAV["PrimaryNav.vue"]
    end

    subgraph Future["V2+ - Dynamic"]
        NAV_API["GET /api/v1/nav/entries"]
        NAV_SVC["NavigationService"]
        NAV2["PrimaryNav.vue"]
    end

    ROUTER -->|import| NAV
    NAV_API -->|fetch| NAV2
    NAV_SVC -->|returns| NAV_API
```

---

## 4. Route Transition Flow

```mermaid
sequenceDiagram
    participant User
    participant Nav as PrimaryNav
    participant Router as Vue Router
    participant Shell as AppShell
    participant PH as PageHeader
    participant Old as Previous PageView
    participant New as New PageView

    User->>Nav: Click nav item
    Nav->>Router: router.push(item.path)
    Router->>Router: Resolve route + meta
    Router->>Shell: Route change event
    Shell->>PH: Update title/subtitle from route.meta
    Shell->>Old: Unmount previous page
    Shell->>New: Mount new page via router-view
    New->>New: onMounted() — load page data
    Nav->>Nav: Update active indicator to new navKey
```

### Route Metadata Delivery

Each route defines a `ShellPageConfig` in its `meta` field:

```mermaid
graph LR
    ROUTE["Route Definition<br/>meta: ShellPageConfig"]
    PH["PageHeader<br/>title, subtitle, actions"]
    NAV["PrimaryNav<br/>navKey for active state"]
    DR["DataRibbon<br/>page context"]

    ROUTE -->|meta.title, meta.subtitle| PH
    ROUTE -->|meta.navKey| NAV
    ROUTE -->|meta.navKey| DR
```

---

## 5. Theme Toggle Flow

```mermaid
sequenceDiagram
    participant User
    participant GAB as GlobalActionBar
    participant Theme as useTheme composable
    participant LS as localStorage
    participant HTML as document.documentElement

    User->>GAB: Click theme toggle
    GAB->>Theme: toggleTheme()
    Theme->>Theme: current = (current === 'dark') ? 'light' : 'dark'
    Theme->>LS: setItem('sdlc-tower-theme', newTheme)
    Theme->>HTML: setAttribute('data-theme', newTheme)
    HTML-->>HTML: CSS variables switch
    HTML-->>User: UI re-renders with new theme
```

### Theme State Machine

```mermaid
graph LR
    INIT["Init<br/>Read localStorage"]
    DARK["dark<br/>data-theme=dark"]
    LIGHT["light<br/>data-theme=light"]

    INIT -->|"stored = dark OR no stored"| DARK
    INIT -->|"stored = light"| LIGHT
    DARK -->|toggleTheme| LIGHT
    LIGHT -->|toggleTheme| DARK
```

Default is `dark` ("Tactical Command" mode). Theme preference persists
in `localStorage` under key `sdlc-tower-theme`.

---

## 6. Page-Shell Communication

```mermaid
graph TD
    subgraph Shell["Shell Provides"]
        WC["WorkspaceContext (Pinia)"]
        THEME["Theme state"]
        NAV_STATE["Active nav key"]
        AI_CONTAINER["AI panel container"]
    end

    subgraph Page["Page Provides"]
        META["Route meta (ShellPageConfig)"]
        BODY["Page body content"]
        AI_CONTENT["AI panel content (future)"]
    end

    subgraph Contract["Communication Contract"]
        ROUTER_META["Vue Router meta"]
        PINIA["Pinia stores"]
        SLOT["router-view slot"]
    end

    META -->|via| ROUTER_META
    WC -->|via| PINIA
    BODY -->|via| SLOT
```

### Communication Channels

| Direction | Channel | Data |
|-----------|---------|------|
| Shell to Page | Pinia store (`workspaceStore`) | WorkspaceContext, loading, error |
| Shell to Page | `useTheme()` composable | Current theme, toggle function |
| Page to Shell | Route `meta` field | navKey, title, subtitle, actions |
| Page to Shell | `router-view` slot | Page body content |
| Page to Shell (future) | TBD | AI panel content for current page |

---

## 7. Error Cascade Boundaries

```mermaid
graph TD
    subgraph Always["Always Renders"]
        NAV["PrimaryNav"]
        GAB["GlobalActionBar"]
        AI["AiCommandPanel (container)"]
    end

    subgraph Degradable["Can Degrade Independently"]
        CTX["TopContextBar<br/>Shows error state"]
        PH["PageHeader<br/>Fallback title"]
        PAGE["PageContent<br/>Page-level error"]
    end

    CTX -->|"API fail"| CTX_ERR["Context unavailable + retry"]
    PH -->|"No meta"| PH_FALLBACK["navKey OPERATIONAL VIEW"]
    PAGE -->|"Page error"| PAGE_ERR["Page error — shell intact"]
```

### Error Isolation Rules

| Failure | Impact | Shell Behavior |
|---------|--------|----------------|
| Workspace API fails | TopContextBar only | Shows error + retry; rest of shell normal |
| Page view throws | Page content only | Shell frame remains; only router-view affected |
| Navigation API fails (future) | Nav items | Falls back to static NAVIGATION_ITEMS |
| Theme fails to persist | Theme toggle | Defaults to dark; no visible error |

---

## 8. Backend Processing Chain

```mermaid
graph LR
    subgraph Frontend["Frontend"]
        FE_WS["workspaceApi.ts"]
        FE_NAV["navigationApi.ts"]
        CLIENT["client.ts<br/>fetchJson"]
    end

    subgraph Proxy["Vite Dev Proxy"]
        VP["/api/v1 -> :8080"]
    end

    subgraph Backend["Spring Boot"]
        WCC["WorkspaceContextController"]
        WCS["WorkspaceContextService"]
        WCR["WorkspaceContextRepository"]
        NC["NavigationController"]
        NS["NavigationService"]
    end

    subgraph DB["Database"]
        H2["H2 (dev) / Oracle (prod)"]
    end

    FE_WS -->|calls| CLIENT
    FE_NAV -->|calls| CLIENT
    CLIENT -->|HTTP| VP
    VP -->|forward| WCC
    VP -->|forward| NC
    WCC -->|delegates| WCS
    WCS -->|queries| WCR
    WCR -->|JPA| H2
    NC -->|delegates| NS
    NS -->|static list V1| NS
```

### Backend Call Chain — Workspace Context

1. `WorkspaceContextController.getWorkspaceContext()` receives GET
2. Delegates to `WorkspaceContextService.getCurrentWorkspaceContext()`
3. Service queries `WorkspaceContextRepository` (JPA)
4. Entity converted to `WorkspaceContextDto.fromEntity(entity)`
5. Wrapped in `ApiResponse.ok(dto)`, returned as JSON

### Backend Call Chain — Navigation Entries

1. `NavigationController.getEntries()` receives GET
2. Delegates to `NavigationService.getEntries()`
3. Service returns hardcoded `List<NavItem>` (V1 — no DB)
4. Wrapped in `ApiResponse.ok(list)`, returned as JSON
