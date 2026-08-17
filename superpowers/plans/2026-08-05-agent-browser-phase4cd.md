# Agent 内置浏览器 Phase 4C+4D（GUI loopback 握手接入 + 三平台 CI 矩阵）Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking. **本 plan 跨两个 git 仓库**——每个 task 标注所属仓库与分支，分别提交/开 PR。

**Goal:** 让 Wails GUI 在 Phase 4B 加固后的 loopback serve 上正常工作（回归修复：GUI 的 SSE bridge 与 HTTP 调用现在缺 bearer token 会被 403），并把 `{baseURL, token}` 握手暴露给前端；同时把 chromium 测试改为平台可移植（经 PAL 定位）并建三平台 CI 矩阵，让 open/click/type/screenshot e2e 在 Win/mac/Linux 各跑一遍。

**Architecture:** Phase 4B 让 GUI-default 的 `127.0.0.1:0` 绑定自动 mint 一次性 token 并要求 bearer；但 `cli.ServeResult` 未暴露该 token，GUI 的 `sse_bridge.go`（`/v1/events`）与 `app.go` 的 `apiGet` 都不带 token → **对加固 serve 全部 403**。修法：(A) legionAgent 的 `ServeResult` 暴露 `Token`/`BaseURL`；(B) GUI 的 `serve_manager` 捕获 Token，`sse_bridge`+`apiGet` 带 `Authorization: Bearer`，`app.go` 加 `GetBrowserEndpoint()` Wails 绑定方法把 `{baseURL, token}` 交前端（前端据此直连 Phase 2 的 `/v1/browser/sessions/{id}/stream`）。4D：把 chromium 测试的 `systemChromeForTest()` 改用 `NewPlatformAdapter().ResolveChromiumPath()`（4A 成果），再加 CI 矩阵 job 在三平台跑 chromium e2e。

**Tech Stack:** Go 1.26、Wails v2（`runtime.EventsEmit`）、go-rod、GitHub Actions（现有 `.github/workflows/agent-ci.yml`，单 ubuntu）。**go.work + replace** 使 GUI 直接编译本地 `../legionAgent`，故 ServeResult 改动对 GUI 立即可见。

**跨仓库分支约定:**
- legionAgent（`github.com/jxncyjq/stardust-agent-server`）：新分支 `feat/browser-phase4cd`，基 master。含 Task 1/2/3。→ PR。
- legionAgentGUI（`github.com/jxncyjq/stardust-agent-gui`）：新分支 `feat/browser-loopback-auth`，基 master。含 Task 4/5/6。→ PR。**先合 legionAgent PR**（GUI 经 replace 用本地路径不需发布，但逻辑上 GUI 依赖 ServeResult.Token）。

**范围边界（本 plan 不做）:** 前端 React 的浏览器视图 UI（`<canvas>` 渲染 screencast 帧）= GUI 产品工作，本 plan 只到"暴露 endpoint 给前端"的绑定方法 + 验证它返回正确数据；Wails App 三平台打包与真机 App 运行 = 需三 OS 真机，本 plan 靠 CI 矩阵 + Go 单测覆盖逻辑；内置捆绑 Chromium 进 App 资源 = 打包工程，`BundledChromiumPath` 字段已就位待填。

**关联文档:** spec §3.4（握手）、§11.5（三平台 CI）；Phase 4A+4B plan（`2026-08-05-agent-browser-phase4.md`）；[[legion-gui-wails-gotchas]]（GUI 启动/config/走 Go 绑定避 CORS）。

---

## 背景：Phase 4B 引入的 GUI 回归（本 plan 首要修复）

`internal/cli/command.go` 的 `BuildServeService`（GUI 经 `serve.BuildService` 调它）在 `guiDefaultAddr && isLoopbackAddr` 时 `loopbackHardening=true` → mint 一次性 token 设为 `AdminToken` → `authorized()`（http.go route 前单一 gate）要求所有请求带 `Authorization: Bearer <token>`。但：
- `cli.ServeResult` 只有 `{Service, Listener, Close}`——**不暴露 token**；
- GUI 的 `sse_bridge.go consumeSSE` 与 `app.go apiGet` 都不带 token。

结果：**master 上 GUI 的 SSE 事件流与所有 HTTP 列表调用（ListSessions/ListTasks/ListRuntimeEvents）都会 403**。Task 1 从源头暴露 token，Task 4/5 让 GUI 用上它。

---

## 文件结构

| 仓库 | 文件 | 职责 | Task |
|---|---|---|---|
| legionAgent | `internal/cli/command.go` | `ServeResult` 加 `Token`/`BaseURL`，`BuildServeService` 填充 | 1 |
| legionAgent | `internal/browser/*_integration_test.go` 等 | `systemChromeForTest()` 改用 PAL `ResolveChromiumPath` | 2 |
| legionAgent | `.github/workflows/agent-ci.yml`（或新 `browser-matrix.yml`） | 三平台 chromium e2e 矩阵 job | 3 |
| legionAgentGUI | `serve_manager.go` | 捕获 `result.Token`，`Token()` 访问器 | 4 |
| legionAgentGUI | `sse_bridge.go` · `app.go` | SSE bridge + apiGet 带 bearer；token 注入 | 5 |
| legionAgentGUI | `app.go` | `GetBrowserEndpoint()` Wails 绑定方法 | 6 |

---

## Task 1 [legionAgent] — ServeResult 暴露 Token/BaseURL

**Repo:** legionAgent（分支 `feat/browser-phase4cd`）
**Files:**
- Modify: `internal/cli/command.go`（`ServeResult` 结构 + `BuildServeService` 填充）
- Test: `internal/cli/serve_token_test.go`

- [ ] **Step 1: 写失败测试（GUI-default 模式返回非空 Token）**

Create `internal/cli/serve_token_test.go`:

```go
package cli

import (
	"context"
	"strings"
	"testing"
)

// GUI 默认绑 127.0.0.1:0 时应自动 mint token 并在 ServeResult 暴露它。
func TestBuildServeServiceExposesLoopbackToken(t *testing.T) {
	res, err := BuildServeService(context.Background(), ServeOptions{Addr: "127.0.0.1:0", ConfigPath: ""})
	if err != nil {
		t.Fatalf("BuildServeService: %v", err)
	}
	defer res.Close()
	if res.Token == "" {
		t.Fatal("expected non-empty loopback Token exposed on ServeResult")
	}
	if len(res.Token) < 32 {
		t.Fatalf("token too short: %q", res.Token)
	}
	if !strings.HasPrefix(res.BaseURL, "http://127.0.0.1:") {
		t.Fatalf("BaseURL = %q, want http://127.0.0.1:<port>", res.BaseURL)
	}
}
```

> 若 `BuildServeService` 需要一个可用 config 才能构造（config.Load 对空路径的行为），先 `grep -n "func BuildServeService\|config.Load" internal/cli/command.go` 看它对空 ConfigPath 的处理；若空路径会失败，测试改用 `t.TempDir()` 下写一个最小 config，或用已有测试构造 serve 的 helper（`grep -rn "BuildServeService(" internal/cli/*_test.go`）。

- [ ] **Step 2: 跑，确认失败**

Run: `go test ./internal/cli/ -run TestBuildServeServiceExposesLoopbackToken -v`
Expected: FAIL（`ServeResult` 无 `Token`/`BaseURL` 字段）。

- [ ] **Step 3: 实现**

Edit `internal/cli/command.go`：

`ServeResult` 加字段：
```go
type ServeResult struct {
	Service  *service.Service
	Listener net.Listener
	Close    func()
	// Token 是 loopback 加固模式下 mint 的一次性 bearer token（未加固时为空）。
	// in-process 消费者（Wails GUI）据此带 Authorization: Bearer 自连。
	Token string
	// BaseURL 是服务实际监听的地址（含随机端口），供 in-process 消费者与握手用。
	BaseURL string
}
```

在 `BuildServeService` 里，拿到 `listener` 与 `adminToken` 后（`adminToken` 变量已存在，见 command.go:2170 附近；`listener.Addr()` 已用于握手 baseURL），在构造返回的 `ServeResult{...}` 处填：
```go
	baseURL := "http://" + listener.Addr().String()
	// ...existing ServeResult construction...
	return ServeResult{
		Service:  svc,
		Listener: listener,
		Close:    closeFn,
		Token:    adminToken, // 加固时为 mint 的一次性 token；未加固/显式配置时为该值
		BaseURL:  baseURL,
	}, nil
```
（复用握手已算出的 `baseURL`；若握手分支里已有 `baseURL` 变量，提升其作用域到返回处共用，避免重复拼。`adminToken` 在非加固模式可能为空串——那也正确地表示"无需 token"。）

- [ ] **Step 4: 跑，确认通过**

Run: `go test ./internal/cli/ -run TestBuildServeServiceExposesLoopbackToken -v`
Expected: PASS。也 `go build ./...` + `go test ./internal/cli/ 2>&1 | tail -3` 确认不回归。

- [ ] **Step 5: Commit**

```bash
git add internal/cli/command.go internal/cli/serve_token_test.go
git commit -m "feat(cli): expose loopback token and baseURL on ServeResult"
```
（附空行 + `Co-Authored-By: Claude Opus 4.8 <noreply@anthropic.com>`。）

---

## Task 2 [legionAgent] — chromium 测试改用 PAL 定位（可移植）

**Repo:** legionAgent（同分支 `feat/browser-phase4cd`）
**Files:**
- Modify: chromium-tagged 测试里的 `systemChromeForTest()` 定义处（Phase 2/3 引入，`grep -rln "systemChromeForTest" internal/`）

- [ ] **Step 1: 定位现有 helper**

Run:
```
grep -rn "func systemChromeForTest\|systemChromeForTest()" internal/
grep -rn "C:\\\\Program Files\\\\Google" internal/ | grep -i test
```
找到 `systemChromeForTest()` 的定义（可能在 browser 与 server 各一份 chromium-tagged 文件）与所有硬编码 Windows Chrome 路径。

- [ ] **Step 2: 改用 PAL ResolveChromiumPath**

把 `systemChromeForTest()` 的实现改为经 PAL 定位（跨平台）：
```go
// systemChromeForTest 返回本机可用的 Chromium 路径（跨平台，经 PAL 定位）。
// 返回 "" 表示让 go-rod launcher 自动下载（CI runner 一般预装 Chrome，PAL 能找到）。
func systemChromeForTest() string {
	return NewPlatformAdapter().ResolveChromiumPath() // browser 包内直接调；server 包内用 browser.NewPlatformAdapter()
}
```
- browser 包内的那份：`NewPlatformAdapter().ResolveChromiumPath()`。
- server 包内（Phase 2 e2e 若有独立 helper 或硬编码路径）：`browser.NewPlatformAdapter().ResolveChromiumPath()`，需 `import ".../internal/browser"`（e2e 已 import browser）。
把测试里所有硬编码 `C:\Program Files\Google\Chrome\...` 常量改为调用 `systemChromeForTest()`（含 Phase 3 的 `zzBin`/`TestRebuild...` 若有硬编码，统一）。

> **注意**：`ResolveChromiumPath()` 在本机 Windows 返回系统 Chrome/Edge，在 CI 的 ubuntu/macos 返回各自系统 Chrome（runner 预装）。若返回 ""（没探到），测试会走 go-rod 下载——CI 上可能慢或失败，故 CI job（Task 3）显式安装 Chrome 保证 ResolveChromiumPath 命中。

- [ ] **Step 3: 本机验证仍过**

Run: `go test -tags chromium ./internal/browser/ -run 'TestManagerAcquireReleaseContext|TestScreencastEmitsFrames' -v 2>&1 | grep -vE "Progress|Download" | tail -6`
Expected: PASS（本机经 PAL 定位系统 Chrome，与之前硬编码等效）。

- [ ] **Step 4: Commit**

```bash
git add internal/browser/ internal/server/  # 只 add 实际改动的 chromium-tagged 测试文件
git commit -m "test(browser): resolve Chromium via PAL in integration tests (portable)"
```
（用 `git status` 确认只 add 了改动的测试文件，never `-A`。）

---

## Task 3 [legionAgent] — 三平台 CI 矩阵 job

**Repo:** legionAgent（同分支 `feat/browser-phase4cd`）
**Files:**
- Create: `.github/workflows/browser-matrix.yml`

- [ ] **Step 1: 写矩阵 workflow**

Create `.github/workflows/browser-matrix.yml`:

```yaml
name: Browser Matrix

on:
  push:
    branches: [main, master]
    paths:
      - "internal/browser/**"
      - "internal/server/**"
      - ".github/workflows/browser-matrix.yml"
  pull_request:
    paths:
      - "internal/browser/**"
      - "internal/server/**"
      - ".github/workflows/browser-matrix.yml"
  workflow_dispatch:

jobs:
  browser-e2e:
    name: Browser e2e (${{ matrix.os }})
    strategy:
      fail-fast: false
      matrix:
        os: [ubuntu-latest, macos-latest, windows-latest]
    runs-on: ${{ matrix.os }}
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-go@v5
        with:
          go-version-file: go.mod
          cache-dependency-path: go.sum
      - name: Setup Chrome
        uses: browser-actions/setup-chrome@v1
      - name: Cross-compile check
        run: go build ./internal/browser/...
      - name: Browser unit tests
        run: go test ./internal/browser/ ./internal/server/
      - name: Browser chromium e2e
        run: go test -tags chromium ./internal/browser/ ./internal/server/ -run "TestE2E|TestRuntime|TestScreencast|TestRebuild|TestCaptureRestore|TestBrowserStreamE2E|TestManager" -count=1
```

> 要点：`browser-actions/setup-chrome@v1` 在三平台装 Chrome 并加入 PATH，`PlatformAdapter.ResolveChromiumPath()` 的探测路径应能命中（若 runner 的 Chrome 装在非标准路径导致探不到，退路：该 action 输出 chrome 路径，可用 `env`/`ROD_...` 或给测试传路径——本期先靠标准路径探测，探不到则该平台的 chromium e2e 用 go-rod 下载兜底）。`fail-fast: false` 让三平台各自独立报结果。

- [ ] **Step 2: 本地 lint（yaml 合法性）**

Run（无 act/actionlint 时至少 yaml 解析）:
```
python -c "import yaml,sys; yaml.safe_load(open('.github/workflows/browser-matrix.yml')); print('yaml ok')"
```
Expected: `yaml ok`。（真正的 job 运行只能在 push 后于 GitHub Actions 验证——本 Task 不能本地跑 CI。）

- [ ] **Step 3: Commit**

```bash
git add .github/workflows/browser-matrix.yml
git commit -m "ci(browser): three-platform chromium e2e matrix"
```

- [ ] **Step 4: 开 legionAgent PR**（本 Task 后，Part A 三 task 都在分支上）

```bash
git push -u origin feat/browser-phase4cd
gh pr create --base master --head feat/browser-phase4cd --title "feat(browser): expose loopback token on ServeResult + portable chromium tests + 3-OS CI matrix" --body "<摘要：修复 4B 引入的 GUI token 回归的后端侧；chromium 测试经 PAL 可移植；三平台 CI 矩阵>"
```
> CI 矩阵 job 会在此 PR 上首次实跑——观察三平台结果，红则据实修（可能是 setup-chrome 路径探测问题，回到 Task 2 的 ResolveChromiumPath 或给测试注入路径）。

---

## Task 4 [legionAgentGUI] — serve_manager 捕获 Token

**Repo:** legionAgentGUI（新分支 `feat/browser-loopback-auth`，基 master）
**Files:**
- Modify: `serve_manager.go`
- Test: `serve_manager_test.go`（若无则新建；参考现有 `serve_restart_test.go` 构造方式）

- [ ] **Step 1: 切 GUI 仓库 + 建分支**

```bash
cd F:/source/stardust/Legion/legion/legionAgentGUI
git checkout -b feat/browser-loopback-auth
```

- [ ] **Step 2: 写失败测试**

在 `serve_manager_test.go`（新建或追加）:

```go
package main

import (
	"context"
	"testing"
)

func TestServeManagerExposesToken(t *testing.T) {
	m := NewServeManager()
	if err := m.Start(context.Background(), context.Background(), ""); err != nil {
		t.Fatalf("Start: %v", err)
	}
	defer m.Stop()
	if m.Token() == "" {
		t.Fatal("expected non-empty Token after Start (loopback hardening mints one)")
	}
}
```
> `Start` 的签名以现有 `serve_manager.go` 为准（`grep -n "func (m \*ServeManager) Start" serve_manager.go`）——上面示例签名按需对齐。若 Start 需要一个可用 config，参考 `serve_restart_test.go` 如何构造。

- [ ] **Step 3: 跑，确认失败**

Run: `go test . -run TestServeManagerExposesToken -v`
Expected: FAIL（`Token()` 未定义）。

- [ ] **Step 4: 实现**

Edit `serve_manager.go`：`ServeManager` 加 `token string` 字段；`Start` 里从 `result` 捕获：
```go
	m.port = listenerPort(result.Listener)
	m.token = result.Token // Phase 4C：加固模式 mint 的一次性 bearer token
```
加访问器（对齐 `Port()`）：
```go
// Token 返回 loopback 加固模式下 mint 的 bearer token（未加固时为空）。
func (m *ServeManager) Token() string { return m.token }
```
`Restart` 后 token 会随新 serve 变化——确保 Restart 也刷新 `m.token`（同 `m.port`）。

- [ ] **Step 5: 跑，确认通过**

Run: `go test . -run TestServeManagerExposesToken -v`
Expected: PASS。`go build .` 干净。

- [ ] **Step 6: Commit**

```bash
git add serve_manager.go serve_manager_test.go
git commit -m "feat(gui): capture loopback token from serve result"
```

---

## Task 5 [legionAgentGUI] — SSE bridge + apiGet 带 bearer（回归修复）

**Repo:** legionAgentGUI（同分支）
**Files:**
- Modify: `sse_bridge.go`（`consumeSSE`/`startSSEBridge` 带 token）
- Modify: `app.go`（`apiGet` 带 token；`StartSSEBridge` 传 tokenFn）
- Test: `sse_bridge_test.go`（追加带 token 用例）

- [ ] **Step 1: 写失败测试（带 token 请求命中受保护端点）**

追加到 `sse_bridge_test.go`:

```go
func TestConsumeSSESendsBearerToken(t *testing.T) {
	var gotAuth string
	srv := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		gotAuth = r.Header.Get("Authorization")
		w.Header().Set("Content-Type", "text/event-stream")
		w.WriteHeader(200)
		_, _ = w.Write([]byte("event: ping\ndata: {}\n\n"))
	}))
	defer srv.Close()

	_ = consumeSSEWithToken(context.Background(), srv.URL, "tok-123", func(string, any) {})
	if gotAuth != "Bearer tok-123" {
		t.Fatalf("Authorization = %q, want Bearer tok-123", gotAuth)
	}
}
```

- [ ] **Step 2: 跑，确认失败**

Run: `go test . -run TestConsumeSSESendsBearerToken -v`
Expected: FAIL（`consumeSSEWithToken` 未定义）。

- [ ] **Step 3: 实现——SSE bridge 带 token**

Edit `sse_bridge.go`：
- `startSSEBridge` 增参 `tokenFn func() string`；`StartSSEBridge` 同步增参并透传。
- `consumeSSE` 改为 `consumeSSEWithToken(ctx, url, token, emit)`（或保留 `consumeSSE` 签名，内部读 tokenFn）；在 `http.NewRequestWithContext` 后，若 `token != ""` 则 `req.Header.Set("Authorization", "Bearer "+token)`。
```go
func consumeSSEWithToken(ctx context.Context, url, token string, emit func(event string, data any)) error {
	req, err := http.NewRequestWithContext(ctx, http.MethodGet, url, nil)
	if err != nil {
		return fmt.Errorf("build SSE request for %s: %w", url, err)
	}
	if token != "" {
		req.Header.Set("Authorization", "Bearer "+token)
	}
	// ...existing consumeSSE body (do request, scan SSE)...
}
```
`startSSEBridge` 的循环里改成 `consumeSSEWithToken(ctx, url, tokenFn(), emit)`（每次重连读当前 token，因 Restart 会换）。

- [ ] **Step 4: 实现——apiGet 带 token**

Edit `app.go`：`apiGet` 用带 Authorization 的请求替代 `a.client.Get`：
```go
func (a *App) apiGet(path string) ([]byte, error) {
	req, err := http.NewRequest(http.MethodGet, a.BaseURL()+path, nil)
	if err != nil {
		return nil, err
	}
	if tok := a.serve.Token(); tok != "" {
		req.Header.Set("Authorization", "Bearer "+tok)
	}
	resp, err := a.client.Do(req)
	// ...existing body read/return...
}
```
并把 `startup` 里 `StartSSEBridge(ctx, ctx, a.BaseURL)` 改为 `StartSSEBridge(ctx, ctx, a.BaseURL, a.serve.Token)`（传 tokenFn，随 Restart 动态读）。

- [ ] **Step 5: 跑，确认通过 + 全量**

Run:
```
go test . -run 'TestConsumeSSESendsBearerToken|TestSSE|TestServeManager' -v
go build . && go test . 2>&1 | tail -8
```
Expected: 新测试 PASS；既有 GUI 测试不回归（尤其现有 sse_bridge_test 的用例——若它们调 `consumeSSE` 旧签名，同步改为新签名或保留兼容包装）。

- [ ] **Step 6: Commit**

```bash
git add sse_bridge.go sse_bridge_test.go app.go
git commit -m "fix(gui): send bearer token on SSE bridge and HTTP calls (loopback hardening)"
```

---

## Task 6 [legionAgentGUI] — GetBrowserEndpoint 绑定方法（前端读握手）

**Repo:** legionAgentGUI（同分支）
**Files:**
- Modify: `app.go`（新增 Wails 绑定方法）
- Test: `app_test.go`（追加）

- [ ] **Step 1: 写失败测试**

追加到 `app_test.go`（参考现有 `app_test.go` 如何构造 `*App`）:

```go
func TestGetBrowserEndpointReturnsBaseURLAndToken(t *testing.T) {
	a := NewApp()
	// 用现有测试构造/启动 serve 的方式让 a.serve 就绪（参考 app_test.go 里其他用例）。
	// 若无法真启动 serve，可用一个注入了已知 port/token 的 ServeManager stub。
	ep := a.GetBrowserEndpoint()
	if ep.BaseURL == "" {
		t.Fatal("BaseURL empty")
	}
	// token 可能为空（未加固）——只断言字段存在与 BaseURL 形状
	if !strings.HasPrefix(ep.BaseURL, "http://127.0.0.1:") {
		t.Fatalf("BaseURL = %q", ep.BaseURL)
	}
}
```
> 若真启 serve 在单测里太重，改为把 `ServeManager` 抽一个小接口（`Port()`/`Token()`）注入 stub；或跳过真启动只测 `GetBrowserEndpoint` 从 `a.serve` 读值的组装逻辑。以现有 `app_test.go` 的构造范式为准。

- [ ] **Step 2: 跑，确认失败**

Run: `go test . -run TestGetBrowserEndpoint -v`
Expected: FAIL（`GetBrowserEndpoint` 未定义）。

- [ ] **Step 3: 实现绑定方法**

Edit `app.go`:
```go
// BrowserEndpoint 是交给前端自连内置浏览器流的握手信息（spec §3.4）。
type BrowserEndpoint struct {
	BaseURL string `json:"baseURL"`
	Token   string `json:"token"`
}

// GetBrowserEndpoint 暴露给前端（Wails 绑定）：前端据此用 fetch/EventSource 直连
// /v1/browser/sessions/{id}/stream（带 Authorization: Bearer token）观看 Agent 浏览。
func (a *App) GetBrowserEndpoint() BrowserEndpoint {
	return BrowserEndpoint{BaseURL: a.BaseURL(), Token: a.serve.Token()}
}
```
（Wails 会自动把导出方法绑定给前端；无需额外注册——确认 `App` 已在 `main.go` 的 `Bind` 列表里。）

- [ ] **Step 4: 跑 + 全量 + 构建**

Run:
```
go test . -run TestGetBrowserEndpoint -v
go build . && go test . 2>&1 | tail -8
```
Expected: PASS；GUI 全量绿；`go build .` 干净（Wails 前端构建不在本 Task——纯 Go 侧）。

- [ ] **Step 5: Commit + 开 GUI PR**

```bash
git add app.go app_test.go
git commit -m "feat(gui): GetBrowserEndpoint binding exposes baseURL+token to frontend"
git push -u origin feat/browser-loopback-auth
gh pr create --base master --head feat/browser-loopback-auth --title "fix(gui): work with hardened loopback serve (bearer token) + expose browser endpoint" --body "<摘要：修复 4B loopback 加固导致 GUI SSE/HTTP 403 的回归；GetBrowserEndpoint 交前端握手>"
```

---

## 验证 Phase 4C+4D DoD

- [ ] **回归修复**：GUI 的 SSE bridge 与 apiGet 都带 bearer token（Task 5 单测）；`ServeResult.Token` 暴露（Task 1）
- [ ] `GetBrowserEndpoint()` 返回 `{baseURL, token}`（Task 6）
- [ ] chromium 测试经 PAL 可移植，本机仍绿（Task 2）
- [ ] 三平台 CI 矩阵 job 存在且 yaml 合法（Task 3）；PR 上三平台实跑结果（push 后观察）
- [ ] legionAgent 全量 `go test ./...` 绿、drift 绿；GUI 全量 `go test .` 绿
- [ ] 两仓各一 PR，base=master

---

## 已知边界与后续

| 项 | 本期 | 后续 |
|---|---|---|
| 前端 React 浏览器视图 UI | 只到 `GetBrowserEndpoint` 绑定 | GUI 产品：`<canvas>` 渲染 screencast + 观测树交互 |
| Wails App 三平台打包 | CI 矩阵跑 e2e + Go 单测覆盖逻辑 | 三 OS 真机 wails build + 装 App 验证 |
| 内置捆绑 Chromium | `BundledChromiumPath` 字段就位 | 打包把固定版 Chromium 放进 App 资源填此路径 |
| CI setup-chrome 路径命中 | 靠 PAL 标准路径探测；探不到走 go-rod 下载 | 若某平台探不到，给测试注入 setup-chrome 输出的路径 |
| 真沙箱/内存采样 | Phase 4A 占位 | Phase 5/6 |
```
