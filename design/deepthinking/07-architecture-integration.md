---
id: "deepthinking-integration-007"
title: "系统集成架构深度设计"
aliases: ["architecture integration deep design", "系统集成设计", "engine integration"]
type: "deepthinking"
category: "design/deepthinking"
tags: ["integration", "architecture", "event-driven", "microservices", "deep-design"]
version: "1.0.0"
created: "2026-05-04"
updated: "2026-05-04"
author: "jxncyjq"
status: "review"
parent: null
related_docs:
  - id: "architecture-legion"
    relation: "parent_design"
    path: "../architecture/Legion.md"
---

<!-- @section: overview -->
# 系统集成架构深度设计

## 核心命题

> 三大引擎之间应该"紧耦合"还是"松耦合"？

答案：**松耦合 + 事件驱动**。三大引擎（MaaS、Agent、Wiki）各自独立可替换，通过标准化接口和事件总线通信。这份文档回答"三个引擎如何协同工作"这一系统级问题。

## 一、系统全景 — 三大引擎的职责边界

### 1.1 清晰的职责划分

```
┌─────────────────────────────────────────────────────────┐
│                    Legion 系统边界                        │
│                                                          │
│  ┌──────────────────┐  ┌──────────────────┐             │
│  │   MaaS 模型调度层  │  │  LLM Wiki 知识库  │             │
│  │                  │  │                  │             │
│  │  负责:            │  │  负责:            │             │
│  │  · 模型注册       │  │  · 知识条目管理    │             │
│  │  · 智能路由       │  │  · 混合检索       │             │
│  │  · 响应规范化     │  │  · 知识图谱       │             │
│  │  · 配额管控       │  │  · 版本管理       │             │
│  │  · 成本追踪       │  │  · 审核流程       │             │
│  │                  │  │                  │             │
│  │  不负责:          │  │  不负责:          │             │
│  │  · Agent 逻辑     │  │  · LLM 调用       │             │
│  │  · 工具执行       │  │  · 任务编排       │             │
│  │  · 知识管理       │  │  · 模型调度       │             │
│  └────────┬─────────┘  └────────┬─────────┘             │
│           │                     │                        │
│           │    ┌───────────────┐│                        │
│           └───→│  Agent 引擎   │←───────────────────────┘│
│                │               │                         │
│                │  负责:         │                         │
│                │  · 认知内核    │                         │
│                │  · 主循环      │                         │
│                │  · 工具执行    │                         │
│                │  · 记忆管理    │                         │
│                │  · 进化学习    │                         │
│                │  · 多Agent协作 │                         │
│                │               │                         │
│                │  不负责:       │                         │
│                │  · 模型选择    │                         │
│                │  · 知识存储    │                         │
│                │  · 配额管控    │                         │
│                └───────────────┘                         │
│                                                          │
│  ┌──────────────────────────────────────────────────┐   │
│  │              事件总线 (Event Bus)                  │   │
│  │  模型调用 / 知识更新 / Agent 状态 / 审批事件 / ...  │   │
│  └──────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────┘
```

### 1.2 为什么不紧耦合

| 紧耦合 (如果这样做) | 问题 | 松耦合 (Legion 的选择) |
|-------------------|------|----------------------|
| Agent 直接调 LLM API | 无法统一管控成本和路由 | Agent 通过 MaaS 层调模型 |
| Agent 直接写文件 | 知识无法共享和检索 | Agent 通过 Wiki API 存储知识 |
| MaaS 层管理知识 | 职责混乱 | Wiki 独立管理知识 |
| Wiki 调 LLM | 无法利用 MaaS 路由优化 | Wiki 也通过 MaaS 层调模型 |

<!-- @end-section -->

<!-- @section: engine-interfaces -->
## 二、引擎间接口定义

### 2.1 MaaS → Agent 接口

MaaS 层对 Agent 引擎暴露的接口：

```go
// MaaS 层提供的 Agent 调用接口
type MaaSProvider interface {
    // 核心能力
    ChatCompletion(ctx context.Context, req ChatRequest) (*NormalizedResponse, error)
    StreamChatCompletion(ctx context.Context, req ChatRequest) (<-chan StreamEvent, error)

    // 模型查询
    GetAvailableModels(filter ModelFilter) ([]ModelCapability, error)
    GetModelInfo(modelID string) (*ModelCapability, error)

    // 预算查询
    GetBudgetRemaining(scope BudgetScope) (*BudgetStatus, error)
}

// Agent 引擎调用 MaaS 层时不需要知道:
// - 实际用了哪个模型 (MaaS 层自动路由)
// - 模型 API 的具体格式 (MaaS 层规范化)
// - 费用预扣/结算 (MaaS 层透明处理)
// - 故障转移 (MaaS 层自动执行)
```

### 2.2 Wiki → Agent 接口

Wiki 对 Agent 暴露的接口：

```go
type WikiProvider interface {
    // 知识检索
    Search(ctx context.Context, query string, opts SearchOpts) ([]SearchResult, error)
    GetEntry(ctx context.Context, entryID string) (*KnowledgeEntry, error)
    GetRelated(ctx context.Context, entryID string, depth int) ([]KnowledgeEntry, error)

    // 知识写入 (Agent 贡献)
    SubmitDraft(ctx context.Context, draft KnowledgeDraft) (string, error) // → entryID
    ProposeUpdate(ctx context.Context, entryID string, update ContentBlock) error
    ReportConflict(ctx context.Context, entryA, entryB string, reason string) error
}

type SearchOpts struct {
    Limit        int
    EntryType    []EntryType    // 可选过滤
    Domain       string          // 可选过滤
    MinQuality   float64         // 最低质量评分
    IncludeDraft bool            // 是否包含待审核条目
}
```

### 2.3 Agent → Agent 接口 (通讯)

Agent 间协作通过组织通讯接口：

```go
type CommunicationProvider interface {
    // 消息收发
    SendMessage(ctx context.Context, msg Message) error
    ReceiveMessages(ctx context.Context, agentID string) ([]Message, error)
    AcknowledgeMessage(ctx context.Context, msgID string) error

    // 协作
    DelegateTask(ctx context.Context, task DelegateRequest) error
    ReportStatus(ctx context.Context, taskID string, status TaskStatus) error
    RequestApproval(ctx context.Context, req ApprovalRequest) error
}
```

### 2.4 事件总线接口

所有引擎通过事件总线松耦合通信：

```go
type EventBus interface {
    Publish(ctx context.Context, event Event) error
    Subscribe(ctx context.Context, pattern EventPattern, handler EventHandler) error
}

type Event struct {
    ID        string      `json:"id"`
    Type      EventType   `json:"type"`
    Source    string      `json:"source"`   // engine:component
    Payload   interface{} `json:"payload"`
    Timestamp time.Time   `json:"timestamp"`
}

// 核心事件流:
// model.call.completed    → MaaS 发布 → Agent 消费 (更新成本)
// agent.task.completed    → Agent 发布 → Wiki 消费 (触发知识提取)
// agent.evolution.applied → Agent 发布 → 系统消费 (审计)
// wiki.entry.updated      → Wiki 发布 → Agent 消费 (心跳感知)
// governance.approval_needed → 工作流发布 → 通知人类决策者
```

<!-- @end-section -->

<!-- @section: data-flows -->
## 三、核心跨引擎数据流

### 3.1 数据流全景

```
                    ┌──────────────┐
                    │  LLM Wiki    │
                    │              │
                    │  知识库       │
                    └──┬───────┬──┘
                       │       │
        ② Agent 检索知识│       │③ Agent 提出知识更新
        ④ 进化经验沉淀   │       │⑦ 知识状态变更通知
                       │       │
              ┌────────┴───────┴────────┐
              │      Agent 引擎          │
              │                          │
              │  认知内核 · 主循环        │
              │  记忆系统 · 工具系统      │
              │  进化学习 · 多Agent协作   │
              └────────┬─────────────────┘
                       │
         ① Agent 请求LLM推理
         ⑤ 进化引擎请求LLM分析
                       │
              ┌────────┴───────┐
              │  MaaS 模型调度  │
              │                │
              │  智能路由       │
              │  配额管控       │
              │  响应规范化     │
              └────────────────┘

数据流说明:
  ① Agent → MaaS: LLM 推理请求 (最频繁)
  ② Wiki → Agent: 知识检索 (任务启动时)
  ③ Agent → Wiki: 经验知识提交 (任务完成后)
  ④ Agent → Wiki: 进化资产沉淀 (定期)
  ⑤ Agent → MaaS: LLM 辅助分析 (信号提取第 3 层)
  ⑦ Wiki → Agent: 知识变更通知 (心跳感知)
```

### 3.2 流 ①: Agent 请求 LLM 推理 (最频繁的主流程)

```
时机: Agent 主循环中每次需要 LLM 推理时

数据格式:
  Agent → MaaS:
    {
      "agent_id": "dev-001",
      "task_id": "task-123",
      "messages": [...],          // 认知内核组装好的上下文
      "tools": [...],             // 可用工具定义
      "hints": {
        "task_type": "code_generation",
        "preferred_model_tier": "high",
        "max_cost_tolerance": 0.05  // USD
      }
    }

  MaaS → Agent:
    {
      "content": "...",
      "tool_calls": [...],
      "finish_reason": "tool_calls",
      "usage": { "input": 5000, "output": 500 },
      "model_id": "claude-sonnet-4",     // 实际使用的模型
      "latency_ms": 1200,
      "cost": { "total": 0.0225 },
      "route_decision": "role_binding"    // 为什么选这个模型
    }

Agent 不需要知道:
  - 模型 API 的原始请求/响应格式
  - 费用预扣和结算过程
  - 故障转移的发生
```

### 3.3 流 ②: Agent 检索知识

```
时机: 认知内核组装上下文时 → 任务开始时 → 遇到未知领域时

数据格式:
  Agent → Wiki:
    {
      "query": "如何编写 Go 语言的表驱动测试",
      "context": {
        "task_type": "code_generation",
        "agent_role": "developer",
        "current_step": "writing_unit_tests"
      },
      "opts": {
        "limit": 5,
        "min_quality": 0.7,
        "prefer_type": ["strategy", "best_practice"]
      }
    }

  Wiki → Agent:
    [
      {
        "entry_id": "wiki:go-testing-pattern:v3",
        "title": "Go 表驱动测试最佳实践",
        "type": "best_practice",
        "content": { "summary": "...", "steps": [...] },
        "quality_score": 0.95,
        "relations": ["wiki:go-testing-examples:v2"]
      },
      ...
    ]
```

### 3.4 流 ③: Agent 提出知识更新

```
时机: 任务完成后 → 经验提取 → 生成知识草稿

数据格式:
  Agent → Wiki:
    {
      "source": {
        "agent_id": "dev-001",
        "task_id": "task-123",
        "capsule_id": "cap_abc456"
      },
      "draft": {
        "type": "best_practice",
        "title": "REST API 分页处理模式",
        "content": {
          "context": "在处理大量数据列表时...",
          "strategy": ["使用 cursor-based 分页", "设置合理的 page_size 上限"],
          "evidence": { "capsule_id": "cap_abc456", "success_rate": 0.95 }
        },
        "domain": "engineering",
        "tags": ["api", "pagination", "performance"]
      }
    }

  Wiki → Agent:
    {
      "entry_id": "wiki:api-pagination-pattern:v1",
      "status": "review_pending",
      "review_eta": "24h"
    }

后续: 审核通过 → 发布事件 "wiki.entry.published" → 通知所有相关 Agent
```

### 3.5 流 ⑤: 进化引擎请求 LLM 分析

```
时机: 信号提取第 3 层 → 深度语义分析 → 周期性触发

数据格式:
  Agent (进化引擎) → MaaS:
    {
      "agent_id": "dev-001",
      "purpose": "signal_analysis",        // 标记为进化用途
      "messages": [
        {
          "role": "user",
          "content": "分析以下 Agent 最近 20 次任务的模式:\n" +
                     "[任务日志摘要...]\n\n" +
                     "识别: 1) 重复的失败模式 2) 效率下降趋势 3) 可改进的决策点"
        }
      ],
      "hints": {
        "task_type": "analysis",           // 分析类任务
        "preferred_model_tier": "low",     // 用便宜模型做分析
        "max_cost_tolerance": 0.01
      }
    }

  MaaS 自动选择低成本模型进行辅助分析
```

<!-- @end-section -->

<!-- @section: deployment -->
## 四、部署架构

### 4.1 组件部署拓扑

```
                    ┌──────────────┐
                    │  负载均衡器    │
                    │  (Nginx)     │
                    └──┬───┬───┬──┘
                       │   │   │
         ┌─────────────┼───┼───┼─────────────┐
         │             │   │   │             │
    ┌────┴────┐  ┌─────┴───┴───┴──────┐  ┌──┴──────────┐
    │ MaaS    │  │   Agent 引擎        │  │  LLM Wiki   │
    │ Service │  │   Service           │  │  Service    │
    │         │  │                     │  │             │
    │ :8080   │  │   :8081 (gRPC)      │  │  :8082      │
    │         │  │   :8080 (HTTP API)  │  │             │
    └────┬────┘  └─────────┬───────────┘  └──────┬──────┘
         │                 │                     │
         └─────────┬───────┴─────────────────────┘
                   │
         ┌────────┴────────┐
         │  PostgreSQL     │
         │  + pgvector     │
         │  + TimescaleDB  │
         └────────┬────────┘
                  │
         ┌────────┴────────┐
         │  NATS / Redis   │
         │  (事件总线+缓存)  │
         └─────────────────┘
```

### 4.2 技术选型

| 组件 | 技术 | 理由 |
|------|------|------|
| API 网关 | Nginx / Envoy | 反向代理、TLS 终止、限流 |
| MaaS Service | Go (stardust) | 高性能、原生并发 |
| Agent Service | Go (stardust) | 类型安全、goroutine 并发执行工具 |
| Wiki Service | Go (stardust) | 统一技术栈 |
| 数据库 | PostgreSQL + pgvector + TimescaleDB | 结构化 + 向量 + 时序 |
| 事件总线 | NATS | 高性能、持久化、支持多种模式 |
| 缓存 | Redis | 语义缓存、会话缓存 |
| 对象存储 | MinIO / S3 | 文件、附件、备份 |

### 4.3 为什么不拆成更多微服务

```
方案 A: 每个引擎一个服务 (3 个服务)
  ✅ 职责清晰
  ✅ 独立部署
  ✅ 团队可以独立工作

方案 B: 每个引擎再拆子服务 (10+ 个服务)
  ❌ 过度拆分 (认知内核 和 工具系统 不应是不同的服务)
  ❌ 增加网络延迟 (Agent 循环中频繁跨服务调用)
  ❌ 分布式事务复杂度

Legion 的选择: 方案 A — 三个服务，每个服务内部模块化但整体部署
```

<!-- @end-section -->

<!-- @section: technology-stack -->
## 五、完整技术栈

### 5.1 技术栈全景

```
┌─────────────────────────────────────────────────────────┐
│                      技术栈全景                           │
│                                                          │
│  语言层:                                                  │
│    后端: Go (stardust 基础库)                              │
│    前端: TypeScript + React 19 + Vite + Tailwind CSS v4   │
│    DSL: YAML (工作流定义)                                  │
│                                                          │
│  存储层:                                                  │
│    主数据库: PostgreSQL 16                                │
│    向量扩展: pgvector                                     │
│    时序扩展: TimescaleDB (审计日志/指标)                    │
│    缓存: Redis (语义缓存/会话/限流)                         │
│    对象存储: MinIO (文件/附件)                              │
│                                                          │
│  通信层:                                                  │
│    内部 API: gRPC (服务间高性能通信)                        │
│    外部 API: RESTful HTTP/JSON                            │
│    事件总线: NATS (pub/sub, 持久化, 流式)                   │
│    Agent 通讯: 消息队列 + WebSocket (实时推送)              │
│                                                          │
│  基础设施:                                                 │
│    容器化: Docker + docker-compose (开发/测试)              │
│    编排: Kubernetes (生产)                                 │
│    监控: Prometheus + Grafana                             │
│    追踪: OpenTelemetry                                    │
│    日志: Loki + Promtail                                   │
│                                                          │
│  安全:                                                    │
│    传输: TLS 1.3                                          │
│    认证: JWT + API Key                                    │
│    加密: AES-256-GCM (敏感字段)                            │
│    审计: 不可变 Hash 链                                    │
└─────────────────────────────────────────────────────────┘
```

### 5.2 为什么选 Go 而不是 Python/Rust/JavaScript

| 选项 | 优势 | 劣势 | Verdict |
|------|------|------|---------|
| Go | 高性能、原生并发、stardust 基础库、部署简单 | 泛型较新、生态不如 Python | ✅ 选择 |
| Python | hermes 验证了可行性、AI 生态好 | 性能瓶颈 (14KLOC 一个文件)、并发弱 | ❌ |
| Rust | claw-code 证明极致性能、内存安全 | 开发效率低、团队门槛高 | ❌ |
| JavaScript | evolver 就是用它的 | 核心混淆、动态类型风险 | ❌ |

<!-- @end-section -->

<!-- @section: principles -->
## 六、参考项目对 Legion 架构的验证

### 6.1 架构决策的参考项目支撑

| Legion 架构决策 | claw-code | new-api | hermes-agent | evolver | 支撑强度 |
|---------------|-----------|---------|-------------|---------|---------|
| 三层引擎分离 | ⚠️ | ✅ (网关独立) | ⚠️ (Agent+工具耦合) | ✅ (进化独立) | **强** |
| Transport 规范化 | ✅ (MCP) | ✅ (40+ Adaptor) | ✅ (ProviderTransport) | ❌ | **强** |
| Agent 自进化 | ❌ | ❌ | ❌ | ✅ (GEP) | **Evolver 独有** |
| 知识资产化 | ❌ | ❌ | ✅ (技能) | ✅ (Gene/Capsule) | **中** |
| 多 Agent 协作 | ❌ | ❌ | ⚠️ (delegate) | ✅ (ATP) | **各自不足** |
| 事件驱动通信 | ❌ | ⚠️ (SSE) | ⚠️ (WebSocket) | ⚠️ (HTTP) | **需要自研** |

> ✅ = 强支撑 | ⚠️ = 部分支撑 | ❌ = 无支撑

### 6.2 Legion 比所有参考项目都做得更好的地方

1. **三层引擎的清晰分离** — 所有参考项目都在某个维度上耦合了
2. **组织化多 Agent 协作** — 没有一个参考项目有完整的组织架构模型
3. **渗透式三原则治理** — 参考项目各自有部分安全机制，但没有系统性的治理框架
4. **人机共建的知识体系** — hermes 的技能是"人写 AI 读"，evolver 的资产是"Agent 写 Agent 读"，Legion 做到"人机共读共写"
5. **内建式进化机制** — evolver 是寄生式，Legion 是内建式

<!-- @end-section -->

<!-- @section: design-decisions -->
## 七、关键设计决策总表

| # | 决策 | 选择 | 参考依据 |
|---|------|------|---------|
| 1 | 引擎数量 | 3 个 (MaaS + Agent + Wiki) | Legion 架构方案 |
| 2 | 引擎间通信 | gRPC (同步) + NATS (异步) | 微服务最佳实践 |
| 3 | Agent 数 | 每服务 N 个 Agent goroutine | Go 原生并发 |
| 4 | 数据存储 | PostgreSQL (统一) | 简化运维 |
| 5 | 缓存 | Redis (语义缓存) | hermes 的 prompt caching 启示 |
| 6 | 审计 | 不可变 Hash 链 | evolver 的仅追加日志 |
| 7 | 部署 | Docker → K8s | 渐进式复杂度 |
| 8 | 前端 | React 19 SPA | hermes Web 仪表盘验证 |
| 9 | 语言 | Go | stardust + 性能 + 类型安全 |

### 架构演进路线图

```
Phase 1 (MVP): 三个引擎各自独立但可协作
  - MaaS: 基本路由 + 预算管控
  - Agent: 认知内核 + 工具系统 + 单 Agent
  - Wiki: 知识条目 CRUD + 搜索

Phase 2: 核心能力增强
  - MaaS: 智能路由 + 语义缓存
  - Agent: 记忆系统 + 进化学习 + 多 Agent 协作
  - Wiki: 知识图谱 + 混合检索 + 审核流程

Phase 3: 企业级特性
  - 多租户隔离
  - SSO 集成
  - 高级工作流编排
  - Agent 经济层 (可选)
```

### 不做什么

1. **不做单体应用** — 三个引擎独立可替换
2. **不做过度微服务** — 3 个服务是合理的粒度
3. **不做多语言混合** — Go 统一后端
4. **不做重依赖基础设施** — Docker Compose 可跑起来

<!-- @end-section -->

<!-- @section: related -->
## 相关文档

- [[../architecture/Legion|Legion 项目方案]] — 系统架构总纲
- [[01-maas-layer-deep-design|MaaS 模型调度层深度设计]]
- [[02-agent-runtime-deep-design|Agent 运行时深度设计]]
- [[03-evolution-learning-deep-design|进化学习系统深度设计]]
- [[04-wiki-knowledge-deep-design|LLM Wiki 知识引擎深度设计]]
- [[05-multi-agent-collaboration|多智能体协作深度设计]]
- [[06-security-governance|安全治理体系深度设计]]

<!-- @end-section -->
