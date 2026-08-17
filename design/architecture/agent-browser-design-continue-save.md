---
id: "reference-agent-browser-continue-001"
title: "Agent 内置浏览器子系统 — 进度存档与接续指南"
aliases: ["Agent Browser Continue", "浏览器接续存档", "browser handoff"]
type: "reference"
category: "design/architecture"
tags: ["agent", "browser", "handoff", "progress", "continue", "roadmap"]
version: "1.0.0"
created: "2026-08-09"
updated: "2026-08-09"
author: "jxncyjq"
status: "published"
parent: null
children: []
related_docs:
  - id: "spec-agent-browser-001"
    relation: "references"
    path: "./agent-browser-prd.md"
  - id: "design-agent-browser-runtime-001"
    relation: "references"
    path: "../../superpowers/specs/2026-08-04-agent-browser-design.md"
  - id: "design-agent-browser-takeover-001"
    relation: "references"
    path: "../../superpowers/specs/2026-08-09-agent-browser-takeover-design.md"
---

# Agent 内置浏览器子系统 — 进度存档与接续指南

> **用途**：记录 Agent 内置浏览器全链路的**已完成 / 进行中 / 未开始**工作，以及接续所需的一切上下文（架构决策、不可违反的约束、真机踩坑、仓库/分支/PR 拓扑、恢复入口）。后续任何人（或 AI）据此可无缝接手，无需重读整个会话历史。
>
> 源头设计文档：[[agent-browser-design]]（v2.4，架构定案）。PRD：[[spec-agent-browser-001]]。实现级 spec：[[design-agent-browser-runtime-001]]。

<!-- @section: status -->
## 1. 总体状态一览

| 阶段 / 工作 | 内容 | 状态 | PR / 归属 |
|---|---|---|---|
| **Phase 1** 最小闭环 | open/read/click/type + a11y observation + 单进程多 Context | ✅ 已合 master | server #68（2e8a3a9）|
| **Phase 2** 流式观测 | SSE `/v1/browser/sessions/{id}/stream`（observation/frame/progress）+ Hub + Last-Event-ID + screencast | ✅ 已合 | server #69（c48d2ac）|
| **Phase 3** 会话持久化 | storageState(cookies)落 SQLite + TTL reaper + 重建 + 启动恢复 | ✅ 已合 | server #70（5872d83）|
| **Phase 4A+4B** PAL + loopback 加固 | 平台抽象层三平台 + Chromium 分发 + 一次性 token + Origin guard + 握手 | ✅ 已合 | server #72（f7416a7）|
| **Phase 4C/4D 后端** | ServeResult 暴露 Token + chromium 测试 PAL 可移植 + 三平台 CI 矩阵 | ✅ 已合 | server #73（471371b）|
| **Phase 4C GUI** | GUI bearer 回归修复（SSE/apiGet 带 token）+ GetBrowserEndpoint | ✅ 已合 | gui #14 |
| **浏览器视图 后端事件** | browser_open/close 发 `browser:session_*` 到 /v1/events | ✅ 已合 | server #74（3620a6e）|
| **浏览器视图 UI** | React canvas 渲染 screencast + 观测树 + 只读 | ✅ 已合 | gui #15（5576948）|
| **接管模式（Phase 7）** | 人工回注鼠标/键盘到 go-rod 会话 + 暂停 Agent | ✅ **已交付，双 PR 开启待合** | plan=`2026-08-10-agent-browser-takeover.md`；server PR #77 / gui PR #20 |
| **Phase 5** 安全基线 | 真沙箱 WrapWithSandbox + SSRF DNS-rebinding 完整 + SSE token 吊销 | ⬜ 未开始 | — |
| **Phase 6** 资源与进程池 | 进程池扩缩容 + Reap + 内存采样(PSAPI/proc/task_info) + ResourcePolicy 压测 | ⬜ 未开始 | — |
| **4C 前端产品化** | 浏览器视图更完整 UI（当前是 status 栏 tab 切换的最小实现） | ⬜ 未开始 | — |
| **Wails App 三平台真机打包** | 内置捆绑 Chromium 进 App 资源 + 三 OS 真机 wails build | ⬜ 未开始 | — |
| set-of-marks 视觉标注 | 观测切 screenshot + 元素标注框 | ⬜ 未开始（分叉）| — |
| WebRTC 帧升级 | 高保真高帧率时替换 base64-SSE（Steel.dev 式，传输层直换）| ⬜ 未开始（分叉）| — |

**两仓 master 现状**：server（`github.com/jxncyjq/stardust-agent-server`）tip = **3620a6e**；GUI（`github.com/jxncyjq/stardust-agent-gui`）tip = **5576948**。两仓 build + test 全绿；三平台 CI 矩阵绿。

<!-- @end-section -->

<!-- @section: completed -->
## 2. 已完成工作详情（可运行、已合入、有测试）

### Phase 1 — 最小闭环
- 新包 `legionAgent/internal/browser/`：`api.go`(RuntimeAPI)、`session.go`(SessionStore + 会话锁)、`manager.go`(单进程 incognito Context)、`observation.go`(a11y→稳定 ref + 预算裁剪，纯函数)、`errors.go`(语义错误码)、`runtime.go`(go-rod 驱动)。
- 工具层 `internal/tool/browser.go`：`RegisterBrowserTools` 照 `RegisterWebTools` 抄；九工具 Sensitive 判定（open/click/type=true 触发 Manual gate，read/close=false）；全登记 `toolauth/catalog.go` 的 gateable。
- **review 修的真 bug**（都在 master）：观测 fail-loud（observe 返 error）、共享 Runtime 防 Chrome 泄漏、**ref→元素用 BackendDOMNodeID 精确映射**（原按序取第 n 个会点错元素）、失效 session→CONTEXT_EVICTED、重导航关旧 page。

### Phase 2 — 流式观测
- `stream.go`：每会话 Hub（广播 + 单调 seq + status 环形缓冲；frame 不缓可丢）。`RuntimeAPI.Subscribe`。动作发 observation+progress。
- `screencast.go`：go-rod CDP `Page.startScreencast` → 帧入 Hub，仅有订阅者时开、限帧率~8fps。
- server `browser_stream.go`：SSE handler，`event:`/`id:`/`data:`，Last-Event-ID 补发。`BrowserStreamer` 接口（Subscribe+ReplaySince，仅 `*Runtime` 满足，不进 RuntimeAPI 避免污染 fake）。
- **review 修**：per-session 生命周期锁修 screencast 起停 **TOCTOU 竞态**（SubscriberCount 检查与 Subscribe 非原子）。

### Phase 3 — 会话持久化
- SQLite `browser_sessions` 表（迁移 v7→v8）+ **字段级 Touch**（只动 url/时间不碰 storage_state）。`internal/browser/store.go` 端口（nil=纯内存）。cookie 抓/恢复（go-rod GetCookies/SetCookies）。`reaper.go` TTL 空闲回收。cli 适配器桥 SQLite。
- **review 修**：reaper 与动作对 `sess.Context/ActivePage` 竞态→**全部读写入会话锁**（-race 验证）；`LastUsedAt` 动作刷新改 **idle-based** 回收（原 age-based 会杀活跃会话）；Close 前逐会话落盘。

### Phase 4A+4B — PAL + loopback 加固
- `platform.go`(PlatformAdapter 接口) + `platform_{windows,linux,darwin}.go`（build tag，各 `newPlatformAdapter`）。除 PAL 文件外全包**零 `runtime.GOOS`**。
- `chromium_dist.go`：分发优先级 **config BinPath > 内置捆绑 > 系统 Chrome/Edge > go-rod 下载**。Manager 接 PAL 定位 + `l.HeadlessNew`。
- loopback 加固：`loopback_auth.go` 一次性 token(crypto/rand 256bit，尊重显式 AdminToken) + Origin guard 中间件 + 握手文件 0600。触发条件=`guiDefaultAddr(127.0.0.1:0) || cfg.Server.LoopbackHardening`。
- **占位（后续 Phase 填）**：`WrapWithSandbox` 透传 no-op（真沙箱 Phase 5）、`SampleProcessMemory`/`AvailableSystemMemory` 返回 0（Phase 6/8）、`BundledChromiumPath` 字段就位待打包填。

### Phase 4C/4D — token 暴露 + CI 矩阵 + GUI 接入
- server：`cli.ServeResult` 加 `Token`/`BaseURL`（in-process GUI 消费者拿 mint 的 token）。`systemChromeForTest()` 改用 PAL（可移植）。`.github/workflows/browser-matrix.yml` 三平台（ubuntu/macos/windows）跑 chromium e2e。
- **CI 抓真 Linux bug**：ubuntu Chromium `ZygoteHostImpl::Init` 崩（无 user namespaces）→ linux `DefaultLaunchArgs` 加 `--no-sandbox --disable-dev-shm-usage`（commit 376e669），三平台绿。
- GUI（gui#14）：serve_manager 捕获 Token、SSE bridge + apiGet 带 Bearer（修 4B 加固导致的 GUI 全 403 回归）、`GetBrowserEndpoint()` 绑定。**review 修**：port/token 由 `sync.RWMutex` 守（token 跨 goroutine 撕裂读）。

### 浏览器视图 UI
- server（#74）：`tool.BrowserEventSink` 端口 + browser_open/close 发事件；`cli.platformBrowserEventSink` 镜像 approval_sink → `EventBus.Publish` `browser:session_opened/closed`。
- GUI（#15）：`sse_bridge` 转发 `browser:session`；React `browserStore`(zustand) + `sseReader`(**fetch+ReadableStream 带 Bearer**，因 EventSource 不能设 header) + `useBrowserSession`/`useBrowserStream` + `BrowserView`(canvas drawImage)。App status 栏 tab 切换 StatusPanel/BrowserView。只读。
- **review 修**：干净流 EOF 结束时紧循环重连→success 路径加 2s backoff；Image onload 竞态 cancelled 标志；event id NaN 守卫；sseReader 尾 buffer flush。前端 vitest 130 passed。

<!-- @end-section -->

<!-- @section: inprogress -->
## 3. 进行中 / 下一步：接管模式（Phase 7）

**状态**：✅ **已交付**（2026-08-10）。plan=`docs/superpowers/plans/2026-08-10-agent-browser-takeover.md`；实现经 subagent-driven（每任务 TDD + review）；双 PR 待合：**server #77**（B1-B7，9 commits，chromium 真跑注入 click 触发 onclick + -race 净）、**gui #20**（F1-F3，216 tests 绿）。

**接续入口（合前/合后）**：① 两 PR CI 绿 + 手动 `wails dev` 验证接管闭环后合（server 先）。② 跟进 Minor（记入 PR 正文）：修饰键契约（Ctrl+C 丢 chord，单字符走 char）→ 定 GUI↔后端 key 事件契约后决定是否补 `namedKeys`（Shift/Ctrl/Alt/F1-12）；`InjectInput` 错误码语义细化；`/input` 未知会话 400 vs `/takeover` 404 一致性。③ review 已抓修的真 bug：reaper/Close 忽略 takeover→接管中会话被 idle 回收（reaper 跳过接管会话修）；key 可映射性注入时才查→违反整批拒（移到校验期预检）。

**核心设计（已定，写 plan 直接用）**：
- 跨两仓。后端：Session 加 `takeover bool`（会话锁）；写动作(open/click/type/scroll/back/forward)遇 takeover 返回新错误码 `SESSION_UNDER_TAKEOVER`（read/screenshot 放行）；新方法 `SetTakeover`/`InjectInput`；HTTP `POST /v1/browser/sessions/{id}/takeover` + `.../input`（bearer 守，`BrowserStreamer` 接口扩展加这两方法，仅 `*Runtime` 满足）。
- `InputEvent` **归一化坐标 0..1**，后端 × 视口 px，go-rod Mouse/Keyboard 注入。事件类型白名单：mousemove/down/up/click/wheel/keydown/keyup/char（实用集，不做拖拽）。
- **输入严格校验**（学 hermes-desktop）：坐标范围/类型白名单/button/key/text 长度/批量条数上限，越界整批拒（fail-loud）。
- 前端：BrowserView 加接管开关；canvas pointer-events+tabIndex 捕获鼠标/键盘→映射归一化→节流批量 POST；"接管中"横幅。
- **决定性约束**：接管**必须注入 Agent 真实的 go-rod 会话本身**（同 cookie 同页面态），不是另开浏览器——否则接管的是错误实例（详见 spec §1 决策记录）。

**已否决的替代方案（别再走）**：前端嵌原生 webview。理由（有据）：Wails v2 无多 webview 原语；WKWebView/WebKitGTK 无 CDP；**go-webview2（`pkg/edge/chromium.go`）有 Embed/Navigate/ExecuteScript 但无 `CallDevToolsProtocolMethod`（无 CDP）**；分离 WebView2 ≠ Agent 会话（对象错误）；行业(Browserbase/Steel/OpenAI Operator)全是 stream+注入。

<!-- @end-section -->

<!-- @section: notstarted -->
## 4. 未开始工作（按优先级/依赖）

- **Phase 5 安全基线**：`WrapWithSandbox` 真实现（Win AppContainer+Job Object `KILL_ON_JOB_CLOSE`、mac App Sandbox、Linux namespaces+seccomp）；SSRF **DNS-rebinding 完整**（当前只协议白名单+LookupIP 私网+IsUnspecified，缺 resolve→Chromium 二次 resolve 的 TOCTOU 防护）；SSE 长连 token 过期/吊销（`event: reauth`）。
- **Phase 6 资源与进程池**：进程池扩缩容 + 健康检查/`Reap`（pal.KillProcess 就位）；内存采样真实现填 PAL 占位（PSAPI/proc/task_info）；两种 `ResourcePolicy`（App vs 服务）压测。
- **浏览器视图产品化**：当前是 status 栏 tab 切换的最小挂载；独立布局位/多视图需调 `ThreePanelLayout`。多 tab 视图。
- **Wails App 三平台打包**：`BundledChromiumPath` 填内置固定版 Chromium（150~300MB/平台）；三 OS 真机 `wails build` + 装 App 验证（本机 Windows 无法全验，靠 CI 矩阵覆盖逻辑）。
- **分叉点**：set-of-marks 元素标注（go-rod `page.Eval` 注 JS 覆盖层，类比 hermes 隔离世界选元素）；WebRTC 帧升级（传输层直换）；WebView2-CDP Windows-only 高保真（需 CDP 绑定，本 go-webview2 无；永非主路）；纯抽取退化模式；多租户强隔离/计费。

<!-- @end-section -->

<!-- @section: constraints -->
## 5. 不可违反的约束（接手必读）

1. **fail-loud 铁律**（见 legionAgent/CLAUDE.md）：禁兜底/fallback/静默吞错；错误 `%w` 包装 + 边界 Warn/Error 记录。observe 曾吞 AX 错误返回假观测→被 review 打回。
2. **drift-guard**：`internal/runtime/toolauth_drift_test.go` 的 `TestEveryProductionToolIsGateable` 断言"生产注册工具 ⊆ toolauth.gateable"。**加新浏览器工具必须同步登记 `internal/toolauth/catalog.go` gateable + drift helper**，否则测试必红。
3. **PAL 外零 GOOS**：`internal/browser` 除 `platform_*.go` 外禁止 `runtime.GOOS` 分支；OS 差异全走 PAL。
4. **API 优先、部署无关**：核心一套，支持 dev/服务(无头 Linux)/App 三模式。**任何依赖 GUI/显示的东西不能进核心**（这是否决 native webview 的根因之一）。服务模式无显示无 webview。
5. **写穿字段级 + 先落盘后释放**（[[legion-task-state-persistence]]）：持久化 Touch 不能全行 UPSERT 清 storage_state；回收先落盘登录态再释放 Context。
6. **状态字段全读写同一把锁**：Context/ActivePage/takeover 等跨 goroutine 读的字段必须会话锁下；起停型资源(screencast)靠"查计数再动作"必 TOCTOU，用 per-session 生命周期锁串行整个决策。
7. **加固/鉴权改动必审所有 in-process 消费者**：4B 只测后端，漏了 GUI 这个 in-process serve 消费者 → 合 master 即 GUI 全 403。
8. **BrowserStreamer 接口只 `*Runtime` 满足**（Subscribe/ReplaySince/未来 SetTakeover/InjectInput 不进 RuntimeAPI 接口，避免给三个 fake 加负担）。
9. **接管注入 = Agent 真实 go-rod 会话**，非另开实例（语义正确性）。

<!-- @end-section -->

<!-- @section: gotchas -->
## 6. 真机 / 测试踩坑（省时间）

- **go-rod 自动下载的 Chromium 在本机 Windows 坏**：解压后 `rename chrome-win` Access denied + `side-by-side configuration incorrect`（缺 VC++ 运行库）。**解法**：`Browser.BinPath` 或 PAL `ResolveChromiumPath()` 指系统 Chrome（`C:\Program Files\Google\Chrome\Application\chrome.exe`）。chromium 集成测试现已用 PAL 自动定位。
- **CI Linux Chromium 需 `--no-sandbox`**：runner 无 user namespaces，zygote 沙箱崩。已在 linux `DefaultLaunchArgs` 加 `--no-sandbox --disable-dev-shm-usage`。
- **集成测试 build tag**：chromium 相关走 `//go:build chromium`；普通 `go test ./...` 跳过（CI 无 Chromium 也全绿）。跑真机：`go test -tags chromium ./internal/browser/ ./internal/server/`。
- **-race 本机 Windows 可跑**（CGO/gcc present）：`go test -race ./internal/browser/` 通过；GUI 包 `go test -race .` 也通过。旧 memory 说"-race 用 WSL"对本浏览器工作**不适用**。
- **gopls 诊断常滞后**：subagent 编辑后 `<new-diagnostics>` 常报"undefined"假阳性；以 `go build`/`go test` 实跑为准。
- **前端**：`cd frontend && npx vitest run`（130 passed）、`npx tsc --noEmit`、`npm run build`。首跑偶发 miscount 是 vitest worker 一次性，重跑稳定。

<!-- @end-section -->

<!-- @section: map -->
## 7. 仓库/文件/符号地图

**三个独立 git 仓库**（[[legion-git-repo-topology]]）：
- `docs`（顶层 Legion/docs）：文档。spec/plan 在 `docs/superpowers/specs|plans/`；本存档在 `docs/design/architecture/`。
- `legion/legionAgent` → `github.com/jxncyjq/stardust-agent-server`：后端。
- `legion/legionAgentGUI` → `github.com/jxncyjq/stardust-agent-gui`：GUI（Wails+React）。经 go.work + replace 编译本地 legionAgent。

**关键文件**：
- 后端核心：`legionAgent/internal/browser/{api,session,manager,observation,errors,runtime,stream,screencast,store,reaper,chromium_dist,platform,platform_windows,platform_linux,platform_darwin}.go`
- 工具/鉴权：`internal/tool/browser.go`、`internal/toolauth/catalog.go`、`internal/runtime/toolauth_drift_test.go`
- server 传输：`internal/server/{browser_stream,loopback_auth}.go`、`http.go`(路由)、`internal/cli/{browser_store,browser_event_sink}.go`、`command.go`(装配)
- CI：`.github/workflows/{agent-ci.yml, browser-matrix.yml}`
- GUI Go：`serve_manager.go`、`sse_bridge.go`、`app.go`（GetBrowserEndpoint）
- GUI React：`frontend/src/{stores/browserStore.ts, lib/sseReader.ts, hooks/useBrowserSession.ts, hooks/useBrowserStream.ts, components/BrowserView.tsx}`

**文档**：
- 架构 v2.4：[[agent-browser-design]]（`docs/design/architecture/agent-browser-design.md`）
- PRD：[[spec-agent-browser-001]]、实现 spec：[[design-agent-browser-runtime-001]]
- Phase 1-4/视图/接管 plan+spec：`docs/superpowers/{plans,specs}/2026-08-0x-agent-browser-*.md`
- memory：`agent-browser-design-prd-spec-plan`（会话记忆，含所有 PR/踩坑/教训）

<!-- @end-section -->

<!-- @section: resume -->
## 8. 恢复入口（想接着做，从这看）

- **做接管模式** → 读接管 spec [[design-agent-browser-takeover-001]] → `/superpowers:writing-plans` 出 plan（跨两仓，参照浏览器视图 plan 的两仓两 PR 范式）→ subagent-driven 执行 → 每 phase code review 必抓真 bug → 双 PR。
- **做 Phase 5 安全** → 起点=PAL `WrapWithSandbox` 占位 + `runtime.go` `checkURL`（SSRF）+ `loopback_auth`（token 吊销）。brainstorming 先。
- **做 Phase 6 资源** → 起点=`manager.go`(单进程，待扩池) + PAL 内存采样占位 + Reap（pal.KillProcess 就位）。
- **执行范式（固化）**：brainstorming(有开放决策时) → spec → writing-plans → subagent-driven TDD（纯函数普通 test，go-rod 走 -tags chromium，前端 vitest）→ code review(必抓真 bug) → 修复(-race/真机验证) → PR → CI/真机 → 合入。跨两仓 = 两分支两 PR，后端先合。

<!-- @end-section -->

## 相关文档

- [[agent-browser-design]] — 架构设计 v2.4（定案来源）
- [[spec-agent-browser-001|PRD]] · [[design-agent-browser-runtime-001|实现 spec]] · [[design-agent-browser-view-ui-001|视图 UI spec]] · [[design-agent-browser-takeover-001|接管 spec]]
- [[legion-git-repo-topology]] — 三仓拓扑 · [[legion-task-state-persistence]] — 写穿教训 · [[legion-per-agent-tool-auth]] — toolauth 门控
