---
id: "plans-agent-wails-gui-design-001"
title: "Agent GUI 设计文档 — Wails Desktop Application"
type: "design"
category: "plans/agent"
tags: ["plan", "agent", "gui", "wails", "react", "desktop"]
version: "0.1.0"
created: "2026-06-24"
updated: "2026-06-24"
status: "draft"
related_docs:
  - path: "./task-breakdown.md"
    relation: "extends"
  - path: "./cli-tui-plan.md"
    relation: "sibling"
---

# Agent GUI 设计文档 — Wails Desktop Application

## §1 架构

### 并行存在策略

保留现有 `agent tui` 命令不动，新增 `agent gui` 命令启动 Wails 桌面应用。两者独立，用户可按需选用。

### 目录结构

```
legion/
├── legionAgent/          # 现有 CLI + TUI（不变）
│   └── internal/
│       ├── tui/          # Bubble Tea TUI
│       └── service/      # HTTP 服务实现（agent serve）
└── legionAgentGUI/       # 新建 Wails 桌面应用
    ├── main.go           # Wails 入口
    ├── app.go            # App struct，Wails 生命周期
    ├── serve_manager.go  # 进程内 HTTP 服务管理
    └── frontend/         # React 前端
        ├── src/
        │   ├── components/
        │   ├── stores/    # Zustand
        │   └── App.tsx
        └── package.json
```

### serve_manager.go — 进程内模式

`serve_manager.go` **直接 import** `legionAgent/internal/service` 包，在 goroutine 内启动 HTTP 服务器，与 Wails 应用同进程运行：

```go
// serve_manager.go
import "github.com/stardust/legion/legionAgent/internal/service"

type ServeManager struct {
    server *service.Server
    port   int
    cancel context.CancelFunc
}

func (m *ServeManager) Start(cfg *service.Config) error {
    ctx, cancel := context.WithCancel(context.Background())
    m.cancel = cancel
    m.server = service.NewServer(cfg)
    go m.server.Run(ctx, m.port)  // 同进程，goroutine 内
    return nil
}

func (m *ServeManager) Stop() {
    m.cancel()
}
```

优点：单二进制发布，GUI 窗口关闭即服务停止，无需进程间通信管理。

---

## §2 UI 布局

三栏布局，水平排列：

```
┌─────────────┬──────────────────────┬─────────────────┐
│   Sidebar   │       Chat           │   Status Panel  │
│             │                      │                 │
│  Agent 列表 │  消息历史             │  Agent 运行状态 │
│  Session 列表│  用户输入框          │  事件流         │
│  功能菜单   │                      │  任务列表       │
│             │                      │  Audit 日志     │
└─────────────┴──────────────────────┴─────────────────┘
```

- **Sidebar**（左，~220px，可折叠）：Agent 切换、Session 切换、新建 Session、技能管理菜单
- **Chat**（中，弹性）：消息历史滚动区 + 底部输入框，支持 Markdown 渲染 + 代码高亮
- **Status Panel**（右，~280px，默认展开，可折叠缩进）：对应 TUI 的 audit/event/tasks/inbox 视图，通过 Tab 或菜单切换

Status Panel 默认展开，用户可点击边缘收起（缩进为图标栏），等效于 TUI 中 `/event`、`/audit`、`/tasks`、`/inbox` 视图的常驻可视区。

---

## §3 数据流

### 请求响应（HTTP）

```
React → fetch/axios → legionAgent HTTP API（:port）→ 返回 JSON
```

覆盖：发送消息、查询 session 列表、切换 session、安装 skill、消息历史等。

### 流式输出（SSE → Wails Events）

```
agent serve SSE stream
    → Go goroutine (serve_manager.go) 读取 SSE
    → runtime.EventsEmit(ctx, "agent:token", payload)
    → React EventsOn("agent:token", handler)
    → Zustand store 追加 token → Chat 组件渲染
```

### 服务状态事件

```
ServeManager.Start/Stop
    → runtime.EventsEmit(ctx, "serve:status", {running, port})
    → React 显示连接状态 badge
```

---

## §4 技术栈

| 层 | 选型 |
|----|------|
| 桌面框架 | Wails v2（Go backend + Web frontend，单二进制） |
| 前端框架 | React 18 + TypeScript |
| 构建工具 | Vite（Wails 内置） |
| 样式 | Tailwind CSS v4 |
| 组件库 | shadcn/ui |
| Markdown 渲染 | react-markdown + shiki（代码高亮） |
| 状态管理 | Zustand |
| HTTP 客户端 | 原生 fetch（Wails 环境） |
| SSE 消费 | Go 侧消费 + `runtime.EventsEmit` 转发 |

### TypeScript 自动绑定

Wails 自动从 Go struct 生成 TypeScript bindings，Go 侧暴露的方法可直接在 React 中调用：

```typescript
import { GetConfig, ListSessions } from '../wailsjs/go/main/App'

const sessions = await ListSessions()
```

---

## §5 MVP 功能范围

### 必须有（对齐 TUI 核心功能）

- 发送 prompt，流式展示回复
- Session 创建、切换、清空
- Agent 切换（`@agent` 路由）
- Status Panel：事件流、任务列表、Audit 日志、Inbox 查看
- Skill 安装 / 更新 / 卸载

### GUI 独有功能

- 持久化布局偏好（panel 展开/折叠状态）
- Markdown 富文本渲染（TUI 仅文本）
- 代码块语法高亮
- 消息复制按钮

### 超出 MVP，不做

- 多窗口 / 多 Tab Agent 并排
- 内嵌浏览器工具预览
- 插件市场 UI

---

## §6 后续步骤

1. 用 `wails init` 创建 `legion/legionAgentGUI/` 项目骨架
2. 实现 `ServeManager`，验证进程内启动 `agent serve`
3. 搭建三栏 React 布局（shadcn/ui + Tailwind）
4. 接入 Chat 流式渲染（SSE → Wails Events）
5. 实现 Status Panel 四个视图 tab
6. 实现 Sidebar Session / Agent 切换
7. 端到端联调验证
