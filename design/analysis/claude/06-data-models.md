---
id: "analysis-clawcode-datamodel-006"
title: "Claw Code 核心数据模型与类型系统"
aliases: ["数据模型", "claw-code types", "类型系统", "序列化"]
type: "analysis"
category: "design/analysis/claude"
tags: ["claw-code", "data-model", "types", "serialization", "session"]
version: "1.0.0"
created: "2026-05-03"
updated: "2026-05-03"
author: "jxncyjq"
status: "review"
parent: "analysis-clawcode-overview-001"
children: []
related_docs:
  - id: "analysis-clawcode-overview-001"
    relation: "depends_on"
    path: "./01-overview.md"
  - id: "analysis-clawcode-flows-005"
    relation: "related_to"
    path: "./05-architecture-flows.md"
---

<!-- @section: overview -->
# Claw Code 核心数据模型与类型系统

> 深入分析 claw-code 的数据模型设计、类型转换和序列化策略。

---

## 一、数据模型关系总览

```
══════════ 配置层 ══════════
ConfigLoader ──► 发现 ConfigEntry 列表 (User/Project/Local 三级)
  └─► RuntimeConfig ──► 合并后的 BTreeMap + RuntimeFeatureConfig
       (hooks, plugins, mcp, oauth, model, aliases, permission_mode,
        permission_rules, sandbox, provider_fallbacks, trusted_roots)

══════════ 会话与消息层 ══════════
Session (1) ──► (0..1) SessionCompaction
             ├─► (0..1) SessionFork
             ├─► (N)   SessionPromptEntry
             └─► (N)   ConversationMessage
                            └─► (1..N) ContentBlock
                            └─► (0..1) TokenUsage

══════════ API 通信层 ══════════
MessageRequest ──► InputMessage[] ──► InputContentBlock[]
                ├─► ToolDefinition[]
                └─► ToolChoice

MessageResponse ──► OutputContentBlock[] + Usage

StreamEvent (tagged union)
  ├─► MessageStart, MessageDelta, ContentBlockStart
  ├─► ContentBlockDelta, ContentBlockStop, MessageStop

══════════ 计费与用量层 ══════════
API.Usage ──convert──► runtime.TokenUsage
TokenUsage ──compute──► UsageCostEstimate
TokenUsage ──aggregate──► UsageTracker
ModelPricing (sonnet/haiku/opus) ──▲
UsageTracker ──reconstitute── Session

══════════ 权限层 ══════════
RuntimeConfig ──feed──► PermissionPolicy
  ├─► PermissionMode (枚举声明顺序，仅供 derive(Ord) 比较；详见 §2.4)
  ├─► tool_requirements: BTreeMap<String, PermissionMode>
  ├─► allow/deny/ask rules
  └─► PermissionOutcome (Allow / Deny)

══════════ Bash 与文件操作层 ══════════
BashCommandInput ──► execute_bash() ──► BashCommandOutput
file_ops: read_file / write_file / edit_file / glob_search / grep_search

══════════ MCP 协议层 ══════════
RuntimeConfig ──► McpConfigCollection ──► ScopedMcpServerConfig
McpServerManager ──► ManagedMcpServer[] + tool_index + unsupported_servers
JsonRpcRequest <──► JsonRpcResponse (initialize/tools/list/tools/call/resources)

══════════ LSP 层 ══════════
LspRegistry ──► LspServerState[] (Connected/Disconnected/Starting/Error)
             ──► LspAction (Diagnostics/Hover/Definition/References/Completion/Symbols/Format)

══════════ Task 生命周期层 ══════════
TaskPacket ──validate──► ValidatedPacket
TaskRegistry ──► Task[] (Created→Running→Completed/Failed/Stopped)
TeamRegistry ──► Team[] (Created/Running/Completed/Deleted)
CronRegistry ──► CronJob[] (Active/Inactive/Blocked/Deleted)
```

---

## 二、核心数据结构详解

### 2.1 API 请求类型 (`api::types.rs`)

**MessageRequest**
| 字段 | 类型 | 策略 | 说明 |
|------|------|------|------|
| `model` | `String` | 始终 | 模型标识符 |
| `max_tokens` | `u32` | 始终 | 最大输出 token |
| `messages` | `Vec<InputMessage>` | 始终 | 对话历史 |
| `system` | `Option<String>` | skip_none | 系统提示词 |
| `tools` | `Option<Vec<ToolDefinition>>` | skip_none | 可用工具 |
| `tool_choice` | `Option<ToolChoice>` | skip_none | 工具选择策略 |
| `stream` | `bool` | skip_if_false | 流式开关 |
| `temperature` | `Option<f64>` | skip_none | 温度 |
| `top_p` | `Option<f64>` | skip_none | top-p 采样 |
| `stop` | `Option<Vec<String>>` | skip_none | 停止序列 |
| `reasoning_effort` | `Option<String>` | skip_none | 推理力度 |

**InputContentBlock** (tagged enum)
| 变体 | 字段 |
|------|------|
| `Text` | `text: String` |
| `ToolUse` | `id, name, input: Value` |
| `ToolResult` | `tool_use_id, content, is_error` |

**StreamEvent** (流式事件 tagged enum)
| 变体 | 内容 |
|------|------|
| `MessageStart` | 完整 MessageResponse |
| `MessageDelta` | delta + usage |
| `ContentBlockStart` | index + OutputContentBlock |
| `ContentBlockDelta` | index + ContentBlockDelta |
| `ContentBlockStop` | index |
| `MessageStop` | 空 |

**ContentBlockDelta**
| 变体 | 字段 |
|------|------|
| `TextDelta` | `text: String` |
| `InputJsonDelta` | `partial_json: String` |
| `ThinkingDelta` | `thinking: String` |
| `SignatureDelta` | `signature: String` |

### 2.2 会话类型 (`session.rs`)

**Session**
| 字段 | 类型 | 说明 |
|------|------|------|
| `version` | `u32` | 固定为 1 |
| `session_id` | `String` | format: `session-{ms}-{counter}` |
| `created_at_ms` | `u64` | 创建毫秒戳 |
| `updated_at_ms` | `u64` | 最后修改 |
| `messages` | `Vec<ConversationMessage>` | 消息列表 |
| `compaction` | `Option<SessionCompaction>` | 压缩元数据 |
| `fork` | `Option<SessionFork>` | 分叉来源 |
| `workspace_root` | `Option<PathBuf>` | 绑定工作区 |
| `prompt_history` | `Vec<SessionPromptEntry>` | 提示历史 |
| `model` | `Option<String>` | 使用的模型 |

**ConversationMessage**
| 字段 | 类型 |
|------|------|
| `role` | `MessageRole` (System/User/Assistant/Tool) |
| `blocks` | `Vec<ContentBlock>` |
| `usage` | `Option<TokenUsage>` (仅 assistant 消息) |

**ContentBlock** (会话层，与 API 层不同，input 存为 String)
| 变体 | 字段 |
|------|------|
| `Text` | `text: String` |
| `ToolUse` | `id, name, input: String` (JSON 字符串) |
| `ToolResult` | `tool_use_id, tool_name, output: String, is_error: bool` |

### 2.3 配置类型 (`config.rs`)

**ConfigSource** (有 `Ord`, 优先级递增)
`User < Project < Local`

**RuntimeFeatureConfig**
| 字段 | 类型 |
|------|------|
| `hooks` | `RuntimeHookConfig` (pre/post_tool_use) |
| `plugins` | `RuntimePluginConfig` |
| `mcp` | `McpConfigCollection` |
| `oauth` | `Option<OAuthConfig>` |
| `model` | `Option<String>` |
| `aliases` | `BTreeMap<String, String>` |
| `permission_mode` | `Option<ResolvedPermissionMode>` |
| `permission_rules` | `RuntimePermissionRuleConfig` (allow/deny/ask) |
| `sandbox` | `SandboxConfig` |
| `provider_fallbacks` | `ProviderFallbackConfig` |
| `trusted_roots` | `Vec<String>` |

**配置发现顺序** (ConfigLoader.discover):
1. `<config_home>/../.claw.json` (User，兼容旧格式)
2. `<config_home>/settings.json` (User)
3. `<cwd>/.claw.json` (Project)
4. `<cwd>/.claw/settings.json` (Project)
5. `<cwd>/.claw/settings.local.json` (Local)

**MCP 配置层次**
```
McpConfigCollection
  └─ servers: BTreeMap<String, ScopedMcpServerConfig>
        └─ config: McpServerConfig (enum)
              ├─ Stdio(command, args, env, timeout)
              ├─ Sse(url, headers, oauth)
              ├─ Http(url, headers, oauth)
              ├─ Ws(url, headers)
              ├─ Sdk(name)
              └─ ManagedProxy(url, id)
```

### 2.4 权限类型 (`permissions.rs`)

**PermissionMode** (有 `Ord`，按枚举声明顺序)
`ReadOnly < WorkspaceWrite < DangerFullAccess < Prompt < Allow`

> ⚠️ 这个序排是 `derive(Ord)` 按变体声明顺序生成的**结构序**，并非"信任递增"的语义序。`Prompt` / `Allow` 排在 `DangerFullAccess` 之后是为了让交互/无条件放行的两种特殊模式参与"模式 ≥ 要求"的比较时不被误判，但在 `permission_enforcer.rs` 中 `Prompt` 走的是"自动放行交给上层 prompter"分支、`Allow` 走的是无条件放行分支，它们和 `DangerFullAccess` 不构成可比较的权限层级。读代码时不要依赖这个 `Ord` 做安全判断，应使用枚举模式匹配。

**权限评估流程**:
```
PermissionPolicy.authorize():
  1. deny_rules 检查 → Deny (短路)
  2. PermissionContext.override_decision:
     Deny → Deny | Ask → prompt_or_deny() | Allow → 继续
  3. ask_rules 检查 → prompt_or_deny()
  4. allow_rules / Allow mode / mode ≥ required → Allow
  5. Prompt 模式或需要升级 → prompt_or_deny()
  6. 否则 → Deny
```

**权限规则解析**：`tool_name(subject)` 格式，从工具 JSON input 提取匹配对象：command → path → file_path → url → pattern → code → message

### 2.5 用量与计费 (`usage.rs`)

**ModelPricing** (每百万 token，美元)
| 模型族 | Input | Output | Cache Write | Cache Read |
|--------|-------|--------|-------------|------------|
| sonnet | $15.00 | $75.00 | $18.75 | $1.50 |
| haiku | $1.00 | $5.00 | $1.25 | $0.10 |
| opus | $15.00 | $75.00 | $18.75 | $1.50 |

**TokenUsage** (Copy, 四维计数)
`input_tokens | output_tokens | cache_creation_input_tokens | cache_read_input_tokens`

**UsageCostEstimate** (Copy)
`input_cost + output_cost + cache_creation_cost + cache_read_cost`

**UsageTracker** — 跨 turn 累计追踪
`latest_turn + cumulative + turns`

### 2.6 Bash 操作类型 (`bash.rs`)

**BashCommandInput** (camelCase)
command / timeout / description / run_in_background / dangerously_disable_sandbox / namespace_restrictions / isolate_network / filesystem_mode / allowed_mounts

**BashCommandOutput**
stdout / stderr (均截断到 16KB) / raw_output_path / interrupted / is_image / background_task_id / sandbox_status / return_code_interpretation / structured_content

### 2.7 文件操作类型 (`file_ops.rs`)

| 操作 | 输出类型 | 关键字段 |
|------|----------|----------|
| read_file | ReadFileOutput | TextFilePayload (file_path, content, num_lines, start_line, total_lines) |
| write_file | WriteFileOutput | file_path, content, structured_patch, original_file, git_diff |
| edit_file | EditFileOutput | file_path, old_string, new_string, structured_patch, replace_all |
| glob_search | GlobSearchOutput | duration_ms, filenames (最多 100), truncated |
| grep_search | GrepSearchOutput | mode, num_files, filenames, content, num_matches |

### 2.8 Task 系统

**TaskPacket** — 子代理任务规格
objective / scope (Workspace/Module/SingleFile/Custom) / scope_path / repo / worktree / branch_policy / acceptance_tests / commit_policy / reporting_contract / escalation_policy

**验证规则**：
1. 6 个必填字段非空白
2. Module/SingleFile/Custom 必须提供 scope_path
3. acceptance_tests 每项非空白

**Task** — 运行时任务实例
task_id / prompt / description / task_packet / status (Created→Running→Completed/Failed/Stopped) / created_at / updated_at / messages / output / team_id

### 2.9 MCP Wire 类型 (`mcp_stdio.rs`)

**JsonRpcId** (untagged): `Number(u64) | String(String) | Null`

**JsonRpcRequest<T>**: `{ jsonrpc, id, method, params? }`

**JsonRpcResponse<T>**: `{ jsonrpc, id, result?, error? }`

**MCP 工具**: `McpTool { name, description?, input_schema?, annotations?, _meta? }`

**已管理工具**: `ManagedMcpTool { server_name, qualified_name: "<server>--<tool>", raw_name, tool }`

### 2.10 LSP 类型 (`lsp_client.rs`)

**LspAction**: Diagnostics / Hover / Definition / References / Completion / Symbols / Format

**结果类型**: LspDiagnostic / LspLocation / LspHoverResult / LspCompletionItem / LspSymbol

**扩展名→语言映射**: rs→rust, ts/tsx→typescript, js/jsx→javascript, py→python, go→go, java→java, c/h→c, cpp/hpp/cc→cpp, rb→ruby, lua→lua

---

## 三、类型转换关系

### 3.1 API ↔ Runtime 转换
```
API.Usage ──.token_usage()──► TokenUsage
API.OutputContentBlock ──► ContentBlock (Value → String)
API.Thinking → Text (仅 thinking); RedactedThinking → 忽略
```

### 3.2 配置 → 运行时对象
```
ConfigLoader.load()
  → 发现 5 个 ConfigEntry
  → parse + validate 每个
  → merge_mcp_servers()
  → deep_merge_objects() (JSON 深度合并)
  → 解析 RuntimeFeatureConfig (11 个子系统)
```

### 3.3 Session 持久化
```
Session ──to_json()──► 完整 JSON
       ──render_jsonl_snapshot()──► JSONL (增量追加)
       ──from_json()/from_jsonl()──► 兼容加载

每次 push_message() → 增量追加 JSONL
超过 256KB → 自动轮转 (保留 3 个备份)
```

### 3.4 MCP 发现流
```
McpServerManager.from_runtime_config()
  → 为每个 Stdio 服务器创建 ManagedMcpServer
  → discover_tools():
      initialize (handshake) → tools/list → 构建 ManagedMcpTool[]
  → 失败收集: McpDiscoveryFailure[] → McpToolDiscoveryReport
```

---

## 四、序列化策略

### 两大体系对比

| 维度 | API 层 | 运行时 Session |
|------|--------|----------------|
| JSON 库 | `serde_json::Value` | 自研 `JsonValue` |
| 序列化 | `#[derive(Serialize)]` | 手动 `to_json()/from_json()` |
| 字段命名 | snake_case / camelCase | snake_case |
| 类型标记 | `#[serde(tag="type")]` | 手动 `"type"` 字段 |
| 数值安全 | 直接 u32 | u32 → i64 → u32 (范围检查) |

### Session 双格式持久化

**JSONL** (主流):
```json
{"type":"session_meta","version":1,"session_id":"session-xxx",...}
{"type":"message","message":{"role":"assistant","blocks":[...]}}
```

**完整 JSON** (兼容):
```json
{"version":1,"session_id":"session-xxx","messages":[...],"compaction":{...}}
```

**加载优先级**：完整 JSON (有 messages 键) → JSONL 逐行解析

### MCP 层序列化
- `JsonRpcId`: `#[serde(untagged)]`
- `McpToolCallContent`: `#[serde(flatten)]`
- 大部分字段: `#[serde(rename_all = "camelCase")]`
- `_meta`: `#[serde(rename = "_meta")]` 保留协议规范键名

---

## 五、错误类型体系

**ApiError** (13 个变体):
| 变体 | 说明 |
|------|------|
| `MissingCredentials` | 凭证缺失（含环境变量提示） |
| `ContextWindowExceeded` | 上下文超限 |
| `ExpiredOAuthToken` | OAuth 过期 |
| `Auth` | 通用认证错误 |
| `InvalidApiKeyEnv` | API Key 环境变量解析失败（包装 `std::env::VarError`） |
| `Http` | HTTP 传输层 |
| `Io` | I/O 错误 |
| `Json` | JSON 解析（含 200 字符截断） |
| `Api` | 完整 API 错误（状态码/类型/消息/请求 ID） |
| `RetriesExhausted` | 重试耗尽（自引用） |
| `InvalidSseFrame` | 无效 SSE 帧 |
| `BackoffOverflow` | 指数退避计算溢出 |
| `RequestBodySizeExceeded` | 请求体过大 |

**分类方法**: `is_retryable()` / `is_context_window_failure()` / `safe_failure_class()` (返回 provider_auth/context_window/rate_limit/internal/transport/runtime_io/request_size/retry_exhausted)
<!-- @end-section -->

<!-- @section: related -->
## 相关文档

- [[01-overview|项目架构总览]]
- [[02-rust-crates-analysis|Rust Crate 功能模块分析]]
- [[05-architecture-flows|架构流程分析]]
<!-- @end-section -->
