---
id: "analysis-dsh-capabilities-004"
title: "DeepSeek Harness 核心能力"
aliases: ["dsh capabilities", "DeepSeek Harness 能力", "tool pipeline"]
type: "analysis"
category: "design/analysis/deepseek-harness"
tags: ["deepseek-harness", "tools", "sandbox", "subagent", "compaction", "session", "capability"]
version: "1.0.0"
created: "2026-08-15"
updated: "2026-08-15"
author: "jxncyjq"
status: "review"
parent: "analysis-dsh-index"
children: []
related_docs:
  - id: "analysis-dsh-plugin-system-003"
    relation: "extends"
    path: "./03-plugin-system.md"
  - id: "analysis-dsh-architecture-002"
    relation: "depends_on"
    path: "./02-architecture.md"
  - id: "analysis-dsh-insights-005"
    relation: "related_to"
    path: "./05-insights.md"
---

# DeepSeek Harness 核心能力

<!-- @section: overview -->
## 概述

本篇按能力域拆解 dsh 到底提供了什么。所有能力都以 [[analysis-dsh-plugin-system-003|capability seam]] 的形式存在，因此「能力清单」同时也是「可替换点清单」。

<!-- @end-section -->

<!-- @section: tool-pipeline -->
## 工具执行管线

这是 dsh 最工程化的一块。`ctx.tools.execute()` 接受调用方拥有的 `ToolExecutionInput`（必须带 readonly `signal`），把解析后的 JSON 参数一次性物化成管线拥有的 `ToolExecution`，然后走：

```mermaid
flowchart TD
  model["assistant 消息中的 tool-call 块"] --> call["session 事件 tool/call（执行前先落日志）"]
  call --> pre["tools/pre-execute waterfall<br/>hook、权限、沙箱"]
  pre -->|allow| guards["已注册单调守护<br/>deny 或弃权，identity 受保护"]
  pre -->|ask| approval["ctx.approval 一次性审批<br/>缺席或无法作答 = 拒绝"]
  pre -->|deny| denied["拒绝：跳过工具体"]
  approval -->|allowed-once| guards
  approval -->|rejected/cancelled/unavailable| denied
  guards -->|allow| around["tools/execute waterfall<br/>超时、重试、指标（around 派发）"]
  guards -->|deny| denied
  around --> body["工具 execute() 体"]
  body --> fsgate["fs/write-intent / fs/edit-intent<br/>仅 tool-fs 的变更"]
  body --> owned["工具自有 session 事件<br/>todo/write、fs/observed、hook/*、tool/code-dispatch"]
  body --> around
  around --> post["tools/post-execute waterfall<br/>接受 / 阻断 / 替换 / 追加上下文"]
  denied --> post
  post --> finalize["ToolDefinition.finalizeContent<br/>最后一道 content-only 不变式"]
  finalize --> final["tools/result 同步通知<br/>冻结的权威结果"]
  final --> result["session 事件 tool/result<br/>唯一面向模型的结果"]
  result --> ctxq["活动批次 additionalContexts FIFO<br/>在已记录结果之后注入 user/message"]
```

设计细节里有几条值得抄：

- **`tool/call` 在执行之前落日志**，所以「模型请求过什么」与「执行结果如何」是两条独立可重放的事实。
- **schema 允许清单**：注册表的 `schemas()` 用显式白名单构造模型可见的 `ToolSchema`——`output` / `execute` / `finalizeContent` / `timeoutMs` / `isConcurrencySafe` / `presentCall` / `presentResult` **永远不会泄漏进模型请求**。
- **强制 canonical output**：每个工具必须声明 `output.schema`（受支持的 JSON Schema 子集）+ 纯函数 `render(args, value)` 投影出模型内容。工具体只返回无损 JSON 值，渲染是可重放的纯投影。
- **UI 渲染意图是工具设计的一部分**，在设计期就要定（`generic` / `terminal` / `diff`、`locations`）。`presentCall(args)` / `presentResult(args, result)` 必须是纯函数——UI 在实时流式和日志回放两种场景都会调用它们。
- **并发安全是显式 opt-in**：`isConcurrencySafe(args)` 只有返回 `true` 才加入并行组；省略、抛错、非 `true` 一律视为独占。opt-in 的执行不得变更父级拥有的状态。
- **超时是协作式的**：`timeoutMs` 由 `dsh-tool-call-timeout-policy`（一个 `tools/execute` 包装器）执行，声明它等于断言该工具把 `exec.signal` 转发给了能响应中断的实现。
- **Code Mode**：注册表自己拥有保留的 `run_code` 传输。模型写的程序在 `ctx.codeRuntime`（worker-thread 后端）里跑，其中的子调用同样穿过完整管线——携带父 token、记 `tool/code-dispatch`、拒绝以 binding rejection 形式返回、且省略 `additionalContexts` 以保持 call/result 相邻。
- **失败归一化**：管线任一环抛错都被外层归一化成 `isError` 结果，`finalizeContent` 对**每个**归一化结果恰好调用一次（包括绕过 `post-execute` 的管线失败）。

<!-- @end-section -->

<!-- @section: tool-catalog -->
## 模型可见工具清单

`docs/tool-catalog.md` 是生成的完整目录。按能力域归类：

| 域 | 工具 |
|---|---|
| 文件 | `read` / `write` / `edit` / `read_image`、`glob` / `grep`、`str_replace_editor` |
| 执行 | `bash`（含 `run_in_background`）、`pwsh`、持久 `bash`（bash-persistent） |
| 终端 | `terminal_open` / `read` / `send` / `signal` / `list` / `close` |
| 后台任务 | `job_list` / `job_output` / `job_kill` |
| 委派 | `subagent`、`send_message` / `interrupt_agent` / `list_agents`、`report`、`ralph` |
| 编排 | `workflow`、`run_code`（Code Mode 传输） |
| 检索 | `web_search` / `web_fetch`、`lsp`、`session_search` / `session_trace` / `session_event_read` / `session_event_search` / `session_event_trace` |
| 规划与状态 | `todo_write`、`exit_plan_mode`、`create_goal` / `get_goal` / `update_goal`、`schedule_create` / `schedule_list` / `schedule_delete` |
| 知识 | `skill` |
| 人机交互 | `ask_user_question` |
| 自我修改 | `cordis_define` / `cordis_undefine` / `cordis_run` / `cordis_stop` / `cordis_inspect_list` / `cordis_inspect_query` / `cordis_inspect_self` |

<!-- @end-section -->

<!-- @section: execution -->
## 执行与隔离

四层叠起来，每层都是独立 seam：

```
tool-bash (Consumer)
  → ctx.shell (Service Definition)        执行器语义：run / start / 背景进程句柄
    → ctx.subprocess (seam)               进程坐标、进程树/会话生命周期、stdio 处置、kill 升级
    → ctx.sandbox (seam)                  把即将 spawn 的 argv 包进文件效果策略
      → sandbox-local                     Linux bwrap/Landlock · macOS Seatbelt · Windows ACL 受限令牌
```

- `ctx.shell` 每个上下文只允许一个实现，`run` 只在基础设施故障时 reject——非零退出、超时 kill、abort kill 都以 `ShellRunResult` 正常返回。
- **`SandboxMode` 只管文件效果**：`read-only`（只放行 `/dev/null` 之类必需 sink）、`workspace-write`（工作区根 + 后端承诺的临时区）、`danger-full-access`（绕过封禁，此时消费者根本不调用 `ctx.sandbox`）。网络与进程可见性**不在**这套词汇里——这个边界写得很克制。
- **强制程度是被上报的事实**：`full` vs `partial`。老 Landlock ABI、Windows ACL 的 Everyone/硬链接边界都是当前的 `partial` 案例，要求绝对边界的调用方必须自己拒绝或上抛这个差异。
- `workspaceRoot` 从调用会话的不可变 cwd 派生（无 agent 时才回落到部署配置），并且**先按文件系统语义规范化再做词法规范化**，所以 cwd 里含 `symlink/..` 也能指向进程真正运行的目录。
- `ctx.sandboxPolicy` 是「部署默认模式 + 工作区根」的唯一家；bash 执行器与 `fs-sandbox` **都**读它，因此 bash 和 fs 不可能封禁到不同的根。
- E2B 组是 POC：`ctx.e2b` 持有共享 SDK 句柄与远程工作目录，`fs-e2b` / `subprocess-e2b` 两个基础 provider 因此栖息在同一个 Linux 运行时里——这就是「换 provider 搬走整个执行世界」的示范。

<!-- @end-section -->

<!-- @section: delegation -->
## 委派与编排

### Subagent（`ctx.subagents`）

唯一一个**允许多 provider 并存**的 seam（按名注册，参照 LLM 适配器注册表而非单服务模式）。provider 覆盖面很宽：

`subagent-spawn-in-process`（新建子 agent）、`subagent-fork-in-process`（fork 当前会话）、`subagent-acp`、`subagent-codex`、`subagent-claude-code`、`subagent-dsh-sdk`。

两类能力，两种发现方式：

- **start-time 能力**（`outputSchema` / `depthLimit` / `toolFilter` / `persona`）在静态描述符上广告，服务在 `start()` **之前**检查；请求需要 provider 没有的能力时**响亮拒绝**（`SubagentError('UNSUPPORTED_CAPABILITY')`），绝不「接受后静默忽略」。
- **可续跑（continuable）子 agent** 由续跑管理器自己组合，因此以一个可选方法 `prepareContinuable` 的存在与否作为能力标志，用 TS 收窄做发现机制。

消费者分工：`tool-subagent`（按 provider 委派）、`tool-subagent-control`（全局 `send_message` / `interrupt_agent` / `list_agents`）、`tool-subagent-report`（子 agent 作用域内的 `report` 回传通道）。

### Workflow（`ctx.workflowEngine`）

让 agent 跑一段**模型编写的编排脚本**，脚本内的 `agent()` 调用扇出到 `ctx.subagents`。

- 每上下文一个引擎（无具名注册表），provider 是 `workflow-worker-thread`：一次运行一个 worker，脚本的 vm 上下文在 worker 内。
- `meta` 与 `args` 是**纯 JSON 数据**：引擎先按 schema 校验 `meta` 并在**执行任何脚本文本之前**响亮拒绝。
- `parent` 必填——脚本启动的每个子 agent 都归属它，cwd、lineage、depth 经 subagent seam 透传。
- 消费方可以选择引擎级 `subagentProvider` 覆盖与 per-run 的 `maxTotalAgents` 上限，但**脚本本身无法观察或替换这两个策略**。
- `ralph` 是在 workflow + subagent 原语之上组合出来的固定策略工具：一轮 = 一个全新子会话，子会话不接收父会话或前序子会话的对话种子，跨轮状态靠共享工作区 + 一份有界的结构化 handoff（status / summary / evidence / next steps / blocker）承载。

### Jobs（`ctx.jobs`）

通用后台任务运行时：后台 bash、PTY 发送、subagent 委派都作为生产者登记运行中的工作，`tool-jobs` 是模型可见的控制器（读 / 列 / 杀），`jobs-local` 是进程本地注册表。这样执行器不必知道会话，会话不必知道进程。

<!-- @end-section -->

<!-- @section: context -->
## 上下文治理

这是 harness 类项目最难的部分，dsh 拆成了四件事：

### 压缩（`ctx.compaction`）

- 触发点：`agent/pre-step` 上的压力检测（请求派生**之前**）与 `agent/request-error` 上的规范上下文溢出恢复。**没有**模型可见的 compact 工具；人类走 `dsh-command-compact`。
- 三个 log-only 事件 `compaction/start` / `summary` / `end` 记录锁、摘要、被选范围、被遮蔽的事件 seq、token 计数与模型调用；**摘要本身不新增 surface 事件类型**，而是骑在一条带 `surfaceOp: { op: 'replace', start, end }` 的 `user/message` 上。
- 锁括住整个操作，`compaction/end` 最后释放——于是中途崩溃留下的是「可检测的孤儿锁」（有 start 无 end），而不是一个谎称完成的 end。
- `compaction/summary` 里带 `llmStreamCall: true` 标记与完整 `rawOutput`，让这次一次性请求可以从「日志 + 代码」重建。

### 剪枝与溢出

- `ctx.toolResultPruner`（`compaction-tool-result-pruner`）在摘要压缩之前，把超大的**当前**工具结果改写成可重放的单节点 surface 替换。
- `ctx.spillStore`（`spill-local` + `spill-policy`）把超大工具文本落盘，返回模型可见的定位符 + 取回提示；`spill-policy` 是 `tools/post-execute` 的消费者，决定何时溢出。

### 计量

`ctx.tokenMeter` 拥有 per-session 的隔离回放折叠，压力消费者共享不可变的带修订号测量值。

### 检索（`ctx.sessionQuery`）

接口提供精确读、过滤器、关系追踪；SQLite 后端补上全文对账、排名、片段、游标代数。模型消费者 `tool-session-query` 拥有工作区权限与无游标渲染。`ctx.sessionReferenceResolver` 把有界的跨会话对话快照投影成持久的**不可信**消息上下文（mention 语法归宿主适配器）。

<!-- @end-section -->

<!-- @section: knowledge -->
## 知识与人格

- **System prompt**（`ctx.systemPrompt`）：插件注册 prompt 分段与模型可见工具 schema，每个 step 重新装配。`system-prompt/assemble` 是**专家级协作型整体变换**——返回的装配结果是权威的，因此监听器作者有义务保留仍然活跃的 Code Mode 与结构化输出协议贡献。需要工具过滤时应优先用 `ctx.tools.restrict()`，因为它能让展示、查找、执行三者保持一致。
- **Skills**（`ctx.skills`）：合并多 provider 的技能目录（`skill-filesystem` 本地目录、`skill-badge` 打包徽章），host + per-scope 分层。注册是同步的，远程初始化与发现放在被 await 的 `list()` 里。发现结果可以显式标记「不完整」——候选仍可加载但结果不可缓存。`tool-skill` 负责渲染会话前缀目录并加载完整技能体。
- **AGENTS.md**：根目录版本 = 一个读文件的 section provider；子目录 on-touch 版本 + 文件变更通知 = 从 watcher / 工具结果监听器调 `agent.inject()`。

<!-- @end-section -->

<!-- @section: interaction -->
## 人机协作面

| 能力 | 机制 |
|---|---|
| 审批 | `ctx.approval` 一次性权限决策，经 `approval/request` waterfall 派发；应答者是监听器，**缺席时 fail-closed 为 `unavailable`** |
| 权限预设 | `ctx.permissionPresets` 把沙箱模式与审批策略两个旋钮打包成 `workspace-write` / `danger-full-access`，一次切换写一条 `permission/preset` 事件并贯通两个旋钮事件 |
| 提问 | `ctx.userQuestions` 由 UI 前端提供应答 provider，`tool-ask-user` 在 provider 无关的 `ask()` promise 上挂起该工具调用 |
| 人类命令 | `ctx.commands` 注册斜杠命令，**不经过模型 turn** 直接派发 |
| Plan 模式 | `ctx.planMode` 折叠 log-only 的 `plan/mode` 状态，在 turn 边界冲刷用户选择，渲染部署方拥有的指引，注册 `/plan`，并在状态迁移间保持 `exit_plan_mode` schema 稳定 |
| Goal | `ctx.goals` 持有带修订号的 `active`/`paused`/`blocked`/`complete` 阶段与「goal round 上限」；**goal 是状态，不是调度器也不是独立对话**，会话日志仍是真相源。goal activation 刻意不进持久回放，所以 resume 与 fork 必须经人类授权的 `/goal` 或模型工具重新武装 |
| 反馈 | `ctx.messageFeedback` 每条 assistant 消息的本地反馈，带乐观版本与 per-item CAS，**不进 session 历史也不进遥测** |

<!-- @end-section -->

<!-- @section: data -->
## 数据与运维面

| 能力 | 说明 |
|---|---|
| 持久化 | `ctx.sessionPersistence`：JSONL / SQLite 两后端持久化同一套 `SessionEvent` 词汇，无平行的持久化事件类型；批写窗口 + `session/flush` 检查点；崩溃恢复补合成 `turn/end{interrupted}` 而非截断 |
| 投影 | `ctx.sessionProjections` 让各域注册状态驱动的折叠单元；`ctx.sessionProjectionCache` 持久化 checkpoint（节流 + turn/end/detach 强制点），提供「缓存行 + 持久化尾部重放」的冷读阶梯，**让列表不必加载完整日志** |
| 标题 | `ctx.sessionTitle` 拥有确定性回退与「最新标题」折叠，异步 provider 可选（首条 prompt / 全 prompt 两种 LLM 实现） |
| 遥测 | `ctx.sessionTelemetry` 捕获、脱敏、交给单一后端（OTel）；它的输出离开进程，因此进程内无人消费该服务 |
| 通用存储 | `ctx.storage`（json / sqlite 后端并列注册）+ `ctx.storageDomain`（等齐所有后端后，把 domain form 作为一个生命周期绑定的服务发布） |
| 附件 | `ctx.attachments`：宿主在 session 事件之前提交已接受的图片，provider 适配器把授权的持久引用解析成 provider 原生内容 |
| 凭据 | `ctx.credentials`：**配置里只放引用，值归 provider**；消费者按操作解析，因此轮换后的凭据下一次请求就生效 |
| 设置 | `ctx.settings`：插件注册命名空间 schema，分层解析（defaults → 组合 base → 用户文档）；Web 网关只提供脱敏的分层描述符 |
| 不变式 | `ctx.invariants`：包自有的运行时检查，服务负责选择、唯一性、子 fiber 与按包归因的失败 |
| 守护 | `guard/` 组：循环卫生（重复调用的建议式提醒）+ `tools/execute` 上的截止时间执行器 |

<!-- @end-section -->

<!-- @section: interop -->
## 对外互操作

| 通道 | 包 | 定位 |
|---|---|---|
| ACP | `packages/acp` | 仅自动化的 Agent Client Protocol JSON-RPC stdio server：开全新文本会话、发出已提交的 assistant 文本、为自己拥有的 agent 注册一次性机器审批应答者 |
| JSON-RPC SDK | `packages/sdk` | 进程外运行时 SDK：协议 + server 插件 + TypeScript 客户端 |
| Python SDK | `python/sdk` | 外挂 Python 客户端与打包运行时 |
| Hook 桥 | `packages/hooks` | 把 Claude Code / Codex 的 hook 配置文件映射到 dsh 原生扩展点，共享一份 wire-protocol 库 |
| MCP | 示例 `examples/mcp-memory` | 每个 server 一个插件：发现工具 → `ctx.tools.register()`（MCP 工具正是以 raw JSON-Schema `ToolDefinition` 形式进入注册表的） |
| Web GUI | `packages/host` + `packages/client` + `apps/web` | Typert RPC + Connection + 30+ `ui-*` 浏览器插件 |

协议驱动器的通用契约值得记：低层 prompt 请求返回的是**持久入队回执**，它不通过把 `MessageId` 与 `turn/end` 关联来获取结果；整体 agent 状态另行发布。自动化方法可以从回执等到下一次 idle 并总结这段**自己明确拥有的区间**，而 UI 通常保持观察开放式事件流。

<!-- @end-section -->

## 相关文档

- [[analysis-dsh-plugin-system-003|插件系统]] — 这些能力如何被组装
- [[analysis-dsh-architecture-002|系统架构]] — 生命周期与事件域
- [[analysis-dsh-insights-005|评估与借鉴]] — 对 Legion 的可移植点
- [[analysis-dsh-overview-001|项目总览]]
- [[analysis-dsh-index|DeepSeek Harness 分析索引]]
