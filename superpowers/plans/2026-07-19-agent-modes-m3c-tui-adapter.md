---
title: 实施计划 — Milestone 3c（TUI 适配器：/mode /cwd + 状态栏 + working_dir 沙箱 + 终端内审批）
type: plan
status: active
created: 2026-07-19
scope: legion/legionAgent（internal/tui + internal/app + internal/cli）
related:
  - "[[2026-07-15-agent-working-modes-design]]"
  - "[[m3-explore-tui]]"
tags: [agent, tui, working-mode, working-dir, approval, milestone-3c, plan]
---

# Agent Working Modes — Milestone 3c (TUI 适配器) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax.

**Goal:** 让交互式 TUI（`internal/tui/interactive.go`）支持 per-session 工作模式（`/mode`）与工作目录（`/cwd`）、状态栏常驻显示、任务工具沙箱锁 working_dir，并在 Manual 模式下用**进程内同步阻塞 gate**（方案 Y）在终端就地弹批准/拒绝。这是 Phase 2 的最后一块，使 TUI 与 GUI 前端能力对等。

**Architecture:** TUI 会话状态复用 `conversationStore` 的 `domain.AgentSession`（SQLite，已有 `Mode`/`WorkingDir` 列，M2a/M3a 加）——`SessionManager` 加 mode/cwd 存取，与 serve 同一份真相源。`/mode`/`/cwd` 走 interactive.go 现有 slash 分发链（`domain.NormalizeMode` 校验 mode；`os.Stat`+`IsDir` 校验 cwd）。`runTUITask` 把 session 的 mode/working_dir 解析进 `RunTaskOptions`→`domain.Task`；`app.RunTask` 用 `task.WorkingDir` 作工具沙箱根（仿 M3a agentToolRoot），按 `task.Mode` 应用 Plan 只读子集。**审批方案 Y**：给 `RunTaskOptions` 加 `ToolGate`，实现一个 TUI 专用同步 gate——`ShouldSuspend` 恒返 false（不走 suspend/checkpoint），`Resolve` 在 Manual+敏感工具时**经 channel 把待决请求推给 bubbletea 主循环 → 终端 prompt → 阻塞等用户答 → 返 allow/deny**（select on ctx.Done() 防退出死锁）。不落盘、不跨重启、不进 ToolGateStore——适合 TUI 前台单用户同步交互（用户就在场）。

**Tech Stack:** Go 1.26；bubbletea + lipgloss（TUI）；`internal/runtime.ToolGate` 接口；`internal/domain`（Mode/WorkingDir 已就绪）；`internal/app`（RunTaskOptions/RunTask）；`internal/cli`（tuiSessionController/runTUITask）；`internal/tool`（WorkspaceRegistry 沙箱根）；WSL race（方案 Y 有 channel/goroutine 并发）。

## Global Constraints

- **审批方案 = Y（已拍板）**：进程内同步阻塞 gate，Resolve 阻塞等答，不落盘/不跨重启/不进 ToolGateStore。与 serve 的 ManualToolGate（落盘 suspend/resume）是**两套独立实现**，别混用。方案 Y 偏离 spec §4.0/4.3 的统一票据/落盘语义——这是本里程碑显式接受的取舍（spec §4.7「像 hermes CLI callback」支持之），因 TUI 前台单用户同步、无「重启后回来」场景。
- **持久化统一**：mode/working_dir 存 `conversationStore` 的 `AgentSession`（SQLite mode+working_dir 列，serve 读同一份）——**不另起 TUI 专属存储**（避免双真相源，违 fail-loud/单一 resolver）。
- **Fail-Loud 铁律**（`legionAgent/CLAUDE.md §0`）：`/mode` 非法值→报错（`model.err` 非空）不静默默当 auto；`/cwd` 非目录/不存在→报错不静默；审批 gate 的 ctx 取消→返回 error 不吞；禁兜底/`_ = err`。
- **working_dir 校验**：`/cwd <path>` 非空则 `os.Stat`+`IsDir`，否则报错（spec §4.4 fail-loud）。
- **沙箱根**：`task.WorkingDir` 非空 → 工具文件操作锁其内（`tool.NewWorkspaceRegistry` root=working_dir，越界 `WorkspacePathGuard` 拒），仿 M3a。
- **Plan 模式**：`task.Mode==plan` → 工具集只读子集（read_file/search_content/list_files 等），副作用工具不可达（仿 M2a serve 侧行为——核实 runtime 是否已按 task.Mode 应用；若 app.RunTask 路径未应用需补）。
- **公开 API Go doc**（以标识符名开头）。gofmt 只碰改动文件（仓库 `core.autocrlf=true`）。
- **完成门禁**：Windows `go build ./... && go vet ./... && go test ./...` 全绿；WSL `-race` 绿（方案 Y 的 gate channel/goroutine 务必 race）。
- **仓库**：全部代码在 `legion/legionAgent`（master，已含 M2c/M3a/EventBus）。分支 `feat/agent-modes-m3c`（从 master `7e91a33` 切）。

## WSL race recipe
```bash
wsl -d Ubuntu-22.04 -- bash -lc 'cd /mnt/f/source/stardust/Legion/legion/legionAgent && GOOS=linux GOARCH=amd64 CGO_ENABLED=1 CC=gcc $HOME/sdk/go/bin/go test -race ./internal/tui/ ./internal/app/ ./internal/cli/ ./internal/runtime/'
```

---

## File Structure

- **Modify** `internal/cli/command.go` — `tuiSessionController` 加 mode/cwd 存取（读写 AgentSession.Mode/WorkingDir）；`SessionManager` 接口扩容；`runTUITask` 解析 session mode/cwd 进 RunTaskOptions。Task 1/4。
- **Modify** `internal/tui/interactive.go` — `interactiveCommands` 加 `/mode`/`/cwd`；`handleModeCommand`/`handleCwdCommand` 分发；model 加 `mode`/`workingDir` 字段；状态栏（renderFooter/renderPlan）显示；审批视图 + 消息类型。Task 2/3/5。
- **Modify** `internal/app/app.go` — `RunTaskOptions` 加 `Mode`/`WorkingDir`/`ToolGate`；`RunTask` 用 WorkingDir 作工具根、按 Mode 应用 Plan 子集、传 ToolGate 给 runtime。Task 4/5。
- **Create** `internal/tui/approvalgate.go`（或 internal/tui 内）— 方案 Y 同步阻塞 gate（实现 runtime.ToolGate）。Task 5。
- 各配套 `_test.go`（interactive_test.go / command_test.go / app 层）。

---

### Task 1: SessionManager mode/cwd 存取（conversationStore AgentSession）

**Files:**
- Modify: `internal/tui/interactive.go`（`SessionManager` 接口 @105-111）
- Modify: `internal/cli/command.go`（`tuiSessionController` @870，NewSession @935）
- Test: `internal/cli/command_test.go`（tuiSessionController mode/cwd 存取）

**Interfaces:**
- Consumes: `domain.AgentSession.Mode`/`.WorkingDir`（已有）；`conversationStore`（SQLite 持久化 AgentSession）；`domain.NormalizeMode`（校验）。
- Produces: `SessionManager` 加 `CurrentMode() string` / `SetMode(ctx, mode string) error` / `CurrentWorkingDir() string` / `SetWorkingDir(ctx, dir string) error`；`tuiSessionController` 实现（读写当前 session 的 AgentSession.Mode/WorkingDir 并持久化）。

- [ ] **Step 1: 写 tuiSessionController mode/cwd 存取失败测试**

`internal/cli/command_test.go` 追加（仿现有 session controller 测试 @774-840）：

```go
func TestTUISessionControllerSetAndGetMode(t *testing.T) {
	// 构造 tuiSessionController（用现有测试骨架的 conversationStore/in-memory）
	// NewSession → SetMode(ctx,"manual") → CurrentMode()=="manual"
	// 且持久化：重新 Get session 断言 AgentSession.Mode=="manual"
	// SetMode(ctx,"bogus") → 返回 error（NormalizeMode fail-loud）
}
func TestTUISessionControllerSetAndGetWorkingDir(t *testing.T) {
	// SetWorkingDir(ctx, t.TempDir()) → CurrentWorkingDir()==该目录 + 持久化
}
```

> 先读现有 tuiSessionController 测试骨架（command_test.go:774-840）+ conversationStore 接口对齐，勿臆造。

- [ ] **Step 2: 跑确认失败** → **Step 3: 实现**

- `SessionManager` 接口（interactive.go:105-111）加 `CurrentMode()`/`SetMode(ctx,mode)`/`CurrentWorkingDir()`/`SetWorkingDir(ctx,dir)`。
- `tuiSessionController`（command.go:870）实现：`SetMode` 先 `domain.NormalizeMode(mode)` 校验（非法返 error），合法则更新当前 session 的 `AgentSession.Mode` 并经 conversationStore 持久化；`CurrentMode` 读当前 session 的 Mode（空返 "auto"？或空字符串——与 NormalizeMode 一致，空=auto）。`SetWorkingDir` 校验非空则 `os.Stat`+`IsDir`（否则 error），持久化 `AgentSession.WorkingDir`；`CurrentWorkingDir` 读之。
- **注意**：conversationStore 需支持更新 session 的 mode/working_dir（若现有 Save/Update AgentSession 覆盖全字段则直接用；确认持久化路径）。

- [ ] **Step 4: 跑确认通过**

Run: `go build ./... && go test ./internal/cli/ -run TestTUISessionController`
Expected: PASS。

- [ ] **Step 5: gofmt + 提交**

```bash
gofmt -w internal/tui/interactive.go internal/cli/command.go internal/cli/command_test.go
git add internal/tui/interactive.go internal/cli/command.go internal/cli/command_test.go
git commit -m "feat(tui): SessionManager 加 mode/cwd 存取(持久化 AgentSession)"
```

---

### Task 2: /mode /cwd slash 命令

**Files:**
- Modify: `internal/tui/interactive.go`（interactiveCommands @136-152、Enter 分发链 @346-397、加 handleModeCommand/handleCwdCommand；model 加 mode/workingDir 字段 @47-89）
- Test: `internal/tui/interactive_test.go`

**Interfaces:**
- Consumes: Task 1 的 `SessionManager.SetMode`/`SetWorkingDir`/`CurrentMode`/`CurrentWorkingDir`；`domain.NormalizeMode`。
- Produces: `/mode manual|plan|auto`、`/cwd <path>` 命令；model `mode string`/`workingDir string` 字段供状态栏读。

- [ ] **Step 1: 写 slash 命令失败测试**

`internal/tui/interactive_test.go` 追加（仿 `TestInteractiveModelSessionCommandsUseSessionManager` @523 + fakeSessionManager @1191）：

```go
func TestInteractiveModelModeCommandSetsMode(t *testing.T) {
	// fakeSessionManager 加 SetMode/CurrentMode；输入 "/mode manual" + Enter
	// 断言 manager.SetMode 被调("manual") + model.mode=="manual"（供状态栏）
}
func TestInteractiveModelModeCommandRejectsInvalid(t *testing.T) {
	// "/mode bogus" → model.err 非空（fail-loud）；manager.SetMode 未持久化非法值
}
func TestInteractiveModelCwdCommandSetsWorkingDir(t *testing.T) {
	// "/cwd <t.TempDir()>" → SetWorkingDir 被调 + model.workingDir 更新
	// "/cwd <不存在路径>" → model.err 非空
}
```

> 先读 fakeSessionManager（interactive_test.go:1191）扩 SetMode/CurrentMode/SetWorkingDir/CurrentWorkingDir。

- [ ] **Step 2: 跑确认失败** → **Step 3: 实现**

- `interactiveCommands`（:136）加 `{Name:"/mode ", Description:"设置工作模式 manual|plan|auto"}`、`{Name:"/cwd ", Description:"设置工作目录 <path>"}`（尾空格=带参）。
- model（:47-89）加 `mode string`、`workingDir string` 字段；`NewInteractiveModel` 初始化时从 SessionManager.CurrentMode/CurrentWorkingDir 读。
- Enter 分发链（:368 附近，与 handleSessionCommand 并列）加 `handleModeCommand`/`handleCwdCommand`（返回 `(handled bool, next InteractiveModel)`）：
  - `handleModeCommand`：`fields[0]=="/mode"` → `SetMode(ctx, fields[1])`；err → 设 model.err（fail-loud 提示）；成功 → 更新 model.mode。
  - `handleCwdCommand`：`/cwd` → `SetWorkingDir(ctx, fields[1])`；err→model.err；成功→model.workingDir。

- [ ] **Step 4: 跑确认通过**

Run: `go build ./... && go test ./internal/tui/ -run 'TestInteractiveModel(Mode|Cwd)Command'`
Expected: PASS。

- [ ] **Step 5: gofmt + 提交**

```bash
gofmt -w internal/tui/interactive.go internal/tui/interactive_test.go
git add internal/tui/interactive.go internal/tui/interactive_test.go
git commit -m "feat(tui): /mode /cwd slash 命令(NormalizeMode+目录校验 fail-loud)"
```

---

### Task 3: 状态栏 mode + cwd 显示

**Files:**
- Modify: `internal/tui/interactive.go`（renderFooter @832、可选 renderPlan @752）
- Test: `internal/tui/interactive_test.go`（View 含 mode/cwd）

**Interfaces:**
- Consumes: model `mode`/`workingDir` 字段（Task 2）。
- Produces: 状态栏常驻显示当前模式 + 工作目录。

- [ ] **Step 1: 写状态栏失败测试**

```go
func TestInteractiveModelFooterShowsModeAndCwd(t *testing.T) {
	// 设 model.mode="manual"、workingDir="/proj/app"（经 WindowSizeMsg 设尺寸）
	// 断言 model.View() 含 "manual" 与 cwd（或其 basename）
}
```

- [ ] **Step 2-3: 实现**

`renderFooter`（:832-836，左侧 `agentName · modelName · sessionID` 拼接串）加 `· <mode> · <cwd>`（cwd 可截断显 basename）；仿 sessionID 先例。可选把 `renderPlan`（:752）「Plan」标签位标题改为动态显示当前 mode（spec §4.7 明示可复用）。空 mode 显 "auto"，空 cwd 显 "(default)" 或省略。

- [ ] **Step 4: 跑确认通过**

Run: `go test ./internal/tui/ -run TestInteractiveModelFooter`

- [ ] **Step 5: gofmt + 提交**

```bash
gofmt -w internal/tui/interactive.go internal/tui/interactive_test.go
git add internal/tui/interactive.go internal/tui/interactive_test.go
git commit -m "feat(tui): 状态栏常驻显示 mode + cwd"
```

---

### Task 4: RunTaskOptions + app.RunTask 应用 Mode(Plan 子集) + WorkingDir(沙箱根)

**Files:**
- Modify: `internal/app/app.go`（RunTaskOptions @34-56、RunTask @150-249：runtime 构造 @189、task 构造 @199）
- Modify: `internal/cli/command.go`（runTUITask @446-488：解析 session mode/cwd 进 RunTaskOptions）
- Test: `internal/app/app_test.go`（若有）+ `internal/cli/command_test.go`

**Interfaces:**
- Consumes: Task 1 的 session mode/cwd；`domain.Task.Mode`/`.WorkingDir`；`tool.NewWorkspaceRegistry`（沙箱根）；runtime 的 Plan 子集逻辑。
- Produces: `RunTaskOptions` 加 `Mode string`/`WorkingDir string`；`app.RunTask` 把它们写进 `domain.Task`、用 WorkingDir 作工具根、按 Mode 应用 Plan 只读子集；`runTUITask` 从 session 解析 mode/cwd。

- [ ] **Step 1: 写 working_dir 沙箱 + Plan 子集失败测试**

`internal/app/app_test.go`（仿 M3a 的沙箱测试风格——用 toolProbingMaas 驱动真 read_file 穿 WorkspacePathGuard）：

```go
func TestRunTaskSandboxesToolsToWorkingDir(t *testing.T) {
	wd := t.TempDir()
	// 在 wd 写文件；RunTask(RunTaskOptions{WorkingDir: wd, ...}) 工具 read_file wd 内文件成功、../ 越界拒 ErrPathOutsideWorkspace
}
func TestRunTaskPlanModeRestrictsToReadOnlyTools(t *testing.T) {
	// RunTaskOptions{Mode:"plan"} → 断言副作用工具(write_file 等)不在工具集/不可达
}
```

> 先读 app.go RunTask 现有 tool root 构造（`tool.NewWorkspaceRegistry(toolRoot,...)` 约 app.go:180-185）+ M3a 的 agentToolRoot 沙箱测试风格对齐。**先确认 runtime 是否已按 task.Mode 应用 Plan 子集**（若 effectiveTools 已 keys off task.Mode，则传 Mode 进 task 即自动生效；否则需在 app.RunTask 或 runtime 补）——读 runtime effectiveTools/Plan 逻辑，报告结论。

- [ ] **Step 2: 跑确认失败** → **Step 3: 实现**

- `RunTaskOptions`（app.go:34）加 `Mode string`、`WorkingDir string`。
- `RunTask`：`domain.Task` 构造（:199）加 `Mode: opts.Mode`、`WorkingDir: opts.WorkingDir`；工具根：`root := opts.WorkingDir; if root=="" { root = opts.ToolRoot }`（或现有 fallback），`tool.NewWorkspaceRegistry(root,...)`；Plan 子集：确保 runtime 按 task.Mode 应用（若自动则无需额外；否则应用只读 Subset）。
- `runTUITask`（command.go:446）：从当前 session 解析 `Mode = sessionManager.CurrentMode()`、`WorkingDir = sessionManager.CurrentWorkingDir()` 填入 RunTaskOptions。

- [ ] **Step 4: 跑确认通过**

Run: `go build ./... && go test ./internal/app/ ./internal/cli/ ./internal/runtime/`

- [ ] **Step 5: gofmt + 提交**

```bash
gofmt -w internal/app/app.go internal/app/app_test.go internal/cli/command.go internal/cli/command_test.go
git add internal/app/ internal/cli/command.go internal/cli/command_test.go
git commit -m "feat(tui): app.RunTask 应用 session mode(Plan 子集) + working_dir 沙箱根"
```

---

### Task 5: 方案 Y — 进程内同步阻塞审批 gate + bubbletea 终端 prompt

**Files:**
- Create: `internal/tui/approvalgate.go`（TUI 同步 gate，实现 runtime.ToolGate）
- Modify: `internal/app/app.go`（RunTaskOptions 加 `ToolGate`；RunTask 传给 runtime.Config.ToolGate）
- Modify: `internal/tui/interactive.go`（审批消息类型 + 终端 prompt 视图 + 决定回传）
- Modify: `internal/cli/command.go`（runTUITask：Manual 模式时构造 gate 接入 bubbletea 通道）
- Test: `internal/tui/approvalgate_test.go`、`internal/tui/interactive_test.go`

**Interfaces:**
- Consumes: `runtime.ToolGate` 接口（`ShouldSuspend(ctx,task,calls,tools)(bool,error)` + `Resolve(ctx,task,call,tools)(bool,error)`）；`tool.Registry.Descriptors().Sensitive`；bubbletea 消息机制（streamCh 先例 interactive.go:396/1467）。
- Produces: `tui` 同步 gate 类型（ShouldSuspend 恒 false；Resolve 在 Manual+敏感时经 channel 阻塞等决定）；`RunTaskOptions.ToolGate runtime.ToolGate`；bubbletea 审批视图 + 决定回传通道。

**方案 Y 设计（binding）**：
- gate 实现 `runtime.ToolGate`：
  - `ShouldSuspend(...) (bool, error)` → **恒 `(false, nil)`**（Y 不走 round 级 suspend/checkpoint）。
  - `Resolve(ctx, task, call, tools) (bool, error)`：非 Manual 或非敏感 → `(true, nil)`；Manual+敏感 → 构造待决 `{tool, args}` **经 `pendingCh` 发给 bubbletea 主循环**，然后 `select { case decision := <-decisionCh: return decision.allow, nil; case <-ctx.Done(): return false, ctx.Err() }`（阻塞等答，ctx 取消 fail-loud 返回 error 防死锁）。
- bubbletea 侧：主循环收到 pending（如 streamCh 那样的 tea.Msg 通道）→ 进入审批视图（`interactiveViewApproval` 或复用）显示工具名+参数+「批准(y)/拒绝(n)」→ 用户按键 → 经 `decisionCh` 回传 `{allow bool}` → gate 解阻塞。
- gate 与 bubbletea 的通道在 `runTUITask` 构造时创建并双向接线；gate 传入 RunTaskOptions.ToolGate。

- [ ] **Step 1: 写 gate 单元测试（不依赖 bubbletea）**

`internal/tui/approvalgate_test.go`：

```go
func TestApprovalGateShouldSuspendAlwaysFalse(t *testing.T) {
	// 任意 task/calls → ShouldSuspend 返回 (false, nil)（Y 不 suspend）
}
func TestApprovalGateResolveAllowsNonManual(t *testing.T) {
	// task.Mode=auto → Resolve 敏感工具也直接 (true, nil)（不阻塞不 prompt）
}
func TestApprovalGateResolveBlocksAndReturnsDecision(t *testing.T) {
	// task.Mode=manual + 敏感工具：goroutine 里调 Resolve（会阻塞）；
	// 主 goroutine 从 pendingCh 收到待决 → 往 decisionCh 发 {allow:true} →
	// 断言 Resolve 返回 (true, nil)。再测 {allow:false}→(false,nil)。
}
func TestApprovalGateResolveCtxCancelFailsLoud(t *testing.T) {
	// manual+敏感，不回决定，cancel ctx → Resolve 返回 (false, ctx.Err())（非静默、非死锁）
}
```

> gate 设计成通道注入（`newApprovalGate(pendingCh chan<- pendingApproval, decisionCh <-chan approvalDecision)` 或类似），便于测试直驱通道，不碰 bubbletea。敏感判定复用 `tool.Registry.Descriptors().Sensitive`（仿 manualgate.resolveRealTool 但不落盘）。

- [ ] **Step 2: 跑确认失败** → **Step 3: 实现 gate**

`internal/tui/approvalgate.go`：实现上述 Y gate。ShouldSuspend 恒 false；Resolve manual+敏感经通道阻塞 select（decisionCh / ctx.Done()）。敏感工具判定：遍历 tools.Descriptors() 找 call 对应工具的 Sensitive（lazy call_tool 解包同 manualgate）。

- [ ] **Step 4: RunTaskOptions + app.RunTask 接 ToolGate**

- `RunTaskOptions` 加 `ToolGate runtime.ToolGate`。
- `app.RunTask` runtime 构造（app.go:189）传 `ToolGate: opts.ToolGate`（现为 nil）。

- [ ] **Step 5: bubbletea 审批视图 + 决定回传 + runTUITask 接线**

- interactive.go：加 `pendingApproval`/`approvalDecision` 消息类型 + 审批视图（显示工具名+参数+y/n 提示）；收到 pending → 切审批视图；y/n 按键 → 往 decisionCh 发决定 → 回原视图。
- runTUITask（command.go:446）：Manual 模式（或恒建）时创建 pendingCh/decisionCh，`newApprovalGate(...)` 接入 RunTaskOptions.ToolGate，并把 pendingCh 接到 bubbletea 消息循环（仿 streamCh 消费 :396/1467）。
- 写 bubbletea 侧测试（interactive_test.go）：注入一个会触发 Resolve 的场景（或用 fake gate 发 pendingApproval msg），模拟 Update(pendingApproval) → 进审批视图断言 View 含工具名 → Update(KeyMsg "y") → 断言 decisionCh 收到 allow。

- [ ] **Step 6: 跑确认通过 + race**

Run: `go build ./... && go test ./internal/tui/ ./internal/app/`
Expected: PASS。方案 Y 的通道并发 → 本任务后务必 WSL race（Task 6 统一跑，但本任务可先 `go test -race ./internal/tui/`（Windows 无 gcc 则留 Task 6 WSL））。

- [ ] **Step 7: gofmt + 提交**

```bash
gofmt -w internal/tui/approvalgate.go internal/tui/approvalgate_test.go internal/tui/interactive.go internal/tui/interactive_test.go internal/app/app.go internal/cli/command.go
git add internal/tui/ internal/app/app.go internal/cli/command.go
git commit -m "feat(tui): 方案 Y 同步阻塞审批 gate + 终端 prompt(Manual 敏感工具就地批准/拒绝)"
```

---

### Task 6: 全链路整合 + 门禁

**Files:**
- Modify: 视需要小修（整合）

- [ ] **Step 1: 全量门禁（Windows）**

Run: `go build ./... ; go vet ./... ; go test ./...`
Expected: 全 PASS。若 FAIL 先诊断修复。

- [ ] **Step 2: gofmt 检查触碰文件**

Run: `gofmt -l internal/tui/ internal/app/app.go internal/cli/command.go`
Expected: 空（误报既有 CRLF 除外）。

- [ ] **Step 3: WSL race 门禁**

Run: 见顶部 WSL race recipe。
Expected: 全 PASS 无 race（方案 Y gate channel/goroutine 重点）。

- [ ] **Step 4: 人工验证清单（写入报告，供用户跑 `legion tui` 逐条核对）**

TUI 交互不在自动化门禁内，列人工清单：
1. `/mode manual|plan|auto` 切换 → 状态栏反映 + per-session（切会话保持）。
2. `/cwd <dir>` → 状态栏显示 + 该 session 任务工具沙箱在其内；`/cwd 非目录` → 报错。
3. `/mode plan` → 发任务只调研不执行副作用工具。
4. `/mode manual` → 发敏感任务 → 终端弹批准/拒绝 prompt → y 执行 / n 拒绝；Ctrl-C/退出中途 → 不死锁。

- [ ] **Step 5: 提交**

```bash
git add -A && git commit -m "test(tui): M3c 整合 + 门禁 + 人工验证清单"
```

---

## Self-Review

**Spec coverage（§4.7）:**
- `/mode manual|plan|auto` / `/cwd <path>`（走同一 session mode/working_dir 存取）→ Task 1/2 ✅
- 状态栏常驻显示模式 + 工作目录 → Task 3 ✅（Plan 标签位可复用）
- working_dir 沙箱（任务工具锁其内）→ Task 4 ✅
- Plan 模式只读 → Task 4 ✅
- 审批进程内消费（终端 prompt + 就地决定）→ Task 5（方案 Y）✅
- 与 GUI 对等（模式/cwd/审批）→ 全部 ✅

**方案 Y 取舍记录:** 不落盘/不跨重启/不进 ToolGateStore，与 serve ManualToolGate 两套独立实现——本里程碑显式接受（spec §4.7「hermes CLI callback」+ TUI 前台单用户同步）。

**Placeholder scan:** Task 1/2/3/4 测试给骨架 + 明确「先读现有 fakeSessionManager/tuiSessionController/app.RunTask 对齐,勿臆造」；Task 5 gate 给完整通道设计 + 4 个具体单测。crux（Task 5 Y gate）设计具体到 select/ctx。

**Type consistency:** `SessionManager` 的 CurrentMode/SetMode/CurrentWorkingDir/SetWorkingDir 贯穿 Task 1/2/4；`RunTaskOptions.Mode/WorkingDir/ToolGate` Task 4/5 一致；gate 实现 `runtime.ToolGate`（ShouldSuspend 恒 false + Resolve 阻塞）——签名匹配 runtime.ToolGate。

**风险提示（给执行者）:** Task 5（方案 Y gate + bubbletea 通道 + ctx 取消防死锁）是最险，用 opus reviewer + WSL race 必跑。先确认 runtime 是否已按 task.Mode 应用 Plan 子集（Task 4 决定是否需补）。gate 敏感判定复用 tool.Descriptors().Sensitive + lazy call_tool 解包（仿 manualgate 但不落盘）。
