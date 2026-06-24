---
id: "design-maas-scheduling-008"
title: "MaaS 模型调度层技术设计"
aliases: ["MaaS 技术设计", "模型调度", "api 设计"]
type: "design"
category: "design/analysis/claude"
tags: ["maas", "model-routing", "provider", "sse", "api-client", "budget"]
version: "1.0.0"
created: "2026-05-03"
updated: "2026-05-03"
author: "jxncyjq"
status: "review"
parent: null
children: ["design-agent-runtime-009"]
related_docs:
  - id: "analysis-clawcode-rust-002"
    relation: "references"
    path: "./02-rust-crates-analysis.md"
  - id: "analysis-clawcode-datamodel-006"
    relation: "references"
    path: "./06-data-models.md"
---

<!-- @section: overview -->
# MaaS 模型调度层 — 技术设计

**文档版本**：V1.0
**编写日期**：2026年5月
**文档性质**：技术设计
**参考来源**：claw-code api crate 架构分析

---

## 目录

- [一、设计依据](#一、设计依据)
- [二、系统架构](#二、系统架构)
- [三、核心模块设计](#三、核心模块设计)
  - [3.1 模型注册中心](#31-模型注册中心)
  - [3.2 智能路由引擎](#32-智能路由引擎)
  - [3.3 配额管控与熔断](#33-配额管控与熔断)
  - [3.4 认证与凭证管理](#34-认证与凭证管理)
  - [3.5 HTTP 客户端与代理](#35-http-客户端与代理)
  - [3.6 SSE 流式解析](#36-sse-流式解析)
  - [3.7 请求格式转换](#37-请求格式转换)
  - [3.8 提示缓存与飞行前检查](#38-提示缓存与飞行前检查)
- [四、数据模型](#四、数据模型)
- [五、三原则落地](#五、三原则落地)

---

## 一、设计依据

### 1.1 参考来源

本设计参考 claw-code 的 `api` crate（约 10 个模块、14 项提供商需求全部已实现），其核心设计经实际验证有效：

| 参考模块 | 核心能力 | Legion 映射 |
|----------|----------|-------------|
| `providers/anthropic.rs` | Anthropic API 客户端（Key/Bearer/重试/OAuth） | MaaS 提供商适配器 |
| `providers/openai_compat.rs` | OpenAI 兼容协议（格式转换、多后端） | 统一协议转换层 |
| `providers/mod.rs` | 模型注册表、别名解析、自动检测、飞行前检查 | 模型注册中心 + 智能路由 |
| `client.rs` | 统一客户端枚举、流式消息类型 | 统一调用入口 |
| `http_client.rs` | 代理配置（HTTP_PROXY/HTTPS_PROXY/NO_PROXY） | 网络层 |
| `error.rs` | 11 变体错误枚举、重试分类、安全失败分类 | 错误处理 |
| `sse.rs` | SSE 流解析器（分帧、过滤 ping/DONE） | 流式响应处理 |
| `prompt_cache.rs` | FNV 哈希追踪、命中/断裂统计 | 提示缓存 |
| `types.rs` | API 通信数据类型（请求/响应/工具/用量/费用） | API 类型系统 |

### 1.2 设计目标

MaaS 层是 Legion 的模型基础设施，承担三个核心职责：

1. **统一纳管** — 多厂商、多类型模型通过标准化接口接入
2. **智能调度** — 根据角色/任务/预算自动选择最优模型
3. **成本管控** — 多维配额 + 三级熔断确保不超预算

---

## 二、系统架构

```
┌─────────────────────────────────────────────────────────────┐
│                       MaaS 模型调度层                          │
│                                                             │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌────────────┐  │
│  │ 模型注册  │  │ 智能路由  │  │ 配额管控  │  │ 认证凭证    │  │
│  │ 中心     │  │ 引擎     │  │ 引擎     │  │ 管理       │  │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘  └────┬───────┘  │
│       │              │              │              │          │
│  ┌────┴──────────────┴──────────────┴──────────────┴───────┐ │
│  │                    统一调用入口                           │ │
│  │         ProviderClient (Anthropic/OpenAI/xAI/...)        │ │
│  └──────────────────────┬──────────────────────────────────┘ │
│                         │                                    │
│  ┌──────────────────────┴──────────────────────────────────┐ │
│  │                  HTTP 传输层                              │ │
│  │   代理配置 / 指数退避重试 / SSE 流解析 / 请求转换          │ │
│  └─────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
```

### 数据流

```
Agent 请求（role/task/budget_context）
  │
  ▼
智能路由引擎
  ├── 角色匹配：该角色偏好什么模型？
  ├── 任务匹配：该任务需要什么能力？
  ├── 预算检查：当前预算剩余是否允许？
  └── 降级链：首选模型不可用时用什么替代？
  │
  ▼
统一调用入口 → 提供商适配器 → HTTP 请求
  │
  ▼
SSE 流解析 → 实时用量追踪 → 结果返回 Agent
```

---

## 三、核心模块设计

### 3.1 模型注册中心

#### 3.1.1 模型注册表

参考 claw-code 的 `MODEL_REGISTRY`，维护所有可用模型的能力标签：

```rust
struct ModelRegistry {
    models: BTreeMap<String, ModelEntry>,
    aliases: AliasRegistry,
}

struct ModelEntry {
    id: String,                    // 唯一标识
    display_name: String,          // 展示名
    provider: ProviderKind,        // 提供商
    capabilities: ModelCapabilities,
    pricing: ModelPricing,
}

struct ModelCapabilities {
    reasoning_level: ReasoningLevel,  // low/medium/high
    context_window: u32,              // token 数
    max_output_tokens: u32,
    modalities: Vec<Modality>,        // Text/Image/Code/Audio
    supports_streaming: bool,
    supports_cache: bool,
    supports_tools: bool,
}

struct ModelPricing {
    input_per_million: f64,           // 美元
    output_per_million: f64,
    cache_write_per_million: f64,
    cache_read_per_million: f64,
    currency: Currency,               // USD/CNY
}
```

#### 3.1.2 提供商枚举

参考 claw-code 的 `ProviderKind`：

```rust
enum ProviderKind {
    Anthropic,     // Anthropic Messages API
    OpenAI,        // OpenAI Chat Completions API
    DeepSeek,      // 深度求索
    Qwen,          // 通义千问 (DashScope)
    Xai,           // xAI Grok
    OpenRouter,    // 聚合代理
    Ollama,        // 本地推理
    Custom(String), // 自定义端点
}
```

#### 3.1.3 模型别名系统

参考 claw-code 的别名解析，支持从别名自动推断提供商：

```rust
struct AliasRegistry {
    builtin: BTreeMap<String, String>,   // opus → claude-opus-4
    user_defined: BTreeMap<String, String>, // my-dev → deepseek-coder
}

impl AliasRegistry {
    fn resolve(&self, alias: &str) -> Option<&str> {
        self.user_defined.get(alias)
            .or_else(|| self.builtin.get(alias))
            .or(Some(alias)) // 如果没有匹配，原样返回
    }
}
```

#### 3.1.4 提供商自动检测

参考 claw-code 的 `detect_provider_kind()`，根据模型名前缀自动确定提供商：

| 前缀 | 提供商 |
|------|--------|
| `claude-` | Anthropic |
| `gpt-` / `o1-` / `o3-` | OpenAI |
| `deepseek-` | DeepSeek |
| `qwen-` / `dashscope-` | Qwen (DashScope) |
| `grok-` | xAI |
| `openrouter/` | OpenRouter |

---

### 3.2 智能路由引擎

#### 3.2.1 路由策略类型

```rust
enum RoutingStrategy {
    /// 按角色：CEO → opus, Developer → sonnet
    RoleBased {
        rules: BTreeMap<AgentRole, ModelPreference>,
    },
    /// 按任务：战略规划 → 强推理, 日常编码 → 性价比
    TaskBased {
        rules: BTreeMap<TaskCategory, ModelPreference>,
    },
    /// 按预算约束：剩余 20% 降级, 剩余 5% 最低档
    BudgetAware {
        thresholds: Vec<BudgetThreshold>,
    },
}

struct ModelPreference {
    primary: String,           // 首选模型
    fallbacks: Vec<String>,    // 降级链（按优先级排列）
    min_reasoning: ReasoningLevel,
    prefer_low_cost: bool,
}
```

#### 3.2.2 路由决策流程

```rust
impl RoutingEngine {
    async fn route(&self, context: &RoutingContext) -> Result<ModelSelection> {
        // 1. 强制覆盖：上级手动指定的模型
        if let Some(override_model) = &context.force_model {
            return self.select_model(override_model);
        }

        // 2. 角色策略：该 Agent 角色的偏好模型
        if let Some(preference) = self.role_rules.get(&context.agent_role) {
            return self.try_with_fallback(preference, context.budget_remaining);
        }

        // 3. 任务策略：该任务类型需要的模型能力
        if let Some(preference) = self.task_rules.get(&context.task_category) {
            return self.try_with_fallback(preference, context.budget_remaining);
        }

        // 4. 预算约束降级
        for threshold in &self.budget_thresholds {
            if context.budget_remaining < threshold.ratio {
                return self.select_model(&threshold.force_model);
            }
        }

        // 5. 默认模型
        self.select_model(&self.default_model)
    }

    fn try_with_fallback(&self, pref: &ModelPreference, budget: f64) -> Result<ModelSelection> {
        // 如果预算紧张且 prefer_low_cost，跳过高级模型
        let candidates = if pref.prefer_low_cost || budget < 0.2 {
            pref.fallbacks.iter()
        } else {
            std::iter::once(&pref.primary).chain(pref.fallbacks.iter())
        };

        for model_id in candidates {
            if let Ok(selection) = self.check_availability(model_id) {
                return Ok(selection);
            }
        }

        Err(RoutingError::NoAvailableModel)
    }
}
```

#### 3.2.3 路由上下文

```rust
struct RoutingContext {
    agent_id: String,
    agent_role: AgentRole,
    task_category: TaskCategory,
    budget_remaining: f64,       // 该 Agent 当前预算使用比例 (0.0~1.0)
    force_model: Option<String>, // 手动覆盖
    min_capabilities: ModelCapabilities, // 最低能力要求
}
```

---

### 3.3 配额管控与熔断

#### 3.3.1 多维预算模型

```rust
struct BudgetManager {
    company_budget: CompanyBudget,
    department_budgets: BTreeMap<DepartmentId, DepartmentBudget>,
    agent_budgets: BTreeMap<AgentId, AgentBudget>,
}

struct CompanyBudget {
    monthly_limit_cny: f64,
    consumed_cny: f64,
    alert_thresholds: Vec<f64>,  // [0.8, 0.95, 1.0]
}

struct AgentBudget {
    monthly_token_limit: u64,
    consumed_tokens: u64,
    monthly_cost_limit_cny: f64,
    consumed_cost_cny: f64,
    per_task_cost_limit_cny: f64,
    model_tier_limit: ReasoningLevel,
}
```

#### 3.3.2 三级熔断机制

```rust
enum FuseState {
    Normal,        // 正常
    Warning(f64),  // 告警（使用率）
    Degraded,      // 降级（强制使用低档模型）
    Frozen,        // 冻结（禁止调用）
}

impl BudgetManager {
    fn check(&self, agent_id: &AgentId) -> FuseState {
        let budget = &self.agent_budgets[agent_id];
        let ratio = budget.consumed_cost_cny / budget.monthly_cost_limit_cny;

        if ratio >= 1.0 {
            FuseState::Frozen                     // 超预算：冻结
        } else if ratio >= 0.95 {
            FuseState::Degraded                   // 95%：降级
        } else if ratio >= 0.80 {
            FuseState::Warning(ratio)             // 80%：告警
        } else {
            FuseState::Normal
        }
    }
}
```

#### 3.3.3 用量追踪与费用估算

参考 claw-code 的 `usage.rs`（四维计数 + 模型定价表）：

```rust
struct TokenUsage {
    input_tokens: u64,
    output_tokens: u64,
    cache_creation_input_tokens: u64,
    cache_read_input_tokens: u64,
}

impl TokenUsage {
    fn estimate_cost(&self, pricing: &ModelPricing) -> CostEstimate {
        CostEstimate {
            input_cost: self.input_tokens as f64 / 1_000_000.0 * pricing.input_per_million,
            output_cost: self.output_tokens as f64 / 1_000_000.0 * pricing.output_per_million,
            cache_create_cost: self.cache_creation_input_tokens as f64 / 1_000_000.0 * pricing.cache_write_per_million,
            cache_read_cost: self.cache_read_input_tokens as f64 / 1_000_000.0 * pricing.cache_read_per_million,
        }
    }
}
```

---

### 3.4 认证与凭证管理

参考 claw-code 的 `providers/anthropic.rs` 双认证模式：

#### 3.4.1 认证方式

```rust
enum AuthMethod {
    /// x-api-key: 直接 API Key（Anthropic 标准方式）
    ApiKey {
        header_name: String,       // "x-api-key"
        env_var: String,           // "ANTHROPIC_API_KEY"
    },
    /// Authorization: Bearer token（OpenAI 兼容方式）
    BearerToken {
        env_var: String,           // "OPENAI_API_KEY"
    },
    /// OAuth 2.0 + PKCE
    OAuth {
        config: OAuthConfig,
    },
}
```

#### 3.4.2 OAuth 2.0 + PKCE 流程

参考 claw-code 的 `oauth.rs`：

```
1. 生成 PKCE code_verifier + code_challenge (S256)
2. 打开浏览器跳转授权页面
3. 本地 HTTP 服务器接收回调 (loopback redirect_uri)
4. 用 code + code_verifier 换取 access_token + refresh_token
5. 令牌持久化存储
6. 自动刷新：过期前 5 分钟或收到 401 时触发 token refresh
```

#### 3.4.3 凭证错误检测

参考 claw-code 的 `ApiError` 的凭证检测逻辑：

```rust
impl AuthManager {
    fn detect_credential_misplacement(&self, error: &ApiError) -> Option<String> {
        // 检测 sk-ant- 开头的 key 被误放入 Bearer 槽
        if error.is_auth_error() {
            if let Some(key) = &self.bearer_token {
                if key.starts_with("sk-ant-") {
                    return Some("检测到 Anthropic API Key 被配置为 Bearer Token，请改用 x-api-key header".into());
                }
            }
        }
        None
    }
}
```

---

### 3.5 HTTP 客户端与代理

参考 claw-code 的 `http_client.rs`：

#### 3.5.1 代理配置

```rust
struct ProxyConfig {
    http_proxy: Option<String>,    // HTTP_PROXY 环境变量
    https_proxy: Option<String>,   // HTTPS_PROXY 环境变量
    no_proxy: Vec<String>,         // NO_PROXY 默认列表
    proxy_url: Option<String>,     // 统一代理 URL（配置文件）
}
```

#### 3.5.2 指数退避重试

参考 claw-code 的 `providers/anthropic.rs`，最多 8 次重试：

```rust
struct RetryPolicy {
    max_retries: u32,            // 8
    initial_backoff_ms: u64,     // 1000
    backoff_multiplier: f64,     // 2.0
    max_backoff_ms: u64,         // 60000
    retryable_errors: Vec<RetryableErrorClass>,
}

enum RetryableErrorClass {
    RateLimit,       // 429
    ServerError,     // 5xx
    NetworkError,    // 连接超时/中断
    Overloaded,      // 529 Anthropic 过载
}
```

---

### 3.6 SSE 流式解析

参考 claw-code 的 `sse.rs`（分段缓冲 + 按 `\n\n` 分帧）：

```rust
struct SseParser {
    buffer: Vec<u8>,
}

impl SseParser {
    fn push(&mut self, chunk: &[u8]) -> Vec<StreamFrame> {
        self.buffer.extend_from_slice(chunk);
        let mut frames = Vec::new();

        while let Some(pos) = self.find_frame_boundary() {
            let frame = self.buffer.drain(..pos).collect::<Vec<_>>();
            if let Some(parsed) = self.parse_frame(&frame) {
                frames.push(parsed);
            }
        }

        frames
    }

    fn parse_frame(&self, frame: &[u8]) -> Option<StreamFrame> {
        // 跳过 ping 帧
        if frame.starts_with(b": ping") { return None; }
        // 跳过 [DONE] 帧
        if frame == b"data: [DONE]\n" { return None; }
        // 解析 data: {...} 行
        // ...
    }
}
```

---

### 3.7 请求格式转换

参考 claw-code 的 `providers/openai_compat.rs`，实现 Anthropic ↔ OpenAI 格式互转：

```rust
struct FormatTranslator;

impl FormatTranslator {
    /// Anthropic Messages → OpenAI Chat Completions
    fn to_openai(request: &AnthropicRequest) -> OpenAiRequest {
        OpenAiRequest {
            model: request.model.clone(),
            messages: request.messages.iter().map(convert_message).collect(),
            stream: request.stream,
            max_tokens: request.max_tokens,
            tools: request.tools.as_ref().map(convert_tools),
            // ...
        }
    }

    /// OpenAI Chat Completions → Anthropic Messages
    fn to_anthropic(request: &OpenAiRequest) -> AnthropicRequest {
        // 反向转换...
    }
}
```

#### 3.7.1 推理模型参数清理

参考 claw-code 对于推理类模型（如 o1/grok）自动清理不兼容参数：

```rust
fn sanitize_for_reasoning_model(request: &mut RequestParams) {
    if request.model.is_reasoning_model() {
        request.temperature = None;  // 推理模型不接受 temperature
        request.top_p = None;
        request.stop = None;
    }
}
```

---

### 3.8 提示缓存与飞行前检查

#### 3.8.1 提示缓存

参考 claw-code 的 `prompt_cache.rs`：

```rust
struct PromptCache {
    fingerprint: u64,      // FNV 哈希
    stats: CacheStats,
}

struct CacheStats {
    hits: u64,
    misses: u64,
    breaks: u64,           // 缓存断裂次数
    tokens_saved: u64,
}

impl PromptCache {
    fn compute_fingerprint(&self, content: &str) -> u64 {
        // FNV-1a 哈希
    }

    fn detect_break(&self, previous_read: u64, current_read: u64) -> bool {
        // token_drop 超过阈值 → 缓存断裂
    }
}
```

#### 3.8.2 飞行前 Token 估算

参考 claw-code 的 `preflight_message_request()`：

```rust
fn preflight_check(request: &ApiRequest, model: &ModelEntry) -> Result<()> {
    let estimated_tokens = estimate_tokens(&request.messages);
    if estimated_tokens > model.capabilities.context_window as u64 {
        return Err(ApiError::ContextWindowExceeded {
            estimated: estimated_tokens,
            limit: model.capabilities.context_window,
        });
    }
    Ok(())
}
```

---

## 四、数据模型

### 4.1 API 请求/响应

参考 claw-code 的 `types.rs`：

```rust
struct ApiRequest {
    model: String,
    max_tokens: u32,
    messages: Vec<InputMessage>,
    system: Option<String>,
    tools: Option<Vec<ToolDefinition>>,
    tool_choice: Option<ToolChoice>,
    stream: bool,
    temperature: Option<f64>,
    top_p: Option<f64>,
    stop: Option<Vec<String>>,
}

struct ApiResponse {
    id: String,
    model: String,
    content: Vec<OutputContentBlock>,
    usage: TokenUsage,
    stop_reason: StopReason,
}

enum StopReason {
    EndTurn,
    MaxTokens,
    StopSequence,
    ToolUse,
}

enum OutputContentBlock {
    Text { text: String },
    ToolUse { id: String, name: String, input: serde_json::Value },
    Thinking { thinking: String },
}
```

### 4.2 流式事件

```rust
enum StreamEvent {
    MessageStart { message: ApiResponse },
    ContentBlockStart { index: u32, block: OutputContentBlock },
    ContentBlockDelta { index: u32, delta: ContentDelta },
    ContentBlockStop { index: u32 },
    MessageDelta { stop_reason: StopReason, usage: TokenUsage },
    MessageStop,
}

enum ContentDelta {
    TextDelta { text: String },
    InputJsonDelta { partial_json: String },
    ThinkingDelta { thinking: String },
}
```

### 4.3 错误体系

参考 claw-code 的 `error.rs`：

```rust
enum MaaSError {
    MissingCredentials { provider: String, hints: Vec<String> },
    ContextWindowExceeded { estimated: u64, limit: u32 },
    ExpiredOAuthToken { provider: String, can_refresh: bool },
    Auth { provider: String, message: String },
    RateLimited { provider: String, retry_after_secs: u64 },
    ServerError { provider: String, status: u16, message: String },
    NetworkError { message: String },
    RetriesExhausted { attempts: u32, last_error: Box<MaaSError> },
    InvalidSseFrame { raw: String },
    RequestBodyTooLarge { size_bytes: u64, limit_bytes: u64 },
}

impl MaaSError {
    fn is_retryable(&self) -> bool {
        matches!(self, MaaSError::RateLimited { .. }
                     | MaaSError::ServerError { .. }
                     | MaaSError::NetworkError { .. })
    }
}
```

---

## 五、三原则落地

### 5.1 可观测

| 观测点 | 数据 | 用途 |
|--------|------|------|
| 调用链路追踪 | `{agent_id, task_id, routed_model, actual_model, tokens}` | 每次调用的完整路径 |
| 模型选择日志 | `{context, candidates, selected, reason}` | 为什么选了 A 而不是 B |
| Token 消耗明细 | 每次调用的四维计数 + 费用 | 按 Agent/部门/公司聚合 |
| 错误分布 | `{provider, error_type, frequency}` | 提供商质量监控 |
| 缓存效率 | `{hits, misses, breaks, tokens_saved}` | 缓存效果评估 |

### 5.2 可治理

| 治理能力 | 实现方式 |
|----------|----------|
| 手动绑定/禁用模型 | `Agent.model_policy.force_model` / `disallowed_models` |
| 实时修改路由策略 | 路由引擎配置热更新 |
| 模型升级审批 | Agent 请求使用更高级别模型 → 工作流审批门控 |
| 预算重分配 | 管理员按部门/Agent 调整预算限额 |

### 5.3 可控风险

| 风险 | 控制手段 |
|------|----------|
| 预算超支 | 80% 告警 → 95% 降级 → 100% 冻结 |
| 模型不可用 | 降级链自动切换到备用模型 |
| 单次调用失控 | `per_task_cost_limit_cny` 硬限制 |
| 过度使用高级模型 | `model_tier_limit` 限制单 Agent 可用模型档位 |
| 认证泄露 | 凭证检测 + 环境变量提示 |
<!-- @end-section -->

<!-- @section: related -->
## 相关文档

- [[02-rust-crates-analysis|Rust Crate 功能模块分析]] — API 层参考来源
- [[06-data-models|数据模型分析]] — 类型系统参考
- [[design-agent-runtime-009|Agent 运行时引擎技术设计]] — 下游依赖模块
<!-- @end-section -->
