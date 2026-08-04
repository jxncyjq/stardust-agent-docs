---
id: "plans-agent-wails-gui-design-001"
title: "Agent GUI 设计文档 — Wails Desktop Application"
aliases: ["wails gui 设计", "legionAgentGUI 设计"]
type: "design"
category: "plans/agent"
tags: ["design", "agent", "gui", "wails", "react", "desktop", "tailwind"]
version: "1.0.0"
created: "2026-06-24"
updated: "2026-06-24"
author: "jxncyjq"
status: "published"
parent: null
children: []
related_docs:
  - id: "plans-agent-wails-gui-impl-001"
    relation: "implemented_by"
    path: "./2026-06-24-wails-gui-plan.md"
  - id: "reference-worklog-20260624-001"
    relation: "references"
    path: "../../memory/2026-06-24-remaining-work-items.md"
  - id: "agent-http-api-001"
    relation: "depends_on"
    path: "../../agents/legion-agent/http-api.md"
---

# Agent GUI 设计文档 — Wails Desktop Application

**状态：** 已实现（首版落地，单进程 Windows 桌面应用）
**目标读者：** 实现工程师、架构评审

> 本文已根据实际落地结果更新（v1.0）。原 v0.1 草案中若干假设在实现时被推翻，关键差异集中在 [§6 关键实现决策与偏离](#6-关键实现决策与偏离)。

<!-- @section: architecture -->
## §1 架构

### 并行存在策略

`legionAgentGUI` 是**独立 Go module / 独立可执行文件**（`legionAgentGUI.exe`），与现有 `agent serve` / `agent tui` 并存，用户按需选用。

> 修正（v0.1→v1.0）：原设计设想新增 `agent gui` 子命令，落地时改为**独立二进制**——GUI 通过 `go.work` 工作区引用 legion-agent，而非作为其 CLI 子命令，避免把 Wails/WebView2 依赖引入 CLI。

### 目录结构

```text
legion/
├── go.work                     # 工作区：legionAgent + legionAgentGUI
├── run.ps1                     # 构建/启动脚本（dev/build/run/serve）
├── legionAgent/                # 现有 CLI + TUI（module: github.com/stardust/legion-agent）
│   ├── serve/serve.go          # 公共包装包（GUI 复用入口，§6.1）
│   └── internal/cli/command.go # BuildServeService / ServeOptions / ServeResult
└── legionAgentGUI/             # 新建 Wails 桌面应用（module: legionAgentGUI）
    ├── main.go                 # Wails 入口（cfgPath 透传）
    ├── app.go                  # App 绑定层（Wails 生命周期 + 绑定方法）
    ├── serve_manager.go        # 进程内 serve 生命周期 + 端口
    ├── sse_bridge.go           # SSE → Wails 事件桥接
    └── frontend/               # React 前端（见 §7）
```

### serve_manager.go — 进程内模式

`ServeManager` 通过**公共 `serve` 包**在进程内启动 HTTP 服务（随机端口），与 Wails 应用同进程；窗口关闭即停止：

```go
// serve_manager.go（实际实现）
import "github.com/stardust/legion-agent/serve"

func (m *ServeManager) Start(appCtx context.Context, configPath string) error {
    ctx, cancel := context.WithCancel(context.Background())
    m.cancel = cancel
    result, err := serve.BuildService(ctx, serve.Options{ConfigPath: configPath, Addr: "127.0.0.1:0"})
    if err != nil { cancel(); return err }
    m.port = listenerPort(result.Listener)            // listener 绑定后取真实端口
    runtime.EventsEmit(appCtx, "serve:status", map[string]any{"running": true, "port": m.port})
    go func() { defer result.Close(); _ = result.Service.Start(ctx) }()
    return nil
}
```

> 修正（v0.1→v1.0）：原设计 `import legionAgent/internal/service` 并调用 `service.NewServer`。**Go 禁止跨 module 导入 `internal/`**，且包路径/API 与设想不同。改为复用 `internal/cli.BuildServeService`，并以公共 `serve` 包对外暴露（详见 §6.1）。

<!-- @end-section -->

<!-- @section: ui-layout -->
## §2 UI 布局

三栏水平布局，顶部一条连接状态栏：

```text
┌───────────────────────── ConnectionBadge（连接中 :port / 未连接）┐
├─────────────┬──────────────────────┬─────────────────┤
│   Sidebar   │       Chat           │   Status Panel  │
│  Session 列表│  消息历史 + 流式     │  事件/任务/      │
│  新建 Session│  输入框（Enter 发送）│  Audit/Inbox Tab │
└─────────────┴──────────────────────┴─────────────────┘
```

- **Sidebar**（左，w-56，可折叠至 w-12）：Session 列表、新建 Session
- **Chat**（中，弹性）：消息历史滚动区 + 底部输入框，Markdown 渲染 + 用户/助手气泡 + 复制按钮
- **Status Panel**（右，w-72，可折叠）：事件 / 任务 / Audit / Inbox 四个 Tab
- **布局缩放**：根容器 `h-full` 随窗口缩放；折叠状态持久化到 `localStorage`
- **配色**：Claude Desktop 风格——暖炭灰背景 + coral（赤陶橙）强调（§6.5）

<!-- @end-section -->

<!-- @section: data-flow -->
## §3 数据流

### 请求响应（HTTP）

```text
React → fetch(BaseURL() + path) → 进程内 legion-agent HTTP API → JSON
```

`BaseURL()` 由 Wails 绑定返回 `http://127.0.0.1:<随机端口>`。覆盖：发送消息（`POST /api/v1/tasks`）、session 列表（`ListSessions` 绑定）、新建 session（`NewSession` 绑定）、任务轮询（`GET /api/v1/tasks`）。

### 流式输出（SSE → Wails Events）

```text
serve /api/v1/events SSE
  → Go goroutine (sse_bridge.go) 解析 event/data
  → runtime.EventsEmit("agent:event" | "agent:token", payload)
  → React EventsOn(...) (hooks/useAgentEvents.ts)
  → Zustand chatStore 追加 token → ChatPanel 渲染
```

### 服务状态事件

```text
ServeManager.Start/Stop → EventsEmit("serve:status", {running, port})
  → serveStore → ConnectionBadge 显示绿/红
```

<!-- @end-section -->

<!-- @section: tech-stack -->
## §4 技术栈（落地实际版本）

| 层 | 选型 | 落地版本 |
|----|------|----------|
| 桌面框架 | Wails v2 | v2.12.0 |
| 前端框架 | React + TypeScript | React 18 / TS 5 |
| 构建工具 | Vite | **v6**（脚手架自带 v3 已升级，§6.3） |
| 样式 | Tailwind CSS v4 + 主题 token | tailwindcss 4.3 + `@tailwindcss/vite` |
| 组件风格 | 手动 shadcn 风格 token | `cn()` + CSS 变量（§6.4） |
| Markdown | react-markdown + shiki | react-markdown 10（§6.7） |
| 状态管理 | Zustand | 5.x |
| HTTP | 原生 fetch | — |
| SSE | Go 侧消费 + `EventsEmit` 转发 | — |

### TypeScript 自动绑定

Wails 从 `App` struct 生成 TS bindings，React 直接调用：

```typescript
import { BaseURL, ListSessions, NewSession } from '../wailsjs/go/main/App'
const sessions = await ListSessions()
```

> 修正（v0.1→v1.0）：原设计示例绑定 `GetConfig`/`ListSessions`；实际暴露 `Port` / `BaseURL` / `ListSessions` / `NewSession`。

<!-- @end-section -->

<!-- @section: implementation-decisions -->
## §6 关键实现决策与偏离

> 落地相对 v0.1 草案的关键技术决策，供维护/移植参考。完整过程见 [[reference-worklog-20260624-001]]。

### 6.1 serve 公共包装包（绕过 Go internal 限制）

GUI（独立 module）需调用 serve 装配逻辑，但它位于 `legionAgent/internal/cli`，**Go 禁止跨 module 导入 `internal/`**。解决：在 legion-agent 内新增公共包 `github.com/stardust/legion-agent/serve`，re-export `internal/cli.BuildServeService`（同模块可合法导入自身 internal）。

### 6.2 go.work + replace

两个独立 module 用 `go.work` 联编，但 go.work 单独**不足以离线解析未发布模块**，补 `replace github.com/stardust/legion-agent => ../legionAgent`。

### 6.3 工具链升级（Vite 3 → 6）

Wails `react-ts` 脚手架自带 vite 3，而 `@tailwindcss/vite@4` peer 要求 vite ≥ 5；非显式安装时 npm 因 ERESOLVE **静默跳过** tailwind 包。解决：升级 vite 6 + plugin-react 4 + typescript 5。

### 6.4 手动主题（替代 shadcn init）

`shadcn init` 交互式、非交互 shell 挂起。改为手动 `cn()`（clsx + tailwind-merge）+ `style.css` 内 `@theme inline` + `:root`/`.dark` token，`<html class="dark">` 默认暗色。

### 6.5 Claude 暖色主题

配色采用 Claude Desktop 风格：暖炭灰背景 + coral 强调（替换 zinc）。主色与聚焦环为 coral；连接 Badge 绿/红为语义色，不随主题变。

### 6.6 三栏布局与缩放

根容器 `h-full`（非 `h-screen`），避免顶部状态栏叠加导致整页溢出，随窗口缩放；两侧 `aside` 为 `flex flex-col` + 单一 `flex-1 min-h-0 overflow` 滚动链。

### 6.7 react-markdown v10

v10 移除 `className` prop，助手消息改用 wrapper `div` 承载 `prose` 类。

<!-- @end-section -->

<!-- @section: frontend-layout -->
## §7 前端模块

```text
frontend/src/
├── App.tsx                 # 顶栏 Badge + ThreePanelLayout
├── components/             # ThreePanelLayout / Sidebar / ChatPanel / StatusPanel / MessageBubble / ConnectionBadge
├── components/status/      # EventsTab / TasksTab / AuditTab / InboxTab
├── hooks/useAgentEvents.ts # 订阅 agent:token / agent:event / serve:status
├── stores/                 # Zustand: serveStore / chatStore / sessionStore
├── lib/utils.ts            # cn()
└── style.css               # Tailwind v4 + Claude 主题 token
```

<!-- @end-section -->

<!-- @section: scope -->
## §8 功能范围与现状

### 已实现

- 发送 prompt、流式展示回复（依赖后端模型；默认 `DemoResponse` 兜底）
- Session 列表、新建、切换
- Status Panel：事件流、任务列表（每 3s 轮询）
- Markdown 富文本渲染 + 消息复制按钮
- 布局折叠偏好持久化（localStorage）+ 连接状态 Badge

### 占位 / 待办

- Status Panel 的 **Audit / Inbox Tab** 为占位，待接真实 API
- Agent 切换（`@agent` 路由）、Skill 安装/更新/卸载 UI
- 端到端流式回复需配置可用 MaaS profile
- macOS / Linux 构建（当前仅 Windows）

### 超出范围（不做）

- 多窗口 / 多 Tab Agent 并排、内嵌浏览器工具预览、插件市场 UI

<!-- @end-section -->

## 相关文档

- [[plans-agent-wails-gui-impl-001|Wails GUI 实现计划]] — 逐 Task 实现步骤
- [[reference-worklog-20260624-001|2026-06-24 工作日志]] — 落地过程、偏离与验证记录
- [[agent-http-api-001|legion-agent HTTP API]] — GUI 调用的后端接口
