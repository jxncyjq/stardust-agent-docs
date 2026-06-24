---
id: "design-llm-infra-abstraction-001"
title: "LLM 底层基础设施抽象化设计"
aliases: ["LLM抽象层", "模型基础设施", "Provider抽象"]
type: "design"
category: "design/architecture"
tags: ["maas", "sub2api", "llm", "abstraction", "provider", "relay", "billing", "architecture"]
version: "1.0.0"
created: "2026-05-09"
updated: "2026-05-09"
author: "jxncyjq"
status: "draft"
parent: "[[Legion]]"
related_docs:
  - path: "./Legion.md"
  - path: "../../services/maas/design-analysis.md"
  - path: "../analysis/maas/07-maas-insights.md"
  - path: "../analysis/sub2api/02-architecture.md"
---

<!-- @section: overview -->
# LLM 底层基础设施抽象化设计

## 文档目的

基于对 **sub2api**（订阅转 API 网关）和 **maas**（New API，LLM 网关）两个成熟系统的深度分析，提炼出 Legion 平台的 **LLM 底层基础设施抽象层**设计方案。

该抽象层是 Legion MaaS 模型调度层的**技术基础**，向上支撑智能路由、配额管控、可观测三大能力，向下统一管理异构 AI 提供商。

---

## 一、总体架构

```
┌──────────────────────────────────────────────────────────────────┐
│                  Legion MaaS 上层（消费者）                        │
│  Agent 引擎 · 智能路由引擎 · 配额管控引擎 · 可观测平台              │
└──────────────────────────┬───────────────────────────────────────┘
                           │ ModelRequest / ModelResponse
┌──────────────────────────▼───────────────────────────────────────┐
│                 LLM 基础设施抽象层（本文设计对象）                   │
│                                                                  │
│  ┌─────────────┐  ┌──────────────┐  ┌─────────────────────────┐  │
│  │ Provider    │  │ Request      │  │ Billing                 │  │
│  │ Registry    │  │ Pipeline     │  │ Engine                  │  │
│  │ 提供商注册  │  │ 请求管道     │  │ 计费引擎                │  │
│  └─────────────┘  └──────────────┘  └─────────────────────────┘  │
│  ┌─────────────┐  ┌──────────────┐  ┌─────────────────────────┐  │
│  │ Stream      │  │ Failover     │  │ Observability           │  │
│  │ Proxy       │  │ Manager      │  │ Collector               │  │
│  │ 流式代理    │  │ 故障转移     │  │ 遥测采集                │  │
│  └─────────────┘  └──────────────┘  └─────────────────────────┘  │
└──────────────────────────┬───────────────────────────────────────┘
                           │ HTTP / SSE / gRPC
┌──────────────────────────▼───────────────────────────────────────┐
│                   外部 AI 提供商（异构）                            │
│  Anthropic · OpenAI · Google ·  Bedrock · Azure · 本地 Ollama  │
└──────────────────────────────────────────────────────────────────┘
```

<!-- @end-section -->

<!-- @section: provider-abstraction -->
---

## 二、提供商抽象（Provider Abstraction）

### 2.1 核心接口

这是整个抽象层最关键的设计。参照 maas（New API）`Adaptor` 接口，结合 sub2api 的多平台实践，定义统一的 `ModelProvider` 接口：

```go
// ModelProvider 是所有 AI 提供商的统一抽象。
// 每个提供商实现此接口，框架负责生命周期管理和路由。
type ModelProvider interface {
    // --- 元数据 ---
    ProviderID() string       // 唯一标识，如 "anthropic", "openai"
    Protocol() ProviderProtocol // REST_SSE | GRPC | WEBSOCKET
    SupportedModels() []ModelCapability

    // --- 请求生命周期 ---
    BuildRequest(ctx *RequestContext) (*UpstreamRequest, error)
    ExecuteRequest(ctx context.Context, req *UpstreamRequest) (*UpstreamResponse, error)
    ParseResponse(ctx *RequestContext, resp *UpstreamResponse) (*ModelOutput, error)

    // --- 流式支持 ---
    IsStreamCapable() bool
    ParseStreamChunk(chunk []byte) (*StreamEvent, error)

    // --- 健康与元数据 ---
    HealthCheck(ctx context.Context) error
}

// AsyncModelProvider 异步任务提供商（视频/音频生成等长时间任务）
type AsyncModelProvider interface {
    ModelProvider
    SubmitTask(ctx *RequestContext) (*TaskSubmission, error)
    PollTask(ctx context.Context, taskID string) (*TaskResult, error)
    EstimateBilling(req *RequestContext) BillingEstimate
    FinalizeBilling(result *TaskResult) BillingSettlement
}
```

**设计原则**：
- **无状态**：Provider 结构体本身不保存请求状态，状态通过 `RequestContext` 传递
- **可组合**：支持委托模式（如 DeepSeek → Claude/OpenAI 委托）
- **最小接口**：不是所有提供商都实现全部能力（Rerank、Embedding 为可选扩展）

### 2.2 请求上下文（RequestContext）

取代 maas RelayInfo "上帝对象"，按职责拆分为独立结构体：

```go
// RequestContext 是贯穿整个请求生命周期的上下文，按职责分区。
type RequestContext struct {
    // 请求标识
    RequestID string
    TraceID   string

    // 提供商上下文（路由决策后填充）
    Provider *ProviderContext

    // 计费上下文（由计费引擎填充）
    Billing *BillingContext

    // 流式状态（流式请求时有效）
    Stream *StreamContext

    // 原始请求（统一格式）
    Input *ModelInput

    // 转换链记录（调试用）
    ConversionChain []string
}

type ProviderContext struct {
    ProviderID   string
    ProviderType string     // "cloud" | "private" | "local"
    Endpoint     string
    Credentials  Credentials
    ModelID      string     // 提供商侧的实际 ModelID（可能与请求不同，经过映射）
    Timeout      time.Duration
    RetryPolicy  RetryPolicy
}

type BillingContext struct {
    TenantID        string
    ProjectID       string
    AgentID         string
    PreConsumed     int64      // 预扣额度（配额单位）
    ActualConsumed  int64      // 实际消耗
    FundingSource   string     // "wallet" | "subscription" | "free_tier"
    BillingMode     string     // "token_ratio" | "fixed_price" | "tiered_expr"
}

type StreamContext struct {
    IsStream     bool
    PingInterval time.Duration
    Timeout      time.Duration
}
```

### 2.3 提供商工厂与注册

```go
// ProviderRegistry 管理所有已注册的提供商。
type ProviderRegistry interface {
    Register(provider ModelProvider)
    Get(providerID string) (ModelProvider, error)
    GetByModel(modelID string) ([]ModelProvider, error) // 一个模型可能在多个提供商
    ListAll() []ModelProvider
    ListHealthy() []ModelProvider
}

// 工厂函数（编译期注册）
func NewProvider(providerType ProviderType) ModelProvider {
    switch providerType {
    case ProviderAnthropic: return &anthropic.Provider{}
    case ProviderOpenAI:    return &openai.Provider{}
    case ProviderGemini:    return &gemini.Provider{}
    case ProviderBedrock:   return &bedrock.Provider{}
    case ProviderOllama:    return &ollama.Provider{}
    // OpenAI 兼容提供商复用 openai.Provider（委托模式）
    case ProviderAzure, ProviderDeepSeek, ProviderSiliconFlow:
        return openai.NewCompatibleProvider(providerType)
    }
}
```

### 2.4 模型能力描述

```go
// ModelCapability 描述单个模型的能力和定价，用于路由决策。
type ModelCapability struct {
    ModelID         string
    DisplayName     string
    ProviderID      string

    // 能力标签
    Modalities      []Modality    // TEXT | IMAGE | AUDIO | CODE | VIDEO
    ReasoningLevel  ReasoningLevel // LOW | MEDIUM | HIGH | ULTRA
    ContextWindow   int           // tokens
    MaxOutputTokens int

    // 性能指标（运行时动态更新）
    AvgLatencyMs    float64
    SuccessRate     float64       // 过去 1h 的成功率

    // 定价（用于成本路由）
    InputPricePer1M  float64     // USD per 1M input tokens
    OutputPricePer1M float64
    BillingMode      BillingMode
}
```

<!-- @end-section -->

<!-- @section: request-pipeline -->
---

## 三、请求管道（Request Pipeline）

### 3.1 管道架构

借鉴 maas 中间件链和 sub2api failover_loop，设计可插拔的中间件管道：

```
入站请求（统一格式 ModelRequest）
    │
    ▼
┌───────────────────────────────────────────────┐
│               RequestPipeline                  │
│                                               │
│  [1] AuthGate         — API Key 验证           │
│  [2] RateLimiter      — 多维度限流             │
│  [3] QuotaChecker     — 配额预检               │
│  [4] ModelRouter      — 路由决策               │
│  [5] RequestTransformer — 格式转换到提供商格式  │
│  [6] ProviderExecutor — 实际执行               │
│  [7] ResponseNormalizer — 响应标准化           │
│  [8] BillingSettler   — 计费结算               │
│  [9] TelemetryEmitter — 遥测上报               │
│                                               │
└───────────────────────────────────────────────┘
    │
    ▼
出站响应（统一格式 ModelResponse）
```

### 3.2 中间件接口

```go
// PipelineMiddleware 是管道中每个处理节点的接口。
type PipelineMiddleware interface {
    Name() string
    Process(ctx *RequestContext, next PipelineHandler) error
}

type PipelineHandler func(ctx *RequestContext) error
```

### 3.3 模型映射（Model Mapping）

支持多级映射，隔离用户模型名与提供商实际模型名：

```go
// ModelMapper 处理用户请求模型名到提供商模型名的映射。
// 支持链式重定向（最多 5 层，防循环）。
type ModelMapper interface {
    // Map 将用户请求的模型名映射到 (提供商ID, 提供商模型名)
    Map(requestedModel string, tenantID string) (providerID string, providerModel string, err error)
    // RegisterAlias 注册别名（支持运行时热更新）
    RegisterAlias(alias string, target ModelAlias)
}

type ModelAlias struct {
    ProviderID    string
    ProviderModel string
    // 为 nil 时表示透传，由路由引擎决策
}
```

<!-- @end-section -->

<!-- @section: billing-engine -->
---

## 四、计费引擎（Billing Engine）

### 4.1 设计原则

参照 maas BillingSession 生命周期和 sub2api 的并发控制，设计预扣→结算→退款三段式：

```
PreConsume(estimate)
    ├── 信任旁路检查（高余额用户跳过预扣，减少锁争用）
    ├── 扣减令牌配额
    └── 扣减资金来源（钱包 or 订阅）

Execute()  ← 上游 API 调用

Settle(actual)
    ├── delta = actual - estimate
    ├── delta > 0 → 补扣
    ├── delta < 0 → 退还多余
    └── 触发告警（余额低于阈值时）

失败时 → Refund()（幂等，通过 RequestID 去重）
```

### 4.2 资金来源抽象

```go
// FundingSource 抽象资金来源，钱包和订阅实现相同接口。
type FundingSource interface {
    SourceType() string              // "wallet" | "subscription" | "free_tier"
    PreConsume(amount int64) error
    Settle(delta int64) error
    Refund(requestID string) error   // 幂等
    Balance() (int64, error)
}

// FundingSourceSelector 根据租户策略选择资金来源。
type FundingSourceSelector interface {
    Select(ctx *BillingContext) (FundingSource, error)
}

// 支持策略：
//   "subscription_first" — 先用订阅额度（默认）
//   "wallet_first"       — 先用钱包余额
//   "subscription_only"  — 仅用订阅
//   "wallet_only"        — 仅用钱包
```

### 4.3 定价引擎

```go
// PricingEngine 计算请求的实际成本。
type PricingEngine interface {
    // EstimateQuota 在请求前估算配额（用于预扣）
    EstimateQuota(modelID string, estimatedInput, estimatedOutput int) (int64, error)

    // CalculateQuota 根据实际 usage 计算最终配额
    CalculateQuota(modelID string, usage *TokenUsage) (int64, error)
}

// 三种定价模式（对应 maas billingexpr 的三种模式）
type BillingMode int

const (
    // BillingModeRatio 倍率模式：quota = tokens × model_ratio × group_ratio
    BillingModeRatio BillingMode = iota
    // BillingModeFixed 固定价格：quota = model_price × QuotaPerUnit
    BillingModeFixed
    // BillingModeTieredExpr 阶梯表达式：支持缓存折扣、分时定价等复杂规则
    BillingModeTieredExpr
)

// TokenUsage 标准化 Token 用量（跨提供商统一结构）
type TokenUsage struct {
    InputTokens       int
    OutputTokens      int
    CacheReadTokens   int  // Anthropic prompt cache
    CacheWriteTokens  int
    AudioInputTokens  int
    AudioOutputTokens int
    ImageTokens       int
}
```

<!-- @end-section -->

<!-- @section: stream-proxy -->
---

## 五、流式代理（Stream Proxy）

### 5.1 双 Goroutine 模式

参照 maas SSE 双 goroutine 模式，适用于所有 SSE/流式场景：

```go
// StreamProxy 处理从上游到下游的 SSE 流代理。
type StreamProxy struct {
    pingInterval time.Duration
    timeout      time.Duration
    bufferSize   int
}

func (p *StreamProxy) Relay(
    ctx context.Context,
    upstream io.ReadCloser,
    downstream ResponseWriter,
    handler StreamChunkHandler,
) (*StreamResult, error) {
    dataChan := make(chan []byte, p.bufferSize)
    stopChan := make(chan struct{})

    // Scanner Goroutine: 读上游 SSE 流
    go func() {
        defer close(dataChan)
        scanner := bufio.NewScanner(upstream)
        pingTicker := time.NewTicker(p.pingInterval)
        defer pingTicker.Stop()

        for {
            select {
            case <-stopChan:
                return
            case <-pingTicker.C:
                downstream.WritePing() // 保活
            default:
                if scanner.Scan() {
                    dataChan <- scanner.Bytes()
                } else {
                    return
                }
            }
        }
    }()

    // Handler Goroutine: 处理并写回客户端
    var result StreamResult
    for chunk := range dataChan {
        event, err := handler.ParseChunk(chunk)
        if err != nil {
            close(stopChan)
            return nil, err
        }
        if event != nil {
            downstream.WriteEvent(event)
            result.Accumulate(event)
        }
    }
    return &result, nil
}
```

### 5.2 流式格式标准化

不同提供商的 SSE 格式各异，统一转换为 Legion 内部标准格式后再输出：

```go
// StreamEvent 是所有提供商流式响应的统一内部格式。
type StreamEvent struct {
    Type    StreamEventType // DELTA | DONE | ERROR | PING
    Delta   *ContentDelta
    Usage   *TokenUsage     // 仅 DONE 事件有值
    Error   *ProviderError
}

// StreamChunkHandler 各提供商实现各自的 chunk 解析逻辑。
type StreamChunkHandler interface {
    ParseChunk(chunk []byte) (*StreamEvent, error)
}
```

<!-- @end-section -->

<!-- @section: failover-manager -->
---

## 六、故障转移管理器（Failover Manager）

### 6.1 设计目标

参照 sub2api `failover_loop.go` 的实战设计，抽象为独立的 `FailoverManager`：

```go
// FailoverManager 管理单次请求的故障转移策略。
type FailoverManager struct {
    maxProviderSwitches   int           // 最多切换提供商次数
    maxSameProviderRetry  int           // 同一提供商最大重试次数
    sameProviderDelay     time.Duration // 同提供商重试间隔
    switchBackoffBase     time.Duration // 切换递增退避基数

    failedProviders map[string]int  // providerID → 失败次数
    retryCount      map[string]int  // providerID → 当前重试次数
}

// ShouldRetry 根据错误类型决策是否重试及策略。
func (m *FailoverManager) ShouldRetry(err error) RetryDecision {
    switch ClassifyError(err) {
    case ErrClassRateLimit:
        return RetryDecision{Action: SwitchProvider, Delay: 0}
    case ErrClassTransient:
        return RetryDecision{Action: RetryCurrentProvider, Delay: m.sameProviderDelay}
    case ErrClassFatal:
        return RetryDecision{Action: Abort}
    case ErrClassTimeout:
        return RetryDecision{Action: SwitchProvider, Delay: m.switchBackoffBase}
    }
}
```

### 6.2 错误分类

```go
// ErrorClass 用于决策重试策略，对上层屏蔽提供商的错误码差异。
type ErrorClass int

const (
    ErrClassTransient  ErrorClass = iota // 可重试：网络抖动、临时过载
    ErrClassRateLimit                    // 限流：切换提供商或等待
    ErrClassQuota                        // 配额耗尽：切换提供商
    ErrClassBadRequest                   // 请求错误：不重试
    ErrClassFatal                        // 严重错误：不重试，记录告警
    ErrClassTimeout                      // 超时：切换提供商
)

// ProviderErrorMapper 各提供商实现自己的错误码到 ErrorClass 的映射。
type ProviderErrorMapper interface {
    ClassifyError(statusCode int, body []byte) ErrorClass
}
```

<!-- @end-section -->

<!-- @section: provider-implementations -->
---

## 七、提供商实现层

### 7.1 目录结构

```
legion/maas/
├── provider/
│   ├── provider.go            ← ModelProvider 接口定义
│   ├── registry.go            ← ProviderRegistry 实现
│   ├── factory.go             ← 工厂函数
│   │
│   ├── anthropic/             ← Anthropic Claude
│   │   ├── provider.go        ← 实现 ModelProvider
│   │   ├── request.go         ← OpenAI → Claude 格式转换
│   │   ├── response.go        ← Claude → 标准格式转换
│   │   ├── stream.go          ← SSE chunk 解析
│   │   └── errors.go          ← 错误分类映射
│   │
│   ├── openai/                ← OpenAI（被多个 OpenAI 兼容提供商复用）
│   │   ├── provider.go
│   │   ├── compatible.go      ← NewCompatibleProvider（Azure/DeepSeek/SiliconFlow 等）
│   │   └── ...
│   │
│   ├── gemini/                ← Google Gemini
│   ├── bedrock/               ←  Bedrock
│   ├── vertex/                ← Google Vertex AI
│   └── ollama/                ← 本地 Ollama（私有部署）
│
├── pipeline/
│   ├── pipeline.go            ← RequestPipeline 实现
│   ├── middleware/
│   │   ├── auth.go
│   │   ├── rate_limit.go
│   │   ├── quota_check.go
│   │   ├── router.go
│   │   └── telemetry.go
│   └── transformer/
│       └── model_mapper.go
│
├── billing/
│   ├── engine.go              ← PricingEngine 实现
│   ├── session.go             ← BillingSession 生命周期
│   ├── funding/
│   │   ├── wallet.go          ← WalletFunding
│   │   └── subscription.go    ← SubscriptionFunding
│   └── pricing/
│       ├── ratio.go           ← 倍率定价
│       ├── fixed.go           ← 固定价格
│       └── tiered.go          ← 阶梯表达式
│
├── stream/
│   ├── proxy.go               ← StreamProxy（双 goroutine）
│   └── event.go               ← StreamEvent 标准格式
│
└── failover/
    ├── manager.go             ← FailoverManager
    └── classifier.go          ← ErrorClass 分类
```

### 7.2 OpenAI 兼容提供商（委托模式）

参照 maas 的 DeepSeek 委托模式，OpenAI 兼容的提供商无需重复实现：

```go
// CompatibleOpenAIProvider 用于 Azure、DeepSeek、SiliconFlow 等 OpenAI 兼容提供商。
// 复用 openai.Provider 的格式转换逻辑，仅覆盖 URL 构建和认证方式。
type CompatibleOpenAIProvider struct {
    *openai.Provider                  // 委托给 openai.Provider
    providerID  string
    endpointFn  func(model string) string // 自定义 URL 构建
    authFn      func(req *http.Header)    // 自定义认证注入
}

func (p *CompatibleOpenAIProvider) BuildRequest(ctx *RequestContext) (*UpstreamRequest, error) {
    req, err := p.Provider.BuildRequest(ctx) // 复用 OpenAI 格式转换
    if err != nil {
        return nil, err
    }
    req.URL = p.endpointFn(ctx.Provider.ModelID) // 覆盖 URL
    p.authFn(&req.Headers)                        // 覆盖认证
    return req, nil
}
```

<!-- @end-section -->

<!-- @section: key-design-decisions -->
---

## 八、关键设计决策

### 8.1 统一输入格式

Legion 内部统一使用一种中间格式（**不绑定 OpenAI 格式**），避免 maas 中以 OpenAI 为核心导致的"Claude → OpenAI → Gemini"二级转换链：

```go
// ModelInput 是 Legion 统一的请求格式。
// 各提供商实现将其转换为自身格式。
type ModelInput struct {
    Messages    []Message
    System      string
    Model       string
    MaxTokens   int
    Temperature float64
    Stream      bool
    Tools       []Tool
    // ... 其他通用参数
}

// ModelOutput 是统一的响应格式。
type ModelOutput struct {
    Content  string
    Usage    *TokenUsage
    StopReason string
    ToolCalls  []ToolCall
}
```

### 8.2 类型安全的 Context 传递

修正 maas 中大量 `c.Set(key, value)` + 类型断言的问题：

```go
// 不使用 map[string]any，而是强类型结构体
// Bad（maas 做法）:
//   c.Set("provider", provider)
//   provider := c.Get("provider").(*ProviderContext)

// Good（Legion 做法）:
//   ctx.Provider = &ProviderContext{...}
//   provider := ctx.Provider  // 编译期类型安全
```

### 8.3 提供商健康状态管理

```go
// ProviderHealthMonitor 持续监控提供商健康状态。
type ProviderHealthMonitor interface {
    // ReportSuccess/Failure 由 FailoverManager 调用，更新健康指标
    ReportSuccess(providerID string, latencyMs float64)
    ReportFailure(providerID string, errClass ErrorClass)

    // GetHealthScore 返回 0.0~1.0 的健康分，用于路由权重
    GetHealthScore(providerID string) float64

    // TempDisable 临时禁用提供商（熔断）
    TempDisable(providerID string, until time.Time, reason string)
    IsHealthy(providerID string) bool
}
```

### 8.4 并发控制

参照 sub2api Redis Sorted Set 并发槽位设计：

```go
// ConcurrencyLimiter 控制对单个提供商账号的并发请求数。
// 使用 Redis ZSET 实现分布式并发计数，支持多实例部署。
type ConcurrencyLimiter interface {
    // Acquire 尝试获取并发槽位，返回释放函数
    Acquire(ctx context.Context, providerID string, requestID string) (release func(), err error)
    // CurrentCount 返回当前并发数（含过期清理）
    CurrentCount(ctx context.Context, providerID string) (int, error)
}
```

<!-- @end-section -->

<!-- @section: data-models -->
---

## 九、核心数据模型

### 9.1 提供商配置（数据库）

```sql
-- provider_configs 提供商配置表
CREATE TABLE provider_configs (
    id              BIGSERIAL PRIMARY KEY,
    provider_id     VARCHAR(64) NOT NULL UNIQUE,  -- "anthropic", "openai-azure-eastus"
    provider_type   VARCHAR(32) NOT NULL,          -- "cloud" | "private" | "local"
    protocol        VARCHAR(16) NOT NULL,          -- "rest_sse" | "grpc" | "websocket"
    endpoint        VARCHAR(512) NOT NULL,
    credentials     JSONB NOT NULL DEFAULT '{}',  -- 加密存储
    extra           JSONB NOT NULL DEFAULT '{}',  -- 提供商特定配置
    status          VARCHAR(16) NOT NULL DEFAULT 'active',
    priority        INT NOT NULL DEFAULT 100,
    concurrency     INT NOT NULL DEFAULT 10,       -- 最大并发
    rpm_limit       INT,                           -- 每分钟请求限制
    health_score    FLOAT NOT NULL DEFAULT 1.0,
    last_health_at  TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- model_capabilities 模型能力注册表
CREATE TABLE model_capabilities (
    id               BIGSERIAL PRIMARY KEY,
    model_id         VARCHAR(128) NOT NULL,        -- Legion 统一模型名
    provider_id      VARCHAR(64) NOT NULL REFERENCES provider_configs(provider_id),
    provider_model   VARCHAR(128) NOT NULL,        -- 提供商侧实际模型名
    modalities       TEXT[] NOT NULL DEFAULT '{"text"}',
    reasoning_level  VARCHAR(16) NOT NULL DEFAULT 'medium',
    context_window   INT NOT NULL,
    max_output       INT NOT NULL,
    input_price      NUMERIC(12, 8),               -- USD per 1M tokens
    output_price     NUMERIC(12, 8),
    billing_mode     VARCHAR(32) NOT NULL DEFAULT 'token_ratio',
    billing_config   JSONB NOT NULL DEFAULT '{}', -- 阶梯表达式等复杂配置
    enabled          BOOLEAN NOT NULL DEFAULT TRUE,
    UNIQUE (model_id, provider_id)
);

-- model_aliases 模型别名/映射表
CREATE TABLE model_aliases (
    alias       VARCHAR(128) PRIMARY KEY,
    model_id    VARCHAR(128) NOT NULL,
    tenant_id   VARCHAR(64),    -- NULL 表示全局别名
    UNIQUE (alias, tenant_id)
);
```

### 9.2 调用日志（时序数据）

```sql
-- inference_logs 推理调用日志
CREATE TABLE inference_logs (
    id              BIGSERIAL PRIMARY KEY,
    request_id      VARCHAR(64) NOT NULL UNIQUE,
    trace_id        VARCHAR(64),
    tenant_id       VARCHAR(64) NOT NULL,
    project_id      VARCHAR(64),
    agent_id        VARCHAR(64),
    provider_id     VARCHAR(64) NOT NULL,
    model_id        VARCHAR(128) NOT NULL,
    input_tokens    INT,
    output_tokens   INT,
    cache_read      INT,
    cache_write     INT,
    quota_consumed  BIGINT NOT NULL DEFAULT 0,
    billing_mode    VARCHAR(32),
    latency_ms      INT,
    is_stream       BOOLEAN NOT NULL DEFAULT FALSE,
    status          VARCHAR(16) NOT NULL,   -- "success" | "error" | "timeout"
    error_class     VARCHAR(32),
    retry_count     INT NOT NULL DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
-- 按时间分区（按月）
```

<!-- @end-section -->

<!-- @section: comparison -->
---

## 十、与现有系统的对比

### 10.1 相比 maas（New API）的改进

| 方面 | maas 现状 | Legion 改进 |
|------|-----------|-------------|
| Context 传递 | `map[string]any` 类型不安全，运行时类型断言 | 强类型 `RequestContext` 结构体，编译期安全 |
| Provider 接口 | `Adaptor` 以 OpenAI 为核心（其他格式二级转换） | `ModelProvider` 以 Legion 中间格式为核心，直接转换 |
| RelayInfo 膨胀 | 顶层 61 字段 + 多个嵌入子结构的上帝对象 | 按职责拆分为 `Provider/Billing/Stream` 三个独立 Context |
| 错误分类 | 散落在各适配器，无统一标准 | `ProviderErrorMapper` 接口，每个提供商实现各自映射 |
| 健康管理 | 基于手动 auto_ban 逻辑 | `ProviderHealthMonitor` 主动监控 + 自动熔断 |
| 异步任务计费 | 三段计费仅在 TaskAdaptor 接口中 | 独立 `AsyncModelProvider` 接口，一等公民 |

### 10.2 相比 sub2api 的改进

| 方面 | sub2api 现状 | Legion 改进 |
|------|-------------|-------------|
| 平台范围 | 以订阅账号为核心，平台固定（Claude/OpenAI/Gemini） | 提供商注册制，运行时动态扩展 |
| 并发控制 | Redis ZSET + Lua 脚本（优秀，直接复用） | 同方案，抽象为 `ConcurrencyLimiter` 接口 |
| 故障转移 | `FailoverState` 结构体内联在 handler 中 | 独立 `FailoverManager`，与业务逻辑解耦 |
| 计费模型 | 以账号和用户余额为中心 | 扩展为租户/项目/Agent 多级配额 |

<!-- @end-section -->

<!-- @section: implementation-guide -->
---

## 十一、实施指南

### 11.1 新增提供商步骤

1. 在 `provider/<name>/` 下创建目录
2. 实现 `ModelProvider` 接口的所有方法
3. 实现 `ProviderErrorMapper` 接口（错误分类）
4. 实现 `StreamChunkHandler` 接口（如支持流式）
5. 在 `provider/factory.go` 的工厂函数中注册
6. 在数据库 `provider_configs` 中添加配置
7. 在 `model_capabilities` 中注册该提供商支持的模型

### 11.2 添加新计费模式

1. 在 `billing/pricing/` 下实现 `PricingCalculator` 接口
2. 在 `BillingMode` 常量中注册新模式
3. `PricingEngine` 工厂函数中添加 case

### 11.3 中间件扩展

管道中每个节点都是独立中间件，新增功能只需：
1. 实现 `PipelineMiddleware` 接口
2. 在 `pipeline.go` 的中间件链中插入合适位置

<!-- @end-section -->

<!-- @section: related -->
---

## 相关文档

- [[Legion.md|Legion 整体架构]]
- [[design-analysis|MaaS 模型调度层设计分析]]
- [[07-maas-insights|MaaS 洞察与 Legion 参考]]
- [[02-architecture|Sub2API 架构分析]]
- [[03-channel-adapter-system|渠道适配器系统分析（maas）]]
- [[04-billing-quota-system|计费与配额系统分析（maas）]]

<!-- @end-section -->
