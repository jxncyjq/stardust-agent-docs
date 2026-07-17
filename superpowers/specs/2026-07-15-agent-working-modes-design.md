---
title: Agent 工作模式（Manual/Plan/Auto）+ 工作目录 + 真并发执行
type: design-spec
status: draft
created: 2026-07-15
scope: legion/legionAgent（后端 runtime）· legion/legionAgentGUI（GUI）· legionAgent TUI
related:
  - "[[legion-git-repo-topology]]"
  - "[[legion-gui-wails-gotchas]]"
research:
  - legionAgentGUI/.superpowers/research-hermes.md
  - legionAgentGUI/.superpowers/research-evolver.md
  - legionAgentGUI/.superpowers/research-runtime-feasibility.md
tags: [agent, runtime, approval, concurrency, tui, gui, working-mode]
---

# Agent 工作模式 + 工作目录 + 真并发执行（Phase 2）

## 1. 目标

给 Legion Agent 增加三项能力，**GUI 与 TUI 对等**：

1. **工作模式** Manual / Plan / Auto（per-session）：
   - **Manual**：有副作用的工具调用前，暂停等人工批准/拒绝。
   - **Plan**：只用只读工具调研并产出计划，不执行副作用。
   - **Auto**：不拦截（现行为）。
2. **工作目录**（per-session）：用户指定一个目录，Agent 的文件类工具在其中沙箱化操作。
3. **真并发执行**（基础层）：多个会话/任务可同时运行，一个任务等待审批不阻塞其他任务。

调研见 `research-hermes.md`（审批票据/模式两轴/hardline 兜底）、`research-evolver.md`（两阶段审批/cwd resolver）、`research-runtime-feasibility.md`（链路实测/接线量）。

## 2. 锁定决策

| 维度 | 决定 | 依据 |
|---|---|---|
| 执行模型 | **真并发 + suspend/resume**：每任务一 goroutine；遇审批**挂起**（恢复点落盘、释放 goroutine），决定到达（含重启后）从磁盘**恢复**接着跑 | 用户要真并发 + 票据跨重启不重来 → 唯一 coherent 解 |
| 模式作用域 | **per-session**，默认 **Auto**，会话内任务继承 | 用户选 |
| Manual gate 范围 | **只 gate 有副作用工具**；只读（read/search/list）自动放行 | 用户选；hermes 只 gate 危险操作 |
| Plan 语义 | `Registry.Subset` 只留只读工具 + 系统提示「出计划不执行」 | 用户选；`Subset` 现成 |
| working_dir | per-session；创建任务时校验**存在且是目录**；`WorkspacePathGuard` 沙箱 | 用户选 |
| 审批传输 | **真 SSE `/v1/events`**（接 `PlatformEvents` + 桥接事件）；决定走 HTTP POST | 用户选 |
| 审批超时 | 可配（默认 300s）→ **拒**（拒绝结果回给模型，不杀任务） | hermes/evolver 均 timeout→deny |
| 审批持久化 | **落盘会话目录，跨重启存活，随会话删除**（非内存） | 用户要求 |
| 会话目录 | `<working_dir>/.stardust/session/<id>/`；无 working_dir 则 `<workspace.root>/session/<id>/` | 用户指定 |
| workspace.root | agent.json 新增字段；默认 `<home>/.stardust`；配错/不存在→回退默认 + warn；`~` 展开 | 用户指定 |
| Plan 产出格式 | **OKF markdown**（YAML frontmatter `type:Plan` + 正文），除非用户另指定 | 用户指定（[OKF](https://cloud.google.com/blog/products/data-analytics/how-the-open-knowledge-format-can-improve-data-sharing)）|
| TUI 控制面 | slash 命令（`/mode`、`/cwd`）+ 状态栏显示当前模式/目录；审批 = 终端内 prompt | 用户选 |
| 前端对等 | GUI 与 TUI 均为一等消费者，能力在核心 | 用户要求 |

## 3. 架构原则：核心优先，前端是适配器

所有能力落在**后端核心**（`internal/runtime` + `internal/approval` + session 存储），GUI 与 TUI 只是**消费适配器**，共享同一语义：

```
                 ┌──────────────── 后端核心 ────────────────┐
                 │  runtime.dispatchToolCall（唯一 gate 点）  │
                 │  ├─ mode=manual & 敏感工具 → 开票据+检查点  │
                 │  │     任务挂起(Suspended)、释放 goroutine   │
                 │  ├─ mode=plan → Registry.Subset(只读)      │
                 │  └─ mode=auto → 直接执行                    │
                 │  session{mode, working_dir}  agentToolRoot │
                 └───────────▲───────────────────▲───────────┘
          transport-agnostic │                   │ transport-agnostic
        ┌────────────────────┴───┐        ┌──────┴────────────────────┐
        │ GUI 适配器               │        │ TUI 适配器                  │
        │ SSE 推 approval_pending  │        │ 进程内：终端 prompt 批准/拒  │
        │ POST 决定 → resolve      │        │ 直接 resolve 同一票据        │
        │ /mode·/cwd 走 GUI 控件   │        │ /mode·/cwd slash + 状态栏    │
        └──────────────────────────┘        └────────────────────────────┘
```

关键：核心是「工具循环遇审批→挂起+检查点落盘→决定到达→从盘恢复」，`approval.Decide` **不关心决定从 HTTP 还是终端来**——两个前端各自对同一票据落决定、触发同一任务恢复。

## 4. 分层组件设计

### 4.0 会话目录与持久化（贯穿全设计）

引入**会话目录**：per-session 的磁盘状态之家，**审批票据**与 **Plan 计划产出**持久化于此，**跨重启存活，直到会话被手动删除**才清除。

**位置规则**（单一 resolver）：
- 会话**已设 working_dir** → base = `<working_dir>/.stardust`
- 会话**未设 working_dir** → base = **`workspace.root`**（见下）
- 会话目录 = `<base>/session/<sessionID>/`

**`workspace.root` 解析**（agent.json 新增字段，见配置变更）：
- 配置值经 `~` 展开；**配置了且是已存在目录** → 用它。
- 没配 / 配错 / 目录不存在 → 回退 **用户主目录下 `.stardust`**（Windows `%USERPROFILE%\.stardust`；Linux/macOS `$HOME/.stardust`）。
- **Fail-Loud**：当配置了非空 `root` 却无效/不存在而回退时，**记 warn**（`configured workspace.root %q not a dir, falling back to %q`），不静默吞打错的路径。
- `docs_root`/`memory_root` 相对 `workspace.root` 解析。
- `.stardust` 前缀沿用本仓库既有约定（如 skills `install_root` 默认 `.stardust/skills`）。

**配置变更**（`config.WorkspaceConfig` + `agent.json` 的 `workspace`）：
```json
"workspace": {
  "root": "~/.stardust",     // 新增；默认 <home>/.stardust；配错/不存在则回退默认
  "docs_root": "docs",       // 相对 root
  "memory_root": "memory"    // 相对 root
}
```
GUI 设置里 `workspace.root` 可编辑（`docs_root`/`memory_root` 仍只读路径类）。

**目录内容**：
```
<base>/.stardust/session/<sessionID>/
├── approvals/            # 审批票据（每票据一 JSON），resolve/过期后归档或删
│   └── <ticketID>.json
├── task-state.json       # 挂起任务的检查点（工具循环恢复点，见 §4.1b）
└── plans/                # Plan 模式产出（OKF markdown，见 §4.2）
    └── <plan>.md
```

**持久化语义**：
- 审批票据落盘后即使 serve 重启也在：重启后未决票据可被重新加载，对应任务从持久化状态恢复等待（配合里程碑 1 的任务恢复），而非静默丢弃。
- 删除会话（`DELETE /v1/sessions/{id}`）时**连带删除**该会话目录。
- **单一 resolver**（借 hermes `runtime_cwd.py` / evolver `paths.js` 的集中解析教训）：会话目录路径由一个函数解析（working_dir 优先 → 否则用户主目录），别在多处散拼；按 sessionID 隔离，避免并发串写。

> 注：会话**元数据/turns** 当前存 SQLite（`agent.db`）。本设计先把**新增的持久化产物**（票据、计划）放会话目录；是否把会话元数据整体迁到 `.stardust/session`（替换 SQLite）是更大的存储层决定，标记为 planning 阶段待确认项，不在本 spec 强制。

### 4.1 基础层 · 真并发 + 可挂起工具循环（里程碑 1，最高风险，先做先验）

**现状**（可行性报告 §B）：`BackgroundScheduler` 单 goroutine、1s ticker、`Heartbeat` 弹一个任务 `RunTask` **同步 inline** 跑完——全服务一次一个任务；`RunTask` 工具循环（`runtime.go:185-222`）是一个同步栈帧，无中途可持久化状态。

**改造两件事**：

**(a) 并发**
- `Coordinator.Heartbeat` / 调度：弹到任务后 **`go RunTask(...)`** 起独立 goroutine，不再 inline 阻塞；可连续弹多个待处理任务并发跑。
- **并发上限**：可配 worker 上限（默认如 4），防无界 goroutine + LLM 并发打爆配额；超限排队。
- **线程安全审计**：`RunTask` 路径的共享态——`task.Scheduler`、`port.EventBus`、`taskledger.Ledger`、`internal/approval.Service`、audit log——逐个确认并发安全。

**(b) 可挂起/恢复的工具循环**（satisfy 票据跨重启不重来）
- 把 `RunTask` 工具循环的中间态做成**可检查点**：累积的对话/工具上下文（`toolCtx`）、当前轮次、待执行的那个工具调用——序列化为**任务检查点**写会话目录（§4.0）`task-state.json`。
- 遇 Manual 审批（§4.3）时：写检查点 + 开票据 → 任务转 `TaskSuspended`（`domain` 已有该状态，转移表已允许 `Suspended→Running`，可行性报告 §B）→ **goroutine 返回、被释放**（不空转阻塞）。
- 决定到达（HTTP/TUI，**含 serve 重启后**）→ 任务转 `Running` → 调度重新拾取 → 从 `task-state.json` **恢复**工具循环、应用决定（执行工具 / 回拒绝结果）→ 继续。
- 重启恢复：serve 启动扫会话目录，未决票据 + 其任务检查点一起加载，任务留在 `Suspended` 等决定；已决未跑完的转 `Running` 续跑。

- **验收**：`go test -race ./...` 全绿；并发压测（N 任务并发、一个挂起不阻塞他人）；**挂起→（模拟重启：重建 runtime + 从盘加载）→恢复→接着跑出正确结果**的端到端测试。

> 全设计风险最大且最深的一块（尤其 (b) 检查点）。里程碑 1 独立完成 + `-race` + 挂起/恢复 e2e 验收后，再叠上层。

### 4.2 模式层（里程碑 2）

- **session 加 `mode` 字段**（`manual|plan|auto`，默认 `auto`）：`SessionStore` schema + `createSession`/`patchSession` 存取。
- 任务创建时把所属 session 的 `mode` **解析进 `domain.Task`**（新增 `Task.Mode`），使 runtime 无需回查 session。空/无 session 的一次性任务默认 `auto`。
- **Plan**：`AgentRuntimeResolver` / 默认 runtime 构造时，若 `mode=plan` → `tools = tools.Subset("read_file","search_content","list_files", <其它只读>)`，并在系统提示前置「先调研、产出结构化计划、不要执行任何副作用」。
- **Plan 产出格式 = OKF markdown**（除非用户显式指定其他）：计划以 markdown 写出，遵循 **Open Knowledge Format**（[Google Cloud OKF](https://cloud.google.com/blog/products/data-analytics/how-the-open-knowledge-format-can-improve-data-sharing)）规范——即「markdown + YAML frontmatter」：
  - **YAML frontmatter**：必填 `type: Plan`；标配 `title` / `description` / `tags`（数组）/ `timestamp`（ISO 8601）；`resource` 可选（指向相关任务/会话）。
  - **正文**：markdown 标题/列表/表格描述计划步骤；交叉引用用**普通 markdown 链接**（OKF 用 `[x](path.md)` 而非 WikiLink）。
  - 系统提示明确要求模型按此 frontmatter + 正文结构产出。
  - **落盘**：写入会话目录 `plans/<plan>.md`（§4.0），随会话持久化；一个计划一文件（OKF「一概念一文件」）。多次计划可用 `index.md` 汇总（OKF 保留名）。
- **Auto**：不改现行为。

### 4.3 审批层 · Manual（里程碑 2）

- **工具敏感位**：给 `tool.Registry` 的工具项加 `Sensitive bool`（或复用/映射 `domain.ToolCall.RiskLevel`）。read/search/list = 安全；ledger 写（create/claim/update_task…）、`send_message`、`fetch_url`、`delegate_task` = 敏感。
- **gate 点**：`runtime.dispatchToolCall`（`lazytools.go`，唯一 choke，覆盖 lazy + eager 两协议）。伪码（suspend/resume）：
  ```
  if task.Mode == manual && tool.Sensitive {
      if 无已决定(task, thisCall) {
          ticket := approval.Open(sessionID, taskID, toolName, args)   // 落盘
          checkpoint(task, toolCtx, round, pendingCall)                // 落盘会话目录
          publish SSE approval_pending{taskID, ticketID, tool, args}
          transition(task, TaskSuspended)
          return ErrSuspended    // goroutine 干净返回、被释放
      }
      decision := 已决定(task, thisCall)
      if decision == deny { return 拒绝结果给模型 }
  }
  执行工具
  ```
  恢复：决定到达（含重启后）→ 任务转 `Running` → 调度拾取 → 从检查点重进 `RunTask`，走到同一 call 时命中「已决定」分支执行/拒绝。**挂起期间不占 goroutine，别的任务照跑。**
- **`internal/approval` 扩展**：现为纯内存 poll/CRUD，改为**会话目录持久化 + 触发恢复**（§4.0）：
  - 每张票据落盘 `.stardust/session/<sessionID>/approvals/<ticketID>.json`：`session_id` + `task_id` + `tool_call_id` + `tool_name` + `arguments` + `status`(pending|approved|denied) + `created_at`。
  - `Decide`/`DecideBy`：更新盘上票据状态，并**把任务从 `Suspended` 转 `Running`**（触发调度重新拾取、从检查点恢复）。不再需要「阻塞 channel 等待」原语——挂起模型下没有 goroutine 在等。
  - **跨重启**：serve 启动扫各会话目录 `approvals/` + `task-state.json`，未决票据 → 任务留 `Suspended` 等决定；已决未跑完 → 转 `Running` 续跑。删除会话即删其目录。
  - 磁盘 JSON 是真相源，无内存-only 状态。
- **超时**：后台清扫（复用背景调度）对 pending 超过可配时限（默认 300s）的票据 → 标 `denied` + 触发任务恢复走拒绝分支；拒绝作为工具结果回给模型（模型自适应），**不杀任务**（记 warn 事件）。
- **hardline 兜底**（借 hermes）：可选的不可绕过黑名单（如未来出现 `write_file` 删根之类），Auto 也拦。首版可留占位接口。

### 4.4 工作目录层（里程碑 3）

- **session 加 `working_dir`**；任务创建时解析进 `domain.Task.WorkingDir`。
- **校验**（创建任务时，fail-loud）：非空则必须是**已存在的目录**（`os.Stat` + `IsDir`），否则 400。
- `agentToolRoot(rootCfg, agentCfg, task)` 优先返回 `task.WorkingDir`；`WorkspacePathGuard` 照常把工具文件操作锁在其内。
- **覆盖默认 agent**：并发改造后每任务自建 runtime 的路径要保证默认 agent 也经 per-task tool root（把默认执行也走 per-task resolve，或让 coordinator 用 `task.WorkingDir` 重建 tool 注册表）——消除可行性报告 §C 指出的「默认 agent 预建 runtime 吃不到 working_dir」问题。

### 4.5 SSE 层（里程碑 2）

- **接活 `/v1/events`**：可行性报告 §E 指出该 SSE 端点已注册但 `serve` 从没接 `PlatformEvents`（恒 503，死代码）。本设计**把 `PlatformEvents` 接进 `command.go` 的 `serve` 装配**，并**桥接** `domain.RuntimeEvent` / `workflowEvents` → `observability.EventEnvelope`。
- **推送**：`approval_pending{task_id, ticket_id, tool, arguments}`、`approval_resolved{ticket_id, decision}`，以及现有生命周期事件（task_started/tool_executed/task_completed…）。
- 顺带修复该 latent bug。

### 4.6 GUI 适配器（里程碑 3）

- **Go 绑定**：`SetSessionMode(sessionID, mode)`、`SetSessionWorkingDir(sessionID, dir)`（内部 patchSession）、`PickDirectory()`（Wails `runtime.OpenDirectoryDialog`）、`DecideApproval(taskID, ticketID, decision)`（POST）。
- **SSE 桥**：复用现有 `sse_bridge.go`，Go 侧订阅 `/v1/events` → `runtime.EventsEmit` 转成 Wails 事件推前端。
- **前端**：
  - 输入栏**模式选择器**（Manual/Plan/Auto，per-session，切换调 `SetSessionMode`）。
  - `+` 菜单：选文件=图片附件 / **选目录=working_dir**，选中目录以 chip 显示在 `+` 前（调 `PickDirectory` + `SetSessionWorkingDir`）。
  - **审批提示 UI**：收到 `approval_pending` 事件 → 显示工具名 + 参数 + 批准/拒绝按钮 → `DecideApproval`。

### 4.7 TUI 适配器（里程碑 3，与 GUI 并列）

- **slash 命令**：`/mode manual|plan|auto`、`/cwd <path>`（走同一 session mode/working_dir 存取）。
- **状态栏**：常驻显示当前模式 + 工作目录。
- **审批**：进程内消费——任务遇审批挂起后，TUI 从待决票据在终端 prompt「批准/拒绝」，就地 `approval.Decide` 触发同一任务恢复（像 hermes CLI callback），不经 HTTP/SSE。
- `internal/tui/interactive.go` 已有「Plan」标签位可复用为模式指示。

## 5. 数据流

**Manual 一次（suspend/resume）**：
```
发任务(session.mode=manual, working_dir) → 任务 goroutine 跑 RunTask
  → 工具循环遇敏感工具 → 写检查点(task-state.json) + approval.Open(落盘) + SSE approval_pending
  → 任务转 Suspended、goroutine 返回释放  ← 其他任务照跑，本任务不占 goroutine
  ...（可跨越 serve 重启：票据+检查点在盘上）...
  → GUI 弹批准/拒绝（或 TUI 终端 prompt）→ POST 决定 / TUI 就地决定
  → approval.Decide(更新盘) + 任务转 Running → 调度拾取 → 从检查点恢复 RunTask
  → 命中「已决定」：批准=执行工具 / 拒绝=拒绝结果回模型 → 循环续 → 出答案
```

**Plan 一次**：发任务(mode=plan) → runtime 用只读 Subset + plan 提示 → 模型只调研+出计划 → 无副作用 → 用户看计划后手动切 Auto/Manual 执行。

## 6. 数据模型变更

- `Session`：`+ Mode string`（manual|plan|auto，默认 auto）、`+ WorkingDir string`。
- `domain.Task`：`+ Mode string`、`+ WorkingDir string`（创建时从 session 解析）。
- `createTaskRequest`：不直接收 mode/working_dir（从 session 取）；session 的 create/patch 收。
- `tool` 注册项：`+ Sensitive bool`。
- `approval.Ticket`：`+ SessionID`、`+ TaskID`、`+ ToolCallID`、`+ ToolName`、`+ Arguments map[string]string`、`+ Status`、`+ CreatedAt`；**盘上存于会话目录 `approvals/<ticketID>.json`**；`approval.Service` 磁盘持久化后端 + 启动加载；`Decide` 更新盘 + 触发任务 `Suspended→Running`。
- **任务检查点**：`task-state.json`（工具循环恢复点：toolCtx/round/pendingCall），落会话目录；`domain.Task` 复用 `TaskSuspended` 状态。
- **`config.WorkspaceConfig`**：`+ Root string`（`workspace.root`，agent.json 新增；默认 `<home>/.stardust`，配错/不存在回退默认 + warn；`docs_root`/`memory_root` 相对之）。
- **会话目录 resolver**：`sessionDir(session) string` 单一解析（session 有 working_dir → `<working_dir>/.stardust/session/<id>`；否则 `<workspace.root>/session/<id>`，见 §4.0）。
- **Plan 产出**：会话目录 `plans/<plan>.md`，OKF frontmatter（`type: Plan` 必填 + title/description/tags/timestamp）。
- SSE 事件：`approval_pending` / `approval_resolved` envelope。

## 7. API 面

- `POST /v1/tasks/{id}/approvals/{ticketID}` body `{decision: "approve"|"deny"}` → resolve 票据。
- `GET /v1/events`（接活）：SSE，含 `approval_pending`/`approval_resolved` + 生命周期。
- session mode/working_dir 经现有 `POST /v1/sessions` / `PATCH /v1/sessions/{id}`（加字段）。

## 8. 错误处理（Fail-Loud）

- working_dir 非目录/不存在 → 创建任务 400，不静默。
- 审批超时 → deny + warn 事件记录，不静默吞、不杀任务。
- 未知 mode 值 → 拒绝（校验枚举），不默默当 auto。
- 并发共享态竞争 → `-race` 门禁；发现即修，不容忍。
- SSE 桥接失败 → 记录，不静默丢事件。
- 会话目录创建/写入失败（权限/磁盘）→ 返回 error 不静默；票据写盘失败视为审批不可用、fail-loud（宁可报错也不丢审批状态）。
- 加载盘上票据时 JSON 损坏 → 报错并跳过该票据并记录，不静默当无票据。

## 9. 测试

- **并发**（里程碑 1）：`go test -race`；N 任务并发压测断言无 race + 各自结果正确 + 一个卡审批不阻塞他人。
- **审批**：Manual + 敏感工具 → 开票据 + 任务挂起（检查点落盘）；approve→恢复执行、deny→恢复走拒绝回模型、timeout→deny；只读工具不触发审批。
- **Plan**：mode=plan → 工具集只剩只读（断言 Subset），副作用工具不可达。
- **working_dir**：校验非目录报错；工具文件操作被沙箱在 working_dir 内（越界拒）。
- **模式作用域**：session mode 被会话内任务继承；切模式即时生效。
- **会话目录/持久化**：目录位置规则（有/无 working_dir）正确；票据落盘、serve 重启后未决票据被加载、任务恢复等待；删除会话连带删除会话目录；并发按 sessionID 隔离无串写。
- **workspace.root 解析**：配置了且存在→用之；空/不存在→回退 `<home>/.stardust` **并记 warn**；`~` 展开；`docs_root`/`memory_root` 相对 root。
- **Plan/OKF**：Plan 产出含合规 OKF frontmatter（`type: Plan` 必填 + title/description/tags/timestamp）、写入会话目录 `plans/`；只读工具集断言。
- **SSE**：approval_pending 事件推出、字段完整。
- **GUI/TUI**：各自审批消费（GUI POST、TUI 终端）resolve 同一票据；模式/cwd 控件生效。
- 每个 fail-loud 分支有断言。

## 10. 里程碑排序

1. **并发基础**（§4.1）—— 独立完成 + `-race` 验收。**最险，先做。**
2. **模式 + 审批 + SSE**（§4.2/4.3/4.5）—— Manual/Plan/Auto + 审批原语/端点 + 接活 SSE。
3. **working_dir + GUI 整合 + TUI 整合**（§4.4/4.6/4.7）—— 三前端能力对齐。

每个里程碑可独立验收；跨 `legion/legionAgent`（后端）与 `legion/legionAgentGUI`（GUI），TUI 在 legionAgent 内。

## 11. 非目标（YAGNI）

- 不做真正的「计划→逐步执行并逐步确认」编排（Plan 只出计划，不接管执行）。
- 不做多用户/多租户审批路由（单用户桌面）。
- **审批票据持久化到会话目录（§4.0），跨重启存活，直到会话删除**——已纳入设计（非「非目标」）。
- 首版不做把会话元数据/turns 从 SQLite 整体迁到 `.stardust/session`（仅新增产物落该目录；整体迁移是 planning 待确认项）。
- 首版不做 hardline 黑名单的完整规则集（留接口占位）。
- 不做 per-task 覆盖 session 模式（纯 per-session）。

## 12. 风险

- **可挂起工具循环（§4.1b）是最深的一块**：把 `RunTask` 同步栈帧改成「检查点可序列化 + 从检查点重进」，涉及 toolCtx/对话态的可持久化建模——比单纯并发大。列为里程碑 1 核心风险，须 挂起→重启→恢复 e2e 验收。
- **并发线程安全**：runtime/event bus/task store 共享态，`-race` 门禁。
- 并发后事件顺序/审计一致性需复核。
- SSE 桥接两套事件系统（`domain.RuntimeEvent` vs `observability`）有集成成本。
- 检查点序列化格式需版本化（未来改工具循环别让旧盘上检查点解不了——加载失败即 fail-loud 让任务重来，不静默跑错）。
