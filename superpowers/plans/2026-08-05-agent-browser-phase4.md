# Agent 内置浏览器 Phase 4A+4B（PAL 平台抽象层 + App-mode loopback 加固）Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 把浏览器运行时的 OS 差异收敛到一个平台抽象层（PAL），三平台行为一致；并给 App 模式的 loopback HTTP 服务补齐"加固四件套"缺口（一次性 token + Origin 校验 + 握手），让 Wails 单 exe 分发时前端能安全自连。

**Architecture:** 新增 `internal/browser` 的 PAL——`platform.go` 定义 `PlatformAdapter` 接口，`platform_{windows,linux,darwin}.go` 用 build tag 各自实现（Chromium 定位/启动参数/进程终止/数据目录/路径转换/安全删除；沙箱与内存采样本期为文档化占位，真实实现留 Phase 5/6）。Browser Manager 改为经 PAL 定位 Chromium（内置→系统→config 兜底）与终止进程，除 PAL 外全包禁止 `runtime.GOOS`。loopback 加固复用现有 GUI 模式的 `127.0.0.1:0` 监听与 `AdminToken` bearer 校验，补上：App/GUI 模式**每次启动生成一次性随机 token**、**Origin/自定义头校验中间件**、**握手**（把 `{baseURL, token}` 写到约定文件供前端读）。

**Tech Stack:** Go 1.26、go-rod launcher、现有 `internal/server`（AdminToken bearer 已在 http.go:338）、现有 GUI 模式 loopback（command.go:2135 `127.0.0.1:0`）。跨平台编译用 `GOOS=linux/darwin go build` 校验（本机 Windows）。

**范围边界（本 plan 不做）:** 真实 OS 沙箱（AppContainer/App Sandbox/seccomp）= Phase 5；内存采样的精确实现（PSAPI/proc/task_info）与 ResourcePolicy = Phase 6/8（本期 `SampleProcessMemory`/`AvailableSystemMemory` 为 best-effort 占位）；Wails webview 宿主与 React 前端握手消费、三平台 App 打包 = 4C（另一仓 legionAgentGUI）；三平台 CI 矩阵 = 4D。

**关联文档:** spec §11（PAL/沙箱/Chromium 分发）、§3.4（loopback 加固四件套）、§14 phase4；Phase 1-3 plan 同目录；[[legion-config-resolution-roots]]（数据目录解析）。

---

## 锁定的设计决策

| 决策 | 选择 | 理由 |
|---|---|---|
| PAL 结构 | `platform.go` 接口 + `newPlatformAdapter()` 各 build-tag 文件提供，`NewPlatformAdapter()` 工厂调用它 | Go 惯例；工厂在所有平台编译 |
| 本期 PAL 方法 | ResolveChromiumPath / DefaultLaunchArgs / KillProcess / AppDataDir / CacheDir / ToNativePath / SafeDelete 做实；SampleProcessMemory / AvailableSystemMemory / WrapWithSandbox 文档化占位 | 聚焦 runtime 现在真需要的；难点留后续 Phase |
| Chromium 分发 | ResolveChromiumPath 顺序：**config BinPath > 内置捆绑路径 > 系统 Chrome/Edge > go-rod 自动下载** | spec §11.4 内置为主+系统兜底；config 覆盖最高（本机 go-rod 下载坏，靠这个绕过） |
| Manager 接 PAL | BinPath 为空时用 `pal.ResolveChromiumPath()`；进程终止走 `pal.KillProcess` | 除 PAL 外无 GOOS 分支 |
| 一次性 token | App/GUI 模式（loopback）且**未配置** AdminToken 时，每启动 `crypto/rand` 生成；已配置则尊重配置 | spec §3.4 随启动轮换；不破坏显式配置 |
| 握手 | 把 `{baseURL, token}` 写到 `--handshake-file`（或会话/数据目录下约定文件），前端读取 | webview 只能连 TCP，需带外交换 endpoint+token |
| Origin 校验 | loopback 模式中间件：校验 `Origin`/自定义头，非法拒绝 | spec §3.4 挡同机其他进程越权 |
| 服务模式不变 | 加固中间件只在 loopback/App 模式启用；服务模式沿用现有鉴权 | 不影响 §10 服务模式 |

---

## 文件结构

| 文件 | 职责 | 状态 |
|---|---|---|
| `internal/browser/platform.go` | `PlatformAdapter` 接口 + `NewPlatformAdapter()` 工厂 | 创建（Task 1） |
| `internal/browser/platform_windows.go` | Windows 实现（`//go:build windows`） | 创建（Task 2） |
| `internal/browser/platform_linux.go` · `platform_darwin.go` | Linux/macOS 实现（build tag） | 创建（Task 3） |
| `internal/browser/manager.go` | 改走 PAL 定位 Chromium + 终止进程 | 修改（Task 4） |
| `internal/browser/chromium_dist.go` | ResolveChromiumPath 的分发优先级逻辑（跨平台部分） | 创建（Task 4） |
| `internal/server/loopback_auth.go` | 一次性 token 生成 + Origin/自定义头中间件 | 创建（Task 5/6） |
| `internal/cli/command.go` | loopback 模式生成 token + 握手写文件 + 挂中间件 | 修改（Task 5/6） |
| `internal/config/config.go` | `Server` 加 loopback 加固开关 / handshake 文件路径 | 修改（Task 5） |

---

## Task 1: PlatformAdapter 接口 + 工厂

**Files:**
- Create: `internal/browser/platform.go`
- Test: `internal/browser/platform_test.go`

- [ ] **Step 1: 写失败测试（当前 OS 工厂返回可用实现）**

Create `internal/browser/platform_test.go`:

```go
package browser

import (
	"strings"
	"testing"
)

func TestNewPlatformAdapterCurrentOS(t *testing.T) {
	pal := NewPlatformAdapter()
	if pal == nil {
		t.Fatal("NewPlatformAdapter returned nil")
	}
	// 数据/缓存目录应非空且是绝对路径样式
	if d := pal.AppDataDir(); d == "" {
		t.Fatal("AppDataDir empty")
	}
	if c := pal.CacheDir(); c == "" {
		t.Fatal("CacheDir empty")
	}
	// ToNativePath 幂等于本平台分隔符
	got := pal.ToNativePath("a/b/c")
	if got == "" || strings.Contains(got, "//") {
		t.Fatalf("ToNativePath bad: %q", got)
	}
	// DefaultLaunchArgs 至少给出一批参数（非 nil）
	if pal.DefaultLaunchArgs() == nil {
		t.Fatal("DefaultLaunchArgs nil")
	}
}
```

- [ ] **Step 2: 跑，确认失败**

Run: `go test ./internal/browser/ -run TestNewPlatformAdapter -v`
Expected: FAIL（`NewPlatformAdapter`/`PlatformAdapter` 未定义）。

- [ ] **Step 3: 实现接口 + 工厂**

Create `internal/browser/platform.go`:

```go
package browser

import "os/exec"

// PlatformAdapter 收敛所有 OS 差异（spec §11.2）。除本文件与 platform_{os}.go 外，
// browser 包禁止出现 runtime.GOOS 分支。
type PlatformAdapter interface {
	// 进程
	ResolveChromiumPath() string              // 定位可执行文件（分发优先级见 chromium_dist.go）
	DefaultLaunchArgs() []string              // 平台相关启动参数
	KillProcess(pid int, graceful bool) error // 信号 vs TerminateProcess

	// 资源（本期 best-effort 占位，Phase 6/8 精化）
	SampleProcessMemory(pid int) uint64
	AvailableSystemMemory() uint64

	// 文件系统
	AppDataDir() string           // XDG / ~/Library / %LOCALAPPDATA%
	CacheDir() string
	ToNativePath(posix string) string
	SafeDelete(path string) error // Windows 强制锁：先关句柄/重试

	// 隔离（本期文档化占位——透传；真实沙箱 = Phase 5）
	WrapWithSandbox(cmd *exec.Cmd) *exec.Cmd
}

// NewPlatformAdapter 返回当前 OS 的实现（各 platform_{os}.go 提供 newPlatformAdapter）。
func NewPlatformAdapter() PlatformAdapter {
	return newPlatformAdapter()
}
```

> `newPlatformAdapter()`（小写）由每个 `platform_{os}.go` 用 build tag 各提供一份，工厂在所有平台编译。

- [ ] **Step 4: 跑（此时仍会因缺 newPlatformAdapter 而编译失败——Task 2/3 补齐后过）**

Run: `go build ./internal/browser/ 2>&1 | head`
Expected: 报 `undefined: newPlatformAdapter`（预期——Task 2 提供 windows 版后本机可编译）。**本 Task 不单独提交**，与 Task 2 一起编译通过后提交（见 Task 2 Step 5）。

---

## Task 2: Windows 实现

**Files:**
- Create: `internal/browser/platform_windows.go`（`//go:build windows`）
- Test: `internal/browser/platform_windows_test.go`（`//go:build windows`）

- [ ] **Step 1: 写失败测试**

Create `internal/browser/platform_windows_test.go`:

```go
//go:build windows

package browser

import (
	"os"
	"path/filepath"
	"strings"
	"testing"
)

func TestWindowsAppDataDir(t *testing.T) {
	pal := newPlatformAdapter()
	d := pal.AppDataDir()
	if d == "" {
		t.Fatal("AppDataDir empty")
	}
	// 应落在 %LOCALAPPDATA% 下（若环境有该变量）
	if la := os.Getenv("LOCALAPPDATA"); la != "" && !strings.HasPrefix(d, la) {
		t.Fatalf("AppDataDir %q not under LOCALAPPDATA %q", d, la)
	}
}

func TestWindowsToNativePath(t *testing.T) {
	pal := newPlatformAdapter()
	got := pal.ToNativePath("a/b/c")
	if got != `a\b\c` {
		t.Fatalf("ToNativePath = %q, want a\\b\\c", got)
	}
}

func TestWindowsSafeDeleteMissingFileOK(t *testing.T) {
	pal := newPlatformAdapter()
	// 删不存在的文件应不报错（幂等）
	if err := pal.SafeDelete(filepath.Join(t.TempDir(), "nope")); err != nil {
		t.Fatalf("SafeDelete missing: %v", err)
	}
	// 删存在的文件应成功
	f := filepath.Join(t.TempDir(), "x")
	_ = os.WriteFile(f, []byte("hi"), 0o644)
	if err := pal.SafeDelete(f); err != nil {
		t.Fatalf("SafeDelete existing: %v", err)
	}
	if _, err := os.Stat(f); !os.IsNotExist(err) {
		t.Fatal("file not deleted")
	}
}
```

- [ ] **Step 2: 跑，确认失败**

Run: `go test ./internal/browser/ -run TestWindows -v`
Expected: FAIL（`newPlatformAdapter` 未定义）。

- [ ] **Step 3: 实现 Windows PAL**

Create `internal/browser/platform_windows.go`:

```go
//go:build windows

package browser

import (
	"fmt"
	"os"
	"os/exec"
	"path/filepath"
	"strings"
	"time"
)

type windowsAdapter struct{}

func newPlatformAdapter() PlatformAdapter { return windowsAdapter{} }

func (windowsAdapter) ResolveChromiumPath() string {
	// 系统 Chrome/Edge 常见路径；内置捆绑与 config 覆盖在 chromium_dist.go 统一编排。
	candidates := []string{
		filepath.Join(os.Getenv("ProgramFiles"), `Google\Chrome\Application\chrome.exe`),
		filepath.Join(os.Getenv("ProgramFiles(x86)"), `Google\Chrome\Application\chrome.exe`),
		filepath.Join(os.Getenv("ProgramFiles(x86)"), `Microsoft\Edge\Application\msedge.exe`),
		filepath.Join(os.Getenv("ProgramFiles"), `Microsoft\Edge\Application\msedge.exe`),
	}
	for _, p := range candidates {
		if p != "" && fileExists(p) {
			return p
		}
	}
	return "" // 交给 chromium_dist.go 兜底（go-rod 下载）
}

func (windowsAdapter) DefaultLaunchArgs() []string {
	return []string{"--disable-gpu", "--no-first-run", "--no-default-browser-check"}
}

func (windowsAdapter) KillProcess(pid int, graceful bool) error {
	// Windows 无 POSIX 信号；os.Process.Kill 走 TerminateProcess。graceful 在
	// Windows 上无对应轻量信号，这里直接终止（进程组清理由 Job Object 负责，Phase 5）。
	p, err := os.FindProcess(pid)
	if err != nil {
		return fmt.Errorf("find process %d: %w", pid, err)
	}
	if err := p.Kill(); err != nil {
		return fmt.Errorf("terminate process %d: %w", pid, err)
	}
	return nil
}

func (windowsAdapter) SampleProcessMemory(pid int) uint64 { return 0 } // 占位：Phase 6 用 PSAPI
func (windowsAdapter) AvailableSystemMemory() uint64      { return 0 } // 占位：Phase 8

func (windowsAdapter) AppDataDir() string {
	base := os.Getenv("LOCALAPPDATA")
	if base == "" {
		base = filepath.Join(os.Getenv("USERPROFILE"), `AppData\Local`)
	}
	return filepath.Join(base, "stardust", "browser")
}

func (windowsAdapter) CacheDir() string {
	return filepath.Join(windowsAdapter{}.AppDataDir(), "cache")
}

func (windowsAdapter) ToNativePath(posix string) string {
	return strings.ReplaceAll(posix, "/", `\`)
}

// SafeDelete 处理 Windows 强制文件锁：占用中不可删，短暂重试；不存在视为成功（幂等）。
func (windowsAdapter) SafeDelete(path string) error {
	if _, err := os.Stat(path); os.IsNotExist(err) {
		return nil
	}
	var lastErr error
	for i := 0; i < 5; i++ {
		if err := os.RemoveAll(path); err == nil {
			return nil
		} else {
			lastErr = err
		}
		time.Sleep(50 * time.Millisecond) // 等占用句柄释放
	}
	return fmt.Errorf("safe delete %q after retries: %w", path, lastErr)
}

func (windowsAdapter) WrapWithSandbox(cmd *exec.Cmd) *exec.Cmd {
	return cmd // 占位：Phase 5 接 AppContainer + Job Object
}

func fileExists(p string) bool {
	info, err := os.Stat(p)
	return err == nil && !info.IsDir()
}
```

> 注：`time.Sleep` 在 SafeDelete 重试里是对"等外部句柄释放"的合理等待（非轮询自己的状态），可保留。

- [ ] **Step 4: 跑，确认通过**

Run:
```
go test ./internal/browser/ -run 'TestWindows|TestNewPlatformAdapter' -v
go build ./internal/browser/
```
Expected: PASS（Task 1 的 `TestNewPlatformAdapterCurrentOS` 现在也过，因为 windows 版补齐了 `newPlatformAdapter`）。

- [ ] **Step 5: Commit（含 Task 1 的 platform.go）**

```bash
git add internal/browser/platform.go internal/browser/platform_test.go internal/browser/platform_windows.go internal/browser/platform_windows_test.go
git commit -m "feat(browser): PlatformAdapter interface + Windows implementation"
```
（附空行 + `Co-Authored-By: Claude Opus 4.8 <noreply@anthropic.com>`。）

---

## Task 3: Linux + macOS 实现（跨平台编译校验）

**Files:**
- Create: `internal/browser/platform_linux.go`（`//go:build linux`）
- Create: `internal/browser/platform_darwin.go`（`//go:build darwin`）

- [ ] **Step 1: 实现 Linux PAL**

Create `internal/browser/platform_linux.go`:

```go
//go:build linux

package browser

import (
	"fmt"
	"os"
	"os/exec"
	"path/filepath"
	"syscall"
)

type linuxAdapter struct{}

func newPlatformAdapter() PlatformAdapter { return linuxAdapter{} }

func (linuxAdapter) ResolveChromiumPath() string {
	for _, name := range []string{"chromium", "chromium-browser", "google-chrome", "google-chrome-stable"} {
		if p, err := exec.LookPath(name); err == nil {
			return p
		}
	}
	return ""
}

func (linuxAdapter) DefaultLaunchArgs() []string {
	// --no-sandbox 在容器/无 userns 环境常需，但有安全代价——默认不加，由 Phase 5 沙箱策略决定。
	return []string{"--disable-gpu", "--no-first-run", "--no-default-browser-check", "--headless=new"}
}

func (linuxAdapter) KillProcess(pid int, graceful bool) error {
	p, err := os.FindProcess(pid)
	if err != nil {
		return fmt.Errorf("find process %d: %w", pid, err)
	}
	sig := syscall.SIGKILL
	if graceful {
		sig = syscall.SIGTERM
	}
	if err := p.Signal(sig); err != nil {
		return fmt.Errorf("signal %v to %d: %w", sig, pid, err)
	}
	return nil
}

func (linuxAdapter) SampleProcessMemory(pid int) uint64 { return 0 } // 占位：Phase 6 读 /proc/<pid>/status
func (linuxAdapter) AvailableSystemMemory() uint64      { return 0 } // 占位：Phase 8 读 /proc/meminfo

func (linuxAdapter) AppDataDir() string {
	base := os.Getenv("XDG_DATA_HOME")
	if base == "" {
		base = filepath.Join(os.Getenv("HOME"), ".local", "share")
	}
	return filepath.Join(base, "stardust", "browser")
}

func (linuxAdapter) CacheDir() string {
	base := os.Getenv("XDG_CACHE_HOME")
	if base == "" {
		base = filepath.Join(os.Getenv("HOME"), ".cache")
	}
	return filepath.Join(base, "stardust", "browser")
}

func (linuxAdapter) ToNativePath(posix string) string { return posix } // 已是 POSIX

func (linuxAdapter) SafeDelete(path string) error {
	if err := os.RemoveAll(path); err != nil {
		return fmt.Errorf("safe delete %q: %w", path, err)
	}
	return nil
}

func (linuxAdapter) WrapWithSandbox(cmd *exec.Cmd) *exec.Cmd { return cmd } // 占位：Phase 5 namespaces+seccomp
```

- [ ] **Step 2: 实现 macOS PAL**

Create `internal/browser/platform_darwin.go`:

```go
//go:build darwin

package browser

import (
	"fmt"
	"os"
	"os/exec"
	"path/filepath"
	"syscall"
)

type darwinAdapter struct{}

func newPlatformAdapter() PlatformAdapter { return darwinAdapter{} }

func (darwinAdapter) ResolveChromiumPath() string {
	candidates := []string{
		"/Applications/Chromium.app/Contents/MacOS/Chromium",
		"/Applications/Google Chrome.app/Contents/MacOS/Google Chrome",
		"/Applications/Microsoft Edge.app/Contents/MacOS/Microsoft Edge",
	}
	for _, p := range candidates {
		if fileExistsDarwin(p) {
			return p
		}
	}
	return ""
}

func (darwinAdapter) DefaultLaunchArgs() []string {
	return []string{"--disable-gpu", "--no-first-run", "--no-default-browser-check"}
}

func (darwinAdapter) KillProcess(pid int, graceful bool) error {
	p, err := os.FindProcess(pid)
	if err != nil {
		return fmt.Errorf("find process %d: %w", pid, err)
	}
	sig := syscall.SIGKILL
	if graceful {
		sig = syscall.SIGTERM
	}
	if err := p.Signal(sig); err != nil {
		return fmt.Errorf("signal %v to %d: %w", sig, pid, err)
	}
	return nil
}

func (darwinAdapter) SampleProcessMemory(pid int) uint64 { return 0 } // 占位：Phase 6 task_info/ps
func (darwinAdapter) AvailableSystemMemory() uint64      { return 0 } // 占位：Phase 8

func (darwinAdapter) AppDataDir() string {
	return filepath.Join(os.Getenv("HOME"), "Library", "Application Support", "stardust", "browser")
}

func (darwinAdapter) CacheDir() string {
	return filepath.Join(os.Getenv("HOME"), "Library", "Caches", "stardust", "browser")
}

func (darwinAdapter) ToNativePath(posix string) string { return posix }

func (darwinAdapter) SafeDelete(path string) error {
	if err := os.RemoveAll(path); err != nil {
		return fmt.Errorf("safe delete %q: %w", path, err)
	}
	return nil
}

func (darwinAdapter) WrapWithSandbox(cmd *exec.Cmd) *exec.Cmd { return cmd } // 占位：Phase 5 App Sandbox

func fileExistsDarwin(p string) bool {
	info, err := os.Stat(p)
	return err == nil && !info.IsDir()
}
```

- [ ] **Step 3: 跨平台编译校验（本机 Windows 交叉编译）**

Run:
```
GOOS=linux GOARCH=amd64 go build ./internal/browser/
GOOS=darwin GOARCH=amd64 go build ./internal/browser/
GOOS=windows GOARCH=amd64 go build ./internal/browser/
```
（PowerShell 下用 `$env:GOOS='linux'; go build ./internal/browser/; Remove-Item Env:GOOS` 或在 bash 工具里如上。）
Expected: 三平台都编译通过（build-tag 各选一份 `newPlatformAdapter`，无重复定义、无缺失）。若 linux/darwin 报未用 import 或签名不符，据实修。

- [ ] **Step 4: 本机测试仍绿**

Run: `go test ./internal/browser/ -run 'TestWindows|TestNewPlatformAdapter' -v`
Expected: PASS（Windows 路径不受影响）。

- [ ] **Step 5: Commit**

```bash
git add internal/browser/platform_linux.go internal/browser/platform_darwin.go
git commit -m "feat(browser): PAL Linux and macOS implementations"
```

---

## Task 4: Manager 接 PAL + Chromium 分发优先级

**Files:**
- Create: `internal/browser/chromium_dist.go`
- Modify: `internal/browser/manager.go`
- Test: `internal/browser/chromium_dist_test.go`

- [ ] **Step 1: 写失败测试（分发优先级，纯函数）**

Create `internal/browser/chromium_dist_test.go`:

```go
package browser

import (
	"path/filepath"
	"testing"
)

// resolveChromiumBin 的优先级：config override > 内置捆绑(存在) > PAL 系统探测 > ""(交 go-rod 下载)。
func TestResolveChromiumBinPriority(t *testing.T) {
	tmp := t.TempDir()
	bundled := filepath.Join(tmp, "bundled-chrome")
	writeFile(t, bundled)
	system := filepath.Join(tmp, "system-chrome")
	writeFile(t, system)

	// 1. config override 最高
	got := resolveChromiumBin(ChromiumDist{ConfigBinPath: "/explicit/chrome", BundledPath: bundled, SystemPath: system})
	if got != "/explicit/chrome" {
		t.Fatalf("config override should win, got %q", got)
	}
	// 2. 无 config，内置存在则用内置
	got = resolveChromiumBin(ChromiumDist{BundledPath: bundled, SystemPath: system})
	if got != bundled {
		t.Fatalf("bundled should win over system, got %q", got)
	}
	// 3. 无 config、内置不存在，用系统
	got = resolveChromiumBin(ChromiumDist{BundledPath: filepath.Join(tmp, "missing"), SystemPath: system})
	if got != system {
		t.Fatalf("system fallback expected, got %q", got)
	}
	// 4. 都无 → 空串（go-rod 自动下载）
	got = resolveChromiumBin(ChromiumDist{})
	if got != "" {
		t.Fatalf("empty expected for auto-download, got %q", got)
	}
}
```
（`writeFile` 是本测试小 helper：`os.WriteFile(path, []byte("x"), 0o755)`；若包内已有类似 helper 复用。）

- [ ] **Step 2: 跑，确认失败**

Run: `go test ./internal/browser/ -run TestResolveChromiumBin -v`
Expected: FAIL（`resolveChromiumBin`/`ChromiumDist` 未定义）。

- [ ] **Step 3: 实现分发逻辑**

Create `internal/browser/chromium_dist.go`:

```go
package browser

import "os"

// ChromiumDist 汇集定位 Chromium 的各来源。
type ChromiumDist struct {
	ConfigBinPath string // config 显式指定（最高优先）
	BundledPath   string // 随 App 内置捆绑的固定版（存在才用）
	SystemPath    string // PAL 探测到的系统 Chrome/Edge
}

// resolveChromiumBin 按优先级返回可执行文件路径；返回 "" 表示交给 go-rod launcher 自动下载。
// 优先级：config override > 内置捆绑(存在) > 系统探测 > 空(下载)。
func resolveChromiumBin(d ChromiumDist) string {
	if d.ConfigBinPath != "" {
		return d.ConfigBinPath // 尊重显式配置，不校验存在性（让 launch 阶段报清晰错误）
	}
	if d.BundledPath != "" && binExists(d.BundledPath) {
		return d.BundledPath
	}
	if d.SystemPath != "" {
		return d.SystemPath
	}
	return ""
}

func binExists(p string) bool {
	info, err := os.Stat(p)
	return err == nil && !info.IsDir()
}
```

- [ ] **Step 4: Manager 接 PAL**

Edit `internal/browser/manager.go`：给 `Manager`/`ManagerConfig` 引入 PAL 与分发。

在 `ManagerConfig` 加 `BundledChromiumPath string`（App 打包时指向内置版；默认空）。在 `Manager` 加字段 `pal PlatformAdapter`。`NewManager` 里：
```go
	pal := NewPlatformAdapter()
	binPath := resolveChromiumBin(ChromiumDist{
		ConfigBinPath: cfg.BinPath,
		BundledPath:   cfg.BundledChromiumPath,
		SystemPath:    pal.ResolveChromiumPath(),
	})
	l := launcher.New().Headless(cfg.Headless)
	if binPath != "" {
		l = l.Bin(binPath)
	}
	for _, arg := range pal.DefaultLaunchArgs() {
		l = l.Set(flags.Flag(strings.TrimPrefix(arg, "--"))) // 或 l.Append；据 go-rod launcher API 调整
	}
	// ...existing Launch()...
	m.pal = pal
```
> **go-rod 参数注入**：`go doc github.com/go-rod/rod/lib/launcher Launcher.Set` / `.Append` 确认如何加自定义 flag，据实写（`DefaultLaunchArgs` 返回的是 `--flag` 形式；launcher 可能要去掉 `--` 前缀）。若嫌复杂，本 Task 可只接 `ResolveChromiumPath`，把 `DefaultLaunchArgs` 注入留到验证 launcher API 后——但优先按上面接全。

在 `Close`/`Reap` 相关处：若 Manager 记录了 Chromium 进程 pid（`launcher` 或 `rod.Browser` 可取），用 `m.pal.KillProcess(pid, false)` 替代裸 Cleanup 的进程终止部分（保留 `launcher.Cleanup()` 做临时目录清理）。若当前无处显式 kill（Phase 1 只 Cleanup），本 Task 至少把 pal 存好、并在文档注明 Reap 走 pal.KillProcess（Reap 完整实现在 Phase 6）。

- [ ] **Step 5: 跑 + 跨平台编译**

Run:
```
go test ./internal/browser/ -run 'TestResolveChromiumBin|TestWindows|TestNewPlatformAdapter' -v
go build ./internal/browser/
GOOS=linux go build ./internal/browser/ && GOOS=darwin go build ./internal/browser/
go test ./internal/browser/ 2>&1 | tail -3
```
Expected: 全过；三平台编译通过；既有 browser 测试不回归。

- [ ] **Step 6: 真机验证（系统 Chrome 经 PAL 定位）**

Run（确认 PAL 路径能真启动，不再需手写 BinPath）:
```
go test -tags chromium ./internal/browser/ -run TestManagerAcquireReleaseContext -v
```
> 本机 go-rod 自动下载损坏，但现在 `ResolveChromiumPath()` 应探测到系统 Chrome/Edge 并用它，无需 config BinPath。若测试用例硬编码 BinPath，可另加一个不带 BinPath 的用例验证 PAL 自动定位（可选）。若仍失败（系统无 Chrome），DONE_WITH_CONCERNS 说明。

- [ ] **Step 7: Commit**

```bash
git add internal/browser/chromium_dist.go internal/browser/chromium_dist_test.go internal/browser/manager.go
git commit -m "feat(browser): resolve Chromium via PAL (config > bundled > system > download)"
```

---

## Task 5: loopback 一次性 token + 握手

**Files:**
- Create: `internal/server/loopback_auth.go`
- Modify: `internal/config/config.go`（`Server` 加 loopback 加固字段）
- Modify: `internal/cli/command.go`（loopback 模式生成 token + 写握手文件）
- Test: `internal/server/loopback_auth_test.go`

- [ ] **Step 1: 写失败测试（token 生成 + 握手序列化，无网络）**

Create `internal/server/loopback_auth_test.go`:

```go
package server

import (
	"encoding/json"
	"strings"
	"testing"
)

func TestGenerateLoopbackTokenUnique(t *testing.T) {
	a, err := GenerateLoopbackToken()
	if err != nil {
		t.Fatalf("gen: %v", err)
	}
	b, _ := GenerateLoopbackToken()
	if a == "" || len(a) < 32 {
		t.Fatalf("token too short: %q", a)
	}
	if a == b {
		t.Fatal("tokens not unique across calls")
	}
}

func TestHandshakeJSON(t *testing.T) {
	h := Handshake{BaseURL: "http://127.0.0.1:54321", Token: "abc"}
	b, err := json.Marshal(h)
	if err != nil {
		t.Fatalf("marshal: %v", err)
	}
	if !strings.Contains(string(b), `"baseURL"`) || !strings.Contains(string(b), `"token"`) {
		t.Fatalf("handshake json shape: %s", b)
	}
}
```

- [ ] **Step 2: 跑，确认失败**

Run: `go test ./internal/server/ -run 'TestGenerateLoopbackToken|TestHandshake' -v`
Expected: FAIL（`GenerateLoopbackToken`/`Handshake` 未定义）。

- [ ] **Step 3: 实现 token + 握手**

Create `internal/server/loopback_auth.go`:

```go
package server

import (
	"crypto/rand"
	"encoding/hex"
	"encoding/json"
	"fmt"
	"os"
)

// Handshake 是 App 模式下交给前端的自连凭证（spec §3.4 握手）。
type Handshake struct {
	BaseURL string `json:"baseURL"`
	Token   string `json:"token"`
}

// GenerateLoopbackToken 生成一次性随机 bearer token（每次启动轮换）。
func GenerateLoopbackToken() (string, error) {
	b := make([]byte, 32)
	if _, err := rand.Read(b); err != nil {
		return "", fmt.Errorf("generate loopback token: %w", err)
	}
	return hex.EncodeToString(b), nil
}

// WriteHandshake 把 {baseURL, token} 写到约定文件（前端读取后带 Bearer 自连）。
// 文件权限 0600（仅当前用户可读），避免同机其他用户窃取 token。
func WriteHandshake(path string, h Handshake) error {
	b, err := json.Marshal(h)
	if err != nil {
		return fmt.Errorf("marshal handshake: %w", err)
	}
	if err := os.WriteFile(path, b, 0o600); err != nil {
		return fmt.Errorf("write handshake %q: %w", path, err)
	}
	return nil
}
```

- [ ] **Step 4: config 字段 + cli 生成/写握手**

`internal/config/config.go` 的 `Server` 配置加：
```go
	LoopbackHardening bool   `json:"loopback_hardening" yaml:"loopback_hardening"` // App/GUI 模式启用一次性 token + Origin 校验
	HandshakeFile     string `json:"handshake_file" yaml:"handshake_file"`         // 握手写入路径（空则默认 AppDataDir 下）
```

`internal/cli/command.go` 的 loopback/GUI 模式（`addr == "127.0.0.1:0"` 或 `LoopbackHardening` 开）装配处：若未显式配置 `AdminToken`，`GenerateLoopbackToken()` 生成一个并用作 `AdminToken`（复用现有 bearer 校验 http.go:338）；服务器 `Listen` 拿到实际端口后，`WriteHandshake(handshakeFile, Handshake{BaseURL: "http://"+listener.Addr().String(), Token: token})`。handshakeFile 缺省用 `pal.AppDataDir()/handshake.json`（经 browser.NewPlatformAdapter 或直接 config）。

> 调查步：`grep -n "AdminToken\|listener.Addr\|net.Listen" internal/cli/command.go` 找到 listener 拿端口的位置，在其后写握手。生成 token 的分支：仅当 loopback 且 `cfg.Server.AdminToken==""` 时生成（尊重显式配置）。

- [ ] **Step 5: 跑 + 全量**

Run:
```
go test ./internal/server/ -run 'TestGenerateLoopbackToken|TestHandshake' -v
go build ./... && go test ./internal/server/ ./internal/cli/ 2>&1 | tail -6
```
Expected: PASS；全量编译；相关包绿。

- [ ] **Step 6: Commit**

```bash
git add internal/server/loopback_auth.go internal/server/loopback_auth_test.go internal/config/config.go internal/cli/command.go
git commit -m "feat(server): one-time loopback bearer token + handshake file for App mode"
```

---

## Task 6: Origin/自定义头校验中间件

**Files:**
- Modify: `internal/server/loopback_auth.go`（加中间件）
- Modify: `internal/server/http.go`（loopback 模式挂中间件）
- Test: `internal/server/loopback_auth_test.go`（追加中间件测试）

- [ ] **Step 1: 写失败测试（中间件放行/拒绝）**

追加到 `internal/server/loopback_auth_test.go`:

```go
func TestLoopbackOriginMiddleware(t *testing.T) {
	next := http.HandlerFunc(func(w http.ResponseWriter, _ *http.Request) { w.WriteHeader(200) })
	mw := LoopbackOriginGuard(next, "http://127.0.0.1:54321")

	// 合法：无 Origin（同源 fetch 常不带）或匹配的 Origin → 放行
	rec := httptest.NewRecorder()
	mw.ServeHTTP(rec, httptest.NewRequest(http.MethodGet, "/v1/events", nil))
	if rec.Code != 200 {
		t.Fatalf("no-Origin should pass, got %d", rec.Code)
	}
	// 非法：跨站 Origin → 拒绝
	rec = httptest.NewRecorder()
	req := httptest.NewRequest(http.MethodGet, "/v1/events", nil)
	req.Header.Set("Origin", "https://evil.example.com")
	mw.ServeHTTP(rec, req)
	if rec.Code != http.StatusForbidden {
		t.Fatalf("cross-origin should be 403, got %d", rec.Code)
	}
}
```
（补 `import "net/http"`、`"net/http/httptest"`。）

- [ ] **Step 2: 跑，确认失败**

Run: `go test ./internal/server/ -run TestLoopbackOrigin -v`
Expected: FAIL（`LoopbackOriginGuard` 未定义）。

- [ ] **Step 3: 实现中间件**

追加到 `internal/server/loopback_auth.go`:

```go
import "net/http" // 若文件未 import，补上

// LoopbackOriginGuard 是 App/loopback 模式的 Origin 校验中间件（spec §3.4 加固四件套之一）：
// 挡住同机其他进程/网页对这个 loopback 端口的越权访问。策略：
//   - 无 Origin 头（webview 同源 fetch/EventSource 通常不带）→ 放行；
//   - Origin 等于本服务 baseURL → 放行；
//   - 其他 → 403。
// 与 bearer token（http.go 现有校验）叠加，构成 §3.4 的双重防线。
func LoopbackOriginGuard(next http.Handler, allowedOrigin string) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		origin := r.Header.Get("Origin")
		if origin != "" && origin != allowedOrigin {
			http.Error(w, "forbidden origin", http.StatusForbidden)
			return
		}
		next.ServeHTTP(w, r)
	})
}
```

- [ ] **Step 4: http.go loopback 模式挂中间件**

在 `internal/cli/command.go` 构造 `http.Server{Handler: ...}` 处（loopback/GUI 模式），把 `HTTPServer` 包一层 `LoopbackOriginGuard(httpServer, "http://"+listener.Addr().String())`——仅在 loopback 加固开启时。服务模式不包（沿用原鉴权）。

> 调查步：找到 `http.Server{Handler:` 或 `HTTPServer:` 装配处（command.go:2604 附近），在 loopback 分支替换 Handler 为包裹后的。allowedOrigin 用握手里同一个 baseURL。

- [ ] **Step 5: 跑 + 全量回归**

Run:
```
go test ./internal/server/ -run 'TestLoopbackOrigin|TestGenerateLoopbackToken|TestHandshake' -v
go build ./... && go test ./... 2>&1 | tail -15
GOOS=linux go build ./... 2>&1 | tail -3
go test ./internal/runtime/ -run TestEveryProductionToolIsGateable
```
Expected: 中间件测试 PASS；全量绿（chromium-tag 跳过）；linux 交叉编译过；drift 绿。

- [ ] **Step 6: Commit**

```bash
git add internal/server/loopback_auth.go internal/server/loopback_auth_test.go internal/server/http.go internal/cli/command.go
git commit -m "feat(server): loopback Origin-guard middleware for App mode hardening"
```

---

## 验证 Phase 4A+4B DoD

- [ ] PAL：`platform.go` 接口 + 三平台 build-tag 实现；三平台 `GOOS=... go build` 全过（Task 1-3）
- [ ] 除 PAL 文件外，`grep -rn "runtime.GOOS" internal/browser` 无命中（Task 全程保持）
- [ ] Chromium 分发优先级 config>内置>系统>下载（Task 4 纯函数测试）；本机经 PAL 自动定位系统 Chrome 启动（Task 4 真机）
- [ ] loopback 加固四件套齐：127.0.0.1（已有）+ 随机端口（已有）+ **一次性 token**（Task 5）+ **Origin 校验**（Task 6）
- [ ] 握手文件 0600 权限写 {baseURL, token}（Task 5）
- [ ] 服务模式不受加固中间件影响；全量 `go test ./...` 绿、drift 绿

---

## 已知占位与后续 Phase

| 占位 | 本期 | 后续 |
|---|---|---|
| `WrapWithSandbox` | 透传 no-op | Phase 5：Win AppContainer+Job Object / mac App Sandbox / Linux namespaces+seccomp |
| `SampleProcessMemory`/`AvailableSystemMemory` | 返回 0 占位 | Phase 6/8：PSAPI / /proc / task_info + ResourcePolicy |
| `KillProcess` 进程组 | 单进程 Kill | Phase 5：Win Job Object `KILL_ON_JOB_CLOSE` 连带清理 Chromium |
| 内置捆绑 Chromium | `BundledChromiumPath` 字段就位，值由打包提供 | 4C：Wails 打包把固定版 Chromium 放进 App 资源并填此路径 |
| Wails webview 宿主 + 前端读握手 | 后端写握手文件 | 4C（legionAgentGUI）：React 读文件/注入点拿 {baseURL,token} 自连 |
| 三平台 CI | 本机交叉编译校验 | 4D：CI 矩阵跑三平台 chromium e2e |
| Reap 完整实现 | pal.KillProcess 就位 | Phase 6：进程池健康检查 + 僵死 Reap |
```
