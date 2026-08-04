---
title: 实施计划 — Milestone 3b（GUI 适配器：模式选择器 + working_dir + SSE 审批 UI）
type: plan
status: active
created: 2026-07-19
scope: legion/legionAgentGUI（独立 Wails 仓库）
related:
  - "[[2026-07-15-agent-working-modes-design]]"
  - "[[m3-explore-gui]]"
tags: [agent, gui, wails, sse, approval, working-mode, milestone-3b, plan]
---

# Agent Working Modes — Milestone 3b (GUI 适配器) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax.

**Goal:** 让 GUI（legionAgentGUI）能设置会话的**工作模式**（Manual/Plan/Auto，per-session）与 **working_dir**，并通过 SSE 实时消费**审批事件**弹出批准/拒绝 UI，把 M2/M3a 后端能力接到桌面前端。

**Architecture:** GUI 一律走 **Go 侧 HTTP** 调后端（规避浏览器 CORS）。新增 4 个 Wails 绑定挂在 `App` 上（`SetSessionMode`/`SetSessionWorkingDir` 薄封装现有 `patchSession`；`PickDirectory` 用 Wails `runtime.OpenDirectoryDialog`；`DecideApproval` 用现有 `postJSON`）。接活现已存在但从未被调用的 `sse_bridge.go`：在 `startup` 起 SSE 长连消费 `/v1/events`，把 `approval_pending`/`approval_resolved` 分流成 Wails 事件推前端；serve 重启换端口时重连。前端（React 18 + TS + Zustand + Tailwind）加三个控件：模式选择器（仿现有 `AgentSelector`）、`+` 菜单选目录 + 目录 chip（仿图片 chip）、审批提示 UI（新 `approvalStore` + `ApprovalPrompt`，以 `GET /v1/approvals` 兜底 SSE 漏事件）。

**Tech Stack:** Wails v2.12.0；Go 1.26（GUI module `legionAgentGUI`，经 `replace github.com/stardust/legion-agent => ../legionAgent` 依赖后端 serve 包，**不得 import `internal/**`**）；React 18 + TypeScript + Zustand + Tailwind v4 + Vite 6 + vitest。后端契约来自 legionAgent master（M2c SSE + M3a working_dir，已合入）。

## Global Constraints

- **绝不 import 后端 `internal/**`**：GUI 只经 HTTP + `serve` 公共装配入口。新绑定继续走 Go 侧 HTTP（`app.client` pooled client / `patchSession` / `postJSON`），**切勿**让前端 fetch 直连随机端口（CORS）。
- **后端契约逐字对齐（M2c 下划线类型）**：
  - SSE 事件类型是**下划线 verbatim**：`approval_pending`、`approval_resolved`（不是点号）。
  - `approval_pending.data` = `{task_id, ticket_id, tool, arguments}`（`arguments` 为 map，仅非 nil 时带；SSE 写边界已递归脱敏+截断，UI 需容忍被截断/缺键）。
  - `approval_resolved.data` = `{task_id, ticket_id, decision}`（decision 值 `"approved"`/`"denied"`）。
  - **审批决定端点值是动词**：`POST /v1/tasks/{taskID}/approvals/{ticketID}` body `{"decision":"approve"|"deny"}`（**`approve`/`deny`，不是 `approved`/`denied`**）；成功 200、票据不存在 404、已决定 409。
  - `SetSessionMode` = `PATCH /v1/sessions/{id}` body `{"mode":"manual|plan|auto"}`（后端就绪，M2a）。
  - `SetSessionWorkingDir` = `PATCH /v1/sessions/{id}` body `{"working_dir":"<abs path>"}`（M3a 就绪）。**working_dir set-once**：后端拒绝把已非空 working_dir 改成不同值（400）——GUI 要处理该 400（提示「working_dir 绑定后不可变」，非静默吞）。
- **Fail-Loud**：新绑定/桥接的 HTTP 错误必须暴露（返回 error / EventsEmit 错误事件 / 写 `serve-startup.log`），不静默吞。Windows GUI build 无 console（stderr 丢弃），错误靠 EventsEmit 或该日志。
- **模式 per-session 非全局**：模式跟随 `currentSession`（切会话刷新显示、切模式 patch 到当前会话），不能照抄 `agentStore` 的全局 `selected` 语义。
- **SSE 重连**：serve 重启换随机端口（`serve_manager.Restart`），SSE 长连的 baseURL 是启动快照——重连要重读 `a.BaseURL()`，别连旧端口。
- **Wails 陷阱**（见 [[legion-gui-wails-gotchas]]）：`main.go` 里**绝不 chdir**（破坏 binding 生成）；chdir 只在 `app.startup` 真实运行时做。新绑定改 Wails 绑定后需重新生成 TS 绑定（`wails dev`/`build` 自动生成 `frontend/wailsjs/go/main/App`）。
- **验证边界（重要）**：自动化门禁 = GUI `go build ./...` + `go vet ./...` + 前端 `npm run build`（`tsc && vite build`）+ `npm test`（vitest，覆盖 store/组件逻辑）。**Wails 桌面 app 的实际视觉/交互验证不在自动化范围**——每任务写可测的 store/纯逻辑单测；整机视觉验证在最后标记为需人工（用户）跑 `wails dev` 确认。
- **仓库**：全部代码在 `legion/legionAgentGUI`（独立 git 仓库，remote 见其 origin）。分支 `feat/agent-modes-m3b`（从 master `7114168` 切）。**不碰 legionAgent 后端**（M3a 已提供全部所需契约）。

## 门禁命令
```bash
# Go 侧（GUI 仓库根）
go build ./... && go vet ./...
# 前端（frontend/ 目录）
cd frontend && npm run build && npm test
```

---

## File Structure

- **Modify** `app.go` — 4 个 Wails 绑定（`SetSessionMode`/`SetSessionWorkingDir`/`PickDirectory`/`DecideApproval`）；`startup` 起 SSE bridge。Task 1/2。
- **Modify** `sse_bridge.go` — approval 事件分流 + serve-restart 重连（重读 BaseURL）。Task 2。
- **Modify** `frontend/src/stores/sessionStore.ts` — `Session` 接口加 `mode`/`workingDir`；setSessions 映射带上。Task 3/4。
- **Create** `frontend/src/components/ModeSelector.tsx` — 模式选择器（仿 AgentSelector）。Task 3。
- **Modify** `frontend/src/components/ChatPanel.tsx` — 挂 ModeSelector；`+` 按钮改弹出菜单（图片/目录）；目录 chip。Task 3/4。
- **Create** `frontend/src/stores/approvalStore.ts` — 待决审批状态（SSE + `/v1/approvals` 兜底）。Task 5。
- **Create** `frontend/src/components/ApprovalPrompt.tsx` — 审批 UI（工具名+参数+批准/拒绝）。Task 5。
- **Modify** `frontend/src/hooks/useAgentEvents.ts` — 订阅新 `agent:approval` 事件路由到 approvalStore。Task 5。
- **Modify** `app.go` — `ListPendingApprovals` 绑定（GET /v1/approvals 兜底）。Task 5。
- 各配套 `*.test.ts(x)`（vitest）。

---

### Task 1: Go 绑定 — SetSessionMode / SetSessionWorkingDir / PickDirectory / DecideApproval

**Files:**
- Modify: `app.go`（App 上加 4 个 exported 方法）
- Test: `app_test.go`（若已有 Go 测试则追加；否则以 httptest 起假后端验证 patch/post body。若 GUI 无 Go 测试基建则以 build/vet + 前端集成覆盖，实现者报告说明）

**Interfaces:**
- Consumes: 现有 `App.patchSession(id string, patch map[string]any) error`（app.go:313，PATCH /v1/sessions/{id}，fail-loud）；`App.postJSON(path string, body any) (…)`（app.go:470，POST + JSON + drain）；Wails `runtime.OpenDirectoryDialog`（`runtime` 已在 app.go:14 import）；`a.ctx`（startup 存的 context，app.go:45）。
- Produces:
  - `App.SetSessionMode(sessionID, mode string) error`
  - `App.SetSessionWorkingDir(sessionID, dir string) error`
  - `App.PickDirectory() (string, error)`
  - `App.DecideApproval(taskID, ticketID, decision string) error`

- [ ] **Step 1: 实现 SetSessionMode / SetSessionWorkingDir（薄封装 patchSession）**

```go
// SetSessionMode sets a session's working mode (manual|plan|auto) via PATCH.
// It is a thin wrapper over patchSession, mirroring RenameSession /
// SetSessionArchived. Mode validation is the server's responsibility (400 on an
// unknown value), surfaced here as the returned error.
func (a *App) SetSessionMode(sessionID, mode string) error {
	return a.patchSession(sessionID, map[string]any{"mode": mode})
}

// SetSessionWorkingDir binds a session's working directory via PATCH. The server
// treats working_dir as set-once: changing an already-set working_dir to a
// different value returns 400, surfaced here as an error the UI must show
// (working_dir cannot be changed once bound) rather than swallow.
func (a *App) SetSessionWorkingDir(sessionID, dir string) error {
	return a.patchSession(sessionID, map[string]any{"working_dir": dir})
}
```

> 先读 `patchSession`（app.go:313）确认签名（`patch map[string]any` 或别的）与 400 是否已包成 error——对齐实际签名，勿臆造。

- [ ] **Step 2: 实现 PickDirectory（Wails 目录对话框）**

```go
// PickDirectory opens the native directory picker and returns the chosen
// absolute path (empty string if the user cancels — a legitimate outcome, not an
// error). The frontend pairs this with SetSessionWorkingDir.
func (a *App) PickDirectory() (string, error) {
	return runtime.OpenDirectoryDialog(a.ctx, runtime.OpenDialogOptions{
		Title: "选择工作目录",
	})
}
```

> 确认 `runtime` import 别名与 `a.ctx` 字段名（app.go:14/45）。用户取消返回空串 err=nil 是合法态（非 fail-loud 违规）。

- [ ] **Step 3: 实现 DecideApproval（POST 决定端点，校验状态码）**

```go
// DecideApproval posts a human's approve/deny decision on a Manual-mode tool
// approval ticket. decision must be "approve" or "deny" (the verb form the
// server's endpoint expects — NOT the "approved"/"denied" past tense used in the
// approval_resolved SSE event). A non-2xx response fails loud: 404 = ticket gone,
// 409 = already decided, other = surfaced verbatim.
func (a *App) DecideApproval(taskID, ticketID, decision string) error {
	// path 用 postJSON;若 postJSON 不返回 status，改用 a.client 直接发并判 resp.StatusCode。
	return a.postJSON("/v1/tasks/"+taskID+"/approvals/"+ticketID, map[string]any{"decision": decision})
}
```

> **关键**：先读 `postJSON`（app.go:470）看它是否已对非 2xx fail-loud。若 `postJSON` 吞了状态码，则 `DecideApproval` 改为用 `a.client` 直发并显式判 200/404/409（仿 `SkillCommand` app.go:553 的状态判定）。实现者按实际 `postJSON` 行为定，报告说明。

- [ ] **Step 4: 构建 + 生成绑定 + 测试**

Run: `go build ./... && go vet ./...`
Expected: 通过（4 个方法编译）。若 GUI 有 Go 测试基建，加 httptest 假后端断言 4 个绑定发出的 method/path/body 正确（SetSessionMode→PATCH mode、DecideApproval→POST decision=approve）。若无 Go 测试基建，报告说明并靠前端集成 + 后端契约测试覆盖。

- [ ] **Step 5: 提交**

```bash
git add app.go app_test.go
git commit -m "feat(gui): 加 SetSessionMode/SetSessionWorkingDir/PickDirectory/DecideApproval 绑定"
```

---

### Task 2: 接活 SSE bridge + 审批事件分流 + serve 重启重连

**Files:**
- Modify: `sse_bridge.go`（approval 分流 + baseURL 重读）
- Modify: `app.go`（startup 起 StartSSEBridge）

**Interfaces:**
- Consumes: `StartSSEBridge(ctx, appCtx, baseURL)`（sse_bridge.go:15，现死代码）；`consumeSSE`（sse_bridge.go:34，手写 SSE 解析 + `runtime.EventsEmit(appCtx, "agent:event", …)`）；`a.BaseURL()`（app.go:96）；`serve:status` 事件（serve_manager.go）。
- Produces: startup 后 SSE 长连消费 `/v1/events`；`approval_pending`/`approval_resolved` → Wails 事件 `agent:approval`；serve 重启后重连新端口。

- [ ] **Step 1: sse_bridge 加 approval 事件分流**

在 `consumeSSE`（sse_bridge.go:54-64，token 分流范式旁）加：对 `event.type == "approval_pending" || event.type == "approval_resolved"` 额外 `runtime.EventsEmit(appCtx, "agent:approval", map[string]any{"type": eventType, "data": data})`（保留原 `agent:event` 泛化 emit 不破）。

- [ ] **Step 2: startup 起 SSE bridge（serve running 后）+ 重连**

`app.startup`（app.go:44-66，serve.Start 成功后）起 SSE bridge。因 serve 随机端口异步起 + 可重启换端口：**baseURL 每次（重）连时重读 `a.BaseURL()`**，别用启动快照。最小实现：改 `StartSSEBridge`/`consumeSSE` 的重试循环里每次拨号前 `baseURL = a.BaseURL()`（或传一个 `baseURLFn func() string`）；并在收到 `serve:status` 变化时触发重连。实现者选最小可靠方案（`baseURLFn` 闭包最简），报告说明。

> `StartSSEBridge` 现失败静默重试（sse_bridge.go:23-28，注释 retry silently）——对未就绪 serve 是有意的，但**重试要记可见日志**（至少首次失败 / 连接建立 EventsEmit 一个 `serve:sse` 状态），别全静默（fail-loud：SSE 是 approval UI 数据源，长期连不上要能诊断）。

- [ ] **Step 3: 构建 + 测试**

Run: `go build ./... && go vet ./...`
若有 Go 测试基建：用 httptest SSE server 发 `event: approval_pending\ndata: {...}\n\n`，断言 `consumeSSE` 对其 EventsEmit `agent:approval`（可注入一个假 emit 收集器；若 `runtime.EventsEmit` 难 mock，抽一个 `emit func(ctx,name,data)` 参数便于测试）。报告说明所选测试策略。

- [ ] **Step 4: 提交**

```bash
git add sse_bridge.go app.go
git commit -m "feat(gui): 接活 SSE bridge，分流 approval 事件 + serve 重启重连"
```

---

### Task 3: 前端 — 会话模式选择器（per-session）

**Files:**
- Modify: `frontend/src/stores/sessionStore.ts`（`Session` 加 `mode`）
- Create: `frontend/src/components/ModeSelector.tsx`
- Modify: `frontend/src/components/ChatPanel.tsx`（工具行挂 ModeSelector）
- Test: `frontend/src/components/ModeSelector.test.tsx`、`sessionStore` 相关测试

**Interfaces:**
- Consumes: Wails 绑定 `SetSessionMode`（`frontend/wailsjs/go/main/App`，Task 1 生成）；`useSessionStore`（currentSession、setSessions）。
- Produces: `<ModeSelector/>` 控件；`Session.mode: string`。

- [ ] **Step 1: sessionStore.Session 加 mode + 映射**

`sessionStore.ts`（:5-12 `Session` 接口）加 `mode?: string`；`setSessions`/session 加载映射时带上后端返回的 `mode`（后端 `AgentSession.Mode` 已有）。加/更新 vitest 断言映射带 mode。

- [ ] **Step 2: 写 ModeSelector 失败测试**

`ModeSelector.test.tsx`：渲染 → 显示当前 session mode（默认 auto）→ change 到 manual → 断言调用 `SetSessionMode(currentSessionId, "manual")` 且更新 store。仿 AgentSelector 测试（若无则新建）。mock `wailsjs/go/main/App` 的 `SetSessionMode`。

- [ ] **Step 3: 实现 ModeSelector（仿 AgentSelector.tsx）**

`<select>` 三选项 Manual/Plan/Auto，值 = `currentSession?.mode ?? "auto"`；onChange → `await SetSessionMode(currentSessionId, value)` 成功后更新 sessionStore 里该 session 的 mode（**per-session**，非全局）。错误（如后端 400）toast/系统消息提示，不静默。挂到 `ChatPanel.tsx:616-628` 工具行（AgentSelector 旁）。

- [ ] **Step 4: 构建 + 测试**

Run: `cd frontend && npm test && npm run build`
Expected: ModeSelector 测试绿；tsc + vite build 过。

- [ ] **Step 5: 提交**

```bash
git add frontend/src/components/ModeSelector.tsx frontend/src/components/ModeSelector.test.tsx frontend/src/stores/sessionStore.ts frontend/src/components/ChatPanel.tsx
git commit -m "feat(gui): 输入栏会话模式选择器(Manual/Plan/Auto per-session)"
```

---

### Task 4: 前端 — `+` 菜单选目录 + 目录 chip

**Files:**
- Modify: `frontend/src/components/ChatPanel.tsx`（`+` 单按钮→弹出菜单；目录 chip）
- Modify: `frontend/src/stores/sessionStore.ts`（`Session` 加 `workingDir`）
- Test: `frontend/src/components/ChatPanel`（目录选择流程）相关测试

**Interfaces:**
- Consumes: Wails 绑定 `PickDirectory`、`SetSessionWorkingDir`（Task 1）；现有图片附件机制（`onPickImages` ChatPanel.tsx:128、图片 chip 渲染 :520-541）。
- Produces: `+` 弹出菜单（图片/目录两项）；选目录 → `PickDirectory` → `SetSessionWorkingDir` → 目录 chip 显示在 `+` 前；`Session.workingDir`。

- [ ] **Step 1: sessionStore.Session 加 workingDir + 映射**

`sessionStore.ts` `Session` 加 `workingDir?: string`；映射带后端 `working_dir`。

- [ ] **Step 2: 写目录选择流程失败测试**

测试：点 `+` → 菜单出现（图片/目录）→ 点目录 → `PickDirectory` 返回路径 → `SetSessionWorkingDir(sid, path)` 被调 → chip 显示该路径；PickDirectory 返回空串（取消）→ 不调 SetSessionWorkingDir、无 chip。mock 两个绑定。

- [ ] **Step 3: 实现 `+` 弹出菜单 + 目录 chip**

- 把 ChatPanel.tsx:604-626 的单 `+` 按钮（现直接触发图片 fileInput）改为弹出小菜单两项：「图片」（走现有 `fileInputRef.click`）/「工作目录」（`const dir = await PickDirectory(); if (dir) { await SetSessionWorkingDir(currentSessionId, dir); 更新 store }`）。
- 目录 chip 仿图片 chip（:520-541）渲染在 `+` 前，显示目录名（可截断），带移除按钮（移除 = `SetSessionWorkingDir(sid, "")`？**注意 set-once**：后端拒绝非空→改值；移除即改成空——若后端也拒空→GUI 应禁用移除或提示。实现者按后端 set-once 语义定 chip 是否可移除，报告说明；保守：chip 只读展示 + 提示「绑定后不可变」）。
- SetSessionWorkingDir 返回 400（set-once 违反）→ 系统消息提示，不静默。

- [ ] **Step 4: 构建 + 测试**

Run: `cd frontend && npm test && npm run build`

- [ ] **Step 5: 提交**

```bash
git add frontend/src/components/ChatPanel.tsx frontend/src/stores/sessionStore.ts frontend/src/components/*.test.tsx
git commit -m "feat(gui): + 菜单选工作目录 + 目录 chip"
```

---

### Task 5: 前端 — 审批提示 UI（SSE + /v1/approvals 兜底）

**Files:**
- Create: `frontend/src/stores/approvalStore.ts`
- Create: `frontend/src/components/ApprovalPrompt.tsx`
- Modify: `frontend/src/hooks/useAgentEvents.ts`（订阅 `agent:approval`）
- Modify: `app.go`（加 `ListPendingApprovals` 绑定）+ `frontend/src/components/ChatPanel.tsx`（挂 ApprovalPrompt）
- Test: `approvalStore.test.ts`、`ApprovalPrompt.test.tsx`

**Interfaces:**
- Consumes: Wails 事件 `agent:approval`（Task 2）；绑定 `DecideApproval`（Task 1）；新绑定 `ListPendingApprovals`（GET /v1/approvals）；`EventsOn`/`EventsOff`（useAgentEvents 范式 :41-43）。
- Produces: `approvalStore`（pending 票据集合 + add/resolve/load）；`<ApprovalPrompt/>`；`App.ListPendingApprovals() ([]…, error)`。

- [ ] **Step 1: app.go 加 ListPendingApprovals 兜底绑定**

```go
// ListPendingApprovals returns the pending Manual-mode approval tickets the
// server has on disk, so the UI can reconcile any approval_pending events it
// missed over the at-most-once SSE stream (or before the frontend subscribed).
func (a *App) ListPendingApprovals() ([]map[string]any, error) {
	// GET /v1/approvals?status=pending via a.apiGet;返回 body 的 approvals 数组。
}
```
> 用现有 `apiGet`（app.go:113）；对齐其返回形状。后端 `GET /v1/approvals?status=pending` 返回 `{"approvals":[{ticket_id,task_id,tool_name,arguments,…}]}`（M2c Task 5）。

- [ ] **Step 2: 写 approvalStore 失败测试**

`approvalStore.test.ts`：`onApprovalPending({task_id,ticket_id,tool,arguments})` → pending 里有该票据；`onApprovalResolved({ticket_id,decision})` → 移除；`load()`（调 ListPendingApprovals）→ 填充 pending，与 SSE 去重（同 ticket_id 不重复）。

- [ ] **Step 3: 实现 approvalStore + useAgentEvents 路由**

- `approvalStore.ts`（zustand）：`pending: ApprovalTicket[]`、`onPending`、`onResolved`（按 ticket_id 移除）、`load`（ListPendingApprovals + 去重 merge）。
- `useAgentEvents.ts` 加 `EventsOn('agent:approval', e => e.type==='approval_pending' ? store.onPending(e.data) : store.onResolved(e.data))`，`EventsOff` 清理（仿 :64-69）。挂载时 `store.load()` 兜底拉未决（防订阅前/漏事件）。

- [ ] **Step 4: 写 ApprovalPrompt 失败测试**

`ApprovalPrompt.test.tsx`：pending 有票据 → 渲染工具名 + 参数（arguments 键值，容忍截断/缺键）+ 批准/拒绝按钮；点批准 → `DecideApproval(taskId, ticketId, "approve")`；点拒绝 → `"deny"`（**动词，非 approved/denied**）。resolve 后从列表消失。

- [ ] **Step 5: 实现 ApprovalPrompt + 挂载**

渲染 `approvalStore.pending` 每张票据：工具名 + arguments（键值列表，值可能被后端截断，显示「…」容忍）+ 批准/拒绝。按钮 → `await DecideApproval(taskId, ticketId, "approve"|"deny")`；成功后本地 `onResolved`（也会经 SSE approval_resolved 到，去重）。挂到 ChatPanel 合适位置（仿系统消息 `addSystem` :90-92 或独立浮层）。DecideApproval 404/409 → 提示（票据已失效/已决），不静默。

- [ ] **Step 6: 构建 + 测试**

Run: `cd frontend && npm test && npm run build` + GUI 根 `go build ./... && go vet ./...`

- [ ] **Step 7: 提交**

```bash
git add frontend/src/stores/approvalStore.ts frontend/src/components/ApprovalPrompt.tsx frontend/src/hooks/useAgentEvents.ts frontend/src/components/ChatPanel.tsx app.go frontend/src/**/*.test.* 
git commit -m "feat(gui): SSE 审批提示 UI + /v1/approvals 对账兜底"
```

---

### Task 6: 全链路整合 + 门禁 + 人工验证清单

**Files:**
- 视需要小修（整合各控件、样式）

- [ ] **Step 1: 全量门禁**

Run:
```bash
go build ./... && go vet ./...
cd frontend && npm test && npm run build
```
Expected: 全绿。若 FAIL 先诊断修复。

- [ ] **Step 2: 生成/校验 Wails 绑定**

确认新 Go 绑定已生成到 `frontend/wailsjs/go/main/App`（`wails dev` 或 build 触发）。若绑定 TS 声明缺失导致前端 tsc 失败，跑一次 `wails generate module`（或 `wails dev` 短暂启动生成）并把生成物纳入提交（若仓库跟踪 wailsjs 生成文件）。实现者按仓库现状（wailsjs 是否 gitignore）处理，报告说明。

- [ ] **Step 3: 人工验证清单（写入报告，供用户跑 `wails dev` 逐条核对）**

自动化覆盖不到桌面 app 视觉/交互，列清单供人工：
1. 输入栏模式选择器：切 Manual/Plan/Auto → 切会话保持 per-session。
2. `+` 菜单选目录 → chip 显示 → 新任务工具操作沙箱在该目录内。
3. Manual 模式发敏感任务 → 审批 UI 弹出（工具名+参数）→ 批准 → 任务继续；拒绝 → 任务走拒绝分支。
4. serve 重启（若可触发）→ SSE 重连 → 审批仍能收到。
5. working_dir 已设再改 → 提示不可变（set-once 400）。

- [ ] **Step 4: 提交**

```bash
git add -A
git commit -m "chore(gui): M3b 整合 + 人工验证清单"
```

---

## Self-Review

**Spec coverage（§4.6）:**
- Go 绑定 SetSessionMode/SetSessionWorkingDir/PickDirectory/DecideApproval → Task 1 ✅
- SSE 桥 Go 侧订阅 /v1/events → EventsEmit 转 Wails 事件 → Task 2 ✅
- 输入栏模式选择器（per-session）→ Task 3 ✅
- `+` 菜单选目录=working_dir + chip → Task 4 ✅
- 审批提示 UI（approval_pending → 工具名+参数+批准/拒绝 → DecideApproval）→ Task 5 ✅
- SSE 漏事件兜底（/v1/approvals）→ Task 5 ✅（超出 §4.6 但 explore §9.5 指出的可靠性必需）

**契约一致性:** decision 动词 `approve`/`deny`（Task 1/5，区别于 approval_resolved 的 `approved`/`denied`）；SSE 类型下划线 `approval_pending`/`approval_resolved`（Task 2/5）；working_dir set-once 400 处理（Task 1/4）；模式 per-session 非全局（Task 3）；SSE 重连重读 BaseURL（Task 2）。

**Placeholder scan:** Task 1/2/5 的 Go 部分给了实现骨架 + 明确「先读 patchSession/postJSON/apiGet 实际签名对齐」；前端组件仿现有 AgentSelector/图片 chip/useAgentEvents 范式，标注可仿先例 file:line——非占位，是对既有代码的诚实引用。

**验证边界:** GUI 桌面 app 视觉验证不在自动化内 → 每任务写 vitest store/组件逻辑单测 + Go build/vet，整机视觉验证列人工清单（Task 6）。这是 GUI 任务的固有限制，已在 Global Constraints + Task 6 明确。

**风险提示（给执行者）:** Task 2（SSE 桥接 + 重连）是最易出错处（serve 随机端口/重启/静默重试），用心测。前端任务先读 AgentSelector.tsx / ChatPanel.tsx 图片 chip / useAgentEvents.ts 现有范式再仿写。Wails 绑定改动后必须重新生成 TS 绑定否则前端 tsc 失败。
