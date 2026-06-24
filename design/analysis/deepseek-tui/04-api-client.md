---
id: "analysis-deepseek-tui-api-004"
title: "DeepSeek-TUI API 客户端与流式处理"
aliases: ["deepseek-tui api", "DeepSeek-TUI流式处理"]
type: "analysis"
category: "design/analysis/deepseek-tui"
tags: ["deepseek-tui", "api-client", "sse", "streaming", "rust", "retry"]
version: "1.0.0"
created: "2026-05-07"
updated: "2026-05-07"
author: "jxncyjq"
status: "review"
parent: "analysis-deepseek-tui-overview-001"
related_docs:
  - id: "analysis-deepseek-tui-overview-001"
    relation: "parent"
    path: "./01-overview.md"
  - id: "analysis-deepseek-tui-crates-002"
    relation: "related_to"
    path: "./02-crate-analysis.md"
  - id: "analysis-deepseek-tui-storage-006"
    relation: "related_to"
    path: "./06-session-storage.md"
---

<!-- @section: overview -->

# DeepSeek-TUI API 客户端与流式处理

## 概述

`crates/tui/src/client.rs`（约 84K 行）是 DeepSeek-TUI 的 LLM 通信核心，负责与 7 家 AI 提供商进行 HTTP 通信，处理流式 SSE 响应，管理速率限制、重试策略和连接健康。

**关键特性**：
- OpenAI 兼容 API（Chat Completions 接口）
- HTTP/2 + rustls（加密传输）
- SSE（Server-Sent Events）流式处理
- DeepSeek V4 思考模式（reasoning_content）
- 7 个提供商的统一抽象
- 令牌桶速率限制 + 指数退避重试
- 连接健康监测与自动恢复

<!-- @end-section -->

<!-- @section: client-init -->

## 客户端初始化

```rust
impl DeepSeekClient {
    pub fn new(config: &Config) -> Result<Self> {
        let http_client = Self::build_http_client(&config.api_key)?;
        Ok(Self {
            http_client,
            base_url: config.base_url.clone(),
            model: config.default_text_model.clone(),
            rate_limiter: TokenBucket::new(8.0, 16.0), // 8 RPS，突发 16
            health: Arc::new(Mutex::new(ConnectionHealth::default())),
        })
    }
    
    fn build_http_client(api_key: &str) -> Result<reqwest::Client> {
        let mut headers = HeaderMap::new();
        headers.insert(
            AUTHORIZATION,
            HeaderValue::from_str(&format!("Bearer {api_key}"))?
        );
        reqwest::Client::builder()
            .default_headers(headers)
            .connect_timeout(Duration::from_secs(30))
            .http2_keep_alive_interval(Some(Duration::from_secs(15)))
            .http2_keep_alive_timeout(Duration::from_secs(20))
            .build()
    }
}
```

**环境变量覆盖**：
```bash
DEEPSEEK_FORCE_HTTP1=1      # 强制使用 HTTP/1.1（某些代理不支持 HTTP/2）
SSL_CERT_FILE=/path/to/cert # 自定义 SSL 证书（企业网络代理）
DEEPSEEK_RATE_LIMIT_RPS=8.0 # 速率限制（每秒请求数）
DEEPSEEK_RATE_LIMIT_BURST=16.0 # 突发容量
```

<!-- @end-section -->

<!-- @section: providers -->

## 多提供商支持

| 提供商 | Endpoint | 代表模型 | 特殊说明 |
|--------|----------|---------|---------|
| DeepSeek（默认） | `https://api.deepseek.com` | `deepseek-v4-pro`, `deepseek-v4-flash` | 原生思考模式支持 |
| DeepSeek CN | `https://api.deepseek.com/cn` | 同上 | 中国区优化 |
| NVIDIA NIM | `https://integrate.api.nvidia.com/v1` | `deepseek-ai/deepseek-v4-pro` | NVIDIA 托管 |
| OpenAI | `https://api.openai.com/v1` | `gpt-4.1`, `gpt-4.1-mini` | 标准 OpenAI 兼容 |
| OpenRouter | `https://openrouter.ai/api/v1` | `deepseek/deepseek-v4-pro` | 多模型聚合中继 |
| Novita | `https://api.novita.ai/v1` | `deepseek/deepseek-v4-pro` | 经济型中继 |
| Fireworks | `https://api.fireworks.ai/inference/v1` | `accounts/fireworks/models/deepseek-v4-pro` | 高性能推理 |
| SGLang | `http://localhost:30000/v1` | `deepseek-ai/DeepSeek-V4-Pro` | 本地自托管 |

**模型上下文窗口**：
```rust
pub fn context_window_for_model(model: &str) -> Option<u32> {
    match model.to_lowercase() {
        m if m.contains("v4") => Some(1_000_000),   // DeepSeek V4：1M tokens
        m if m.contains("v3") => Some(128_000),      // DeepSeek V3：128K tokens
        m if m.contains("claude") => Some(200_000),  // Claude：200K tokens
        m if m.contains("gpt-4") => Some(128_000),   // GPT-4：128K tokens
        _ => None,
    }
}
```

<!-- @end-section -->

<!-- @section: request-response -->

## 请求/响应数据结构

### MessageRequest（请求体）

```rust
pub struct MessageRequest {
    pub model: String,
    pub messages: Vec<Message>,         // 完整对话历史
    pub max_tokens: u32,                // 最大输出 tokens
    pub system: Option<SystemPrompt>,   // 系统提示词
    pub tools: Option<Vec<Tool>>,       // 工具定义列表
    pub tool_choice: Option<Value>,     // auto | required | none
    pub thinking: Option<Value>,        // 思考模式配置
    pub reasoning_effort: Option<String>, // off|low|medium|high|max
    pub temperature: Option<f32>,
    pub top_p: Option<f32>,
    pub stream: Option<bool>,           // 是否使用 SSE
    pub stream_options: Option<Value>,  // { "include_usage": true }
}
```

**HTTP 请求示例**：
```
POST /v1/chat/completions
Authorization: Bearer sk-xxxxxxxx
Content-Type: application/json

{
  "model": "deepseek-v4-pro",
  "messages": [
    { "role": "system", "content": "You are helpful" },
    { "role": "user", "content": "Hello" }
  ],
  "max_tokens": 8192,
  "reasoning_effort": "high",
  "tools": [...],
  "stream": true,
  "stream_options": { "include_usage": true }
}
```

### MessageResponse（响应体）

```rust
pub struct MessageResponse {
    pub id: String,
    pub r#type: String,              // "message"
    pub role: String,                // "assistant"
    pub content: Vec<ContentBlock>,  // 响应内容块
    pub model: String,
    pub stop_reason: Option<String>, // end_turn | tool_use | max_tokens
    pub usage: Usage,
}

pub struct Usage {
    pub input_tokens: u32,
    pub output_tokens: u32,
    pub prompt_cache_hit_tokens: Option<u32>,   // 前缀缓存命中
    pub prompt_cache_miss_tokens: Option<u32>,  // 前缀缓存未命中
    pub reasoning_tokens: Option<u32>,          // 思考消耗 tokens
    pub reasoning_replay_tokens: Option<u32>,   // 思考重放 tokens
}
```

<!-- @end-section -->

<!-- @section: streaming -->

## SSE 流式处理管道

### 总体流程

```
HTTP 响应字节流
  ↓
[字节缓冲区管理]
  ├── 缓冲区大小：256KB ~ 10MB（动态）
  ├── 高水位警戒：8MB（触发背压控制）
  └── 批处理：每批最多 256 行

  ↓
[SSE 行解析]
  ├── 扫描 "\r\n" 或 "\n" 行分隔符
  ├── 过滤 "data: " 前缀
  ├── 检测 "[DONE]" 流结束标记
  └── JSON 反序列化

  ↓
parse_sse_chunk()
  ├── 提取 choices[0].delta
  ├── 文本增量 → ContentBlockDelta { TextDelta }
  ├── 思考增量 → ContentBlockDelta { ThinkingDelta }
  ├── 工具调用 → ContentBlockStart { ToolUse } / InputJsonDelta
  └── finish_reason → ContentBlockStop + MessageDelta

  ↓
StreamEvent 序列 → UI 增量更新
```

### StreamEvent 枚举

```rust
pub enum StreamEvent {
    MessageStart {
        message: MessageResponse,              // 合成的消息头
    },
    ContentBlockStart {
        index: u32,
        content_block: ContentBlockStart,      // Text | Thinking | ToolUse
    },
    ContentBlockDelta {
        index: u32,
        delta: Delta,                          // TextDelta | ThinkingDelta | InputJsonDelta
    },
    ContentBlockStop {
        index: u32,
    },
    MessageDelta {
        delta: MessageDelta,
        usage: Option<Usage>,                  // 最终 token 统计
    },
    MessageStop,
}
```

### parse_sse_chunk() 状态跟踪

```rust
pub fn parse_sse_chunk(
    chunk: &Value,
    content_index: &mut u32,           // 当前块索引（单调递增）
    text_started: &mut bool,            // 文本块已开始
    thinking_started: &mut bool,        // 思考块已开始
    tool_indices: &mut HashMap<u32, u32>, // tool_calls 索引 → content_block 索引
    is_reasoning_model: bool,           // 是否为 V4/R1 推理模型
) -> Vec<StreamEvent>
```

**状态转移规则**：
- 首个 `delta.content` → 发出 `ContentBlockStart { Text }` + 设置 `text_started = true`
- `delta.reasoning` → 发出 `ContentBlockStart { Thinking }` + 设置 `thinking_started = true`
- 新 tool_call `index` → 发出 `ContentBlockStart { ToolUse }`，注册 `tool_indices[n]`
- 已知 tool_call `index` → 发出 `InputJsonDelta`（追加 arguments）
- `finish_reason` 存在 → 发出所有待关闭块的 `ContentBlockStop` + `MessageDelta`

### 流式超时与恢复

```rust
const SSE_IDLE_TIMEOUT: Duration = Duration::from_secs(300);      // 5 分钟空闲超时
const SSE_BACKPRESSURE_HIGH_WATERMARK: usize = 8 * 1024 * 1024;   // 8MB 背压
const SSE_MAX_LINES_PER_BATCH: usize = 256;                        // 批处理行数限制
const SSE_TRANSPARENT_RETRY_MAX: u32 = 3;                          // 流停滞透明重试
```

当检测到流停滞（`stream stall`）时，客户端自动透明重试（最多 3 次），无需用户感知。

<!-- @end-section -->

<!-- @section: thinking-mode -->

## DeepSeek V4 思考模式

### 推理努力级别

```rust
pub enum ReasoningEffort {
    Off,     // 禁用思考：thinking.type = "disabled"
    Low,     // 最小思考
    Medium,  // 适度思考
    High,    // 扩展思考（默认）
    Max,     // 最大深度思考
}

// API 映射
pub fn apply_reasoning_effort(body: &mut Value, effort: &str, provider: ApiProvider) {
    match effort {
        "off" => {
            body["thinking"] = json!({ "type": "disabled" });
        },
        "high" | "max" => {
            body["reasoning_effort"] = json!("high");
            body["thinking"] = json!({ "type": "enabled" });
        },
        _ => {} // 使用提供商默认值
    }
}
```

### reasoning_content 消毒

DeepSeek V4 API 要求所有 `assistant` 消息中的 `reasoning_content` 字段必须存在（即使为空），否则 API 会报错。

```rust
pub fn sanitize_thinking_mode_messages(
    messages: &mut Vec<Message>,
    model: &str,
    effort: Option<&str>,
) -> Option<u32> {
    // 遍历所有 assistant 消息
    // 如果 reasoning_content 缺失，注入占位符："(reasoning omitted)"
    // 返回估计的重放 tokens 数量（约 4 字符/token）
}
```

### 工具名称编码/解码

DeepSeek API 要求工具名称只含字母数字和短横线，但内部工具名称可能包含点（`.`）等特殊字符。

```rust
// 编码：内部名 → API 名
// "web.search"  → "web-x00002E-search"
// "file-read"   → "file--read"（短横线重复）
// 非字母数字字符 → "-xHHHHHH-"（Unicode 十六进制）
pub fn to_api_tool_name(name: &str) -> String { ... }

// 解码：API 名 → 内部名（容忍模型可能破坏的格式）
pub fn from_api_tool_name(name: &str) -> String { ... }
```

<!-- @end-section -->

<!-- @section: rate-limit-retry -->

## 速率限制与重试

### 令牌桶算法（Token Bucket）

```rust
pub struct TokenBucket {
    enabled: bool,
    capacity: f64,          // 突发容量（默认 16 tokens）
    tokens: f64,            // 当前可用 tokens
    refill_per_sec: f64,    // 补充速率（默认 8 RPS）
    last_refill: Instant,
}

impl TokenBucket {
    pub fn delay_until_available(&mut self, tokens: f64) -> Option<Duration> {
        self.refill();
        if self.tokens >= tokens {
            self.tokens -= tokens;
            None  // 立即可用
        } else {
            // 计算需要等待的时间
            let needed = tokens - self.tokens;
            Some(Duration::from_secs_f64(needed / self.refill_per_sec))
        }
    }
}
```

### 指数退避重试

```rust
pub struct RetryConfig {
    pub enabled: bool,
    pub max_retries: u32,               // 默认 3 次
    pub initial_delay: Duration,        // 默认 1 秒
    pub max_delay: Duration,            // 默认 30 秒
}

// 延迟公式：delay = min(initial * 2^attempt + jitter, max)
// attempt=0: ~1s, attempt=1: ~2s, attempt=2: ~4s, ...

// 可重试错误：
// - RateLimited (429)
// - ServerError (5xx)
// - NetworkError
// - Timeout
// 不可重试错误：
// - AuthenticationError (401)
// - InvalidInput (400)
```

### 连接健康监测

```rust
pub struct ConnectionHealth {
    state: ConnectionState,           // Healthy | Degraded | Recovering
    consecutive_failures: u32,        // 连续失败计数
    last_failure: Option<Instant>,
    last_success: Option<Instant>,
    last_probe: Option<Instant>,      // 恢复探针时间
}

// 状态转移
// 正常 → 连续 2 次失败 → Degraded
// Degraded → 每 15 秒发送恢复探针 → Recovering
// Recovering → 探针成功 → Healthy
// Recovering → 探针失败 → 继续 Degraded
```

**UI 表现**：状态栏显示连接状态（绿色 = Healthy，黄色 = Degraded/Recovering）。

<!-- @end-section -->

<!-- @section: compaction -->

## 上下文压缩（Compaction）

**文件**：`crates/tui/src/compaction.rs`（84K）

### 触发条件

```rust
pub fn should_compact(
    messages: &[Message],
    config: &CompactionConfig,
    pins: Option<&HashSet<usize>>,
) -> bool {
    let token_count = estimate_token_count(messages);
    let window = context_window_for_model(&config.model).unwrap_or(128_000);
    
    // 条件1：tokens 超过窗口的 80%
    token_count as f64 / window as f64 > 0.8
    // 条件2：消息数超过 1000
    || messages.len() > 1000
    // 条件3：手动触发（/compact 命令）
}
```

### 分级压缩阈值（配置项）

| 级别 | 阈值 | 说明 |
|------|------|------|
| L1 | 192,000 tokens | 轻度压缩 |
| L2 | 384,000 tokens | 中度压缩 |
| L3 | 576,000 tokens | 重度压缩 |
| Cycle | 768,000 tokens | 硬触发循环检查点 |

### 压缩算法

```rust
pub async fn compact_messages_safe(
    client: &DeepSeekClient,
    messages: &[Message],
    config: &CompactionConfig,
) -> Result<CompactionResult> {
    // 策略1："summarize" — 调用 LLM 生成摘要（默认关闭）
    // 策略2："checkpoint" — 归档当前循环，重置消息缓冲区
    
    // 保留：固定消息（pinned）、最近 N 轮（verbatim_window_turns）
    // 压缩：中间段消息 → 摘要文本
    // 返回：CompactionResult { messages, summary_prompt, retries_used }
}
```

<!-- @end-section -->

<!-- @section: related -->

## 相关文档

- [[01-overview|项目总览]]
- [[02-crate-analysis|Crate 职责分析]]
- [[05-tool-system|工具系统与 MCP]]
- [[06-session-storage|会话管理与持久化]]

<!-- @end-section -->
