---
title: Milestone 2b 实现计划 — Manual 审批 gate（suspend/resume + 会话目录持久化 + resume 派发）
type: plan
status: active
created: 2026-07-18
scope: legion/legionAgent（后端 runtime）
related:
  - "[[2026-07-18-m2b-handoff-manual-approval-gate]]"
  - "[[2026-07-15-agent-working-modes-design]]"
  - "[[2026-07-18-agent-modes-m2a-mode-plan]]"
tags: [agent, runtime, approval, manual, toolgate, milestone-2b, plan]
---

# Milestone 2b Implementation Plan — Manual 审批 gate

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 让 `manual` 模式的会话在敏感（有副作用）工具调用前挂起、落盘审批票据，人工 approve/deny 后从检查点恢复继续跑；deny 作为契约内的拒绝结果回给模型，不杀任务。

**Architecture:** 复用 M1b 的 round 级 suspend/resume（挂起发生在 `executeToolCalls` 之前，checkpoint 是整轮 PendingCalls）。新增 `ManualToolGate` 实现 `runtime.ToolGate`：round 边界遍历本轮工具调用（含 lazy `call_tool` 解包），敏感且未决则开票据（落会话目录 JSON）并令 `checkSuspend` 存检查点 + 返 `ErrSuspended`；派发层对已决的敏感调用查决定，denied 返回拒绝结果。恢复派发用**方案 B**：`Decide` 令任务 `Suspended→Running`，`Coordinator.Heartbeat` 新增 resume 扫描（Running + 有 checkpoint + 锁可获取 → 起 goroutine 从检查点续跑）。

**Tech Stack:** Go 1.26；标准库 `encoding/json`/`os`/`path/filepath`/`sync`；既有内部包 `internal/{runtime,approval,tool,task,sessionstate,domain,server,cli}`。无新第三方依赖。

## Global Constraints

- **Fail-Loud 铁律**（`legionAgent/CLAUDE.md §0`，覆盖一切便利写法）：禁 fallback / zero-value 兜底、禁 `_ = err`、禁静默 `continue`/`return`/`default` 吞非预期状态。审批 deny = 契约内的拒绝结果回模型（**非兜底**）。盘写失败 / 损坏票据 JSON → 返回 error 或报错跳过并记录，绝不静默当无票据。约定字段缺失、解码失败、依赖不可用一律 `fmt.Errorf("<动作> <标识>: %w", err)` 包装传播。
- **契约豁免**：仅契约显式声明可选者（`Task.Mode` 空→auto、无 session 一次性任务、checkpoint/ticket 文件不存在=fresh）按可选处理；存疑按 fail-loud。
- **门禁**：`go build ./... && go vet ./... && go test ./...` 全绿、`gofmt -l` 触碰文件内容 LF 干净（仓库 `core.autocrlf=true`，`gofmt -l` 会误报既有 CRLF 文件——靠剥 `\r` 验证被改文件内容干净，别整文件重排）。
- **`-race` 只能在 WSL 跑**（Windows 无 gcc）。配方见 §「验收环境」。Windows 宿主用普通 `go test ./...`。
- **公开 API 必须有 Go doc 注释**（以标识符名开头）。
- **模块根**：所有代码在 `F:\source\stardust\Legion\legion\legionAgent`（server 仓库，remote `github.com/jxncyjq/jxncyjq-stardust-agent-server`）。改代码前 `git rev-parse --show-toplevel` 确认。中文写 commit / PR 正文。
- **分支**：从 master（HEAD `e644b20`）切 `feat/agent-modes-m2b`。

## 验收环境

WSL `-race` 全套（每个任务结束跑一次相关包）：
```bash
wsl -d Ubuntu-22.04 -- bash -lc 'cd /mnt/f/source/stardust/Legion/legion/legionAgent && GOOS=linux GOARCH=amd64 CGO_ENABLED=1 CC=gcc $HOME/sdk/go/bin/go test -race ./internal/runtime/ ./internal/task/ ./internal/approval/ ./internal/server/ ./internal/sessionstate/ ./internal/tool/'
```
Windows 门禁：`go build ./... && go vet ./... && go test ./...`。

---

## 已核对的真实签名（写代码前的地基，行号为约数）

> 实现者：以下签名已在 M2a HEAD 逐一读源核对。脚本行号是约数，改前用 Grep/Read 复核结构。

- `domain.Task{ID,CompanyID,AgentID,SessionID,Mode,Status,Input,MaxIterations,CreatedAt,Images}`（`internal/domain/types.go:62`）；`domain.ModeManual/ModePlan/ModeAuto`（裸 string `"manual"/"plan"/"auto"`，:30）；`domain.NormalizeMode(raw)(mode,ok)`（:38）；`domain.TaskSuspended/TaskRunning/TaskPending/...`（:17）。
- `domain.ToolCall{ID,Name,Arguments map[string]string,RiskLevel}`；`domain.ToolResult{CallID,Success,Output,Error}`。
- `sessionstate.Checkpoint`（`internal/sessionstate/checkpoint.go:34`）**无 Mode 字段**；`CheckpointSchemaVersion=1`（:17）；`Store.Save/Load/Delete/WritePlan/ListSuspended`；`Store.Load` 遇 schema 不符 fail-loud（:105）。
- `sessionstate.SessionDir(base,key)=<base>/session/<key>`（`resolver.go:76`）；`ResolveWorkspaceRoot(configured)(root,warning)`（:24）。
- `runtime.ToolGate interface{ ShouldSuspend(ctx, task, calls)(bool,error) }`（`internal/runtime/runtime.go:40`）——**本计划要扩展它**。
- `runtime.Config.Checkpoints *sessionstate.Store` + `Config.ToolGate ToolGate`（:76-78）；`Runtime.checkpoints/toolGate` 字段（:108）。
- `runtime.checkSuspend(ctx,task,st)(bool,error)`（:351）：`toolGate==nil||checkpoints==nil`→false；否则 `ShouldSuspend(ctx,task,st.resp.ToolCalls)`，true→Save→true。
- `runtime.RunTask`（:210）resume 分支（:233 `if r.checkpoints != nil` → Load → 重建 loopState → runToolLoop）；`loopState`（:115）有 `tools *tool.Registry`（M2a 已线程）；`effectiveTools(task)`（:141）。
- `runtime.dispatchToolCall(ctx,agent,task,call,tools)`（`lazytools.go:160`，唯一 choke）；`dispatchCallTool`（:181，lazy 真实工具在 `call.Arguments["tool_name"]` / `arguments_json`）；`metaToolCallTool="call_tool"`、`metaToolListTools="list_tools"`（:20-23）；`isMetaTool`（:65）。
- `runtime.sessionKeyForTask(task)`（`checkpoint.go:11`）= SessionID else ID。
- `tool.Registry.Descriptors()[]Descriptor`（`registry.go:99`）；`Descriptor.Sensitive bool`（`descriptor.go:17`）、`.Name`、`.RiskLevel`；`Registry.SafeToolNames()`（:109）；`Registry.Subset`（:82）；`Registry.Execute`（:134，`ErrToolNotFound` :16）。敏感工具已分类：`write_file`/`fetch_url`/`send_message`/`create_task`/`claim_task`/`update_task`/`append_task_message`/`rebuild_task`/`delegate_task`/`moa_consult`（`Sensitive:true`）；安全：`read_file`/`search_content`/`list_files`/`read_task`/`read_messages`/`session_search`。
- `approval.Service`（`service.go:64`，纯内存，被 workflow engine + coordinator 硬循环票据用）；`Ticket{ID,Type,SubjectID,Status,Decision,DeciderID,Reason,Comment,CreatedAt,UpdatedAt}`。**本计划不改 Service**，新增独立的 `ToolGateStore`（见任务 2 决策）。
- `task.Scheduler`（`scheduler.go:18`）：`Add/Next/List/Get/Transition`；`Next`（:42）**只挑 TaskPending**；`canTransition`（:108）`Suspended↔Running` 已允许、`Running→Suspended` 已允许。
- `task.LockStore`（`lock.go:9`）：`TryLock(ctx,taskID,ownerID,ttl)(bool,error)`（:24）、`ReapExpired`（:42）——**无 Unlock**（任务 4 要加）。
- `runtime.Coordinator`（`coordinator.go:51`）：`Heartbeat`（:96，弹 Pending 并发跑）、`runAssigned`（:138，`TryLock`:139、Running 转移:149、`ErrSuspended`→Suspended:199-211）、`RecoverSuspended(ctx,store)`（:274，重扫 checkpoint 重注册为 Suspended、**丢 Mode**）、`Wait`（:130）；`CoordinatorConfig`（:34）；字段 `scheduler/locks/runtime/approvals/audit/events/lockTTL/sem/wg`。
- serve 装配（`cli/command.go`）：`workspaceRoot,warning := ResolveWorkspaceRoot(...)`（:1796）、`checkpointStore := NewStore(workspaceRoot)`（:1797）、`defaultTools`（:1755）、`defaultRuntime := NewRuntime(Config{...Checkpoints:checkpointStore})`（:1802，**无 ToolGate**）、`coordinator := NewCoordinator(...)`（:1813）、`resolver := NewAgentRuntimeResolver(...)` + `// TODO(M2)`（:1731-1740，**resolver runtimes 没接 Checkpoints/ToolGate**）、`RecoverSuspended`（:1884）、`httpServer := NewHTTPServer(server.Config{...})`（:1906）、`logger`（:1877）。
- `server.Config`（`http.go:75`）/`HTTPServer`（:98）/`NewHTTPServer`（:123）/`ServeHTTP` 路由 switch（:186，`/v1/tasks` @:221-228）；`TaskStore interface{Add,Get,List}`（:26）；`handleCreateTask`（`taskMode := domain.ModeAuto` @:672）。
- `agent_resolver.go`：`AgentRuntimeResolverConfig`（:26）、`ResolveTaskRunner`（:58）→ `NewRuntime(Config{...})`（:94，**无 Checkpoints/ToolGate**）。

---

## 文件结构（本计划创建 / 修改）

**创建：**
- `internal/approval/toolgate_store.go` — 会话目录 JSON 持久化的工具审批票据 store（`ToolApproval` 记录 + `ToolGateStore`）。**独立于既有 `Service`**（见任务 2 决策）。
- `internal/approval/toolgate_store_test.go`
- `internal/manualgate/manualgate.go` — `ManualToolGate` 实现 `runtime.ToolGate`（round 级 ShouldSuspend + 派发级 Resolve + lazy peek + 敏感度判定）。放独立包避免与 runtime 循环依赖。
- `internal/manualgate/manualgate_test.go`
- `internal/manualgate/decider.go` — `ApprovalCoordinator`：包装 `ToolGateStore` + `task.Scheduler`，`Decide` 落盘决定并在全部票据已决时令任务 `Suspended→Running`（供 HTTP + 超时清扫调用）。
- `internal/manualgate/decider_test.go`

**修改：**
- `internal/sessionstate/checkpoint.go` — `Checkpoint` 加 `Mode string`；`CheckpointSchemaVersion`→2。
- `internal/runtime/runtime.go` — `ToolGate` 接口扩签名（加 `tools` 参数 + `Resolve` 方法）；`checkSuspend` 传 `st.tools` 并把 `task.Mode` 存进 checkpoint；`RunTask` resume 分支把 `cp.Mode` 放回重建 task。
- `internal/runtime/lazytools.go` — `dispatchToolCall` 接 gate 派发级决定检查。
- `internal/runtime/coordinator.go` — `CoordinatorConfig`/`Coordinator` 加 `Checkpoints`；`Heartbeat` 加 resume 扫描；`runAssigned` 提取 `afterRun` 尾段 + 新增 `runResume`；suspend 路径释放锁；`RecoverSuspended` 重建带 `cp.Mode`。
- `internal/task/lock.go` — 加 `Unlock(ctx,taskID,ownerID)(bool,error)`。
- `internal/task/scheduler.go` — （若需）无改动：`Suspended→Running`/`Running→Suspended` 已允许，先确认。
- `internal/server/http.go` — `Config`/`HTTPServer` 加 `ToolApprovals ApprovalDecider`；路由加 `POST /v1/tasks/{id}/approvals/{ticketID}`；`handleDecideApproval`。
- `internal/runtime/agent_resolver.go` — `AgentRuntimeResolverConfig` 加 `Checkpoints`+`ToolGate`；`ResolveTaskRunner` 的 `NewRuntime` 接上。
- `internal/cli/command.go` — serve 装配：构造 `ToolGateStore`+`ManualToolGate`+`ApprovalCoordinator`；注入 default runtime + resolver + coordinator + httpServer；加超时清扫 background job；启动 reconcile 已决未跑完票据。

---

## Task 1: 🔴 安全前置 — checkpoint 携带 Mode（升 schema v2）

**为什么最先**：Manual gate 一旦挂起，resume 必须仍认得 `task.Mode==manual`，否则恢复后 gate 不触发、敏感调用被直接执行——安全洞。`RecoverSuspended` 重建任务同样必须带回 Mode。此任务先落地，后续 gate 任务才安全。

**Files:**
- Modify: `internal/sessionstate/checkpoint.go:17,34-49`（加 `Mode` 字段、升版本）
- Modify: `internal/runtime/runtime.go:362-377`（checkSuspend 写 Mode）、`:238-253`（resume 重建 task 带 Mode）
- Modify: `internal/runtime/coordinator.go:289-294`（RecoverSuspended 重建带 Mode）
- Test: `internal/sessionstate/checkpoint_test.go`、`internal/runtime/checkpoint_helpers_test.go`（或新增 `runtime/resume_mode_test.go`）、`internal/runtime/coordinator_test.go`

**Interfaces:**
- Produces: `sessionstate.Checkpoint.Mode string`（json `"mode,omitempty"`）；`sessionstate.CheckpointSchemaVersion == 2`。
- Consumes: `sessionKeyForTask`、`r.effectiveTools`（已存在）。

- [ ] **Step 1: 写失败测试 — checkpoint 往返保留 Mode**

`internal/sessionstate/checkpoint_test.go` 追加：

```go
func TestCheckpointRoundTripPreservesMode(t *testing.T) {
	dir := t.TempDir()
	store := NewStore(dir)
	cp := Checkpoint{
		SchemaVersion: CheckpointSchemaVersion,
		TaskID:        "t1",
		SessionKey:    "s1",
		Mode:          "manual",
		BasePrompt:    "p",
		Round:         1,
	}
	if err := store.Save(cp); err != nil {
		t.Fatalf("Save: %v", err)
	}
	got, ok, err := store.Load("s1")
	if err != nil || !ok {
		t.Fatalf("Load: ok=%v err=%v", ok, err)
	}
	if got.Mode != "manual" {
		t.Fatalf("Mode = %q, want manual", got.Mode)
	}
}

func TestLoadRejectsV1Checkpoint(t *testing.T) {
	dir := t.TempDir()
	sessDir := SessionDir(dir, "s1")
	if err := os.MkdirAll(sessDir, 0o755); err != nil {
		t.Fatal(err)
	}
	// A v1 checkpoint on disk must fail loud, not half-decode into a modeless task.
	if err := os.WriteFile(filepath.Join(sessDir, "task-state.json"),
		[]byte(`{"schema_version":1,"task_id":"t1","session_key":"s1"}`), 0o644); err != nil {
		t.Fatal(err)
	}
	if _, _, err := NewStore(dir).Load("s1"); err == nil {
		t.Fatal("Load of v1 checkpoint: want fail-loud error, got nil")
	}
}
```

（`os`/`path/filepath` 已在测试或需补 import。）

- [ ] **Step 2: 跑测试确认失败**

Run: `go test ./internal/sessionstate/ -run 'Mode|V1Checkpoint' -v`
Expected: FAIL — `Mode` 字段不存在（编译错）/ v1 未被拒。

- [ ] **Step 3: 加 Mode 字段 + 升版本**

`internal/sessionstate/checkpoint.go`：`const CheckpointSchemaVersion = 2`（原 1）。`Checkpoint` struct 在 `SessionKey` 后插：

```go
	// Mode is the task's working mode (manual|plan|auto) captured at suspend time,
	// so a resumed run re-applies the same gating (e.g. Manual still gates sensitive
	// tools) instead of losing it and executing side effects unguarded.
	Mode string `json:"mode,omitempty"`
```

（`Load` 的 schema 校验已存在于 :105，自动拒 v1。）

- [ ] **Step 4: 跑测试确认通过**

Run: `go test ./internal/sessionstate/ -run 'Mode|V1Checkpoint' -v`
Expected: PASS

- [ ] **Step 5: 写失败测试 — checkSuspend 落 Mode、resume 重建带 Mode**

`internal/runtime/` 新建 `resume_mode_test.go`。用一个总是要求挂起的假 gate 驱动：

```go
package runtime

import (
	"context"
	"testing"

	"github.com/stardust/legion-agent/internal/adapter"
	"github.com/stardust/legion-agent/internal/domain"
	"github.com/stardust/legion-agent/internal/sessionstate"
	"github.com/stardust/legion-agent/internal/tool"
)

// alwaysSuspendGate suspends on the first round and, once a checkpoint exists,
// resolves every call as allowed — the minimal gate to exercise mode plumbing.
type alwaysSuspendGate struct{ suspended bool }

func (g *alwaysSuspendGate) ShouldSuspend(_ context.Context, _ domain.Task, calls []domain.ToolCall, _ *tool.Registry) (bool, error) {
	if g.suspended {
		return false, nil
	}
	g.suspended = true
	return len(calls) > 0, nil
}
func (g *alwaysSuspendGate) Resolve(context.Context, domain.Task, domain.ToolCall, *tool.Registry) (bool, error) {
	return true, nil
}

func TestCheckSuspendPersistsMode(t *testing.T) {
	dir := t.TempDir()
	store := sessionstate.NewStore(dir)
	reg := planRegistry(t) // reuses helper from plan_mode_test.go (read_x safe, write_x sensitive)
	maas := &oneToolThenTextMaas{toolName: "read_x"} // see Step 7 helper
	runner := NewRuntime(Config{
		Maas: maas, Audit: adapter.NewMemoryAuditLog(), Events: adapter.NewMemoryEventBus(),
		Tools: reg, Checkpoints: store, ToolGate: &alwaysSuspendGate{},
	})
	task := domain.Task{ID: "t1", SessionID: "s1", AgentID: "a1", Status: domain.TaskRunning, Mode: domain.ModeManual, Input: "go"}
	_, err := runner.RunTask(context.Background(), domain.Agent{ID: "a1"}, task)
	if err != ErrSuspended {
		t.Fatalf("RunTask err = %v, want ErrSuspended", err)
	}
	cp, ok, err := store.Load("s1")
	if err != nil || !ok {
		t.Fatalf("Load: ok=%v err=%v", ok, err)
	}
	if cp.Mode != domain.ModeManual {
		t.Fatalf("checkpoint Mode = %q, want manual", cp.Mode)
	}
}
```

- [ ] **Step 6: 跑测试确认失败**

Run: `go test ./internal/runtime/ -run TestCheckSuspendPersistsMode -v`
Expected: FAIL — checkpoint `Mode` 为空（checkSuspend 未写）。

- [ ] **Step 7: checkSuspend 写 Mode + resume 重建带 Mode + 补测试辅助 maas**

`internal/runtime/runtime.go` `checkSuspend` 的 `cp := sessionstate.Checkpoint{...}`（:362）加一行 `Mode: task.Mode,`（放在 `SessionKey` 后）。

resume 分支（:238-253）当前用传入的 `task` 跑，但 `RunTask` 收到的 `task` 在 resume 场景由 coordinator 重建——为确保即使调用方漏传 Mode 也以 checkpoint 为准，在重建 loopState 前把 `task.Mode = cp.Mode`：

```go
		if ok {
			// The checkpoint is authoritative for the resumed run's mode: a caller
			// (coordinator resume) may hand us a task rebuilt from the scheduler; the
			// mode captured at suspend time must win so gating stays consistent.
			task.Mode = cp.Mode
			st := loopState{
				started:          started,
				basePrompt:       cp.BasePrompt,
				...
				tools:            r.effectiveTools(task),
			}
			return r.runToolLoop(ctx, requestID, agent, task, st)
		}
```

（注意：`task` 是值参数，就地改安全。`effectiveTools(task)` 现读到正确 Mode。）

新建测试辅助 `internal/runtime/resume_mode_test.go` 顶部补一个受控 maas（round1 发一个工具调用、之后回文本）：

```go
type oneToolThenTextMaas struct {
	toolName string
	calls    int
}

func (m *oneToolThenTextMaas) Generate(ctx context.Context, req port.InferenceRequest) (port.InferenceResponse, error) {
	if err := ctx.Err(); err != nil {
		return port.InferenceResponse{}, err
	}
	m.calls++
	if m.calls == 1 {
		return port.InferenceResponse{ToolCalls: []domain.ToolCall{{ID: "c1", Name: m.toolName, Arguments: map[string]string{}}}}, nil
	}
	return port.InferenceResponse{Text: "done"}, nil
}
```

（import `port`。）

> ⚠️ 本步引入 `ToolGate` 新签名（`ShouldSuspend(...,tools)` + `Resolve`）。若此时接口尚未扩展，测试编译不过——**任务 3 才正式扩接口**。为保任务 1 独立可测：本任务先只把接口扩成新签名（纯签名，无实现体），即在 `runtime.go:40` 就地改成任务 3 的最终签名（见任务 3 Step 3 的接口定义），并更新既有 ToolGate 实现/mock。先 Grep `ShouldSuspend` 找出所有实现者一并改签名。

- [ ] **Step 8: 跑测试确认通过**

Run: `go test ./internal/runtime/ -run 'TestCheckSuspendPersistsMode|TestPlanMode|TestAutoMode' -v`
Expected: PASS（既有 plan/auto 测试不回归）。

- [ ] **Step 9: 写失败测试 — RecoverSuspended 重建带 Mode**

`internal/runtime/coordinator_test.go` 追加（复用既有测试骨架：scheduler + store + coordinator；参照文件内既有 `RecoverSuspended` 测试）：

```go
func TestRecoverSuspendedRestoresMode(t *testing.T) {
	dir := t.TempDir()
	store := sessionstate.NewStore(dir)
	if err := store.Save(sessionstate.Checkpoint{
		SchemaVersion: sessionstate.CheckpointSchemaVersion,
		TaskID:        "t1", AgentID: "a1", SessionKey: "s1", Mode: domain.ModeManual,
	}); err != nil {
		t.Fatal(err)
	}
	sched := task.NewScheduler()
	c := NewCoordinator(CoordinatorConfig{Agent: domain.Agent{ID: "a1"}, Scheduler: sched, Locks: task.NewLockStore()})
	if _, err := c.RecoverSuspended(context.Background(), store); err != nil {
		t.Fatalf("RecoverSuspended: %v", err)
	}
	got, ok, err := sched.Get(context.Background(), "t1")
	if err != nil || !ok {
		t.Fatalf("Get: ok=%v err=%v", ok, err)
	}
	if got.Mode != domain.ModeManual {
		t.Fatalf("recovered task Mode = %q, want manual", got.Mode)
	}
}
```

- [ ] **Step 10: 跑测试确认失败 → 修 RecoverSuspended → 通过**

`coordinator.go:289` 的 `c.scheduler.Add(ctx, domain.Task{...})` 加 `Mode: cp.Mode,`。
Run: `go test ./internal/runtime/ -run TestRecoverSuspendedRestoresMode -v` → PASS

- [ ] **Step 11: 全包门禁 + commit**

```bash
go build ./... && go vet ./... && go test ./internal/sessionstate/ ./internal/runtime/
git add internal/sessionstate/checkpoint.go internal/runtime/runtime.go internal/runtime/coordinator.go internal/runtime/resume_mode_test.go internal/sessionstate/checkpoint_test.go internal/runtime/coordinator_test.go
git commit -m "feat(sessionstate): checkpoint carries task Mode across suspend/resume (schema v2)"
```

---

## Task 2: approval 工具审批票据的会话目录持久化（ToolGateStore）

**设计决策（偏离交接的「改 Service」）**：既有 `approval.Service` 服务于 workflow / 硬循环票据，其 `Ticket` schema（Type/SubjectID/Reason）与 Manual gate 所需（SessionID/TaskID/ToolCallID/ToolName/Arguments）不同，且被多处内存消费。把它改成磁盘后端会牵动无关调用方并把两套 schema 挤进一个类型。**改为新增独立类型 `approval.ToolGateStore` + `ToolApproval`**，只服务 Manual gate，会话目录 JSON 落盘。DRY 不违反——两者职责不同。

**Files:**
- Create: `internal/approval/toolgate_store.go`
- Test: `internal/approval/toolgate_store_test.go`

**Interfaces:**
- Produces:
  ```go
  type ApprovalStatus string
  const ( ApprovalPending ApprovalStatus = "pending"; ApprovalApproved ApprovalStatus = "approved"; ApprovalDenied ApprovalStatus = "denied" )

  type ToolApproval struct {
      TicketID   string            `json:"ticket_id"`
      SessionKey string            `json:"session_key"`
      TaskID     string            `json:"task_id"`
      ToolCallID string            `json:"tool_call_id"`
      ToolName   string            `json:"tool_name"`
      Arguments  map[string]string `json:"arguments,omitempty"`
      Status     ApprovalStatus    `json:"status"`
      CreatedAt  time.Time         `json:"created_at"`
      UpdatedAt  time.Time         `json:"updated_at"`
  }

  type ToolGateStore struct{ base string }
  func NewToolGateStore(base string) *ToolGateStore

  // TicketID derives a deterministic id from (taskID, toolCallID) so Open is
  // idempotent per call and lookups need no index.
  func TicketID(taskID, toolCallID string) string

  func (s *ToolGateStore) Open(rec ToolApproval) (ToolApproval, error)                       // idempotent: existing pending/decided ticket for same call is returned unchanged
  func (s *ToolGateStore) Get(sessionKey, ticketID string) (ToolApproval, bool, error)
  func (s *ToolGateStore) Decide(sessionKey, ticketID string, status ApprovalStatus) (ToolApproval, error) // status must be approved|denied
  func (s *ToolGateStore) ListForTask(sessionKey, taskID string) ([]ToolApproval, error)
  func (s *ToolGateStore) ListPending() ([]ToolApproval, error)                               // across all sessions, for timeout sweep
  ```
- Consumes: `sessionstate.SessionDir(base, sessionKey)` → `<sessionDir>/approvals/<ticketID>.json`。

- [ ] **Step 1: 写失败测试 — Open 幂等 + Decide + 查询**

`internal/approval/toolgate_store_test.go`：

```go
package approval

import (
	"path/filepath"
	"testing"

	"github.com/stardust/legion-agent/internal/sessionstate"
)

func newRec(task, call, tool string) ToolApproval {
	return ToolApproval{SessionKey: "s1", TaskID: task, ToolCallID: call, ToolName: tool}
}

func TestToolGateStoreOpenIsIdempotent(t *testing.T) {
	s := NewToolGateStore(t.TempDir())
	a, err := s.Open(newRec("t1", "c1", "write_file"))
	if err != nil {
		t.Fatal(err)
	}
	if a.Status != ApprovalPending {
		t.Fatalf("status = %q, want pending", a.Status)
	}
	b, err := s.Open(newRec("t1", "c1", "write_file"))
	if err != nil {
		t.Fatal(err)
	}
	if b.TicketID != a.TicketID {
		t.Fatalf("second Open minted new ticket %q != %q", b.TicketID, a.TicketID)
	}
}

func TestToolGateStoreDecidePersists(t *testing.T) {
	dir := t.TempDir()
	s := NewToolGateStore(dir)
	a, _ := s.Open(newRec("t1", "c1", "write_file"))
	if _, err := s.Decide("s1", a.TicketID, ApprovalApproved); err != nil {
		t.Fatal(err)
	}
	// Re-read from a fresh store: disk is the source of truth.
	got, ok, err := NewToolGateStore(dir).Get("s1", a.TicketID)
	if err != nil || !ok {
		t.Fatalf("Get: ok=%v err=%v", ok, err)
	}
	if got.Status != ApprovalApproved {
		t.Fatalf("status = %q, want approved", got.Status)
	}
	// Deciding an already-decided ticket must fail loud.
	if _, err := s.Decide("s1", a.TicketID, ApprovalDenied); err == nil {
		t.Fatal("re-decide: want error, got nil")
	}
}

func TestToolGateStoreListForTaskAndPending(t *testing.T) {
	s := NewToolGateStore(t.TempDir())
	_, _ = s.Open(newRec("t1", "c1", "write_file"))
	a2, _ := s.Open(newRec("t1", "c2", "send_message"))
	_, _ = s.Open(ToolApproval{SessionKey: "s2", TaskID: "t2", ToolCallID: "c9", ToolName: "fetch_url"})
	forT1, err := s.ListForTask("s1", "t1")
	if err != nil {
		t.Fatal(err)
	}
	if len(forT1) != 2 {
		t.Fatalf("ListForTask t1 = %d, want 2", len(forT1))
	}
	if _, err := s.Decide("s1", a2.TicketID, ApprovalApproved); err != nil {
		t.Fatal(err)
	}
	pending, err := s.ListPending()
	if err != nil {
		t.Fatal(err)
	}
	// c1(s1) + c9(s2) still pending; a2 decided.
	if len(pending) != 2 {
		t.Fatalf("ListPending = %d, want 2", len(pending))
	}
}

func TestToolGateStoreCorruptJSONFailsLoud(t *testing.T) {
	dir := t.TempDir()
	s := NewToolGateStore(dir)
	a, _ := s.Open(newRec("t1", "c1", "write_file"))
	path := filepath.Join(sessionstate.SessionDir(dir, "s1"), "approvals", a.TicketID+".json")
	if err := writeFileHelper(path, "{ not json"); err != nil { // helper: os.WriteFile
		t.Fatal(err)
	}
	if _, _, err := s.Get("s1", a.TicketID); err == nil {
		t.Fatal("Get on corrupt JSON: want fail-loud error, got nil")
	}
}
```

（`writeFileHelper` = `os.WriteFile(path,[]byte(c),0o644)` 小助手，或直接内联。）

- [ ] **Step 2: 跑测试确认失败**

Run: `go test ./internal/approval/ -run ToolGateStore -v`
Expected: FAIL — 类型/方法未定义。

- [ ] **Step 3: 实现 ToolGateStore**

`internal/approval/toolgate_store.go`。要点（fail-loud）：
- `TicketID(taskID, toolCallID)`：拼 `taskID + "__" + toolCallID`，用 `strings.NewReplacer` 把 `/ \ : *` 等文件系统敏感字符换 `_`，避免路径逃逸。
- 路径 = `filepath.Join(sessionstate.SessionDir(s.base, sessionKey), "approvals", ticketID+".json")`。
- `Open`：算 ticketID；先 `Get`，存在则原样返回（幂等）；否则 `MkdirAll(approvals)` + 原子写（temp+rename，仿 `checkpoint.go:Save`）；`Status=ApprovalPending`，`CreatedAt=UpdatedAt=time.Now()`。空 `SessionKey`/`TaskID`/`ToolCallID` → `fmt.Errorf` 拒绝。
- `Get`：`os.ReadFile`；`os.ErrNotExist`→`(zero,false,nil)`；其他读错或 `json.Unmarshal` 错→`fmt.Errorf(...: %w)`（**损坏 JSON fail-loud**）。
- `Decide`：`status` 必须 approved|denied，否则 error；`Get` 取，`!ok`→error（未知票据）；`Status != ApprovalPending`→`fmt.Errorf("ticket %s already decided (%s)", ...)`；置 status+`UpdatedAt`，原子写。
- `ListForTask(sessionKey, taskID)`：读 `<sessionDir>/approvals/` 目录，`os.ErrNotExist`→空；逐 `.json` 解码（损坏→error），过滤 `TaskID==taskID`。
- `ListPending()`：`filepath.Glob(filepath.Join(base,"session","*","approvals","*.json"))`，逐个解码（损坏→error），收集 `Status==ApprovalPending`。

- [ ] **Step 4: 跑测试确认通过 + gofmt + commit**

Run: `go test ./internal/approval/ -run ToolGateStore -v` → PASS
```bash
gofmt -w internal/approval/toolgate_store.go internal/approval/toolgate_store_test.go
go build ./... && go vet ./...
git add internal/approval/toolgate_store.go internal/approval/toolgate_store_test.go
git commit -m "feat(approval): session-dir JSON store for manual tool-approval tickets"
```

---

## Task 3: ManualToolGate — round 级挂起 + 派发级决定（含 lazy peek）

**核心安全任务（用 opus reviewer）。**

**Files:**
- Create: `internal/manualgate/manualgate.go`
- Test: `internal/manualgate/manualgate_test.go`
- Modify: `internal/runtime/runtime.go:40-42`（扩 `ToolGate` 接口——任务 1 已就地扩签名，此处确认最终形态）、`internal/runtime/lazytools.go:160-176`（dispatchToolCall 接决定检查）

**Interfaces:**
- Consumes: `approval.ToolGateStore`（任务 2）；`tool.Registry.Descriptors()`（查 `Sensitive`）；lazy meta 常量语义（`call_tool`/`tool_name`/`arguments_json`）。
- Produces（`runtime.ToolGate` 最终接口）：
  ```go
  type ToolGate interface {
      // ShouldSuspend reports whether the runtime must suspend before executing
      // this round's calls (Manual mode + an undecided sensitive call). It opens a
      // persisted approval ticket for each such call as a side effect.
      ShouldSuspend(ctx context.Context, task domain.Task, calls []domain.ToolCall, tools *tool.Registry) (bool, error)
      // Resolve reports, at dispatch time for one call, whether it may execute.
      // Non-manual or non-sensitive → (true,nil). Manual+sensitive+approved →
      // (true,nil). Manual+sensitive+denied → (false,nil) (caller returns a reject
      // ToolResult to the model). Undecided sensitive call at dispatch → fail-loud
      // error (should never happen: the round-level gate already suspended).
      Resolve(ctx context.Context, task domain.Task, call domain.ToolCall, tools *tool.Registry) (allow bool, err error)
  }
  ```
- `manualgate.ManualToolGate` implements it; `manualgate.New(store *approval.ToolGateStore) *ManualToolGate`.

- [ ] **Step 1: 确认 runtime.ToolGate 已是最终签名**

任务 1 Step 7 已就地把接口扩成上方形态并更新实现者。此处 Grep 复核：`Grep "ShouldSuspend|Resolve" internal/runtime` 确认 `checkSuspend`（:355）调用点已传 `st.tools`：把 `r.toolGate.ShouldSuspend(ctx, task, st.resp.ToolCalls)` 改为 `...(ctx, task, st.resp.ToolCalls, st.tools)`。若任务 1 未改此调用点，现在改并跑 `go build ./internal/runtime/`。

- [ ] **Step 2: 写失败测试 — 敏感度判定 + lazy peek + 开票据 + 挂起**

`internal/manualgate/manualgate_test.go`：

```go
package manualgate

import (
	"context"
	"testing"

	"github.com/stardust/legion-agent/internal/approval"
	"github.com/stardust/legion-agent/internal/domain"
	"github.com/stardust/legion-agent/internal/tool"
)

func gateRegistry() *tool.Registry {
	reg := tool.NewRegistry(
		tool.NewStaticPolicy(tool.DecisionAllow),
		tool.NewRolePermissionEnforcer(map[string]bool{}),
		tool.NoopGuardrails{},
	)
	reg.RegisterDescriptor(tool.Descriptor{Name: "read_file", Sensitive: false}, tool.HandlerFunc(noHandler))
	reg.RegisterDescriptor(tool.Descriptor{Name: "write_file", Sensitive: true}, tool.HandlerFunc(noHandler))
	return reg
}
func noHandler(context.Context, domain.ToolCall) (domain.ToolResult, error) {
	return domain.ToolResult{Success: true}, nil
}

func manualTask() domain.Task {
	return domain.Task{ID: "t1", SessionID: "s1", AgentID: "a1", Mode: domain.ModeManual}
}

func TestShouldSuspendOnSensitiveDirectCall(t *testing.T) {
	store := approval.NewToolGateStore(t.TempDir())
	g := New(store)
	reg := gateRegistry()
	calls := []domain.ToolCall{{ID: "c1", Name: "write_file", Arguments: map[string]string{"path": "x"}}}
	suspend, err := g.ShouldSuspend(context.Background(), manualTask(), calls, reg)
	if err != nil {
		t.Fatal(err)
	}
	if !suspend {
		t.Fatal("want suspend for sensitive write_file in manual mode")
	}
	// A pending ticket must exist for the call.
	tid := approval.TicketID("t1", "c1")
	if a, ok, _ := store.Get("s1", tid); !ok || a.Status != approval.ApprovalPending {
		t.Fatalf("expected pending ticket, ok=%v a=%+v", ok, a)
	}
}

func TestShouldSuspendSkipsReadOnly(t *testing.T) {
	g := New(approval.NewToolGateStore(t.TempDir()))
	calls := []domain.ToolCall{{ID: "c1", Name: "read_file", Arguments: map[string]string{"path": "x"}}}
	suspend, err := g.ShouldSuspend(context.Background(), manualTask(), calls, gateRegistry())
	if err != nil {
		t.Fatal(err)
	}
	if suspend {
		t.Fatal("read-only call must not suspend")
	}
}

func TestShouldSuspendLazyCallToolPeeksRealTool(t *testing.T) {
	g := New(approval.NewToolGateStore(t.TempDir()))
	// lazy meta call wrapping the sensitive write_file
	calls := []domain.ToolCall{{ID: "c1", Name: "call_tool", Arguments: map[string]string{"tool_name": "write_file", "arguments_json": "{}"}}}
	suspend, err := g.ShouldSuspend(context.Background(), manualTask(), calls, gateRegistry())
	if err != nil {
		t.Fatal(err)
	}
	if !suspend {
		t.Fatal("lazy call_tool wrapping sensitive tool must suspend")
	}
}

func TestShouldSuspendAutoModeNeverSuspends(t *testing.T) {
	g := New(approval.NewToolGateStore(t.TempDir()))
	task := domain.Task{ID: "t1", SessionID: "s1", Mode: domain.ModeAuto}
	calls := []domain.ToolCall{{ID: "c1", Name: "write_file"}}
	suspend, err := g.ShouldSuspend(context.Background(), task, calls, gateRegistry())
	if err != nil || suspend {
		t.Fatalf("auto mode: suspend=%v err=%v, want false,nil", suspend, err)
	}
}

func TestResolveDeniedReturnsDisallow(t *testing.T) {
	store := approval.NewToolGateStore(t.TempDir())
	g := New(store)
	reg := gateRegistry()
	call := domain.ToolCall{ID: "c1", Name: "write_file"}
	// open + deny
	if _, err := g.ShouldSuspend(context.Background(), manualTask(), []domain.ToolCall{call}, reg); err != nil {
		t.Fatal(err)
	}
	if _, err := store.Decide("s1", approval.TicketID("t1", "c1"), approval.ApprovalDenied); err != nil {
		t.Fatal(err)
	}
	allow, err := g.Resolve(context.Background(), manualTask(), call, reg)
	if err != nil {
		t.Fatal(err)
	}
	if allow {
		t.Fatal("denied call must resolve to disallow")
	}
}

func TestResolveApprovedAllows(t *testing.T) {
	store := approval.NewToolGateStore(t.TempDir())
	g := New(store)
	reg := gateRegistry()
	call := domain.ToolCall{ID: "c1", Name: "write_file"}
	_, _ = g.ShouldSuspend(context.Background(), manualTask(), []domain.ToolCall{call}, reg)
	_, _ = store.Decide("s1", approval.TicketID("t1", "c1"), approval.ApprovalApproved)
	allow, err := g.Resolve(context.Background(), manualTask(), call, reg)
	if err != nil || !allow {
		t.Fatalf("approved: allow=%v err=%v, want true,nil", allow, err)
	}
}

func TestResolveReadOnlyAlwaysAllows(t *testing.T) {
	g := New(approval.NewToolGateStore(t.TempDir()))
	call := domain.ToolCall{ID: "c1", Name: "read_file"}
	allow, err := g.Resolve(context.Background(), manualTask(), call, gateRegistry())
	if err != nil || !allow {
		t.Fatalf("read-only: allow=%v err=%v, want true,nil", allow, err)
	}
}
```

- [ ] **Step 3: 跑测试确认失败**

Run: `go test ./internal/manualgate/ -v`
Expected: FAIL — 包不存在。

- [ ] **Step 4: 实现 ManualToolGate**

`internal/manualgate/manualgate.go`：

```go
// Package manualgate implements the Manual-mode approval gate: at each tool-round
// boundary it suspends a task whose model wants to run a sensitive (side-effecting)
// tool until a human approves, and at dispatch time it enforces the recorded
// decision. It satisfies runtime.ToolGate.
package manualgate

import (
	"context"
	"fmt"
	"strings"

	"github.com/stardust/legion-agent/internal/approval"
	"github.com/stardust/legion-agent/internal/domain"
	"github.com/stardust/legion-agent/internal/tool"
)

const (
	metaToolCallTool = "call_tool" // must match runtime/lazytools.go
	metaToolListTools = "list_tools"
)

type ManualToolGate struct {
	store *approval.ToolGateStore
}

func New(store *approval.ToolGateStore) *ManualToolGate {
	return &ManualToolGate{store: store}
}

// resolveRealTool unwraps a lazy call_tool meta call to the underlying real tool
// name and reports its sensitivity from the registry. Non-meta calls resolve to
// their own name. A call whose target tool is unknown to the registry reports
// (name, false, false): unknown tools cannot be sensitive here — dispatch will
// fail loud via ErrToolNotFound.
func (g *ManualToolGate) resolveRealTool(call domain.ToolCall, tools *tool.Registry) (name string, sensitive bool, ok bool) {
	name = call.Name
	if call.Name == metaToolCallTool {
		name = strings.TrimSpace(call.Arguments["tool_name"])
	}
	if name == "" || name == metaToolListTools || name == metaToolCallTool {
		return name, false, false
	}
	for _, d := range tools.Descriptors() {
		if d.Name == name {
			return name, d.Sensitive, true
		}
	}
	return name, false, false
}

func (g *ManualToolGate) ShouldSuspend(ctx context.Context, task domain.Task, calls []domain.ToolCall, tools *tool.Registry) (bool, error) {
	if task.Mode != domain.ModeManual {
		return false, nil
	}
	needApproval := false
	for _, call := range calls {
		name, sensitive, ok := g.resolveRealTool(call, tools)
		if !ok || !sensitive {
			continue
		}
		sessionKey := sessionKeyForTask(task)
		ticketID := approval.TicketID(task.ID, call.ID)
		existing, found, err := g.store.Get(sessionKey, ticketID)
		if err != nil {
			return false, fmt.Errorf("check approval for task %s call %s: %w", task.ID, call.ID, err)
		}
		if found && existing.Status != approval.ApprovalPending {
			continue // already decided — do not re-suspend on this call
		}
		if _, err := g.store.Open(approval.ToolApproval{
			SessionKey: sessionKey, TaskID: task.ID, ToolCallID: call.ID,
			ToolName: name, Arguments: call.Arguments,
		}); err != nil {
			return false, fmt.Errorf("open approval for task %s call %s: %w", task.ID, call.ID, err)
		}
		needApproval = true
	}
	return needApproval, nil
}

func (g *ManualToolGate) Resolve(ctx context.Context, task domain.Task, call domain.ToolCall, tools *tool.Registry) (bool, error) {
	if task.Mode != domain.ModeManual {
		return true, nil
	}
	_, sensitive, ok := g.resolveRealTool(call, tools)
	if !ok || !sensitive {
		return true, nil
	}
	rec, found, err := g.store.Get(sessionKeyForTask(task), approval.TicketID(task.ID, call.ID))
	if err != nil {
		return false, fmt.Errorf("resolve approval for task %s call %s: %w", task.ID, call.ID, err)
	}
	switch {
	case found && rec.Status == approval.ApprovalApproved:
		return true, nil
	case found && rec.Status == approval.ApprovalDenied:
		return false, nil
	default:
		// Undecided sensitive call reaching dispatch is a control-flow invariant
		// violation (round gate should have suspended). Fail loud, never execute.
		return false, fmt.Errorf("dispatch reached undecided sensitive call %s for task %s (found=%v)", call.ID, task.ID, found)
	}
}

// sessionKeyForTask mirrors runtime.sessionKeyForTask (SessionID else ID); kept
// local to avoid importing runtime (which defines the ToolGate interface).
func sessionKeyForTask(task domain.Task) string {
	if task.SessionID != "" {
		return task.SessionID
	}
	return task.ID
}
```

> 复核点：`tool.NewStaticPolicy`/`DecisionAllow`/`NewRolePermissionEnforcer` 签名（`registry.go:212-236`）——测试构造用。`Descriptors()` 返回无序，`resolveRealTool` 线性查找即可（工具数少）。

- [ ] **Step 5: 跑测试确认通过**

Run: `go test ./internal/manualgate/ -v`
Expected: PASS

- [ ] **Step 6: 写失败测试 — dispatchToolCall 对 denied 返回拒绝结果**

`internal/runtime/manual_dispatch_test.go`（在 runtime 包内，能触达非导出 `dispatchToolCall`）。用 manualgate 真实实现驱动：

```go
package runtime

import (
	"context"
	"strings"
	"testing"

	"github.com/stardust/legion-agent/internal/approval"
	"github.com/stardust/legion-agent/internal/domain"
	"github.com/stardust/legion-agent/internal/manualgate"
	"github.com/stardust/legion-agent/internal/tool"
)

func TestDispatchDeniedSensitiveReturnsRejectResult(t *testing.T) {
	dir := t.TempDir()
	store := approval.NewToolGateStore(dir)
	gate := manualgate.New(store)
	var writeCalled bool
	reg := tool.NewRegistry(tool.NewStaticPolicy(tool.DecisionAllow),
		tool.PermissionEnforcerFunc(func(domain.Agent, domain.ToolCall) error { return nil }),
		tool.NoopGuardrails{})
	reg.RegisterDescriptor(tool.Descriptor{Name: "write_file", Sensitive: true}, tool.HandlerFunc(func(context.Context, domain.ToolCall) (domain.ToolResult, error) {
		writeCalled = true
		return domain.ToolResult{Success: true}, nil
	}))
	r := NewRuntime(Config{Maas: &oneToolThenTextMaas{}, Tools: reg, Checkpoints: nil, ToolGate: gate})
	task := domain.Task{ID: "t1", SessionID: "s1", Mode: domain.ModeManual}
	call := domain.ToolCall{ID: "c1", Name: "write_file", Arguments: map[string]string{}}
	// open + deny the ticket first
	if _, err := gate.ShouldSuspend(context.Background(), task, []domain.ToolCall{call}, reg); err != nil {
		t.Fatal(err)
	}
	if _, err := store.Decide("s1", approval.TicketID("t1", "c1"), approval.ApprovalDenied); err != nil {
		t.Fatal(err)
	}
	res, err := r.dispatchToolCall(context.Background(), domain.Agent{ID: "a1"}, task, call, reg)
	if err != nil {
		t.Fatalf("dispatchToolCall err = %v, want nil (reject is a result, not a Go error)", err)
	}
	if res.Success || !strings.Contains(res.Error, "denied") {
		t.Fatalf("want unsuccessful denied result, got %+v", res)
	}
	if writeCalled {
		t.Fatal("denied sensitive tool must not execute")
	}
}
```

（`tool.PermissionEnforcerFunc` 见 `plan_mode_test.go` 用法，确认存在。）

- [ ] **Step 7: 跑测试确认失败 → 接 dispatchToolCall → 通过**

`internal/runtime/lazytools.go` `dispatchToolCall`（:160）开头插入 gate 决定检查：

```go
func (r *Runtime) dispatchToolCall(ctx context.Context, agent domain.Agent, task domain.Task, call domain.ToolCall, tools *tool.Registry) (domain.ToolResult, error) {
	if r.toolGate != nil {
		allow, err := r.toolGate.Resolve(ctx, task, call, tools)
		if err != nil {
			return domain.ToolResult{}, fmt.Errorf("gate resolve for task %s call %s: %w", task.ID, call.ID, err)
		}
		if !allow {
			return domain.ToolResult{CallID: call.ID, Success: false, Error: "tool call denied by human approver"}, nil
		}
	}
	if !r.lazyTools || !isMetaTool(call.Name) {
		return tools.Execute(ctx, agent, call)
	}
	...
}
```

Run: `go test ./internal/runtime/ -run TestDispatchDeniedSensitive -v` → PASS

- [ ] **Step 8: 端到端 — manual+敏感 approve/deny 全程（runtime 层）**

`internal/runtime/manual_e2e_test.go`：一个 runtime + 真 store + 真 gate + checkpoint store。
- 场景 A（deny）：round1 model 发 `write_file` → `RunTask` 返 `ErrSuspended`（checkpoint 落盘，含 Mode=manual）→ `store.Decide(deny)` → 再 `RunTask`（resume）→ 应完成、`write_file` 未执行、结果为最终文本。
- 场景 B（approve）：同上但 `Decide(approve)` → resume 后 `write_file` 执行、任务完成。

```go
func TestManualGateDenyThenResume(t *testing.T) {
	dir := t.TempDir()
	cpStore := sessionstate.NewStore(dir)
	apStore := approval.NewToolGateStore(dir)
	gate := manualgate.New(apStore)
	var writeCalled bool
	reg := tool.NewRegistry(tool.NewStaticPolicy(tool.DecisionAllow),
		tool.PermissionEnforcerFunc(func(domain.Agent, domain.ToolCall) error { return nil }), tool.NoopGuardrails{})
	reg.RegisterDescriptor(tool.Descriptor{Name: "write_file", Sensitive: true}, tool.HandlerFunc(func(context.Context, domain.ToolCall) (domain.ToolResult, error) {
		writeCalled = true
		return domain.ToolResult{Success: true, Output: "wrote"}, nil
	}))
	maas := &oneToolThenTextMaas{toolName: "write_file"}
	r := NewRuntime(Config{Maas: maas, Audit: adapter.NewMemoryAuditLog(), Events: adapter.NewMemoryEventBus(),
		Tools: reg, Checkpoints: cpStore, ToolGate: gate})
	task := domain.Task{ID: "t1", SessionID: "s1", AgentID: "a1", Status: domain.TaskRunning, Mode: domain.ModeManual, Input: "go"}
	if _, err := r.RunTask(context.Background(), domain.Agent{ID: "a1"}, task); err != ErrSuspended {
		t.Fatalf("first run err = %v, want ErrSuspended", err)
	}
	if _, err := apStore.Decide("s1", approval.TicketID("t1", "c1"), approval.ApprovalDenied); err != nil {
		t.Fatal(err)
	}
	run, err := r.RunTask(context.Background(), domain.Agent{ID: "a1"}, task)
	if err != nil {
		t.Fatalf("resume run err = %v", err)
	}
	if writeCalled {
		t.Fatal("denied write_file executed on resume")
	}
	if run.Result == "" {
		t.Fatal("resume produced no final answer")
	}
}
```

（approve 版对称，断言 `writeCalled == true`。`oneToolThenTextMaas` 的工具 call ID 需为 `c1` 才匹配 ticket——调整该辅助 maas round1 返回 `ID:"c1"`。）

- [ ] **Step 9: 跑 e2e + 全 runtime 门禁 + commit**

Run: `go test ./internal/manualgate/ ./internal/runtime/ -v`
Expected: PASS（含既有 plan/auto/checkpoint 测试不回归）
```bash
gofmt -w internal/manualgate/ internal/runtime/lazytools.go internal/runtime/manual_dispatch_test.go internal/runtime/manual_e2e_test.go
go build ./... && go vet ./...
git add internal/manualgate/ internal/runtime/lazytools.go internal/runtime/manual_dispatch_test.go internal/runtime/manual_e2e_test.go
git commit -m "feat(manualgate): Manual-mode tool approval gate (round-level suspend + dispatch-level decision, lazy-aware)"
```

---

## Task 4: resume 派发（方案 B）— Decide→Running + Heartbeat resume 扫描 + 挂起释放锁

**核心并发任务（用 opus reviewer）。** 补齐交接 §3.3 标注的真架构缺口：调度只挑 Pending，无机制重拾 Suspended→Running。

**Files:**
- Modify: `internal/task/lock.go`（加 `Unlock`）
- Test: `internal/task/lock_test.go`
- Modify: `internal/runtime/coordinator.go`（`CoordinatorConfig`/`Coordinator` 加 `Checkpoints`；`runAssigned` 提取 `afterRun` + suspend 释放锁；新增 `runResume` + `resumeScan`；`Heartbeat` 调 `resumeScan`）
- Create: `internal/manualgate/decider.go`（`ApprovalCoordinator.Decide`）
- Test: `internal/runtime/coordinator_resume_test.go`、`internal/manualgate/decider_test.go`

**Interfaces:**
- Produces:
  - `func (s *LockStore) Unlock(ctx context.Context, taskID, ownerID string) (bool, error)` — 仅当 owner 匹配才释放；返回是否释放。
  - `CoordinatorConfig.Checkpoints *sessionstate.Store`；`Coordinator.checkpoints` 字段。
  - `manualgate.ApprovalCoordinator`：
    ```go
    type SchedulerGate interface {
        Get(ctx context.Context, taskID string) (domain.Task, bool, error)
        Transition(ctx context.Context, taskID string, next domain.TaskStatus) error
    }
    func NewApprovalCoordinator(store *approval.ToolGateStore, sched SchedulerGate) *ApprovalCoordinator
    // Decide records the decision on disk and, when every ticket for the task is
    // decided, transitions the task Suspended→Running so the coordinator's resume
    // scan picks it up. Returns the decided ToolApproval.
    func (a *ApprovalCoordinator) Decide(ctx context.Context, taskID, ticketID string, status approval.ApprovalStatus) (approval.ToolApproval, error)
    ```
    （`*task.Scheduler` 满足 `SchedulerGate`。）
- Consumes: `sessionstate.Store.Load`（判 checkpoint 存在）；`task.LockStore.TryLock/Unlock`；`domain.TaskRunning/TaskSuspended`。

- [ ] **Step 1: 写失败测试 — LockStore.Unlock**

`internal/task/lock_test.go` 追加：

```go
func TestUnlockReleasesOnlyForOwner(t *testing.T) {
	s := NewLockStore()
	ctx := context.Background()
	if ok, _ := s.TryLock(ctx, "t1", "owner-a", time.Minute); !ok {
		t.Fatal("initial lock failed")
	}
	// wrong owner cannot release
	if ok, err := s.Unlock(ctx, "t1", "owner-b"); err != nil || ok {
		t.Fatalf("Unlock wrong owner: ok=%v err=%v, want false,nil", ok, err)
	}
	// right owner releases
	if ok, err := s.Unlock(ctx, "t1", "owner-a"); err != nil || !ok {
		t.Fatalf("Unlock owner: ok=%v err=%v, want true,nil", ok, err)
	}
	// now re-lockable by anyone
	if ok, _ := s.TryLock(ctx, "t1", "owner-b", time.Minute); !ok {
		t.Fatal("re-lock after unlock failed")
	}
}
```

- [ ] **Step 2: 跑确认失败 → 实现 Unlock → 通过**

`internal/task/lock.go` 加：

```go
// Unlock releases a task lock iff ownerID currently holds it, returning whether a
// release happened. A lock held by someone else (or already expired/absent) is
// left untouched and reported as (false, nil): releasing another owner's lock
// would let two workers run the same task.
func (s *LockStore) Unlock(ctx context.Context, taskID string, ownerID string) (bool, error) {
	if err := ctx.Err(); err != nil {
		return false, err
	}
	s.mu.Lock()
	defer s.mu.Unlock()
	lock, ok := s.locks[taskID]
	if !ok || lock.OwnerID != ownerID {
		return false, nil
	}
	delete(s.locks, taskID)
	return true, nil
}
```

Run: `go test ./internal/task/ -run TestUnlock -v` → PASS

- [ ] **Step 3: runAssigned 提取 afterRun + suspend 释放锁**

`coordinator.go`：
1. `CoordinatorConfig` 加 `Checkpoints *sessionstate.Store`；`Coordinator` 加字段 `checkpoints *sessionstate.Store`；`NewCoordinator` 赋值。
2. `runAssigned` 的 suspend 分支（:199-211）在 `Transition(...TaskSuspended)` + audit 后、`return` 前，释放锁：
   ```go
   if _, err := c.locks.Unlock(ctx, taskToRun.ID, c.agent.ID); err != nil {
       return domain.Task{}, false, fmt.Errorf("release lock on suspend for task %s: %w", taskToRun.ID, err)
   }
   ```
   > 注意 lock owner 是 `c.agent.ID`（`TryLock` :139 用它）。
3. 提取 `runAssigned` 中 RunTask 之后的尾段（eval → hardloop → quality review → done，:215-266）为：
   ```go
   func (c *Coordinator) afterRun(ctx context.Context, taskToRun domain.Task, runnerAgent domain.Agent, run domain.TaskRun) (domain.Task, bool, error) { ... }
   ```
   `runAssigned` 成功路径改为 `return c.afterRun(ctx, taskToRun, runnerAgent, run)`。（纯重构，行为不变；既有 coordinator 测试守护。）

- [ ] **Step 4: 写失败测试 — resumeScan 续跑 Running+checkpoint 任务**

`internal/runtime/coordinator_resume_test.go`。构造：scheduler 有一个 `TaskRunning` 任务 t1（SessionID s1）+ 磁盘上有其 checkpoint；一个受控 runner 在 RunTask 时返回成功 run。断言 `Heartbeat` 后 t1 走到终态（Done 或 QualityReview→Done，取决于 reviewer/evaluator mock）且 checkpoint 被删。

```go
func TestHeartbeatResumesRunningCheckpointedTask(t *testing.T) {
	dir := t.TempDir()
	store := sessionstate.NewStore(dir)
	// seed a checkpoint (schema v2, with Mode) for t1/s1
	if err := store.Save(sessionstate.Checkpoint{
		SchemaVersion: sessionstate.CheckpointSchemaVersion,
		TaskID: "t1", AgentID: "a1", SessionKey: "s1", Mode: domain.ModeManual,
		PendingCalls: []domain.ToolCall{{ID: "c1", Name: "read_x"}},
	}); err != nil {
		t.Fatal(err)
	}
	sched := task.NewScheduler()
	// task already Running (as if Decide flipped it)
	if err := sched.Add(context.Background(), domain.Task{ID: "t1", AgentID: "a1", SessionID: "s1", Mode: domain.ModeManual, Status: domain.TaskRunning}); err != nil {
		t.Fatal(err)
	}
	runner := &recordingRunner{result: domain.TaskRun{Result: "ok"}}
	c := NewCoordinator(CoordinatorConfig{
		Agent: domain.Agent{ID: "a1"}, Scheduler: sched, Locks: task.NewLockStore(),
		Runtime: runner, Reviewer: approvingReviewer{}, Evaluator: passEvaluator{},
		Checkpoints: store,
	})
	if _, _, err := c.Heartbeat(context.Background()); err != nil {
		t.Fatal(err)
	}
	c.Wait()
	got, _, _ := sched.Get(context.Background(), "t1")
	if got.Status != domain.TaskDone {
		t.Fatalf("t1 status = %s, want done", got.Status)
	}
	if !runner.ran {
		t.Fatal("resume did not invoke RunTask")
	}
}
```

> 复核点：`recordingRunner`/`approvingReviewer`/`passEvaluator` — 复用 `coordinator_test.go` 里既有的 mock（Grep 确认命名；若不同名，沿用文件内既有 helper）。`Reviewer`/`Evaluator` 必须让任务能走到 Done。

- [ ] **Step 5: 跑确认失败 → 实现 resumeScan + runResume + Heartbeat 接线 → 通过**

`coordinator.go`：

```go
// resumeScan dispatches suspended tasks whose human decision已到（they were flipped
// to Running by ApprovalCoordinator.Decide) and that carry a persisted checkpoint.
// It claims each via TryLock so only one worker resumes a task; an actively-running
// fresh task holds its lock and a Running task without a checkpoint (mid fresh run)
// is skipped. Called from Heartbeat each tick.
func (c *Coordinator) resumeScan(ctx context.Context) error {
	if c.checkpoints == nil {
		return nil
	}
	tasks, err := c.scheduler.List(ctx)
	if err != nil {
		return fmt.Errorf("list tasks for resume scan: %w", err)
	}
	for _, t := range tasks {
		if t.Status != domain.TaskRunning {
			continue
		}
		_, hasCP, err := c.checkpoints.Load(sessionKeyForCoordinatorTask(t))
		if err != nil {
			return fmt.Errorf("load checkpoint for resume of task %s: %w", t.ID, err)
		}
		if !hasCP {
			continue // fresh Running task mid-flight, not a resume candidate
		}
		select {
		case c.sem <- struct{}{}:
		default:
			return nil // no worker slots; try next tick
		}
		locked, err := c.locks.TryLock(ctx, t.ID, c.agent.ID, c.lockTTL)
		if err != nil {
			<-c.sem
			return fmt.Errorf("lock task %s for resume: %w", t.ID, err)
		}
		if !locked {
			<-c.sem
			continue // an active worker holds it
		}
		c.wg.Add(1)
		go func(rt domain.Task) {
			defer c.wg.Done()
			defer func() { <-c.sem }()
			if _, _, err := c.runResume(ctx, rt); err != nil {
				_ = c.publishLearning(ctx, c.agent.ID, rt.ID, evolution.SignalFailure, "task_resume_error", true)
			}
		}(t)
	}
	return nil
}

// runResume runs a task that is already Running and lock-held (claimed by
// resumeScan) from its checkpoint. Unlike runAssigned it skips the Pending→Running
// transition (the task is already Running) and re-enters the runner, which
// auto-resumes from the checkpoint. On ErrSuspended it re-suspends (another
// undecided call) and releases the lock; otherwise it finalises via afterRun.
func (c *Coordinator) runResume(ctx context.Context, t domain.Task) (domain.Task, bool, error) {
	runnerAgent := c.agent
	runner := c.runtime
	if c.taskRunnerResolver != nil {
		ra, rr, resolved, err := c.taskRunnerResolver.ResolveTaskRunner(ctx, t)
		if err != nil {
			_ = c.scheduler.Transition(ctx, t.ID, domain.TaskFailed)
			return domain.Task{}, false, fmt.Errorf("resolve runner for resume of task %s: %w", t.ID, err)
		}
		if resolved {
			if rr == nil {
				_ = c.scheduler.Transition(ctx, t.ID, domain.TaskFailed)
				return domain.Task{}, false, fmt.Errorf("resolve runner for resume of task %s: runner is nil", t.ID)
			}
			runnerAgent, runner = ra, rr
		}
	}
	if runner == nil {
		_ = c.scheduler.Transition(ctx, t.ID, domain.TaskFailed)
		return domain.Task{}, false, fmt.Errorf("resume task %s: runtime is nil", t.ID)
	}
	run, err := runner.RunTask(ctx, runnerAgent, t)
	if err != nil {
		if errors.Is(err, ErrSuspended) {
			if txErr := c.scheduler.Transition(ctx, t.ID, domain.TaskSuspended); txErr != nil {
				return domain.Task{}, false, fmt.Errorf("re-suspend task %s: %w", t.ID, txErr)
			}
			if _, err := c.locks.Unlock(ctx, t.ID, c.agent.ID); err != nil {
				return domain.Task{}, false, fmt.Errorf("release lock on re-suspend for task %s: %w", t.ID, err)
			}
			return c.currentTask(ctx, t.ID)
		}
		_ = c.scheduler.Transition(ctx, t.ID, domain.TaskFailed)
		return domain.Task{}, false, fmt.Errorf("resume run task %s: %w", t.ID, err)
	}
	return c.afterRun(ctx, t, runnerAgent, run)
}
```

`Heartbeat`（:96）在弹 Pending 的 `for` 之前或之后调用 `resumeScan`；建议**之前**（先续跑已批准的挂起任务）：在函数体首行加
```go
	if err := c.resumeScan(ctx); err != nil {
		return domain.Task{}, false, fmt.Errorf("resume scan: %w", err)
	}
```

新增本地 `sessionKeyForCoordinatorTask(t domain.Task) string`（SessionID else ID）——或复用已存在的 helper（Grep coordinator.go/runtime 包内是否已导出可用；`runtime.sessionKeyForTask` 非导出但同包，coordinator.go 在 `runtime` 包内，**可直接调 `sessionKeyForTask`**）。→ 直接用 `sessionKeyForTask(t)`，删掉新 helper。

Run: `go test ./internal/runtime/ -run TestHeartbeatResumes -v` → PASS

- [ ] **Step 6: 写失败测试 — ApprovalCoordinator.Decide 全决→翻 Running**

`internal/manualgate/decider_test.go`：

```go
func TestDecideFlipsTaskToRunningWhenAllDecided(t *testing.T) {
	dir := t.TempDir()
	store := approval.NewToolGateStore(dir)
	sched := task.NewScheduler()
	_ = sched.Add(context.Background(), domain.Task{ID: "t1", SessionID: "s1", Status: domain.TaskRunning})
	_ = sched.Transition(context.Background(), "t1", domain.TaskSuspended)
	// two pending tickets
	_, _ = store.Open(approval.ToolApproval{SessionKey: "s1", TaskID: "t1", ToolCallID: "c1", ToolName: "write_file"})
	_, _ = store.Open(approval.ToolApproval{SessionKey: "s1", TaskID: "t1", ToolCallID: "c2", ToolName: "send_message"})
	ac := NewApprovalCoordinator(store, sched)
	// decide first → still one pending → stays Suspended
	if _, err := ac.Decide(context.Background(), "t1", approval.TicketID("t1", "c1"), approval.ApprovalApproved); err != nil {
		t.Fatal(err)
	}
	if got, _, _ := sched.Get(context.Background(), "t1"); got.Status != domain.TaskSuspended {
		t.Fatalf("after 1/2 decided: status=%s, want suspended", got.Status)
	}
	// decide second → all decided → flips to Running
	if _, err := ac.Decide(context.Background(), "t1", approval.TicketID("t1", "c2"), approval.ApprovalDenied); err != nil {
		t.Fatal(err)
	}
	if got, _, _ := sched.Get(context.Background(), "t1"); got.Status != domain.TaskRunning {
		t.Fatalf("after all decided: status=%s, want running", got.Status)
	}
}
```

- [ ] **Step 7: 跑确认失败 → 实现 decider.go → 通过**

`internal/manualgate/decider.go`：

```go
package manualgate

import (
	"context"
	"fmt"

	"github.com/stardust/legion-agent/internal/approval"
	"github.com/stardust/legion-agent/internal/domain"
)

type SchedulerGate interface {
	Get(ctx context.Context, taskID string) (domain.Task, bool, error)
	Transition(ctx context.Context, taskID string, next domain.TaskStatus) error
}

type ApprovalCoordinator struct {
	store *approval.ToolGateStore
	sched SchedulerGate
}

func NewApprovalCoordinator(store *approval.ToolGateStore, sched SchedulerGate) *ApprovalCoordinator {
	return &ApprovalCoordinator{store: store, sched: sched}
}

func (a *ApprovalCoordinator) Decide(ctx context.Context, taskID, ticketID string, status approval.ApprovalStatus) (approval.ToolApproval, error) {
	t, ok, err := a.sched.Get(ctx, taskID)
	if err != nil {
		return approval.ToolApproval{}, fmt.Errorf("lookup task %s for decision: %w", taskID, err)
	}
	if !ok {
		return approval.ToolApproval{}, fmt.Errorf("decide approval: task %s not found", taskID)
	}
	sessionKey := sessionKeyForTask(t)
	rec, err := a.store.Decide(sessionKey, ticketID, status)
	if err != nil {
		return approval.ToolApproval{}, fmt.Errorf("record decision for ticket %s: %w", ticketID, err)
	}
	remaining, err := a.store.ListForTask(sessionKey, taskID)
	if err != nil {
		return approval.ToolApproval{}, fmt.Errorf("list tickets for task %s: %w", taskID, err)
	}
	allDecided := true
	for _, r := range remaining {
		if r.Status == approval.ApprovalPending {
			allDecided = false
			break
		}
	}
	if allDecided && t.Status == domain.TaskSuspended {
		if err := a.sched.Transition(ctx, taskID, domain.TaskRunning); err != nil {
			return approval.ToolApproval{}, fmt.Errorf("resume task %s after decision: %w", taskID, err)
		}
	}
	return rec, nil
}
```

Run: `go test ./internal/manualgate/ -run TestDecide -v` → PASS

- [ ] **Step 8: WSL `-race` 并发无双重派发**

写 `internal/runtime/coordinator_resume_test.go` 追加一个并发压测：N 个 Running+checkpoint 任务，多次并发 `Heartbeat`，断言每个任务只被 RunTask 一次（recordingRunner 用 `sync/atomic` 计数 per-taskID），无 race。

Run（WSL）:
```bash
wsl -d Ubuntu-22.04 -- bash -lc 'cd /mnt/f/source/stardust/Legion/legion/legionAgent && CGO_ENABLED=1 CC=gcc $HOME/sdk/go/bin/go test -race ./internal/runtime/ ./internal/task/ ./internal/manualgate/ ./internal/approval/'
```
Expected: PASS，无 `DATA RACE`。

- [ ] **Step 9: 门禁 + commit**

```bash
go build ./... && go vet ./... && go test ./internal/task/ ./internal/runtime/ ./internal/manualgate/
gofmt -w internal/task/lock.go internal/runtime/coordinator.go internal/manualgate/decider.go
git add internal/task/lock.go internal/task/lock_test.go internal/runtime/coordinator.go internal/runtime/coordinator_resume_test.go internal/manualgate/decider.go internal/manualgate/decider_test.go
git commit -m "feat(runtime): resume dispatch (plan B) — decide flips Suspended→Running, Heartbeat resume scan, lock release on suspend"
```

---

## Task 5: 审批 HTTP 端点

**Files:**
- Modify: `internal/server/http.go`（`Config`/`HTTPServer` 加 `ToolApprovals`；路由 + `handleDecideApproval`）
- Test: `internal/server/http_test.go`

**Interfaces:**
- Produces（server 包内接口，由 `manualgate.ApprovalCoordinator` 满足）：
  ```go
  type ApprovalDecider interface {
      Decide(ctx context.Context, taskID, ticketID string, status approval.ApprovalStatus) (approval.ToolApproval, error)
  }
  ```
  路由：`POST /v1/tasks/{taskID}/approvals/{ticketID}`，body `{"decision":"approve"|"deny"}`。
- Consumes: `approval.ApprovalStatus`（server 需 import approval，或定义映射避免耦合——直接 import approval，同仓内部包）。

> 复核点：server 已 import 哪些包、`writeError`/`writeJSON`/`json.NewDecoder` 用法（`http.go` 内既有）。路由匹配现用 `strings.HasPrefix`+`HasSuffix` 手工解析 path，参照 `handleAgentMessages`（:217）从 path 抽 id 的既有写法。

- [ ] **Step 1: 写失败测试**

`internal/server/http_test.go` 追加（参照文件内既有 `NewHTTPServer(Config{...})` + `httptest` 骨架，:889 附近有 session 测试样板）：

```go
type stubDecider struct {
	gotTask, gotTicket string
	gotStatus          approval.ApprovalStatus
	err                error
}

func (s *stubDecider) Decide(_ context.Context, taskID, ticketID string, status approval.ApprovalStatus) (approval.ToolApproval, error) {
	s.gotTask, s.gotTicket, s.gotStatus = taskID, ticketID, status
	if s.err != nil {
		return approval.ToolApproval{}, s.err
	}
	return approval.ToolApproval{TicketID: ticketID, TaskID: taskID, Status: status}, nil
}

func TestDecideApprovalRoutesApprove(t *testing.T) {
	dec := &stubDecider{}
	srv := NewHTTPServer(Config{ToolApprovals: dec})
	req := httptest.NewRequest(http.MethodPost, "/v1/tasks/t1/approvals/t1__c1", strings.NewReader(`{"decision":"approve"}`))
	rec := httptest.NewRecorder()
	srv.ServeHTTP(rec, req)
	if rec.Code != http.StatusOK {
		t.Fatalf("status = %d, want 200; body=%s", rec.Code, rec.Body.String())
	}
	if dec.gotTask != "t1" || dec.gotTicket != "t1__c1" || dec.gotStatus != approval.ApprovalApproved {
		t.Fatalf("decider got task=%q ticket=%q status=%q", dec.gotTask, dec.gotTicket, dec.gotStatus)
	}
}

func TestDecideApprovalInvalidDecision400(t *testing.T) {
	srv := NewHTTPServer(Config{ToolApprovals: &stubDecider{}})
	req := httptest.NewRequest(http.MethodPost, "/v1/tasks/t1/approvals/tk1", strings.NewReader(`{"decision":"maybe"}`))
	rec := httptest.NewRecorder()
	srv.ServeHTTP(rec, req)
	if rec.Code != http.StatusBadRequest {
		t.Fatalf("status = %d, want 400", rec.Code)
	}
}

func TestDecideApprovalUnknownTicket404(t *testing.T) {
	srv := NewHTTPServer(Config{ToolApprovals: &stubDecider{err: approval.ErrTicketNotFound}})
	req := httptest.NewRequest(http.MethodPost, "/v1/tasks/t1/approvals/nope", strings.NewReader(`{"decision":"deny"}`))
	rec := httptest.NewRecorder()
	srv.ServeHTTP(rec, req)
	if rec.Code != http.StatusNotFound {
		t.Fatalf("status = %d, want 404", rec.Code)
	}
}

func TestDecideApprovalNilStore503(t *testing.T) {
	srv := NewHTTPServer(Config{})
	req := httptest.NewRequest(http.MethodPost, "/v1/tasks/t1/approvals/tk1", strings.NewReader(`{"decision":"approve"}`))
	rec := httptest.NewRecorder()
	srv.ServeHTTP(rec, req)
	if rec.Code != http.StatusServiceUnavailable {
		t.Fatalf("status = %d, want 503", rec.Code)
	}
}
```

> `approval.ErrTicketNotFound` 已存在（`service.go:39`）。ToolGateStore 的 `Decide` 未知票据应返回 wrap 了 `ErrTicketNotFound` 的 error——**回到任务 2 Step 3 让 `Get` `!ok` 分支返回 `fmt.Errorf("... %w", ErrTicketNotFound)`**，使 HTTP 层可 `errors.Is` 判 404。（调整任务 2 实现：`Decide` 的未知票据、`Get` 供 Decide 用时 `!ok` → wrap `ErrTicketNotFound`。）

- [ ] **Step 2: 跑确认失败**

Run: `go test ./internal/server/ -run DecideApproval -v`
Expected: FAIL — 路由/字段不存在。

- [ ] **Step 3: 实现路由 + handler**

`http.go`：
1. `Config` 加 `ToolApprovals ApprovalDecider`；`HTTPServer` 加 `toolApprovals ApprovalDecider`；`NewHTTPServer` 赋值；定义 `ApprovalDecider` 接口 + import `approval`、`errors`。
2. 路由 switch（在 `/v1/tasks/` 段之前，因更具体）加：
   ```go
   case r.Method == http.MethodPost && strings.HasPrefix(r.URL.Path, "/v1/tasks/") && strings.Contains(r.URL.Path, "/approvals/"):
       s.handleDecideApproval(rec, r)
   ```
   放在 `case ... "/v1/tasks/") && strings.HasSuffix(..., "/result")`（:225）同侧、`handleGetTask`（:227）之前，避免被 GET 分支影响（此为 POST，天然不冲突，但顺序清晰）。
3. handler：
   ```go
   // handleDecideApproval records a human approve/deny on a Manual-mode tool
   // approval ticket and lets the coordinator resume the task. Path:
   // POST /v1/tasks/{taskID}/approvals/{ticketID}, body {"decision":"approve"|"deny"}.
   func (s *HTTPServer) handleDecideApproval(w http.ResponseWriter, r *http.Request) {
       if s.toolApprovals == nil {
           writeError(w, http.StatusServiceUnavailable, "approval store is unavailable")
           return
       }
       rest := strings.TrimPrefix(r.URL.Path, "/v1/tasks/")
       parts := strings.SplitN(rest, "/approvals/", 2)
       if len(parts) != 2 || parts[0] == "" || parts[1] == "" {
           writeError(w, http.StatusBadRequest, "malformed approval path")
           return
       }
       taskID, ticketID := parts[0], parts[1]
       var req struct{ Decision string `json:"decision"` }
       if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
           writeError(w, http.StatusBadRequest, "invalid approval request")
           return
       }
       var status approval.ApprovalStatus
       switch req.Decision {
       case "approve":
           status = approval.ApprovalApproved
       case "deny":
           status = approval.ApprovalDenied
       default:
           writeError(w, http.StatusBadRequest, "decision must be approve or deny")
           return
       }
       rec, err := s.toolApprovals.Decide(r.Context(), taskID, ticketID, status)
       if err != nil {
           if errors.Is(err, approval.ErrTicketNotFound) {
               writeError(w, http.StatusNotFound, "approval ticket not found")
               return
           }
           writeError(w, http.StatusInternalServerError, "decide approval failed")
           return
       }
       writeJSON(w, http.StatusOK, rec)
   }
   ```

- [ ] **Step 4: 跑确认通过 + 门禁 + commit**

Run: `go test ./internal/server/ -run DecideApproval -v` → PASS
```bash
gofmt -w internal/server/http.go internal/server/http_test.go
go build ./... && go vet ./... && go test ./internal/server/
git add internal/server/http.go internal/server/http_test.go
git commit -m "feat(server): POST /v1/tasks/{id}/approvals/{ticketID} decides a Manual tool-approval ticket"
```

---

## Task 6: 超时 deny 清扫

**Files:**
- Create: `internal/manualgate/timeout.go`（`NewTimeoutSweepJob`）
- Test: `internal/manualgate/timeout_test.go`

**Interfaces:**
- Produces:
  ```go
  // NewTimeoutSweepJob returns a background job that denies every pending tool
  // approval older than ttl (measured against now()), routing each through the
  // ApprovalCoordinator so the owning task resumes down the deny branch. A denied-
  // on-timeout ticket is a contract outcome (reject result to the model), not a
  // silent drop — the job logs a warn per timeout. ttl<=0 disables the sweep.
  func NewTimeoutSweepJob(store *approval.ToolGateStore, dec *ApprovalCoordinator, ttl time.Duration, now func() time.Time, logger *slog.Logger) func(context.Context) error
  ```
- Consumes: `store.ListPending()`（任务 2）、`dec.Decide(...ApprovalDenied)`（任务 4）。签名对齐 `task.BackgroundScheduler.AddJob`（`func(context.Context) error`，见 command.go:1833 用法）。

- [ ] **Step 1: 写失败测试**

`internal/manualgate/timeout_test.go`：注入固定 `now`，构造两张 pending 票据（一张 CreatedAt 早于 ttl 窗口、一张新），跑 job，断言旧票据被 deny、新票据仍 pending，且旧票据所属任务被翻 Running（若 Suspended 且全决）。

```go
func TestTimeoutSweepDeniesStalePending(t *testing.T) {
	dir := t.TempDir()
	store := approval.NewToolGateStore(dir)
	sched := task.NewScheduler()
	_ = sched.Add(context.Background(), domain.Task{ID: "t1", SessionID: "s1", Status: domain.TaskRunning})
	_ = sched.Transition(context.Background(), "t1", domain.TaskSuspended)
	// stale ticket — but ToolApproval.CreatedAt is set by Open to time.Now();
	// to control age, Open then rewrite CreatedAt on disk, or expose a seam.
	old, _ := store.Open(approval.ToolApproval{SessionKey: "s1", TaskID: "t1", ToolCallID: "c1", ToolName: "write_file"})
	_ = old
	dec := NewApprovalCoordinator(store, sched)
	fixedNow := func() time.Time { return time.Now().Add(10 * time.Minute) } // far future → everything stale
	job := NewTimeoutSweepJob(store, dec, 5*time.Minute, fixedNow, slog.New(slog.NewTextHandler(io.Discard, nil)))
	if err := job(context.Background()); err != nil {
		t.Fatal(err)
	}
	got, _, _ := store.Get("s1", approval.TicketID("t1", "c1"))
	if got.Status != approval.ApprovalDenied {
		t.Fatalf("stale ticket status = %s, want denied", got.Status)
	}
	if st, _, _ := sched.Get(context.Background(), "t1"); st.Status != domain.TaskRunning {
		t.Fatalf("task after timeout-deny = %s, want running", st.Status)
	}
}
```

> 用「未来 now + 短 ttl」制造 staleness，免去改 CreatedAt。断言 age 判定 = `now().Sub(rec.CreatedAt) > ttl`。

- [ ] **Step 2: 跑确认失败 → 实现 timeout.go → 通过**

```go
func NewTimeoutSweepJob(store *approval.ToolGateStore, dec *ApprovalCoordinator, ttl time.Duration, now func() time.Time, logger *slog.Logger) func(context.Context) error {
	return func(ctx context.Context) error {
		if ttl <= 0 {
			return nil
		}
		pending, err := store.ListPending()
		if err != nil {
			return fmt.Errorf("list pending approvals for timeout sweep: %w", err)
		}
		for _, rec := range pending {
			if now().Sub(rec.CreatedAt) <= ttl {
				continue
			}
			if _, err := dec.Decide(ctx, rec.TaskID, rec.TicketID, approval.ApprovalDenied); err != nil {
				return fmt.Errorf("timeout-deny ticket %s: %w", rec.TicketID, err)
			}
			if logger != nil {
				logger.Warn("approval timed out, auto-denied",
					"task_id", rec.TaskID, "ticket_id", rec.TicketID, "tool", rec.ToolName)
			}
		}
		return nil
	}
}
```

Run: `go test ./internal/manualgate/ -run TestTimeoutSweep -v` → PASS

- [ ] **Step 3: 门禁 + commit**

```bash
gofmt -w internal/manualgate/timeout.go internal/manualgate/timeout_test.go
go build ./... && go vet ./... && go test ./internal/manualgate/
git add internal/manualgate/timeout.go internal/manualgate/timeout_test.go
git commit -m "feat(manualgate): background sweep auto-denies timed-out pending approvals"
```

---

## Task 7: serve 接线（default runtime + resolver + coordinator + HTTP + 超时 job + 启动 reconcile）

**Files:**
- Modify: `internal/runtime/agent_resolver.go`（config 加 `Checkpoints`+`ToolGate`，`NewRuntime` 接上）
- Modify: `internal/cli/command.go`（serve 装配）
- Test: `internal/runtime/agent_resolver_test.go`（若存在则加断言；否则轻量新增）+ 现有 serve/集成测试保持绿

**Interfaces:**
- Consumes: 任务 2-6 全部产物。闭合 `command.go:1731` 的 `TODO(M2)`。

- [ ] **Step 1: resolver 接 Checkpoints+ToolGate（TDD）**

`agent_resolver.go`：
- `AgentRuntimeResolverConfig` 加 `Checkpoints *sessionstate.Store` + `ToolGate runtime`?——注意 resolver 在 `runtime` 包内（`package runtime`? 确认：agent_resolver.go 顶部 `package runtime`? 实为 `internal/runtime`，同包）。故直接用 `ToolGate`（本包接口类型）+ `*sessionstate.Store`。
- `AgentRuntimeResolver` 加对应字段；`NewAgentRuntimeResolver` 赋值。
- `ResolveTaskRunner` 的 `NewRuntime(Config{...})`（:94）加 `Checkpoints: r.checkpoints, ToolGate: r.toolGate,`。

先写测试断言 resolver 构造的 runtime 携带 gate：最简做法——`ResolveTaskRunner` 返回的 `TaskRunner` 是 `*Runtime`，类型断言后检查 `.toolGate != nil`/`.checkpoints != nil`（同包可访问非导出字段）。

```go
func TestResolverInjectsCheckpointsAndGate(t *testing.T) {
	// build a minimal resolver with a registered agent + stub maas factory
	// (reuse existing agent_resolver_test.go helpers if present) ...
	cfgStore := sessionstate.NewStore(t.TempDir())
	gate := manualgate.New(approval.NewToolGateStore(t.TempDir()))
	res := NewAgentRuntimeResolver(AgentRuntimeResolverConfig{
		Registry: reg, RootConfig: rootCfg, MaasFactory: stubFactory,
		Checkpoints: cfgStore, ToolGate: gate,
	})
	_, runner, ok, err := res.ResolveTaskRunner(context.Background(), domain.Task{AgentID: "a1"})
	if err != nil || !ok {
		t.Fatalf("resolve: ok=%v err=%v", ok, err)
	}
	rt := runner.(*Runtime)
	if rt.checkpoints == nil || rt.toolGate == nil {
		t.Fatalf("resolver runtime missing checkpoints/gate: cp=%v gate=%v", rt.checkpoints, rt.toolGate)
	}
}
```

> 复核点：`agent_resolver.go` 是否 `package runtime`（是——`internal/runtime/agent_resolver.go`）。若 resolver 导入 `manualgate` 会否成环？`manualgate` 不 import `runtime`（gate 结构化满足接口），故无环。但测试里 `import manualgate` 在 `runtime` 包测试中——`manualgate` import `runtime`? 不——manualgate 只 import approval/tool/domain。安全。

Run: `go test ./internal/runtime/ -run TestResolverInjects -v` → PASS

- [ ] **Step 2: command.go serve 装配**

在 `command.go` serve 内（行号约数，Read 复核）：
1. `workspaceRoot` 解析后（:1796）、`checkpointStore` 构造后（:1797），加：
   ```go
   toolApprovalStore := approval.NewToolGateStore(workspaceRoot)
   manualGate := manualgate.New(toolApprovalStore)
   ```
2. resolver 构造（:1732）加 `Checkpoints: checkpointStore, ToolGate: manualGate,`，删除 `// TODO(M2)` 注释（换成说明已接线的注释）。
3. `defaultRuntime := NewRuntime(Config{...})`（:1802）加 `ToolGate: manualGate,`（`Checkpoints` 已有）。
4. coordinator 构造（:1813）加 `Checkpoints: checkpointStore,`。
5. `approvalCoordinator := manualgate.NewApprovalCoordinator(toolApprovalStore, liveTasks)`（`liveTasks` = scheduler，:1709）。
6. `background.AddJob("approval-timeout-sweep", manualgate.NewTimeoutSweepJob(toolApprovalStore, approvalCoordinator, <ttl>, time.Now, logger))`——ttl 从配置读（见 Step 3）；放在其他 `AddJob` 附近（:1832-1869）。注意 `logger` 在 :1877 才定义——把 sweep job 的 AddJob 挪到 logger 定义之后，或用闭包延迟取 logger。**建议**：把该 `AddJob` 放到 `background.SetLogger(logger)`（:1895）之后。
7. `httpServer := NewHTTPServer(server.Config{...})`（:1906）加 `ToolApprovals: approvalCoordinator,`。
8. **启动 reconcile**：`RecoverSuspended`（:1884）之后，对每个恢复的挂起任务，若其票据已全决（重启前已 approve 但没跑完）→ 翻 Running 让 resume 扫描续跑。实现：遍历 `liveTasks.List` 中 `Status==Suspended` 的任务，调 `approvalCoordinator.ReconcileResume(ctx, taskID)`（新增薄方法：若该任务有票据且全决 → Transition Suspended→Running；无票据则跳过——那是纯 checkpoint 挂起等其它决定）。
   > 给 `ApprovalCoordinator` 加 `ReconcileResume(ctx, taskID) error`：`ListForTask` 空→跳过；有 pending→留 Suspended；全决→翻 Running。复用 Decide 的 all-decided 逻辑（抽 helper）。补一个单测放 decider_test.go。

- [ ] **Step 3: 配置项 — 审批超时 ttl**

Grep `config.RuntimeConfig`（`internal/config`）确认字段位置；加 `ApprovalTimeoutSeconds int`（默认 300）。若 config 结构复杂，最小侵入：读 `cfg.Runtime.ApprovalTimeoutSeconds`，`<=0` 时用 `300`。加一行默认值注释。补 config 默认值测试（若 config 包有默认值测试惯例）。

- [ ] **Step 4: 全仓门禁 + WSL race 全绿**

```bash
go build ./... && go vet ./... && go test ./...
gofmt -l internal/ | <剥 \r 验证被改文件干净>
```
WSL：
```bash
wsl -d Ubuntu-22.04 -- bash -lc 'cd /mnt/f/source/stardust/Legion/legion/legionAgent && CGO_ENABLED=1 CC=gcc $HOME/sdk/go/bin/go test -race ./...'
```
Expected: 全 PASS，无 race。

- [ ] **Step 5: commit**

```bash
git add internal/runtime/agent_resolver.go internal/runtime/agent_resolver_test.go internal/cli/command.go internal/manualgate/decider.go internal/manualgate/decider_test.go internal/config/...
git commit -m "feat(serve): wire Manual approval gate into default + per-agent runtimes, coordinator, HTTP, timeout sweep, restart reconcile"
```

---

## Self-Review（对照 spec §4.3 / §6 / §7 / §8 / §9）

- **§4.3 gate 点唯一 choke**：`dispatchToolCall` 接决定检查（任务 3 Step 7）✓；round 级 `ShouldSuspend`（任务 3）✓。
- **§4.3 lazy + eager 两协议**：`resolveRealTool` 解 `call_tool`（任务 3）；eager 直名同样查 Sensitive ✓。
- **§4.3 approval 会话目录持久化 + 磁盘真相源 + 启动加载**：`ToolGateStore`（任务 2）+ 启动 reconcile（任务 7 Step 2.8）✓。
- **§4.3 Decide 翻 Suspended→Running**：`ApprovalCoordinator.Decide`（任务 4）✓。
- **§4.3 超时→deny+warn+不杀任务**：任务 6 ✓。
- **§4.1b 检查点携带恢复所需态 + Mode**：任务 1（Mode，schema v2）✓；toolCtx/round/pendingCalls M1b 已存 ✓。
- **§6 数据模型**：`ToolApproval` 字段（SessionID/TaskID/ToolCallID/ToolName/Arguments/Status/CreatedAt）✓；checkpoint schema 版本化 ✓。
- **§7 API**：`POST /v1/tasks/{id}/approvals/{ticketID}`（任务 5）✓。（`GET /v1/events` SSE = M2c，非本里程碑。列票据端点 = 可选，未纳入——若 reviewer 要求再加。）
- **§8 fail-loud**：损坏 JSON 报错（任务 2）；未知 mode = domain 已管；盘写失败传播 error；deny=契约结果非兜底 ✓。
- **§9 测试**：manual+敏感开票据+挂起 ✓；approve→执行 / deny→拒绝回模型 ✓；timeout→deny ✓；只读不触发 ✓;lazy call_tool 指名敏感同样 gate ✓;`-race` 并发无双重派发 ✓。
- **架构缺口（交接 §3.3）**：resumeScan 补齐 Running+checkpoint+lockable 重拾 ✓；挂起释放锁 ✓。

**遗留（非 M2b 阻塞，记入分支收尾说明）：**
- resolver runtimes 接 store 后 Plan 也能落 OKF（任务 7 顺带获得，因 `Checkpoints` 接上）。
- 列票据 `GET` 端点（spec 标「可选」）未做。
- SSE `approval_pending` 推送 = M2c。
- 死代码 `func max`(runtime.go:722)、`errStaticResolver`(coordinator_test.go) — 另开清理任务。

---

## 执行说明

- **分支**：`git checkout -b feat/agent-modes-m2b`（从 master `e644b20`）。
- **SDD ledger**：`legionAgent/.superpowers/sdd/progress.md`；每任务手工抽 brief（`task-brief` 脚本按英文 "Task N" 匹配，中文标题「任务 N」认不出 → 手工抽到 `.superpowers/sdd/task-N-brief.md`）。
- **子 agent 报告写文件**（`task-N-report.md`/`task-N-review.md`），控制方读文件核验，别信被 hook 吞的 chat 返回；诊断里的编译错常是 mid-TDD-red 快照，自己 `go build` 确认最终态。
- **reviewer 分级**：任务 3/4（gate + 并发）用 opus code-reviewer；其余 sonnet。修 Critical/Important 后记 ledger。
- **收尾**：整分支 opus final review → superpowers:finishing-a-development-branch → PR（中文正文）+ 合并。
