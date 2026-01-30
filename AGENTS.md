# AGENTS.md

## Collaboration Protocol (协作协议)

本文件旨在规范 AI Agent（Claude Code、Antigravity 等）在本项目中的代码与文档协作行为。

- **Core Language**: Output MUST be in **Chinese (Simplified)** unless serving code/technical constraints.
- **Tone**: Professional, precise, and evidence-based.

## Project Positioning (项目定位)

本项目定位于 **Agentic AI Cognizes（认知增强）** 领域的学术研究与工程实践，核心聚焦于 **Cognizes Engine（认知引擎）** 的基础设施构建。该引擎采用 **仿生架构（Bionic Architecture）** 设计，涵盖：

- **🫀 The Pulse（脉搏系统）**：会话状态管理与实时心跳 (Session/State)
- **🧠 The Hippocampus（仿生记忆）**：记忆整合、衰减与长短期记忆存储 (Memory Bank)
- **👁️ The Perception（神经感知）**：多模态内容摄取、混合检索与 RAG 管道 (RAG Pipeline/Hybrid Search)
- **🔮 The Realm of Mind（心智空间）**：Agent 运行时、工具注册与编排调度 (Runtime/Sandbox/Tools)
- **Infrastructure**：PostgreSQL All-in-One 三位一体存储与全链路可观测性 (O11y)

_注：业务应用层（Business Layer）详情请见 `docs/000-prd-architecture.md` 之「产品愿景 · 核心定位」。_

## Engineering Code of Conduct (工程行为准则)

**Core Philosophy**: **Entropy Reduction (熵减)**. 通过上下文锚定、复用驱动与标准化流水线，对抗软件系统的无序熵增。

### 道 (Mindset - 认知心法)

- **Context-Driven (上下文驱动)**: 上下文是第一性要素 (Context Quality First)。任何变更需建立在深度理解之上（CDD），拒绝基于关键字匹配的机械式修改。
- **Minimal Intervention (最小干预)**: 遵循奥卡姆剃刀与 YAGNI 原则，仅实施必要的变更，推崇演进式设计 (Evolutionary Design) 而非过度设计。
- **Evidence-Based (循证工程)**: 杜绝主观臆断，核心决策需以权威文献（IEEE 格式）为佐证，构建 Feedback Loops 以验证假设。
- **Systemic Integrity (系统完整性)**: 具备全局视角与二阶思维 (Second-Order Thinking)，评估变更对上下游依赖及整个生态（Engine, Adapter, Agent, UI）的“涟漪效应”，优先保障整体稳定性与逻辑自洽。

### 法 (Strategy - 架构原则)

- **Reuse-Driven (复用驱动)**: Composition over Construction。优先采用业界成熟方案与最佳实践；通过“拿来主义”站在巨人的肩膀上进行微创新。优先组合与集成，通过连接成熟组件构建系统，而非重复造轮子
- **Boundary Management (边界管理)**: 严控模块/Agent 间的职责边界与契约，确保高内聚低耦合，防范隐式依赖穿透。
- **Orthogonal Decomposition (正交分解)**: 坚持“正交地提取概念主体”。识别系统中独立变化的维度并进行解耦（如机制与策略分离），确保单一概念主体的变更具备局部性，避免逻辑纠缠。
- **Feedback Loops (反馈闭环)**：构建“设计-实现-验证”的完整闭环，确保每一项工程行动都能产生可观测的反馈信号（测试、日志、监控），以验证假设并指导迭代。
- **Evolutionary Design (演进式设计)**: 将系统视为有机体，通过将 AI 错误转化为经验约束 (Negative Prompts) 和持久化知识，实现系统的自我进化与熵减。
- **Second-Order Thinking (二阶思维)**：不只关注变更的直接结果，更要预测“结果的结果”（如引入缓存导致的陈旧数据、重试机制引发的雪崩），未雨绸缪防范隐性风险。
- **Single Source of Truth (单一事实源)**：严格维护唯一的权威定义源。引用时**必须**使用轻量级指针 (Link/ID) 而非数据副本 (Copy-Paste)，从根源消除断裂 (Split-Brain) 风险。

### 术 (Tactics - 执行规范)

- **Vibe Coding Pipeline**: 遵循 **Specification-Driven (规划驱动)** + **Context-Anchored (上下文锚定)** + **AI-Pair (AI 结对)** 模式，将开发固化为可审计的流水线，避免代码腐化为无法维护的“大泥球 (Big Ball of Mud)”。
- **Visual Documentation (图文并茂)**: 对于复杂逻辑，优先使用 Mermaid 图表（Sequence/Flowchart/Class）辅助说明，构建“图文并茂”的直观文档。
- **Direct Hyperlinking (直接跳转)**: 在文档中提及 Repo 内其他资源（文档/代码）时，**必须**构建可跳转的相对路径链接（如 `[Doc Name](./path.md)`），严禁使用“死文本”引用，以降低信息检索熵。
- **Operational Excellence (卓越运营)**:
  1. **Git Hygiene**: 如非显性要求，严禁调用 git commit；
  2. **Temp Management**: 临时产物（执行计划等）一律收敛至 `.temp/` 并及时清理；
  3. **Link Validity**: 确保所有引用的 URL 可访问且具备明确的上下文价值。

## Documentation Standards (文档规范)

### Mermaid Visualization Norms (Mermaid 可视化规范)

- **色彩语义与兼容性**：为图表节点配置具备语义辨识度的色彩，并确保在深色模式（Dark Mode）下具有极高的对比度与清晰度。
- **逻辑模块化解构**：针对业务跨度较大的架构流程，强制采用 `subgraph` 容器进行层级解构与边界划分，以增强图表的自解说（Self-explaining）能力。

### Reference Specifications (IEEE)

为保障工程决策的可追溯性与学术严谨性，核心引用需遵循 **IEEE 标准引用格式**。

> **模版准则**：[编号] 作者缩写. 姓, "文章标题," _刊名/会议名缩写 (斜体)_, 卷号, 期数, 页码, 年份.

```latex
[1] A. Author, B. Author, and C. Author, "Title of paper," *Abbrev. Title of Journal*, vol. X, no. Y, pp. XX–XX, Year.
```

**引用实践**

- **文内锚定**：采用标准上标链接形式：`描述内容<sup>[[1]](#ref1)</sup>`。
- **文献索引**：底层采用 HTML 锚点 `id` 实现跳转稳定性。

```latex
<a id="ref1"></a>[1] A. Vaswani et al., "Attention is all you need," Adv. Neural Inf. Process. Syst., vol. 30, pp. 5998–6008, 2017.
```

## Knowledge Map (知识索引)

- [🗺️ 产品需求与架构](docs/000-prd-architecture.md) - 产品需求与概要设计
- [📖 实施计划](docs/001-implementation-plan.md) - Agentic AI 学术研究与工程应用平台实施计划
- [✅ 任务清单](docs/002-task-checklist.md) - 任务执行清单

---

### Cognizes Engine

- [🗺️ Engine 路线图](docs/engine/000-roadmap.md) - 项目整体开发计划和进度
- [📖 Engine 任务清单](docs/engine/001-task-checklist.md) - 任务清单
- [🫀 脉搏系统](docs/engine/010-the-pulse.md) - 脉搏系统
- [🧠 仿生记忆](docs/engine/020-the-hippocampus.md) - 仿生记忆
- [👁️ 神经感知](docs/engine/030-the-perception.md) - 神经感知
- [🔮 心智空间](docs/engine/040-the-realm-of-mind.md) - 心智空间
- [🚀 集成验证](docs/engine/050-integrated-demo.md) - 集成验证
- [🧩 Engine 架构记录](docs/engine/060-agentic-ai-engine.md) - Roadmap & TCO

### Research

- [🤖 认知增强](docs/research/000-cognitive-enhancement.md) - 认知增强
- [📡 上下文工程](docs/research/010-context-engineering.md) - 上下文工程
- [🤖 Agent Runtime](docs/research/020-agent-runtime-frameworks.md) - Agent Runtime
- [📡 Vector Search Algorithm](docs/research/030-vector-search-algorithm.md) - Vector Search Algorithm
- [📡 Vector Databases Selection](docs/research/031-vector-databases-selection.md) - Vector Databases Selection
- [📡 Vector Databases](docs/research/032-vector-databases.md) - Vector Databases
- [📡 OceanBase](docs/research/033-oceanbase.md) - OceanBase 调研
- [📡 Knowledge Base](docs/research/034-knowledge-base.md) - RAG Pipeline & Hybrid Search
- [📡 Knowledge Base Platform](docs/research/035-knowledge-base-platform.md) - 腾讯 WeKnora 深度调研
- [📡 Cognee](docs/research/040-cognee.md) - Cognee
- [📡 Neo4j](docs/research/050-neo4j.md) - Neo4j
- [📡 Bettafish](docs/research/060-bettafish.md) - Bettafish
- [📡 AG-UI](docs/research/070-ag-ui.md) - AG-UI
- [📡 Agent Sandbox](docs/research/080-agent-sandbox.md) - Agent Sandbox
- [📡 Agent Evaluation](docs/research/090-agent-evaluation.md) - Agent Evaluation
- [📡 Agent Observability](docs/research/100-agent-obser.md) - Jaeger vs Langfuse
