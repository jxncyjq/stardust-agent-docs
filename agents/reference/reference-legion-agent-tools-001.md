---
id: "reference-legion-agent-tools-001"
title: "Legion Agent 工具能力"
aliases: ["Agent tools", "内置工具", "工具调用"]
type: "reference"
category: "agents/reference"
tags: ["agent", "tools", "taskledger", "message", "workspace"]
version: "1.1.0"
created: "2026-05-25"
updated: "2026-05-25"
author: "codex"
status: "published"
parent: "reference-legion-agent-user-manual-001"
children: []
related_docs:
  - id: "reference-legion-agent-config-context-001"
    relation: "related_to"
    path: "./reference-legion-agent-config-context-001.md"
  - id: "reference-legion-agent-tasks-md-001"
    relation: "related_to"
    path: "./reference-legion-agent-tasks-md-001.md"
---

# Legion Agent 工具能力

本文说明模型运行时可以调用的内置工具。普通用户不需要在 TUI 中手写 `read_file({...})` 这类伪调用；正确方式是用自然语言提出任务，由模型按工具 schema 发起工具调用。

## 工具执行链路

Legion Agent 的工具机制分为五步：

```text
Tool Descriptor + Handler
  -> Runtime exposes InferenceTool schema
  -> MaaS/OpenAI-compatible model returns tool_calls
  -> Registry executes real tool handler
  -> Runtime appends Tool results and asks model for final answer
```

对应代码职责：

| 环节 | 代码位置 | 说明 |
|------|----------|------|
| 工具注册 | `internal/tool.Registry` | 注册 `Descriptor` 和真实 `Handler` |
| 工具 schema 暴露 | `internal/runtime.Runtime.inferenceTools` | 把 descriptors 转成 MaaS `InferenceTool` |
| OpenAI-compatible 转换 | `internal/adapter.HTTPMaasClient` | 转成 `tools:function`，并解析返回的 `tool_calls` |
| 工具执行 | `internal/runtime.Runtime.executeToolCalls` | 发布事件、调用 Registry、收集结果 |
| 结果回填 | `promptWithToolResults` | 将工具结果拼入下一轮 prompt，让模型生成最终回答 |

当前实现是“工具结果回填 prompt”的兼容模式，而不是完整保留 OpenAI `assistant tool_calls` 和 `tool` role message 的消息数组模式。它已经能完成真实工具调用闭环，但如果后续要提高对不同模型服务的兼容性，可以演进为标准多消息 tool-calling 协议。

## Registry 执行管线

`Registry.Execute` 是工具安全边界的中心。一次工具调用会依次经过：

1. 查找工具 handler。
2. 按 `Descriptor.InputSchema` 校验必填参数和简单类型。
3. 检查角色权限。
4. 执行策略判断，高风险且未自动允许的工具会被拒绝。
5. 执行 guardrails。
6. 按工具 timeout 创建执行上下文。
7. 调用真实 handler。
8. 对结果执行 after guardrails。
9. 对输出做脱敏和控制字符清理。
10. 写入 audit。

这意味着“模型说自己调用了工具”和“真实工具被执行”不是一回事。只有模型返回结构化 `tool_calls`，并通过 Registry 执行成功，才算真实调用。

## 工作区文件工具

| 工具 | 参数 | 说明 |
|------|------|------|
| `list_files` | `directory` 可选 | 列出工作区内目录和文件 |
| `read_file` | `path` | 读取工作区内 UTF-8 文本文件 |
| `search_content` | `pattern`、`directory` 可选、`file_types` 可选 | 在工作区内搜索文本内容 |
| `write_file` | `path`、`content`、`overwrite` 可选 | 写入工作区文件；文件已存在时默认失败，需显式 `overwrite=true` |

路径可以是相对 `context_files.root` 的路径，也可以是仍位于工作区 root 内的绝对路径。越界路径会被拒绝。

只读运行时不会注册 `write_file`，适合 researcher 这类只读角色。

## 主 Agent 与子 Agent 的工具差异

| 运行时 | 默认工具 |
|--------|----------|
| 主 Agent | `read_file`、`search_content`、`list_files`、`write_file`、TaskLedger、AgentMessage |
| 子 Agent | `read_file`、`search_content`、`list_files`、TaskLedger、AgentMessage |

子 Agent resolver 当前默认使用只读 workspace registry，因此 `@researcher`、`@writer` 这类子 Agent 不会直接获得 `write_file`。如果需要写文件，推荐由 writer 产出内容后通过主 Agent 审阅写入，或后续引入按角色配置的工具权限。

## TaskLedger 工具

TaskLedger 工具用于多个 Agent 通过 `tasks.md` 协作：

| 工具 | 说明 |
|------|------|
| `create_task` | 创建共享任务条目 |
| `claim_task` | 声明某个 Agent 正在处理任务 |
| `update_task` | 更新任务状态、摘要、结果或产物 |
| `append_task_message` | 向任务追加协作消息 |
| `read_task` | 读取单个任务详情 |
| `rebuild_tasks` | 从结构化状态重建 `tasks.md` |

TaskLedger 适合可人工审阅的协作状态。任务很多时，应把长结果沉淀到 `docs/` 或 `memory/`，只在 `tasks.md` 中保留摘要和链接。

## AgentMessage 工具

AgentMessage 工具用于 Agent 间 inbox/outbox 消息：

| 工具 | 说明 |
|------|------|
| `send_message` | 向目标 Agent 发送消息，支持 `task_id`、`type`、`summary`、`artifact` |
| `read_messages` | 读取当前 Agent 的消息，可按状态过滤并标记已读 |

TUI 中的 `/send`、`/inbox`、`@agent --inbox` 会使用同一套消息能力。HTTP API 的 `/v1/agents/{agent_id}/messages` 也读写同一份消息数据。

## 调用与展示

- 工具调用由模型通过 OpenAI-compatible `tool_calls` 或运行时内部接口发起。
- TUI 会把工具执行过程作为事件记录，可通过 `/event` 查看。
- 大段工具输出应进入可滚动输出区或日志，不应在底部状态栏截断为最终答案。
- 伪工具调用文本会被当作普通模型输出处理；如果模型声称要调用工具但没有真实 tool call，应优先检查模型兼容性和工具 schema。
- `runtime.max_tool_rounds` 控制最多连续工具轮数，默认是 4；模型超过轮数仍要求工具时，任务会失败并提示超过限制。
- 工具结果会发布 `tool_result` 事件，然后进入下一轮模型推理，最终用户看到的是模型基于工具结果整理后的回答。

## 常见排查

| 现象 | 判断方向 |
|------|----------|
| 模型输出 `search_content({...})` 文本 | 这是伪工具调用，说明模型没有返回结构化 `tool_calls` |
| OpenAI-compatible 返回 schema 错误 | 检查工具 schema 是否有 `type: object`；当前适配器会为 nil schema 自动补 object |
| 子 Agent 不能写文件 | 子 Agent 默认只读，当前设计如此 |
| 工具结果没有进入最终回答 | 检查 `runtime.max_tool_rounds`、模型是否在第二轮基于 `Tool results` 回答 |
| TUI 看不到工具过程 | 用 `/event` 查看事件；工具结果也会写入日志或事件流 |
| 文件路径被拒绝 | 路径必须位于 `context_files.root`/ToolRoot 内 |

## 权限边界

当前内置工具主要面向 `developer:*` 能力域。实际安全边界包括：

- 工作区 root 限制。
- 只读 runtime 不包含 `write_file`。
- HTTP 层通过 Bearer token 和 RBAC 保护治理接口。
- 事件、诊断和 SSE 输出会脱敏敏感字段。
