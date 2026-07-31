---
id: "deepthinking-interop-009"
title: "Agent 互操作协议实现现状深度分析：A2A / ACP / MCP"
aliases: ["互操作现状", "A2A ACP MCP 现状", "agent interop status", "interop deepthinking"]
type: "deepthinking"
category: "design/deepthinking"
tags: ["agent-interop", "a2a", "acp", "mcp", "protocol", "status", "deep-design", "hermes-reference"]
version: "1.0.0"
created: "2026-07-30"
updated: "2026-07-30"
author: "jxncyjq"
status: "draft"
parent: null
children: []
related_docs:
  - id: "design-agent-interop-protocols-001"
    relation: "references"
    path: "../architecture/agent_components/agent-interop-protocols-comparison.md"
  - id: "deepthinking-collaboration-005"
    relation: "related_to"
    path: "./05-multi-agent-collaboration.md"
  - id: "spec-agent-coordinator-002"
    relation: "references"
    path: "../architecture/agent_components/agent-coordinator-spec.md"
---

# Agent 互操作协议实现现状深度分析：A2A / ACP / MCP

<!-- @section: overview -->
## 概述

盘点三个 agent 互操作协议在 **Legion** 与参考实现 **hermes-agent** 中的真实落地现状（含 file:line 取证），回答「谁实现了什么、缺什么、能抄谁」。配套决策文档见 [[design-agent-interop-protocols-001|A2A vs ACP vs MCP 落地成本对比]]。

**三协议方向不同（反复强调，避免混淆）：**
- **A2A**（Agent2Agent）：agent ↔ agent，跨厂商网状互操作。
- **ACP**（Agent Client Protocol）：客户端/编辑器 ↔ agent（Zed/Cursor 驱动 agent）。
- **MCP**（Model Context Protocol）：agent ↔ 工具/数据（挂工具，或把自己暴露成工具）。

**一句话现状：**

| 协议 | Legion | hermes-agent |
|---|---|---|
| A2A | ❌ 无 | ❌ 无 |
| ACP | ❌ 无 | ✅ 完整（`acp_adapter/`） |
| MCP | ❌ 无 | ✅ 完整（`mcp_serve.py`） |

Legion 的「多 agent」是**进程内编排**（非标准协议）；hermes 覆盖 ACP+MCP，**都不做 A2A**。
<!-- @end-section -->

<!-- @section: a2a -->
## 1. A2A — 无人实现

### 是什么
Google→Linux Foundation 的跨厂商 agent 互操作标准：HTTP + JSON-RPC 2.0 + SSE；**Agent Card** 发现（`/.well-known/agent-card.json`，含 url/skills/capabilities/securitySchemes）；核心方法 `message/send`·`message/stream`·`tasks/get`·`tasks/cancel`·`tasks/pushNotificationConfig`；Task 生命周期（submitted/working/input-required/completed/failed/canceled）；Message/Part(text/file/data)/Artifact 对象；SSE 事件 `TaskStatusUpdateEvent`/`TaskArtifactUpdateEvent`；OIDC securitySchemes；**opaque agents**（不暴露内部工具/逻辑）。

### Legion 现状：全缺
| A2A 要件 | Legion | 证据 |
|---|---|---|
| JSON-RPC 2.0 | ❌ 自定义 REST（path+method switch） | `internal/server/http.go:248+` |
| Agent Card `/.well-known/...` | ❌ `GET /v1/agents` 只返回名字 `[]string` | `http.go:270` |
| 标准方法名 | ❌ `POST /v1/tasks`·`GET /v1/tasks/{id}`·`/result` | `http.go:272-280` |
| Task 状态集 | ⚠️ pending/assigned/running/quality_review/done/failed/suspended（无 input-required/canceled） | `domain/types.go:17-23` |
| Message/Part | ❌ `task.Input` 纯 string，images 单列 | `domain/types.go` |
| Artifact 对象 | ❌ `Artifact` 字段是 agent_message 自由文本（撞名不撞义） | `domain/types.go:207` |
| SSE A2A 事件 | ⚠️ `GET /v1/events` 有流但 schema 是自有 `RuntimeEvent` | `http.go:248` |
| OIDC securitySchemes | ❌ company_id header 式 | — |

grep `a2a`/`agent-card`/`jsonrpc`/`message/send` 全空（命中的 `Artifact` 是 Legion 内部字段）。

### hermes 现状：也无
grep 全为 package-lock/CSS/md 偶然字串，无真实现。→ **A2A 无任何可抄参考**，要做只能照官方 spec 从零建 gateway。
<!-- @end-section -->

<!-- @section: acp -->
## 2. ACP — hermes 完整实现，Legion 无

### 是什么
Agent Client Protocol（Zed 等编辑器 ↔ agent）：**JSON-RPC 2.0 over stdio**。编辑器发起 session、prompt，agent 流式回；内建权限/编辑审批。

### hermes 实现（干净参考，全 file:line 取证）
- 入口：`acp_adapter/entry.py:265` `asyncio.run(acp.run_agent(agent, use_unstable_protocol=True))`
- Agent 类：`acp_adapter/server.py:565` `class HermesACPAgent(acp.Agent)`
- **方法处理器**（`acp_adapter/server.py`）：
  - `initialize()` (1042) → `InitializeResponse`，声明 capabilities
  - `authenticate(method_id)` (1076)
  - `new_session()` (1338) / `load_session()` (1358，从 DB 恢复) / `resume_session()` (1405) / `fork_session()` (1461) / `list_sessions()` (1481) / `cancel(session_id)` (1440)
  - `prompt(...)` (1528) → 流式 `PromptResponse`
  - `set_session_model(model_id, session_id)` (2337) / `set_session_mode(mode_id, session_id)` (2371，编辑审批策略)
- **capabilities 声明**（`server.py:1061-1074`）：`load_session=True`、`prompt_capabilities.image=True`、`session_capabilities`(fork/list/resume)、`auth_methods`。
- **工具→ACP ToolKind 映射**（`acp_adapter/tools.py:24-59`）：read_file→read、write_file→edit、terminal→execute、browser_navigate→fetch、`_thinking`→think…
- **会话持久化**（`acp_adapter/session.py`）：落 `~/.hermes/state.db`，重启 `load_session` 重建全历史。
- **流式回调**（`acp_adapter/events.py`）：`make_message_cb`/`make_thinking_cb`/`make_step_cb`/`make_tool_progress_cb`，经 `conn.session_update()` + `run_coroutine_threadsafe`（AI 在 worker 线程、ACP 事件循环在主线程）。
- **权限/编辑审批**（`acp_adapter/permissions.py:18-27` + `edit_approval.py:25-36`）：allow_once/session/always/deny 映射；`EditProposal{tool_name,path,old_text,new_text,arguments}`；经 `ContextVar` 绑定，仅 ACP 会话触发，CLI/gateway 旁路。

### Legion 现状：无 ACP，但资产高度契合
Legion 已有 ACP 所需内核，缺的是「JSON-RPC/stdio 包壳」：
| ACP 要件 | Legion 对应资产 | 证据 |
|---|---|---|
| session new/load/resume/list | SessionStore + sqlite 跨轮历史 | `server/http.go:54`,`storage/sqlite.go` |
| prompt 流式 | `GET /v1/events` token 流 | `http.go:248` |
| 权限/编辑审批 | `manualgate` 人工审批网关 | `internal/manualgate/` |
| capabilities/工具集 | Registry + 每 agent 工具/模型 | `agentregistry/` |
→ 契合度最高；移植成本「中」，直接抄 hermes 结构。
<!-- @end-section -->

<!-- @section: mcp -->
## 3. MCP — hermes 完整实现，Legion 无

### 是什么
Model Context Protocol（Anthropic）：把能力/数据暴露成**工具**给 MCP 客户端（Claude Code/Cursor/Codex）。JSON-RPC（stdio / HTTP+SSE）；`initialize`·`tools/list`·`tools/call`·`resources/*`·`prompts/*`。

### hermes 实现
- `mcp_serve.py:1-50`：stdio MCP server，把「消息会话」暴露成工具。
- 工具：`conversations_list`·`conversation_get`·`messages_read`·`attachments_fetch`·`events_poll`·`events_wait`·`messages_send`·`permissions_list_open`·`permissions_respond`·`channels_list`。
- 用途 = **把 hermes 暴露成工具**给别的 agent 消费（非消费外部工具）。

### Legion 现状：无 MCP（server 或 client 均无）
- go.mod 无任何 MCP SDK；代码零 mcp 痕迹。
- 若做：新 stdio MCP server 把 Legion 的 `SessionStore`/`TaskStore` 暴露成工具（sessions_list/session_get/turns_read/task_submit/task_get/messages_send…），仿 hermes `mcp_serve.py`。可调现有服务层（`server/http.go:30 TaskStore`,`:54 SessionStore`）。
<!-- @end-section -->

<!-- @section: legion-native -->
## 4. Legion 的原生「多 agent」= 进程内编排（非标准协议）

Legion 不经任何标准互操作协议，而是**单进程内**编排：
- **Coordinator 路由**：按 `task.AgentID` resolve 注册 agent（`runtime/coordinator.go:203`）或走 default-agent。
- **委派**：`delegate_task` 工具（`runtime/delegation_tool.go:39`）——orchestrator **克隆自己**成子 runtime（`delegation.go:71 newSubRuntime`，共用同一 maas/tools，`agent_id` 仅标签），单/批量/后台，`maxSpawnDepth=2`/`maxConcurrent=3`。
- **agent 间消息**：`send_message`/`read_messages` 工具（`tool/agent_message.go:22`）+ `agent_messages` 表（内部便签，非协议）。
- **GUI**：AgentSelector 选注册 agent 跑主对话（`SubmitTask(agent_id)`）；`/send` 手工发消息。

→ 本质：**单厂商单进程编排**，非跨进程/跨组织互操作。与 A2A（跨厂商网状）目标层根本不同；与 ACP（客户端驱动）/MCP（暴露工具）是**互补**而非替代——Legion 有内核能力，缺的是把它**标准化对外暴露**的那层。
<!-- @end-section -->

<!-- @section: synthesis -->
## 5. 综合结论

1. **A2A**：Legion 无、hermes 无、无参考。跨厂商联邦愿景大但落地早期；除非有明确跨组织互操作需求，暂缓。
2. **ACP**：Legion 无、**hermes 有干净参考**、**Legion 资产契合度最高**（session/流式/审批现成）。想让编辑器/客户端驱动 Legion → 首选、性价比最高。
3. **MCP**：Legion 无、**hermes 有参考**、生态最成熟、独立成本小。想把 Legion 暴露成工具给别的 agent → 走这个。
4. **Legion 原生编排**与上述三者**互补**：三个协议都是「对外标准化暴露/互操作」层，Legion 内核（编排/会话/审批）是它们的下游被包装对象。

**演进建议**：先 ACP 或 MCP 验证一条对外互操作链（复用内核，加薄适配层），A2A 待真实跨厂商场景再上；三者不互斥可并存（对内客户端 ACP、对外工具 MCP、对外 agent A2A）。
<!-- @end-section -->

## 相关文档

- [[design-agent-interop-protocols-001|A2A vs ACP vs MCP 落地成本对比]] — 配套决策文档（对比表 + 成本档 + 决策树）
- [[deepthinking-collaboration-005|多智能体协作深度设计]] — Legion 原生进程内编排的设计背景
- [[spec-agent-coordinator-002|AgentCoordinator 组件规范]] — 任务路由 / 委派内核
