---
title: 会话交接 — Milestone 2c（接活 /v1/events SSE + 审批事件桥接）从零接续
type: handoff
status: active
created: 2026-07-18
scope: legion/legionAgent（后端）
related:
  - "[[2026-07-18-m2b-handoff-manual-approval-gate]]"
  - "[[2026-07-15-agent-working-modes-design]]"
tags: [agent, runtime, sse, events, approval, milestone-2c, handoff]
---

# 会话交接 · Milestone 2c — 接活 /v1/events SSE（从零接续）

> 本文件是**新会话从零接续 M2c 的全部上下文**：现状、锁定设计、任务拆分、代码指针（已核对 master HEAD）、坑、工作流。读完即可直接写 M2c plan（superpowers:writing-plans）→ subagent-driven 执行。

## 0. 一句话现状 + 接续入口

Phase 2「Agent 工作模式 + 工作目录 + 真并发」：**M1a/M1b/M2a/M2b 已实现 + 逐任务 review + opus final review + 合入 legionAgent master**。M2 拆 M2a/M2b/M2c。**M2c = 接活 `/v1/events` SSE + 桥接审批/生命周期事件，尚未开始。**

- master 当前 HEAD：`dab6162`（M2b PR #3 已合，含 M2b 7 任务 + 2 后续 fix）。M2b 核心 `3988585`，2 后续 `b7cfb2f`(Manual 无 gate fail-loud) + `7a9b4bf`(已决票据 wrap sentinel+409+sweep 容忍)。
- **接续动作**：读本文件 → 读 spec §4.5（`docs/superpowers/specs/2026-07-15-agent-working-modes-design.md`）→ `/writing-plans` 出 M2c plan（见 §4 任务拆分）→ subagent-driven 执行（§6 流程；`-race` 用 WSL）。
- 之后：M3（working_dir + GUI + TUI 对等）。

## 1. 三仓库拓扑（不变，务必知道）

`F:\source\stardust\Legion` 根**不是** git 仓库；三个独立仓库各有 github remote，分别提交：
- **server** = `legion/legionAgent/`，remote `github.com/jxncyjq/jxncyjq-stardust-agent-server`，M2c 全部代码在此。改代码前 `git rev-parse --show-toplevel` 确认。从 master `dab6162` 切 `feat/agent-modes-m2c`。
- docs = `docs/`（本文件所在），独立仓库，AI 改动通常**不自动提交**。
- GUI = `legion/legionAgentGUI/`，独立 module，不得 import server `internal/**`（M2c 不碰 GUI 前端，那是 M3；但 M2c 的 SSE 是 GUI 的数据源，要留好契约）。
- `legion/.git` 遗留聚合仓库**勿用**。
- push/merge 可能被 Claude Code 权限分类器拦；以中文写 PR 正文。M2b 经 `gh pr create`+`gh pr merge` 成功合入。

## 2. M2b 已交付的、M2c 直接复用/相关的接口

- `internal/manualgate`（不 import runtime，结构性满足 `runtime.ToolGate`）：
  - `ManualToolGate{store}`；`New(store *approval.ToolGateStore) *ManualToolGate`。
  - `ShouldSuspend(ctx, task, calls, tools)(bool,error)` —— **敏感工具开票据（`store.Open`）+ 返 true 挂起的地方，这里是 `approval_pending` 事件的发射点**。
  - `Resolve(ctx, task, call, tools)(allow bool, err error)` —— 派发级决定检查。
  - `ApprovalCoordinator{store, sched SchedulerGate}`；`NewApprovalCoordinator(store, sched)`；`Decide(ctx, taskID, ticketID, status)(ToolApproval,error)` —— **全票据决讫且 Suspended→翻 Running 的地方，这里是 `approval_resolved` 事件的发射点**；`ReconcileResume(ctx, taskID) error`（重启对账）。
  - `NewTimeoutSweepJob(store, dec, ttl, now, logger) func(ctx)error` —— **超时 deny 也应发 `approval_resolved`（decision=deny, reason=timeout）**。
- `internal/approval`：`ToolGateStore`（会话目录 JSON 票据，mutex 序列化）；`ToolApproval{TicketID,SessionKey,TaskID,ToolCallID,ToolName,Arguments map[string]string,Status,CreatedAt,UpdatedAt}`；`ApprovalStatus{ApprovalPending/Approved/Denied}`；`ListPending()`（全 session，可支撑 §4 的 list-tickets 端点）。
- serve 装配（`internal/cli/command.go`）已把 manualGate/approvalCoordinator/toolGateStore 全接线（见 M2b handoff §5）。M2c 在此基础上加 platformEvents + 桥接。

## 3. M2c 锁定设计（依据 spec §4.5 + 已核对代码）

### 3.1 现状：两套事件系统**互不连通**（核心难点）
- **`port.EventBus`**（`port/ports.go:61`）= runtime/coordinator/workflow 用的 `workflowEvents`：只有 `Publish(ctx, domain.RuntimeEvent) error` + `Events() []domain.RuntimeEvent`。**poll-only，无 subscribe/streaming**。实现 = `adapter.MemoryEventBus`（`adapter/memory.go:40`），Publish 只 append 到 slice，`Events()` 返快照。GEP/trust/degradation 后台 job 靠 `.Events()` 轮询消费。
- **`observability.EventBus`**（`observability/eventbus.go:21`）= SSE 用的 `platformEvents`：`Publish(ctx, EventEnvelope) error` + `Subscribe(ctx)(<-chan EventEnvelope, cancel)` + `Close()`。**真 push/streaming**。`EventEnvelope{ID,Type,SubjectID,Data map[string]any,CreatedAt}`。新订阅者会重放已缓冲事件（`Subscribe` 里遍历 `b.events`）。
- **死代码 bug**：`GET /v1/events`→`handleEvents`（`server/events.go:13`）订阅 `s.platformEvents`；`platformEvents==nil`→503。`server.Config.PlatformEvents`（`http.go:82`）**serve 从没设**→恒 503。M2c 修这个。

### 3.2 桥接方案（锁定）：**tee 型 EventBus**
不要写轮询 diff。做一个实现 `port.EventBus` 的**桥接总线**，`Publish(RuntimeEvent)` 时：① append 到内部 slice（保 `Events()` 给既有 poll 消费者不破）；② 把 `RuntimeEvent` 翻译成 `observability.EventEnvelope` 并 `platformEvents.Publish`。在 serve 里用它替换 `workflowEvents := adapter.NewMemoryEventBus()`（`command.go:1708` 一带），持有 `platformEvents` 引用。这样 runtime/coordinator 现有的所有 `events.Publish(RuntimeEvent{...})`（task_started/tool_call_requested/tool_result/tool_executed/tool_failed/inference_completed/task_completed…）**零改动**就流到 SSE。
- 翻译映射：`EventEnvelope{Type: <映射>, SubjectID: RuntimeEvent.TaskID, Data: {task_id, message, prompt_tokens...}, CreatedAt: RuntimeEvent.CreatedAt}`。
- **⚠️ 类型命名约定要拍板**：`RuntimeEvent.Type` 是下划线（`task_completed`）；既有 SSE 测试 `events_test.go` 用点号（`task.completed`，但那只是测试直接 publish 的字面量，不是约定）。M2c 定一个约定（保持下划线 or 统一映射到点号），并与 GUI 消费端对齐（M3 GUI 要用）。建议保持下划线、与 RuntimeEvent 一致，减少映射面。

### 3.3 审批事件（新增，SSE 专属）
`approval_pending` / `approval_resolved` 不走 RuntimeEvent（其固定字段 `Type/TaskID/Message/tokens` 塞不下 ticket_id/tool/arguments）。用**独立 sink**，避免污染 RuntimeEvent、也避免 manualgate import observability：
- 在 `internal/manualgate` 定义小接口，如 `type ApprovalEventSink interface { ApprovalPending(ctx, taskID, ticketID, tool string, args map[string]string) error; ApprovalResolved(ctx, taskID, ticketID, decision string) error }`。`ManualToolGate` 与 `ApprovalCoordinator` 持一个**可选**（nil 则不发）的 sink。
  - `ShouldSuspend` 在 `store.Open` 成功后（每张新开的 pending 票据）→ `sink.ApprovalPending(...)`。
  - `ApprovalCoordinator.Decide` 决定落盘后 → `sink.ApprovalResolved(...decision...)`。
  - `NewTimeoutSweepJob` 每张超时 deny 后 → `sink.ApprovalResolved(...deny...)`（可复用 dec 里的 sink 或 job 单独持）。
- serve 侧提供 sink 实现：翻译成 `EventEnvelope{Type:"approval_pending"/"approval_resolved", SubjectID:taskID, Data:{task_id,ticket_id,tool,arguments/decision}}` → `platformEvents.Publish`。sink impl 放 cli 或一个新小包，import observability + manualgate。
- **fail-loud**：sink Publish 出错——审批**不能因事件发不出去而失败**（票据已落盘是真相源）。故 sink 错误应**记 Warn 不阻断** ShouldSuspend/Decide（SSE 是通知层，非权威）。这是「契约允许的可选能力」豁免，不算兜底——但**必须记日志**，不得 `_ = err` 静默。ManualToolGate/ApprovalCoordinator 拿到 sink error 时的处理要写清（建议：包一层带 logger 的 sink，在 sink 内部 Warn 后返回 nil，使调用方 fail-loud 契约不被破坏）。

### 3.4 两个必须在 plan 里拍板的设计决策
1. **arguments 是否/如何过 SSE**：`approval_pending` 带 `arguments`（spec §4.5 原文）。但 SSE 的 `sanitizeEventData`（`events.go:59`）**只删顶层 key**（prompt/input/secret/api_key/*secret*/*token*），`arguments` 作为一个 map 是嵌套的——里面若有 `content`（write_file 的文件内容，可能巨大/敏感）或 `input` 子键**不会被删**。选项：(a) 只发 tool 名 + arguments 的**键列表**（不发值）；(b) 发 arguments 但截断大值 + 递归 sanitize；(c) 发全量并接受风险。**倾向 (b)**：人工审批需要看到工具要干什么（路径/命令），但要截断+递归脱敏。plan 里定并加测试断言脱敏生效。
2. **SSE 是 at-most-once**：`observability.EventBus.Publish`（`eventbus.go:55-59`）对满 buffer 的订阅者是**非阻塞丢弃**（`select{case ch<-event: default:}`）。即慢订阅者会丢事件。`approval_pending` 丢了 = GUI 漏审批提示。**缓解**：票据在盘上是真相源；补一个 **`GET /v1/tasks/{id}/approvals` 或 `GET /v1/approvals?status=pending` 列票据端点**（M2b spec §7 标「可选」、当时未做），让 GUI 能对账 SSE 漏掉的 pending。建议纳入 M2c（是 GUI 可靠消费的前提）。

### 3.5 生命周期事件（顺带打通）
桥接 tee 一旦接上，`task_started`/`tool_call_requested`/`tool_result`/`tool_executed`/`tool_failed`/`inference_completed`/`task_completed` 这些 runtime 已在发的 RuntimeEvent 自动流到 SSE（spec §4.5 要求）。无需改 runtime 发射点，只需桥接 + 映射。

## 4. M2c 任务拆分（写 plan 时细化为 TDD 步骤）

1. **桥接 EventBus**：新类型（放 `internal/adapter` 或新 `internal/eventbridge`）实现 `port.EventBus`，`Publish(RuntimeEvent)` = append(保 `Events()`) + 翻译→`platformEvents.Publish(EventEnvelope)`。定类型命名映射（§3.2）。测试：Publish 一个 RuntimeEvent → platformEvents 订阅者收到对应 EventEnvelope（Type/SubjectID/Data 正确）；`Events()` 仍返回原 RuntimeEvent 快照（poll 消费者不破）。
2. **serve 接 platformEvents + 桥接**：`command.go` 构造 `platformEvents := observability.NewEventBus(buf)`；`workflowEvents` 换成桥接总线（持 platformEvents）；`httpServer` 的 `server.Config{... PlatformEvents: platformEvents}`（杀 503）；shutdown 里 `platformEvents.Close()`（`ServeResult.Close` 现有 `coordinator.Wait()+closeStore()` 一带）。测试：serve 起来后 `GET /v1/events` 不再 503（既有 cli/serve 集成测试或新加）。
3. **审批 sink + 发射点**：manualgate 定 `ApprovalEventSink` 接口 + ManualToolGate/ApprovalCoordinator 持可选 sink；`ShouldSuspend` 开票据后发 `approval_pending`、`Decide` 发 `approval_resolved`、超时 sweep deny 发 `approval_resolved(deny)`。serve 侧 sink 实现翻译→platformEvents。sink 错误 Warn 不阻断（§3.3 fail-loud 边界）。测试：manual 敏感调用挂起→platformEvents 收到 approval_pending{task_id,ticket_id,tool,arguments}；approve/deny/timeout→收到 approval_resolved{ticket_id,decision}；sink 出错时审批不被打断（且有 Warn）。
4. **arguments 脱敏决策**（§3.4.1）：按选 (b) 递归+截断脱敏 `approval_pending.arguments`（或强化 `sanitizeEventData` 递归）。测试：arguments 里的敏感子键/大值被脱敏/截断，SSE 不泄漏。
5. **list-pending-tickets 端点**（§3.4.2）：`GET /v1/approvals?status=pending`（或 per-task），基于 `ToolGateStore.ListPending`/`ListForTask`。测试：返回盘上 pending 票据；供 GUI 对账。（若 plan 判为可延后到 M3，明确记为决定。）
6. **serve 全链路 + 门禁**：整合 1-5，全仓 `go build/vet/test ./...` 绿 + WSL race 绿；`GET /v1/events` 端到端收到生命周期 + 审批事件。

> M2c 比 M2b 小。核心风险=桥接的事件语义（丢弃/顺序/重放）+ 脱敏边界。任务 1/3 是重点。

## 5. 关键代码指针（已核对 master `dab6162`，行号约数）

- `internal/server/events.go`：`handleEvents`(@13，订阅 `s.platformEvents`，nil→503)、`writeSSEEvent`(@37)、`sanitizeEventData`(@59，**只删顶层 key**)、`isSensitiveEventField`(@70)。
- `internal/server/http.go`：`Config.PlatformEvents *observability.EventBus`(@82)、`HTTPServer.platformEvents`(@105)、`NewHTTPServer` 赋值(@145)、路由 `GET /v1/events`→`handleEvents`(@200)。M2b 已加 `ToolApprovals`/`handleDecideApproval`（审批决定端点）。
- `internal/server/events_test.go`：SSE 测试骨架（`NewHTTPServer(Config{PlatformEvents:bus})` + `httptest` + goroutine `bus.Publish` + `bus.Close`）。注意它用点号 type `task.completed`（见 §3.2 命名决策）。
- `internal/observability/eventbus.go`：`EventEnvelope`(@12)、`NewEventBus(buffer)`(@29)、`Publish`(@39，**满 buffer 非阻塞丢弃** @55-59)、`Subscribe`(@64，重放已缓冲 @77)、`Close`(@100)。
- `internal/port/ports.go`：`EventBus interface{ Publish(ctx,RuntimeEvent)error; Events()[]RuntimeEvent }`(@61，**无 subscribe**)。
- `internal/adapter/memory.go`：`MemoryEventBus`/`NewMemoryEventBus`(@40)/`Publish`(@44)/`Events`(@54)。桥接类型可放这里或新包。
- `internal/domain/types.go`：`RuntimeEvent{Type,TaskID,Message,PromptTokens,CompletionTokens,CachedTokens,TotalTokens,ElapsedMs,CreatedAt}`(@124)。
- `internal/cli/command.go`：`workflowEvents := adapter.NewMemoryEventBus()`(@~1708，桥接替换点)、resolver/defaultRuntime/coordinator 都用 workflowEvents（改桥接后自动 tee）、`httpServer := server.NewHTTPServer(server.Config{...})`(@~1906，加 `PlatformEvents`)、`ServeResult.Close`(@~1959，加 `platformEvents.Close()`)、M2b 装的 `toolGateStore/manualGate/approvalCoordinator`(@~1731 一带，加 sink 接线)。
- `internal/manualgate/manualgate.go`：`ShouldSuspend`（开票据点，发 approval_pending）；`internal/manualgate/decider.go`：`Decide`（发 approval_resolved）；`internal/manualgate/timeout.go`：`NewTimeoutSweepJob`（超时 deny 发 approval_resolved）。
- workflow engine（`internal/workflow/engine.go`）也 publish RuntimeEvent 到同一 workflowEvents，桥接后一并流到 SSE（顺带，spec §4.5 生命周期）。

## 6. 环境 + 工作流（照 M2b）

- **`-race` 只能 WSL**（Windows 无 gcc）。**`GOOS=linux GOARCH=amd64` 必须显式加**（WSL Go 工具链持久化了 GOOS=windows，否则 cgo MinGW 参数在 linux gcc 上失败 `-mthreads`）：
  ```bash
  wsl -d Ubuntu-22.04 -- bash -lc 'cd /mnt/f/source/stardust/Legion/legion/legionAgent && GOOS=linux GOARCH=amd64 CGO_ENABLED=1 CC=gcc $HOME/sdk/go/bin/go test -race ./internal/server/ ./internal/manualgate/ ./internal/adapter/ ./internal/observability/ ./internal/cli/'
  ```
  Windows 宿主：`go test ./...`。
- **完成门禁**：`go build ./... && go vet ./... && go test ./...` 全绿；`gofmt -l` 触碰文件内容 LF 干净（仓库 `core.autocrlf=true`，`gofmt -l` 会误报既有 CRLF 文件——只 `gofmt -w` 自己改的文件，靠剥 `\r` 验证内容干净，别整仓重排）。
- **Fail-Loud 铁律**（legionAgent/CLAUDE.md §0）：禁 fallback/zero-value 兜底、禁 `_ = err`、禁静默。**M2c 特有边界**：SSE 是通知层非权威——sink/桥接 Publish 失败**记 Warn 不阻断**审批/runtime（票据/checkpoint 盘上是真相源），但**必须记录**，不得静默吞。EventBus 满 buffer 丢事件是既有契约行为（§3.4.2），靠 list 端点对账，不算兜底。
- **SDD 流程**（superpowers:subagent-driven-development）：从 master 切 `feat/agent-modes-m2c`；ledger 在 `legionAgent/.superpowers/sdd/progress.md`（gitignore scratch，可覆盖 M2b）；每任务：手工抽 brief 写 `.superpowers/sdd/task-N-brief.md`（**task-brief 脚本按英文 "Task N" 匹配，中文「任务 N」认不出→手工抽或 plan 用英文 `## Task N:` 标题**）→ 派 executor(sonnet) 实现 → `scripts/review-package BASE HEAD` → 派 code-reviewer 审 → 修 Critical/Important → 记 ledger。桥接/事件语义(任务 1/3)可用 opus reviewer。最终整分支 opus final review → finishing-a-development-branch → PR(中文)+合并。
- **⚠️ 子 agent 最终消息被 OMC/caveman hook 吞** → 让实现者/审查者**报告写文件**（`.superpowers/sdd/task-N-report.md` / `task-N-review.md`），控制方读文件核验，别信返回的 chat 消息。诊断里的编译错常是 mid-TDD-red 快照，自己 `go build` 确认最终态。
- **plan 文件路径**：`docs/superpowers/plans/2026-07-18-agent-modes-m2c-*.md`（docs 仓库，绝对路径给 task-brief 脚本，因 git cwd 在 legionAgent）。

## 7. 待办（跨里程碑，非 M2c 必须但记牢）

- **M2b 遗留（final review 残留，非阻塞，已 spawn task/或已合）**：`b7cfb2f` 已加 Manual 无 gate fail-loud；`7a9b4bf` 已 wrap ErrTicketAlreadyDecided+409+sweep 容忍。M2c 若碰 approval sink，注意别回退这些。
- **潜在 gateless-Manual**：委派子 runtime（`newSubRuntime` delegation.go:86）不 copy toolGate/checkpoints；今天安全因 child=Auto 且 delegate_task/moa_consult 本身 Sensitive。`b7cfb2f` 已加 RunTask 层 fail-loud 断言兜底。M2c 不碰委派。
- **agent.json 密钥**：曾含实时 deepseek key 进过旧 remote 历史，需去 deepseek 轮换；新 remote 干净（agent.json/db 已 gitignore）。见 [[legion-git-repo-topology]]。
- 死代码 `func max`(runtime.go)、`errStaticResolver`(coordinator_test.go)——可另开清理任务。
- **M3**（下一里程碑）：working_dir（per-session 沙箱）+ GUI 适配器（SSE 桥 EventsEmit、模式选择器、审批 UI、目录选择）+ TUI 适配器（/mode /cwd slash + 状态栏 + 终端内审批 prompt）。M2c 的 SSE 事件契约（approval_pending/approval_resolved/生命周期 + list 端点）是 GUI 审批 UI 的数据源，M2c 定好契约 M3 才好接。

## 8. 本轮（2026-07-18）工作日志

- subagent-driven 执行 M2b（7 任务，13 commits base e644b20..3988585 + 2 后续 b7cfb2f/7a9b4bf，opus final MERGE READY）→ PR #3 合入 master `dab6162`。
- 过程抓出并修 2 个真 bug：resume 超 lockTTL 二次派发（进程内 in-flight guard）、ToolGateStore Windows rename-vs-read race（sync.Mutex）。
- 进入 M2c：探明 SSE 现状（两套事件系统互不连通、`/v1/events` 恒 503 死代码、EventBus 满 buffer 丢弃、sanitize 只删顶层 key）→ 写本交接文件。

**接续入口（新会话）**：读本文件 → 读 spec §4.5 → `/writing-plans` 出 M2c plan（§4 的 6 任务，任务 1 桥接 + 任务 3 审批 sink 是核心）→ subagent-driven（§6 流程）执行。
