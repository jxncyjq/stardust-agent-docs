# legionAgentGUI 设置菜单 — Agent 配置可视化 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 给 legionAgentGUI（Wails v2 桌面应用）加一个 Sidebar 底部齿轮入口的设置模态，把 `agent.json` 做结构化可视化编辑，保存后热重启内嵌 serve 生效。

**Architecture:** 新增 legionAgent 侧公开桥 `serve.ValidateConfig`（GUI 独立 module 无法 import `internal/config`）；GUI 侧加 `ServeManager.Restart` 与 `App` 的 `GetConfig`/`GetConfigPath`/`SaveConfig` 绑定；前端一个 zustand `configStore` + 元数据驱动的 `SettingsModal`。数据流：GetConfig → 表单草稿 → SaveConfig（写临时文件→权威校验→备份→原子替换→重启 serve）→ `serve:status` 事件 → 前端接新端口。

**Tech Stack:** Go 1.26（两个 module：`legionAgent`、`legionAgentGUI`）、Wails v2、React 18 + zustand 5 + Tailwind、vitest 2（仅 node 环境，无 RTL/jsdom）。

## Global Constraints

- **Fail-Loud 铁律**（`legionAgentGUI/CLAUDE.md`、`legionAgent/CLAUDE.md`）：配置读写全程禁止兜底，出错必报必记（`%w` 包装 + 路径上下文），校验未过绝不写盘。
- **模块边界**：`legionAgentGUI` 是独立 module，**禁止** import `github.com/stardust/legion-agent/internal/**`；跨模块能力一律经 `serve` 公开包桥接。
- **Go 完成标准**：对应 module 内 `go build ./... && go vet ./... && go test ./...` 全绿、`gofmt -l .` 为空。
- **前端完成标准**：`npm run build`（含 `tsc` 类型检查）通过；`npm run test`（vitest）全绿。
- **工作目录**：命令里的相对路径以仓库根 `F:\source\stardust\Legion` 为基准；`legion/legionAgent` 与 `legion/legionAgentGUI` 是两个 Go module。
- **Git**：本仓库当前非 git 仓库。计划**不含 commit 步骤**；每个任务以其构建/测试**验证关卡**作为完成标志。若后续 `git init`，可在每个验证关卡后自行提交。
- **危险路径只读**：凡文件系统路径/目录/驱动/监听地址字段一律只读展示（见 Task 7 元数据 `readonly: true`），保存时原样写回，UI 禁止编辑。
- **密钥打码**：`maas.api_key`、各 profile `api_key`、`server.admin_token` 用 SecretField（默认打码 + 👁 reveal）。

---

### Task 1: legionAgent `serve.ValidateConfig` 公开桥

**Files:**
- Create: `legion/legionAgent/serve/validate.go`
- Test: `legion/legionAgent/serve/validate_test.go`

**Interfaces:**
- Consumes: `internal/config.Load(ctx, config.Options{Path})`（同 module 内可用）。
- Produces: `func ValidateConfig(ctx context.Context, path string) error` —— GUI module 用它做权威校验。

- [ ] **Step 1: 写失败测试**

`legion/legionAgent/serve/validate_test.go`：
```go
package serve_test

import (
	"context"
	"os"
	"path/filepath"
	"testing"

	"github.com/stardust/legion-agent/serve"
)

func TestValidateConfigAcceptsValid(t *testing.T) {
	dir := t.TempDir()
	good := filepath.Join(dir, "good.json")
	if err := os.WriteFile(good, []byte(`{"storage":{"driver":"memory"}}`), 0o644); err != nil {
		t.Fatal(err)
	}
	if err := serve.ValidateConfig(context.Background(), good); err != nil {
		t.Fatalf("valid config rejected: %v", err)
	}
}

func TestValidateConfigRejectsMalformed(t *testing.T) {
	dir := t.TempDir()
	bad := filepath.Join(dir, "bad.json")
	// storage must be an object; a string is a type mismatch json.Unmarshal rejects.
	if err := os.WriteFile(bad, []byte(`{"storage":"nope"}`), 0o644); err != nil {
		t.Fatal(err)
	}
	if err := serve.ValidateConfig(context.Background(), bad); err == nil {
		t.Fatal("malformed config accepted; want error")
	}
}
```

- [ ] **Step 2: 跑测试确认失败**

Run: `cd legion/legionAgent && go test ./serve/ -run TestValidateConfig -v`
Expected: 编译失败 `undefined: serve.ValidateConfig`。

- [ ] **Step 3: 实现**

`legion/legionAgent/serve/validate.go`：
```go
package serve

import (
	"context"

	"github.com/stardust/legion-agent/internal/config"
)

// ValidateConfig reports whether the config file at path can be loaded by the
// authoritative loader. It returns the same error config.Load would, so callers
// in other modules (e.g. legionAgentGUI, which cannot import internal/config)
// can validate a candidate agent.json before replacing the live file.
func ValidateConfig(ctx context.Context, path string) error {
	_, err := config.Load(ctx, config.Options{Path: path})
	return err
}
```

- [ ] **Step 4: 跑测试确认通过**

Run: `cd legion/legionAgent && go test ./serve/ -run TestValidateConfig -v`
Expected: PASS（两个用例）。

- [ ] **Step 5: 验证关卡**

Run: `cd legion/legionAgent && go build ./... && go vet ./serve/ && gofmt -l serve/`
Expected: 无输出（构建/vet 通过、gofmt 干净）。

---

### Task 2: `ServeManager` 可注入 emit + `Restart` 方法

**Files:**
- Modify: `legion/legionAgentGUI/serve_manager.go`
- Test: `legion/legionAgentGUI/serve_restart_test.go`

**Interfaces:**
- Consumes: 现有 `serve.BuildService`、`m.running atomic.Bool`、`m.Start`、`m.Stop`。
- Produces:
  - `ServeManager.emit func(ctx context.Context, event string, data ...any)`（字段；`NewServeManager` 默认 `runtime.EventsEmit`，测试可覆盖以绕开 Wails runtime）。
  - `func (m *ServeManager) Restart(appCtx context.Context, configPath string) error`。

- [ ] **Step 1: 写测试（门控 e2e，覆盖 emit 绕开 Wails）**

`legion/legionAgentGUI/serve_restart_test.go`：
```go
package main

import (
	"context"
	"os"
	"path/filepath"
	"testing"
)

// TestServeManagerRestart starts the embedded service, restarts it, and asserts
// it is running again on a (new) port. Gated behind LEGION_E2E=1 because it
// builds a real service. The emit func is overridden to a no-op so Start does
// not call the Wails runtime (which needs a Wails-managed context).
func TestServeManagerRestart(t *testing.T) {
	if os.Getenv("LEGION_E2E") != "1" {
		t.Skip("set LEGION_E2E=1 to run")
	}
	cfg := resolveConfigPath()
	if cfg == "" {
		t.Fatal("no config resolved")
	}
	abs, _ := filepath.Abs(cfg)
	if err := os.Chdir(filepath.Dir(abs)); err != nil {
		t.Fatal(err)
	}

	m := NewServeManager()
	m.emit = func(context.Context, string, ...any) {} // bypass Wails runtime
	ctx := context.Background()

	if err := m.Start(ctx, abs); err != nil {
		t.Fatalf("start: %v", err)
	}
	if !m.Running() {
		t.Fatal("not running after Start")
	}
	if err := m.Restart(ctx, abs); err != nil {
		t.Fatalf("restart: %v", err)
	}
	if !m.Running() {
		t.Fatal("not running after Restart")
	}
	t.Logf("restart ok, port=%d", m.Port())
	m.Stop()
}
```

- [ ] **Step 2: 跑测试确认失败**

Run: `cd legion/legionAgentGUI && go vet ./...`
Expected: 编译失败 `m.emit undefined` / `m.Restart undefined`。

- [ ] **Step 3: 改 `serve_manager.go`——加 emit 字段并改 Start 用它**

把结构体与构造函数改为（替换现有 `type ServeManager struct {...}` 和 `NewServeManager`）：
```go
type ServeManager struct {
	cancel  context.CancelFunc
	port    int
	running atomic.Bool
	// emit sends frontend events. It defaults to runtime.EventsEmit; tests
	// override it to bypass the Wails runtime (which requires a Wails context).
	emit func(ctx context.Context, event string, data ...any)
}

func NewServeManager() *ServeManager {
	return &ServeManager{emit: runtime.EventsEmit}
}
```
把 `Start` 内三处 `runtime.EventsEmit(appCtx, ...)` 改为 `m.emit(appCtx, ...)`（`serve:status` 两处、`serve:error` 一处）。

- [ ] **Step 4: 加 `Restart` 方法**

在 `serve_manager.go` 末尾（`listenerPort` 之前或之后）加：
```go
// Restart stops the running embedded service, waits for it to fully stop, then
// starts it again against configPath (which may point at freshly-written
// config). It reuses the serve:status event so the frontend reconnects to the
// new random port. A stop that does not complete within the timeout is reported
// as an error rather than racing a second Start against a still-running service.
func (m *ServeManager) Restart(appCtx context.Context, configPath string) error {
	m.Stop()
	deadline := time.Now().Add(5 * time.Second)
	for m.running.Load() {
		if time.Now().After(deadline) {
			return fmt.Errorf("serve did not stop within 5s; refusing to restart")
		}
		time.Sleep(20 * time.Millisecond)
	}
	if err := m.Start(appCtx, configPath); err != nil {
		return fmt.Errorf("restart serve with config %q: %w", configPath, err)
	}
	return nil
}
```
在 import 块补 `"fmt"` 和 `"time"`（保留现有 `context`/`net`/`sync/atomic`/`runtime`/`serve`）。

- [ ] **Step 5: 验证关卡（编译 + gofmt；e2e 可选）**

Run: `cd legion/legionAgentGUI && go build ./... && go vet ./... && gofmt -l serve_manager.go serve_restart_test.go`
Expected: 无输出。
可选 e2e：`LEGION_E2E=1 go test ./ -run TestServeManagerRestart -v` → PASS。

---

### Task 3: `App.GetConfig` / `GetConfigPath` 绑定

**Files:**
- Create: `legion/legionAgentGUI/app_config.go`
- Test: `legion/legionAgentGUI/app_config_test.go`

**Interfaces:**
- Consumes: `App.cfgPath`。
- Produces:
  - `func (a *App) GetConfig() (string, error)`
  - `func (a *App) GetConfigPath() (string, error)`

- [ ] **Step 1: 写失败测试**

`legion/legionAgentGUI/app_config_test.go`：
```go
package main

import (
	"os"
	"path/filepath"
	"testing"
)

func TestGetConfigReturnsRawFile(t *testing.T) {
	dir := t.TempDir()
	p := filepath.Join(dir, "agent.json")
	content := `{"storage":{"driver":"memory"}}`
	if err := os.WriteFile(p, []byte(content), 0o644); err != nil {
		t.Fatal(err)
	}
	a := NewApp(p)
	got, err := a.GetConfig()
	if err != nil {
		t.Fatalf("GetConfig: %v", err)
	}
	if got != content {
		t.Fatalf("GetConfig = %q, want %q", got, content)
	}
}

func TestGetConfigEmptyPathErrors(t *testing.T) {
	a := NewApp("")
	if _, err := a.GetConfig(); err == nil {
		t.Fatal("expected error for empty config path")
	}
}

func TestGetConfigPath(t *testing.T) {
	a := NewApp("/tmp/x/agent.json")
	got, err := a.GetConfigPath()
	if err != nil {
		t.Fatal(err)
	}
	if got != "/tmp/x/agent.json" {
		t.Fatalf("GetConfigPath = %q", got)
	}
}
```

- [ ] **Step 2: 跑测试确认失败**

Run: `cd legion/legionAgentGUI && go test ./ -run TestGetConfig -v`
Expected: 编译失败 `a.GetConfig undefined`。

- [ ] **Step 3: 实现**

`legion/legionAgentGUI/app_config.go`：
```go
package main

import (
	"fmt"
	"os"
)

// GetConfig returns the raw JSON bytes of the active config file as a string.
// The settings form renders this verbatim (not passed through config.Load), so
// "what you see is what's in the file" — no env overrides or defaults leak in.
// Called by React via the Wails bindings.
func (a *App) GetConfig() (string, error) {
	if a.cfgPath == "" {
		return "", fmt.Errorf("no config path resolved; cannot read config")
	}
	data, err := os.ReadFile(a.cfgPath)
	if err != nil {
		return "", fmt.Errorf("read config %q: %w", a.cfgPath, err)
	}
	return string(data), nil
}

// GetConfigPath returns the absolute path of the active config file so the UI
// can show which file it is editing. Called by React via the Wails bindings.
func (a *App) GetConfigPath() (string, error) {
	if a.cfgPath == "" {
		return "", fmt.Errorf("no config path resolved")
	}
	return a.cfgPath, nil
}
```

- [ ] **Step 4: 跑测试确认通过**

Run: `cd legion/legionAgentGUI && go test ./ -run TestGetConfig -v`
Expected: PASS（三个用例）。

- [ ] **Step 5: 验证关卡**

Run: `cd legion/legionAgentGUI && go build ./... && gofmt -l app_config.go app_config_test.go`
Expected: 无输出。

---

### Task 4: `App.writeConfig` + `App.SaveConfig`

**Files:**
- Modify: `legion/legionAgentGUI/app_config.go`
- Test: `legion/legionAgentGUI/app_config_test.go`

**Interfaces:**
- Consumes: `serve.ValidateConfig`（Task 1）、`App.serve.Restart`（Task 2）、`App.ctx`、`App.cfgPath`、`App.serve.emit`。
- Produces:
  - `func (a *App) writeConfig(raw string) error` —— 纯文件系统（校验+备份+原子替换），无 serve 重启，可单测。
  - `func (a *App) SaveConfig(raw string) error` —— writeConfig 后重启 serve。

- [ ] **Step 1: 写失败测试（针对 writeConfig；SaveConfig 的重启由 Task 2 e2e 覆盖）**

追加到 `legion/legionAgentGUI/app_config_test.go`：
```go
import "strings" // 加到已有 import 块

func TestWriteConfigReplacesAndBacksUp(t *testing.T) {
	dir := t.TempDir()
	p := filepath.Join(dir, "agent.json")
	original := `{"storage":{"driver":"memory"}}`
	if err := os.WriteFile(p, []byte(original), 0o644); err != nil {
		t.Fatal(err)
	}
	a := NewApp(p)
	newRaw := `{"storage":{"driver":"memory"},"runtime":{"max_tool_rounds":7}}`
	if err := a.writeConfig(newRaw); err != nil {
		t.Fatalf("writeConfig: %v", err)
	}
	got, _ := os.ReadFile(p)
	if string(got) != newRaw {
		t.Fatalf("config not replaced: %s", got)
	}
	bak, err := os.ReadFile(p + ".bak")
	if err != nil {
		t.Fatalf("backup missing: %v", err)
	}
	if string(bak) != original {
		t.Fatalf("backup wrong: %s", bak)
	}
}

func TestWriteConfigRejectsInvalidLeavingFileUntouched(t *testing.T) {
	dir := t.TempDir()
	p := filepath.Join(dir, "agent.json")
	original := `{"storage":{"driver":"memory"}}`
	if err := os.WriteFile(p, []byte(original), 0o644); err != nil {
		t.Fatal(err)
	}
	a := NewApp(p)
	if err := a.writeConfig(`{"storage":"not-an-object"}`); err == nil {
		t.Fatal("invalid config accepted")
	}
	got, _ := os.ReadFile(p)
	if string(got) != original {
		t.Fatalf("live file changed on invalid save: %s", got)
	}
	if _, err := os.Stat(p + ".bak"); !os.IsNotExist(err) {
		t.Fatal("backup written for a rejected save")
	}
	entries, _ := os.ReadDir(dir)
	for _, e := range entries {
		if strings.HasPrefix(e.Name(), "agent.json.tmp-") {
			t.Fatalf("temp file left behind: %s", e.Name())
		}
	}
}
```

- [ ] **Step 2: 跑测试确认失败**

Run: `cd legion/legionAgentGUI && go test ./ -run TestWriteConfig -v`
Expected: 编译失败 `a.writeConfig undefined`。

- [ ] **Step 3: 实现（追加到 `app_config.go`）**

把 import 块改为：
```go
import (
	"context"
	"fmt"
	"os"
	"path/filepath"

	"github.com/stardust/legion-agent/serve"
)
```
追加方法：
```go
// writeConfig validates raw against the authoritative loader and, only if valid,
// atomically replaces the config file — backing the current one up to
// <file>.bak first. It performs no service restart, so it is safe to unit-test.
// Any failure before the rename leaves the live config file untouched.
func (a *App) writeConfig(raw string) error {
	if a.cfgPath == "" {
		return fmt.Errorf("no config path resolved; cannot save config")
	}
	dir := filepath.Dir(a.cfgPath)
	tmp, err := os.CreateTemp(dir, "agent.json.tmp-*")
	if err != nil {
		return fmt.Errorf("create temp config in %q: %w", dir, err)
	}
	tmpPath := tmp.Name()
	// Remove the temp file on any early return; a successful rename consumes it
	// first, making this a harmless no-op.
	defer os.Remove(tmpPath)

	if _, err := tmp.WriteString(raw); err != nil {
		tmp.Close()
		return fmt.Errorf("write temp config %q: %w", tmpPath, err)
	}
	if err := tmp.Close(); err != nil {
		return fmt.Errorf("close temp config %q: %w", tmpPath, err)
	}

	// Authoritative validation through the serve bridge (GUI cannot import
	// internal/config). Reject malformed/type-mismatched config before the live
	// file is touched.
	if err := serve.ValidateConfig(context.Background(), tmpPath); err != nil {
		return fmt.Errorf("validate new config: %w", err)
	}

	// Back up the current file so a loadable-but-service-breaking config (e.g. an
	// unreachable storage path) can be restored by hand.
	current, err := os.ReadFile(a.cfgPath)
	if err != nil {
		return fmt.Errorf("read current config %q for backup: %w", a.cfgPath, err)
	}
	bakPath := a.cfgPath + ".bak"
	if err := os.WriteFile(bakPath, current, 0o644); err != nil {
		return fmt.Errorf("write config backup %q: %w", bakPath, err)
	}

	if err := os.Rename(tmpPath, a.cfgPath); err != nil {
		return fmt.Errorf("replace config %q: %w", a.cfgPath, err)
	}
	return nil
}

// SaveConfig persists a new config and restarts the embedded service so the
// change takes effect without an app restart. On a restart failure the new file
// is already in place and the previous one is at <file>.bak; the error names the
// backup and a serve:error event is emitted so the badge explains the outage.
// Called by React via the Wails bindings from the settings modal.
func (a *App) SaveConfig(raw string) error {
	if err := a.writeConfig(raw); err != nil {
		return err
	}
	if err := a.serve.Restart(a.ctx, a.cfgPath); err != nil {
		a.serve.emit(a.ctx, "serve:error", map[string]any{"error": err.Error()})
		return fmt.Errorf("config saved but serve restart failed (backup at %q): %w", a.cfgPath+".bak", err)
	}
	return nil
}
```

- [ ] **Step 4: 跑测试确认通过**

Run: `cd legion/legionAgentGUI && go test ./ -run TestWriteConfig -v`
Expected: PASS（两个用例）。

- [ ] **Step 5: 验证关卡**

Run: `cd legion/legionAgentGUI && go build ./... && go vet ./... && go test ./... && gofmt -l .`
Expected: 测试全绿（e2e 用例 SKIP），gofmt 无输出。

---

### Task 5: 重新生成 Wails TS 绑定

**Files:**
- Modify（自动生成）: `legion/legionAgentGUI/frontend/wailsjs/go/main/App.d.ts`、`App.js`

- [ ] **Step 1: 生成绑定**

Run: `cd legion/legionAgentGUI && wails generate module`
（若 `wails` CLI 不在 PATH：`go run github.com/wailsapp/wails/v2/cmd/wails generate module`）

- [ ] **Step 2: 确认新方法已导出**

Run: `grep -E "GetConfig|GetConfigPath|SaveConfig" legion/legionAgentGUI/frontend/wailsjs/go/main/App.d.ts`
Expected: 三个函数签名都出现（`GetConfig():Promise<string>`、`GetConfigPath():Promise<string>`、`SaveConfig(arg1:string):Promise<void>`）。

> 若 CLI 不可用，可手动在 `App.d.ts`/`App.js` 按现有 `RenameSession` 等模式补三个导出，签名同上。

---

### Task 6: 对象路径读写工具 `lib/objectPath.ts`

**Files:**
- Create: `legion/legionAgentGUI/frontend/src/lib/objectPath.ts`
- Test: `legion/legionAgentGUI/frontend/src/lib/objectPath.test.ts`

**Interfaces:**
- Produces:
  - `getPath(obj: any, path: string): any` —— 按 `"a.b.c"` 读，缺失返回 `undefined`。
  - `setPath<T>(obj: T, path: string, value: any): T` —— 返回**新对象**（不可变），沿途浅拷贝，末端写入 value。

- [ ] **Step 1: 写失败测试**

`legion/legionAgentGUI/frontend/src/lib/objectPath.test.ts`：
```ts
import { describe, it, expect } from 'vitest'
import { getPath, setPath } from './objectPath'

describe('objectPath', () => {
  it('reads nested values', () => {
    const o = { a: { b: { c: 5 } } }
    expect(getPath(o, 'a.b.c')).toBe(5)
    expect(getPath(o, 'a.b.x')).toBeUndefined()
    expect(getPath(o, 'z.y')).toBeUndefined()
  })

  it('sets nested values immutably', () => {
    const o = { a: { b: { c: 1 } }, keep: 9 }
    const next = setPath(o, 'a.b.c', 2)
    expect(next.a.b.c).toBe(2)
    expect(next.keep).toBe(9)
    expect(o.a.b.c).toBe(1) // original untouched
    expect(next).not.toBe(o)
    expect(next.a).not.toBe(o.a)
  })

  it('creates missing intermediate objects', () => {
    const next = setPath({} as any, 'x.y.z', 7)
    expect(next.x.y.z).toBe(7)
  })
})
```

- [ ] **Step 2: 跑测试确认失败**

Run: `cd legion/legionAgentGUI/frontend && npx vitest run src/lib/objectPath.test.ts`
Expected: FAIL（无法解析 `./objectPath`）。

- [ ] **Step 3: 实现**

`legion/legionAgentGUI/frontend/src/lib/objectPath.ts`：
```ts
// getPath reads a dot-separated path from a nested object, returning undefined
// when any segment is missing.
export function getPath(obj: any, path: string): any {
  return path.split('.').reduce((acc, key) => (acc == null ? undefined : acc[key]), obj)
}

// setPath returns a shallow-cloned copy of obj with value written at the
// dot-separated path, creating missing intermediate objects. The input is never
// mutated, so React state comparisons see a new reference along the changed path.
export function setPath<T>(obj: T, path: string, value: any): T {
  const keys = path.split('.')
  const clone: any = Array.isArray(obj) ? [...(obj as any)] : { ...(obj as any) }
  let cursor = clone
  for (let i = 0; i < keys.length - 1; i++) {
    const key = keys[i]
    const child = cursor[key]
    cursor[key] = child != null && typeof child === 'object' ? (Array.isArray(child) ? [...child] : { ...child }) : {}
    cursor = cursor[key]
  }
  cursor[keys[keys.length - 1]] = value
  return clone
}
```

- [ ] **Step 4: 跑测试确认通过**

Run: `cd legion/legionAgentGUI/frontend && npx vitest run src/lib/objectPath.test.ts`
Expected: PASS。

---

### Task 7: 配置 schema 与字段元数据 `types/config.ts`

**Files:**
- Create: `legion/legionAgentGUI/frontend/src/types/config.ts`
- Test: `legion/legionAgentGUI/frontend/src/types/config.test.ts`

**Interfaces:**
- Produces:
  - `type FieldWidget = 'toggle' | 'number' | 'text' | 'secret' | 'color' | 'profiles' | 'stringlist' | 'readonly'`
  - `interface FieldSpec { path: string; label: string; widget: FieldWidget }`
  - `interface SectionSpec { key: string; title: string; help: string; advanced: boolean; fields: FieldSpec[] }`
  - `const CONFIG_SECTIONS: SectionSpec[]` —— 驱动 SettingsModal 渲染。

- [ ] **Step 1: 写失败测试（守卫：段齐全 + 只读白名单准确）**

`legion/legionAgentGUI/frontend/src/types/config.test.ts`：
```ts
import { describe, it, expect } from 'vitest'
import { CONFIG_SECTIONS } from './config'

const allFields = CONFIG_SECTIONS.flatMap((s) => s.fields)
const paths = allFields.map((f) => f.path)

describe('CONFIG_SECTIONS', () => {
  it('covers all 14 config sections', () => {
    const keys = new Set(CONFIG_SECTIONS.map((s) => s.key))
    for (const k of [
      'maas', 'runtime', 'session', 'tui', 'context_files',
      'server', 'service', 'tasks', 'skills', 'web', 'evolution',
      'storage', 'workspace', 'agents',
    ]) {
      expect(keys.has(k)).toBe(true)
    }
  })

  it('marks every path/dir field readonly', () => {
    const mustBeReadonly = [
      'storage.driver', 'storage.path', 'server.listen_addr',
      'context_files.root', 'context_files.soul_path', 'context_files.tools_path',
      'context_files.user_path', 'context_files.memory_path',
      'workspace.docs_root', 'workspace.memory_root',
      'tasks.index_path', 'tasks.root', 'tasks.archive_root', 'skills.install_root',
    ]
    for (const p of mustBeReadonly) {
      const f = allFields.find((x) => x.path === p)
      expect(f, `missing field ${p}`).toBeTruthy()
      expect(f!.widget, `${p} should be readonly`).toBe('readonly')
    }
  })

  it('marks secret fields', () => {
    for (const p of ['maas.api_key', 'server.admin_token']) {
      expect(allFields.find((x) => x.path === p)?.widget).toBe('secret')
    }
  })

  it('has no duplicate paths', () => {
    expect(new Set(paths).size).toBe(paths.length)
  })
})
```

- [ ] **Step 2: 跑测试确认失败**

Run: `cd legion/legionAgentGUI/frontend && npx vitest run src/types/config.test.ts`
Expected: FAIL（无法解析 `./config`）。

- [ ] **Step 3: 实现**

`legion/legionAgentGUI/frontend/src/types/config.ts`：
```ts
// FieldWidget selects the control the settings modal renders for a config field.
export type FieldWidget =
  | 'toggle'
  | 'number'
  | 'text'
  | 'secret'
  | 'color'
  | 'profiles'
  | 'stringlist'
  | 'readonly'

// FieldSpec describes one editable (or read-only) config field by its dot-path
// into the agent.json object.
export interface FieldSpec {
  path: string
  label: string
  widget: FieldWidget
}

// SectionSpec groups fields under a collapsible section. `advanced` sections
// render collapsed by default. `help` comes from the agent.complete.example.json
// _comment for that section.
export interface SectionSpec {
  key: string
  title: string
  help: string
  advanced: boolean
  fields: FieldSpec[]
}

const THEME_COLORS: FieldSpec[] = [
  { path: 'tui.theme.accent', label: 'accent', widget: 'color' },
  { path: 'tui.theme.accent2', label: 'accent2', widget: 'color' },
  { path: 'tui.theme.text', label: 'text', widget: 'color' },
  { path: 'tui.theme.dim', label: 'dim', widget: 'color' },
  { path: 'tui.theme.error', label: 'error', widget: 'color' },
  { path: 'tui.theme.status_fg', label: 'status_fg', widget: 'color' },
  { path: 'tui.theme.status_bg', label: 'status_bg', widget: 'color' },
  { path: 'tui.theme.shell_bg', label: 'shell_bg', widget: 'color' },
]

// CONFIG_SECTIONS is the single source of truth for the settings form layout.
// Ordered: common (advanced:false) first, then advanced (collapsed).
export const CONFIG_SECTIONS: SectionSpec[] = [
  {
    key: 'maas',
    title: '模型 (MaaS)',
    help: '设置 model 时走 OpenAI 兼容 /chat/completions；model 为空时走 Legion 内部 MaaS。',
    advanced: false,
    fields: [
      { path: 'maas.base_url', label: 'base_url', widget: 'text' },
      { path: 'maas.api_key', label: 'api_key', widget: 'secret' },
      { path: 'maas.default_profile', label: 'default_profile', widget: 'text' },
      { path: 'maas.profiles', label: 'profiles', widget: 'profiles' },
    ],
  },
  {
    key: 'runtime',
    title: '运行时',
    help: 'demo_response 仅在无真实 MaaS 时降级演示；max_tool_rounds 控制工具调用闭环轮数。',
    advanced: false,
    fields: [
      { path: 'runtime.demo_response', label: 'demo_response', widget: 'text' },
      { path: 'runtime.max_tool_rounds', label: 'max_tool_rounds', widget: 'number' },
      { path: 'runtime.lazy_tools', label: 'lazy_tools', widget: 'toggle' },
    ],
  },
  {
    key: 'session',
    title: '会话',
    help: '多轮会话连续性：把最近 N 轮写入 SQLite 并注入下一轮 prompt。',
    advanced: false,
    fields: [
      { path: 'session.enabled', label: 'enabled', widget: 'toggle' },
      { path: 'session.default_recent_turns', label: 'default_recent_turns', widget: 'number' },
      { path: 'session.max_turn_chars', label: 'max_turn_chars', widget: 'number' },
      { path: 'session.restore_latest_on_tui_start', label: 'restore_latest_on_tui_start', widget: 'toggle' },
      { path: 'session.cache_enabled', label: 'cache_enabled', widget: 'toggle' },
      { path: 'session.cache_max_entries', label: 'cache_max_entries', widget: 'number' },
    ],
  },
  {
    key: 'tui',
    title: '界面主题 (TUI)',
    help: '交互式 TUI 配置；theme 支持 ANSI 色号或 truecolor hex。',
    advanced: false,
    fields: [
      { path: 'tui.show_prompt', label: 'show_prompt', widget: 'toggle' },
      { path: 'tui.show_thinking', label: 'show_thinking', widget: 'toggle' },
      { path: 'tui.color_profile', label: 'color_profile', widget: 'text' },
      ...THEME_COLORS,
    ],
  },
  {
    key: 'context_files',
    title: '上下文文件',
    help: '运行时注入上下文。路径类字段只读（改错会导致 serve 启动失败）。',
    advanced: false,
    fields: [
      { path: 'context_files.enabled', label: 'enabled', widget: 'toggle' },
      { path: 'context_files.max_file_chars', label: 'max_file_chars', widget: 'number' },
    ],
  },
  {
    key: 'server',
    title: '服务器 (高级)',
    help: 'agent serve 的 HTTP 管理面。注意：GUI 内嵌 serve 固定用随机端口，listen_addr 不生效。',
    advanced: true,
    fields: [
      { path: 'server.listen_addr', label: 'listen_addr (不生效)', widget: 'readonly' },
      { path: 'server.admin_token', label: 'admin_token', widget: 'secret' },
      { path: 'server.public_health_enabled', label: 'public_health_enabled', widget: 'toggle' },
      { path: 'server.request_id_header', label: 'request_id_header', widget: 'text' },
    ],
  },
  {
    key: 'service',
    title: '后台 (高级)',
    help: '后台 coordinator heartbeat 间隔（如 "1s"）。',
    advanced: true,
    fields: [
      { path: 'service.background_interval', label: 'background_interval', widget: 'text' },
    ],
  },
  {
    key: 'storage',
    title: '存储 (高级·只读)',
    help: '存储驱动与路径。危险字段，只读展示。',
    advanced: true,
    fields: [
      { path: 'storage.driver', label: 'driver', widget: 'readonly' },
      { path: 'storage.path', label: 'path', widget: 'readonly' },
    ],
  },
  {
    key: 'tasks',
    title: '任务账本 (高级)',
    help: '多 Agent 协作任务账本。路径类只读，上限与状态数组可改。',
    advanced: true,
    fields: [
      { path: 'tasks.index_path', label: 'index_path', widget: 'readonly' },
      { path: 'tasks.root', label: 'root', widget: 'readonly' },
      { path: 'tasks.archive_root', label: 'archive_root', widget: 'readonly' },
      { path: 'tasks.max_index_lines', label: 'max_index_lines', widget: 'number' },
      { path: 'tasks.max_task_lines', label: 'max_task_lines', widget: 'number' },
      { path: 'tasks.max_message_chars', label: 'max_message_chars', widget: 'number' },
      { path: 'tasks.active_statuses', label: 'active_statuses', widget: 'stringlist' },
      { path: 'tasks.done_statuses', label: 'done_statuses', widget: 'stringlist' },
    ],
  },
  {
    key: 'context_files_paths',
    title: '上下文路径 (高级·只读)',
    help: 'AGENTS/SOUL/TOOLS/USER/MEMORY 注入路径。危险字段，只读展示。',
    advanced: true,
    fields: [
      { path: 'context_files.root', label: 'root', widget: 'readonly' },
      { path: 'context_files.soul_path', label: 'soul_path', widget: 'readonly' },
      { path: 'context_files.tools_path', label: 'tools_path', widget: 'readonly' },
      { path: 'context_files.user_path', label: 'user_path', widget: 'readonly' },
      { path: 'context_files.memory_path', label: 'memory_path', widget: 'readonly' },
    ],
  },
  {
    key: 'workspace',
    title: '工作区 (高级·只读)',
    help: 'docs/memory 产出目录。危险字段，只读展示。',
    advanced: true,
    fields: [
      { path: 'workspace.docs_root', label: 'docs_root', widget: 'readonly' },
      { path: 'workspace.memory_root', label: 'memory_root', widget: 'readonly' },
    ],
  },
  {
    key: 'skills',
    title: '技能 (高级)',
    help: 'registry_url 可改；install_root 路径只读。',
    advanced: true,
    fields: [
      { path: 'skills.registry_url', label: 'registry_url', widget: 'text' },
      { path: 'skills.install_root', label: 'install_root', widget: 'readonly' },
    ],
  },
  {
    key: 'web',
    title: 'Web 工具 (高级)',
    help: 'fetch_url 工具。默认开启 SSRF 防护，allow_private_hosts 才允许内网 IP。',
    advanced: true,
    fields: [
      { path: 'web.enabled', label: 'enabled', widget: 'toggle' },
      { path: 'web.allow_private_hosts', label: 'allow_private_hosts', widget: 'toggle' },
      { path: 'web.timeout_seconds', label: 'timeout_seconds', widget: 'number' },
      { path: 'web.max_response_kb', label: 'max_response_kb', widget: 'number' },
      { path: 'web.allowlist', label: 'allowlist', widget: 'stringlist' },
    ],
  },
  {
    key: 'evolution',
    title: '退化检测 (高级)',
    help: '周期性退化检测任务的阈值/窗口/扫描间隔。',
    advanced: true,
    fields: [
      { path: 'evolution.degradation_threshold', label: 'degradation_threshold', widget: 'number' },
      { path: 'evolution.degradation_window_days', label: 'degradation_window_days', widget: 'number' },
      { path: 'evolution.degradation_scan_minutes', label: 'degradation_scan_minutes', widget: 'number' },
    ],
  },
  {
    key: 'agents',
    title: '子 Agent (高级·只读)',
    help: '子 Agent 配置文件路径映射。危险字段，只读展示。',
    advanced: true,
    fields: [
      { path: 'agents', label: 'agents', widget: 'readonly' },
    ],
  },
]
```

> 注：测试里断言的 `context_files.agents_path` / `config_root` 属废弃字段，未纳入表单（YAGNI）；故 Step 1 测试的 `mustBeReadonly` 列表**不含**这两项——已如此编写，勿另加。

- [ ] **Step 4: 跑测试确认通过**

Run: `cd legion/legionAgentGUI/frontend && npx vitest run src/types/config.test.ts`
Expected: PASS。

---

### Task 8: `configStore`（zustand）

**Files:**
- Create: `legion/legionAgentGUI/frontend/src/stores/configStore.ts`
- Test: `legion/legionAgentGUI/frontend/src/stores/configStore.test.ts`

**Interfaces:**
- Consumes: `GetConfig`/`GetConfigPath`/`SaveConfig`（`../../wailsjs/go/main/App`）、`setPath`（Task 6）。
- Produces（zustand store）：
  - state: `path: string`、`draft: any`、`baseline: string`、`loading: boolean`、`saving: boolean`、`error: string`、`dirty: boolean`
  - actions: `load(): Promise<void>`、`update(path: string, value: any): void`、`save(): Promise<void>`、`reset(): void`

- [ ] **Step 1: 写失败测试（mock Wails 绑定）**

`legion/legionAgentGUI/frontend/src/stores/configStore.test.ts`：
```ts
import { describe, it, expect, vi, beforeEach } from 'vitest'

const mocks = {
  GetConfig: vi.fn(),
  GetConfigPath: vi.fn(),
  SaveConfig: vi.fn(),
}
vi.mock('../../wailsjs/go/main/App', () => mocks)

import { useConfigStore } from './configStore'

beforeEach(() => {
  mocks.GetConfig.mockReset()
  mocks.GetConfigPath.mockReset()
  mocks.SaveConfig.mockReset()
  useConfigStore.setState({ path: '', draft: null, baseline: '', loading: false, saving: false, error: '', dirty: false })
})

describe('configStore', () => {
  it('loads config into draft and is not dirty', async () => {
    mocks.GetConfig.mockResolvedValue('{"runtime":{"max_tool_rounds":4}}')
    mocks.GetConfigPath.mockResolvedValue('/x/agent.json')
    await useConfigStore.getState().load()
    const s = useConfigStore.getState()
    expect(s.path).toBe('/x/agent.json')
    expect(s.draft.runtime.max_tool_rounds).toBe(4)
    expect(s.dirty).toBe(false)
    expect(s.error).toBe('')
  })

  it('update marks dirty and mutates draft immutably', async () => {
    mocks.GetConfig.mockResolvedValue('{"runtime":{"max_tool_rounds":4}}')
    mocks.GetConfigPath.mockResolvedValue('/x/agent.json')
    await useConfigStore.getState().load()
    useConfigStore.getState().update('runtime.max_tool_rounds', 9)
    const s = useConfigStore.getState()
    expect(s.draft.runtime.max_tool_rounds).toBe(9)
    expect(s.dirty).toBe(true)
  })

  it('save sends serialized draft and clears dirty', async () => {
    mocks.GetConfig.mockResolvedValue('{"runtime":{"max_tool_rounds":4}}')
    mocks.GetConfigPath.mockResolvedValue('/x/agent.json')
    mocks.SaveConfig.mockResolvedValue(undefined)
    await useConfigStore.getState().load()
    useConfigStore.getState().update('runtime.max_tool_rounds', 9)
    await useConfigStore.getState().save()
    expect(mocks.SaveConfig).toHaveBeenCalledTimes(1)
    const sent = mocks.SaveConfig.mock.calls[0][0]
    expect(JSON.parse(sent).runtime.max_tool_rounds).toBe(9)
    expect(useConfigStore.getState().dirty).toBe(false)
  })

  it('records error when load fails', async () => {
    mocks.GetConfig.mockRejectedValue(new Error('read config: boom'))
    mocks.GetConfigPath.mockResolvedValue('/x/agent.json')
    await useConfigStore.getState().load()
    expect(useConfigStore.getState().error).toContain('boom')
  })
})
```

- [ ] **Step 2: 跑测试确认失败**

Run: `cd legion/legionAgentGUI/frontend && npx vitest run src/stores/configStore.test.ts`
Expected: FAIL（无法解析 `./configStore`）。

- [ ] **Step 3: 实现**

`legion/legionAgentGUI/frontend/src/stores/configStore.ts`：
```ts
import { create } from 'zustand'
import { GetConfig, GetConfigPath, SaveConfig } from '../../wailsjs/go/main/App'
import { setPath } from '../lib/objectPath'

// serialize renders the draft as pretty JSON — the exact form written to disk
// and compared against the baseline for the dirty flag.
function serialize(obj: any): string {
  return JSON.stringify(obj, null, 2)
}

interface ConfigState {
  path: string
  draft: any
  baseline: string // serialize() of the last loaded/saved draft
  loading: boolean
  saving: boolean
  error: string
  dirty: boolean
  load: () => Promise<void>
  update: (path: string, value: any) => void
  save: () => Promise<void>
  reset: () => void
}

export const useConfigStore = create<ConfigState>((set, get) => ({
  path: '',
  draft: null,
  baseline: '',
  loading: false,
  saving: false,
  error: '',
  dirty: false,

  load: async () => {
    set({ loading: true, error: '' })
    try {
      const [raw, path] = await Promise.all([GetConfig(), GetConfigPath()])
      const draft = JSON.parse(raw)
      set({ path, draft, baseline: serialize(draft), dirty: false, loading: false })
    } catch (err: any) {
      // Fail loud: surface the reason instead of showing an empty form.
      set({ error: String(err?.message ?? err), loading: false })
    }
  },

  update: (path, value) => {
    const draft = setPath(get().draft, path, value)
    set({ draft, dirty: serialize(draft) !== get().baseline })
  },

  save: async () => {
    const { draft, baseline } = get()
    set({ saving: true, error: '' })
    try {
      const raw = serialize(draft)
      await SaveConfig(raw)
      set({ baseline: raw, dirty: false, saving: false })
    } catch (err: any) {
      set({ error: String(err?.message ?? err), saving: false })
      throw err // let the modal keep itself open and show the failure
    }
    void baseline
  },

  reset: () => {
    const { baseline } = get()
    if (!baseline) return
    set({ draft: JSON.parse(baseline), dirty: false })
  },
}))
```

- [ ] **Step 4: 跑测试确认通过**

Run: `cd legion/legionAgentGUI/frontend && npx vitest run src/stores/configStore.test.ts`
Expected: PASS（四个用例）。

---

### Task 9: 字段控件组件 `components/settings/fields/`

**Files:**
- Create: `legion/legionAgentGUI/frontend/src/components/settings/fields/FieldRow.tsx`
- Create: `legion/legionAgentGUI/frontend/src/components/settings/fields/SecretField.tsx`
- Create: `legion/legionAgentGUI/frontend/src/components/settings/fields/ProfilesEditor.tsx`
- Create: `legion/legionAgentGUI/frontend/src/components/settings/fields/StringListField.tsx`
- Create: `legion/legionAgentGUI/frontend/src/components/settings/fields/FieldRenderer.tsx`

**Interfaces:**
- Consumes: `useConfigStore`（`update`/`draft`）、`getPath`、`FieldSpec`。
- Produces: `FieldRenderer({ field }: { field: FieldSpec })` —— 按 `field.widget` 分派渲染，内部用 store 读写。

> 无 RTL/jsdom，故本任务不写单测；验证 = `tsc` 类型检查 + 构建（Task 12），交互在手动 e2e 核对。

- [ ] **Step 1: `FieldRow.tsx`（标签 + 控件容器 + 基础 toggle/number/text/readonly）**

```tsx
import type { ReactNode } from 'react'

// FieldRow lays out a label beside its control in the settings form.
export function FieldRow({ label, children }: { label: string; children: ReactNode }) {
  return (
    <div className="flex items-center gap-3 py-1">
      <label className="text-xs text-muted-foreground w-48 shrink-0 truncate">{label}</label>
      <div className="flex-1 min-w-0">{children}</div>
    </div>
  )
}

export function ToggleControl({ value, onChange }: { value: boolean; onChange: (v: boolean) => void }) {
  return (
    <input type="checkbox" checked={!!value} onChange={(e) => onChange(e.target.checked)} />
  )
}

export function NumberControl({ value, onChange }: { value: number; onChange: (v: number) => void }) {
  return (
    <input
      type="number"
      className="text-xs px-2 py-1 rounded border border-input bg-background w-full"
      value={Number.isFinite(value) ? value : ''}
      onChange={(e) => {
        const n = Number(e.target.value)
        if (!Number.isNaN(n)) onChange(n)
      }}
    />
  )
}

export function TextControl({ value, onChange }: { value: string; onChange: (v: string) => void }) {
  return (
    <input
      type="text"
      className="text-xs px-2 py-1 rounded border border-input bg-background w-full"
      value={value ?? ''}
      onChange={(e) => onChange(e.target.value)}
    />
  )
}

export function ReadonlyControl({ value }: { value: any }) {
  const text = typeof value === 'object' && value !== null ? JSON.stringify(value) : String(value ?? '')
  return (
    <span
      className="text-xs px-2 py-1 rounded border border-input bg-muted text-muted-foreground w-full inline-block truncate"
      title={text}
    >
      {text || '—'}
    </span>
  )
}
```

- [ ] **Step 2: `SecretField.tsx`（打码 + reveal）**

```tsx
import { useState } from 'react'

// SecretField masks a secret value with a reveal toggle. It never truncates the
// stored value — only the on-screen rendering is masked.
export function SecretField({ value, onChange }: { value: string; onChange: (v: string) => void }) {
  const [shown, setShown] = useState(false)
  return (
    <div className="flex gap-1">
      <input
        type={shown ? 'text' : 'password'}
        className="text-xs px-2 py-1 rounded border border-input bg-background w-full"
        value={value ?? ''}
        onChange={(e) => onChange(e.target.value)}
      />
      <button
        type="button"
        className="text-xs px-2 rounded hover:bg-muted text-muted-foreground"
        onClick={() => setShown((v) => !v)}
        title={shown ? '隐藏' : '显示'}
      >
        {shown ? '🙈' : '👁'}
      </button>
    </div>
  )
}
```

- [ ] **Step 3: `StringListField.tsx`（字符串数组增删）**

```tsx
// StringListField edits a string[] as one comma-free item per row.
export function StringListField({ value, onChange }: { value: string[]; onChange: (v: string[]) => void }) {
  const list = Array.isArray(value) ? value : []
  const setAt = (i: number, v: string) => onChange(list.map((x, j) => (j === i ? v : x)))
  const remove = (i: number) => onChange(list.filter((_, j) => j !== i))
  const add = () => onChange([...list, ''])
  return (
    <div className="flex flex-col gap-1">
      {list.map((item, i) => (
        <div key={i} className="flex gap-1">
          <input
            className="text-xs px-2 py-1 rounded border border-input bg-background w-full"
            value={item}
            onChange={(e) => setAt(i, e.target.value)}
          />
          <button type="button" className="text-xs px-2 rounded hover:bg-muted text-muted-foreground" onClick={() => remove(i)}>
            ✕
          </button>
        </div>
      ))}
      <button type="button" className="text-xs px-2 py-1 rounded border border-input hover:bg-muted text-left" onClick={add}>
        + 添加
      </button>
    </div>
  )
}
```

- [ ] **Step 4: `ProfilesEditor.tsx`（maas.profiles map 增删改）**

```tsx
interface Profile {
  model?: string
  base_url?: string
  api_key?: string
  prompt_cache?: boolean
}

// ProfilesEditor edits the maas.profiles map: add/remove named profiles and edit
// each profile's model/base_url/api_key.
export function ProfilesEditor({
  value,
  onChange,
}: {
  value: Record<string, Profile>
  onChange: (v: Record<string, Profile>) => void
}) {
  const profiles = value && typeof value === 'object' ? value : {}
  const names = Object.keys(profiles)

  const setField = (name: string, key: keyof Profile, v: string) =>
    onChange({ ...profiles, [name]: { ...profiles[name], [key]: v } })
  const remove = (name: string) => {
    const next = { ...profiles }
    delete next[name]
    onChange(next)
  }
  const add = () => {
    let n = 'new-profile'
    let i = 1
    while (profiles[n]) n = `new-profile-${i++}`
    onChange({ ...profiles, [n]: { model: '', base_url: '', api_key: '' } })
  }

  return (
    <div className="flex flex-col gap-2">
      {names.map((name) => (
        <div key={name} className="border border-border rounded p-2 flex flex-col gap-1">
          <div className="flex items-center justify-between">
            <span className="text-xs font-semibold">{name}</span>
            <button type="button" className="text-xs px-2 rounded hover:bg-muted text-muted-foreground" onClick={() => remove(name)}>
              删除
            </button>
          </div>
          {(['model', 'base_url', 'api_key'] as const).map((k) => (
            <div key={k} className="flex items-center gap-2">
              <label className="text-[10px] uppercase text-muted-foreground w-16 shrink-0">{k}</label>
              <input
                type={k === 'api_key' ? 'password' : 'text'}
                className="text-xs px-2 py-1 rounded border border-input bg-background w-full"
                value={profiles[name]?.[k] ?? ''}
                onChange={(e) => setField(name, k, e.target.value)}
              />
            </div>
          ))}
        </div>
      ))}
      <button type="button" className="text-xs px-2 py-1 rounded border border-input hover:bg-muted text-left" onClick={add}>
        + 添加 profile
      </button>
    </div>
  )
}
```

- [ ] **Step 5: `FieldRenderer.tsx`（按 widget 分派，接 store）**

```tsx
import type { FieldSpec } from '../../../types/config'
import { useConfigStore } from '../../../stores/configStore'
import { getPath } from '../../../lib/objectPath'
import { FieldRow, ToggleControl, NumberControl, TextControl, ReadonlyControl } from './FieldRow'
import { SecretField } from './SecretField'
import { StringListField } from './StringListField'
import { ProfilesEditor } from './ProfilesEditor'

// FieldRenderer reads the field's current value from the config draft and
// renders the control for its widget, writing edits back through the store.
export function FieldRenderer({ field }: { field: FieldSpec }) {
  const draft = useConfigStore((s) => s.draft)
  const update = useConfigStore((s) => s.update)
  const value = getPath(draft, field.path)
  const set = (v: any) => update(field.path, v)

  switch (field.widget) {
    case 'toggle':
      return <FieldRow label={field.label}><ToggleControl value={value} onChange={set} /></FieldRow>
    case 'number':
      return <FieldRow label={field.label}><NumberControl value={value} onChange={set} /></FieldRow>
    case 'text':
    case 'color':
      return (
        <FieldRow label={field.label}>
          <div className="flex items-center gap-2">
            <TextControl value={value} onChange={set} />
            {field.widget === 'color' && (
              <span className="w-4 h-4 rounded border border-border shrink-0" style={{ background: value || 'transparent' }} />
            )}
          </div>
        </FieldRow>
      )
    case 'secret':
      return <FieldRow label={field.label}><SecretField value={value} onChange={set} /></FieldRow>
    case 'stringlist':
      return <FieldRow label={field.label}><StringListField value={value} onChange={set} /></FieldRow>
    case 'profiles':
      return <FieldRow label={field.label}><ProfilesEditor value={value} onChange={set} /></FieldRow>
    case 'readonly':
      return <FieldRow label={field.label}><ReadonlyControl value={value} /></FieldRow>
    default:
      // Fail loud: an unhandled widget is a programming error, not something to
      // silently skip.
      throw new Error(`FieldRenderer: unhandled widget "${field.widget}" for ${field.path}`)
  }
}
```

- [ ] **Step 6: 验证关卡（类型检查）**

Run: `cd legion/legionAgentGUI/frontend && npx tsc --noEmit`
Expected: 无类型错误。

---

### Task 10: `SettingsModal`

**Files:**
- Create: `legion/legionAgentGUI/frontend/src/components/settings/SettingsModal.tsx`

**Interfaces:**
- Consumes: `useConfigStore`、`CONFIG_SECTIONS`、`FieldRenderer`、`ListTasks`（`../../wailsjs/go/main/App`，用于保存前守卫）。
- Produces: `SettingsModal({ open, onClose }: { open: boolean; onClose: () => void })`。

- [ ] **Step 1: 实现**

`legion/legionAgentGUI/frontend/src/components/settings/SettingsModal.tsx`：
```tsx
import { useEffect, useState } from 'react'
import { useConfigStore } from '../../stores/configStore'
import { CONFIG_SECTIONS, type SectionSpec } from '../../types/config'
import { FieldRenderer } from './fields/FieldRenderer'
import { ListTasks } from '../../wailsjs/go/main/App'

// activeTaskCount returns how many tracked tasks are still in a non-terminal
// state, so save can warn that a serve restart will interrupt them.
async function activeTaskCount(): Promise<number> {
  try {
    const tasks = (await ListTasks()) || []
    const done = new Set(['done', 'cancelled', 'failed', 'completed'])
    return tasks.filter((t: any) => !done.has(String(t?.status ?? '').toLowerCase())).length
  } catch {
    return 0 // if the service is unreachable there is nothing running to interrupt
  }
}

function Section({ section }: { section: SectionSpec }) {
  const [open, setOpen] = useState(!section.advanced)
  return (
    <div className="border-b border-border py-2">
      <button className="w-full text-left text-sm font-semibold flex items-center gap-1" onClick={() => setOpen((v) => !v)}>
        <span>{open ? '▾' : '▸'}</span>
        <span>{section.title}</span>
      </button>
      {open && (
        <div className="pl-4 pt-1">
          <p className="text-[11px] text-muted-foreground mb-1">{section.help}</p>
          {section.fields.map((f) => (
            <FieldRenderer key={f.path} field={f} />
          ))}
        </div>
      )}
    </div>
  )
}

export function SettingsModal({ open, onClose }: { open: boolean; onClose: () => void }) {
  const { path, draft, dirty, saving, error, load, save } = useConfigStore()

  useEffect(() => {
    if (open) load()
    // eslint-disable-next-line react-hooks/exhaustive-deps
  }, [open])

  if (!open) return null

  async function onSave() {
    const n = await activeTaskCount()
    if (n > 0 && !window.confirm(`有 ${n} 个进行中的任务。保存会重启内嵌服务并中断它们，继续？`)) {
      return
    }
    try {
      await save()
      onClose()
    } catch {
      // store already recorded the error; keep the modal open to show it.
    }
  }

  return (
    <div className="fixed inset-0 z-50 flex items-center justify-center bg-black/40" onClick={onClose}>
      <div
        className="bg-background border border-border rounded-lg shadow-xl w-[720px] max-h-[80vh] flex flex-col"
        onClick={(e) => e.stopPropagation()}
      >
        <div className="flex items-center justify-between px-4 py-2 border-b border-border">
          <div className="flex flex-col">
            <span className="text-sm font-semibold">设置 · Agent 配置</span>
            <span className="text-[10px] text-muted-foreground truncate max-w-[560px]" title={path}>{path}</span>
          </div>
          <button className="text-muted-foreground hover:text-foreground" onClick={onClose}>✕</button>
        </div>

        <div className="flex-1 overflow-y-auto px-4">
          {!draft && !error && <p className="text-xs text-muted-foreground py-4">加载中…</p>}
          {draft && CONFIG_SECTIONS.map((s) => <Section key={s.key} section={s} />)}
        </div>

        {error && <p className="text-xs text-destructive px-4 py-1 break-all">保存/加载失败：{error}</p>}

        <div className="flex items-center justify-end gap-2 px-4 py-2 border-t border-border">
          <button className="text-xs px-3 py-1 rounded hover:bg-muted text-muted-foreground" onClick={onClose}>取消</button>
          <button
            className="text-xs px-3 py-1 rounded bg-primary text-primary-foreground disabled:opacity-50"
            disabled={!dirty || saving}
            onClick={onSave}
          >
            {saving ? '保存中…' : '保存并重启'}
          </button>
        </div>
      </div>
    </div>
  )
}
```

- [ ] **Step 2: 验证关卡**

Run: `cd legion/legionAgentGUI/frontend && npx tsc --noEmit`
Expected: 无类型错误。
> 若 `ListTasks` 类型缺失，确认 Task 5 的绑定已生成（它是既有绑定，应已存在）。

---

### Task 11: Sidebar 底部齿轮入口 + App 挂载模态

**Files:**
- Modify: `legion/legionAgentGUI/frontend/src/components/Sidebar.tsx`
- Modify: `legion/legionAgentGUI/frontend/src/App.tsx`
- Create: `legion/legionAgentGUI/frontend/src/stores/uiStore.ts`

**Interfaces:**
- Produces: `uiStore` —— `settingsOpen: boolean`、`openSettings()`、`closeSettings()`（用小 store 避免 Sidebar↔App 的 prop drilling）。

- [ ] **Step 1: 建 `uiStore.ts`**

`legion/legionAgentGUI/frontend/src/stores/uiStore.ts`：
```ts
import { create } from 'zustand'

// uiStore holds cross-panel UI flags. settingsOpen drives the settings modal,
// toggled from the sidebar gear and consumed by App.
interface UIState {
  settingsOpen: boolean
  openSettings: () => void
  closeSettings: () => void
}

export const useUIStore = create<UIState>((set) => ({
  settingsOpen: false,
  openSettings: () => set({ settingsOpen: true }),
  closeSettings: () => set({ settingsOpen: false }),
}))
```

- [ ] **Step 2: Sidebar 底部加齿轮按钮**

在 `Sidebar.tsx` 顶部 import 处加：
```tsx
import { useUIStore } from '../stores/uiStore'
```
把最外层 `return (` 的容器改为「上部内容可滚动 + 底部固定齿轮」。将现有根节点
`<div className="p-2 flex flex-col gap-2">` 改为 `<div className="h-full flex flex-col">`，
把原有内容包进一个 `<div className="p-2 flex flex-col gap-2 flex-1 overflow-y-auto">`，
并在其后、根 `</div>` 之前插入底部齿轮栏：
```tsx
      {/* Bottom settings entry */}
      <div className="border-t border-border p-2">
        <button
          className="w-full text-left text-xs px-2 py-1 rounded hover:bg-muted text-muted-foreground flex items-center gap-2"
          onClick={() => useUIStore.getState().openSettings()}
        >
          <span>⚙</span>
          <span>设置</span>
        </button>
      </div>
```
（即：根容器变纵向 flex 满高，原树放进可滚动区，齿轮栏固定底部。`menu` 的 `ContextMenu` 仍留在原可滚动内容内即可。）

- [ ] **Step 3: App 挂载模态**

`App.tsx` 改为：
```tsx
import { ThreePanelLayout } from './components/layout/ThreePanelLayout'
import { Sidebar } from './components/Sidebar'
import { ChatPanel } from './components/ChatPanel'
import { StatusPanel } from './components/StatusPanel'
import { ConnectionBadge } from './components/ConnectionBadge'
import { SettingsModal } from './components/settings/SettingsModal'
import { useUIStore } from './stores/uiStore'

function App() {
  const settingsOpen = useUIStore((s) => s.settingsOpen)
  const closeSettings = useUIStore((s) => s.closeSettings)
  return (
    <div className="flex flex-col h-screen">
      <div className="flex items-center justify-end border-b border-border px-2 py-0.5 bg-background">
        <ConnectionBadge />
      </div>
      <div className="flex-1 min-h-0">
        <ThreePanelLayout
          sidebar={<Sidebar />}
          chat={<ChatPanel />}
          status={<StatusPanel />}
        />
      </div>
      <SettingsModal open={settingsOpen} onClose={closeSettings} />
    </div>
  )
}

export default App
```

- [ ] **Step 4: 验证关卡**

Run: `cd legion/legionAgentGUI/frontend && npx tsc --noEmit && npm run test`
Expected: 类型检查通过；vitest 全绿（objectPath / config / configStore 测试）。

---

### Task 12: 整体构建 + 手动 e2e 核对

**Files:** 无（集成验证）。

- [ ] **Step 1: 前端构建**

Run: `cd legion/legionAgentGUI/frontend && npm run build`
Expected: `tsc` + `vite build` 成功，无错误。

- [ ] **Step 2: Go 全量校验（GUI module）**

Run: `cd legion/legionAgentGUI && go build ./... && go vet ./... && go test ./... && gofmt -l .`
Expected: 测试全绿（e2e SKIP），gofmt 无输出。

- [ ] **Step 3: Go 全量校验（legionAgent module）**

Run: `cd legion/legionAgent && go build ./... && go vet ./... && go test ./serve/ && gofmt -l serve/`
Expected: 通过，gofmt 无输出。

- [ ] **Step 4: 可选 serve 重启 e2e**

Run: `cd legion/legionAgentGUI && LEGION_E2E=1 go test ./ -run TestServeManagerRestart -v`
Expected: PASS（端口从 p1 变 p2，仍 running）。

- [ ] **Step 5: 手动 e2e 核对（交互启动 GUI）**

由用户交互启动 GUI（非后台，见 [[legion-gui-wails-gotchas]]），核对：
1. Sidebar 底部有 ⚙ 设置入口，点击弹出模态，顶部显示当前 `agent.json` 绝对路径。
2. 常用段（模型/运行时/会话/主题/上下文）默认展开；高级段折叠可展开。
3. 改 `maas.default_profile` 或某 profile 的 `model` → 「保存并重启」；
   `serve-startup.log` 出现新端口、ConnectionBadge 重新连上；聊天用到新模型。
4. 危险路径字段（storage.path、各 *_path、tasks.*_path、agents）灰显不可编辑。
5. 密钥字段默认打码，👁 可显隐。
6. 存在进行中任务时保存 → 弹确认框。
7. 故意把某数字字段写非法（前端阻止）或构造非法 JSON 落到 SaveConfig → 报错提示、
   原 `agent.json` 不变、生成 `agent.json.bak`。

---

## Self-Review

**1. Spec coverage**
- 组件 1（GetConfig/GetConfigPath/SaveConfig）→ Task 3、4。
- 组件 2（ServeManager.Restart）→ Task 2。
- 组件 0（serve.ValidateConfig 桥）→ Task 1。
- 3.3 字段分区/只读白名单/密钥 → Task 7 元数据 + Task 9 控件。
- 3.4 前端（types/store/modal/fields/入口）→ Task 6–11。
- 3.5 保存前 active-task 守卫 → Task 10 `activeTaskCount`。
- 第 4 节 Fail-Loud → Task 1/4 校验+备份+不写盘、Task 9 unhandled widget throw、store 记录 error。
- 第 5 节测试 → Task 1/2/3/4（Go）、Task 6/7/8（前端）。
- 第 6 节非目标（不做 listen_addr 生效/目录选择器/多备份）→ 已遵守（listen_addr 只读标注、路径只读、单份 .bak）。

**2. Placeholder scan**：无 TBD/TODO；每个代码步骤含完整代码；命令含预期输出。

**3. Type consistency**
- `ServeManager.emit`、`Restart` 签名 Task 2 定义，Task 4 `a.serve.emit`/`a.serve.Restart` 一致。
- `serve.ValidateConfig(ctx, path)` Task 1 定义，Task 4 一致调用。
- `getPath`/`setPath` Task 6 定义，Task 8/9 一致使用。
- `FieldWidget`/`FieldSpec`/`SectionSpec`/`CONFIG_SECTIONS` Task 7 定义，Task 8 测试与 Task 9/10 一致消费。
- `useConfigStore` action 名（load/update/save/reset）Task 8 定义，Task 9/10 一致。
- `GetConfig/GetConfigPath/SaveConfig` Task 3/4 产出、Task 5 生成绑定、Task 8/10 消费，签名一致。

## Execution Handoff

见文末交接选项。
