# Codex Prompt: Phase B — Backend Implementation (Spring Boot)

## Task

Build the backend for **Agentic SDLC Control Tower**. The frontend (Vue 3) is already complete under `frontend/` and uses mocked data. Your job is to create a Spring Boot backend that serves the same data via REST APIs, so the frontend can switch from mocks to live API calls.

## Tech Stack

| Layer | Technology |
|-------|-----------|
| Framework | Spring Boot 3.x (Java 21) |
| Build | Maven (with Maven Wrapper) |
| ORM | JPA / Hibernate |
| DB (local) | H2 in-memory (Spring profile: `local`) |
| DB (production) | Oracle (Spring profile: `prod`) |
| API style | REST, JSON |
| Monitoring | Spring Boot Actuator |

## Package Architecture

Use **package-by-feature** (not package-by-layer). The system has three layers:

| Layer | Package | Purpose | Used by |
|-------|---------|---------|---------|
| `platform` | Shared platform capabilities | Workspace, Navigation, Audit, Access, Template, Configuration | All domains |
| `domain` | SDLC business domains | Dashboard, Project, Incident, Requirement, etc. | Each domain is independent |
| `shared` | Cross-cutting utilities | DTOs, exceptions, common utils | Everyone |

### Why this structure

The PRD defines **platform capabilities** (§12) that every team and every SDLC stage reuses — workspace context, navigation, audit, access, templates, configuration. These live in `platform/`.

Each **SDLC stage** (Incident Management, Deployment, Testing, etc.) is a separate business domain with its own data model, service logic, and API. These live in `domain/`. Domains do NOT directly import each other — if they need to communicate, they go through `platform/` services or events.

`shared/` holds truly generic code: base exception classes, response wrappers, and utility methods.

## Project Structure

Create the backend under `backend/` at the project root:

```
backend/
├── pom.xml
├── mvnw / mvnw.cmd
├── src/
│   ├── main/
│   │   ├── java/com/sdlctower/
│   │   │   ├── SdlcTowerApplication.java
│   │   │   │
│   │   │   ├── config/                          # Global configuration
│   │   │   │   └── CorsConfig.java
│   │   │   │
│   │   │   ├── platform/                        # Shared platform capabilities
│   │   │   │   ├── workspace/                   # Workspace context (V1 focus)
│   │   │   │   │   ├── WorkspaceContext.java           (entity)
│   │   │   │   │   ├── WorkspaceContextRepository.java
│   │   │   │   │   ├── WorkspaceContextService.java
│   │   │   │   │   ├── WorkspaceContextController.java
│   │   │   │   │   └── WorkspaceContextSeeder.java     (H2 seed data)
│   │   │   │   └── navigation/                  # Navigation configuration (V1 focus)
│   │   │   │       ├── NavItem.java                    (DTO, not an entity)
│   │   │   │       ├── NavigationService.java
│   │   │   │       └── NavigationController.java
│   │   │   │
│   │   │   ├── domain/                          # Business domains (empty in V1, ready for future)
│   │   │   │   └── .gitkeep
│   │   │   │
│   │   │   └── shared/                          # Cross-cutting utilities
│   │   │       ├── exception/
│   │   │       │   └── ResourceNotFoundException.java
│   │   │       └── dto/
│   │   │           └── ApiResponse.java
│   │   │
│   │   └── resources/
│   │       ├── application.yml
│   │       ├── application-local.yml
│   │       └── application-prod.yml
│   │
│   └── test/
│       └── java/com/sdlctower/
│           ├── platform/
│           │   ├── workspace/
│           │   │   └── WorkspaceContextControllerTest.java
│           │   └── navigation/
│           │       └── NavigationControllerTest.java
│           └── SdlcTowerApplicationTest.java
```

### Future domain expansion example (do NOT implement, just for context)

When Incident Management is added later, it will follow the same pattern:

```
domain/
└── incident/
    ├── Incident.java              (entity)
    ├── IncidentRepository.java
    ├── IncidentService.java
    └── IncidentController.java
```

It can depend on `platform.workspace` (for context scoping) and `shared.exception` (for error handling), but NOT on `domain.dashboard` or `domain.project`.

## API Contracts

These must match the existing frontend TypeScript types exactly.

### API 1: `GET /api/v1/workspace-context`

Returns the current workspace context. The frontend type it must match:

```typescript
interface WorkspaceContext {
  workspace: string;
  application: string;
  snowGroup?: string | null;    // JSON: "snowGroup" (camelCase)
  project?: string | null;
  environment?: string | null;
}
```

Example response:

```json
{
  "workspace": "Global SDLC Tower",
  "application": "Payment-Gateway-Pro",
  "snowGroup": "FIN-TECH-OPS",
  "project": "Q2-Cloud-Migration",
  "environment": "Production"
}
```

Implementation notes:
- JPA entity in `platform.workspace` package with `@Entity`
- Seed H2 with sample data via `WorkspaceContextSeeder` (`CommandLineRunner`, active on `local` profile only)
- `snowGroup`, `project`, `environment` are nullable columns
- V1: return the first (or default) workspace context row. Multi-workspace selection is deferred.

### API 2: `GET /api/v1/nav/entries`

Returns the 13 navigation items. The frontend type it must match:

```typescript
interface NavItem {
  key: string;
  label: string;
  path: string;
  comingSoon?: boolean;
}
```

Expected response (exact order):

```json
[
  { "key": "dashboard", "label": "Dashboard", "path": "/", "comingSoon": false },
  { "key": "team", "label": "Team Space", "path": "/team", "comingSoon": true },
  { "key": "project-space", "label": "Project Space", "path": "/project-space", "comingSoon": false },
  { "key": "requirements", "label": "Requirement Management", "path": "/requirements", "comingSoon": true },
  { "key": "project-management", "label": "Project Management", "path": "/project-management", "comingSoon": true },
  { "key": "design", "label": "Design Management", "path": "/design", "comingSoon": true },
  { "key": "code", "label": "Code & Build", "path": "/code", "comingSoon": true },
  { "key": "testing", "label": "Testing", "path": "/testing", "comingSoon": true },
  { "key": "deployment", "label": "Deployment", "path": "/deployment", "comingSoon": true },
  { "key": "incidents", "label": "Incident Management", "path": "/incidents", "comingSoon": false },
  { "key": "ai-center", "label": "AI Center", "path": "/ai-center", "comingSoon": true },
  { "key": "reports", "label": "Report Center", "path": "/reports", "comingSoon": true },
  { "key": "platform", "label": "Platform Center", "path": "/platform", "comingSoon": false }
]
```

Implementation notes:
- `NavItem` is a DTO (record or plain class), NOT a JPA entity
- V1: navigation data is static, hardcoded in `NavigationService`
- Served from backend to enable future permission-based filtering

## Spring Profiles

### `application.yml` (shared)

```yaml
server:
  port: 8080

spring:
  profiles:
    active: local
  jpa:
    open-in-view: false

management:
  endpoints:
    web:
      exposure:
        include: health,info
```

### `application-local.yml`

```yaml
spring:
  datasource:
    url: jdbc:h2:mem:sdlctower;DB_CLOSE_DELAY=-1
    driver-class-name: org.h2.Driver
    username: sa
    password:
  h2:
    console:
      enabled: true
      path: /h2-console
  jpa:
    hibernate:
      ddl-auto: create-drop
    show-sql: true
```

### `application-prod.yml`

```yaml
spring:
  datasource:
    url: jdbc:oracle:thin:@${ORACLE_HOST:localhost}:${ORACLE_PORT:1521}:${ORACLE_SID:ORCL}
    driver-class-name: oracle.jdbc.OracleDriver
    username: ${ORACLE_USER}
    password: ${ORACLE_PASSWORD}
  jpa:
    hibernate:
      ddl-auto: validate
    show-sql: false
```

## CORS Configuration

The frontend runs on `http://localhost:5173` (Vite dev server). Configure CORS in `config/CorsConfig.java`:

- Origin: `http://localhost:5173`
- Methods: GET, POST, PUT, DELETE, OPTIONS
- Headers: all
- Credentials: true

## H2 Seed Data

Seed via `WorkspaceContextSeeder.java` in `platform.workspace` package:
- Implements `CommandLineRunner`
- Active only on `local` profile (`@Profile("local")`)

```java
WorkspaceContext ctx = new WorkspaceContext();
ctx.setWorkspace("Global SDLC Tower");
ctx.setApplication("Payment-Gateway-Pro");
ctx.setSnowGroup("FIN-TECH-OPS");
ctx.setProject("Q2-Cloud-Migration");
ctx.setEnvironment("Production");
repository.save(ctx);
```

## Shared Utilities

### `ApiResponse<T>` (optional wrapper)

```java
public record ApiResponse<T>(T data, String error) {
    public static <T> ApiResponse<T> ok(T data) {
        return new ApiResponse<>(data, null);
    }
    public static <T> ApiResponse<T> fail(String error) {
        return new ApiResponse<>(null, error);
    }
}
```

V1 APIs can return raw JSON directly. The wrapper is available for future use when error handling becomes more structured.

### `ResourceNotFoundException`

Standard 404 exception with `@ResponseStatus(HttpStatus.NOT_FOUND)`.

## Testing Requirements

Write tests using Spring Boot Test + MockMvc. Test classes mirror the source package structure:

- `platform.workspace.WorkspaceContextControllerTest`: verify `GET /api/v1/workspace-context` returns 200 with correct JSON shape and camelCase field names
- `platform.navigation.NavigationControllerTest`: verify `GET /api/v1/nav/entries` returns 200 with 13 items in correct order
- `SdlcTowerApplicationTest`: verify application context loads

Use H2 for test profile (same as local).

## Maven Dependencies

Required in `pom.xml`:

```xml
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>3.4.4</version>
</parent>

<properties>
    <java.version>21</java.version>
</properties>

<dependencies>
    <!-- Web -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <!-- JPA -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-jpa</artifactId>
    </dependency>
    <!-- Actuator -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-actuator</artifactId>
    </dependency>
    <!-- H2 (local/test) -->
    <dependency>
        <groupId>com.h2database</groupId>
        <artifactId>h2</artifactId>
        <scope>runtime</scope>
    </dependency>
    <!-- Oracle (prod) -->
    <dependency>
        <groupId>com.oracle.database.jdbc</groupId>
        <artifactId>ojdbc11</artifactId>
        <scope>runtime</scope>
    </dependency>
    <!-- Test -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-test</artifactId>
        <scope>test</scope>
    </dependency>
</dependencies>
```

## Acceptance Criteria

- [ ] `./mvnw spring-boot:run -Dspring.profiles.active=local` starts without errors
- [ ] `GET http://localhost:8080/actuator/health` returns `{"status":"UP"}`
- [ ] `GET http://localhost:8080/api/v1/workspace-context` returns JSON matching the frontend contract
- [ ] `GET http://localhost:8080/api/v1/nav/entries` returns 13 items in correct order
- [ ] `GET http://localhost:8080/h2-console` is accessible in local profile
- [ ] `./mvnw test` passes all tests
- [ ] JSON field names are camelCase (matching frontend TypeScript types)
- [ ] Package structure follows `platform/` + `domain/` + `shared/` convention
- [ ] Oracle profile is configured but does not fail on startup when Oracle is unavailable

## What NOT To Do

- Do NOT modify any frontend code (the `frontend/` directory)
- Do NOT add authentication or security filters (deferred to later slice)
- Do NOT add complex business logic — this is a foundation slice
- Do NOT implement any `domain/` modules — only `platform/workspace` and `platform/navigation`
- Do NOT use Lombok
- Do NOT add Swagger/OpenAPI (can be added later)
- Do NOT use package-by-layer (flat `controller/`, `service/`, `model/`) — use package-by-feature as specified
