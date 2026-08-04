---
title: 会话交接 — Phase 2（工作模式+工作目录+真并发）全交付 + post-merge 待办
type: handoff
status: active
created: 2026-07-19
scope: legion/legionAgent（后端+TUI）+ legion/legionAgentGUI（GUI）
related:
  - "[[2026-07-15-agent-working-modes-design]]"
  - "[[2026-07-18-m2c-handoff-sse-events]]"
tags: [agent, working-mode, working-dir, concurrency, phase2, postmerge, handoff]
---

# 会话交接 · Phase 2 全交付 + post-merge 待办（新会话从零接续）

> 本文件是**新会话接续的全部上下文**：Phase 2 已全部交付并合入，剩下的是 12 个非阻塞 post-merge task chip + 2 组人工验证。读完即可挑任一 chip 执行，或做人工验证。

## 0. 一句话现状

Phase 2「Agent 工作模式（Manual/Plan/Auto）+ per-session working_dir + 真并发/可挂起工具循环」**全部 8 个里程碑已交付并合入 master**（M1a/M1b/M2a/M2b/M2c/M3a/M3b/M3c + 2 个附带修复）。三前端（HTTP serve / GUI / TUI）工作模式 + working_dir + 审批能力**对等**。剩余仅非阻塞 post-merge 项（12 个 task chip）+ 桌面/终端人工验证。

## 1. 三仓库拓扑（不变）

`F:\source\stardust\Legion` 根**不是** git 仓库；三个独立仓库各有 github remote：
- **server + TUI** = `legion/legionAgent/`，remote `github.com/jxncyjq/stardust-agent-server`（旧 URL `jxncyjq-stardust-agent-server` 仍可 push，会 redirect）。**master HEAD = `ab0b8c2`**（Merge PR #8）。
- **GUI** = `legion/legionAgentGUI/`，remote `github.com/jxncyjq/stardust-agent-gui`，独立 module，经 `replace github.com/stardust/legion-agent => ../legionAgent` 依赖后端；**不得 import server `internal/**`**（走 Go 侧 HTTP + serve 公共包）。**master HEAD = `3158a31`**（Merge PR #1）。Wails v2 + React 18 + TS + Zustand + Tailwind + Vite + vitest。
- docs = `docs/`（本文件所在），独立仓库，AI 改动通常不自动提交。
- `legion/.git` 遗留聚合仓库**勿用**。
- 改代码前 `git rev-parse --show-toplevel` 确认在哪个仓库。

## 2. Phase 2 全景（里程碑 + 合并状态）

| 里程碑 | 内容 | 合并 |
|---|---|---|
| M1a/M1b | 真并发 + 可挂起/恢复工具循环 + 会话目录持久化 + 重启恢复 | ✅ master（早期） |
| M2a | session mode（manual/plan/auto）+ 工具 Sensitive 位 + Plan OKF | ✅ master |
| M2b | Manual 审批 gate（suspend/resume + 会话目录 JSON 票据 + 超时 sweep） | ✅ master |
| M2c | 接活 /v1/events SSE + 桥接审批/生命周期事件 | ✅ PR #4 |
| M3a | working_dir 后端（per-session 沙箱 + 会话目录随 working_dir + 多 base 恢复 + set-once） | ✅ PR #5 |
| （附带）| SQLite busy_timeout 防 SQLITE_BUSY | ✅ PR #6 |
| （附带）| observability.EventBus 环形上限 + 重放取最近 | ✅ PR #7 |
| M3b | GUI 适配器（模式选择器 + working_dir + SSE 审批 UI，4 Go 绑定 + 接活 sse_bridge） | ✅ GUI PR #1 |
| M3c | TUI 适配器（/mode /cwd + 状态栏 + working_dir 沙箱 + **方案 Y** 终端审批） | ✅ PR #8 |

**门禁现状（两 master 均已独立验证）**：legionAgent `go build/vet/test ./...` 全绿 + WSL race 绿；GUI `go build/vet` + 前端 `vitest`(52) + `npm run build` 全绿。两仓库工作树干净。

## 3. 环境 + 门禁命令

- **`-race` 只能 WSL**（Windows 无 gcc）。**必须显式 `GOOS=linux GOARCH=amd64`**（WSL Go 工具链持久化了 GOOS=windows）：
  ```bash
  wsl -d Ubuntu-22.04 -- bash -lc 'cd /mnt/f/source/stardust/Legion/legion/legionAgent && GOOS=linux GOARCH=amd64 CGO_ENABLED=1 CC=gcc $HOME/sdk/go/bin/go test -race ./internal/tui/ ./internal/app/ ./internal/cli/ ./internal/runtime/ ./internal/server/ ./internal/observability/ ./internal/manualgate/ ./internal/approval/ ./internal/sessionstate/ ./internal/storage/'
  ```
- Windows 门禁：legionAgent `go build ./... && go vet ./... && go test ./...`；GUI 根 `go build ./... && go vet ./...` + `cd frontend && npm test && npm run build`。
- gofmt：仓库 `core.autocrlf=true`，`gofmt -l` 对既有 CRLF 文件误报——只 `gofmt -w` 自己改的文件。
- 子 agent 最终消息常被 hook 吞 → 让其**报告写文件**再读核验。

## 4. Post-merge task chip（12 个，已 spawn，非阻塞）

> 每个 chip 自带完整上下文（文件指针 + 来源 review + cwd + 门禁）。可单点开新 worktree 执行。若 chip 已失效（app 重启后 chip 不持久），按下表信息重建。

### legionAgent（cwd `F:/source/stardust/Legion/legion/legionAgent`）

| # | 标题 | 要点 | 来源 |
|---|---|---|---|
| B1 | TUI 审批接 per-task 可取消 ctx（runtime+model 同批） | `interactive.go` run() 现传 `context.Background()` 使 gate ctx 逃生休眠；未来接 cancelable ctx 时，gate（approvalgate.go Resolve）+ model（`sendApprovalDecision` 加 select ctx/doneCh + 清 approvalActive）**必须同批**，否则 ctx 取消致 decisionCh 永久阻塞泄漏 | M3c final-review #2 |
| B2 | TUI 审批期锁鼠标滚轮 | `interactive.go` Update 的 `tea.MouseMsg` 分支排在 approvalActive 键盘拦截前，审批期滚轮可把审批框滚出视野（y/n 仍锁，纯 UX）；approvalActive 时 MouseMsg `return m,nil` | M3c final-review #3 |
| B3 | /mode /cwd 缺参补回归测试 | `handleModeCommand/handleCwdCommand` 的 `len(fields)<2` guard 无显式测试；补 MissingArg 用例断言 model.err + 不 panic | M3c T2 |
| B4 | 修 /v1/audit-events /v1/runtime-events 空闲 SSE 挂起 | `handleAuditEvents`/`handleRuntimeEvents` 疑同款「订阅前不发头→空闲挂起」（statusRecorder.Flush 已给能力）；仿 events.go handleEvents 修（若确是长连 SSE） | M2c T2 遗留 |
| B5 | SSE truncateEventString 按 rune 截断 | `server/events.go` 按字节切多字节 UTF-8（json 替 U+FFFD 无害）→ 按 rune / ToValidUTF8 | M2c T4#2 |
| B6 | OpenAPI 登记 /v1/approvals 等缺失端点 | `server/openapi.go` BuildOpenAPISpec 未登记 /v1/approvals 及其它既有端点 | M2c T5#2 |
| B7 | per-agent 补 session_search/MoA 工具 | `agent_resolver.go` ResolveTaskRunner 只注册 ledger/message/web，默认路径有全套 6 工具（不对称，pre-existing）；评估对齐或注释 | M3a T7 |
| B8 | 清理死代码/lint | `runtime.go` 未用 `max`、`coordinator_test` `errStaticResolver`、`interactive.go` statusText/renderPanel 未用 + MouseWheel deprecated/atomictypes、command.go writestring/QF1012/stringsbuilder、各 test rangeint/slicescontains（改 MouseWheel 前查 bubbletea 版本 API） | M2c handoff §7 |

### GUI stardust-agent-gui（cwd `F:/source/stardust/Legion/legion/legionAgentGUI`）

| # | 标题 | 要点 | 来源 |
|---|---|---|---|
| C2 | consumeSSE 设 scanner.Buffer 上限 | `sse_bridge.go` bufio.Scanner 默认 64KB 行缓冲，超长行 ErrTooLong→静默变重连；显式设上限 1MB + 可诊断 | M3b Issue 1 |
| C3 | 补 SSE 重连循环 + useAgentEvents 单测 | 重连「逐轮重读 baseURLFn→连新端口」仅代码走查；useAgentEvents agent:approval 接线/parse 失败仅间接覆盖 | M3b T2/T5 |
| C4 | 修 Sidebar loadSessions 静默 catch{} | `Sidebar.tsx:77` catch{} 全静默吞错（pre-existing）→ console.debug/warn | M3b Issue 4 |
| C5 | ApprovalPrompt deciding 成功后复位 | 成功只 onResolved 移除，未复位 deciding[ticket_id]（当前无害，理论边界按钮永久 disabled） | M3b T5 |

## 5. 人工验证（你亲跑，自动化门禁覆盖不到桌面/终端）

### A1. GUI `wails dev`（legionAgentGUI）
1. 模式选择器切 Manual/Plan/Auto → 切会话保持 per-session。
2. `+` 菜单选目录 → chip 显示 → 新任务工具沙箱在该目录内。
3. Manual 发敏感任务 → 审批 UI 弹（工具名+参数）→ 批准继续 / 拒绝走拒绝分支。
4. serve 重启（设置 SaveAll）→ SSE 重连（`serve:sse`）→ 审批仍收到。**（同时验证 C2/C3 相关：重连是否秒级——依赖后端 ServeManager.Restart 用 http.Server.Close 断在途连接）**
5. working_dir 已设再改 → set-once 400 提示。

### A2. TUI `legion tui`（legionAgent）
1. `/mode manual|plan|auto` → 状态栏反映 + per-session（切会话保持）。
2. `/cwd <dir>` → 沙箱生效；`/cwd 非目录` → 报错；改已设 → set-once 报错。
3. `/mode plan` → 只调研不执行副作用工具。
4. `/mode manual` → 终端弹 y/n → y 执行 / n 拒绝；**审批 pending 时 Ctrl-C 退出不卡死**（今日靠进程退出回收，B1 修复前无 ctx 逃生）。

## 6. 关键契约（跨仓库对齐，改动勿破）

- **SSE 事件类型全下划线**（与 `domain.RuntimeEvent.Type` 一致）：生命周期 `task_started`/`tool_call_requested`/`tool_result`/`tool_executed`/`tool_failed`/`inference_completed`/`task_completed`；审批 `approval_pending{task_id,ticket_id,tool,arguments}`/`approval_resolved{task_id,ticket_id,decision}`。
- **审批 decision 用动词** `approve`/`deny`（POST /v1/tasks/{id}/approvals/{ticketID}）——**≠** approval_resolved 事件回的 `approved`/`denied`（过去式）。
- **键名差异**：SSE pending 的工具名键 = `tool`；`GET /v1/approvals?status=pending` 返回 = `tool_name`。GUI normalizeTicket 用 `tool ?? tool_name` 归一。
- **working_dir set-once**：session working_dir 一经设为非空，改成不同值 → 400（serve HTTP 层 handlePatchSession + TUI SetWorkingDir 都强制，防挂起 checkpoint/票据 strand 在旧 base 后重启静默丢）。
- **会话目录 base**：单一 resolver `sessionstate.SessionBase(workspaceRoot, workingDir)` = workingDir 非空 → `<workingDir>/.stardust`，否则 workspaceRoot。checkpoint/审批票据/plans 落 `<base>/session/<sessionKey>/`。重启恢复枚举 SessionStore 全部 working_dir 得 base 集合逐个扫。
- **两套审批实现（勿混）**：serve = `internal/manualgate` ManualToolGate（落盘 suspend/resume + ToolGateStore + 跨重启 + SSE approval_pending）；TUI = `internal/tui/approvalgate.go` **方案 Y** 同步阻塞 gate（Resolve 阻塞等答，不落盘/不跨重启/不进 ToolGateStore，前台单用户）。TUI task 无 SessionID → checkpoint 空间与 serve 不相交。
- **Plan 模式只读**：`runtime.effectiveTools(task)` 按 task.Mode==plan 返回 SafeToolNames 子集，offer（给模型）+ dispatch（实际执行）同一 effTools 变量，无「看不到但能 dispatch」旁路。

## 7. SDD 产物位置

- legionAgent：`legionAgent/.superpowers/sdd/`（progress.md 各里程碑总账被覆盖，当前是 M3c；final-review.md、task-N-report/review.md、m3-explore-{workingdir,gui,tui}.md 探索报告）。`.superpowers` gitignore。
- GUI：`legionAgentGUI/.superpowers/sdd/`（M3b 产物）。
- 计划：`docs/superpowers/plans/2026-07-{18,19}-agent-modes-m{2c,3a,3b,3c}-*.md`。

## 8. 独立遗留（非 Phase 2，运维项）

- `legionAgent/agent.json` 的实时 deepseek key 曾进旧 remote 历史 → **需去 deepseek 轮换**。见 [[legion-git-repo-topology]]。

## 9. 新会话接续动作

1. 读本文件。
2. 挑一个 post-merge chip（§4 表），`git rev-parse --show-toplevel` 确认仓库 → 从 master 切 `fix/<slug>` 分支 → TDD 实现 → 门禁（含 WSL race 若涉并发）→ PR（中文）。
3. 或做 §5 人工验证（`wails dev` / `legion tui`）。
4. 多个独立 chip 可并行（各自 worktree）。
