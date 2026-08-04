---
title: 会话交接 — Milestone 2b（Manual 审批 gate）从零接续
type: handoff
status: active
created: 2026-07-18
scope: legion/legionAgent（后端）
related:
  - "[[2026-07-15-agent-working-modes-design]]"
  - "[[2026-07-15-session-handoff-agent-modes]]"
  - "[[2026-07-17-agent-modes-m1b-checkpoint-resume]]"
  - "[[2026-07-18-agent-modes-m2a-mode-plan]]"
tags: [agent, runtime, approval, manual, toolgate, milestone-2b, handoff]
---

# 会话交接 · Milestone 2b — Manual 审批 gate（从零接续）

> 本文件是**新会话从零接续 M2b 的全部上下文**：现状、锁定设计、任务拆分、代码指针、环境坑、工作流。读完即可直接写 M2b plan（superpowers:writing-plans）→ subagent-driven 执行。

## 0. 一句话现状 + 接续入口

Phase 2「Agent 工作模式 + 工作目录 + 真并发」：**M1a/M1b/M2a 已实现 + 逐任务 review + opus final review + 合入 legionAgent master**。M2 拆 M2a/M2b/M2c。**M2b = Manual 审批 gate，尚未开始。**

- master 当前 HEAD（M2a 合入后）：merge `e644b20`（PR #2）。M1b merge `4f5abd0`（PR #1）。M1a `3bd5127`。
- **接续动作**：读本文件 → 读 spec §4.3（`docs/superpowers/specs/2026-07-15-agent-working-modes-design.md`）→ `/writing-plans` 出 M2b plan（7 任务，见 §4）→ subagent-driven 执行（用 §6 的 WSL `-race` + SDD 流程）。
- 之后：M2c（接活 `/v1/events` SSE）、M3（working_dir + GUI + TUI 对等）。

## 1. 三仓库拓扑（不变，务必知道）

`F:\source\stardust\Legion` 根**不是** git 仓库；三个独立仓库各有 github remote，分别提交：
- **server** = `legion/legionAgent/`，remote `github.com/jxncyjq/jxncyjq-stardust-agent-server`，M2b 全部代码在此。改代码前 `git rev-parse --show-toplevel` 确认。
- docs = `docs/`（本文件所在），独立仓库，AI 改动通常**不自动提交**，用户按流程处理。
- GUI = `legion/legionAgentGUI/`，独立 module，不得 import server `internal/**`（M2b 不碰 GUI，那是 M3）。
- `legion/.git` 是遗留聚合仓库**勿用**。
- push/merge 可能被 Claude Code 权限分类器拦；M1b/M2a 均先 push 成功、`gh pr merge` M2a 通过、M1b 被拦由用户手动合。以中文写 PR 正文。

## 2. M1b + M2a 已交付的、M2b 直接复用的接口

**M1b（`internal/sessionstate` + `internal/runtime`）：**
- `sessionstate.Store`：`NewStore(base)`、`Save(cp)`/`Load(key)(Checkpoint,bool,error)`/`Delete(key)`/`ListSuspended()`、`WritePlan(key,file,content)`（M2a 加）。
- `sessionstate.Checkpoint{SchemaVersion,TaskID,AgentID,SessionKey,BasePrompt,Round,ToolEntries,PendingCalls[]domain.ToolCall,PromptTokens,CompletionTokens,CachedTokens,TotalTokens,Images,CreatedAt}` —— **⚠️ 无 Mode 字段（M2b 任务 1 要加）**。`CheckpointSchemaVersion=1`（改结构要升版本）。
- `sessionstate.ResolveWorkspaceRoot(configured)(root,warning)`、`SessionDir(base,key)`。
- `runtime.ErrSuspended`；`runtime.ToolGate interface{ ShouldSuspend(ctx, task domain.Task, calls []domain.ToolCall)(bool,error) }`（**M2b 实现它 + 可能扩展**）。
- `runtime.Config.Checkpoints *sessionstate.Store` + `Config.ToolGate ToolGate`；`Runtime.checkpoints`/`toolGate` 字段。
- `RunTask` = 入口(有 checkpoint 则自动恢复)→`runToolLoop(ctx,requestID,agent,task,st loopState)`→`finishRun`。`loopState{started,basePrompt,round,toolCtx,resp,4 token 计数,images,tools}`。
- `checkSuspend(ctx,task,st)(bool,error)`（runtime.go ~352）：`toolGate==nil||checkpoints==nil` 返 false；否则调 `toolGate.ShouldSuspend(ctx,task,st.resp.ToolCalls)`，true → Save checkpoint（**round 级，执行前**）→ 返 true → runToolLoop 返 `ErrSuspended`。
- `sessionKeyForTask(task)` = SessionID else ID。
- `Coordinator`：`runAssigned` 遇 `ErrSuspended` → `TaskSuspended`（coordinator.go ~199）；`RecoverSuspended(ctx,store)(int,error)`（~274）重启扫 checkpoint 重注册为 Suspended（**丢 Mode，任务 1 要修**）。

**M2a（`internal/domain`/`tool`/`storage`/`server`/`runtime`）：**
- `domain.ModeManual/ModePlan/ModeAuto`（裸 string 常量）；`domain.NormalizeMode(raw)(mode,ok)`（大小写敏感，空→auto，未知→ok=false）；`domain.AgentSession.Mode`、`domain.Task.Mode`。
- `tool.Descriptor.Sensitive bool`；`tool.Registry.SafeToolNames()`（非敏感、非 meta 工具排序名）。**已分类**：敏感=`write_file`/`fetch_url`/`send_message`/`create_task`/`claim_task`/`update_task`/`append_task_message`/`rebuild_task`/`delegate_task`/`moa_consult`；安全=`read_file`/`search_content`/`list_files`/`read_task`/`read_messages`/`session_search`。
- sqlite schema v5：`agent_sessions.mode` 列已持久化。
- HTTP：`POST`/`PATCH /v1/sessions` 收+校验 mode；`POST /v1/tasks` 时 `task.Mode` 从 session 继承（缺失 session→404）。
- runtime Plan 模式：`effectiveTools(task)` = `task.Mode==plan` → `r.tools.Subset(SafeToolNames()...)`，否则 `r.tools`。**已把 `tools *tool.Registry` 线程进** `generate`/`inferenceTools`/`executeToolCalls`/`dispatchToolCall`/`dispatchCallTool`/`listToolsCatalog`（+ `loopState.tools`，fresh+resume 分支都设）。**M2b 的 gate 逻辑接在这条已线程好的链上。**

## 3. M2b 锁定设计（用户已拍板）

### 3.1 执行模型：保 M1b round 级挂起（挂起发生在执行前，无需 per-call checkpoint）
`checkSuspend` 在 `runToolLoop` 每轮 `executeToolCalls` **之前** 调 `ShouldSuspend`。因挂起在执行前，checkpoint 仍是 round 级（存整轮 PendingCalls），**不需要** per-call/mid-round checkpoint。resume 时整轮重跑、per-call 查决定。

### 3.2 lazy 协议敏感度探测（关键）
Manual gate 要看**真实工具**敏感度。lazy 协议下 round 顶层的 `resp.ToolCalls` 是 `call_tool`（meta），真实工具名在 `call.Arguments["tool_name"]`。→ `ShouldSuspend` 遍历 calls 时：
- `call.Name` 是真实工具 → 直接查 `Descriptor.Sensitive`。
- `call.Name == "call_tool"` → 取 `call.Arguments["tool_name"]` → 查那个工具的 `Sensitive`。
（meta 常量：lazytools.go `metaToolCallTool="call_tool"`、`metaToolListTools="list_tools"`；`call_tool` args 键 = `tool_name` + `arguments_json`。）

### 3.3 resume 派发 = **方案 B**（用户选定）
> ⚠️ **真架构缺口**：coordinator `Heartbeat`→`scheduler.Next` **只挑 `TaskPending`**（scheduler.go:50 `if task.Status != domain.TaskPending { continue }`）。没有任何机制重新拾取 `Suspended→Running` 的任务。M1b 从没接这条链（e2e 直接调 RunTask 绕过）。M2b 必须补。

**方案 B 实现**：`Decide`（该任务全部票据已决）→ 任务转 `Suspended→Running`（转移表 scheduler.go:118-119 已允许）。`Coordinator.Heartbeat` **新增 resume 扫描**：遍历 scheduler 中 `Status==TaskRunning` 且**有 checkpoint**（`store.Load` ok）且**锁可获取**（`locks.TryLock` 成功 = 无活跃 goroutine 在跑）的任务 → 起 goroutine 跑一条 **resume 版 runAssigned**（跳过 Pending→Assigned→Running 初始转移，因已是 Running；直接 `RunTask` 自动从 checkpoint 恢复 → 走完 review/done 或再次 suspend）。
- 用 `LockStore`（coordinator 已有 `c.locks`，`runAssigned` 用 `TryLock(taskID, agentID, lockTTL)`）区分「有无活跃 worker」：挂起时 goroutine 返回、锁随 TTL（默认 1min）过期或需显式释放 —— **M2b 要确保挂起时释放锁**，否则 resume 扫描要等 TTL。检查 `runAssigned` 挂起路径是否释放锁；不释放则加释放。
- 防重复派发：resume 扫描靠 TryLock 独占；同一任务只有一个 goroutine。
- 注意与既有 Pending 派发不冲突：Next 挑 Pending，resume 扫描挑 Running+checkpoint+lockable，两者互斥集合。

### 3.4 审批持久化 + 决定链
- `internal/approval` 现为纯内存（`Service{tickets map}`，`OpenTicket`/`Decide`/`DecideBy`/`Tickets`；`Ticket{ID,Type,SubjectID,Status,Decision,DeciderID,Reason,Comment,CreatedAt,UpdatedAt}`）。
- M2b 改**会话目录 JSON 持久化**：票据存 `<sessionDir>/approvals/<ticketID>.json`，字段加 `SessionID/TaskID/ToolCallID/ToolName/Arguments/Status(pending|approved|denied)/CreatedAt`（spec §6）。按 `(taskID, toolCallID)` 可查「是否已决 + 决定」。启动扫盘加载。磁盘是真相源。
- `Decide`：更新盘上票据 + 检查该任务是否**全部票据已决** → 是则触发任务 `Suspended→Running`（配合 §3.3 resume 扫描）。
- 与 M1b `RecoverSuspended` 协同：重启后未决票据 + checkpoint 一起在盘上，任务留 Suspended 等决定。

### 3.5 gate 判定逻辑（spec §4.3 伪码，落到本设计）
- `ShouldSuspend(ctx, task, calls)`（round 级，checkSuspend 调）：
  ```
  if task.Mode != manual: return false
  needApproval := false
  for call in calls:
      realTool := resolveRealTool(call)      // §3.2 lazy peek
      if realTool.Sensitive && 无已决(task, call.ID):
          approval.OpenTicket(session,task,call.ID,realTool,args)   // 落盘
          needApproval = true
  return needApproval      // true → checkSuspend 存 checkpoint + ErrSuspended
  ```
- 派发层（`dispatchToolCall`/`dispatchCallTool`，resume 后每 call）：
  ```
  if task.Mode == manual && realTool.Sensitive:
      decision := approval.Decision(task, call.ID)
      if decision == denied: return 拒绝结果给模型（ToolResult{Success:false, Error:"denied by human"}）
      // approved → 正常执行
  执行工具
  ```
  （安全工具/非 manual 直接执行。denied 是 fail-loud-but-recoverable：拒绝结果回模型，不杀任务。）
- **gate 需访问**：approval store（查/开票据）+ tool 敏感度（经 `r.tools`/`effectiveTools` 的 Descriptor）。ManualToolGate 持有 approval store 引用；敏感度查 registry。dispatchToolCall 经 `r.toolGate` 调决定检查（可能扩展 ToolGate 接口加 `Resolve(ctx,task,call)(decision,error)`，或让 gate 暴露 store）。

### 3.6 超时
后台清扫（复用 background scheduler）对 pending 超时（默认 300s，可配）票据 → 标 denied + 触发任务恢复走拒绝分支。记 warn，不杀任务。

## 4. M2b 任务拆分（7 个，写 plan 时细化为 TDD 步骤）

1. **🔴 安全前置**：`sessionstate.Checkpoint` 加 `Mode string`；升 `CheckpointSchemaVersion`→2（加载旧版 fail-loud，见 M1b Load）；`runtime.checkSuspend` 存 `task.Mode` 进 checkpoint；`RunTask` resume 分支把 `cp.Mode` 放回重建的 task/loopState（使 resume 后 gate 仍认 manual）；`Coordinator.RecoverSuspended` 重建任务时 `Mode: cp.Mode`。测试：suspend(manual)→重启→resume 仍 manual（gate 仍会触发）。**必须先于任何 gate 挂起落地。**
2. **approval 会话目录持久化**：`Ticket` 扩字段；`Service` 改磁盘后端（`approvals/<id>.json`，按 sessionDir）；按 `(taskID,toolCallID)` 查决定；启动加载；fail-loud（损坏 JSON 报错跳过并记录，spec §8）。
3. **ManualToolGate（实现 ToolGate）**：`ShouldSuspend`（§3.5，含 §3.2 lazy peek + OpenTicket）；派发层决定检查（denied→拒绝结果）。接进 `dispatchToolCall`/`dispatchCallTool`（M2a 已线程 tools，敏感度查 Descriptor）。测试：manual+敏感→开票据+挂起；approve→resume 执行；deny→resume 拒绝回模型；只读工具不触发；lazy `call_tool` 指名敏感工具同样被 gate（复用 M2a `TestPlanModeLazyCallToolCannotReachSensitiveTool` 模式）。
4. **resume 派发（方案 B）**：`Decide` 全决→`Suspended→Running`；`Coordinator.Heartbeat` 加 resume 扫描（Running+checkpoint+lockable→resume runAssigned）；确保挂起时释放锁。测试：suspend→Decide(approve)→Heartbeat→任务续跑到 Done；`-race` 并发下无双重派发。
5. **审批 HTTP 端点**：`POST /v1/tasks/{id}/approvals/{ticketID}` body `{decision:"approve"|"deny"}` → `approval.Decide` + 触发 §4 resume。列票据端点（可选）。测试：approve/deny 路由、非法 decision 400、未知票据 404。
6. **超时 deny 清扫**：background job 扫 pending 超时→deny+触发恢复（§3.6）。测试：超时→自动 deny→任务走拒绝分支。
7. **serve 接线**：构造 approval 磁盘 store + ManualToolGate，注入**默认运行时 + resolver 建的 per-agent 运行时**（M1b/M2a 在 `command.go:1731` 留了 `TODO(M2)`：resolver runtimes 没接 checkpoint store —— M2b 一并接 store + gate）。启动加载未决票据。全仓门禁 + WSL race 全绿。

> M2b 大且硬（尤其任务 3/4）。写 plan 时若某任务过大可再拆。任务 1 是安全闸，务必最先。

## 5. 关键代码指针（已核对，行号约数）

- `internal/runtime/runtime.go`：`RunTask`(@~183 入口/resume 分支)、`runToolLoop`(@~258)、`checkSuspend`(@~352)、`finishRun`(@386，plan 写 @~444)、`effectiveTools`、`generate`(@426)、`inferenceTools`(@461)、`executeToolCalls`(@483)、`buildPrompt`/`toolNames`(@~702-780)。
- `internal/runtime/lazytools.go`：`dispatchToolCall`（唯一 choke，M2a 已加 `tools` 参数）、`dispatchCallTool`（lazy 真实工具在此解包，`tool_name`/`arguments_json`）、`metaToolCallTool`/`metaToolListTools`、`listToolsCatalog(query,tools)`。
- `internal/runtime/coordinator.go`：`Heartbeat`(@94，加 resume 扫描)、`runAssigned`(@136，挂起路径 @199/@204，锁 `TryLock`@139)、`RecoverSuspended`(@274)。
- `internal/task/scheduler.go`：`Next`(@42，**只挑 Pending**)、`Transition`(@90)、`canTransition`(@108，Suspended↔Running 已允许；方案 B 无需改转移表；若某处要 Suspended→Pending 才需加)。
- `internal/approval/service.go`：`Service`/`Ticket`/`OpenTicket`/`Decide`/`DecideBy`（现纯内存）。
- `internal/tool/registry.go`：`Descriptor.Sensitive`、`SafeToolNames`、`Subset`、`Execute`（`ErrToolNotFound` 在此）。
- `internal/server/http.go`：路由 `ServeHTTP`(@~199)、`handleCreateTask`(@~654)、`handleResumeWorkflow`(@1011，workflow 的、非 task resume，别混)、`Config`(@75，有 `PlatformEvents *observability.EventBus` @82 —— 那是 M2c SSE)。测试骨架：`NewHTTPServer(Config{Sessions:repo,Tasks:scheduler})` + `httptest` + `srv.ServeHTTP`；repo = sqlite（`OpenSQLite`）。
- `internal/cli/command.go`：serve 装配，默认运行时 `NewRuntime`(@~1797)、`NewCoordinator`(@~1807)、resolver `NewAgentRuntimeResolver`(@1730，`TODO(M2)` 在此)、`checkpointStore` 构造(@~1795)、`RecoverSuspended` 调用(@~1884)、logger(@~1871)、listener(@net.Listen ~1871)。
- `internal/runtime/agent_resolver.go`：`ResolveTaskRunner`(@58) 建 per-agent runtime（`NewRuntime` @94，**没接 Checkpoints/ToolGate** —— 任务 7 要接）。

## 6. 环境 + 工作流（照 M1b/M2a）

- **`-race` 只能 WSL 跑**（Windows 无 gcc）。配方：
  ```bash
  wsl -d Ubuntu-22.04 -- bash -lc 'cd /mnt/f/source/stardust/Legion/legion/legionAgent && GOOS=linux GOARCH=amd64 CGO_ENABLED=1 CC=gcc $HOME/sdk/go/bin/go test -race ./internal/runtime/ ./internal/task/ ./internal/approval/ ./internal/server/ ./internal/sessionstate/'
  ```
  Windows 宿主：普通 `go test ./...`（go 1.26 在 PATH）。WSL go 1.26.0 在 `$HOME/sdk/go`，gcc 11.4，已验证可用。
- **完成门禁**：`go build ./... && go vet ./... && go test ./...` 全绿；`gofmt -l` 触碰文件内容 LF 干净（仓库 `core.autocrlf=true`，`gofmt -l` 会误报既有 CRLF 文件 —— 靠剥 `\r` 验证内容干净，别整文件重排）。
- **Fail-Loud 铁律**（legionAgent/CLAUDE.md §0）：禁 fallback/zero-value 兜底、禁 `_ = err`、禁静默跳过。审批 deny = 契约内的拒绝结果回模型（非兜底）。盘写失败/损坏票据 → 报错不静默。
- **SDD 流程**（superpowers:subagent-driven-development）：先 `git checkout -b feat/agent-modes-m2b`（从 master）；ledger 在 `legionAgent/.superpowers/sdd/progress.md`（gitignore scratch，可覆盖前里程碑）；每任务：手工抽 brief 写 `.superpowers/sdd/task-N-brief.md`（**task-brief 脚本按英文 "Task N" 匹配，中文 plan 标题「任务 N」它认不出 → 手工抽**）→ 派 executor(sonnet) 实现 → 生成 review-package(`scripts/review-package BASE HEAD`)→ 派 code-reviewer 审 → 修 Critical/Important → 记 ledger。核心/安全任务(3/4)用 opus reviewer。最终整分支 opus final review → finishing-a-development-branch → PR(中文正文)+合并。
- **⚠️ 子 agent 最终消息被 OMC/caveman hook 吞** → 让实现者/审查者**报告写文件**（`.superpowers/sdd/task-N-report.md` / `task-N-review.md`），控制方读文件核验，别信返回的 chat 消息。诊断显示的编译错误常是 mid-TDD-red 快照，务必自己 `go build` 确认最终态。
- 实现者 brief 里要点：真实测试骨架/签名先核对（脚本 line 号是约数）；标注哪些是"核对真实代码"点；给出精确 fail-loud/门禁/commit 要求。

## 7. 待办（跨里程碑，非 M2b 必须但记牢）

- **agent.json 密钥**：`legionAgent/agent.json` 曾含实时 deepseek key 进过旧 remote 历史，**需去 deepseek 轮换**；新 remote 干净（agent.json/db 已 gitignore）。见 [[legion-git-repo-topology]]。
- M2b 遗留的 Minor（M2a final review 提）：resolver runtimes 接 store 后 Plan 也能落 OKF（任务 7 顺带）；`buildPrompt`/`toolNames` 用全集 → Plan lazy hint 列敏感名（纯 hygiene，可线程 tools 进 buildPrompt 修，非 M2b 必须）。
- 既有死代码 `func max`(runtime.go)、`errStaticResolver`(coordinator_test.go) —— 可另开清理任务。

## 8. 本轮（2026-07-18）工作日志

- 恢复 M1b 会话 → subagent-driven 执行 M1b（6 任务，9 提交 26f4cfe..af96db2，opus final MERGE READY）→ PR #1 → 用户手动合入 master `4f5abd0`。
- 拆 M2=M2a/M2b/M2c；写 M2a plan（中文）→ subagent-driven 执行 M2a（6 任务，7 提交 663cbe6..5653000，opus final MERGE READY，安全不变量对抗性验证+测试固化）→ PR #2 → `gh pr merge` 合入 master `e644b20`。
- 进入 M2b：探明设计（round 级挂起 / lazy peek / resume 派发缺口）→ 用户拍板 resume 派发用**方案 B**、开新会话续 M2b → 写本交接文件。

**接续入口（新会话）**：读本文件 → 读 spec §4.3 → `/writing-plans` 出 M2b plan（§4 的 7 任务，任务 1 安全前置最先）→ subagent-driven（§6 流程）执行。
