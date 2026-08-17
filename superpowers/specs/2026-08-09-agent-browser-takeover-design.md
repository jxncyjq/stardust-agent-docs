---
id: "design-agent-browser-takeover-001"
title: "Agent 内置浏览器接管模式技术 spec（人工回注输入 + 暂停 Agent）"
aliases: ["Browser Takeover", "接管模式", "browser takeover spec"]
type: "design"
category: "superpowers/specs"
tags: ["agent", "browser", "takeover", "cdp", "input-injection", "gui", "react", "spec"]
version: "1.0.0"
created: "2026-08-09"
updated: "2026-08-09"
author: "jxncyjq"
status: "draft"
parent: null
children: []
related_docs:
  - id: "design-agent-browser-view-ui-001"
    relation: "extends"
    path: "./2026-08-08-agent-browser-view-ui-design.md"
  - id: "design-agent-browser-runtime-001"
    relation: "depends_on"
    path: "./2026-08-04-agent-browser-design.md"
---

# Agent 内置浏览器接管模式技术 spec

<!-- @section: overview -->
## 概述

### 目标（为什么做）

Agent 自主浏览时会遇到**它过不去、必须人来的一步**——验证码、登录二次验证、支付确认、需要人判断的弹窗。当前浏览器视图（[[design-agent-browser-view-ui-001]]）是**只读**的（用户只能看，不能操作）。**接管模式**让用户在**卡住的那一刻临时手动操作 Agent 正在用的那个浏览器会话**，过掉这一步后交还 Agent 继续。

### 结果（做完是什么样）

用户在 GUI 浏览器视图点"接管"→ canvas 变为可交互，用户的鼠标/键盘操作**注入到 Agent 真实的 go-rod Chromium 会话**（同 cookie、同页面态），screencast 照常刷新让用户看到自己的操作生效；接管期间 Agent 对该会话的写动作被挡（返回 `SESSION_UNDER_TAKEOVER`，Agent 退让等待）；用户点"退出接管"→ 交还，Agent 恢复。**净结果**：验证码/登录这类"必须人来"的一步不再让整个 Agent 任务卡死。

<!-- @end-section -->

<!-- @section: decision -->
## 1. 架构决策记录：为何"注入 go-rod 会话"而非"原生 webview"

> 本节记录一个被认真评估并否决的替代方案，供审阅者理解取舍。**目标**：接管既然要交互，能否直接在前端嵌一个原生可交互浏览器视图（效率更高、交互原生）？**结果**：否——对我们的形态既不可行也语义错误，注入 go-rod 会话是正确且必需的路。

**评估的替代方案**：前端嵌原生 webview（参考 Electron 的 hermes-desktop：`<webview>` 原生可交互 + 隔离世界 JS 覆盖层选元素）。

**否决依据（证据）**：

| 维度 | 原生 webview | screencast + 注入 go-rod（选中） |
|---|---|---|
| Wails 支持 | v2 **无多 webview 原语**（Electron 的 `<webview>`/BrowserView Wails 没有；v3 仅多顶层窗口且 beta） | 已有：帧走 SSE，注入走反向 POST，同一传输 |
| 跨平台 | WebView2-CDP 仅 Windows；WKWebView/WebKitGTK **无 CDP**（自建要维护 WebKit patch） | go-rod/CDP 三平台一致 |
| 无头/服务模式 | 原生控件在无显示的 Linux 服务不存在 | CDP screencast 只依赖 Chromium，可 headless/xvfb |
| **接管对象正确性（决定性）** | 原生 WebView2 是**另一个浏览器实例**，不同 cookie/session/页面态——用户在里面操作**影响不到 Agent 卡住的会话** | 注入的是 **Agent 真实的 go-rod 会话本身**，用户解的验证码正是 Agent 卡的那个 |
| go-webview2 实测 | `Embed/Navigate/ExecuteScript` 有，但**无 `CallDevToolsProtocolMethod`**（无 CDP）→ Agent 驱不了它，退化成弱 JS 注入 + 全核心重写 | — |
| 行业验证 | 无一产品做 native 双挂载 | Browserbase Live View / Steel.dev / OpenAI Operator **全是 stream + 注入** |

**结论**：接管**必须注入 Agent 真实的 go-rod 会话**（否则接管的是错误的浏览器实例）。native webview 既受 Wails/跨平台/无头限制，又对象错误。**证据来源**：Browserbase Live View 文档、Steel.dev WebRTC 博文、OpenAI Operator、go-webview2 `pkg/edge/chromium.go`（无 CDP 方法）、Wails v2 单 webview 事实。

<!-- @end-section -->

<!-- @section: architecture -->
## 2. 架构与数据流

```
进入接管
  React POST /v1/browser/sessions/{id}/takeover {enabled:true} (bearer)
    → 后端 session.takeover = true
    → Agent 对该 session 的写动作(open/click/type/scroll/back/forward)返回 SESSION_UNDER_TAKEOVER

输入注入（接管期间）
  用户在 canvas 操作 → 前端捕获 mouse/keyboard → 映射归一化坐标(0..1，相对 canvas 显示矩形)
    → 批量 + 节流 POST /v1/browser/sessions/{id}/input {events:[...]} (bearer)
    → 后端 Runtime.InjectInput：校验 → 归一化 × 当前视口px → go-rod Mouse/Keyboard 注入活跃 Page
    → 该 Page 的 screencast 照常推流 → 用户看到自己的操作生效

退出接管
  React POST .../takeover {enabled:false} → session.takeover = false → Agent 恢复
  （session 关闭时自动退接管）
```

**关键不变量**：注入作用于 `session.ActivePage`（Agent 正用的那个 go-rod page）；接管期间只有人在写，Agent 写动作被挡——避免人机并发点击竞争（对齐设计文档 §3.2/§5.2 只读默认 + 会话内串行）。

<!-- @end-section -->

<!-- @section: details -->
## 3. 组件详细设计

### 3.1 后端 legionAgent

**Session 状态**（`internal/browser/session.go`）：`Session` 加 `takeover bool`（会话锁下读写）。

**门控**（`internal/browser/runtime.go`）：
- **目标**：接管期间 Agent 不与人打架。
- **结果**：写动作 `Open/Click/Type/Scroll/Back/Forward` 进入时检查 `takeover`，为真则返回 `NewBrowserError(CodeTakeover, "session under manual takeover")`（`errors.go` 加 `CodeTakeover Code = "SESSION_UNDER_TAKEOVER"`）。**只读动作 `Read/Screenshot/Extract` 放行**（Agent 可继续观测，无害）——与 Sensitive 判定一致。

**新方法**（`RuntimeAPI` 扩展或 server 侧 `BrowserController` 接口，`*Runtime` 满足）：
```go
// SetTakeover 设置会话接管标志（会话锁下）。enabled=false 恢复 Agent。
SetTakeover(sessionID string, enabled bool) error
// InjectInput 把一批输入事件注入会话活跃页（会话锁下，接管必须先开）。
InjectInput(sessionID string, events []InputEvent) error
```
`InputEvent`（归一化坐标模型）：
```go
type InputEvent struct {
    Type   string  `json:"type"`   // mousemove|mousedown|mouseup|click|wheel|keydown|keyup|char
    X      float64 `json:"x"`      // 0..1 归一化（相对视口）；鼠标类必填
    Y      float64 `json:"y"`      // 0..1
    Button string  `json:"button,omitempty"` // left|right|middle（鼠标类）
    DeltaX float64 `json:"deltaX,omitempty"` // wheel
    DeltaY float64 `json:"deltaY,omitempty"`
    Key    string  `json:"key,omitempty"`    // keydown/keyup 的键名
    Text   string  `json:"text,omitempty"`   // char：要输入的文本
}
```
`InjectInput` 实现（会话锁下，`takeover` 必须为 true 否则拒）：归一化 (X,Y) × 当前视口宽高 → px；按 Type 分发到 go-rod：`mousemove→page.Mouse.MoveTo`、`mousedown/up→Mouse.Down/Up`、`click→Down+Up`、`wheel→Mouse.Scroll(deltaX,deltaY)`、`keydown/keyup→Keyboard.Press/或 Down/Up`、`char→page.InsertText(text)`。

**输入严格校验**（借鉴 hermes-desktop 的 `normalizeSelection` 硬校验；fail-loud）：
- 事件类型必须在白名单（上列 8 种），否则整批拒（返回 error，不静默跳过单条）。
- X/Y 必须 `0 ≤ v ≤ 1` 且 finite；越界拒。
- Button ∈ {left,right,middle}；Key 长度上限（如 ≤ 32）；Text 长度上限（如 ≤ 1024）。
- 单批 events 条数上限（如 ≤ 256），防滥用。

**HTTP 端点**（`internal/server/browser_input.go`，`internal/server/http.go` 加路由）：
- `POST /v1/browser/sessions/{id}/takeover`，body `{"enabled":bool}` → `SetTakeover`。
- `POST /v1/browser/sessions/{id}/input`，body `{"events":[InputEvent...]}` → 要求该会话 takeover=true，否则 409；→ `InjectInput`。
- 二者经现有 `authorized()` bearer 守（仅 loopback/服务受控），复用 Phase 4B 加固。server 侧消费的 browser 接口（Phase 2 的 `BrowserStreamer`）扩展加 `SetTakeover`/`InjectInput`（`*Runtime` 满足；不进 `RuntimeAPI` 避免污染 fake，同 ReplaySince 先例）。

### 3.2 前端 GUI React

**BrowserView 加接管开关**（`components/BrowserView.tsx`）：
- **目标**：一键进入/退出接管，接管态清晰可辨。
- **结果**：开关按钮；开 → `POST takeover{enabled:true}`（用 `GetBrowserEndpoint()` 的 token）；canvas 加 `pointer-events` 恢复 + `tabIndex`（获键盘焦点）；显醒目"接管中"横幅（区别只读态）。关 → `POST takeover{enabled:false}`，canvas 恢复只读。

**输入捕获 + 映射**（`lib/browserInput.ts`）：
- canvas 事件监听：`mousemove`(节流 ~40fps)/`mousedown`/`mouseup`/`click`/`wheel`/`keydown`/`keyup`。
- 坐标映射：用 canvas `getBoundingClientRect()`，`x = (e.clientX - rect.left) / rect.width`，`y = (e.clientY - rect.top) / rect.height`，clamp 0..1。
- 键盘：`keydown/keyup` 发 `{type, key: e.key}`；可打印字符经 `char` 事件发 `{type:'char', text}`（或直接映射 key）。
- 批量+节流 POST `/input`（合并同一 tick 的 mousemove，减请求）。

**坐标模型**：前端只发归一化 0..1，后端 × 视口——鲁棒于 canvas CSS 缩放 / 帧分辨率变化，无需前端知道设备像素。

<!-- @end-section -->

<!-- @section: error-handling -->
## 4. 错误处理（fail-loud，不静默）

- **input POST 失败**：前端 `console.error` + 保持接管态（或据错误码提示）；连续失败可提示"注入失败"。
- **takeover 切换失败**：回滚 UI 开关态 + `console.error`。
- **后端未知 session / 无活跃页**：404 / 语义错误（不静默返回成功）。
- **input 校验失败**：整批拒 + 返回 error（越界坐标/非法类型/超限），不静默跳过单条。
- **takeover 未开就调 /input**：409（必须先进接管）。
- **Agent 写动作遇 takeover**：返回 `SESSION_UNDER_TAKEOVER`（Agent 可见，退让/重试，非崩溃）。
- **session 关闭**：`Close` 里若 takeover 为真，自动清标志（避免悬挂）。

<!-- @end-section -->

<!-- @section: testing -->
## 5. 测试

**后端 legionAgent**
- `SetTakeover` 置位/清位；takeover=true 时 `Click/Type/Open` 返回 `SESSION_UNDER_TAKEOVER`，`Read` 放行（单测，无 Chromium）。
- `InjectInput` 归一化→px 映射正确（如 (0.5,0.5)×视口 → 中心）；校验：越界坐标/非法类型/超限批量被拒（单测）。
- HTTP handler：`/takeover` 置标志；`/input` 未接管时 409、接管时调 InjectInput；bearer 守（httptest）。
- chromium 集成（`//go:build chromium`）：进接管 → InjectInput click 到一个按钮坐标 → 页面 onclick 触发（断言 fetch 命中）；InjectInput char 输入到 input → 值变。

**前端 GUI React（vitest）**
- `browserInput` 坐标映射：给定 rect + clientX/Y → 归一化正确 + clamp。
- 节流：连续 mousemove 合并。
- BrowserView：接管开关 POST takeover（mock fetch/GetBrowserEndpoint）；接管态 canvas 有 pointer-events + "接管中"横幅；退出恢复只读。

<!-- @end-section -->

<!-- @section: scope -->
## 6. 范围边界与后续

| 项 | 本 spec | 后续 |
|---|---|---|
| 拖拽选择（down→move→up 序列态） | 不做（实用集：点击+移动+滚轮+键盘） | 需要时补拖拽序列 |
| 多 tab 接管 | 只作用活跃 tab | tab 管理后 |
| 帧保真/帧率不够 | screencast base64（Phase 2） | **WebRTC 升级**（Steel.dev 式，传输层直接替换，非架构重写） |
| Windows-only 高保真 | 不做 | 未来可选：WebView2-CDP（需 CDP 绑定，本 go-webview2 无；跨平台/无头受限，永非主路） |
| 元素选取器（set-of-marks 式） | 不做 | 视觉分叉：go-rod `page.Eval` 注 JS 覆盖层（类比 hermes 隔离世界选元素） |
| IME/组合输入 | char 走 InsertText 基础覆盖 | 复杂 IME 组合态后续 |

<!-- @end-section -->

## 相关文档

- [[design-agent-browser-view-ui-001|浏览器视图 UI spec]] — 接管建立在只读视图之上（本 spec 扩展它）
- [[design-agent-browser-runtime-001|Browser Runtime 技术 spec]] — go-rod 会话/Page（注入目标）、§3.4 loopback bearer（端点鉴权）
- [[agent-browser-design]] — 架构 v2.4 §3.2 只读默认 + 接管分叉、§13 人工接管分叉点
