---
id: "plans-agent-wails-gui-impl-001"
title: "Agent GUI 实现计划 — Wails Desktop Application"
type: "implementation-plan"
category: "plans/agent"
tags: ["plan", "agent", "gui", "wails", "react", "desktop"]
version: "0.1.0"
created: "2026-06-24"
updated: "2026-06-24"
status: "draft"
related_docs:
  - path: "./2026-06-24-wails-gui-design.md"
    relation: "implements"
  - path: "./task-breakdown.md"
    relation: "extends"
---

# Agent GUI 实现计划 — Wails Desktop Application

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** 新增 `legion/legionAgentGUI/` Wails 桌面应用，复用 legionAgent 的 `agent serve` 内部服务，React 前端提供三栏 Chat + Status Panel + Sidebar 界面，单二进制发布。

**Architecture:** `legionAgentGUI` 作为独立 Go module，通过 `go.work` workspace import `github.com/stardust/legion-agent`。`serve_manager.go` 调用从 cli 包提取的 `BuildServeService()` 函数，在同进程 goroutine 内启动 HTTP 服务器。React 通过 `fetch` 调用本地 HTTP API，流式输出通过 Go 侧消费 SSE 再用 `runtime.EventsEmit` 转发到 React。

**Tech Stack:** Wails v2、React 18 + TypeScript、Tailwind CSS v4、shadcn/ui、react-markdown + shiki、Zustand、Vite

**前置条件：**
- 安装 Wails v2: `go install github.com/wailsapp/wails/v2/cmd/wails@latest`
- Node.js 18+，npm 可用
- Go 1.21+

---

## Task 1: 提取 BuildServeService 可复用函数

**背景：** `newServeCommand`（`legion/legionAgent/internal/cli/command.go:1121`）有 ~160 行复杂依赖装配。legionAgentGUI 需调用相同逻辑，必须先提取成函数，避免重复。

**Files:**
- Modify: `legion/legionAgent/internal/cli/command.go`
- Create: `legion/legionAgent/internal/cli/command_test.go`（已存在，添加测试）

**Step 1: 在 command.go 末尾添加 ServeOptions struct 和 BuildServeService 函数**

在 `command.go` 文件末尾（`newDoctorCommand` 之后）添加：

```go
// ServeOptions holds the parameters needed to build a serve Service.
// Extracted so legionAgentGUI can call this without the cobra command wrapper.
type ServeOptions struct {
    ConfigPath string
    Addr       string
    Logger     *slog.Logger
}

// ServeResult holds the running service and a cleanup function.
type ServeResult struct {
    Service  *service.Service
    Listener net.Listener
    Close    func()
}

// BuildServeService constructs and returns a ready-to-Start service from the
// same dependency wiring as newServeCommand, but without cobra.
func BuildServeService(ctx context.Context, opts ServeOptions) (ServeResult, error) {
    cfg, err := config.Load(ctx, config.Options{Path: opts.ConfigPath})
    if err != nil {
        return ServeResult{}, err
    }
    addr := opts.Addr
    if addr == "" {
        addr = cfg.Server.ListenAddr
    }
    if addr == "" {
        addr = "127.0.0.1:0" // random port for GUI mode
    }

    taskStore, workflowStore, sessionStore, readiness, closeStore, err := serviceStores(ctx, cfg)
    if err != nil {
        return ServeResult{}, err
    }

    var auditLog port.AuditLog
    var qualityEvals server.QualityEvalStore
    var messageStore tool.AgentMessageStore
    if repo, ok := taskStore.(*storage.SQLiteRepository); ok {
        auditLog = storage.NewSQLiteAuditLog(repo)
        qualityEvals = repo
        messageStore = repo
    }
    if auditLog == nil {
        auditLog = adapter.NewMemoryAuditLog()
    }

    workflowEvents := adapter.NewMemoryEventBus()
    liveTasks := task.NewScheduler()
    httpTasks := server.TaskStore(liveTasks)
    if taskStore != nil {
        httpTasks = teeTaskStore{live: liveTasks, persistent: taskStore}
    }
    approvals := approval.NewService()
    workflowEngine := workflow.NewEngine(workflow.Config{
        Scheduler: liveTasks,
        Approvals: approvals,
        Events:    workflowEvents,
        Audit:     auditLog,
    })
    registry, err := loadServeAgentRegistry(ctx, cfg, opts.ConfigPath)
    if err != nil {
        closeStore()
        return ServeResult{}, err
    }
    taskLedger, err := newCommandTaskLedger(cfg)
    if err != nil {
        closeStore()
        return ServeResult{}, err
    }
    resolver := agentruntime.NewAgentRuntimeResolver(agentruntime.AgentRuntimeResolverConfig{
        Registry:     registry,
        RootConfig:   cfg,
        Audit:        auditLog,
        Events:       workflowEvents,
        TaskLedger:   taskLedger,
        MessageStore: messageStore,
        MaasFactory:  maasFactoryFromConfig(cfg.Maas),
    })
    defaultMaas, err := adapter.NewMaasClientFromProfile(cfg.Maas, "")
    if err != nil {
        closeStore()
        return ServeResult{}, err
    }
    if defaultMaas == nil {
        defaultMaas = adapter.NewRecordingMaas(cfg.Runtime.DemoResponse)
    }
    defaultDisplay := tuiDisplayConfig(cfg.Maas, "", "")
    defaultContext, err := buildRunContextPrefix(ctx, cfg, false, defaultDisplay.ModelName)
    if err != nil {
        closeStore()
        return ServeResult{}, err
    }
    defaultTools := tool.NewReadOnlyWorkspaceRegistry(cfg.ContextFiles.Root, auditLog)
    tool.RegisterTaskLedgerTools(defaultTools, taskLedger)
    tool.RegisterAgentMessageTools(defaultTools, messageStore)
    coordinator := agentruntime.NewCoordinator(agentruntime.CoordinatorConfig{
        Agent: domain.Agent{
            ID:        "default-agent",
            CompanyID: "default-company",
            Role:      "developer",
            Status:    domain.AgentActive,
        },
        Scheduler: liveTasks,
        Locks:     task.NewLockStore(),
        Runtime: agentruntime.NewRuntime(agentruntime.Config{
            Maas:           defaultMaas,
            Audit:          auditLog,
            Events:         workflowEvents,
            ContextBuilder: cognitive.NewCore(cognitive.NoopCompressor{}).WithContextFiles(defaultContext),
            Tools:          defaultTools,
            MaxToolRounds:  cfg.Runtime.MaxToolRounds,
        }),
        TaskRunnerResolver: resolver,
        Reviewer:           quality.NewAegisReviewer(),
        Evaluator:          quality.NewEvalEngine(3),
        Approvals:          approvals,
        Audit:              auditLog,
        Events:             workflowEvents,
    })
    background := task.NewBackgroundScheduler()
    background.AddJob("agent-coordinator-heartbeat", func(ctx context.Context) error {
        _, _, err := coordinator.Heartbeat(ctx)
        return err
    })
    metrics := observability.NewMetricsRecorder(nil)
    listener, err := net.Listen("tcp", addr)
    if err != nil {
        closeStore()
        return ServeResult{}, fmt.Errorf("listen on %q: %w", addr, err)
    }

    logger := opts.Logger
    if logger == nil {
        logger = defaultLogger()
    }

    httpServer := server.NewHTTPServer(server.Config{
        Tasks:               httpTasks,
        Workflows:           workflowStore,
        WorkflowEngine:      workflowEngine,
        WorkflowEvents:      workflowEvents,
        Readiness:           readiness,
        AdminToken:          cfg.Server.AdminToken,
        PublicHealthEnabled: cfg.Server.PublicHealthEnabled,
        RequestIDHeader:     cfg.Server.RequestIDHeader,
        Audit:               auditLog,
        QualityEvals:        qualityEvals,
        Sessions:            sessionStore,
        Messages:            messageStore,
        Logger:              logger,
        Metrics:             metrics,
        Diagnostics: observability.NewDiagnostics(observability.DiagnosticsConfig{
            Version:             "dev",
            StorageDriver:       cfg.Storage.Driver,
            StoragePath:         cfg.Storage.Path,
            MaasBaseURL:         cfg.Maas.BaseURL,
            MaasAPIKey:          cfg.Maas.APIKey,
            AdminToken:          cfg.Server.AdminToken,
            RuntimeDemoResponse: cfg.Runtime.DemoResponse,
            SchedulerEnabled:    true,
            SchedulerRunning:    true,
            Metrics:             metrics,
        }),
    })
    svc, err := service.New(service.ServiceConfig{
        Config:    cfg,
        Scheduler: background,
        HTTPServer: &http.Server{
            Handler: httpServer,
        },
        Listener: listener,
        Logger:   logger,
    })
    if err != nil {
        _ = listener.Close()
        closeStore()
        return ServeResult{}, err
    }
    return ServeResult{
        Service:  svc,
        Listener: listener,
        Close:    closeStore,
    }, nil
}
```

**Step 2: 更新 newServeCommand 调用 BuildServeService（消除重复）**

将 `newServeCommand` 的 `RunE` 函数体替换为：

```go
RunE: func(cmd *cobra.Command, _ []string) error {
    if cmd.Context().Err() != nil {
        _, err := fmt.Fprintln(out, "agent service stopped")
        return err
    }
    result, err := BuildServeService(cmd.Context(), ServeOptions{
        ConfigPath: configPath,
        Addr:       addr,
    })
    if err != nil {
        return err
    }
    defer result.Close()
    if err := result.Service.Start(cmd.Context()); err != nil {
        return err
    }
    _, err = fmt.Fprintln(out, "agent service stopped")
    return err
},
```

**Step 3: 编译验证**

```powershell
cd legion/legionAgent
go build ./...
go vet ./...
```

期望：无错误

**Step 4: 确认 serve 命令功能未退化**

```powershell
go test ./internal/cli/... -run TestServe -v -count=1
```

**Step 5: Commit**

```
git add legion/legionAgent/internal/cli/command.go
git commit -m "refactor(cli): extract BuildServeService for reuse by legionAgentGUI"
```

---

## Task 2: 初始化 legionAgentGUI Wails 项目

**Files:**
- Create: `legion/legionAgentGUI/` (整个目录)
- Create: `legion/go.work`

**Step 1: 在 legion/ 目录下初始化 go.work**

```powershell
cd legion
go work init ./legionAgent
```

验证生成 `legion/go.work`：

```
go 1.21

use ./legionAgent
```

**Step 2: 初始化 Wails 项目**

```powershell
cd legion
wails init -n legionAgentGUI -t react-ts -d legionAgentGUI
```

这会生成：
```
legionAgentGUI/
├── main.go
├── app.go
├── go.mod          # module github.com/stardust/legion-agent-gui
├── go.sum
├── wails.json
└── frontend/
    ├── src/
    │   ├── App.tsx
    │   └── main.tsx
    ├── package.json
    └── vite.config.ts
```

**Step 3: 将 legionAgentGUI 加入 go.work**

```powershell
cd legion
go work use ./legionAgentGUI
```

`legion/go.work` 变为：

```
go 1.21

use (
    ./legionAgent
    ./legionAgentGUI
)
```

**Step 4: 在 legionAgentGUI/go.mod 添加 legionAgent 依赖**

编辑 `legion/legionAgentGUI/go.mod`，添加：

```
require github.com/stardust/legion-agent v0.0.0
```

因为有 go.work，`replace` 指令不需要，go.work 的 `use` 已处理。

**Step 5: 验证工程可构建**

```powershell
cd legion/legionAgentGUI
wails build
```

期望：输出 `legionAgentGUI.exe`（Windows），无编译错误

**Step 6: Commit**

```
git add legion/go.work legion/legionAgentGUI/
git commit -m "feat(gui): initialize legionAgentGUI Wails project with go.work workspace"
```

---

## Task 3: 实现 ServeManager

**Files:**
- Create: `legion/legionAgentGUI/serve_manager.go`

**Step 1: 创建 serve_manager.go**

```go
package main

import (
    "context"
    "fmt"
    "net"

    "github.com/wailsapp/wails/v2/pkg/runtime"

    "github.com/stardust/legion-agent/internal/cli"
)

type ServeManager struct {
    cancel   context.CancelFunc
    port     int
    appCtx   context.Context
}

func NewServeManager() *ServeManager {
    return &ServeManager{}
}

// Start launches the legion-agent HTTP service in-process.
// It picks a random port if configPath's config has no ListenAddr.
func (m *ServeManager) Start(appCtx context.Context, configPath string) error {
    ctx, cancel := context.WithCancel(context.Background())
    m.cancel = cancel

    result, err := cli.BuildServeService(ctx, cli.ServeOptions{
        ConfigPath: configPath,
        Addr:       "127.0.0.1:0",
    })
    if err != nil {
        cancel()
        return fmt.Errorf("build serve service: %w", err)
    }

    m.port = listenerPort(result.Listener)

    runtime.EventsEmit(appCtx, "serve:status", map[string]any{
        "running": true,
        "port":    m.port,
    })

    go func() {
        defer result.Close()
        if err := result.Service.Start(ctx); err != nil {
            runtime.EventsEmit(appCtx, "serve:error", map[string]any{"error": err.Error()})
        }
        runtime.EventsEmit(appCtx, "serve:status", map[string]any{
            "running": false,
            "port":    0,
        })
    }()

    return nil
}

func (m *ServeManager) Stop() {
    if m.cancel != nil {
        m.cancel()
    }
}

func (m *ServeManager) Port() int {
    return m.port
}

func listenerPort(l net.Listener) int {
    if l == nil {
        return 0
    }
    addr, ok := l.Addr().(*net.TCPAddr)
    if !ok {
        return 0
    }
    return addr.Port
}
```

**Step 2: 验证编译**

```powershell
cd legion/legionAgentGUI
go build ./...
```

**Step 3: Commit**

```
git add legion/legionAgentGUI/serve_manager.go
git commit -m "feat(gui): add ServeManager — in-process legion-agent HTTP service lifecycle"
```

---

## Task 4: 实现 App struct（Wails 绑定层）

**Files:**
- Modify: `legion/legionAgentGUI/app.go`

**Step 1: 替换 Wails 生成的 app.go**

```go
package main

import (
    "context"
    "fmt"
    "net/http"
    "io"
    "encoding/json"
)

type App struct {
    ctx     context.Context
    serve   *ServeManager
    cfgPath string
}

func NewApp(cfgPath string) *App {
    return &App{
        serve:   NewServeManager(),
        cfgPath: cfgPath,
    }
}

func (a *App) startup(ctx context.Context) {
    a.ctx = ctx
    if err := a.serve.Start(ctx, a.cfgPath); err != nil {
        // serve start failure is non-fatal; UI shows disconnected state
        _ = err
    }
}

func (a *App) shutdown(_ context.Context) {
    a.serve.Stop()
}

// Port returns the port the embedded HTTP service is listening on.
func (a *App) Port() int {
    return a.serve.Port()
}

// BaseURL returns the base URL for the embedded HTTP service.
func (a *App) BaseURL() string {
    return fmt.Sprintf("http://127.0.0.1:%d", a.serve.Port())
}

// apiGet is a helper for Go-side HTTP calls to the local service.
func (a *App) apiGet(path string) ([]byte, error) {
    resp, err := http.Get(a.BaseURL() + path)
    if err != nil {
        return nil, err
    }
    defer resp.Body.Close()
    return io.ReadAll(resp.Body)
}

// ListSessions returns session IDs for the default agent.
// Called by React via Wails TypeScript bindings.
func (a *App) ListSessions() ([]map[string]any, error) {
    body, err := a.apiGet("/api/v1/sessions?agent_id=default-agent&company_id=default-company")
    if err != nil {
        return nil, err
    }
    var result []map[string]any
    if err := json.Unmarshal(body, &result); err != nil {
        return nil, err
    }
    return result, nil
}
```

**Step 2: 更新 main.go 传入 cfgPath**

替换生成的 `main.go`：

```go
package main

import (
    "embed"
    "os"

    "github.com/wailsapp/wails/v2"
    "github.com/wailsapp/wails/v2/pkg/options"
    "github.com/wailsapp/wails/v2/pkg/options/assetserver"
)

//go:embed all:frontend/dist
var assets embed.FS

func main() {
    cfgPath := ""
    if len(os.Args) > 1 {
        cfgPath = os.Args[1]
    }

    app := NewApp(cfgPath)
    err := wails.Run(&options.App{
        Title:  "Legion Agent",
        Width:  1280,
        Height: 800,
        AssetServer: &assetserver.Options{
            Assets: assets,
        },
        OnStartup:  app.startup,
        OnShutdown: app.shutdown,
        Bind: []interface{}{
            app,
        },
    })
    if err != nil {
        println("Error:", err.Error())
    }
}
```

**Step 3: 生成 Wails TypeScript bindings**

```powershell
cd legion/legionAgentGUI
wails generate module
```

确认 `frontend/wailsjs/go/main/App.d.ts` 包含 `Port()`, `BaseURL()`, `ListSessions()` 方法。

**Step 4: 验证构建**

```powershell
wails build
```

**Step 5: Commit**

```
git add legion/legionAgentGUI/app.go legion/legionAgentGUI/main.go
git commit -m "feat(gui): wire App struct with ServeManager startup/shutdown and Wails bindings"
```

---

## Task 5: 安装前端依赖（Tailwind v4 + shadcn/ui + Zustand）

**Files:**
- Modify: `legion/legionAgentGUI/frontend/package.json`
- Create: `legion/legionAgentGUI/frontend/tailwind.config.ts`（Tailwind v4 via Vite plugin，无需此文件）
- Modify: `legion/legionAgentGUI/frontend/vite.config.ts`

**Step 1: 安装 npm 依赖**

```powershell
cd legion/legionAgentGUI/frontend
npm install zustand react-markdown @shikijs/rehype rehype-raw clsx tailwind-merge
npm install -D tailwindcss @tailwindcss/vite
npx shadcn@latest init
```

shadcn init 时选择：
- Style: New York
- Base color: Zinc
- CSS variables: yes

**Step 2: 配置 Tailwind v4（vite plugin 方式）**

编辑 `vite.config.ts`：

```typescript
import { defineConfig } from 'vite'
import react from '@vitejs/plugin-react'
import tailwindcss from '@tailwindcss/vite'

export default defineConfig({
  plugins: [
    react(),
    tailwindcss(),
  ],
})
```

**Step 3: 在 src/index.css 顶部添加 Tailwind 导入**

```css
@import "tailwindcss";
```

**Step 4: 验证前端构建**

```powershell
npm run build
```

期望：`dist/` 目录生成无报错

**Step 5: Commit**

```
git add legion/legionAgentGUI/frontend/
git commit -m "feat(gui): install Tailwind v4, shadcn/ui, Zustand, react-markdown, shiki"
```

---

## Task 6: 三栏布局骨架

**Files:**
- Modify: `legion/legionAgentGUI/frontend/src/App.tsx`
- Create: `legion/legionAgentGUI/frontend/src/components/Sidebar.tsx`
- Create: `legion/legionAgentGUI/frontend/src/components/ChatPanel.tsx`
- Create: `legion/legionAgentGUI/frontend/src/components/StatusPanel.tsx`
- Create: `legion/legionAgentGUI/frontend/src/components/layout/ThreePanelLayout.tsx`

**Step 1: 创建 ThreePanelLayout**

```tsx
// src/components/layout/ThreePanelLayout.tsx
import { useState } from 'react'
import { cn } from '../../lib/utils'

interface Props {
  sidebar: React.ReactNode
  chat: React.ReactNode
  status: React.ReactNode
}

export function ThreePanelLayout({ sidebar, chat, status }: Props) {
  const [sidebarOpen, setSidebarOpen] = useState(true)
  const [statusOpen, setStatusOpen] = useState(true)

  return (
    <div className="flex h-screen bg-background text-foreground overflow-hidden">
      {/* Sidebar */}
      <aside className={cn(
        'flex-shrink-0 border-r border-border transition-all duration-200',
        sidebarOpen ? 'w-56' : 'w-12'
      )}>
        <button
          className="w-full p-2 text-xs text-muted-foreground hover:text-foreground"
          onClick={() => setSidebarOpen(o => !o)}
        >
          {sidebarOpen ? '◀' : '▶'}
        </button>
        {sidebarOpen && sidebar}
      </aside>

      {/* Chat (flex-1) */}
      <main className="flex-1 min-w-0 flex flex-col">
        {chat}
      </main>

      {/* Status Panel */}
      <aside className={cn(
        'flex-shrink-0 border-l border-border transition-all duration-200',
        statusOpen ? 'w-72' : 'w-12'
      )}>
        <button
          className="w-full p-2 text-xs text-muted-foreground hover:text-foreground"
          onClick={() => setStatusOpen(o => !o)}
        >
          {statusOpen ? '▶' : '◀'}
        </button>
        {statusOpen && status}
      </aside>
    </div>
  )
}
```

**Step 2: 创建占位组件**

```tsx
// src/components/Sidebar.tsx
export function Sidebar() {
  return <div className="p-4 text-sm text-muted-foreground">Sidebar</div>
}

// src/components/ChatPanel.tsx
export function ChatPanel() {
  return <div className="flex-1 p-4 text-sm text-muted-foreground">Chat</div>
}

// src/components/StatusPanel.tsx
export function StatusPanel() {
  return <div className="p-4 text-sm text-muted-foreground">Status</div>
}
```

**Step 3: 更新 App.tsx**

```tsx
import { ThreePanelLayout } from './components/layout/ThreePanelLayout'
import { Sidebar } from './components/Sidebar'
import { ChatPanel } from './components/ChatPanel'
import { StatusPanel } from './components/StatusPanel'

function App() {
  return (
    <ThreePanelLayout
      sidebar={<Sidebar />}
      chat={<ChatPanel />}
      status={<StatusPanel />}
    />
  )
}

export default App
```

**Step 4: 验证布局渲染**

```powershell
cd legion/legionAgentGUI
wails dev
```

在浏览器中验证：三栏可见，点击 ◀/▶ 折叠正常。

**Step 5: Commit**

```
git add legion/legionAgentGUI/frontend/src/
git commit -m "feat(gui): add three-panel layout skeleton with collapsible sidebar and status panel"
```

---

## Task 7: Zustand 状态 Store

**Files:**
- Create: `legion/legionAgentGUI/frontend/src/stores/chatStore.ts`
- Create: `legion/legionAgentGUI/frontend/src/stores/sessionStore.ts`
- Create: `legion/legionAgentGUI/frontend/src/stores/serveStore.ts`

**Step 1: 创建 serveStore（连接状态）**

```typescript
// src/stores/serveStore.ts
import { create } from 'zustand'

interface ServeState {
  running: boolean
  port: number
  setStatus: (running: boolean, port: number) => void
}

export const useServeStore = create<ServeState>((set) => ({
  running: false,
  port: 0,
  setStatus: (running, port) => set({ running, port }),
}))
```

**Step 2: 创建 chatStore（消息 + 流式 token）**

```typescript
// src/stores/chatStore.ts
import { create } from 'zustand'

export interface Message {
  id: string
  role: 'user' | 'assistant'
  content: string
  streaming?: boolean
}

interface ChatState {
  messages: Message[]
  addMessage: (msg: Message) => void
  appendToken: (id: string, token: string) => void
  finalizeMessage: (id: string) => void
  clearMessages: () => void
}

export const useChatStore = create<ChatState>((set) => ({
  messages: [],
  addMessage: (msg) =>
    set((s) => ({ messages: [...s.messages, msg] })),
  appendToken: (id, token) =>
    set((s) => ({
      messages: s.messages.map((m) =>
        m.id === id ? { ...m, content: m.content + token } : m
      ),
    })),
  finalizeMessage: (id) =>
    set((s) => ({
      messages: s.messages.map((m) =>
        m.id === id ? { ...m, streaming: false } : m
      ),
    })),
  clearMessages: () => set({ messages: [] }),
}))
```

**Step 3: 创建 sessionStore**

```typescript
// src/stores/sessionStore.ts
import { create } from 'zustand'

interface SessionState {
  currentSessionId: string
  sessions: string[]
  setCurrentSession: (id: string) => void
  setSessions: (ids: string[]) => void
}

export const useSessionStore = create<SessionState>((set) => ({
  currentSessionId: '',
  sessions: [],
  setCurrentSession: (id) => set({ currentSessionId: id }),
  setSessions: (ids) => set({ sessions: ids }),
}))
```

**Step 4: Commit**

```
git add legion/legionAgentGUI/frontend/src/stores/
git commit -m "feat(gui): add Zustand stores for serve status, chat messages, and sessions"
```

---

## Task 8: SSE 桥接 — Go 侧消费推 Wails 事件

**背景：** React 不能直接消费 Go 进程内的 SSE（同源限制），需 Go 侧开 goroutine 读 SSE，用 `runtime.EventsEmit` 转发到 React。

**Files:**
- Create: `legion/legionAgentGUI/sse_bridge.go`
- Modify: `legion/legionAgentGUI/app.go`

**Step 1: 创建 sse_bridge.go**

```go
package main

import (
    "bufio"
    "context"
    "fmt"
    "net/http"
    "strings"

    "github.com/wailsapp/wails/v2/pkg/runtime"
)

// StartSSEBridge opens a persistent SSE connection to the local agent serve
// and forwards each event to React via runtime.EventsEmit.
func StartSSEBridge(ctx context.Context, appCtx context.Context, baseURL string) {
    go func() {
        url := baseURL + "/api/v1/events"
        for {
            if err := ctx.Err(); err != nil {
                return
            }
            if err := consumeSSE(ctx, appCtx, url); err != nil {
                // retry silently; serve may not be ready yet
                select {
                case <-ctx.Done():
                    return
                default:
                }
            }
        }
    }()
}

func consumeSSE(ctx context.Context, appCtx context.Context, url string) error {
    req, err := http.NewRequestWithContext(ctx, http.MethodGet, url, nil)
    if err != nil {
        return err
    }
    req.Header.Set("Accept", "text/event-stream")
    resp, err := http.DefaultClient.Do(req)
    if err != nil {
        return err
    }
    defer resp.Body.Close()

    scanner := bufio.NewScanner(resp.Body)
    var eventType string
    for scanner.Scan() {
        line := scanner.Text()
        switch {
        case strings.HasPrefix(line, "event:"):
            eventType = strings.TrimSpace(strings.TrimPrefix(line, "event:"))
        case strings.HasPrefix(line, "data:"):
            data := strings.TrimSpace(strings.TrimPrefix(line, "data:"))
            if eventType != "" && data != "" {
                runtime.EventsEmit(appCtx, "agent:event", map[string]any{
                    "type": eventType,
                    "data": data,
                })
                // Token events get a dedicated channel for the chat stream
                if eventType == "runtime.token" || eventType == "token" {
                    runtime.EventsEmit(appCtx, "agent:token", data)
                }
            }
            eventType = ""
        case line == "":
            eventType = ""
        }
    }
    return fmt.Errorf("SSE stream ended: %w", scanner.Err())
}
```

**Step 2: 在 App.startup 中启动 SSE 桥接**

在 `app.go` 的 `startup` 函数末尾添加：

```go
func (a *App) startup(ctx context.Context) {
    a.ctx = ctx
    if err := a.serve.Start(ctx, a.cfgPath); err != nil {
        _ = err
        return
    }
    // Give the HTTP server a moment to bind, then start SSE bridge
    StartSSEBridge(ctx, ctx, a.BaseURL())
}
```

**Step 3: 验证 SSE 桥接（手动测试）**

```powershell
wails dev
```

在 DevTools Console 中：

```javascript
import { EventsOn } from '../wailsjs/runtime/runtime'
EventsOn('agent:token', (data) => console.log('token:', data))
```

发送一个 prompt 后，检查 Console 是否输出 token 事件。

**Step 4: Commit**

```
git add legion/legionAgentGUI/sse_bridge.go
git add legion/legionAgentGUI/app.go
git commit -m "feat(gui): add SSE bridge — Go goroutine consumes agent serve SSE, forwards via Wails events"
```

---

## Task 9: Chat 组件（输入 + 流式输出）

**Files:**
- Modify: `legion/legionAgentGUI/frontend/src/components/ChatPanel.tsx`
- Create: `legion/legionAgentGUI/frontend/src/components/MessageBubble.tsx`
- Create: `legion/legionAgentGUI/frontend/src/hooks/useAgentEvents.ts`

**Step 1: 创建 useAgentEvents hook**

```typescript
// src/hooks/useAgentEvents.ts
import { useEffect } from 'react'
import { EventsOn, EventsOff } from '../../wailsjs/runtime/runtime'
import { useChatStore } from '../stores/chatStore'
import { useServeStore } from '../stores/serveStore'

export function useAgentEvents() {
  const { appendToken } = useChatStore()
  const { setStatus } = useServeStore()

  useEffect(() => {
    let currentStreamId = ''

    const handleToken = (data: string) => {
      if (!currentStreamId) {
        currentStreamId = `stream-${Date.now()}`
        useChatStore.getState().addMessage({
          id: currentStreamId,
          role: 'assistant',
          content: '',
          streaming: true,
        })
      }
      appendToken(currentStreamId, data)
    }

    const handleEvent = (payload: { type: string }) => {
      if (payload.type === 'runtime.done' || payload.type === 'done') {
        if (currentStreamId) {
          useChatStore.getState().finalizeMessage(currentStreamId)
          currentStreamId = ''
        }
      }
    }

    const handleStatus = (status: { running: boolean; port: number }) => {
      setStatus(status.running, status.port)
    }

    EventsOn('agent:token', handleToken)
    EventsOn('agent:event', handleEvent)
    EventsOn('serve:status', handleStatus)

    return () => {
      EventsOff('agent:token')
      EventsOff('agent:event')
      EventsOff('serve:status')
    }
  }, [appendToken, setStatus])
}
```

**Step 2: 创建 MessageBubble 组件**

```tsx
// src/components/MessageBubble.tsx
import ReactMarkdown from 'react-markdown'
import { cn } from '../lib/utils'
import type { Message } from '../stores/chatStore'

interface Props {
  message: Message
}

export function MessageBubble({ message }: Props) {
  return (
    <div className={cn(
      'max-w-[80%] rounded-lg px-4 py-3 text-sm',
      message.role === 'user'
        ? 'self-end bg-primary text-primary-foreground ml-auto'
        : 'self-start bg-muted text-foreground'
    )}>
      {message.role === 'assistant' ? (
        <ReactMarkdown className="prose prose-sm dark:prose-invert max-w-none">
          {message.content || (message.streaming ? '▋' : '')}
        </ReactMarkdown>
      ) : (
        <p className="whitespace-pre-wrap">{message.content}</p>
      )}
      {/* Copy button */}
      <button
        className="mt-1 text-xs text-muted-foreground hover:text-foreground"
        onClick={() => navigator.clipboard.writeText(message.content)}
      >
        Copy
      </button>
    </div>
  )
}
```

**Step 3: 实现 ChatPanel**

```tsx
// src/components/ChatPanel.tsx
import { useRef, useEffect, useState } from 'react'
import { BaseURL } from '../../wailsjs/go/main/App'
import { useChatStore } from '../stores/chatStore'
import { useAgentEvents } from '../hooks/useAgentEvents'
import { MessageBubble } from './MessageBubble'

export function ChatPanel() {
  useAgentEvents()

  const messages = useChatStore((s) => s.messages)
  const addMessage = useChatStore((s) => s.addMessage)
  const [input, setInput] = useState('')
  const [sending, setSending] = useState(false)
  const bottomRef = useRef<HTMLDivElement>(null)

  useEffect(() => {
    bottomRef.current?.scrollIntoView({ behavior: 'smooth' })
  }, [messages])

  async function sendMessage() {
    const prompt = input.trim()
    if (!prompt || sending) return

    addMessage({ id: `user-${Date.now()}`, role: 'user', content: prompt })
    setInput('')
    setSending(true)

    try {
      const base = await BaseURL()
      await fetch(`${base}/api/v1/tasks`, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ prompt, agent_id: 'default-agent' }),
      })
    } finally {
      setSending(false)
    }
  }

  return (
    <div className="flex flex-col h-full">
      {/* Message list */}
      <div className="flex-1 overflow-y-auto p-4 flex flex-col gap-3">
        {messages.map((msg) => (
          <MessageBubble key={msg.id} message={msg} />
        ))}
        <div ref={bottomRef} />
      </div>

      {/* Input */}
      <div className="border-t border-border p-3 flex gap-2">
        <textarea
          className="flex-1 resize-none rounded-md border border-input bg-background px-3 py-2 text-sm"
          rows={3}
          value={input}
          onChange={(e) => setInput(e.target.value)}
          onKeyDown={(e) => {
            if (e.key === 'Enter' && !e.shiftKey) {
              e.preventDefault()
              sendMessage()
            }
          }}
          placeholder="输入消息... (Enter 发送, Shift+Enter 换行)"
          disabled={sending}
        />
        <button
          className="px-4 py-2 bg-primary text-primary-foreground rounded-md text-sm disabled:opacity-50"
          onClick={sendMessage}
          disabled={sending}
        >
          {sending ? '...' : '发送'}
        </button>
      </div>
    </div>
  )
}
```

**Step 4: 端到端手动测试**

```powershell
wails dev
```

- 输入 prompt，按 Enter
- 验证消息出现在 Chat
- 验证 Assistant 回复流式展示

**Step 5: Commit**

```
git add legion/legionAgentGUI/frontend/src/components/
git add legion/legionAgentGUI/frontend/src/hooks/
git commit -m "feat(gui): implement ChatPanel with streaming message rendering and send input"
```

---

## Task 10: Status Panel（四个视图 Tab）

**Files:**
- Modify: `legion/legionAgentGUI/frontend/src/components/StatusPanel.tsx`
- Create: `legion/legionAgentGUI/frontend/src/components/status/EventsTab.tsx`
- Create: `legion/legionAgentGUI/frontend/src/components/status/TasksTab.tsx`
- Create: `legion/legionAgentGUI/frontend/src/components/status/AuditTab.tsx`
- Create: `legion/legionAgentGUI/frontend/src/components/status/InboxTab.tsx`

**Step 1: 实现 StatusPanel 框架（Tab 切换）**

```tsx
// src/components/StatusPanel.tsx
import { useState } from 'react'
import { cn } from '../lib/utils'
import { EventsTab } from './status/EventsTab'
import { TasksTab } from './status/TasksTab'
import { AuditTab } from './status/AuditTab'
import { InboxTab } from './status/InboxTab'

type Tab = 'events' | 'tasks' | 'audit' | 'inbox'

const tabs: { id: Tab; label: string }[] = [
  { id: 'events', label: '事件' },
  { id: 'tasks', label: '任务' },
  { id: 'audit', label: 'Audit' },
  { id: 'inbox', label: 'Inbox' },
]

export function StatusPanel() {
  const [activeTab, setActiveTab] = useState<Tab>('events')

  return (
    <div className="flex flex-col h-full">
      {/* Tab bar */}
      <div className="flex border-b border-border">
        {tabs.map((tab) => (
          <button
            key={tab.id}
            className={cn(
              'flex-1 py-2 text-xs font-medium',
              activeTab === tab.id
                ? 'text-foreground border-b-2 border-primary'
                : 'text-muted-foreground hover:text-foreground'
            )}
            onClick={() => setActiveTab(tab.id)}
          >
            {tab.label}
          </button>
        ))}
      </div>

      {/* Tab content */}
      <div className="flex-1 overflow-y-auto">
        {activeTab === 'events' && <EventsTab />}
        {activeTab === 'tasks' && <TasksTab />}
        {activeTab === 'audit' && <AuditTab />}
        {activeTab === 'inbox' && <InboxTab />}
      </div>
    </div>
  )
}
```

**Step 2: EventsTab（订阅 agent:event Wails 事件）**

```tsx
// src/components/status/EventsTab.tsx
import { useEffect, useState } from 'react'
import { EventsOn } from '../../../wailsjs/runtime/runtime'

interface EventItem {
  type: string
  data: string
  ts: number
}

export function EventsTab() {
  const [events, setEvents] = useState<EventItem[]>([])

  useEffect(() => {
    EventsOn('agent:event', (payload: { type: string; data: string }) => {
      setEvents((prev) => [
        { type: payload.type, data: payload.data, ts: Date.now() },
        ...prev.slice(0, 99),  // keep last 100
      ])
    })
  }, [])

  return (
    <div className="p-2 flex flex-col gap-1">
      {events.length === 0 && (
        <p className="text-xs text-muted-foreground">等待事件...</p>
      )}
      {events.map((e) => (
        <div key={e.ts} className="text-xs border-b border-border py-1">
          <span className="text-muted-foreground font-mono">{e.type}</span>
          <p className="truncate text-foreground">{e.data}</p>
        </div>
      ))}
    </div>
  )
}
```

**Step 3: TasksTab（轮询 /api/v1/tasks）**

```tsx
// src/components/status/TasksTab.tsx
import { useEffect, useState } from 'react'
import { BaseURL } from '../../../wailsjs/go/main/App'

interface Task {
  id: string
  status: string
  prompt?: string
}

export function TasksTab() {
  const [tasks, setTasks] = useState<Task[]>([])

  async function refresh() {
    try {
      const base = await BaseURL()
      const resp = await fetch(`${base}/api/v1/tasks`)
      const data = await resp.json()
      setTasks(Array.isArray(data) ? data : [])
    } catch {
      // ignore network errors while serve starts
    }
  }

  useEffect(() => {
    refresh()
    const id = setInterval(refresh, 3000)
    return () => clearInterval(id)
  }, [])

  return (
    <div className="p-2 flex flex-col gap-1">
      {tasks.length === 0 && (
        <p className="text-xs text-muted-foreground">无任务</p>
      )}
      {tasks.map((t) => (
        <div key={t.id} className="text-xs border-b border-border py-1">
          <span className="font-mono text-muted-foreground">{t.status}</span>
          <p className="truncate">{t.id}</p>
        </div>
      ))}
    </div>
  )
}
```

**Step 4: AuditTab 和 InboxTab（占位，结构同 TasksTab，URL 不同）**

```tsx
// src/components/status/AuditTab.tsx
export function AuditTab() {
  return <div className="p-2 text-xs text-muted-foreground">Audit 日志（待实现）</div>
}

// src/components/status/InboxTab.tsx
export function InboxTab() {
  return <div className="p-2 text-xs text-muted-foreground">Inbox（待实现）</div>
}
```

**Step 5: 验证 Status Panel**

```powershell
wails dev
```

切换四个 Tab，验证 Events Tab 在 Agent 运行时显示事件流，Tasks Tab 每 3 秒刷新。

**Step 6: Commit**

```
git add legion/legionAgentGUI/frontend/src/components/status/
git add legion/legionAgentGUI/frontend/src/components/StatusPanel.tsx
git commit -m "feat(gui): implement Status Panel with Events/Tasks/Audit/Inbox tabs"
```

---

## Task 11: Sidebar（Session + Agent 切换）

**Files:**
- Modify: `legion/legionAgentGUI/frontend/src/components/Sidebar.tsx`
- Modify: `legion/legionAgentGUI/app.go`（添加更多 Wails 绑定）

**Step 1: 在 app.go 添加 Session 操作绑定**

在 `app.go` 的 `App` struct 方法中追加：

```go
// NewSession creates a new session via the HTTP API.
func (a *App) NewSession() error {
    resp, err := http.Post(
        a.BaseURL()+"/api/v1/sessions",
        "application/json",
        strings.NewReader(`{"agent_id":"default-agent","company_id":"default-company","title":"GUI session"}`),
    )
    if err != nil {
        return err
    }
    defer resp.Body.Close()
    return nil
}
```

重新生成 bindings：

```powershell
wails generate module
```

**Step 2: 实现 Sidebar**

```tsx
// src/components/Sidebar.tsx
import { useEffect } from 'react'
import { ListSessions, NewSession } from '../../wailsjs/go/main/App'
import { useSessionStore } from '../stores/sessionStore'
import { cn } from '../lib/utils'

export function Sidebar() {
  const { sessions, currentSessionId, setSessions, setCurrentSession } = useSessionStore()

  async function loadSessions() {
    try {
      const result = await ListSessions()
      const ids = (result || []).map((s: any) => s.id || s)
      setSessions(ids)
    } catch {
      // serve not ready yet
    }
  }

  useEffect(() => {
    loadSessions()
    const id = setInterval(loadSessions, 5000)
    return () => clearInterval(id)
  }, [])

  return (
    <div className="p-2 flex flex-col gap-2 h-full">
      {/* New session button */}
      <button
        className="w-full text-xs py-1.5 px-2 bg-primary text-primary-foreground rounded"
        onClick={async () => {
          await NewSession()
          loadSessions()
        }}
      >
        + 新建 Session
      </button>

      {/* Session list */}
      <div className="flex flex-col gap-1 overflow-y-auto">
        <p className="text-xs text-muted-foreground px-1">Sessions</p>
        {sessions.map((id) => (
          <button
            key={id}
            className={cn(
              'text-left text-xs px-2 py-1 rounded truncate',
              currentSessionId === id
                ? 'bg-accent text-accent-foreground'
                : 'hover:bg-muted text-muted-foreground hover:text-foreground'
            )}
            onClick={() => setCurrentSession(id)}
          >
            {id}
          </button>
        ))}
      </div>
    </div>
  )
}
```

**Step 3: 验证 Sidebar**

```powershell
wails dev
```

点击「新建 Session」，列表更新，点击 Session 高亮切换。

**Step 4: Commit**

```
git add legion/legionAgentGUI/frontend/src/components/Sidebar.tsx
git add legion/legionAgentGUI/app.go
git commit -m "feat(gui): implement Sidebar with session list and new session creation"
```

---

## Task 12: 布局状态持久化 + 连接状态 Badge

**Files:**
- Modify: `legion/legionAgentGUI/frontend/src/components/layout/ThreePanelLayout.tsx`
- Create: `legion/legionAgentGUI/frontend/src/components/ConnectionBadge.tsx`
- Modify: `legion/legionAgentGUI/frontend/src/App.tsx`

**Step 1: 持久化折叠状态到 localStorage**

修改 `ThreePanelLayout.tsx` 中的 state：

```tsx
const [sidebarOpen, setSidebarOpen] = useState(
  () => localStorage.getItem('sidebarOpen') !== 'false'
)
const [statusOpen, setStatusOpen] = useState(
  () => localStorage.getItem('statusOpen') !== 'false'
)

// persist on change
const toggleSidebar = () => setSidebarOpen((o) => {
  const next = !o
  localStorage.setItem('sidebarOpen', String(next))
  return next
})
const toggleStatus = () => setStatusOpen((o) => {
  const next = !o
  localStorage.setItem('statusOpen', String(next))
  return next
})
```

**Step 2: 创建 ConnectionBadge**

```tsx
// src/components/ConnectionBadge.tsx
import { useServeStore } from '../stores/serveStore'

export function ConnectionBadge() {
  const { running, port } = useServeStore()
  return (
    <div className="flex items-center gap-1.5 text-xs px-2 py-1">
      <span className={`w-2 h-2 rounded-full ${running ? 'bg-green-500' : 'bg-red-400'}`} />
      <span className="text-muted-foreground">
        {running ? `连接中 :${port}` : '未连接'}
      </span>
    </div>
  )
}
```

**Step 3: 在 App.tsx 顶部添加 Badge**

```tsx
function App() {
  return (
    <div className="flex flex-col h-screen">
      <div className="flex items-center justify-end border-b border-border px-2 py-0.5 bg-background">
        <ConnectionBadge />
      </div>
      <div className="flex-1 min-h-0">
        <ThreePanelLayout
          sidebar={<Sidebar />}
          chat={<ChatPanel />}
          status={<StatusPanel />}
        />
      </div>
    </div>
  )
}
```

**Step 4: 最终验证**

```powershell
wails build
```

运行生成的二进制，验证：
- 启动后连接 Badge 变绿
- 折叠/展开侧边栏后重启，状态恢复
- 发送 prompt，Chat 流式回复
- Status Panel Events Tab 显示事件

**Step 5: Commit**

```
git add legion/legionAgentGUI/frontend/src/
git commit -m "feat(gui): persist panel layout state to localStorage, add connection status badge"
```

---

## 验证清单

```powershell
# Go 侧
cd legion/legionAgent
go test ./internal/cli/... -count=1     # serve 命令测试通过
go build ./...                           # legionAgent 编译成功

cd ../legionAgentGUI
go build ./...                           # legionAgentGUI 编译成功
wails build                              # 生成单二进制

# 前端
cd frontend
npm run build                            # Vite 构建无错误
```

---

## 任务表

| 任务 | 描述 | 前置 |
|------|------|------|
| Task 1 | 提取 BuildServeService | — |
| Task 2 | 初始化 Wails 项目 + go.work | Task 1 |
| Task 3 | ServeManager | Task 2 |
| Task 4 | App struct + main.go | Task 3 |
| Task 5 | 前端依赖（Tailwind/shadcn/Zustand） | Task 2 |
| Task 6 | 三栏布局骨架 | Task 5 |
| Task 7 | Zustand stores | Task 5 |
| Task 8 | SSE 桥接 | Task 4 |
| Task 9 | ChatPanel + 流式渲染 | Task 7, 8 |
| Task 10 | StatusPanel 四 Tab | Task 7, 8 |
| Task 11 | Sidebar Session 切换 | Task 4, 7 |
| Task 12 | 布局持久化 + Badge | Task 6, 7, 9 |
