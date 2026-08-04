# Agent 内置浏览器 Phase 1（最小闭环）Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 给 legionAgent 加内置浏览器最小闭环——`browser_open/read/click/type` 四工具 + a11y 语义树观测 + 单进程多 incognito Context + 工具注册 + toolauth 门控，开发本地模式端到端跑通。

**Architecture:** 新包 `internal/browser`（平台无关运行时核心：RuntimeAPI/Session/Manager/Observation/errors），薄工具层 `internal/tool/browser.go` 照现有 `web.go` 的 `RegisterWebTools` 模式把四工具挂上 `tool.Registry`。工具 handler 只调 `browser.RuntimeAPI` 接口，核心用 go-rod 驱动 Chromium。Observation Engine 与错误码是纯函数/纯数据，优先 TDD；go-rod 集成层用 build-tag 隔离的集成测试（需真实 Chromium）。

**Tech Stack:** Go 1.26、go-rod（CDP 驱动，本 plan 新引入）、现有 `internal/tool` + `internal/domain` + `internal/toolauth`。

**范围边界（本 plan 不做，留后续 Phase）:** SSE 观测流（Phase 2）、storageState 持久化/TTL 回收（Phase 3）、Wails/loopback/PAL/跨平台（Phase 4）、SSRF/沙箱完整安全基线（Phase 5，本 plan 只做协议白名单 + 私网基础拦截）、scroll/back/forward/screenshot/extract/download（Phase 1 后按需补，本 plan 只交付 open/read/click/type + close）。

**关联文档:** spec = `docs/superpowers/specs/2026-08-04-agent-browser-design.md`；PRD = `docs/design/architecture/agent-browser-prd.md`。

---

## 文件结构

| 文件 | 职责 | 本 plan 状态 |
|---|---|---|
| `internal/browser/errors.go` | 语义错误码 + `BrowserError` 类型 | 创建（Task 1） |
| `internal/browser/observation.go` | a11y 节点 → 稳定 ref + 预算裁剪（纯函数） | 创建（Task 2） |
| `internal/browser/api.go` | `RuntimeAPI` 接口 + 请求/响应类型 | 创建（Task 3） |
| `internal/browser/manager.go` | Browser Manager：进程 + Context 池（单进程） | 创建（Task 4） |
| `internal/browser/session.go` | Session Manager：CRUD + 会话内串行锁 | 创建（Task 5） |
| `internal/browser/runtime.go` | `RuntimeAPI` 实现：go-rod 驱动 Open/Read/Click/Type/Close | 创建（Task 6） |
| `internal/tool/browser.go` | 四工具 Descriptor + Handler（照 `web.go`） | 创建（Task 7） |
| `internal/toolauth/catalog.go` | gateable 追加五工具 | 修改（Task 8） |
| `internal/runtime/toolauth_drift_test.go` | drift helper 追加 `RegisterBrowserTools` | 修改（Task 8） |
| `internal/app/app.go` · `internal/cli/command.go` · `internal/runtime/agent_resolver.go` | 生产注册点接线 + config 字段 | 修改（Task 9） |
| `internal/browser/e2e_test.go` | 端到端：open→read→type→click（build tag `chromium`） | 创建（Task 10） |

---

## Task 0: 引入 go-rod 依赖 + 冒烟

**Files:**
- Modify: `legion/legionAgent/go.mod`
- Test: `legion/legionAgent/internal/browser/smoke_test.go`（临时，Task 6 后可删）

- [ ] **Step 1: 加依赖**

在 `legion/legionAgent` 目录运行：

```bash
go get github.com/go-rod/rod@latest
```

Expected: `go.mod` 新增 `github.com/go-rod/rod` 与其间接依赖 `github.com/ysmood/...`。

- [ ] **Step 2: 写编译冒烟测试（不连 Chromium）**

Create `legion/legionAgent/internal/browser/smoke_test.go`:

```go
package browser

import (
	"testing"

	"github.com/go-rod/rod/lib/launcher"
)

// TestGoRodImportable 只验证 go-rod 可编译链接，不启动浏览器。
func TestGoRodImportable(t *testing.T) {
	l := launcher.New()
	if l == nil {
		t.Fatal("launcher.New() returned nil")
	}
}
```

- [ ] **Step 3: 跑冒烟**

Run: `go test ./internal/browser/ -run TestGoRodImportable -v`
Expected: PASS（首次会下载/解析 go-rod 源码，不启动 Chromium）。

- [ ] **Step 4: Commit**

```bash
git add go.mod go.sum internal/browser/smoke_test.go
git commit -m "chore(browser): add go-rod dependency"
```

---

## Task 1: 语义错误码

**Files:**
- Create: `internal/browser/errors.go`
- Test: `internal/browser/errors_test.go`

- [ ] **Step 1: 写失败测试**

Create `internal/browser/errors_test.go`:

```go
package browser

import (
	"errors"
	"testing"
)

func TestBrowserErrorCodeAndMessage(t *testing.T) {
	err := NewBrowserError(CodeElementNotFound, "ref e12 not found")
	var be *BrowserError
	if !errors.As(err, &be) {
		t.Fatalf("expected *BrowserError, got %T", err)
	}
	if be.Code != CodeElementNotFound {
		t.Fatalf("code = %q, want %q", be.Code, CodeElementNotFound)
	}
	if be.Error() != "ELEMENT_NOT_FOUND: ref e12 not found" {
		t.Fatalf("Error() = %q", be.Error())
	}
}
```

- [ ] **Step 2: 跑，确认失败**

Run: `go test ./internal/browser/ -run TestBrowserErrorCodeAndMessage -v`
Expected: FAIL（`NewBrowserError` / `BrowserError` / `CodeElementNotFound` 未定义）。

- [ ] **Step 3: 实现**

Create `internal/browser/errors.go`:

```go
package browser

import "fmt"

// Code 是回给 Agent 的可自恢复语义错误码（见 spec §5.1）。
type Code string

const (
	CodeElementNotFound   Code = "ELEMENT_NOT_FOUND"   // ref 失效，建议重新 read
	CodeNavigationTimeout Code = "NAVIGATION_TIMEOUT"  // 导航超时，建议重试或换策略
	CodeBlockedByCaptcha  Code = "BLOCKED_BY_CAPTCHA"  // 遇验证码
	CodeDownloadTooLarge  Code = "DOWNLOAD_TOO_LARGE"  // 超文件上限
	CodeContextEvicted    Code = "CONTEXT_EVICTED"     // Context 被回收，需重建 Session
	CodeProtocolBlocked   Code = "PROTOCOL_BLOCKED"    // 危险协议被拦（file://、chrome://、data:）
	CodePrivateHostBlocked Code = "PRIVATE_HOST_BLOCKED" // 私网地址被 SSRF 拦截
)

// BrowserError 携带语义码，供工具层映射到 domain.ToolResult.Error。
type BrowserError struct {
	Code Code
	Msg  string
}

func (e *BrowserError) Error() string {
	return fmt.Sprintf("%s: %s", e.Code, e.Msg)
}

// NewBrowserError 构造一个带语义码的错误。
func NewBrowserError(code Code, msg string) *BrowserError {
	return &BrowserError{Code: code, Msg: msg}
}
```

- [ ] **Step 4: 跑，确认通过**

Run: `go test ./internal/browser/ -run TestBrowserErrorCodeAndMessage -v`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add internal/browser/errors.go internal/browser/errors_test.go
git commit -m "feat(browser): semantic error codes"
```

---

## Task 2: Observation Engine（a11y → 稳定 ref + 预算裁剪，纯函数）

这是成功率与 token 成本的核心，也是最可 TDD 的部分——纯函数，不碰 Chromium。

**Files:**
- Create: `internal/browser/observation.go`
- Test: `internal/browser/observation_test.go`

- [ ] **Step 1: 写失败测试**

Create `internal/browser/observation_test.go`:

```go
package browser

import (
	"strings"
	"testing"
)

// rawA11yNode 是从 CDP Accessibility 域抽出的中间结构（Task 6 由 go-rod 填充）。
func node(role, name string, interactive, visible bool) RawA11yNode {
	return RawA11yNode{Role: role, Name: name, Interactive: interactive, Visible: visible}
}

func TestBuildObservationKeepsInteractiveVisibleAssignsStableRef(t *testing.T) {
	raw := []RawA11yNode{
		node("button", "搜索", true, true),
		node("StaticText", "无关文本", false, true),     // 非交互 → 丢弃
		node("link", "隐藏链接", true, false),           // 不可见 → 丢弃
		node("textbox", "关键词框", true, true),
	}
	obs := BuildObservation(raw, ObservationBudget{MaxElements: 50})

	if len(obs.Elements) != 2 {
		t.Fatalf("kept %d elements, want 2 (interactive+visible only)", len(obs.Elements))
	}
	if obs.Elements[0].Ref != "e1" || obs.Elements[1].Ref != "e2" {
		t.Fatalf("refs = %q,%q, want e1,e2", obs.Elements[0].Ref, obs.Elements[1].Ref)
	}
	if !strings.Contains(obs.Text, "[e1]") || !strings.Contains(obs.Text, "搜索") {
		t.Fatalf("render missing ref/name: %q", obs.Text)
	}
	if obs.Truncated {
		t.Fatalf("should not be truncated under budget")
	}
}

func TestBuildObservationTruncatesAtMaxElements(t *testing.T) {
	var raw []RawA11yNode
	for i := 0; i < 10; i++ {
		raw = append(raw, node("button", "b", true, true))
	}
	obs := BuildObservation(raw, ObservationBudget{MaxElements: 3})
	if len(obs.Elements) != 3 {
		t.Fatalf("kept %d, want 3", len(obs.Elements))
	}
	if !obs.Truncated {
		t.Fatalf("expected Truncated=true when clipped")
	}
}
```

- [ ] **Step 2: 跑，确认失败**

Run: `go test ./internal/browser/ -run TestBuildObservation -v`
Expected: FAIL（`RawA11yNode`/`BuildObservation`/`ObservationBudget`/`Observation` 未定义）。

- [ ] **Step 3: 实现**

Create `internal/browser/observation.go`:

```go
package browser

import (
	"fmt"
	"strings"
)

// RawA11yNode 是从 CDP Accessibility 域抽出、尚未裁剪的可访问性节点。
type RawA11yNode struct {
	Role        string
	Name        string
	Value       string
	Interactive bool
	Visible     bool
}

// Element 是裁剪后、分配了会话内稳定 ref 的可交互元素。
type Element struct {
	Ref   string `json:"ref"`  // 会话内稳定，如 "e1"
	Role  string `json:"role"`
	Name  string `json:"name"`
	Value string `json:"value,omitempty"`
}

// Observation 是回给 Agent 的页面表示（默认 a11y 树）。
type Observation struct {
	Elements  []Element `json:"elements"`
	Text      string    `json:"text"`      // 渲染成 [e1] <button> 搜索 的可读文本
	Truncated bool      `json:"truncated"` // 是否因预算裁剪
}

// ObservationBudget 控制裁剪预算。
type ObservationBudget struct {
	MaxElements int
}

const defaultMaxElements = 100

// BuildObservation 把原始 a11y 节点裁剪为 token 可控的观测：
// 只保留可交互且可见的节点，按顺序分配稳定 ref（e1、e2…），超预算截断。
// 纯函数，不碰 Chromium——ref 的会话内稳定性由调用方保证输入顺序稳定。
func BuildObservation(raw []RawA11yNode, budget ObservationBudget) Observation {
	max := budget.MaxElements
	if max <= 0 {
		max = defaultMaxElements
	}
	var obs Observation
	for _, n := range raw {
		if !n.Interactive || !n.Visible {
			continue
		}
		if len(obs.Elements) >= max {
			obs.Truncated = true
			break
		}
		obs.Elements = append(obs.Elements, Element{
			Ref:   fmt.Sprintf("e%d", len(obs.Elements)+1),
			Role:  n.Role,
			Name:  n.Name,
			Value: n.Value,
		})
	}
	obs.Text = renderObservation(obs.Elements)
	return obs
}

func renderObservation(elems []Element) string {
	var b strings.Builder
	for _, e := range elems {
		fmt.Fprintf(&b, "[%s] <%s> %s", e.Ref, e.Role, e.Name)
		if e.Value != "" {
			fmt.Fprintf(&b, " (value=%q)", e.Value)
		}
		b.WriteByte('\n')
	}
	return b.String()
}
```

- [ ] **Step 4: 跑，确认通过**

Run: `go test ./internal/browser/ -run TestBuildObservation -v`
Expected: PASS（两个子测试都过）。

- [ ] **Step 5: Commit**

```bash
git add internal/browser/observation.go internal/browser/observation_test.go
git commit -m "feat(browser): a11y observation with stable refs and budget trimming"
```

---

## Task 3: RuntimeAPI 接口 + 请求/响应类型

定义后端核心对外唯一契约。本 Task 只定义类型，不实现——工具层（Task 7）先对着接口写测试用假实现。

**Files:**
- Create: `internal/browser/api.go`
- Test: `internal/browser/api_test.go`

- [ ] **Step 1: 写失败测试（编译级契约测试）**

Create `internal/browser/api_test.go`:

```go
package browser

import (
	"context"
	"testing"
)

// fakeRuntime 是一个满足 RuntimeAPI 的空实现，仅用于锁定接口签名。
type fakeRuntime struct{}

func (fakeRuntime) Open(context.Context, OpenReq) (Observation, error)  { return Observation{}, nil }
func (fakeRuntime) Read(context.Context, ReadReq) (Observation, error)  { return Observation{}, nil }
func (fakeRuntime) Click(context.Context, ClickReq) (Observation, error) { return Observation{}, nil }
func (fakeRuntime) Type(context.Context, TypeReq) (Observation, error)  { return Observation{}, nil }
func (fakeRuntime) Close(context.Context, CloseReq) error               { return nil }

func TestRuntimeAPISatisfied(t *testing.T) {
	var _ RuntimeAPI = fakeRuntime{}
	// 验证请求类型带必要字段
	_ = OpenReq{URL: "https://x", SessionID: ""}
	_ = ReadReq{SessionID: "s1"}
	_ = ClickReq{SessionID: "s1", Ref: "e1"}
	_ = TypeReq{SessionID: "s1", Ref: "e1", Text: "hi", Submit: true}
	_ = CloseReq{SessionID: "s1"}
}
```

- [ ] **Step 2: 跑，确认失败**

Run: `go test ./internal/browser/ -run TestRuntimeAPISatisfied -v`
Expected: FAIL（`RuntimeAPI`/`OpenReq` 等未定义）。

- [ ] **Step 3: 实现**

Create `internal/browser/api.go`:

```go
package browser

import "context"

// OpenResult 打开页面的返回，含新建/复用的 session 与首次观测。
type OpenReq struct {
	URL       string
	SessionID string // 空则新建 Session
	TaskID    string // 新建时绑定
}

type ReadReq struct {
	SessionID string
	Mode      string // 空=默认 a11y 树；本 Phase 只实现 a11y
}

type ClickReq struct {
	SessionID string
	Ref       string
}

type TypeReq struct {
	SessionID string
	Ref       string
	Text      string
	Submit    bool // true 则输入后回车提交
}

type CloseReq struct {
	SessionID string
}

// OpenObservation 在 Observation 之外带上 session id（open 可能新建会话）。
type OpenObservation struct {
	SessionID   string
	Observation Observation
}

// RuntimeAPI 是后端核心对外唯一契约（spec §3.1）。工具层与传输适配器都封装它。
// 本 Phase 只含最小闭环方法；后续 Phase 追加 Scroll/Back/Screenshot/Extract/Download。
type RuntimeAPI interface {
	Open(ctx context.Context, req OpenReq) (Observation, error)
	Read(ctx context.Context, req ReadReq) (Observation, error)
	Click(ctx context.Context, req ClickReq) (Observation, error)
	Type(ctx context.Context, req TypeReq) (Observation, error)
	Close(ctx context.Context, req CloseReq) error
}
```

> 注：接口方法返回 `Observation`；open 新建的 session id 通过 `Observation` 无法回传，故 Task 6 的实现里 `Open` 在返回前把 session id 塞进 handler 可取的位置。**修正**：为让工具层拿到 session id，把 `Open` 的返回改为 `(Observation, error)` 且 session id 由工具层从 req/runtime 侧信道取——见 Step 3b。

- [ ] **Step 3b: 修正 Open 返回 session id**

把 `api.go` 的 `RuntimeAPI.Open` 签名与 fake 改为返回 `OpenObservation`：

`api.go` 中：
```go
	Open(ctx context.Context, req OpenReq) (OpenObservation, error)
```
`api_test.go` 中 fake 改为：
```go
func (fakeRuntime) Open(context.Context, OpenReq) (OpenObservation, error) { return OpenObservation{}, nil }
```

- [ ] **Step 4: 跑，确认通过**

Run: `go test ./internal/browser/ -run TestRuntimeAPISatisfied -v`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add internal/browser/api.go internal/browser/api_test.go
git commit -m "feat(browser): RuntimeAPI contract and request types"
```

---

## Task 4: Browser Manager（单进程 + Context 池）

go-rod 集成，需真实 Chromium。用 build tag `chromium` 隔离集成测试，避免无 Chromium 的 CI 失败。

**Files:**
- Create: `internal/browser/manager.go`
- Test: `internal/browser/manager_integration_test.go`（`//go:build chromium`）

- [ ] **Step 1: 写集成测试（build tag chromium）**

Create `internal/browser/manager_integration_test.go`:

```go
//go:build chromium

package browser

import (
	"testing"
)

func TestManagerAcquireReleaseContext(t *testing.T) {
	m, err := NewManager(ManagerConfig{Headless: true})
	if err != nil {
		t.Fatalf("NewManager: %v", err)
	}
	defer m.Close()

	c1, err := m.AcquireContext(ContextOpts{})
	if err != nil {
		t.Fatalf("AcquireContext: %v", err)
	}
	c2, err := m.AcquireContext(ContextOpts{})
	if err != nil {
		t.Fatalf("AcquireContext 2: %v", err)
	}
	if c1.id == c2.id {
		t.Fatalf("two contexts share id %q — not isolated", c1.id)
	}
	if err := m.ReleaseContext(c1); err != nil {
		t.Fatalf("ReleaseContext: %v", err)
	}
}
```

- [ ] **Step 2: 跑，确认失败**

Run: `go test -tags chromium ./internal/browser/ -run TestManagerAcquireReleaseContext -v`
Expected: FAIL（`NewManager`/`ManagerConfig`/`BrowserContext` 未定义）。

- [ ] **Step 3: 实现**

Create `internal/browser/manager.go`:

```go
package browser

import (
	"fmt"
	"sync"

	"github.com/go-rod/rod"
	"github.com/go-rod/rod/lib/launcher"
)

// ContextOpts 见 spec §3.4（本 Phase 只用零值；Proxy/UA/Stealth 后续 Phase）。
type ContextOpts struct {
	Proxy     string
	UserAgent string
	Stealth   bool
}

// BrowserContext 对应 go-rod 的 incognito browser（隔离上下文）。
type BrowserContext struct {
	id      string
	browser *rod.Browser // incognito browser
}

// ManagerConfig 配置进程。本 Phase 单进程。
type ManagerConfig struct {
	Headless bool
	BinPath  string // 空则由 go-rod launcher 定位/下载
}

// Manager 是单进程 + 多 incognito Context 的两级池的最小实现（spec §3.4）。
type Manager struct {
	mu       sync.Mutex
	launcher *launcher.Launcher
	browser  *rod.Browser // 一条 CDP 连接 = 一个 Chromium 进程
	seq      int
}

// NewManager 拉起一个 Chromium 进程并连接。
func NewManager(cfg ManagerConfig) (*Manager, error) {
	l := launcher.New().Headless(cfg.Headless)
	if cfg.BinPath != "" {
		l = l.Bin(cfg.BinPath)
	}
	controlURL, err := l.Launch()
	if err != nil {
		return nil, fmt.Errorf("launch chromium: %w", err)
	}
	b := rod.New().ControlURL(controlURL)
	if err := b.Connect(); err != nil {
		return nil, fmt.Errorf("connect chromium: %w", err)
	}
	return &Manager{launcher: l, browser: b}, nil
}

// AcquireContext 开一个隔离 incognito Context。本 Phase 不复用、不排队。
func (m *Manager) AcquireContext(_ ContextOpts) (*BrowserContext, error) {
	m.mu.Lock()
	defer m.mu.Unlock()
	incog, err := m.browser.Incognito()
	if err != nil {
		return nil, fmt.Errorf("create incognito context: %w", err)
	}
	m.seq++
	return &BrowserContext{id: fmt.Sprintf("ctx-%d", m.seq), browser: incog}, nil
}

// ReleaseContext 关闭 Context 的所有 page 并释放。
func (m *Manager) ReleaseContext(c *BrowserContext) error {
	if c == nil || c.browser == nil {
		return nil
	}
	pages, err := c.browser.Pages()
	if err == nil {
		for _, p := range pages {
			_ = p.Close()
		}
	}
	return nil
}

// Close 关闭浏览器进程。
func (m *Manager) Close() {
	if m.browser != nil {
		_ = m.browser.Close()
	}
	if m.launcher != nil {
		m.launcher.Cleanup()
	}
}
```

- [ ] **Step 4: 跑集成测试（本地有 Chromium 时）**

Run: `go test -tags chromium ./internal/browser/ -run TestManagerAcquireReleaseContext -v`
Expected: PASS（本地需可用 Chromium；无则 go-rod launcher 首次会下载）。
若 CI 无 Chromium：普通 `go test ./...`（不带 `-tags chromium`）应仍编译通过且跳过此测试。

- [ ] **Step 5: 确认非集成构建仍绿**

Run: `go build ./internal/browser/ && go test ./internal/browser/ -run 'TestBuildObservation|TestBrowserError|TestRuntimeAPISatisfied' -v`
Expected: PASS（无 Chromium 依赖的纯函数测试全过）。

- [ ] **Step 6: Commit**

```bash
git add internal/browser/manager.go internal/browser/manager_integration_test.go
git commit -m "feat(browser): single-process incognito context manager"
```

---

## Task 5: Session Manager（CRUD + 会话内串行锁）

**Files:**
- Create: `internal/browser/session.go`
- Test: `internal/browser/session_test.go`

- [ ] **Step 1: 写失败测试（串行性 + CRUD，不碰 Chromium）**

Create `internal/browser/session_test.go`:

```go
package browser

import (
	"sync"
	"testing"
)

func TestSessionStoreCreateGet(t *testing.T) {
	s := NewSessionStore()
	sess := s.Create("task-1")
	if sess.ID == "" {
		t.Fatal("Create returned empty ID")
	}
	got, ok := s.Get(sess.ID)
	if !ok || got.ID != sess.ID {
		t.Fatalf("Get(%q) miss", sess.ID)
	}
	s.Delete(sess.ID)
	if _, ok := s.Get(sess.ID); ok {
		t.Fatalf("expected deleted")
	}
}

// TestSessionSerialLock 验证 WithLock 对同一会话串行——并发累加无丢失即证明互斥。
func TestSessionSerialLock(t *testing.T) {
	s := NewSessionStore()
	sess := s.Create("task-1")
	var counter int
	var wg sync.WaitGroup
	for i := 0; i < 100; i++ {
		wg.Add(1)
		go func() {
			defer wg.Done()
			sess.WithLock(func() { counter++ })
		}()
	}
	wg.Wait()
	if counter != 100 {
		t.Fatalf("counter = %d, want 100 (lock not serializing)", counter)
	}
}
```

- [ ] **Step 2: 跑，确认失败**

Run: `go test ./internal/browser/ -run TestSession -v`
Expected: FAIL（`NewSessionStore`/`Session.WithLock` 未定义）。

- [ ] **Step 3: 实现**

Create `internal/browser/session.go`:

```go
package browser

import (
	"fmt"
	"sync"
	"time"
)

// Session 见 spec §3.3。本 Phase 只含最小闭环字段；StorageState/ActionLog/TTL 后续 Phase。
type Session struct {
	ID          string
	TaskID      string
	Context     *BrowserContext // 回收后为 nil
	ActivePage  *pageHandle     // 当前活跃页（Task 6 定义 pageHandle）
	Refs        map[string]string // ref → CDP backendNodeID/selector，会话内稳定
	CreatedAt   time.Time
	LastUsedAt  time.Time

	mu sync.Mutex // 会话内串行锁（spec §3.3 关键决策）
}

// WithLock 在会话串行锁下执行 fn——同 Session 动作串行，跨 Session 并行。
func (s *Session) WithLock(fn func()) {
	s.mu.Lock()
	defer s.mu.Unlock()
	fn()
}

// SessionStore 是 Session 的内存 CRUD（Phase 3 再加落盘）。
type SessionStore struct {
	mu   sync.Mutex
	seq  int
	byID map[string]*Session
}

func NewSessionStore() *SessionStore {
	return &SessionStore{byID: make(map[string]*Session)}
}

func (st *SessionStore) Create(taskID string) *Session {
	st.mu.Lock()
	defer st.mu.Unlock()
	st.seq++
	sess := &Session{
		ID:         fmt.Sprintf("sess-%d", st.seq),
		TaskID:     taskID,
		Refs:       make(map[string]string),
		CreatedAt:  time.Now(),
		LastUsedAt: time.Now(),
	}
	st.byID[sess.ID] = sess
	return sess
}

func (st *SessionStore) Get(id string) (*Session, bool) {
	st.mu.Lock()
	defer st.mu.Unlock()
	s, ok := st.byID[id]
	return s, ok
}

func (st *SessionStore) Delete(id string) {
	st.mu.Lock()
	defer st.mu.Unlock()
	delete(st.byID, id)
}
```

> `pageHandle` 在 Task 6 定义；本 Task 编译需要它存在。**做法**：Task 6 前先在 `session.go` 顶部加占位 `type pageHandle struct{}`，Task 6 用真实定义替换（同文件或 runtime.go，保持单一定义）。为避免前向依赖，把 `type pageHandle struct{}` 占位放进本 Task 的 `session.go`，Task 6 Step 3 用真实字段填充它。

- [ ] **Step 3b: 加 pageHandle 占位**

在 `session.go` 末尾加：

```go
// pageHandle 由 Task 6 填充为对 go-rod page 的封装。
type pageHandle struct {
	page interface{} // Task 6 替换为 *rod.Page
}
```

- [ ] **Step 4: 跑，确认通过**

Run: `go test ./internal/browser/ -run TestSession -v`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add internal/browser/session.go internal/browser/session_test.go
git commit -m "feat(browser): session store with per-session serial lock"
```

---

## Task 6: RuntimeAPI 实现（go-rod 驱动 Open/Read/Click/Type/Close）

组装前面所有件。含 go-rod 页面操作与 CDP a11y 抽取，集成测试走 `chromium` tag。

**Files:**
- Create: `internal/browser/runtime.go`
- Modify: `internal/browser/session.go`（用真实 `pageHandle` 替换占位）
- Test: `internal/browser/runtime_integration_test.go`（`//go:build chromium`）

- [ ] **Step 1: 先查 go-rod a11y API**

Run（了解签名，不改码）：
```bash
go doc github.com/go-rod/rod/lib/proto AccessibilityGetFullAXTree
go doc github.com/go-rod/rod Page.Navigate
go doc github.com/go-rod/rod Element.Input
```
Expected: 打印 `AccessibilityGetFullAXTree`（有 `Nodes []*AccessibilityAXNode`，节点含 `Role`/`Name`/`BackendDOMNodeID`/`Ignored`）、`Page.Navigate(url string) error`、`Element.Input(text string) error` 的签名。据此微调 Step 3 的字段名。

- [ ] **Step 2: 写集成测试**

Create `internal/browser/runtime_integration_test.go`:

```go
//go:build chromium

package browser

import (
	"context"
	"net/http"
	"net/http/httptest"
	"strings"
	"testing"
)

func newTestServer(t *testing.T) *httptest.Server {
	t.Helper()
	s := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, _ *http.Request) {
		w.Header().Set("Content-Type", "text/html; charset=utf-8")
		_, _ = w.Write([]byte(`<html><body>
			<input id="kw" aria-label="关键词框">
			<button id="go">搜索</button>
		</body></html>`))
	}))
	t.Cleanup(s.Close)
	return s
}

func TestRuntimeOpenReadType(t *testing.T) {
	rt, err := NewRuntime(RuntimeConfig{Headless: true, AllowPrivateHosts: true})
	if err != nil {
		t.Fatalf("NewRuntime: %v", err)
	}
	defer rt.Close(context.Background(), CloseReq{})

	srv := newTestServer(t)
	open, err := rt.Open(context.Background(), OpenReq{URL: srv.URL, TaskID: "t1"})
	if err != nil {
		t.Fatalf("Open: %v", err)
	}
	if open.SessionID == "" {
		t.Fatal("Open returned empty SessionID")
	}
	if !strings.Contains(open.Observation.Text, "搜索") {
		t.Fatalf("observation missing button: %q", open.Observation.Text)
	}

	// 找到关键词框 ref 并输入
	var kwRef string
	for _, e := range open.Observation.Elements {
		if strings.Contains(e.Name, "关键词") {
			kwRef = e.Ref
		}
	}
	if kwRef == "" {
		t.Fatal("no ref for keyword box")
	}
	if _, err := rt.Type(context.Background(), TypeReq{SessionID: open.SessionID, Ref: kwRef, Text: "hello"}); err != nil {
		t.Fatalf("Type: %v", err)
	}
}

func TestRuntimeOpenBlocksPrivateHost(t *testing.T) {
	rt, err := NewRuntime(RuntimeConfig{Headless: true, AllowPrivateHosts: false})
	if err != nil {
		t.Fatalf("NewRuntime: %v", err)
	}
	defer rt.Close(context.Background(), CloseReq{})
	_, err = rt.Open(context.Background(), OpenReq{URL: "http://127.0.0.1:1/", TaskID: "t1"})
	if err == nil {
		t.Fatal("expected private host to be blocked")
	}
}
```

- [ ] **Step 3: 实现 runtime.go**

Create `internal/browser/runtime.go`:

```go
package browser

import (
	"context"
	"fmt"
	"net"
	"net/url"
	"strings"

	"github.com/go-rod/rod"
	"github.com/go-rod/rod/lib/proto"
)

// RuntimeConfig 配置运行时。
type RuntimeConfig struct {
	Headless          bool
	BinPath           string
	AllowPrivateHosts bool // 仅测试放开；生产默认 false（SSRF 基础拦截）
	MaxElements       int
}

// Runtime 是 RuntimeAPI 的 go-rod 实现。
type Runtime struct {
	mgr      *Manager
	sessions *SessionStore
	cfg      RuntimeConfig
}

var _ RuntimeAPI = (*Runtime)(nil)

func NewRuntime(cfg RuntimeConfig) (*Runtime, error) {
	mgr, err := NewManager(ManagerConfig{Headless: cfg.Headless, BinPath: cfg.BinPath})
	if err != nil {
		return nil, err
	}
	return &Runtime{mgr: mgr, sessions: NewSessionStore(), cfg: cfg}, nil
}

func (r *Runtime) Open(ctx context.Context, req OpenReq) (OpenObservation, error) {
	if err := r.checkURL(req.URL); err != nil {
		return OpenObservation{}, err
	}
	sess, _ := r.sessions.Get(req.SessionID)
	if sess == nil {
		sess = r.sessions.Create(req.TaskID)
		c, err := r.mgr.AcquireContext(ContextOpts{})
		if err != nil {
			return OpenObservation{}, err
		}
		sess.Context = c
	}
	var obs Observation
	var opErr error
	sess.WithLock(func() {
		page, err := sess.Context.browser.Page(proto.TargetCreateTarget{URL: req.URL})
		if err != nil {
			opErr = NewBrowserError(CodeNavigationTimeout, err.Error())
			return
		}
		if err := page.WaitLoad(); err != nil {
			opErr = NewBrowserError(CodeNavigationTimeout, err.Error())
			return
		}
		sess.ActivePage = &pageHandle{page: page}
		obs = r.observe(page)
	})
	if opErr != nil {
		return OpenObservation{}, opErr
	}
	return OpenObservation{SessionID: sess.ID, Observation: obs}, nil
}

func (r *Runtime) Read(ctx context.Context, req ReadReq) (Observation, error) {
	sess, page, err := r.activePage(req.SessionID)
	if err != nil {
		return Observation{}, err
	}
	var obs Observation
	sess.WithLock(func() { obs = r.observe(page) })
	return obs, nil
}

func (r *Runtime) Click(ctx context.Context, req ClickReq) (Observation, error) {
	sess, page, err := r.activePage(req.SessionID)
	if err != nil {
		return Observation{}, err
	}
	var obs Observation
	var opErr error
	sess.WithLock(func() {
		el, err := r.elementByRef(sess, page, req.Ref)
		if err != nil {
			opErr = err
			return
		}
		if err := el.Click(proto.InputMouseButtonLeft, 1); err != nil {
			opErr = NewBrowserError(CodeElementNotFound, err.Error())
			return
		}
		_ = page.WaitLoad()
		obs = r.observe(page)
	})
	if opErr != nil {
		return Observation{}, opErr
	}
	return obs, nil
}

func (r *Runtime) Type(ctx context.Context, req TypeReq) (Observation, error) {
	sess, page, err := r.activePage(req.SessionID)
	if err != nil {
		return Observation{}, err
	}
	var obs Observation
	var opErr error
	sess.WithLock(func() {
		el, err := r.elementByRef(sess, page, req.Ref)
		if err != nil {
			opErr = err
			return
		}
		if err := el.Input(req.Text); err != nil {
			opErr = NewBrowserError(CodeElementNotFound, err.Error())
			return
		}
		if req.Submit {
			_ = el.Type(proto.InputDispatchKeyEventTypeKeyDown) // 占位，Step 3b 修正为回车
		}
		obs = r.observe(page)
	})
	if opErr != nil {
		return Observation{}, opErr
	}
	return obs, nil
}

func (r *Runtime) Close(ctx context.Context, req CloseReq) error {
	if req.SessionID != "" {
		if sess, ok := r.sessions.Get(req.SessionID); ok {
			_ = r.mgr.ReleaseContext(sess.Context)
			r.sessions.Delete(req.SessionID)
			return nil
		}
	}
	r.mgr.Close()
	return nil
}

// ---- 内部 ----

func (r *Runtime) activePage(sessionID string) (*Session, *rod.Page, error) {
	sess, ok := r.sessions.Get(sessionID)
	if !ok {
		return nil, nil, NewBrowserError(CodeContextEvicted, "unknown session "+sessionID)
	}
	if sess.ActivePage == nil || sess.ActivePage.page == nil {
		return nil, nil, NewBrowserError(CodeContextEvicted, "session has no active page")
	}
	return sess, sess.ActivePage.page.(*rod.Page), nil
}

// observe 抽 CDP a11y 树 → 裁剪观测，并把 ref→选择依据写回 session.Refs。
func (r *Runtime) observe(page *rod.Page) Observation {
	tree, err := proto.AccessibilityGetFullAXTree{}.Call(page)
	if err != nil {
		return Observation{Text: "(a11y unavailable)"}
	}
	var raw []RawA11yNode
	for _, n := range tree.Nodes {
		if n.Ignored {
			continue
		}
		role := ""
		if n.Role != nil {
			role = fmt.Sprintf("%v", n.Role.Value)
		}
		name := ""
		if n.Name != nil {
			name = fmt.Sprintf("%v", n.Name.Value)
		}
		raw = append(raw, RawA11yNode{
			Role:        role,
			Name:        name,
			Interactive: isInteractiveRole(role),
			Visible:     true, // Phase 1 近似：未被 Ignored 视为可见
		})
	}
	return BuildObservation(raw, ObservationBudget{MaxElements: r.cfg.MaxElements})
}

func isInteractiveRole(role string) bool {
	switch role {
	case "button", "link", "textbox", "checkbox", "radio", "combobox", "menuitem", "tab", "searchbox":
		return true
	}
	return false
}

// elementByRef 把观测里的 ref 映射回页面元素。Phase 1 近似实现：按 ref 序号在
// 当前可交互元素列表里取第 n 个（与 observe 的顺序一致）。Phase 2 换成
// backendNodeID 稳定映射（见 spec §3.5，需在 observe 时记 BackendDOMNodeID）。
func (r *Runtime) elementByRef(sess *Session, page *rod.Page, ref string) (*rod.Element, error) {
	obs := r.observe(page)
	idx := -1
	for i, e := range obs.Elements {
		if e.Ref == ref {
			idx = i
			break
		}
	}
	if idx < 0 {
		return nil, NewBrowserError(CodeElementNotFound, "ref "+ref+" not found; re-read")
	}
	els, err := page.Elements("a, button, input, textarea, select, [role=button], [role=link]")
	if err != nil || idx >= len(els) {
		return nil, NewBrowserError(CodeElementNotFound, "ref "+ref+" stale; re-read")
	}
	return els[idx], nil
}

func (r *Runtime) checkURL(raw string) error {
	u, err := url.Parse(raw)
	if err != nil {
		return NewBrowserError(CodeNavigationTimeout, "parse url: "+err.Error())
	}
	switch strings.ToLower(u.Scheme) {
	case "http", "https":
	default:
		return NewBrowserError(CodeProtocolBlocked, "scheme "+u.Scheme+" blocked")
	}
	if r.cfg.AllowPrivateHosts {
		return nil
	}
	host := u.Hostname()
	ips, err := net.LookupIP(host)
	if err != nil {
		return NewBrowserError(CodePrivateHostBlocked, "resolve "+host+": "+err.Error())
	}
	for _, ip := range ips {
		if ip.IsLoopback() || ip.IsPrivate() || ip.IsLinkLocalUnicast() {
			return NewBrowserError(CodePrivateHostBlocked, "host "+host+" resolves to private ip "+ip.String())
		}
	}
	return nil
}
```

- [ ] **Step 3b: 修正 Submit 回车 + pageHandle 真实定义**

在 `runtime.go` 的 `Type` 里把占位 Submit 改为按回车（据 Step 1 `go doc` 得到的键入 API 调整；go-rod 常用 `page.Keyboard.Type(input.Enter)`）：

```go
		if req.Submit {
			_ = page.Keyboard.Type(input.Enter) // import "github.com/go-rod/rod/lib/input"
		}
```

把 `session.go` 里的占位 `pageHandle` 保持 `page interface{}`（runtime 用 `.(*rod.Page)` 断言），无需改——但加注释说明存 `*rod.Page`。

- [ ] **Step 4: 跑集成测试**

Run: `go test -tags chromium ./internal/browser/ -run 'TestRuntime' -v`
Expected: PASS（`TestRuntimeOpenReadType` 与 `TestRuntimeOpenBlocksPrivateHost` 都过）。

- [ ] **Step 5: 确认非集成构建绿**

Run: `go build ./internal/browser/ && go vet ./internal/browser/`
Expected: 无错误。

- [ ] **Step 6: Commit**

```bash
git add internal/browser/runtime.go internal/browser/session.go internal/browser/runtime_integration_test.go
git commit -m "feat(browser): go-rod runtime implementing open/read/click/type/close"
```

---

## Task 7: 工具层 internal/tool/browser.go（四工具 + close）

照 `web.go` 的 `RegisterWebTools` 模式。用**假 RuntimeAPI** 做单测，不依赖 Chromium。

**Files:**
- Create: `internal/tool/browser.go`
- Test: `internal/tool/browser_test.go`

- [ ] **Step 1: 写失败测试（假 runtime，普通 go test）**

Create `internal/tool/browser_test.go`:

```go
package tool

import (
	"context"
	"testing"

	"github.com/stardust/legion-agent/internal/browser"
	"github.com/stardust/legion-agent/internal/domain"
)

type fakeBrowserRuntime struct {
	lastOpenURL string
	lastClickRef string
}

func (f *fakeBrowserRuntime) Open(_ context.Context, req browser.OpenReq) (browser.OpenObservation, error) {
	f.lastOpenURL = req.URL
	return browser.OpenObservation{
		SessionID:   "sess-1",
		Observation: browser.Observation{Text: "[e1] <button> 搜索\n"},
	}, nil
}
func (f *fakeBrowserRuntime) Read(context.Context, browser.ReadReq) (browser.Observation, error) {
	return browser.Observation{Text: "read-ok"}, nil
}
func (f *fakeBrowserRuntime) Click(_ context.Context, req browser.ClickReq) (browser.Observation, error) {
	f.lastClickRef = req.Ref
	return browser.Observation{Text: "clicked"}, nil
}
func (f *fakeBrowserRuntime) Type(context.Context, browser.TypeReq) (browser.Observation, error) {
	return browser.Observation{Text: "typed"}, nil
}
func (f *fakeBrowserRuntime) Close(context.Context, browser.CloseReq) error { return nil }

func newBrowserRegistry(t *testing.T, rt browser.RuntimeAPI) *Registry {
	t.Helper()
	reg := NewRegistry(NewStaticPolicy(DecisionAllow), nil, NoopGuardrails{})
	RegisterBrowserTools(reg, BrowserToolOptions{Enabled: true, Runtime: rt})
	return reg
}

func exec(t *testing.T, reg *Registry, name string, args map[string]string) domain.ToolResult {
	t.Helper()
	res, err := reg.Execute(context.Background(), domain.Agent{ID: "a", Role: "developer"},
		domain.ToolCall{ID: "c1", Name: name, Arguments: args})
	if err != nil {
		t.Fatalf("%s: %v", name, err)
	}
	return res
}

func TestBrowserOpenReturnsSessionAndObservation(t *testing.T) {
	f := &fakeBrowserRuntime{}
	reg := newBrowserRegistry(t, f)
	res := exec(t, reg, "browser_open", map[string]string{"url": "https://example.com"})
	if !res.Success {
		t.Fatalf("open failed: %q", res.Error)
	}
	if f.lastOpenURL != "https://example.com" {
		t.Fatalf("url not passed through: %q", f.lastOpenURL)
	}
	if !contains(res.Output, "sess-1") || !contains(res.Output, "搜索") {
		t.Fatalf("output missing session/observation: %q", res.Output)
	}
}

func TestBrowserClickPassesRef(t *testing.T) {
	f := &fakeBrowserRuntime{}
	reg := newBrowserRegistry(t, f)
	res := exec(t, reg, "browser_click", map[string]string{"session_id": "sess-1", "ref": "e1"})
	if !res.Success || f.lastClickRef != "e1" {
		t.Fatalf("click ref not passed: success=%v ref=%q", res.Success, f.lastClickRef)
	}
}

func contains(s, sub string) bool { return len(s) >= len(sub) && (s == sub || indexOf(s, sub) >= 0) }
func indexOf(s, sub string) int {
	for i := 0; i+len(sub) <= len(s); i++ {
		if s[i:i+len(sub)] == sub {
			return i
		}
	}
	return -1
}
```

- [ ] **Step 2: 跑，确认失败**

Run: `go test ./internal/tool/ -run 'TestBrowserOpen|TestBrowserClick' -v`
Expected: FAIL（`RegisterBrowserTools`/`BrowserToolOptions` 未定义）。

- [ ] **Step 3: 实现**

Create `internal/tool/browser.go`:

```go
package tool

import (
	"context"
	"fmt"

	"github.com/stardust/legion-agent/internal/browser"
	"github.com/stardust/legion-agent/internal/domain"
)

// BrowserToolOptions 见 spec §2.2。Enabled 为 false 时 RegisterBrowserTools 是 no-op。
type BrowserToolOptions struct {
	Enabled bool
	Runtime browser.RuntimeAPI
}

// RegisterBrowserTools 注册 browser_open/read/click/type/close。照 RegisterWebTools 语义：
// registry 为 nil、opts.Enabled 为 false、或 Runtime 为 nil 时是 no-op。
func RegisterBrowserTools(registry *Registry, opts BrowserToolOptions) {
	if registry == nil || !opts.Enabled || opts.Runtime == nil {
		return
	}
	rt := opts.Runtime

	registry.RegisterDescriptor(browserOpenDescriptor(), HandlerFunc(func(ctx context.Context, call domain.ToolCall) (domain.ToolResult, error) {
		url := call.Arguments["url"]
		if url == "" {
			return failure(call.ID, "url is required"), nil
		}
		out, err := rt.Open(ctx, browser.OpenReq{URL: url, SessionID: call.Arguments["session_id"]})
		if err != nil {
			return failure(call.ID, err.Error()), nil
		}
		return success(call.ID, fmt.Sprintf("session_id: %s\n%s", out.SessionID, out.Observation.Text)), nil
	}))

	registry.RegisterDescriptor(browserReadDescriptor(), HandlerFunc(func(ctx context.Context, call domain.ToolCall) (domain.ToolResult, error) {
		obs, err := rt.Read(ctx, browser.ReadReq{SessionID: call.Arguments["session_id"], Mode: call.Arguments["mode"]})
		if err != nil {
			return failure(call.ID, err.Error()), nil
		}
		return success(call.ID, obs.Text), nil
	}))

	registry.RegisterDescriptor(browserClickDescriptor(), HandlerFunc(func(ctx context.Context, call domain.ToolCall) (domain.ToolResult, error) {
		obs, err := rt.Click(ctx, browser.ClickReq{SessionID: call.Arguments["session_id"], Ref: call.Arguments["ref"]})
		if err != nil {
			return failure(call.ID, err.Error()), nil
		}
		return success(call.ID, obs.Text), nil
	}))

	registry.RegisterDescriptor(browserTypeDescriptor(), HandlerFunc(func(ctx context.Context, call domain.ToolCall) (domain.ToolResult, error) {
		submit := call.Arguments["submit"] == "true"
		obs, err := rt.Type(ctx, browser.TypeReq{
			SessionID: call.Arguments["session_id"], Ref: call.Arguments["ref"],
			Text: call.Arguments["text"], Submit: submit,
		})
		if err != nil {
			return failure(call.ID, err.Error()), nil
		}
		return success(call.ID, obs.Text), nil
	}))

	registry.RegisterDescriptor(browserCloseDescriptor(), HandlerFunc(func(ctx context.Context, call domain.ToolCall) (domain.ToolResult, error) {
		if err := rt.Close(ctx, browser.CloseReq{SessionID: call.Arguments["session_id"]}); err != nil {
			return failure(call.ID, err.Error()), nil
		}
		return success(call.ID, "ok"), nil
	}))
}

func success(id, out string) domain.ToolResult {
	return domain.ToolResult{CallID: id, Success: true, Output: out}
}
func failure(id, msg string) domain.ToolResult {
	return domain.ToolResult{CallID: id, Success: false, Error: msg}
}

func browserOpenDescriptor() Descriptor {
	return Descriptor{
		Name:        "browser_open",
		Description: "Open a URL in the agent's built-in browser. Returns a session_id and an accessibility-tree observation with stable refs.",
		RiskLevel:   "medium",
		Group:       "browser",
		Sensitive:   true, // 导航有副作用 → Manual 模式 gate
		InputSchema: map[string]any{
			"type":     "object",
			"required": []string{"url"},
			"properties": map[string]any{
				"url":        map[string]any{"type": "string", "description": "Absolute http/https URL."},
				"session_id": map[string]any{"type": "string", "description": "Reuse an existing session; omit to create one."},
			},
		},
	}
}

func browserReadDescriptor() Descriptor {
	return Descriptor{
		Name:        "browser_read",
		Description: "Read the current page as an accessibility tree with stable refs. Read-only.",
		RiskLevel:   "low",
		Group:       "browser",
		Sensitive:   false, // 只读 → Manual 放行、Plan 保留
		InputSchema: map[string]any{
			"type":     "object",
			"required": []string{"session_id"},
			"properties": map[string]any{
				"session_id": map[string]any{"type": "string"},
				"mode":       map[string]any{"type": "string", "description": "Reserved; only a11y supported in phase 1."},
			},
		},
	}
}

func browserClickDescriptor() Descriptor {
	return Descriptor{
		Name:        "browser_click",
		Description: "Click the element identified by a ref from the latest observation.",
		RiskLevel:   "medium",
		Group:       "browser",
		Sensitive:   true,
		InputSchema: map[string]any{
			"type":     "object",
			"required": []string{"session_id", "ref"},
			"properties": map[string]any{
				"session_id": map[string]any{"type": "string"},
				"ref":        map[string]any{"type": "string", "description": "Element ref like e12."},
			},
		},
	}
}

func browserTypeDescriptor() Descriptor {
	return Descriptor{
		Name:        "browser_type",
		Description: "Type text into the element identified by ref. Set submit=true to press Enter after.",
		RiskLevel:   "medium",
		Group:       "browser",
		Sensitive:   true,
		InputSchema: map[string]any{
			"type":     "object",
			"required": []string{"session_id", "ref", "text"},
			"properties": map[string]any{
				"session_id": map[string]any{"type": "string"},
				"ref":        map[string]any{"type": "string"},
				"text":       map[string]any{"type": "string"},
				"submit":     map[string]any{"type": "boolean"},
			},
		},
	}
}

func browserCloseDescriptor() Descriptor {
	return Descriptor{
		Name:        "browser_close",
		Description: "Close a browser session and release its context.",
		RiskLevel:   "low",
		Group:       "browser",
		Sensitive:   false,
		InputSchema: map[string]any{
			"type":     "object",
			"required": []string{"session_id"},
			"properties": map[string]any{
				"session_id": map[string]any{"type": "string"},
			},
		},
	}
}
```

> 注：`success`/`failure` 若与 `web.go` 已有同名 helper 冲突，改名为 `browserSuccess`/`browserFailure`。先 `grep -rn "func success\|func failure" internal/tool/` 确认；`web.go` 用的是 `webFailure`，故本文件用 `success`/`failure` 应无冲突，但仍需确认无其他文件占用。

- [ ] **Step 3b: 确认 helper 无重名**

Run: `grep -rn "func success(\|func failure(" internal/tool/`
Expected: 只匹配 `browser.go`。若命中别处，把 `browser.go` 的两个 helper 改名 `browserSuccess`/`browserFailure` 并替换全文件引用。

- [ ] **Step 4: 跑，确认通过**

Run: `go test ./internal/tool/ -run 'TestBrowserOpen|TestBrowserClick' -v`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add internal/tool/browser.go internal/tool/browser_test.go
git commit -m "feat(tool): browser_open/read/click/type/close tools"
```

---

## Task 8: toolauth 门控登记 + drift 守卫更新

drift-guard（`internal/runtime/toolauth_drift_test.go`）强制"生产已注册工具 ⊆ gateable"。登记五工具，并让 drift helper 也注册 browser 工具以保持覆盖诚实。

**Files:**
- Modify: `internal/toolauth/catalog.go`
- Modify: `internal/runtime/toolauth_drift_test.go`

- [ ] **Step 1: gateable 追加五工具**

Edit `internal/toolauth/catalog.go`，在 `gateable` 切片里（保持字母序合适位置）追加：

```go
	{"browser_open", "在内置浏览器打开一个 URL"},
	{"browser_read", "读取当前页的可访问性树"},
	{"browser_click", "点击 ref 指向的元素"},
	{"browser_type", "向 ref 指向的元素输入文本"},
	{"browser_close", "关闭一个浏览器会话"},
```

- [ ] **Step 2: drift helper 注册 browser 工具**

Edit `internal/runtime/toolauth_drift_test.go` 的 `productionToolRegistryForTest`，在 `tool.RegisterWebTools(...)` 之后加一行（用假 runtime，因为 drift 只看 descriptor 名，不执行）：

```go
	tool.RegisterBrowserTools(tools, tool.BrowserToolOptions{Enabled: true, Runtime: driftGuardBrowserRuntime{}})
```

并在该测试文件里加最小假实现（放文件末尾）：

```go
type driftGuardBrowserRuntime struct{}

func (driftGuardBrowserRuntime) Open(context.Context, browser.OpenReq) (browser.OpenObservation, error) {
	return browser.OpenObservation{}, nil
}
func (driftGuardBrowserRuntime) Read(context.Context, browser.ReadReq) (browser.Observation, error) {
	return browser.Observation{}, nil
}
func (driftGuardBrowserRuntime) Click(context.Context, browser.ClickReq) (browser.Observation, error) {
	return browser.Observation{}, nil
}
func (driftGuardBrowserRuntime) Type(context.Context, browser.TypeReq) (browser.Observation, error) {
	return browser.Observation{}, nil
}
func (driftGuardBrowserRuntime) Close(context.Context, browser.CloseReq) error { return nil }
```

加 import `"github.com/stardust/legion-agent/internal/browser"`。

- [ ] **Step 3: 跑 drift 守卫**

Run: `go test ./internal/runtime/ -run TestEveryProductionToolIsGateable -v`
Expected: PASS（五工具都在 gateable，drift 不报缺）。

- [ ] **Step 4: 跑 toolauth 包测试**

Run: `go test ./internal/toolauth/ -v`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add internal/toolauth/catalog.go internal/runtime/toolauth_drift_test.go
git commit -m "feat(toolauth): gate browser tools + cover them in drift guard"
```

---

## Task 9: 生产注册接线 + config 字段

把 `RegisterBrowserTools` 挂到三个生产装配点，浏览器运行时按 config 开关构造（默认关，避免无 Chromium 环境启动失败）。

**Files:**
- Modify: `internal/app/app.go`（约 line 241 附近，`RegisterWebTools` 调用旁）
- Modify: `internal/cli/command.go`（约 line 1994 附近）
- Modify: `internal/runtime/agent_resolver.go`（约 line 244 附近）
- Modify: 相关 config 结构（与 `Web` 配置同层，新增 `Browser`）
- Test: `internal/app/` 或对应包已有装配测试；新增一个"默认关时不注册"用例

- [ ] **Step 1: 先定位 config 结构**

Run:
```bash
grep -rn "WebToolOptions\|type WebConfig\|Web " internal/config/*.go internal/app/*.go 2>/dev/null | grep -iv test | head
grep -rn "opts.WebTools\|d.webOptions\|webToolOptions(" internal/ | grep -iv test
```
Expected: 找到 `Web` 配置字段定义处与三个 `WebTools`/`webOptions` 来源。据此加对应 `Browser` 字段（`Enabled bool`、`Headless bool`）。

- [ ] **Step 2: 加 config 字段**

在 web 配置同结构体加（示例，按实际结构体名调整）：

```go
// Browser 配置内置浏览器运行时。默认关闭——开启需运行环境有可用 Chromium。
type BrowserConfig struct {
	Enabled  bool `json:"enabled" yaml:"enabled"`
	Headless bool `json:"headless" yaml:"headless"`
}
```

并在顶层 config 里加 `Browser BrowserConfig`。

- [ ] **Step 3: 构造 Runtime 并注册（三处一致）**

在每个生产装配点，`RegisterWebTools` 调用之后加（以 `internal/app/app.go` 为例）：

```go
	if opts.Browser.Enabled {
		brt, err := browser.NewRuntime(browser.RuntimeConfig{Headless: opts.Browser.Headless})
		if err != nil {
			return nil, fmt.Errorf("init browser runtime: %w", err)
		}
		tool.RegisterBrowserTools(tools, tool.BrowserToolOptions{Enabled: true, Runtime: brt})
	}
```

在 `internal/cli/command.go` 与 `internal/runtime/agent_resolver.go` 做等价接线（用各自的 config 来源与错误返回风格）。加 import `"github.com/stardust/legion-agent/internal/browser"`。

> **生命周期**：本 Phase Runtime 随进程存活，退出时应 `Close`。若装配点有 shutdown 钩子，注册 `brt.Close(context.Background(), browser.CloseReq{})`；无则记为 Phase 3 待办（TTL/回收章节统一处理）。

- [ ] **Step 4: 写"默认关不注册"测试（放 internal/tool，不依赖 app 内部）**

Create `internal/tool/browser_disabled_test.go`:

```go
package tool

import (
	"strings"
	"testing"
)

// Browser.Enabled=false（或 Runtime=nil）时，RegisterBrowserTools 是 no-op：
// 注册表里不应出现任何 browser_* 工具。这守护 Task 9 生产接线的"默认关"契约。
func TestRegisterBrowserToolsNoopWhenDisabled(t *testing.T) {
	reg := NewRegistry(NewStaticPolicy(DecisionAllow), nil, NoopGuardrails{})
	RegisterBrowserTools(reg, BrowserToolOptions{Enabled: false, Runtime: &fakeBrowserRuntime{}})
	for _, d := range reg.Descriptors() {
		if strings.HasPrefix(d.Name, "browser_") {
			t.Fatalf("browser tool %q registered while disabled", d.Name)
		}
	}
	// Runtime=nil 也必须 no-op（防生产接线在 Enabled=true 但构造失败时半注册）
	reg2 := NewRegistry(NewStaticPolicy(DecisionAllow), nil, NoopGuardrails{})
	RegisterBrowserTools(reg2, BrowserToolOptions{Enabled: true, Runtime: nil})
	if len(reg2.Descriptors()) != 0 {
		t.Fatalf("expected no tools when Runtime is nil, got %d", len(reg2.Descriptors()))
	}
}
```

Run: `go test ./internal/tool/ -run TestRegisterBrowserToolsNoopWhenDisabled -v`
Expected: PASS（`fakeBrowserRuntime` 复用 Task 7 的 `browser_test.go` 定义，同包可见）。

- [ ] **Step 5: 编译 + 全量测试**

Run: `go build ./... && go test ./internal/app/ ./internal/cli/ ./internal/runtime/ ./internal/tool/ ./internal/toolauth/ -v`
Expected: PASS（默认关，无 Chromium 也全绿）。

- [ ] **Step 6: Commit**

```bash
git add internal/app/app.go internal/cli/command.go internal/runtime/agent_resolver.go internal/config/ internal/tool/browser_disabled_test.go
git commit -m "feat(browser): wire browser runtime into production registries (off by default)"
```

---

## Task 10: 端到端闭环测试（open→read→type→click）

跨工具层 + 真实 Chromium，验证 spec §7.1/§7.2 时序。build tag `chromium`。

**Files:**
- Create: `internal/browser/e2e_test.go`（`//go:build chromium`）

- [ ] **Step 1: 写端到端测试**

Create `internal/browser/e2e_test.go`:

```go
//go:build chromium

package browser

import (
	"context"
	"net/http"
	"net/http/httptest"
	"strings"
	"testing"
)

// TestE2EOpenReadTypeClick 走完最小闭环：打开一个带表单的页，读到 ref，输入，点提交。
func TestE2EOpenReadTypeClick(t *testing.T) {
	var submitted bool
	mux := http.NewServeMux()
	mux.HandleFunc("/", func(w http.ResponseWriter, _ *http.Request) {
		w.Header().Set("Content-Type", "text/html; charset=utf-8")
		_, _ = w.Write([]byte(`<html><body>
			<input aria-label="关键词框">
			<button onclick="fetch('/submit')">搜索</button>
		</body></html>`))
	})
	mux.HandleFunc("/submit", func(w http.ResponseWriter, _ *http.Request) { submitted = true })
	srv := httptest.NewServer(mux)
	defer srv.Close()

	rt, err := NewRuntime(RuntimeConfig{Headless: true, AllowPrivateHosts: true})
	if err != nil {
		t.Fatalf("NewRuntime: %v", err)
	}
	defer rt.Close(context.Background(), CloseReq{})
	ctx := context.Background()

	open, err := rt.Open(ctx, OpenReq{URL: srv.URL, TaskID: "e2e"})
	if err != nil {
		t.Fatalf("Open: %v", err)
	}
	sid := open.SessionID

	read, err := rt.Read(ctx, ReadReq{SessionID: sid})
	if err != nil {
		t.Fatalf("Read: %v", err)
	}
	var kwRef, btnRef string
	for _, e := range read.Elements {
		if strings.Contains(e.Name, "关键词") {
			kwRef = e.Ref
		}
		if strings.Contains(e.Name, "搜索") {
			btnRef = e.Ref
		}
	}
	if kwRef == "" || btnRef == "" {
		t.Fatalf("refs missing: kw=%q btn=%q in %q", kwRef, btnRef, read.Text)
	}

	if _, err := rt.Type(ctx, TypeReq{SessionID: sid, Ref: kwRef, Text: "legion"}); err != nil {
		t.Fatalf("Type: %v", err)
	}
	if _, err := rt.Click(ctx, ClickReq{SessionID: sid, Ref: btnRef}); err != nil {
		t.Fatalf("Click: %v", err)
	}
	// 给 fetch 一点时间落地
	if err := rt.mustWaitSubmit(&submitted); err != nil {
		t.Fatalf("submit not observed: %v", err)
	}
}
```

- [ ] **Step 2: 加测试辅助（轮询等待，避免裸 sleep）**

在 `e2e_test.go` 加：

```go
func (r *Runtime) mustWaitSubmit(flag *bool) error {
	// 轮询最多 ~2s，命中即返回；避免固定 sleep（spec §5.1 反对裸 sleep）。
	deadline := 200
	for i := 0; i < deadline; i++ {
		if *flag {
			return nil
		}
		// 10ms 一跳
		<-timeAfter10ms()
	}
	if *flag {
		return nil
	}
	return errSubmitTimeout
}
```

> `timeAfter10ms` / `errSubmitTimeout`：用 `time.After(10*time.Millisecond)` 与 `errors.New("submit timeout")`；此 helper 属测试文件，直接用标准库，无需藏进 Runtime——**修正**：把 `mustWaitSubmit` 改为测试内自由函数 `waitFlag(flag *bool) error`，不要挂在 `*Runtime` 上（避免污染生产类型）。相应把 Step 1 里 `rt.mustWaitSubmit(&submitted)` 改为 `waitFlag(&submitted)`。

- [ ] **Step 3: 跑端到端**

Run: `go test -tags chromium ./internal/browser/ -run TestE2EOpenReadTypeClick -v`
Expected: PASS（表单输入 + 点击触发 `/submit`，`submitted` 变 true）。

- [ ] **Step 4: 全量回归（非 chromium）**

Run: `go build ./... && go test ./... 2>&1 | tail -20`
Expected: 全绿（chromium-tag 测试被跳过，其余通过）。

- [ ] **Step 5: Commit**

```bash
git add internal/browser/e2e_test.go
git commit -m "test(browser): end-to-end open/read/type/click closed loop"
```

---

## 收尾：清理临时冒烟

- [ ] **Step 1: 删 Task 0 的临时冒烟**

删除 `internal/browser/smoke_test.go`（其价值已被 Task 4/6 集成测试取代）。

```bash
git rm internal/browser/smoke_test.go
git commit -m "chore(browser): drop temporary go-rod smoke test"
```

---

## 验证 Phase 1 DoD（对照 spec §8 Phase 1）

- [ ] 四工具 + close 端到端可用（Task 10 通过）
- [ ] 每动作自动附带最新 observation（Open/Read/Click/Type 均返回 `Observation`，工具层回传其 `Text`）
- [ ] a11y `ref` 会话内可用、`click(ref)` 命中（Task 10 用 ref 点击成功）
- [ ] 预算裁剪把大页面压到可控（Task 2 `MaxElements` 截断测试）
- [ ] 同 Session 串行（Task 5 `WithLock` 100 并发累加测试）
- [ ] toolauth 能看到并禁用五工具、drift 守卫绿（Task 8）
- [ ] Manual 模式对 open/click/type 触发审批、对 read/close 放行（由 `Descriptor.Sensitive` 决定——open/click/type=true，read/close=false；此为设计约束，M2b gate 逻辑复用现有 `dispatchToolCall`，本 plan 不改 gate 代码，只设 Sensitive 值）
- [ ] 默认关时无 Chromium 环境全绿（Task 9 Step 5）

---

## 已知近似与后续 Phase 债务（诚实记录，非 placeholder）

| 近似 | 本 Phase 做法 | 后续 |
|---|---|---|
| ref→元素映射 | 按观测顺序取第 n 个可交互元素（`elementByRef`） | Phase 2：改 `BackendDOMNodeID` 稳定映射，`observe` 时记录 |
| 可见性判定 | 未被 a11y `Ignored` 即视为可见 | Phase 2：接 CDP 布局盒/视口裁剪 |
| Context 池 | 单进程、不复用、不排队 | Phase 6：进程池扩缩容 + 排队 + 健康检查/Reap |
| SSRF | 协议白名单 + LookupIP 私网基础拦截 | Phase 5：DNS-rebinding 完整防护 + 沙箱 |
| 持久化 | 全内存 SessionStore | Phase 3：storageState + agent.db 落盘 + TTL 回收 |
| 观测流 | 无（工具返回同步观测） | Phase 2：SSE `/sessions/{id}/stream` |
| 生命周期 Close | 进程存活到退出 | Phase 3：优雅关闭钩子 |
