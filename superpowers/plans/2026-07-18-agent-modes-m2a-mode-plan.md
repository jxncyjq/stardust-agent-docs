---
title: Milestone 2a 实现计划 — 会话模式流转 + 工具敏感位 + Plan 模式
type: plan
status: ready
created: 2026-07-18
scope: legion/legionAgent（后端）
related:
  - "[[2026-07-15-agent-working-modes-design]]"
  - "[[2026-07-15-session-handoff-agent-modes]]"
  - "[[2026-07-17-agent-modes-m1b-checkpoint-resume]]"
tags: [agent, runtime, mode, plan, sensitive, milestone-2a]
---

# Milestone 2a 实现计划 — 会话模式 + 工具敏感位 + Plan 模式

> **给智能体执行者：** 必用子技能：用 superpowers:subagent-driven-development（推荐）或 superpowers:executing-plans 按任务逐个实现本计划。步骤用复选框（`- [ ]`）跟踪。

**目标：** 引入 per-session 的 `mode`（manual|plan|auto，默认 auto），让它流转进每个任务；把每个工具分类为敏感/安全；实现 **Plan 模式** —— 一次只提供安全（只读）工具 + 「产出计划、不要执行」的提示的运行，并把结果作为 OKF markdown 计划写进会话目录。Auto 模式行为不变；Manual 拦截属 M2b。

**架构：** `mode` 存在 `domain.AgentSession`（SQLite 列，schema v5），在 `POST`/`PATCH /v1/sessions` 处接收+校验，任务创建时从所属 session 解析进 `domain.Task.Mode`（一次性任务默认 auto）。工具的 `Descriptor` 加 `Sensitive bool`；`Registry.SafeToolNames()` 列出非敏感工具。Plan 模式**在 `RunTask` 内**按 `task.Mode` 处理 —— 因此对默认运行时和每个 resolver 建的 per-agent 运行时**统一生效**，无需碰它们的构造：当 `task.Mode==plan` 时，该次运行只提供/派发 `r.tools.Subset(safeNames...)`，在提示前置一条 plan 指令，完成时经 M1b 的 `sessionstate.Store` 把结果作为 OKF markdown 写到 `<sessionDir>/plans/<ts>.md`。

**技术栈：** Go 1.26，标准库，既有 `internal/domain`、`internal/tool`、`internal/storage`（modernc.org/sqlite）、`internal/server`、`internal/runtime`、`internal/sessionstate`（M1b）。

## 全局约束

- **Fail-Loud 铁律**（legionAgent/CLAUDE.md §0）：非法 mode 值 → 拒绝（HTTP 返 400，domain normalize 返错）；`session_id` 存在但 session 缺失，在现有 createTask 流里已是客户端错误 —— 保持之。禁止 zero-value 兜底、禁止 `_ = err`、禁止静默跳过。唯一契约可选情形：空/缺失 mode = 文档化默认 `auto`。
- **mode 枚举**（精确值，逐字）：`"manual"`、`"plan"`、`"auto"`。默认 `"auto"`。其余任何值非法。
- **Go 1.26**；GUI 是独立 module，不得 import server `internal/**`。
- **`-race` 只能在 WSL 跑**（Windows 宿主无 gcc）。触碰 `internal/runtime`/`internal/storage` 的任务用：
  ```bash
  wsl -d Ubuntu-22.04 -- bash -lc 'cd /mnt/f/source/stardust/Legion/legion/legionAgent && GOOS=linux GOARCH=amd64 CGO_ENABLED=1 CC=gcc $HOME/sdk/go/bin/go test -race ./internal/runtime/ ./internal/storage/ ./internal/server/ ./internal/tool/'
  ```
  Windows 宿主：普通 `go test ./...`。
- **完成门禁**：`go build ./... && go vet ./... && go test ./...` 全绿；`gofmt -l` 对触碰文件为空（仓库用 `core.autocrlf`；靠剥 `\r` 验证内容 LF 干净 —— 别整文件重排既有文件）。
- **Auto 零行为变更**：`mode==auto`（或未设）的任务必须与今天完全一样跑。每个既有测试须保持绿。
- 公开 API 要 Go doc 注释（以标识符名开头）。
- **仓库：** 全在 `legion/legionAgent/`（git toplevel 在此，remote `jxncyjq-stardust-agent-server`）。先从 `master` 开分支（`feat/agent-modes-m2a`）。未经用户授权别 push。
- **既有欠账（别碰）：** `func max`（runtime.go）、`errStaticResolver`（coordinator_test.go）、仓库范围 CRLF gofmt 标记 —— 范围外。

---

## 文件结构

**修改：**
- `internal/domain/types.go` —— `SessionMode` 类型 + 常量 + `NormalizeMode`；`AgentSession.Mode`；`Task.Mode`。
- `internal/tool/registry.go` —— `Descriptor.Sensitive`；`Registry.SafeToolNames()`。
- `internal/tool/builtin.go` + ledger/message/web 工具注册处 —— 逐工具设 `Sensitive`。
- `internal/storage/sqlite.go` —— schema v5 迁移（`agent_sessions.mode`）；INSERT/SELECT/scan 带 mode。
- `internal/server/http.go` —— `createSessionRequest.Mode`、`patchSessionRequest.Mode`、校验、任务 mode 解析。
- `internal/runtime/runtime.go` —— `RunTask` 内 Plan 模式分支（安全工具子集 + plan 提示 + OKF 产出）；复用 `Config.Checkpoints`。
- `internal/sessionstate/checkpoint.go`（或新建 `plans.go`）—— `Store.WritePlan`。

**新增测试**：各修改包旁。

---

## 任务 1：mode 领域类型 + Session/Task 字段

**文件：**
- 修改：`internal/domain/types.go`
- 测试：`internal/domain/mode_test.go`（新建）

**接口：**
- 产出：
  - `type SessionMode = string`（别名，让 JSON/DB 保持裸 string），常量 `ModeManual = "manual"`、`ModePlan = "plan"`、`ModeAuto = "auto"`。
  - `func NormalizeMode(raw string) (mode string, ok bool)` —— trim；空 → (`"auto"`, true)；`manual|plan|auto` → (值, true)；其余 → (`""`, false)。
  - `AgentSession.Mode string`（json `mode`）；`Task.Mode string`（json `mode`）。

- [ ] **步骤 1：写失败测试**

新建 `internal/domain/mode_test.go`：

```go
package domain

import "testing"

func TestNormalizeMode(t *testing.T) {
	cases := []struct {
		in     string
		want   string
		wantOK bool
	}{
		{"", ModeAuto, true},
		{"  ", ModeAuto, true},
		{"auto", ModeAuto, true},
		{"manual", ModeManual, true},
		{"plan", ModePlan, true},
		{" manual ", ModeManual, true},
		{"MANUAL", "", false}, // 大小写敏感：拒绝，不静默转换
		{"bogus", "", false},
	}
	for _, c := range cases {
		got, ok := NormalizeMode(c.in)
		if got != c.want || ok != c.wantOK {
			t.Errorf("NormalizeMode(%q) = (%q,%v), want (%q,%v)", c.in, got, ok, c.want, c.wantOK)
		}
	}
}
```

- [ ] **步骤 2：运行确认失败**

运行：`go test ./internal/domain/ -run TestNormalizeMode`
预期：FAIL —— `undefined: NormalizeMode`、`undefined: ModeAuto`。

- [ ] **步骤 3：在 `internal/domain/types.go` 实现**

在顶部附近（`TaskStatus` 常量之后）加：

```go
// 会话/任务工作模式。Manual 把有副作用工具挡在人工审批后；Plan 只提供只读工具、
// 产出计划而无副作用；Auto 是默认的不受限行为。以裸 string 存在 Session/Task 上，
// 便于 JSON/DB 平凡往返。
const (
	ModeManual = "manual"
	ModePlan   = "plan"
	ModeAuto   = "auto"
)

// NormalizeMode 校验并规范化一个原始 mode 字符串。空/空白值是合法默认（auto）。
// 已识别值原样返回。其余任何值被拒绝（ok=false），使调用方 fail-loud 而非把未知
// mode 静默转成 auto。
func NormalizeMode(raw string) (mode string, ok bool) {
	switch strings.TrimSpace(raw) {
	case "":
		return ModeAuto, true
	case ModeManual:
		return ModeManual, true
	case ModePlan:
		return ModePlan, true
	case ModeAuto:
		return ModeAuto, true
	default:
		return "", false
	}
}
```

（`strings` 已在 types.go import。）

给 `AgentSession` 加 `Mode`（在 `Title` 之后）：

```go
	Title     string    `json:"title"`
	Mode      string    `json:"mode"`
	Archived  bool      `json:"archived"`
```

给 `Task` 加 `Mode`（在 `SessionID` 之后）：

```go
	SessionID     string     `json:"session_id"`
	Mode          string     `json:"mode,omitempty"`
	Status        TaskStatus `json:"status"`
```

- [ ] **步骤 4：运行确认通过**

运行：`go test ./internal/domain/`
预期：PASS。

- [ ] **步骤 5：验证构建**

运行：`go build ./...`
预期：OK（新字段是追加的；JSON tag 不破坏既有解码器）。

- [ ] **步骤 6：提交**

```bash
git add internal/domain/types.go internal/domain/mode_test.go
git commit -m "feat(domain): SessionMode consts + NormalizeMode + Session/Task Mode fields"
```

---

## 任务 2：工具敏感性分类

**文件：**
- 修改：`internal/tool/registry.go`（Descriptor 结构体；加 `SafeToolNames`）
- 修改：`internal/tool/builtin.go`（内建工具设 `Sensitive`）
- 修改：ledger/message/web 工具注册文件（设 `Sensitive`）—— 用 `grep -rn "RegisterDescriptor" internal/tool/` 定位
- 测试：`internal/tool/sensitive_test.go`（新建）

**接口：**
- 消费：既有 `Descriptor`、`RegisterDescriptor`、`Registry`。
- 产出：
  - `Descriptor.Sensitive bool`（json `sensitive,omitempty`）。
  - `func (r *Registry) SafeToolNames() []string` —— 已注册非敏感工具的排序名（排除 meta 工具 list_tools/call_tool，若存在）。

**分类（依 spec §4.3）：** 安全 = `read_file`、`search_content`、`list_files`，加任何只读 ledger/session 工具（如 `get_task`、`list_tasks`、`session_search` —— 在注册处确认真实名）。敏感 = `write_file`、`send_message`、`fetch_url`、`delegate_task`，及 ledger 写（`create_task`/`claim_task`/`update_task`/… —— 确认真实名）。存疑时标**敏感**（fail-safe：误标安全的工具会绕过 M2b 的 gate；误标敏感的工具只是多问一次审批）。

- [ ] **步骤 1：写失败测试**

新建 `internal/tool/sensitive_test.go`：

```go
package tool

import (
	"context"
	"testing"

	"github.com/stardust/legion-agent/internal/domain"
)

func TestSafeToolNamesExcludesSensitive(t *testing.T) {
	reg := NewRegistry(
		NewExecutionPolicy(ExecutionPolicyConfig{AutoAllowTools: []string{"reader", "writer"}}),
		PermissionEnforcerFunc(func(domain.Agent, domain.ToolCall) error { return nil }),
		NoopGuardrails{},
	)
	reg.RegisterDescriptor(Descriptor{Name: "reader", Sensitive: false}, HandlerFunc(func(context.Context, domain.ToolCall) (domain.ToolResult, error) {
		return domain.ToolResult{Success: true}, nil
	}))
	reg.RegisterDescriptor(Descriptor{Name: "writer", Sensitive: true}, HandlerFunc(func(context.Context, domain.ToolCall) (domain.ToolResult, error) {
		return domain.ToolResult{Success: true}, nil
	}))

	safe := reg.SafeToolNames()
	if len(safe) != 1 || safe[0] != "reader" {
		t.Fatalf("SafeToolNames = %v, want [reader]", safe)
	}
}
```

- [ ] **步骤 2：运行确认失败**

运行：`go test ./internal/tool/ -run TestSafeToolNames`
预期：FAIL —— `unknown field 'Sensitive'` 与 `undefined: (*Registry).SafeToolNames`。

- [ ] **步骤 3：在 `internal/tool/registry.go` 加字段 + helper**

给 `Descriptor` 结构体加 `Sensitive`（定位结构体；加在 `RiskLevel` 之后）：

```go
	// Sensitive 标记一个工具为有副作用：Manual 模式下对它的调用被挡在人工审批后
	// （M2b），Plan 模式把它排除出所提供的工具集。只读工具（read/search/list）非敏感。
	Sensitive bool `json:"sensitive,omitempty"`
```

加 helper（在 `Descriptors` 附近）：

```go
// SafeToolNames 返回已注册工具中 NOT 敏感（且非 lazy 协议 meta 工具）的排序名。
// Plan 模式恰好提供这个集合，使规划运行无法触及有副作用工具。
func (r *Registry) SafeToolNames() []string {
	names := make([]string, 0, len(r.describes))
	for name, descriptor := range r.describes {
		if descriptor.Sensitive {
			continue
		}
		if name == "list_tools" || name == "call_tool" {
			continue
		}
		names = append(names, name)
	}
	sort.Strings(names)
	return names
}
```

确认 registry.go 已 import `sort`。

- [ ] **步骤 4：运行 helper 测试确认通过**

运行：`go test ./internal/tool/ -run TestSafeToolNames`
预期：PASS。

- [ ] **步骤 5：分类内建工具**

在 `internal/tool/builtin.go`，给 `write_file` descriptor 设 `Sensitive: true`；`read_file`/`search_content`/`list_files` 保持默认（false）。然后找每一个 `RegisterDescriptor`/注册处：

运行：`grep -rn "RegisterDescriptor\|Descriptor{" internal/tool/`

对每个，给有副作用工具（`send_message`、`fetch_url`、`delegate_task`，及每个 ledger 写工具如 create/claim/update task）设 `Sensitive: true`，只读工具（get/list/search）保持 false。在每个 `Sensitive: true` 加一行注释说明原因（如 `// Sensitive: writes to the task ledger`）。别改任何工具行为 —— 只加这个标志。

- [ ] **步骤 6：加分类回归测试**

在 `internal/tool/sensitive_test.go` 追加一个用真实 workspace registry、断言已知分类的测试，使未来新增工具不能悄悄漏分类而上线：

```go
func TestBuiltinWorkspaceToolsClassification(t *testing.T) {
	reg := NewReadOnlyWorkspaceRegistry(t.TempDir(), nil)
	want := map[string]bool{ // name -> sensitive
		"read_file":      false,
		"search_content": false,
		"list_files":     false,
		"write_file":     true,
	}
	got := map[string]bool{}
	for _, d := range reg.Descriptors() {
		got[d.Name] = d.Sensitive
	}
	for name, sensitive := range want {
		g, ok := got[name]
		if !ok {
			t.Errorf("tool %q not registered by NewReadOnlyWorkspaceRegistry", name)
			continue
		}
		if g != sensitive {
			t.Errorf("tool %q Sensitive = %v, want %v", name, g, sensitive)
		}
	}
}
```

> 对照真实的 `NewReadOnlyWorkspaceRegistry`（`internal/tool/builtin.go`）确认它注册了哪些工具 —— 它也注册 `write_file`（builtin.go:86）。若它 NOT 注册 `write_file`，删掉那行、改断言它实际注册的工具；要点是分类被断言，而非精确的 registry。把 `want` 调成该 registry 实际注册的工具。

- [ ] **步骤 7：跑全 tool 套件 + 构建**

运行：`go test ./internal/tool/ && go build ./... && go vet ./...`
预期：PASS（既有 tool 测试不受影响 —— `Sensitive` 默认 false，故行为不变）。

- [ ] **步骤 8：提交**

```bash
git add internal/tool/
git commit -m "feat(tool): Sensitive descriptor flag + SafeToolNames + built-in classification"
```

---

## 任务 3：SQLite schema v5 —— 持久化会话 mode

**文件：**
- 修改：`internal/storage/sqlite.go`（`CurrentSchemaVersion`、`migrate`、`SaveAgentSession`、`GetAgentSession`、`ListAgentSessions`、最新 session 查询、`scanAgentSession`）
- 测试：`internal/storage/sqlite_test.go`（追加）

**接口：**
- 消费：`domain.AgentSession.Mode`（任务 1）。
- 产出：`agent_sessions.mode` 列（`TEXT NOT NULL DEFAULT 'auto'`）；Save/Get/List 往返 `Mode`。

- [ ] **步骤 1：写失败测试**

追加到 `internal/storage/sqlite_test.go`（沿用文件既有测试库 setup helper —— 找其他测试怎么开 repo，如 `OpenSQLite(ctx, ":memory:")` 或临时文件）：

```go
func TestAgentSessionModeRoundTrips(t *testing.T) {
	ctx := context.Background()
	repo := newTestRepo(t) // 复用本文件既有测试 helper
	sess := domain.AgentSession{
		ID:        "sess-mode-1",
		CompanyID: "c1",
		AgentID:   "a1",
		Mode:      domain.ModeManual,
		CreatedAt: time.Now(),
		UpdatedAt: time.Now(),
	}
	if err := repo.SaveAgentSession(ctx, sess); err != nil {
		t.Fatalf("SaveAgentSession: %v", err)
	}
	got, ok, err := repo.GetAgentSession(ctx, "sess-mode-1")
	if err != nil || !ok {
		t.Fatalf("GetAgentSession ok=%v err=%v", ok, err)
	}
	if got.Mode != domain.ModeManual {
		t.Errorf("Mode = %q, want %q", got.Mode, domain.ModeManual)
	}
}

func TestAgentSessionModeDefaultsAutoForLegacyRows(t *testing.T) {
	ctx := context.Background()
	repo := newTestRepo(t)
	// 未显式设 mode 保存的 session 必须读回 auto（列默认），绝不为空串。
	sess := domain.AgentSession{ID: "sess-legacy", CompanyID: "c1", AgentID: "a1", CreatedAt: time.Now(), UpdatedAt: time.Now()}
	if err := repo.SaveAgentSession(ctx, sess); err != nil {
		t.Fatalf("save: %v", err)
	}
	got, _, err := repo.GetAgentSession(ctx, "sess-legacy")
	if err != nil {
		t.Fatalf("get: %v", err)
	}
	if got.Mode != domain.ModeAuto {
		t.Errorf("legacy Mode = %q, want %q", got.Mode, domain.ModeAuto)
	}
}
```

> 若 `SaveAgentSession` 逐字写 `session.Mode`，`TestAgentSessionModeDefaultsAutoForLegacyRows` 保存的是 `Mode==""`。定契约：`SaveAgentSession` 应在写前把空 mode 规范化为 `auto`（使 DB 从不存 ""），或列默认只对迁移插入的行生效。让 `SaveAgentSession` 在 `session.Mode==""` 时写 `auto`（见步骤 3），使新行和旧行一致为 `auto`。

- [ ] **步骤 2：运行确认失败**

运行：`go test ./internal/storage/ -run TestAgentSessionMode`
预期：FAIL（mode 未持久化；读回 ""）。

- [ ] **步骤 3：实现迁移 + 列**

在 `internal/storage/sqlite.go`：

1. 把 `const CurrentSchemaVersion = 5`。
2. 在 `migrate`（找那个加了 v3 `archived` 的分版本迁移 switch/if 阶梯），加 v5 步：
   ```go
   // Version 5 adds agent_sessions.mode (manual|plan|auto) for per-session
   // working mode. Existing rows default to auto.
   if from < 5 {
       if _, err := tx.ExecContext(ctx, `ALTER TABLE agent_sessions ADD COLUMN mode TEXT NOT NULL DEFAULT 'auto'`); err != nil {
           return fmt.Errorf("migrate v5 add agent_sessions.mode: %w", err)
       }
   }
   ```
   （对齐既有迁移步的 EXACT 形态 —— 同样的 `tx`/`from` 变量名、同样的错误包装风格。先读 v3/v4 步再照做。）
3. 更新 `SaveAgentSession` INSERT 包含 `mode`（现 9 列），并让 `ON CONFLICT DO UPDATE SET` 设 `mode`。空 → auto 规范化：
   ```go
   mode := session.Mode
   if mode == "" {
       mode = domain.ModeAuto
   }
   _, err := r.db.ExecContext(ctx, `
       INSERT INTO agent_sessions (
           id, company_id, agent_id, project, title, mode, archived, created_at, updated_at
       ) VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?)
       ON CONFLICT(id) DO UPDATE SET
           company_id=excluded.company_id, agent_id=excluded.agent_id,
           project=excluded.project, title=excluded.title, mode=excluded.mode,
           archived=excluded.archived, updated_at=excluded.updated_at
   `, session.ID, session.CompanyID, session.AgentID, session.Project, session.Title, mode, session.Archived, formatTime(session.CreatedAt), formatTime(session.UpdatedAt))
   ```
   （对齐真实列表 + 既有 `formatTime`/参数风格 —— 先读当前 INSERT；ON CONFLICT set 列表须镜像既有的再加 `mode`。）
4. 给每个 `SELECT ... FROM agent_sessions` 列表加 `mode`（GetAgentSession ~289、ListAgentSessions ~306、最新 session 查询 ~269）—— 在与 INSERT 同位置插入 `mode`（`title` 之后、`archived` 之前）。
5. 更新 `scanAgentSession` 把新 `mode` 列 scan 进 `session.Mode`（在与 SELECT 对应的正确位置加 `&session.Mode`）。

- [ ] **步骤 4：运行确认通过**

运行：`go test ./internal/storage/`
预期：PASS（新测试 + 所有既有 storage 测试；既有无 mode 构造的 session 现读回 `auto`）。

- [ ] **步骤 5：构建 + vet + race**

运行（Windows）：`go build ./... && go vet ./...`
运行（WSL）：`-race` 命令（含 storage）。
预期：绿。

- [ ] **步骤 6：提交**

```bash
git add internal/storage/sqlite.go internal/storage/sqlite_test.go
git commit -m "feat(storage): schema v5 agent_sessions.mode column + round-trip"
```

---

## 任务 4：HTTP —— 接收/校验会话 mode + 解析任务 mode

**文件：**
- 修改：`internal/server/http.go`（`createSessionRequest`、`patchSessionRequest`、`handleCreateSession` ~287、`handlePatchSession` ~387、`handleCreateTask` ~654）
- 测试：`internal/server/http_test.go`（追加）

**接口：**
- 消费：`domain.NormalizeMode`（任务 1）、`domain.AgentSession.Mode`、`domain.Task.Mode`、`SessionStore.GetAgentSession`。
- 产出：`createSessionRequest.Mode string`；`patchSessionRequest.Mode *string`；任务继承其 session 的 mode。

- [ ] **步骤 1：写失败测试**

追加到 `internal/server/http_test.go`（复用文件既有 HTTP 测试骨架 —— 找它怎么用假的/sqlite `SessionStore` 和 `TaskStore` 建 `HTTPServer`）：

```go
func TestCreateSessionStoresValidMode(t *testing.T) {
	srv := newTestHTTPServer(t) // 既有骨架
	body := `{"company_id":"c1","agent_id":"a1","mode":"manual"}`
	rec := doRequest(t, srv, http.MethodPost, "/v1/sessions", body)
	if rec.Code != http.StatusCreated {
		t.Fatalf("status = %d, want 201; body=%s", rec.Code, rec.Body.String())
	}
	var got domain.AgentSession
	if err := json.Unmarshal(rec.Body.Bytes(), &got); err != nil {
		t.Fatalf("decode: %v", err)
	}
	if got.Mode != domain.ModeManual {
		t.Errorf("session Mode = %q, want manual", got.Mode)
	}
}

func TestCreateSessionRejectsInvalidMode(t *testing.T) {
	srv := newTestHTTPServer(t)
	rec := doRequest(t, srv, http.MethodPost, "/v1/sessions", `{"company_id":"c1","agent_id":"a1","mode":"bogus"}`)
	if rec.Code != http.StatusBadRequest {
		t.Fatalf("status = %d, want 400 for invalid mode", rec.Code)
	}
}

func TestCreateSessionDefaultsAutoWhenModeOmitted(t *testing.T) {
	srv := newTestHTTPServer(t)
	rec := doRequest(t, srv, http.MethodPost, "/v1/sessions", `{"company_id":"c1","agent_id":"a1"}`)
	var got domain.AgentSession
	_ = json.Unmarshal(rec.Body.Bytes(), &got)
	if got.Mode != domain.ModeAuto {
		t.Errorf("default Mode = %q, want auto", got.Mode)
	}
}

func TestPatchSessionUpdatesMode(t *testing.T) {
	srv := newTestHTTPServer(t)
	create := doRequest(t, srv, http.MethodPost, "/v1/sessions", `{"company_id":"c1","agent_id":"a1"}`)
	var s domain.AgentSession
	_ = json.Unmarshal(create.Body.Bytes(), &s)
	rec := doRequest(t, srv, http.MethodPatch, "/v1/sessions/"+s.ID, `{"mode":"plan"}`)
	if rec.Code != http.StatusOK {
		t.Fatalf("patch status = %d, want 200", rec.Code)
	}
	var got domain.AgentSession
	_ = json.Unmarshal(rec.Body.Bytes(), &got)
	if got.Mode != domain.ModePlan {
		t.Errorf("patched Mode = %q, want plan", got.Mode)
	}
}

func TestCreateTaskInheritsSessionMode(t *testing.T) {
	srv := newTestHTTPServer(t)
	create := doRequest(t, srv, http.MethodPost, "/v1/sessions", `{"company_id":"c1","agent_id":"a1","mode":"plan"}`)
	var s domain.AgentSession
	_ = json.Unmarshal(create.Body.Bytes(), &s)
	body := `{"id":"task-1","company_id":"c1","agent_id":"a1","session_id":"` + s.ID + `","input":"hi"}`
	rec := doRequest(t, srv, http.MethodPost, "/v1/tasks", body)
	if rec.Code != http.StatusCreated {
		t.Fatalf("task status = %d, want 201; body=%s", rec.Code, rec.Body.String())
	}
	var task domain.Task
	_ = json.Unmarshal(rec.Body.Bytes(), &task)
	if task.Mode != domain.ModePlan {
		t.Errorf("task Mode = %q, want plan (inherited from session)", task.Mode)
	}
}
```

> 把 `newTestHTTPServer`/`doRequest` 适配成 `http_test.go` 里真实 helper 名。若既有骨架用非 sqlite 的内存 `SessionStore`，确保那个假 store 往返 `Mode`（map 存储天然往返整个结构）。若骨架里任务不经真实 `SessionStore`，接一个进去让 `TestCreateTaskInheritsSessionMode` 能查到 session。

- [ ] **步骤 2：运行确认失败**

运行：`go test ./internal/server/ -run 'TestCreateSession|TestPatchSession|TestCreateTaskInherits'`
预期：FAIL —— mode 未被接收/校验/继承。

- [ ] **步骤 3：给请求结构体加 mode**

```go
type createSessionRequest struct {
	Project   string `json:"project"`
	CompanyID string `json:"company_id"`
	AgentID   string `json:"agent_id"`
	Title     string `json:"title"`
	Mode      string `json:"mode"`
}
```

```go
type patchSessionRequest struct {
	Title    *string `json:"title"`
	Project  *string `json:"project"`
	Archived *bool   `json:"archived"`
	Mode     *string `json:"mode"`
}
```

- [ ] **步骤 4：在 `handleCreateSession` 校验+存 mode**

解码 `req` 后、构建 session 前，规范化 mode（非法 fail-loud）：

```go
	mode, ok := domain.NormalizeMode(req.Mode)
	if !ok {
		writeError(w, http.StatusBadRequest, fmt.Sprintf("invalid mode %q (want manual|plan|auto)", req.Mode))
		return
	}
```

给 `domain.AgentSession{...}` literal 加 `Mode: mode,`。

- [ ] **步骤 5：在 `handlePatchSession` 校验+应用 mode**

在 `Archived` 块之后：

```go
	if req.Mode != nil {
		mode, ok := domain.NormalizeMode(*req.Mode)
		if !ok {
			writeError(w, http.StatusBadRequest, fmt.Sprintf("invalid mode %q (want manual|plan|auto)", *req.Mode))
			return
		}
		session.Mode = mode
	}
```

- [ ] **步骤 6：在 `handleCreateTask` 从 session 解析任务 mode**

在 `sessionID := strings.TrimSpace(req.SessionID)` 之后、构建 `task` 之前解析 mode。有 session 时任务继承其 mode；一次性任务（无 session）为 auto。「存在但缺失」的 session 在下面 `recordUserTurn` 已是客户端错误 —— 但我们要在构建任务前拿到 session 的 mode，故在这里查并在缺失时 fail-loud：

```go
	taskMode := domain.ModeAuto
	if sessionID != "" {
		if s.sessions == nil {
			writeError(w, http.StatusServiceUnavailable, "session store is unavailable")
			return
		}
		sess, ok, err := s.sessions.GetAgentSession(r.Context(), sessionID)
		if err != nil {
			writeError(w, http.StatusInternalServerError, fmt.Sprintf("load session for task mode: %v", err))
			return
		}
		if !ok {
			writeError(w, http.StatusBadRequest, fmt.Sprintf("session %q not found", sessionID))
			return
		}
		resolved, ok := domain.NormalizeMode(sess.Mode)
		if !ok {
			// 盘上非法 mode 是损坏状态，非客户端输入 —— fail-loud。
			writeError(w, http.StatusInternalServerError, fmt.Sprintf("session %q has invalid stored mode %q", sessionID, sess.Mode))
			return
		}
		taskMode = resolved
	}
```

给 `domain.Task{...}` literal 加 `Mode: taskMode,`。

> 注：这在任务创建时多一次 session 查询。紧接其后的 `recordUserTurn` 路径也隐式需要 session；合并到这一次查询即可。若 `recordUserTurn` 已加载 session，复用其结果而非二次查询 —— 先读 `recordUserTurn`，若廉价就把已加载的 session 传下去。若不廉价，多一次读可接受。

- [ ] **步骤 7：运行确认通过**

运行：`go test ./internal/server/`
预期：PASS（新 + 既有）。既有无 session 的任务创建测试仍得 `Mode==auto`。

- [ ] **步骤 8：构建 + vet + race + 提交**

运行：`go build ./... && go vet ./...`；WSL `-race`（含 server）。
```bash
git add internal/server/http.go internal/server/http_test.go
git commit -m "feat(server): accept+validate session mode; tasks inherit session mode"
```

---

## 任务 5：`RunTask` 内 Plan 模式 —— 安全工具子集 + plan 指令

**文件：**
- 修改：`internal/runtime/runtime.go`（`RunTask` 入口 / `inferenceTools` / `dispatchToolCall` 路径、及 `buildPrompt`）
- 测试：`internal/runtime/plan_mode_test.go`（新建）

**接口：**
- 消费：`domain.Task.Mode`（任务 1）、`Registry.SafeToolNames`/`Subset`（任务 2）、`domain.ModePlan`。
- 产出（内部）：当 `task.Mode==ModePlan`，该次运行有效工具注册表 = `r.tools.Subset(r.tools.SafeToolNames()...)`，提示带一条 plan 指令。Auto/Manual 运行不变。

**设计：** 在 `RunTask` 顶部按 `task.Mode` 计算一次有效注册表，贯穿该次运行。`Runtime.tools` 被 `inferenceTools`、`toolNames`、`dispatchToolCall` 使用 —— 引入 helper `effectiveTools(task) *tool.Registry`，正常返回 `r.tools`，`task.Mode==ModePlan` 时返回安全子集，并把选定注册表贯穿循环。为避免大改签名，把有效注册表作为 run-scoped 值放在传给循环的结构里。**最小改动：** 在 RunTask 入口解析注册表并传进 `runToolLoop`/`executeToolCalls`/`inferenceTools`（经 run-scoped 字段）。实现时确认精确管线；改动局部化，逐字保留 nil/auto 路径。

- [ ] **步骤 1：写失败测试**

新建 `internal/runtime/plan_mode_test.go`：

```go
package runtime

import (
	"context"
	"strings"
	"testing"

	"github.com/stardust/legion-agent/internal/adapter"
	"github.com/stardust/legion-agent/internal/domain"
	"github.com/stardust/legion-agent/internal/port"
	"github.com/stardust/legion-agent/internal/tool"
)

// planProbeMaas 记录第一次调用时提供给模型的工具名，然后以纯文本作答。
type planProbeMaas struct {
	offeredTools []string
	calls        int
}

func (m *planProbeMaas) Generate(ctx context.Context, req port.InferenceRequest) (port.InferenceResponse, error) {
	if err := ctx.Err(); err != nil {
		return port.InferenceResponse{}, err
	}
	m.calls++
	if m.calls == 1 {
		for _, tl := range req.Tools {
			m.offeredTools = append(m.offeredTools, tl.Name)
		}
	}
	return port.InferenceResponse{Text: "plan text"}, nil
}

// planRegistry 有一个安全工具和一个敏感工具。
func planRegistry(t *testing.T) *tool.Registry {
	t.Helper()
	reg := tool.NewRegistry(
		tool.NewExecutionPolicy(tool.ExecutionPolicyConfig{AutoAllowTools: []string{"read_x", "write_x"}}),
		tool.PermissionEnforcerFunc(func(domain.Agent, domain.ToolCall) error { return nil }),
		tool.NoopGuardrails{},
	)
	reg.RegisterDescriptor(tool.Descriptor{Name: "read_x", Sensitive: false}, tool.HandlerFunc(func(context.Context, domain.ToolCall) (domain.ToolResult, error) {
		return domain.ToolResult{Success: true}, nil
	}))
	reg.RegisterDescriptor(tool.Descriptor{Name: "write_x", Sensitive: true}, tool.HandlerFunc(func(context.Context, domain.ToolCall) (domain.ToolResult, error) {
		return domain.ToolResult{Success: true}, nil
	}))
	return reg
}

func TestPlanModeOffersOnlySafeToolsAndInstruction(t *testing.T) {
	maas := &planProbeMaas{}
	runner := NewRuntime(Config{
		Maas:      maas,
		Audit:     adapter.NewMemoryAuditLog(),
		Events:    adapter.NewMemoryEventBus(),
		Tools:     planRegistry(t),
		LazyTools: false, // eager：所提供工具 == 有效注册表的工具
	})
	_, err := runner.RunTask(context.Background(), domain.Agent{ID: "a1"}, domain.Task{
		ID: "t1", AgentID: "a1", Status: domain.TaskRunning, Input: "plan it", Mode: domain.ModePlan,
	})
	if err != nil {
		t.Fatalf("RunTask: %v", err)
	}
	for _, name := range maas.offeredTools {
		if name == "write_x" {
			t.Errorf("plan mode offered sensitive tool write_x; offered=%v", maas.offeredTools)
		}
	}
	foundSafe := false
	for _, name := range maas.offeredTools {
		if name == "read_x" {
			foundSafe = true
		}
	}
	if !foundSafe {
		t.Errorf("plan mode did not offer safe tool read_x; offered=%v", maas.offeredTools)
	}
}

func TestAutoModeOffersAllTools(t *testing.T) {
	maas := &planProbeMaas{}
	runner := NewRuntime(Config{
		Maas: maas, Audit: adapter.NewMemoryAuditLog(), Events: adapter.NewMemoryEventBus(),
		Tools: planRegistry(t), LazyTools: false,
	})
	_, err := runner.RunTask(context.Background(), domain.Agent{ID: "a1"}, domain.Task{
		ID: "t2", AgentID: "a1", Status: domain.TaskRunning, Input: "do it", Mode: domain.ModeAuto,
	})
	if err != nil {
		t.Fatalf("RunTask: %v", err)
	}
	has := func(n string) bool {
		for _, x := range maas.offeredTools {
			if x == n {
				return true
			}
		}
		return false
	}
	if !has("write_x") || !has("read_x") {
		t.Errorf("auto mode should offer all tools; offered=%v", maas.offeredTools)
	}
}
```

- [ ] **步骤 2：运行确认失败**

运行：`go test ./internal/runtime/ -run 'TestPlanModeOffers|TestAutoModeOffers'`
预期：FAIL —— plan 模式当前提供所有工具（含 write_x），因为 mode 被忽略。

- [ ] **步骤 3：实现 plan 模式有效工具 + 指令**

在 `internal/runtime/runtime.go`：

1. 加 helper：
   ```go
   // effectiveTools 返回一次运行应使用的工具注册表。Plan 模式下只暴露非敏感（只读）
   // 工具，使规划运行可调研但不产生副作用。其余任何模式使用完整注册表。
   func (r *Runtime) effectiveTools(task domain.Task) *tool.Registry {
       if r.tools != nil && task.Mode == domain.ModePlan {
           return r.tools.Subset(r.tools.SafeToolNames()...)
       }
       return r.tools
   }
   ```
   若 runtime.go 未 import `internal/tool` 则加（已 import —— 用到 `*tool.Registry`）。

2. 把有效注册表贯穿该次运行。`inferenceTools`、`toolNames`、`dispatchToolCall` 当前读 `r.tools`。最小安全改动：在 `RunTask` 顶部解析 `tools := r.effectiveTools(task)` 并传进循环，让 `inferenceTools`/`toolNames`/`dispatchToolCall` 以注册表为参数（或经 run-scoped 结构携带）。选让 diff 最小者，务必保证**所提供 schema（`inferenceTools`）与派发（`dispatchToolCall`/`executeToolCalls`）用 SAME 有效注册表** —— 提供安全工具却对完整注册表派发是 bug。逐字保留 Auto 路径（effectiveTools 返回 `r.tools`）。

3. `task.Mode==ModePlan` 时给提示前置 plan 指令。在 `buildPrompt`（或 `RunTask` 里紧随其后），plan 模式时追加：
   ```go
   const planInstruction = "\n\n[系统] 当前为 Plan 模式：只做调研与分析，产出一份结构化的执行计划（步骤、涉及文件、验证方式），不要执行任何有副作用的操作。只可使用只读工具。"
   ```
   把它加进 base prompt，使其成为稳定前缀的一部分。

> 管线局部化。若传注册表参数触碰很多签名，改为加一个 unexported run-scoped 字段（如每次 RunTask 解析并放在与 `loopState` 并行传递的 per-call 结构上的 `*tool.Registry`）。**别 mutate `r.tools`**（Runtime 被并发任务共享 —— mutate 它是数据竞争）。有效注册表必须是 per-call 局部。

- [ ] **步骤 4：运行确认通过**

运行：`go test ./internal/runtime/`
预期：PASS —— plan 模式只提供 `read_x`；auto 提供两者；所有既有 runtime 测试仍绿（它们无 Mode → auto → `r.tools` 不变）。

- [ ] **步骤 5：竞态检查（并发任务、混合模式）**

运行（WSL）：`-race` 命令。并发压测 + 这些 plan 测试须无数据竞争 —— 佐证有效注册表是 per-call 局部、非共享 mutation。

- [ ] **步骤 6：提交**

```bash
git add internal/runtime/runtime.go internal/runtime/plan_mode_test.go
git commit -m "feat(runtime): Plan mode offers only safe tools + plan instruction (per-call, race-safe)"
```

---

## 任务 6：Plan 产出 —— 写 OKF markdown 计划到会话目录

**文件：**
- 修改：`internal/sessionstate/checkpoint.go`（或新建 `internal/sessionstate/plans.go`）—— 加 `Store.WritePlan`。
- 修改：`internal/runtime/runtime.go`（`finishRun`）—— plan 模式时把结果写成 OKF 计划。
- 测试：`internal/sessionstate/plans_test.go`（新建）+ `internal/runtime/plan_mode_test.go`（追加）

**接口：**
- 消费：`sessionstate.Store`（M1b）、`SessionDir`。
- 产出：`func (s *Store) WritePlan(sessionKey, filename, content string) (path string, err error)` —— 写到 `<base>/session/<sessionKey>/plans/<filename>`，建目录；返回路径。
- Runtime：Plan 模式完成的运行，`finishRun` 把 `run.Result` 框成 OKF markdown 写到 `plans/plan-<unixnano>.md`。

- [ ] **步骤 1：写失败的 store 测试**

新建 `internal/sessionstate/plans_test.go`：

```go
package sessionstate

import (
	"os"
	"path/filepath"
	"strings"
	"testing"
)

func TestWritePlanCreatesFileUnderPlansDir(t *testing.T) {
	base := t.TempDir()
	store := NewStore(base)
	path, err := store.WritePlan("sess-1", "plan-1.md", "# hi\n")
	if err != nil {
		t.Fatalf("WritePlan: %v", err)
	}
	wantDir := filepath.Join(SessionDir(base, "sess-1"), "plans")
	if filepath.Dir(path) != wantDir {
		t.Errorf("plan dir = %q, want %q", filepath.Dir(path), wantDir)
	}
	data, err := os.ReadFile(path)
	if err != nil {
		t.Fatalf("read plan: %v", err)
	}
	if !strings.Contains(string(data), "# hi") {
		t.Errorf("plan content = %q, want it to contain the body", string(data))
	}
}

func TestWritePlanEmptyKeyFailsLoud(t *testing.T) {
	store := NewStore(t.TempDir())
	if _, err := store.WritePlan("", "p.md", "x"); err == nil {
		t.Fatal("WritePlan empty key err = nil, want error")
	}
}
```

- [ ] **步骤 2：运行确认失败**

运行：`go test ./internal/sessionstate/ -run TestWritePlan`
预期：FAIL —— `undefined: (*Store).WritePlan`。

- [ ] **步骤 3：实现 `Store.WritePlan`**

加到 `internal/sessionstate/checkpoint.go`（或同包新建 `plans.go`）：

```go
// WritePlan 把一个 Plan 模式产物写到 <base>/session/<sessionKey>/plans/<filename>，
// 建目录。返回所写的绝对路径。空 sessionKey 或 filename 被拒绝（fail-loud —— 绝不
// 写到畸形路径）。这是 OKF「一概念一文件」的计划落地处（设计 §4.2）。
func (s *Store) WritePlan(sessionKey, filename, content string) (string, error) {
	if sessionKey == "" || filename == "" {
		return "", fmt.Errorf("write plan: empty sessionKey or filename (key=%q file=%q)", sessionKey, filename)
	}
	dir := filepath.Join(SessionDir(s.base, sessionKey), "plans")
	if err := os.MkdirAll(dir, 0o755); err != nil {
		return "", fmt.Errorf("create plans dir %q: %w", dir, err)
	}
	path := filepath.Join(dir, filename)
	if err := os.WriteFile(path, []byte(content), 0o644); err != nil {
		return "", fmt.Errorf("write plan %q: %w", path, err)
	}
	return path, nil
}
```

（所有 import —— `fmt`、`os`、`path/filepath` —— checkpoint.go 已有。）

- [ ] **步骤 4：运行 store 测试确认通过**

运行：`go test ./internal/sessionstate/`
预期：PASS。

- [ ] **步骤 5：写失败的 runtime plan 产出测试**

追加到 `internal/runtime/plan_mode_test.go`：

```go
func TestPlanModeWritesOKFPlanFile(t *testing.T) {
	dir := t.TempDir()
	store := sessionstate.NewStore(dir)
	maas := &planProbeMaas{}
	runner := NewRuntime(Config{
		Maas: maas, Audit: adapter.NewMemoryAuditLog(), Events: adapter.NewMemoryEventBus(),
		Tools: planRegistry(t), LazyTools: false, Checkpoints: store,
	})
	_, err := runner.RunTask(context.Background(), domain.Agent{ID: "a1"}, domain.Task{
		ID: "t1", SessionID: "sess-1", AgentID: "a1", Status: domain.TaskRunning, Input: "plan it", Mode: domain.ModePlan,
	})
	if err != nil {
		t.Fatalf("RunTask: %v", err)
	}
	plansDir := filepath.Join(sessionstate.SessionDir(dir, "sess-1"), "plans")
	entries, err := os.ReadDir(plansDir)
	if err != nil {
		t.Fatalf("read plans dir: %v", err)
	}
	if len(entries) != 1 {
		t.Fatalf("plans written = %d, want 1", len(entries))
	}
	data, _ := os.ReadFile(filepath.Join(plansDir, entries[0].Name()))
	body := string(data)
	if !strings.Contains(body, "type: Plan") {
		t.Errorf("plan file missing OKF frontmatter 'type: Plan':\n%s", body)
	}
	if !strings.Contains(body, "plan text") {
		t.Errorf("plan file missing model result 'plan text':\n%s", body)
	}
}
```

（加 import：`os`、`path/filepath`、`strings`、`github.com/stardust/legion-agent/internal/sessionstate`。）

- [ ] **步骤 6：在 `finishRun` 实现 plan 产出**

在 `internal/runtime/runtime.go` 的 `finishRun`，在 run 组装好后、删检查点前（或紧在 `return run, nil` 前），当 `task.Mode==ModePlan` 且 `r.checkpoints != nil` 时写 OKF 计划：

```go
	if task.Mode == domain.ModePlan && r.checkpoints != nil {
		if err := r.writePlanArtifact(task, st.resp.Text); err != nil {
			return domain.TaskRun{}, fmt.Errorf("write plan artifact for task %s: %w", task.ID, err)
		}
	}
```

加 helper（用 `time.Now()` 做文件名 + ISO 时间戳 —— 这是真实 runtime 代码，非 workflow 脚本，故 `time.Now()` 没问题）：

```go
// writePlanArtifact 把模型的计划结果框成 OKF markdown（YAML frontmatter，type: Plan
// + title/description/tags/timestamp，然后正文）写到 session 的 plans/ 目录。设计 §4.2。
func (r *Runtime) writePlanArtifact(task domain.Task, result string) error {
	now := time.Now().UTC()
	ts := now.Format(time.RFC3339)
	title := firstNonEmptyLine(result)
	if title == "" {
		title = "Plan for task " + task.ID
	}
	content := fmt.Sprintf(`---
type: Plan
title: %q
description: "Plan produced in Plan mode for task %s"
tags: [plan, agent]
timestamp: %q
resource: %q
---

%s
`, title, task.ID, ts, task.ID, result)
	filename := fmt.Sprintf("plan-%d.md", now.UnixNano())
	if _, err := r.checkpoints.WritePlan(sessionKeyForTask(task), filename, content); err != nil {
		return err
	}
	return nil
}

// firstNonEmptyLine 返回 s 的首个非空行（trim 后），用作可读的计划标题。
func firstNonEmptyLine(s string) string {
	for _, line := range strings.Split(s, "\n") {
		if t := strings.TrimSpace(line); t != "" {
			return t
		}
	}
	return ""
}
```

（`strings`、`fmt`、`time` 已在 runtime.go import。）

> 注：`r.checkpoints` 是 M1b 的 `*sessionstate.Store`。复用它写计划是刻意的 —— 一个 store 拥有会话目录。serve 里它总非 nil；`r.checkpoints != nil` 守卫使那些没接 store 的测试/其他调用方的 plan 模式运行不崩（它们只是跳过产物 —— 可接受，运行结果照返）。这是契约可选情形（未配 store ⇒ 无盘上计划），**非**吞错。

- [ ] **步骤 7：运行确认通过**

运行：`go test ./internal/runtime/ ./internal/sessionstate/`
预期：PASS（写出带 OKF frontmatter 的计划文件；所有先前测试绿）。

- [ ] **步骤 8：完整门禁 + race + 提交**

运行（Windows）：`go build ./... && go vet ./... && go test ./...`
运行（WSL）：`-race` 命令。
运行：`gofmt -l` 触碰文件（内容 LF 干净）。
```bash
git add internal/sessionstate/ internal/runtime/runtime.go internal/runtime/plan_mode_test.go
git commit -m "feat(runtime): Plan mode writes OKF markdown plan to session dir"
```

---

## 自审（计划作者已完成）

**1. spec 覆盖**（设计 §4.2 M2a 范围）：
- session `mode` 字段（manual|plan|auto，默认 auto），SessionStore + create/patch → 任务 1,3,4。✓
- 任务从 session 解析 mode；一次性 = auto → 任务 4。✓
- 未知 mode 被拒（fail-loud，非静默 auto）→ 任务 1（domain）、4（HTTP 400）。✓
- Plan = `Registry.Subset` 只读工具 + 「产出计划、不执行」指令 → 任务 5。✓
- Plan 产出 = OKF markdown（frontmatter `type: Plan` + title/description/tags/timestamp）到会话目录 `plans/`，一计划一文件 → 任务 6。✓
- 工具 `Sensitive` 位（Plan 安全子集需要；M2b gate 复用）→ 任务 2。✓
- Auto 不变 → 任务 5（TestAutoModeOffersAllTools）+ 所有既有测试保持绿 佐证。✓
- 推迟到 M2b/M2c（非缺口）：Manual 模式下 gate 敏感工具的 approval-backed `ToolGate`；审批持久化；SSE。M2a 落 mode 流转 + Sensitive 分类 + Plan；Manual 被接收/存储，但从 M2b 才开始拦截。

**2. placeholder 扫描：** 每个代码步都有真实 Go。四处标注的核对点（任务 2 registry 工具集；任务 3 迁移步形态 + INSERT 列表；任务 4 测试骨架 helper 名；任务 5 工具穿线管线）是真实代码库核对 —— 断言的行为已完全指定；标注引导实现者对齐既有精确签名。

**3. 类型一致性：** `domain.ModeManual/ModePlan/ModeAuto` + `NormalizeMode` 在 domain/storage/server/runtime 一致使用。`Descriptor.Sensitive`、`SafeToolNames`、`Subset`、`Store.WritePlan`、`effectiveTools`、`writePlanArtifact` 名在定义与使用间一致。`Task.Mode`/`AgentSession.Mode` 是端到端穿线的单一真相源。

---

## 执行交接

推荐 **Subagent-Driven**（每任务新 subagent + 双阶段 review），依 M1b 交接注意 —— 实现者/审查者把报告写文件（`.superpowers/sdd/task-N-*.md`），因 OMC/caveman hook 吞 subagent 最终消息；靠读文件 + 自己重跑 WSL `-race` 配方验收。M2a 之后：M2b（Manual 审批 gate）再 M2c（SSE）各出一份 plan。
