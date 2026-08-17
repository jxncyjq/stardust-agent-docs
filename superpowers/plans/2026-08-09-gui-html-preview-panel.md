# GUI HTML 预览面板 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 在 Wails GUI 右侧面板新增"预览"标签页，用 `<iframe srcdoc sandbox="">` 静态渲染纯 HTML（agent 富 HTML、markdown 转出的完整 HTML、本地 .html 文件）；聊天里的 http(s) 链接改为丢系统浏览器打开。

**Architecture:** 纯前端 React 组件 + zustand store + 一个 Go 绑定（读本地 HTML 文件，fail-loud 校验）+ 一个 Wails 事件监听 hook（`preview:open`）。预览面板作为 `RightPanel` 的第三个 tab 复用现有可调宽的 status 列，无需自建宽度持久化。远程 URL 一律 `BrowserOpenURL`，不进 iframe（规避 X-Frame-Options）。iframe `sandbox=""` 零脚本零交互。

**Tech Stack:** React 18 + TypeScript + zustand 5 + Tailwind 4 + Vite/Vitest；Go + Wails v2 runtime。

**Spec:** `docs/superpowers/specs/2026-08-09-gui-html-preview-panel-design.md`

**仓库根：** 所有前端路径相对 `legion/legionAgentGUI/frontend/`，Go 路径相对 `legion/legionAgentGUI/`。命令在对应目录下运行。

---

## 文件结构

| 文件 | 责任 | 动作 |
|---|---|---|
| `frontend/src/stores/previewStore.ts` | 预览状态：当前 source、open/close | 新建 |
| `frontend/src/lib/openLink.ts` | 链接分流：http(s) → BrowserOpenURL | 新建 |
| `frontend/src/components/WebPreviewPanel.tsx` | 面板：工具条 + iframe srcdoc（纯组件，props 驱动） | 新建 |
| `frontend/src/hooks/useHtmlPreviewEvents.ts` | 监听 `preview:open`，localFile→ReadHTMLFile→open | 新建 |
| `frontend/src/components/MessageBubble.tsx` | ① 链接经 openLink ② 抽 ```html 块加"预览"按钮 | 改 |
| `frontend/src/App.tsx` | RightPanel 加"预览"tab + 挂载事件 hook | 改 |
| `frontend/src/stores/uiStore.ts` | 新增 rightView 状态（三 tab 共享，供自动切换） | 改 |
| `app_preview.go` | `ReadHTMLFile(path)` 绑定 + fail-loud 校验 | 新建 |
| `app_preview_test.go` | ReadHTMLFile 的 happy + 错误路径测试 | 新建 |

> 各测试文件与被测源码同目录同名 `.test.tsx` / `_test.go`（跟随现有约定）。

---

### Task 1: previewStore（预览状态）

**Files:**
- Create: `frontend/src/stores/previewStore.ts`
- Test: `frontend/src/stores/previewStore.test.ts`

数据类型：进面板的一切最终都是一段 HTML 字符串。`localFile` 只是"待后端解析成 html 的中间态"，store 里只存已解析的最终形态 `PreviewSource`。

- [ ] **Step 1: 写失败测试**

```ts
// frontend/src/stores/previewStore.test.ts
import { describe, it, expect, beforeEach } from 'vitest'
import { usePreviewStore } from './previewStore'

describe('previewStore', () => {
  beforeEach(() => usePreviewStore.getState().close())

  it('starts empty', () => {
    expect(usePreviewStore.getState().source).toBeNull()
  })

  it('open sets the source', () => {
    usePreviewStore.getState().open({ kind: 'html', html: '<h1>hi</h1>', title: 'T' })
    const s = usePreviewStore.getState().source
    expect(s).toEqual({ kind: 'html', html: '<h1>hi</h1>', title: 'T' })
  })

  it('close clears the source', () => {
    usePreviewStore.getState().open({ kind: 'html', html: '<h1>hi</h1>' })
    usePreviewStore.getState().close()
    expect(usePreviewStore.getState().source).toBeNull()
  })
})
```

- [ ] **Step 2: 跑测试确认失败**

Run: `cd frontend && npx vitest run src/stores/previewStore.test.ts`
Expected: FAIL — Cannot find module './previewStore'

- [ ] **Step 3: 实现**

```ts
// frontend/src/stores/previewStore.ts
import { create } from 'zustand'

// PreviewSource is the fully-resolved payload the panel renders: always an HTML
// string fed to <iframe srcdoc>. A local file is resolved to this shape by the
// backend before it reaches the store — the store never holds a bare path.
export interface PreviewSource {
  kind: 'html'
  html: string
  title?: string
  // sourceUrl, when present, is the origin URL for the "open in system browser"
  // toolbar action. Absent for agent-generated or fenced-block HTML.
  sourceUrl?: string
}

interface PreviewState {
  source: PreviewSource | null
  open: (src: PreviewSource) => void
  close: () => void
}

// previewStore holds the single active HTML preview. It is a singleton panel:
// a new open() replaces whatever was showing. Kept deliberately pure (no
// cross-store writes) so it stays trivially testable; the right-column tab
// auto-switch lives in RightPanel as an effect on `source`.
export const usePreviewStore = create<PreviewState>((set) => ({
  source: null,
  open: (src) => set({ source: src }),
  close: () => set({ source: null }),
}))
```

- [ ] **Step 4: 跑测试确认通过**

Run: `cd frontend && npx vitest run src/stores/previewStore.test.ts`
Expected: PASS (3 tests)

- [ ] **Step 5: 提交**

```bash
git add frontend/src/stores/previewStore.ts frontend/src/stores/previewStore.test.ts
git commit -m "feat(gui): add previewStore for HTML preview panel"
```

---

### Task 2: openLink（链接分流工具）

**Files:**
- Create: `frontend/src/lib/openLink.ts`
- Test: `frontend/src/lib/openLink.test.ts`

只有 `http:` / `https:` 走系统浏览器；`javascript:` / `file:` 等一律忽略（不打开，防协议注入）。

- [ ] **Step 1: 写失败测试**

```ts
// frontend/src/lib/openLink.test.ts
import { describe, it, expect, vi, beforeEach } from 'vitest'

const runtimeMocks = vi.hoisted(() => ({ BrowserOpenURL: vi.fn() }))
vi.mock('../../wailsjs/runtime/runtime', () => runtimeMocks)

import { openLink } from './openLink'

describe('openLink', () => {
  beforeEach(() => runtimeMocks.BrowserOpenURL.mockClear())

  it('opens http urls in the system browser', () => {
    openLink('http://example.com/a')
    expect(runtimeMocks.BrowserOpenURL).toHaveBeenCalledWith('http://example.com/a')
  })

  it('opens https urls in the system browser', () => {
    openLink('https://example.com')
    expect(runtimeMocks.BrowserOpenURL).toHaveBeenCalledWith('https://example.com')
  })

  it('ignores javascript: urls', () => {
    openLink('javascript:alert(1)')
    expect(runtimeMocks.BrowserOpenURL).not.toHaveBeenCalled()
  })

  it('ignores file: urls', () => {
    openLink('file:///etc/passwd')
    expect(runtimeMocks.BrowserOpenURL).not.toHaveBeenCalled()
  })

  it('ignores malformed input', () => {
    openLink('not a url')
    expect(runtimeMocks.BrowserOpenURL).not.toHaveBeenCalled()
  })
})
```

- [ ] **Step 2: 跑测试确认失败**

Run: `cd frontend && npx vitest run src/lib/openLink.test.ts`
Expected: FAIL — Cannot find module './openLink'

- [ ] **Step 3: 实现**

```ts
// frontend/src/lib/openLink.ts
import { BrowserOpenURL } from '../../wailsjs/runtime/runtime'

// openLink routes a URL to the system browser when — and only when — it is
// http(s). Any other scheme (javascript:, file:, data:, …) is ignored rather
// than opened: those are either injection vectors or local-resource reads we
// never want a chat link to trigger. Malformed input is ignored too.
export function openLink(href: string): void {
  let url: URL
  try {
    url = new URL(href)
  } catch {
    return
  }
  if (url.protocol === 'http:' || url.protocol === 'https:') {
    BrowserOpenURL(href)
  }
}
```

- [ ] **Step 4: 跑测试确认通过**

Run: `cd frontend && npx vitest run src/lib/openLink.test.ts`
Expected: PASS (5 tests)

- [ ] **Step 5: 提交**

```bash
git add frontend/src/lib/openLink.ts frontend/src/lib/openLink.test.ts
git commit -m "feat(gui): add openLink helper routing http(s) to system browser"
```

---

### Task 3: WebPreviewPanel（面板组件）

**Files:**
- Create: `frontend/src/components/WebPreviewPanel.tsx`
- Test: `frontend/src/components/WebPreviewPanel.test.tsx`

纯组件：props 传 `source` + `onClose`，不直接读 store（便于测试）。iframe **`sandbox=""`**（无 `allow-scripts`、无 `allow-same-origin`）。空 source 显示占位空态。工具条：标题 + （有 sourceUrl 时）外部打开按钮 + 关闭按钮。

- [ ] **Step 1: 写失败测试**

```tsx
// frontend/src/components/WebPreviewPanel.test.tsx
import { describe, it, expect, vi, beforeEach } from 'vitest'
import { render, screen, fireEvent } from '@testing-library/react'

const runtimeMocks = vi.hoisted(() => ({ BrowserOpenURL: vi.fn() }))
vi.mock('../../wailsjs/runtime/runtime', () => runtimeMocks)

import { WebPreviewPanel } from './WebPreviewPanel'

describe('WebPreviewPanel', () => {
  beforeEach(() => runtimeMocks.BrowserOpenURL.mockClear())

  it('shows an empty state when there is no source', () => {
    render(<WebPreviewPanel source={null} onClose={() => {}} />)
    expect(screen.queryByTitle('HTML 预览内容')).toBeNull()
    expect(screen.getByText('暂无预览内容')).toBeInTheDocument()
  })

  it('renders the html into a sandboxed iframe srcdoc', () => {
    render(
      <WebPreviewPanel
        source={{ kind: 'html', html: '<h1>hello</h1>', title: '报告' }}
        onClose={() => {}}
      />
    )
    const iframe = screen.getByTitle('HTML 预览内容') as HTMLIFrameElement
    expect(iframe.getAttribute('srcdoc')).toBe('<h1>hello</h1>')
    // Security: sandbox present and does NOT grant scripts or same-origin.
    const sandbox = iframe.getAttribute('sandbox')
    expect(sandbox).not.toBeNull()
    expect(sandbox).not.toContain('allow-scripts')
    expect(sandbox).not.toContain('allow-same-origin')
  })

  it('shows the title in the toolbar', () => {
    render(
      <WebPreviewPanel source={{ kind: 'html', html: '<p>x</p>', title: '报告' }} onClose={() => {}} />
    )
    expect(screen.getByText('报告')).toBeInTheDocument()
  })

  it('calls onClose when the close button is clicked', () => {
    const onClose = vi.fn()
    render(<WebPreviewPanel source={{ kind: 'html', html: '<p>x</p>' }} onClose={onClose} />)
    fireEvent.click(screen.getByRole('button', { name: '关闭预览' }))
    expect(onClose).toHaveBeenCalledOnce()
  })

  it('opens sourceUrl externally and hides the button without one', () => {
    const { rerender } = render(
      <WebPreviewPanel source={{ kind: 'html', html: '<p>x</p>' }} onClose={() => {}} />
    )
    expect(screen.queryByRole('button', { name: '在系统浏览器打开' })).toBeNull()

    rerender(
      <WebPreviewPanel
        source={{ kind: 'html', html: '<p>x</p>', sourceUrl: 'https://example.com' }}
        onClose={() => {}}
      />
    )
    fireEvent.click(screen.getByRole('button', { name: '在系统浏览器打开' }))
    expect(runtimeMocks.BrowserOpenURL).toHaveBeenCalledWith('https://example.com')
  })
})
```

- [ ] **Step 2: 跑测试确认失败**

Run: `cd frontend && npx vitest run src/components/WebPreviewPanel.test.tsx`
Expected: FAIL — Cannot find module './WebPreviewPanel'

- [ ] **Step 3: 实现**

```tsx
// frontend/src/components/WebPreviewPanel.tsx
import { memo } from 'react'
import { BrowserOpenURL } from '../../wailsjs/runtime/runtime'
import type { PreviewSource } from '../stores/previewStore'
import { XIcon, ExternalLinkIcon } from './icons'

interface Props {
  source: PreviewSource | null
  onClose: () => void
}

// WebPreviewPanel renders an HTML preview inside a fully sandboxed iframe. It is
// a pure, props-driven component (no store access) so it is trivial to test.
//
// Security: srcdoc + `sandbox=""` (empty) means the content is treated as an
// opaque origin with EVERY capability withheld — no scripts, no same-origin, no
// forms. Agent-supplied HTML that contains <script> simply does not execute.
// Remote URLs never reach this iframe; they open in the system browser instead
// (see openLink / the toolbar action), sidestepping X-Frame-Options entirely.
export const WebPreviewPanel = memo(function WebPreviewPanel({ source, onClose }: Props) {
  if (!source) {
    return (
      <div className="flex h-full items-center justify-center p-4 text-xs text-muted-foreground">
        暂无预览内容
      </div>
    )
  }
  return (
    <div className="flex h-full flex-col">
      <div className="flex items-center gap-2 border-b border-border px-2 py-1">
        <span className="flex-1 truncate text-xs font-medium text-foreground">
          {source.title ?? 'HTML 预览'}
        </span>
        {source.sourceUrl && (
          <button
            type="button"
            className="interactive rounded p-1 text-muted-foreground hover:bg-muted hover:text-foreground"
            onClick={() => BrowserOpenURL(source.sourceUrl!)}
            aria-label="在系统浏览器打开"
            title="在系统浏览器打开"
          >
            <ExternalLinkIcon className="h-3.5 w-3.5" />
          </button>
        )}
        <button
          type="button"
          className="interactive rounded p-1 text-muted-foreground hover:bg-muted hover:text-foreground"
          onClick={onClose}
          aria-label="关闭预览"
          title="关闭预览"
        >
          <XIcon className="h-3.5 w-3.5" />
        </button>
      </div>
      <iframe
        title="HTML 预览内容"
        sandbox=""
        srcDoc={source.html}
        className="min-h-0 flex-1 w-full border-0 bg-white"
      />
    </div>
  )
})
```

> **注意 icons：** 已确认 `XIcon` 存在于 `frontend/src/components/icons.tsx`，但 `ExternalLinkIcon` **不存在**，需先补上。跟随该文件现有 icon 的写法（`export function XIcon({ className }: { className?: string }) { return <svg …/> }`），追加：
>
> ```tsx
> // ExternalLinkIcon marks the "open in system browser" action (lucide external-link).
> export function ExternalLinkIcon({ className }: { className?: string }) {
>   return (
>     <svg className={className} width="16" height="16" viewBox="0 0 24 24" fill="none"
>       stroke="currentColor" strokeWidth="2" strokeLinecap="round" strokeLinejoin="round">
>       <path d="M15 3h6v6" />
>       <path d="M10 14 21 3" />
>       <path d="M18 13v6a2 2 0 0 1-2 2H5a2 2 0 0 1-2-2V8a2 2 0 0 1 2-2h6" />
>     </svg>
>   )
> }
> ```
>
> 先核对现有 icon 的 props 形状（`grep -nA3 "export function XIcon" frontend/src/components/icons.tsx`），若与上面签名不同则对齐现有写法。

- [ ] **Step 4: 跑测试确认通过**

Run: `cd frontend && npx vitest run src/components/WebPreviewPanel.test.tsx`
Expected: PASS (5 tests)

- [ ] **Step 5: 提交**

```bash
git add frontend/src/components/WebPreviewPanel.tsx frontend/src/components/WebPreviewPanel.test.tsx frontend/src/components/icons.tsx
git commit -m "feat(gui): add WebPreviewPanel (sandboxed iframe srcdoc)"
```

---

### Task 4: Go ReadHTMLFile 绑定（fail-loud 校验）

**Files:**
- Create: `app_preview.go`
- Test: `app_preview_test.go`

fail-loud（遵守 GUI CLAUDE.md 铁律）：扩展名非 `.html/.htm`、路径含 `..`、文件不存在/非普通文件 → 一律返回 `fmt.Errorf` 包装的 error，绝不返回空串冒充成功。

- [ ] **Step 1: 写失败测试**

```go
// app_preview_test.go
package main

import (
	"os"
	"path/filepath"
	"strings"
	"testing"
)

func TestReadHTMLFileReadsHTML(t *testing.T) {
	dir := t.TempDir()
	p := filepath.Join(dir, "report.html")
	if err := os.WriteFile(p, []byte("<h1>ok</h1>"), 0o644); err != nil {
		t.Fatal(err)
	}
	a := NewApp("")
	got, err := a.ReadHTMLFile(p)
	if err != nil {
		t.Fatalf("unexpected error: %v", err)
	}
	if got != "<h1>ok</h1>" {
		t.Fatalf("got %q, want %q", got, "<h1>ok</h1>")
	}
}

func TestReadHTMLFileRejectsNonHTMLExtension(t *testing.T) {
	dir := t.TempDir()
	p := filepath.Join(dir, "secret.txt")
	if err := os.WriteFile(p, []byte("nope"), 0o644); err != nil {
		t.Fatal(err)
	}
	a := NewApp("")
	got, err := a.ReadHTMLFile(p)
	if err == nil {
		t.Fatal("expected error for non-.html extension, got nil")
	}
	if got != "" {
		t.Fatalf("expected empty result on error, got %q", got)
	}
	if !strings.Contains(err.Error(), "extension") {
		t.Fatalf("error should mention extension, got: %v", err)
	}
}

func TestReadHTMLFileRejectsTraversal(t *testing.T) {
	a := NewApp("")
	_, err := a.ReadHTMLFile("../../etc/passwd.html")
	if err == nil {
		t.Fatal("expected error for path traversal, got nil")
	}
	if !strings.Contains(err.Error(), "traversal") {
		t.Fatalf("error should mention traversal, got: %v", err)
	}
}

func TestReadHTMLFileRejectsMissingFile(t *testing.T) {
	dir := t.TempDir()
	a := NewApp("")
	_, err := a.ReadHTMLFile(filepath.Join(dir, "nope.html"))
	if err == nil {
		t.Fatal("expected error for missing file, got nil")
	}
}
```

- [ ] **Step 2: 跑测试确认失败**

Run: `go test ./... -run TestReadHTMLFile`
Expected: FAIL — `a.ReadHTMLFile undefined`

- [ ] **Step 3: 实现**

```go
// app_preview.go
package main

import (
	"fmt"
	"os"
	"path/filepath"
	"strings"
)

// ReadHTMLFile reads a local HTML file and returns its contents for the preview
// panel's iframe srcdoc. It is fail-loud (see CLAUDE.md): any unexpected state —
// wrong extension, path traversal, missing or non-regular file — returns a
// wrapped error rather than an empty string masquerading as success. Only
// .html/.htm files are allowed; the content is rendered in a fully sandboxed
// iframe (scripts disabled) on the frontend, so this reads bytes without
// interpreting them.
func (a *App) ReadHTMLFile(path string) (string, error) {
	// Reject traversal before touching the filesystem. Clean collapses the path;
	// a remaining ".." segment means the caller tried to escape upward.
	clean := filepath.Clean(path)
	if clean == ".." || strings.HasPrefix(clean, ".."+string(filepath.Separator)) ||
		strings.Contains(clean, string(filepath.Separator)+".."+string(filepath.Separator)) {
		return "", fmt.Errorf("read html %q: path traversal rejected", path)
	}

	ext := strings.ToLower(filepath.Ext(clean))
	if ext != ".html" && ext != ".htm" {
		return "", fmt.Errorf("read html %q: unsupported extension %q (want .html/.htm)", path, ext)
	}

	info, err := os.Stat(clean)
	if err != nil {
		return "", fmt.Errorf("stat html %q: %w", path, err)
	}
	if !info.Mode().IsRegular() {
		return "", fmt.Errorf("read html %q: not a regular file", path)
	}

	data, err := os.ReadFile(clean)
	if err != nil {
		return "", fmt.Errorf("read html %q: %w", path, err)
	}
	return string(data), nil
}
```

- [ ] **Step 4: 跑测试确认通过**

Run: `go test ./... -run TestReadHTMLFile -v && gofmt -l app_preview.go`
Expected: 4 tests PASS；gofmt 输出为空

- [ ] **Step 5: 重新生成 Wails 绑定**

新 App 方法要暴露给前端，需重生成绑定（写出 `frontend/wailsjs/go/main/App.{js,d.ts}`）。

Run: `wails generate module`
Expected: 命令成功；`grep -n ReadHTMLFile frontend/wailsjs/go/main/App.d.ts` 能看到 `export function ReadHTMLFile(arg1:string):Promise<string>;`

> 若环境无 `wails` CLI：手动在 `frontend/wailsjs/go/main/App.d.ts` 加 `export function ReadHTMLFile(arg1:string):Promise<string>;`，在 `App.js` 仿照现有导出加 `export function ReadHTMLFile(arg1){ return window['go']['main']['App']['ReadHTMLFile'](arg1); }`。

- [ ] **Step 6: 提交**

```bash
git add app_preview.go app_preview_test.go frontend/wailsjs/go/main/App.js frontend/wailsjs/go/main/App.d.ts
git commit -m "feat(gui): add ReadHTMLFile binding with fail-loud path validation"
```

---

### Task 5: useHtmlPreviewEvents（监听 preview:open）

**Files:**
- Create: `frontend/src/hooks/useHtmlPreviewEvents.ts`
- Test: `frontend/src/hooks/useHtmlPreviewEvents.test.tsx`

监听 Wails 事件 `preview:open`。payload 两形态：`{kind:'html', html, title?, sourceUrl?}` 直接 open；`{kind:'localFile', path, title?}` 先调 `ReadHTMLFile(path)` 拿到 html 再 open。解析失败/后端报错 → `console.error`（前端边界，不静默）。跟随 `useBrowserSession` 的 EventsOn/EventsOff 模式。

- [ ] **Step 1: 写失败测试**

```tsx
// frontend/src/hooks/useHtmlPreviewEvents.test.tsx
import { describe, it, expect, vi, beforeEach } from 'vitest'
import { render, waitFor } from '@testing-library/react'

const appMocks = vi.hoisted(() => ({ ReadHTMLFile: vi.fn() }))
vi.mock('../../wailsjs/go/main/App', () => appMocks)

const runtimeMocks = vi.hoisted(() => {
  const listeners: Record<string, Array<(...a: any[]) => void>> = {}
  return {
    listeners,
    EventsOn: vi.fn((name: string, cb: (...a: any[]) => void) => {
      ;(listeners[name] ??= []).push(cb)
      return () => { listeners[name] = (listeners[name] ?? []).filter((c) => c !== cb) }
    }),
    EventsOff: vi.fn((name: string) => { delete listeners[name] }),
  }
})
vi.mock('../../wailsjs/runtime/runtime', () => ({
  EventsOn: runtimeMocks.EventsOn,
  EventsOff: runtimeMocks.EventsOff,
}))

import { useHtmlPreviewEvents } from './useHtmlPreviewEvents'
import { usePreviewStore } from '../stores/previewStore'

function Harness() { useHtmlPreviewEvents(); return null }
function emit(payload: unknown) {
  for (const cb of runtimeMocks.listeners['preview:open'] ?? []) cb(payload)
}

describe('useHtmlPreviewEvents', () => {
  beforeEach(() => {
    usePreviewStore.getState().close()
    appMocks.ReadHTMLFile.mockReset()
  })

  it('opens inline html payloads directly', () => {
    render(<Harness />)
    emit({ kind: 'html', html: '<h1>x</h1>', title: 'T' })
    expect(usePreviewStore.getState().source).toEqual({ kind: 'html', html: '<h1>x</h1>', title: 'T' })
  })

  it('resolves a localFile payload via ReadHTMLFile then opens it', async () => {
    appMocks.ReadHTMLFile.mockResolvedValue('<h1>file</h1>')
    render(<Harness />)
    emit({ kind: 'localFile', path: '/tmp/r.html', title: 'R' })
    await waitFor(() =>
      expect(usePreviewStore.getState().source).toEqual({ kind: 'html', html: '<h1>file</h1>', title: 'R' })
    )
    expect(appMocks.ReadHTMLFile).toHaveBeenCalledWith('/tmp/r.html')
  })

  it('does not open when ReadHTMLFile rejects', async () => {
    const err = vi.spyOn(console, 'error').mockImplementation(() => {})
    appMocks.ReadHTMLFile.mockRejectedValue(new Error('bad ext'))
    render(<Harness />)
    emit({ kind: 'localFile', path: '/tmp/r.txt' })
    await waitFor(() => expect(err).toHaveBeenCalled())
    expect(usePreviewStore.getState().source).toBeNull()
    err.mockRestore()
  })

  it('ignores malformed payloads', () => {
    const err = vi.spyOn(console, 'error').mockImplementation(() => {})
    render(<Harness />)
    emit({ kind: 'nonsense' })
    expect(usePreviewStore.getState().source).toBeNull()
    err.mockRestore()
  })
})
```

- [ ] **Step 2: 跑测试确认失败**

Run: `cd frontend && npx vitest run src/hooks/useHtmlPreviewEvents.test.tsx`
Expected: FAIL — Cannot find module './useHtmlPreviewEvents'

- [ ] **Step 3: 实现**

```ts
// frontend/src/hooks/useHtmlPreviewEvents.ts
import { useEffect } from 'react'
import { EventsOn, EventsOff } from '../../wailsjs/runtime/runtime'
import { ReadHTMLFile } from '../../wailsjs/go/main/App'
import { usePreviewStore } from '../stores/previewStore'

// PreviewOpenPayload is what the backend emits on `preview:open`. Either inline
// HTML, or a local file path the frontend resolves via ReadHTMLFile. Kept as a
// discriminated union so a malformed payload is caught, not guessed at.
type PreviewOpenPayload =
  | { kind: 'html'; html: string; title?: string; sourceUrl?: string }
  | { kind: 'localFile'; path: string; title?: string }

// useHtmlPreviewEvents listens for backend-driven preview requests and pushes
// them into previewStore. Mounted once at App level (alongside
// useBrowserSession). A localFile payload is resolved through the fail-loud
// ReadHTMLFile binding before opening; any resolution/parse failure is logged
// at the boundary (never silently swallowed) and the panel is left untouched.
export function useHtmlPreviewEvents() {
  const open = usePreviewStore((s) => s.open)
  useEffect(() => {
    const handle = (payload: PreviewOpenPayload) => {
      if (!payload || typeof payload !== 'object') {
        console.error('preview:open payload not an object:', payload)
        return
      }
      if (payload.kind === 'html' && typeof payload.html === 'string') {
        open({ kind: 'html', html: payload.html, title: payload.title, sourceUrl: payload.sourceUrl })
        return
      }
      if (payload.kind === 'localFile' && typeof payload.path === 'string') {
        ReadHTMLFile(payload.path)
          .then((html) => open({ kind: 'html', html, title: payload.title }))
          .catch((err) => console.error('preview:open ReadHTMLFile failed:', payload.path, err))
        return
      }
      console.error('preview:open payload unrecognized:', payload)
    }
    EventsOn('preview:open', handle)
    return () => EventsOff('preview:open')
  }, [open])
}
```

- [ ] **Step 4: 跑测试确认通过**

Run: `cd frontend && npx vitest run src/hooks/useHtmlPreviewEvents.test.tsx`
Expected: PASS (4 tests)

- [ ] **Step 5: 提交**

```bash
git add frontend/src/hooks/useHtmlPreviewEvents.ts frontend/src/hooks/useHtmlPreviewEvents.test.tsx
git commit -m "feat(gui): add useHtmlPreviewEvents listener for preview:open"
```

---

### Task 6: MessageBubble 集成（链接分流 + html 块预览按钮）

**Files:**
- Modify: `frontend/src/components/MessageBubble.tsx`
- Modify: `frontend/src/components/MessageBubble.test.tsx`（追加 describe 块）

两处改动：
1. ReactMarkdown 加 `components={{ a }}` 覆盖：点链接 `preventDefault` 后走 `openLink`（防止 Wails webview 整页跳走）。
2. 助手消息里抽出 ```` ```html ```` 围栏块（正则），在气泡底部为每个块渲染"预览 HTML"按钮 → `usePreviewStore.open({kind:'html', html, title:'HTML 预览'})`。

- [ ] **Step 1: 写失败测试（追加到现有测试文件末尾）**

```tsx
// 追加到 frontend/src/components/MessageBubble.test.tsx
import { vi } from 'vitest'
import { fireEvent } from '@testing-library/react'
import { usePreviewStore } from '../stores/previewStore'

const runtimeMocks = vi.hoisted(() => ({ BrowserOpenURL: vi.fn() }))
vi.mock('../../wailsjs/runtime/runtime', () => runtimeMocks)

describe('MessageBubble link routing', () => {
  it('routes http links to the system browser instead of navigating', () => {
    runtimeMocks.BrowserOpenURL.mockClear()
    render(<MessageBubble message={{ id: 'a1', role: 'assistant', content: '看 [这里](https://example.com)' }} />)
    fireEvent.click(screen.getByText('这里'))
    expect(runtimeMocks.BrowserOpenURL).toHaveBeenCalledWith('https://example.com')
  })
})

describe('MessageBubble html preview button', () => {
  beforeEach(() => usePreviewStore.getState().close())

  it('offers a preview button for a fenced html block and opens it', () => {
    const md = '前言\n```html\n<h1>Hi</h1>\n```\n'
    render(<MessageBubble message={{ id: 'a2', role: 'assistant', content: md }} />)
    fireEvent.click(screen.getByRole('button', { name: '预览 HTML' }))
    const src = usePreviewStore.getState().source
    expect(src?.kind).toBe('html')
    expect(src?.html).toContain('<h1>Hi</h1>')
  })

  it('shows no preview button when there is no html block', () => {
    render(<MessageBubble message={{ id: 'a3', role: 'assistant', content: '```ts\nconst x=1\n```' }} />)
    expect(screen.queryByRole('button', { name: '预览 HTML' })).toBeNull()
  })
})
```

- [ ] **Step 2: 跑测试确认失败**

Run: `cd frontend && npx vitest run src/components/MessageBubble.test.tsx`
Expected: FAIL — 新增 3 个用例失败（无 openLink 路由 / 无预览按钮）

- [ ] **Step 3: 实现**

在 `MessageBubble.tsx` 顶部 import 区加：

```tsx
import { openLink } from '../lib/openLink'
import { usePreviewStore } from '../stores/previewStore'
```

在 `downloadMarkdown` 下方加 html 块抽取工具：

```tsx
// extractHtmlBlocks pulls the body of every ```html fenced block out of raw
// markdown. Runs on message.content (not the rendered/highlighted output) so it
// stays independent of Shiki. Returns [] when there are none.
function extractHtmlBlocks(content: string): string[] {
  const re = /```html\r?\n([\s\S]*?)```/g
  const out: string[] = []
  let m: RegExpExecArray | null
  while ((m = re.exec(content)) !== null) out.push(m[1].replace(/\r?\n$/, ''))
  return out
}

// LinkRenderer routes anchor clicks through openLink: in a Wails webview a bare
// <a href> would navigate the whole single-page app away. http(s) opens in the
// system browser; other schemes are ignored (see openLink).
function LinkRenderer({ href, children }: { href?: string; children?: React.ReactNode }) {
  return (
    <a
      href={href}
      onClick={(e) => {
        e.preventDefault()
        if (href) openLink(href)
      }}
      className="cursor-pointer underline"
    >
      {children}
    </a>
  )
}
```

把助手分支的 ReactMarkdown 调用加上 `components`：

```tsx
          <ReactMarkdown
            remarkPlugins={[remarkGfm]}
            rehypePlugins={[rehypeShikiPlugin]}
            components={{ a: LinkRenderer }}
          >
            {message.content || (message.streaming ? '▋' : '')}
          </ReactMarkdown>
```

在助手分支的 `</div>`（prose 容器）之后、meta 行之前，插入预览按钮行：

```tsx
      {isAssistant && (() => {
        const htmlBlocks = extractHtmlBlocks(message.content)
        if (htmlBlocks.length === 0) return null
        return (
          <div className="mt-2 flex flex-wrap gap-1">
            {htmlBlocks.map((html, i) => (
              <button
                key={i}
                type="button"
                className="interactive rounded border border-border px-2 py-0.5 text-xs text-muted-foreground hover:bg-background hover:text-foreground"
                onClick={() => usePreviewStore.getState().open({ kind: 'html', html, title: 'HTML 预览' })}
              >
                预览 HTML{htmlBlocks.length > 1 ? ` ${i + 1}` : ''}
              </button>
            ))}
          </div>
        )
      })()}
```

- [ ] **Step 4: 跑测试确认通过**

Run: `cd frontend && npx vitest run src/components/MessageBubble.test.tsx`
Expected: PASS（原有 + 新增全绿）

- [ ] **Step 5: 提交**

```bash
git add frontend/src/components/MessageBubble.tsx frontend/src/components/MessageBubble.test.tsx
git commit -m "feat(gui): route chat links to system browser + html block preview button"
```

---

### Task 7: uiStore 加 rightView（三 tab 共享状态）

**Files:**
- Modify: `frontend/src/stores/uiStore.ts`
- Test: `frontend/src/stores/uiStore.test.ts`（新建）

把右列当前 tab 从 `RightPanel` 局部 state 提到 store，使"新预览到达时自动切到预览 tab"和"关闭时切回状态 tab"可跨组件驱动。

- [ ] **Step 1: 写失败测试**

```ts
// frontend/src/stores/uiStore.test.ts
import { describe, it, expect, beforeEach } from 'vitest'
import { useUIStore } from './uiStore'

describe('uiStore rightView', () => {
  beforeEach(() => useUIStore.getState().setRightView('status'))

  it('defaults to status', () => {
    expect(useUIStore.getState().rightView).toBe('status')
  })

  it('setRightView switches the active right column view', () => {
    useUIStore.getState().setRightView('preview')
    expect(useUIStore.getState().rightView).toBe('preview')
  })
})
```

- [ ] **Step 2: 跑测试确认失败**

Run: `cd frontend && npx vitest run src/stores/uiStore.test.ts`
Expected: FAIL — `rightView` / `setRightView` undefined

- [ ] **Step 3: 实现（改 uiStore.ts）**

在 `UIState` 接口加：

```ts
  rightView: RightView
  setRightView: (v: RightView) => void
```

在文件顶部（interface 之前）加类型：

```ts
// RightView selects which panel fills the right column: status tabs, the
// read-only agent browser view, or the HTML preview.
export type RightView = 'status' | 'browser' | 'preview'
```

在 `create<UIState>` 的返回对象里加初值与 setter：

```ts
  rightView: 'status',
  setRightView: (v) => set({ rightView: v }),
```

- [ ] **Step 4: 跑测试确认通过**

Run: `cd frontend && npx vitest run src/stores/uiStore.test.ts`
Expected: PASS (2 tests)

- [ ] **Step 5: 提交**

```bash
git add frontend/src/stores/uiStore.ts frontend/src/stores/uiStore.test.ts
git commit -m "feat(gui): lift right-column active view into uiStore"
```

---

### Task 8: App 集成（预览 tab + 自动切换 + 挂载 hook）

**Files:**
- Modify: `frontend/src/App.tsx`
- Test: `frontend/src/App.test.tsx`（新建，聚焦 RightPanel 行为）

`RightPanel` 改为读 `uiStore.rightView`；新增"预览"tab 渲染 `WebPreviewPanel`；用 effect 在 `previewStore.source` 变非空时切到 preview tab；面板 `onClose` 清 source 并切回 status。App 顶层挂 `useHtmlPreviewEvents`。

- [ ] **Step 1: 写失败测试**

```tsx
// frontend/src/App.test.tsx
import { describe, it, expect, vi, beforeEach } from 'vitest'
import { render, screen, act } from '@testing-library/react'

// App pulls in many bound methods/hooks; stub the wails layer wholesale.
vi.mock('../wailsjs/go/main/App', () => ({
  ReadHTMLFile: vi.fn(),
  ListSessions: vi.fn().mockResolvedValue([]),
  ServeStatus: vi.fn().mockResolvedValue({}),
  ListAgents: vi.fn().mockResolvedValue([]),
}))
vi.mock('../wailsjs/runtime/runtime', () => ({
  EventsOn: vi.fn(() => () => {}),
  EventsOff: vi.fn(),
  BrowserOpenURL: vi.fn(),
}))

import App from './App'
import { usePreviewStore } from './stores/previewStore'
import { useUIStore } from './stores/uiStore'

describe('App right column preview integration', () => {
  beforeEach(() => {
    usePreviewStore.getState().close()
    useUIStore.getState().setRightView('status')
  })

  it('auto-switches to the preview tab when a preview opens', () => {
    render(<App />)
    act(() => {
      usePreviewStore.getState().open({ kind: 'html', html: '<h1>hi</h1>', title: '报告' })
    })
    expect(useUIStore.getState().rightView).toBe('preview')
    expect(screen.getByTitle('HTML 预览内容')).toBeInTheDocument()
  })
})
```

- [ ] **Step 2: 跑测试确认失败**

Run: `cd frontend && npx vitest run src/App.test.tsx`
Expected: FAIL — 无预览 tab / 未自动切换

- [ ] **Step 3: 实现（改 App.tsx）**

顶部 import 追加：

```tsx
import { WebPreviewPanel } from './components/WebPreviewPanel'
import { usePreviewStore } from './stores/previewStore'
import { useHtmlPreviewEvents } from './hooks/useHtmlPreviewEvents'
import { useEffect } from 'react'
import type { RightView } from './stores/uiStore'
```

（`useState` import 保留；`RightView` 类型改从 uiStore 引入，删掉本文件里原 `type RightView = ...` 局部定义。）

`RightPanel` 重写为读 store：

```tsx
function RightPanel() {
  const view = useUIStore((s) => s.rightView)
  const setView = useUIStore((s) => s.setRightView)
  const source = usePreviewStore((s) => s.source)
  const closePreview = usePreviewStore((s) => s.close)

  // A newly-opened preview pulls the right column to its tab so backend-pushed
  // or button-triggered previews surface without a manual click.
  useEffect(() => {
    if (source) setView('preview')
  }, [source, setView])

  const views: { id: RightView; label: string }[] = [
    { id: 'status', label: '状态' },
    { id: 'browser', label: '浏览器' },
    { id: 'preview', label: '预览' },
  ]
  return (
    <div className="flex flex-col h-full">
      <div className="flex border-b border-border">
        {views.map((v) => (
          <button
            key={v.id}
            type="button"
            role="tab"
            aria-selected={view === v.id}
            className={cn(
              'interactive flex-1 py-1.5 text-xs font-medium',
              view === v.id
                ? 'text-foreground border-b-2 border-primary'
                : 'text-muted-foreground hover:text-foreground hover:bg-muted'
            )}
            onClick={() => setView(v.id)}
          >
            {v.label}
          </button>
        ))}
      </div>
      <div className="flex-1 min-h-0">
        {view === 'status' && <StatusPanel />}
        {view === 'browser' && <BrowserView />}
        {view === 'preview' && (
          <WebPreviewPanel
            source={source}
            onClose={() => {
              closePreview()
              setView('status')
            }}
          />
        )}
      </div>
    </div>
  )
}
```

在 `App()` 函数体内，`useBrowserSession()` 旁挂新监听：

```tsx
  useBrowserSession()
  useHtmlPreviewEvents()
```

- [ ] **Step 4: 跑测试确认通过**

Run: `cd frontend && npx vitest run src/App.test.tsx`
Expected: PASS

- [ ] **Step 5: 全量前端测试 + 类型检查**

Run: `cd frontend && npx vitest run && npx tsc --noEmit`
Expected: 全绿；无类型错误

- [ ] **Step 6: 提交**

```bash
git add frontend/src/App.tsx frontend/src/App.test.tsx
git commit -m "feat(gui): wire HTML preview tab into right column with auto-switch"
```

---

### Task 9: 全量校验 + 手动验证

**Files:** 无（验证）

- [ ] **Step 1: Go 全量**

Run: `go build ./... && go vet ./... && go test ./... && gofmt -l .`
Expected: 编译/vet/test 全绿；gofmt 输出为空

- [ ] **Step 2: 前端全量**

Run: `cd frontend && npx vitest run && npx tsc --noEmit && npm run build`
Expected: 测试全绿、无类型错误、build 成功

- [ ] **Step 3: 手动验证（wails dev）**

Run: `wails dev`（在 `legion/legionAgentGUI/`）
逐条确认：
1. 助手消息含 ```` ```html ```` 块 → 底部出现"预览 HTML"按钮 → 点击 → 右列自动切到"预览"tab、iframe 渲染出该 HTML。
2. 预览面板"关闭"按钮 → 清空并切回"状态"tab。
3. 助手消息里的 http 链接 → 点击弹系统浏览器，GUI 本身不跳走。
4. iframe 内含 `<script>alert(1)</script>` 的 HTML → 脚本**不执行**（sandbox 生效）。

> 触发链 4（agent 工具主动推送 `preview:open` + localFile 打开）依赖 server 仓 legionAgent 侧新增发射端，属本计划**范围外**的后续任务；前端监听（Task 5）与绑定（Task 4）已就绪，server 侧接线时可直接 `runtime.EventsEmit(ctx, "preview:open", payload)` 驱动。

- [ ] **Step 4: 收尾提交（若手动验证有微调）**

```bash
git add -A
git commit -m "chore(gui): finalize HTML preview panel after manual verification"
```

---

## 范围外（后续任务，非本计划）

- **legionAgent server 仓**：新增一个 agent 工具/能力，在合适时机 `runtime.EventsEmit(ctx, "preview:open", {kind, html|path, title, sourceUrl})` 驱动前端弹出预览（触发链 4 的发射端）。
- 本地 HTML 若出现"带 JS 图表的报告"需求 → 对 `localFile` 单独放开 iframe `allow-scripts`（需独立安全评审）。
- `ReadHTMLFile` 的工作目录根限定（当前仅做扩展名 + traversal + 普通文件校验）。

---

## Self-Review

**Spec 覆盖：**
- iframe srcdoc 静态渲染 → Task 3 ✓
- sandbox="" 零脚本 → Task 3（测试断言无 allow-scripts）✓
- 远程 URL 走外部浏览器 → Task 2 openLink + Task 6 链接分流 ✓
- 完整 HTML / agent 富 HTML → Task 6 预览按钮 + Task 5 事件 ✓
- 本地 HTML 文件 → Task 4 ReadHTMLFile + Task 5 localFile 分支 ✓
- 右侧分栏面板 → Task 8 预览 tab（复用现有 status 列）✓
- 触发链：链接/代码块按钮/agent 推送 → Task 6 / Task 6 / Task 5 ✓
- ReadHTMLFile 路径校验 fail-loud → Task 4（3 条错误路径测试）✓
- 无地址栏 → Task 3 工具条仅标题/外部打开/关闭 ✓

**类型一致性：** `PreviewSource`（Task 1）在 Task 3/5/6/8 一致引用；`RightView`（Task 7）在 Task 8 引用；`ReadHTMLFile`（Task 4 Go / 绑定）在 Task 5 调用签名 `(path)=>Promise<string>` 一致。

**占位符：** 无 TBD/TODO；每个代码步骤含完整代码。
