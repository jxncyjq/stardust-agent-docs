---
id: "deepthinking-maas-001"
title: "MaaS 模型调度层深度设计"
aliases: ["MaaS deep design", "模型调度层设计", "MaaS layer design"]
type: "deepthinking"
category: "design/deepthinking"
tags: ["maas", "model-routing", "billing", "gateway", "deep-design"]
version: "1.0.0"
created: "2026-05-04"
updated: "2026-05-04"
author: "jxncyjq"
status: "review"
parent: null
related_docs:
  - id: "analysis-maas-overview-001"
    relation: "reference"
    path: "../analysis/maas/01-overview.md"
  - id: "analysis-maas-channel-003"
    relation: "reference"
    path: "../analysis/maas/03-channel-adapter-system.md"
  - id: "analysis-maas-billing-004"
    relation: "reference"
    path: "../analysis/maas/04-billing-quota-system.md"
  - id: "analysis-hermes-runtime-002"
    relation: "reference"
    path: "../analysis/hermes/02-agent-runtime.md"
---

<!-- @section: overview -->
# MaaS 模型调度层深度设计

## 核心命题

> MaaS 层应该是"透明网关"还是"智能调度平台"？

答案：**智能调度平台**。new-api 证明了"透明网关"模式的天花板——它擅长协议适配和费用控制，但完全不具备"为不同任务智能选择最优模型"的能力。Legion 的 MaaS 层必须在网关的基础上叠加调度智能。

## 一、从 new-api 学到的：网关层的正确做法

### 1.1 适配器模式 — 解耦模型接入

new-api 的 40+ 渠道适配器证明了适配器模式是模型接入的最佳实践：

```
new-api 的 Adaptor 接口 (Go):
  type Adaptor interface {
      Init(info *Channel)                           // 初始化配置
      DoRequest(c *gin.Context, info, request)      // 执行请求
      GetChannelName() string                       // 渠道标识
      GetModelList() []string                       // 支持的模型
  }

Legion 应该继承的设计:
  type ModelProvider interface {
      // 核心能力
      SupportedModels() []ModelCapability
      ChatCompletion(ctx, request) → NormalizedResponse
      StreamChatCompletion(ctx, request) → StreamReader

      // 运维能力
      HealthCheck() HealthStatus
      EstimateCost(model, tokens) Cost
      ValidateCredential() error
  }
```

**关键改进**：new-api 的 Adaptor 接口 `DoRequest` 直接接收 HTTP 上下文（`*gin.Context`），耦合了 Web 框架。Legion 的 ModelProvider 应该是纯粹的领域接口，不依赖任何 HTTP 框架。

### 1.2 计费会话模式 — 优雅的消费控制

new-api 的 `BillingSession` 模式是计费设计的精华：

```
PreConsume (预扣) → Execute (执行) → Settle (结算) / Refund (退款)
```

**Legion 应直接复用此模式**，但需要增加维度：

| new-api 计费 | Legion 增强          |
| ---------- | ------------------ |
| 单用户维度      | 公司 → 部门 → Agent 三维 |
| 模型倍率       | 增加任务重要性权重          |
| 消费日志       | 增加决策溯源（为什么选这个模型？）  |
| 无预算熔断      | 告警 → 降级 → 冻结三级熔断   |

### 1.3 中间件管道 — 可组合的请求处理

new-api 的 20+ 中间件管道是 Legion MaaS 层的最佳参考：

```
new-api 管道:
  Auth → RateLimit → Distributor → Relay → Settlement

Legion MaaS 管道 (增强版):
  Auth → TenantContext → RateLimit → SmartRoute → PreConsume
       → ModelProvider.Relay → ResponseNormalize → Settle
       → AuditLog → ObservableMetrics
```

**新增中间件**：
- **TenantContext**: 从请求中提取公司/部门/Agent 上下文
- **SmartRoute**: 替代 Distributor，执行智能路由决策
- **ResponseNormalize**: 统一不同模型的响应格式
- **AuditLog**: 写入不可篡改的审计日志
- **ObservableMetrics**: 实时更新监控指标

<!-- @end-section -->

<!-- @section: smart-routing -->
## 二、智能路由引擎 — Legion MaaS 的核心差异化

这是 new-api 完全不具备、Legion 必须自研的核心能力。

### 2.1 路由决策的三维模型

```
路由决策 = f(任务特征, 模型能力, 约束条件)

任务特征:
  - 任务类型: strategic_planning | code_generation | content_writing | data_extraction
  - 复杂度: low | medium | high | critical
  - 模态需求: text | image | code | multimodal
  - 上下文大小: 需要的 token 数

模型能力 (来自模型注册中心):
  - 推理强度: low | medium | high | super
  - 上下文窗口: 4K | 8K | 32K | 128K | 200K
  - 支持的模态: [text, image, code, ...]
  - 成本: 每 1K token 价格
  - 延迟: 平均响应时间 (ms)
  - 可用性: 当前健康状态、并发槽位

约束条件:
  - 预算约束: 公司/部门/Agent 剩余预算
  - 角色绑定: Agent 角色是否有模型限制
  - 优先级: 是否允许使用高成本模型
  - 熔断状态: 是否触发降级
```

### 2.2 路由策略

```yaml
routing_strategies:
  # 策略一: 角色绑定路由 (最高优先级)
  role_binding:
    ceo:
      primary: { provider: "anthropic", model: "claude-opus-4" }
      fallback_chain: ["claude-sonnet-4", "gpt-4o"]
    developer:
      primary: { provider: "anthropic", model: "claude-sonnet-4" }
      fallback_chain: ["deepseek-coder", "gpt-4o-mini"]
    customer_service:
      primary: { provider: "deepseek", model: "deepseek-chat" }
      fallback_chain: ["qwen-turbo"]
      max_model_tier: "medium"  # 不允许使用高级模型

  # 策略二: 任务特征匹配 (中等优先级)
  task_matching:
    rules:
      - if: { task_type: "strategic_planning" }
        then: { min_reasoning: "high", prefer: "low_latency" }
      - if: { task_type: "code_generation" }
        then: { require: ["code"], prefer: "cost_efficient" }
      - if: { task_type: "content_writing" }
        then: { min_reasoning: "medium", prefer: "quality" }

  # 策略三: 预算感知降级 (条件触发)
  budget_aware:
    - when: { budget_remaining_pct: "< 20%" }
      action: { force_downgrade: true, max_tier: "medium" }
    - when: { budget_remaining_pct: "< 5%" }
      action: { force_downgrade: true, max_tier: "low" }
    - when: { budget_remaining_pct: "< 1%" }
      action: { freeze: true, require_approval: true }
```

### 2.3 路由决策流程

```
请求进入 SmartRoute
  │
  ├── 1. 解析 Agent 身份 → 查角色绑定
  │     └── 有绑定且模型可用? → 直接路由 ✅
  │
  ├── 2. 分析任务特征 → 任务匹配
  │     ├── 提取任务类型、复杂度、模态需求
  │     └── 从模型注册中心筛选候选模型
  │
  ├── 3. 预算检查 → 预算感知
  │     ├── 三类预算 (公司/部门/Agent) 逐一检查
  │     ├── 触发降级? → 重选模型 (降低 tier)
  │     └── 触发冻结? → 返回预算耗尽错误
  │
  ├── 4. 可用性检查 → 健康过滤
  │     ├── 过滤不健康的模型
  │     ├── 过滤无可用并发槽位的模型
  │     └── 候选集为空? → 返回服务不可用
  │
  └── 5. 最优选择 → 评分排序
        ├── 综合评分: 能力匹配度 × 成本效率 × 延迟
        ├── 选择最高分模型
        └── 记录决策原因 (用于审计)
```

### 2.4 决策溯源

每个路由决策必须记录：

```json
{
  "route_id": "rte_abc123",
  "timestamp": "2026-05-04T10:30:00Z",
  "agent_id": "dev-001",
  "task_type": "code_generation",
  "candidates": ["claude-sonnet-4", "deepseek-coder", "gpt-4o"],
  "selected": "claude-sonnet-4",
  "reason": "role_binding_match + best_code_generation_score",
  "budget_remaining": { "company": "85%", "department": "60%", "agent": "45%" },
  "fallback_triggered": false
}
```

<!-- @end-section -->

<!-- @section: model-registry -->
## 三、模型注册中心

### 3.1 模型能力标签体系

借鉴 hermes-agent 的 model_metadata.py，每个注册模型需要携带标准化的能力标签：

```yaml
model_registry_entry:
  id: "anthropic-claude-sonnet-4"
  provider: "anthropic"
  api_mode: "anthropic_messages"    # 决定使用哪个 Transport

  capabilities:
    reasoning_level: "high"         # low | medium | high | super
    context_window: 200000          # tokens
    max_output_tokens: 64000
    modalities: ["text", "image", "code"]
    supports_streaming: true
    supports_tool_calls: true
    supports_parallel_tool_calls: true
    supports_prompt_caching: true
    supports_vision: true

  performance:
    avg_latency_ms: 1200
    p95_latency_ms: 3500
    tokens_per_second: 80

  pricing:
    input_per_1k: 0.003            # USD
    output_per_1k: 0.015
    cache_write_per_1k: 0.00375
    cache_read_per_1k: 0.0003
    currency: "USD"

  constraints:
    max_concurrent: 50              # 最大并发
    rate_limit_per_minute: 100
    availability: 0.999             # SLA

  endpoint:
    base_url: "https://api.anthropic.com"
    health_check_path: "/v1/messages"
    auth_type: "api_key"
```

### 3.2 模型来源

借鉴 new-api 的渠道管理，支持四种来源：

| 来源 | 接入方式 | 鉴权 | 示例 |
|------|---------|------|------|
| 商业 API | API Key 注册 | 平台级 Key | OpenAI, Anthropic, DeepSeek |
| 私有部署 | 内网 Endpoint | 内部认证 | vLLM, TGI 自建 |
| 本地推理 | 自动发现 | 本机信任 | Ollama, llama.cpp |
| 代理渠道 | 渠道配置 | 渠道 Key | new-api, one-api 上游 |

**与 new-api 的关键差异**：new-api 的渠道管理是"纯转发"——收到什么模型名就转发什么。Legion 需要在注册时将外部模型名映射为内部统一的能力标签。

<!-- @end-section -->

<!-- @section: response-normalization -->
## 四、响应规范化层 — 借鉴 hermes 的 Transport 设计

### 4.1 为什么需要规范化

hermes-agent 的 `ProviderTransport` ABC 是整个参考项目分析中最优雅的设计之一。它的核心思想是：**Agent 循环只看到 `NormalizedResponse`，完全不感知底层是哪个提供商的 API**。

Legion 的 MaaS 层应该承担这个规范化职责：

```
                  ┌──────────────────────────┐
                  │    NormalizedResponse     │
                  │  - content: string         │
                  │  - tool_calls: []ToolCall  │
                  │  - finish_reason: string   │
                  │  - reasoning: string       │
                  │  - usage: Usage            │
                  │  - model_id: string         │
                  │  - latency_ms: int          │
                  │  - cost: CostBreakdown      │
                  └──────────────────────────┘
                            ▲
          ┌─────────────────┼─────────────────┐
          │                 │                 │
  AnthropicNormalizer  OpenAINormalizer  GeminiNormalizer
          │                 │                 │
  AnthropicTransport  OpenAITransport  GeminiTransport
```

### 4.2 NormalizedResponse 结构

```go
type NormalizedResponse struct {
    Content      string          `json:"content"`       // 纯文本内容
    ToolCalls    []ToolCall      `json:"tool_calls"`    // 工具调用
    FinishReason string          `json:"finish_reason"` // stop|tool_calls|length|error
    Reasoning    string          `json:"reasoning"`     // 思维链内容
    Usage        Usage           `json:"usage"`         // Token 统计
    ModelID      string          `json:"model_id"`      // 实际使用的模型
    LatencyMs    int             `json:"latency_ms"`    // 响应延迟
    Cost         CostBreakdown   `json:"cost"`          // 成本分解
    Cached       bool            `json:"cached"`        // 是否命中缓存
}

type ToolCall struct {
    ID       string         `json:"id"`
    Type     string         `json:"type"`     // "function"
    Function FunctionCall   `json:"function"`
}

type FunctionCall struct {
    Name      string `json:"name"`
    Arguments string `json:"arguments"` // JSON string
}
```

### 4.3 与 hermes 的差异

| 维度 | hermes | Legion MaaS |
|------|--------|------------|
| 规范化位置 | Agent 循环内部 | MaaS 层（Agent 循环上游） |
| 优势 | Agent 可直连模型 | Agent 完全不知道模型细节 |
| 劣势 | Agent 需要知道模型 | MaaS 层增加一层抽象 |
| 故障转移 | 在 Agent 内部 | 在 MaaS 层透明处理 |

**结论**：Legion 将规范化放在 MaaS 层的优势是 Agent 引擎更简单、更专注。Agent 只需要说"我要完成任务 X"，MaaS 层负责"用哪个模型、怎么调用、如何规范化结果"。

<!-- @end-section -->

<!-- @section: billing-quota -->
## 五、多维配额与计费系统

### 5.1 四维预算模型

new-api 的计费是单用户维度的。Legion 需要在四个维度上管控：

```
公司级预算 (Company Budget)
  └── 部门级预算 (Department Budget)
        └── 项目级预算 (Project Budget)
              └── Agent 级预算 (Agent Budget)

每个维度:
  - 月度 Token 上限
  - 月度费用上限 (CNY)
  - 单次任务费用上限
  - 模型等级限制
```

### 5.2 三级熔断机制

```
告警 (Warning): 预算消耗 80%
  → 通知管理员
  → Agent 收到预算提醒
  → 自动开始偏好低成本模型

降级 (Downgrade): 预算消耗 95%
  → 强制切换至中低级模型
  → 禁止使用高级模型 (Opus 级)
  → 延迟非关键任务

冻结 (Freeze): 预算消耗 100%
  → 停止所有模型调用
  → 关键业务需管理员手动审批放行
  → "预算耗尽"事件写入审计日志
```

### 5.3 预消费 + 后结算模式

直接复用 new-api 的成熟模式：

```go
type BillingSession struct {
    SessionID    string
    AgentID      string
    ModelID      string
    PreConsumed  float64   // 预扣费用
    ActualCost   float64   // 实际费用
    Status       string    // pre_consumed | executing | settled | refunded
    CreatedAt    time.Time
}

// 1. 预扣
session := PreConsume(agentID, modelID, estimatedTokens)
if !session.Sufficient() {
    return ErrBudgetExceeded
}

// 2. 执行
response := provider.ChatCompletion(ctx, request)

// 3. 结算
actualCost := CalculateActualCost(response.Usage, model.Pricing)
Settle(session, actualCost)

// 4. 退款 (超额预扣)
if session.PreConsumed > actualCost {
    Refund(session, session.PreConsumed - actualCost)
}
```

<!-- @end-section -->

<!-- @section: caching -->
## 六、语义缓存与成本优化

### 6.1 多层缓存策略

借鉴 new-api 和 hermes 的经验：

| 缓存层 | 命中条件 | TTL | 节省 |
|--------|---------|-----|------|
| 精确匹配缓存 | 完全相同请求 | 24h | 100% |
| 语义相似缓存 | Embedding 相似度 > 0.95 | 1h | 100% |
| 摘要缓存 | 相同主题 + 新数据 | 随源变 | 30-50% |
| 提示缓存 | Anthropic cache_control | 5min | 90% (输入) |

### 6.2 提示缓存稳定性

hermes 的关键洞察：**对话中期永不修改系统提示/工具集**。这保持了 Anthropic 提示缓存的稳定性。Legion 的 MaaS 层应将此作为缓存策略的核心约束。

<!-- @end-section -->

<!-- @section: design-decisions -->
## 七、关键设计决策

| 决策 | 选择 | 理由 |
|------|------|------|
| 适配器耦合 | 纯领域接口 | new-api 耦合 gin 的教训 |
| 路由模式 | 策略链 (角色→任务→预算) | 支持不同维度的灵活组合 |
| 规范化位置 | MaaS 层 | Agent 引擎更简单 |
| 计费模式 | 预消费 + 后结算 | new-api 验证过的成熟模式 |
| 预算维度 | 公司/部门/项目/Agent 四维 | Legion 的组织架构特性 |
| 熔断策略 | 告警→降级→冻结三级 | 渐进式、不中断业务 |
| 缓存粒度 | 精确+语义+摘要+提示 四层 | 覆盖不同场景的成本优化 |
| 技术栈 | Go | stardust 基础库 + 高性能并发 |

### 不做什么

1. **不做纯透明代理** — new-api 已经做得很好了，Legion 要做的是智能调度
2. **不在 MaaS 层处理 Agent 逻辑** — 那是 Agent 引擎的职责
3. **不做自建模型推理** — 那是 vLLM/TGI 的职责，Legion 只做调度
4. **不耦合 Web 框架** — 适配器接口是纯 Go 接口

<!-- @end-section -->

<!-- @section: related -->
## 相关文档

- [[../analysis/maas/01-overview|new-api 项目架构总览]]
- [[../analysis/maas/03-channel-adapter-system|渠道适配器分析]]
- [[../analysis/maas/04-billing-quota-system|计费配额系统分析]]
- [[../analysis/hermes/02-agent-runtime|hermes Transport 传输层分析]]
- [[02-agent-runtime-deep-design|Agent 运行时深度设计]]
- [[07-architecture-integration|系统集成架构深度设计]]

<!-- @end-section -->
