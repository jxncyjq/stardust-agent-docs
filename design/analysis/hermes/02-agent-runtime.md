---
id: "analysis-hermes-runtime-002"
title: "Agent 运行时引擎分析"
aliases: ["hermes agent runtime", "AIAgent loop", "Agent运行时"]
type: "analysis"
category: "design/analysis/hermes"
tags: ["hermes-agent", "agent-runtime", "agent-loop", "transports", "llm"]
version: "1.0.0"
created: "2026-05-03"
updated: "2026-05-03"
author: "jxncyjq"
status: "review"
parent: "analysis-hermes-overview-001"
related_docs:
  - id: "analysis-hermes-overview-001"
    relation: "parent"
    path: "./01-overview.md"
  - id: "analysis-hermes-tools-003"
    relation: "related_to"
    path: "./03-tools-skills-plugins.md"
---

<!-- @section: overview -->
# Agent 运行时引擎分析

## 系统概述

Hermes Agent 的核心运行时在 `run_agent.py` 中实现（14,123 行），`AIAgent` 类（声明在 873 行）是其核心。Agent 内部模块在 `agent/` 目录下（56 个 .py 文件，含 `transports/` 子包 6 个文件），负责提供商适配、上下文管理、提示构建、记忆管理等关键功能。

## AIAgent 核心类

### 初始化参数

`AIAgent.__init__()` 接受 **63 个参数**（包含 `self`），分组：

- **凭证**: api_key, provider, base_url, model
- **路由**: fallback_models（故障转移链）, credential_pool
- **工具集**: enabled_toolsets（工具集列表）
- **会话**: session_id, system_prompt, workdir
- **记忆**: memory_store, memory_manager
- **回调**: message_callback, step_callback, thinking_callback
- **预算**: iteration_budget（迭代预算）, max_iterations
- **UI**: kawaii 模式, tool_preview_max_len

### 主循环

`run_conversation()` 中的核心循环：

```python
while (api_call_count < max_iterations
       and iteration_budget.remaining > 0
       or _budget_grace_call):
    # 1. 中断检查
    if self._interrupt_requested:
        break

    # 2. 迭代消耗
    iteration_budget.consume()

    # 3. 构建 API 消息
    api_messages = prepare_messages(
        system_prompt + context + tool_definitions + history
    )

    # 4. API 调用（带重试循环）
    response = _interruptible_api_call(
        model, messages=api_messages, tools=tool_defs, stream=True
    )

    # 5. 解析 NormalizedResponse
    normalized = transport.normalize_response(response)
    #    → content, tool_calls, finish_reason, reasoning, usage

    # 6. 如果没有工具调用且无内容 → 空响应重试
    # 7. 如果有工具调用 → 执行
    if normalized.tool_calls:
        results = _execute_tool_calls(normalized.tool_calls)
        messages.append(tool_results)

    # 8. 持久化 → SessionDB
    # 9. 触发 post_llm_call 钩子
```

### 中断处理

支持两种中断方式：
- **用户中断**: Ctrl+C → 优雅停止当前操作
- **编程中断**: `agent.interrupt()` → 设置 `_interrupt_requested` 标志
- 中断后当前 API 调用完成但不继续下一轮

## 传输层 (Transports)

### 架构

传输层是 Hermes Agent 最核心的抽象，将提供商特定的 API 与 Agent 循环解耦：

```
ProviderTransport (ABC)
  ├── convert_messages()    — OpenAI 格式 → 提供商原生格式
  ├── convert_tools()       — OpenAI 工具定义 → 提供商原生格式
  ├── build_kwargs()        — 完整 API 调用参数字典
  └── normalize_response()  — 提供商原始响应 → NormalizedResponse
```

### NormalizedResponse（规范响应）

```python
@dataclass
class NormalizedResponse:
    content: str                    # 文本内容
    tool_calls: List[ToolCall]      # 工具调用列表
    finish_reason: str              # 完成原因
    reasoning: Optional[str]        # 推理内容（思维链）
    usage: Usage                    # Token 使用统计
```

### 已支持的传输

`agent/transports/` 下定义了 4 个 `ProviderTransport` 子类：

| Transport | 文件 | api_mode | 提供商 |
|-----------|------|----------|--------|
| ChatCompletionsTransport | `agent/transports/chat_completions.py` | `chat_completions` | OpenAI, OpenRouter, 多数兼容 API |
| AnthropicTransport | `agent/transports/anthropic.py` | `anthropic_messages` | Anthropic Claude |
| ResponsesApiTransport | `agent/transports/codex.py` | `codex_responses` | OpenAI Responses API, Codex |
| BedrockTransport | `agent/transports/bedrock.py` | `bedrock` | AWS Bedrock |

> **Gemini 适配器的位置**：`gemini_native_adapter.py` / `gemini_cloudcode_adapter.py` 直接放在 `agent/` 顶层（不在 `transports/` 下），它们复用 `ChatCompletionsTransport` 的格式约定但走独立的请求/响应改写路径。这一历史分裂在重构 Transport 时需要并入 `transports/` 子包。

### API 调用客户端

- 主 Agent: `self.client`（openai.OpenAI 或 anthropic.Anthropic）
- 辅助任务（压缩/视觉/搜索）: `agent/auxiliary_client.py` → `call_llm()`
- 自动检测提供者链路: 主提供者 → OpenRouter → Nous → 自定义端点 → Anthropic → 直接 API Key

## 提供商适配器

### Anthropic 适配器

**文件**: `agent/anthropic_adapter.py` (84KB)

**核心功能**:
- 消息格式转换: `convert_messages_to_anthropic()` — OpenAI messages → Anthropic messages
- OAuth 持有者认证
- Claude Code 凭证解析
- 自适应思考努力级别: xhigh/high/medium/low
- 模型特定输出限制: haiku 16K, sonnet 64-128K, opus 32-128K
- 提示缓存控制点注入

### Bedrock 适配器

**文件**: `agent/bedrock_adapter.py` (50KB)

**核心功能**:
- 元数据头管理
- PEM 签名验证
- 凭证轮换机制
- AWS 区域路由

### Codex/Responses API 适配器

**文件**: `agent/codex_responses_adapter.py` (46KB)

**核心功能**:
- Chat Completions → Responses API 输入项转换
- 思考签名处理
- GitHub Copilot 后端集成

### Gemini 适配器

**文件**: `agent/gemini_native_adapter.py` (35KB), `agent/gemini_cloudcode_adapter.py` (35KB)

**核心功能**:
- Gemini 原生 Content 格式转换
- Gemini 3 思考配置 (thinkingLevel: low/medium/high)
- Google Cloud Code (AI Studio) 适配
- 安全配置映射 (`gemini_schema.py`)

## 提示构建系统

### 系统提示组装流程

`agent/prompt_builder.py` (56KB) — `AIAgent._build_system_prompt()` 按顺序组装：

```
1. Agent 身份
   "You are Hermes Agent, an intelligent AI assistant by Nous Research..."

2. 平台提示
   操作系统、Shell、工作目录、日期/时间

3. 技能索引
   每个已安装技能的: 触发条件 + 描述 + 路径

4. 上下文文件
   加载 .hermes.md / HERMES.md (cwd → 父目录)
   + SOUL.md 自定义角色

5. 模型特定执行指南
   ├── TOOL_USE_ENFORCEMENT_GUIDANCE (GPT/Codex/Gemini 通用)
   ├── OPENAI_MODEL_EXECUTION_GUIDANCE (GPT 特定)
   └── GOOGLE_MODEL_OPERATIONAL_GUIDANCE (Gemini 特定)

6. 记忆指南
   何时保存记忆 vs 技能的区别

7. 看板模式
   多 Agent 任务调度的工作器生命周期协议

8. 安全检查
   扫描注入模式 (ignore previous instructions、隐藏 Unicode)
```

### 提示缓存策略

`agent/prompt_caching.py` — 为 Anthropic 模型注入 `cache_control` 断点：
- **关键原则**: 对话中期永不修改系统提示/工具集，保持缓存稳定
- 斜杠命令默认延迟生效，需要 `--now` 强制立即应用

## 上下文压缩

### ContextEngine ABC

```python
class ContextEngine(ABC):
    def on_session_start(self, messages, system_prompt)
    def update_from_response(self, response)
    def should_compress(self, messages) -> bool
    def compress(self, messages) -> List[Message]
    def on_session_end(self)
```

### 默认引擎: ContextCompressor

`agent/context_compressor.py` (67KB):

**压缩算法**:
1. **裁剪旧工具输出**: 超过 1 回合的工具结果替换为 `[Old tool output cleared...]`
2. **保护头部**: 前 3 条消息永不压缩
3. **保护尾部**: 最近 6 条消息受令牌预算保护
4. **摘要中间部分**: 调用辅助 LLM 对中间回合进行摘要
   - 之前的摘要迭代 → 更新
   - 活动和待处理问题跟踪
   - 剩余工作（措辞为参考，而非指令）
5. **迭代更新**: 摘要前缀标记为仅参考

### 可插拔上下文引擎

支持通过 `plugins/context_engine/` 替换压缩引擎。

## 错误处理与恢复

### 错误分类学

`agent/error_classifier.py` (39KB) — `FailoverReason` 枚举：

| 错误类型 | 恢复策略 |
|----------|----------|
| `auth` / `auth_permanent` | 刷新/轮换凭证 |
| `billing` (402) | 立即轮换凭证 |
| `rate_limit` (429) | 退避后轮换 |
| `context_overflow` | 压缩而非故障转移 |
| `model_not_found` | 回退到备用模型 |
| `timeout` | 抖动退避重试 |
| `thinking_signature` | 删除思考块重试 |
| `provider_policy_blocked` | 跳过被阻止端点 |
| `overloaded` | 退避等待 |

每个分类携带恢复提示: `retryable`, `should_compress`, `should_rotate_credential`, `should_fallback`

### 凭证池

`agent/credential_pool.py` (69KB) — 同提供者多凭证故障转移：

- `PooledCredential` 管理单个凭证的状态
- 策略: `fill_first`, `round_robin`, `random`, `least_used`
- 处理 OAuth 令牌刷新、API Key 轮换、耗尽冷却
- 凭证来源: `agent/credential_sources.py` — 环境变量、auth.json、密钥链

### 速率限制保护

- `agent/nous_rate_guard.py` — 跨会话速率限制，429 响应时写共享文件
- `agent/rate_limit_tracker.py` — 通用速率限制跟踪器（JSON 状态文件）
- 退避: `agent/retry_utils.py` — 带随机抖动的指数退避

## 记忆系统

### MemoryProvider ABC

```python
class MemoryProvider(ABC):
    def initialize(self)
    def system_prompt_block(self) -> str
    def prefetch(self, messages) -> str
    def sync_turn(self, messages)
    def get_tool_schemas(self) -> List[Dict]
    def handle_tool_call(self, name, args) -> str
    def shutdown(self)
```

### MemoryManager

`agent/memory_manager.py` (21KB) — 编排器：

- **内置提供者**: 始终存在（MEMORY.md / USER.md）
- **外部提供者**: 最多一个活跃 (Honcho, Mem0, Supermemory 等 8 种)
- 为系统提示、回合前预取、回合后同步提供单一集成点
- `StreamingContextScrubber` 从流式文本中移除 `<memory-context>` 标签

### 已支持的外部记忆提供者

| 提供者 | 目录 | 特点 |
|--------|------|------|
| Honcho | `plugins/memory/honcho/` | 对话记忆 |
| Mem0 | `plugins/memory/mem0/` | 向量记忆 |
| Supermemory | `plugins/memory/supermemory/` | 云端记忆 |
| Byterover | `plugins/memory/byterover/` | 字节级存储 |
| Hindsight | `plugins/memory/hindsight/` | 反思记忆 |
| Holographic | `plugins/memory/holographic/` | SQLite + FTS5 + HRR |
| OpenViking | `plugins/memory/openviking/` | 开放标准 |
| RetainDB | `plugins/memory/retaindb/` | 向量数据库 |

## 工具执行安全

### 工具看门狗

`agent/tool_guardrails.py` (17KB)：每回合追踪工具调用模式。

**检测维度**：
- **精确重复失败**（同名工具 + 同参数）—— 阈值默认 2 次，超过升级为 `warn`、再次升级为 `block`
- **同类工具连续失败**（不同参数但相同 `tool_name`）—— 阈值默认 3 次
- **幂等工具无进展**（写入/读取无新信息）—— 检查输出是否与上一轮相同

**4 级决策**：
| 决策 | 含义 | 配置项 |
|------|------|--------|
| `allow` | 默认放行 | — |
| `warn` | 注入警告到下一轮系统消息 | 第 N 次重复触发 |
| `block` | 拒绝执行，返回错误结果 | 警告后继续重复触发 |
| `halt` | 终止整个 Agent 循环 | 阻止后仍重复触发 |

阈值在 `config.yaml` 的 `guardrails:` 段下可定制。

### IterationBudget 迭代预算

`run_agent.py::IterationBudget` 是线程安全的轮次计数器：

| 来源 | 默认值 | 说明 |
|------|--------|------|
| 父 Agent | `max_iterations=90` | 顶层会话上限 |
| 子 Agent | `delegation.max_iterations=50` | 每个 `delegate_task` 子 Agent 独立预算 |

子 Agent 的预算是独立的，因此父+子加起来可以超过 90。`execute_code`（程序化工具调用）通过 `IterationBudget.refund()` 退还预算，避免占用业务轮次。

### 文件安全

`agent/file_safety.py` — 绝对文件写入黑名单:
`~/.ssh/`, `~/.aws/`, `/etc/sudoers`, `/etc/shadow` 等关键路径

### 日志脱敏

`agent/redact.py` (15KB) — 基于正则的秘密信息删除，覆盖 95+ API Key 前缀模式

## 其他核心模块

### 模型元数据

`agent/model_metadata.py` (62KB):
- 模型上下文长度查找
- 粗略 Token 估算 `estimate_messages_tokens_rough()`
- 提供商前缀检测
- Ollama 私有网络检测

### 用量定价

`agent/usage_pricing.py` (27KB):
- `CanonicalUsage`, `BillingRoute`, `PricingEntry`, `CostResult` 数据类
- 多源定价（提供商 API、models.dev、官方文档、用户覆盖）

### 会话洞察

`agent/insights.py` (40KB):
- Token 消耗、成本估算、工具使用模式
- 活动趋势、模型/平台细分
- 类似 Claude Code 的 `/insights` 命令

### 轨迹保存

`agent/trajectory.py` — ShareGPT 格式 JSONL，可选失败轨迹文件

### 标题生成

`agent/title_generator.py` — 辅助模型摘要 → 简洁会话标题

<!-- @end-section -->

<!-- @section: related -->
## 相关文档

- [[01-overview|Hermes Agent 项目架构总览]]
- [[03-tools-skills-plugins|工具、技能与插件系统分析]]
- [[05-data-models|状态持久化与数据模型分析]]

<!-- @end-section -->
