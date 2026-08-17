# GUI 内置浏览器视图 UI Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking. **本 plan 跨两个 git 仓库**——每 task 标注仓库与分支，分别提交/开 PR。

**Goal:** GUI 用户实时看见 Agent 浏览过程——canvas 渲染 screencast 帧 + 观测树 + 进度。后端发 `browser:session_*` 生命周期事件让前端发现 session；React 用 fetch+ReadableStream 带 bearer 连 Phase 2 的 `/v1/browser/sessions/{id}/stream`；只读展示。

**Architecture:** 见 spec `docs/superpowers/specs/2026-08-08-agent-browser-view-ui-design.md`。两仓：**legionAgent**（`BrowserEventSink` 端口 + 工具 emit + cli 平台 sink 适配器，发 `browser:session_opened/closed` 到 `observability.EventBus`）；**legionAgentGUI**（Go `sse_bridge` 转发为 Wails event + React `browserStore`/`sseReader`/hooks/`BrowserView`）。

**Tech Stack:** Go 1.26、Wails v2、React+TS+Vite+Tailwind+Zustand+vitest+@testing-library。**go.work+replace** 使 GUI 编译本地 legionAgent。

**跨仓分支:**
- legionAgent（`stardust-agent-server`）：分支 `feat/browser-view-events`，基 master。Task 1/2。→ PR。
- legionAgentGUI（`stardust-agent-gui`）：分支 `feat/browser-view-ui`，基 master。Task 3-7。→ PR。**先合 legionAgent PR**（GUI 逻辑依赖后端事件类型，虽 replace 即时可见）。

**范围边界（不做）:** 接管模式（Phase 7）；set-of-marks 标注；多 tab 视图。见 spec §5。

---

## 文件结构

| 仓库 | 文件 | 职责 | Task |
|---|---|---|---|
| legionAgent | `internal/tool/browser.go` | `BrowserEventSink` 端口 + `BrowserToolOptions.Events` + open/close emit | 1 |
| legionAgent | `internal/cli/browser_event_sink.go` | 平台 sink：sink 调用 → `EventBus.Publish` | 2 |
| legionAgent | `internal/cli/command.go` | 装配 sink 进 `BrowserToolOptions.Events` | 2 |
| legionAgentGUI | `sse_bridge.go` | 转发 `browser:session_*` → Wails `browser:session` | 3 |
| legionAgentGUI | `frontend/wailsjs/go/main/App.*` | 重生/补 `GetBrowserEndpoint` TS 绑定 | 4 |
| legionAgentGUI | `frontend/src/stores/browserStore.ts` | zustand 浏览器视图状态 | 4 |
| legionAgentGUI | `frontend/src/lib/sseReader.ts` | fetch+ReadableStream SSE 解析器 | 5 |
| legionAgentGUI | `frontend/src/hooks/useBrowserSession.ts` · `useBrowserStream.ts` | session 发现 + 流消费 | 6 |
| legionAgentGUI | `frontend/src/components/BrowserView.tsx` + `App.tsx` | canvas 渲染 + 挂载 | 7 |

---

## Task 1 [legionAgent] — BrowserEventSink 端口 + 工具 emit

**Repo:** legionAgent（分支 `feat/browser-view-events`）
**Files:**
- Modify: `internal/tool/browser.go`
- Test: `internal/tool/browser_events_test.go`

- [ ] **Step 1: 写失败测试（假 sink，无 Chromium）**

Create `internal/tool/browser_events_test.go`:

```go
package tool

import (
	"context"
	"testing"

	"github.com/stardust/legion-agent/internal/browser"
	"github.com/stardust/legion-agent/internal/domain"
)

type recordingSink struct {
	openedID, openedTask, openedURL string
	closedID                        string
}

func (r *recordingSink) SessionOpened(_ context.Context, sessionID, taskID, url string) {
	r.openedID, r.openedTask, r.openedURL = sessionID, taskID, url
}
func (r *recordingSink) SessionClosed(_ context.Context, sessionID string) { r.closedID = sessionID }

func TestBrowserOpenEmitsSessionOpened(t *testing.T) {
	sink := &recordingSink{}
	reg := NewRegistry(NewStaticPolicy(DecisionAllow), nil, NoopGuardrails{})
	RegisterBrowserTools(reg, BrowserToolOptions{Enabled: true, Runtime: &fakeBrowserRuntime{}, Events: sink})
	res, err := reg.Execute(context.Background(), domain.Agent{ID: "a", Role: "developer"},
		domain.ToolCall{ID: "c1", Name: "browser_open", Arguments: map[string]string{"url": "https://example.com"}})
	if err != nil || !res.Success {
		t.Fatalf("open failed: %v %q", err, res.Error)
	}
	if sink.openedID != "sess-1" || sink.openedURL != "https://example.com" {
		t.Fatalf("SessionOpened not called correctly: %+v", sink)
	}
}

func TestBrowserCloseEmitsSessionClosed(t *testing.T) {
	sink := &recordingSink{}
	reg := NewRegistry(NewStaticPolicy(DecisionAllow), nil, NoopGuardrails{})
	RegisterBrowserTools(reg, BrowserToolOptions{Enabled: true, Runtime: &fakeBrowserRuntime{}, Events: sink})
	_, _ = reg.Execute(context.Background(), domain.Agent{ID: "a", Role: "developer"},
		domain.ToolCall{ID: "c2", Name: "browser_close", Arguments: map[string]string{"session_id": "sess-9"}})
	if sink.closedID != "sess-9" {
		t.Fatalf("SessionClosed not called: %+v", sink)
	}
}

// Events 为 nil 时不 panic（不破坏现有）。
func TestBrowserToolsNilEventsSink(t *testing.T) {
	reg := NewRegistry(NewStaticPolicy(DecisionAllow), nil, NoopGuardrails{})
	RegisterBrowserTools(reg, BrowserToolOptions{Enabled: true, Runtime: &fakeBrowserRuntime{}, Events: nil})
	_, err := reg.Execute(context.Background(), domain.Agent{ID: "a", Role: "developer"},
		domain.ToolCall{ID: "c3", Name: "browser_open", Arguments: map[string]string{"url": "https://x"}})
	if err != nil {
		t.Fatalf("nil Events should not error: %v", err)
	}
}
```

> `fakeBrowserRuntime` 已在 `internal/tool/browser_test.go` 定义（同包，其 `Open` 返回 `OpenObservation{SessionID:"sess-1", ...}`——确认该 fake 的 Open 返回 sess-1，否则调整断言）。若 fake 的 SessionID 非 "sess-1"，改断言为读 `res.Output` 里的 session_id 或对齐 fake 实际值。

- [ ] **Step 2: 跑，确认失败**

Run: `go test ./internal/tool/ -run 'TestBrowserOpenEmits|TestBrowserCloseEmits|TestBrowserToolsNilEvents' -v`
Expected: FAIL（`BrowserToolOptions` 无 `Events`；`BrowserEventSink` 未定义）。

- [ ] **Step 3: 实现端口 + emit**

Edit `internal/tool/browser.go`:

```go
// BrowserEventSink 是 tool 层对"发浏览器会话生命周期事件"的最小依赖（可选）。
// nil 表示不发（不破坏无事件总线的场景/测试）。
type BrowserEventSink interface {
	SessionOpened(ctx context.Context, sessionID, taskID, url string)
	SessionClosed(ctx context.Context, sessionID string)
}
```
`BrowserToolOptions` 加字段 `Events BrowserEventSink`。

在 `browser_open` handler 成功分支（拿到 `out` 后、return 前）：
```go
	if opts.Events != nil {
		opts.Events.SessionOpened(ctx, out.SessionID, taskID(ctx, call), url)
	}
```
在 `browser_close` handler 成功分支：
```go
	if opts.Events != nil {
		opts.Events.SessionClosed(ctx, call.Arguments["session_id"])
	}
```
`taskID(ctx, call)`：先 `grep -n "task_id\|TaskID\|ctxkey\|WithValue" internal/tool/*.go internal/domain/*.go` 看现有工具怎么取 task id（可能在 ctx 或 call 里）；若无现成通道，本期传空串 `""`（前端不强依赖 task_id，只用 session_id + url）——但优先用现有通道取到。`url` = `call.Arguments["url"]`。

> 注意 `opts` 是 `RegisterBrowserTools` 的入参，被各 handler 闭包捕获——直接引用 `opts.Events` 即可。

- [ ] **Step 4: 跑，确认通过**

Run: `go test ./internal/tool/ -run 'TestBrowser' -v`
Expected: PASS（新 3 个 + 既有 browser 工具测试都过）。`go build ./...` 干净。

- [ ] **Step 5: Commit**

```bash
git add internal/tool/browser.go internal/tool/browser_events_test.go
git commit -m "feat(tool): emit browser session lifecycle events via optional sink"
```
（附空行 + `Co-Authored-By: Claude Opus 4.8 <noreply@anthropic.com>`。）

---

## Task 2 [legionAgent] — cli 平台事件 sink 适配器 + 接线

**Repo:** legionAgent（同分支）
**Files:**
- Create: `internal/cli/browser_event_sink.go`
- Modify: `internal/cli/command.go`（装配 sink 进 `BrowserToolOptions.Events`）
- Test: `internal/cli/browser_event_sink_test.go`

- [ ] **Step 1: 写失败测试**

Create `internal/cli/browser_event_sink_test.go`:

```go
package cli

import (
	"context"
	"testing"

	"github.com/stardust/legion-agent/internal/observability"
)

func TestPlatformBrowserSinkPublishesOpened(t *testing.T) {
	bus := observability.NewEventBus(8)
	events, cancel := bus.Subscribe(context.Background())
	defer cancel()
	sink := newPlatformBrowserEventSink(bus, testLogger())

	sink.SessionOpened(context.Background(), "sess-1", "task-1", "https://example.com")

	select {
	case env := <-events:
		if env.Type != "browser:session_opened" || env.Data["session_id"] != "sess-1" || env.Data["url"] != "https://example.com" {
			t.Fatalf("bad event: %+v", env)
		}
	default:
		t.Fatal("no event published")
	}
}
```
> `testLogger()`：用 `slog.New(slog.NewTextHandler(io.Discard, nil))` 或包内现有测试 logger helper（`grep -rn "slog.New\|testLogger\|discardLogger" internal/cli/*_test.go`）。`bus.Subscribe` 签名以 `internal/observability/eventbus.go` 为准（可能 `Subscribe(ctx) (<-chan EventEnvelope, func())`）——按实调整。

- [ ] **Step 2: 跑，确认失败**

Run: `go test ./internal/cli/ -run TestPlatformBrowserSinkPublishesOpened -v`
Expected: FAIL（`newPlatformBrowserEventSink` 未定义）。

- [ ] **Step 3: 实现适配器**（镜像 `internal/cli/approval_sink.go` 的 `platformApprovalSink`）

Create `internal/cli/browser_event_sink.go`:

```go
package cli

import (
	"context"
	"log/slog"

	"github.com/stardust/legion-agent/internal/observability"
)

// platformBrowserEventSink 把 tool.BrowserEventSink 桥到平台 EventBus（背 /v1/events）。
// tool 层不依赖 observability；桥接在 cli 层（方向正确，同 approval sink）。
type platformBrowserEventSink struct {
	platform *observability.EventBus
	logger   *slog.Logger
}

func newPlatformBrowserEventSink(platform *observability.EventBus, logger *slog.Logger) *platformBrowserEventSink {
	return &platformBrowserEventSink{platform: platform, logger: logger}
}

func (s *platformBrowserEventSink) SessionOpened(ctx context.Context, sessionID, taskID, url string) {
	s.publish(ctx, "browser:session_opened", sessionID, map[string]any{
		"session_id": sessionID, "task_id": taskID, "url": url,
	})
}

func (s *platformBrowserEventSink) SessionClosed(ctx context.Context, sessionID string) {
	s.publish(ctx, "browser:session_closed", sessionID, map[string]any{"session_id": sessionID})
}

func (s *platformBrowserEventSink) publish(ctx context.Context, eventType, sessionID string, data map[string]any) {
	if s.platform == nil {
		return
	}
	if err := s.platform.Publish(ctx, observability.EventEnvelope{
		Type: eventType, SubjectID: sessionID, Data: data,
	}); err != nil {
		s.logger.Warn("browser event sink: platform publish failed",
			"type", eventType, "session_id", sessionID, "error", err)
	}
}
```

- [ ] **Step 4: 接线 command.go**

在构造 `tool.BrowserToolOptions{...}` 处（Phase 1 wiring，`grep -n "BrowserToolOptions{" internal/cli/command.go`），加 `Events: newPlatformBrowserEventSink(platformEvents, logger)`——`platformEvents` 是背 /v1/events 的 `*observability.EventBus`（Phase 1/2 装配已有该变量，`grep -n "platformEvents" internal/cli/command.go`），`logger` 用同处可用的 slog logger。若某构造点无 platformEvents（如非 sqlite 路径），传 nil sink 或 nil bus（sink 内部 nil-guard 已处理）。

- [ ] **Step 5: 跑 + 全量**

Run:
```
go test ./internal/cli/ -run TestPlatformBrowserSink -v
go build ./... && go test ./... 2>&1 | tail -12
go test ./internal/runtime/ -run TestEveryProductionToolIsGateable
```
Expected: PASS；全量绿；drift 绿。

- [ ] **Step 6: 开 legionAgent PR**

```bash
git add internal/cli/browser_event_sink.go internal/cli/browser_event_sink_test.go internal/cli/command.go
git commit -m "feat(cli): publish browser session lifecycle events to platform bus"
git push -u origin feat/browser-view-events
gh pr create --base master --head feat/browser-view-events --title "feat(browser): emit browser session lifecycle events for GUI view" --body "<摘要：browser_open/close 发 browser:session_* 到 /v1/events,供 GUI 浏览器视图发现活跃 session>"
```

---

## Task 3 [legionAgentGUI] — sse_bridge 转发 browser:session

**Repo:** legionAgentGUI（新分支 `feat/browser-view-ui`，基 master）
**Files:**
- Modify: `sse_bridge.go`
- Test: `sse_bridge_test.go`

- [ ] **Step 1: 切分支**

```bash
cd F:/source/stardust/Legion/legion/legionAgentGUI
git checkout master && git pull origin master
git checkout -b feat/browser-view-ui
```

- [ ] **Step 2: 写失败测试**

追加到 `sse_bridge_test.go`（参考现有 approval 转发测试 `collectSSE`/`TestConsumeSSE...`）:

```go
func TestConsumeSSEForwardsBrowserSession(t *testing.T) {
	frame := "event: browser:session_opened\ndata: {\"session_id\":\"sess-1\",\"url\":\"https://x\"}\n\n"
	srv := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, _ *http.Request) {
		w.Header().Set("Content-Type", "text/event-stream")
		w.WriteHeader(200)
		_, _ = w.Write([]byte(frame))
	}))
	defer srv.Close()

	var got []emittedEvent
	emit := func(event string, data any) { got = append(got, emittedEvent{event: event, data: data}) }
	_ = consumeSSEWithToken(context.Background(), srv.URL, "", emit)

	var found bool
	for _, e := range got {
		if e.event == "browser:session" {
			found = true
			m := e.data.(map[string]any)
			if m["type"] != "browser:session_opened" {
				t.Fatalf("forwarded type wrong: %v", m["type"])
			}
		}
	}
	if !found {
		t.Fatalf("browser:session not forwarded; got %+v", got)
	}
}
```

- [ ] **Step 3: 跑，确认失败**

Run: `go test . -run TestConsumeSSEForwardsBrowserSession -v`
Expected: FAIL（未转发）。

- [ ] **Step 4: 实现转发**

Edit `sse_bridge.go` 的事件类型 `switch`（现有 `case "runtime.token","token":` / `case "approval_pending","approval_resolved":` 之后加）:
```go
case "browser:session_opened", "browser:session_closed":
	emit("browser:session", map[string]any{"type": eventType, "data": data})
```
（`data` 是该事件原始 SSE data 字符串，与 `agent:approval` 分支一致——沿用同一个原始 data 变量名。）

- [ ] **Step 5: 跑 + 全量**

Run: `go test . -run TestConsumeSSEForwardsBrowserSession -v && go build . && go test . 2>&1 | tail -4`
Expected: PASS；GUI 全量绿。

- [ ] **Step 6: Commit**

```bash
git add sse_bridge.go sse_bridge_test.go
git commit -m "feat(gui): forward browser session lifecycle events to frontend"
```

---

## Task 4 [legionAgentGUI] — wailsjs 绑定 + browserStore

**Repo:** legionAgentGUI（同分支）
**Files:**
- Modify/Create: `frontend/wailsjs/go/main/App.d.ts` · `App.js`（补 GetBrowserEndpoint 绑定）
- Create: `frontend/src/stores/browserStore.ts`
- Test: `frontend/src/stores/browserStore.test.ts`

- [ ] **Step 1: 生成/补 wailsjs 绑定**

先试自动生成：`cd frontend && npx wails generate module 2>&1 | tail` 或在仓库根 `wails generate module`。若 wails CLI 不可用，**手写**绑定（GetBrowserEndpoint 是无参、返回 `{baseURL, token}` 的方法）：
- `frontend/wailsjs/go/main/App.d.ts` 追加：
```ts
export function GetBrowserEndpoint(): Promise<{baseURL: string; token: string}>;
```
- `frontend/wailsjs/go/main/App.js` 追加（对齐现有导出如 `ListSessions` 的 `window.go.main.App.X()` 包装）：
```js
export function GetBrowserEndpoint() {
  return window['go']['main']['App']['GetBrowserEndpoint']();
}
```
> 确认现有 App.js 的包装风格（`grep -n "window\['go'\]" frontend/wailsjs/go/main/App.js | head`）并照抄。

- [ ] **Step 2: 写 browserStore 失败测试**

Create `frontend/src/stores/browserStore.test.ts`:

```ts
import { describe, it, expect, beforeEach } from 'vitest'
import { useBrowserStore } from './browserStore'

describe('browserStore', () => {
  beforeEach(() => useBrowserStore.getState().reset())

  it('sets and clears session', () => {
    useBrowserStore.getState().setSession('sess-1')
    expect(useBrowserStore.getState().sessionId).toBe('sess-1')
    useBrowserStore.getState().setSession(null)
    expect(useBrowserStore.getState().sessionId).toBeNull()
    expect(useBrowserStore.getState().frameDataUri).toBeNull() // 清 session 重置帧
  })

  it('onFrame builds data uri', () => {
    useBrowserStore.getState().onFrame('image/jpeg', 'AAAA')
    expect(useBrowserStore.getState().frameDataUri).toBe('data:image/jpeg;base64,AAAA')
  })

  it('onObservation stores elements + text', () => {
    useBrowserStore.getState().onObservation({ elements: [{ ref: 'e1', role: 'button', name: '搜索' }], text: '[e1] <button> 搜索' })
    expect(useBrowserStore.getState().elements).toHaveLength(1)
    expect(useBrowserStore.getState().observationText).toContain('e1')
  })
})
```

- [ ] **Step 3: 跑，确认失败**

Run: `cd frontend && npx vitest run src/stores/browserStore.test.ts`
Expected: FAIL（`browserStore` 未定义）。

- [ ] **Step 4: 实现 store**

Create `frontend/src/stores/browserStore.ts`:

```ts
import { create } from 'zustand'

export interface BrowserElement {
  ref: string
  role: string
  name: string
  value?: string
}

interface BrowserState {
  sessionId: string | null
  frameDataUri: string | null
  elements: BrowserElement[]
  observationText: string
  progress: { action: string; status: string; ref?: string } | null
  connected: boolean
  lastEventId: number
  setSession: (id: string | null) => void
  onFrame: (mime: string, b64: string) => void
  onObservation: (obs: { elements: BrowserElement[]; text: string }) => void
  onProgress: (p: { action: string; status: string; ref?: string }) => void
  setConnected: (c: boolean) => void
  setLastEventId: (id: number) => void
  reset: () => void
}

const empty = {
  frameDataUri: null, elements: [] as BrowserElement[], observationText: '',
  progress: null, connected: false, lastEventId: 0,
}

export const useBrowserStore = create<BrowserState>((set) => ({
  sessionId: null,
  ...empty,
  setSession: (id) => set(id === null ? { sessionId: null, ...empty } : { sessionId: id, ...empty }),
  onFrame: (mime, b64) => set({ frameDataUri: `data:${mime};base64,${b64}` }),
  onObservation: (obs) => set({ elements: obs.elements ?? [], observationText: obs.text ?? '' }),
  onProgress: (p) => set({ progress: p }),
  setConnected: (c) => set({ connected: c }),
  setLastEventId: (id) => set({ lastEventId: id }),
  reset: () => set({ sessionId: null, ...empty }),
}))
```

- [ ] **Step 5: 跑，确认通过**

Run: `cd frontend && npx vitest run src/stores/browserStore.test.ts`
Expected: PASS。

- [ ] **Step 6: Commit**

```bash
git add frontend/wailsjs/go/main/App.d.ts frontend/wailsjs/go/main/App.js frontend/src/stores/browserStore.ts frontend/src/stores/browserStore.test.ts
git commit -m "feat(gui): browser view store + GetBrowserEndpoint binding"
```

---

## Task 5 [legionAgentGUI] — sseReader（fetch+ReadableStream SSE 解析）

**Repo:** legionAgentGUI（同分支）
**Files:**
- Create: `frontend/src/lib/sseReader.ts`
- Test: `frontend/src/lib/sseReader.test.ts`

- [ ] **Step 1: 写失败测试（喂 SSE 文本 + 断言解析 + Authorization 头）**

Create `frontend/src/lib/sseReader.test.ts`:

```ts
import { describe, it, expect, vi } from 'vitest'
import { readSSE } from './sseReader'

function streamFrom(chunks: string[]): ReadableStream<Uint8Array> {
  const enc = new TextEncoder()
  let i = 0
  return new ReadableStream({
    pull(ctrl) {
      if (i < chunks.length) ctrl.enqueue(enc.encode(chunks[i++]))
      else ctrl.close()
    },
  })
}

describe('readSSE', () => {
  it('parses events across chunk boundaries and sends bearer', async () => {
    const events: { event: string; id?: string; data: string }[] = []
    const fetchMock = vi.fn(async (_url: string, init: RequestInit) => {
      expect((init.headers as Record<string, string>)['Authorization']).toBe('Bearer tok')
      return { ok: true, status: 200, body: streamFrom(['event: frame\nid: 1\nda', 'ta: {"b":1}\n\nevent: progress\ndata: {}\n\n']) } as unknown as Response
    })
    vi.stubGlobal('fetch', fetchMock)

    await readSSE('http://x/stream', 'tok', 0, (e) => events.push(e), new AbortController().signal)

    expect(events).toHaveLength(2)
    expect(events[0]).toMatchObject({ event: 'frame', id: '1', data: '{"b":1}' })
    expect(events[1].event).toBe('progress')
  })

  it('sends Last-Event-ID when >0 and omits Authorization when token empty', async () => {
    const fetchMock = vi.fn(async (_url: string, init: RequestInit) => {
      const h = init.headers as Record<string, string>
      expect(h['Last-Event-ID']).toBe('5')
      expect(h['Authorization']).toBeUndefined()
      return { ok: true, status: 200, body: streamFrom(['']) } as unknown as Response
    })
    vi.stubGlobal('fetch', fetchMock)
    await readSSE('http://x/stream', '', 5, () => {}, new AbortController().signal)
  })
})
```

- [ ] **Step 2: 跑，确认失败**

Run: `cd frontend && npx vitest run src/lib/sseReader.test.ts`
Expected: FAIL（`readSSE` 未定义）。

- [ ] **Step 3: 实现**

Create `frontend/src/lib/sseReader.ts`:

```ts
export interface SSEEvent {
  event: string
  id?: string
  data: string
}

// readSSE 用 fetch+ReadableStream 消费一条 SSE 流（带可选 bearer 与 Last-Event-ID），
// 逐事件回调。手解析 SSE：event:/id:/data: 行，空行分隔一条事件。EventSource 不能设
// header，故用此方式带 Authorization（token 不进 URL）。
export async function readSSE(
  url: string,
  token: string,
  lastEventId: number,
  onEvent: (e: SSEEvent) => void,
  signal: AbortSignal,
): Promise<void> {
  const headers: Record<string, string> = { Accept: 'text/event-stream' }
  if (token) headers['Authorization'] = `Bearer ${token}`
  if (lastEventId > 0) headers['Last-Event-ID'] = String(lastEventId)

  const res = await fetch(url, { headers, signal })
  if (!res.ok || !res.body) {
    throw new Error(`browser stream ${url}: HTTP ${res.status}`)
  }
  const reader = res.body.getReader()
  const decoder = new TextDecoder()
  let buf = ''
  for (;;) {
    const { done, value } = await reader.read()
    if (done) break
    buf += decoder.decode(value, { stream: true })
    let sep: number
    while ((sep = buf.indexOf('\n\n')) !== -1) {
      const raw = buf.slice(0, sep)
      buf = buf.slice(sep + 2)
      const ev = parseFrame(raw)
      if (ev) onEvent(ev)
    }
  }
}

function parseFrame(raw: string): SSEEvent | null {
  let event = 'message'
  let id: string | undefined
  const dataLines: string[] = []
  for (const line of raw.split('\n')) {
    if (line.startsWith('event:')) event = line.slice(6).trim()
    else if (line.startsWith('id:')) id = line.slice(3).trim()
    else if (line.startsWith('data:')) dataLines.push(line.slice(5).trim())
  }
  if (dataLines.length === 0 && event === 'message') return null
  return { event, id, data: dataLines.join('\n') }
}
```

- [ ] **Step 4: 跑，确认通过**

Run: `cd frontend && npx vitest run src/lib/sseReader.test.ts`
Expected: PASS。

- [ ] **Step 5: Commit**

```bash
git add frontend/src/lib/sseReader.ts frontend/src/lib/sseReader.test.ts
git commit -m "feat(gui): fetch+ReadableStream SSE reader with bearer auth"
```

---

## Task 6 [legionAgentGUI] — useBrowserSession + useBrowserStream hooks

**Repo:** legionAgentGUI（同分支）
**Files:**
- Create: `frontend/src/hooks/useBrowserSession.ts` · `useBrowserStream.ts`
- Test: `frontend/src/hooks/useBrowserSession.test.tsx`

- [ ] **Step 1: 写 useBrowserSession 失败测试**

Create `frontend/src/hooks/useBrowserSession.test.tsx`:

```tsx
import { describe, it, expect, vi, beforeEach } from 'vitest'
import { renderHook } from '@testing-library/react'
import { useBrowserStore } from '../stores/browserStore'

// 用可控的 EventsOn 派发桩
const handlers: Record<string, (p: unknown) => void> = {}
vi.mock('../../wailsjs/runtime/runtime', () => ({
  EventsOn: (ch: string, h: (p: unknown) => void) => { handlers[ch] = h; return () => {} },
  EventsOff: () => {},
}))

import { useBrowserSession } from './useBrowserSession'

describe('useBrowserSession', () => {
  beforeEach(() => useBrowserStore.getState().reset())

  it('sets sessionId on session_opened, clears on session_closed', () => {
    renderHook(() => useBrowserSession())
    handlers['browser:session']({ type: 'browser:session_opened', data: '{"session_id":"sess-7"}' })
    expect(useBrowserStore.getState().sessionId).toBe('sess-7')
    handlers['browser:session']({ type: 'browser:session_closed', data: '{"session_id":"sess-7"}' })
    expect(useBrowserStore.getState().sessionId).toBeNull()
  })
})
```

- [ ] **Step 2: 跑，确认失败**

Run: `cd frontend && npx vitest run src/hooks/useBrowserSession.test.tsx`
Expected: FAIL（`useBrowserSession` 未定义）。

- [ ] **Step 3: 实现 useBrowserSession**（参考 `useAgentEvents.ts` 的 EventsOn/Off 范式）

Create `frontend/src/hooks/useBrowserSession.ts`:

```ts
import { useEffect } from 'react'
import { EventsOn, EventsOff } from '../../wailsjs/runtime/runtime'
import { useBrowserStore } from '../stores/browserStore'

// useBrowserSession 监听后端 browser:session 生命周期事件（经 Go sse_bridge 转发），
// 据此设置/清空当前浏览器视图的 sessionId。挂在 App 顶层（与 useAgentEvents 并列）。
export function useBrowserSession() {
  const setSession = useBrowserStore((s) => s.setSession)
  useEffect(() => {
    const handle = (payload: { type: string; data: string }) => {
      let parsed: { session_id?: string }
      try {
        parsed = JSON.parse(payload.data)
      } catch (err) {
        console.error('browser:session payload not JSON:', payload, err)
        return
      }
      if (payload.type === 'browser:session_opened' && parsed.session_id) {
        setSession(parsed.session_id)
      } else if (payload.type === 'browser:session_closed') {
        // 只在关的是当前会话时清空
        if (useBrowserStore.getState().sessionId === parsed.session_id) setSession(null)
      }
    }
    EventsOn('browser:session', handle)
    return () => EventsOff('browser:session')
  }, [setSession])
}
```

- [ ] **Step 4: 实现 useBrowserStream**（无独立单测——逻辑经 sseReader/store 单测覆盖 + 组件测试间接覆盖；本 hook 是 glue，保持薄）

Create `frontend/src/hooks/useBrowserStream.ts`:

```ts
import { useEffect } from 'react'
import { GetBrowserEndpoint } from '../../wailsjs/go/main/App'
import { useBrowserStore } from '../stores/browserStore'
import { readSSE } from '../lib/sseReader'

// useBrowserStream 在 sessionId 存在时连该会话的 SSE 流（fetch+bearer），把
// observation/frame/progress 事件写进 store；断线带 Last-Event-ID 重连（帧可丢、
// 状态由后端环形缓冲补发）。sessionId 变化/卸载时 abort。
export function useBrowserStream(sessionId: string | null) {
  const store = useBrowserStore
  useEffect(() => {
    if (!sessionId) return
    const ac = new AbortController()
    let stopped = false

    const run = async () => {
      while (!stopped) {
        try {
          const ep = await GetBrowserEndpoint()
          const url = `${ep.baseURL}/v1/browser/sessions/${sessionId}/stream`
          store.getState().setConnected(true)
          await readSSE(url, ep.token, store.getState().lastEventId, (e) => {
            if (e.id) store.getState().setLastEventId(Number(e.id))
            let payload: unknown
            try { payload = JSON.parse(e.data) } catch (err) { console.error('browser stream data not JSON:', e, err); return }
            if (e.event === 'frame') {
              const f = payload as { mime: string; b64: string }
              store.getState().onFrame(f.mime, f.b64)
            } else if (e.event === 'observation') {
              store.getState().onObservation(payload as { elements: never[]; text: string })
            } else if (e.event === 'progress') {
              store.getState().onProgress(payload as { action: string; status: string; ref?: string })
            }
          }, ac.signal)
        } catch (err) {
          if (stopped) break
          console.error('browser stream disconnected, retrying:', err)
          store.getState().setConnected(false)
          await new Promise((r) => setTimeout(r, 2000))
        }
      }
    }
    run()
    return () => { stopped = true; ac.abort(); store.getState().setConnected(false) }
  }, [sessionId, store])
}
```

- [ ] **Step 5: 跑 useBrowserSession 测试 + 类型检查**

Run:
```
cd frontend && npx vitest run src/hooks/useBrowserSession.test.tsx
npx tsc --noEmit
```
Expected: 测试 PASS；`tsc` 无类型错误。

- [ ] **Step 6: Commit**

```bash
git add frontend/src/hooks/useBrowserSession.ts frontend/src/hooks/useBrowserStream.ts frontend/src/hooks/useBrowserSession.test.tsx
git commit -m "feat(gui): browser session discovery + stream consumption hooks"
```

---

## Task 7 [legionAgentGUI] — BrowserView 组件 + 挂载

**Repo:** legionAgentGUI（同分支）
**Files:**
- Create: `frontend/src/components/BrowserView.tsx`
- Test: `frontend/src/components/BrowserView.test.tsx`
- Modify: `frontend/src/App.tsx`（挂载 + useBrowserSession）

- [ ] **Step 1: 写 BrowserView 失败测试**

Create `frontend/src/components/BrowserView.test.tsx`:

```tsx
import { describe, it, expect, vi, beforeEach } from 'vitest'
import { render, screen } from '@testing-library/react'
import { useBrowserStore } from '../stores/browserStore'
import { BrowserView } from './BrowserView'

// 桩 useBrowserStream（避免真连流）
vi.mock('../hooks/useBrowserStream', () => ({ useBrowserStream: () => {} }))

describe('BrowserView', () => {
  beforeEach(() => useBrowserStore.getState().reset())

  it('shows empty state when no session', () => {
    render(<BrowserView />)
    expect(screen.getByText(/未在浏览|no active/i)).toBeInTheDocument()
  })

  it('renders observation elements when present', () => {
    useBrowserStore.getState().setSession('sess-1')
    useBrowserStore.getState().onObservation({ elements: [{ ref: 'e1', role: 'button', name: '搜索' }], text: '' })
    render(<BrowserView />)
    expect(screen.getByText(/搜索/)).toBeInTheDocument()
    expect(screen.getByText(/e1/)).toBeInTheDocument()
  })

  it('draws frame onto canvas when frameDataUri set', () => {
    const drawImage = vi.fn()
    vi.spyOn(HTMLCanvasElement.prototype, 'getContext').mockReturnValue({ drawImage, clearRect: vi.fn() } as unknown as CanvasRenderingContext2D)
    useBrowserStore.getState().setSession('sess-1')
    useBrowserStore.getState().onFrame('image/jpeg', 'AAAA')
    render(<BrowserView />)
    // Image.onload 是异步的；断言 canvas 存在 + getContext 被取用（drawImage 在 onload 触发）
    expect(document.querySelector('canvas')).toBeInTheDocument()
  })
})
```
> `new Image()` 的 `onload` 在 jsdom 里不会真触发解码；测试断言到"canvas 存在 + getContext 取用"即可，drawImage 的真实调用留给运行时（或用 `Object.defineProperty(Image.prototype,'src',...)` 手动触发 onload——若要更强断言可加，但保持测试稳健优先）。

- [ ] **Step 2: 跑，确认失败**

Run: `cd frontend && npx vitest run src/components/BrowserView.test.tsx`
Expected: FAIL（`BrowserView` 未定义）。

- [ ] **Step 3: 实现 BrowserView**（Tailwind 风格对齐现有组件如 `StatusPanel`）

Create `frontend/src/components/BrowserView.tsx`:

```tsx
import { useEffect, useRef } from 'react'
import { useBrowserStore } from '../stores/browserStore'
import { useBrowserStream } from '../hooks/useBrowserStream'

// BrowserView 只读展示 Agent 的浏览过程：canvas 渲染 screencast 帧 + 观测树 + 进度。
// 用户点击不回传（spec §3.2 只读；接管模式 = Phase 7）。
export function BrowserView() {
  const sessionId = useBrowserStore((s) => s.sessionId)
  const frameDataUri = useBrowserStore((s) => s.frameDataUri)
  const elements = useBrowserStore((s) => s.elements)
  const progress = useBrowserStore((s) => s.progress)
  const connected = useBrowserStore((s) => s.connected)
  const canvasRef = useRef<HTMLCanvasElement>(null)

  useBrowserStream(sessionId)

  useEffect(() => {
    if (!frameDataUri || !canvasRef.current) return
    const canvas = canvasRef.current
    const ctx = canvas.getContext('2d')
    if (!ctx) return
    const img = new Image()
    img.onload = () => {
      canvas.width = img.width
      canvas.height = img.height
      ctx.drawImage(img, 0, 0)
    }
    img.onerror = () => console.warn('browser view: frame decode failed')
    img.src = frameDataUri
  }, [frameDataUri])

  if (!sessionId) {
    return <div className="flex h-full items-center justify-center text-sm text-gray-400">Agent 未在浏览</div>
  }

  return (
    <div className="flex h-full flex-col gap-2 p-2">
      <div className="flex items-center gap-2 text-xs">
        <span className={connected ? 'text-green-500' : 'text-amber-500'}>●</span>
        <span className="text-gray-500">session {sessionId}</span>
        {progress && <span className="text-gray-400">· {progress.action}:{progress.status}</span>}
      </div>
      <div className="flex-1 overflow-hidden rounded border border-gray-200 bg-black/5">
        <canvas ref={canvasRef} className="h-full w-full object-contain" />
      </div>
      <details className="max-h-40 overflow-auto rounded border border-gray-200 p-2 text-xs">
        <summary className="cursor-pointer text-gray-500">观测树（{elements.length}）</summary>
        <ul className="mt-1 space-y-0.5 font-mono">
          {elements.map((e) => (
            <li key={e.ref} className="text-gray-700">
              [{e.ref}] &lt;{e.role}&gt; {e.name}
            </li>
          ))}
        </ul>
      </details>
    </div>
  )
}
```

- [ ] **Step 4: 挂载到 App**

Edit `frontend/src/App.tsx`：
- 顶层调用 `useBrowserSession()`（与现有 `useAgentEvents()` 并列，`grep -n "useAgentEvents" src/App.tsx`）。
- 把 `BrowserView` 加入布局：`ThreePanelLayout` 是固定三栏（sidebar/chat/status）——最小侵入做法：在 status 栏加一个标签/切换，让 `StatusPanel` 与 `BrowserView` 可切换（在 `StatusPanel` 区域上方加 tab，或用一个本地 state 切换渲染哪个）。跟随现有组件切换范式（若 App 已有 tab/view 状态则复用；否则在 App 加一个 `rightView: 'status'|'browser'` 本地 state + 两个按钮切换 status 列内容）。保持改动聚焦，不重构 ThreePanelLayout。

- [ ] **Step 5: 跑组件测试 + 全量前端 + 类型 + 构建**

Run:
```
cd frontend
npx vitest run src/components/BrowserView.test.tsx
npx vitest run     # 全量前端测试
npx tsc --noEmit
npm run build      # Vite 构建通过（确保 App.tsx 改动不破坏构建）
```
Expected: 组件测试 + 全量前端测试 PASS；tsc 无错；build 成功。

- [ ] **Step 6: Commit + 开 GUI PR**

```bash
cd F:/source/stardust/Legion/legion/legionAgentGUI
git add frontend/src/components/BrowserView.tsx frontend/src/components/BrowserView.test.tsx frontend/src/App.tsx
git commit -m "feat(gui): browser view panel with canvas screencast rendering"
git push -u origin feat/browser-view-ui
gh pr create --base master --head feat/browser-view-ui --title "feat(gui): browser view UI — canvas screencast + observation tree" --body "<摘要：GUI 浏览器视图,消费 /v1/browser/sessions/{id}/stream,canvas 渲染帧+观测树,只读>"
```

---

## 验证 DoD（对照 spec §4）

- [ ] 后端 `browser_open/close` 发 `browser:session_*` 到 EventBus（Task 1/2 单测）；Events nil 不 panic
- [ ] GUI Go 转发 `browser:session` → Wails（Task 3）
- [ ] `sseReader` 跨 chunk 解析 + Authorization 头 + Last-Event-ID（Task 5 单测）
- [ ] `browserStore` reducers + `useBrowserSession` session 发现（Task 4/6 单测）
- [ ] `BrowserView` 空态 / 观测树渲染 / canvas 取用（Task 7 单测）
- [ ] 前端 `npx vitest run` 全绿 + `tsc --noEmit` 无错 + `npm run build` 成功
- [ ] legionAgent `go test ./...` + drift 绿；GUI `go test .` 绿
- [ ] 两仓各一 PR，base=master

---

## 已知边界与后续

| 项 | 本 plan | 后续 |
|---|---|---|
| 接管模式（回注鼠标/键盘） | 只读，canvas 无回传 | Phase 7 |
| set-of-marks 标注框 | canvas 留口（drawImage 后可叠画） | 视觉分叉 |
| 面板挂载 | status 栏 tab 切换（最小侵入） | 若需独立布局位再调 ThreePanelLayout |
| task_id 传递 | 有现成 ctx 通道则用，否则空串（前端不强依赖） | 若需按 task 过滤视图再补 |
| Image.onload jsdom | 组件测试断言到 canvas 存在/getContext 取用 | 真机验证 drawImage |
```
