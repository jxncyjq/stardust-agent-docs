---
id: "spec-component-model-provider-001"
title: "ModelProvider 组件规范"
aliases: ["ModelProvider规范", "提供商组件", "provider-spec"]
type: "spec"
category: "design/architecture/components"
tags: ["component-spec", "provider", "maas", "interface", "contract"]
version: "0.2.0"
created: "2026-05-09"
updated: "2026-05-09"
author: "jxncyjq"
status: "draft"
parent: "spec-llm-infra-abstraction-001"

# 依赖关系（见 component-registry.md）
component_id: "C01"
layer: "L1"
depends_on: []          # 无必须依赖，是最底层扩展点
optional_deps: []
conflicts_with: []
required_by:
  - "C03"               # ProviderRegistry 聚合 ModelProvider 实例
  - "C05"               # ProviderHealthMonitor 监控 ModelProvider 的健康
  - "C06"               # ConcurrencyLimiter 对 ModelProvider 限流
  - "C50"               # StreamProxy 通过 ModelProvider 解析流式 chunk
assembly_profiles:
  - minimal
  - standard
  - enterprise
  - embedded
---

<!-- @section: overview -->
# ModelProvider 组件规范

## 1. 组件定位

`ModelProvider` 是 LLM 基础设施抽象层的**核心扩展点**。每个 AI 提供商（Anthropic、OpenAI、Gemini 等）实现此接口，框架负责路由、重试、计费和遥测——提供商实现**只负责协议适配**。

```
框架（路由 / 重试 / 计费）
        │
        │ 调用
        ▼
  ModelProvider          ← 本规范定义的接口
        │
        │ HTTP/gRPC
        ▼
  上游 AI 服务
```

**本规范的读者**：实现新提供商的工程师、框架内部调用方。

<!-- @end-section -->

<!-- @section: interface -->
## 2. 接口定义

### 2.1 必须实现（Required）

```go
// ModelProvider 是所有同步 AI 提供商的统一接口。
// 实现方必须保证所有 Required 方法是并发安全的（goroutine-safe）。
// Provider 结构体本身应当是无状态的，请求状态通过 RequestContext 传递。
type ModelProvider interface {
    // --- 身份 ---

    // ID 返回提供商的唯一标识符，格式为 kebab-case，如 "anthropic"、"openai-azure-eastus"。
    // 在整个进程生命周期内返回值不变。
    ID() string

    // --- 请求构造 ---

    // BuildRequest 将 Legion 统一格式的 ModelInput 转换为上游 API 所需的请求体和头部。
    // 调用方保证 ctx 和 ctx.Input 非 nil。
    // 实现不得修改 ctx.Input 的内容。
    // 失败时返回 *ProviderError（见第 5 节）。
    BuildRequest(ctx *RequestContext) (*UpstreamRequest, error)

    // --- 响应解析 ---

    // ParseResponse 将上游返回的原始响应解析为 ModelOutput。
    // 仅在 HTTP 状态码为 2xx 时被调用；非 2xx 由框架通过 ClassifyHTTPError 处理。
    // resp.Body 已被框架读取并缓存，实现方可多次读取。
    ParseResponse(resp *UpstreamResponse) (*ModelOutput, error)

    // --- 流式 ---

    // ParseStreamChunk 将 SSE 的单行 data 字节解析为 StreamEvent。
    // chunk 是去除 "data: " 前缀后的原始内容。
    // 返回 (nil, nil) 表示忽略该行（如注释行、空行）。
    // 返回 StreamEvent{Type: StreamDone} 表示流结束。
    ParseStreamChunk(chunk []byte) (*StreamEvent, error)

    // --- 错误分类 ---

    // ClassifyHTTPError 将提供商特有的 HTTP 错误码 + 响应体映射为 ErrorClass。
    // 框架根据 ErrorClass 决策是否重试以及如何重试。
    // 在 HTTP 状态码非 2xx 时调用，body 可能为空。
    ClassifyHTTPError(statusCode int, body []byte) ErrorClass
}
```

### 2.2 可选扩展（Optional）

框架通过类型断言检查提供商是否实现以下可选接口，未实现时使用默认行为。

```go
// ProviderWithHealthCheck 支持主动健康检测（未实现则框架依赖被动错误计数）。
type ProviderWithHealthCheck interface {
    ModelProvider
    // HealthCheck 主动探测上游可用性。ctx 含超时（默认 5s）。
    // 返回 nil 表示健康。
    HealthCheck(ctx context.Context) error
}

// ProviderWithModelList 支持动态获取模型列表（未实现则使用静态注册列表）。
type ProviderWithModelList interface {
    ModelProvider
    // ListModels 返回该提供商当前可用的模型 ID 列表。
    ListModels(ctx context.Context) ([]string, error)
}

// ProviderWithEmbedding 支持文本嵌入（不实现则框架拒绝该提供商的 Embedding 请求）。
type ProviderWithEmbedding interface {
    ModelProvider
    BuildEmbeddingRequest(ctx *RequestContext) (*UpstreamRequest, error)
    ParseEmbeddingResponse(resp *UpstreamResponse) (*EmbeddingOutput, error)
}

// ProviderWithRerank 支持文档重排。
type ProviderWithRerank interface {
    ModelProvider
    BuildRerankRequest(ctx *RequestContext) (*UpstreamRequest, error)
    ParseRerankResponse(resp *UpstreamResponse) (*RerankOutput, error)
}
```

### 2.3 异步任务提供商（独立接口）

视频/音频等长时间生成任务使用独立接口，**不继承** `ModelProvider`：

```go
// AsyncTaskProvider 异步任务提供商（视频/音频/图像生成等）。
type AsyncTaskProvider interface {
    ID() string

    // SubmitTask 提交任务，返回提供商侧的任务 ID。
    SubmitTask(ctx *RequestContext) (*TaskSubmission, error)

    // PollTask 轮询任务状态。框架负责轮询调度，实现方只需返回当前状态。
    PollTask(ctx context.Context, taskID string) (*TaskPollResult, error)

    // EstimateBilling 在提交前估算计费（用于预扣额度）。
    EstimateBilling(input *ModelInput) BillingEstimate

    // SettleBilling 任务完成时最终结算（返回实际消耗）。
    // result 为 nil 表示任务失败，实现方应返回零值而非误扣。
    SettleBilling(result *TaskPollResult) BillingSettlement

    ClassifyHTTPError(statusCode int, body []byte) ErrorClass
}
```

<!-- @end-section -->

<!-- @section: data-types -->
## 3. 数据类型定义

### 3.1 输入输出

```go
// ModelInput 是 Legion 统一的推理请求格式，不绑定任何特定提供商协议。
type ModelInput struct {
    Messages    []Message
    System      string        // 系统提示（空字符串表示不设置）
    Model       string        // 提供商侧实际模型 ID（已经过 ModelMapper 转换）
    MaxTokens   int           // 0 表示使用提供商默认值
    Temperature *float64      // nil 表示使用提供商默认值
    TopP        *float64
    Stream      bool
    Tools       []Tool        // nil 或空切片均表示不启用工具
    ToolChoice  *ToolChoice   // nil 表示 auto
    StopSeqs    []string
    ExtraParams map[string]any // 透传给上游的额外参数（提供商特有）
}

// ModelOutput 是统一的推理响应格式。
type ModelOutput struct {
    Content    string
    Usage      TokenUsage
    StopReason StopReason  // STOP | MAX_TOKENS | TOOL_USE | ERROR
    ToolCalls  []ToolCall  // 空切片表示无工具调用
    RawBody    []byte      // 上游原始响应，仅用于调试和日志
}

// TokenUsage 跨提供商统一的 Token 用量。
// 提供商不支持某一字段时，该字段值为 0（而非 -1 或其他哨兵值）。
type TokenUsage struct {
    InputTokens       int
    OutputTokens      int
    CacheReadTokens   int // Anthropic prompt cache / OpenAI cached tokens
    CacheWriteTokens  int
    AudioInputTokens  int
    AudioOutputTokens int
}
```

### 3.2 上下游传输

```go
// UpstreamRequest 是 BuildRequest 的输出，由框架负责实际发送。
// 实现方不应直接发送 HTTP 请求。
type UpstreamRequest struct {
    Method  string
    URL     string
    Headers map[string]string
    Body    []byte
}

// UpstreamResponse 是框架传给 ParseResponse 的原始响应。
type UpstreamResponse struct {
    StatusCode int
    Headers    map[string][]string
    Body       []byte              // 框架已完整读取，可多次访问
    Latency    time.Duration
}
```

### 3.3 流式事件

```go
// StreamEvent 是 ParseStreamChunk 返回的标准流事件。
type StreamEvent struct {
    Type    StreamEventType
    Delta   string          // DELTA 类型时有值
    Usage   *TokenUsage     // 仅 DONE 类型有值，部分提供商在中间块就上报
    Error   *ProviderError  // ERROR 类型时有值
}

type StreamEventType int

const (
    StreamDelta StreamEventType = iota // 内容增量
    StreamDone                         // 流结束（框架停止读取）
    StreamError                        // 提供商报告流内错误
)
```

<!-- @end-section -->

<!-- @section: error-types -->
## 4. 错误分类体系

### 4.1 ErrorClass（用于重试决策）

```go
// ErrorClass 是框架路由重试决策的依据。
// 由 ClassifyHTTPError 返回，框架根据此值决定策略（见下表）。
type ErrorClass int

const (
    // ErrClassTransient 临时错误，可立即重试当前提供商。
    // 场景：网络抖动、服务临时过载（HTTP 500/502/503 非速率限制原因）
    ErrClassTransient ErrorClass = iota

    // ErrClassRateLimit 速率限制，应切换提供商或按 Retry-After 等待。
    // 场景：HTTP 429，响应头含 Retry-After
    ErrClassRateLimit

    // ErrClassQuotaExhausted 提供商侧配额耗尽，应切换提供商。
    // 场景：账单额度不足、订阅到期
    ErrClassQuotaExhausted

    // ErrClassBadRequest 请求本身有问题，不重试（重试无效）。
    // 场景：HTTP 400，模型不支持该参数、内容被安全过滤
    ErrClassBadRequest

    // ErrClassAuthFailure 认证失败，应告警并禁用该提供商配置。
    // 场景：HTTP 401/403，API Key 无效或过期
    ErrClassAuthFailure

    // ErrClassUnavailable 提供商不可用，应切换并触发熔断。
    // 场景：HTTP 503 且 Retry-After 过长、连接超时
    ErrClassUnavailable

    // ErrClassContentFiltered 内容被过滤，不重试，向上层返回明确错误。
    // 场景：提供商因内容政策拒绝请求
    ErrClassContentFiltered
)
```

**框架对各 ErrorClass 的默认处理策略**：

| ErrorClass | 是否重试当前提供商 | 是否切换提供商 | 是否触发熔断 |
|---|---|---|---|
| Transient | 是（最多 2 次） | 超限后切换 | 否 |
| RateLimit | 否 | 是 | 否 |
| QuotaExhausted | 否 | 是 | 是（该提供商暂时禁用） |
| BadRequest | 否 | 否 | 否 |
| AuthFailure | 否 | 否 | 是（告警，人工介入） |
| Unavailable | 否 | 是 | 是 |
| ContentFiltered | 否 | 否 | 否 |

### 4.2 ProviderError（返回给调用方）

```go
// ProviderError 是提供商方法返回的错误类型。
// 使用 errors.As 检测，不要直接类型断言。
type ProviderError struct {
    Class      ErrorClass
    Code       string    // 提供商原始错误码，如 "invalid_api_key"
    Message    string    // 人类可读描述
    Retryable  bool      // 冗余字段，与 Class 保持一致，方便快速检查
    RetryAfter time.Duration // 非零时框架遵从此等待时间
}

func (e *ProviderError) Error() string {
    return fmt.Sprintf("provider error [%s]: %s", e.Code, e.Message)
}
```

> **约定**：`BuildRequest` 和 `ParseResponse` 只返回 `*ProviderError`，框架统一处理。
> 不得返回其他 error 类型（会导致框架无法正确分类）。

<!-- @end-section -->

<!-- @section: lifecycle -->
## 5. 生命周期

### 5.1 初始化流程

`ModelProvider` 的实例由 `ProviderFactory` 创建，遵循以下流程：

```
ProviderFactory.Build(cfg ProviderConfig) (ModelProvider, error)
    │
    ├─ 1. 校验必填配置（ID、Endpoint、Credentials）
    ├─ 2. 初始化 HTTP 客户端（连接池、超时、代理）
    ├─ 3. 解密 Credentials（框架在此之前完成）
    └─ 4. 返回 Provider 实例（不发送任何网络请求）

// 注意：Build 不做健康检测，不应有副作用。
// 上线后由 HealthMonitor 周期性调用 HealthCheck（如实现了该可选接口）。
```

### 5.2 销毁流程

```go
// ProviderCloser 是可选接口。
// 如果 Provider 持有连接池或后台 goroutine，必须实现此接口。
type ProviderCloser interface {
    ModelProvider
    // Close 释放资源。框架在进程退出或动态卸载提供商时调用。
    // 调用后提供商不再接收新请求（框架保证）。
    // 应设置超时，不得无限阻塞。
    Close(ctx context.Context) error
}
```

### 5.3 并发安全要求

| 场景 | 要求 |
|---|---|
| 同时多个 goroutine 调用 `BuildRequest` | 必须安全 |
| 同时多个 goroutine 调用 `ParseResponse` | 必须安全 |
| `BuildRequest` 与 `ParseResponse` 并发 | 必须安全 |
| `Close` 与 `BuildRequest` 并发 | Close 等待进行中请求完成，或返回错误（框架负责不再派发新请求） |

实现方**不得**在 Provider 结构体上保存请求级别的可变状态。

<!-- @end-section -->

<!-- @section: config-schema -->
## 6. 配置 Schema

### 6.1 通用配置（所有提供商共享）

```go
// ProviderConfig 是注册提供商时的配置，来自数据库或配置文件。
type ProviderConfig struct {
    // 必填
    ID       string `validate:"required,kebab-case"`
    Type     string `validate:"required,oneof=cloud private local"`
    Endpoint string `validate:"required,url"`

    // 认证（至少提供一种）
    Credentials ProviderCredentials `validate:"required"`

    // 连接配置（有默认值）
    Timeout        time.Duration `default:"30s"  validate:"min=1s,max=300s"`
    MaxConnections int           `default:"20"   validate:"min=1,max=500"`
    MaxIdleConns   int           `default:"10"   validate:"min=0"`
    KeepAlive      time.Duration `default:"90s"`

    // 重试（由框架 FailoverManager 使用，Provider 不直接消费）
    MaxRetries     int           `default:"2"    validate:"min=0,max=5"`
    RetryDelay     time.Duration `default:"500ms"`

    // 限流保护（框架使用，防止超过提供商 RPM 限制）
    RPMLimit       int           `default:"0"    validate:"min=0"` // 0 = 不限
    ConcurrencyMax int           `default:"10"   validate:"min=1,max=200"`

    // 提供商特有扩展（由各提供商自行定义和解析）
    Extra map[string]any
}

// ProviderCredentials 支持多种认证方式（互斥，有且仅有一种非零）。
type ProviderCredentials struct {
    APIKey       string // Bearer Token / x-api-key
    OAuthToken   string // 动态 OAuth Token（sub2api 订阅账号场景）
    AccessKey    string //  Bedrock 等云服务
    SecretKey    string
    SessionToken string
}
```

### 6.2 提供商特有配置示例（Anthropic）

```go
// AnthropicExtra 是 ProviderConfig.Extra 对于 Anthropic 提供商的结构。
// 通过 mapstructure.Decode(cfg.Extra, &extra) 反序列化。
type AnthropicExtra struct {
    APIVersion   string `default:"2023-06-01"` // anthropic-version 请求头
    BetaFeatures []string                       // anthropic-beta 请求头，如 ["interleaved-thinking-2025-05-14"]
    DefaultModel string `default:"claude-sonnet-4-6"`
}
```

<!-- @end-section -->

<!-- @section: behavior-contracts -->
## 7. 行为契约

实现 `ModelProvider` 时**必须遵守**以下契约，违反会导致框架计费错误或重试逻辑失效。

### 7.1 BuildRequest

| 契约 | 说明 |
|---|---|
| 不发起网络请求 | BuildRequest 是纯内存转换，框架负责发送 |
| 不修改 ctx 内容 | ctx 是只读的，修改会影响并发请求 |
| Credentials 不出现在 Body | 认证信息只能写入 Headers |
| 流式请求必须在 Headers 设置 `Accept: text/event-stream` | 框架据此判断是否走流式代理 |
| 返回的 URL 必须是完整 URL | 包含 scheme 和 host，框架不拼接 BaseURL |

### 7.2 ParseResponse

| 契约 | 说明 |
|---|---|
| 必须填充 Usage.InputTokens 和 Usage.OutputTokens | 即使提供商未返回，也需估算（如按内容长度估算），不得返回 0（影响计费） |
| 提供商不支持的 TokenUsage 字段填 0 | 不使用 -1 或其他哨兵值 |
| StopReason 不得为空字符串 | 至少返回 StopReasonStop |
| ToolCalls 为空时返回 nil 切片而非空切片 | 统一约定，方便调用方判空 |

### 7.3 ParseStreamChunk

| 契约 | 说明 |
|---|---|
| `[DONE]` 行必须返回 StreamDone | 框架据此结束流读取 |
| 注释行（以 `:` 开头）返回 (nil, nil) | 框架跳过此行 |
| StreamDone 事件必须携带完整的 Usage | 不得在 StreamDone 时返回 nil Usage |
| 单个 chunk 解析失败返回 StreamError | 不得 panic 或返回非 ProviderError 的 error |

### 7.4 ClassifyHTTPError

| 契约 | 说明 |
|---|---|
| HTTP 401/403 必须返回 ErrClassAuthFailure | 框架会触发告警和禁用 |
| HTTP 429 必须返回 ErrClassRateLimit | 框架会切换提供商 |
| 解析 body 失败时回退到基于 statusCode 的分类 | 不得返回错误，只能"尽力而为" |
| 不读取 body 以外的数据 | 此方法是同步纯函数 |

<!-- @end-section -->

<!-- @section: testing -->
## 8. 测试支持

### 8.1 Mock 接口

```go
// MockModelProvider 用于框架单元测试和提供商集成测试，位于 provider/mock 包。
type MockModelProvider struct {
    // 控制各方法的返回值
    BuildRequestFn      func(ctx *RequestContext) (*UpstreamRequest, error)
    ParseResponseFn     func(resp *UpstreamResponse) (*ModelOutput, error)
    ParseStreamChunkFn  func(chunk []byte) (*StreamEvent, error)
    ClassifyHTTPErrorFn func(statusCode int, body []byte) ErrorClass

    // 记录调用历史（测试断言用）
    BuildRequestCalls  []*RequestContext
    ParseResponseCalls []*UpstreamResponse
}

func (m *MockModelProvider) ID() string { return "mock" }

func (m *MockModelProvider) BuildRequest(ctx *RequestContext) (*UpstreamRequest, error) {
    m.BuildRequestCalls = append(m.BuildRequestCalls, ctx)
    if m.BuildRequestFn != nil {
        return m.BuildRequestFn(ctx)
    }
    // 默认行为：返回一个合法的最小请求
    return &UpstreamRequest{
        Method: "POST",
        URL:    "https://mock.provider/v1/chat",
        Headers: map[string]string{"Content-Type": "application/json"},
        Body:   []byte(`{"model":"mock","messages":[]}`),
    }, nil
}
```

### 8.2 契约测试套件

每个提供商实现都应运行以下通用测试套件（`provider/testing` 包提供）：

```go
// RunProviderContractTests 验证 provider 实现是否符合第 7 节定义的行为契约。
// 在各提供商的 xxx_test.go 中调用：
//
//   func TestAnthropicProviderContract(t *testing.T) {
//       provider := anthropic.NewProvider(testConfig())
//       testing.RunProviderContractTests(t, provider)
//   }
func RunProviderContractTests(t *testing.T, p ModelProvider) {
    t.Run("BuildRequest/DoesNotModifyInput", ...)
    t.Run("BuildRequest/CredentialsOnlyInHeaders", ...)
    t.Run("ParseResponse/UsageNeverZero", ...)
    t.Run("ParseResponse/ToolCallsNilNotEmpty", ...)
    t.Run("ParseStreamChunk/DoneCarriesUsage", ...)
    t.Run("ClassifyHTTPError/401IsAuthFailure", ...)
    t.Run("ClassifyHTTPError/429IsRateLimit", ...)
    t.Run("ConcurrencySafety/ParallelBuildRequest", ...)
}
```

### 8.3 录制回放测试（Record/Replay）

```go
// RecordingTransport 在集成测试中录制真实 HTTP 交互，保存为 testdata/fixtures/*.json。
// 回放时不发起真实请求，用于 CI 环境。
//
// 使用方式（在 provider 的集成测试中）：
//
//   transport := provider.NewRecordingTransport("testdata/fixtures/chat_stream.json")
//   p := anthropic.NewProvider(testConfigWithTransport(transport))
//   output, err := p.ParseResponse(transport.LastResponse())
type RecordingTransport interface {
    http.RoundTripper
    // Replay 切换到回放模式（CI 中使用）
    Replay()
}
```

<!-- @end-section -->

<!-- @section: implementation-checklist -->
## 9. 实现检查清单

新增一个提供商时，按顺序完成以下检查项：

```
核心实现
  ☐ 实现 ModelProvider 接口的全部 Required 方法
  ☐ BuildRequest：将 ModelInput → 提供商原生格式（含认证头）
  ☐ ParseResponse：将提供商响应 → ModelOutput（Usage 必须非零）
  ☐ ParseStreamChunk：SSE chunk 解析（含 DONE 和 ERROR 处理）
  ☐ ClassifyHTTPError：覆盖提供商全部已知错误码

配置
  ☐ 定义 Extra 结构体（含默认值标签）
  ☐ 在 ProviderFactory.Build 中注册新 ProviderType

测试
  ☐ 运行 RunProviderContractTests（全部通过）
  ☐ 录制真实 HTTP 交互（chat/stream/error 场景各一组）
  ☐ 编写 ClassifyHTTPError 的单元测试（覆盖 401/429/500/其他）

可选能力（按需）
  ☐ 实现 ProviderWithHealthCheck（推荐）
  ☐ 实现 ProviderWithEmbedding（如提供商支持）
  ☐ 实现 ProviderCloser（如持有连接池）

注册
  ☐ 在数据库 provider_configs 插入配置行
  ☐ 在 model_capabilities 注册支持的模型列表
```

<!-- @end-section -->

<!-- @section: related -->
## 相关文档

- [[llm-infrastructure-abstraction|LLM 底层基础设施抽象化设计]]（总览）
- `spec-billing-session-002`（计费引擎组件规范，待写）
- `spec-failover-manager-003`（故障转移组件规范，待写）
- [[03-channel-adapter-system|渠道适配器系统分析（maas 参考实现）]]

<!-- @end-section -->
