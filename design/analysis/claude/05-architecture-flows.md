---
id: "analysis-clawcode-flows-005"
title: "Claw Code 关键架构流程分析"
aliases: ["架构流程", "claw-code flows", "对话循环", "权限流程"]
type: "analysis"
category: "design/analysis/claude"
tags: ["claw-code", "architecture", "conversation", "permissions", "compact", "sse"]
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
  - id: "analysis-clawcode-rust-002"
    relation: "references"
    path: "./02-rust-crates-analysis.md"
  - id: "analysis-clawcode-datamodel-006"
    relation: "related_to"
    path: "./06-data-models.md"
---

<!-- @section: overview -->
# Claw Code 关键架构流程分析

> 深入分析 claw-code 运行时的核心流程：对话循环、工具调度、权限检查、会话压缩和 token 预算管理。

---

## 一、系统架构层次

> 注：下图中 `LiveCli` 定义在 `rusty-claude-cli/src/main.rs`（CLI 二进制内私有结构，不是 `runtime` crate 的公共 API）；`ConversationRuntime` 才是 runtime crate 中真正可复用的对话状态机入口。

```
main.rs (run/run_repl)    ── CLI 入口，REPL 循环
    │
    ▼
LiveCli (CLI 内部)         ── 会话管理，顶层 turn 执行
    │
    ▼
ConversationRuntime        ── 核心对话状态机
    │         │           │
    ▼         ▼           ▼
ApiClient  ToolExecutor  PermissionEnforcer
(API调用)  (工具+插件)    (多层权限)
    │
    ▼
api crate ── AnthropicClient / OpenAICompat / SSE 解析
    │
    ▼
Session ── 持久化、消息管理、Compact 压缩
```

核心数据流：`用户输入 → LiveCli.run_turn() → ConversationRuntime.run_turn() → 自动压缩 → API 流式调用 → SSE 解析 → 构建助手消息 → 工具调用循环 → 权限检查 → Pre/Post Hook → 结果回传 → 再次压缩 → TurnSummary`

---

## 二、对话循环完整状态机

实现在 `conversation.rs` 的 `ConversationRuntime::run_turn()`。

### 2.1 状态机流程

```
Idle
  │
  ▼ (用户输入到达)
TurnStarted
  │ 记录 turn 启动遥测
  │ 检查 compaction health
  │ push_user_text 到 session
  ▼
ApiRequesting (循环入口)
  │ 组装 ApiRequest{ system_prompt, messages }
  │ api_client.stream(request)
  │ 收集 AssistantEvent 流
  ▼
ProcessingResponse
  │ build_assistant_message(events) → (message, usage, cache_events)
  │ 记录 usage 到 UsageTracker
  │ push assistant_message 到 session
  │ 提取 pending_tool_uses = [(id, name, input), ...]
  ▼
  ├─ pending_tool_uses 为空 → [结束 Turn]
  │     │
  │     ▼
  │   AutoCompacting
  │     │ maybe_auto_compact(): 检查 input_tokens 阈值
  │     │ 若触发: compact_session(), 替换 self.session
  │     ▼
  │   TurnSummary 输出
  │
  └─ pending_tool_uses 非空 → [工具执行循环]
        │
        ▼
      ToolExecuting (for each tool_use)
        │ run_pre_tool_use_hook(tool_name, input)
        │ 若 pre_hook cancelled/denied → Deny 结果
        │ permission_policy.authorize_with_context()
        │   Prompt 模式 + prompter: 交互式询问
        │   否则基于 policy 自动判定
        │ 若 Allow: tool_executor.execute() → post_hook
        │ push tool_result 到 session
        ▼
      回到 ApiRequesting (下一轮迭代)
```

### 2.2 关键步骤详解

**步骤 1 — 用户输入入栈**
```rust
session.push_user_text(user_input)
// 同时增量写入 JSONL 持久化
```

**步骤 2 — API 请求组装**
```rust
let request = ApiRequest {
    system_prompt: self.system_prompt.clone(),
    messages: self.session.messages.clone(),
};
```

**步骤 3 — 流式事件收集**
```rust
let events = self.api_client.stream(request)
// 返回 Vec<AssistantEvent>: TextDelta / ToolUse / Usage / PromptCache / MessageStop
```

**步骤 4 — 构建助手消息**
`build_assistant_message(events)` 将流事件拼接：
- 累积 TextDelta 为文本块
- 遇到 ToolUse 时 flush 当前文本，新建 ToolUse 块
- 必须收到 MessageStop，否则报错
- 必须产生至少一个 ContentBlock，否则报错

**步骤 5 — 工具调用循环**
对每个 `(tool_use_id, tool_name, input)`：
1. `run_pre_tool_use_hook` — PreToolUse 钩子检查
2. 若 hook cancelled/denied → 直接返回 Deny 结果
3. `permission_policy.authorize_with_context` — 权限判定
4. 若 Allow → `tool_executor.execute(&tool_name, &effective_input)`
5. `merge_hook_feedback` — 合并 hook 消息
6. post_tool_use_hook / post_tool_use_failure_hook
7. push tool_result 到 session

**步骤 6 — 限流保护**
```rust
if iterations > self.max_iterations {
    // 返回错误，防止无限循环
}
```

---

## 三、工具调用 Dispatch 链

### 3.1 两层 Dispatch

**第一层 — 运行时层 (conversation.rs)**
```
API 响应 → AssistantEvent::ToolUse{id, name, input}
  → run_pre_tool_use_hook()        [Hook 系统]
  → permission_policy.authorize()  [权限策略]
  → tool_executor.execute()        [ToolExecutor trait]
  → run_post_tool_use_hook()       [Hook 系统]
  → ConversationMessage::tool_result()
```

**第二层 — 工具执行层 (tools/lib.rs)**
```
GlobalToolRegistry::execute(name, input)
  → mvp_tool_specs() 内置工具查找
  → execute_tool_with_enforcer()
     → match name:
        "bash"       → classify_bash_permission() → run_bash()
        "read_file"  → maybe_enforce → run_read_file()
        "write_file" → maybe_enforce → run_write_file()
        "edit_file"  → run_edit_file()
        "glob_search"→ run_glob_search()
        "grep_search"→ run_grep_search()
        ...
```

### 3.2 Bash 命令动态权限分级
```rust
fn classify_bash_permission(command: &str) -> PermissionMode {
    // 只读命令 (cat, grep, ls, find, ...) → WorkspaceWrite
    // 非只读命令 (rm, mv, ...) 或危险路径 → DangerFullAccess
}
```

### 3.3 SSE 流解析链 (API 层)
```
原始字节流
  → SseParser.push(chunk)
    → next_frame(): 按 \n\n 或 \r\n\r\n 分帧
    → parse_frame(): 解析 event/data 行，跳过 ping/[DONE]
    → JSON 反序列化为 StreamEvent
  → Vec<StreamEvent>
```

### 3.4 AssistantEvent 转换桥 (main.rs)
```
StreamEvent                      → AssistantEvent
ContentBlockStart(text)          → 开始累积 TextDelta
ContentBlockDelta(TextDelta)     → TextDelta + 实时 Markdown 渲染
ContentBlockStart(tool_use)      → 暂存 pending_tool
ContentBlockDelta(input_json)    → 合并到 pending_tool.input
ContentBlockStart 完成            → flush → ToolUse{id,name,input}
MessageDelta(usage)              → Usage + PromptCache 事件
MessageStop                      → MessageStop
```

---

## 四、权限检查三层架构

| 层级 | 位置 | 检查内容 | 失败后果 |
|------|------|----------|----------|
| **Pre-Hook** | conversation.rs | 用户自定义 PreToolUse 脚本 | tool_result 为 error，不执行 |
| **Policy Auth** | conversation.rs | 权限模式 + prompter 交互 | tool_result 为 error Deny |
| **Enforcer** | tools/lib.rs | 模式匹配 + 文件边界检查 | 返回 Err(String) 拒绝 |

### 4.1 Pre-Tool-Use Hook
```rust
let pre_hook_result = self.run_pre_tool_use_hook(&tool_name, &input);
```
- Hook 可返回 cancelled / denied / failed / allowed
- Hook 可覆盖 input (updated_input) 和 permission_override
- cancelled/denied 直接短路，工具不执行

### 4.2 PermissionPolicy 判定
```rust
self.permission_policy.authorize_with_context(&tool_name, &input, &permission_context, Some(prompter))
```
- 检查 PermissionContext 是否有 hook 覆盖
- actve_mode == Prompt 且 prompter 存在 → 交互确认
- 否则基于静态规则判定
- 返回 `PermissionOutcome::Allow` 或 `Deny{reason}`

### 4.3 PermissionEnforcer 文件级检查
```rust
PermissionEnforcer::check():
  Prompt 模式 → 自动 Allow（由层级 2 的 prompter 处理）
  ReadOnly 模式 → write_file 拒绝
  WorkspaceWrite 模式 → 拒绝 workspace 外的路径
```

---

## 五、会话压缩完整流程

### 5.1 触发条件

```rust
fn maybe_auto_compact(&mut self) -> Option<AutoCompactionEvent> {
    // 条件 1: 累计 input_tokens 超过阈值 (默认 100,000)
    if cumulative_usage().input_tokens < auto_compaction_input_tokens_threshold {
        return None;
    }
    // 条件 2: compact_session 确实有消息被移除
    // 条件 3: removed_message_count > 0
}
```
- 阈值可通过 `CLAUDE_CODE_AUTO_COMPACT_INPUT_TOKENS` 环境变量自定义
- 触发时机：每次 `run_turn()` 结束前

### 5.2 压缩执行步骤

**步骤 1 — 预检查**：若不需要压缩直接返回

**步骤 2 — 计算边界**：
```
compacted_prefix_len = 上次 compaction summary 的起始位置
preserve_recent = 默认保留最近 4 条
keep_from = messages.len() - preserve_recent
```

**步骤 3 — 保护 ToolUse/ToolResult 配对**：
- 若保留部分第一条消息是 ToolResult，向前查找配对 ToolUse
- 将边界前移到配对 ToolUse 之前
- 保证不切断 assistant(ToolUse) + user(ToolResult) 配对

**步骤 4 — 生成摘要**：
`summarize_messages(removed)` 从被移除消息提取：
- 消息统计: user N, assistant M, tool K
- 工具名称列表
- 最近 3 个用户请求摘要
- 待处理工作 (包含 "todo"/"next"/"pending" 的消息)
- 关键文件引用 (.rs/.ts/.tsx/.js/.json/.md)
- 关键时间线 (每条消息单行摘要，截断到 160 字符)

**步骤 5 — 构建 continuation 消息**：
```
System: "This session is being continued from a previous conversation..."
  + formatted_summary
  + "Recent messages are preserved verbatim."
  + "Continue the conversation from where it left off..."
```

**步骤 6 — 重组 session**：
```rust
compacted_messages = vec![system_message]  // 摘要作为第一条
compacted_messages.extend(preserved)       // 追加保留消息
session.messages = compacted_messages
session.record_compaction(summary, removed.len())
```

### 5.3 多层压缩合并

再次压缩时，`merge_compact_summaries` 保留：
- "Previously compacted context" 区域
- "Newly compacted context" 区域

### 5.4 摘要文本二次压缩 (summary_compression.rs)

**行规范化**：折叠空格、截断每行到 160 字符、按小写去重

**优先级选择**：
- 优先级 0: 核心细节行（"Scope:"、"Current work:"、"Pending work:" 等）
- 优先级 1: 段落标题（以冒号结尾）
- 优先级 2: 列表项（"- " 开头）
- 优先级 3: 其余行

**预算控制**：max_chars 默认 1200，max_lines 默认 24

---

## 六、Token 预算管理

### 6.1 UsageTracker 累计追踪

```rust
// 每次 API 响应后
self.usage_tracker.record(usage);
```
跟踪：input_tokens / output_tokens / cache_creation_input_tokens / cache_read_input_tokens

### 6.2 指令文件截断
```
MAX_INSTRUCTION_FILE_CHARS: usize = 4_000   // 每文件上限
MAX_TOTAL_INSTRUCTION_CHARS: usize = 12_000  // 总计上限
```
超出后追加 "_Additional instruction content omitted..."

### 6.3 模型 Token 上限
```rust
fn max_tokens_for_model(model: &str) -> u32 {
    // opus → 32,000，其他 → 64,000
}
```

### 6.4 自动压缩阈值
```
DEFAULT_AUTO_COMPACTION_INPUT_TOKENS_THRESHOLD = 100,000
可自定义: CLAUDE_CODE_AUTO_COMPACT_INPUT_TOKENS
```

### 6.5 Prompt Cache 断裂检测

检测 token_drop 特征，生成 `PromptCacheEvent`：
```rust
pub struct PromptCacheEvent {
    pub unexpected: bool,
    pub reason: String,
    pub previous_cache_read_input_tokens: u32,
    pub current_cache_read_input_tokens: u32,
    pub token_drop: u32,
}
```

---

## 七、REPL 主循环

```
loop {
    editor.read_line() → ReadOutcome
    | Submit(input):
    |   > "/exit" | "/quit": persist_session(), break
    |   > SlashCommand::parse(): handle_repl_command()
    |   > bare-word skill match: run_turn(skill_prompt)
    |   > 普通文本: run_turn(&trimmed)
    |   > 每次后 persist_session()
    | Cancel: 继续循环
    | Exit: persist_session(), break
}
```

---

## 八、Session 持久化策略

### 格式
- **JSONL**（主流，增量追加）：`{type: "message", message: {...}}\n`
- **完整 JSON**（兼容）：`{version:1, session_id:..., messages:[...]}`

### 轮转
```
ROTATE_AFTER_BYTES = 256KB
MAX_ROTATED_FILES = 3
```

### 写入
- `write_atomic()`：先写临时文件再 rename
- `push_message()` 时增量追加 JSONL

### 兼容加载
```
先尝试完整 JSON 解析 → 成功且有 messages 键 → 使用
否则回退 JSONL 逐行解析
```
<!-- @end-section -->

<!-- @section: related -->
## 相关文档

- [[01-overview|项目架构总览]]
- [[02-rust-crates-analysis|Rust Crate 功能模块分析]]
- [[06-data-models|数据模型分析]]
<!-- @end-section -->
