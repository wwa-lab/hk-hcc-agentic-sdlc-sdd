# Agentic SDLC Control Tower 产品说明书（PRD V0.9）

## 1. 产品基本信息

**产品名称**：Agentic SDLC Control Tower  
**中文定位**：AI 原生的软件交付指挥台  
**产品形态**：企业级 Web 平台，桌面端优先  
**适用对象**：研发团队、DevOps 团队、SRE/运维团队、平台治理团队、管理层  
**核心价值**：以可视化、可追踪、可治理的方式呈现 Agentic SDLC 在实际项目中的落地过程，降低理解门槛、使用门槛和推广门槛。

本文档用于定义产品方向、信息架构、核心对象、模块能力、治理模型与 MVP 边界，并作为后续设计稿、前端实现、后端接口和 Spec 拆解的上层输入。

---

## 2. 产品背景

当前企业在软件交付过程中普遍存在四类问题。

第一，研发与交付工具链分散。需求、设计、代码、测试、发布、事故处理分布在不同系统里，链路断裂，难以形成闭环。  
第二，AI 能力多数仍停留在“建议工具”层，无法体现 Agentic SDLC 的真实落地价值。  
第三，平台化治理不足。模板、配置、权限、审计分散，团队之间难以复用，推广成本高。  
第四，多团队协作语境复杂。企业往往按 **Application** 和 **SNOW Group** 组织团队，但现有系统很难同时满足“能力复用”和“数据隔离”。

因此，需要一个统一的控制塔平台，把 Agentic SDLC 的各阶段串联起来，不仅能展示价值，还能承载团队实际工作，并以平台能力支持长期扩展和治理。

---

## 3. 产品定位

本产品不是一个普通 DevOps Dashboard，也不是传统 ITSM 门户，而是一个 **AI-native 的软件交付控制塔**。

它同时承担三类职责：

### 3.1 展示层

向管理者、团队负责人和推广对象直观展示 Agentic SDLC 在真实项目中的落地效果，包括效率提升、质量改善、风险降低、AI 参与度和治理状态。

### 3.2 工作层

向 PM、架构师、开发、测试、DevOps、SRE 提供围绕 SDLC 阶段的工作台页面，用于查看状态、追踪链路、审阅 AI 结果、执行治理动作。

### 3.3 平台层

向平台管理员提供模板、配置、审计、权限、策略等底座能力，保证系统可复用、可隔离、可维护。

---

## 4. 产品目标

本产品的核心目标包括以下四项。

### 4.1 支持多团队协作与隔离

产品需适配企业按 **Application** 和 **SNOW Group** 划分团队的方式，支持多团队共用平台能力，同时保持数据、权限、审计和配置的隔离。

### 4.2 提供平台级治理能力

平台需内置基础治理中心，至少覆盖以下四项：  
**Template Management、Configuration Management、Audit Management、Access Management**。

### 4.3 围绕 SDLC 提供分阶段工作台

系统应提供 Requirement、Project、Design、Code & Build、Testing、Deployment、Incident 等页面，并支持页面级定制、模块级启停、布局级配置，降低功能间耦合，便于扩展与维护。

### 4.4 可视化展示 Agentic SDLC 的落地价值

产品必须清楚展示 AI 在哪些阶段参与、做了什么、带来了什么影响，以及哪些动作仍需要人工治理与审批，从而降低理解和推广门槛。

---

## 5. 非目标

第一阶段产品不以完全替代 Jira、GitLab、Jenkins、ServiceNow、监控平台为目标。  
更合理的定位是 **统一入口、统一追踪、统一治理、AI 增强、逐步替换**。

第一阶段也不追求构建一个只适合演示的大屏系统。  
产品必须能够兼顾展示价值与团队实际使用。

第一阶段不以“全自动无人治理”为目标。  
AI 可以成为主要操作者，但关键动作仍需纳入策略、审批和审计约束。

---

## 6. 目标用户

### 6.1 平台管理员

关注模板、配置、权限、审计、策略、集成治理。

### 6.2 Team Lead / Delivery Manager / Application Owner

关注团队状态、项目健康度、风险、资源、交付节奏、跨阶段链路和事故情况。

### 6.3 产品经理 / 业务分析师 / PMO

关注需求、优先级、Spec、计划、里程碑和交付进度。

### 6.4 架构师 / Tech Lead

关注架构设计、设计评审、代码与设计一致性、变更影响范围。

### 6.5 开发 / QA / DevOps / SRE

关注各阶段执行页面、AI 建议、执行动作、质量与发布状态、事故闭环。

### 6.6 管理层 / 推广对象 / 访客

关注整体价值展示、组织级指标、AI 落地情况和治理成熟度。

---

## 7. 核心设计原则

### 7.1 共享能力，不共享上下文

平台能力统一复用，但团队上下文独立隔离。  
共享的是页面模板、组件、流程、技能、指标模型；隔离的是数据、权限、配置、凭据和审计。

### 7.2 共享页面，不共享数据

不同团队可使用同一套页面模板，但同一页面在不同 Workspace 下展示不同数据、策略与默认视图。

### 7.3 共享模板，不共享权限

模板和流程可复用，但访问范围、审批链和执行权限必须受 Workspace 和角色控制。

### 7.4 AI 是参与者，不是旁观者

AI 不应只是聊天框或建议框，而应在产品中以“执行者、分析者、解释者、学习者”的身份显式出现。

### 7.5 人是治理者，不是默认执行者

在 AI 原生场景中，人工主要承担审批、覆盖、风险确认、策略治理和审计责任。

### 7.6 Spec 是执行中枢

产品遵循 **Spec Driven Development** 思路，Spec 是从需求到实现、从设计到 review 的核心契约对象，不应只是附属文档。

### 7.7 全链路可追踪

Requirement、User Story、Spec、Architecture、Design、Code、Test、Deploy、Incident、Learning 之间必须能够形成链路追踪。

### 7.8 配置驱动优先

页面布局、字段、表单、指标、流程、策略、AI 面板、审批规则尽量通过配置实现，而不是每次改代码。

---

## 8. 核心对象模型

### 8.1 组织与上下文模型

平台使用 Workspace 作为隔离与承载边界。

推荐 V1 定义为：

**1 个 Workspace = 1 个 Application + 1 个 Primary SNOW Group**

其上层为 Tenant / Organization，Workspace 下包含 Project 与 Environment。

对于没有 SNOW Group 管理方式的组织，V1 可采用兼容模式：

**1 个 Workspace = 1 个 Application，Primary SNOW Group 字段可为空。**

此时仍保留 SNOW Group 作为可选上下文字段，以便后续接入更细粒度的团队映射。

### 8.2 关键业务对象

平台需要管理以下核心对象：

- Requirement
- User Story
- Spec
- Architecture Artifact
- Design Artifact
- Task
- Code Change / Build
- Test Plan / Test Run
- Deployment / Release
- Incident
- Learning Artifact
- Skill
- Skill Execution
- Policy
- Template

### 8.3 上下文要求

所有业务页面都必须显式展示当前上下文，至少包含：

**Workspace、Application、SNOW Group、Project、Environment**

该上下文不仅用于展示，也用于数据隔离、权限控制、审计记录和配置继承。

---

## 9. 多团队与隔离模型

### 9.1 平台复用层

平台层统一提供页面模板、组件库、流程模板、策略、AI 技能、指标定义、集成适配器和治理能力。

### 9.2 Workspace 隔离层

每个 Workspace 拥有自己的：

- 业务数据空间
- 配置命名空间
- 权限范围
- 审计空间
- 集成凭据
- AI 运行历史

### 9.3 配置继承模型

系统采用四层继承机制：

**Platform Default → Application Default → SNOW Group Override → Project Override**

同一个页面模板可由平台定义默认行为，由 Application 和 SNOW Group 覆盖局部配置，再由项目级做细化。

### 9.4 跨团队可见性

默认不共享明细数据。  
跨团队场景下优先提供 **聚合视图** 或 **关联视图**，而不是复制一份对象数据。

例如，一个 Incident 可以有 `owner_workspace` 和 `related_workspaces`，主属团队拥有完整控制权，相关团队仅获得关联可见性。

### 9.5 隔离策略

V1 采用强逻辑隔离；高敏感团队可在后续版本支持更强的物理隔离能力。

---

## 10. 信息架构

系统一级信息架构建议如下：

- Dashboard / Control Tower
- Team Space
- Project Space
- Requirement Management
- Project Management
- Design Management
- Code & Build Management
- Testing Management
- Deployment Management
- Incident Management
- AI Center
- Report Center
- Platform Center

其中，Spec 作为一等对象，不应被隐藏。  
V1 可以将 Spec 作为 Requirement 页面中的核心标签页和链路对象，后续可独立升级为专门的 Spec Management 能力。

---

## 11. 页面与模块定义

### 11.1 Dashboard / Control Tower

首页是产品展示与运营指挥的主入口。  
它应同时服务管理者、团队负责人和日常用户。

首页需要展示组织级、团队级、项目级的状态汇总，包括：

- 交付节奏
- 需求流转
- 测试与质量
- 发布与事故
- AI 参与度
- 治理状态
- 最近关键动作
- 价值证明故事线

首页不应只是图表集合，而要体现一条完整的软件交付故事线。

当首页展示 SDLC 主链路时，应使用完整的 11 节点链路：

**Requirement → User Story → Spec → Architecture → Design → Tasks → Code → Test → Deploy → Incident → Learning**

其中 Spec 必须被显式突出为执行中枢。

Dashboard 侧重实时运营态势、异常信号和跨阶段故事线；  
Report Center 侧重历史统计、筛选、导出和面向利益相关方的结构化沟通。

### 11.2 Team Space

用于展示团队级视图。  
重点承载 Workspace、成员、团队默认模板、团队指标、团队风险、团队项目分布等内容。

该页面是多团队隔离模型在产品中的主要入口之一，也是 Dashboard 与 Project Space 之间的上下文承接层。

V1 中，Team Space 应被定义为“单一 Workspace 的运营主页”，用于回答以下问题：

- 当前团队是谁，负责哪些 Application / SNOW Group / Project 边界
- 当前团队的默认治理方式、模板和 AI Operating Model 是什么
- 当前团队当前最值得关注的项目、风险、阻塞和审批积压是什么
- 当前团队在 Requirement → User Story → Spec → Delivery → Incident 主链上的健康度如何

Team Space 的信息边界应明确如下：

- Team Space 中的团队级数据，应以 Workspace 为汇总边界
- Project 数据在该页中以“子集聚合”形式出现，用于解释团队现状，而不是替代 Project Space 的执行明细
- Template、Policy、Workflow、AI Default 等对象在该页中以“继承 / 覆盖 / 生效状态”视角展示，而不是进入 Platform Center 级别的底层配置编辑
- 当页面展示团队级 SDLC 主链路时，应沿用统一的 11 节点 vocabulary，并继续显式突出 Spec 这一执行中枢

V1 Team Space 至少包含以下模块：

- Workspace Summary：展示 Workspace 基本信息、所属 Application、Primary SNOW Group、活跃项目数、环境数、当前健康状态和核心责任边界
- Team Operating Model：展示该 Workspace 当前生效的 Operating Mode、默认审批模式、AI Autonomy Level、值班策略和责任人
- Member & Role Matrix：展示成员、角色、值班责任、关键权限、最近活跃度和关键岗位覆盖情况
- Team Default Templates：展示该 Workspace 继承或覆盖的模板、策略、流程、Skill 包和 AI 默认配置
- Requirement & Spec Pipeline：展示团队当前需求流入、User Story 拆解、Spec 生成 / 审阅 / 阻塞状态，以及从 Team Space 进入 Requirement Management 的入口
- Team Metrics：展示交付效率、质量、稳定性、治理成熟度、AI 参与度和关键趋势变化
- Team Risk Radar：展示当前高风险项目、依赖阻塞、审批积压、配置漂移、事故热点和风险升级入口
- Project Distribution：展示该 Workspace 下项目分布、状态分层、关键项目卡片和进入 Project Space 的导航入口

该页面至少应支持以下动作：

- 切换或确认当前 Workspace 上下文
- 查看成员与角色分工，并识别值班与责任覆盖空洞
- 查看默认模板、策略和 AI 配置的继承情况，以及例外覆盖点
- 从团队级需求 / Spec 信号进入对应 Requirement Management 或相关页面
- 进入高风险项目、重点项目或异常项目的 Project Space
- 查看团队级 AI 建议、治理提示、审批积压和风险处置入口

它与其他关键页面的区别在于：  
Dashboard 面向跨团队、跨项目的全局运营视角；  
Team Space 面向单个 Workspace 的负责人和成员，是带有上下文边界和可执行动作的团队运营页；  
Project Space 面向单项目执行态，强调里程碑、依赖和交付推进；  
Platform Center 面向平台级模板、策略和治理能力的设计与维护；  
Report Center 面向历史分析、复盘、导出和结构化沟通。

### 11.3 Project Space

用于展示单项目级总览，包括里程碑、状态、角色、依赖、版本、环境、风险和相关页面入口。  
Project Space 是用户从组织视角切换到执行视角的主要桥梁。

### 11.4 Requirement Management

用于管理需求、需求分类、优先级、业务背景、接受标准和需求链路。  
在 Spec Driven Development 体系下，该页面应支持从 Requirement 派生 User Story 与 Spec，并显式展示这些对象之间的关系。

核心能力包括需求列表、看板、优先级矩阵、需求拆解、Spec 生成入口、链路追踪和 AI 辅助分析。

### 11.5 Project Management

用于管理计划、里程碑、容量、资源、风险、依赖和交付进度。  
该页面偏项目运营与执行总览，重点不在文档，而在节奏控制与交付透明度。

### 11.6 Design Management

用于管理 Architecture 和 Design 资产，包括设计文档、评审记录、接口设计、影响分析和设计与 Spec 的对应关系。  
该页面应支持从上游 Spec 追踪到设计输出，并支持设计与实现的一致性校验。

### 11.7 Code & Build Management

用于展示代码变更、分支策略、Merge / Review、构建流水线、质量门禁和 AI 代码辅助结果。  
页面需清楚体现代码与 Task、Spec、Design、Build 的关联链路。

### 11.8 Testing Management

用于管理测试计划、用例、执行结果、缺陷、覆盖率和风险建议。  
该页面除了展示测试结果，更要体现“测试是否覆盖了 Spec 和设计中的关键约束”。

### 11.9 Deployment Management

用于管理版本、环境、发布窗口、审批、灰度、回滚和发布风险。  
重点不只是展示“发了什么”，而是解释“为什么能发、谁批准、AI 做了什么、风险如何控制”。

### 11.10 Incident Management

这是产品差异化最强的页面之一。  
它不是传统的 production incident ticket 页，而是一个 **AI-native 的运行指挥中心**。

传统事故系统的核心流程通常是：  
告警 → 人发现 → 人分析 → 人定位 → 人修复 → 人复盘。

本产品的 Incident 页应体现的流程是：  
**Signal → AI Detection → AI Diagnosis → AI Action → Human Governance → Resolution → Learning**

换句话说，AI 是事故处理的主要执行者，人主要承担审批、覆盖、策略治理和风险确认职责。

核心页面模块包括：

- Incident Header
- AI/Skill Execution Timeline
- AI Diagnosis & Reasoning
- AI Actions Taken
- Human Governance
- Related SDLC Chain
- AI Learning & Prevention
- AI Command Panel

Incident 页面必须显式表达以下信息：

- 事故由谁处理：AI / Human / Hybrid
- 当前自治级别：Autonomy Level
- 当前控制模式：Auto / Approval / Manual
- AI 已经做了哪些动作
- 哪些动作仍待人工审批
- 事故与 Requirement / Spec / Design / Code / Test / Deploy 的关系
- AI 学到了什么、如何防止再次发生

### 11.11 AI Center

用于展示 AI 能力、Agent、Skill、Prompt/Policy、运行历史、建议采纳率、技能执行状态等内容。  
AI Center 是“把 AI 作为平台能力”而不是“页面附属组件”的主要体现。

AI Center 与各业务页右侧 AI Command Panel 的关系应明确为：

- AI Center：全局 AI 能力管理、策略配置、技能目录、运行历史和效果评估入口
- AI Command Panel：当前页面上下文内的 AI 交互与执行投影，用于摘要、推理、建议、证据和动作触发

两者共享同一套 Skill Execution、Policy、Evidence 与 Audit 模型，但承担的交互层级不同。

### 11.12 Report Center

用于提供组织级、团队级和项目级报表，包括效率、质量、稳定性、治理、AI 贡献度等。  
该页面同时也是推广 Agentic SDLC 的重要载体。

Report Center 应与 Dashboard 明确区分：

- Dashboard：实时、运营中、偏指挥和故事线表达
- Report Center：历史、可筛选、可导出、可复用，用于复盘、汇报和对外沟通

### 11.13 Platform Center

用于承载平台底座能力，包括模板、配置、审计、权限、策略、技能注册、集成适配等。  
Platform Center 是系统可复用与可维护的关键基础。

---

## 12. 平台基础能力定义

### 12.1 Template Management

用于沉淀页面模板、流程模板、项目模板、策略模板、指标模板和 AI 模板。  
模板必须支持继承、版本与覆盖关系。

### 12.2 Configuration Management

用于管理页面配置、字段配置、组件编排、流程规则、视图规则、通知规则和 AI 配置。  
建议支持团队和项目级覆盖。

### 12.3 Audit Management

用于记录关键操作、配置变更、权限变更、审批记录、AI 动作、Skill 执行和策略命中情况。  
在 AI 原生系统中，审计不是附加能力，而是信任基础。

### 12.4 Access Management

支持以 RBAC 为主、ABAC 为辅的访问控制。  
权限范围至少应支持平台级、Application 级、Workspace 级、Project 级和页面/动作级。

### 12.5 Policy & Governance

虽然不在最初四项基础能力清单里，但在 AI-native 场景中必须作为一等能力存在。  
平台需要支持动作策略、审批规则、自治级别配置、风险阈值和例外处理。

### 12.6 Integration Framework

产品不替代 Jira、GitLab、Jenkins、ServiceNow 等系统，因此必须提供明确的集成框架。

集成层建议采用 **Adapter Model**：

- 平台定义统一的领域对象与事件模型
- 外部系统通过 Adapter 映射为 Requirement、Code Change、Build、Deployment、Incident 等平台对象
- Adapter 应支持 API 拉取、Webhook 推送、事件订阅和按需同步等模式

V1 建议优先覆盖以下 MVP 集成：

- Jira：需求、需求状态与优先级同步
- GitLab：代码变更、Merge Request、分支和 Review 状态同步
- Jenkins：构建、流水线、质量门禁和发布前信号同步
- ServiceNow：事故、变更、值班和审批上下文同步

集成凭据必须按 Workspace 隔离管理：

- 每个 Workspace 拥有独立的连接配置、凭据命名空间、权限范围和审计记录
- 平台管理员可以定义连接模板，但不能绕过 Workspace 的授权边界读取业务明细

数据流策略建议区分为两类：

- Sync / Pull：用于按需读取详情页数据、补拉历史记录和执行人工确认
- Async / Push：用于状态变更、流水线事件、事故告警和 AI 触发的近实时同步

V1 优先保证链路完整和审计可追溯，不要求所有系统都做到强一致实时镜像。

---

## 13. Spec Driven Development 设计

本产品遵循 **Spec Driven Development** 思路。  
Spec 不是文档附件，而是开发与交付过程中的核心契约对象。

### 13.1 主链路定义

系统应显式支持如下主链路：

**Requirement → User Story → Spec → Architecture → Design → Tasks → Code → Test → Deploy → Incident → Learning**

所有展示主链路的总览型页面，默认应渲染完整 11 节点链路，并突出 Spec。  
对于 Incident 等需要压缩展示的页面，可以折叠部分节点，但必须：

- 显示 Spec 节点
- 明确哪些节点被折叠
- 保持用户可追溯回完整链路

### 13.2 Spec 的产品地位

Spec 必须支持：

- 来源追踪
- 版本管理
- 与设计的映射
- 与任务的映射
- 与代码和测试的映射
- 与 Incident 的反向追踪

### 13.3 面向执行的 Spec

Spec 不应只是自然语言文档。  
平台需要逐步支持结构化 Contract，例如状态定义、对象模型、接口契约、权限规则和异常流程。

---

## 14. Skill 体系与执行模型

当前产品不仅展示 SDLC 过程，还应支持将 SDLC 与运行活动建模为可执行 Skill。

现有技能流水线可参考以下类型：

- req-to-user-story
- user-story-to-spec
- spec-to-architecture
- architecture-to-design
- design-to-tasks
- tasks-to-code
- tasks-to-implementation
- review-code-against-design
- review-doc-quality

运行阶段可继续扩展：

- incident-detection
- incident-correlation
- incident-diagnosis
- incident-remediation
- incident-learning

产品层面的要求是：

**所有关键活动都可以被登记为 Skill，并具有执行记录、步骤、状态、触发来源、输入输出、证据和审计轨迹。**

这样，Dashboard、Deployment、Incident、Report 等页面，就不仅是在展示状态，而是在可视化 Skill Execution。

---

## 15. 前端体验原则

产品前端设计需遵循以下原则：

### 15.1 统一壳层

所有页面共享统一导航、顶部上下文栏、搜索、通知、审计入口和 AI 面板。

### 15.2 固定上下文栏

所有业务页面都应在顶部明确显示：

**Workspace / Application / SNOW Group / Project / Environment**

### 15.3 卡片化与模块化

页面以 Widget/Card 为主要承载方式，支持按角色和场景进行组合。

### 15.4 右侧 AI Command Panel

AI 不应只是一个聊天抽屉。  
右侧面板应承担摘要、推理、建议、执行状态、模式切换和证据展示的角色。

该面板是 AI Center 的上下文投影，而不是独立的数据孤岛。  
它应复用全局 AI 能力、Skill Execution、Policy 与 Evidence 模型，但输出内容需严格受当前页面上下文约束。

### 15.5 高密度、企业级、控制塔风格

整体风格应接近企业级 SaaS 工作台，而非营销页面或传统工单系统。

---

## 16. 权限、审计与合规

### 16.1 角色建议

建议至少支持以下角色层次：

- Platform Admin
- Application Owner / Viewer
- Workspace Admin / Member / Viewer
- Project Admin / Contributor / Reader
- Auditor
- Visitor / Demo Viewer

### 16.2 审计要求

所有关键动作需要留下审计记录，尤其包括：

- 配置变更
- 权限变更
- AI 建议与执行
- 审批与拒绝
- Skill 触发与执行
- 发布与回滚
- 事故处置与复盘

### 16.3 合规要求

系统必须支持组织级审计追踪和关键策略检查，以满足企业对可控性、可解释性和责任边界的要求。

---

## 17. 核心指标体系

产品建议围绕五类指标建立报表与首页展示能力。

### 17.1 交付效率

需求周期、Lead Time、部署频率、迭代完成率、瓶颈阶段。

### 17.2 质量与测试

构建成功率、测试通过率、缺陷密度、Spec 覆盖情况、设计与代码一致性。

### 17.3 稳定性

故障次数、变更失败率、回滚率、MTTR、恢复方式。

### 17.4 治理成熟度

模板复用率、配置漂移情况、审计覆盖率、权限合规率、策略命中率。

### 17.5 AI 价值

AI 使用率、自治级别、建议采纳率、自动执行成功率、AI 节省工时、AI 参与阶段覆盖率。

### 17.6 NFR 目标定义方式

本 PRD 先定义指标类别，不在此阶段强行给出最终数值目标。  
可度量的非功能目标，例如 p95 延迟、可用性、审计保留期、恢复时间和数据同步时效，应在后续 `spec.md` 中明确。

---

## 18. MVP 范围

V1 建议聚焦“可展示、可联通、可扩展、可治理”。

### 18.1 必须具备

- Dashboard / Control Tower
- Team Space / Project Space
- Requirement Management（含 Spec 能力）
- Code & Build Management
- Testing Management
- Deployment Management
- Incident Management（AI-native）
- AI Center（轻量版）
- Platform Center 的四项基础能力
- 统一上下文栏、权限、审计、AI 面板
- 基础链路追踪能力

### 18.2 可轻量化上线

- Design Management
- Project Management
- 深度报表和高级策略
- 高级页面编排器

### 18.3 明确不作为 V1 目标

- 完整替代所有现有外部工具
- 大规模物理隔离
- 无审批的全自动高风险动作
- 复杂自定义低代码平台

---

## 19. 关键成功标准

产品上线后应重点验证以下结果：

第一，用户能否快速理解 Agentic SDLC 的全链路。  
第二，团队能否在同一平台中看到从 Requirement 到 Incident 的闭环。  
第三，AI 是否从“建议工具”升级为“可治理的执行主体”。  
第四，平台是否在支持多团队复用的同时保持了清晰的数据与权限隔离。  
第五，模板、配置、权限、审计能力是否足以支撑后续扩展。

---

## 20. 产品一句话定义

**Agentic SDLC Control Tower 是一个以 Workspace 为隔离边界、以 Spec 为执行中枢、以 Skill 为行为单元、以 AI 为主要操作者、以人类治理为安全边界的 AI 原生软件交付控制塔。**

---

## 21. 附：产品核心判断标准

当用户打开这个系统时，应该能一眼感受到以下四件事：

**第一，这不是普通的 DevOps Dashboard，而是一个正在运行的软件交付控制塔。**  
**第二，AI 不是在旁边提建议，而是在系统里真正执行工作。**  
**第三，所有动作都能追溯到 Requirement / Spec / Design / Code / Test / Deploy / Incident。**  
**第四，平台能力可复用，但团队上下文、权限和数据是隔离的。**
