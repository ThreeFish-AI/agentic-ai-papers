---
id: the-realm-of-mind-implementation
sidebar_position: 4.0
title: Phase 4：The Realm of Mind 验证实施方案
last_update:
  author: Aurelius Huang
  created_at: 2026-01-09
  updated_at: 2026-01-14
  version: 1.1
  status: Reviewed
tags:
  - The Realm of Mind
  - Agent Runtime
  - ADK Adapter
  - Implementation Plan
  - PostgreSQL
  - Glass-Box Tracing
---

> [!NOTE]
>
> **文档定位**：本文档是 [000-roadmap.md](./000-roadmap.md) Phase 4 的详细工程实施方案，旨在指导 Pillar IV「**The Realm of Mind (心智空间)**」的落地与白盒化验证。内容涵盖技术调研、架构设计、代码实现及全链路测试。
>
> **前置依赖**：本阶段构建于 Phase 1-3 的基座之上，复用其 **Unified Schema** ([The Pulse](./010-the-pulse.md))、**Memory Consolidation** ([The Hippocampus](./020-the-hippocampus.md)) 与 **Fusion Retrieval** ([The Perception](./030-the-perception.md)) 能力。

---

## 1. 执行摘要

### 1.1 定位与目标 (Phase 4)

**Phase 4: The Realm of Mind** 是整个验证计划的**集成核心阶段**，对标人类大脑的**前额叶皮层 (Prefrontal Cortex)** —— 负责执行控制、计划决策与工具调度的中枢。

本阶段的核心任务是 **"Connecting the Dots"**：将前三个阶段构建的感知 (Perception)、记忆 (Hippocampus) 与状态 (Pulse) 能力，通过标准化的 **Agent Runtime** 编排起来，形成具备完整心智的智能体。

**关键交付目标**：

1. **Framework Adaptation (骨架适配)**：实现 `PostgresSessionService` 与 `PostgresMemoryService`，完成与 **Google ADK** 的标准化接口对接。
2. **Glass-Box Runtime (白盒运行时)**：构建完全透明的 `AgentExecutor`，取代黑盒 API，确保推理过程的每一步（Thought -> Action -> Observation）均可观测、可调试。
3. **Dynamic Capability (动态能力)**：构建数据库驱动的 **Tool Registry**，支持工具的热加载与权限控制。
4. **Safe Execution (安全执行)**：集成 **Sandbox (沙箱)** 环境，确保代码解释器 (Code Interpreter) 与外部工具的安全隔离。

```mermaid
graph LR
    subgraph Foundation ["Phase 1-3: The Cortex Integration"]
        direction TB
        P1[Pulse<br>Session State]
        P2[Hippocampus<br>Long-term Memory]
        P3[Perception<br>Fusion Retrieval]
    end

    subgraph Realm ["Phase 4: The Realm of Mind"]
        direction TB
        Adapter[ADK Adapter<br>Standard Interface]
        Registry[Tool Registry<br>Dynamic Capabilities]
        Runtime[Glass-Box Runtime<br>Tracing & Orchestration]
        Sandbox[Sandbox<br>Safe Execution]
    end

    P1 & P2 & P3 --> Adapter
    Adapter --> Runtime
    Registry --> Runtime
    Sandbox -.-> Runtime

    Runtime --> V[Phase 5: Integrated Demo]

    style Foundation fill:#1e293b,stroke:#475569,color:#e2e8f0
    style Realm fill:#0f172a,stroke:#3b82f6,stroke-width:2px,color:#fff
    style Adapter fill:#1e3a8a,stroke:#60a5fa,color:#fff
    style Registry fill:#1e3a8a,stroke:#60a5fa,color:#fff
    style Runtime fill:#1e40af,stroke:#93c5fd,stroke-width:3px,color:#fff
    style Sandbox fill:#3730a3,stroke:#818cf8,color:#fff
    style V fill:#064e3b,stroke:#34d399,color:#fff
```

### 1.2 核心概念解析

#### 1.2.1 Glass-Box vs Black-Box

> [!IMPORTANT]
>
> **核心价值**：Phase 4 将 Google Vertex AI Agent Engine 的黑盒托管能力，解构为完全透明、可调试、可掌控的 **Glass-Box Implementation**。

| 维度           | Black-Box (Google Vertex AI)           | Glass-Box (Open Agent Engine) |
| :------------- | :------------------------------------- | :---------------------------- |
| **可观测性**   | 仅 Input/Output 与计费 Token           | OpenTelemetry 级全链路追踪    |
| **调试能力**   | 难以定位推理死循环或幻觉               | Step-by-Step Trace 可视化调试 |
| **存储后端**   | Firestore/Redis + Vertex Vector Search | PostgreSQL 统一存储           |
| **运维成本**   | Serverless (Managed)                   | Self-hosted / 多地多活        |
| **供应商锁定** | 强依赖 GCP                             | Vendor Agnostic               |

#### 1.2.2 ADK Service 接口适配

本方案采用 **Adapter Pattern**，将 Cognizes Engine 的底层能力（Pulse, Hippocampus）映射为 Google ADK 的标准接口。

```mermaid
graph TB
    subgraph ADK ["Google ADK Framework"]
        direction TB
        Runner[Agent Runner<br>Event Loop]
        Agent[GenAI Agent<br>Probabilistic Logic]
    end

    subgraph Interfaces ["Service Interface Layer (Abstract)"]
        direction TB
        SS[SessionService]
        MS[MemoryService]
    end

    subgraph Implementation ["Open Agent Engine (Adapters)"]
        direction TB
        PSS[PostgresSessionService<br>The Pulse Adapter]
        PMS[PostgresMemoryService<br>The Hippocampus Adapter]
    end

    Agent --> Runner
    Runner --> SS & MS

    SS -.-> PSS
    MS -.-> PMS

    PSS -.-> Pulse[(The Pulse<br>Postgres DB)]
    PMS -.-> Hippo[(The Hippocampus<br>Postgres DB)]

    style ADK fill:#1e293b,stroke:#475569,color:#fff
    style Interfaces fill:#334155,stroke:#64748b,stroke-dasharray: 5 5,color:#fff
    style Implementation fill:#0f172a,stroke:#3b82f6,stroke-width:2px,color:#fff
    style PSS fill:#1e40af,stroke:#60a5fa,color:#fff
    style PMS fill:#1e40af,stroke:#60a5fa,color:#fff
    style Pulse fill:#064e3b,stroke:#34d399,color:#fff
    style Hippo fill:#064e3b,stroke:#34d399,color:#fff
```

### 1.3 执行导图 (Execution Map)

#### 1.3.1 任务-文档锚定

> [!NOTE]
>
> 本矩阵建立了 **Planning (任务)**、**Navigation (文档)** 与 **Delivery (产物)** 的三维映射。

| 任务模块 (Module)    | 任务 ID (Range)   | 预估工期 | 交付物 (Deliverables)    | 对应章节 (Navigation)                                        |
| :------------------- | :---------------- | :------- | :----------------------- | :----------------------------------------------------------- |
| **1. ADK 调研**      | P4-1-1 ~ P4-1-5   | 0.25 Day | 接口分析笔记 + 时序图    | [2. 技术调研](#2-技术调研adk-runtime-深度分析)               |
| **2. Session 适配**  | P4-2-1 ~ P4-2-8   | 0.50 Day | `PostgresSessionService` | [4.1 SessionService](#41-step-1-postgressessionservice-实现) |
| **3. Memory 适配**   | P4-2-9 ~ P4-2-12  | 0.25 Day | `PostgresMemoryService`  | [4.2 MemoryService](#42-step-2-postgresmemoryservice-实现)   |
| **4. Tool Registry** | P4-2-13 ~ P4-2-18 | 0.25 Day | `ToolRegistry` + Schema  | [4.3 Tool Registry](#43-step-3-tool-registry-实现)           |
| **5. Runtime Loop**  | P4-2-19 ~ P4-2-23 | 0.25 Day | `AgentExecutor`          | [4.4 AgentExecutor](#44-step-4-agentexecutor-实现)           |
| **6. Testing**       | P4-3-1 ~ P4-3-4   | 0.50 Day | 测试套件 + 技术文档      | [4.5 测试实现](#45-step-5-测试实现)                          |
| **7. Tracing**       | P4-4-1 ~ P4-4-4   | 0.25 Day | OpenTelemetry 集成       | [4.6 OpenTelemetry](#46-step-6-opentelemetry-集成)           |
| **8. Sandbox**       | P4-4-5 ~ P4-4-10  | 0.25 Day | 安全隔离环境             | [4.7 安全沙箱](#47-step-7-sandboxed-execution-实现)          |
| **9. Validation**    | P4-4-11 ~ P4-4-13 | 0.25 Day | 可视化 Trace 验证        | [4.8 可视化验证](#48-step-8-可视化验证)                      |
| **10. UI Protocol**  | P4-5-1 ~ P4-5-10  | 0.50 Day | AG-UI Protocol Adapter   | [4.9 AG-UI 协议](#49-step-9-ag-ui-协议集成)                  |

#### 1.3.2 实施甘特图 (Gantt Chart)

```mermaid
gantt
    title Phase 4 Execution Timeline (Total: ~3.5 Days)
    dateFormat  XA
    axisFormat  Day %d

    section Foundation & Pulse
    ADK Audit & Design       :active, t1, 0, 2h
    PostgresSession Impl     :t2, after t1, 4h

    section Hippocampus
    PostgresMemory Impl      :t3, after t2, 2h

    section Mind Runtime
    Tool Registry            :t4, after t3, 2h
    AgentExecutor Loop       :t5, after t4, 2h
    Unit & Integ Testing     :t6, after t5, 4h

    section Observability
    OpenTelemetry Tracing    :t7, after t6, 2h
    Glass-Box Validation     :t8, after t7, 2h

    section Safety & UI
    Sandbox Isolation        :t9, after t8, 2h
    AG-UI Integration        :t10, after t9, 4h
    Final Acceptance         :t11, after t10, 2h
```

---

## 2. 核心参考模型：ADK Runtime

本章旨在确立 **Golden Standard (基准规范)**。通过解构 Google ADK 的抽象接口，明确 Pillar I (Pulse) 与 Pillar II (Hippocampus) 在代码层面的标准化契约。

### 2.1 契约定义：SessionService

`SessionService` (对标 `google/adk-python` [base_session_service.py](https://github.com/google/adk-python/blob/main/src/google/adk/sessions/base_session_service.py)) 是 **The Pulse (脉搏系统)** 的运行时接口。它不仅负责 Session 的 CRUD，还通过 `append_event` 方法承载了 **状态流转 (State Transitions)** 的核心逻辑。

> [!IMPORTANT]
>
> **关键发现**：`append_event()` **不是抽象方法**，基类提供了默认实现来处理 `temp:` 前缀过滤和 State 更新。子类只需覆写 CRUD 方法。
>
> 这种设计模式确保了所有 adapter (如 InMemory, Firestore, Postgres) 共享完全一致的 **State Transition Logic**，防止因实现差异导致的行为不一致。`SessionService` 在此扮演了 **Boundary Object** 的角色，定义了 Pulse 与外部存储交互的唯一合法途径。

#### 2.1.1 Session 数据结构

```python
@dataclass
class Session:
    """会话对象"""
    id: str                    # 会话唯一标识 (UUID)
    app_name: str              # 应用名称
    user_id: str               # 用户标识
    state: dict[str, Any]      # Key-Value 状态存储
    events: list[Event]        # 交互事件序列 (append-only)
    last_update_time: float    # 最后更新时间戳
```

#### 2.1.2 State 前缀机制

ADK 通过 Key 前缀实现不同作用域的状态管理：

| 前缀    | 作用域           | 生命周期              | PostgreSQL 存储策略   |
| :------ | :--------------- | :-------------------- | :-------------------- |
| 无前缀  | Session Scope    | 取决于 SessionService | `threads.state` JSONB |
| `user:` | User Scope       | 持久化                | `user_states` 表      |
| `app:`  | App Scope        | 持久化                | `app_states` 表       |
| `temp:` | Invocation Scope | 仅当前调用            | 内存缓存，不持久化    |

### 2.2 ADK MemoryService 接口契约

```python
from abc import ABC, abstractmethod
from google.adk.memory import SearchMemoryResponse

class BaseMemoryService(ABC):
    """Memory 管理服务抽象基类"""

    @abstractmethod
    async def add_session_to_memory(
        self,
        session: Session
    ) -> None:
        """将 Session 中的对话转化为可搜索的记忆"""
        ...

    @abstractmethod
    async def search_memory(
        self,
        *,
        app_name: str,
        user_id: str,
        query: str
    ) -> SearchMemoryResponse:
        """基于 Query 检索相关记忆"""
        ...
```

### 2.3 ADK Runner 与 Service 交互流程

```mermaid
sequenceDiagram
    participant User
    participant Runner
    participant Agent
    participant SS as SessionService
    participant MS as MemoryService

    User->>Runner: User Query
    Runner->>SS: get_session()
    SS-->>Runner: Session (with events, state)
    Runner->>MS: search_memory(query)
    MS-->>Runner: Relevant Memories

    Runner->>Agent: invoke(context)
    loop Event Loop
        Agent-->>Runner: Yield Event
        Runner->>SS: append_event(event)
        Note over SS: Apply state_delta
        Runner-->>User: Stream Response
    end

    Agent-->>Runner: Final Event
    Runner->>SS: finalize_session()
    opt Memory Consolidation
        Runner->>MS: add_session_to_memory()
    end
```

### 2.4 调研交付物摘要

#### 2.4.1 SessionService 关键行为分析 (P4-1-1, P4-1-3)

| 行为               | 描述                                            | 实现要点                       |
| :----------------- | :---------------------------------------------- | :----------------------------- |
| **State Commit**   | `state_delta` 仅在 Event 被 Runner 处理后才提交 | 需在 `append_event` 中原子更新 |
| **Dirty Reads**    | 同一 Invocation 内可见未提交的 State 变更       | 内存缓存 + 最终 PG 持久化      |
| **Event Ordering** | Events 必须严格按序列号排序                     | 使用 `BIGSERIAL` 保证顺序      |
| **Prefix Routing** | 不同前缀路由到不同存储位置                      | 解析前缀后分发到对应表         |

#### 2.4.2 MemoryService 关键行为分析 (P4-1-2, P4-1-4)

| 行为                  | 描述                                    | 实现要点                        |
| :-------------------- | :-------------------------------------- | :------------------------------ |
| **Session Ingestion** | 将 Session Events 转化为可检索的 Memory | 调用 Phase 2 的 consolidate()   |
| **Semantic Search**   | 基于向量相似度检索相关记忆              | 复用 Phase 3 的 hybrid_search() |
| **User Isolation**    | 不同用户的 Memory 严格隔离              | WHERE user_id = $user_id        |

#### 2.4.3 InMemorySessionService 源码分析 (P4-1-3)

> [!NOTE]
>
> **源码位置**: `google/adk-python/src/google/adk/sessions/in_memory_session_service.py`

```python
# 关键数据结构分析 (简化版)
class InMemorySessionService(BaseSessionService):
    """
    核心设计模式:
    1. 使用 dict 嵌套存储: {app_name: {user_id: {session_id: Session}}}
    2. 深拷贝隔离: 每次返回 Session 的深拷贝，防止外部修改
    3. 状态快照: append_event 后立即更新内存状态
    """

    def __init__(self):
        # 三层嵌套字典: app_name -> user_id -> session_id -> Session
        self._sessions: dict[str, dict[str, dict[str, Session]]] = {}

    async def create_session(self, *, app_name, user_id, state=None, session_id=None):
        session_id = session_id or str(uuid.uuid4())
        session = Session(
            id=session_id,
            app_name=app_name,
            user_id=user_id,
            state=dict(state or {}),  # 深拷贝初始状态
            events=[],
            last_update_time=time.time()
        )
        # 确保嵌套结构存在
        self._sessions.setdefault(app_name, {}).setdefault(user_id, {})[session_id] = session
        return copy.deepcopy(session)  # 返回深拷贝

    async def get_session(self, *, app_name, user_id, session_id, config=None):
        session = self._sessions.get(app_name, {}).get(user_id, {}).get(session_id)
        if not session:
            return None
        result = copy.deepcopy(session)
        # 应用 GetSessionConfig
        if config:
            if config.num_recent_events:
                result.events = result.events[-config.num_recent_events:]
            if config.after_timestamp:
                result.events = [e for e in result.events if e.timestamp > config.after_timestamp]
        return result
```

**PostgreSQL 实现对标要点**:

| InMemory 行为  | PostgreSQL 实现                                           |
| :------------- | :-------------------------------------------------------- |
| 嵌套 dict 存储 | `threads` 表 + 组合主键 `(app_name, user_id, session_id)` |
| 深拷贝隔离     | 每次查询返回独立 Row 对象                                 |
| 状态快照       | 事务内 `UPDATE threads SET state = $new`                  |
| Event 过滤     | `WHERE seq > $after_seq LIMIT $num_recent`                |

#### 2.4.4 InMemoryMemoryService 源码分析 (P4-1-4)

```python
# 关键设计模式分析
class InMemoryMemoryService(BaseMemoryService):
    """
    核心设计:
    1. 简化实现: 仅存储 Session 的文本摘要
    2. 无真实向量: search_memory 使用字符串匹配
    3. 命名空间隔离: {app_name: {user_id: [MemoryEntry]}}
    """

    def __init__(self):
        self._memories: dict[str, dict[str, list[MemoryEntry]]] = {}

    async def add_session_to_memory(self, session: Session):
        # 简化: 将所有 Event 内容合并为单条记忆
        content = " ".join([str(e.content) for e in session.events])
        entry = MemoryEntry(
            id=str(uuid.uuid4()),
            content=content,
            session_id=session.id,
            created_at=datetime.now()
        )
        self._memories.setdefault(session.app_name, {}) \
                      .setdefault(session.user_id, []).append(entry)

    async def search_memory(self, *, app_name, user_id, query):
        memories = self._memories.get(app_name, {}).get(user_id, [])
        # 简化: 字符串包含匹配 (无真实向量搜索)
        matched = [m for m in memories if query.lower() in m.content.lower()]
        return SearchMemoryResponse(memories=matched)
```

**PostgreSQL 实现增强点**:

| InMemory 局限 | PostgreSQL 增强                                |
| :------------ | :--------------------------------------------- |
| 无向量搜索    | PGVector `<=>` 操作符进行真实语义搜索          |
| 无记忆巩固    | 调用 Phase 2 `consolidation_worker` 提取 Facts |
| 无遗忘机制    | `retention_score` 权重衰减 + 定期清理          |
| 无混合检索    | `hybrid_search` 融合语义+关键词+元数据         |

---

## 3. 架构设计：Mind Schema 扩展

### 3.1 Schema 扩展设计

在 Phase 1-3 的 Unified Schema 基础上，新增以下运行时相关表：

```mermaid
erDiagram
    threads ||--o{ events : contains
    threads ||--o{ runs : has
    tools ||--o{ tool_executions : tracked_by

    tools {
        uuid id PK "工具唯一标识"
        varchar app_name "应用名称"
        varchar name "工具名称"
        varchar display_name "显示名称"
        text description "工具描述"
        jsonb openapi_schema "OpenAPI 规范"
        jsonb permissions "权限配置"
        boolean is_active "是否启用"
        integer call_count "调用次数"
        float avg_latency_ms "平均延迟"
        timestamp created_at "创建时间"
        timestamp updated_at "更新时间"
    }

    tool_executions {
        uuid id PK "执行记录标识"
        uuid tool_id FK "工具 ID"
        uuid run_id FK "执行链路 ID"
        jsonb input_params "输入参数"
        jsonb output_result "输出结果"
        varchar status "执行状态"
        float latency_ms "执行耗时"
        text error "错误信息"
        timestamp created_at "执行时间"
    }

    traces {
        uuid id PK "Trace 唯一标识"
        uuid run_id FK "执行链路 ID"
        varchar trace_id "OpenTelemetry Trace ID"
        varchar span_id "Span ID"
        varchar parent_span_id "父 Span ID"
        varchar operation_name "操作名称"
        varchar span_kind "Span 类型"
        jsonb attributes "属性"
        jsonb events "事件列表"
        timestamp start_time "开始时间"
        timestamp end_time "结束时间"
        integer duration_ns "耗时 (纳秒)"
        varchar status_code "状态码"
    }

    sandbox_executions {
        uuid id PK "沙箱执行标识"
        uuid run_id FK "执行链路 ID"
        varchar sandbox_type "沙箱类型"
        text code "执行代码"
        jsonb environment "环境变量"
        text stdout "标准输出"
        text stderr "标准错误"
        integer exit_code "退出码"
        float execution_time_ms "执行时间"
        jsonb resource_usage "资源使用"
        timestamp created_at "创建时间"
    }
```

### 3.2 表职责说明

| 表名                   | 职责                     | 对标概念              | 生命周期   |
| :--------------------- | :----------------------- | :-------------------- | :--------- |
| **tools**              | 工具注册表 (动态加载)    | ADK Function Registry | 持久化     |
| **tool_executions**    | 工具执行记录 (审计追踪)  | Tool Call Audit Log   | 按策略归档 |
| **traces**             | OpenTelemetry Trace 存储 | Langfuse/OTLP         | 按策略清理 |
| **sandbox_executions** | 沙箱执行记录             | Code Interpreter Log  | 按策略清理 |

### 3.3 核心表 DDL 设计

#### 3.3.1 tools 表 (工具注册表)

```sql
-- tools: 动态工具注册表
CREATE TABLE IF NOT EXISTS tools (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    app_name        VARCHAR(255) NOT NULL,
    name            VARCHAR(255) NOT NULL,
    display_name    VARCHAR(255),
    description     TEXT,

    -- OpenAPI Schema (JSON 格式)
    openapi_schema  JSONB NOT NULL,

    -- 权限与配置
    permissions     JSONB DEFAULT '{}',
    -- 示例: {"allowed_users": ["*"], "rate_limit": 100, "requires_confirmation": false}

    -- 状态
    is_active       BOOLEAN NOT NULL DEFAULT true,

    -- 统计信息
    call_count      INTEGER NOT NULL DEFAULT 0,
    avg_latency_ms  FLOAT DEFAULT 0,

    -- 时间戳
    created_at      TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at      TIMESTAMP WITH TIME ZONE DEFAULT NOW(),

    -- 每个应用的工具名称唯一
    CONSTRAINT tools_app_name_unique UNIQUE (app_name, name)
);

CREATE INDEX IF NOT EXISTS idx_tools_app_name ON tools(app_name);
CREATE INDEX IF NOT EXISTS idx_tools_is_active ON tools(app_name, is_active);
```

#### 3.3.2 traces 表 (OpenTelemetry 存储)

```sql
-- traces: OpenTelemetry Trace 结构化存储
CREATE TABLE IF NOT EXISTS traces (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    run_id              UUID REFERENCES runs(id) ON DELETE CASCADE,

    -- OpenTelemetry 标识
    trace_id            VARCHAR(32) NOT NULL,
    span_id             VARCHAR(16) NOT NULL,
    parent_span_id      VARCHAR(16),

    -- Span 信息
    operation_name      VARCHAR(255) NOT NULL,
    span_kind           VARCHAR(20) NOT NULL DEFAULT 'INTERNAL',
    -- CHECK (span_kind IN ('INTERNAL', 'SERVER', 'CLIENT', 'PRODUCER', 'CONSUMER'))

    -- 属性与事件
    attributes          JSONB DEFAULT '{}',
    events              JSONB DEFAULT '[]',

    -- 时间信息
    start_time          TIMESTAMP WITH TIME ZONE NOT NULL,
    end_time            TIMESTAMP WITH TIME ZONE,
    duration_ns         BIGINT,

    -- 状态
    status_code         VARCHAR(10) DEFAULT 'UNSET',
    -- CHECK (status_code IN ('UNSET', 'OK', 'ERROR'))
    status_message      TEXT,

    -- 创建时间
    created_at          TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

CREATE INDEX IF NOT EXISTS idx_traces_run_id ON traces(run_id);
CREATE INDEX IF NOT EXISTS idx_traces_trace_id ON traces(trace_id);
CREATE INDEX IF NOT EXISTS idx_traces_start_time ON traces(start_time DESC);
```

---

## 4. 实施指南

### 4.1 Step 1: PostgresSessionService 实现

#### 4.1.1 实现目标

实现完全兼容 ADK `BaseSessionService` 接口的 PostgreSQL 适配器。

**任务清单**：

| 任务 ID | 任务描述                                | 验收标准                    |
| :------ | :-------------------------------------- | :-------------------------- |
| P4-2-1  | 创建 `adk-postgres` Python Package 结构 | `pyproject.toml` + 目录结构 |
| P4-2-2  | 实现 `PostgresSessionService` 类框架    | 继承 `BaseSessionService`   |
| P4-2-3  | 实现 `create_session()` 方法            | 创建新 Session 并返回       |
| P4-2-4  | 实现 `get_session()` 方法               | 根据 ID 加载 Session        |
| P4-2-5  | 实现 `list_sessions()` 方法             | 列出用户所有 Sessions       |
| P4-2-6  | 实现 `delete_session()` 方法            | 删除指定 Session            |
| P4-2-7  | 实现 `append_event()` 方法              | 追加 Event 到 Session       |
| P4-2-8  | 实现 State 前缀处理                     | 不同前缀存储至不同作用域    |

#### 4.1.2 目录结构决策

**决策：采用统一项目结构 (Integrated Structure)**

为了降低初期开发和联调的复杂度，建议将 `adk-postgres` 作为 `cognizes` 引擎的一个子模块（Adapter），共用主项目的 `pyproject.toml` 和开发环境。

**推荐目录结构：**

```text
src/cognizes/
├── adapters/
│   └── postgres/               # 适配器核心实现
│       ├── __init__.py
│       ├── session_service.py  # PostgresSessionService
│       ├── memory_service.py   # PostgresMemoryService
│       ├── tool_registry.py    # ToolRegistry
│       ├── tracing.py          # OpenTelemetry 集成
│       ├── sandbox/            # 沙箱实现
│       │   ├── __init__.py
│       │   ├── base.py
│       │   └── microsandbox_runner.py
│       └── models/             # 适配器私有模型（或使用全局模型）
│           ├── __init__.py
│           ├── session.py
│           └── event.py
tests/
├── unittests/adapters/postgres
└── integrationtests/adapters/postgres
```

**理由：**

1. **开发效率**：避免在多个项目间频繁切换和执行 `pip install -e`。
2. **原子提交**：引擎接口的变更及其在 Postgres 适配器中的实现可以在同一个 Commit 中完成，保证代码一致性。
3. **简化 CI/CD**：统一的测试运行环境和依赖管理，降低维护成本。
4. **未来解耦**：`adapters/postgres` 目录保持高度内聚，未来若需独立发布为 `adk-postgres` 包，只需将其移动到独立仓库并添加 `pyproject.toml` 即可。

#### 4.1.3 核心实现代码

创建 `src/cognizes/adapters/postgres/session_service.py`：

```python
"""
PostgresSessionService: ADK SessionService 的 PostgreSQL 实现

实现对标 Google ADK BaseSessionService 的会话管理能力：
- Session CRUD 操作
- Event 追加与 State Delta 应用
- State 前缀作用域路由
"""

from __future__ import annotations

import json
import uuid
from dataclasses import dataclass, field
from datetime import datetime
from typing import Any

import asyncpg

# ADK 接口类型 (根据实际 ADK 版本调整)
# from google.adk.sessions import BaseSessionService, Session, Event


@dataclass
class Session:
    """会话对象 - 对标 ADK Session"""
    id: str
    app_name: str
    user_id: str
    state: dict[str, Any] = field(default_factory=dict)
    events: list[dict] = field(default_factory=list)
    last_update_time: float = 0.0


@dataclass
class Event:
    """事件对象 - 对标 ADK Event"""
    id: str
    author: str
    content: dict[str, Any] = field(default_factory=dict)
    actions: dict[str, Any] = field(default_factory=dict)
    timestamp: float = 0.0


class PostgresSessionService:
    """
    PostgreSQL 实现的 SessionService

    核心职责：
    1. Session 生命周期管理 (CRUD)
    2. Event 追加与 State 更新
    3. State 前缀路由 (无前缀/user:/app:/temp:)
    """

    def __init__(self, pool: asyncpg.Pool):
        self._pool = pool
        self._temp_state: dict[str, dict] = {}  # temp: 前缀的内存缓存

    async def create_session(
        self,
        *,
        app_name: str,
        user_id: str,
        session_id: str | None = None,
        state: dict | None = None
    ) -> Session:
        """创建新会话"""
        sid = session_id or str(uuid.uuid4())
        initial_state = state or {}

        async with self._pool.acquire() as conn:
            await conn.execute(
                """
                INSERT INTO threads (id, app_name, user_id, state)
                VALUES ($1, $2, $3, $4)
                """,
                uuid.UUID(sid), app_name, user_id, json.dumps(initial_state)
            )

        return Session(
            id=sid,
            app_name=app_name,
            user_id=user_id,
            state=initial_state,
            events=[],
            last_update_time=datetime.now().timestamp()
        )

    async def get_session(
        self,
        *,
        app_name: str,
        user_id: str,
        session_id: str
    ) -> Session | None:
        """获取会话"""
        async with self._pool.acquire() as conn:
            # 获取 Thread
            row = await conn.fetchrow(
                """
                SELECT id, app_name, user_id, state, updated_at
                FROM threads
                WHERE id = $1 AND app_name = $2 AND user_id = $3
                """,
                uuid.UUID(session_id), app_name, user_id
            )
            if not row:
                return None

            # 获取 Events
            events = await conn.fetch(
                """
                SELECT id, author, event_type, content, actions, created_at
                FROM events
                WHERE thread_id = $1
                ORDER BY sequence_num ASC
                """,
                uuid.UUID(session_id)
            )

            return Session(
                id=str(row['id']),
                app_name=row['app_name'],
                user_id=row['user_id'],
                state=json.loads(row['state']) if row['state'] else {},
                events=[self._row_to_event(e) for e in events],
                last_update_time=row['updated_at'].timestamp()
            )

    async def list_sessions(
        self,
        *,
        app_name: str,
        user_id: str
    ) -> list[Session]:
        """列出用户所有会话"""
        async with self._pool.acquire() as conn:
            rows = await conn.fetch(
                """
                SELECT id, app_name, user_id, state, updated_at
                FROM threads
                WHERE app_name = $1 AND user_id = $2
                ORDER BY updated_at DESC
                """,
                app_name, user_id
            )

        return [
            Session(
                id=str(row['id']),
                app_name=row['app_name'],
                user_id=row['user_id'],
                state=json.loads(row['state']) if row['state'] else {},
                events=[],  # 列表不加载 events
                last_update_time=row['updated_at'].timestamp()
            )
            for row in rows
        ]

    async def delete_session(
        self,
        *,
        app_name: str,
        user_id: str,
        session_id: str
    ) -> None:
        """删除会话"""
        async with self._pool.acquire() as conn:
            await conn.execute(
                """
                DELETE FROM threads
                WHERE id = $1 AND app_name = $2 AND user_id = $3
                """,
                uuid.UUID(session_id), app_name, user_id
            )

    async def append_event(
        self,
        session: Session,
        event: Event
    ) -> Event:
        """追加事件并应用 state_delta"""
        event_id = event.id or str(uuid.uuid4())
        invocation_id = str(uuid.uuid4())

        async with self._pool.acquire() as conn:
            async with conn.transaction():
                # 1. 插入 Event
                await conn.execute(
                    """
                    INSERT INTO events
                    (id, thread_id, invocation_id, author, event_type, content, actions)
                    VALUES ($1, $2, $3, $4, $5, $6, $7)
                    """,
                    uuid.UUID(event_id),
                    uuid.UUID(session.id),
                    uuid.UUID(invocation_id),
                    event.author,
                    'message',  # 默认类型
                    json.dumps(event.content),
                    json.dumps(event.actions)
                )

                # 2. 应用 state_delta (如果有)
                state_delta = event.actions.get('state_delta', {})
                if state_delta:
                    await self._apply_state_delta(
                        conn, session, state_delta
                    )

        event.id = event_id
        return event

    async def _apply_state_delta(
        self,
        conn: asyncpg.Connection,
        session: Session,
        state_delta: dict[str, Any]
    ) -> None:
        """应用 state_delta，根据前缀路由到不同存储"""
        session_updates = {}
        user_updates = {}
        app_updates = {}

        for key, value in state_delta.items():
            if key.startswith('temp:'):
                # temp: 前缀 -> 内存缓存
                cache_key = f"{session.id}:{key}"
                self._temp_state[cache_key] = value
            elif key.startswith('user:'):
                # user: 前缀 -> user_states 表
                real_key = key[5:]  # 去掉 'user:' 前缀
                user_updates[real_key] = value
            elif key.startswith('app:'):
                # app: 前缀 -> app_states 表
                real_key = key[4:]  # 去掉 'app:' 前缀
                app_updates[real_key] = value
            else:
                # 无前缀 -> threads.state
                session_updates[key] = value

        # 更新 Session State
        if session_updates:
            await conn.execute(
                """
                UPDATE threads
                SET state = state || $1::jsonb, updated_at = NOW()
                WHERE id = $2
                """,
                json.dumps(session_updates),
                uuid.UUID(session.id)
            )
            session.state.update(session_updates)

        # 更新 User State
        if user_updates:
            await conn.execute(
                """
                INSERT INTO user_states (user_id, app_name, state)
                VALUES ($1, $2, $3)
                ON CONFLICT (user_id, app_name)
                DO UPDATE SET state = user_states.state || $3::jsonb,
                              updated_at = NOW()
                """,
                session.user_id, session.app_name, json.dumps(user_updates)
            )

        # 更新 App State
        if app_updates:
            await conn.execute(
                """
                INSERT INTO app_states (app_name, state)
                VALUES ($1, $2)
                ON CONFLICT (app_name)
                DO UPDATE SET state = app_states.state || $2::jsonb,
                              updated_at = NOW()
                """,
                session.app_name, json.dumps(app_updates)
            )

    def _row_to_event(self, row) -> dict:
        """将数据库行转换为 Event 字典"""
        return {
            'id': str(row['id']),
            'author': row['author'],
            'content': json.loads(row['content']) if row['content'] else {},
            'actions': json.loads(row['actions']) if row['actions'] else {},
            'timestamp': row['created_at'].timestamp()
        }
```

---

### 4.2 Step 2: PostgresMemoryService 实现

#### 4.2.1 实现目标

实现完全兼容 ADK `BaseMemoryService` 接口的 PostgreSQL 适配器，复用 Phase 2 的记忆巩固能力。

**任务清单**：

| 任务 ID | 任务描述                            | 验收标准                 |
| :------ | :---------------------------------- | :----------------------- |
| P4-2-9  | 实现 `PostgresMemoryService` 类框架 | 继承 `BaseMemoryService` |
| P4-2-10 | 实现 `add_session_to_memory()` 方法 | Session 记忆摄入         |
| P4-2-11 | 实现 `search_memory()` 方法         | 向量相似度检索           |
| P4-2-12 | 实现 `list_memories()` 方法         | 列出所有记忆             |

#### 4.2.2 核心实现代码

创建 `src/cognizes/adapters/postgres/memory_service.py`：

```python
"""
PostgresMemoryService: ADK MemoryService 的 PostgreSQL 实现

复用 Phase 2 Hippocampus 的记忆巩固能力，实现：
- Session 到 Memory 的转化 (add_session_to_memory)
- 语义检索 (search_memory)
"""

from __future__ import annotations

import json
import uuid
from dataclasses import dataclass
from typing import Any

import asyncpg


@dataclass
class Memory:
    """记忆对象"""
    id: str
    content: str
    metadata: dict[str, Any]
    relevance_score: float = 0.0


@dataclass
class SearchMemoryResponse:
    """记忆检索响应"""
    memories: list[Memory]


class PostgresMemoryService:
    """
    PostgreSQL 实现的 MemoryService

    核心职责：
    1. 将 Session 对话转化为可搜索的记忆 (复用 Phase 2 consolidate)
    2. 基于语义相似度检索相关记忆 (复用 Phase 3 hybrid_search)
    """

    def __init__(
        self,
        pool: asyncpg.Pool,
        embedding_fn: callable = None,
        consolidation_worker = None
    ):
        self._pool = pool
        self._embedding_fn = embedding_fn  # 向量化函数
        self._consolidation_worker = consolidation_worker  # Phase 2 Worker

    async def add_session_to_memory(
        self,
        session  # Session 对象
    ) -> None:
        """将 Session 中的对话转化为可搜索的记忆"""
        if self._consolidation_worker:
            # 使用 Phase 2 的 consolidate 函数
            await self._consolidation_worker.consolidate(
                thread_id=session.id,
                user_id=session.user_id,
                app_name=session.app_name
            )
        else:
            # 简化实现：直接将 Events 向量化存储
            await self._simple_consolidate(session)

    async def _simple_consolidate(self, session) -> None:
        """简化版记忆巩固 (用于测试)"""
        # 提取所有 user 消息
        user_messages = [
            e.get('content', {}).get('text', '')
            for e in session.events
            if e.get('author') == 'user' and e.get('content')
        ]

        if not user_messages:
            return

        # 合并为单条记忆
        combined_content = '\n'.join(user_messages)

        # 生成向量 (如果有 embedding 函数)
        embedding = None
        if self._embedding_fn:
            embedding = await self._embedding_fn(combined_content)

        async with self._pool.acquire() as conn:
            await conn.execute(
                """
                INSERT INTO memories
                (thread_id, user_id, app_name, memory_type, content, embedding, metadata)
                VALUES ($1, $2, $3, $4, $5, $6, $7)
                """,
                uuid.UUID(session.id) if session.id else None,
                session.user_id,
                session.app_name,
                'episodic',
                combined_content,
                embedding,
                json.dumps({'source': 'session', 'event_count': len(session.events)})
            )

    async def search_memory(
        self,
        *,
        app_name: str,
        user_id: str,
        query: str,
        limit: int = 10
    ) -> SearchMemoryResponse:
        """基于 Query 检索相关记忆"""
        # 生成查询向量
        query_embedding = None
        if self._embedding_fn:
            query_embedding = await self._embedding_fn(query)

        async with self._pool.acquire() as conn:
            if query_embedding:
                # 向量检索
                rows = await conn.fetch(
                    """
                    SELECT id, content, metadata,
                           1 - (embedding <=> $1) AS relevance_score
                    FROM memories
                    WHERE user_id = $2 AND app_name = $3
                    ORDER BY embedding <=> $1
                    LIMIT $4
                    """,
                    query_embedding, user_id, app_name, limit
                )
            else:
                # 降级为全文检索
                rows = await conn.fetch(
                    """
                    SELECT id, content, metadata,
                           ts_rank_cd(search_vector, plainto_tsquery($1)) AS relevance_score
                    FROM memories
                    WHERE user_id = $2 AND app_name = $3
                      AND search_vector @@ plainto_tsquery($1)
                    ORDER BY relevance_score DESC
                    LIMIT $4
                    """,
                    query, user_id, app_name, limit
                )

        memories = [
            Memory(
                id=str(row['id']),
                content=row['content'],
                metadata=json.loads(row['metadata']) if row['metadata'] else {},
                relevance_score=float(row['relevance_score'] or 0)
            )
            for row in rows
        ]

        return SearchMemoryResponse(memories=memories)

    async def list_memories(
        self,
        *,
        app_name: str,
        user_id: str,
        limit: int = 100
    ) -> list[Memory]:
        """列出用户所有记忆"""
        async with self._pool.acquire() as conn:
            rows = await conn.fetch(
                """
                SELECT id, content, metadata, retention_score
                FROM memories
                WHERE user_id = $1 AND app_name = $2
                ORDER BY created_at DESC
                LIMIT $3
                """,
                user_id, app_name, limit
            )

        return [
            Memory(
                id=str(row['id']),
                content=row['content'],
                metadata=json.loads(row['metadata']) if row['metadata'] else {},
                relevance_score=float(row['retention_score'] or 1.0)
            )
            for row in rows
        ]
```

### 4.3 Step 3: Tool Registry 实现

#### 4.3.1 实现目标

实现数据库驱动的动态工具注册表，支持 OpenAPI Schema 动态加载与热更新。

**任务清单**：

| 任务 ID | 任务描述                          | 验收标准                 |
| :------ | :-------------------------------- | :----------------------- |
| P4-2-13 | 设计 `tools` 表 Schema            | DDL + JSONB 存储 OpenAPI |
| P4-2-14 | 实现 `register_tool()` 方法       | 工具注册到 DB            |
| P4-2-15 | 实现 `get_available_tools()` 方法 | 动态加载工具列表         |
| P4-2-16 | 实现工具热更新机制                | 新增工具无需重启服务     |
| P4-2-17 | 集成权限配置                      | 工具调用鉴权与限流       |
| P4-2-18 | 实现执行统计                      | 工具调用计数与性能追踪   |

#### 4.3.2 核心实现代码

创建 `src/cognizes/adapters/postgres/tool_registry.py`：

```python
"""
ToolRegistry: 数据库驱动的动态工具注册表
"""

from __future__ import annotations
import json, uuid
from dataclasses import dataclass
from typing import Any, Callable
import asyncpg

@dataclass
class ToolDefinition:
    id: str
    name: str
    display_name: str
    description: str
    openapi_schema: dict
    permissions: dict
    is_active: bool
    call_count: int
    avg_latency_ms: float

class ToolRegistry:
    def __init__(self, pool: asyncpg.Pool, app_name: str):
        self._pool = pool
        self._app_name = app_name
        self._function_registry: dict[str, Callable] = {}

    async def register_tool(
        self, name: str, func: Callable, *,
        display_name: str | None = None,
        openapi_schema: dict | None = None,
        permissions: dict | None = None
    ) -> ToolDefinition:
        """注册工具到数据库"""
        tool_id = str(uuid.uuid4())
        async with self._pool.acquire() as conn:
            await conn.execute(
                """
                INSERT INTO tools (id, app_name, name, display_name, openapi_schema, permissions)
                VALUES ($1, $2, $3, $4, $5, $6)
                ON CONFLICT (app_name, name) DO UPDATE SET
                    display_name = $4, openapi_schema = $5, permissions = $6
                """,
                uuid.UUID(tool_id), self._app_name, name,
                display_name or name,
                json.dumps(openapi_schema or {}),
                json.dumps(permissions or {'allowed_users': ['*']})
            )
        self._function_registry[name] = func
        return ToolDefinition(id=tool_id, name=name, display_name=display_name or name,
                             description='', openapi_schema=openapi_schema or {},
                             permissions=permissions or {}, is_active=True,
                             call_count=0, avg_latency_ms=0)

    async def get_available_tools(self, user_id: str | None = None) -> list[ToolDefinition]:
        """获取可用工具列表"""
        async with self._pool.acquire() as conn:
            rows = await conn.fetch(
                "SELECT * FROM tools WHERE app_name = $1 AND is_active = true",
                self._app_name
            )
        return [ToolDefinition(
            id=str(r['id']), name=r['name'], display_name=r['display_name'],
            description=r['description'] or '',
            openapi_schema=json.loads(r['openapi_schema']),
            permissions=json.loads(r['permissions']),
            is_active=r['is_active'], call_count=r['call_count'],
            avg_latency_ms=r['avg_latency_ms']
        ) for r in rows]

    async def invoke_tool(self, name: str, params: dict, *, run_id: str | None = None) -> Any:
        """调用工具并记录统计"""
        import time
        func = self._function_registry.get(name)
        if not func:
            raise ValueError(f"Tool '{name}' not found")
        start = time.time()
        result = await func(**params) if asyncio.iscoroutinefunction(func) else func(**params)
        latency = (time.time() - start) * 1000
        # 更新统计
        async with self._pool.acquire() as conn:
            await conn.execute(
                "UPDATE tools SET call_count = call_count + 1, "
                "avg_latency_ms = (avg_latency_ms * call_count + $1) / (call_count + 1) "
                "WHERE app_name = $2 AND name = $3",
                latency, self._app_name, name
            )
        return result
```

---

### 4.4 Step 4: AgentExecutor 实现

#### 4.4.1 实现目标

实现 Python 驱动的 Agent 执行器，管理 `Thought -> Action -> Observation` 循环。

**任务清单**：

| 任务 ID | 任务描述                           | 验收标准                        |
| :------ | :--------------------------------- | :------------------------------ |
| P4-2-19 | 实现 `AgentExecutor` 类框架        | 类定义 + 生命周期方法           |
| P4-2-20 | 实现 `run()` 主循环                | ReAct Loop 正确执行             |
| P4-2-21 | 实现 `invoke_tool()` 方法          | 调用注册工具并返回结果          |
| P4-2-22 | 实现最大步数限制与超时控制         | 防止无限循环                    |
| P4-2-23 | 验证 Executor 与 ADK Runner 的协同 | 与 `InvocationContext` 正确交互 |

#### 4.4.2 核心实现代码

创建 `src/cognizes/engine/mind/agent_executor.py`：

```python
"""
AgentExecutor: Agent 执行编排器 - 管理 Thought -> Action -> Observation 循环
"""

from dataclasses import dataclass
from datetime import datetime
from enum import Enum
from typing import AsyncGenerator

class ExecutionStatus(Enum):
    PENDING = "pending"
    RUNNING = "running"
    COMPLETED = "completed"
    FAILED = "failed"
    TIMEOUT = "timeout"
    MAX_STEPS_REACHED = "max_steps_reached"

@dataclass
class ThinkingStep:
    step_number: int
    thought: str
    action: str | None
    action_input: dict | None
    observation: str | None
    timestamp: datetime

@dataclass
class ExecutionResult:
    status: ExecutionStatus
    final_answer: str | None
    steps: list[ThinkingStep]
    total_duration_ms: float
    error: str | None = None

class AgentExecutor:
    def __init__(self, llm_client, tool_registry, *, max_steps: int = 10, timeout_seconds: float = 300.0):
        self._llm = llm_client
        self._tool_registry = tool_registry
        self._max_steps = max_steps
        self._timeout = timeout_seconds

    async def run(self, user_input: str, *, run_id: str | None = None) -> ExecutionResult:
        start_time = datetime.now()
        steps = []
        for step_num in range(1, self._max_steps + 1):
            # 检查超时
            if (datetime.now() - start_time).total_seconds() > self._timeout:
                return ExecutionResult(ExecutionStatus.TIMEOUT, None, steps,
                                       (datetime.now() - start_time).total_seconds() * 1000,
                                       "Execution timeout")
            # 调用 LLM
            llm_response = await self._llm.generate(user_input)  # 简化示例
            thought, action, action_input, is_final = self._parse_response(llm_response)

            step = ThinkingStep(step_num, thought, action, action_input, None, datetime.now())
            if is_final:
                steps.append(step)
                return ExecutionResult(ExecutionStatus.COMPLETED, thought, steps,
                                       (datetime.now() - start_time).total_seconds() * 1000)
            if action:
                try:
                    observation = await self._tool_registry.invoke_tool(action, action_input or {}, run_id=run_id)
                    step.observation = str(observation)
                except Exception as e:
                    step.observation = f"Error: {e}"
            steps.append(step)

        return ExecutionResult(ExecutionStatus.MAX_STEPS_REACHED, None, steps,
                               (datetime.now() - start_time).total_seconds() * 1000,
                               f"Max steps ({self._max_steps}) reached")

    def _parse_response(self, response: str) -> tuple[str, str | None, dict | None, bool]:
        if "Final Answer:" in response:
            return response.split("Final Answer:")[-1].strip(), None, None, True
        thought = action = None
        action_input = {}
        if "Thought:" in response:
            thought = response.split("Thought:")[-1].split("Action:")[0].strip()
        if "Action:" in response:
            action = response.split("Action:")[-1].split("Action Input:")[0].strip()
        return thought or "", action, action_input, False
```

---

### 4.5 Step 5: 测试实现

| 任务 ID | 任务描述                                 | 验收标准             |
| :------ | :--------------------------------------- | :------------------- |
| P4-3-1  | 编写 `PostgresSessionService` 单元测试   | 覆盖所有接口方法     |
| P4-3-2  | 编写 `PostgresMemoryService` 单元测试    | 覆盖所有接口方法     |
| P4-3-3  | 跑通 ADK 官方 Interface Compliance Tests | 官方测试通过         |
| P4-3-4  | 编写 E2E 集成测试                        | 完整对话流程测试通过 |

#### 4.5.1 SessionService 单元测试 (P4-3-1)

创建 `tests/unittests/mind/test_session_service.py`:

```python
"""
PostgresSessionService 单元测试
覆盖 ADK BaseSessionService 接口所有方法
"""

import pytest
import asyncio
from datetime import datetime
from cognizes.adapters.postgres.session_service import PostgresSessionService, Session, Event

# pytest-asyncio 配置
pytestmark = pytest.mark.asyncio


class TestPostgresSessionService:
    """SessionService 单元测试套件"""

    @pytest.fixture
    async def service(self, db_pool):
        """创建测试服务实例"""
        return PostgresSessionService(pool=db_pool)

    @pytest.fixture
    async def db_pool(self):
        """创建测试数据库连接池"""
        import asyncpg
        pool = await asyncpg.create_pool(
            "postgresql://test:test@localhost:5432/test_db",
            min_size=1, max_size=5
        )
        yield pool
        await pool.close()

    # ========== create_session 测试 ==========

    async def test_create_session_basic(self, service):
        """测试基础会话创建"""
        session = await service.create_session(
            app_name="test_app",
            user_id="user_001"
        )
        assert session.id is not None
        assert session.app_name == "test_app"
        assert session.user_id == "user_001"
        assert session.state == {}
        assert session.events == []

    async def test_create_session_with_initial_state(self, service):
        """测试带初始状态的会话创建"""
        initial_state = {"user:language": "zh-CN", "app:theme": "dark"}
        session = await service.create_session(
            app_name="test_app",
            user_id="user_002",
            state=initial_state
        )
        assert session.state["user:language"] == "zh-CN"
        assert session.state["app:theme"] == "dark"

    async def test_create_session_with_custom_id(self, service):
        """测试自定义会话 ID"""
        custom_id = "custom-session-123"
        session = await service.create_session(
            app_name="test_app",
            user_id="user_003",
            session_id=custom_id
        )
        assert session.id == custom_id

    # ========== get_session 测试 ==========

    async def test_get_session_exists(self, service):
        """测试获取已存在的会话"""
        created = await service.create_session(
            app_name="test_app", user_id="user_004"
        )
        retrieved = await service.get_session(
            app_name="test_app",
            user_id="user_004",
            session_id=created.id
        )
        assert retrieved is not None
        assert retrieved.id == created.id

    async def test_get_session_not_found(self, service):
        """测试获取不存在的会话"""
        session = await service.get_session(
            app_name="test_app",
            user_id="user_005",
            session_id="non-existent-id"
        )
        assert session is None

    async def test_get_session_with_config(self, service):
        """测试带配置的会话获取 (分页)"""
        from cognizes.adapters.postgres.session_service import GetSessionConfig

        session = await service.create_session(
            app_name="test_app", user_id="user_006"
        )
        # 添加多个事件
        for i in range(10):
            await service.append_event(session, Event(
                id=f"event_{i}", author="user", content={"msg": f"hello {i}"}
            ))

        # 仅获取最后 3 条事件
        config = GetSessionConfig(num_recent_events=3)
        result = await service.get_session(
            app_name="test_app",
            user_id="user_006",
            session_id=session.id,
            config=config
        )
        assert len(result.events) == 3

    # ========== list_sessions 测试 ==========

    async def test_list_sessions_by_user(self, service):
        """测试列出用户的所有会话"""
        # 创建多个会话
        for i in range(3):
            await service.create_session(
                app_name="test_app", user_id="user_007"
            )
        response = await service.list_sessions(
            app_name="test_app", user_id="user_007"
        )
        assert len(response.sessions) >= 3

    async def test_list_sessions_all_users(self, service):
        """测试列出所有用户会话 (admin)"""
        response = await service.list_sessions(
            app_name="test_app", user_id=None
        )
        assert isinstance(response.sessions, list)

    # ========== delete_session 测试 ==========

    async def test_delete_session(self, service):
        """测试删除会话"""
        session = await service.create_session(
            app_name="test_app", user_id="user_008"
        )
        await service.delete_session(
            app_name="test_app",
            user_id="user_008",
            session_id=session.id
        )
        # 验证已删除
        deleted = await service.get_session(
            app_name="test_app",
            user_id="user_008",
            session_id=session.id
        )
        assert deleted is None

    # ========== State 前缀处理测试 ==========

    async def test_state_prefix_user_scope(self, service):
        """测试 user: 前缀 - 跨会话持久"""
        session1 = await service.create_session(
            app_name="test_app", user_id="user_009"
        )
        # 设置 user 级别状态
        await service.append_event(session1, Event(
            id="e1", author="agent",
            actions={"state_delta": {"user:preference": "dark_mode"}}
        ))
        # 新会话应继承 user: 状态
        session2 = await service.create_session(
            app_name="test_app", user_id="user_009"
        )
        assert session2.state.get("user:preference") == "dark_mode"

    async def test_state_prefix_temp_not_persisted(self, service):
        """测试 temp: 前缀 - 不持久化"""
        session = await service.create_session(
            app_name="test_app", user_id="user_010"
        )
        await service.append_event(session, Event(
            id="e1", author="agent",
            actions={"state_delta": {"temp:cache": "value", "app:config": "saved"}}
        ))
        # temp: 不应被持久化
        reloaded = await service.get_session(
            app_name="test_app",
            user_id="user_010",
            session_id=session.id
        )
        assert "temp:cache" not in reloaded.state
        assert reloaded.state.get("app:config") == "saved"
```

#### 4.5.2 ADK LlmAgent 集成示例 (P4-2-23)

创建 `tests/integration/mind/test_adk_llmagent.py`:

> [!IMPORTANT]
>
> **核心验收**: 验证 `PostgresSessionService` 与 ADK `LlmAgent` + `Runner` 的完整协同。

```python
"""
ADK Runner 集成示例
演示 PostgresSessionService 与 Google ADK LlmAgent 的协同工作
"""

import asyncio
from google.adk.agents import LlmAgent
from google.adk.runners import Runner
from google.genai import types

from cognizes.adapters.postgres.session_service import PostgresSessionService
from cognizes.adapters.postgres.memory_service import PostgresMemoryService


async def run_agent_with_postgres():
    """使用 PostgreSQL 后端运行 ADK Agent"""

    # 1. 初始化 PostgreSQL 服务
    import asyncpg
    pool = await asyncpg.create_pool("postgresql://user:pass@localhost/agent_db")

    session_service = PostgresSessionService(pool=pool)
    memory_service = PostgresMemoryService(pool=pool)

    # 2. 定义 Agent
    agent = LlmAgent(
        name="travel_assistant",
        model="gemini-2.0-flash",
        instruction="""
        You are a helpful travel assistant.
        Remember user preferences from past conversations.
        Use the search_flights tool to find flights.
        """,
        tools=[search_flights],  # 注册 Function Tool
    )

    # 3. 创建 Runner (核心集成点)
    runner = Runner(
        agent=agent,
        app_name="travel_app",
        session_service=session_service,   # 关键: 注入 PostgreSQL Session
        memory_service=memory_service,      # 关键: 注入 PostgreSQL Memory
    )

    # 4. 创建会话并执行
    session = await session_service.create_session(
        app_name="travel_app",
        user_id="user_123",
        state={"user:language": "zh-CN"}  # 初始用户偏好
    )

    # 5. 运行 Agent (InvocationContext 自动管理)
    user_message = types.Content(
        role="user",
        parts=[types.Part(text="帮我查一下明天北京到上海的航班")]
    )

    async for event in runner.run_async(
        session_id=session.id,
        user_id="user_123",
        new_message=user_message
    ):
        # 处理流式事件
        if event.is_final_response():
            print(f"Agent 回复: {event.text}")
        elif event.get_function_calls():
            print(f"工具调用: {event.get_function_calls()}")

    # 6. 验证状态持久化
    updated_session = await session_service.get_session(
        app_name="travel_app",
        user_id="user_123",
        session_id=session.id
    )
    print(f"会话事件数: {len(updated_session.events)}")
    print(f"会话状态: {updated_session.state}")

    # 7. 可选: 将会话存入长期记忆
    await memory_service.add_session_to_memory(updated_session)

    await pool.close()


# Function Tool 示例
def search_flights(origin: str, destination: str, date: str) -> dict:
    """搜索航班的工具函数"""
    return {
        "flights": [
            {"number": "CA1234", "departure": "08:00", "price": 680},
            {"number": "MU5678", "departure": "10:30", "price": 720},
        ]
    }


if __name__ == "__main__":
    asyncio.run(run_agent_with_postgres())
```

#### 4.5.3 E2E 集成测试 (P4-3-4)

创建 `tests/integration/mind/test_e2e.py`:

```python
"""
E2E 集成测试 - 完整对话流程
验证 Session -> Agent -> Tool -> Memory 全链路
"""

import pytest
from cognizes.adapters.postgres.session_service import PostgresSessionService
from cognizes.adapters.postgres.memory_service import PostgresMemoryService
from cognizes.adapters.postgres.tool_registry import ToolRegistry

pytestmark = pytest.mark.asyncio


class TestE2EIntegration:
    """端到端集成测试"""

    async def test_complete_conversation_flow(self, db_pool):
        """测试完整对话流程"""
        session_svc = PostgresSessionService(pool=db_pool)
        memory_svc = PostgresMemoryService(pool=db_pool)
        tool_registry = ToolRegistry(pool=db_pool, app_name="test_app")

        # 1. 创建会话
        session = await session_svc.create_session(
            app_name="test_app", user_id="e2e_user"
        )

        # 2. 注册工具
        await tool_registry.register_tool(
            name="calculator",
            func=lambda x, y: x + y,
            openapi_schema={"type": "function", "name": "calculator"}
        )

        # 3. 模拟多轮对话
        for turn in range(3):
            # 用户输入
            await session_svc.append_event(session, {
                "id": f"user_{turn}",
                "author": "user",
                "content": {"text": f"Turn {turn} message"}
            })
            # Agent 响应
            await session_svc.append_event(session, {
                "id": f"agent_{turn}",
                "author": "agent",
                "content": {"text": f"Response to turn {turn}"},
                "actions": {"state_delta": {f"app:turn_{turn}": True}}
            })

        # 4. 验证事件记录
        final_session = await session_svc.get_session(
            app_name="test_app",
            user_id="e2e_user",
            session_id=session.id
        )
        assert len(final_session.events) == 6  # 3轮 * 2

        # 5. 存入长期记忆
        await memory_svc.add_session_to_memory(final_session)

        # 6. 验证记忆可搜索
        memories = await memory_svc.search_memory(
            app_name="test_app",
            user_id="e2e_user",
            query="Turn 2 message"
        )
        assert len(memories.memories) > 0

    async def test_cross_session_memory_recall(self, db_pool):
        """测试跨会话记忆召回"""
        session_svc = PostgresSessionService(pool=db_pool)
        memory_svc = PostgresMemoryService(pool=db_pool)

        # 会话 1: 记录偏好
        session1 = await session_svc.create_session(
            app_name="test_app", user_id="memory_user"
        )
        await session_svc.append_event(session1, {
            "id": "pref_1", "author": "user",
            "content": {"text": "I prefer window seats on flights"}
        })
        await memory_svc.add_session_to_memory(session1)

        # 会话 2: 验证记忆召回
        memories = await memory_svc.search_memory(
            app_name="test_app",
            user_id="memory_user",
            query="flight seat preference"
        )
        assert any("window" in m.content.lower() for m in memories.memories)
```

---

### 4.6 Step 6: OpenTelemetry 集成

#### 4.6.1 任务清单

| 任务 ID | 任务描述                      | 验收标准                              |
| :------ | :---------------------------- | :------------------------------------ |
| P4-4-1  | 安装 OpenTelemetry Python SDK | 依赖安装成功                          |
| P4-4-2  | 在 Adapter 层添加 Span 埋点   | `Chain start/end`, `Tool call/return` |
| P4-4-3  | 配置 Trace Exporter           | Trace 数据导出成功                    |
| P4-4-4  | 设计 `traces` 表结构化存储    | DDL + 结构定义                        |

#### 4.6.2 导出策略设计

> [!IMPORTANT]
>
> **双路导出策略**：同时将 Trace 数据发送到两个目的地，满足不同需求。
>
> | 导出目标            | 用途                        | 数据保留   |
> | :------------------ | :-------------------------- | :--------- |
> | **PostgreSQL**      | 审计、历史查询、合规留存    | 按策略归档 |
> | **OTLP (Langfuse)** | 实时可视化调试 + Evaluation | 按项目配置 |

#### 4.6.3 核心实现

创建 `src/cognizes/adapters/postgres/tracing.py`:

```python
"""
OpenTelemetry 双路导出集成

架构:
┌─────────────┐     ┌──────────────────┐
│ Agent       │────▶│ TracingManager   │
│ Executor    │     │                  │
└─────────────┘     └─────────┬────────┘
                              │
              ┌───────────────┼───────────────┐
              ▼               ▼               ▼
    ┌─────────────┐  ┌─────────────┐  ┌─────────────┐
    │ PostgreSQL  │  │ OTLP/Langfuse│  │ Console     │
    │ (审计留存)  │  │ (实时调试)  │  │ (开发环境)  │
    └─────────────┘  └─────────────┘  └─────────────┘
"""

import json
import uuid
from datetime import datetime
from contextlib import asynccontextmanager
from typing import Any

import asyncpg
from opentelemetry import trace
from opentelemetry.sdk.trace import TracerProvider, ReadableSpan
from opentelemetry.sdk.trace.export import (
    BatchSpanProcessor,
    SpanExporter,
    SpanExportResult,
    ConsoleSpanExporter,
)
from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import OTLPSpanExporter
from opentelemetry.trace import Status, StatusCode


class PostgresSpanExporter(SpanExporter):
    """将 Span 持久化到 PostgreSQL traces 表"""

    def __init__(self, pool: asyncpg.Pool):
        self._pool = pool

    def export(self, spans: list[ReadableSpan]) -> SpanExportResult:
        """同步导出 (内部异步)"""
        import asyncio
        try:
            asyncio.get_event_loop().run_until_complete(self._async_export(spans))
            return SpanExportResult.SUCCESS
        except Exception:
            return SpanExportResult.FAILURE

    async def _async_export(self, spans: list[ReadableSpan]) -> None:
        async with self._pool.acquire() as conn:
            for span in spans:
                await conn.execute(
                    """
                    INSERT INTO traces
                    (trace_id, span_id, parent_span_id, operation_name, span_kind,
                     attributes, events, start_time, end_time, duration_ns,
                     status_code, status_message)
                    VALUES ($1, $2, $3, $4, $5, $6, $7, $8, $9, $10, $11, $12)
                    """,
                    format(span.context.trace_id, '032x'),
                    format(span.context.span_id, '016x'),
                    format(span.parent.span_id, '016x') if span.parent else None,
                    span.name,
                    span.kind.name if span.kind else 'INTERNAL',
                    json.dumps(dict(span.attributes or {})),
                    json.dumps([self._event_to_dict(e) for e in (span.events or [])]),
                    datetime.fromtimestamp(span.start_time / 1e9),
                    datetime.fromtimestamp(span.end_time / 1e9) if span.end_time else None,
                    (span.end_time - span.start_time) if span.end_time else None,
                    span.status.status_code.name if span.status else 'UNSET',
                    span.status.description if span.status else None,
                )

    def _event_to_dict(self, event) -> dict:
        return {"name": event.name, "timestamp": event.timestamp,
                "attributes": dict(event.attributes or {})}

    def shutdown(self) -> None:
        pass


class TracingManager:
    """
    Trace 管理器 - 支持双路导出

    使用方式:
        tracing = TracingManager(pg_pool=pool, otlp_endpoint="localhost:4317")

        @tracing.trace_tool_call("calculator")
        async def calculate(x, y):
            return x + y
    """

    def __init__(
        self,
        service_name: str = "open-agent-engine",
        pg_pool: asyncpg.Pool | None = None,
        otlp_endpoint: str | None = None,
        console_export: bool = False,
        otlp_exporter: SpanExporter | None = None,  # For testing
    ):
        provider = TracerProvider()

        # 双路导出配置
        if pg_pool:
            # PostgreSQL: 持久化审计
            provider.add_span_processor(
                BatchSpanProcessor(PostgresSpanExporter(pg_pool))
            )

        # OTLP: 实时可视化 (Langfuse)
        if otlp_exporter:
            # 优先使用注入的 Exporter (测试用)
            provider.add_span_processor(
                BatchSpanProcessor(otlp_exporter)
            )
        elif otlp_endpoint:
             provider.add_span_processor(
                BatchSpanProcessor(OTLPSpanExporter(endpoint=otlp_endpoint, insecure=True))
            )

        if console_export:
            # Console: 开发调试
            provider.add_span_processor(
                BatchSpanProcessor(ConsoleSpanExporter())
            )

        trace.set_tracer_provider(provider)
        self.provider = provider
        self._tracer = self.provider.get_tracer(service_name)

    @asynccontextmanager
    async def span(self, name: str, *, attributes: dict[str, Any] | None = None):
        """创建 Span 上下文"""
        with self._tracer.start_as_current_span(name) as span:
            if attributes:
                for k, v in attributes.items():
                    span.set_attribute(k, str(v))
            try:
                yield span
                span.set_status(Status(StatusCode.OK))
            except Exception as e:
                span.set_status(Status(StatusCode.ERROR, str(e)))
                span.record_exception(e)
                raise

    def trace_tool_call(self, tool_name: str):
        """工具调用追踪装饰器"""
        def decorator(func):
            async def wrapper(*args, **kwargs):
                async with self.span(f"tool.{tool_name}", attributes={"tool.name": tool_name}) as span:
                    result = await func(*args, **kwargs)
                    span.set_attribute("tool.result_size", len(str(result)))
                    return result
            return wrapper
        return decorator

    def trace_llm_call(self, model: str):
        """LLM 调用追踪装饰器"""
        def decorator(func):
            async def wrapper(*args, **kwargs):
                async with self.span(f"llm.generate", attributes={"llm.model": model}) as span:
                    result = await func(*args, **kwargs)
                    span.set_attribute("llm.response_length", len(str(result)))
                    return result
            return wrapper
        return decorator
```

---

### 4.7 Step 7: Sandboxed Execution 实现

#### 4.7.1 任务清单

| 任务 ID | 任务描述                                     | 验收标准                    |
| :------ | :------------------------------------------- | :-------------------------- |
| P4-4-5  | 评估沙箱方案 (Docker / gVisor / WebAssembly) | 方案对比分析文档            |
| P4-4-6  | 实现 `SandboxRunner` 接口抽象                | 接口定义 + 基础实现         |
| P4-4-7  | 实现 Docker 沙箱执行                         | Python 代码在隔离容器中执行 |
| P4-4-8  | 实现资源限制 (CPU/Memory/Timeout)            | 资源超限时强制终止          |
| P4-4-9  | 实现网络隔离策略                             | 沙箱无法访问外部网络        |
| P4-4-10 | 验证恶意代码防护                             | 恶意操作被拦截              |

#### 4.7.2 沙箱方案对比

> [!NOTE]
>
> **核心考量**：Docker 容器级隔离共享内核，存在逃逸风险；启动时间秒级影响交互体验。推荐使用基于 microVM 的轻量级沙箱。

| 方案                    | 隔离级别 | 启动速度 | 内核独立 | MCP 支持 | 推荐场景                           |
| :---------------------- | :------- | :------- | :------- | :------- | :--------------------------------- |
| **microsandbox** (推荐) | VM 级    | <200ms   | ✅       | ✅ 原生  | **首选** - 真正 VM 隔离 + 快速启动 |
| Docker                  | 容器级   | 秒级     | ❌       | ❌       | 成熟环境 (OCI 生态)                |
| gVisor                  | 系统调用 | 秒级     | ❌       | ❌       | 高安全建设无 VM 能力时             |
| WebAssembly             | 语言级   | 毫秒级   | -        | ❌       | 纯计算轻量场景                     |

**microsandbox 核心优势** ([zerocore-ai/microsandbox](https://github.com/zerocore-ai/microsandbox), 4.3k+ stars):

- **真正 VM 隔离**：基于 libkrun 的 microVM，独立内核，完全隔离宿主机
- **极速启动**：<200ms 冷启动 (Docker 秒级)
- **OCI 兼容**：支持标准容器镜像
- **MCP 原生集成**：内置 MCP Server，可直接与 AI Agent 对接

#### 4.7.3 SandboxRunner 抽象接口 (P4-4-6)

> [!NOTE]
>
> **设计原则**: 定义统一的沙箱接口，支持多种后端实现 (microsandbox/Docker/WebAssembly)。

创建 `src/cognizes/adapters/postgres/sandbox/base.py`:

```python
"""
SandboxRunner 抽象基类
支持多种沙箱后端的统一接口
"""

from abc import ABC, abstractmethod
from dataclasses import dataclass
from enum import Enum
from typing import Any


class SandboxBackend(Enum):
    """沙箱后端类型"""
    MICROSANDBOX = "microsandbox"  # 推荐: microVM 隔离
    DOCKER = "docker"              # 备选: 容器隔离
    WASM = "wasm"                   # 轻量: WebAssembly


@dataclass
class SandboxConfig:
    """沙箱配置 - 通用参数"""
    name: str = "agent-sandbox"
    image: str = "python:3.11-slim"
    memory_mb: int = 256
    cpu_cores: float = 0.5
    timeout_seconds: int = 30
    network_enabled: bool = False
    allow_file_access: bool = False


@dataclass
class SandboxResult:
    """沙箱执行结果"""
    success: bool
    stdout: str
    stderr: str
    exit_code: int
    execution_time_ms: float
    metadata: dict = None  # 额外信息 (如资源使用)


class BaseSandboxRunner(ABC):
    """
    沙箱执行器抽象基类

    使用方式:
        runner = MicrosandboxRunner(config)
        result = await runner.execute("print('Hello!')")
    """

    def __init__(self, config: SandboxConfig | None = None):
        self._config = config or SandboxConfig()

    @property
    @abstractmethod
    def backend(self) -> SandboxBackend:
        """返回后端类型"""
        pass

    @abstractmethod
    async def execute(self, code: str) -> SandboxResult:
        """
        执行代码

        Args:
            code: 要执行的代码字符串

        Returns:
            SandboxResult: 执行结果
        """
        pass

    @abstractmethod
    async def execute_file(self, file_path: str) -> SandboxResult:
        """执行文件"""
        pass

    async def execute_safe(self, code: str) -> SandboxResult:
        """
        带预检查的安全执行

        在执行前进行静态分析，拦截危险代码模式
        """
        danger_patterns = [
            'os.system', 'subprocess', '__import__',
            'eval(', 'exec(', 'open(',
            'import socket', 'import requests',
        ]
        for pattern in danger_patterns:
            if pattern in code:
                return SandboxResult(
                    success=False,
                    stdout="",
                    stderr=f"Security violation: '{pattern}' is not allowed",
                    exit_code=-2,
                    execution_time_ms=0
                )
        return await self.execute(code)

    async def health_check(self) -> bool:
        """检查沙箱服务是否可用"""
        try:
            result = await self.execute("print('health')")
            return result.success and "health" in result.stdout
        except Exception:
            return False

    async def cleanup(self) -> None:
        """清理资源 (子类可覆写)"""
        pass
```

**工厂函数**:

创建 `src/cognizes/adapters/postgres/sandbox/__init__.py`:

```python
def create_sandbox_runner(
    backend: SandboxBackend = SandboxBackend.MICROSANDBOX,
    config: SandboxConfig | None = None
) -> BaseSandboxRunner:
    """
    创建沙箱执行器的工厂函数

    Args:
        backend: 沙箱后端类型
        config: 可选配置

    Returns:
        BaseSandboxRunner 实现
    """
    if backend == SandboxBackend.MICROSANDBOX:
        from .microsandbox_runner import MicrosandboxRunner
        return MicrosandboxRunner(config)
    elif backend == SandboxBackend.DOCKER:
        from .docker_runner import DockerSandboxRunner
        return DockerSandboxRunner(config)
    else:
        raise ValueError(f"Unsupported backend: {backend}")
```

#### 4.7.4 核心实现

**安装 microsandbox**:

```bash
# 安装 microsandbox CLI
curl -sSL https://get.microsandbox.dev | sh

# 启动 microsandbox server (开发模式)
msb server start --dev
```

**Python 实现** (`src/cognizes/adapters/postgres/sandbox/microsandbox_runner.py`):

```python
"""
MicrosandboxRunner: 基于 microsandbox 的安全沙箱

核心特性:
- microVM 级别隔离 (独立内核)
- <200ms 冷启动
- MCP 原生支持
- OCI 镜像兼容
"""

import asyncio
from dataclasses import dataclass
from microsandbox import PythonSandbox  # pip install microsandbox

@dataclass
class SandboxConfig:
    name: str = "agent-sandbox"
    image: str = "python:3.11-slim"  # OCI 镜像
    memory_mb: int = 256
    timeout_seconds: int = 30
    network_enabled: bool = False

@dataclass
class SandboxResult:
    success: bool
    stdout: str
    stderr: str
    exit_code: int
    execution_time_ms: float

class MicrosandboxRunner:
    """基于 microsandbox 的轻量级沙箱执行器"""

    def __init__(self, config: SandboxConfig | None = None):
        self._config = config or SandboxConfig()

    async def execute(self, code: str) -> SandboxResult:
        """在 microVM 中安全执行代码"""
        import time
        start = time.time()

        try:
            async with PythonSandbox.create(
                name=self._config.name,
                image=self._config.image,
                memory=self._config.memory_mb,
                network=self._config.network_enabled
            ) as sandbox:
                # 执行代码
                execution = await asyncio.wait_for(
                    sandbox.run(code),
                    timeout=self._config.timeout_seconds
                )

                stdout = await execution.output()
                stderr = await execution.errors() if hasattr(execution, 'errors') else ""
                exit_code = execution.exit_code if hasattr(execution, 'exit_code') else 0

                return SandboxResult(
                    success=(exit_code == 0),
                    stdout=stdout,
                    stderr=stderr,
                    exit_code=exit_code,
                    execution_time_ms=(time.time() - start) * 1000
                )

        except asyncio.TimeoutError:
            return SandboxResult(
                success=False,
                stdout="",
                stderr=f"Execution timeout ({self._config.timeout_seconds}s)",
                exit_code=-1,
                execution_time_ms=(time.time() - start) * 1000
            )
        except Exception as e:
            return SandboxResult(
                success=False,
                stdout="",
                stderr=str(e),
                exit_code=-1,
                execution_time_ms=(time.time() - start) * 1000
            )

    async def execute_safe(self, code: str) -> SandboxResult:
        """带基础安全检查的执行"""
        # 过滤危险模式
        dangerous = ['os.system', 'subprocess', '__import__', 'eval(', 'exec(']
        for pattern in dangerous:
            if pattern in code:
                return SandboxResult(
                    success=False, stdout="", stderr=f"Blocked: {pattern}",
                    exit_code=-2, execution_time_ms=0
                )
        return await self.execute(code)
```

**MCP 集成示例** (直接与 AI Agent 对接):

创建 `src/cognizes/adapters/postgres/sandbox/mcp.py`:

```python
# microsandbox 内置 MCP Server，可直接作为 Tool 调用
from mcp import ClientSession, StdioServerParameters

async def create_mcp_sandbox_tool():
    """Agent 通过 MCP 调用沙箱"""
    server_params = StdioServerParameters(
        command="msb",
        args=["mcp", "serve"],
        env={}
    )
    return server_params  # 注册到 ToolRegistry
```

---

### 4.8 Step 8: 可视化验证 (Langfuse)

> [!NOTE]
>
> **工具变更**: 采用 **Langfuse** (基于 OTLP) 实现全链路追踪，以获得更强的 LLM Agent 调试能力（Prompt Management, Model Evaluation）。

| 任务 ID | 任务描述                | 验收标准                                      |
| :------ | :---------------------- | :-------------------------------------------- |
| P4-4-11 | 部署 Langfuse 服务      | UI 可访问 (localhost:3000)                    |
| P4-4-12 | 验证完整 Trace 链路还原 | `User Input -> Reasoning -> Action -> Answer` |
| P4-4-13 | 验证调试能力            | 可通过 Trace/Session 视图定位问题             |

**Langfuse 部署命令** (Localhost):

```bash
# 1. 获取官方 Docker Compose 配置
# curl -L https://github.com/langfuse/langfuse/blob/main/docker-compose.yml -o assets/docker-compose.yml

# 2. 启动服务 (包含了 Postgres/ClickHouse 依赖)
docker compose up -d

# 3. 访问 http://localhost:3000 并创建账号获取 Keys
```

**环境变量配置** (用于 `tracing.py`):

```bash
export LANGFUSE_SECRET_KEY=sk-lf-...  # 从 UI 获取
export LANGFUSE_PUBLIC_KEY=pk-lf-...  # 从 UI 获取
export LANGFUSE_HOST=http://localhost:3000
```

---

### 4.9 Step 9: AG-UI 协议集成

> [!NOTE]
>
> **对标 Roadmap 4.4**: AG-UI 作为 Agent-User 交互的标准化协议层，增强 The Realm of Mind 的前端交互能力。
>
> **参考资源**:
>
> - [AG-UI 协议调研](../research/070-ag-ui.md)
> - [AG-UI 官方文档](https://docs.ag-ui.com/)

#### 4.9.1 AG-UI 核心概念

AG-UI 是连接 AI Agent 与用户界面的"**最后一公里**"协议，定义了 16 种标准事件类型：

| 事件类别         | 事件类型                     | 描述         | Pulse 对标           |
| :--------------- | :--------------------------- | :----------- | :------------------- |
| **Lifecycle**    | `RUN_STARTED/FINISHED/ERROR` | 运行生命周期 | `runs` 表状态        |
| **Text Message** | `TEXT_MESSAGE_*`             | 文本消息流   | `events` 表          |
| **Tool Call**    | `TOOL_CALL_*`                | 工具调用     | `tool_executions` 表 |
| **State**        | `STATE_SNAPSHOT/DELTA`       | 状态同步     | `threads.state`      |

#### 4.9.2 任务清单

| 任务 ID | 任务描述                     | 状态      | 验收标准           |
| :------ | :--------------------------- | :-------- | :----------------- |
| P4-5-1  | 阅读 AG-UI 官方文档          | 🔲 待开始 | 协议理解笔记       |
| P4-5-2  | 分析事件类型与 Pulse 对齐    | 🔲 待开始 | 事件映射表         |
| P4-5-3  | 分析前端定义工具机制         | 🔲 待开始 | 工具集成方案       |
| P4-5-4  | 实现 `AgUiEventEmitter`      | 🔲 待开始 | 发射 16 种事件     |
| P4-5-5  | 集成前端工具到 Tool Registry | 🔲 待开始 | 前端工具可热更新   |
| P4-5-6  | 实现 STATE_DELTA 状态同步    | 🔲 待开始 | JSON Patch 正确    |
| P4-5-7  | 实现 Human-in-the-Loop       | 🔲 待开始 | confirmAction 正常 |
| P4-5-8  | 创建 CopilotKit React 项目   | 🔲 待开始 | 项目可运行         |
| P4-5-9  | 配置 AG-UI 服务端点          | 🔲 待开始 | 连接成功           |
| P4-5-10 | 实现自定义工具渲染器         | 🔲 待开始 | 工具 UI 正常       |

#### 4.9.3 AgUiEventEmitter 实现

创建 `src/cognizes/engine/agui/event_emitter.py`：

```python
"""
AG-UI 事件发射器
将 Agent 执行事件转换为 AG-UI 标准事件格式
"""

from enum import Enum
from dataclasses import dataclass, field
from typing import Any, Optional
import json
import time


class EventType(str, Enum):
    """AG-UI 16 种标准事件类型"""
    # Lifecycle Events
    RUN_STARTED = "RUN_STARTED"
    RUN_FINISHED = "RUN_FINISHED"
    RUN_ERROR = "RUN_ERROR"
    STEP_STARTED = "STEP_STARTED"
    STEP_FINISHED = "STEP_FINISHED"

    # Text Message Events
    TEXT_MESSAGE_START = "TEXT_MESSAGE_START"
    TEXT_MESSAGE_CONTENT = "TEXT_MESSAGE_CONTENT"
    TEXT_MESSAGE_END = "TEXT_MESSAGE_END"

    # Tool Call Events
    TOOL_CALL_START = "TOOL_CALL_START"
    TOOL_CALL_ARGS = "TOOL_CALL_ARGS"
    TOOL_CALL_END = "TOOL_CALL_END"

    # State Management Events
    STATE_SNAPSHOT = "STATE_SNAPSHOT"
    STATE_DELTA = "STATE_DELTA"
    MESSAGES_SNAPSHOT = "MESSAGES_SNAPSHOT"

    # Special Events
    RAW = "RAW"
    CUSTOM = "CUSTOM"


@dataclass
class BaseEvent:
    """AG-UI 基础事件"""
    type: EventType
    timestamp: float = field(default_factory=time.time)
    run_id: Optional[str] = None


@dataclass
class RunStartedEvent(BaseEvent):
    """运行开始事件"""
    type: EventType = EventType.RUN_STARTED


@dataclass
class TextMessageContentEvent(BaseEvent):
    """文本消息内容事件 (流式)"""
    type: EventType = EventType.TEXT_MESSAGE_CONTENT
    message_id: str = ""
    delta: str = ""  # 增量文本内容


@dataclass
class ToolCallStartEvent(BaseEvent):
    """工具调用开始事件"""
    type: EventType = EventType.TOOL_CALL_START
    tool_call_id: str = ""
    tool_call_name: str = ""


@dataclass
class StateDeltaEvent(BaseEvent):
    """状态增量事件"""
    type: EventType = EventType.STATE_DELTA
    delta: list = field(default_factory=list)  # JSON Patch 操作列表


class AgUiEventEmitter:
    """AG-UI 事件发射器"""

    def __init__(self, run_id: str):
        self.run_id = run_id
        self._event_buffer: list[BaseEvent] = []

    def emit_run_started(self) -> RunStartedEvent:
        """发射运行开始事件"""
        event = RunStartedEvent(run_id=self.run_id)
        self._event_buffer.append(event)
        return event

    def emit_text_content(self, message_id: str, delta: str) -> TextMessageContentEvent:
        """发射文本内容增量事件"""
        event = TextMessageContentEvent(
            run_id=self.run_id,
            message_id=message_id,
            delta=delta
        )
        self._event_buffer.append(event)
        return event

    def emit_tool_call_start(self, tool_call_id: str, name: str) -> ToolCallStartEvent:
        """发射工具调用开始事件"""
        event = ToolCallStartEvent(
            run_id=self.run_id,
            tool_call_id=tool_call_id,
            tool_call_name=name
        )
        self._event_buffer.append(event)
        return event

    def emit_state_delta(self, delta_operations: list[dict]) -> StateDeltaEvent:
        """发射状态增量事件 (JSON Patch)"""
        event = StateDeltaEvent(
            run_id=self.run_id,
            delta=delta_operations
        )
        self._event_buffer.append(event)
        return event

    def to_sse(self) -> str:
        """将事件缓冲区转换为 SSE 格式"""
        lines = []
        for event in self._event_buffer:
            data = json.dumps({
                "type": event.type.value,
                "timestamp": event.timestamp,
                "run_id": event.run_id,
                **{k: v for k, v in event.__dict__.items()
                   if k not in ("type", "timestamp", "run_id")}
            })
            lines.append(f"data: {data}\n\n")
        return "".join(lines)
```

#### 4.9.4 前端工具集成

扩展 Tool Registry 支持前端定义工具：

```python
# 在 tool_registry.py 中扩展

@dataclass
class FrontendTool:
    """前端定义工具"""
    name: str
    description: str
    parameters: dict  # JSON Schema
    render_component: str  # React 组件名称
    requires_confirmation: bool = False  # Human-in-the-Loop

class ToolRegistry:
    """扩展的工具注册表"""

    def __init__(self, pool: asyncpg.Pool):
        self._pool = pool
        self._frontend_tools: dict[str, FrontendTool] = {}

    async def register_frontend_tool(
        self,
        app_name: str,
        tool: FrontendTool
    ) -> None:
        """注册前端定义工具"""
        self._frontend_tools[f"{app_name}:{tool.name}"] = tool

        # 同时持久化到数据库
        await self._pool.execute("""
            INSERT INTO tools (app_name, name, description, openapi_schema, permissions)
            VALUES ($1, $2, $3, $4, $5)
            ON CONFLICT (app_name, name) DO UPDATE
            SET description = EXCLUDED.description,
                openapi_schema = EXCLUDED.openapi_schema,
                updated_at = NOW()
        """, app_name, tool.name, tool.description,
            json.dumps(tool.parameters),
            json.dumps({"requires_confirmation": tool.requires_confirmation}))

    def get_frontend_tools(self, app_name: str) -> list[FrontendTool]:
        """获取应用的前端工具列表"""
        return [
            tool for key, tool in self._frontend_tools.items()
            if key.startswith(f"{app_name}:")
        ]
```

#### 4.9.5 Human-in-the-Loop 实现

```python
# src/cognizes/engine/agui/hitl.py
"""Human-in-the-Loop 工具实现"""

from dataclasses import dataclass
from typing import Any
import asyncio


@dataclass
class ConfirmationRequest:
    """确认请求"""
    action: str
    importance: str  # low, medium, high, critical
    timeout_seconds: int = 60


class HumanInTheLoop:
    """Human-in-the-Loop 管理器"""

    def __init__(self):
        self._pending_confirmations: dict[str, asyncio.Future] = {}

    async def request_confirmation(
        self,
        request_id: str,
        request: ConfirmationRequest
    ) -> dict[str, Any]:
        """
        请求用户确认
        返回: {"confirmed": bool, "user_input": str}
        """
        future = asyncio.Future()
        self._pending_confirmations[request_id] = future

        try:
            result = await asyncio.wait_for(
                future,
                timeout=request.timeout_seconds
            )
            return result
        except asyncio.TimeoutError:
            return {"confirmed": False, "user_input": "timeout"}
        finally:
            self._pending_confirmations.pop(request_id, None)

    def resolve_confirmation(
        self,
        request_id: str,
        confirmed: bool,
        user_input: str = ""
    ) -> None:
        """解决确认请求 (前端调用)"""
        if request_id in self._pending_confirmations:
            future = self._pending_confirmations[request_id]
            if not future.done():
                future.set_result({
                    "confirmed": confirmed,
                    "user_input": user_input
                })
```

#### 4.9.6 CopilotKit 服务端集成

创建 FastAPI 端点供 CopilotKit 客户端连接：

```python
# src/cognizes/engine/agui/copilotkit_server.py
"""CopilotKit AG-UI 服务端"""

from fastapi import FastAPI, Request
from fastapi.responses import StreamingResponse
from typing import AsyncGenerator
import json

app = FastAPI()


@app.post("/api/copilotkit")
async def copilotkit_endpoint(request: Request) -> StreamingResponse:
    """CopilotKit AG-UI 端点"""
    body = await request.json()

    # 获取消息和工具定义
    messages = body.get("messages", [])
    frontend_tools = body.get("tools", [])

    async def generate_events() -> AsyncGenerator[str, None]:
        from .event_emitter import AgUiEventEmitter, EventType
        import uuid

        run_id = str(uuid.uuid4())
        emitter = AgUiEventEmitter(run_id)

        # 发送运行开始事件
        yield f"data: {json.dumps({'type': 'RUN_STARTED', 'run_id': run_id})}\n\n"

        # 处理消息并生成响应
        # ... (集成 ADK Agent 执行逻辑)

        # 发送运行完成事件
        yield f"data: {json.dumps({'type': 'RUN_FINISHED', 'run_id': run_id})}\n\n"

    return StreamingResponse(
        generate_events(),
        media_type="text/event-stream"
    )
```

### 4.10 Step 10: 思维链可视化调试

> [!NOTE]
>
> **对标 AG-UI 协议**：本节实现 The Realm of Mind 与 AG-UI 可视化层的深度集成，提供 Agent 思维链、工具调用和决策过程的实时可视化调试能力。
>
> **核心目标**：将 Glass-Box Runtime 的透明可观测性通过 AG-UI 事件呈现到前端。

#### 4.10.1 思维链可视化架构

```mermaid
graph TB
    subgraph "Realm of Mind 执行层"
        AE[AgentExecutor]
        TR[Tool Registry]
        OT[OpenTelemetry]
    end

    subgraph "可视化接口层"
        TV[ThinkingVisualizer]
        TC[ToolCallVisualizer]
        TD[TraceDebugger]
    end

    subgraph "AG-UI 事件"
        STEP[STEP_STARTED/FINISHED]
        TOOL[TOOL_CALL_*]
        CUST[CUSTOM Events]
    end

    AE -->|思维步骤| TV
    TR -->|工具调用| TC
    OT -->|追踪数据| TD

    TV --> STEP
    TC --> TOOL
    TD --> CUST

    style TV fill:#f472b6,stroke:#db2777,color:#000
    style TC fill:#60a5fa,stroke:#2563eb,color:#000
    style TD fill:#4ade80,stroke:#16a34a,color:#000
```

#### 4.10.2 AG-UI 事件映射表 (Mind 层)

| Mind 功能         | 触发条件                        | AG-UI 事件类型        | 展示组件     |
| :---------------- | :------------------------------ | :-------------------- | :----------- |
| 思维步骤开始      | AgentExecutor 进入 Thought 阶段 | `STEP_STARTED`        | 思维卡片     |
| 思维步骤完成      | Thought 输出生成                | `STEP_FINISHED`       | 思维内容展示 |
| 工具调用开始      | Tool Registry 调用工具          | `TOOL_CALL_START`     | 工具调用面板 |
| 工具参数流式      | 参数生成中                      | `TOOL_CALL_ARGS`      | 参数展示     |
| 工具调用完成      | 工具返回结果                    | `TOOL_CALL_END`       | 结果展示     |
| 推理链追踪        | OpenTelemetry Span 完成         | `CUSTOM (trace_span)` | Trace 时间线 |
| Human-in-the-Loop | confirmAction 触发              | 前端工具回调          | 审批对话框   |

#### 4.10.3 ThinkingVisualizer 实现

创建 `src/cognizes/engine/mind/thinking_visualizer.py`：

```python
"""
Mind ThinkingVisualizer: 思维链可视化接口

职责:
1. 提供 Agent 思维过程的实时可视化
2. 展示工具调用的参数和结果
3. 集成 OpenTelemetry Trace 可视化
"""

from __future__ import annotations

import json
from dataclasses import dataclass, field
from typing import Any, Optional
from datetime import datetime
from enum import Enum


class MindEventType(str, Enum):
    """Mind 层 AG-UI 事件类型"""
    THINKING_STEP = "thinking_step"
    TOOL_EXECUTION = "tool_execution"
    TRACE_SPAN = "trace_span"
    DECISION_POINT = "decision_point"


@dataclass
class ThinkingStep:
    """思维步骤"""
    step_id: str
    step_type: str  # thought, action, observation
    content: str
    reasoning: Optional[str] = None
    confidence: float = 1.0
    timestamp: float = field(default_factory=lambda: datetime.now().timestamp())


@dataclass
class ToolExecution:
    """工具执行"""
    tool_call_id: str
    tool_name: str
    args: dict
    result: Optional[Any] = None
    status: str = "pending"  # pending, running, completed, failed
    latency_ms: float = 0.0


@dataclass
class TraceSpan:
    """追踪 Span"""
    span_id: str
    parent_span_id: Optional[str]
    operation_name: str
    start_time: float
    end_time: Optional[float] = None
    attributes: dict = field(default_factory=dict)
    status: str = "running"


class ThinkingVisualizer:
    """思维链可视化器"""

    def __init__(self, event_emitter=None):
        """
        Args:
            event_emitter: AG-UI 事件发射器
        """
        self._event_emitter = event_emitter
        self._current_steps: list[ThinkingStep] = []
        self._tool_executions: dict[str, ToolExecution] = {}

    async def emit_thinking_started(
        self,
        run_id: str,
        step: ThinkingStep
    ) -> None:
        """
        发射思维步骤开始事件

        Args:
            run_id: 当前运行 ID
            step: 思维步骤
        """
        self._current_steps.append(step)

        if self._event_emitter:
            await self._event_emitter.emit_step_started(
                run_id=run_id,
                step_name=f"thinking_{step.step_type}",
                data={
                    "stepId": step.step_id,
                    "stepType": step.step_type,
                    "stepIndex": len(self._current_steps)
                }
            )

    async def emit_thinking_content(
        self,
        run_id: str,
        step_id: str,
        content_delta: str
    ) -> None:
        """
        发射思维内容增量事件 (流式)

        Args:
            run_id: 当前运行 ID
            step_id: 步骤 ID
            content_delta: 内容增量
        """
        if self._event_emitter:
            await self._event_emitter.emit_custom(
                run_id=run_id,
                event_name=MindEventType.THINKING_STEP.value,
                data={
                    "stepId": step_id,
                    "delta": content_delta
                }
            )

    async def emit_thinking_finished(
        self,
        run_id: str,
        step: ThinkingStep
    ) -> None:
        """
        发射思维步骤完成事件

        Args:
            run_id: 当前运行 ID
            step: 思维步骤
        """
        if self._event_emitter:
            await self._event_emitter.emit_step_finished(
                run_id=run_id,
                step_name=f"thinking_{step.step_type}",
                data={
                    "stepId": step.step_id,
                    "content": step.content[:500],  # 截断长内容
                    "reasoning": step.reasoning,
                    "confidence": step.confidence
                }
            )

    async def emit_tool_call_start(
        self,
        run_id: str,
        execution: ToolExecution
    ) -> None:
        """
        发射工具调用开始事件

        Args:
            run_id: 当前运行 ID
            execution: 工具执行
        """
        self._tool_executions[execution.tool_call_id] = execution
        execution.status = "running"

        if self._event_emitter:
            await self._event_emitter.emit_tool_call_start(
                run_id=run_id,
                tool_call_id=execution.tool_call_id,
                tool_call_name=execution.tool_name
            )

    async def emit_tool_call_args(
        self,
        run_id: str,
        tool_call_id: str,
        args_delta: str
    ) -> None:
        """
        发射工具参数增量事件 (流式)

        Args:
            run_id: 当前运行 ID
            tool_call_id: 工具调用 ID
            args_delta: 参数增量
        """
        if self._event_emitter:
            await self._event_emitter.emit_tool_call_args(
                run_id=run_id,
                tool_call_id=tool_call_id,
                delta=args_delta
            )

    async def emit_tool_call_end(
        self,
        run_id: str,
        tool_call_id: str,
        result: Any,
        latency_ms: float
    ) -> None:
        """
        发射工具调用完成事件

        Args:
            run_id: 当前运行 ID
            tool_call_id: 工具调用 ID
            result: 执行结果
            latency_ms: 延迟
        """
        if tool_call_id in self._tool_executions:
            execution = self._tool_executions[tool_call_id]
            execution.result = result
            execution.status = "completed"
            execution.latency_ms = latency_ms

        if self._event_emitter:
            await self._event_emitter.emit_tool_call_end(
                run_id=run_id,
                tool_call_id=tool_call_id,
                result=str(result)[:1000]  # 截断长结果
            )

    async def emit_trace_span(
        self,
        run_id: str,
        span: TraceSpan
    ) -> None:
        """
        发射追踪 Span 事件

        用于在前端展示 OpenTelemetry Trace

        Args:
            run_id: 当前运行 ID
            span: 追踪 Span
        """
        if self._event_emitter:
            await self._event_emitter.emit_custom(
                run_id=run_id,
                event_name=MindEventType.TRACE_SPAN.value,
                data={
                    "spanId": span.span_id,
                    "parentSpanId": span.parent_span_id,
                    "operationName": span.operation_name,
                    "startTime": span.start_time,
                    "endTime": span.end_time,
                    "durationMs": (
                        (span.end_time - span.start_time) * 1000
                        if span.end_time else None
                    ),
                    "attributes": span.attributes,
                    "status": span.status
                }
            )

    def get_thinking_summary(self) -> dict:
        """
        获取思维过程摘要

        Returns:
            思维摘要
        """
        return {
            "totalSteps": len(self._current_steps),
            "steps": [
                {
                    "id": s.step_id,
                    "type": s.step_type,
                    "preview": s.content[:100] if s.content else ""
                }
                for s in self._current_steps
            ],
            "toolCalls": [
                {
                    "id": e.tool_call_id,
                    "name": e.tool_name,
                    "status": e.status,
                    "latencyMs": e.latency_ms
                }
                for e in self._tool_executions.values()
            ]
        }
```

#### 4.10.4 前端展示组件规范

| 组件名称             | 数据源                  | 展示内容                     |
| :------------------- | :---------------------- | :--------------------------- |
| `ThinkingStepCard`   | STEP_STARTED/FINISHED   | 思维类型图标、内容、置信度   |
| `ToolCallPanel`      | TOOL*CALL*\*            | 工具名称、参数、结果、延迟   |
| `TraceTimeline`      | CUSTOM (trace_span)     | Span 层级树、时间线          |
| `DecisionFlowChart`  | CUSTOM (decision_point) | 决策分支可视化               |
| `HitlApprovalDialog` | 前端工具回调            | 确认按钮、输入框、超时倒计时 |

#### 4.10.5 任务清单 (补充)

| 任务 ID | 任务描述                     | 状态      | 验收标准         |
| :------ | :--------------------------- | :-------- | :--------------- |
| P4-5-11 | 实现 `ThinkingVisualizer` 类 | 🔲 待开始 | 4 种事件类型支持 |
| P4-5-12 | 实现思维步骤事件发射         | 🔲 待开始 | 思维链完整可见   |
| P4-5-13 | 实现工具调用可视化           | 🔲 待开始 | 参数/结果可展示  |
| P4-5-14 | 集成 OpenTelemetry Trace     | 🔲 待开始 | Span 可追溯      |
| P4-5-15 | 编写可视化接口测试           | 🔲 待开始 | 覆盖率 > 80%     |

#### 4.10.6 验收标准 (补充)

| 验收项   | 验收标准                            | 验证方法      |
| :------- | :---------------------------------- | :------------ |
| 思维链   | 实时展示 Thought/Action/Observation | 集成测试      |
| 工具调用 | 参数流式展示、结果完整展示          | E2E 测试      |
| Trace    | OpenTelemetry Span 可视化           | Langfuse 验证 |
| HITL     | 审批对话框正确弹出                  | 手动测试      |

---

## 5. 验证 SOP (Phase 4)

### 5.1 功能验收

| 验收项             | 验收标准               | 验证方法              |
| :----------------- | :--------------------- | :-------------------- |
| **SessionService** | 完全实现 ADK 接口契约  | 官方 Compliance Tests |
| **MemoryService**  | 完全实现 ADK 接口契约  | 官方 Compliance Tests |
| **State 前缀路由** | 四种前缀正确路由       | 单元测试覆盖          |
| **Tool Registry**  | 动态注册与热更新       | E2E 测试              |
| **AgentExecutor**  | ReAct 循环正确执行     | E2E 测试              |
| **OpenTelemetry**  | Trace 完整记录并可视化 | Langfuse UI 验证      |
| **安全沙箱**       | 恶意代码被拦截         | 安全测试用例          |

### 5.2 性能验收

| 指标               | 目标值      | 验证方法 |
| :----------------- | :---------- | :------- |
| Session CRUD 延迟  | P99 < 50ms  | 压力测试 |
| Memory Search 延迟 | P99 < 100ms | 压力测试 |
| 沙箱启动时间       | < 2s        | 基准测试 |

### 5.3 详细验收流程 (P4-5-1 ~ P4-5-4)

#### 5.3.1 验收前置条件

```bash
# 1. 环境准备
export DATABASE_URL="postgresql://aigc:@localhost:5432/cognizes-engine"
export GOOGLE_API_KEY="your-gemini-api-key"
export OTEL_EXPORTER_OTLP_ENDPOINT="http://localhost:4317"

# 2. 启动依赖服务
# docker-compose up -d postgres langfuse microsandbox
# 需参照前文，手动挨个启动

# 3. 初始化数据库
psql -d 'cognizes-engine' -f src/cognizes/engine/schema/agent_schema.sql
psql -d 'cognizes-engine' -f src/cognizes/engine/schema/hippocampus_schema.sql
psql -d 'cognizes-engine' -f src/cognizes/engine/schema/perception_schema.sql
psql -d 'cognizes-engine' -f src/cognizes/engine/schema/mind_schema.sql

# 4. 安装 Open Agent Engine (包含 adk-postgres 适配器)
# 在项目根目录执行
pip install -e ".[dev]"
```

#### 5.3.2 逐项验收检查清单

> [!IMPORTANT]
>
> 逐项执行以下验收检查，全部通过后方可进入 Phase 5。

**SessionService 验收 (P4-2-2 ~ P4-2-8)**

|  #  | 验收项                     | 验收命令                                                                                                                  | 预期结果 | 状态 |
| :-: | :------------------------- | :------------------------------------------------------------------------------------------------------------------------ | :------- | :--: |
|  1  | create_session 基础创建    | `pytest tests/unittests/mind/test_session_service.py::TestPostgresSessionService::test_create_session_basic`              | PASSED   |  ⬜  |
|  2  | create_session 带初始状态  | `pytest tests/unittests/mind/test_session_service.py::TestPostgresSessionService::test_create_session_with_initial_state` | PASSED   |  ⬜  |
|  3  | get_session 存在的会话     | `pytest tests/unittests/mind/test_session_service.py::TestPostgresSessionService::test_get_session_exists`                | PASSED   |  ⬜  |
|  4  | get_session 带分页配置     | `pytest tests/unittests/mind/test_session_service.py::TestPostgresSessionService::test_get_session_with_config`           | PASSED   |  ⬜  |
|  5  | list_sessions 列出用户会话 | `pytest tests/unittests/mind/test_session_service.py::TestPostgresSessionService::test_list_sessions_by_user`             | PASSED   |  ⬜  |
|  6  | delete_session 删除验证    | `pytest tests/unittests/mind/test_session_service.py::TestPostgresSessionService::test_delete_session`                    | PASSED   |  ⬜  |
|  7  | user: 前缀跨会话持久       | `pytest tests/unittests/mind/test_session_service.py::TestPostgresSessionService::test_state_prefix_user_scope`           | PASSED   |  ⬜  |
|  8  | temp: 前缀不持久化         | `pytest tests/unittests/mind/test_session_service.py::TestPostgresSessionService::test_state_prefix_temp_not_persisted`   | PASSED   |  ⬜  |

**MemoryService 验收 (P4-2-9 ~ P4-2-12)**

|  #  | 验收项                 | 验收命令                                                                                                    | 预期结果 | 状态 |
| :-: | :--------------------- | :---------------------------------------------------------------------------------------------------------- | :------- | :--: |
|  9  | add_session_to_memory  | `pytest tests/unittests/mind/test_memory_service.py::TestPostgresMemoryService::test_add_session_to_memory` | PASSED   |  ⬜  |
| 10  | search_memory 语义检索 | `pytest tests/unittests/mind/test_memory_service.py::TestPostgresMemoryService::test_search_memory`         | PASSED   |  ⬜  |
| 11  | list_memories 列出记忆 | `pytest tests/unittests/mind/test_memory_service.py::TestPostgresMemoryService::test_list_memories`         | PASSED   |  ⬜  |
| 12  | 跨会话记忆召回         | `pytest tests/integration/mind/test_e2e.py::TestE2EIntegration::test_cross_session_memory_recall`           | PASSED   |  ⬜  |

**Tool Registry 验收 (P4-2-13 ~ P4-2-18)**

|  #  | 验收项                       | 验收命令                                                                                        | 预期结果 | 状态 |
| :-: | :--------------------------- | :---------------------------------------------------------------------------------------------- | :------- | :--: |
| 13  | register_tool 注册工具       | `pytest tests/unittests/mind/test_tool_registry.py::TestToolRegistry::test_register_tool`       | PASSED   |  ⬜  |
| 14  | get_available_tools 获取列表 | `pytest tests/unittests/mind/test_tool_registry.py::TestToolRegistry::test_get_available_tools` | PASSED   |  ⬜  |
| 15  | invoke_tool 调用与统计       | `pytest tests/unittests/mind/test_tool_registry.py::TestToolRegistry::test_invoke_tool`         | PASSED   |  ⬜  |
| 16  | 热更新 (无需重启)            | `pytest tests/unittests/mind/test_tool_registry.py::TestToolRegistry::test_hot_update`          | PASSED   |  ⬜  |

**OpenTelemetry 验收 (P4-4-1 ~ P4-4-4)**

|  #  | 验收项                  | 验收命令                                                                                          | 预期结果   | 状态 |
| :-: | :---------------------- | :------------------------------------------------------------------------------------------------ | :--------- | :--: |
| 17  | Trace 导出到 PostgreSQL | `pytest tests/integration/mind/test_tracing.py::TestPostgresSpanExporter::test_trace_to_postgres` | PASSED     |  ⬜  |
| 18  | Trace 导出到 Langfuse   | 访问 `http://localhost:3000` + Langfuse UI 验证                                                   | Trace 可见 |  ⬜  |
| 19  | Span 层级正确           | `pytest tests/integration/mind/test_tracing.py::TestSpanHierarchy::test_span_hierarchy`           | PASSED     |  ⬜  |

**安全沙箱验收 (P4-4-5 ~ P4-4-10)**

|  #  | 验收项       | 验收命令                                                                                                  | 预期结果 | 状态 |
| :-: | :----------- | :-------------------------------------------------------------------------------------------------------- | :------- | :--: |
| 20  | 正常代码执行 | `pytest tests/unittests/mind/test_sandbox.py::TestBaseSandboxRunner::test_execute_normal_code`            | PASSED   |  ⬜  |
| 21  | 恶意代码拦截 | `pytest tests/unittests/mind/test_sandbox.py::TestBaseSandboxRunner::test_block_malicious_code_os_system` | PASSED   |  ⬜  |
| 22  | 超时控制     | `pytest tests/unittests/mind/test_sandbox.py::TestBaseSandboxRunner::test_timeout_control`                | PASSED   |  ⬜  |
| 23  | 网络隔离     | `pytest tests/unittests/mind/test_sandbox.py::TestBaseSandboxRunner::test_network_isolation`              | PASSED   |  ⬜  |

#### 5.3.3 ADK 集成验收

创建：`tests/integration/mind/test_adk_integration.py`:

```python
# test_adk_integration.py - ADK 集成验收脚本
"""
验收目标: 验证 adk-postgres 与 Google ADK LlmAgent 的完整集成
"""

import asyncio
from google.adk.agents import LlmAgent
from google.adk.runners import Runner
from cognizes.adapters.postgres.session_service import PostgresSessionService
from cognizes.adapters.postgres.memory_service import PostgresMemoryService

async def verify_adk_integration():
    """ADK 集成验收"""
    import asyncpg
    pool = await asyncpg.create_pool(os.environ["DATABASE_URL"])

    # 1. 创建服务
    session_svc = PostgresSessionService(pool=pool)
    memory_svc = PostgresMemoryService(pool=pool)

    # 2. 创建 Agent
    agent = LlmAgent(
        name="test_agent",
        model="gemini-2.0-flash",
        instruction="You are a helpful assistant.",
    )

    # 3. 创建 Runner 并注入 PostgreSQL 服务
    runner = Runner(
        agent=agent,
        app_name="integration_test",
        session_service=session_svc,
        memory_service=memory_svc,
    )

    # 4. 执行对话
    session = await session_svc.create_session(
        app_name="integration_test",
        user_id="verifier"
    )

    async for event in runner.run_async(
        session_id=session.id,
        user_id="verifier",
        new_message="Hello, how are you?"
    ):
        if event.is_final_response():
            assert event.text is not None, "Agent 应返回响应"
            print(f"✅ ADK 集成验收通过: {event.text[:50]}...")
            break

    await pool.close()

if __name__ == "__main__":
    asyncio.run(verify_adk_integration())
```

#### 5.3.4 性能压测流程

```bash
# 1. SessionService 压力测试 (100 并发, 1000 请求)
uv run locust -f tests/performance/mind/locustfile.py \
  --users 100 --spawn-rate 10 --run-time 60s \
  --host $DATABASE_URL \
  --html report_session.html

# 2. 验收标准
# - P99 Latency < 50ms
# - Error Rate < 0.1%
# - Throughput > 500 RPS
```

#### 5.3.5 验收报告模板

```markdown
# Phase 4 验收报告

## 验收信息

- **验收日期**: YYYY-MM-DD
- **验收人**: [Name]
- **Git Commit**: [hash]

## 功能验收结果

| 模块           | 通过项 | 总项数 | 通过率 |
| :------------- | -----: | -----: | -----: |
| SessionService |      X |      8 |     X% |
| MemoryService  |      X |      4 |     X% |
| Tool Registry  |      X |      4 |     X% |
| OpenTelemetry  |      X |      3 |     X% |
| 安全沙箱       |      X |      4 |     X% |
| **总计**       |  **X** | **23** | **X%** |

## 性能验收结果

| 指标              | 目标   | 实测 | 状态  |
| :---------------- | :----- | :--- | :---: |
| Session CRUD P99  | <50ms  | Xms  | ✅/❌ |
| Memory Search P99 | <100ms | Xms  | ✅/❌ |
| 沙箱启动时间      | <2s    | Xs   | ✅/❌ |

## ADK 集成验收

- [ ] LlmAgent 正常响应
- [ ] Session 状态正确持久化
- [ ] Memory 搜索返回相关结果
- [ ] Trace 完整记录

## 遗留问题

1. [描述问题及处理计划]

## 验收结论

- [ ] **通过**: 全部验收项达标，可进入 Phase 5
- [ ] **有条件通过**: 存在非阻塞问题，跟踪处理
- [ ] **不通过**: 存在阻塞问题，需修复后重新验收
```

#### 5.3.6 回归测试指引

> [!NOTE]
>
> 后续代码变更后，执行以下回归测试确保未引入回归。

```bash
# 完整回归测试套件
pytest tests/ -v --cov=cognizes.adapters.postgres --cov=cognizes.engine.mind --cov-report=html

# 快速冒烟测试 (CI 用)
pytest tests/ -v -m "smoke" --maxfail=3

# 生成测试报告
pytest tests/ --html=regression_report.html --self-contained-html
```

**pytest.ini 标记配置**:

```ini
[pytest]
markers =
    smoke: 冒烟测试 (快速验证核心功能)
    integration: 集成测试 (需要数据库)
    performance: 性能测试 (需要较长时间)
    security: 安全测试 (沙箱相关)
```

### 5.4 Phase 4 完整验收 SOP

> [!IMPORTANT]
>
> 本节提供 Phase 4 的完整验收流程，确保所有功能点按照 `000-roadmap.md` 的验证矩阵进行系统性验证。按照以下步骤逐项执行，全部通过后方可进入 Phase 5。

#### 5.4.1 环境核验

**Step 1: 数据库环境检查**

```bash
# 1. 验证 PostgreSQL 运行状态
pg_isready -h localhost -p 5432

# 2. 验证数据库存在
psql -d cognizes-engine -c "SELECT version();"

# 3. 验证扩展安装
psql -d cognizes-engine -c "SELECT extname, extversion FROM pg_extension WHERE extname IN ('vector', 'pg_cron');"

# 预期输出:
#  extname  | extversion
# ----------+------------
#  vector   | 0.7.0
#  pg_cron  | 1.6
```

**Step 2: Langfuse 环境检查**

```bash
# 1. 验证 Langfuse 服务可达 (Docker 部署)
curl -s https://langfuse.automizely.me/api/public/health | jq .

# 预期输出:
# { "status": "ok" }

# 2. 验证环境变量配置
echo "LANGFUSE_PUBLIC_KEY: ${LANGFUSE_PUBLIC_KEY:0:8}..."
echo "LANGFUSE_SECRET_KEY: ${LANGFUSE_SECRET_KEY:0:8}..."
echo "LANGFUSE_HOST: ${LANGFUSE_HOST:-https://cloud.langfuse.com}"

# 若未配置，请设置:
# export LANGFUSE_PUBLIC_KEY="pk-lf-xxx"
# export LANGFUSE_SECRET_KEY="sk-lf-xxx"
# export LANGFUSE_HOST="http://localhost:3000"
```

**Step 3: Python 环境检查**

```bash
# 1. 验证 Python 版本
python --version  # 预期: Python 3.13+

# 2. 验证依赖安装
python -c "import google.adk; import asyncpg; print('✅ ADK 和 asyncpg 正常')"

# 3. 可选: 验证 Langfuse SDK
python -c "import langfuse; print('✅ Langfuse SDK 正常')"

# 4. 验证项目模块可导入
python -c "from cognizes.adapters.postgres import session_service, memory_service, tracing; print('✅ 模块正常')"
```

#### 5.4.2 自动化核查

**Step 1: 执行单元测试**

```bash
# 运行 Mind 模块全部单元测试 (Mock 隔离，不需数据库)
pytest tests/unittests/mind/ -v --tb=short

# 预期: 全部 PASSED
# ==================== XX passed in X.XXs ====================
```

**Step 2: 执行集成测试**

```bash
# 运行 Mind 模块全部集成测试 (需要数据库)
pytest tests/integration/mind/ -v --tb=short -m "not skip"

# 预期: 全部 PASSED (跳过需要真实 Langfuse 的测试)
```

**Step 3: 执行 Langfuse 集成测试**

```bash
# 运行 Langfuse 集成测试 (需要配置环境变量)
# pytest tests/integration/mind/test_langfuse_integration.py -v --tb=short
pytest tests/integration/mind/test_langfuse_real.py -v --tb=short

# 预期:
# - 模拟测试 PASSED
# - 真实 Langfuse 测试 (如配置) 输出 Trace ID
```

**Step 4: 生成覆盖率报告**

```bash
# 生成测试覆盖率报告
pytest tests/ -v --cov=cognizes.adapters.postgres --cov=cognizes.engine.mind \
    --cov-report=html --cov-report=term

# 预期: 覆盖率 >= 80%
# 报告路径: htmlcov/index.html
```

#### 5.4.3 控制台验证

**Step 1: Langfuse UI 验证**

1. 打开浏览器访问: `http://localhost:3000` (本地) 或 `https://cloud.langfuse.com` (云端)
2. 登录并进入项目 Dashboard
3. 验证以下内容:

| 验证项          | 预期结果                                | 状态 |
| :-------------- | :-------------------------------------- | :--: |
| Traces 列表可见 | 显示测试产生的 Trace 记录               |  ⬜  |
| Span 层级正确   | 父子 Span 关系正确展示                  |  ⬜  |
| Generation 事件 | 显示模型名称、Token 消耗                |  ⬜  |
| Evaluation 分数 | 显示 `relevance`, `faithfulness` 等分数 |  ⬜  |

**Step 2: PostgreSQL traces 表验证**

```bash
# 查询最近的 Trace 记录
psql -d 'cognizes-engine' -c "
SELECT trace_id, span_id, operation_name, span_kind,
       start_time, status_code
FROM traces
ORDER BY start_time DESC
LIMIT 10;
"

# 验证层级关系
psql -d 'cognizes-engine' -c "
SELECT t1.operation_name AS parent_op,
       t2.operation_name AS child_op
FROM traces t1
JOIN traces t2 ON t2.parent_span_id = t1.span_id
WHERE t1.parent_span_id IS NULL
LIMIT 5;
"
```

#### 5.4.4 指标对齐验证

根据 `000-roadmap.md` 验证矩阵 (Section 2.1)，确认以下指标对齐:

| 指标                        | 目标 (Roadmap)       | 验证方法                 | 状态 |
| :-------------------------- | :------------------- | :----------------------- | :--: |
| **Session CRUD P99**        | < 50ms               | 压测报告 / 单元测试      |  ⬜  |
| **Memory Search P99**       | < 100ms              | 压测报告 / 集成测试      |  ⬜  |
| **Trace 完整性**            | 100% (全链路可追踪)  | Langfuse UI 验证         |  ⬜  |
| **Token Cost 可追踪**       | ✅ (Generation 事件) | Langfuse Generation 面板 |  ⬜  |
| **Evaluation Score 可记录** | ✅ (多维度评分)      | Langfuse Evaluation 面板 |  ⬜  |
| **可调试性**                | 能定位推理死循环     | Trace Timeline 可视化    |  ⬜  |

> [!NOTE]
>
> 以上指标对应 Roadmap 中 **The Realm of Mind** 支柱的核心核验指标:
>
> - **可调试性 (Debuggability)**: 能否精准定位推理死循环或幻觉
> - **Glass-Box Tracing**: 完整记录 Thought Chain、Tool IO 与 Slot Updates

#### 5.4.5 验收总结检查清单

```markdown
# Phase 4 验收检查清单

## 环境核验

- [ ] PostgreSQL 运行正常，扩展已安装
- [ ] Langfuse 服务可达 (或环境变量已配置)
- [ ] Python 依赖安装完整

## 功能验收

- [ ] 单元测试全部通过 (pytest tests/unittests/mind/)
- [ ] 集成测试全部通过 (pytest tests/integration/mind/)
- [ ] Langfuse 集成测试通过 (Trace ID 透传验证)
- [ ] ADK 接口兼容性验证通过

## 可观测性验收

- [ ] Langfuse Traces 可见
- [ ] Span 层级关系正确
- [ ] Generation 事件包含 Token 消耗
- [ ] Evaluation 分数正确记录
- [ ] PostgreSQL traces 表有数据

## 指标对齐验收

- [ ] Session CRUD P99 < 50ms
- [ ] Memory Search P99 < 100ms
- [ ] Trace 完整性 100%
- [ ] Token Cost 可追踪
- [ ] Evaluation Score 可记录

## 验收结论

- [ ] **通过**: 全部验收项达标，可进入 Phase 5
- [ ] **有条件通过**: 存在非阻塞问题，跟踪处理
- [ ] **不通过**: 存在阻塞问题，需修复后重新验收
```

### 5.5 核验命令快速参考

> [!TIP]
>
> 以下命令可直接复制执行，用于快速验证 Phase 4 各模块的实现状态。

#### 5.5.1 测试执行命令

```bash
# Phase 4 完整测试套件 (82 tests expected)
uv run pytest tests/unittests/mind/ tests/integration/mind/ -v --tb=short

# 仅运行单元测试
uv run pytest tests/unittests/mind/ -v

# 仅运行集成测试
uv run pytest tests/integration/mind/ -v

# 运行特定模块测试
uv run pytest tests/unittests/mind/test_session_service.py -v
uv run pytest tests/unittests/mind/test_sandbox.py -v
uv run pytest tests/integration/mind/test_tracing.py -v
```

#### 5.5.2 PostgreSQL 数据验证

```bash
# 验证 traces 表有数据
psql -d 'cognizes-engine' -c "SELECT COUNT(*) as trace_count FROM traces;"

# 查看最近 10 条 Trace 记录
psql -d 'cognizes-engine' -c "
SELECT trace_id, span_id, operation_name, start_time
FROM traces
ORDER BY start_time DESC
LIMIT 10;
"

# 验证 tools 表注册情况
psql -d 'cognizes-engine' -c "
SELECT app_name, name, is_active, call_count, avg_latency_ms
FROM tools
ORDER BY updated_at DESC
LIMIT 10;
"

# 验证 Span 层级关系
psql -d 'cognizes-engine' -c "
SELECT t1.operation_name AS parent_op,
       t2.operation_name AS child_op,
       t2.parent_span_id
FROM traces t1
JOIN traces t2 ON t2.parent_span_id = t1.span_id
WHERE t1.parent_span_id IS NULL
LIMIT 5;
"
```

#### 5.5.3 Langfuse 集成验证

```bash
# 发送测试 Trace 到 Langfuse
set -a && source .env && set +a && uv run python tests/integration/mind/test_langfuse_real.py

# 验证 Langfuse 环境变量
echo "LANGFUSE_HOST: ${LANGFUSE_HOST:-not set}"
echo "LANGFUSE_PUBLIC_KEY: ${LANGFUSE_PUBLIC_KEY:0:8}..."
```

#### 5.5.4 TracingManager 功能验证

```bash
# 验证 PostgresSpanExporter 正常工作
set -a && source .env && set +a && uv run python -c "
import asyncio
from cognizes.adapters.postgres.tracing import TracingManager
import os

async def test_postgres_tracing():
    dsn = os.environ['DATABASE_URL']
    manager = TracingManager(service_name='verification_test', pg_dsn=dsn)

    async with manager.span('verification_root') as root:
        root.set_attribute('test.source', 'verification_script')
        async with manager.span('verification_child') as child:
            child.set_attribute('test.child', 'true')

    manager.provider.force_flush()
    print('✅ Trace 已发送到 PostgreSQL')

    # 验证写入
    await asyncio.sleep(2)
    import asyncpg
    conn = await asyncpg.connect(dsn)
    count = await conn.fetchval('SELECT COUNT(*) FROM traces WHERE operation_name LIKE \\'verification_%\\'')
    print(f'✅ 查询到 {count} 条验证记录')
    await conn.close()

asyncio.run(test_postgres_tracing())
"
```

#### 5.5.5 技术指标核验

| 指标             | 目标   | 验证命令                                                      | 预期结果          |
| :--------------- | :----- | :------------------------------------------------------------ | :---------------- |
| 单元测试通过     | 100%   | `uv run pytest tests/unittests/mind/ -v`                      | 全绿              |
| 集成测试通过     | ≥ 90%  | `uv run pytest tests/integration/mind/ -v`                    | ≥ 82 pass         |
| PostgreSQL Trace | 可写入 | `psql -d 'cognizes-engine' -c "SELECT COUNT(*) FROM traces;"` | ≥ 1               |
| Langfuse 连接    | 可达   | `curl -s ${LANGFUSE_HOST}/api/public/health`                  | `{"status":"OK"}` |

---

## 6. 验收基准

| 模块               | 文件/目录                                                 | 描述                  |
| :----------------- | :-------------------------------------------------------- | :-------------------- |
| **技术文档**       | `docs/040-the-realm-of-mind.md`                           | 本文档                |
| **Python Package** | `src/cognizes/adapters/postgres/`                         | ADK PostgreSQL 适配器 |
| **SessionService** | `src/cognizes/adapters/postgres/session_service.py`       | PostgreSQL 会话服务   |
| **MemoryService**  | `src/cognizes/adapters/postgres/memory_service.py`        | PostgreSQL 记忆服务   |
| **Tool Registry**  | `src/cognizes/adapters/postgres/tool_registry.py`         | 动态工具注册表        |
| **AgentExecutor**  | `src/cognizes/engine/mind/agent_executor.py`              | Agent 执行编排器      |
| **Tracing**        | `src/cognizes/adapters/postgres/tracing.py`               | OpenTelemetry 集成    |
| **Sandbox**        | `src/cognizes/adapters/postgres/sandbox/docker_runner.py` | Docker 沙箱实现       |
| **Schema**         | `src/cognizes/engine/schema/mind_schema.sql`              | Phase 4 Schema 扩展   |

---

## 7. 风险与缓解策略

## 8. 参考资料

<a id="ref1"></a>1. Google. (2025). _ADK Sessions Documentation_. [Link](https://google.github.io/adk-docs/sessions/)

<a id="ref2"></a>2. Google. (2025). _ADK Context Documentation_. [Link](https://google.github.io/adk-docs/context/)

<a id="ref3"></a>3. OpenTelemetry. (2025). _Python SDK_. [Link](https://opentelemetry.io/docs/instrumentation/python/)

<a id="ref4"></a>4. Docker. (2025). _Docker SDK for Python_. [Link](https://docker-py.readthedocs.io/)
