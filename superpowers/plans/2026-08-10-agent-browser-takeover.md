---
id: "plan-agent-browser-takeover-001"
title: "Agent 内置浏览器接管模式 实现计划"
aliases: ["Browser Takeover Plan", "接管模式实现计划"]
type: "plan"
category: "superpowers/plans"
tags: ["agent", "browser", "takeover", "input-injection", "plan", "tdd"]
version: "1.0.0"
created: "2026-08-10"
updated: "2026-08-10"
author: "jxncyjq"
status: "draft"
parent: null
children: []
related_docs:
  - id: "design-agent-browser-takeover-001"
    relation: "implements"
    path: "../specs/2026-08-09-agent-browser-takeover-design.md"
  - id: "reference-agent-browser-continue-001"
    relation: "references"
    path: "../../design/architecture/agent-browser-design-continue-save.md"
---

# Agent 内置浏览器接管模式 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 让用户在 GUI 浏览器视图临时接管 Agent 正在用的 go-rod 会话（注入鼠标/键盘），过掉验证码/登录这类"必须人来"的一步，期间挡住 Agent 写动作，退出后交还。

**Architecture:** 跨两仓。后端给 `Session` 加 `takeover` 标志（会话锁下读写）；写动作（Open/Click/Type）进入时若接管中返回新错误码 `SESSION_UNDER_TAKEOVER`，只读动作放行；新增 `SetTakeover`/`InjectInput` 方法（前端只发归一化坐标 0..1，后端 × 当前视口 px 后经 go-rod Mouse/Keyboard 注入活跃 Page）；两个 HTTP 端点走现有 bearer 守。前端 BrowserView 加接管开关 + canvas 捕获鼠标/键盘 → 映射归一化 → 节流批量 POST + "接管中"横幅。

**Tech Stack:** Go 1.x + go-rod v0.116.2（`Mouse.MoveTo/Down/Up/Click/Scroll`、`Keyboard.Press/Release`、`Page.InsertText`、`Page.Eval`）；React + zustand + Vitest（前端）；Wails `GetBrowserEndpoint()` 暴露 {baseURL, token}。

## Global Constraints

- **fail-loud 铁律**（legionAgent/CLAUDE.md §0）：禁兜底/fallback/静默吞错；错误 `fmt.Errorf("<动作> <标识>: %w", err)` 包装；边界记 Warn/Error。校验失败整批拒返回 error，不静默跳过单条。
- **状态字段全读写同一把锁**：`Session.takeover` 与 `Context`/`ActivePage` 一样，跨 goroutine 读写必须在 `sess.WithLock(...)` 下（会话锁 `sess.mu`）。
- **BrowserStreamer 接口只 `*browser.Runtime` 满足**：`SetTakeover`/`InjectInput` 加到 server 侧 `BrowserStreamer` 接口，不进 `browser.RuntimeAPI`（避免给三个 fake 加负担，同 `ReplaySince` 先例）。
- **接管注入 = Agent 真实 go-rod 会话本身**：注入作用于 `session.ActivePage`，非另开实例。
- **归一化坐标模型**：前端只发 `0 ≤ x,y ≤ 1`，后端 × 当前视口宽高得 px；鲁棒于 canvas CSS 缩放/帧分辨率变化。
- **无新 Agent 工具**：接管是 HTTP 端点，不是注册给 Agent 的工具，**不触发 toolauth.gateable / drift-guard**（无需改 `internal/toolauth/catalog.go`）。
- **端点鉴权复用 Phase 4B**：`/takeover`、`/input` 经现有 `HTTPServer.authorized()`（`Authorization: Bearer <adminToken>`）统一守，handler 内不重复校验。
- **测试门槛**：`go build ./... && go vet ./... && go test ./...` 全绿、`gofmt -l .` 为空；chromium 集成走 `//go:build chromium`；前端 `cd frontend && npx vitest run && npx tsc --noEmit`。
- **两分支两 PR，后端先合**：后端 = `github.com/jxncyjq/stardust-agent-server`（`legion/legionAgent/`）；GUI = `github.com/jxncyjq/stardust-agent-gui`（`legion/legionAgentGUI/`）。GUI 依赖后端端点，后端 PR 合入后再合 GUI。

---

## File Structure

**后端 `legion/legionAgent/`**
- `internal/browser/errors.go`（改）：加 `CodeTakeover`。
- `internal/browser/session.go`（改）：`Session` 加 `takeover bool` 字段。
- `internal/browser/input.go`（新）：`InputEvent` 类型、`validateInputEvents`、`keyToInputKey` 映射——纯逻辑，无 go-rod 依赖，好单测。
- `internal/browser/runtime.go`（改）：`SetTakeover`、`InjectInput`、`takeoverOf` helper；Open/Click/Type 加接管门控；Close/evictSession 清标志。
- `internal/browser/input_test.go`（新）：校验 + key 映射单测。
- `internal/browser/runtime_takeover_test.go`（新）：SetTakeover/门控/清标志单测（无 Chromium）。
- `internal/browser/takeover_chromium_test.go`（新，`//go:build chromium`）：真注入端到端。
- `internal/server/browser_stream.go`（改）：`BrowserStreamer` 接口加 `SetTakeover`/`InjectInput`。
- `internal/server/browser_input.go`（新）：`handleBrowserTakeover`、`handleBrowserInput`、路径解析。
- `internal/server/http.go`（改）：两条路由。
- `internal/server/browser_input_test.go`（新）：httptest + fake。

**前端 `legion/legionAgentGUI/frontend/`**
- `src/lib/browserInput.ts`（新）：坐标映射 `mapToNormalized`、`Throttler`、事件构造——纯函数，好单测。
- `src/lib/browserInput.test.ts`（新）。
- `src/stores/browserStore.ts`（改）：加 `takeover` 状态 + `setTakeover`。
- `src/components/BrowserView.tsx`（改）：接管开关 + canvas 捕获 + 横幅 + POST。
- `src/components/BrowserView.test.tsx`（新）：接管开关/横幅/POST（mock fetch + GetBrowserEndpoint）。

---

## 后端 — stardust-agent-server（PR 先合）

### Task B1: `CodeTakeover` + `Session.takeover` + `SetTakeover`

**Files:**
- Modify: `legion/legionAgent/internal/browser/errors.go`
- Modify: `legion/legionAgent/internal/browser/session.go:13-24`
- Modify: `legion/legionAgent/internal/browser/runtime.go`
- Test: `legion/legionAgent/internal/browser/runtime_takeover_test.go`

**Interfaces:**
- Produces: `browser.CodeTakeover Code = "SESSION_UNDER_TAKEOVER"`；`Session.takeover bool`（会话锁下）；`(*Runtime).SetTakeover(sessionID string, enabled bool) error`；`(*Runtime).takeoverOf(sess *Session) bool`。

- [ ] **Step 1: 写失败测试**

`legion/legionAgent/internal/browser/runtime_takeover_test.go`：
```go
package browser

import "testing"

// newTestSession 直接建一个带会话的 store-only Runtime，不起 Chromium。
func newTakeoverRuntime(t *testing.T) (*Runtime, *Session) {
	t.Helper()
	r := &Runtime{sessions: NewSessionStore(), hubs: newHubRegistry()}
	sess := r.sessions.Create("task-1")
	return r, sess
}

func TestSetTakeoverTogglesFlag(t *testing.T) {
	r, sess := newTakeoverRuntime(t)
	if r.takeoverOf(sess) {
		t.Fatal("new session should not be under takeover")
	}
	if err := r.SetTakeover(sess.ID, true); err != nil {
		t.Fatalf("SetTakeover(true): %v", err)
	}
	if !r.takeoverOf(sess) {
		t.Fatal("takeover flag should be set")
	}
	if err := r.SetTakeover(sess.ID, false); err != nil {
		t.Fatalf("SetTakeover(false): %v", err)
	}
	if r.takeoverOf(sess) {
		t.Fatal("takeover flag should be cleared")
	}
}

func TestSetTakeoverUnknownSession(t *testing.T) {
	r, _ := newTakeoverRuntime(t)
	err := r.SetTakeover("sess-does-not-exist", true)
	if err == nil {
		t.Fatal("expected error for unknown session")
	}
	var be *BrowserError
	if !asBrowserError(err, &be) || be.Code != CodeContextEvicted {
		t.Fatalf("want CONTEXT_EVICTED, got %v", err)
	}
}
```
在文件顶部加测试 helper（若仓库已有 errors.As 包装则复用）：
```go
import "errors"

func asBrowserError(err error, target **BrowserError) bool { return errors.As(err, target) }
```

- [ ] **Step 2: 跑测试确认失败**

Run: `cd legion/legionAgent && go test ./internal/browser/ -run TestSetTakeover -v`
Expected: FAIL —— `r.takeoverOf`、`r.SetTakeover` 未定义（编译错）。

- [ ] **Step 3: 加错误码**

`internal/browser/errors.go` 的 const 块末尾加：
```go
	CodeTakeover           Code = "SESSION_UNDER_TAKEOVER" // 会话被人工接管，Agent 写动作暂挡
```

- [ ] **Step 4: 加 Session 字段**

`internal/browser/session.go` 的 `Session` 结构体（`LastUsedAt time.Time` 之后、`mu sync.Mutex` 之前）加：
```go
	// takeover 为真时该会话进入人工接管：Agent 的写动作（Open/Click/Type）被挡，
	// 只有前端注入的输入能写。会话锁（mu）下读写，与 Context/ActivePage 同一把锁。
	takeover bool
```

- [ ] **Step 5: 实现 SetTakeover + takeoverOf**

`internal/browser/runtime.go` 末尾（`checkURL` 之后）加：
```go
// SetTakeover 置/清会话的人工接管标志（会话锁下）。enabled=false 恢复 Agent。
// 未知会话按 CONTEXT_EVICTED 报错（不静默成功——fail-loud）。
func (r *Runtime) SetTakeover(sessionID string, enabled bool) error {
	sess, ok := r.sessions.Get(sessionID)
	if !ok {
		return NewBrowserError(CodeContextEvicted, "unknown session "+sessionID)
	}
	sess.WithLock(func() { sess.takeover = enabled })
	return nil
}

// takeoverOf 在会话锁下读接管标志。nil 会话视为未接管。
func (r *Runtime) takeoverOf(sess *Session) bool {
	if sess == nil {
		return false
	}
	var v bool
	sess.WithLock(func() { v = sess.takeover })
	return v
}
```

- [ ] **Step 6: 跑测试确认通过**

Run: `cd legion/legionAgent && go test ./internal/browser/ -run TestSetTakeover -v`
Expected: PASS（两个用例）。

- [ ] **Step 7: 提交**

```bash
cd legion/legionAgent && git add internal/browser/errors.go internal/browser/session.go internal/browser/runtime.go internal/browser/runtime_takeover_test.go && git commit -m "feat(browser): add takeover flag and SetTakeover"
```

---

### Task B2: 写动作接管门控（Open/Click/Type 挡，Read 放行）

**Files:**
- Modify: `legion/legionAgent/internal/browser/runtime.go`（Open/Click/Type）
- Test: `legion/legionAgent/internal/browser/runtime_takeover_test.go`

**Interfaces:**
- Consumes: `takeoverOf`、`CodeTakeover`（Task B1）。
- Produces: Open/Click/Type 在 `takeover==true` 时返回 `CodeTakeover` 错误；Read 不受影响。

- [ ] **Step 1: 写失败测试**

追加到 `runtime_takeover_test.go`。因 Open/Click/Type 需活跃页，这里只断言"接管中入口即被挡"——门控发生在取活跃页之前，故无需 Chromium：
```go
import "context"

func TestClickBlockedUnderTakeover(t *testing.T) {
	r, sess := newTakeoverRuntime(t)
	if err := r.SetTakeover(sess.ID, true); err != nil {
		t.Fatalf("SetTakeover: %v", err)
	}
	_, err := r.Click(context.Background(), ClickReq{SessionID: sess.ID, Ref: "e1"})
	var be *BrowserError
	if !asBrowserError(err, &be) || be.Code != CodeTakeover {
		t.Fatalf("want SESSION_UNDER_TAKEOVER, got %v", err)
	}
}

func TestTypeBlockedUnderTakeover(t *testing.T) {
	r, sess := newTakeoverRuntime(t)
	_ = r.SetTakeover(sess.ID, true)
	_, err := r.Type(context.Background(), TypeReq{SessionID: sess.ID, Ref: "e1", Text: "x"})
	var be *BrowserError
	if !asBrowserError(err, &be) || be.Code != CodeTakeover {
		t.Fatalf("want SESSION_UNDER_TAKEOVER, got %v", err)
	}
}

func TestOpenBlockedUnderTakeover(t *testing.T) {
	r, sess := newTakeoverRuntime(t)
	_ = r.SetTakeover(sess.ID, true)
	// Open 会先 checkURL；用一个能过白名单解析的公网 http url，门控须在导航前触发。
	_, err := r.Open(context.Background(), OpenReq{SessionID: sess.ID, URL: "http://example.com"})
	var be *BrowserError
	if !asBrowserError(err, &be) || be.Code != CodeTakeover {
		t.Fatalf("want SESSION_UNDER_TAKEOVER, got %v", err)
	}
}
```

> 注：`Open` 的现有实现先 `checkURL(req.URL)`（会 DNS 解析 host）。为让门控**先于**任何副作用触发，Step 3 把接管检查放在 `Open` 函数体最前面（`checkURL` 之前）。`example.com` 仅作占位，测试在门控处即返回，不会真导航。

- [ ] **Step 2: 跑测试确认失败**

Run: `cd legion/legionAgent && go test ./internal/browser/ -run "Blocked" -v`
Expected: FAIL —— 当前无门控，`Click`/`Type` 走到 `activePage` 返回 `CONTEXT_EVICTED`（非 `CodeTakeover`），`Open` 走到 checkURL/新建。

- [ ] **Step 3: 加门控**

`internal/browser/runtime.go`：

在 `Open` 函数体**第一行**（`if err := r.checkURL(req.URL); err != nil {` 之前）插入：
```go
	// 接管门控：会话接管中时 Agent 的写动作退让（read/screenshot 不受此限）。
	// 置于函数最前，先于 checkURL 的 DNS 解析等任何副作用。传了 SessionID 才可能接管；
	// 空 SessionID 是新建会话，天然不在接管态。
	if req.SessionID != "" {
		if sess, ok := r.sessions.Get(req.SessionID); ok && r.takeoverOf(sess) {
			return OpenObservation{}, NewBrowserError(CodeTakeover, "session "+req.SessionID+" under manual takeover")
		}
	}
```

在 `Click` 函数体，取到 sess 后、动作前加门控。`Click` 现在开头是 `sess, page, err := r.activePage(req.SessionID)`；改为在其**后**紧接：
```go
	if r.takeoverOf(sess) {
		return Observation{}, NewBrowserError(CodeTakeover, "session "+req.SessionID+" under manual takeover")
	}
```
`Type` 同样在 `sess, page, err := r.activePage(req.SessionID)` 之后加同一段。

> `Read` 不加门控（接管中 Agent 仍可观测，无害，与 Sensitive=false 判定一致）。

- [ ] **Step 4: 跑测试确认通过**

Run: `cd legion/legionAgent && go test ./internal/browser/ -run "Blocked|Takeover" -v`
Expected: PASS（全部接管用例）。

- [ ] **Step 5: 全包回归**

Run: `cd legion/legionAgent && go test ./internal/browser/`
Expected: ok（不破坏既有测试）。

- [ ] **Step 6: 提交**

```bash
cd legion/legionAgent && git add internal/browser/runtime.go internal/browser/runtime_takeover_test.go && git commit -m "feat(browser): block agent write actions while under takeover"
```

---

### Task B3: `InputEvent` + 校验 + key 映射（纯逻辑）

**Files:**
- Create: `legion/legionAgent/internal/browser/input.go`
- Test: `legion/legionAgent/internal/browser/input_test.go`

**Interfaces:**
- Produces:
  - `type InputEvent struct{ Type string; X,Y float64; Button string; DeltaX,DeltaY float64; Key,Text string }`（带 json tag）。
  - `const maxInputBatch = 256`、`const maxKeyLen = 32`、`const maxTextLen = 1024`。
  - `func validateInputEvents(events []InputEvent) error`。
  - `func keyToInputKey(key string) (input.Key, error)`（JS `e.key` → go-rod `input.Key`）。

- [ ] **Step 1: 写失败测试**

`legion/legionAgent/internal/browser/input_test.go`：
```go
package browser

import (
	"strings"
	"testing"

	"github.com/go-rod/rod/lib/input"
)

func TestValidateInputEventsOK(t *testing.T) {
	err := validateInputEvents([]InputEvent{
		{Type: "mousemove", X: 0.5, Y: 0.5},
		{Type: "click", X: 0, Y: 1, Button: "left"},
		{Type: "wheel", X: 0.2, Y: 0.3, DeltaY: 120},
		{Type: "keydown", Key: "Enter"},
		{Type: "char", Text: "hello"},
	})
	if err != nil {
		t.Fatalf("valid batch rejected: %v", err)
	}
}

func TestValidateInputEventsRejects(t *testing.T) {
	cases := map[string]InputEvent{
		"unknown type":  {Type: "drag", X: 0.1, Y: 0.1},
		"x below range": {Type: "mousemove", X: -0.01, Y: 0.5},
		"y above range": {Type: "mousemove", X: 0.5, Y: 1.01},
		"bad button":    {Type: "mousedown", X: 0.5, Y: 0.5, Button: "sideways"},
		"key too long":  {Type: "keydown", Key: strings.Repeat("a", maxKeyLen+1)},
		"text too long": {Type: "char", Text: strings.Repeat("z", maxTextLen+1)},
	}
	for name, ev := range cases {
		if err := validateInputEvents([]InputEvent{ev}); err == nil {
			t.Errorf("%s: expected rejection, got nil", name)
		}
	}
}

func TestValidateInputEventsBatchCap(t *testing.T) {
	big := make([]InputEvent, maxInputBatch+1)
	for i := range big {
		big[i] = InputEvent{Type: "mousemove", X: 0.5, Y: 0.5}
	}
	if err := validateInputEvents(big); err == nil {
		t.Fatal("expected batch-size rejection")
	}
}

func TestValidateInputEventsEmpty(t *testing.T) {
	if err := validateInputEvents(nil); err == nil {
		t.Fatal("empty batch should be rejected (nothing to inject)")
	}
}

func TestKeyToInputKey(t *testing.T) {
	if k, err := keyToInputKey("Enter"); err != nil || k != input.Enter {
		t.Fatalf("Enter map: %v %v", k, err)
	}
	if k, err := keyToInputKey("a"); err != nil || k != input.Key('a') {
		t.Fatalf("printable map: %v %v", k, err)
	}
	if _, err := keyToInputKey("F13Nonsense"); err == nil {
		t.Fatal("unknown key should error")
	}
	if _, err := keyToInputKey(""); err == nil {
		t.Fatal("empty key should error")
	}
}
```

- [ ] **Step 2: 跑测试确认失败**

Run: `cd legion/legionAgent && go test ./internal/browser/ -run "InputEvents|KeyToInputKey" -v`
Expected: FAIL（未定义符号，编译错）。

- [ ] **Step 3: 实现 input.go**

`legion/legionAgent/internal/browser/input.go`：
```go
package browser

import (
	"fmt"
	"math"
	"utf8" // 见下：用 unicode/utf8
)

// 修正 import（占位说明——实际用下方 import 块）。

// InputEvent 是前端接管期间注入的一条归一化输入事件（见 takeover spec §3.1）。
// 坐标 X/Y 为相对视口的 0..1 归一化值，后端 × 当前视口 px 后经 go-rod 注入。
type InputEvent struct {
	Type   string  `json:"type"`             // mousemove|mousedown|mouseup|click|wheel|keydown|keyup|char
	X      float64 `json:"x"`                // 0..1；鼠标类必填
	Y      float64 `json:"y"`                // 0..1
	Button string  `json:"button,omitempty"` // left|right|middle
	DeltaX float64 `json:"deltaX,omitempty"` // wheel
	DeltaY float64 `json:"deltaY,omitempty"`
	Key    string  `json:"key,omitempty"`  // keydown/keyup 的键名（JS e.key）
	Text   string  `json:"text,omitempty"` // char：要插入的文本
}

const (
	maxInputBatch = 256  // 单批事件条数上限，防滥用
	maxKeyLen     = 32   // 键名长度上限
	maxTextLen    = 1024 // char 文本长度上限
)

// mouseTypes / keyTypes 是类型白名单。
var mouseTypes = map[string]bool{
	"mousemove": true, "mousedown": true, "mouseup": true, "click": true, "wheel": true,
}
var buttonNames = map[string]bool{"left": true, "right": true, "middle": true}

// validateInputEvents 硬校验一整批注入事件；任一条越界即整批拒（fail-loud，不静默跳过单条）。
func validateInputEvents(events []InputEvent) error {
	if len(events) == 0 {
		return fmt.Errorf("input batch is empty")
	}
	if len(events) > maxInputBatch {
		return fmt.Errorf("input batch too large: %d > %d", len(events), maxInputBatch)
	}
	for i, ev := range events {
		switch {
		case mouseTypes[ev.Type]:
			if err := checkNormalized(ev.X, ev.Y); err != nil {
				return fmt.Errorf("event %d (%s): %w", i, ev.Type, err)
			}
			if ev.Button != "" && !buttonNames[ev.Button] {
				return fmt.Errorf("event %d (%s): bad button %q", i, ev.Type, ev.Button)
			}
		case ev.Type == "keydown" || ev.Type == "keyup":
			if ev.Key == "" {
				return fmt.Errorf("event %d (%s): missing key", i, ev.Type)
			}
			if len(ev.Key) > maxKeyLen {
				return fmt.Errorf("event %d (%s): key too long", i, ev.Type)
			}
		case ev.Type == "char":
			if len(ev.Text) > maxTextLen {
				return fmt.Errorf("event %d (char): text too long", i)
			}
		default:
			return fmt.Errorf("event %d: unknown type %q", i, ev.Type)
		}
	}
	return nil
}

// checkNormalized 断言 x,y ∈ [0,1] 且有限。
func checkNormalized(x, y float64) error {
	for _, v := range []float64{x, y} {
		if math.IsNaN(v) || math.IsInf(v, 0) || v < 0 || v > 1 {
			return fmt.Errorf("coordinate %v out of [0,1]", v)
		}
	}
	return nil
}
```
> **实现者注意 import**：把上面占位的 import 块替换为真实的
> ```go
> import (
> 	"fmt"
> 	"math"
> 	"unicode/utf8"
>
> 	"github.com/go-rod/rod/lib/input"
> )
> ```
> （`utf8` 用于 keyToInputKey 判定单字符；`input` 用于 Key 映射。）

在同文件继续加 key 映射：
```go
// namedKeys 把常用特殊键的 JS e.key 名映射到 go-rod input.Key 常量（白名单）。
// 可打印单字符走 keyToInputKey 的 rune 分支，不进这张表。
var namedKeys = map[string]input.Key{
	"Enter":      input.Enter,
	"Backspace":  input.Backspace,
	"Tab":        input.Tab,
	"Escape":     input.Escape,
	"Delete":     input.Delete,
	"ArrowLeft":  input.ArrowLeft,
	"ArrowRight": input.ArrowRight,
	"ArrowUp":    input.ArrowUp,
	"ArrowDown":  input.ArrowDown,
	"Home":       input.Home,
	"End":        input.End,
	"PageUp":     input.PageUp,
	"PageDown":   input.PageDown,
}

// keyToInputKey 把 JS e.key 转成 go-rod input.Key：命名键查白名单，
// 单个可打印字符按 rune 直取；其它一律报错（fail-loud，不猜键）。
func keyToInputKey(key string) (input.Key, error) {
	if key == "" {
		return 0, fmt.Errorf("empty key")
	}
	if k, ok := namedKeys[key]; ok {
		return k, nil
	}
	if utf8.RuneCountInString(key) == 1 {
		r, _ := utf8.DecodeRuneInString(key)
		return input.Key(r), nil
	}
	return 0, fmt.Errorf("unsupported key %q", key)
}
```

> **实现者注意**：先确认 `input.Home/End/PageUp/PageDown/ArrowRight/ArrowUp/ArrowDown` 常量在 `github.com/go-rod/rod/lib/input`（keymap.go）确实存在；缺哪个就从 `namedKeys` 删哪个（`grep -n "Home\|End\|PageUp\|PageDown\|ArrowRight" $(go env GOMODCACHE)/github.com/go-rod/rod@v0.116.2/lib/input/keymap.go`）。`Enter/Backspace/Tab/Escape/Delete/ArrowLeft` 已确认存在。

- [ ] **Step 4: 跑测试确认通过**

Run: `cd legion/legionAgent && go test ./internal/browser/ -run "InputEvents|KeyToInputKey" -v`
Expected: PASS。

- [ ] **Step 5: gofmt + vet**

Run: `cd legion/legionAgent && gofmt -l internal/browser/input.go && go vet ./internal/browser/`
Expected: 无输出（gofmt 干净）、vet 通过。

- [ ] **Step 6: 提交**

```bash
cd legion/legionAgent && git add internal/browser/input.go internal/browser/input_test.go && git commit -m "feat(browser): add InputEvent model, validation and key mapping"
```

---

### Task B4: `InjectInput`（归一化 → px + go-rod 派发）+ chromium 集成

**Files:**
- Modify: `legion/legionAgent/internal/browser/runtime.go`
- Test: `legion/legionAgent/internal/browser/takeover_chromium_test.go`（`//go:build chromium`）

**Interfaces:**
- Consumes: `InputEvent`/`validateInputEvents`/`keyToInputKey`（B3）；`takeoverOf`（B1）；`activePage`（既有）。
- Produces: `(*Runtime).InjectInput(sessionID string, events []InputEvent) error`。

- [ ] **Step 1: 写失败集成测试**

`legion/legionAgent/internal/browser/takeover_chromium_test.go`：
```go
//go:build chromium

package browser

import (
	"context"
	"net/http"
	"net/http/httptest"
	"sync/atomic"
	"testing"
)

// 集成测试用系统 Chrome（PAL 定位），进接管后注入 click，断言页面 onclick 触发。
func TestInjectInputClickFiresOnClick(t *testing.T) {
	var clicked int32
	mux := http.NewServeMux()
	mux.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
		w.Header().Set("Content-Type", "text/html")
		_, _ = w.Write([]byte(`<html><body>
			<button id="b" style="position:absolute;left:0;top:0;width:100%;height:100%"
			onclick="fetch('/hit')">X</button></body></html>`))
	})
	mux.HandleFunc("/hit", func(w http.ResponseWriter, r *http.Request) { atomic.AddInt32(&clicked, 1) })
	srv := httptest.NewServer(mux)
	defer srv.Close()

	r := newChromiumRuntime(t) // 复用既有集成测试 helper（PAL 定位系统 Chrome，AllowPrivateHosts=true）
	defer r.Close(context.Background(), CloseReq{})

	obs, err := r.Open(context.Background(), OpenReq{URL: srv.URL})
	if err != nil {
		t.Fatalf("open: %v", err)
	}
	sid := obs.SessionID
	if err := r.SetTakeover(sid, true); err != nil {
		t.Fatalf("SetTakeover: %v", err)
	}
	// 按钮铺满视口，点中心必命中。
	if err := r.InjectInput(sid, []InputEvent{{Type: "click", X: 0.5, Y: 0.5, Button: "left"}}); err != nil {
		t.Fatalf("InjectInput: %v", err)
	}
	// 轮询等 onclick 的 fetch 落地。
	waitFor(t, func() bool { return atomic.LoadInt32(&clicked) == 1 })
}

func TestInjectInputRequiresTakeover(t *testing.T) {
	r := newChromiumRuntime(t)
	defer r.Close(context.Background(), CloseReq{})
	obs, err := r.Open(context.Background(), OpenReq{URL: "about:blank"})
	if err != nil {
		// about:blank 可能被 checkURL 挡；改用一个真实的空白页服务同上。这里仅示意契约：
		t.Skipf("open about:blank not supported: %v", err)
	}
	err = r.InjectInput(obs.SessionID, []InputEvent{{Type: "mousemove", X: 0.5, Y: 0.5}})
	var be *BrowserError
	if !asBrowserError(err, &be) || be.Code != CodeTakeover {
		t.Fatalf("inject without takeover should error CodeTakeover, got %v", err)
	}
}
```
> **实现者注意**：`newChromiumRuntime`、`waitFor` 若既有 chromium 测试文件已提供则直接复用；若无，照既有 `*_chromium_test.go` 的构造惯例新增（`NewRuntime(RuntimeConfig{Headless:true, AllowPrivateHosts:true, BinPath: PAL 定位})`）。`asBrowserError` 已在 B1 的 `runtime_takeover_test.go` 定义——因它是非 build-tag 文件，chromium tag 下同包可见。

- [ ] **Step 2: 跑测试确认失败**

Run: `cd legion/legionAgent && go test -tags chromium ./internal/browser/ -run TestInjectInput -v`
Expected: FAIL —— `InjectInput` 未定义。

- [ ] **Step 3: 实现 InjectInput**

`internal/browser/runtime.go` 末尾加（放在 `SetTakeover` 之后）：
```go
// InjectInput 把一批归一化输入事件注入会话活跃页（接管必须先开，否则拒）。
// 每批先整体校验，再读一次当前视口宽高，把 0..1 坐标 × px 后经 go-rod 派发。
// 与 Read/Click 等一样在会话锁下取活跃页，但注入本身在锁外执行（go-rod 调用可能阻塞，
// 不宜久持会话锁）——接管期间 Agent 写动作已被门控，无并发写者，故锁外注入安全。
func (r *Runtime) InjectInput(sessionID string, events []InputEvent) error {
	if err := validateInputEvents(events); err != nil {
		return NewBrowserErrorWrap(CodeElementNotFound, "invalid input batch", err)
	}
	sess, page, err := r.activePage(sessionID)
	if err != nil {
		return err
	}
	if !r.takeoverOf(sess) {
		return NewBrowserError(CodeTakeover, "session "+sessionID+" not under takeover; enable takeover before injecting")
	}
	vw, vh, err := viewportSize(page)
	if err != nil {
		return NewBrowserErrorWrap(CodeContextEvicted, "read viewport for injection", err)
	}
	for i, ev := range events {
		if err := injectOne(page, ev, vw, vh); err != nil {
			return NewBrowserErrorWrap(CodeElementNotFound, fmt.Sprintf("inject event %d (%s)", i, ev.Type), err)
		}
	}
	r.touch(sess) // 人工也算活跃：刷新 LastUsedAt，避免接管中会话被 reaper 回收
	return nil
}

// viewportSize 读当前视口的 CSS 像素宽高（window.innerWidth/innerHeight）。
func viewportSize(page *rod.Page) (float64, float64, error) {
	res, err := page.Eval("() => ({w: window.innerWidth, h: window.innerHeight})")
	if err != nil {
		return 0, 0, err
	}
	w := res.Value.Get("w").Num()
	h := res.Value.Get("h").Num()
	if w <= 0 || h <= 0 {
		return 0, 0, fmt.Errorf("non-positive viewport %vx%v", w, h)
	}
	return w, h, nil
}

// injectOne 把一条归一化事件 × 视口 px 后派发到 go-rod。鼠标类先 MoveTo 定位再动作。
func injectOne(page *rod.Page, ev InputEvent, vw, vh float64) error {
	px := proto.Point{X: ev.X * vw, Y: ev.Y * vh}
	switch ev.Type {
	case "mousemove":
		return page.Mouse.MoveTo(px)
	case "mousedown":
		if err := page.Mouse.MoveTo(px); err != nil {
			return err
		}
		return page.Mouse.Down(mouseButton(ev.Button), 1)
	case "mouseup":
		if err := page.Mouse.MoveTo(px); err != nil {
			return err
		}
		return page.Mouse.Up(mouseButton(ev.Button), 1)
	case "click":
		if err := page.Mouse.MoveTo(px); err != nil {
			return err
		}
		return page.Mouse.Click(mouseButton(ev.Button), 1)
	case "wheel":
		if err := page.Mouse.MoveTo(px); err != nil {
			return err
		}
		return page.Mouse.Scroll(ev.DeltaX, ev.DeltaY, 1)
	case "keydown":
		k, err := keyToInputKey(ev.Key)
		if err != nil {
			return err
		}
		return page.Keyboard.Press(k)
	case "keyup":
		k, err := keyToInputKey(ev.Key)
		if err != nil {
			return err
		}
		return page.Keyboard.Release(k)
	case "char":
		return page.InsertText(ev.Text)
	default:
		// validateInputEvents 已挡掉未知类型；到这里属编程错误。
		return fmt.Errorf("unhandled input type %q", ev.Type)
	}
}

// mouseButton 把事件 button 名映射到 go-rod 常量；空/未知回落 left（validate 已挡未知非空值）。
func mouseButton(name string) proto.InputMouseButton {
	switch name {
	case "right":
		return proto.InputMouseButtonRight
	case "middle":
		return proto.InputMouseButtonMiddle
	default:
		return proto.InputMouseButtonLeft
	}
}
```
> **实现者注意**：`runtime.go` 顶部 import 已含 `fmt`? 若无则加 `"fmt"`。`proto`、`rod`、`input` 已 import（文件顶部可见）。`res.Value.Get(...).Num()` 用 go-rod 的 gson.JSON API；若 `.Num()` 不存在改用 `.Int()`（keymap 返回整型像素亦可）。

- [ ] **Step 4: 跑集成测试确认通过**

Run: `cd legion/legionAgent && go test -tags chromium ./internal/browser/ -run TestInjectInput -v`
Expected: PASS（click 触发 onclick）。若本机 Chromium 缺失，PAL 定位系统 Chrome（见踩坑存档）。

- [ ] **Step 5: 普通构建 + vet（无 tag，确认不破坏非 chromium 构建）**

Run: `cd legion/legionAgent && go build ./... && go vet ./internal/browser/ && gofmt -l internal/browser/runtime.go`
Expected: 构建通过、vet 通过、gofmt 无输出。

- [ ] **Step 6: 提交**

```bash
cd legion/legionAgent && git add internal/browser/runtime.go internal/browser/takeover_chromium_test.go && git commit -m "feat(browser): inject normalized input events into active page"
```

---

### Task B5: `BrowserStreamer` 接口扩展 + HTTP 端点 + 路由

**Files:**
- Modify: `legion/legionAgent/internal/server/browser_stream.go:17-20`
- Create: `legion/legionAgent/internal/server/browser_input.go`
- Modify: `legion/legionAgent/internal/server/http.go:284-285`（路由）
- Test: `legion/legionAgent/internal/server/browser_input_test.go`

**Interfaces:**
- Consumes: `(*browser.Runtime).SetTakeover`/`InjectInput`（B1/B4）；`browser.InputEvent`（B3）。
- Produces: `BrowserStreamer` 接口加两方法；`POST /v1/browser/sessions/{id}/takeover`、`POST /v1/browser/sessions/{id}/input` 两端点。

- [ ] **Step 1: 写失败测试**

`legion/legionAgent/internal/server/browser_input_test.go`：
```go
package server

import (
	"bytes"
	"encoding/json"
	"net/http"
	"net/http/httptest"
	"testing"

	"github.com/stardust/legion-agent/internal/browser"
)

// fakeBrowser 满足扩展后的 BrowserStreamer；记录调用供断言。
type fakeBrowser struct {
	takeoverCalls []struct {
		id      string
		enabled bool
	}
	injected  [][]browser.InputEvent
	injectErr error
}

func (f *fakeBrowser) Subscribe(string) (<-chan browser.StreamEvent, func(), error) {
	ch := make(chan browser.StreamEvent)
	return ch, func() {}, nil
}
func (f *fakeBrowser) ReplaySince(string, uint64) []browser.StreamEvent { return nil }
func (f *fakeBrowser) SetTakeover(id string, enabled bool) error {
	f.takeoverCalls = append(f.takeoverCalls, struct {
		id      string
		enabled bool
	}{id, enabled})
	return nil
}
func (f *fakeBrowser) InjectInput(id string, events []browser.InputEvent) error {
	if f.injectErr != nil {
		return f.injectErr
	}
	f.injected = append(f.injected, events)
	return nil
}

func newBrowserTestServer(fb *fakeBrowser) *HTTPServer {
	return NewHTTPServer(Config{Browser: fb})
}

func TestHandleTakeoverSetsFlag(t *testing.T) {
	fb := &fakeBrowser{}
	s := newBrowserTestServer(fb)
	body, _ := json.Marshal(map[string]bool{"enabled": true})
	req := httptest.NewRequest(http.MethodPost, "/v1/browser/sessions/sess-1/takeover", bytes.NewReader(body))
	rec := httptest.NewRecorder()
	s.ServeHTTP(rec, req)
	if rec.Code != http.StatusOK {
		t.Fatalf("status = %d, want 200; body=%s", rec.Code, rec.Body.String())
	}
	if len(fb.takeoverCalls) != 1 || fb.takeoverCalls[0].id != "sess-1" || !fb.takeoverCalls[0].enabled {
		t.Fatalf("SetTakeover not called as expected: %+v", fb.takeoverCalls)
	}
}

func TestHandleInputInjects(t *testing.T) {
	fb := &fakeBrowser{}
	s := newBrowserTestServer(fb)
	body, _ := json.Marshal(map[string]any{
		"events": []browser.InputEvent{{Type: "click", X: 0.5, Y: 0.5, Button: "left"}},
	})
	req := httptest.NewRequest(http.MethodPost, "/v1/browser/sessions/sess-1/input", bytes.NewReader(body))
	rec := httptest.NewRecorder()
	s.ServeHTTP(rec, req)
	if rec.Code != http.StatusOK {
		t.Fatalf("status = %d, want 200; body=%s", rec.Code, rec.Body.String())
	}
	if len(fb.injected) != 1 || len(fb.injected[0]) != 1 || fb.injected[0][0].Type != "click" {
		t.Fatalf("InjectInput not called as expected: %+v", fb.injected)
	}
}

func TestHandleInputNilBrowser503(t *testing.T) {
	s := NewHTTPServer(Config{}) // Browser nil
	req := httptest.NewRequest(http.MethodPost, "/v1/browser/sessions/sess-1/input", bytes.NewReader([]byte(`{"events":[]}`)))
	rec := httptest.NewRecorder()
	s.ServeHTTP(rec, req)
	if rec.Code != http.StatusServiceUnavailable {
		t.Fatalf("status = %d, want 503", rec.Code)
	}
}

func TestHandleTakeoverBearer(t *testing.T) {
	fb := &fakeBrowser{}
	s := NewHTTPServer(Config{Browser: fb, AdminToken: "secret"})
	req := httptest.NewRequest(http.MethodPost, "/v1/browser/sessions/sess-1/takeover", bytes.NewReader([]byte(`{"enabled":true}`)))
	rec := httptest.NewRecorder()
	s.ServeHTTP(rec, req) // 无 Authorization 头
	if rec.Code != http.StatusUnauthorized {
		t.Fatalf("status = %d, want 401 without bearer", rec.Code)
	}
}
```

- [ ] **Step 2: 跑测试确认失败**

Run: `cd legion/legionAgent && go test ./internal/server/ -run "Takeover|HandleInput" -v`
Expected: FAIL —— 路由未注册（404）、fake 与接口不匹配编译错。

- [ ] **Step 3: 扩展 BrowserStreamer 接口**

`internal/server/browser_stream.go` 的接口改为：
```go
type BrowserStreamer interface {
	Subscribe(sessionID string) (<-chan browser.StreamEvent, func(), error)
	ReplaySince(sessionID string, lastID uint64) []browser.StreamEvent
	// SetTakeover 置/清会话人工接管标志；InjectInput 注入一批归一化输入事件。
	// 二者仅具体 *browser.Runtime 满足（同 ReplaySince），不进 browser.RuntimeAPI。
	SetTakeover(sessionID string, enabled bool) error
	InjectInput(sessionID string, events []browser.InputEvent) error
}
```

- [ ] **Step 4: 实现 handler**

`legion/legionAgent/internal/server/browser_input.go`：
```go
package server

import (
	"encoding/json"
	"net/http"
	"strings"

	"github.com/stardust/legion-agent/internal/browser"
)

// parseBrowserActionID 从 /v1/browser/sessions/{id}/{action} 抽 id 与 action。
func parseBrowserActionID(path string) (id, action string, ok bool) {
	const prefix = "/v1/browser/sessions/"
	if !strings.HasPrefix(path, prefix) {
		return "", "", false
	}
	rest := strings.TrimPrefix(path, prefix)
	slash := strings.LastIndex(rest, "/")
	if slash <= 0 || slash == len(rest)-1 {
		return "", "", false
	}
	id, action = rest[:slash], rest[slash+1:]
	if id == "" || strings.Contains(id, "/") {
		return "", "", false
	}
	return id, action, true
}

type takeoverRequest struct {
	Enabled bool `json:"enabled"`
}

type inputRequest struct {
	Events []browser.InputEvent `json:"events"`
}

// handleBrowserTakeover 置/清会话接管标志。auth 由 HTTPServer.authorized 统一守。
func (s *HTTPServer) handleBrowserTakeover(w http.ResponseWriter, r *http.Request) {
	if s.browser == nil {
		writeError(w, http.StatusServiceUnavailable, "browser runtime is unavailable")
		return
	}
	id, _, ok := parseBrowserActionID(r.URL.Path)
	if !ok {
		writeError(w, http.StatusNotFound, "bad browser takeover path")
		return
	}
	var req takeoverRequest
	if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
		writeError(w, http.StatusBadRequest, "invalid takeover request")
		return
	}
	if err := s.browser.SetTakeover(id, req.Enabled); err != nil {
		writeError(w, http.StatusNotFound, err.Error())
		return
	}
	writeJSON(w, http.StatusOK, map[string]any{"session_id": id, "takeover": req.Enabled})
}

// handleBrowserInput 注入一批输入事件；要求该会话已进入接管（InjectInput 内校验后拒非接管）。
func (s *HTTPServer) handleBrowserInput(w http.ResponseWriter, r *http.Request) {
	if s.browser == nil {
		writeError(w, http.StatusServiceUnavailable, "browser runtime is unavailable")
		return
	}
	id, _, ok := parseBrowserActionID(r.URL.Path)
	if !ok {
		writeError(w, http.StatusNotFound, "bad browser input path")
		return
	}
	var req inputRequest
	if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
		writeError(w, http.StatusBadRequest, "invalid input request")
		return
	}
	if err := s.browser.InjectInput(id, req.Events); err != nil {
		// 未接管即注入 → InjectInput 返回 CodeTakeover；映射 409（须先进接管）。
		// 其它（校验失败/无活跃页）→ 400。
		if strings.Contains(err.Error(), string(browser.CodeTakeover)) {
			writeError(w, http.StatusConflict, err.Error())
			return
		}
		writeError(w, http.StatusBadRequest, err.Error())
		return
	}
	writeJSON(w, http.StatusOK, map[string]any{"session_id": id, "injected": len(req.Events)})
}
```

- [ ] **Step 5: 注册路由**

`internal/server/http.go` 的 `ServeHTTP` switch，在现有 browser stream 分支（`case r.Method == http.MethodGet && strings.HasPrefix(... "/stream")`）**之后**插入：
```go
		case r.Method == http.MethodPost && strings.HasPrefix(r.URL.Path, "/v1/browser/sessions/") && strings.HasSuffix(r.URL.Path, "/takeover"):
			s.handleBrowserTakeover(rec, r)
		case r.Method == http.MethodPost && strings.HasPrefix(r.URL.Path, "/v1/browser/sessions/") && strings.HasSuffix(r.URL.Path, "/input"):
			s.handleBrowserInput(rec, r)
```

- [ ] **Step 6: 跑测试确认通过**

Run: `cd legion/legionAgent && go test ./internal/server/ -run "Takeover|HandleInput" -v`
Expected: PASS（全部 4 用例）。

- [ ] **Step 7: 全 server 包 + 全仓回归**

Run: `cd legion/legionAgent && go build ./... && go vet ./... && go test ./internal/server/ ./internal/browser/ && gofmt -l internal/server/ internal/browser/`
Expected: 全绿、gofmt 无输出。

- [ ] **Step 8: 提交**

```bash
cd legion/legionAgent && git add internal/server/browser_stream.go internal/server/browser_input.go internal/server/http.go internal/server/browser_input_test.go && git commit -m "feat(server): add browser takeover and input HTTP endpoints"
```

---

### Task B6: Close/evict 清接管标志（防悬挂）

**Files:**
- Modify: `legion/legionAgent/internal/browser/runtime.go`（Close 的 per-session 分支）
- Test: `legion/legionAgent/internal/browser/runtime_takeover_test.go`

**Interfaces:**
- Consumes: 既有 `Close`、`Session.takeover`。
- Produces: 关闭会话时 `takeover` 一并清零（Session 从内存删除即随之消失；本任务确保 evict/复用路径也清）。

- [ ] **Step 1: 写失败测试**

追加到 `runtime_takeover_test.go`（无 Chromium：Close 的 per-session 分支在 sess 无 Context 时仍会走删表 + 清标志路径，`ReleaseContext(nil)` 需容忍 nil——若既有实现对 nil Context 报错则本用例改断言 error 后标志仍清）：
```go
func TestCloseClearsTakeover(t *testing.T) {
	r, sess := newTakeoverRuntime(t)
	_ = r.SetTakeover(sess.ID, true)
	// per-session Close：nil Context 下 ReleaseContext 可能返回 error，这里只关心标志被清。
	_ = r.Close(context.Background(), CloseReq{SessionID: sess.ID})
	// 会话已从表删除；再置接管应因未知会话报错，证明其不再残留。
	if err := r.SetTakeover(sess.ID, true); err == nil {
		t.Fatal("closed session should be gone, SetTakeover must error")
	}
}
```
> 若 `newTakeoverRuntime` 的 `Runtime` 无 `mgr`，`Close` 的 `ReleaseContext(nil)` 会 nil 解引用。为让此单测可跑，Step 3 在 per-session Close 里对 `sess.Context == nil` 时跳过 `ReleaseContext`（本就无 Context 可释放），并清 `takeover`。

- [ ] **Step 2: 跑测试确认失败**

Run: `cd legion/legionAgent && go test ./internal/browser/ -run TestCloseClearsTakeover -v`
Expected: FAIL 或 panic（nil mgr / 标志未清）。

- [ ] **Step 3: 清标志 + nil-Context 守卫**

`internal/browser/runtime.go` 的 `Close` per-session 分支，`sess.WithLock` 内改为：
```go
			var relErr error
			sess.WithLock(func() {
				if sess.Context != nil {
					relErr = r.mgr.ReleaseContext(sess.Context)
				}
				sess.Context = nil
				sess.ActivePage = nil
				sess.takeover = false // 关闭即退接管，防标志悬挂
			})
```

- [ ] **Step 4: 跑测试确认通过**

Run: `cd legion/legionAgent && go test ./internal/browser/ -run TestCloseClearsTakeover -v`
Expected: PASS。

- [ ] **Step 5: -race 全 browser 包**

Run: `cd legion/legionAgent && go test -race ./internal/browser/`
Expected: ok，无 race（本机 Windows 可跑 -race，见踩坑存档）。

- [ ] **Step 6: 提交**

```bash
cd legion/legionAgent && git add internal/browser/runtime.go internal/browser/runtime_takeover_test.go && git commit -m "feat(browser): clear takeover flag on session close"
```

---

### Task B7: 后端收尾 — 全量验证 + 开 PR

- [ ] **Step 1: 全量测试门槛**

Run: `cd legion/legionAgent && go build ./... && go vet ./... && go test ./... && gofmt -l .`
Expected: 全绿、`gofmt -l .` 无输出。

- [ ] **Step 2: chromium 集成再跑一遍**

Run: `cd legion/legionAgent && go test -tags chromium ./internal/browser/ ./internal/server/`
Expected: PASS（含 InjectInput click 端到端）。

- [ ] **Step 3: 请 code review（必抓真 bug）**

用 `superpowers:requesting-code-review` 或 OMC `code-reviewer` agent 审：门控是否覆盖全部写动作、锁边界（takeover 读写是否都在会话锁下）、InjectInput 锁外注入是否有并发写者、409/400 映射是否稳（勿用 err 字符串包含判定作为唯一契约——如脆弱则改 InjectInput 返回可判别错误类型让 handler `errors.As`）。修掉发现的真 bug。

- [ ] **Step 4: 推分支 + 开 PR**

```bash
cd legion/legionAgent && git push -u origin HEAD
```
PR 标题 `feat(browser): manual takeover — input injection + agent write gating`；正文引用 spec `design-agent-browser-takeover-001`。等 CI（含 browser-matrix 三平台）绿再合 master。

---

## 前端 — stardust-agent-gui（后端合入后再合）

### Task F1: `browserInput.ts` — 坐标映射 + 节流 + 事件构造

**Files:**
- Create: `legion/legionAgentGUI/frontend/src/lib/browserInput.ts`
- Test: `legion/legionAgentGUI/frontend/src/lib/browserInput.test.ts`

**Interfaces:**
- Produces:
  - `interface InputEvent { type: string; x?: number; y?: number; button?: string; deltaX?: number; deltaY?: number; key?: string; text?: string }`
  - `function mapToNormalized(rect: {left:number;top:number;width:number;height:number}, clientX:number, clientY:number): {x:number;y:number}`（clamp 0..1）
  - `class Throttler { constructor(intervalMs:number); ready(nowMs:number): boolean }`
  - `function postInput(baseURL:string, token:string, sessionId:string, events:InputEvent[]): Promise<void>`
  - `function postTakeover(baseURL:string, token:string, sessionId:string, enabled:boolean): Promise<void>`

- [ ] **Step 1: 写失败测试**

`legion/legionAgentGUI/frontend/src/lib/browserInput.test.ts`：
```ts
import { describe, it, expect, vi, beforeEach } from 'vitest'
import { mapToNormalized, Throttler, postInput, postTakeover } from './browserInput'

describe('mapToNormalized', () => {
  const rect = { left: 100, top: 50, width: 200, height: 400 }
  it('maps center to 0.5,0.5', () => {
    expect(mapToNormalized(rect, 200, 250)).toEqual({ x: 0.5, y: 0.5 })
  })
  it('clamps below-left to 0,0', () => {
    expect(mapToNormalized(rect, 0, 0)).toEqual({ x: 0, y: 0 })
  })
  it('clamps beyond bottom-right to 1,1', () => {
    expect(mapToNormalized(rect, 9999, 9999)).toEqual({ x: 1, y: 1 })
  })
})

describe('Throttler', () => {
  it('gates within interval, opens after', () => {
    const t = new Throttler(25)
    expect(t.ready(0)).toBe(true)
    expect(t.ready(10)).toBe(false)
    expect(t.ready(30)).toBe(true)
  })
})

describe('postInput / postTakeover', () => {
  beforeEach(() => { vi.restoreAllMocks() })
  it('POSTs input with bearer and events', async () => {
    const fetchMock = vi.fn().mockResolvedValue({ ok: true })
    vi.stubGlobal('fetch', fetchMock)
    await postInput('http://h:1', 'tok', 'sess-1', [{ type: 'click', x: 0.5, y: 0.5, button: 'left' }])
    expect(fetchMock).toHaveBeenCalledWith(
      'http://h:1/v1/browser/sessions/sess-1/input',
      expect.objectContaining({
        method: 'POST',
        headers: expect.objectContaining({ Authorization: 'Bearer tok' }),
      }),
    )
  })
  it('throws loud on non-ok', async () => {
    vi.stubGlobal('fetch', vi.fn().mockResolvedValue({ ok: false, status: 409 }))
    await expect(postTakeover('http://h:1', '', 'sess-1', true)).rejects.toThrow(/409/)
  })
})
```

- [ ] **Step 2: 跑测试确认失败**

Run: `cd legion/legionAgentGUI/frontend && npx vitest run src/lib/browserInput.test.ts`
Expected: FAIL（模块不存在）。

- [ ] **Step 3: 实现 browserInput.ts**

`legion/legionAgentGUI/frontend/src/lib/browserInput.ts`：
```ts
// browserInput 负责接管模式下把 DOM 鼠标/键盘事件映射为后端归一化 InputEvent，
// 并经 fetch+bearer POST 到会话端点。坐标只发 0..1，后端 × 视口 px。

export interface InputEvent {
  type: string
  x?: number
  y?: number
  button?: string
  deltaX?: number
  deltaY?: number
  key?: string
  text?: string
}

// mapToNormalized 用 canvas 显示矩形把 client 坐标换成 0..1（clamp 越界）。
export function mapToNormalized(
  rect: { left: number; top: number; width: number; height: number },
  clientX: number,
  clientY: number,
): { x: number; y: number } {
  const clamp = (v: number) => (v < 0 ? 0 : v > 1 ? 1 : v)
  return {
    x: clamp((clientX - rect.left) / rect.width),
    y: clamp((clientY - rect.top) / rect.height),
  }
}

// Throttler 按最小间隔放行（用于合并高频 mousemove）。调用方传入单调 now(ms)。
export class Throttler {
  private last = -Infinity
  constructor(private intervalMs: number) {}
  ready(nowMs: number): boolean {
    if (nowMs - this.last >= this.intervalMs) {
      this.last = nowMs
      return true
    }
    return false
  }
}

// postInput 注入一批事件；非 2xx 抛错（fail-loud，让调用方提示注入失败）。
export async function postInput(
  baseURL: string,
  token: string,
  sessionId: string,
  events: InputEvent[],
): Promise<void> {
  await postJSON(`${baseURL}/v1/browser/sessions/${sessionId}/input`, token, { events })
}

// postTakeover 置/清接管标志。
export async function postTakeover(
  baseURL: string,
  token: string,
  sessionId: string,
  enabled: boolean,
): Promise<void> {
  await postJSON(`${baseURL}/v1/browser/sessions/${sessionId}/takeover`, token, { enabled })
}

async function postJSON(url: string, token: string, body: unknown): Promise<void> {
  const headers: Record<string, string> = { 'Content-Type': 'application/json' }
  if (token) headers['Authorization'] = `Bearer ${token}`
  const res = await fetch(url, { method: 'POST', headers, body: JSON.stringify(body) })
  if (!res.ok) {
    throw new Error(`browser POST ${url}: HTTP ${res.status}`)
  }
}
```

- [ ] **Step 4: 跑测试确认通过**

Run: `cd legion/legionAgentGUI/frontend && npx vitest run src/lib/browserInput.test.ts`
Expected: PASS。

- [ ] **Step 5: 提交**

```bash
cd legion/legionAgentGUI && git add frontend/src/lib/browserInput.ts frontend/src/lib/browserInput.test.ts && git commit -m "feat(gui): browser input mapping, throttle and POST helpers"
```

---

### Task F2: `browserStore` 接管状态

**Files:**
- Modify: `legion/legionAgentGUI/frontend/src/stores/browserStore.ts`
- Test: `legion/legionAgentGUI/frontend/src/stores/browserStore.test.ts`（新，若既有则追加）

**Interfaces:**
- Consumes: 既有 `useBrowserStore`。
- Produces: state 加 `takeover: boolean`；action `setTakeover: (v: boolean) => void`；`setSession(null)`/`reset()` 时 `takeover` 归 false。

- [ ] **Step 1: 写失败测试**

`legion/legionAgentGUI/frontend/src/stores/browserStore.test.ts`：
```ts
import { describe, it, expect, beforeEach } from 'vitest'
import { useBrowserStore } from './browserStore'

describe('browserStore takeover', () => {
  beforeEach(() => useBrowserStore.getState().reset())
  it('defaults to false', () => {
    expect(useBrowserStore.getState().takeover).toBe(false)
  })
  it('setTakeover toggles', () => {
    useBrowserStore.getState().setTakeover(true)
    expect(useBrowserStore.getState().takeover).toBe(true)
  })
  it('clears takeover when session cleared', () => {
    useBrowserStore.getState().setSession('sess-1')
    useBrowserStore.getState().setTakeover(true)
    useBrowserStore.getState().setSession(null)
    expect(useBrowserStore.getState().takeover).toBe(false)
  })
})
```

- [ ] **Step 2: 跑测试确认失败**

Run: `cd legion/legionAgentGUI/frontend && npx vitest run src/stores/browserStore.test.ts`
Expected: FAIL（`takeover`/`setTakeover` 未定义）。

- [ ] **Step 3: 加状态**

`src/stores/browserStore.ts`：
- `interface BrowserState` 加：
```ts
  takeover: boolean
  setTakeover: (v: boolean) => void
```
- `const empty` 加 `takeover: false,`（这样 `setSession`/`reset` 复用 `empty` 时自动清零）：
```ts
const empty = {
  frameDataUri: null, elements: [] as BrowserElement[], observationText: '',
  progress: null, connected: false, lastEventId: 0, takeover: false,
}
```
- store 实现加 action：
```ts
  setTakeover: (v) => set({ takeover: v }),
```

- [ ] **Step 4: 跑测试确认通过**

Run: `cd legion/legionAgentGUI/frontend && npx vitest run src/stores/browserStore.test.ts`
Expected: PASS。

- [ ] **Step 5: 提交**

```bash
cd legion/legionAgentGUI && git add frontend/src/stores/browserStore.ts frontend/src/stores/browserStore.test.ts && git commit -m "feat(gui): add takeover state to browserStore"
```

---

### Task F3: `BrowserView` 接管开关 + canvas 捕获 + 横幅

**Files:**
- Modify: `legion/legionAgentGUI/frontend/src/components/BrowserView.tsx`
- Test: `legion/legionAgentGUI/frontend/src/components/BrowserView.test.tsx`

**Interfaces:**
- Consumes: `mapToNormalized`/`Throttler`/`postInput`/`postTakeover`（F1）；`useBrowserStore.takeover`/`setTakeover`（F2）；`GetBrowserEndpoint()`（既有 Wails 绑定，返回 `{baseURL, token}`）。
- Produces: BrowserView 带接管开关按钮、接管态 canvas 捕获鼠标/键盘并 POST、"接管中"横幅。

- [ ] **Step 1: 写失败测试**

`legion/legionAgentGUI/frontend/src/components/BrowserView.test.tsx`：
```tsx
import { describe, it, expect, vi, beforeEach } from 'vitest'
import { render, screen, fireEvent, waitFor } from '@testing-library/react'
import { BrowserView } from './BrowserView'
import { useBrowserStore } from '../stores/browserStore'

// mock Wails 绑定与 stream hook（避免真连 SSE）。
vi.mock('../../wailsjs/go/main/App', () => ({
  GetBrowserEndpoint: vi.fn().mockResolvedValue({ baseURL: 'http://h:1', token: 'tok' }),
}))
vi.mock('../hooks/useBrowserStream', () => ({ useBrowserStream: () => {} }))

describe('BrowserView takeover', () => {
  beforeEach(() => {
    useBrowserStore.getState().reset()
    useBrowserStore.getState().setSession('sess-1')
    vi.restoreAllMocks()
  })

  it('toggles takeover via POST and shows banner', async () => {
    const fetchMock = vi.fn().mockResolvedValue({ ok: true })
    vi.stubGlobal('fetch', fetchMock)
    render(<BrowserView />)
    const btn = screen.getByRole('button', { name: /接管/ })
    fireEvent.click(btn)
    await waitFor(() =>
      expect(fetchMock).toHaveBeenCalledWith(
        'http://h:1/v1/browser/sessions/sess-1/takeover',
        expect.objectContaining({ method: 'POST' }),
      ),
    )
    await waitFor(() => expect(screen.getByText(/接管中/)).toBeInTheDocument())
  })
})
```

- [ ] **Step 2: 跑测试确认失败**

Run: `cd legion/legionAgentGUI/frontend && npx vitest run src/components/BrowserView.test.tsx`
Expected: FAIL（无接管按钮）。

- [ ] **Step 3: 改 BrowserView**

`src/components/BrowserView.tsx` 全量替换为（在只读版基础上加接管）：
```tsx
import { useEffect, useRef, useCallback } from 'react'
import { GetBrowserEndpoint } from '../../wailsjs/go/main/App'
import { useBrowserStore } from '../stores/browserStore'
import { useBrowserStream } from '../hooks/useBrowserStream'
import { mapToNormalized, Throttler, postInput, postTakeover, type InputEvent } from '../lib/browserInput'

// BrowserView：只读展示 Agent 浏览过程；接管开时 canvas 捕获鼠标/键盘注入 Agent 会话。
export function BrowserView() {
  const sessionId = useBrowserStore((s) => s.sessionId)
  const frameDataUri = useBrowserStore((s) => s.frameDataUri)
  const elements = useBrowserStore((s) => s.elements)
  const progress = useBrowserStore((s) => s.progress)
  const connected = useBrowserStore((s) => s.connected)
  const takeover = useBrowserStore((s) => s.takeover)
  const setTakeover = useBrowserStore((s) => s.setTakeover)
  const canvasRef = useRef<HTMLCanvasElement>(null)
  const moveThrottle = useRef(new Throttler(25)) // ~40fps 合并 mousemove

  useBrowserStream(sessionId)

  useEffect(() => {
    if (!frameDataUri || !canvasRef.current) return
    const canvas = canvasRef.current
    const ctx = canvas.getContext('2d')
    if (!ctx) return
    let cancelled = false
    const img = new Image()
    img.onload = () => {
      if (cancelled) return
      canvas.width = img.width
      canvas.height = img.height
      ctx.drawImage(img, 0, 0)
    }
    img.onerror = () => console.warn('browser view: frame decode failed')
    img.src = frameDataUri
    return () => { cancelled = true }
  }, [frameDataUri])

  // send 把一批事件经 GetBrowserEndpoint 的 token POST 到后端；失败响亮报（不静默）。
  const send = useCallback(async (events: InputEvent[]) => {
    if (!sessionId) return
    try {
      const ep = await GetBrowserEndpoint()
      await postInput(ep.baseURL, ep.token, sessionId, events)
    } catch (err) {
      console.error('browser takeover: inject failed', err)
    }
  }, [sessionId])

  const toggleTakeover = useCallback(async () => {
    if (!sessionId) return
    const next = !takeover
    try {
      const ep = await GetBrowserEndpoint()
      await postTakeover(ep.baseURL, ep.token, sessionId, next)
      setTakeover(next)
    } catch (err) {
      console.error('browser takeover: toggle failed', err)
    }
  }, [sessionId, takeover, setTakeover])

  const norm = (e: { clientX: number; clientY: number }) => {
    const rect = canvasRef.current!.getBoundingClientRect()
    return mapToNormalized(rect, e.clientX, e.clientY)
  }

  const onMouseMove = (e: React.MouseEvent) => {
    if (!takeover) return
    if (!moveThrottle.current.ready(e.timeStamp)) return
    const { x, y } = norm(e)
    void send([{ type: 'mousemove', x, y }])
  }
  const onMouseDown = (e: React.MouseEvent) => {
    if (!takeover) return
    const { x, y } = norm(e)
    void send([{ type: 'mousedown', x, y, button: mouseButtonName(e.button) }])
  }
  const onMouseUp = (e: React.MouseEvent) => {
    if (!takeover) return
    const { x, y } = norm(e)
    void send([{ type: 'mouseup', x, y, button: mouseButtonName(e.button) }])
  }
  const onClick = (e: React.MouseEvent) => {
    if (!takeover) return
    const { x, y } = norm(e)
    void send([{ type: 'click', x, y, button: mouseButtonName(e.button) }])
  }
  const onWheel = (e: React.WheelEvent) => {
    if (!takeover) return
    const { x, y } = norm(e)
    void send([{ type: 'wheel', x, y, deltaX: e.deltaX, deltaY: e.deltaY }])
  }
  const onKeyDown = (e: React.KeyboardEvent) => {
    if (!takeover) return
    e.preventDefault()
    if (e.key.length === 1) void send([{ type: 'char', text: e.key }])
    else void send([{ type: 'keydown', key: e.key }])
  }
  const onKeyUp = (e: React.KeyboardEvent) => {
    if (!takeover) return
    if (e.key.length !== 1) void send([{ type: 'keyup', key: e.key }])
  }

  if (!sessionId) {
    return (
      <div className="flex h-full items-center justify-center text-sm text-muted-foreground">
        Agent 未在浏览
      </div>
    )
  }

  return (
    <div className="flex h-full flex-col gap-2 p-2">
      <div className="flex items-center gap-2 text-xs">
        <span className={connected ? 'text-green-500' : 'text-amber-500'}>●</span>
        <span className="text-muted-foreground">session {sessionId}</span>
        {progress && (
          <span className="text-muted-foreground">· {progress.action}:{progress.status}</span>
        )}
        <button
          onClick={toggleTakeover}
          className="ml-auto rounded border border-border px-2 py-0.5 text-xs hover:bg-muted"
        >
          {takeover ? '退出接管' : '接管'}
        </button>
      </div>
      {takeover && (
        <div className="rounded bg-amber-500/20 px-2 py-1 text-center text-xs font-medium text-amber-600">
          接管中 · 你的鼠标/键盘正操作 Agent 的浏览器会话
        </div>
      )}
      <div className="flex-1 overflow-hidden rounded border border-border bg-muted">
        <canvas
          ref={canvasRef}
          tabIndex={takeover ? 0 : -1}
          className="h-full w-full object-contain"
          style={{ pointerEvents: takeover ? 'auto' : 'none', cursor: takeover ? 'crosshair' : 'default' }}
          onMouseMove={onMouseMove}
          onMouseDown={onMouseDown}
          onMouseUp={onMouseUp}
          onClick={onClick}
          onWheel={onWheel}
          onKeyDown={onKeyDown}
          onKeyUp={onKeyUp}
        />
      </div>
      <details className="max-h-40 overflow-auto rounded border border-border p-2 text-xs">
        <summary className="cursor-pointer text-muted-foreground">观测树（{elements.length}）</summary>
        <ul className="mt-1 space-y-0.5 font-mono">
          {elements.map((e) => (
            <li key={e.ref} className="text-foreground">[{e.ref}] &lt;{e.role}&gt; {e.name}</li>
          ))}
        </ul>
      </details>
    </div>
  )
}

// mouseButtonName 把 DOM MouseEvent.button (0/1/2) 映射到后端 button 名。
function mouseButtonName(button: number): string {
  switch (button) {
    case 1: return 'middle'
    case 2: return 'right'
    default: return 'left'
  }
}
```

- [ ] **Step 4: 跑测试确认通过**

Run: `cd legion/legionAgentGUI/frontend && npx vitest run src/components/BrowserView.test.tsx`
Expected: PASS。

- [ ] **Step 5: 全前端门槛**

Run: `cd legion/legionAgentGUI/frontend && npx vitest run && npx tsc --noEmit && npm run build`
Expected: 全 passed、tsc 无错、build 成功。

- [ ] **Step 6: 提交**

```bash
cd legion/legionAgentGUI && git add frontend/src/components/BrowserView.tsx frontend/src/components/BrowserView.test.tsx && git commit -m "feat(gui): browser view takeover — capture input and inject"
```

---

### Task F4: 前端收尾 — 手动验证 + 开 PR

- [ ] **Step 1: wails dev 手动验证（见 GUI 踩坑存档）**

启 `wails dev`，触发 Agent 一次浏览（有 session_opened 事件）→ status 栏切到 BrowserView → 点"接管" → 横幅出现、canvas 变可交互 → 在 canvas 点击/输入 → screencast 帧刷新显示操作生效 → 点"退出接管" → 恢复只读。观察 `console` 无 inject 报错、后端日志无 4xx。

> vitest 必须在 `frontend/` 目录跑（父目录另一套无 jsdom 的配置会假失败，见存档）。

- [ ] **Step 2: 推分支 + 开 PR**

```bash
cd legion/legionAgentGUI && git push -u origin HEAD
```
PR 标题 `feat(gui): browser takeover — capture and inject input`；正文引用 spec + 后端 PR。后端 PR 合入后再合本 PR。CI 绿 + 手动验证通过。

- [ ] **Step 3: code review**

用 `superpowers:requesting-code-review`：重点审 canvas 键盘焦点（tabIndex 切换）、接管开关失败回滚（当前 toggle 失败不置 store，UI 不误显接管态——确认无误）、mousemove 节流是否用单调 timeStamp、`GetBrowserEndpoint` 每次事件都调是否过频（如需可缓存 endpoint，但 token 可能轮换——评估后决定）。

---

## Self-Review（对照 spec）

**Spec coverage：**
- §1 决策（注入 go-rod 而非 webview）→ 架构前提，不需任务（Global Constraints 已固化"注入真实会话"）。✅
- §2 数据流（进入接管/注入/退出）→ B1(SetTakeover)/B5(端点)/F3(开关+捕获)/F1(POST)。✅
- §3.1 Session.takeover + 门控 → B1/B2；InputEvent + InjectInput + 校验 → B3/B4；HTTP 端点 → B5。✅
- §3.2 前端开关+捕获+映射+节流+归一化 → F1/F2/F3。✅
- §4 错误处理：input POST 失败前端 console.error 保持态 → F3 send/toggle catch；takeover 切换失败回滚 → F3(失败不置 store)；未知 session 404 → B5(SetTakeover err→404)；校验失败整批拒 → B3/B4；未接管调 /input 409 → B5(CodeTakeover→409)；Agent 写动作遇 takeover 返回码 → B2；session 关闭清标志 → B6。✅
- §5 测试：SetTakeover/门控/映射/校验单测 → B1/B2/B3；HTTP handler httptest → B5；chromium 集成注入 → B4；前端 vitest 映射/节流/开关 → F1/F3。✅
- §6 范围边界：不做拖拽/多 tab/WebRTC/webview2/set-of-marks/复杂 IME —— 计划未纳入，char 走 InsertText 基础覆盖。✅

**Placeholder scan：** input.go 的 import 块有意标注"占位→替换为真实 import"，已给出确切替换内容（非 TODO）。keyToInputKey 的 `namedKeys` 附"先确认常量存在"的确切 grep 命令。均非空泛占位。✅

**Type consistency：** `SetTakeover(string,bool)error`、`InjectInput(string,[]InputEvent)error`、`takeoverOf(*Session)bool` 在 B1/B4/B5 一致；前端 `InputEvent` 字段（type/x/y/button/deltaX/deltaY/key/text）与后端 json tag 一致；`postInput/postTakeover/mapToNormalized/Throttler` 在 F1 定义、F3 消费一致；store `takeover/setTakeover` 在 F2 定义、F3 消费一致。✅

---

## 相关文档

- [[design-agent-browser-takeover-001]] — 实现来源 spec
- [[reference-agent-browser-continue-001]] — 进度存档与接续指南（接管 = Phase 7）
- [[legion-git-repo-topology]] — 三仓拓扑（后端/GUI 分属独立 git 仓）
