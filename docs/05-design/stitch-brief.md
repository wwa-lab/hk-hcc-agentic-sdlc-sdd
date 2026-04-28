# Stitch Brief: Agentic SDLC Control Tower Frontend Design

## Purpose

This document is a ready-to-use brief for generating the first-round frontend design in Stitch.
It is derived from:

- `docs/01-requirements/agentic_sdlc_control_tower_prd_v0.9.md`
- `docs/05-design/README.md`

The goal is to generate a desktop-first enterprise SaaS UI direction for an AI-native SDLC control tower, with outputs that can be translated into a Vue 3 application.

## Recommended Usage

Do not ask Stitch to design the entire product in one pass.

Use three rounds:

1. Round 1: product shell + representative pages
2. Round 2: execution pages
3. Round 3: operating and governance refinement

For Round 1, focus on:

- Dashboard / Control Tower
- Project Space
- Incident Management
- Platform Center

These pages best express the product's control tower, AI-native, and governance characteristics.

## Product Context

- Product name: Agentic SDLC Control Tower
- Product positioning: AI-native software delivery control tower
- Product form: enterprise web platform, desktop first
- Primary users: Platform Admin, Team Lead, Delivery Manager, PM, Architect, Developer, QA, DevOps, SRE, leadership viewers

## Core Product Intent

This is not:

- a marketing website
- a demo-only big screen
- a traditional ticketing console
- a generic DevOps dashboard

This should feel like:

- a high-density enterprise workbench
- an operational control surface
- an AI-native execution and governance platform
- a system where AI is visible as an actor, not just a sidebar chatbot

## Global Design Constraints

The design must satisfy all of the following:

### 1. Unified Product Shell

All pages share:

- left navigation
- top context bar
- global search
- notifications
- audit entry
- persistent right-side AI Command Panel

### 2. Fixed Context Bar

Every business page must explicitly show:

- Workspace
- Application
- SNOW Group
- Project
- Environment

This context is part of both product comprehension and data isolation.

### 3. Card-Based Modular Layout

Pages should be composed from reusable widgets/cards/modules that can be rearranged by role and scenario.

### 4. AI Command Panel Is a Core Product Surface

The right-side panel is not just a chat drawer.
It should support:

- summary
- reasoning
- recommendations
- execution status
- control mode switching
- evidence display

### 5. Spec-Driven Delivery Must Be Visible

The UI should make the SDLC chain legible:

`Requirement -> User Story -> Spec -> Architecture -> Design -> Tasks -> Code -> Test -> Deploy -> Incident -> Learning`

Spec is a first-class object and should not be visually hidden.

### 6. Enterprise Control Tower Visual Direction

The design language should be:

- dense but readable
- operational rather than promotional
- structured and trustworthy
- suitable for enterprise users spending long sessions in the tool

## Round 1 Design Scope

Stitch should design the following.

### A. Global App Shell

Include:

- left primary navigation
- top context bar
- page title area
- global action area
- persistent right AI Command Panel
- content area that supports dashboard and detail layouts

Expected deliverables:

- shell structure
- navigation grouping
- spacing rhythm
- layout behavior for wide desktop screens

### B. Dashboard / Control Tower

Purpose:
Main landing page for product value demonstration and operational overview.

Must show:

- delivery rhythm
- requirement flow
- testing and quality
- release and incident status
- AI participation
- governance status
- recent key actions
- value story / proof narrative

Important constraint:
This page should not feel like a loose collection of charts. It should tell an end-to-end delivery story.

### C. Project Space

Purpose:
Bridge organization-level visibility and execution-level work.

Must show:

- milestones
- project status
- roles
- dependencies
- versions
- environments
- risks
- entry points to related SDLC pages

### D. Incident Management

Purpose:
AI-native operations command center.

Must include these modules:

- Incident Header
- AI / Skill Execution Timeline
- AI Diagnosis and Reasoning
- AI Actions Taken
- Human Governance
- Related SDLC Chain
- AI Learning and Prevention
- AI Command Panel

Must make these states explicit:

- incident handled by AI / Human / Hybrid
- autonomy level
- control mode: Auto / Approval / Manual
- actions already executed by AI
- actions pending human approval
- relation to Requirement / Spec / Design / Code / Test / Deploy

This page should feel materially different from a traditional incident ticket screen.

### E. Platform Center

Purpose:
Governance and platform capability hub.

Must cover:

- Template Management
- Configuration Management
- Audit Management
- Access Management
- Policy and Governance

The design should express:

- shared platform capabilities
- isolated workspace context
- policy-aware actions
- trust, traceability, and control

## Vue 3 Implementation Constraints

The design should be easy to translate into a Vue 3 codebase.

Assume:

- Vue 3
- Composition API
- `<script setup>`
- Vue Router
- Pinia

Stitch should think in Vue application structure, not React terminology.

Please avoid:

- React hook naming
- React-only state assumptions
- component guidance that depends on React patterns

Please include Vue-oriented implementation guidance for:

- route structure
- layout hierarchy
- page vs global state boundaries
- reusable module boundaries
- composables that would likely exist

Suggested examples:

- `AppShell.vue`
- `TopContextBar.vue`
- `AiCommandPanel.vue`
- `DashboardView.vue`
- `ProjectSpaceView.vue`
- `IncidentDetailView.vue`
- `PlatformCenterView.vue`
- `useWorkspaceContext()`
- `useAiCommandPanel()`
- `useAuditFilters()`

## Required Output From Stitch

Please generate:

1. A shared visual direction
2. A wireframe pass for the app shell and the four Round 1 pages
3. A higher-fidelity direction for the same scope
4. Information hierarchy for each page
5. Core cards/modules/components per page
6. Key interactions and states:
   - normal
   - loading
   - empty
   - error
   - approval pending
7. Suggested Vue 3 component decomposition
8. Suggested route map
9. Suggested state ownership boundaries

## Copy-Paste Prompt For Stitch

### Chinese Prompt

```text
请为以下产品生成一套桌面端优先的企业级 SaaS 前端设计。

产品名称：Agentic SDLC Control Tower
产品定位：AI 原生的软件交付指挥台
主要用户：Platform Admin、Team Lead、Delivery Manager、PM、架构师、开发、QA、DevOps、SRE、管理层查看者

这不是：
- 营销官网
- 大屏演示系统
- 通用 DevOps Dashboard
- 传统工单或 Ticket 系统

它应该更像：
- 高密度企业级工作台
- 可运营的控制面
- AI 原生的执行与治理平台

全局设计约束：
- 采用统一产品壳层
- 左侧一级导航
- 顶部固定上下文栏，必须展示：Workspace / Application / SNOW Group / Project / Environment
- 顶部提供全局搜索、通知、审计入口
- 右侧固定 AI Command Panel
- 页面以卡片化、模块化方式组织
- Spec 是一等对象，界面中应能看出完整 SDLC 链路：
  Requirement -> User Story -> Spec -> Architecture -> Design -> Tasks -> Code -> Test -> Deploy -> Incident -> Learning
- AI 必须以“参与者/执行者”的身份出现，而不只是聊天助手
- 整体视觉方向应为：高密度、企业级、可信赖、控制塔风格

本轮只设计以下范围：
1. Global App Shell
2. Dashboard / Control Tower
3. Project Space
4. Incident Management
5. Platform Center

页面要求：

Dashboard / Control Tower：
- 展示交付节奏、需求流转、测试与质量、发布与事故、AI 参与度、治理状态、最近关键动作、价值证明故事线
- 页面不应只是图表拼贴，而要能讲清楚端到端交付故事

Project Space：
- 展示里程碑、项目状态、角色、依赖、版本、环境、风险，以及进入相关 SDLC 页面的入口
- 它应该承担从组织级视角切换到执行级视角的桥梁作用

Incident Management：
- 这是 AI-native 的运行指挥中心
- 必须包含：Incident Header、AI / Skill Execution Timeline、AI Diagnosis and Reasoning、AI Actions Taken、Human Governance、Related SDLC Chain、AI Learning and Prevention、AI Command Panel
- 必须清晰表达：AI / Human / Hybrid 处理模式、Autonomy Level、Auto / Approval / Manual 控制模式、AI 已执行动作、待人工审批动作，以及与 Requirement / Spec / Design / Code / Test / Deploy 的关联
- 视觉和交互上要明显区别于传统 incident ticket 页面

Platform Center：
- 覆盖 Template Management、Configuration Management、Audit Management、Access Management、Policy and Governance
- 要体现平台能力共享、团队上下文隔离、策略治理与可审计性

实现约束：
- 前端框架：Vue 3
- 使用 Composition API 与 <script setup>
- 请按 Vue 应用结构思考，而不是 React 术语和模式
- 默认工程形态：Vue Router + Pinia
- 输出应方便后续落地为 Vue SFC
- 避免使用 React hooks 命名和 React 特有状态组织方式

请同时提供面向 Vue 3 的实现建议：
- 建议的路由结构
- Layout 层级划分
- 全局状态 / 页面状态 / 组件局部状态的边界
- 可复用模块与组件拆分
- 可能需要的 composables

组件命名示例：
- AppShell.vue
- TopContextBar.vue
- AiCommandPanel.vue
- DashboardView.vue
- ProjectSpaceView.vue
- IncidentDetailView.vue
- PlatformCenterView.vue
- useWorkspaceContext()
- useAiCommandPanel()
- useAuditFilters()

请输出：
- 一套统一视觉方向
- 一版 wireframe
- 一版高保真方向
- 每个页面的信息层级
- 每个页面的核心模块 / 卡片 / 组件
- 关键状态：normal、loading、empty、error、approval pending
- Vue 3 组件拆分建议
- 路由结构建议
- 状态边界建议
```

### English Prompt

```text
Please generate a desktop-first enterprise SaaS frontend design for the following product.

Product name: Agentic SDLC Control Tower
Product positioning: AI-native software delivery control tower
Users: Platform Admin, Team Lead, Delivery Manager, PM, Architect, Developer, QA, DevOps, SRE, leadership viewers

This is not:
- a marketing website
- a big-screen demo
- a generic DevOps dashboard
- a traditional ticketing console

This should feel like:
- a high-density enterprise workbench
- an operational control surface
- an AI-native execution and governance platform

Global design constraints:
- Use a unified app shell
- Left-side primary navigation
- Fixed top context bar showing: Workspace / Application / SNOW Group / Project / Environment
- Global search, notifications, and audit entry in the top area
- Persistent right-side AI Command Panel
- Card-based modular layout
- Spec is a first-class object and the SDLC chain should be legible:
  Requirement -> User Story -> Spec -> Architecture -> Design -> Tasks -> Code -> Test -> Deploy -> Incident -> Learning
- AI must appear as an actor, not just a chatbot
- Visual style should be high-density, enterprise, trustworthy, and control-tower oriented

Round 1 design scope:
1. Global App Shell
2. Dashboard / Control Tower
3. Project Space
4. Incident Management
5. Platform Center

Page requirements:

Dashboard / Control Tower:
- Show delivery rhythm, requirement flow, testing and quality, release and incident status, AI participation, governance status, recent key actions, and a value story
- It should not feel like a random chart wall; it should tell a delivery story

Project Space:
- Show milestones, project status, roles, dependencies, versions, environments, risks, and navigation to related SDLC pages
- It should bridge organization-level visibility and execution-level work

Incident Management:
- This is an AI-native operations command center
- Include: Incident Header, AI / Skill Execution Timeline, AI Diagnosis and Reasoning, AI Actions Taken, Human Governance, Related SDLC Chain, AI Learning and Prevention, AI Command Panel
- Clearly show: AI / Human / Hybrid handling mode, Autonomy Level, Auto / Approval / Manual control mode, actions already executed, actions waiting for approval, and links to Requirement / Spec / Design / Code / Test / Deploy
- It should feel meaningfully different from a traditional incident ticket page

Platform Center:
- Cover Template Management, Configuration Management, Audit Management, Access Management, Policy and Governance
- Express shared platform capabilities with isolated workspace context

Implementation constraints:
- Frontend framework: Vue 3
- Use Composition API and <script setup>
- Think in Vue application structure, not React terminology
- Assume Vue Router + Pinia
- Output should be easy to translate into Vue SFCs
- Avoid React hooks naming and React-specific implementation patterns

Please also provide Vue-oriented implementation guidance:
- suggested route structure
- layout hierarchy
- global state vs page state vs local component state
- reusable modules/components
- likely composables

Suggested component examples:
- AppShell.vue
- TopContextBar.vue
- AiCommandPanel.vue
- DashboardView.vue
- ProjectSpaceView.vue
- IncidentDetailView.vue
- PlatformCenterView.vue
- useWorkspaceContext()
- useAiCommandPanel()
- useAuditFilters()

Please output:
- one unified visual direction
- one wireframe pass
- one higher-fidelity direction
- page information hierarchy
- core modules/cards/components
- key states: normal, loading, empty, error, approval pending
- Vue 3 component decomposition suggestions
- route map suggestions
- state ownership suggestions
```

## Suggested Next Step After Stitch

After receiving the first design output:

1. save the accepted design direction into `docs/05-design/design.md`
2. normalize page names, modules, and component vocabulary
3. convert approved design into `docs/06-tasks/tasks.md`
4. then start Vue 3 implementation page by page
