---
id: "analysis-pi-architecture-002"
title: "pi-agent 运行时架构与 Agent 主循环"
aliases: ["pi runtime", "pi agent-loop", "pi AgentSession"]
type: "design"
category: "design/analysis/pi-agent"
tags: ["pi-agent", "analysis", "agent-loop", "runtime", "session"]
version: "1.0.0"
created: "2026-08-15"
updated: "2026-08-15"
author: "jxncyjq"
status: "review"
parent: "analysis-pi-index"
children: []
related_docs:
  - id: "analysis-pi-overview-001"
    relation: "extends"
    path: "./01-overview.md"
  - id: "analysis-pi-tools-004"
    relation: "references"
    path: "./04-tools.md"
---

# pi-agent 运行时架构与 Agent 主循环

<!-- @section: layers -->
## 三层运行时

```
AgentSession (coding-agent/src/core/agent-session.ts, 3344 行)
   │  会话文件、工具注册表、扩展 runner、system prompt、压缩、模型切换、队列
   ▼
Agent (agent/src/agent.ts, 592 行)
   │  持有 state(systemPrompt/model/tools/messages)、事件订阅、steer/followUp 队列、abort
   ▼
runAgentLoop (agent/src/agent-loop.ts, 796 行)
   │  纯函数式循环：流式请求 → 工具批执行 → turn 结算 → 队列轮询
   ▼
streamFn (pi-ai Models.streamSimple)
```

三层的职责切分很干净：**loop 不持状态，Agent 持状态但不懂业务，AgentSession 懂业务但不碰协议**。
<!-- @end-section -->

<!-- @section: message-model -->
## 消息模型：AgentMessage 与双次变换

内核不直接用 LLM 的 `Message`，而是用 `AgentMessage = Message | CustomAgentMessages[keyof ...]`——应用侧通过 **TypeScript declaration merging** 往联合类型里塞自定义消息（`coding-agent` 就是这样加入 `CustomMessage` 的）。

每次请求前做两次变换（`agent/README.md` 明确画出）：

```
AgentMessage[] --transformContext()--> AgentMessage[] --convertToLlm()--> Message[] --> LLM
                   (可选，裁剪/注入)          (必需，过滤 UI-only 消息)
```

两个回调的契约都写死在类型注释里：**不得抛异常**，必须返回安全兜底值，否则会打断底层循环、事件序列不完整。这是把「扩展点不可信」这件事写进类型契约的做法。

`AgentToolResult.addedToolNames` 是一个精巧的小设计：工具执行后可以声明「从此刻起新增了哪些工具」，这条信息写进 `ToolResultMessage`，于是**会话回放时能还原当时可用的工具集**（扩展动态注册工具的场景，见 [[analysis-pi-tools-004]]）。
<!-- @end-section -->

<!-- @section: loop -->
## 主循环 runLoop 的精确语义

`agent-loop.ts` 的 `runLoop` 是双层循环：

```
外层 while(true):                       # 处理 followUp 队列
  内层 while(hasMoreToolCalls || pending):
    turn_start
    注入 pendingMessages（steer）
    streamAssistantResponse()           # 流式，边流边更新 context 最后一条
    if stopReason in (error, aborted): turn_end + agent_end, return
    toolCalls = message.content.filter(toolCall)
    if stopReason == "length": 全批失败（参数可能被截断）
    else: executeToolCalls()            # sequential 或 parallel
    turn_end
    prepareNextTurn()                   # 可替换 context/model/thinkingLevel
    if shouldStopAfterTurn(): agent_end, return
    pendingMessages = getSteeringMessages()
  followUp = getFollowUpMessages(); if 非空 continue; else break
agent_end
```

几个值得记的细节：

1. **`stopReason === "length"` 全批拒绝执行**。理由写在注释里：流式工具参数用「尽力 JSON 抢救解析」收尾，被输出上限截断的调用可能**解析成功且 schema 校验通过，但内容静默残缺**。于是全部返回错误结果让模型重发。这是一条真实事故驱动的规则。
2. **steering vs followUp**。steering 在「本轮工具执行完、下一次请求前」注入；followUp 在「Agent 本来要停」时注入并继续。两个队列各有 `QueueMode`（`"all"` 一次全放 / `"one-at-a-time"` 每次一条，默认后者）。
3. **`terminate` 的合取语义**：`shouldTerminateToolBatch()` 要求**批内每一个** finalized 结果都 `terminate === true` 才提前结束。单个工具想终止运行必须整批同意——避免并行批里一个工具偷偷掐断其他工具的结果。
4. **abort 后仍然产出完整事件序列**：`Agent.handleRunFailure()` 手工补发 `message_start/message_end/turn_end/agent_end`，保证 UI 状态机不会卡住。
5. **`agent_end` 不等于 idle**：注释明说 `agent_end` 只表示「不再有 loop 事件」，run 真正结束要等所有 `await` 的监听器结算完 `finishRun()`。
<!-- @end-section -->

<!-- @section: tool-exec -->
## 工具批执行：并行的三段式

默认 `toolExecution: "parallel"`。但「并行」是有限并行：

| 阶段 | 顺序 |
|------|------|
| prepare（查表 / prepareArguments / schema 校验 / `beforeToolCall`） | **严格源序、串行** |
| execute（`tool.execute`） | **并发** |
| `tool_execution_end` 事件 | 完成序 |
| 工具结果消息（`message_start/end`） | **回到源序** |

任一被调用工具声明 `executionMode: "sequential"`，或全局配置为 sequential，则整批退回串行。

这个「preflight 串行、执行并发、消息回源序」的三段式，解决的是「并行执行但会话记录必须确定性」的矛盾——Legion 若做并行工具执行，这是可以直接照抄的排序规则。
<!-- @end-section -->

<!-- @section: session -->
## AgentSession：把内核接进产品

`AgentSession` 在构造时把自己的能力**装到 `Agent` 的回调字段上**，而不是继承或包装：

```ts
// _installAgentToolHooks()：回调内部每次都读 this._extensionRunner
this.agent.beforeToolCall = async ({ toolCall, args }) => runner.emitToolCall({...});
this.agent.afterToolCall  = async ({ ... }) => { /* emitToolResult + 图片归一化 */ };
```

注释点明了动机：**回调在执行时才读 `this._extensionRunner`，所以 `/reload` 换掉 runner 不需要重装 hook**。热重载的正确姿势。

`_installAgentNextTurnRefresh()` 则在每个 turn 之间把 systemPrompt、tools、model、thinkingLevel 从 session 状态**重新灌回** loop：这就是「运行中 `/model` 切换模型、扩展动态注册工具即刻生效」的实现原理。

AgentSession 还负责：会话文件（`SessionManager`，支持树形分支/fork/navigate）、压缩（threshold/overflow/manual 三种触发）、模型解析与鉴权（`ModelRuntime`/`ModelRegistry`）、导出 HTML、slash 命令聚合、resource 重载。
<!-- @end-section -->

<!-- @section: session-tree -->
## 会话是树，不是线

`SessionManager` 的条目模型是**树**：普通消息、压缩条目（compaction）、分支摘要（branchSummary）、自定义条目（custom entry，不进 LLM 上下文，仅 UI 持久化）。

由此派生的产品能力：

- `/fork`：从某条目分叉出新会话文件
- `/tree` 导航：跳到树上另一点，可选生成「分支摘要」把被离开的分支压成一条
- 压缩：把前缀折成 compaction 条目 + `retainedTail`

这套模型在下一代 `AgentHarness` 里被正式化为「entry tree + lanes + facts + usage ledger」四件套，见 [[analysis-pi-harness-003]]。
<!-- @end-section -->

<!-- @section: remote -->
## 远程接入面

- `packages/protocol`：协议版本 1，帧格式 = 4 字节大端长度 + 一个 definite-length CBOR item；首条消息必须是 `hello`。**Session/Server snapshot 是权威状态，progress 事件只是瞬时 UI 提示、禁止归约进权威状态**——这条规则值得记，是分布式 UI 一致性的常见坑。
- `packages/client`：`PiClient` 只依赖抽象的 `ByteTransport`，零 Node 依赖，可跑浏览器。
- `packages/server`：`PiServer` + `createUnixServer`，服务实现方只需提供 `listSessions/listModels/createSession/openSession`。
- RPC 模式另有一条更简单的 JSONL 通道（严格 `\n` 分帧，文档特别警告不要用 Node `readline`，因为它会在 Unicode 分隔符处切行）。
<!-- @end-section -->

## 相关文档

- [[analysis-pi-overview-001|项目架构总览]]
- [[analysis-pi-harness-003|Harness 分层与下一代规范]]
- [[analysis-pi-tools-004|工具系统与装载机制]]
- [[analysis-pi-extensions-006|扩展系统与 No-MCP 立场]]
