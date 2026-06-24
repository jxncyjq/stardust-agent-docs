---
id: "architecture-agent-engine-design-nocode-001"
title: "Legion Agent 引擎设计方案（纯设计版）"
aliases: ["agent engine design nocode", "Legion引擎设计无代码版"]
type: "architecture"
category: "design/architecture"
tags: ["legion", "agent-engine", "maas", "llm-wiki", "adapter", "design", "architecture", "quality-gate", "trust-score", "eval"]
version: "2.0.0"
created: "2026-05-08"
updated: "2026-05-08"
author: "jxncyjq"
status: "draft"
related_docs:
  - id: "architecture-agent-engine-design-001"
    relation: "derived_from"
    path: "./agent-engine-design.md"
  - id: "architecture-legion-001"
    relation: "parent"
    path: "./Legion.md"
---

<!-- @section: overview -->
# Legion Agent 引擎设计方案（纯设计版）

## 文档说明

本文档是 **[[agent-engine-design|agent-engine-design.md]]** 的纯设计版本，保留所有架构决策、设计原则与组件说明，去除全部伪代码。适合快速阅读设计意图、做方案评审、以及向非工程背景的干系人传达架构思路。

本文档是 **Legion V3.0 架构**（[[Legion|Legion.md]]）的落地设计方案，以三原则（可观测·可治理·可控风险）为顶层约束，系统性地从五个参考项目中提取最优设计，映射到 Legion 的三大引擎（MaaS / Agent / LLM Wiki）与 Adapter 层。

### 五个参考项目对 Legion 的贡献定位

| 项目 | 语言 | 最大贡献领域 | 核心借鉴点 |
|------|------|------------|----------|
| **Claw Code** | Rust | 权限执行框架 | permission_enforcer + bootstrap 12阶段 + LSP 集成 |
| **DeepSeek-TUI** | Rust | 生产级流式管道 | SSE 管道 + 审批策略引擎 + 循环检查点 + Crate 分层 |
| **Hermes Agent** | Python | Agent 运行时完整度 | NormalizedResponse + 错误分类学 + 上下文压缩 + 工具注册 |
| **Evolver** | JavaScript | 自我进化形式化 | GEP 三层信号 + Gene/Capsule/Event 资产 + 固化流程 |
| **Mission Control** | TypeScript | Agent 舰队编排控制 | Aegis 质量门控 + Trust Score + 四层 Eval + 调度器架构 + 技能安全扫描 |

### 设计方法论

Legion.md 的核心思想是：**不让 AI 工具运作，而让 AI 团队运作**。这要求每一个组件设计都必须同时满足：

- `[可观测]`：每个决策、每次调用、每条记忆都可追溯
- `[可治理]`：人类在任何时刻都能介入、审核、纠偏
- `[可控风险]`：自主行为始终在预设安全边界内

<!-- @end-section -->

---

<!-- @section: maas-design -->
## 一、MaaS 模型调度层设计

### 1.1 设计目标回顾（来自 Legion.md §3.1）

MaaS 层的职责是将"模型选择"从 Agent 定义中解耦，由平台统一调度。关键需求：
- 按角色/任务/预算三维路由策略
- 五级配额（公司→部门→团队→Agent→模型）
- 三级预算熔断（告警→降级→冻结）
- 完整的调用链路追踪

### 1.2 传输规范化层 —— 来自 Hermes

Hermes 最优雅的设计是 `ProviderTransport` ABC + `NormalizedResponse`：

```
Agent 循环 ← NormalizedResponse ← ProviderTransport ← [OpenAI|Anthropic|DeepSeek|...]
```

**Legion 采纳方案**：定义 `ModelProvider` trait（Rust）将所有 LLM API 差异在传输层消除。`ModelProvider` 提供四个接口：同步调用（`call`）、流式调用（`stream`）、能力查询（`capabilities`）、健康状态（`health`）。

统一响应结构 `ModelResponse` 包含：统一内容块（`content`）、统一工具调用格式（`tool_calls`）、统一用量统计（`usage`）、实际使用的模型（`model_used`，路由后）、响应延迟（`latency_ms`）、路由溯源（`provider_id`）。

`[可观测]` `model_used` + `provider_id` 字段确保每次调用都记录了实际路由决策，而非计划路由。

### 1.3 SSE 流式管道 —— 来自 DeepSeek-TUI

DeepSeek-TUI 的 SSE 管道是目前分析的五个项目中最完整的生产实现：

```
字节流 → 缓冲(8MB限) → 行解析 → JSON解析 → 类型安全事件
   ↑            ↑          ↑          ↑
 背压控制    分批处理    CRLF兼容   状态追踪
(高水位警戒) (256行/批)  处理      (content_index)
```

**Legion 采纳方案**：SSE 解析器封装为有状态对象 `SseParser`（修正了 DeepSeek-TUI 的函数签名问题），维护解析状态（`content_index`、`text_started`、`thinking_started`、`tool_indices`、高水位标记、批次大小）。

**关键机制保留**：
- 高水位背压控制（8MB → 暂停消费）
- 分批行处理（256行/批，防止 UI 卡顿）
- 空闲超时（5分钟无数据 → 断开重连）
- 流停滞透明重试（最多 3 次，用户无感知）

### 1.4 错误分类与自动恢复 —— 来自 Hermes

Hermes 的 `FailoverReason` 枚举 + 恢复处理是最完整的错误分类实践。

**Legion `LlmError` 枚举及恢复动作**：

| 错误类型 | 分类 | 恢复动作 |
|---------|------|---------|
| `RateLimit` | 可重试 | `RetryWithBackoff`（指数退避） |
| `Timeout` | 可重试 | `RetryWithBackoff` |
| `ServiceUnavailable` | 可重试 | `RetryWithBackoff` |
| `ContextWindowExceeded` | 触发压缩 | `CompressContext`（再重试） |
| `AuthFailed` | 触发凭证轮换 | `RotateCredential` |
| `QuotaExceeded` | 触发故障转移 | `Fallback`（换提供商；配额耗尽属账户级别问题，轮换密钥无效——同一账户下所有密钥共享相同配额） |
| `ModelNotAvailable` | 触发故障转移 | `Fallback`（换提供商） |
| `ProviderError` | 触发故障转移 | `Fallback` |
| `InvalidRequest` | 不可恢复 | `Fail` |
| `ContentFiltered` | 不可恢复 | `Fail` |

`[可治理]` 错误分类后的恢复动作（降级到哪个模型、如何压缩）均可在配置中调整，管理员可以覆盖默认行为。

### 1.5 连接健康状态机 —— 来自 DeepSeek-TUI

DeepSeek-TUI 的三态健康状态机直接对应 Legion.md 的三级熔断需求：

```
Healthy (正常)
   │ (30s 内连续 3 次失败)
   ▼
Degraded (降级)
   │ (15秒后，发送探针)
   ▼
Recovering (恢复中)
   ├── (探针成功) → Healthy
   └── (探针失败) → Degraded (重计时器)
```

**连接健康三态与 Legion 配额语义的映射**：

| 状态 | Legion 配额语义 | 触发条件 |
|------|--------------|---------|
| `Healthy` | 正常路由 | 默认状态 |
| `Degraded` | 可用性告警 → 强制降级 | 30s 内连续 3 次调用失败（单次网络抖动不触发，防止过度敏感） |
| `Recovering` | 探针验证中 | 等待 15s 后发送轻量探针请求 |

> ⚠️ **`Frozen`（预算冻结）是独立的预算熔断概念，不属于连接健康状态机**。连接健康描述**提供商可用性**，预算熔断描述**消费配额耗尽**，两者正交：提供商可用但预算耗尽时触发 Frozen，不影响健康状态机流转。预算熔断状态由 §1.6 `MultiDimensionalRateLimit` 的 `per_company` 令牌桶管理，触发条件：`budget_remaining < 0`；退出条件：管理员手动解除或下一计费周期刷新。

`[可控风险]` 连接健康状态机确保模型不可用时自动降级而非中断业务，同时给予自动恢复机会；预算熔断独立于可用性，两套机制互不干扰。

### 1.6 速率限制 —— 来自 DeepSeek-TUI

DeepSeek-TUI 的 `TokenBucket` 令牌桶直接用于 Legion 的每提供商限速。Legion 在此基础上扩展为**多维度令牌桶** `MultiDimensionalRateLimit`，同时维护三个维度的令牌桶：按提供商（`per_provider`）、按 Agent（`per_agent`）、按公司（`per_company`，配额管控）。

### 1.7 智能路由引擎实现

`IntelligentRouter` 组合了 `ModelRegistry`（模型注册表）、`MultiDimensionalRateLimit`（速率限制）、`HealthMonitor`（健康监控）和 `RoutingConfig`（路由配置）四个组件。

路由入口逻辑：
- **有角色上下文**（正常 Agent 任务）→ 委托 `route_with_role()`（§1.8.5，10步完整路径）
- **无角色上下文**（系统任务、预热请求）→ 简化路径：Standard Tier + 健康过滤 + 速率限制

`[可观测]` 每一步路由决策记录到追踪日志：候选集合、过滤原因、最终选择、选择依据。角色感知路径的完整 10 步追踪见 §1.8.5 `RoutingTrace`。

---

### 1.8 角色-模型路由矩阵 —— Legion 核心差异化

#### 1.8.1 设计背景

Legion.md §3.1.1 明确指出："CEO Agent 做战略决策需要最强推理模型，客服 Agent 做日常回复用轻量模型即可。" 这要求 MaaS 层为每一类角色建立精细的**模型路由档案**。

四个参考项目都回避了这个问题的深度：Hermes 是"静态选择 + 故障转移"，DeepSeek-TUI 是手动配置单个模型，Claw Code 是用户手动切换，Evolver 不涉及多模型。Legion 在此做出真正的差异化：**每个角色的每类任务，都有精确的模型分配策略**。

#### 1.8.2 模型等级定义

在路由策略中使用**能力等级（Tier）**而非具体模型名称，解耦路由逻辑与具体模型版本：

| Tier | 定位 | 适用场景 | 参考模型 |
|------|------|---------|---------|
| **Light（轻量级）** | 低延迟、低成本 | 格式转换、简单分类、模板填充、简短回复 | deepseek-chat / qwen-turbo / gpt-4o-mini |
| **Standard（标准级）** | 均衡能力 | 内容创作、代码实现、数据分析、日常推理 | claude-sonnet-4 / deepseek-coder / gpt-4o |
| **Advanced（高级）** | 强推理能力 | 复杂架构设计、深度分析、跨文档综合判断 | claude-sonnet-4-extended / gpt-4o-reasoner |
| **Flagship（旗舰级）** | 最强推理，成本最高 | 战略决策、创新性问题、高风险评审 | claude-opus-4 / deepseek-reasoner |

每个 Tier 可附加能力约束：最小上下文窗口（`min_context_window`）、是否需要工具调用能力（`require_tool_use`）、是否需要结构化输出（`require_structured_output`）、延迟预算（`latency_budget_ms`，客服场景严格）。

#### 1.8.3 角色路由档案

Legion.md §4.2.1 定义了多类 Agent 角色，每类角色的模型需求差异显著。以下为完整路由档案：

| 角色 | 默认 Tier | 典型任务覆盖（override） | fallback 链 | 成本敏感度 | 特殊能力要求 |
|------|---------|----------------------|------------|-----------|-----------|
| `ceo` | Flagship | 状态审查→Standard；战略规划→Flagship；资源分配→Advanced；常规审批→Standard | Flagship→Advanced→Standard | 低 | — |
| `cto` | Advanced | 架构评审→Flagship；代码评审→Advanced；技术路线→Flagship；日常站会→Standard | Advanced→Standard→Light | 中 | — |
| `project_manager` | Standard | 风险评估→Advanced；任务拆解→Standard；进度报告→Light | Standard→Light | 中 | — |
| `architect` | Flagship | 系统设计→Flagship；性能分析→Advanced；代码生成→Advanced；文档→Standard | Flagship→Advanced→Standard | 低 | code, structured_output |
| `backend_developer` | Standard | 复杂算法→Advanced；CRUD→Light；Bug修复→Standard；代码评审→Advanced；单元测试生成→Light | Standard→Light | 高 | code |
| `frontend_developer` | Standard | 复杂UI逻辑→Advanced；组件生成→Light；CSS样式→Light；性能调试→Advanced | Standard→Light | 高 | code |
| `devops` | Standard | 故障分析→Advanced；CI/CD→Standard；常规部署→Light；安全审计→Advanced | Standard→Light | 高 | — |
| `product_manager` | Standard | 市场调研→Advanced；需求分析→Standard；竞品分析→Advanced；线框描述→Light | Standard→Light | 中 | — |
| `ui_designer` | Standard | 设计概念→Advanced；设计评审→Standard；素材描述→Light | Standard→Light | 中 | vision |
| `ux_researcher` | Standard | 用户洞察综合→Advanced；调研设计→Standard；数据编码→Light | Standard→Light | 中 | — |
| `content_editor` | Standard | 长篇文章→Advanced；短文案→Light；内容审核→Light；SEO优化→Light | Standard→Light | 高 | — |
| `customer_service` | Light | 复杂投诉→Standard；常规FAQ→Light；退款决策→Standard；升级→Advanced | Light→Standard | 极高 | latency≤2000ms |
| `social_media_operator` | Light | 营销策略→Standard；内容发布→Light；危机公关→Advanced | Light→Standard | 极高 | — |
| `qa_engineer` | Standard | 测试用例生成→Light；Bug分析→Standard；安全测试→Advanced；回归测试→Light | Standard→Light | 高 | code |
| `security_auditor` | Advanced | 漏洞分析→Flagship；合规检查→Advanced；常规扫描→Standard；渗透测试→Flagship | Advanced→Standard | 低 | — |
| `data_analyst` | Standard | 复杂统计分析→Advanced；仪表板生成→Light；数据清洗→Light；异常检测→Advanced | Standard→Light | 中 | code, structured_output |
| `bi_engineer` | Standard | 数据建模→Advanced；ETL流程→Standard；查询优化→Advanced；常规报告→Light | Standard→Light | 高 | code |

#### 1.8.4 任务复杂度动态评估

同一角色执行相同类型的任务，复杂度不同时应动态调整模型选择。`TaskComplexityScorer` 从五个维度打分，输出 Tier 推荐：

| 维度 | 区间 | 得分 |
|------|------|------|
| 输入规模（token 估算） | ≤1K / ≤8K / ≤32K / >32K | 0/1/2/3 |
| 工具调用深度（预期轮次） | ≤1 / ≤5 / ≤15 / >15 | 0/1/2/3 |
| 跨文档综合（上下文文档数） | ≤2 / ≤10 / >10 | 0/1/2 |
| 决策风险等级 | Low / Medium / High / Critical | 0/1/2/3 |
| 需要创意/开放性推理 | 否 / 是 | 0/2 |

**总分 → Tier 推荐**：0~2 → Light；3~5 → Standard；6~8 → Advanced；≥9 → Flagship。

**"工具调用深度（预期轮次）"的预估来源**（执行前）：优先级从高到低：① 任务提交方在请求中通过 `expected_tool_rounds` 字段显式声明；② 若未声明，则通过 `EpisodicMemoryStore` 检索同一 Agent 历史相似任务（余弦相似度 > 0.7）的实际工具调用轮次，取中位数；③ 若无历史记录，则按任务类型（`task_type` 字段）使用预置默认值（如 `code_review` 默认 3 轮，`data_analysis` 默认 5 轮）。预估值仅影响初始 Tier 选择，执行中如实际轮次超出预期，`ContextCompressor` 的 80% 触发阈值会自动接管。

路由时取角色档案 Tier 与复杂度推荐 Tier 两者的**较高值**（复杂度可向上推升，不可向下压降）。

#### 1.8.5 完整路由决策流水线

`route_with_role()` 将角色档案、任务复杂度、预算约束、健康状态全部融合为一条 10 步决策流水线：

1. **确定基础 Tier**：角色档案 × 任务类型（task_overrides 匹配，未匹配则用 default_tier）
2. **复杂度动态修正**：取基础 Tier 与复杂度推荐 Tier 中的较高值
3. **Agent 级上限约束**：`model_policy.max_model_tier` 硬上限截断
4. **预算感知降级**：读取 `BudgetContext`（含 `budget_remaining_ratio`、`monthly_spend_ratio`，来自 §5.1 四级组织数据模型中 Agent 的 `budget` 配置）——当 `budget_remaining_ratio < 20%` 时将当前 Tier 强制降一级（如 Advanced → Standard），保留预算用于后续更高优先级任务；`budget_remaining_ratio < 5%` 时降两级
5. **特殊能力过滤**：筛选满足角色 + 任务能力要求的候选模型集
6. **延迟预算过滤**：客服等场景按 `latency_budget_ms` 过滤高延迟模型
7. **健康检查过滤**：排除当前 Degraded/Recovering 的提供商
8. **fallback 降级**：无健康候选时按 `fallback_chain` 逐级降级重选
9. **成本敏感度选择**（对应 §1.8.3 角色档案中的"成本敏感度"列）：极高 → 同 Tier 绝对最低价；高 → 同 Tier 最便宜；中 → 质价均衡；低 → 同 Tier 最高质量
10. **构建路由追踪记录**：`RoutingTrace` 记录每个中间状态（基础Tier来源、是否复杂度推升、是否预算压降、是否触发 fallback）

`[可观测]` `RoutingTrace` 记录了路由决策的每一个中间状态，管理者可在仪表盘中精确查看每次调用为何使用了该模型。

`[可治理]` 三处硬约束均可被管理员覆盖：①`model_policy.max_model_tier`（Agent 级上限）；②`routing_config.role_overrides`（临时覆盖某角色路由策略）；③`routing_config.force_bind[agent_id]`（强制某 Agent 使用指定模型）。

#### 1.8.6 工作流中的多 Agent 模型分工

在 Legion 的工作流场景下，一个流程中多个不同角色的 Agent 协作时，各自的模型选择策略形成互补的**模型分工格局**：

```
需求到上线全流程 — 模型分工示意

┌─────────────────────────────────────────────────────────────────┐
│  工作流节点          Agent 角色          路由 Tier    模型示例     │
├─────────────────────────────────────────────────────────────────┤
│  收集需求            product_manager     Standard    sonnet-4    │
│  ─────────────────── ✋ 高影响力需求审批门控 ───────────────────  │
│  技术方案设计        architect           Flagship    opus-4      │ ← 最贵
│  UI 原型设计         ui_designer         Standard    sonnet-4    │
│  ─────────────────── ⊕ 并行汇聚等待 ─────────────────────────── │
│  后端开发            backend_developer   Standard    deepseek    │
│    - 复杂算法模块                        Advanced    sonnet-ext  │ ← 动态升级
│    - CRUD 接口                           Light       qwen-turbo  │ ← 动态降级
│  前端开发            frontend_developer  Standard    sonnet-4    │
│  ─────────────────── ✋ 预算门控 ─────────────────────────────── │
│  测试执行            qa_engineer         Light       qwen-turbo  │ ← 批量测试轻量
│    - 安全测试                            Advanced    sonnet-ext  │ ← 安全升级
│  部署上线            devops              Standard    deepseek    │
└─────────────────────────────────────────────────────────────────┘

整个流程：旗舰级(1次) + 高级(3次) + 标准级(n次) + 轻量级(m次)
成本优化效果：相比全程使用旗舰级，预计节省 60-75% 的模型成本
```

#### 1.8.7 角色路由的运行时覆盖接口

`[可治理]` 管理员可以在不修改配置文件的情况下，通过管理 API 实时调整路由策略。支持三类覆盖操作：

- **ForceModel**：强制某 Agent 始终使用指定模型（含过期时间，必填理由字段，写入审计日志）
- **AdjustRoleTier**：临时调整某角色的默认 Tier（`tier_delta` = ±1，指定公司/部门/团队范围，含过期时间）
- **BanModel**：禁止使用某个模型（如模型下线或安全事件），指定 fallback 模型

覆盖范围（`OverrideScope`）对应五级组织层次中的上三层：Company / Department / Team。Agent 级和模型级覆盖通过 ForceModel / BanModel 实现。

`[可控风险]` 所有运行时覆盖操作写入不可篡改的审计日志，包含操作者身份、理由、生效时间范围，保证可追溯且防止滥用。

<!-- @end-section -->

---

<!-- @section: agent-engine-design -->
## 二、AI Agent 引擎设计

### 2.1 设计目标回顾（来自 Legion.md §3.2）

Legion 的 Agent 引擎的核心差异化在于**自然进化**：Agent 在工作中积累经验、从反馈中修正行为、越用越强。设计关键：
- 认知内核：7 要素上下文组装
- 三层记忆：工作/情景/能力
- 四路学习：反馈/反思/协作观察/知识库驱动
- 进化评估器：5 维度量化追踪

### 2.2 Crate 分层架构 —— 来自 DeepSeek-TUI

DeepSeek-TUI 的 Cargo 工作区分层是整个分析中**最重要的架构决策**，Legion 直接采纳：

```
legion-protocol      [Layer 0] 纯数据结构：Message/Tool/Event/Request/Response
legion-config        [Layer 0] 配置加载：TOML + 环境变量 + 优先级合并
legion-model         [Layer 0] 数据库模型：SQLite schema + 查询
legion-state         [Layer 0] 运行时状态：Agent状态机 + 任务状态

legion-tools         [Layer 1] 工具原始类型：ToolSpec + ToolInvocation
legion-skills        [Layer 1] 技能系统：可插拔技能 + 运行时注入
legion-execpolicy    [Layer 1] 执行策略：ApprovalPolicy + SandboxMode
legion-hooks         [Layer 1] 生命周期钩子：pre-turn/post-turn
legion-mcp           [Layer 1] MCP 客户端：stdio/HTTP 协议

legion-memory        [Layer 2] 记忆系统：工作/情景/能力三层
legion-learning      [Layer 2] 学习引擎：GEP信号提取 + Gene资产
legion-evolution     [Layer 2] 进化评估：5维度指标 + 成长报告

legion-maas          [Layer 3] MaaS 调度：路由 + 配额 + 熔断
legion-core          [Layer 3] Agent 引擎：认知内核 + 执行循环

legion-wiki          [Layer 4] LLM Wiki：知识管道 + 混合检索
legion-gateway       [Layer 4] HTTP API：REST + WebSocket + SSE

legion-app-web       [Layer 5] Web 应用：React 前端服务
legion-app-cli       [Layer 5] CLI 工具：开发者调试命令
legion-app-mobile    [Layer 5] 移动端（预留，iOS/Android）
```

**关键约束**：每层只能依赖下层或同层，绝对禁止循环依赖。`legion-protocol` 作为零依赖的纯数据层，所有其他 crate 共享其类型定义。

**同层单向依赖规则**：同一 Layer 内允许单向依赖，但反向禁止。唯一受认可的同层依赖：
- `legion-core`（Layer 3）→ `legion-maas`（Layer 3）：Agent 引擎通过 MaaS 层发起 LLM 调用
- 反向严格禁止：`legion-maas` 不得感知 `legion-core` 的存在，MaaS 层只提供路由/配额/熔断服务，不引用具体 Agent 类型

### 2.3 认知内核（Cognitive Core）—— Legion 原创 + 四项目融合

Legion.md §3.2.2 定义了认知内核的 7 要素公式：

```
执行上下文 = 角色人设
           + 当前任务指令
           + 目标传导链（为什么做）
           + 相关经验记忆（以前类似任务怎么做的）
           + 挂载技能（能用什么工具）
           + 组织上下文（我在团队中的位置、协作者是谁）
           + 约束规则（预算限制、权限边界、安全红线）
```

**设计关键**：这不是 Prompt 拼接，而是有优先级的上下文编排。`CognitiveCore` 持有以下组件：

- **`agent_def: Arc<AgentDef>`**：Agent 定义（角色人设、约束规则、能力声明），构造时注入，对话期间不可变
- **`memory: Arc<MemorySystem>`**：三层记忆系统（§2.8），检索任务相关的情景记忆和能力记忆
- **`skills: Arc<SkillSystem>`**：技能加载器（§4.5），运行时按任务注入可用技能
- **`context_compressor: ContextCompressor`**：上下文压缩器的所有权在此，`cycle_count` 由它内部维护（§2.6）
- **`system_prompt_locked: AtomicBool`**：提示缓存稳定性标志，`AtomicBool` 允许通过 `&self` 修改；首次 `assemble_context()` 完成后设为 `true`，此后 P0（安全红线）与 P1（角色人设）直接从缓存读取，不重新拼接——防止系统提示哈希在对话中途变动、破坏 Anthropic 提示缓存并引发成本飙升
- **`system_prompt_hash: u64`**：系统提示哈希，用于提示缓存命中检测
- **`init_phase: InitPhase`**：来自 Claw Code 的 bootstrap 阶段结构化初始化状态

`assemble_context()` 按优先级编排 7 要素（P0 最高 → P6 最低），最后执行冲突解决：

| 优先级 | 内容 | 来源 |
|--------|------|------|
| P0 安全红线 | 约束规则（永远最高，不可覆盖） | `agent_def.constraints` |
| P1 角色人设 | 对话期间锁定（AtomicBool 幂等写入） | `agent_def.persona` |
| P2 组织上下文 | 团队位置、协作者 | `req.org_context` |
| P3 目标传导链 | 为什么做这个任务 | `req.goal_chain` |
| P4 当前任务指令 | 具体任务描述 | `req.task` |
| P5 相关经验记忆 | 情景记忆检索（TopK=5） | `memory.retrieve_relevant` |
| P6 挂载技能 | 运行时注入可用工具 | `skills.load_for_task` |

`force_cycle_checkpoint()` 方法：SoftLoop 触发时调用，委托给 `context_compressor.force_checkpoint()` 强制执行循环检查点（忽略 80% 阈值，立即执行）。此方法需要 `&mut self`（修改消息历史）。

`[可控风险]` 安全红线（P0）通过优先级系统设计上保证不可被任务指令覆盖，而非依赖 LLM 的理解。

### 2.4 Agent 主循环 —— 来自 Hermes

**Crate 归属**：`legion-core`（Layer 3）。`AgentRuntime` 是**单次任务执行引擎**，只负责 LLM 调用循环本身——不关心任务来源、不持有锁、不写审计链，这些由 §5.2 的 `AgentCoordinator` 负责。两者的边界：

```
AgentCoordinator（任务编排层）
  │  负责：心跳协议、原子锁、任务状态流转、审计链写入、路由决策
  │  调用 ↓（传入 Task + RoutingDecision，路由已在此层完成）
AgentRuntime（执行层）
  │  负责：LLM 循环、工具执行、上下文组装、流式推送
  │  输入：Task（已锁定）+ RoutingDecision（已选定模型）  输出：TaskResult
```

`AgentRuntime` 持有的组件：
- **`cognitive_core: CognitiveCore`**：上下文组装与压缩
- **`tool_registry: Arc<ToolRegistry>`**：工具注册与执行
- **`tool_guardrails: ToolGuardrails`**：来自 Hermes，工具执行看门狗
- **`memory: Arc<MemorySystem>`**：记忆系统
- **`maas_client: Arc<MaasClient>`**：仅用于 LLM 流式调用，不做路由决策
- **`eval_engine: Arc<EvalEngine>`**：Layer 2 收敛比检测（§2.12）
- **`permission_enforcer: Arc<PermissionEnforcer>`**：来自 Claw Code，权限检查
- **`event_bus: Arc<EventBus>`**：流式 token 推送到 UI

**执行上下文的跨迭代积累**：`run_task()` 在**迭代循环外**初始化消息历史列表（`message_history`）并调用一次 `cognitive_core.assemble_context()` 注入静态元素（角色人设、组织上下文、约束规则等）。每次迭代只向已有历史中**追加**新内容——LLM 回复与工具调用结果——而非重建。这样 LLM 在每轮迭代都能看到完整的前序工具调用链，支持多轮推理。`[可观测]` 消息历史与上下文压缩记录一并写入 `EpisodicMemoryStore`（`post_task_learning` 步骤）。

`run_task()` 采用 `&mut self`（`ContextCompressor.cycle_count` 需要跨迭代持久化）。循环内的防护机制：

1. **迭代预算检查**（来自 Hermes `iteration_budget`）：超出 `max_iterations` → **先向 `event_bus` 发布轻量 `LearningEvent(signal: Failure, reason: BudgetExhausted)`**（不写 EpisodicMemory 完整轨迹，避免记录不完整状态）→ 返回 `BudgetExhausted`
2. **中断检查**：用户可随时发出中断信号 → **先发布轻量 `LearningEvent(signal: Failure, reason: Interrupted)`** → 返回 `Interrupted`
3. **流式 LLM 调用**：使用 `routing.selected_model`（已由 AgentCoordinator 确定），每个 token delta 推送到 `event_bus`
4. **工具执行链**：`tool_guardrails.before_call` → `permission_enforcer.check_batch` → `check_approval` → 并发执行 → `tool_guardrails.after_call`
5. **Layer 2 收敛比检测**（每次工具执行后实时检测，§2.12）：`eval_engine.eval_trace()` → 匹配 `LoopingStatus`：
   - `SoftLoop`：触发循环检查点，**同时向 `tasks` 表写入 `soft_loop_reset_at = now()`**；后续 `eval_trace()` 调用自动过滤 `soft_loop_reset_at` 之前的 `mcp_call_log` 记录，防止旧调用数据持续触发 SoftLoop（与 §6.2.1 HardLoop 的 `loop_reset_at` 机制完全对称）
   - `HardLoop`：返回 `AnomalyDetected`（强制中断）
6. **工作记忆更新**（参考 DeepSeek-TUI 循环检查点）
7. **无工具调用时**：触发 `post_task_learning` 后返回 `Completed`。`post_task_learning` 完成两件事：① 将本次执行的情景记忆（任务描述、工具调用序列、执行结果）写入 `EpisodicMemoryStore`；② 向 `event_bus` 发布 `LearningEvent`（含任务结果与信号分类：`Success` / `Failure`），供 `gep_failure_scan` 后台任务异步消费——GEP 周期由此触发，而非由 `run_task()` 内部直接调用（保持执行层与学习层解耦）

### 2.5 批准策略引擎 —— 来自 DeepSeek-TUI

DeepSeek-TUI 的 `ApprovalPolicy × SandboxMode` 正交设计直接满足 Legion 的工具执行风险管控。

**`ApprovalPolicy`（批准策略）**：

| 策略 | 语义 |
|------|------|
| `AutoAllow` | 所有工具自动允许（受 `auto_allow` 列表约束） |
| `OnRequest` | 默认允许，但特定工具需批准 |
| `Untrusted` | 默认拒绝，每次需要显式批准 |
| `Never` | 禁止所有工具执行 |

**`SandboxMode`（沙盒模式）**：

| 模式 | 语义 |
|------|------|
| `ReadOnly` | 只读模式：禁止所有写操作 |
| `WorkspaceWrite` | 工作区写：只能修改项目目录内文件 |
| `DangerFullAccess` | 完全访问：所有操作（含系统命令） |
| `ExternalSandbox` | 外部沙盒：在隔离容器中执行 |

Legion 扩展 `ExecutionPolicy` 增加 RBAC 集成：`role_overrides`（不同角色有不同默认批准策略）和 `audit_all`（记录每次工具执行审计日志）。

`[可治理]` 不同角色的 Agent 拥有不同的默认批准策略：CEO Agent 可以自动允许更多工具，新入职 Agent 的沙盒更严格。

`[可控风险]` `auto_allow_prefixes` 配置驱动，管理员无需发布新版本即可调整允许范围；`ExternalSandbox` 将危险操作完全隔离到容器中。

### 2.6 上下文压缩 —— 来自 Hermes + DeepSeek-TUI

综合 Hermes 的 4 层压缩算法和 DeepSeek-TUI 的循环检查点机制。`ContextCompressor` 持有：压缩策略（`strategy`）、最大检查点轮次（`cycle_length`，构造时由 `CognitiveCore.max_cycle_length` 传入）、当前轮次计数（`cycle_count`，由 `compress()` 内部维护，到达 `cycle_length` 后重置）、循环摘要列表（`cycle_briefings`）。

**四层压缩策略（`CompressionStrategy`）**：

| 层级 | 策略 | 机制 | 成本 |
|------|------|------|------|
| 第 1 层 | `TrimOldToolOutputs` | 裁剪旧工具输出，保留最新 N 条 | 零 LLM 调用 |
| 第 2 层 | `ProtectHeadTail` | 保护首尾消息，截断中间 | 零 LLM 调用 |
| 第 3 层 | `AuxiliarySummary` | 辅助模型（轻量级）摘要中间历史 | 低 LLM 调用 |
| 第 4 层 | `CycleCheckpoint` | 检查点摘要替代全部历史，重置计数 | 低 LLM 调用 |

`compress()` 执行逻辑：始终先执行第 1 层（零成本裁剪）；若仍过大，按当前 `strategy` 选择后续路径。`should_compress()` 以 80% 窗口利用率为触发阈值。

`[可观测]` 每次压缩记录：压缩策略、压缩前后 Token 数、摘要内容、压缩原因，管理者可查看 Agent 的上下文管理历史。

### 2.7 学习引擎 —— 来自 Evolver GEP

Evolver 的 GEP（Genome Evolution Protocol）正是 Legion 学习引擎的核心实现参考，直接映射到 Legion.md §3.2.4 的四路学习路径。其形式化理论基础来自论文《From Procedural Skills to Strategy Genes: Towards Experience-Driven Test-Time Evolution》（Wang et al., 2026，arXiv:2604.15097），在 4,590 次实验中验证了 Strategy Gene 表示方法的有效性：**单个 Gene（~230 tokens）比 baseline 提升 +3.0pp，而过于详细的 Procedural Skill（~2,500 tokens）反而下降 -1.1pp**。

**2.7.0 形式化定义 — Strategy Gene 六元组**

**Gene 六元组 g = (m, u, π, α, c, v)**（论文 §3 核心定义）：

| 符号 | 字段名 | 含义 | 实验贡献 |
|------|--------|------|---------|
| m | matching signals | 任务识别触发关键词 | 任务召回精度 |
| u | compact summary | 高层意图描述（~50 tokens） | 上下文压缩 |
| π | strategy steps | 有序过程指导（核心控制逻辑） | 主要提升来源 |
| **α** | **AVOID cues** | **失败模式编码，禁止动作列表** | **+4.6pp（最高单字段贡献）** |
| c | constraints | 可选执行边界（爆炸半径限制） | 安全隔离 |
| v | validation hooks | 可选可执行验证检查 | 质量门控 |

> ⚠️ **AVOID 线索（α）是 Gene 中价值最高的字段**：编码「不应该做什么」比编码「应该做什么」对性能提升更大（+4.6pp）。失败模式的泛化能力远强于成功模式，因此 Gene 模板设计时 α 字段应强制非空。

**Capsule 六元组 κ = (q, Gκ, Tκ, oκ, Vκ, ℓκ)**（执行证明记录）：

| 符号 | 含义 |
|------|------|
| q | 任务签名（task signature，用于检索匹配） |
| Gκ | 本次执行使用的 Gene 集合 |
| Tκ | 完整执行轨迹（execution trace） |
| oκ | 执行结果（outcome） |
| Vκ | 验证记录（validation record） |
| ℓκ | 血统指针（lineage pointer → 父 Capsule） |

**进化事件九元组 e = (t, ρ, a_src, a_dst, σ, ι, Δ, ν, τ)**（不可变审计记录）：

| 符号 | 含义 |
|------|------|
| t | 事件类型（distill / mutate / crossover / repair） |
| ρ | 父运行 ID（parent run ID，溯源链） |
| a_src / a_dst | 源资产 / 目标资产 ID（变更前后） |
| σ | 触发信号（trigger signal） |
| ι | 变更意图描述（intent） |
| Δ | 内容差异（diff，JSON patch 格式） |
| ν | 验证结果（validation result） |
| τ | 时间戳 |

**2.7.1 三层信号提取（对应路径一：任务反馈学习）**

`SignalExtractor` 采用三层递进架构，按优先级顺序执行：

- **第 1 层：正则匹配**（确定性，0ms）——零成本，优先执行，覆盖明确模式
- **第 2 层：关键词打分**（统计，0ms）——补充正则未覆盖的隐含信号
- **第 3 层：LLM 语义分析**（限速，每 5 轮）——发现新信号，成本受 `RateLimiter` 控制

**后处理逻辑**：最近 **8 次 GEP 周期**（`gep_failure_scan` 每次触发一个周期）内出现 ≥ 3 次的相同信号被抑制（防止对同一失败模式重复生成 Gene 的修复循环）；检测到修复循环时强制推送 `Signal::ForceInnovate`，引导产生差异化 Gene；最终按优先级排序（可操作信号 > 表面信号）。

**2.7.2 进化资产三元模型（对应路径二：自我反思学习）**

三类资产的核心字段设计：

**Gene（策略资产）**：`id`（策略ID）、`asset_id`（SHA-256内容寻址）、六元组字段（m/u/π/α/c/v，**六元组内容总长度 ≤ 230 tokens**——这是对 Gene 有效载荷的整体 token 预算约束，不是 asset_id 哈希的限制）、元数据（`created_by`、`version`、`success_rate` 累计成功率、`usage_count` 使用次数，高频 Gene 优先注入，上限 3个/任务）。

**Capsule（执行证明）**：`id`、`asset_id`（SHA-256）、六元组字段（q/Gκ/Tκ/oκ/Vκ/ℓκ）、额外元数据（`success_count`、`environment` 环境快照、`created_by`）。

**EvolutionEvent（不可变审计）**：`id`、`asset_id`（SHA-256）、九元组字段（t/ρ/a_src/a_dst/σ/ι/Δ/ν/τ）、Legion 扩展字段（`agent_id`、`immutable_seal: Ed25519Signature`，防止回溯修改）。

`[可观测]` `EvolutionEvent` 的 `rho_parent_run`（ρ 溯源字段）链接形成完整的进化历史链，管理者可以追溯每一步学习决策。

`[可治理]` Gene 的 `v_validation` 字段确保每个学习规则在写入能力记忆前通过验证；管理员可以审核、删除或注入 Gene。

`[可控风险]` `c_constraints` 字段设置爆炸半径（`max_files_changed`, `forbidden_paths`）；金丝雀检查防止损坏代码写入。

**2.7.3 固化流程（对应路径三：协作观察学习）**

`SolidifyPipeline` 持有：`validator`（变更验证器）、`blast_radius_assessor`（爆炸半径评估）、`capsule_store: Arc<CapsuleStore>`（与 GepState **共享** 同一 Arc，避免双写）、`event_log: Arc<EvolutionEventLog>`（与 GepState 共享同一 Arc）、`post_solidify_hooks`（层间解耦钩子，Git commit 等由 legion-core 注入）。

**代码变更固化流程**（`solidify_code_change`，路径三）：
1. 执行 Gene 的 `v_validation` 钩子——任何失败立即拒绝
2. 金丝雀检查
3. 爆炸半径评估——超出 `c_constraints` 限制则拒绝
4. 创建 Capsule（执行证明，写入共享 capsule_store）
5. 追加 EvolutionEvent（不可变审计链）
6. 触发钩子（Git commit 等 Layer 1 操作由此触发）

**Gene 资产固化流程**（`solidify_gene`，GEP 周期专用）：
1. 将草稿提升为正式 Gene（验证已通过）
2. 写入能力记忆 `capsule_store.upsert_gene()`
3. 创建对应 Capsule（`l_lineage = None` = 新建根节点）
4. 追加 EvolutionEvent（`t_type = Distill`）
5. 返回 `GepStateDelta::Evolved { new_gene }` 供 GepState 更新内存集合

> ⚠️ 架构约束：`legion-learning` 是 Layer 2，不直接依赖 Git（Layer 1 工具）。Git 提交通过 `post_solidify_hooks` 由 `legion-core`（Layer 3）注入，实现层间解耦。

**2.7.4 知识库驱动学习（对应路径四）**

参见 §三 LLM Wiki 设计，Agent 通过 Wiki API 实时检索相关知识注入认知内核。

**2.7.5 GEP 六阶段形式化周期**

论文定义 GEP 为从状态三元组 **(𝒢, 𝒞, ℰ)** 到 **(𝒢′, 𝒞′, ℰ′)** 的变换（𝒢 = Gene 集合，𝒞 = Capsule 集合，ℰ = EvolutionEvent 审计链），包含六个阶段：

```
(𝒢, 𝒞, ℰ)
    │
    ├─ [1] scan     ← 扫描近期执行历史 ℰ，提取候选经验片段
    │
    ├─ [2] signal   ← 三层信号提取（§2.7.1），输出触发信号 σ
    │
    ├─ [3] intent   ← LLM 推断变更意图 ι（为什么要进化？）
    │
    ├─ [4] mutate   ← 蒸馏算子 ψ 生成 Gene 候选 g′（从 σ+ι 提炼）
    │
    ├─ [5] validate ← 执行验证钩子 v + 金丝雀检查，输出 ν
    │
    └─ [6] solidify ← ν=pass → 写入 𝒢，创建 Capsule κ，追加 Event e
    │
(𝒢′, 𝒞′, ℰ′)
```

| 阶段 | 输入 | 核心操作 | 输出 |
|------|------|---------|------|
| **scan** | 最近执行历史 ℰ | 识别近期失败/成功模式 | 候选经验片段 |
| **signal** | 候选片段 | 三层信号提取（§2.7.1） | 触发信号 σ |
| **intent** | 信号 σ | LLM 推断变更意图 | 意图描述 ι |
| **mutate** | 意图 ι + 现有 𝒢 | 蒸馏算子 ψ 生成草稿 | Gene 候选 g′ |
| **validate** | Gene 草稿 g′ | 执行 v 钩子 + 金丝雀 | 验证结果 ν |
| **solidify** | ν=pass + g′ | 原子写入三元组 | (𝒢′, 𝒞′, ℰ′) |

**GepState** 是 GEP 三元组状态载体，持有：
- `genes: Vec<Gene>`：当前有效 Gene 库（按 agent_id 分区，常驻内存，数量有限）
- `capsule_store: Arc<CapsuleStore>`：Capsule 懒加载句柄（90天窗口可能含数万条，不预加载）
- `event_log: Arc<EvolutionEventLog>`：与 SolidifyPipeline 共享同一实例

`GepCycle.run_cycle()` 触发条件是**检测到失败模式（修复驱动）**，而非定时触发。触发路径：`run_task()` 末尾的 `post_task_learning`（§2.4 Step 7）向 `event_bus` 发布 `LearningEvent`；后台任务 `gep_failure_scan`（§2.13，每 15 分钟）消费队列、聚合信号，信号计数达到阈值时调用 `run_cycle()`。Phase 1（scan）阶段必须同时提取执行轨迹（`candidate_traces`），因为 apply() 时已无法重新获取。

**2.7.6 蒸馏算子 ψ（Distillation Operator）**

论文定义两种 Gene 来源，对应两种 ψ 形式：

- **`ψ(s)`**：从单次成功执行轨迹 s 中提炼 Gene（快速，单样本）
- **`ψ(ℋ)`**：从历史执行集合 ℋ 中归纳跨任务模式（慢，多样本）

`DistillationOperator` trait 提供三个方法：`from_trajectory`（ψ(s) 路径）、`from_history`（ψ(ℋ) 路径）、`apply`（GEP 周期统一入口，依据 `candidate_traces` 非空选择路径）。

`LlmDistillationOperator` 的 token 预算为 ≤ 230，Prompt 强制要求 α（AVOID 线索）字段非空。

**token 预算约束**（论文实验结论）：

| 表示方式 | 平均长度 | 相对 baseline 效果 |
|---------|---------|-----------------|
| Strategy Gene（本设计）| ~230 tokens | **+3.0pp** |
| Procedural Skill | ~2,500 tokens | **-1.1pp**（性能下降！）|
| 无经验 baseline | — | 0 |

Gene 比 Skill 短 10 倍，效果好 4.1pp。根本原因：**紧凑表示减少上下文噪声**，Skill 的详细步骤反而使模型过拟合特定场景。

**2.7.7 实验验证与 Legion 设计原则**

**论文核心实验数据**（4,590 次试验，45 个科学场景）：

| 指标 | 数值 | 意义 |
|------|------|------|
| Evolver vs base（CritPt 基准） | 27.14% vs 17.7% | +9.44pp 整体提升 |
| 单 Gene vs 多 Gene 组合 | 单 Gene 更优 | 多 Gene 模糊控制焦点 |
| AVOID cues α 单字段贡献 | **+4.6pp** | 失败模式泛化性最强 |
| Gene vs Skill 对比 | +3.0pp vs -1.1pp | 紧凑表示优于详细步骤 |
| Gene 结构鲁棒性 | 步骤顺序敏感 52.8% | 语义比结构更脆弱 |

**Legion 五项设计原则**（来自论文洞见，优先级从高到低）：

1. **表示即算法（Representation > Algorithm）**：学习引擎的核心价值在于将执行经验压缩为正确的知识表示格式（Gene 六元组），而非设计复杂的进化算法。一个好的 Gene 比一段复杂的代码更有价值。

2. **失败优先（Failure-First Encoding）**：α 字段（AVOID 线索）是 Gene 中最重要的字段（+4.6pp），强制非空。设计 Gene 模板时，未填写 α 的草稿应被 validator 拒绝。

3. **单焦点原则（Single Focus）**：每个 Gene 只解决一类问题。每次任务注入的 Gene 数量限制 ≤ 3 个，多 Gene 组合会模糊控制焦点。

4. **Token 预算纪律（Token Budget）**：Gene 总 token 数控制在 ≤ 230。超过 500 tokens 的 Gene 触发自动压缩。禁止将 Procedural Skill 作为 Gene 的主体内容。

5. **修复驱动进化（Repair-Driven Evolution）**：GEP 周期的主要触发条件是检测到失败模式，而非定时触发或探索性重组。进化是收敛行为，不是随机搜索。

`[可观测]` GEP 周期每次运行记录完整的六阶段状态快照：触发信号 σ → 意图 ι → Gene 草稿 → 验证结果 ν → 最终状态。管理者可追溯每个 Gene 的完整来源轨迹（通过 EvolutionEvent 的 ρ/σ/ι/Δ 字段）。

`[可治理]` 管理者可以：①在 validate 阶段前人工审核 Gene 草稿；②直接编辑 Gene 的 α 字段注入已知失败模式，修正 Agent 的错误倾向；③手动限制或解除每任务 Gene 注入数量上限。

`[可控风险]` 三重保障：①token 预算约束（≤ 230 tokens）防止 Gene 膨胀；②validation 钩子（v 字段）在固化前验证，失败直接拒绝；③单焦点原则限制每个 Gene 的影响范围，防止一个错误 Gene 破坏所有任务。

### 2.8 三层记忆系统

映射 Legion.md §3.2.3 到具体实现：

| 层级 | 组件 | 存储方式 | 生命周期 |
|------|------|---------|---------|
| 层1：工作记忆 | `WorkingMemory` | 内存中维护 | 单次任务 |
| 层2：情景记忆 | `EpisodicMemoryStore` | SQLite 持久化 | 跨任务 |
| 层3：能力记忆 | `CapabilityMemoryStore` | Gene/Capsule 资产 | 长期沉淀 |

来自 Hermes `MemoryProvider` ABC 的可插拔设计：`MemoryProvider` trait 提供三个方法——`system_prompt_block()`（注入系统提示）、`prefetch(task)`（任务前预取相关记忆）、`sync_after_turn(messages)`（回合后同步消息历史）。

内置实现 `LegionMemoryProvider` 集成情景记忆存储、Wiki 客户端和嵌入模型。

### 2.9 进化评估器

量化追踪 Legion.md §3.2.5 的 5 维度。`EvolutionEvaluator` 在指定时间窗口内生成 `AgentReport`，包含五个维度：任务质量（`task_quality`）、决策效率（`decision_efficiency`）、学习进度（`learning_progress`）、协作评分（`collaboration_score`）、风险行为（`risk_behavior`）。

`detect_degradation()` 方法：采用**双条件检测**——① 时间窗口：仅分析最近 **14 天**内的数据；② 样本量门槛：窗口内任务数 ≥ **5 次**时才触发趋势计算（低频 Agent 样本不足时跳过，报告中标注"样本不足，跳过退化检测"）；满足双条件时若质量指标相对下降 ≥ 15% 则触发 `DegradationAlert`（含恶化维度和建议干预措施）。`DegradationAlert` 的传播路径：由 `evolution_report` 后台任务（§2.13，每 24h）调用 `detect_degradation()` 后，通过 `event_bus.broadcast("agent.degradation", { agent_id, alert })` 广播；`legion-gateway` 订阅该事件并通过 SSE 推送至管理面板；同时写入 `security_events` 表（级别 `warning`，类型 `agent.degradation`），确保即使 SSE 连接断开也有持久化记录可查。

`[可观测]` 进化评估器定期生成"Agent 成长报告"，含能力雷达图变化趋势，管理者可随时查看。

`[可治理]` 检测到能力退化自动通知管理者；管理者可以手动触发能力回退、删除错误记忆、注入正确经验。

### 2.10 任务状态机与 Aegis 质量门控 —— 来自 Mission Control

#### 2.10.1 七状态任务状态机

Mission Control 的任务状态机设计是**目前分析的五个项目中最完整的产出质量管控方案**，Legion 直接采纳其核心约束：

| 状态 | 语义 | 进入条件 |
|------|------|---------|
| `Inbox` | 新建待路由 | 任务创建时 |
| `Assigned` | 已路由到 Agent，等待执行 | `autoRoute()` 调度器触发 |
| `InProgress` | Agent 正在执行 | `atomic_claim()` 原子锁定 |
| `QualityReview` | Agent 完成，等待 Aegis 审核 | Agent 提交产出 |
| `Done` | 已通过质量审核，终态（不可逆） | Aegis APPROVED |
| `Failed` | 失败（超出重试次数 / 评审 3 次被拒） | 超出上限 |
| `Suspended` | HardLoop 强制挂起，等待人工审批后恢复 | HardLoop ratio > 10（见 §6.2） |

**状态转换图**：

```
inbox
  ↓ autoRoute()（调度器 task_dispatch，60s）
assigned
  ↓ atomic_claim()（原子 RETURNING UPDATE，无竞态）
in_progress
  ↓ Agent 提交审核
quality_review ──→ Aegis REJECTED（最多 max_cycles 次）──→ in_progress
  ↓ Aegis APPROVED
done（不可逆终态）
quality_review（超过 max_rejection_cycles）──→ failed（升级告警）

in_progress（HardLoop ratio > 10）──→ suspended（等待人工审批）
  ↓ 审批 Approved → in_progress（继续执行）
  ↓ 审批 Denied  → failed

in_progress（卡死超时）──→ stale_requeue ──→ assigned（最多 5 次）──→ failed
```

硬约束：
- 只有通过 Aegis 批准才能进入 `Done`，API 层拒绝任何绕过路径，返回 403
- `Suspended` 状态只能由 `TaskScheduler.force_interrupt()` 写入；恢复（→ `InProgress`）必须通过 §6.2 审批门控

#### 2.10.2 Aegis 质量评审器

**定位澄清**：Aegis 是**固定的 LLM 评审器**，不是可进化的 Agent。它直接调用 MaaS 层，使用固定的 Prompt 模板，不走 `AgentRuntime` 执行路径，不产生 Gene/Capsule，也不需要被 Aegis 自身审核（无自举问题）。

这个定位选择的理由：
- **稳定性优先**：质量门控自身如果可被进化修改，会引入"守门人被篡改"的安全风险
- **无自举依赖**：Aegis 只依赖 MaaS 层（Phase 1 完成即可运行），不依赖 AgentRuntime
- **可理解性**：固定 Prompt 模板让 Aegis 的判断逻辑完全透明，管理员可以直接审阅和修改

`AegisReviewer` 持有：`model_client`（MaaS 直连）、`max_rejection_cycles`（默认 3，超出自动转 failed）、`prompt_template`（可配置但不可自动进化）。

每次审核产生 `QualityReview`：任务 ID、评审者（"aegis" 或自定义）、裁决（`Approved` 或 `Rejected { reason, suggestions }`）、本轮编号、时间戳。

Aegis 默认使用 Advanced Tier 模型，解析固定格式 "VERDICT: APPROVED/REJECTED"。

**与工作流 DSL 集成**：每个 `Sequence` 步骤可配置 `quality_gate`（enabled、reviewer、max_cycles、criteria 验收标准列表）。

`[可观测]` 每轮审核结果（批准/驳回 + 具体理由）写入 `quality_reviews` 表并广播 SSE 事件，UI 实时反馈。

`[可治理]` Aegis 的审查标准 criteria 可在 YAML 中自定义；`max_cycles` 上限防止无限驳回循环。

`[可控风险]` 超过 `max_rejection_cycles` 自动转 failed 并升级告警——不让 Agent 无限重试消耗预算。

### 2.11 Agent 信任评分系统 —— 来自 Mission Control

#### 2.11.1 与进化评估器的正交关系

进化评估器（§2.9）衡量 **能力成长**（任务质量、学习进度），信任评分衡量 **安全行为**（是否试图越权、泄露秘密、触发注入）。两者正交，共同构成完整的 Agent 评估体系：

```
能力维度（§2.9 进化评估器）  ×  安全维度（§2.11 信任评分）
      ↓                              ↓
  学习引擎驱动                 派遣决策 + 安全态势
```

#### 2.11.2 事件驱动实时重算

信任分 0.0 ~ 1.0，**新 Agent 初始值为 0.7**（试用期基准，而非 1.0）——信任需要用实际表现赢得，新 Agent 需积累约 75 次 `TaskSuccess`（75 × 0.004 = +0.30）方可达到满分；每次 `logSecurityEvent()` 调用时实时重算（不使用批量定时更新）。

**事件权重表**（恢复极度困难——50次成功任务才能抵消1次密钥泄露）：

| 事件类型 | 权重 | 恢复所需成功次数 |
|---------|------|--------------|
| `SecretExposure`（密钥泄露） | -0.20 | 50 次 |
| `InjectionAttempt`（注入攻击） | -0.15 | 38 次 |
| `UnauthorizedAccess`（越权访问，Legion 扩展） | -0.10 | 25 次 |
| `AuthFailure`（认证失败） | -0.05 | 13 次 |
| `RateLimitHit`（速率限制） | -0.03 | 8 次 |
| `TaskSuccess`（任务成功） | +0.004 | — |
| `TaskFailure`（任务失败） | -0.01 | 3次成功抵消 |

`AgentTrustProfile` 持有：当前分数（`score`）、各事件类型计数（`event_counters`）、最后异常时间（`last_anomaly_at`）、冷却期截止时间（`cooldown_until`，Legion 扩展）。

**冷却期机制（Legion 扩展 MC 没有的）**：权重绝对值 > 0.10 的严重事件触发 24 小时冷却期，冷却期内 `effective_score()` 上限为 0.6。

**冷却期懒惰求值**：`effective_score()` 在每次查询时实时检查 `cooldown_until > now()`，无需后台定时任务清理，消除定时清理方案最长 1h 的状态延迟。

#### 2.11.3 信任分驱动派遣决策

`IntelligentRouter.apply_trust_constraints()` 依据有效信任分决定处置：

| 有效分数范围 | 处置方式 |
|------------|---------|
| ≥ 0.5 | 正常派遣 |
| 0.3 ~ 0.5 | 仅允许低/中优先级任务 + Light Tier + 全工具审批 |
| < 0.3 | 拒绝派遣，要求人工介入 |

`[可观测]` 信任分变更实时广播 SSE 事件，安全审计面板实时更新每个 Agent 的风险状态。

`[可治理]` 管理员可查看安全事件历史，手动重置错误记录（如误报的注入检测）；冷却期可配置。

`[可控风险]` 低信任分 Agent 自动降为严格沙盒，高风险任务绕行，整体安全态势通过信任分均值调制。

### 2.12 四层 Eval 框架 —— 来自 Mission Control

对现有 §2.9 进化评估器（能力维度）的**行为健康补充**，形成两套正交评估。

| 层级 | 评估维度 | 数据来源 | 时间窗口 | 通过阈值 |
|------|---------|---------|---------|---------|
| **Layer 1 Output Eval** | 任务完成率 + 正确率 | `tasks` 表 | 7 天 | 完成率 ≥ 70%，正确率 ≥ 60% |
| **Layer 2 Trace Eval** | 工具调用收敛比 | `mcp_call_log` | 24 小时 | ratio ≤ 5.0 |
| **Layer 3 Component Eval** | 工具可靠性 | `mcp_call_log` | 14 天 | 工具成功率 ≥ 80% |
| **Layer 4 Drift Detection** | 行为漂移检测 | 多表 | 当前周 vs 4周基线 | 相对变化 ≤ 10% |

**Layer 1 正确率计算**：`correctness = successRate × 0.6 + normalizedRating × 0.4`（有用户反馈时）；无反馈时 `correctness = successRate`。`normalizedRating = (feedback_rating - 1) / 4`（1~5分归一化为0~1）。

**Layer 2 收敛比**：`ratio = totalToolCalls / uniqueTools`；以 5.0（SoftLoop 阈值）归一化得分。

**Layer 4 漂移指标**：`avg_tokens_per_session`、`tool_success_rate`、`task_completion_rate` 三项，任意相对变化 `|current - baseline| / baseline > 10%` 即判定漂移。

**`LoopingStatus` 三档阈值**（Layer 2 专用）：

| 状态 | ratio 范围 | 行动 |
|------|-----------|------|
| `Normal` | ratio ≤ 5.0 | 无操作 |
| `SoftLoop` | 5.0 < ratio ≤ 10.0 | 触发 §2.6 循环检查点 |
| `HardLoop` | ratio > 10.0 | 强制中断 + §6.2 AnomalyEscalation 审批门控 |

**收敛比（Layer 2）与循环检查点（§2.6）的协同**：
- `SoftLoop`（ratio 5~10）：触发 §2.6 的循环检查点——生成本轮执行摘要，压缩上下文，重置循环计数；同时写入 `tasks.soft_loop_reset_at`，后续 eval_trace() 仅统计该时间戳后的调用记录
- `HardLoop`（ratio > 10）：强制中断，触发 §6.2 的 AnomalyEscalation 审批门控；`loop_reset_at` 机制参见 §6.2.1

> ⚠️ **`eval_trace()` 实时检测 vs 监控评估的时间窗口区分**：§2.4 内循环调用的 `eval_trace()` 使用**当前任务窗口**（从 `soft_loop_reset_at` 或 `loop_reset_at` 起算），防止已压缩的历史调用数据持续误判；§2.12 四层 Eval 框架的**监控统计**仍使用各层定义的历史窗口（Layer 2：24h，Layer 3：14d 等），两者数据来源相同但时间过滤不同。

`[可观测]` 四层指标写入 `eval_runs` 表，提供 8 周历史趋势 API，进化评估器可读取漂移数据作为学习触发信号。

### 2.13 后台调度器架构 —— 来自 Mission Control

Legion 引擎的后台维护任务统一由 `BackgroundScheduler` 管理，采用 Mission Control 经验证的三原则：**防重入 + 分散初始延迟 + 动态启用**。

`BackgroundScheduler` 持有任务列表、全局 tick 间隔（60s）和动态配置引用（`Arc<DynamicConfig>`，运行时读取，无需重启）。

每个 `ScheduledTask` 包含：任务 ID、执行间隔、初始延迟（分散延迟防启动争用）、`enabled_config_key`（对应管理 UI 开关）、`running: AtomicBool`（防重入）、`next_run: AtomicI64`。

**Legion 调度任务清单**：

| 任务 ID | 间隔 | 初始延迟 | 职责 |
|---------|------|---------|------|
| `agent_heartbeat` | 5min | 5s | 离线检测 + 僵尸任务触发 |
| `task_dispatch` | 60s | 10s | inbox→assigned→派遣 |
| `quality_review` | 60s | 30s | Aegis 批量审核 |
| `stale_task_requeue` | 60s | 25s | 卡死任务回退 |
| `skill_sync` | 60s | 15s | 技能磁盘↔DB 同步 |
| `memory_fts_update` | 5min | 20s | FTS5 增量索引更新 |
| `eval_drift_detection` | 1h | 60s | Layer 4 漂移定期扫描（⚠️ 仅 Layer 4，Layer 2 收敛比检测为内循环实时，见 §2.4） |
| `gep_failure_scan` | 15min | 90s | 消费 `LearningEvent` 队列（持久化于 `learning_events` 表，含 `consumed_at` 字段；TTL 7 天——超期未消费的事件自动归档，防止禁用期间无限积压），聚合失败信号；信号计数达到阈值时触发 `GepCycle.run_cycle()`（修复驱动，§2.7.5）；每次最多触发一个 GEP 周期（AtomicBool 防重入）；禁用后恢复时批量消费，单批上限 100 条 |
| `evolution_report` | 24h | 120s | 日 Agent 成长报告 |
| `webhook_retry` | 60s | 30s | 失败 Webhook 退避重试 |
| `knowledge_gap_detect` | 24h | 次日 3 点 | Wiki 知识缺口扫描 |
| `auto_backup` | 24h | 次日 4 点 | DB 备份（默认禁用）|

> ⚠️ **`trust_score_cooldown` 任务已移除**：信任分冷却期采用懒惰求值（§2.11.2 `effective_score()` 在每次查询时实时检查 `cooldown_until > now()`），不需要后台任务清理。定时任务方案存在最长 1h 的状态延迟，且与懒惰求值并存会产生状态矛盾。

分散延迟设计：任务首次运行间隔至少 5s 错开，防止服务启动时多个任务同时访问 SQLite。

`[可治理]` `enabled_config_key` 对应管理 UI 中可实时切换的开关，无需重启服务。

`[可控风险]` `running: AtomicBool` 确保慢任务（如漂移检测、知识扫描）不会叠加执行导致内存爆炸。

<!-- @end-section -->

---

<!-- @section: wiki-engine-design -->
## 三、LLM Wiki 知识库引擎设计

### 3.1 设计目标回顾（来自 Legion.md §3.3）

LLM Wiki 是**人机共建、人机共读**的知识系统。关键需求：
- 5 步知识加工管道
- 混合检索（向量 + 全文 + 图谱）
- 严格的知识治理（版本/溯源/审核/冲突裁决）

### 3.2 知识存储引擎 —— 来自 Hermes FTS5 + 内容寻址 + Mission Control BM25/WikiLink

**核心数据表**（SQLite，WAL 模式支持并发读写）：

- **`knowledge_chunks`**：核心知识表，字段含 `id`（SHA-256 内容寻址）、`doc_id`、**`company_id`（多租户隔离，所有查询加 `WHERE company_id = ?` 过滤，联合索引 `(company_id, status)`）**、`content`、`embedding`（向量嵌入）、`created_by`（溯源：人类 ID 或 Agent ID）、`source_type`（`human`/`agent_task`/`agent_learn`）、`status`（`pending`/`approved`/`archived`）、`quality_score`、`version`
- **`audit_log`**：不可变审计日志（来自 Evolver events.jsonl 思想），仅 INSERT 权限，无 UPDATE/DELETE；同样包含 `company_id` 字段

> ⚠️ **`evolution_assets`（Gene/Capsule/EvolutionEvent）不属于 Wiki 引擎数据库**：该表由 `legion-learning`（Layer 2）的 `CapsuleStore` 和 `EvolutionEventLog` 管理，存储于 Agent 专属的 `evolution.sqlite` 文件中（参见 §2.7.2）；Wiki DB（`legion-wiki`，Layer 4）不持有也不访问进化资产，避免 Layer 2 反向依赖 Layer 4 的层违反。

### 3.2b FTS5 精细化检索 —— 来自 Mission Control

**FTS5 虚拟表**：`knowledge_fts` 使用 `porter unicode61` tokenizer（英文词干 + Unicode），索引 `path`、`title`、`content` 三列。

**BM25 权重设计**：标题权重 **5.0**，路径/内容权重 **1.0**（标题重要性 5 倍）。Snippet 提取使用 `<mark>` 高亮标签，上下文 40 词。

**`KnowledgeFtsEngine.search()` 查询语法支持**：

| 语法 | 示例 | 说明 |
|------|------|------|
| 简单词 | `legion` | 自动转前缀匹配 `legion*` |
| 多词 AND | `legion agent` | 隐式 AND（`legion* AND agent*`） |
| 精确短语 | `"model routing"` | 引号包围 |
| NEAR | `NEAR(agent task, 5)` | 5 词内共现 |
| OR / NOT | `agent OR skill NOT test` | 布尔运算 |
| 前缀 | `orche*` | 前缀通配符 |

`sanitize_query()` 安全化规则：已含 FTS5 操作符（AND/OR/NOT/NEAR/引号）→ 直通；普通单词 → 转前缀匹配；多词 → 隐式 AND。

**WikiLink 图谱**：知识库文件间通过 `[[目标文件]]` 语法相互链接，构成连接图谱。图谱持久化于 SQLite `wiki_links` 邻接表（字段：`source_path`、`target_path`、`updated_at`），支持跨重启增量维护，避免每次启动全量扫描。`target_path` 建立索引支持反向查找。

`WikiLinkGraph` 提供的操作：
- `extract_links(content)`：正则提取 `[[目标]]` 或 `[[目标|显示文本]]`
- `upsert_links(source, targets)`：保存文件时替换全部旧出链（先 DELETE 该 source，再 INSERT）
- `detect_gaps()`：检测知识缺口（被引用的 target 在文件系统中不存在），按引用次数排序
- `find_hubs()`：识别中心节点（出度 ≥ 均值 × 2）
- `bfs_neighbors(roots, depth)`：SQL 递归 CTE 实现 BFS，限制递归深度，用于混合检索图遍历

`[可观测]` WikiLink 图谱支持知识缺口扫描（gap-detect）和中心节点识别（consolidate），定期由调度器运行并写入健康报告。

### 3.3 混合检索引擎 —— 来自 Hermes + Legion.md

Legion.md §3.3.4 要求三路融合检索。**架构约束**：全部基于 SQLite，不引入 Neo4j 等外部图数据库——WikiLink 图（§3.2b）已用 SQLite 邻接表实现，维护零额外运维成本。

`HybridRetriever` 组合三个检索源：
- `VectorStore`：sqlite-vss（内嵌，无额外服务），向量相似度检索（TopK 20）
- `KnowledgeFtsEngine`：SQLite FTS5（§3.2b），全文关键词检索（TopK 20）
- `WikiLinkGraph`：SQLite 邻接表（§3.2b），从实体节点出发 BFS 2 跳图遍历

三路并行检索（`tokio::join!`，全部走 SQLite，无跨进程调用），结果通过 **RRF（Reciprocal Rank Fusion）** 融合排序。

### 3.4 渐进式知识加载 —— 来自 Hermes

Hermes 的技能渐进式加载直接应用于 Wiki 知识检索。`KnowledgeLoader` 分两阶段：
- **阶段 1**：只加载元数据索引（触发条件 + 简短描述，无完整内容），Agent 初始化时执行，低成本
- **阶段 2**：Agent 需要时按需加载完整内容（带元数据缓存，避免重复拉取）

### 3.5 知识治理管道

完整实现 Legion.md §3.3.5 的治理机制。`KnowledgeGovernance` 包含：`VersionManager`（版本管理）、`ConflictDetector`（冲突检测）、`ReviewWorkflow`（审核流程）、`QualityScorer`（质量评分）。

**知识状态流转（`KnowledgeStatus`）**：

| 状态 | 语义 |
|------|------|
| `PendingReview` | AI 生成，等待人工审核 |
| `UnderReview` | 审核中 |
| `Approved` | 已批准，进入正式库 |
| `Conflicting` | 与已有知识冲突，待裁决 |
| `Watchlist` | 冲突双版本均保留，等待后续实践验证哪个更准确（裁决结果为"待观察"时进入此态） |
| `Archived` | 已归档（低价值 / 过期） |
| `Rejected` | 已拒绝 |

所有 AI 产出知识默认进入 `PendingReview`，`submit_ai_knowledge()` 提交时自动运行冲突检测，发现矛盾则转为 `Conflicting` 并升级告警。

**`ConflictDetector` 检测机制**：采用两级检测，任一触发即判定冲突：
1. **语义相似冲突**：对新内容提取嵌入向量，在 `knowledge_chunks` 中向量检索 TopK=10，若存在余弦相似度 > 0.85 的已批准知识块，则进入语义比较——若两段内容在关键实体上存在直接矛盾（通过轻量 LLM 调用判断，使用 Light Tier），标记冲突
2. **显式引用冲突**：通过 WikiLink 图谱识别新内容与现有知识的直接引用关系，若新内容断言与已批准内容同一主题下的结论相反，标记冲突

**冲突裁决流程**（`Conflicting → Approved`）：冲突知识写入待裁决队列，由有 `operator` 及以上权限的人类或高权限 Agent 执行裁决：批准新内容并归档旧内容（`Conflicting → Approved`，旧内容 `→ Archived`），或拒绝新内容（`Conflicting → Rejected`），或转为"待观察"（`Conflicting → Watchlist`，双版本均保留，等待后续实践数据验证）。裁决决策写入 `audit_log`，保证可溯源。

`[可治理]` 所有 AI 产出的知识默认进入待审核队列，只有经过人类审核或高权限 Agent 交叉验证后才进入正式库。

<!-- @end-section -->

---

<!-- @section: adapter-design -->
## 四、Adapter 层设计

### 4.1 设计目标回顾（来自 Legion.md §4.4）

Adapter 层的关键原则是**控制平面与执行平面分离**：Legion 是控制平面，不直接运行 Agent；各 Adapter 是执行平面，通过标准接口与控制平面通信。

### 4.2 六类 Adapter 实现策略

| Adapter 类型 | 实现参考 | 关键设计点 |
|------------|---------|----------|
| **LLM API Adapter** | DeepSeek-TUI `ModelProvider` + Hermes `ProviderTransport` | 统一 SSE 流管道 + NormalizedResponse |
| **CLI Agent Adapter** | Claw Code 进程管理 + Hermes `Process Adapter` | 子进程生命周期 + 标准输出解析 |
| **MCP Adapter** | DeepSeek-TUI MCP crate + Hermes MCP 集成 | JSON-RPC stdio/HTTP + 工具代理 |
| **HTTP Webhook Adapter** | Hermes 平台适配器模式 | BasePlatformAdapter + 重试队列 |
| **Process Adapter** | Claw Code bash 执行 + 权限检查 | bash_validation + 沙盒策略 |
| **Composite Adapter** | 来自 Legion.md 设计 | 多 Adapter 组合 + 统一 RunResult |

### 4.3 标准 Adapter 接口

`ServerAdapter` trait 定义三个必须实现的方法：
- `execute(ctx: ExecutionContext) -> RunResult`：执行 Agent 心跳（核心）
- `diagnose() -> EnvironmentDiag`：环境健康检查
- `parse_usage(output: &RunOutput) -> CostData`：解析执行成本

`CliAgentAdapter` 实现：先通过 `permission_enforcer.check_before_spawn()` 权限检查（来自 Claw Code），再通过 `process_manager.spawn_sandboxed()` 在指定沙盒模式中启动进程（来自 DeepSeek-TUI `ExternalSandbox`），最后流式读取输出（借鉴 DeepSeek-TUI SSE 管道思想）。

### 4.4 平台适配器（多入口支持）

参考 Hermes 的 19 平台适配器架构，Legion 支持多种接入方式。`PlatformAdapter` trait 提供：`on_message(event)` 处理入站消息，`send_message(target, content)` 发送消息。

统一投递目标 `DeliveryTarget` 支持四种形式：
- `Origin`：回复到消息来源
- `Specific { platform, id }`：发往指定平台指定 ID
- `Thread { platform, id, thread }`：发往指定线程
- `Broadcast { scope }`：广播全体/组

### 4.5 技能安全扫描器 —— 来自 Mission Control

Legion 的技能系统（`legion-skills`，Layer 1）在接受外部技能时必须经过安全扫描，防止恶意技能被 Agent 加载执行。

#### 4.5.1 安装前静态扫描

`SkillSecurityScanner.scan()` 对技能内容执行 13 条规则检查，分三级：

| 规则 ID | 级别 | 检测目标 |
|---------|------|---------|
| `prompt-injection-system` | Critical | "ignore previous instructions" 类提示注入 |
| `prompt-injection-role` | Critical | 角色操控 / 安全绕过指令 |
| `shell-exec-dangerous` | Critical | `rm -rf` / `curl\|bash` / `eval()` 组合 |
| `data-exfiltration` | Critical | 将数据发送到外部的指令 |
| `path-traversal` | Critical | `../` 路径穿越 |
| `ssrf-internal-network` | Critical | 访问 localhost / 内网 IP |
| `ssrf-metadata-endpoint` | Critical | AWS/GCP/Azure 元数据端点（169.254.x.x）|
| `credential-harvesting` | Warning | 硬编码 API key / secret / password |
| `obfuscated-content` | Warning | base64 / 十六进制编码内容 |
| `hidden-instructions` | Warning | HTML 注释中的可疑指令 |
| `excessive-permissions` | Warning | sudo / chmod 777 等高危权限操作 |
| `network-fetch` | Info | 包含外部 HTTP URL 引用 |
| `suspicious-imports` | Info | Legion 扩展：导入已知恶意库 |

**安全级别语义**：Critical → 阻断安装；Warning → 允许安装，标记 `security_status = 'warning'`；Info → 仅提示。

#### 4.5.2 技能安装流程

`SkillInstaller.install_from_registry()` 五步流程：
1. 从注册表（ClawdHub / skills.sh）拉取内容
2. SHA-256 内容完整性校验（ClawdHub 支持哈希验证，防内容篡改）
3. `SkillSecurityScanner.scan()`：Critical → 写安全事件日志 + 拒绝；Warning → 允许但标记
4. 写入磁盘（`workspace/skills/{name}/SKILL.md`）
5. 更新数据库（`skills` 表，记录 `security_status`、`content_hash`、`registry_slug`）

**磁盘↔DB 双向同步策略**（disk wins）：

```
磁盘有、DB 无 → INSERT
磁盘 SHA-256 变化 → UPDATE（重新扫描安全）
DB 有、磁盘消失 → DELETE（仅限无 registry_slug 的本地技能；注册表安装的保留）
```

`[可治理]` Critical 安全问题阻断安装并写入安全事件，管理员可查看被拦截的技能和原因。

`[可控风险]` 安全扫描规则版本化（可更新规则库）；扫描在安装时和每次磁盘同步时运行，确保已安装技能的安全状态始终最新。

<!-- @end-section -->

---

<!-- @section: org-comm-design -->
## 五、组织架构与通讯机制设计

### 5.1 四级组织数据模型

直接落地 Legion.md §4.1.1 的组织架构。**四张核心表（SQLite）**：

- **`companies`**：公司级（多公司隔离），含全局预算（`global_budget_cny`）和治理规则（`governance_rules` JSON）
- **`departments`**：部门级，归属公司，含月度预算和数据权限
- **`teams`**：团队级（四级层次的第三级），归属部门，`lead_agent_id` 指向负责人 Agent（延迟外键约束）
- **`agents`**：Agent 级，含公司/部门/团队关联、`reports_to`（严格汇报链）、角色（`role`）、人设（`persona` JSON）、模型策略（`model_policy` JSON）、进化配置（`evolution` JSON）、预算配置（`budget` JSON）

原子任务锁表 **`task_locks`**（`task_id`、`agent_id`、`locked_at`、`expires_at`），防止重复执行。`expires_at = locked_at + max_task_lock_duration`，默认值为 **15 分钟**（= 3 × heartbeat_interval；`agent_heartbeat` 每 5min 发送一次，连续 3 次无心跳即判定为僵尸任务）；`stale_task_requeue`（§2.13，每 60s）通过 `WHERE status = 'in_progress' AND expires_at < now()` 查询僵尸任务并回退到 `Assigned`。

### 5.2 通讯协议实现

**Crate 归属**：`legion-core`（Layer 3）。`AgentCoordinator` 是**任务编排协调层**，只负责任务的生命周期管理（锁定、路由决策、审计链、状态流转）——不包含 LLM 调用循环本身，后者由 §2.4 的 `AgentRuntime` 负责。

两者职责不重叠：
- `AgentCoordinator` 使用 `IntelligentRouter` 完成路由决策（选哪个模型），把决策结果（`RoutingDecision`）传给 `AgentRuntime`
- `AgentRuntime` 持有 `maas_client` 仅用于实际调用（`stream()`），不做路由

`AgentCoordinator` 持有：`task_scheduler`、`task_lock`（原子锁）、`intelligent_router`（路由决策）、`agent_runtime: Arc<Mutex<AgentRuntime>>`（Mutex：`run_task` 需要 `&mut self` 维护 `cycle_count`）、`audit_log`（不可篡改审计链）。

**`execute_heartbeat()` 九步流程**：

1. **身份确认**：`verify_identity(agent)`
2. **获取待办任务**：`task_scheduler.fetch_pending(agent.id())`
3. **原子锁定任务**：`task_lock.try_acquire()`，已被其他实例锁定则跳过
4. **路由决策**：`intelligent_router.route()`（不调用 LLM，纯决策）
5. **信任约束检查**：`intelligent_router.apply_trust_constraints(agent)`（§2.11.3）——依据有效信任分决定是否允许执行：score < 0.3 时中止本次心跳、释放锁、返回 `TrustBlocked`，不进入执行阶段
6. **执行工作**：`agent_runtime.lock().await.run_task(task, routing)`（通过 Mutex 获取 `&mut`）
7. **按执行结果分支处理任务状态**（`run_task()` 返回值驱动）：
   - `Completed` → 更新任务状态 `in_progress → quality_review`，进入 Aegis 审核队列（§2.10.2）
   - `AnomalyDetected`（HardLoop）→ **绕过质量审核**，进入 §6.2.1 HardLoop 响应链：`TaskScheduler.force_interrupt()` 写 `Suspended`，`ApprovalService` 创建审批工单，`EventBus` 广播 `task.suspended`；同时由响应链统一发布 `LearningEvent(signal: HardLoopFailure)`，供 `gep_failure_scan` 采集失败模式
   - `BudgetExhausted` / `Interrupted` → 更新任务状态为 `Failed`，写入终止原因（`LearningEvent` 已在 §2.4 内循环 Step 1/2 中提前发布）

   **关键约束**：`AnomalyDetected` 分支是唯一不经过质量审核直接挂起的路径；`Suspended` 状态仅由 `TaskScheduler.force_interrupt()` 写入，API 层拒绝任何外部直接写入
8. **写入审计链**：`audit_log.append(AuditEntry::from_task_result(...))`（所有分支均写入）
9. **释放任务锁**：`lock.release()`

`[可观测]` 每次心跳的完整链路（身份→锁定→路由决策→执行→状态流转）写入不可篡改审计链。

`[可治理]` 通讯权限遵循组织架构，越级通讯或私自广播被架构层拦截。

`[可控风险]` 原子任务锁杜绝重复执行；通讯频率限制防止消息风暴。

<!-- @end-section -->

---

<!-- @section: workflow-dsl -->
## 六、工作流 DSL 执行引擎

### 6.1 8 种原语的执行引擎

Legion.md §4.3.2 定义的 8 种流程原语及其执行语义：

| 原语 | 关键字段 | 执行语义 |
|------|---------|---------|
| `Sequence` | `id`, `agent_role`, `action` | 单步骤顺序执行 |
| `Parallel` | `branches`, `timeout_secs`, `on_partial_failure` | 并行执行所有分支，含超时控制 |
| `Branch` | `condition`, `true_branch`, `false_branch` | 条件路由 |
| `Loop` | `body`, `exit_condition`, `max_iterations` | 受控循环（强制最大迭代数）；达到 `max_iterations` 仍未满足 `exit_condition` → 触发包裹该节点的 `ErrorHandler`；若无 `ErrorHandler` 则整个工作流转为 `Failed` |
| `Join` | `wait_for: Vec<String>`, `timeout_secs` | 等待指定步骤集合完成；超过 `timeout_secs`（默认 3600s）则超时分支视为 `Failed`，按包裹层 `ErrorHandler` 处理 |
| `ApprovalGate` | `approver: ApproverSpec`, `condition` | 暂停流程，等待人类决策 |
| `EventWait` | `event_type`, `timeout_secs` | 等待外部事件到达；超过 `timeout_secs` 未收到事件 → 视为 `Failed`，触发包裹层 `ErrorHandler`；若无 `ErrorHandler` 则工作流转为 `Failed` |
| `ErrorHandler` | `body`, `on_failure: FailureAction` | 包裹步骤，处理失败 |

**并行分支部分失败策略（`ParallelFailurePolicy`）**：

| 策略 | 语义 | 适用场景 |
|------|------|---------|
| `FailFast` | 任意分支失败立即取消其余（默认） | 强依赖场景 |
| `CollectAll` | 等待所有分支，收集成功结果 | 独立任务批量执行 |
| `Quorum { min_success }` | 至少 n 个分支成功即为整体成功 | 冗余执行场景 |

**`FailureAction` 类型**：`Retry { max, back_to }`（重试并回退到指定步骤）、`Escalate { gate_type: ApprovalGateType }`（升级到人类介入，调用 `ApprovalService.create_ticket(trigger: gate_type)`；默认 `gate_type = AnomalyEscalation`，`ErrorHandler` 中可按场景指定其他门控类型，如 `QualityGate`）、`Fallback { step }`（执行备选步骤）。

### 6.2 七大审批门控

完整实现 Legion.md §4.3.4 的治理节点：

| 触发类型 | 触发场景 |
|---------|---------|
| `Recruitment` | Agent 请求创建下属 |
| `BudgetOverrun` | 预计成本超阈值 |
| `StrategyReview` | 管理层方案产出 |
| `ModelUpgrade` | 请求使用更高等级模型 |
| `AnomalyEscalation` | 连续失败或异常行为（含 HardLoop） |
| `QualityGate` | 交付物不达标 |
| `WikiKnowledge` | Agent 写入知识库 |

#### 6.2.1 HardLoop → AnomalyEscalation 完整响应链

`HardLoop`（收敛比 > 10，见 §2.12）是最严重的运行时异常之一，表明 Agent 陷入工具调用死循环，若不干预将持续消耗预算直至任务超时。完整响应链如下：

**触发方**：`AgentRuntime.run_task()` 内循环（§2.4）—— 每次工具执行完成后立即检测，不依赖后台调度器；`eval_drift_detection` 后台任务（§2.13）仅负责 Layer 4 漂移定期扫描，不承担 HardLoop 实时检测职责。

```
AgentRuntime 每次 execute_tools() 后检测 ratio > 10（HardLoop）
  │  返回 TaskResult::AnomalyDetected → AgentCoordinator 接收并启动响应链
  │
  ▼ 1. AgentCoordinator 发送强制中断信号给 TaskScheduler
       TaskScheduler.force_interrupt(task_id)
  │
  ▼ 2. TaskScheduler 原子更新任务状态
       tasks.status = Suspended（原子 CAS，防竞态）
       tasks.suspended_at = now()
       tasks.suspend_reason = "HardLoop: ratio={ratio:.1}"
       tasks.suspend_snapshot = AgentContext 快照（用于恢复上下文）
  │
  ▼ 3. AgentCoordinator 确认执行已停止
       AgentRuntime 已通过返回 TaskResult::AnomalyDetected 退出循环（无需额外 cancel）
       若有并发子任务仍在运行：CancellationToken::cancel()，最多等待 tool_timeout_secs
  │
  ▼ 4. 创建审批工单
       ApprovalService.create_ticket(
           trigger: AnomalyEscalation,
           task_id, agent_id,
           evidence: { ratio, tool_call_log: 最近 50 次调用 },
           approver: 任务创建者 / 管理员
       )
  │
  ▼ 5. SSE 广播前端通知（管理员立即知晓）
       EventBus.broadcast("task.suspended", { task_id, reason })
```

**审批决策处理**：

- **批准（Approved）**：`ApprovalService` 调用 `TaskScheduler.resume_task(task_id, snapshot)` 统一完成以下操作（所有状态写入由 TaskScheduler 执行，与 `force_interrupt()` 的写入职责保持对称）：从 `suspend_snapshot` 重建上下文；可选注入人类指导（降低循环风险）；重置收敛比计数器（防立即再触发）；原子 CAS `Suspended → InProgress`；将任务重新入队
- **拒绝（Denied）**：任务终止为 `Failed`；通知 Agent 任务已终止；SSE 广播

**关键设计约束**：

| 约束 | 实现方式 |
|------|---------|
| Suspended 只由 TaskScheduler 写入 | `tasks.status = Suspended` 仅在 `force_interrupt()` 内执行，API 层拒绝直接写 |
| 审批必须有人类参与 | `ApprovalService` 不提供自动批准接口（不可编程调用 approve） |
| 恢复后防二次触发 | 在 `tasks` 表写入 `loop_reset_at = now()`；Layer 2 收敛比查询 `mcp_call_log` 时自动过滤 `loop_reset_at` 之前的记录（`mcp_call_log` 为 append-only 审计日志，不可删除，通过时间戳过滤实现逻辑重置）；宽限期 5 分钟内 ratio 恒为 0 |
| 上下文完整性 | `suspend_snapshot` 序列化整个 `AgentContext`（含消息历史压缩版本） |
| 审计可追溯 | `ImmutableAuditLog` 记录：触发时间、ratio 值、工具调用证据、审批人、决策 |

`[可控风险]` HardLoop 响应链是 Legion 中**唯一一个人类强制介入点**（其他门控可配置为半自动），确保失控 Agent 不会在无人知晓的情况下持续消耗资源。

<!-- @end-section -->

---

<!-- @section: three-principles-matrix -->
## 七、三原则落地矩阵

### 7.1 每个设计决策的三原则标注

| 设计组件 | 来源 | `[可观测]` | `[可治理]` | `[可控风险]` |
|---------|------|----------|----------|------------|
| MaaS 传输层 NormalizedResponse | Hermes | 路由溯源字段 model_used / provider_id | 传输层可替换 | 统一错误分类学 |
| SSE 流管道 | DeepSeek-TUI | 流事件类型安全，无 any | — | 8MB 背压控制 |
| 连接健康状态机 | DeepSeek-TUI | 状态变更实时可见 | 管理员可手动触发探针 | 三态防崩溃，自动恢复 |
| 错误分类 + 恢复 | Hermes | 分类结果记录日志 | 恢复动作可配置覆盖 | 熔断 + 自动降级 |
| 认知内核优先级编排 | Legion 原创 | 上下文组装过程可追溯 | 安全红线优先级硬编码 | P0 安全红线不可覆盖 |
| 循环检查点 | DeepSeek-TUI | 摘要内容可查 | 管理员可调整 cycle_length | 防上下文窗口耗尽 |
| 批准策略 × 沙盒 | DeepSeek-TUI | 每次工具调用记录批准决策 | RBAC 角色覆盖策略 | SandboxMode 物理隔离 |
| 工具看门狗 | Hermes | before/after 调用记录 | 可配置检测规则 | 精确重复失败检测 |
| 权限执行器 | Claw Code | 每次权限检查记录 | 权限边界可配置 | 越权自动拦截 |
| 三层信号提取 | Evolver | 信号来源可追溯 | 信号规则可调整 | 修复循环检测 |
| Gene/Capsule/Event | Evolver | SHA-256 内容寻址 | Gene 可审核/删除 | 爆炸半径 + 金丝雀 |
| 固化流程 | Evolver | EvolutionEvent 不可变链 | 验证命令可配置 | pre-commit 验证拒绝（防止错误代码写入）；`blast_radius_assessor` 限制变更文件范围 |
| AVOID 线索（α）+ Gene 六元组 | Evolver + 论文 | 每个 Gene 来源轨迹可追溯（σ/ι/Δ）| 管理员可注入/删除 AVOID 线索；审核 Gene 草稿 | 单焦点原则；token 预算 ≤ 230；α 字段强制非空 |
| 蒸馏算子 ψ + GEP 六阶段周期 | Evolver + 论文 | 六阶段状态快照完整记录（scan→solidify）| 管理员在 validate 前人工门控；动态调整 Gene 注入上限 | 修复驱动触发（非定时）；失败模式检测为主 |
| 上下文压缩 | Hermes | 压缩历史可查 | 压缩策略可配置 | 防上下文爆炸 |
| Crate 分层架构 | DeepSeek-TUI | 依赖关系清晰 | 层间接口标准化 | 循环依赖编译拒绝 |
| 不可变审计日志 | Evolver | 每个操作有审计记录 | 仅 INSERT 权限 | 无法回溯修改 |
| 原子任务锁 | Legion.md | 锁定状态可见 | 管理员可强制释放 | 防重复执行 |
| FTS5 + WAL SQLite | Hermes + MC | 全文检索支持；BM25 标题权重 5x | 版本控制 + 回滚 | WAL 并发安全 |
| 知识治理流程 | Legion.md | 知识状态流转透明 | AI 知识必须人工审核 | 冲突检测 + 权限分级 |
| WikiLink 图谱 | Mission Control | gap-detect/consolidate 可查 | 知识缺口定期扫描告警 | 孤立文件检测防知识腐烂 |
| Aegis 质量门控 | Mission Control | 每轮审核结果可查；SSE 广播 | done 终态强制需要 Aegis 批准 | 3 轮上限防无限循环消耗预算 |
| 七状态任务机 | Mission Control | 状态转换历史可追溯 | 原子 claim 防竞态；硬约束拦截绕过 | 僵尸任务 5 次恢复上限防无限；Suspended 仅 TaskScheduler 写入 |
| Trust Score | Mission Control | 每个 Agent 的安全行为历史可查 | 低信任分自动降级派遣 | 冷却期防刷分；严重事件阻断高风险任务 |
| 四层 Eval | Mission Control | 收敛比 / 漂移 / 完成率 8 周趋势 | 循环检测触发强制中断 | Layer 2 实时循环检测，不等任务结束 |
| 后台调度器 | Mission Control | 任务运行状态 / 上次结果可查 | 动态启用：管理 UI 实时切换 | 防重入 + 分散延迟防启动雪崩 |
| 技能安全扫描 | Mission Control | 扫描结果和被拦截技能可审计 | Critical 级阻断安装，管理员可查原因 | 13 条规则覆盖注入/SSRF/路径穿越 |

### 7.2 "避免的坑"总结

从五个项目的教训中提炼，Legion 应避免：

1. **AIAgent 上帝类（来自 Hermes）**：Hermes 的 `run_agent.py` 达 14,123 行。Legion 从设计之初就按 Crate 分层，`legion-core` 是最复杂的 crate，但职责边界清晰：只负责 Agent 主循环编排，不包含具体工具实现。

2. **TUI crate 膨胀（来自 DeepSeek-TUI）**：DeepSeek-TUI 的 `tui` crate 有 230+ 文件，导致编译时间极长。Legion 的前端层不共用一个 crate，按功能域拆分：`legion-app-web`、`legion-app-cli`、`legion-app-mobile`。

3. **parse_sse_chunk 参数膨胀（来自 DeepSeek-TUI）**：7 个 `&mut` 参数。Legion 的 `SseParser` 封装为有状态对象（见 §1.3）。

4. **JSONL 查询性能（来自 Evolver）**：Evolver 的 JSONL 文件随事件增长查询性能下降。Legion 使用 SQLite + FTS5，同时保留 JSONL 格式作为不可变审计日志导出格式。

5. **提示缓存破坏（来自 Hermes）**：对话中期修改系统提示/工具集会破坏 Anthropic 提示缓存，显著增加成本。Legion 的认知内核在 `system_prompt_locked` 后冻结系统提示。

6. **RelayInfo / Session 上帝对象（通用教训）**：按职责拆分上下文结构体：`RequestContext`、`ProviderContext`、`BillingContext`、`StreamContext`，而非一个不断膨胀的 `ExecutionContext`。

7. **核心逻辑混淆（来自 Evolver）**：Evolver 核心 2.9MB 混淆，阻碍社区贡献和安全审计。Legion 完全开源，核心引擎代码可读。

8. **任务外键用字符串（来自 Mission Control）**：MC 的 `tasks.assigned_to` 存储 Agent 名称字符串而非 ID，Agent 改名后所有历史任务出现悬空引用，无法使用 DB 外键约束。Legion 使用不可变 UUID 标识 Agent，名称只用于显示。

9. **两套定时系统职责混淆（来自 Mission Control）**：MC 的 Node.js 调度器（内部运维）和 OpenClaw Cron（业务定时）职责不清晰，UI 混合展示令用户困惑。Legion 明确区分：`BackgroundScheduler`（对用户不可见的系统运维）和 `WorkflowCron`（用户可配置的业务定时任务）。

10. **MCP 工具数量膨胀（来自 Mission Control）**：MC 的 35 个 MCP 工具中存在功能重叠（`mc_agent_costs` 与 `mc_costs_by_agent` 等）。Legion 遵循工具正交原则：每个工具做一件事，通过参数覆盖变体，目标控制在 20 个以内。

11. **任务不能绕过质量门（来自 Mission Control）**：任务可以被代码直接写入 done 是个设计漏洞。Legion 在 **API 层 + 状态机层** 双重硬约束：未通过 Aegis 审核的任务 PUT done 返回 403。

<!-- @end-section -->

---

<!-- @section: implementation-roadmap -->
## 八、实现优先级路线图

### Phase 1：MaaS 基础（最高优先级）

1. `legion-protocol` — 数据结构定义（无依赖，先行）
2. `legion-config` — 配置加载
3. `legion-model` — SQLite Schema（参考 DeepSeek-TUI 4 表结构 + MC 七状态任务机）
4. `legion-maas` — 传输层 + 路由 + 配额（参考 Hermes NormalizedResponse）
5. SSE 流管道（参考 DeepSeek-TUI，封装为 `SseParser` 状态对象）

### Phase 2：Agent 运行时 + 质量管控

6. `legion-execpolicy` — 批准策略 × 沙盒（直接参考 DeepSeek-TUI）
7. `legion-tools` — 工具注册（参考 Hermes 自注册模式）
8. `legion-core` — Agent 主循环（参考 Hermes + Claw Code，含工具看门狗）
9. `legion-memory` — 三层记忆（工作/情景/能力）
10. Aegis 质量门控（参考 MC，§2.10）—— **必须与主循环同期完成**
11. `BackgroundScheduler`（参考 MC，§2.13）—— 防重入 + 动态启用
12. 七状态任务机（参考 MC，§2.10.1）—— API 层 + 状态机层双重保护

### Phase 3：安全体系 + 技能 Hub

13. Trust Score 信任评分（参考 MC，§2.11）—— 事件驱动实时重算
14. 四层 Eval 框架（参考 MC，§2.12）—— 行为健康监控
15. 技能安全扫描器（参考 MC，§4.5）—— 13 条规则 + Critical/Warning 分级
16. `legion-skills` — 技能加载 + 磁盘↔DB 同步（disk wins）

### Phase 4：学习引擎与进化

17. `legion-learning` — 三层信号提取（参考 Evolver `signals.js`，§2.7.1）
18. Gene 六元组资产模型（m/u/π/α/c/v，arXiv:2604.15097）—— **α 字段强制非空，token 预算 ≤ 230**
19. Capsule 六元组 + EvolutionEvent 九元组（含 Ed25519 immutable_seal，§2.7.2）
20. 蒸馏算子 ψ（`ψ(s)` 单次轨迹 + `ψ(ℋ)` 历史归纳，§2.7.6）
21. GEP 六阶段形式化周期（scan→signal→intent→mutate→validate→solidify，§2.7.5）
22. 固化流程（参考 Evolver `solidify.js` + Ed25519 签名，§2.7.3）
23. `legion-evolution` — 进化评估器（5 维度），读取 Phase 3 四层 Eval 数据作为输入（§2.9）

### Phase 5：LLM Wiki 与全平台集成

24. `legion-wiki` — 知识管道 + FTS5 BM25 + WikiLink 图谱（参考 MC §3.2b）
25. 知识治理（版本/溯源/审核/冲突裁决）
26. 知识健康诊断 8 维度 + gap-detect（参考 MC）
27. `legion-gateway` — REST API + WebSocket + SSE 事件流
28. 前端 Web 应用（组织架构可视化 + 工作流画布 + 安全审计面板）

<!-- @end-section -->

---

<!-- @section: related -->
## 相关文档

- [[agent-engine-design|agent-engine-design.md（含完整伪代码版）]]
- [[Legion|Legion V3.0 项目方案]]
- [[../analysis/claude/01-overview|Claw Code 项目架构总览]]
- [[../analysis/deepseek-tui/01-overview|DeepSeek-TUI 项目架构总览]]
- [[../analysis/deepseek-tui/07-insights|DeepSeek-TUI 设计洞察与 Legion 参考]]
- [[../analysis/evolver/01-overview|Evolver 项目架构总览]]
- [[../analysis/evolver/06-evolver-insights|Evolver 洞察与 Legion 参考]]
- [[../analysis/hermes/01-overview|Hermes Agent 项目架构总览]]
- [[../analysis/hermes/06-hermes-insights|Hermes 洞察与 Legion 参考]]
- [[../analysis/mission-control/index|Mission Control 分析索引]]
- [[../analysis/mission-control/03-orchestration-scheduler|Mission Control 编排引擎与调度器]]
- [[../analysis/mission-control/05-security-eval-framework|Mission Control 安全框架与 Agent 评估]]
- [[../analysis/mission-control/07-insights|Mission Control 设计洞察与 Legion 参考]]

<!-- @end-section -->
