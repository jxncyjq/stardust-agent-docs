---
title: 实施计划 — Milestone 3a（working_dir 后端：per-session 沙箱 + 会话目录随 working_dir）
type: plan
status: active
created: 2026-07-19
scope: legion/legionAgent（后端）
related:
  - "[[2026-07-15-agent-working-modes-design]]"
  - "[[m3-explore-workingdir]]"
tags: [agent, working-dir, sandbox, session-dir, milestone-3a, plan]
---

# Agent Working Modes — Milestone 3a (working_dir 后端) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax.

**Goal:** 让每个会话可带 `working_dir`：任务的工具文件操作沙箱在 working_dir 内，且会话状态（审批票据/checkpoint/plans）落 `<working_dir>/.stardust/session/<id>`（无 working_dir 则落 workspace.root），跨重启恢复正确扫描多个 base。

**Architecture:** 单一 resolver `sessionstate.SessionBase(workspaceRoot, workingDir)` 决定会话目录 base（working_dir 非空→`<working_dir>/.stardust`，否则 workspaceRoot）。`Checkpoint` 与 `ToolApproval` 记录内嵌 `WorkingDir`，Store/ToolGateStore 每次操作按记录的 working_dir 解析 base（不再是构造期单一固定 base）。重启恢复与超时 sweep 从 SessionStore 枚举全部会话的 working_dir，得到 base 集合，逐 base 扫描。working_dir 从 session 解析进 `domain.Task`，喂给 agentToolRoot / WorkspacePathGuard 做沙箱根；建任务时校验目录存在（否则 400）。

**Tech Stack:** Go 1.26；`internal/sessionstate`（resolver + checkpoint Store）；`internal/approval`（ToolGateStore）；`internal/domain`；`internal/storage`（SQLite 迁移）；`internal/server`（session/task 端点）；`internal/runtime`（agentToolRoot / checkpoint save-load）；`internal/cli/command.go`（serve 装配 + 多 base 恢复）；WSL race。

## Global Constraints

- **会话目录 base 规则（路线 A，已拍板）**：`SessionBase(workspaceRoot, workingDir)` = workingDir 非空时 `filepath.Join(workingDir, ".stardust")`，否则 workspaceRoot。**单一 resolver**（spec §4.0「别在多处散拼」）——所有落盘路径经它，禁止散拼。
- **Fail-Loud 铁律**（`legionAgent/CLAUDE.md §0`）：禁 fallback/zero-value 兜底、禁 `_ = err`、禁静默 continue/return。working_dir 非目录→**返回 error（建任务 400）**不静默。重启恢复扫描某 base 失败→fail-loud 不跳过（recovery 不得静默丢任务）。
- **working_dir 校验**（spec §4.4/§8）：非空则 `os.Stat` + `IsDir`，否则建任务返 400；空值合法（落 workspace.root）。校验点 = 建任务时（`handleCreateTask`）。
- **YAGNI（spec §11）**：working_dir 只做 per-session 沙箱根 + 会话目录定位；不做 per-task 覆盖 session；不给 tasks 表加列（mode 也没加，working_dir 同样走内存从 session 解析进 domain.Task）。会话元数据仍留 SQLite，只有票据/计划/检查点落会话目录。
- **跨平台**：路径用 `filepath`（不用字符串拼）。working_dir 校验注意 Windows 盘符/UNC；`os.Stat` 跟随 symlink（本里程碑不额外 EvalSymlinks，除非测试暴露越界绕过——若做需在 plan 内记）。
- **sessionKey 双份定义**：`runtime/checkpoint.go` 与 `manualgate/manualgate.go` 各有一份 `sessionKeyForTask`（SessionID 否则 ID），改动保持两处一致。
- **迁移增量**：既有 `agent.db` 已存在，新列必须走 `applyColumnMigrations` 增量 ALTER（仿 mode 列 v5），不能只改 CREATE TABLE。
- **完成门禁**：Windows `go build ./... && go vet ./... && go test ./...` 全绿；`gofmt -l` 对触碰文件 LF 干净（只 `gofmt -w` 自己改的文件）；WSL `-race` 绿（Task 3/4/5 涉持久化并发，务必 race）。
- **仓库**：全部代码在 `legion/legionAgent`（remote `github.com/jxncyjq/stardust-agent-server`）。分支 `feat/agent-modes-m3a`（已从 master `6e4e396` 切出，M2c 已合入）。

## WSL race recipe（Windows 无 gcc）
```bash
wsl -d Ubuntu-22.04 -- bash -lc 'cd /mnt/f/source/stardust/Legion/legion/legionAgent && GOOS=linux GOARCH=amd64 CGO_ENABLED=1 CC=gcc $HOME/sdk/go/bin/go test -race ./internal/sessionstate/ ./internal/approval/ ./internal/runtime/ ./internal/server/ ./internal/storage/ ./internal/cli/ ./internal/manualgate/'
```

---

## File Structure

- **Modify** `internal/domain/types.go` — `AgentSession.WorkingDir`、`Task.WorkingDir`。Task 1。
- **Modify** `internal/sessionstate/resolver.go` — 新增 `SessionBase(workspaceRoot, workingDir) string`；`SessionDir` 语义不变（仍 `<base>/session/<key>`，base 由 SessionBase 给）。Task 1。
- **Modify** `internal/storage/sqlite.go` — 迁移 v6 + agent_sessions.working_dir 列 + Save/Get/List/Scan。Task 2。
- **Modify** `internal/sessionstate/checkpoint.go` — `Checkpoint.WorkingDir`；Store 持 workspaceRoot，`Save`/`Load`/`Delete`/`WritePlan` 按 working_dir 解析 base；`ListSuspendedIn(base)` + 保留/改造 `ListSuspended`。Task 3。
- **Modify** `internal/runtime/*` — checkpoint save/load 传 task.WorkingDir（写检查点点 + resume 加载点）。Task 3。
- **Modify** `internal/approval/toolgate_store.go` — `ToolApproval.WorkingDir`；Open/Get/Decide/ListForTask 按 working_dir 解析 base；`ListPendingIn(base)`。Task 4。
- **Modify** `internal/manualgate/*` — ShouldSuspend 开票据带 working_dir；Decide/Resolve 用 working_dir 定位。Task 4。
- **Modify** `internal/cli/command.go` — 多 base 恢复（枚举 SessionStore working_dir→base 集合）+ 超时 sweep 多 base + serve 装配传 workspaceRoot 给 Store（Store 不再吃 working_dir 构造）。Task 5。
- **Modify** `internal/server/http.go` — createSession/patchSession 收 working_dir；createTask 从 session 解析 working_dir 进 Task + 校验目录（400）。Task 6。
- **Modify** `internal/runtime/agent_resolver.go` — `agentToolRoot(rootCfg, agentCfg, task)` task.WorkingDir 优先。Task 7。
- **Modify** `internal/cli/command.go` — 默认 agent 走 per-task tool root。Task 7。
- **Modify** `internal/server/http.go` + `internal/storage/sqlite.go` — handleDeleteSession 连带删会话目录。Task 8。

各 Modify 配套 `_test.go`。

---

### Task 1: 数据模型 + SessionBase 单一 resolver

**Files:**
- Modify: `internal/domain/types.go`（AgentSession @147-157、Task @62-76）
- Modify: `internal/sessionstate/resolver.go`（加 SessionBase）
- Test: `internal/sessionstate/resolver_test.go`（已存在，追加）

**Interfaces:**
- Produces: `domain.AgentSession.WorkingDir string`（json `working_dir,omitempty`）；`domain.Task.WorkingDir string`（json `working_dir,omitempty`）；`sessionstate.SessionBase(workspaceRoot, workingDir string) string`。

- [ ] **Step 1: 写 SessionBase 失败测试**

`internal/sessionstate/resolver_test.go` 追加：

```go
func TestSessionBaseUsesWorkspaceRootWhenNoWorkingDir(t *testing.T) {
	got := SessionBase("/ws/root", "")
	if got != "/ws/root" {
		t.Fatalf("SessionBase(root, \"\") = %q, want /ws/root", got)
	}
}

func TestSessionBaseUsesWorkingDirDotStardustWhenSet(t *testing.T) {
	got := SessionBase("/ws/root", filepath.Join("/proj", "app"))
	want := filepath.Join("/proj", "app", ".stardust")
	if got != want {
		t.Fatalf("SessionBase(root, /proj/app) = %q, want %q", got, want)
	}
}
```

（`filepath` 已在测试 import；若无则加。）

- [ ] **Step 2: 跑确认失败**

Run: `go test ./internal/sessionstate/ -run TestSessionBase`
Expected: FAIL（`undefined: SessionBase`）。

- [ ] **Step 3: 实现 SessionBase + 加字段**

`internal/sessionstate/resolver.go` 加：

```go
// SessionBase returns the base directory under which a session's state
// (checkpoints, approval tickets, plans) lives. When workingDir is non-empty the
// base is <workingDir>/.stardust (design §4.0: a session bound to a working dir
// keeps its state alongside that dir); otherwise it is the workspace root. This
// is the single resolver for the base — callers must never hand-join it
// elsewhere. SessionDir(SessionBase(root, wd), key) yields the full session dir.
func SessionBase(workspaceRoot, workingDir string) string {
	if strings.TrimSpace(workingDir) == "" {
		return workspaceRoot
	}
	return filepath.Join(workingDir, defaultRootName)
}
```

（`defaultRootName = ".stardust"` 已在 resolver.go:15；`strings` 已 import。）

`internal/domain/types.go`：`AgentSession` 加 `WorkingDir string \`json:"working_dir,omitempty"\``（在 Mode 之后）；`Task` 加 `WorkingDir string \`json:"working_dir,omitempty"\``（在 Mode 之后）。加 Go doc 行注释说明用途。

- [ ] **Step 4: 跑确认通过**

Run: `go build ./... && go test ./internal/sessionstate/ -run TestSessionBase`
Expected: PASS。

- [ ] **Step 5: gofmt + 提交**

```bash
gofmt -w internal/sessionstate/resolver.go internal/sessionstate/resolver_test.go internal/domain/types.go
git add internal/sessionstate/resolver.go internal/sessionstate/resolver_test.go internal/domain/types.go
git commit -m "feat(domain): Session/Task 加 WorkingDir + SessionBase 单一 resolver"
```

---

### Task 2: SQLite 迁移 v6 — agent_sessions.working_dir

**Files:**
- Modify: `internal/storage/sqlite.go`（schemaVersion @30-36、迁移表 @1404-1437、DDL @1651-1660、Save INSERT @217-229、SELECT @274/295/311/551、scanAgentSession @1511）
- Test: `internal/storage/sqlite_test.go`（追加）

**Interfaces:**
- Consumes: `domain.AgentSession.WorkingDir`（Task 1）。
- Produces: SQLite 持久化 session.working_dir；round-trip 保持。

- [ ] **Step 1: 写 round-trip 失败测试**

`internal/storage/sqlite_test.go` 追加（仿现有 agent_session save/get 测试）：

```go
func TestSaveGetAgentSessionPreservesWorkingDir(t *testing.T) {
	ctx := context.Background()
	repo, err := OpenSQLite(ctx, filepath.Join(t.TempDir(), "agent.db"))
	if err != nil {
		t.Fatalf("OpenSQLite error = %v, want nil", err)
	}
	t.Cleanup(func() { _ = repo.Close() })
	sess := domain.AgentSession{
		ID: "s1", CompanyID: "c1", AgentID: "a1", Mode: "manual",
		WorkingDir: filepath.Join("/proj", "app"),
	}
	if err := repo.SaveAgentSession(ctx, sess); err != nil { // 用实际方法名，见注
		t.Fatalf("SaveAgentSession error = %v, want nil", err)
	}
	got, ok, err := repo.GetAgentSession(ctx, "s1")
	if err != nil || !ok {
		t.Fatalf("GetAgentSession = _, %v, %v", ok, err)
	}
	if got.WorkingDir != sess.WorkingDir {
		t.Fatalf("WorkingDir = %q, want %q", got.WorkingDir, sess.WorkingDir)
	}
}
```

> 注：`SaveAgentSession`/`GetAgentSession` 用实际方法名——实现者先读 sqlite.go 现有 agent_session CRUD 方法名对齐，勿臆造。

- [ ] **Step 2: 跑确认失败**

Run: `go test ./internal/storage/ -run TestSaveGetAgentSessionPreservesWorkingDir`
Expected: FAIL（working_dir 未持久化，got 为空）。

- [ ] **Step 3: 实现迁移 + 列**

- schemaVersion 常量 +1（现 v5→v6，sqlite.go:30-36 注释同步）。
- 迁移表加项（仿 mode 列 v5，sqlite.go:1433-1435）：`ALTER TABLE agent_sessions ADD COLUMN working_dir TEXT NOT NULL DEFAULT ''`。
- 建表 DDL 加 `working_dir TEXT NOT NULL DEFAULT ''`（sqlite.go:1651-1660）。
- Save INSERT 列 + 占位 + 值（sqlite.go:217-229）；各 SELECT 列表加 working_dir（:274/295/311/551）；scanAgentSession Scan 加 `&s.WorkingDir`（:1511）。

- [ ] **Step 4: 跑确认通过（含既有 storage 测试不回退）**

Run: `go test ./internal/storage/`
Expected: PASS（新测试 + 既有全绿；迁移对已存在 db 增量生效）。

- [ ] **Step 5: gofmt + 提交**

```bash
gofmt -w internal/storage/sqlite.go internal/storage/sqlite_test.go
git add internal/storage/sqlite.go internal/storage/sqlite_test.go
git commit -m "feat(storage): 迁移 v6 agent_sessions.working_dir 列 + CRUD"
```

---

### Task 3: Checkpoint Store per-session base（working_dir 感知）— 高危核心

**Files:**
- Modify: `internal/sessionstate/checkpoint.go`（Checkpoint @34-53、Store @55-65、Save @69、Load @96、Delete @117、WritePlan @129、ListSuspended @148）
- Modify: `internal/runtime/checkpoint.go` + `internal/runtime/runtime.go`（写检查点点 checkSuspend、resume 加载点）
- Test: `internal/sessionstate/checkpoint_test.go`（追加）；runtime 侧既有 resume 测试更新

**Interfaces:**
- Consumes: `sessionstate.SessionBase`（Task 1）；`Checkpoint.WorkingDir`。
- Produces: `Checkpoint.WorkingDir string`；`Store.Save(cp)`（按 cp.WorkingDir 解析 base）；`Store.Load(sessionKey, workingDir)`；`Store.Delete(sessionKey, workingDir)`；`Store.WritePlan(sessionKey, workingDir, filename, content)`；`Store.ListSuspendedIn(base) ([]Checkpoint, error)`。**Store 构造仍 `NewStore(workspaceRoot)`；workspaceRoot 作为 working_dir 为空时的 base。**

- [ ] **Step 1: 写 working_dir 感知落盘失败测试**

`internal/sessionstate/checkpoint_test.go` 追加：

```go
func TestCheckpointSaveLoadUnderWorkingDir(t *testing.T) {
	workspaceRoot := t.TempDir()
	workingDir := t.TempDir()
	s := NewStore(workspaceRoot)
	cp := Checkpoint{
		SchemaVersion: CheckpointSchemaVersion, TaskID: "t1", SessionKey: "s1",
		WorkingDir: workingDir, BasePrompt: "p", CreatedAt: time.Unix(1, 0),
	}
	if err := s.Save(cp); err != nil {
		t.Fatalf("Save error = %v, want nil", err)
	}
	// Physically under <workingDir>/.stardust/session/s1, NOT workspaceRoot.
	want := filepath.Join(workingDir, ".stardust", "session", "s1", "task-state.json")
	if _, err := os.Stat(want); err != nil {
		t.Fatalf("checkpoint not at %q: %v", want, err)
	}
	got, ok, err := s.Load("s1", workingDir)
	if err != nil || !ok {
		t.Fatalf("Load = _, %v, %v; want found", ok, err)
	}
	if got.TaskID != "t1" {
		t.Fatalf("Load TaskID = %q, want t1", got.TaskID)
	}
}

func TestCheckpointSaveLoadWorkspaceRootWhenNoWorkingDir(t *testing.T) {
	workspaceRoot := t.TempDir()
	s := NewStore(workspaceRoot)
	cp := Checkpoint{SchemaVersion: CheckpointSchemaVersion, TaskID: "t2", SessionKey: "s2", CreatedAt: time.Unix(1, 0)}
	if err := s.Save(cp); err != nil {
		t.Fatalf("Save error = %v, want nil", err)
	}
	want := filepath.Join(workspaceRoot, "session", "s2", "task-state.json")
	if _, err := os.Stat(want); err != nil {
		t.Fatalf("checkpoint not at workspaceRoot %q: %v", want, err)
	}
	if _, ok, _ := s.Load("s2", ""); !ok {
		t.Fatal("Load(s2, \"\") not found, want found")
	}
}

func TestListSuspendedInScansGivenBase(t *testing.T) {
	workspaceRoot := t.TempDir()
	workingDir := t.TempDir()
	s := NewStore(workspaceRoot)
	_ = s.Save(Checkpoint{SchemaVersion: CheckpointSchemaVersion, TaskID: "t1", SessionKey: "s1", WorkingDir: workingDir, CreatedAt: time.Unix(1, 0)})
	base := SessionBase(workspaceRoot, workingDir)
	got, err := s.ListSuspendedIn(base)
	if err != nil {
		t.Fatalf("ListSuspendedIn error = %v", err)
	}
	if len(got) != 1 || got[0].TaskID != "t1" {
		t.Fatalf("ListSuspendedIn = %#v, want 1 checkpoint t1", got)
	}
}
```

- [ ] **Step 2: 跑确认失败**

Run: `go test ./internal/sessionstate/ -run 'TestCheckpoint(SaveLoadUnderWorkingDir|SaveLoadWorkspaceRootWhenNoWorkingDir)|TestListSuspendedIn'`
Expected: FAIL（Save 忽略 WorkingDir 落 workspaceRoot；Load/ListSuspendedIn 签名不符）。

- [ ] **Step 3: 实现 Store working_dir 感知**

`internal/sessionstate/checkpoint.go`：
- `Checkpoint` 加 `WorkingDir string \`json:"working_dir,omitempty"\``（含 doc 注释：捕获挂起时 working_dir，使恢复用同一 base 定位会话目录）。
- Store doc 更新：`base` 字段改名/语义为 `workspaceRoot`（working_dir 为空时的 base）。构造 `NewStore(workspaceRoot)` 不变。
- `Save(cp)`：`base := SessionBase(s.workspaceRoot, cp.WorkingDir)`，其余用 base（现 `SessionDir(s.base, cp.SessionKey)` 改 `SessionDir(base, cp.SessionKey)`）。
- `Load(sessionKey, workingDir string)`：`base := SessionBase(s.workspaceRoot, workingDir)`；路径用 base。
- `Delete(sessionKey, workingDir string)`：同。
- `WritePlan(sessionKey, workingDir, filename, content string)`：同。
- `ListSuspended` 改为 `ListSuspendedIn(base string) ([]Checkpoint, error)`：扫 `<base>/session/*/task-state.json`（现 `ListSuspended` body 改为参数化 base；`s.base`→参数 base）。**保留一个无参 `ListSuspended()` 调 `ListSuspendedIn(s.workspaceRoot)` 供仅扫 workspaceRoot 的旧路径**（Task 5 会补多 base 枚举）。

更新 runtime 调用点：
- 写检查点（`internal/runtime/runtime.go` checkSuspend，约 :297 存 Mode 处）：构造 Checkpoint 时加 `WorkingDir: task.WorkingDir`。
- resume 加载（runtime resume 分支，调 `Load`/`Delete` 处）：传 `task.WorkingDir`。grep `\.Load(` / `\.Delete(` in internal/runtime 定位；两处签名跟改。

> **Store.base 字段重命名**注意全文件引用一致（Save/Load/Delete/WritePlan/ListSuspendedIn 都用）。

- [ ] **Step 4: 跑确认通过（含 runtime resume 测试不回退）**

Run: `go build ./... && go test ./internal/sessionstate/ ./internal/runtime/`
Expected: PASS。若 runtime 侧 resume 测试因签名变动编译失败，更新其对 Load/Delete 的调用传 workingDir（老测试多用空 working_dir，传 `""`）。

- [ ] **Step 5: gofmt + 提交**

```bash
gofmt -w internal/sessionstate/checkpoint.go internal/sessionstate/checkpoint_test.go internal/runtime/checkpoint.go internal/runtime/runtime.go
git add internal/sessionstate/ internal/runtime/
git commit -m "feat(sessionstate): checkpoint Store 按 working_dir 解析会话目录 base"
```

---

### Task 4: ToolGateStore per-session base（working_dir 感知）— 高危核心

**Files:**
- Modify: `internal/approval/toolgate_store.go`（ToolApproval @36-46、ToolGateStore @54-72、Open @104、Get @133、Decide @164、ListForTask @191、ListPending @211）
- Modify: `internal/manualgate/manualgate.go`（ShouldSuspend 开票据 @82、Resolve @99、sessionKeyForTask）、`internal/manualgate/decider.go`（Decide 定位票据）
- Test: `internal/approval/toolgate_store_test.go`、`internal/manualgate/*_test.go`（追加/更新）

**Interfaces:**
- Consumes: `sessionstate.SessionBase`（Task 1）；`ToolApproval.WorkingDir`。
- Produces: `ToolApproval.WorkingDir string`；ToolGateStore 方法按 working_dir 解析 base；`ToolGateStore.ListPendingIn(base) ([]ToolApproval, error)`；`ToolGateStore` 构造仍 `NewToolGateStore(workspaceRoot)`。

- [ ] **Step 1: 写 working_dir 感知票据失败测试**

`internal/approval/toolgate_store_test.go` 追加：

```go
func TestToolGateStoreTicketUnderWorkingDir(t *testing.T) {
	workspaceRoot := t.TempDir()
	workingDir := t.TempDir()
	s := NewToolGateStore(workspaceRoot)
	rec, err := s.Open(ToolApproval{
		SessionKey: "s1", TaskID: "t1", ToolCallID: "c1", ToolName: "write_file",
		WorkingDir: workingDir,
	})
	if err != nil {
		t.Fatalf("Open error = %v, want nil", err)
	}
	want := filepath.Join(workingDir, ".stardust", "session", "s1", "approvals", rec.TicketID+".json")
	if _, err := os.Stat(want); err != nil {
		t.Fatalf("ticket not at %q: %v", want, err)
	}
	got, ok, err := s.Get("s1", rec.TicketID, workingDir) // 见注：Get 签名加 workingDir
	if err != nil || !ok {
		t.Fatalf("Get = _, %v, %v; want found", ok, err)
	}
	if got.ToolName != "write_file" {
		t.Fatalf("Get ToolName = %q, want write_file", got.ToolName)
	}
}

func TestListPendingInScansGivenBase(t *testing.T) {
	workspaceRoot := t.TempDir()
	workingDir := t.TempDir()
	s := NewToolGateStore(workspaceRoot)
	_, _ = s.Open(ToolApproval{SessionKey: "s1", TaskID: "t1", ToolCallID: "c1", ToolName: "write_file", WorkingDir: workingDir})
	got, err := s.ListPendingIn(sessionstate.SessionBase(workspaceRoot, workingDir))
	if err != nil {
		t.Fatalf("ListPendingIn error = %v", err)
	}
	if len(got) != 1 {
		t.Fatalf("ListPendingIn = %d tickets, want 1", len(got))
	}
}
```

> 注：Get/Decide/ListForTask 都需加 workingDir 参数以定位 base。实现者先读现有签名，统一加 workingDir 末参；更新所有既有调用点（多传 `""`）。`sessionstate` 需 import 进 approval 包（检查无环依赖——approval 不被 sessionstate import，安全）。

- [ ] **Step 2: 跑确认失败**

Run: `go test ./internal/approval/ -run 'TestToolGateStoreTicketUnderWorkingDir|TestListPendingIn'`
Expected: FAIL（票据落 workspaceRoot；Get/ListPendingIn 签名不符）。

- [ ] **Step 3: 实现 ToolGateStore working_dir 感知**

- `ToolApproval` 加 `WorkingDir string \`json:"working_dir,omitempty"\``。
- ToolGateStore `base` 字段语义→workspaceRoot。
- `ticketPath`/各方法：`base := sessionstate.SessionBase(s.workspaceRoot, workingDir)`，用 base 定位 `SessionDir(base, sessionKey)/approvals/...`。Open 用 rec.WorkingDir；Get/Decide/ListForTask 加 workingDir 参。
- `ListPending()` 改 `ListPendingIn(base string)`：glob `<base>/session/*/approvals/*.json`。保留无参 `ListPending()` 调 `ListPendingIn(s.workspaceRoot)`（Task 5 补多 base）。
- manualgate：`ShouldSuspend`（manualgate.go:82）`store.Open` 时带 `WorkingDir: task.WorkingDir`；`Resolve`（:107 `store.Get`）传 `task.WorkingDir`；`decider.go` `Decide`（:49 `store.Decide`、:53 `ListForTask`）——Decide 需知 working_dir：从 `sched.Get(taskID)` 拿到的 `domain.Task.WorkingDir` 传入（Decide 已 Get task，见 decider.go:41）。

- [ ] **Step 4: 跑确认通过**

Run: `go build ./... && go test ./internal/approval/ ./internal/manualgate/`
Expected: PASS（既有票据测试传 `""` 后仍绿）。

- [ ] **Step 5: gofmt + 提交**

```bash
gofmt -w internal/approval/toolgate_store.go internal/approval/toolgate_store_test.go internal/manualgate/manualgate.go internal/manualgate/decider.go internal/manualgate/manualgate_test.go internal/manualgate/decider_test.go
git add internal/approval/ internal/manualgate/
git commit -m "feat(approval): ToolGateStore 按 working_dir 解析票据会话目录 base"
```

---

### Task 5: 多 base 重启恢复 + 超时 sweep

**Files:**
- Modify: `internal/cli/command.go`（恢复区 @1918-1948、超时 sweep @1965、serve 装配 Store 构造 @1755-1757）
- Modify: `internal/runtime/coordinator.go`（`RecoverSuspended` 签名，若需按 base 集合）
- Modify: `internal/manualgate/timeout.go`（sweep 用多 base）
- Test: `internal/cli/command_test.go`（追加多 base 恢复 e2e）；`internal/manualgate/timeout_test.go`（更新）

**Interfaces:**
- Consumes: `SessionStore`（枚举会话拿 working_dir）；`Store.ListSuspendedIn`（Task 3）；`ToolGateStore.ListPendingIn`（Task 4）；`SessionBase`。
- Produces: 重启恢复 + 超时 sweep 覆盖 workspace.root ∪ 各 session working_dir 的 base 集合。

**设计（base 集合枚举）**：新增 helper（放 cli 或 sessionstate）`func distinctSessionBases(ctx, sessions SessionLister, workspaceRoot string) ([]string, error)`：从 SessionStore.List 拿全部 session，`set = {workspaceRoot}`；每个 session `set.add(SessionBase(workspaceRoot, s.WorkingDir))`；返回去重 slice。恢复/ sweep 遍历 base 集合。

- [ ] **Step 1: 写多 base 恢复失败测试**

`internal/cli/command_test.go` 追加（起 serve，建一个带 working_dir 的 session、在其 working_dir/.stardust 下放一个挂起 checkpoint，重启 serve 断言被恢复）。因涉及 serve 起停 + 盘上构造，写为集成测试；若过重，退化为直接测 `distinctSessionBases` + `coordinator.RecoverSuspended` 跨两 base 的单元测试：

```go
func TestDistinctSessionBasesUnionsWorkspaceRootAndWorkingDirs(t *testing.T) {
	workspaceRoot := t.TempDir()
	wd1 := t.TempDir()
	sessions := fakeSessionLister{items: []domain.AgentSession{
		{ID: "s1", WorkingDir: wd1},
		{ID: "s2", WorkingDir: ""}, // 无 working_dir → workspaceRoot
	}}
	bases, err := distinctSessionBases(context.Background(), sessions, workspaceRoot)
	if err != nil {
		t.Fatalf("distinctSessionBases error = %v", err)
	}
	// 期望含 workspaceRoot 与 wd1/.stardust，去重（s2 的 base == workspaceRoot）
	assertContains(t, bases, workspaceRoot)
	assertContains(t, bases, sessionstate.SessionBase(workspaceRoot, wd1))
	if len(bases) != 2 {
		t.Fatalf("bases = %v, want 2 distinct", bases)
	}
}
```

> 注：`fakeSessionLister`/`assertContains` 由实现者按现有 SessionStore 接口定义（`List(ctx)` 返回 `[]domain.AgentSession`——先读实际签名对齐）。

- [ ] **Step 2: 跑确认失败**

Run: `go test ./internal/cli/ -run TestDistinctSessionBases`
Expected: FAIL（`undefined: distinctSessionBases`）。

- [ ] **Step 3: 实现多 base 枚举 + 接线恢复/sweep**

- 实现 `distinctSessionBases`。
- `coordinator.RecoverSuspended`：改为接受 base 集合（或接受一个返回全部挂起 checkpoint 的枚举器）。最小改动：command.go 恢复区先算 `bases`，对每个 base 调 `checkpointStore.ListSuspendedIn(base)` 汇总，喂给 coordinator 恢复逻辑（若 RecoverSuspended 现内部调 ListSuspended，改为传入已汇总 checkpoints，或让它遍历 bases）。保持 fail-loud（任一 base 扫描出错即返回 error）。
- 超时 sweep（command.go:1965 + timeout.go）：`NewTimeoutSweepJob` 现调 `store.ListPending()`（单 base）。改为 job 内先算 bases（需 SessionStore 引用）→ 遍历 `ListPendingIn(base)` 汇总。或给 job 传一个 `basesFn func()([]string,error)`。
- serve 装配（command.go:1755-1757）：`sessionstate.NewStore(workspaceRoot)`/`approval.NewToolGateStore(workspaceRoot)` 不变（workspaceRoot 仍是默认 base）。

- [ ] **Step 4: 跑确认通过**

Run: `go build ./... && go test ./internal/cli/ ./internal/manualgate/ ./internal/runtime/`
Expected: PASS。

- [ ] **Step 5: gofmt + 提交**

```bash
gofmt -w internal/cli/command.go internal/cli/command_test.go internal/runtime/coordinator.go internal/manualgate/timeout.go internal/manualgate/timeout_test.go
git add internal/cli/ internal/runtime/coordinator.go internal/manualgate/timeout.go internal/manualgate/timeout_test.go
git commit -m "feat(serve): 重启恢复+超时sweep 跨 working_dir 多 base 扫描"
```

---

### Task 6: 端点 — working_dir accept + 建任务校验

**Files:**
- Modify: `internal/server/http.go`（createSessionRequest @1167-1173、handleCreateSession @287-328、patchSessionRequest @1180-1185、handlePatchSession @388-437、handleCreateTask @671-729）
- Test: `internal/server/http_test.go`（追加）

**Interfaces:**
- Consumes: `domain.AgentSession.WorkingDir`、`domain.Task.WorkingDir`。
- Produces: POST/PATCH `/v1/sessions` 收 `working_dir`；建任务从 session 解析 working_dir 进 Task + 校验目录（400）。

- [ ] **Step 1: 写端点失败测试**

`internal/server/http_test.go` 追加：create session 带 working_dir → get 回显；patch working_dir 生效；建任务 working_dir 从 session 继承进 task；建任务 session.working_dir 非目录 → 400。仿现有 mode 测试（http_test.go:880/904/955）：

```go
func TestCreateTaskRejectsNonDirWorkingDir(t *testing.T) {
	// session 带一个不存在的 working_dir → 建任务应 400
	// （构造 session store 返回 WorkingDir=<不存在路径> 的 session；POST /v1/tasks 断言 400）
	// 具体骨架仿 http_test.go 现有 handleCreateTask 测试。
}
func TestCreateTaskInheritsSessionWorkingDir(t *testing.T) {
	// session.WorkingDir = t.TempDir()（真目录）→ 建任务 task.WorkingDir == 该目录
}
```

> 实现者按 http_test.go 现有 session/task 测试骨架补全（fake SessionStore + httptest）。

- [ ] **Step 2: 跑确认失败** → **Step 3: 实现**

- `createSessionRequest` 加 `WorkingDir string \`json:"working_dir"\``；handleCreateSession 存入 session（trim；空合法）。
- `patchSessionRequest` 加 `WorkingDir *string \`json:"working_dir"\``；handlePatchSession 写 session。
- handleCreateTask（仿 mode 解析 @691-724）：从 `loaded.WorkingDir` 写 `task.WorkingDir`；**校验**：`if wd := strings.TrimSpace(loaded.WorkingDir); wd != "" { info, err := os.Stat(wd); if err != nil || !info.IsDir() { writeError(w, 400, "working_dir not an existing directory"); return } }`。空值合法。

- [ ] **Step 4: 跑确认通过**

Run: `go build ./... && go test ./internal/server/`
Expected: PASS。

- [ ] **Step 5: gofmt + 提交**

```bash
gofmt -w internal/server/http.go internal/server/http_test.go
git add internal/server/http.go internal/server/http_test.go
git commit -m "feat(server): session working_dir 收存 + 建任务校验目录存在"
```

---

### Task 7: 沙箱根 — agentToolRoot(task) + 默认 agent per-task root

**Files:**
- Modify: `internal/runtime/agent_resolver.go`（agentToolRoot @150-155、ResolveTaskRunner @70/102）
- Modify: `internal/cli/command.go`（默认 agent tool registry @1794-1799）
- Test: `internal/runtime/agent_resolver_test.go`、`internal/cli/command_test.go`（追加/更新）

**Interfaces:**
- Consumes: `domain.Task.WorkingDir`。
- Produces: `agentToolRoot(rootCfg, agentCfg, task)` task.WorkingDir 优先；默认 agent 执行经 per-task working_dir 沙箱根。

- [ ] **Step 1: 写沙箱根失败测试**

```go
func TestAgentToolRootPrefersTaskWorkingDir(t *testing.T) {
	wd := t.TempDir()
	got := agentToolRoot(rootCfgWithContextRoot("/ctx"), agentCfgWithContextRoot(""), domain.Task{WorkingDir: wd})
	if got != wd {
		t.Fatalf("agentToolRoot = %q, want task.WorkingDir %q", got, wd)
	}
}
func TestAgentToolRootFallsBackWhenNoWorkingDir(t *testing.T) {
	got := agentToolRoot(rootCfgWithContextRoot("/ctx"), agentCfgWithContextRoot(""), domain.Task{})
	if got != "/ctx" {
		t.Fatalf("agentToolRoot = %q, want /ctx fallback", got)
	}
}
```

> helper 名按实际 config 结构对齐（实现者读 agent_resolver.go 现有 agentToolRoot 签名与 config 类型）。

- [ ] **Step 2-3: 实现**

- `agentToolRoot(rootCfg, agentCfg, task domain.Task) string`：`if wd := strings.TrimSpace(task.WorkingDir); wd != "" { return wd }`，否则现逻辑（agentCfg.ContextFiles.Root 否则 rootCfg.ContextFiles.Root）。调用点 ResolveTaskRunner（:102）传 task（已有 task 参 :70）。
- 默认 agent（command.go:1794）：现预建固定根 registry。改为让默认执行也按 task.WorkingDir 解析——最小方案：把默认 runtime 的 tool registry 构造从 serve 装配期一次性，改为 coordinator 派发默认任务时按 task.WorkingDir 重建（或走与 per-agent 相同的 per-task resolve 路径）。**注意保留 command.go:1795-1799 附加注册的 ledger/message/web/session_search/MoA 工具**（搬进重建路径，别丢）。实现者依 spec §4.4「让 coordinator 对默认执行也走 per-task resolve」定最小改动，并在报告说明所选方案。

- [ ] **Step 4: 跑确认通过**

Run: `go build ./... && go test ./internal/runtime/ ./internal/cli/`
Expected: PASS。

- [ ] **Step 5: gofmt + 提交**

```bash
gofmt -w internal/runtime/agent_resolver.go internal/runtime/agent_resolver_test.go internal/cli/command.go internal/cli/command_test.go
git add internal/runtime/agent_resolver.go internal/runtime/agent_resolver_test.go internal/cli/command.go internal/cli/command_test.go
git commit -m "feat(runtime): 工具沙箱根优先 task.WorkingDir，默认 agent 走 per-task root"
```

---

### Task 8: 删会话连带删会话目录

**Files:**
- Modify: `internal/server/http.go`（handleDeleteSession @443-464）
- Test: `internal/server/http_test.go`（追加）

**Interfaces:**
- Consumes: `SessionBase`、`SessionDir`、被删 session 的 WorkingDir。
- Produces: DELETE `/v1/sessions/{id}` 删 DB 行后连带删 `SessionDir(SessionBase(workspaceRoot, session.WorkingDir), id)` 目录。

- [ ] **Step 1: 写失败测试**

```go
func TestDeleteSessionRemovesSessionDir(t *testing.T) {
	// 建 session（带 working_dir=t.TempDir()）→ 在其会话目录写个文件 →
	// DELETE /v1/sessions/{id} → 断言会话目录被删除
}
```

> handleDeleteSession 需能拿到 workspaceRoot + 被删 session 的 WorkingDir。若 handler 当前无 workspaceRoot 引用，经 HTTPServer 字段注入（serve 装配传入）——实现者定注入点（仿 Task 5 base 解析）。删目录失败记 Warn 不阻断删 DB？**否**：spec §4.0 要求连带删；删目录 error 应 fail-loud 返回（或至少 logger.Error + 不谎报成功）。实现者按 fail-loud 铁律定：DB 删成功但目录删失败 → 返回 500 并记录（不静默）。

- [ ] **Step 2-4: 实现 + 测试通过**

`handleDeleteSession`：删 DB 行成功后 `dir := sessionstate.SessionDir(sessionstate.SessionBase(s.workspaceRoot, sess.WorkingDir), id); os.RemoveAll(dir)`；err 非 nil → logger.Error + writeError(500)。需先 Get session 拿 WorkingDir（在删 DB 前）。

Run: `go build ./... && go test ./internal/server/`
Expected: PASS。

- [ ] **Step 5: gofmt + 提交**

```bash
gofmt -w internal/server/http.go internal/server/http_test.go
git add internal/server/http.go internal/server/http_test.go
git commit -m "feat(server): 删会话连带删会话目录"
```

---

### Task 9: 全链路整合 + 门禁

**Files:**
- Modify: `internal/cli/command_test.go`（working_dir e2e）

- [ ] **Step 1: 写 working_dir 沙箱 e2e**

起 serve → 建带 working_dir 的 session → 建任务（工具写文件）→ 断言文件写在 working_dir 内、越界路径被拒。若过重则以 runtime 层集成测试覆盖沙箱根 = task.WorkingDir（工具 write_file 落 working_dir、`../` 越界 ErrPathOutsideWorkspace）。

- [ ] **Step 2: e2e 通过**

Run: `go test ./internal/cli/ -run TestServeCommand.*WorkingDir -v`

- [ ] **Step 3: 全量门禁（Windows）**

Run: `go build ./... ; go vet ./... ; go test ./...`
Expected: 全 PASS（若 FAIL 先诊断修复）。

- [ ] **Step 4: gofmt 检查触碰文件**

Run: `gofmt -l internal/domain/types.go internal/sessionstate/ internal/approval/ internal/manualgate/ internal/storage/sqlite.go internal/server/http.go internal/runtime/agent_resolver.go internal/runtime/checkpoint.go internal/runtime/runtime.go internal/cli/command.go`
Expected: 空（误报既有 CRLF 文件除外）。

- [ ] **Step 5: WSL race 门禁**

Run: 见顶部 WSL race recipe。
Expected: 全 PASS 无 race（Task 3/4/5 持久化并发重点）。

- [ ] **Step 6: 提交**

```bash
gofmt -w internal/cli/command_test.go
git add internal/cli/command_test.go
git commit -m "test(serve): working_dir 沙箱 e2e + 门禁"
```

---

## Self-Review

**Spec coverage（§4.4 / §4.0 / §6 / §8 / §9）:**
- session 加 working_dir + 任务解析进 Task → Task 1/2/6 ✅
- 建任务校验目录存在（非目录 400）→ Task 6 ✅
- agentToolRoot 优先 task.WorkingDir + WorkspacePathGuard 锁内 → Task 7 ✅
- 覆盖默认 agent per-task root → Task 7 ✅
- 会话目录随 working_dir（`<working_dir>/.stardust`）单一 resolver → Task 1/3/4 ✅
- 删会话连带删目录 → Task 8 ✅
- 跨重启恢复（多 base）→ Task 5 ✅
- 测试：working_dir 校验/沙箱越界/目录位置规则/删目录/继承 → 各任务 + Task 9 ✅

**风险提示（给执行者）:** Task 3/4/5 是 per-session base 的高危核心（牵动 M2b 挂起/恢复），用 opus reviewer；WSL race 必跑。Store `base`→`workspaceRoot` 重命名要全文件一致。Get/Load/Delete/Decide 加 workingDir 参会波及既有调用点（多传 `""`），逐一更新编译。默认 agent per-task root（Task 7）勿丢 ledger/message/web 等附加工具注册。

**Placeholder scan:** 部分测试给的是骨架注释（Task 6/8/9 的集成测试），因依赖既有 http_test/command_test 骨架与 fake——标注「按现有骨架补全，勿臆造 fake 签名」，非占位；crux 任务（1/3/4/5）给了完整测试代码。

**Type consistency:** `SessionBase(workspaceRoot, workingDir)` 签名贯穿 Task 1/3/4/5/8；`WorkingDir` 字段名统一（Session/Task/Checkpoint/ToolApproval）；Store 构造保持 `NewStore(workspaceRoot)`/`NewToolGateStore(workspaceRoot)`；`ListSuspendedIn(base)`/`ListPendingIn(base)` 命名对称。
