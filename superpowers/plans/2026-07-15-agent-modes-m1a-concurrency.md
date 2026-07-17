# Milestone 1a — 并发任务执行基础 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 把 legionAgent 的任务执行从「全服务单 goroutine 串行、一次一个」改为「有上限的每任务一 goroutine 并发」，一个慢/挂起任务不再阻塞其他任务。

**Architecture:** `Coordinator.Heartbeat` 当前同步 inline 跑完整条流水线（`Next → lock → Running → resolve → trust → RunTask → eval → review → Done`），占死唯一的后台调度 goroutine。本里程碑把「拿到已锁定任务之后的整条流水线」抽成 `runAssigned`，`Heartbeat` 在每次 tick 里**趁有空闲 worker 槽就连续 `Next` 并各起一个 goroutine 跑 `runAssigned`**，立即返回。并发上限由信号量（buffered channel）控制，优雅停机用 `sync.WaitGroup` 排空。底层共享态（`Scheduler`/`LockStore`/`MemoryEventBus` 均已 mutex 保护）容忍并发；`-race` 佐证。

**Tech Stack:** Go 1.26，`internal/runtime`（Coordinator）、`internal/task`（Scheduler/BackgroundScheduler/LockStore）、`internal/config`，`go test -race`。

## Global Constraints

- **Fail-Loud 铁律**（`legionAgent/CLAUDE.md`）：goroutine 顶层错误必须结构化记录（用 `events.Publish` 学习事件 / audit），绝不静默吞；`runAssigned` 内每个失败路径都已 `Transition(TaskFailed)` + 包装 error，并发化后这些必须保留并在 goroutine 顶层记录。
- **完成标准**：`cd legion/legionAgent && go build ./... && go vet ./... && go test ./...` 全绿、`gofmt -l .` 为空；**并发相关测试须在 `-race` 下跑**（`go test -race ./internal/runtime/ ./internal/task/`）。
- **并发上限**可配，默认 4；`0`/负值视为默认。
- **不改** `RunTask` 内部逻辑（那是里程碑 1b）；本里程碑只改调度/派发层。
- **不引入** per-task 持久化 / 挂起恢复（里程碑 1b）；`TaskSuspended` 现有语义（trust-block、hard-loop）保持不变。
- 工作目录：命令相对仓库根 `F:\source\stardust\Legion`；Go module 在 `legion/legionAgent`。
- 提交：本仓库当前分支（master）。每个 task 末尾提交；提交信息以 `Co-Authored-By: Claude Opus 4.8 <noreply@anthropic.com>` 结尾。

## 文件结构

- `internal/config/config.go` — 加 `RuntimeConfig.MaxConcurrentTasks int`（`runtime.max_concurrent_tasks`）。
- `internal/runtime/coordinator.go` — 加信号量/WaitGroup 字段 + `MaxWorkers` 配置；抽出 `runAssigned`；`Heartbeat` 改并发派发；加 `Wait()` 排空。
- `internal/runtime/coordinator_test.go` — 现有同步断言（靠 Heartbeat 返回值）迁移为「派发后轮询 `scheduler.Get` 到终态」。
- `internal/runtime/coordinator_concurrency_test.go`（新）— `-race` 并发压测。
- `internal/cli/command.go` — 构造 Coordinator 时传 `MaxWorkers`（读 `cfg.Runtime.MaxConcurrentTasks`）；serve 停机时 `coordinator.Wait()`。

---

### Task 1: 配置项 `runtime.max_concurrent_tasks`

**Files:**
- Modify: `internal/config/config.go`（`RuntimeConfig` 结构 + `defaultConfig`）
- Test: `internal/config/config_test.go`

- [ ] **Step 1: 写失败测试**

追加到 `internal/config/config_test.go`：
```go
func TestLoadMaxConcurrentTasksDefault(t *testing.T) {
	cfg, err := Load(context.Background(), Options{Path: ""})
	if err != nil {
		t.Fatalf("Load: %v", err)
	}
	if cfg.Runtime.MaxConcurrentTasks != 4 {
		t.Fatalf("default MaxConcurrentTasks = %d, want 4", cfg.Runtime.MaxConcurrentTasks)
	}
}
```

- [ ] **Step 2: 跑测试确认失败**

Run: `cd legion/legionAgent && go test ./internal/config/ -run TestLoadMaxConcurrentTasks -v`
Expected: FAIL（字段不存在，编译错误 `cfg.Runtime.MaxConcurrentTasks undefined`）。

- [ ] **Step 3: 加字段 + 默认值**

在 `internal/config/config.go` 的 `RuntimeConfig` 结构加字段（紧随 `LazyTools`）：
```go
	// MaxConcurrentTasks caps how many tasks the coordinator runs simultaneously,
	// each on its own goroutine. Defaults to 4; 0 or negative means the default.
	MaxConcurrentTasks int `json:"max_concurrent_tasks"`
```
在 `defaultConfig()` 的 `Runtime:` 字面量里加 `MaxConcurrentTasks: 4,`。

- [ ] **Step 4: 跑测试确认通过**

Run: `cd legion/legionAgent && go test ./internal/config/ -run TestLoadMaxConcurrentTasks -v`
Expected: PASS。

- [ ] **Step 5: 验证关卡 + 提交**

Run: `cd legion/legionAgent && go build ./... && gofmt -l internal/config/config.go internal/config/config_test.go`
Expected: 无输出。
```bash
git add internal/config/config.go internal/config/config_test.go
git commit -m "feat(config): add runtime.max_concurrent_tasks (default 4)

Co-Authored-By: Claude Opus 4.8 <noreply@anthropic.com>"
```

---

### Task 2: 抽出 `Coordinator.runAssigned`（纯重构，行为不变）

先把「拿到任务并锁定后的整条流水线」抽成独立方法，**不改行为、不并发**，让下一步只改派发。

**Files:**
- Modify: `internal/runtime/coordinator.go`

**Interfaces:**
- Produces: `func (c *Coordinator) runAssigned(ctx context.Context, taskToRun domain.Task) (domain.Task, bool, error)` —— 等价于现 `Heartbeat` 从「`c.locks.TryLock` 之后」到函数结尾的全部逻辑。

- [ ] **Step 1: 抽方法**

在 `coordinator.go`：把现 `Heartbeat`（`coordinator.go:81`）里**从 `locked, err := c.locks.TryLock(...)`（第 89 行）到最后 `return c.currentTask(ctx, taskToRun.ID)`（第 203 行）** 的整段原样剪切为新方法体：
```go
// runAssigned executes the full pipeline for a task the scheduler has already
// handed out: acquire its lock, mark it running, resolve its runner, run it,
// then evaluate/review and land it in a terminal (or suspended) state. It is the
// unit spawned per-task by Heartbeat.
func (c *Coordinator) runAssigned(ctx context.Context, taskToRun domain.Task) (domain.Task, bool, error) {
	locked, err := c.locks.TryLock(ctx, taskToRun.ID, c.agent.ID, c.lockTTL)
	if err != nil {
		return domain.Task{}, false, fmt.Errorf("lock task: %w", err)
	}
	if !locked {
		return domain.Task{}, false, nil
	}
	// ...（第 96–203 行原样搬入）...
	return c.currentTask(ctx, taskToRun.ID)
}
```
`Heartbeat` 暂时改为顺序调用（本步不并发，仅证明重构等价）：
```go
func (c *Coordinator) Heartbeat(ctx context.Context) (domain.Task, bool, error) {
	taskToRun, ok, err := c.scheduler.Next(ctx, c.agent.ID)
	if err != nil {
		return domain.Task{}, false, fmt.Errorf("schedule next task: %w", err)
	}
	if !ok {
		return domain.Task{}, false, nil
	}
	return c.runAssigned(ctx, taskToRun)
}
```

- [ ] **Step 2: 全量测试确认行为不变**

Run: `cd legion/legionAgent && go test ./internal/runtime/ -v 2>&1 | tail -20`
Expected: 全 PASS（纯重构，现有 coordinator 测试仍靠 Heartbeat 同步返回值，未变）。

- [ ] **Step 3: 验证关卡 + 提交**

Run: `cd legion/legionAgent && go build ./... && go vet ./internal/runtime/ && gofmt -l internal/runtime/coordinator.go`
Expected: 无输出。
```bash
git add internal/runtime/coordinator.go
git commit -m "refactor(runtime): extract Coordinator.runAssigned from Heartbeat

Co-Authored-By: Claude Opus 4.8 <noreply@anthropic.com>"
```

---

### Task 3: Coordinator 并发派发（信号量 + WaitGroup）

**Files:**
- Modify: `internal/runtime/coordinator.go`（`CoordinatorConfig`、`Coordinator`、`NewCoordinator`、`Heartbeat`、新增 `Wait`）

**Interfaces:**
- Consumes: `runAssigned`（Task 2）。
- Produces:
  - `CoordinatorConfig.MaxWorkers int`
  - `func (c *Coordinator) Wait()` —— 阻塞至所有在飞任务 goroutine 结束（停机排空）。
  - `Heartbeat` 语义变为：趁空闲 worker 槽连续派发 pending 任务到 goroutine，返回 `(domain.Task{}, dispatched, nil)`。

- [ ] **Step 1: 加字段 + 构造**

`CoordinatorConfig` 加：
```go
	// MaxWorkers caps concurrent task goroutines. 0 or negative → default 4.
	MaxWorkers int
```
`Coordinator` 结构加：
```go
	sem chan struct{}
	wg  sync.WaitGroup
```
（import 补 `"sync"`。）
`NewCoordinator` 里，在 `return &Coordinator{...}` 前：
```go
	if cfg.MaxWorkers <= 0 {
		cfg.MaxWorkers = 4
	}
```
并在字面量加 `sem: make(chan struct{}, cfg.MaxWorkers),`。

- [ ] **Step 2: 改 `Heartbeat` 为并发派发**

替换 `Heartbeat`：
```go
// Heartbeat dispatches as many pending tasks as there are free worker slots,
// each on its own goroutine, then returns immediately. A slow or suspended task
// no longer blocks others. The returned Task is always zero-valued now (work is
// async); the bool reports whether at least one task was dispatched this tick.
func (c *Coordinator) Heartbeat(ctx context.Context) (domain.Task, bool, error) {
	dispatched := false
	for {
		select {
		case c.sem <- struct{}{}: // acquired a worker slot
		default:
			return domain.Task{}, dispatched, nil // all workers busy
		}
		taskToRun, ok, err := c.scheduler.Next(ctx, c.agent.ID)
		if err != nil {
			<-c.sem
			return domain.Task{}, dispatched, fmt.Errorf("schedule next task: %w", err)
		}
		if !ok {
			<-c.sem // no pending task; release the slot
			return domain.Task{}, dispatched, nil
		}
		c.wg.Add(1)
		go func(t domain.Task) {
			defer c.wg.Done()
			defer func() { <-c.sem }()
			if _, _, err := c.runAssigned(ctx, t); err != nil {
				// Goroutine top-level: never swallow. runAssigned already
				// transitioned the task to Failed on error; record the reason so
				// a failed run is diagnosable rather than vanishing.
				_ = c.publishLearning(ctx, c.agent.ID, t.ID, evolution.SignalFailure, "task_run_error", true)
			}
		}(taskToRun)
		dispatched = true
	}
}
```

- [ ] **Step 3: 加 `Wait`（停机排空）**

```go
// Wait blocks until every in-flight task goroutine has finished. The serve
// shutdown path calls it so tasks are not abandoned mid-run.
func (c *Coordinator) Wait() {
	c.wg.Wait()
}
```

- [ ] **Step 4: 迁移现有 coordinator 测试为「派发后轮询到终态」**

`Heartbeat` 不再同步返回跑完的任务，现有断言其返回 Task 状态的测试会失败。**先读** `internal/runtime/coordinator_test.go`，把每个「调用 `Heartbeat` 后断言返回 task 状态 == Done/Failed/...」的用例改为：调用 `Heartbeat` 派发 → 轮询 `scheduler.Get` 到终态再断言。加一个测试辅助：
```go
// awaitTerminal polls the scheduler until the task reaches a terminal or
// suspended status (or times out), so tests can assert the async pipeline's
// outcome after Heartbeat dispatches it.
func awaitTerminal(t *testing.T, sched *task.Scheduler, id string) domain.Task {
	t.Helper()
	deadline := time.Now().Add(2 * time.Second)
	for time.Now().Before(deadline) {
		got, ok, err := sched.Get(context.Background(), id)
		if err != nil {
			t.Fatalf("scheduler.Get: %v", err)
		}
		if ok {
			switch got.Status {
			case domain.TaskDone, domain.TaskFailed, domain.TaskSuspended:
				return got
			}
		}
		time.Sleep(5 * time.Millisecond)
	}
	t.Fatalf("task %s did not reach terminal state in time", id)
	return domain.Task{}
}
```
对每个受影响用例：`_, _, err := coord.Heartbeat(ctx)`（断言 err==nil）→ `got := awaitTerminal(t, sched, taskID)` → 原来的状态断言改断言 `got.Status`。加 `coord.Wait()` 在断言后（或 `t.Cleanup(coord.Wait)`）确保 goroutine 收敛。

- [ ] **Step 5: 跑测试确认通过**

Run: `cd legion/legionAgent && go test ./internal/runtime/ -v 2>&1 | tail -25`
Expected: 全 PASS。

- [ ] **Step 6: 验证关卡 + 提交**

Run: `cd legion/legionAgent && go build ./... && go vet ./internal/runtime/ && gofmt -l internal/runtime/coordinator.go internal/runtime/coordinator_test.go`
Expected: 无输出。
```bash
git add internal/runtime/coordinator.go internal/runtime/coordinator_test.go
git commit -m "feat(runtime): concurrent task dispatch in Coordinator (bounded workers)

Co-Authored-By: Claude Opus 4.8 <noreply@anthropic.com>"
```

---

### Task 4: `-race` 并发压测

**Files:**
- Create: `internal/runtime/coordinator_concurrency_test.go`

**Interfaces:**
- Consumes: `NewCoordinator`（含 `MaxWorkers`）、`Heartbeat`、`Wait`、现有测试里已有的 fake/stub runner（**先读** `coordinator_test.go` 复用其现成的 stub `TaskRunner`/`Scheduler`/`Evaluator`/`Reviewer` 构造方式，别新造）。

- [ ] **Step 1: 写并发压测**

新建 `internal/runtime/coordinator_concurrency_test.go`（用与 `coordinator_test.go` 相同的 stub 装配；下面 `newTestCoordinator` 指代那里已有的构造，若无则按其现有用例内联装配抽出）：
```go
package runtime

import (
	"context"
	"fmt"
	"testing"

	"github.com/stardust/legion-agent/internal/domain"
	"github.com/stardust/legion-agent/internal/task"
)

// TestCoordinatorRunsTasksConcurrently enqueues many tasks and drives the
// coordinator with repeated heartbeats; with bounded worker goroutines all
// tasks must reach a terminal state, and the -race detector must stay silent.
// Run with: go test -race ./internal/runtime/ -run TestCoordinatorRunsTasksConcurrently
func TestCoordinatorRunsTasksConcurrently(t *testing.T) {
	sched := task.NewScheduler()
	coord := newTestCoordinator(t, sched, 4) // MaxWorkers=4; stub runner returns quickly

	const n = 50
	ctx := context.Background()
	for i := 0; i < n; i++ {
		id := fmt.Sprintf("t-%d", i)
		if err := sched.Add(ctx, domain.Task{ID: id, AgentID: "default-agent", Status: domain.TaskPending, Input: "x"}); err != nil {
			t.Fatalf("add %s: %v", id, err)
		}
	}

	// Drive heartbeats until nothing new dispatches, then drain.
	for {
		_, dispatched, err := coord.Heartbeat(ctx)
		if err != nil {
			t.Fatalf("heartbeat: %v", err)
		}
		if !dispatched {
			break
		}
	}
	coord.Wait()
	// One more pass to sweep any tasks freed after workers drained.
	for {
		_, dispatched, err := coord.Heartbeat(ctx)
		if err != nil {
			t.Fatal(err)
		}
		if !dispatched {
			break
		}
	}
	coord.Wait()

	tasks, err := sched.List(ctx)
	if err != nil {
		t.Fatal(err)
	}
	for _, tk := range tasks {
		switch tk.Status {
		case domain.TaskDone, domain.TaskFailed, domain.TaskSuspended:
		default:
			t.Fatalf("task %s not terminal: %s", tk.ID, tk.Status)
		}
	}
}
```
> 若 `coordinator_test.go` 无可复用的 `newTestCoordinator(t, sched, workers)` 工厂，本 Task 第一步先把它从现有用例的内联装配里抽出（stub runner 立即返回一个成功 `domain.TaskRun`，stub evaluator 返回非 hard-loop，stub reviewer Approved）——这是使并发测试可写的必要脚手架，属本 Task 范围。

- [ ] **Step 2: 跑 `-race` 确认无竞争**

Run: `cd legion/legionAgent && go test -race ./internal/runtime/ -run TestCoordinatorRunsTasksConcurrently -v`
Expected: PASS，无 `DATA RACE` 输出。

- [ ] **Step 3: 全包 `-race` 回归**

Run: `cd legion/legionAgent && go test -race ./internal/runtime/ ./internal/task/ 2>&1 | tail -15`
Expected: 全 PASS，无 race。

- [ ] **Step 4: 验证关卡 + 提交**

Run: `cd legion/legionAgent && gofmt -l internal/runtime/`
Expected: 无输出。
```bash
git add internal/runtime/coordinator_concurrency_test.go internal/runtime/coordinator_test.go
git commit -m "test(runtime): -race concurrency test for bounded task dispatch

Co-Authored-By: Claude Opus 4.8 <noreply@anthropic.com>"
```

---

### Task 5: 接线 config + 停机排空

**Files:**
- Modify: `internal/cli/command.go`（Coordinator 构造 + serve 停机路径）

**Interfaces:**
- Consumes: `CoordinatorConfig.MaxWorkers`、`Coordinator.Wait`、`cfg.Runtime.MaxConcurrentTasks`（Task 1）。

- [ ] **Step 1: 传 MaxWorkers**

在 `internal/cli/command.go` 构造 Coordinator 处（`agentruntime.NewCoordinator(agentruntime.CoordinatorConfig{...}`，约 command.go:1807）加一行：
```go
		MaxWorkers: cfg.Runtime.MaxConcurrentTasks,
```

- [ ] **Step 2: 停机排空**

serve 停机/清理路径（`closeStore` / 背景调度停止附近）加 `coordinator.Wait()`，确保进程退出前在飞任务 goroutine 收敛。**先读** command.go 里 serve 的关闭/`defer` 段，把 `coordinator.Wait()` 放在背景调度停止之后、`closeStore()` 之前（避免任务 goroutine 在存储关闭后还写）。

- [ ] **Step 3: 构建 + 全量测试**

Run: `cd legion/legionAgent && go build ./... && go vet ./... && go test ./... 2>&1 | tail -15`
Expected: 全绿。

- [ ] **Step 4: `-race` serve 冒烟（若有 e2e 门控）**

Run: `cd legion/legionAgent && go test -race ./internal/cli/ 2>&1 | tail -10`
Expected: PASS，无 race。

- [ ] **Step 5: gofmt + 提交**

Run: `cd legion/legionAgent && gofmt -l internal/cli/command.go`
Expected: 无输出。
```bash
git add internal/cli/command.go
git commit -m "feat(serve): wire max_concurrent_tasks + drain tasks on shutdown

Co-Authored-By: Claude Opus 4.8 <noreply@anthropic.com>"
```

---

## Self-Review

**1. Spec coverage（对 §4.1a 并发部分）**
- 每任务 goroutine → Task 3 Heartbeat 并发派发。
- 并发上限可配（默认 4）→ Task 1（config）+ Task 3（信号量）+ Task 5（接线）。
- 线程安全审计 → 底层 `Scheduler/LockStore/MemoryEventBus` 已 mutex（已核实）；Task 4 `-race` 佐证；audit 亦经既有 mutex 存储。
- 「一个任务阻塞不影响他人」→ Task 3 语义 + Task 4 压测（50 任务 / 4 worker 全终态）。
- 停机排空 → Task 3 `Wait` + Task 5 接线。
- §4.1b（可挂起工具循环/持久化）**不在本 plan**——是独立的 Milestone 1b plan。

**2. Placeholder scan**：无 TBD；新代码均给出完整实现；测试辅助 `awaitTerminal`/压测给出完整代码。两处「先读现有测试再改」（Task 3 Step4、Task 4 Step1）是**针对已存在文件的定位指令 + 给出改法模板与辅助函数代码**，非占位。

**3. Type consistency**：`MaxWorkers`（CoordinatorConfig）、`MaxConcurrentTasks`（RuntimeConfig）、`sem`/`wg`/`Wait`/`runAssigned` 全程一致；`Heartbeat` 返回签名 `(domain.Task, bool, error)` 保持不变（值语义改为「dispatched」）。

## Execution Handoff

见文末执行选项（本 plan 完成后，Milestone 1b「可挂起工具循环 + 持久化」再单独出 plan）。
