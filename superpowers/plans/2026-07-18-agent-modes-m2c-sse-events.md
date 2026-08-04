---
title: 实施计划 — Milestone 2c（接活 /v1/events SSE + 审批/生命周期事件桥接）
type: plan
status: active
created: 2026-07-18
scope: legion/legionAgent（后端）
related:
  - "[[2026-07-18-m2c-handoff-sse-events]]"
  - "[[2026-07-15-agent-working-modes-design]]"
tags: [agent, runtime, sse, events, approval, milestone-2c, plan]
---

# Agent Working Modes — Milestone 2c (SSE Events) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 接活恒 503 的 `GET /v1/events` SSE 端点，让 runtime 现有生命周期事件与 Manual 审批事件（approval_pending/approval_resolved）实时推送给订阅者，并补一个列票据端点供 GUI 对账。

**Architecture:** 做一个实现 `port.EventBus` 的 **tee 型桥接总线**（`internal/eventbridge`）：`Publish(RuntimeEvent)` 时既 append 到内部 slice（保住 `Events()` 给现有 poll 消费者），又翻译成 `observability.EventEnvelope` 推到 `platformEvents`（SSE 背后的推送总线）。在 `serve` 装配里用它替换 `adapter.NewMemoryEventBus()`，把 `platformEvents` 接进 `server.Config.PlatformEvents`（杀 503）。审批事件走 manualgate 的**独立可选 sink**（error-less，best-effort），serve 侧 sink 实现翻译成 EventEnvelope 推同一总线。SSE 脱敏改为递归 + 截断。

**Tech Stack:** Go 1.26；`internal/observability.EventBus`（push/subscribe）；`internal/port.EventBus`（poll）；`internal/domain.RuntimeEvent`；`internal/manualgate` + `internal/approval`（M2b 审批持久化）；`net/http` 手写路由 + `httptest`；WSL Ubuntu-22.04 跑 `-race`。

## Global Constraints

- **Fail-Loud 铁律**（`legionAgent/CLAUDE.md §0`）：禁 fallback/zero-value 兜底、禁 `_ = err` 静默、禁默默 continue/return。错误 `fmt.Errorf("<动作> <标识>: %w", err)` 包装；边界用项目 logger 结构化记录（Error/Warn）。
- **M2c 特有 fail-loud 边界**：SSE 是**通知层非权威**——盘上票据/checkpoint 是唯一真相源。桥接 tee 向 `platformEvents` publish 失败、审批 sink publish 失败，**记 Warn 不阻断** runtime/审批（这是「契约显式声明的可选能力」豁免，§0 唯一豁免），但**必须记录**，不得 `_ = err` 静默。`observability.EventBus` 满 buffer 非阻塞丢弃是既有契约行为，靠列票据端点对账，不算兜底。
- **事件类型命名约定**：SSE `EventEnvelope.Type` **保持下划线**（`task_completed`/`tool_executed`/`approval_pending`…），与 `domain.RuntimeEvent.Type` 一致（`internal/runtime` 全部发下划线），减少映射面；GUI（M3）消费同一命名空间。（既有 `events_test.go` 里的 `task.completed` 只是该测试直接 publish 的字面量，非约定，不动。）
- **公开 API 必须有 Go doc 风格注释**（以标识符名开头）。
- **完成门禁**：Windows 宿主 `go build ./... && go vet ./... && go test ./...` 全绿；`gofmt -l` 对触碰文件内容 LF 干净（仓库 `core.autocrlf=true`，只 `gofmt -w` 自己改的文件，别整仓重排）；WSL `-race` 绿。
- **仓库**：全部代码在 `legion/legionAgent`（remote `github.com/jxncyjq/jxncyjq-stardust-agent-server`）。从 master `dab6162` 切 `feat/agent-modes-m2c`。改代码前 `git rev-parse --show-toplevel` 确认。GUI/docs 是独立仓库，M2c 不碰 GUI 前端（那是 M3）。

---

## File Structure

- **Create** `internal/eventbridge/eventbridge.go` — `Bridge` 类型（实现 `port.EventBus`）+ `translate` 映射。桥接总线，Task 1。
- **Create** `internal/eventbridge/eventbridge_test.go` — 桥接单测，Task 1。
- **Modify** `internal/cli/command.go` — serve 装配：hoist logger、构造 `platformEvents` + `Bridge` 替换 `workflowEvents`、Config 接 `PlatformEvents`/`ApprovalTickets`、sink 接线、Close 关 platformEvents。Task 2/3/5。
- **Create** `internal/cli/approval_sink.go` — `platformApprovalSink`（实现 `manualgate.ApprovalEventSink`，翻译审批事件→EventEnvelope→platformEvents）。Task 3。
- **Create** `internal/cli/approval_sink_test.go` — sink 单测，Task 3。
- **Modify** `internal/manualgate/manualgate.go` — 加 `ApprovalEventSink` 接口 + `Option`/`WithApprovalSink` + `ShouldSuspend` 发 approval_pending。Task 3。
- **Modify** `internal/manualgate/decider.go` — 加 `CoordinatorOption`/`WithCoordinatorSink` + `Decide` 发 approval_resolved。Task 3。
- **Modify** `internal/manualgate/manualgate_test.go` / `decider_test.go` — sink 发射断言 + spy sink。Task 3。
- **Modify** `internal/server/events.go` — `sanitizeEventData` 改递归 + 截断，加 `sanitizeStringMap`/`truncateEventString`。Task 4。
- **Modify** `internal/server/events_test.go` — 递归/截断脱敏断言。Task 4。
- **Create** `internal/server/approvals_list.go` — `ApprovalLister` 接口 + `handleListApprovals`。Task 5。
- **Create** `internal/server/approvals_list_test.go` — 列票据端点单测。Task 5。
- **Modify** `internal/server/http.go` — `Config.ApprovalTickets`、`HTTPServer.approvalTickets`、New 赋值、`GET /v1/approvals` 路由。Task 5。
- **Modify** `internal/cli/command_test.go` — serve 全链路 e2e：/v1/events 不再 503 + 收到 task_completed。Task 6。

---

### Task 1: 桥接 EventBus（internal/eventbridge）

做一个实现 `port.EventBus` 的 tee 桥接总线。这是把 runtime 现有 RuntimeEvent 发射点零改动接到 SSE 的核心。

**Files:**
- Create: `internal/eventbridge/eventbridge.go`
- Test: `internal/eventbridge/eventbridge_test.go`

**Interfaces:**
- Consumes: `domain.RuntimeEvent`（`internal/domain/types.go:124`：`Type,TaskID,Message,PromptTokens,CompletionTokens,CachedTokens,TotalTokens,ElapsedMs,CreatedAt`）；`observability.EventBus`（`Publish(ctx,EventEnvelope)error`）+ `observability.EventEnvelope{ID,Type,SubjectID,Data map[string]any,CreatedAt}`。
- Produces: `eventbridge.New(platform *observability.EventBus, logger *slog.Logger) *Bridge`；`*Bridge` 满足 `port.EventBus{Publish(ctx,domain.RuntimeEvent)error; Events()[]domain.RuntimeEvent}`。

- [ ] **Step 1: 写失败测试**

`internal/eventbridge/eventbridge_test.go`：

```go
package eventbridge

import (
	"context"
	"testing"
	"time"

	"github.com/stardust/legion-agent/internal/domain"
	"github.com/stardust/legion-agent/internal/observability"
)

func TestBridgeTeesRuntimeEventToPlatform(t *testing.T) {
	platform := observability.NewEventBus(8)
	b := New(platform, nil)
	events, cancel := platform.Subscribe(context.Background())
	defer cancel()

	err := b.Publish(context.Background(), domain.RuntimeEvent{
		Type:        "task_completed",
		TaskID:      "task-1",
		Message:     "done",
		TotalTokens: 5,
		CreatedAt:   time.Unix(1000, 0),
	})
	if err != nil {
		t.Fatalf("Bridge.Publish() error = %v, want nil", err)
	}

	select {
	case env := <-events:
		if env.Type != "task_completed" {
			t.Fatalf("envelope Type = %q, want task_completed", env.Type)
		}
		if env.SubjectID != "task-1" {
			t.Fatalf("envelope SubjectID = %q, want task-1", env.SubjectID)
		}
		if env.Data["task_id"] != "task-1" {
			t.Fatalf("envelope Data[task_id] = %v, want task-1", env.Data["task_id"])
		}
		if env.Data["message"] != "done" {
			t.Fatalf("envelope Data[message] = %v, want done", env.Data["message"])
		}
		if env.Data["total_tokens"] != 5 {
			t.Fatalf("envelope Data[total_tokens] = %v, want 5", env.Data["total_tokens"])
		}
		if !env.CreatedAt.Equal(time.Unix(1000, 0)) {
			t.Fatalf("envelope CreatedAt = %v, want 1000", env.CreatedAt)
		}
	case <-time.After(time.Second):
		t.Fatal("platform subscriber received no event, want teed envelope")
	}
}

func TestBridgePreservesEventsSnapshotForPollConsumers(t *testing.T) {
	platform := observability.NewEventBus(8)
	b := New(platform, nil)
	first := domain.RuntimeEvent{Type: "task_started", TaskID: "t1"}
	second := domain.RuntimeEvent{Type: "task_completed", TaskID: "t1"}
	if err := b.Publish(context.Background(), first); err != nil {
		t.Fatalf("Publish(first) error = %v, want nil", err)
	}
	if err := b.Publish(context.Background(), second); err != nil {
		t.Fatalf("Publish(second) error = %v, want nil", err)
	}
	got := b.Events()
	if len(got) != 2 || got[0].Type != "task_started" || got[1].Type != "task_completed" {
		t.Fatalf("Events() = %#v, want [task_started, task_completed] snapshot", got)
	}
}

func TestBridgePublishSurvivesClosedPlatform(t *testing.T) {
	platform := observability.NewEventBus(8)
	b := New(platform, nil)
	if err := platform.Close(); err != nil {
		t.Fatalf("platform.Close() error = %v, want nil", err)
	}
	// Platform bus is closed: tee fails internally (logged Warn), but the
	// authoritative append half must still succeed and Publish must return nil —
	// SSE is a best-effort notification layer, never a task-flow blocker.
	if err := b.Publish(context.Background(), domain.RuntimeEvent{Type: "task_completed", TaskID: "t1"}); err != nil {
		t.Fatalf("Publish() after platform close error = %v, want nil", err)
	}
	if got := b.Events(); len(got) != 1 {
		t.Fatalf("Events() len = %d, want 1 (append half unaffected by tee failure)", len(got))
	}
}
```

- [ ] **Step 2: 跑测试确认失败**

Run: `go test ./internal/eventbridge/`
Expected: FAIL（`undefined: New` / 包不存在）。

- [ ] **Step 3: 写实现**

`internal/eventbridge/eventbridge.go`：

```go
// Package eventbridge tees the runtime's poll-only RuntimeEvent stream into the
// push/subscribe platform event bus that backs the /v1/events SSE endpoint,
// without changing any existing RuntimeEvent publisher.
package eventbridge

import (
	"context"
	"io"
	"log/slog"
	"sync"

	"github.com/stardust/legion-agent/internal/domain"
	"github.com/stardust/legion-agent/internal/observability"
)

// Bridge is a port.EventBus that tees every RuntimeEvent two ways: it appends to
// an internal slice (preserving the poll-only Events() contract the GEP / trust /
// degradation background jobs still rely on) and translates it into an
// observability.EventEnvelope pushed to the platform bus behind the /v1/events
// SSE stream. It is the single wiring point that connects the runtime's existing
// RuntimeEvent publishers to SSE.
type Bridge struct {
	mu       sync.Mutex
	events   []domain.RuntimeEvent
	platform *observability.EventBus
	logger   *slog.Logger
}

// New returns a Bridge teeing to platform. platform must not be nil (SSE wiring
// is the whole point of this type). logger records tee failures; a nil logger
// discards. A platform publish error is logged Warn and never propagated: the
// appended slice is the authoritative half of the old MemoryEventBus contract,
// and SSE is a best-effort notification layer (design §3.4.2).
func New(platform *observability.EventBus, logger *slog.Logger) *Bridge {
	if platform == nil {
		panic("eventbridge.New: platform event bus must not be nil")
	}
	if logger == nil {
		logger = slog.New(slog.NewTextHandler(io.Discard, nil))
	}
	return &Bridge{platform: platform, logger: logger}
}

// Publish appends the event to the internal snapshot (authoritative, for poll
// consumers) and tees a translated envelope to the platform bus (best-effort,
// for SSE). It returns an error only when the context is already done; a
// platform publish failure is logged Warn, not propagated.
func (b *Bridge) Publish(ctx context.Context, event domain.RuntimeEvent) error {
	if err := ctx.Err(); err != nil {
		return err
	}
	b.mu.Lock()
	b.events = append(b.events, event)
	b.mu.Unlock()

	if err := b.platform.Publish(ctx, translate(event)); err != nil {
		b.logger.Warn("event bridge: platform publish failed",
			"type", event.Type, "task_id", event.TaskID, "error", err)
	}
	return nil
}

// Events returns a snapshot copy of every published RuntimeEvent, satisfying the
// poll-only half of port.EventBus that existing background jobs consume.
func (b *Bridge) Events() []domain.RuntimeEvent {
	b.mu.Lock()
	defer b.mu.Unlock()
	return append([]domain.RuntimeEvent(nil), b.events...)
}

// translate maps a RuntimeEvent to a platform EventEnvelope. Type is preserved
// verbatim (underscore form, matching every internal/runtime publisher) so the
// SSE type namespace equals the RuntimeEvent namespace. Token / elapsed fields
// are included only when non-zero to keep the payload lean. No "prompt"/"input"
// keys are emitted, so sanitizeEventData never has to strip lifecycle payloads.
func translate(ev domain.RuntimeEvent) observability.EventEnvelope {
	data := map[string]any{"task_id": ev.TaskID}
	if ev.Message != "" {
		data["message"] = ev.Message
	}
	if ev.PromptTokens != 0 {
		data["prompt_tokens"] = ev.PromptTokens
	}
	if ev.CompletionTokens != 0 {
		data["completion_tokens"] = ev.CompletionTokens
	}
	if ev.CachedTokens != 0 {
		data["cached_tokens"] = ev.CachedTokens
	}
	if ev.TotalTokens != 0 {
		data["total_tokens"] = ev.TotalTokens
	}
	if ev.ElapsedMs != 0 {
		data["elapsed_ms"] = ev.ElapsedMs
	}
	return observability.EventEnvelope{
		Type:      ev.Type,
		SubjectID: ev.TaskID,
		Data:      data,
		CreatedAt: ev.CreatedAt,
	}
}
```

- [ ] **Step 4: 跑测试确认通过**

Run: `go test ./internal/eventbridge/`
Expected: PASS（3 个测试）。

- [ ] **Step 5: gofmt + 提交**

```bash
gofmt -w internal/eventbridge/eventbridge.go internal/eventbridge/eventbridge_test.go
git add internal/eventbridge/
git commit -m "feat(eventbridge): tee RuntimeEvent 到 platform SSE 总线"
```

---

### Task 2: serve 接 platformEvents + 桥接（杀 503）

在 `serve` 装配里构造 `platformEvents`，用 `Bridge` 替换 `workflowEvents`，把 `platformEvents` 接进 `server.Config.PlatformEvents`，shutdown 关闭它。这一步单独交付「`/v1/events` 不再恒 503」。

**Files:**
- Modify: `internal/cli/command.go`（logger hoist、`workflowEvents` 构造 @~1709、Config @~1957、Close @~2011）

**Interfaces:**
- Consumes: `eventbridge.New`（Task 1）；`observability.NewEventBus(buffer int) *observability.EventBus`（`observability/eventbus.go:29`）。
- Produces: serve 运行后 `GET /v1/events` 返回 SSE 流（200 + `text/event-stream`），不再 503。

- [ ] **Step 1: hoist logger 到 workflowEvents 构造之前**

`command.go` 当前 logger 定义在 `@1892`（`net.Listen` 之后），而 `workflowEvents` 在 `@1709`。桥接需要 logger，故把这 3 行上移到 `@1708`（`if auditLog == nil {...}` 之后、`workflowEvents :=` 之前）：

```go
	logger := opts.Logger
	if logger == nil {
		logger = defaultLogger()
	}
```

删除原 `@1892-1895` 的这 3 行（保留其后的 `if workspaceRootWarning != "" { logger.Warn(...) }` 原位不动——`workspaceRootWarning` 在 `@1737` 定义，仍在 logger 之后）。

- [ ] **Step 2: 构造 platformEvents + 桥接替换 workflowEvents**

把 `command.go:1709` 的：

```go
	workflowEvents := adapter.NewMemoryEventBus()
```

替换为：

```go
	// platformEvents backs the /v1/events SSE stream (push/subscribe). The
	// bridge tees every RuntimeEvent the runtime/coordinator/workflow engine
	// already publishes into it, so lifecycle events reach SSE with zero changes
	// to any publisher. Buffer sized generously: a slow SSE subscriber drops
	// events (at-most-once, design §3.4.2), and the /v1/approvals list endpoint
	// (Task 5) is the reconcile path for missed approval prompts.
	platformEvents := observability.NewEventBus(serveEventBusBuffer)
	workflowEvents := eventbridge.New(platformEvents, logger)
```

在 `serve` 函数上方（或文件顶部 const 区）加常量：

```go
// serveEventBusBuffer is the per-subscriber channel buffer for the platform
// event bus backing /v1/events. Large enough that a normally-paced SSE client
// keeps up; a stalled client still drops rather than blocking publishers.
const serveEventBusBuffer = 256
```

确认 `observability` 已在 command.go import（`@1975` 已用 `observability.NewDiagnostics`，已导入）；确认 `eventbridge` 需新增 import `"github.com/stardust/legion-agent/internal/eventbridge"`。`adapter` 若不再被别处使用需检查——`adapter` 仍被 `NewMemoryAuditLog`/`NewMaasClientFromProfile`/`NewRecordingMaas` 使用，import 保留。

- [ ] **Step 3: Config 接 PlatformEvents**

`command.go:1957` 的 `server.NewHTTPServer(server.Config{...})` 块内，在 `WorkflowEvents: workflowEvents,` 下一行加：

```go
		PlatformEvents:      platformEvents,
```

（`server.Config.PlatformEvents *observability.EventBus` 字段 `http.go:91` 已存在，只是 serve 从没赋值——这是 503 死代码根因。）

- [ ] **Step 4: shutdown 关闭 platformEvents**

`command.go:2011` 的 `Close: func() {` 块改为：

```go
		Close: func() {
			coordinator.Wait()
			if err := platformEvents.Close(); err != nil {
				logger.Warn("close platform event bus", "error", err)
			}
			closeStore()
		},
```

（`coordinator.Wait()` 已保证无 task goroutine 再发事件后才关总线；`EventBus.Close` 幂等，err 记 Warn 不阻断。）

- [ ] **Step 5: 写 e2e 断言 /v1/events 不再 503**

`internal/cli/command_test.go` 追加（复用 `TestServeCommandUsesSQLiteForHTTPTaskState` 的 serve 起停骨架）：

```go
func TestServeCommandEventsEndpointNotUnavailable(t *testing.T) {
	t.Parallel()
	ctx, cancel := context.WithCancel(context.Background())
	defer cancel()
	configPath := filepath.Join(t.TempDir(), "agent.json")
	if err := os.WriteFile(configPath, []byte(`{"service": {"background_interval": "1h"}}`), 0o600); err != nil {
		t.Fatalf("WriteFile(%q) error = %v, want nil", configPath, err)
	}
	listener, err := net.Listen("tcp", "127.0.0.1:0")
	if err != nil {
		t.Fatalf("net.Listen() error = %v, want nil", err)
	}
	addr := listener.Addr().String()
	if err := listener.Close(); err != nil {
		t.Fatalf("listener.Close() error = %v, want nil", err)
	}

	var out bytes.Buffer
	root := NewRoot(app.New(), &out)
	root.SetContext(ctx)
	root.SetArgs([]string{"serve", "--config", configPath, "--addr", addr})
	done := make(chan error, 1)
	go func() { done <- root.Execute() }()

	// Poll until the server is listening, then open the SSE stream.
	streamCtx, streamCancel := context.WithTimeout(ctx, 3*time.Second)
	defer streamCancel()
	var status int
	var contentType string
	for i := 0; i < 100; i++ {
		req, reqErr := http.NewRequestWithContext(streamCtx, http.MethodGet, "http://"+addr+"/v1/events", nil)
		if reqErr != nil {
			t.Fatalf("NewRequest(/v1/events) error = %v, want nil", reqErr)
		}
		resp, doErr := http.DefaultClient.Do(req)
		if doErr != nil {
			time.Sleep(20 * time.Millisecond)
			continue
		}
		status = resp.StatusCode
		contentType = resp.Header.Get("Content-Type")
		_ = resp.Body.Close()
		break
	}
	cancel()
	select {
	case execErr := <-done:
		if execErr != nil {
			t.Fatalf("Execute(serve) error = %v, want nil", execErr)
		}
	case <-time.After(2 * time.Second):
		t.Fatal("Execute(serve) did not stop")
	}
	if status == http.StatusServiceUnavailable {
		t.Fatal("GET /v1/events status = 503, want SSE wired (503 death-code regression)")
	}
	if status != http.StatusOK {
		t.Fatalf("GET /v1/events status = %d, want 200", status)
	}
	if !strings.Contains(contentType, "text/event-stream") {
		t.Fatalf("GET /v1/events Content-Type = %q, want text/event-stream", contentType)
	}
}
```

（`bytes`/`context`/`net`/`net/http`/`os`/`path/filepath`/`strings`/`time` + `app` import 该测试文件已全有——见文件头。）

- [ ] **Step 6: 跑构建 + 测试确认通过**

Run: `go build ./... && go test ./internal/cli/ -run TestServeCommandEventsEndpointNotUnavailable -v`
Expected: PASS（status 200、event-stream）。

- [ ] **Step 7: gofmt + 提交**

```bash
gofmt -w internal/cli/command.go internal/cli/command_test.go
git add internal/cli/command.go internal/cli/command_test.go
git commit -m "feat(serve): 接活 /v1/events SSE，桥接 workflowEvents→platformEvents"
```

---

### Task 3: 审批事件 sink + 发射点（approval_pending / approval_resolved）

审批事件字段（ticket_id/tool/arguments）塞不进固定字段的 RuntimeEvent，故走独立 sink。sink 接口 **error-less**（best-effort 通知层，impl 内部 Warn 后自吞，故调用方无 error 可忽略、不违 fail-loud）。

**Files:**
- Modify: `internal/manualgate/manualgate.go`、`internal/manualgate/decider.go`
- Modify: `internal/manualgate/manualgate_test.go`、`internal/manualgate/decider_test.go`
- Create: `internal/cli/approval_sink.go`、`internal/cli/approval_sink_test.go`
- Modify: `internal/cli/command.go`（sink 接线 @~1740/1745）

**Interfaces:**
- Consumes: `approval.ToolGateStore`；`approval.ApprovalStatus`（`ApprovalApproved`/`ApprovalDenied` = `"approved"`/`"denied"`）；`observability.EventBus.Publish`。M2b：`ShouldSuspend`（`manualgate.go:63` 开票据点）、`Decide`（`decider.go:40` 决定落盘点）、`NewTimeoutSweepJob`（`timeout.go:18`，经 `dec.Decide` 间接发 resolved）。
- Produces: `manualgate.ApprovalEventSink` 接口；`manualgate.WithApprovalSink(sink) Option`；`manualgate.WithCoordinatorSink(sink) CoordinatorOption`；`cli.newPlatformApprovalSink(platform *observability.EventBus, logger *slog.Logger) *platformApprovalSink`。

- [ ] **Step 1: 写 manualgate sink 发射失败测试**

`internal/manualgate/manualgate_test.go` 追加（用一个 spy sink 断言发射）：

```go
type spyApprovalSink struct {
	pending  []string // ticketIDs
	resolved []string // ticketID:decision
}

func (s *spyApprovalSink) ApprovalPending(_ context.Context, _, ticketID, _ string, _ map[string]string) {
	s.pending = append(s.pending, ticketID)
}

func (s *spyApprovalSink) ApprovalResolved(_ context.Context, _, ticketID, decision string) {
	s.resolved = append(s.resolved, ticketID+":"+decision)
}

func TestShouldSuspendEmitsApprovalPending(t *testing.T) {
	store := approval.NewToolGateStore(t.TempDir())
	sink := &spyApprovalSink{}
	gate := New(store, WithApprovalSink(sink))
	tools := registryWithSensitiveWriteFile(t) // 见 Step 3 复用既有 test helper
	task := domain.Task{ID: "task-1", SessionID: "s1", Mode: domain.ModeManual}
	call := domain.ToolCall{ID: "call-1", Name: "write_file", Arguments: map[string]string{"path": "/tmp/x"}}

	suspend, err := gate.ShouldSuspend(context.Background(), task, []domain.ToolCall{call}, tools)
	if err != nil {
		t.Fatalf("ShouldSuspend() error = %v, want nil", err)
	}
	if !suspend {
		t.Fatal("ShouldSuspend() = false, want true for Manual+sensitive")
	}
	wantTicket := approval.TicketID("task-1", "call-1")
	if len(sink.pending) != 1 || sink.pending[0] != wantTicket {
		t.Fatalf("sink.pending = %v, want [%s]", sink.pending, wantTicket)
	}
}
```

> 注：`registryWithSensitiveWriteFile` — 若 manualgate 现有测试已有构造带 `Sensitive:true` 工具的 registry helper，复用它；否则在测试内内联构造一个 `tool.Registry`，注册一个 `Descriptor{Name:"write_file", Sensitive:true}` 的工具。实现者先读 `internal/manualgate/manualgate_test.go` 现有 helper 再定，勿臆造签名。

- [ ] **Step 2: 跑测试确认失败**

Run: `go test ./internal/manualgate/ -run TestShouldSuspendEmitsApprovalPending`
Expected: FAIL（`undefined: WithApprovalSink`）。

- [ ] **Step 3: 实现 manualgate sink 接口 + Option + ShouldSuspend 发射**

`internal/manualgate/manualgate.go`：加接口 + option，改结构体和 `New`，在 `ShouldSuspend` 开票据成功后发射。

结构体（`manualgate.go:30`）加 `sink` 字段：

```go
type ManualToolGate struct {
	store *approval.ToolGateStore
	sink  ApprovalEventSink
}
```

加接口 + option（放 `New` 上方）：

```go
// ApprovalEventSink receives best-effort notifications when a Manual-mode tool
// approval ticket is opened or resolved. It is error-less by contract: SSE is a
// notification layer, not the source of truth (the on-disk ticket is), so an
// implementation must log-and-swallow its own delivery failures rather than
// signal them back — a notification fault must never abort suspend/resume.
type ApprovalEventSink interface {
	// ApprovalPending fires once per newly opened pending ticket.
	ApprovalPending(ctx context.Context, taskID, ticketID, tool string, args map[string]string)
	// ApprovalResolved fires once a ticket is approved or denied (decision is
	// "approved" or "denied").
	ApprovalResolved(ctx context.Context, taskID, ticketID, decision string)
}

// Option configures a ManualToolGate.
type Option func(*ManualToolGate)

// WithApprovalSink attaches an optional ApprovalEventSink. A nil gate.sink means
// notifications are disabled (the gate still gates correctly).
func WithApprovalSink(sink ApprovalEventSink) Option {
	return func(g *ManualToolGate) { g.sink = sink }
}
```

改 `New`（`manualgate.go:35`）：

```go
// New returns a ManualToolGate persisting its approval tickets to store,
// configured by opts.
func New(store *approval.ToolGateStore, opts ...Option) *ManualToolGate {
	g := &ManualToolGate{store: store}
	for _, o := range opts {
		o(g)
	}
	return g
}
```

`ShouldSuspend`（`manualgate.go:82-88`）开票据成功后、`needApproval = true` 前发射：

```go
		if _, err := g.store.Open(approval.ToolApproval{
			SessionKey: sessionKey, TaskID: task.ID, ToolCallID: call.ID,
			ToolName: name, Arguments: call.Arguments,
		}); err != nil {
			return false, fmt.Errorf("open approval for task %s call %s: %w", task.ID, call.ID, err)
		}
		if g.sink != nil {
			g.sink.ApprovalPending(ctx, task.ID, ticketID, name, call.Arguments)
		}
		needApproval = true
```

（`ticketID` 已在 `manualgate.go:74` 算好，直接复用。）

- [ ] **Step 4: 跑 Step 1 测试确认通过**

Run: `go test ./internal/manualgate/ -run TestShouldSuspendEmitsApprovalPending`
Expected: PASS。

- [ ] **Step 5: 写 Decide 发 approval_resolved 测试**

`internal/manualgate/decider_test.go` 追加（复用现有 coordinator 测试骨架里的 store/scheduler 构造）：

```go
func TestDecideEmitsApprovalResolved(t *testing.T) {
	store := approval.NewToolGateStore(t.TempDir())
	rec, err := store.Open(approval.ToolApproval{
		SessionKey: "s1", TaskID: "task-1", ToolCallID: "call-1", ToolName: "write_file",
	})
	if err != nil {
		t.Fatalf("store.Open() error = %v, want nil", err)
	}
	sched := newFakeSchedulerGate(t, "task-1", "s1", domain.TaskSuspended) // 复用现有 fake，见注
	sink := &spyApprovalSink{}
	coord := NewApprovalCoordinator(store, sched, WithCoordinatorSink(sink))

	if _, err := coord.Decide(context.Background(), "task-1", rec.TicketID, approval.ApprovalApproved); err != nil {
		t.Fatalf("Decide() error = %v, want nil", err)
	}
	want := rec.TicketID + ":approved"
	if len(sink.resolved) != 1 || sink.resolved[0] != want {
		t.Fatalf("sink.resolved = %v, want [%s]", sink.resolved, want)
	}
}
```

> 注：`spyApprovalSink` 在 Step 1 已定义于同包测试文件，可直接用。`newFakeSchedulerGate` — 复用 `decider_test.go` 现有的 `SchedulerGate` fake（实现 `Get`/`Transition`）；实现者先读现有测试的 fake 定义再对齐签名，勿臆造。

- [ ] **Step 6: 实现 coordinator sink + Decide 发射**

`internal/manualgate/decider.go`：结构体（`decider.go:26`）加 `sink`：

```go
type ApprovalCoordinator struct {
	store *approval.ToolGateStore
	sched SchedulerGate
	sink  ApprovalEventSink
}
```

加 option（放 `NewApprovalCoordinator` 上方）：

```go
// CoordinatorOption configures an ApprovalCoordinator.
type CoordinatorOption func(*ApprovalCoordinator)

// WithCoordinatorSink attaches an optional ApprovalEventSink so Decide (and the
// timeout sweep that routes through it) emits approval_resolved notifications.
func WithCoordinatorSink(sink ApprovalEventSink) CoordinatorOption {
	return func(a *ApprovalCoordinator) { a.sink = sink }
}
```

改 `NewApprovalCoordinator`（`decider.go:33`）：

```go
// NewApprovalCoordinator returns an ApprovalCoordinator recording decisions to
// store and resuming tasks through sched, configured by opts.
func NewApprovalCoordinator(store *approval.ToolGateStore, sched SchedulerGate, opts ...CoordinatorOption) *ApprovalCoordinator {
	a := &ApprovalCoordinator{store: store, sched: sched}
	for _, o := range opts {
		o(a)
	}
	return a
}
```

`Decide`：在 `store.Decide` 成功后（`decider.go:52` 之后）发射（决定落盘即真相，无论后续 transition 结果）：

```go
	rec, err := a.store.Decide(sessionKey, ticketID, status)
	if err != nil {
		return approval.ToolApproval{}, fmt.Errorf("record decision for ticket %s: %w", ticketID, err)
	}
	if a.sink != nil {
		a.sink.ApprovalResolved(ctx, taskID, ticketID, string(status))
	}
```

（超时 sweep `timeout.go` 无需改：它调 `dec.Decide(...ApprovalDenied)` → 自动发 `approval_resolved(...denied)`。）

- [ ] **Step 7: 跑 manualgate 全包测试**

Run: `go test ./internal/manualgate/`
Expected: PASS（含既有测试——`New(store)` / `NewApprovalCoordinator(store, sched)` 变长参数向后兼容，旧调用不改也编译）。

- [ ] **Step 8: 写 serve 侧 sink 实现测试**

`internal/cli/approval_sink_test.go`：

```go
package cli

import (
	"context"
	"testing"
	"time"

	"github.com/stardust/legion-agent/internal/observability"
)

func TestPlatformApprovalSinkPublishesPending(t *testing.T) {
	bus := observability.NewEventBus(8)
	sink := newPlatformApprovalSink(bus, nil)
	events, cancel := bus.Subscribe(context.Background())
	defer cancel()

	sink.ApprovalPending(context.Background(), "task-1", "ticket-1", "write_file", map[string]string{"path": "/tmp/x"})

	select {
	case env := <-events:
		if env.Type != "approval_pending" {
			t.Fatalf("Type = %q, want approval_pending", env.Type)
		}
		if env.SubjectID != "task-1" || env.Data["ticket_id"] != "ticket-1" || env.Data["tool"] != "write_file" {
			t.Fatalf("envelope = %#v, want task-1/ticket-1/write_file", env)
		}
		args, ok := env.Data["arguments"].(map[string]string)
		if !ok || args["path"] != "/tmp/x" {
			t.Fatalf("Data[arguments] = %#v, want map with path", env.Data["arguments"])
		}
	case <-time.After(time.Second):
		t.Fatal("no approval_pending envelope received")
	}
}

func TestPlatformApprovalSinkPublishesResolved(t *testing.T) {
	bus := observability.NewEventBus(8)
	sink := newPlatformApprovalSink(bus, nil)
	events, cancel := bus.Subscribe(context.Background())
	defer cancel()

	sink.ApprovalResolved(context.Background(), "task-1", "ticket-1", "denied")

	select {
	case env := <-events:
		if env.Type != "approval_resolved" || env.Data["ticket_id"] != "ticket-1" || env.Data["decision"] != "denied" {
			t.Fatalf("envelope = %#v, want approval_resolved/ticket-1/denied", env)
		}
	case <-time.After(time.Second):
		t.Fatal("no approval_resolved envelope received")
	}
}

func TestPlatformApprovalSinkSwallowsPublishError(t *testing.T) {
	bus := observability.NewEventBus(8)
	if err := bus.Close(); err != nil {
		t.Fatalf("bus.Close() error = %v, want nil", err)
	}
	sink := newPlatformApprovalSink(bus, nil)
	// Closed bus: Publish errors internally; sink must not panic and (being
	// error-less) simply logs Warn — approval flow is never blocked.
	sink.ApprovalPending(context.Background(), "task-1", "ticket-1", "write_file", nil)
	sink.ApprovalResolved(context.Background(), "task-1", "ticket-1", "approved")
}
```

- [ ] **Step 9: 实现 serve 侧 sink**

`internal/cli/approval_sink.go`：

```go
package cli

import (
	"context"
	"io"
	"log/slog"

	"github.com/stardust/legion-agent/internal/observability"
)

// platformApprovalSink implements manualgate.ApprovalEventSink by translating
// approval lifecycle notifications into observability.EventEnvelope values on the
// platform bus behind /v1/events. Publish failures are logged Warn and swallowed:
// SSE is a best-effort notification layer, and the on-disk approval ticket — not
// this event — is the source of truth (design §3.3).
type platformApprovalSink struct {
	platform *observability.EventBus
	logger   *slog.Logger
}

// newPlatformApprovalSink returns a sink publishing to platform. A nil logger
// discards. platform must not be nil.
func newPlatformApprovalSink(platform *observability.EventBus, logger *slog.Logger) *platformApprovalSink {
	if platform == nil {
		panic("newPlatformApprovalSink: platform event bus must not be nil")
	}
	if logger == nil {
		logger = slog.New(slog.NewTextHandler(io.Discard, nil))
	}
	return &platformApprovalSink{platform: platform, logger: logger}
}

// ApprovalPending publishes an approval_pending envelope. arguments are carried
// as-is; the SSE write boundary (sanitizeEventData) truncates and strips
// sensitive sub-keys before they leave the process.
func (s *platformApprovalSink) ApprovalPending(ctx context.Context, taskID, ticketID, tool string, args map[string]string) {
	data := map[string]any{"task_id": taskID, "ticket_id": ticketID, "tool": tool}
	if args != nil {
		data["arguments"] = args
	}
	s.publish(ctx, "approval_pending", taskID, ticketID, data)
}

// ApprovalResolved publishes an approval_resolved envelope.
func (s *platformApprovalSink) ApprovalResolved(ctx context.Context, taskID, ticketID, decision string) {
	s.publish(ctx, "approval_resolved", taskID, ticketID, map[string]any{
		"task_id": taskID, "ticket_id": ticketID, "decision": decision,
	})
}

func (s *platformApprovalSink) publish(ctx context.Context, eventType, taskID, ticketID string, data map[string]any) {
	if err := s.platform.Publish(ctx, observability.EventEnvelope{
		Type:      eventType,
		SubjectID: taskID,
		Data:      data,
	}); err != nil {
		s.logger.Warn("approval event sink: platform publish failed",
			"type", eventType, "task_id", taskID, "ticket_id", ticketID, "error", err)
	}
}
```

- [ ] **Step 10: serve 接线 sink**

`command.go`：sink 在 `platformEvents`（Task 2 构造）+ logger（Task 2 hoist）之后可用。`@1740`/`@1745` 改为：

```go
	approvalSink := newPlatformApprovalSink(platformEvents, logger)
	manualGate := manualgate.New(toolGateStore, manualgate.WithApprovalSink(approvalSink))
	// ...（注释保持原样）...
	approvalCoordinator := manualgate.NewApprovalCoordinator(toolGateStore, liveTasks, manualgate.WithCoordinatorSink(approvalSink))
```

（`toolGateStore` 在 `@1739` 定义，`approvalSink` 紧随其后构造即可；确保 `approvalSink` 定义在 `manualGate` 之前。）

- [ ] **Step 11: 跑测试确认全通过**

Run: `go build ./... && go test ./internal/manualgate/ ./internal/cli/`
Expected: PASS。

- [ ] **Step 12: gofmt + 提交**

```bash
gofmt -w internal/manualgate/manualgate.go internal/manualgate/decider.go internal/manualgate/manualgate_test.go internal/manualgate/decider_test.go internal/cli/approval_sink.go internal/cli/approval_sink_test.go internal/cli/command.go
git add internal/manualgate/ internal/cli/approval_sink.go internal/cli/approval_sink_test.go internal/cli/command.go
git commit -m "feat(manualgate): 审批事件 sink，ShouldSuspend/Decide 发 approval_pending/resolved"
```

---

### Task 4: SSE 脱敏改递归 + 截断

现 `sanitizeEventData`（`events.go:59`）只删顶层 key。`approval_pending.arguments` 是嵌套 map（含 `write_file` 的 `content`/`input` 等可能巨大/敏感的子键）不会被处理。改为递归 + 大值截断。

**Files:**
- Modify: `internal/server/events.go`
- Modify: `internal/server/events_test.go`

**Interfaces:**
- Consumes: `EventEnvelope.Data map[string]any`（其中 `arguments` 可能是 `map[string]string`，来自 Task 3 sink）。
- Produces: `sanitizeStringMap(map[string]string) map[string]any`（Task 5 列票据端点复用）；`truncateEventString(string) string`。

- [ ] **Step 1: 写递归/截断脱敏失败测试**

`internal/server/events_test.go` 追加：

```go
func TestSSEEventsSanitizesNestedArgumentsAndTruncates(t *testing.T) {
	bus := observability.NewEventBus(8)
	srv := NewHTTPServer(Config{AdminToken: "token", PlatformEvents: bus})
	req := httptest.NewRequest(http.MethodGet, "/v1/events", nil)
	req.Header.Set("Authorization", "Bearer token")
	rec := httptest.NewRecorder()

	big := strings.Repeat("A", 2000)
	go func() {
		if err := bus.Publish(context.Background(), observability.EventEnvelope{
			Type:      "approval_pending",
			SubjectID: "task-1",
			Data: map[string]any{
				"task_id":   "task-1",
				"ticket_id": "ticket-1",
				"tool":      "write_file",
				"arguments": map[string]string{
					"path":    "/tmp/x",
					"api_key": "SUPER-SECRET-KEY",
					"content": big,
				},
			},
		}); err != nil {
			t.Errorf("Publish(approval_pending) error = %v, want nil", err)
		}
		if err := bus.Close(); err != nil {
			t.Errorf("Close() error = %v, want nil", err)
		}
	}()

	srv.ServeHTTP(rec, req)
	body := rec.Body.String()

	if !strings.Contains(body, "/tmp/x") {
		t.Fatalf("body = %q, want non-sensitive arg path present", body)
	}
	if strings.Contains(body, "SUPER-SECRET-KEY") || strings.Contains(body, "api_key") {
		t.Fatalf("body leaked sensitive nested arg: %q", body)
	}
	if strings.Contains(body, big) {
		t.Fatalf("body carried untruncated 2000-byte content")
	}
	if !strings.Contains(body, "truncated") {
		t.Fatalf("body = %q, want truncation marker for large content", body)
	}
}
```

- [ ] **Step 2: 跑测试确认失败**

Run: `go test ./internal/server/ -run TestSSEEventsSanitizesNestedArgumentsAndTruncates`
Expected: FAIL（当前只删顶层，嵌套 api_key/大 content 泄漏）。

- [ ] **Step 3: 实现递归 + 截断**

`internal/server/events.go` 把 `sanitizeEventData`（`@59-68`）替换为递归版，并加辅助函数 + 常量：

```go
// maxEventStringLen bounds any single string value emitted over SSE. Larger
// values (e.g. a write_file content argument) are truncated with a marker so a
// pending-approval event cannot ship an unbounded/secret-laden payload.
const maxEventStringLen = 512

// sanitizeEventData strips sensitive keys and truncates large string values from
// an event payload before it leaves the process, recursing into nested maps so a
// sensitive sub-key (e.g. arguments.api_key) or a huge sub-value cannot slip
// through under a benign top-level key like "arguments".
func sanitizeEventData(data map[string]any) map[string]any {
	sanitized := make(map[string]any, len(data))
	for key, value := range data {
		if isSensitiveEventField(key) {
			continue
		}
		sanitized[key] = sanitizeEventValue(value)
	}
	return sanitized
}

func sanitizeEventValue(value any) any {
	switch v := value.(type) {
	case map[string]any:
		return sanitizeEventData(v)
	case map[string]string:
		return sanitizeStringMap(v)
	case string:
		return truncateEventString(v)
	default:
		return v
	}
}

// sanitizeStringMap strips sensitive keys and truncates values of a string map
// (e.g. an approval ticket's Arguments). It returns a map[string]any so nested
// results compose with sanitizeEventValue. Exported-within-package for reuse by
// the /v1/approvals list handler.
func sanitizeStringMap(m map[string]string) map[string]any {
	out := make(map[string]any, len(m))
	for key, value := range m {
		if isSensitiveEventField(key) {
			continue
		}
		out[key] = truncateEventString(value)
	}
	return out
}

func truncateEventString(s string) string {
	if len(s) <= maxEventStringLen {
		return s
	}
	return s[:maxEventStringLen] + fmt.Sprintf("…[truncated %d bytes]", len(s)-maxEventStringLen)
}
```

（`fmt` 已在 `events.go` import。）

- [ ] **Step 4: 跑测试确认通过（含既有顶层脱敏测试不回退）**

Run: `go test ./internal/server/ -run TestSSEEvents`
Expected: PASS（新递归测试 + 既有 `TestSSEEventsFiltersAndSanitizesPayload` 都绿）。

- [ ] **Step 5: gofmt + 提交**

```bash
gofmt -w internal/server/events.go internal/server/events_test.go
git add internal/server/events.go internal/server/events_test.go
git commit -m "feat(server): SSE 脱敏改递归+截断，堵住嵌套 arguments 泄漏"
```

---

### Task 5: 列待决票据端点（GET /v1/approvals?status=pending）

SSE 满 buffer 会丢事件（at-most-once）——`approval_pending` 丢了 = GUI 漏审批提示。补一个列盘上待决票据的端点供 GUI 对账。

**Files:**
- Create: `internal/server/approvals_list.go`、`internal/server/approvals_list_test.go`
- Modify: `internal/server/http.go`（Config/字段/New/路由）
- Modify: `internal/cli/command.go`（Config 接 `ApprovalTickets`）

**Interfaces:**
- Consumes: `approval.ToolGateStore.ListPending() ([]approval.ToolApproval, error)`（`toolgate_store.go:211`）；`sanitizeStringMap`（Task 4）。
- Produces: `server.ApprovalLister` 接口；`GET /v1/approvals` → JSON `{"approvals":[...]}`。

- [ ] **Step 1: 写端点失败测试**

`internal/server/approvals_list_test.go`：

```go
package server

import (
	"encoding/json"
	"net/http"
	"net/http/httptest"
	"testing"

	"github.com/stardust/legion-agent/internal/approval"
)

type fakeApprovalLister struct {
	pending []approval.ToolApproval
	err     error
}

func (f fakeApprovalLister) ListPending() ([]approval.ToolApproval, error) {
	return f.pending, f.err
}

func TestListApprovalsReturnsPendingSanitized(t *testing.T) {
	lister := fakeApprovalLister{pending: []approval.ToolApproval{{
		TicketID: "ticket-1", TaskID: "task-1", SessionKey: "s1", ToolName: "write_file",
		ToolCallID: "call-1", Status: approval.ApprovalPending,
		Arguments: map[string]string{"path": "/tmp/x", "api_key": "SECRET"},
	}}}
	srv := NewHTTPServer(Config{AdminToken: "token", ApprovalTickets: lister})
	req := httptest.NewRequest(http.MethodGet, "/v1/approvals?status=pending", nil)
	req.Header.Set("Authorization", "Bearer token")
	rec := httptest.NewRecorder()

	srv.ServeHTTP(rec, req)

	if rec.Code != http.StatusOK {
		t.Fatalf("status = %d, want 200; body=%s", rec.Code, rec.Body.String())
	}
	var resp struct {
		Approvals []map[string]any `json:"approvals"`
	}
	if err := json.Unmarshal(rec.Body.Bytes(), &resp); err != nil {
		t.Fatalf("Unmarshal error = %v, body=%s", err, rec.Body.String())
	}
	if len(resp.Approvals) != 1 {
		t.Fatalf("approvals len = %d, want 1", len(resp.Approvals))
	}
	got := resp.Approvals[0]
	if got["ticket_id"] != "ticket-1" || got["tool_name"] != "write_file" {
		t.Fatalf("approval = %#v, want ticket-1/write_file", got)
	}
	args, ok := got["arguments"].(map[string]any)
	if !ok || args["path"] != "/tmp/x" {
		t.Fatalf("arguments = %#v, want sanitized map with path", got["arguments"])
	}
	if _, leaked := args["api_key"]; leaked {
		t.Fatalf("arguments leaked sensitive api_key: %#v", args)
	}
}

func TestListApprovalsRejectsUnsupportedStatus(t *testing.T) {
	srv := NewHTTPServer(Config{AdminToken: "token", ApprovalTickets: fakeApprovalLister{}})
	req := httptest.NewRequest(http.MethodGet, "/v1/approvals?status=approved", nil)
	req.Header.Set("Authorization", "Bearer token")
	rec := httptest.NewRecorder()
	srv.ServeHTTP(rec, req)
	if rec.Code != http.StatusBadRequest {
		t.Fatalf("status = %d, want 400 for unsupported status filter", rec.Code)
	}
}

func TestListApprovalsUnavailableWithoutStore(t *testing.T) {
	srv := NewHTTPServer(Config{AdminToken: "token"})
	req := httptest.NewRequest(http.MethodGet, "/v1/approvals", nil)
	req.Header.Set("Authorization", "Bearer token")
	rec := httptest.NewRecorder()
	srv.ServeHTTP(rec, req)
	if rec.Code != http.StatusServiceUnavailable {
		t.Fatalf("status = %d, want 503 when approval store unwired", rec.Code)
	}
}
```

- [ ] **Step 2: 跑测试确认失败**

Run: `go test ./internal/server/ -run TestListApprovals`
Expected: FAIL（`unknown field ApprovalTickets` / 路由不存在）。

- [ ] **Step 3: http.go 加 Config 字段 + 路由**

`internal/server/http.go`：
- `Config`（`@84`）在 `ToolApprovals ApprovalDecider` 下加：`ApprovalTickets ApprovalLister`。
- `HTTPServer`（`@108`）在 `toolApprovals` 下加：`approvalTickets ApprovalLister`。
- `NewHTTPServer`（`@149`）在 `toolApprovals: cfg.ToolApprovals,` 下加：`approvalTickets: cfg.ApprovalTickets,`。
- 路由（`@211` `GET /v1/events` case 附近）加：

```go
		case r.Method == http.MethodGet && r.URL.Path == "/v1/approvals":
			s.handleListApprovals(rec, r)
```

- [ ] **Step 4: 实现 handler + 接口**

`internal/server/approvals_list.go`：

```go
package server

import (
	"net/http"

	"github.com/stardust/legion-agent/internal/approval"
)

// ApprovalLister lists persisted Manual-mode approval tickets so a UI can
// reconcile pending approvals it may have missed over the at-most-once SSE
// stream. It is satisfied by *approval.ToolGateStore.
type ApprovalLister interface {
	ListPending() ([]approval.ToolApproval, error)
}

// handleListApprovals serves GET /v1/approvals?status=pending, returning every
// on-disk pending ticket with sensitive/large arguments sanitized. Only the
// "pending" status filter (or none) is supported today; any other value is a
// 400 rather than a silently-ignored filter.
func (s *HTTPServer) handleListApprovals(w http.ResponseWriter, r *http.Request) {
	if s.approvalTickets == nil {
		writeError(w, http.StatusServiceUnavailable, "approval store is unavailable")
		return
	}
	if status := r.URL.Query().Get("status"); status != "" && status != string(approval.ApprovalPending) {
		writeError(w, http.StatusBadRequest, "unsupported status filter; only 'pending' is supported")
		return
	}
	pending, err := s.approvalTickets.ListPending()
	if err != nil {
		s.logger.Error("list pending approvals", "error", err)
		writeError(w, http.StatusInternalServerError, "list pending approvals")
		return
	}
	out := make([]map[string]any, 0, len(pending))
	for _, t := range pending {
		out = append(out, map[string]any{
			"ticket_id":    t.TicketID,
			"task_id":      t.TaskID,
			"session_key":  t.SessionKey,
			"tool_name":    t.ToolName,
			"tool_call_id": t.ToolCallID,
			"arguments":    sanitizeStringMap(t.Arguments),
			"status":       string(t.Status),
			"created_at":   t.CreatedAt,
		})
	}
	writeJSON(w, http.StatusOK, map[string]any{"approvals": out})
}
```

（`writeError`/`writeJSON` 是 server 包既有 helper；`sanitizeStringMap` 来自 Task 4。）

- [ ] **Step 5: 跑测试确认通过**

Run: `go test ./internal/server/ -run TestListApprovals`
Expected: PASS（3 个子测试）。

- [ ] **Step 6: command.go 接线 ApprovalTickets**

`command.go:1957` 的 `server.Config{...}` 块，在 `ToolApprovals: approvalCoordinator,` 下加：

```go
		ApprovalTickets:     toolGateStore,
```

（`toolGateStore` 是 `*approval.ToolGateStore`，结构性满足 `server.ApprovalLister`。）

- [ ] **Step 7: 跑构建 + server/cli 测试**

Run: `go build ./... && go test ./internal/server/ ./internal/cli/`
Expected: PASS。

- [ ] **Step 8: gofmt + 提交**

```bash
gofmt -w internal/server/approvals_list.go internal/server/approvals_list_test.go internal/server/http.go internal/cli/command.go
git add internal/server/approvals_list.go internal/server/approvals_list_test.go internal/server/http.go internal/cli/command.go
git commit -m "feat(server): GET /v1/approvals 列待决票据供 GUI 对账"
```

---

### Task 6: serve 全链路 e2e + 完成门禁

整合 1–5，端到端验证生命周期事件经桥接流到 SSE，并过全量构建/测试/race 门禁。

**Files:**
- Modify: `internal/cli/command_test.go`

**Interfaces:**
- Consumes: 全部前序任务的装配（serve 起 platformEvents + 桥接 + sink + 列端点）。`observability.EventBus.Subscribe` 对新订阅者**重放已缓冲事件**（`eventbus.go:77`）——故任务跑完后再订阅仍能收到 buffered `task_completed`，测试可确定性断言。

- [ ] **Step 1: 写全链路 e2e 测试**

`internal/cli/command_test.go` 追加（POST 任务→轮询完成→订阅 /v1/events 读回放的 task_completed）：

```go
func TestServeCommandStreamsLifecycleEventsOverSSE(t *testing.T) {
	t.Parallel()
	ctx, cancel := context.WithCancel(context.Background())
	defer cancel()
	configPath := filepath.Join(t.TempDir(), "agent.json")
	if err := os.WriteFile(configPath, []byte(`{"service": {"background_interval": "1h"}}`), 0o600); err != nil {
		t.Fatalf("WriteFile(%q) error = %v, want nil", configPath, err)
	}
	listener, err := net.Listen("tcp", "127.0.0.1:0")
	if err != nil {
		t.Fatalf("net.Listen() error = %v, want nil", err)
	}
	addr := listener.Addr().String()
	if err := listener.Close(); err != nil {
		t.Fatalf("listener.Close() error = %v, want nil", err)
	}

	var out bytes.Buffer
	root := NewRoot(app.New(), &out)
	root.SetContext(ctx)
	root.SetArgs([]string{"serve", "--config", configPath, "--addr", addr})
	done := make(chan error, 1)
	go func() { done <- root.Execute() }()

	// Drive one task to completion (demo maas), so task_started/task_completed
	// are buffered on the platform bus.
	resp, err := waitForPostTask(t, "http://"+addr+"/v1/tasks",
		`{"id":"sse-task-1","company_id":"c1","input":"hello sse"}`)
	if err != nil {
		cancel()
		t.Fatalf("POST /v1/tasks error = %v, want nil", err)
	}
	if err := resp.Body.Close(); err != nil {
		cancel()
		t.Fatalf("Body.Close() error = %v, want nil", err)
	}

	// Subscribe AFTER the task ran; buffered events are replayed to new
	// subscribers, so task_completed is deterministically available.
	streamCtx, streamCancel := context.WithTimeout(ctx, 5*time.Second)
	defer streamCancel()
	found := false
	deadline := time.Now().Add(5 * time.Second)
	for time.Now().Before(deadline) && !found {
		req, reqErr := http.NewRequestWithContext(streamCtx, http.MethodGet,
			"http://"+addr+"/v1/events?type=task_completed", nil)
		if reqErr != nil {
			t.Fatalf("NewRequest error = %v, want nil", reqErr)
		}
		sseResp, doErr := http.DefaultClient.Do(req)
		if doErr != nil {
			time.Sleep(50 * time.Millisecond)
			continue
		}
		buf := make([]byte, 4096)
		n, _ := sseResp.Body.Read(buf) // one read is enough; replay is immediate
		_ = sseResp.Body.Close()
		if n > 0 && strings.Contains(string(buf[:n]), "event: task_completed") {
			found = true
		}
	}
	cancel()
	select {
	case execErr := <-done:
		if execErr != nil {
			t.Fatalf("Execute(serve) error = %v, want nil", execErr)
		}
	case <-time.After(2 * time.Second):
		t.Fatal("Execute(serve) did not stop")
	}
	if !found {
		t.Fatal("GET /v1/events never streamed task_completed, want lifecycle event bridged to SSE")
	}
}
```

> 若 demo maas 默认不产生 `task_completed`（实现者先跑一次确认），改断言 `task_started`（`runtime.go:227` 必发）。二者皆可证明桥接生效——选实际会出现的那个。

- [ ] **Step 2: 跑 e2e 测试**

Run: `go test ./internal/cli/ -run TestServeCommandStreamsLifecycleEventsOverSSE -v`
Expected: PASS。若 flaky（读时机），增大 deadline 或改断言 `task_started`。

- [ ] **Step 3: 全量构建 + vet + 测试（Windows 宿主）**

Run:
```powershell
go build ./... ; go vet ./... ; go test ./...
```
Expected: 全 PASS。

- [ ] **Step 4: gofmt 检查（只验触碰文件 LF 干净）**

Run: `gofmt -l internal/eventbridge/ internal/cli/approval_sink.go internal/cli/approval_sink_test.go internal/server/events.go internal/server/approvals_list.go internal/server/approvals_list_test.go internal/manualgate/`
Expected: 空输出（仓库 `core.autocrlf=true` 下 `gofmt -l` 可能误报既有 CRLF 文件——只关心本任务触碰的文件；若误报，`gofmt -w` 后靠 `git diff --stat` 确认只改了内容不是整仓重排 CRLF）。

- [ ] **Step 5: WSL race 门禁**

Run:
```bash
wsl -d Ubuntu-22.04 -- bash -lc 'cd /mnt/f/source/stardust/Legion/legion/legionAgent && GOOS=linux GOARCH=amd64 CGO_ENABLED=1 CC=gcc $HOME/sdk/go/bin/go test -race ./internal/eventbridge/ ./internal/server/ ./internal/manualgate/ ./internal/cli/ ./internal/observability/'
```
Expected: 全 PASS，无 race 报告。

- [ ] **Step 6: 提交（若 Step 1 有新测试文件改动未提交）**

```bash
gofmt -w internal/cli/command_test.go
git add internal/cli/command_test.go
git commit -m "test(serve): e2e 验证生命周期事件经桥接流到 /v1/events SSE"
```

---

## Self-Review

**Spec coverage（spec §4.5 + 数据模型 §6 SSE 项 + API §7 + 错误处理 §8 + 测试 §9 SSE 项）:**
- 接活 `/v1/events` + 桥接 RuntimeEvent→EventEnvelope → Task 1 + Task 2 ✅
- 推送 `approval_pending{task_id,ticket_id,tool,arguments}` / `approval_resolved{ticket_id,decision}` → Task 3 ✅
- 现有生命周期事件（task_started/tool_executed/task_completed…）流到 SSE → Task 1 桥接零改动 + Task 6 e2e ✅
- 修复恒 503 latent bug → Task 2（Config.PlatformEvents 接线）✅
- SSE 桥接失败记录不静默丢（§8）→ Task 1 Bridge Warn + Task 3 sink Warn ✅
- arguments 脱敏（§3.4.1 选 b：递归 + 截断）→ Task 4 ✅
- list-pending 端点对账（§3.4.2）→ Task 5 ✅
- SSE 测试：approval_pending 字段完整 + 脱敏生效（§9）→ Task 3/4 断言 ✅

**Placeholder scan:** 无 TODO/TBD；每个 code step 给出完整 Go 代码；测试 helper（`registryWithSensitiveWriteFile`/`newFakeSchedulerGate`）明确标注「复用现有测试定义，实现者先读再对齐，勿臆造」——这是对既有测试基建的诚实引用，非占位。

**Type consistency:** `ApprovalEventSink` 接口 error-less（两处 impl `platformApprovalSink` + spy 一致）；`WithApprovalSink`(Option) vs `WithCoordinatorSink`(CoordinatorOption) 两个不同 option 类型分属 gate/coordinator，无混用；`sanitizeStringMap` 定义于 Task 4、复用于 Task 5，签名 `map[string]string → map[string]any` 一致；`ApprovalLister.ListPending` 签名与 `approval.ToolGateStore.ListPending` 一致（`[]approval.ToolApproval, error`）；事件类型全下划线（`task_completed`/`approval_pending`/`approval_resolved`）。

**风险提示（给执行者）:** 任务 1（桥接事件语义：丢弃/顺序/重放）+ 任务 3（sink fail-loud 边界）是核心，用 opus reviewer。任务 3 的 spy sink 与 `newFakeSchedulerGate` 依赖既有测试基建，务必先读 `internal/manualgate/*_test.go` 现状。
