---
title: 会话交接 — Agent 工作模式特性（含工作日志）
type: handoff
created: 2026-07-15
status: active
---

# 会话交接 · Legion Agent 工作模式特性

> 上个会话上下文已满。本文件是**从零接续**的全部上下文：现状、下一步、决策、环境坑、命令。

## 0. 一句话现状

Phase 1（Agent 选择器）已上线；Phase 2（工作模式 Manual/Plan/Auto + 工作目录 + 真并发）的 **spec 已定稿**、**Milestone 1a（并发基础）已实现并合入 master 且 `-race` 验证通过**。**下一步 = 写 Milestone 1b 的实现计划并执行。**

## 1. 三个独立 git 仓库（关键拓扑）

`F:\source\stardust\Legion` 根**不是** git 仓库；三个独立仓库各有 github remote，**分别提交/推送**：

| 仓库 | 路径 | remote | 当前 |
|---|---|---|---|
| 文档 | `docs/` | `github.com/jxncyjq/stardust-agent-docs` | main，spec/plan 已提交 |
| **server** | `legion/legionAgent/` | `github.com/jxncyjq/jxncyjq-stardust-agent-server` | master @ `3bd5127`（M1a 已合入）|
| **UI** | `legion/legionAgentGUI/` | `github.com/jxncyjq/stardust-agent-gui` | master，Phase1+UI 已提交 |

- `legion/.git` 是**遗留聚合仓库勿用**。改代码前 `git rev-parse --show-toplevel` + `git remote -v` 确认落对仓库。
- ⚠️ **密钥待办**：`legionAgent/agent.json` 含实时 deepseek `sk-5a94ba…`，曾进过**旧 remote**（`stardust-agent-server`）历史。**必须去 deepseek 轮换该密钥**；旧仓库删/清理。新 remote（`jxncyjq-stardust-agent-server`）干净、`agent.json`/`agent.db` 已 gitignore。
- **推送被权限分类器拦**——需用户授权或自己 `git push`。

## 2. 已完成（本轮及之前）

**Phase 1 及 UI（legionAgentGUI，已提交）**
- 可视化 Agent 配置设置菜单（SettingsModal + 14 段 schema + configStore）。
- 子 Agent 配置表单（钻入页 AgentConfigPage + 模板创建 + `SaveAll` 分阶段提交、全有或全无 + 路径逃逸防护）。
- Agent 选择器（`ListAgents` + `SubmitTask` 加 agent_id 路由）。
- UI 优化：SVG 图标集、深色模式+WCAG 对比度修复、可拖拽面板、聊天空状态、a11y。
- server 侧：`serve.ValidateConfig/ValidateAgentConfig`、`agentregistry.LoadAgentFile`、`GET /v1/agents`。

**Phase 2 · Milestone 1a（legionAgent master，已合入 + -race 验证）**
- `runtime.max_concurrent_tasks` 配置（默认 4）— `5383737`
- 抽出 `Coordinator.runAssigned`（纯重构）— `b683da5`
- 并发派发：`Coordinator` 加 `sem chan struct{}`+`wg`+`Wait()`+`MaxWorkers`，`Heartbeat` 每 tick 趁空闲槽连续 `Next` 并各起 goroutine 跑 `runAssigned`，立即返回 — `2d19507`
- 50 任务/4 worker 并发压测（`coordinator_concurrency_test.go`）— `92dd503`
- 接线 `command.go`（`MaxWorkers: cfg.Runtime.MaxConcurrentTasks` @ ~1824；`coordinator.Wait()` 停机排空 @ ~1942，在调度停止后、`closeStore()` 前）— `3bd5127`

## 3. 下一步（从这里接续）

**写 Milestone 1b 的实现计划**（用 superpowers:writing-plans），然后 subagent-driven 执行。

**Milestone 1b = 可挂起/可恢复工具循环 + 会话目录持久化 + 重启恢复**（spec §4.1b/§4.0/§4.3）：
- 把 `internal/runtime/runtime.go` 的 `RunTask` 工具循环（约 144-222 行，累积 `toolCtx []toolEntry`/round/token 的同步线性循环）改成**可检查点**：中间态序列化写会话目录 `task-state.json`。
- 遇 Manual 审批（M2 才接完整）时：写检查点 → 任务转 `TaskSuspended`（`domain` 已有；转移表 `internal/task/scheduler.go:108` 已允许 `Running↔Suspended`）→ goroutine 返回释放。
- 决定到达（含 serve 重启后）→ 转 `Running` → 从 `task-state.json` 恢复接着跑。
- **会话目录 resolver**（单一函数）：session 有 working_dir → `<working_dir>/.stardust/session/<id>/`，否则 `<workspace.root>/session/<id>/`；`workspace.root` 默认 `<home>/.stardust`，配错/不存在回退默认 + warn，`~` 展开。

之后：**M2**（模式+审批+SSE）、**M3**（working_dir + GUI 整合 + TUI 整合）各出一份 plan。

## 4. 锁定决策（spec 全文见 docs/superpowers/specs/2026-07-15-agent-working-modes-design.md）

- 执行模型：**真并发 + suspend/resume**（每任务 goroutine；遇审批挂起、检查点落盘、释放 goroutine；决定到达含重启后从盘恢复）。
- 模式 **per-session**，默认 **Auto**；Manual 只 gate **有副作用工具**（读/搜/列自动放行，工具标 `Sensitive` 位）；Plan = `Registry.Subset` 只读工具 + 提示，**产出 OKF markdown 计划**（frontmatter `type: Plan` + title/description/tags/timestamp，写会话目录 `plans/`）。
- gate 点：`runtime.dispatchToolCall`（`internal/runtime/lazytools.go`，唯一 choke）。
- 审批：`internal/approval` 已存在（票据/Decide）；**改为会话目录 JSON 持久化 + Decide 触发任务 `Suspended→Running`**；超时（默认 300s）→ 拒（拒绝结果回模型，不杀任务）。
- 审批传输：**真 SSE `/v1/events`**（现注册但 `serve` 没接 `PlatformEvents`、恒 503，是 latent bug；M2 接活 + 桥接 `domain.RuntimeEvent`→observability）。
- **GUI 与 TUI 对等**：能力在核心（runtime+approval），两前端是适配器。TUI：slash 命令 `/mode`、`/cwd` + 状态栏 + 终端内审批 prompt。
- working_dir per-session、创建任务时校验存在+是目录、`WorkspacePathGuard` 沙箱。

## 5. 环境坑（务必知道）

- **`-race` 在 Windows 跑不了**（无 gcc）。用 **WSL Ubuntu-22.04**（有 gcc 11.4）。go 已装在 WSL `$HOME/sdk/go`（1.26.0）。**必须强制 GOOS=linux**（否则 go 继承 windows 工具链、gcc 报 `-mthreads`）。命令：
  ```bash
  wsl -d Ubuntu-22.04 -- bash -lc 'cd /mnt/f/source/stardust/Legion/legion/legionAgent && GOOS=linux GOARCH=amd64 CGO_ENABLED=1 CC=gcc $HOME/sdk/go/bin/go test -race ./internal/runtime/ ./internal/task/'
  ```
  Windows 侧只跑普通 `go test`（go 1.26.0 在 PATH）。
- **子 agent 最终消息常被 OMC/caveman hook 吞**——让实现者/审查者**把报告写到文件**（如 `legionAgent/.sdd-m1a/task-N-report.md`），控制方读文件核验，别信返回消息。
- 前端测试栈：**仅 vitest（node 环境，无 jsdom/RTL）**——组件靠 `tsc`+build 验证，纯逻辑走 vitest。
- 前后端通信走 **Wails Go 绑定**（非 fetch，避 CORS）；GUI 脱离 Wails runtime 无法预览（`EventsTab` 等 mount 调绑定无 try/catch 会崩）。
- **既有 gofmt 债**：`cmd/gateway`、`internal/gateway/*`、`internal/domain/types.go` 等一堆文件 gofmt 不干净（init 提交带进来的，非本特性触碰）——别顺手全改（无关 churn），可另开清理任务。
- go.mod 要 go 1.26；GUI 独立 module 不能 import server `internal/**`，跨模块经 `legionAgent/serve` 公开桥。

## 6. 关键文件指针

- Spec：`docs/superpowers/specs/2026-07-15-agent-working-modes-design.md`
- M1a plan（已执行）：`docs/superpowers/plans/2026-07-15-agent-modes-m1a-concurrency.md`
- 调研（在 legionAgentGUI/.superpowers/，gitignore）：`research-hermes.md`、`research-evolver.md`、`research-runtime-feasibility.md`
- 核心代码：`internal/runtime/{runtime.go(RunTask 工具循环),coordinator.go(已并发化),lazytools.go(dispatchToolCall gate 点),agent_resolver.go(agentToolRoot)}`、`internal/task/scheduler.go(转移表)`、`internal/approval/service.go`、`internal/server/http.go(路由+events.go SSE)`、`internal/config/config.go`、`internal/tui/interactive.go`

## 7. 工作日志（2026-07-15）

- 收尾 Phase 1 Agent 选择器（server `GET /v1/agents` + GUI picker + `SubmitTask` agent_id）；三仓库提交、处理 agent.json 密钥入库问题（untrack+gitignore，用户重新 init 干净仓库，新 remote）。
- Phase 2：并行调研 hermes-agent / evolver + runtime 可行性 → brainstorm 敲定 spec（真并发/suspend-resume、per-session 模式、会话目录持久化、OKF 计划、SSE、GUI+TUI 对等、workspace.root）。
- writing-plans 拆 M1a plan（5 task）→ subagent-driven 执行 → **-race 在 WSL 验证通过** → 合入 master。
- 建立 WSL go 1.26 + `-race` 验证流程（本机无 gcc 的解法）。

**接续入口**：读本文件 → 读 spec §4.1b/§4.0/§4.3 → `/writing-plans` 出 Milestone 1b plan → subagent-driven 执行（用 §5 的 WSL `-race` 配方验收）。
