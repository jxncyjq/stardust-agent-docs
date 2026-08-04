# Agent 内置浏览器 Phase 2（流式观测）Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 让前端（GUI/TUI）实时看见 Agent 的浏览过程——通过一条 SSE 长连接 `/v1/browser/sessions/{id}/stream` 推送三类事件（observation / frame / progress），支持 `Last-Event-ID` 断线重连（帧可丢、状态事件补发）。

**Architecture:** 在 `internal/browser` 加一个**每会话 Hub**（广播 + 单调 seq + 状态事件环形缓冲），`RuntimeAPI` 增 `Subscribe`。Runtime 在 Open/Read/Click/Type 时把 observation+progress 发到该会话 Hub；screencast 帧由 go-rod CDP `Page.startScreencast` 驱动，仅在有订阅者时开、限帧率。`internal/server` 加一个 SSE handler `/v1/browser/sessions/{id}/stream`，订阅 Runtime、把 `StreamEvent` 序列化成 SSE（`event:`/`id:`/`data:`），并按 `Last-Event-ID` 补发缓冲的状态事件、跳过已送帧。复用 Phase 1 的共享 `sharedBrowser` Runtime，接进 `server.Config`。

**Tech Stack:** Go 1.26、go-rod（CDP screencast）、现有 `internal/server` SSE 基础设施（对齐 `handleEvents`/`writeSSEEvent` 于 `internal/server/events.go`）。

**范围边界（本 plan 不做）:** 二进制 WS 旁路（帧走 base64 SSE 即可，spec §3.2 说将来再加）；接管模式（Phase 7）；持久化/TTL（Phase 3）；鉴权强化（Phase 5，本 plan 沿用现有 server 的鉴权中间件，不新增）。前端 `<canvas>` 渲染属 GUI 仓库工作，不在本后端 plan。

**关联文档:** spec `docs/superpowers/specs/2026-08-04-agent-browser-design.md`（§3.2/§4.3/§7）；Phase 1 plan `docs/superpowers/plans/2026-08-04-agent-browser-phase1.md`。

---

## 锁定的设计决策（spec 未定死、本 plan 定）

| 决策 | 选择 | 理由 |
|---|---|---|
| SSE 路径 | `/v1/browser/sessions/{id}/stream` | 保 `/v1` 前缀惯例；`{id}` 是**浏览器会话**id，与 `/v1/sessions`（对话会话）命名空间隔开，不重载 |
| StreamEvent 形状 | `{Type, Seq uint64, Data any}`，SSE 里 `event:<Type>` `id:<Seq>` `data:<json>` | 对齐 spec §4.3 与现有 `writeSSEEvent` |
| Seq | **每会话单调递增**，跨三类事件共享一个计数 | `Last-Event-ID` 用它既能跳已送帧、又能补状态 |
| 环形缓冲 | 只缓 **status 事件**（observation/progress）最近 N 条；frame **不缓** | spec §3.2「帧可丢，状态不可丢」 |
| 帧率 | screencast `EveryNthFrame` + 最小间隔节流（默认 ~8fps）；**仅有订阅者时开**，最后一个断开就停 | spec §3.2 限帧率/按需开关 |
| 帧编码 | JPEG（go-rod screencast 原生）→ base64 塞进 SSE `data` | spec §3.2 接受 +33% |

---

## 文件结构

| 文件 | 职责 | 状态 |
|---|---|---|
| `internal/browser/stream.go` | `StreamEvent` 类型 + 每会话 `Hub`（Subscribe/Publish/seq/环形缓冲/ReplaySince） | 创建（Task 1） |
| `internal/browser/api.go` | `RuntimeAPI` 增 `Subscribe`；`StreamEvent` 引用 | 修改（Task 2） |
| `internal/browser/runtime.go` | 每会话 Hub 注册表；Open/Read/Click/Type 发 observation+progress；Subscribe 实现 | 修改（Task 2） |
| `internal/browser/api_test.go` · `internal/tool/browser_test.go` · `internal/runtime/toolauth_drift_test.go` | 三个 fake 补 `Subscribe` 以满足接口 | 修改（Task 2） |
| `internal/browser/screencast.go` | go-rod `Page.startScreencast` → frame 事件入 Hub，限帧率，按订阅者开关 | 创建（Task 4） |
| `internal/server/browser_stream.go` | SSE handler：订阅 Runtime、序列化 StreamEvent、Last-Event-ID 补发/跳帧 | 创建（Task 3） |
| `internal/server/http.go` | 加路由 + `Config.Browser` 字段 | 修改（Task 3） |
| `internal/cli/command.go`（+ `internal/app/app.go` 若建 server） | 把 `sharedBrowser` 接进 `server.Config{Browser: ...}` | 修改（Task 3） |

---

## Task 1: StreamEvent + 每会话 Hub（纯逻辑，TDD）

**Files:**
- Create: `internal/browser/stream.go`
- Test: `internal/browser/stream_test.go`

- [ ] **Step 1: 写失败测试**

Create `internal/browser/stream_test.go`:

```go
package browser

import (
	"testing"
)

func TestHubBroadcastToMultipleSubscribers(t *testing.T) {
	h := NewHub(8)
	ch1, cancel1 := h.Subscribe()
	defer cancel1()
	ch2, cancel2 := h.Subscribe()
	defer cancel2()

	h.Publish(StreamEvent{Type: EventProgress, Data: map[string]any{"action": "click"}})

	for i, ch := range []<-chan StreamEvent{ch1, ch2} {
		select {
		case ev := <-ch:
			if ev.Type != EventProgress || ev.Seq != 1 {
				t.Fatalf("sub %d got %+v, want progress seq=1", i, ev)
			}
		default:
			t.Fatalf("sub %d received nothing", i)
		}
	}
}

func TestHubSeqMonotonicAcrossTypes(t *testing.T) {
	h := NewHub(8)
	ch, cancel := h.Subscribe()
	defer cancel()
	h.Publish(StreamEvent{Type: EventObservation})
	h.Publish(StreamEvent{Type: EventFrame})
	h.Publish(StreamEvent{Type: EventProgress})
	seqs := []uint64{(<-ch).Seq, (<-ch).Seq, (<-ch).Seq}
	if seqs[0] != 1 || seqs[1] != 2 || seqs[2] != 3 {
		t.Fatalf("seqs = %v, want 1,2,3", seqs)
	}
}

// 只缓 status（observation/progress），frame 不缓；ReplaySince 补发 seq>lastID 的 status。
func TestHubReplaySinceReturnsMissedStatusNotFrames(t *testing.T) {
	h := NewHub(8)
	h.Publish(StreamEvent{Type: EventObservation}) // seq1
	h.Publish(StreamEvent{Type: EventFrame})       // seq2 (不缓)
	h.Publish(StreamEvent{Type: EventProgress})    // seq3
	replay := h.ReplaySince(1)                      // 要 seq>1 的 status
	if len(replay) != 1 || replay[0].Type != EventProgress || replay[0].Seq != 3 {
		t.Fatalf("replay = %+v, want just progress seq=3", replay)
	}
}

func TestHubRingBufferEvictsOldStatus(t *testing.T) {
	h := NewHub(2) // 只留最近 2 条 status
	for i := 0; i < 5; i++ {
		h.Publish(StreamEvent{Type: EventProgress})
	}
	replay := h.ReplaySince(0) // 要全部（>0）
	if len(replay) != 2 {
		t.Fatalf("buffered %d, want 2 (ring cap)", len(replay))
	}
	if replay[0].Seq != 4 || replay[1].Seq != 5 {
		t.Fatalf("kept seqs %d,%d, want 4,5", replay[0].Seq, replay[1].Seq)
	}
}

func TestHubSubscriberCountAndStopSignal(t *testing.T) {
	h := NewHub(8)
	if h.SubscriberCount() != 0 {
		t.Fatal("want 0 subscribers initially")
	}
	_, cancel := h.Subscribe()
	if h.SubscriberCount() != 1 {
		t.Fatal("want 1 after subscribe")
	}
	cancel()
	if h.SubscriberCount() != 0 {
		t.Fatal("want 0 after cancel")
	}
}
```

- [ ] **Step 2: 跑，确认失败**

Run: `go test ./internal/browser/ -run TestHub -v`
Expected: FAIL（`NewHub`/`StreamEvent`/`EventProgress` 等未定义）。

- [ ] **Step 3: 实现**

Create `internal/browser/stream.go`:

```go
package browser

import "sync"

// EventType 是流事件的三类（spec §4.3）。
type EventType string

const (
	EventObservation EventType = "observation" // 观测更新（JSON）
	EventFrame       EventType = "frame"       // 截图帧（base64 JPEG）
	EventProgress    EventType = "progress"    // 动作进度/状态
)

// isStatus 报告该类型是否为「不可丢」的状态事件（进缓冲、可 replay）。frame 可丢。
func (t EventType) isStatus() bool { return t == EventObservation || t == EventProgress }

// StreamEvent 是推给前端的一条事件。Seq 由 Hub 分配，全会话单调。
type StreamEvent struct {
	Type EventType `json:"type"`
	Seq  uint64    `json:"seq"`
	Data any       `json:"data,omitempty"`
}

// Hub 是一个浏览器会话的事件广播器：向所有订阅者扇出，分配单调 seq，
// 并把最近 N 条 status 事件留在环形缓冲里供 Last-Event-ID 重连补发。
// frame 不入缓冲（可丢）。并发安全。
type Hub struct {
	mu       sync.Mutex
	seq      uint64
	subs     map[int]chan StreamEvent
	nextSub  int
	ring     []StreamEvent // 只存 status
	ringCap  int
}

// NewHub 建一个 status 环形缓冲容量为 ringCap 的 Hub。
func NewHub(ringCap int) *Hub {
	if ringCap <= 0 {
		ringCap = 64
	}
	return &Hub{subs: make(map[int]chan StreamEvent), ringCap: ringCap}
}

// Subscribe 返回一个新订阅通道与取消函数。通道有缓冲，满则丢最旧帧级事件由发送方决定——
// 这里用带缓冲通道 + 非阻塞发送（Publish 里），慢订阅者不拖垮广播。
func (h *Hub) Subscribe() (<-chan StreamEvent, func()) {
	h.mu.Lock()
	defer h.mu.Unlock()
	id := h.nextSub
	h.nextSub++
	ch := make(chan StreamEvent, 64)
	h.subs[id] = ch
	return ch, func() {
		h.mu.Lock()
		defer h.mu.Unlock()
		if c, ok := h.subs[id]; ok {
			delete(h.subs, id)
			close(c)
		}
	}
}

// Publish 分配 seq、缓冲 status、非阻塞扇出给所有订阅者。返回分配的 seq。
func (h *Hub) Publish(ev StreamEvent) uint64 {
	h.mu.Lock()
	defer h.mu.Unlock()
	h.seq++
	ev.Seq = h.seq
	if ev.Type.isStatus() {
		h.ring = append(h.ring, ev)
		if len(h.ring) > h.ringCap {
			h.ring = h.ring[len(h.ring)-h.ringCap:]
		}
	}
	for _, ch := range h.subs {
		select {
		case ch <- ev:
		default: // 慢订阅者：丢这条给它（帧可丢；status 它可靠 ReplaySince 补）
		}
	}
	return h.seq
}

// ReplaySince 返回缓冲里 seq>lastID 的 status 事件，供重连补发。
func (h *Hub) ReplaySince(lastID uint64) []StreamEvent {
	h.mu.Lock()
	defer h.mu.Unlock()
	var out []StreamEvent
	for _, ev := range h.ring {
		if ev.Seq > lastID {
			out = append(out, ev)
		}
	}
	return out
}

// SubscriberCount 返回当前订阅者数（screencast 按它开关）。
func (h *Hub) SubscriberCount() int {
	h.mu.Lock()
	defer h.mu.Unlock()
	return len(h.subs)
}
```

- [ ] **Step 4: 跑，确认通过**

Run: `go test ./internal/browser/ -run TestHub -v`
Expected: PASS（全部子测试）。也跑 `go test -race ./internal/browser/ -run TestHub`（若 -race 因 CGO/gcc 缺失报工具链错误而非数据竞争，退回不带 -race 并说明）。

- [ ] **Step 5: Commit**

```bash
git add internal/browser/stream.go internal/browser/stream_test.go
git commit -m "feat(browser): per-session stream hub with seq + status ring buffer"
```
（提交信息附空行 + `Co-Authored-By: Claude Opus 4.8 <noreply@anthropic.com>`。）

---

## Task 2: RuntimeAPI.Subscribe + 会话 Hub 注册表 + 动作发事件

**Files:**
- Modify: `internal/browser/api.go`（接口加 `Subscribe`）
- Modify: `internal/browser/runtime.go`（Hub 注册表 + emit + Subscribe 实现）
- Modify: `internal/browser/api_test.go`（fakeRuntime 补 Subscribe）
- Modify: `internal/tool/browser_test.go`（fakeBrowserRuntime 补 Subscribe）
- Modify: `internal/runtime/toolauth_drift_test.go`（driftGuardBrowserRuntime 补 Subscribe）
- Test: `internal/browser/runtime_subscribe_test.go`（不碰 Chromium）

- [ ] **Step 1: 写失败测试（Subscribe 收到 emit 的事件，无 Chromium）**

Create `internal/browser/runtime_subscribe_test.go`:

```go
package browser

import (
	"testing"
)

// 不启动 Chromium：直接建 Runtime 的会话 Hub，验证 Subscribe 拿到发布的事件。
// hubFor 是内部方法，按会话惰性建 Hub；emitProgress 往会话 Hub 发 progress。
func TestRuntimeSubscribeReceivesEmittedProgress(t *testing.T) {
	r := &Runtime{sessions: NewSessionStore(), hubs: newHubRegistry()}
	sess := r.sessions.Create("t1")

	ch, cancel, err := r.Subscribe(sess.ID)
	if err != nil {
		t.Fatalf("Subscribe: %v", err)
	}
	defer cancel()

	r.emitProgress(sess.ID, "click", "done", "e1")

	select {
	case ev := <-ch:
		if ev.Type != EventProgress {
			t.Fatalf("got %v, want progress", ev.Type)
		}
	default:
		t.Fatal("no event received")
	}
}

func TestRuntimeSubscribeUnknownSession(t *testing.T) {
	r := &Runtime{sessions: NewSessionStore(), hubs: newHubRegistry()}
	if _, _, err := r.Subscribe("nope"); err == nil {
		t.Fatal("expected error for unknown session")
	}
}
```

- [ ] **Step 2: 跑，确认失败**

Run: `go test ./internal/browser/ -run TestRuntimeSubscribe -v`
Expected: FAIL（`Runtime.hubs`/`newHubRegistry`/`Subscribe`/`emitProgress` 未定义）。

- [ ] **Step 3: 接口加 Subscribe**

Edit `internal/browser/api.go` — 在 `RuntimeAPI` 接口里加（放到 Close 之后）：

```go
	// Subscribe 返回一个会话的流事件通道与取消函数。用于前端观看 Agent 浏览过程。
	// 未知会话返回错误。
	Subscribe(sessionID string) (<-chan StreamEvent, func(), error)
```

- [ ] **Step 4: Runtime 实现 Hub 注册表 + emit + Subscribe**

Edit `internal/browser/runtime.go`:

在 `Runtime` 结构体里加字段：
```go
	hubs *hubRegistry
```
在 `NewRuntime` 里初始化：`hubs: newHubRegistry(),`（与 `sessions`/`mgr` 并列）。

在文件里加（新的内部类型与方法）：
```go
// hubRegistry 按会话 id 惰性持有 Hub。并发安全。
type hubRegistry struct {
	mu   sync.Mutex
	byID map[string]*Hub
}

func newHubRegistry() *hubRegistry { return &hubRegistry{byID: make(map[string]*Hub)} }

func (hr *hubRegistry) get(sessionID string) *Hub {
	hr.mu.Lock()
	defer hr.mu.Unlock()
	h, ok := hr.byID[sessionID]
	if !ok {
		h = NewHub(64)
		hr.byID[sessionID] = h
	}
	return h
}

func (hr *hubRegistry) drop(sessionID string) {
	hr.mu.Lock()
	defer hr.mu.Unlock()
	delete(hr.byID, sessionID)
}

// Subscribe 实现 RuntimeAPI：会话必须存在。
func (r *Runtime) Subscribe(sessionID string) (<-chan StreamEvent, func(), error) {
	if _, ok := r.sessions.Get(sessionID); !ok {
		return nil, nil, NewBrowserError(CodeContextEvicted, "unknown session "+sessionID)
	}
	hub := r.hubs.get(sessionID)
	ch, cancel := hub.Subscribe()
	return ch, cancel, nil
}

// emitProgress 往会话 Hub 发一条 progress 事件（无订阅者也安全——Hub 仍分配 seq/缓冲）。
func (r *Runtime) emitProgress(sessionID, action, status, ref string) {
	r.hubs.get(sessionID).Publish(StreamEvent{
		Type: EventProgress,
		Data: map[string]any{"action": action, "status": status, "ref": ref},
	})
}

// emitObservation 往会话 Hub 发一条 observation 事件。
func (r *Runtime) emitObservation(sessionID string, obs Observation) {
	r.hubs.get(sessionID).Publish(StreamEvent{Type: EventObservation, Data: obs})
}
```

在动作方法里发事件（在各自 `WithLock` 内、成功产出 observation 后）：
- `Open`：成功后 `r.emitObservation(sess.ID, obs)`；返回前 `r.emitProgress(sess.ID, "open", "done", "")`。
- `Read`：`r.emitObservation(req.SessionID, obs)`。
- `Click`：成功后 `r.emitProgress(req.SessionID, "click", "done", req.Ref)` + `r.emitObservation(req.SessionID, obs)`。
- `Type`：成功后 `r.emitProgress(req.SessionID, "type", "done", req.Ref)` + `r.emitObservation(req.SessionID, obs)`。
- `Close`：`r.hubs.drop(req.SessionID)`（释放 Hub）。
保持这些 emit 不改变既有返回值/错误路径——只在成功分支追加。若 `sync` 尚未 import（应已有），确保 import 存在。

- [ ] **Step 5: 三个 fake 补 Subscribe（否则编译断）**

加接口方法会让 Phase 1 的三个 `RuntimeAPI` 假实现编译失败。逐个补：

`internal/browser/api_test.go` 的 `fakeRuntime`：
```go
func (fakeRuntime) Subscribe(string) (<-chan StreamEvent, func(), error) { return nil, func() {}, nil }
```
`internal/tool/browser_test.go` 的 `fakeBrowserRuntime`：
```go
func (f *fakeBrowserRuntime) Subscribe(string) (<-chan browser.StreamEvent, func(), error) {
	return nil, func() {}, nil
}
```
`internal/runtime/toolauth_drift_test.go` 的 `driftGuardBrowserRuntime`：
```go
func (driftGuardBrowserRuntime) Subscribe(string) (<-chan browser.StreamEvent, func(), error) {
	return nil, func() {}, nil
}
```

- [ ] **Step 6: 跑，确认通过 + 不破坏 Phase 1**

Run:
```
go test ./internal/browser/ -run 'TestRuntimeSubscribe|TestHub' -v
go build ./...
go test ./internal/browser/ ./internal/tool/ ./internal/runtime/ 2>&1 | tail -8
```
Expected: 新测试 PASS；全部编译通过；Phase 1 测试仍绿（含 drift）。

- [ ] **Step 7: Commit**

```bash
git add internal/browser/api.go internal/browser/runtime.go internal/browser/runtime_subscribe_test.go internal/browser/api_test.go internal/tool/browser_test.go internal/runtime/toolauth_drift_test.go
git commit -m "feat(browser): RuntimeAPI.Subscribe + emit observation/progress to session hub"
```

---

## Task 3: SSE handler `/v1/browser/sessions/{id}/stream` + 路由 + 接线

**Files:**
- Create: `internal/server/browser_stream.go`
- Modify: `internal/server/http.go`（`Config` 加 `Browser`；路由加一条）
- Modify: `internal/cli/command.go`（`server.Config{... Browser: sharedBrowser}`，约 line 2542）
- Test: `internal/server/browser_stream_test.go`（httptest + 假 Subscribe，无 Chromium）

- [ ] **Step 1: 写失败测试（SSE 线格式 + Last-Event-ID 补发）**

Create `internal/server/browser_stream_test.go`:

```go
package server

import (
	"context"
	"net/http"
	"net/http/httptest"
	"strings"
	"testing"
	"time"

	"github.com/stardust/legion-agent/internal/browser"
)

// fakeStreamer 满足 BrowserStreamer：Subscribe 返回一个我们手工喂数据的通道。
type fakeStreamer struct {
	ch      chan browser.StreamEvent
	replay  []browser.StreamEvent
	lastReq uint64
}

func (f *fakeStreamer) Subscribe(sessionID string) (<-chan browser.StreamEvent, func(), error) {
	return f.ch, func() {}, nil
}
func (f *fakeStreamer) ReplaySince(sessionID string, lastID uint64) []browser.StreamEvent {
	f.lastReq = lastID
	return f.replay
}

func TestBrowserStreamWritesSSEEvents(t *testing.T) {
	f := &fakeStreamer{ch: make(chan browser.StreamEvent, 4)}
	srv := &HTTPServer{browser: f}

	f.ch <- browser.StreamEvent{Type: browser.EventProgress, Seq: 7, Data: map[string]any{"action": "click"}}

	req := httptest.NewRequest(http.MethodGet, "/v1/browser/sessions/sess-1/stream", nil)
	ctx, cancel := context.WithCancel(req.Context())
	req = req.WithContext(ctx)
	rec := httptest.NewRecorder()

	go func() { time.Sleep(50 * time.Millisecond); cancel() }()
	srv.handleBrowserStream(rec, req)

	body := rec.Body.String()
	if rec.Code != http.StatusOK {
		t.Fatalf("status=%d", rec.Code)
	}
	if !strings.Contains(body, "event: progress") || !strings.Contains(body, "id: 7") || !strings.Contains(body, `"action":"click"`) {
		t.Fatalf("SSE wire missing pieces:\n%s", body)
	}
}

func TestBrowserStreamReplaysOnLastEventID(t *testing.T) {
	f := &fakeStreamer{
		ch:     make(chan browser.StreamEvent),
		replay: []browser.StreamEvent{{Type: browser.EventObservation, Seq: 5}},
	}
	srv := &HTTPServer{browser: f}

	req := httptest.NewRequest(http.MethodGet, "/v1/browser/sessions/sess-1/stream", nil)
	req.Header.Set("Last-Event-ID", "4")
	ctx, cancel := context.WithCancel(req.Context())
	req = req.WithContext(ctx)
	rec := httptest.NewRecorder()
	go func() { time.Sleep(50 * time.Millisecond); cancel() }()
	srv.handleBrowserStream(rec, req)

	if f.lastReq != 4 {
		t.Fatalf("ReplaySince got lastID=%d, want 4", f.lastReq)
	}
	if !strings.Contains(rec.Body.String(), "id: 5") {
		t.Fatalf("expected replayed obs seq=5:\n%s", rec.Body.String())
	}
}
```

- [ ] **Step 2: 跑，确认失败**

Run: `go test ./internal/server/ -run TestBrowserStream -v`
Expected: FAIL（`HTTPServer.browser`/`handleBrowserStream`/`BrowserStreamer` 未定义）。

- [ ] **Step 3: 实现 handler**

Create `internal/server/browser_stream.go`:

```go
package server

import (
	"encoding/json"
	"fmt"
	"net/http"
	"strconv"
	"strings"

	"github.com/stardust/legion-agent/internal/browser"
)

// BrowserStreamer 是 SSE handler 依赖的最小接口：Subscribe（RuntimeAPI 也有）
// 外加 ReplaySince（仅具体 *browser.Runtime 有，用于 Last-Event-ID 补发）。
// 故 *browser.Runtime 满足它，而 RuntimeAPI 接口本身不满足——这是有意的，
// 避免给三个 RuntimeAPI fake 增加 ReplaySince 负担。
type BrowserStreamer interface {
	Subscribe(sessionID string) (<-chan browser.StreamEvent, func(), error)
	ReplaySince(sessionID string, lastID uint64) []browser.StreamEvent
}

// parseBrowserSessionID 从 /v1/browser/sessions/{id}/stream 抽 id。
func parseBrowserSessionID(path string) (string, bool) {
	const prefix = "/v1/browser/sessions/"
	const suffix = "/stream"
	if !strings.HasPrefix(path, prefix) || !strings.HasSuffix(path, suffix) {
		return "", false
	}
	id := strings.TrimSuffix(strings.TrimPrefix(path, prefix), suffix)
	if id == "" || strings.Contains(id, "/") {
		return "", false
	}
	return id, true
}

func (s *HTTPServer) handleBrowserStream(w http.ResponseWriter, r *http.Request) {
	if s.browser == nil {
		writeError(w, http.StatusServiceUnavailable, "browser runtime is unavailable")
		return
	}
	sessionID, ok := parseBrowserSessionID(r.URL.Path)
	if !ok {
		writeError(w, http.StatusNotFound, "bad browser stream path")
		return
	}
	ch, cancel, err := s.browser.Subscribe(sessionID)
	if err != nil {
		writeError(w, http.StatusNotFound, err.Error())
		return
	}
	defer cancel()

	w.Header().Set("Content-Type", "text/event-stream")
	w.Header().Set("Cache-Control", "no-cache")
	w.Header().Set("Connection", "keep-alive")
	w.WriteHeader(http.StatusOK)
	flush := func() {
		if f, ok := w.(http.Flusher); ok {
			f.Flush()
		}
	}
	flush()

	// Last-Event-ID 补发缓冲的 status 事件（帧不补）。
	if lastID := parseLastEventID(r); lastID > 0 {
		for _, ev := range s.browser.ReplaySince(sessionID, lastID) {
			if err := writeBrowserSSE(w, ev); err != nil {
				return
			}
		}
		flush()
	}

	for {
		select {
		case ev, ok := <-ch:
			if !ok {
				return
			}
			if err := writeBrowserSSE(w, ev); err != nil {
				return
			}
			flush()
		case <-r.Context().Done():
			return
		}
	}
}

func parseLastEventID(r *http.Request) uint64 {
	raw := r.Header.Get("Last-Event-ID")
	if raw == "" {
		raw = r.URL.Query().Get("lastEventId")
	}
	n, err := strconv.ParseUint(strings.TrimSpace(raw), 10, 64)
	if err != nil {
		return 0
	}
	return n
}

func writeBrowserSSE(w http.ResponseWriter, ev browser.StreamEvent) error {
	data, err := json.Marshal(ev.Data)
	if err != nil {
		return err
	}
	if _, err := fmt.Fprintf(w, "event: %s\n", ev.Type); err != nil {
		return err
	}
	if _, err := fmt.Fprintf(w, "id: %d\n", ev.Seq); err != nil {
		return err
	}
	_, err = fmt.Fprintf(w, "data: %s\n\n", data)
	return err
}
```

- [ ] **Step 4: `HTTPServer`/`Config` 加 browser 字段 + 路由**

Edit `internal/server/http.go`:
- 在 `Config` 结构体加：`Browser BrowserStreamer`（放 `PlatformEvents` 附近）。
- 在 `HTTPServer` 结构体加字段：`browser BrowserStreamer`。
- 在 `NewHTTPServer`（构造 `HTTPServer{...}` 处）把 `browser: cfg.Browser` 接上（对齐 `platformEvents: cfg.PlatformEvents`）。
- 在 `ServeHTTP` 的路由 switch 里，`/v1/events` 那条附近加：
```go
	case r.Method == http.MethodGet && strings.HasPrefix(r.URL.Path, "/v1/browser/sessions/") && strings.HasSuffix(r.URL.Path, "/stream"):
		s.handleBrowserStream(rec, r)
```
（`strings` 已在 http.go import。）

- [ ] **Step 5: 让 `*browser.Runtime` 满足 `BrowserStreamer`（补 ReplaySince）**

`BrowserStreamer` 需要 `ReplaySince(sessionID, lastID)`。给 `*browser.Runtime` 加该方法（`internal/browser/runtime.go`）：
```go
// ReplaySince 返回会话 Hub 中 seq>lastID 的缓冲 status 事件，供 SSE 断线重连补发。
func (r *Runtime) ReplaySince(sessionID string, lastID uint64) []StreamEvent {
	if _, ok := r.sessions.Get(sessionID); !ok {
		return nil
	}
	return r.hubs.get(sessionID).ReplaySince(lastID)
}
```
（`RuntimeAPI` 接口**不**加 ReplaySince——它是 server 侧 `BrowserStreamer` 的要求，`*Runtime` 直接满足即可，避免污染三个 fake。server.Config 里 `Browser BrowserStreamer` 由 cli 传入 `sharedBrowser`，而 `sharedBrowser` 的静态类型是 `browser.RuntimeAPI`——所以在 cli 接线处需断言/持有具体 `*browser.Runtime` 或让 `sharedBrowser` 以满足 `BrowserStreamer` 的类型传入。见 Step 6。）

- [ ] **Step 6: 接线 cli/command.go**

在 `internal/cli/command.go` 的 `server.Config{...}`（约 line 2542）加 `Browser:` 字段。`sharedBrowser` 现声明为 `browser.RuntimeAPI`（line 2290）——它需同时满足 `server.BrowserStreamer`（有 `ReplaySince`）。做法：把 `Browser` 字段赋值为一个类型断言后的具体值，或直接传 `sharedBrowser` 并让 `server.Config.Browser` 类型为 `browser.RuntimeAPI` + 在 server 内对可选的 ReplaySince 做能力断言。**采用更简单的做法**：
- 在 `server.Config` 里 `Browser BrowserStreamer`；
- cli 侧：`sharedBrowser` 具体值就是 `*browser.Runtime`（`NewRuntime` 返回 `*Runtime`）。把持有它的变量类型从 `browser.RuntimeAPI` 保留，但在传给 server 时用类型断言：
```go
	var browserStream server.BrowserStreamer
	if bs, ok := sharedBrowser.(server.BrowserStreamer); ok {
		browserStream = bs
	}
	// ... server.Config{ Browser: browserStream, ... }
```
`BrowserStreamer`（`browser_stream.go` 里定义，首字母大写导出）= `Subscribe` + `ReplaySince`。`*browser.Runtime` 同时有 `Subscribe`（接口方法，Task 2）与 `ReplaySince`（Step 5），故断言 `sharedBrowser.(server.BrowserStreamer)` 成立；`sharedBrowser` 为 nil 时 `Browser` 为 nil，handler 返回 503。`RuntimeAPI` 接口本身**不**含 `ReplaySince`，三个 fake 不受影响。

- [ ] **Step 7: 跑，确认通过**

Run:
```
go test ./internal/server/ -run TestBrowserStream -v
go build ./...
go test ./internal/server/ ./internal/cli/ ./internal/browser/ 2>&1 | tail -10
```
Expected: SSE 测试 PASS（线格式 + Last-Event-ID 补发）；全量编译通过；相关包绿。

- [ ] **Step 8: Commit**

```bash
git add internal/server/browser_stream.go internal/server/http.go internal/browser/runtime.go internal/cli/command.go internal/server/browser_stream_test.go
git commit -m "feat(server): SSE /v1/browser/sessions/{id}/stream with Last-Event-ID replay"
```

---

## Task 4: Screencast 帧（go-rod，按订阅者开关，限帧率）

**Files:**
- Create: `internal/browser/screencast.go`
- Modify: `internal/browser/runtime.go`（订阅者 0→1 起、1→0 停 screencast）
- Test: `internal/browser/screencast_integration_test.go`（`//go:build chromium`）

- [ ] **Step 1: 查 go-rod screencast API**

Run（了解签名）:
```
go doc github.com/go-rod/rod/lib/proto PageStartScreencast
go doc github.com/go-rod/rod/lib/proto PageScreencastFrame
go doc github.com/go-rod/rod/lib/proto PageScreencastFrameAck
go doc github.com/go-rod/rod Page.EachEvent
```
据此调整 Step 3 代码字段名（`Format`/`Quality`/`MaxWidth`/`MaxHeight`/`EveryNthFrame`；帧事件的 `Data`(base64) 与 `SessionID`/`Metadata`；ack 用 `SessionID`）。

- [ ] **Step 2: 写集成测试（chromium）**

Create `internal/browser/screencast_integration_test.go`:

```go
//go:build chromium

package browser

import (
	"context"
	"net/http"
	"net/http/httptest"
	"testing"
	"time"
)

func TestScreencastEmitsFramesWhenSubscribed(t *testing.T) {
	srv := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, _ *http.Request) {
		w.Header().Set("Content-Type", "text/html")
		_, _ = w.Write([]byte(`<html><body><h1>hi</h1><script>setInterval(()=>document.body.style.background=Math.random()>0.5?'#fff':'#000',100)</script></body></html>`))
	}))
	defer srv.Close()

	rt, err := NewRuntime(RuntimeConfig{Headless: true, AllowPrivateHosts: true, BinPath: systemChromeForTest()})
	if err != nil {
		t.Fatalf("NewRuntime: %v", err)
	}
	defer rt.Close(context.Background(), CloseReq{})

	open, err := rt.Open(context.Background(), OpenReq{URL: srv.URL, TaskID: "sc"})
	if err != nil {
		t.Fatalf("Open: %v", err)
	}
	ch, cancel, err := rt.Subscribe(open.SessionID)
	if err != nil {
		t.Fatalf("Subscribe: %v", err)
	}
	defer cancel()

	// 订阅后应在 2s 内收到至少一个 frame 事件。
	deadline := time.After(2 * time.Second)
	for {
		select {
		case ev := <-ch:
			if ev.Type == EventFrame {
				return // 成功
			}
		case <-deadline:
			t.Fatal("no frame event within 2s of subscribing")
		}
	}
}

// systemChromeForTest 返回本机可用 Chrome 路径（go-rod 自动下载在部分 Windows 环境损坏）。
func systemChromeForTest() string {
	return `C:\Program Files\Google\Chrome\Application\chrome.exe`
}
```

- [ ] **Step 3: 实现 screencast**

Create `internal/browser/screencast.go`:

```go
package browser

import (
	"sync"
	"time"

	"github.com/go-rod/rod"
	"github.com/go-rod/rod/lib/proto"
)

// screencaster 把一个 page 的 CDP screencast 帧节流后发到会话 Hub。
// 仅在有订阅者时运行；Stop 后可再次 Start（换页时）。
type screencaster struct {
	mu       sync.Mutex
	running  bool
	stop     func()
	minInterval time.Duration // 限帧率：两帧之间最小间隔
	lastSent time.Time
}

func newScreencaster(fps int) *screencaster {
	if fps <= 0 {
		fps = 8
	}
	return &screencaster{minInterval: time.Second / time.Duration(fps)}
}

// Start 在 page 上开 screencast，把节流后的帧作为 EventFrame 发到 hub。
func (s *screencaster) Start(page *rod.Page, hub *Hub) error {
	s.mu.Lock()
	defer s.mu.Unlock()
	if s.running {
		return nil
	}
	if err := (proto.PageStartScreencast{
		Format:        "jpeg",
		Quality:       gInt(60),
		EveryNthFrame: gInt(2),
	}).Call(page); err != nil {
		return NewBrowserErrorWrap(CodeContextEvicted, "start screencast", err)
	}
	// EachEvent 返回一个 stop 函数；在独立 goroutine 里消费帧。
	stop := page.EachEvent(func(e *proto.PageScreencastFrame) {
		// ack 保持流继续
		_ = proto.PageScreencastFrameAck{SessionID: e.SessionID}.Call(page)
		s.mu.Lock()
		now := time.Now()
		if now.Sub(s.lastSent) < s.minInterval {
			s.mu.Unlock()
			return // 丢帧限速
		}
		s.lastSent = now
		s.mu.Unlock()
		hub.Publish(StreamEvent{Type: EventFrame, Data: map[string]any{
			"mime": "image/jpeg",
			"b64":  e.Data, // go-rod 已是 base64 字符串
		}})
	})
	s.stop = func() {
		_ = proto.PageStopScreencast{}.Call(page)
		stop()
	}
	s.running = true
	return nil
}

func (s *screencaster) Stop() {
	s.mu.Lock()
	defer s.mu.Unlock()
	if !s.running {
		return
	}
	if s.stop != nil {
		s.stop()
	}
	s.running = false
}

// gInt 是 go-rod proto 里 *int 可选字段的便捷构造（若 go doc 显示字段是 int 而非 *int，去掉此helper直接赋值）。
func gInt(v int) *int { return &v }
```

> 注：`go doc`（Step 1）可能显示 `Quality`/`EveryNthFrame` 是 `*int` 或 `int`、`PageScreencastFrame.Data` 的类型、`EachEvent` 的确切签名（go-rod 是 `page.EachEvent(handlers ...) (stop func())`）。据实调整；`e.Data` 若是 `[]byte` 则 base64 编码后再放，若已是 string 直接用。

- [ ] **Step 4: Runtime 按订阅者开关 screencast**

Edit `internal/browser/runtime.go`：每会话关联一个 `*screencaster`。最简做法——在 `Runtime` 加 `screencasters *sync.Map`（sessionID→*screencaster），并在 `Subscribe` 里：拿到 hub 后，如果这是**第一个**订阅者（`hub.SubscriberCount()==1`）且会话有活跃 page，则 `Start`；在返回的 cancel 里，如果取消后 `SubscriberCount()==0` 则 `Stop`。

在 `Subscribe` 实现里改为：
```go
func (r *Runtime) Subscribe(sessionID string) (<-chan StreamEvent, func(), error) {
	sess, ok := r.sessions.Get(sessionID)
	if !ok {
		return nil, nil, NewBrowserError(CodeContextEvicted, "unknown session "+sessionID)
	}
	hub := r.hubs.get(sessionID)
	ch, hubCancel := hub.Subscribe()
	r.maybeStartScreencast(sess, hub)
	cancel := func() {
		hubCancel()
		if hub.SubscriberCount() == 0 {
			r.stopScreencast(sessionID)
		}
	}
	return ch, cancel, nil
}
```
加辅助：
```go
func (r *Runtime) maybeStartScreencast(sess *Session, hub *Hub) {
	if hub.SubscriberCount() != 1 {
		return // 已有别的订阅者 → screencast 已在跑
	}
	page := r.pageOf(sess) // 从 sess.ActivePage 取 *rod.Page；无则返回 nil
	if page == nil {
		return
	}
	sc := newScreencaster(r.cfg.ScreencastFPS)
	r.screencasters.Store(sess.ID, sc)
	_ = sc.Start(page, hub) // 失败不致命：观测/进度仍走 SSE
}

func (r *Runtime) stopScreencast(sessionID string) {
	if v, ok := r.screencasters.LoadAndDelete(sessionID); ok {
		v.(*screencaster).Stop()
	}
}

// pageOf 取会话活跃页的 *rod.Page（与 activePage 的断言一致），无则 nil。
func (r *Runtime) pageOf(sess *Session) *rod.Page {
	if sess == nil || sess.ActivePage == nil || sess.ActivePage.page == nil {
		return nil
	}
	p, _ := sess.ActivePage.page.(*rod.Page)
	return p
}
```
在 `RuntimeConfig` 加 `ScreencastFPS int`；`NewRuntime` 里初始化 `screencasters: &sync.Map{}`。`Close` 里 `r.stopScreencast(req.SessionID)`（在 drop hub 前）。

> 换页（Open 复用会话再导航）时活跃 page 变了：本 Phase 简化——若已有订阅者，Open 成功切页后调用 `r.restartScreencastIfActive(sess, hub)`（stop 旧、按新 page start）。加该辅助并在 `Open` 成功切 ActivePage 后调用。若嫌复杂，Phase 2 可只支持「先订阅后 Open 保持同页」，并在债务表记录换页重启为已知限制——**本 plan 采用 restart 版**以求正确。

- [ ] **Step 5: 跑集成测试（系统 Chrome）**

Run: `go test -tags chromium ./internal/browser/ -run TestScreencastEmitsFrames -v`
Expected: PASS（订阅后 2s 内收到 frame）。本机 go-rod 自动下载 Chromium 损坏，测试用 `systemChromeForTest()` 的 BinPath。
非集成门：`go build ./internal/browser/`、`go vet -tags chromium ./internal/browser/` 干净。

- [ ] **Step 6: Commit**

```bash
git add internal/browser/screencast.go internal/browser/runtime.go internal/browser/screencast_integration_test.go
git commit -m "feat(browser): CDP screencast frames on demand with fps throttle"
```

---

## Task 5: 端到端流式（chromium）——订阅经 HTTP SSE 看真实浏览

**Files:**
- Create: `internal/server/browser_stream_e2e_test.go`（`//go:build chromium`）

- [ ] **Step 1: 写端到端测试**

Create `internal/server/browser_stream_e2e_test.go`:

```go
//go:build chromium

package server

import (
	"bufio"
	"context"
	"net/http"
	"net/http/httptest"
	"strings"
	"testing"
	"time"

	"github.com/stardust/legion-agent/internal/browser"
)

func TestBrowserStreamE2EObservationProgressFrame(t *testing.T) {
	target := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, _ *http.Request) {
		w.Header().Set("Content-Type", "text/html")
		_, _ = w.Write([]byte(`<html><body><button aria-label="按钮">click</button></body></html>`))
	}))
	defer target.Close()

	rt, err := browser.NewRuntime(browser.RuntimeConfig{
		Headless: true, AllowPrivateHosts: true,
		BinPath: `C:\Program Files\Google\Chrome\Application\chrome.exe`,
	})
	if err != nil {
		t.Fatalf("NewRuntime: %v", err)
	}
	defer rt.Close(context.Background(), browser.CloseReq{})

	open, err := rt.Open(context.Background(), browser.OpenReq{URL: target.URL, TaskID: "e2e"})
	if err != nil {
		t.Fatalf("Open: %v", err)
	}

	srv := &HTTPServer{browser: rt} // rt 满足 BrowserStreamer
	ts := httptest.NewServer(http.HandlerFunc(srv.handleBrowserStream))
	defer ts.Close()

	// 订阅 SSE
	req, _ := http.NewRequest(http.MethodGet, ts.URL+"/v1/browser/sessions/"+open.SessionID+"/stream", nil)
	resp, err := http.DefaultClient.Do(req)
	if err != nil {
		t.Fatalf("subscribe: %v", err)
	}
	defer resp.Body.Close()

	// 订阅建立后触发一次动作，产生 observation+progress（+frame）
	go func() {
		time.Sleep(200 * time.Millisecond)
		var ref string
		read, _ := rt.Read(context.Background(), browser.ReadReq{SessionID: open.SessionID})
		for _, e := range read.Elements {
			if strings.Contains(e.Name, "按钮") {
				ref = e.Ref
			}
		}
		_, _ = rt.Click(context.Background(), browser.ClickReq{SessionID: open.SessionID, Ref: ref})
	}()

	sc := bufio.NewScanner(resp.Body)
	seenProgress, seenObs, seenFrame := false, false, false
	deadline := time.Now().Add(4 * time.Second)
	for sc.Scan() && time.Now().Before(deadline) {
		line := sc.Text()
		switch {
		case strings.HasPrefix(line, "event: progress"):
			seenProgress = true
		case strings.HasPrefix(line, "event: observation"):
			seenObs = true
		case strings.HasPrefix(line, "event: frame"):
			seenFrame = true
		}
		if seenProgress && seenObs && seenFrame {
			break
		}
	}
	if !seenProgress || !seenObs || !seenFrame {
		t.Fatalf("missing events: progress=%v obs=%v frame=%v", seenProgress, seenObs, seenFrame)
	}
}
```

- [ ] **Step 2: 跑端到端（系统 Chrome）**

Run: `go test -tags chromium ./internal/server/ -run TestBrowserStreamE2E -v`
Expected: PASS（SSE 上同时看到 progress + observation + frame 三类事件）。

- [ ] **Step 3: 全量回归（非 chromium）**

Run: `go build ./... && go test ./... 2>&1 | tail -15`
Expected: 全绿（chromium-tag 测试跳过）；`go test ./internal/runtime/ -run TestEveryProductionToolIsGateable` 仍绿。

- [ ] **Step 4: Commit**

```bash
git add internal/server/browser_stream_e2e_test.go
git commit -m "test(server): e2e browser stream delivers observation/progress/frame over SSE"
```

---

## 验证 Phase 2 DoD（对照 spec §8 Phase 2）

- [ ] 前端可经一条 SSE 长连接实时收到 `observation`/`frame`/`progress` 三类事件（Task 5 e2e）
- [ ] `Last-Event-ID` 重连：补发缓冲的 status 事件、跳过已送帧（Task 3 replay 单测 + Hub ring 单测）
- [ ] 帧可丢、状态事件不丢（Hub 非阻塞扇出丢帧 / status 进环形缓冲，Task 1）
- [ ] 不看视图时停 screencast（订阅者 1→0 触发 Stop，Task 4）
- [ ] 限帧率（screencaster minInterval 节流，Task 4）
- [ ] 全量非 chromium 测试绿、drift 绿、默认关不受影响

---

## 已知近似与后续债务（诚实记录）

| 近似 | 本 Phase 做法 | 后续 |
|---|---|---|
| 帧传输 | base64 塞 SSE（+33%） | 若需高保真高帧率：单开二进制 WS 旁路（spec §3.2），观测/进度仍走 SSE |
| Hub 生命周期 | Close 时 drop；无独立 TTL | Phase 3：随会话持久化/TTL 统一回收 |
| 鉴权 | 沿用现有 server 中间件；未对 stream 单独加 token 过期/`event: reauth` | Phase 5：SSE 长连 token 吊销（spec §10.0） |
| 换页 screencast | Open 切页后 restart | 多 tab 同会话时的帧路由留待 tab 管理 |
| 慢订阅者 | 非阻塞丢事件（含 status 给该订阅者，靠 ReplaySince 补） | 若需严格背压：每订阅者独立缓冲策略 |
```
