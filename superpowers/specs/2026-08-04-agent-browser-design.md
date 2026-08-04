---
id: "design-agent-browser-runtime-001"
title: "Agent 内置浏览器子系统技术 spec（legionAgent 接线）"
aliases: ["Browser Runtime Spec", "内置浏览器技术规格", "browser runtime spec"]
type: "design"
category: "superpowers/specs"
tags: ["agent", "browser", "runtime", "go-rod", "spec", "legionagent", "sse", "pal"]
version: "1.0.0"
created: "2026-08-04"
updated: "2026-08-04"
author: "jxncyjq"
status: "draft"
parent: null
children: []
related_docs:
  - id: "spec-agent-browser-001"
    relation: "implements"
    path: "../../design/architecture/agent-browser-prd.md"
  - id: "design-agent-working-modes-001"
    relation: "depends_on"
    path: "./2026-07-15-agent-working-modes-design.md"
---

# Agent 内置浏览器子系统技术 spec（legionAgent 接线）

> 本 spec 实现 [[spec-agent-browser-001|Agent 内置浏览器 PRD]]，技术选型/接口/并发/安全/跨平台方案沿用架构设计文档 [[agent-browser-design]]（v2.4）。本文档的**增量价值**在架构文档之外：**落到 legionAgent 具体包结构、工具注册、toolauth 门控、工作模式审批接入、现有会话目录/SSE/SQLite 复用、config 字段、分阶段实施与验收/测试计划**。
>
> 覆盖范围：完整子系统（三运行模式 × 三平台 + 服务模式鉴权/配额）。为保持可落地，§7 按 §14 Roadmap 分阶段切分，每阶段带 Definition of Done。

<!-- @section: overview -->
## 概述

浏览器运行时整体落在 **Go 后端**，是一个自包含、API 优先的核心。任何前端（Wails 内嵌 React、独立 Web/桌面前端、调试脚本）只经统一 API 边界消费，并接收流式截图帧/观测更新用于展示。后端核心不感知自己被内嵌还是被远程调用——由外层传输适配决定。

**关键区分**：Wails WebView（渲染 React 应用界面，Win=WebView2/mac=WKWebView/Linux=WebKitGTK）≠ 自动化 Chromium（go-rod 通过 CDP 控制的独立进程，Agent 浏览的网页跑这里）。两者不共享进程、不共享 cookie。

<!-- @end-section -->

<!-- @section: architecture -->
## 1. 架构与分层

```
        任意前端 (Wails 内嵌 React │ 独立 Web/桌面前端 │ 调试脚本)
                          │
        ┌─────────────────┴──────────────────┐
        │  传输层 (统一)：HTTP POST(命令) + SSE(推流) │
        │  App 模式 = 进程内 loopback；服务模式 = 网络 │
        └─────────────────┬──────────────────┘
                          ▼
  Go 后端核心 (Browser Runtime Core, 部署无关)
      Agent  →  Tool Dispatcher  →  { WebSearch , Browser Runtime }
                                        │
    Session Mgr │ Browser Mgr │ Download Mgr │ Observation Engine
                                        │
                                 go-rod (CDP)
                                        │
                   Chromium 进程（自动化目标，独立于 Wails WebView）
                                        │
                   BrowserContext (incognito) ──→ Page / Tab
```

### 1.1 两个核心概念（严格分离）

- **Session（逻辑）** — Agent 视角一次浏览上下文，绑定 `task_id`。持有 cookies/localStorage/登录态、tab 列表、动作历史栈。Agent 只认 `session_id`。
- **BrowserContext（物理）** — 对应 go-rod 的 incognito browser（`browser.Incognito()`），等价"干净隐身窗口"，Context 间 cookie/cache 完全不串。

go-rod 概念映射：

```
launcher 拉起 Chromium 进程
   └─ rod.Browser（一条 CDP 连接 = 一个 Chromium 进程）
        └─ browser.Incognito() → 一个 BrowserContext（隔离）  ← 一个 Session 绑一个
             └─ context.Page()  → 一个 Page/Tab
```

Session 与 Context **解耦**：进程崩溃或被回收后，Session 换新 Context 重建，从持久化 `storageState` 恢复登录态。进程生命周期独立于会话，是弹性与故障恢复的基础。

### 1.2 分层职责

| 层 | 位置 | 职责 | 状态 |
|---|---|---|---|
| 前端 | Wails WebView / 独立前端 | 交互 UI、展示观测/截图流 | — |
| 传输层 | Go 后端外缘 | 统一 HTTP POST + SSE | 无 |
| Tool Dispatcher | 后端核心（复用现有 `internal/tool`） | 路由、校验、结果裁剪 | 无 |
| Session Manager | 后端核心（新 `internal/browser`） | 会话生命周期、登录态持久化、会话内串行锁 | 有 |
| Browser Manager | 后端核心（新 `internal/browser`） | Chromium 进程 + Context 池、扩缩容、健康检查 | 有 |
| Download Manager | 后端核心（新 `internal/browser`） | 下载拦截、File Cache、内容寻址去重 | 有 |
| Observation Engine | 后端核心（新 `internal/browser`） | a11y/Markdown/截图三表示 + 裁剪 | 无（纯函数） |

<!-- @end-section -->

<!-- @section: legionagent-wiring -->
## 2. legionAgent 接线（本 spec 增量核心）

架构文档停在 `RuntimeAPI`/`PAL` 抽象接口级，本节把它落到 legionAgent 现有代码。

### 2.1 新包布局

```
legionAgent/internal/browser/          # 浏览器运行时核心（平台无关上层）
├── runtime.go          # RuntimeAPI 接口 + 实现（对外唯一契约）
├── session.go          # Session Manager：CRUD/TTL/串行锁/storageState
├── manager.go          # Browser Manager：进程池 + Context 池
├── observation.go      # Observation Engine：a11y/markdown/screenshot + 裁剪
├── download.go         # Download Manager + File Cache
├── errors.go           # 语义错误码（ELEMENT_NOT_FOUND 等）
├── store.go            # Session 元数据/storageState 落盘（复用 agent.db SQLite）
├── platform.go         # PAL 接口定义
├── platform_linux.go   # //go:build linux
├── platform_darwin.go  # //go:build darwin
└── platform_windows.go # //go:build windows

legionAgent/internal/tool/browser.go   # 工具注册层（薄适配，照 web.go 抄）
```

**原则**：`internal/browser` 全平台无关，除 `platform_*.go` 外禁止 `runtime.GOOS` 分支；OS 差异全走 PAL。

### 2.2 工具注册（照 `RegisterWebTools` 抄）

新增 `internal/tool/browser.go`，仿现有 `web.go` 的 `RegisterWebTools(registry, opts)` 模式：

```go
// internal/tool/browser.go
type BrowserToolOptions struct {
    Enabled  bool
    Runtime  browser.RuntimeAPI   // 注入运行时核心
    // 预算/超时/白名单等
}

// RegisterBrowserTools 注册九个 browser_* 工具。registry 为 nil 或 opts.Enabled
// 为 false 时是 no-op（对齐 RegisterWebTools 语义）。
func RegisterBrowserTools(registry *Registry, opts BrowserToolOptions) {
    if registry == nil || !opts.Enabled {
        return
    }
    registry.RegisterDescriptor(browserOpenDescriptor(), HandlerFunc(...))
    registry.RegisterDescriptor(browserReadDescriptor(), HandlerFunc(...))
    // …其余七个
}
```

在装配 registry 的地方（现有 `RegisterWebTools` 调用点旁）加一行 `RegisterBrowserTools(registry, browserOpts)`。

### 2.3 Descriptor 的 Sensitive/Group 判定（接工作模式审批）

现有 `Descriptor` 有 `Sensitive bool`、`RiskLevel`、`Group`、`Timeout` 字段。`Sensitive:true` 触发 [[design-agent-working-modes-001|Manual 模式]] 的审批 gate；Plan 模式经 `Registry.Subset` 只留只读工具。

**判定规则**（本 spec 决策，架构文档未定）：

| 工具 | Sensitive | Group | 理由 |
|---|---|---|---|
| `browser_open` | **true** | browser | 出站网络导航，有副作用 |
| `browser_click` | **true** | browser | 改变页面状态 |
| `browser_type` | **true** | browser | 输入/提交，有副作用 |
| `browser_scroll` | **true** | browser | 触发懒加载/滚动事件 |
| `browser_back`/`forward` | **true** | browser | 导航状态变更 |
| `browser_download_list` 触发的下载 | **true** | browser | 落盘副作用 |
| `browser_read` | false | browser | 纯读当前页 a11y 树 |
| `browser_screenshot` | false | browser | 纯读截图 |
| `browser_extract` | false | browser | 纯读抽取 |
| `browser_close` | false | browser | 释放资源，无外部副作用 |

> **对齐原则**：现有工作模式"只 gate 有副作用工具，只读自动放行"。故 `browser_read/screenshot/extract` 在 Manual 模式下不拦截、Plan 模式下保留可用；导航/点击/输入类拦截。

### 2.4 toolauth 门控登记（必做，否则新工具绕过 per-agent 授权）

九个工具**全部**登记到 `internal/toolauth/catalog.go` 的 `gateable` 列表，使 per-agent 配置的 `disabled_tools` 可勾选禁用。

> **教训**（见 [[legion-per-agent-tool-auth]]）：加新工具必须同步登记 `toolauth.gateable`，否则 per-agent 授权界面看不到、无法禁用，形成授权盲区。

```go
// internal/toolauth/catalog.go 的 gateable 追加
var gateable = []GateableTool{
    // …现有…
    {Name: "browser_open"},     {Name: "browser_read"},
    {Name: "browser_click"},    {Name: "browser_type"},
    {Name: "browser_scroll"},   {Name: "browser_back"},
    {Name: "browser_forward"},  {Name: "browser_screenshot"},
    {Name: "browser_extract"},  {Name: "browser_download_list"},
    {Name: "browser_close"},
}
```

### 2.5 复用现有设施

| 需求 | 复用现有 | 不新造 |
|---|---|---|
| Session 元数据 + storageState 落盘 | 现有 **SQLite `agent.db`**（新建 browser 相关表） | 不引 bbolt |
| 下载缓存/会话文件 | 现有**会话目录**（`<working_dir>/.stardust/session/<id>/` 或 `<workspace.root>/session/<id>/`） | 不另设根 |
| 审批推送 | 现有 **SSE `/v1/events`** + `PlatformEvents` 桥接 | 观测视图流另开 `/sessions/{id}/stream`（见 §4） |
| Manual/Plan/Auto gate | 现有 `runtime.dispatchToolCall` 唯一 gate 点 | 靠 Descriptor.Sensitive 自动接入 |
| SSRF 防护 | 现有 `newSSRFGuardedClient`（`web.go`）的私网校验逻辑 | 扩展到 CDP 导航层（见 §6） |
| 结果裁剪 | 现有 `renderToolResultContent` 超限统一处理 + `ToolResult{Truncated,TokenEstimate}` | — |

<!-- @end-section -->

<!-- @section: components -->
## 3. 组件详细设计（Go 接口）

### 3.1 RuntimeAPI（后端核心对外唯一契约）

```go
type RuntimeAPI interface {
    Open(ctx context.Context, req OpenReq) (Observation, error)
    Read(ctx context.Context, req ReadReq) (Observation, error)
    Click(ctx context.Context, req ClickReq) (Observation, error)
    Type(ctx context.Context, req TypeReq) (Observation, error)
    // …§2.2 其余工具面方法
    Subscribe(sessionID string) (<-chan StreamEvent, func()) // 截图帧/观测流
}
```

工具注册层（`internal/tool/browser.go`）与传输适配器都封装它。

### 3.2 Tool Dispatcher（复用现有薄路由层）

只做三件事，绝不放业务逻辑：参数校验（类型/必填/URL 协议白名单）、分发、**结果裁剪**（原始返回可能几十万 token，按工具约定截断/摘要）。复用现有 `ToolResult` 语义：

```go
type ToolResult[T any] struct {
    Data          T      `json:"data"`
    Truncated     bool   `json:"truncated"`
    TokenEstimate int    `json:"tokenEstimate"`
    SessionID     string `json:"sessionId,omitempty"`
}
```

### 3.3 Session Manager

```go
type Session struct {
    ID           string
    TaskID       string
    Context      *BrowserContext // 指向物理 Context，回收后为 nil
    Tabs         []*TabState
    ActiveTabID  string
    StorageState StorageState    // cookies + origins，持久化
    ActionLog    []Action        // 供回溯/复现
    CreatedAt    time.Time
    LastUsedAt   time.Time
    TTL          time.Duration
    mu sync.Mutex                // 会话内串行锁
}
```

职责：CRUD + TTL 空闲回收（归还 Context、落盘 storageState）；登录态持久化（序列化到会话目录/`agent.db`，重启可恢复）；**会话内串行**。

> **关键决策**：同 Session 内动作串行（`session.mu.Lock()`），跨 Session 各自 goroutine 并行。页面"先点开菜单再选项"有顺序依赖，并发点击会状态竞争；跨会话无共享状态可并行。

### 3.4 Browser Manager（进程 + Context 两级池）

```go
type ContextOpts struct {
    Proxy        string
    UserAgent    string
    Viewport     Viewport
    Stealth      bool
    StorageState *StorageState
}

type BrowserManager interface {
    AcquireContext(opts ContextOpts) (*BrowserContext, error) // 优先复用空闲，不足再扩容或排队
    ReleaseContext(c *BrowserContext) error                   // 归还前 clearCookies + 关闭 pages
    Reap()                                                    // 回收泄漏/僵死进程
}
```

- **进程池**：预热少量 Chromium，避免每次冷启动 300~800ms；按内存水位与 tab 数健康检查，超阈值 graceful 重启（走 PAL 进程终止）。
- **Context 池**：单进程多 incognito Context，彼此隔离；一进程服务多 Session，成本比"一进程一会话"低约一个量级。
- **平台无关**：不写任何 OS 分支，可执行文件定位/启动参数/进程杀死/内存采样/沙箱包装全委托 PAL（§5）。

### 3.5 Observation Engine

**默认 A11y 语义树 + 稳定 ref**：遍历可访问性树（go-rod 取 CDP `Accessibility` 域），只保留"可交互 + 可见 + 有语义"节点，为每个分配**会话内稳定**的 `ref`（如 `e12`）。Agent 用 `click(ref="e12")` 而非猜 CSS selector 或点坐标（后者极脆）。

```
[e3]  <button> 搜索
[e7]  <input>  关键词框  (value="")
[e12] <a href="/item/1"> 商品标题…
```

输出前**预算裁剪**：`max_elements`、按视口/可见性排序、折叠重复列表项，把动辄 50k token 页面压到 1~3k。

### 3.6 Download Manager + File Cache

```go
type DownloadManager interface {
    OnDownload(sessionID string, f RawDownload) FileRef // 存入 cache，返回句柄
    Get(ref FileRef) (io.ReadCloser, error)
    List(sessionID string) []FileMeta
}
```

用 go-rod 处理 CDP `Browser.setDownloadBehavior` 与下载事件，落隔离目录。**File Cache**：按 `session_id` 分区、内容寻址（sha256 去重）、LRU+TTL 清理，落 PAL 缓存目录。返回 Agent 的是 `file_id` + 元信息（名/类型/大小），**不是路径**。大文件超上限直接拒绝（`DOWNLOAD_TOO_LARGE`）。

<!-- @end-section -->

<!-- @section: transport -->
## 4. 传输层（HTTP POST + SSE，三模式统一）

### 4.1 一个核心，三种运行模式

| 模式 | 传输 | 后端与前端关系 |
|---|---|---|
| 开发本地 | HTTP POST + SSE on `localhost` | 后端独立进程，前端/Postman/脚本连本地端口，热重载+断点 |
| 独立服务 | HTTP POST + SSE（带鉴权，走网络） | 后端部署为服务，多客户端远程调用 |
| Wails 独立 App | HTTP POST + SSE（进程内 loopback `127.0.0.1`） | 后端 + React 同包，前端本地自连；Wails 仅作 webview 宿主 |

核心只依赖 `RuntimeAPI`；对外只有**一个 HTTP+SSE 适配器**，三模式复用，仅监听地址/端口/鉴权策略不同。

### 4.2 为何流式选 SSE 而非 WebSocket

流（截图帧/观测/进度）是纯服务器→客户端单向推送，命令另走 HTTP POST——正是 SSE 适用场景。SSE 优势：跑普通 HTTP、穿 L7 代理无摩擦、自带断线重连（`Last-Event-ID` 续传）、实现轻。

### 4.3 SSE 事件模型

一条 `GET /sessions/{id}/stream` 长连接承载多类事件，用 `event:` 字段区分：

```
event: observation        // 观测更新（JSON）
data: {"refs":[...], ...}

event: frame              // 截图帧
data: {"seq":128,"mime":"image/jpeg","b64":"..."}

event: progress           // Agent 动作进度/状态
data: {"action":"click","ref":"e12","status":"done"}

id: 128                   // 供断线后 Last-Event-ID 续传
```

**两个要点**：

1. **二进制帧开销**：SSE 是 UTF-8 文本，JPEG 帧要 base64（+33%）。对策：限帧率（5~10 fps）、按需开关（前端不看时停 screencast）、降质/降分辨率、只在页面有变化时推。将来要高保真高帧率再对帧单独开二进制 WS 旁路，观测/进度仍走 SSE。
2. **断线重连**：客户端带 `Last-Event-ID` 重连，服务端据此跳过已送帧、补发错过的观测/进度事件（帧可丢，状态事件不可丢——小环形缓冲保留最近 N 条状态事件）。

**HTTP/2 建议**：SSE 在 HTTP/1.1 下受浏览器"每域名 ~6 连接"限制，多会话并流易顶格；服务器与客户端间优先 HTTP/2（多路复用无此限制）。

> **与现有 `/v1/events` 的关系**：审批/工作模式事件继续走现有 `/v1/events`；浏览器观测视图流是**独立职责**，另开 `/sessions/{id}/stream`（帧率高、可丢帧、按会话订阅），避免和审批控制流互相拖累。

### 4.4 App 模式落地：进程内 loopback（定案）

**启动握手**（解决端口/token 鸡生蛋）：

```
App 启动
  1. Go 侧选空闲端口，绑 127.0.0.1:<port> 起 HTTP+SSE 服务器
  2. 生成一次性 bearer token（进程级，随启动轮换）
  3. 通过唯一保留的 Wails 注入点，把 { baseURL, token } 交给前端
  4. 前端之后所有请求：fetch/EventSource 带 Authorization: Bearer <token>
```

**加固四件套**（App 模式必做）：只绑 `127.0.0.1`（绝不 `0.0.0.0`）、随机端口、每次启动一次性 bearer token、校验 `Origin`/自定义头。

**工程处理**：端口冲突换口重试；防火墙弹窗绑 `127.0.0.1` 通常不触发（个别 Windows 策略附豁免规则）；服务器随 App 启停，退出优雅关闭 + `Reap` 浏览器进程；webview 的 `fetch`/`EventSource` 只能连 TCP（连不了 Unix socket/命名管道），故 loopback 必须 TCP 端口。

<!-- @end-section -->

<!-- @section: reliability-security -->
## 5. 可靠性与安全

### 5.1 可靠性

- **超时分层**：导航超时、动作超时、`networkidle` 等待、整体任务墙钟。任一层超时返回可读错误而非卡死（Go 侧 `context.Context` 串起 deadline/cancel）。
- **自动等待**：go-rod 内建元素可见/可点等待；对 SPA 补"DOM 稳定检测"（一段时间无 mutation 视为稳定）。避免裸 sleep。
- **错误规范化**（`internal/browser/errors.go`）：

```
ELEMENT_NOT_FOUND    // ref 失效，建议重新 read
NAVIGATION_TIMEOUT   // 导航超时，建议重试或换策略
BLOCKED_BY_CAPTCHA   // 遇验证码
DOWNLOAD_TOO_LARGE   // 超文件上限
CONTEXT_EVICTED      // Context 被回收，需重建 Session
```

- **进程健康**：内存/句柄泄漏检测、僵死进程 `Reap`（走 PAL 终止），graceful 重启期间新请求路由到其他进程。

### 5.2 安全边界

**服务模式额外项**：HTTP POST 与 SSE 入口都校验身份（API token/mTLS；App 模式用一次性 bearer token）。SSE 是长连接，长会话要处理 token 过期/吊销——流内下发 `event: reauth` 要求带新凭证重连，服务端对已吊销流主动断开。`session_id` 绑定调用方身份防串号；配额限流（§8.2 容量模型）。

**通用基线**：

- **独立浏览器身份（重要）**：自动化 Chromium 默认独立干净用户数据目录 + incognito，**不接管用户日常 Chrome profile**。确需登录态走"显式授权 + 独立持久化 profile"，对可访问站点做白名单。
- **SSRF/本地网络防护**：禁止访问本机/局域网（`localhost`/`127.0.0.1`/`169.254.169.254`/`10./192.168./172.16.`）。**DNS 解析后对最终 IP 校验**防 rebinding。
  > **接线**：复用 `web.go` 的 `newSSRFGuardedClient`/`checkURLHostAllowed` 私网校验逻辑，但浏览器场景校验点在 **CDP 导航前**（`browser_open`/页内跳转），不是 HTTP client 层——需在 go-rod 导航拦截钩子里对解析后 IP 校验。
- **协议管控**：拦截 `file://`/`chrome://`/`data:` 危险协议，按业务配域名白/黑名单。
- **沙箱**：Chromium 自身渲染沙箱开启；外层叠加 OS 原生隔离，限制文件系统写入到 Session 隔离目录。三平台机制不同，由 PAL 封装（§5.3）。
- **下载隔离**：只落受控缓存目录，Agent 只拿 `file_id`；不允许指定任意落盘路径。
- **反检测（可选）**：需要时给 Context 注入 stealth（UA、`navigator.webdriver`、指纹一致性）+ 代理；上线前明确合规边界。

<!-- @end-section -->

<!-- @section: cross-platform -->
## 6. 跨平台 PAL（Windows / macOS / Linux）

上层（Session/Observation/Tool）完全平台无关，所有 OS 差异收敛到 PAL，用 Go 构建标签分文件实现。除 PAL 外禁止 `runtime.GOOS` 分支。

> **服务器（无显示器）**：服务模式在无头 Linux 跑，`--headless=new` 即可；镜像需自带 CJK/Emoji 字体，否则截图缺字。桌面 App 有真实显示，automation 浏览器可无头可有头（调试有头更直观）。

### 6.1 差异总览

| 关注点 | Linux | macOS | Windows |
|---|---|---|---|
| Wails WebView | WebKitGTK | WKWebView | WebView2 |
| Chromium 定位 | `chrome`/`chromium` | `Chromium.app/.../MacOS/Chromium` | `chrome.exe` |
| 外层沙箱 | 容器/namespaces + seccomp | App Sandbox / `sandbox-exec` | AppContainer + Job Object |
| 进程终止 | `SIGTERM`→`SIGKILL` | 同（POSIX） | `TerminateProcess` |
| 内存采样 | `/proc/<pid>` | `task_info`/`ps` | PSAPI `GetProcessMemoryInfo` |
| 数据/缓存目录 | `$XDG_*`/`~/.config` | `~/Library/Application Support` | `%LOCALAPPDATA%` |
| 路径分隔符 | `/` | `/` | `\`（内部一律 POSIX，落盘转换） |
| 文件锁语义 | 建议性 | 建议性 | 强制性（占用中不可删/改） |
| 字体 | 精简发行版可能缺 CJK | 齐全 | 齐全 |

### 6.2 PlatformAdapter 接口

```go
type PlatformAdapter interface {
    // 进程
    ResolveChromiumPath() string
    DefaultLaunchArgs() []string
    KillProcess(pid int, graceful bool) error
    // 资源
    SampleProcessMemory(pid int) uint64
    AvailableSystemMemory() uint64
    // 文件系统
    AppDataDir() string
    CacheDir() string
    ToNativePath(posix string) string
    SafeDelete(path string) error       // 处理 Windows 强制锁：先关句柄/重试
    // 隔离
    WrapWithSandbox(cmd *exec.Cmd) *exec.Cmd
}
```

工厂按 `runtime.GOOS` 选实现，上层拿同一接口。

### 6.3 三平台沙箱策略

- **Linux** — 优先容器/namespaces + seccomp-bpf；纯桌面无容器至少限制文件系统写入目录，`--no-sandbox` 谨慎评估。
- **macOS** — App Sandbox（打包 entitlements）为长期方案；`sandbox-exec` 可快速起步但 deprecated。
- **Windows** — AppContainer 能力隔离 + Job Object 限制子进程资源，`JOB_OBJECT_LIMIT_KILL_ON_JOB_CLOSE` 保证 App 退出连带清理 Chromium，杜绝僵尸进程。

### 6.4 Chromium 分发（桌面 App）

**内置固定版 Chromium（推荐）** + **系统兜底**：优先内置版（版本可控、离线可用、行为可复现，代价安装包 +150~300MB/平台），缺失回退系统 Chrome/Edge，实际版本写进日志。首启若走 go-rod launcher 下载模式，需处理无网/被防火墙拦截的降级。

<!-- @end-section -->

<!-- @section: resource -->
## 7. 资源与并发（随运行模式而变）

内存基数（两模式共用）：

```
Mem(process) ≈ BASE + contexts × pages × PAGE
BASE ≈ 200 MB (Chromium 基座，Windows 偏高)
PAGE ≈ 60 MB  (单活跃页面，30~80MB 波动)
```

用可配置 `ResourcePolicy` 抽象，两模式注入不同策略：

### 7.1 App/开发模式：别拖垮用户机器

| 参数 | 默认 | 说明 |
|---|---|---|
| `max_contexts` | 2~4 | 单用户并发会话上限 |
| `max_pages_per_context` | 1~2 | 一般一会话一活跃页 |
| `mem_budget` | ≤ 系统内存 25% 或绝对上限（如 1.5GB） | 触顶排队，不扩容 |
| 低配机降级 | <8GB 内存收紧到 1 context | PAL `AvailableSystemMemory()` 判定 |

用 PAL 实测系统内存定预算，不硬编码；OOM 在用户机器上直接毁体感。

### 7.2 独立服务模式：服务器并发容量

| 服务器内存 | 每进程 ≈560MB(6 ctx) | 最大进程数 | 并发 Session ≈ |
|---|---|---|---|
| 8 GB | 560 MB | ~11 | ~66 |
| 16 GB | 560 MB | ~22 | ~132 |
| 32 GB | 560 MB | ~45 | ~270 |

预留 20% headroom。服务模式还需：按调用方配额、全局队列背压、空闲会话 TTL 回收。CPU 侧每活跃渲染页 0.3~0.7 核峰值，实际以压测为准。

<!-- @end-section -->

<!-- @section: phases -->
## 8. 实施阶段、验收与测试计划

按架构文档 §14 Roadmap 切分，每阶段带 Definition of Done（DoD）。前阶段跑通再进下阶段。

### Phase 1 — 最小闭环（对应里程碑 M1）

**范围**：`internal/browser` 核心（Session/Manager/Observation 骨架）+ `open/read/click/type` + a11y observation + 单进程多 Context + `internal/tool/browser.go` 注册 + toolauth 登记 + 开发本地模式（前后端分开跑，最利调试）。

**DoD**：
- 四工具端到端可用；每动作自动附带最新 observation。
- a11y `ref` 会话内稳定，`click(ref)` 命中；预算裁剪把大页面压到 1~3k token。
- 同 Session 串行、跨 Session 并行（并发用例验证无状态竞争）。
- toolauth 界面能看到并禁用九工具；Manual 模式对 open/click/type 触发审批、对 read 放行。

**测试**：`internal/browser` 单测（observation 裁剪纯函数、session 锁）；一条端到端用例 open→read→type→click→read。

### Phase 2 — 流式观测（M2）

**范围**：`/sessions/{id}/stream` SSE，`observation`/`frame`/`progress` 三类事件 + `Last-Event-ID` 重连 + 环形缓冲补发状态事件；前端 `<canvas>`/`<img>` 展示；帧率/开关/降质控制。

**DoD**：前端实时看见 Agent 浏览过程；断线重连帧可丢、状态事件不丢；不看视图时停 screencast。

**测试**：SSE 事件顺序 + 重连补发用例；帧率限流验证。

### Phase 3 — 会话持久化（M3）

**范围**：storageState + Session 元数据落 `agent.db`（新表）+ 会话目录存下载缓存；TTL 空闲回收（归还 Context、落盘）；Context=nil 后可从磁盘重建恢复登录态。

**DoD**：重启后 Session 恢复登录态；空闲回收后再次访问自动重建 Context；File Cache sha256 去重 + LRU/TTL 生效。

**测试**：重启恢复用例；TTL 回收 + 重建用例；下载去重用例。

### Phase 4 — App 打包 + 跨平台（M4）

**范围**：Wails webview 宿主 + loopback 服务器（握手拿 endpoint+token）；PAL 三平台实现；Chromium 内置/兜底分发；三平台 CI 矩阵。

**DoD**：三平台独立 App 打出；loopback 加固四件套生效；CI 跑同一套端到端用例（open/click/type/download/screenshot）三平台全绿。

**测试**：CI 矩阵 e2e；loopback token/Origin 校验用例；Windows `SafeDelete` 强制锁用例。

### Phase 5 — 安全基线（M5）

**范围**：独立身份 + SSRF（CDP 导航前 IP 校验防 rebinding）+ 协议拦截 + 三平台原生沙箱；服务模式鉴权/会话归属/配额。

**DoD**：SSRF 用例（私网 + DNS rebinding）全拦；`file://`/`chrome://`/`data:` 拦截；服务模式跨调用方不可见/不可操作他人 session；沙箱限制写入到会话目录。

**测试**：SSRF/rebinding 用例；协议拦截用例；会话归属越权用例；配额限流用例。

### Phase 6 — 资源与压测（M6）

**范围**：进程池扩缩容 + 健康检查/`Reap` + 两种 `ResourcePolicy` 压测。

**DoD**：App 模式内存触顶排队不 OOM；服务模式并发达容量模型预期；僵死进程被 `Reap`，graceful 重启不掉请求。

**测试**：内存水位压测；并发 Session 吞吐压测；进程泄漏/重启用例。

### Phase 7 — 按需扩展（M7）

**范围**：screenshot/set-of-marks 视觉定位、接管模式（回注鼠标/键盘）、反检测（stealth + 代理）。按分叉点触发。

**DoD**：按实际需求分项交付，非全集。

<!-- @end-section -->

<!-- @section: forks -->
## 9. 分叉点

| 分叉点 | 触发 | 走向 |
|---|---|---|
| 服务模式多租户强隔离/计费 | 商业化 | §5.2 会话归属 + §7.2 配额为起点；再上一租户一进程/容器 + 计量计费。本期不做 |
| 人工接管浏览 | 遇验证码/需人工 | §4.3 只读展示上加"接管模式"：暂停 Agent 动作、前端鼠标/键盘事件回注 Page |
| 纯信息抽取不需交互 | 场景退化 | 砍 Session 锁 + 交互工具，退化为 navigate+render+extract，资源大降 |
| 视觉模型点坐标 | a11y 不够用 | Observation 默认切 screenshot + set-of-marks，a11y 树降为辅助 |

<!-- @end-section -->

<!-- @section: appendix -->
## 附录 A：App 传输选型决策记录（Binding+Events vs loopback HTTP+SSE）

App 模式曾在两种接法取舍，本质区别是走不走网络栈。最终选 **loopback HTTP+SSE**（§4.4），此表保留作决策记录。

| 维度 | Binding + Events | loopback HTTP + SSE（选中） |
|---|---|---|
| 代码路径 | 与 dev/服务是两套 | 与 dev/服务同一套 ✅ |
| 前端可调试性 | 耦合 `window.go.*` | 纯 HTTP 客户端，DevTools 直连 ✅ |
| 流的重连语义 | 广播 fire-and-forget，补发自造 | SSE 自带 Last-Event-ID ✅ |
| 网络暴露面 | 无监听端口 ✅ | 有 loopback 端口，须 127.0.0.1+token 加固 |
| 防火墙/UX | 无弹窗 ✅ | 个别 Windows 或触发弹窗 |
| 性能 | IPC 直传略低 ✅ | 多一层 loopback，命令级可忽略 |
| 生命周期 | Wails 托管 ✅ | 自管端口/启停 |

**取舍**：牺牲"零端口暴露/无防火墙/略低开销"，换"三模式一条代码路径 + 前端独立可调 + SSE 重连复用"。暴露面用加固四件套消化。若日后安全策略禁止任何本地监听端口，可退回 Binding+Events——业务都在 `RuntimeAPI` 接口后，切换只动适配器一层。

## 相关文档

- [[spec-agent-browser-001|Agent 内置浏览器 PRD]] — 本 spec 实现的产品需求
- [[agent-browser-design]] — 架构设计文档 v2.4（技术选型/接口/并发/安全/跨平台定案来源）
- [[design-agent-working-modes-001|Agent 工作模式]] — 浏览器工具接入 Manual/Plan/Auto 审批 gate 的依赖
- [[legion-per-agent-tool-auth]] — per-agent toolauth 门控（新工具必须登记 gateable）
