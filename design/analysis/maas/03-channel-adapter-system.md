---
id: "analysis-newapi-channel-003"
title: "渠道适配器系统分析"
aliases: ["new-api channel adapter", "渠道适配器", "channel adaptor system"]
type: "analysis"
category: "design/analysis/maas"
tags: ["new-api", "channel-adapter", "relay", "provider", "maas"]
version: "1.0.0"
created: "2026-05-03"
updated: "2026-05-03"
author: "jxncyjq"
status: "review"
parent: "analysis-newapi-overview-001"
related_docs:
  - id: "analysis-newapi-overview-001"
    relation: "parent"
    path: "./01-overview.md"
  - id: "analysis-newapi-modules-002"
    relation: "related_to"
    path: "./02-module-analysis.md"
---

<!-- @section: overview -->
# 渠道适配器系统分析

## 系统概述

New API 的渠道适配器系统是整个网关的**核心引擎**，通过统一的 `Adaptor` 接口，将众多上游 AI 提供商的各异构 API 封装为一致的内部抽象。客户端始终使用 OpenAI 兼容格式请求，适配器负责格式转换和协议适配。

关键数据：**37 个渠道适配器目录**（`relay/channel/<name>/`），对应 `constant/channel.go` 中 48 个 `ChannelType*` 常量——比例不是 1:1：所有 OpenAI 兼容渠道（Azure、OpenRouter、Moonshot、Submodel、SiliconFlow、Xinference 等）共享 `openai/` 适配器，通过 `ChannelType` 字段区分。

## 适配器目录结构

```
relay/channel/
├── adapter.go          ← Adaptor / TaskAdaptor / OpenAIVideoConverter 接口定义
├── api_request.go      ← 通用 HTTP 请求发送（含 Header Override）
│
├── openai/             ← OpenAI 适配器（最复杂，被多个渠道复用）
├── claude/             ← Anthropic Claude 适配器
├── gemini/             ← Google Gemini 适配器
├── deepseek/           ← DeepSeek 适配器（委托模式示例）
│
├── aws/                ← AWS Bedrock
├── cloudflare/         ← Cloudflare Workers AI
├── vertex/             ← Google Vertex AI

# 注意：Azure OpenAI 没有独立 channel 目录——它在 `openai/` 适配器内通过
# `ChannelType == ChannelTypeAzure (=3)` 分支处理 deployments URL 和 api-version 拼接。
# 类似的还有 OpenRouter、Moonshot、SiliconFlow、Submodel、Xinference 等"OpenAI 兼容"渠道。
│
├── cohere/             ← Cohere
├── mistral/            ← Mistral AI
├── xai/                ← xAI (Grok)
├── perplexity/         ← Perplexity
│
├── baidu/ baidu_v2/    ← 百度文心
├── ali/                ← 阿里通义
├── zhipu/ zhipu_4v/    ← 智谱 GLM
├── minimax/            ← MiniMax
├── moonshot/           ← Moonshot (月之暗面)
├── tencent/            ← 腾讯混元
├── xunfei/             ← 讯飞星火
├── volcengine/         ← 火山引擎
├── siliconflow/        ← SiliconFlow
├── lingyiwanwu/        ← 零一万物
├── jina/               ← Jina AI
│
├── ollama/             ← Ollama (本地)
├── xinference/         ← Xinference (本地)
├── openrouter/         ← OpenRouter (聚合)
├── replicate/          ← Replicate
│
├── coze/               ← Coze
├── dify/               ← Dify
├── mokaai/             ← Moka AI
├── ai360/              ← 360 AI
├── codex/              ← Codex
├── palm/               ← Google PaLM (旧)
│
├── submodel/           ← 子模型
├── jimeng/             ← 即梦（图像，注意：与 task/jimeng/ 是不同适配器）
├── task/               ← 异步任务平台适配器
│   ├── suno/           ← Suno (音乐)
│   ├── kling/          ← Kling (视频)
│   ├── hailuo/         ← Hailuo (视频)
│   ├── jimeng/         ← 即梦 (视频)
│   ├── sora/           ← Sora (视频)
│   ├── vidu/           ← Vidu (视频)
│   ├── ali/            ← 阿里 (视频)
│   ├── doubao/         ← 豆包 (视频)
│   ├── gemini/         ← Gemini (视频)
│   ├── vertex/         ← Vertex (视频)
│   └── taskcommon/     ← 异步任务共享辅助
│
└── relay/common/       ← 共享 RelayInfo 上下文
    └── relay_info.go   ← RelayInfo（顶层 61 字段 + 多个嵌入子结构，详见下文）
```

<!-- @end-section -->

<!-- @section: adaptor-interface -->
## 核心接口设计

### Adaptor 接口（同步 API）

```go
type Adaptor interface {
    Init(info *relaycommon.RelayInfo)
    GetRequestURL(info *relaycommon.RelayInfo) (string, error)
    SetupRequestHeader(c *gin.Context, req *http.Header, info *relaycommon.RelayInfo) error
    ConvertOpenAIRequest(c *gin.Context, info *relaycommon.RelayInfo, request *dto.GeneralOpenAIRequest) (any, error)
    ConvertRerankRequest(...)
    ConvertEmbeddingRequest(...)
    ConvertAudioRequest(...)
    ConvertImageRequest(...)
    DoRequest(c *gin.Context, info *relaycommon.RelayInfo, requestBody io.Reader) (any, error)
    DoResponse(c *gin.Context, resp *http.Response, info *relaycommon.RelayInfo) (usage any, err *types.NewAPIError)
    GetModelList() []string
    GetChannelName() string
    ConvertClaudeRequest(...)
    ConvertGeminiRequest(...)
}
```

**设计特点**:
- **单向依赖**: 适配器不依赖 Gin 框架以外的业务层
- **无状态**: 大多数适配器是无状态结构体，通过 `RelayInfo` 传递上下文
- **可选实现**: 不是所有渠道都需要实现全部方法（如 Rerank、Embedding）

### TaskAdaptor 接口（异步任务）

```go
type TaskAdaptor interface {
    Init(info *relaycommon.RelayInfo)
    ValidateRequestAndSetAction(...) *dto.TaskError
    EstimateBilling(...) map[string]float64
    AdjustBillingOnSubmit(...) map[string]float64
    AdjustBillingOnComplete(task *model.Task, taskResult *relaycommon.TaskInfo) int
    BuildRequestURL(...) (string, error)
    BuildRequestHeader(...) error
    BuildRequestBody(...) (io.Reader, error)
    DoRequest(...) (*http.Response, error)
    DoResponse(...) (taskID string, taskData []byte, err *dto.TaskError)
    FetchTask(baseUrl, key string, body map[string]any, proxy string) (*http.Response, error)
    ParseTaskResult(respBody []byte) (*relaycommon.TaskInfo, error)
}
```

**异步任务特点**:
- **三段计费**: `EstimateBilling` (预估→预扣) → `AdjustBillingOnSubmit` (提交后调整) → `AdjustBillingOnComplete` (完成时结算)
- **轮询机制**: `FetchTask` + `ParseTaskResult` 支持状态轮询
- 适用于视频生成、音乐生成等长时间任务

### 适配器工厂

```go
func GetAdaptor(apiType int) Adaptor {
    switch apiType {
    case constant.APITypeOpenAI:       return &openai.Adaptor{}
    case constant.APITypeClaude:       return &claude.Adaptor{}
    case constant.APITypeGemini:       return &gemini.Adaptor{}
    case constant.APITypeDeepSeek:     return &deepseek.Adaptor{}
    // ... 40+ case
    default:
        // 兼容 OpenAI 接口的渠道复用 openai.Adaptor
        if isOpenAICompatible(apiType) {
            return &openai.Adaptor{}
        }
    }
}
```

<!-- @end-section -->

<!-- @section: key-adapters -->
## 代表性适配器深度分析

### 1. OpenAI 适配器 — 最复杂、被复用最多

**文件**: `relay/channel/openai/adaptor.go`, `relay-openai.go`, `helper.go`, `audio.go`, `relay_responses.go`

**结构体**:
```go
type Adaptor struct {
    ChannelType    int
    ResponseFormat string
}
```

**请求 URL 构建**:
- 标准: `{baseURL}/v1/chat/completions`
- Azure: 复杂路径拼接 `{baseURL}/openai/deployments/{model}/chat/completions?api-version={version}`
- Custom: 模板替换 `{model}`, `{key}` 占位符
- WebSocket: `wss://` 协议

**请求转换** (`ConvertOpenAIRequest`):
- 非 OpenAI/Azure 渠道: 移除 `StreamOptions`
- OpenRouter: reasoning 格式转换
- o-series/gpt-5 模型: `max_tokens` → `max_completion_tokens`, `system` → `developer` 角色
- 推理力度后缀解析

**响应处理** (`DoResponse`):
- 按 RelayMode 分发到不同 handler（流式/非流式/TTS/STT/图像/WebSocket/Responses）
- Usage 后处理：兼容 DeepSeek、Zhipu、Moonshot、llama.cpp 的特殊缓存 token 格式

**复用者**: OpenRouter、Xinference、Moonshot、Submodel、SiliconFlow 等 OpenAI 兼容渠道直接复用此适配器。

### 2. Claude 适配器 — 独立格式适配

**文件**: `relay/channel/claude/adaptor.go`, `relay-claude.go`, `dto.go`

**请求 URL**: `{baseURL}/v1/messages`（+ 可选 `?beta=true`）

**请求头**: `x-api-key`、`anthropic-version`（默认 `2023-06-01`）、`anthropic-beta`

**格式转换** (`RequestOpenAI2ClaudeMessage`):
```
OpenAI messages[]          →  Claude system + messages[]
OpenAI tools[]             →  Claude tools[]
OpenAI web_search_options  →  Claude web_search tools
OpenAI stop                →  Claude stop_sequences
```

**响应处理**:
- 始终设置 `FinalRequestRelayFormat = RelayFormatClaude`
- 流式: SSE 逐事件转换 (`ClaudeStreamHandler`)
- 非流式: 聚合响应转换 (`ClaudeHandler`)

### 3. Gemini 适配器 — 动态 URL 构建

**请求 URL** (按模型类型动态构建):
| 模型类型 | URL 模式 |
|----------|----------|
| imagen | `{baseURL}/v1/models/{model}:predict` |
| embedding | `{baseURL}/v1/models/{model}:embedContent` |
| 聊天 | `{baseURL}/v1/models/{model}:generateContent` |
| 流式聊天 | `{baseURL}/v1/models/{model}:streamGenerateContent?alt=sse` |

**请求头**: `x-goog-api-key`

**图像参数转换**: OpenAI `size` → Gemini `aspectRatio`, OpenAI `quality` → Gemini `imageSize`

**多级转换链**:
```
Claude Request → [OpenAI 适配器] → OpenAI Request → [Gemini 适配器] → Gemini Request
```

### 4. DeepSeek 适配器 — 委托模式

DeepSeek 适配器展示了**委托/复用模式**，本身不实现完整的格式转换：
- `ConvertClaudeRequest`: 委托 `claude.Adaptor` 处理后叠加 thinking 后缀
- `GetRequestURL`: Claude 格式走 `/anthropic/v1/messages`，OpenAI 格式走 `/beta/v1/chat/completions`
- `DoResponse`: 根据 `RelayFormat` 委托给 `claude.Adaptor{}` 或 `openai.Adaptor{}`

<!-- @end-section -->

<!-- @section: relay-flow -->
## 请求中继流程

### 同步 API 流程（以 TextHelper 为例）

```
用户请求 (POST /v1/chat/completions)
  │
  ▼
1. 初始化渠道元数据 (info.InitChannelMeta)
  │  设置 ChannelType, ChannelId, BaseURL, Key, ModelMapping 等
  │
  ▼
2. 模型映射 (helper.ModelMappedHelper)
  │  支持链式重定向: model_a → model_b → model_c
  │  含循环检测保护
  │
  ▼
3. 获取适配器 (GetAdaptor + adaptor.Init)
  │
  ▼
4. 请求转换 (adaptor.ConvertOpenAIRequest)
  │  OpenAI 统一格式 → 上游原生格式
  │
  ▼
5. 去除禁用字段 (RemoveDisabledFields)
  │  service_tier, safety_identifier, inference_geo 等
  │
  ▼
6. 参数覆盖 (ApplyParamOverrideWithRelayInfo)
  │
  ▼
7. 发送请求 (adaptor.DoRequest)
  │  ├── 构建 HTTP 请求 (URL + Header)
  │  ├── Header Override 处理 (支持 * 全透传、正则匹配)
  │  ├── 代理客户端
  │  └── SSE Ping 保活
  │
  ▼
8. 错误处理
  │  ├── 状态码映射 (status_code_mapping)
  │  ├── 自动禁用渠道 (auto_ban)
  │  └── 重试决策
  │
  ▼
9. 响应处理 (adaptor.DoResponse)
  │  ├── 流式: SSE Scanner goroutine → DataHandler goroutine
  │  └── 非流式: 直接解析
  │
  ▼
10. 计费结算 (PostTextConsumeQuota / SettleBilling)
```

### SSE 流处理架构

```
上游 SSE 流
  │
  ▼
Scanner Goroutine                     DataHandler Goroutine
  │                                       ▲
  ├── 按行读取 body                       │
  ├── 解析 "data:" 行                    │
  ├── 通过 dataChan 传递 ─────────────────┘
  ├── Ping 保活
  └── 超时控制
                    │
                    ▼
              stopChan ←→ 协调退出
```

### 异步任务流程

```
1. 刷新渠道元数据 → 确定 platform
2. 创建 TaskAdaptor → 验证请求
3. 模型映射 → 价格计算 → 计费估算
4. 预扣费
5. 构建请求体 → 发送请求
6. 解析响应 → 提交后计费调整
7. 返回上游任务 ID 和数据（后续轮询）
```

<!-- @end-section -->

<!-- @section: shared-infrastructure -->
## 共享基础设施

### RelayInfo — 中继上下文

`relay/common/relay_info.go:87-181` 中定义，**顶层 61 个字段**（按 grep `^\s+[A-Z][a-zA-Z]` 实测），加上多个嵌入子结构后实际可访问的属性更多——但所谓"180 字段"是误传，请以代码为准。`RelayInfo` 是贯穿整个中继流程的上下文对象。

**嵌入结构**:
- `*ChannelMeta` — 渠道元数据（类型、ID、URL、Key、模型映射等）
- `*TaskRelayInfo` — 异步任务信息（Action、TaskID）
- `ThinkingContentInfo` — 思维链内容处理
- `*ClaudeConvertInfo` — Claude 格式转换中间状态
- `*RerankerInfo` / `*ResponsesUsageInfo` — 专项信息
- `PriceData`, `StreamStatus`, `RequestConversionChain`

**工厂函数**:
- `GenRelayInfoOpenAI()` — OpenAI 格式请求
- `GenRelayInfoClaude()` — Claude 格式请求
- `GenRelayInfoGemini()` — Gemini 格式请求
- `GenRelayInfoImage()` — 图像生成请求

### Header Override 系统

支持三种穿透模式，优先级为：**默认 < SetupRequestHeader < Header Override**：

| 模式 | 语法 | 说明 |
|------|------|------|
| 全透传 | `*` | 将客户端所有请求头转发到上游 |
| 正则匹配 | `re:pattern` | 匹配的头部透传 |
| 客户端引用 | `{client_header:NAME}` | 将客户端指定头的值注入上游请求 |

### 模型映射 (Model Mapped)

- 支持链式重定向: `model_a → model_b → model_c`
- 循环检测保护（最多 10 层）
- 通过 `Channel.ModelMapping` JSON 配置

### 参数覆盖 (Param Override)

- 通过 `Channel.ParamOverride` JSON 配置
- 支持按模型名 + 条件规则覆盖请求参数
- 在 `ConvertOpenAIRequest` 之后、`DoRequest` 之前执行

### RemoveDisabledFields

在请求发出前自动移除不应暴露给上游的字段：
`service_tier`, `safety_identifier`, `inference_geo`, `response_format`（对不支持 JSON mode 的渠道）

<!-- @end-section -->

<!-- @section: patterns -->
## 核心设计模式

### 1. 策略模式 + 工厂模式
`Adaptor` 接口 = 策略，`GetAdaptor()` = 工厂。新增提供商只需新建目录 + 实现接口 + 注册到工厂。

### 2. 委托/复用模式
兼容 OpenAI 的渠道直接复用 `openai.Adaptor{}`。DeepSeek 委托给 Claude/OpenAI 适配器后叠加自身逻辑。

### 3. 多级转换链
`RequestConversionChain` 记录完整转换路径：
```
OpenAI → Claude → Gemini
OpenAI → OpenAIResponses
```

### 4. 三段计费（异步任务）
`EstimateBilling` → `AdjustBillingOnSubmit` → `AdjustBillingOnComplete`

### 5. SSE 双 Goroutine 流处理
Scanner 读上游 → channel → DataHandler 写下游，stopChan 协调退出。

<!-- @end-section -->

<!-- @section: related -->
## 相关文档

- [[01-overview|New API 项目架构总览]]
- [[02-module-analysis|Go 模块功能分析]]
- [[04-billing-quota-system|计费与配额系统分析]]
- [[05-middleware-and-flow|中间件与请求流程分析]]

<!-- @end-section -->
