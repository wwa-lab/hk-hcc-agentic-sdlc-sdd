# Shared App Shell Requirements

## Purpose

This document extracts the requirements relevant to the shared application shell
from the full PRD (V0.9). It defines **what** the shell must deliver, serving as
the upstream input for the spec, architecture, design, and task documents.

## Source

All requirements below are derived from:

- [agentic_sdlc_control_tower_prd_v0.9.md](agentic_sdlc_control_tower_prd_v0.9.md)

Section references use the format `PRD §N.N`.

---

## 1. Scope

The shared app shell is the outermost UI frame of the product. It covers:

- unified navigation
- top context bar
- global utility entry points
- right-side AI Command Panel region
- page title and action region
- route-to-page mapping

It does **not** cover page-specific business modules, backend data APIs beyond
workspace context and navigation, or final visual token finalization.

---

## 2. Product Context Requirements

### REQ-SHELL-01: Enterprise web platform, desktop-first

The product is an enterprise-grade web platform with a desktop-first layout.
Mobile layout is not required for V1.

> Source: PRD §1 — "企业级 Web 平台，桌面端优先"

### REQ-SHELL-02: Target users span all roles

The shell must serve platform admins, team leads, delivery managers, PMs,
architects, developers, QA, DevOps, SRE, management, and visitors. The shell
should not assume a single user type.

> Source: PRD §6

### REQ-SHELL-03: Product name

The operational product name is **SDLC Tower**. The formal long name is
**Agentic SDLC Control Tower**.

> Source: PRD §1

---

## 3. Navigation Requirements

### REQ-SHELL-10: 13-entry primary navigation

The shell must provide a stable primary navigation with the following 13 entries
in fixed order:

1. Dashboard
2. Team Space
3. Project Space
4. Requirement Management
5. Project Management
6. Design Management
7. Code & Build
8. Testing
9. Deployment
10. Incident Management
11. AI Center
12. Report Center
13. Platform Center

> Source: PRD §10

### REQ-SHELL-11: Navigation vocabulary from PRD

Navigation labels must use the vocabulary defined in PRD §10. Shortened forms
(e.g., "Code & Build" instead of "Code & Build Management") are acceptable
where consistent.

> Source: PRD §10, §15.1

### REQ-SHELL-12: Active navigation state

Exactly one navigation item must be visually active at a time, determined by
the current route. The active state must use a consistent visual pattern
across all pages.

> Source: PRD §15.1 — "所有页面共享统一导航"

### REQ-SHELL-13: Permission-aware navigation (deferred)

V1 renders the full 13-entry set for all users. The architecture must not
make it structurally difficult to add permission-based filtering in a later
version.

> Source: PRD §16.1 — role hierarchy implies future role-based menu filtering

### REQ-SHELL-14: Shell-owned navigation

Navigation items are shell-owned, not page-owned. Pages do not define or
modify the navigation structure.

> Source: PRD §15.1 — "所有页面共享统一导航"

---

## 4. Context Bar Requirements

### REQ-SHELL-20: Visible workspace context on every page

All business pages must display the current context at the top of the page.
The required fields are:

- **Workspace**
- **Application**
- **SNOW Group**
- **Project**
- **Environment**

> Source: PRD §8.3, §15.2 — "所有业务页面都应在顶部明确显示"

### REQ-SHELL-21: Fixed field order

Context fields must appear in a stable, fixed order across all pages.

> Source: PRD §15.2

### REQ-SHELL-22: Safe handling of optional fields

SNOW Group, Project, and Environment may be null or absent. Missing values
must render gracefully (e.g., placeholder text) without breaking layout.

> Source: PRD §8.1 — "Primary SNOW Group 字段可为空"

### REQ-SHELL-23: Context for isolation, not just display

The context bar is not decoration. The displayed context represents the data
isolation boundary and must be consistent with workspace isolation rules.

> Source: PRD §8.3 — "该上下文不仅用于展示，也用于数据隔离、权限控制、审计记录和配置继承"

---

## 5. Global Utility Requirements

### REQ-SHELL-30: Global search entry point

The shell must include a global search entry point, accessible from every page.

> Source: PRD §15.1 — "搜索"

### REQ-SHELL-31: Notifications entry point

The shell must include a notifications entry point, accessible from every page.

> Source: PRD §15.1 — "通知"

### REQ-SHELL-32: Audit / history entry point

The shell must include an audit or history entry point, accessible from every page.

> Source: PRD §15.1 — "审计入口"

### REQ-SHELL-33: Shell-level utilities

Search, notifications, and audit are shell-level controls, not page-specific
actions. They must appear in a consistent area of the shell.

> Source: PRD §15.1

---

## 6. AI Command Panel Requirements

### REQ-SHELL-40: Persistent right-side AI panel

The shell must render a persistent right-side AI Command Panel region in
desktop layouts. AI is not a chat drawer — it is a core operating surface.

> Source: PRD §15.4 — "AI 不应只是一个聊天抽屉"

### REQ-SHELL-41: Panel content zones

The AI panel must support the following content zones:

- Summary
- Reasoning
- Evidence
- Action or command input area

> Source: PRD §15.4 — "摘要、推理、建议、执行状态、模式切换和证据展示"

### REQ-SHELL-42: Panel is context-scoped

The panel is the current-page projection of AI state. Its content must be
constrained by the current page context, not global AI state.

> Source: PRD §15.4 — "输出内容需严格受当前页面上下文约束"

### REQ-SHELL-43: AI Center is the global management page

The AI Command Panel is not a duplicate of AI Center. AI Center is the global
management page; the panel is its per-page projection.

> Source: PRD §15.4

### REQ-SHELL-44: Shell owns the container

The shell owns the panel container. Pages provide panel content or
content configuration. The panel structure is not redefined per page.

> Source: PRD §15.4

---

## 7. Page Integration Requirements

### REQ-SHELL-50: Unified shell for all pages

All pages must render through one shared shell abstraction. Pages must not
duplicate layout markup.

> Source: PRD §15.1

### REQ-SHELL-51: Page title and action region

Each page must be able to provide a title, optional subtitle, and optional
page-level actions to the shell.

> Source: PRD §15.3 — "页面以 Widget/Card 为主要承载方式"

### REQ-SHELL-52: Round 1 representative pages

Four pages must be implemented as representative Round 1 pages:

- Dashboard / Control Tower
- Project Space
- Incident Management
- Platform Center

The remaining 9 entries point to a shared placeholder.

> Source: PRD §11.1, §11.3, §11.10, §11.13

---

## 8. Visual and Experience Requirements

### REQ-SHELL-60: High-density, control-tower style

The visual style must be high-density and enterprise-grade, approaching a
"control-tower" or "command center" aesthetic. It must not resemble a
marketing page or a traditional ticketing system.

> Source: PRD §15.5 — "高密度、企业级、控制塔风格"

### REQ-SHELL-61: Card-based, modular layout

Page content should use a card-based, modular layout that supports
role-based and scenario-based composition.

> Source: PRD §15.3

### REQ-SHELL-62: Configuration-driven (future)

Page layout, fields, and AI panel behavior should eventually be
configuration-driven. V1 shell must not make this structurally difficult.

> Source: PRD §7.8

---

## 9. Error and Resilience Requirements

### REQ-SHELL-70: Shell survives page errors

If page content fails to load or encounters an error, the surrounding shell
(navigation, context bar, AI panel) must continue to render.

### REQ-SHELL-71: Graceful degradation for context

If the workspace context cannot be loaded, the shell must degrade gracefully
with placeholder or loading states rather than breaking layout.

### REQ-SHELL-72: Graceful degradation for AI panel

If the AI panel has no page-scoped content, it must render with placeholder
or disabled state rather than disappearing.

---

## 10. Non-Functional Requirements

### REQ-SHELL-80: Vue 3 with Composition API

The frontend must use Vue 3 with Composition API and `<script setup>`.

### REQ-SHELL-81: No backend dependency for shell rendering

The shell foundation slice must be able to render with static or mocked data.
Backend integration is added in a separate phase.

### REQ-SHELL-82: Build must succeed

`npm run dev` and `npm run build` must both succeed for the frontend.

---

## 11. Out of Scope

The following are explicitly out of scope for the shell slice:

- Page-specific business widgets and data modules
- Live data integration (Phase B handles backend connection)
- Permission-driven menu filtering
- Mobile layout optimization
- Full AI action workflow behavior
- Design token finalization
- Responsive mobile behavior

---

## 12. Traceability

| Requirement | PRD Section |
|-------------|-------------|
| REQ-SHELL-01 | §1 |
| REQ-SHELL-02 | §6 |
| REQ-SHELL-03 | §1 |
| REQ-SHELL-10 | §10 |
| REQ-SHELL-11 | §10, §15.1 |
| REQ-SHELL-12 | §15.1 |
| REQ-SHELL-13 | §16.1 |
| REQ-SHELL-14 | §15.1 |
| REQ-SHELL-20 | §8.3, §15.2 |
| REQ-SHELL-21 | §15.2 |
| REQ-SHELL-22 | §8.1 |
| REQ-SHELL-23 | §8.3 |
| REQ-SHELL-30 | §15.1 |
| REQ-SHELL-31 | §15.1 |
| REQ-SHELL-32 | §15.1 |
| REQ-SHELL-33 | §15.1 |
| REQ-SHELL-40 | §15.4 |
| REQ-SHELL-41 | §15.4 |
| REQ-SHELL-42 | §15.4 |
| REQ-SHELL-43 | §15.4 |
| REQ-SHELL-44 | §15.4 |
| REQ-SHELL-50 | §15.1 |
| REQ-SHELL-51 | §15.3 |
| REQ-SHELL-52 | §11 |
| REQ-SHELL-60 | §15.5 |
| REQ-SHELL-61 | §15.3 |
| REQ-SHELL-62 | §7.8 |
| REQ-SHELL-70 | — |
| REQ-SHELL-71 | — |
| REQ-SHELL-72 | — |
| REQ-SHELL-80 | — |
| REQ-SHELL-81 | — |
| REQ-SHELL-82 | — |
