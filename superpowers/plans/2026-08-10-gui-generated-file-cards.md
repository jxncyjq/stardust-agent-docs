# GUI 对话生成文件卡片 Implementation Plan（子项目 B）

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`).

**Goal:** GUI 消费后端 `generated_files`，在 assistant 消息下渲染文件卡片：预览（可预览格式走 /v1/files URL → PreviewContent）、下载（另存拷贝）、外部打开（OS 默认程序）、复制链接。

**Architecture:** 纯 legionAgentGUI 前端 + 2 个 GUI-local Go 绑定（`OpenPath`/`SaveGeneratedFile`，根限定，复用 `resolveInRoot`）。数据经 `GetTaskResult`/`GetSessionTurns`（返 map，`generated_files` 自动透传）挂到 `Message.generatedFiles`。预览复用 PreviewContent/previewStore/预览 tab，token 从 `GetBrowserEndpoint` 活取。legionAgent 后端零改动（子项目 A 已提供）。

**Tech Stack:** React 18 + TS + zustand + Vitest；Go + Wails v2。

**Spec:** `docs/superpowers/specs/2026-08-10-gui-generated-file-cards-design.md`（在 docs 仓）

**仓库根：** 前端相对 `legion/legionAgentGUI/frontend/`，Go 相对 `legion/legionAgentGUI/`。

## CRITICAL：vitest 在 frontend/ 跑
`cd legion/legionAgentGUI/frontend` 再 `npx vitest`（正确版本 `v2.1.9`）。父目录 vitest 无 jsdom → `document is not defined` 假失败。Go 命令在 `legionAgentGUI/` 跑。

---

## 文件结构

| 文件 | 改动 |
|---|---|
| `app_workspace.go` | 加 `OpenPath` + `SaveGeneratedFile`（根限定，复用 `resolveInRoot`）|
| `frontend/src/stores/chatStore.ts` | `GeneratedFile` 类型 + `Message.generatedFiles` |
| `frontend/src/lib/generatedFiles.ts` | `mapGeneratedFiles(raw)` 映射 + `isPreviewable(name)` |
| `frontend/src/lib/fetchPreview.ts` | card.url + token fetch → PreviewSource |
| `frontend/src/components/FileCard.tsx` | 单卡片 + 动作 | 新建 |
| `frontend/src/components/FileCardList.tsx` | 卡片列表 | 新建 |
| `frontend/src/components/MessageBubble.tsx` | assistant 气泡挂 FileCardList |
| `frontend/src/components/ChatPanel.tsx` | 实时/历史装配 generatedFiles |

---

### Task 1: Go 绑定 OpenPath + SaveGeneratedFile

**Files:** `app_workspace.go`；`app_workspace_test.go`（追加）

**Context:** GUI-local，根限定。`resolveInRoot(root, target) (string, error)`（app_workspace.go:30）已存在，复用。`OpenPath` 用 Windows `start`（不走 shell）；`SaveGeneratedFile` 用 `runtime.SaveFileDialog(a.ctx, ...)` + `io.Copy`。仿 hermes（shell.openPath / showSaveDialog+copy）。

- [ ] **Step 1: 失败测试** — argv 构造与根限定抽纯函数便于测；对话框/exec 副作用不在单测跑：
```go
func TestOpenPathRejectsOutsideRoot(t *testing.T) {
	a := NewApp("")
	if err := a.OpenPath(t.TempDir(), "../escape.txt"); err == nil {
		t.Fatal("expected error for path outside root")
	}
}
func TestSaveGeneratedFileRejectsOutsideRoot(t *testing.T) {
	a := NewApp("")
	if err := a.SaveGeneratedFile(t.TempDir(), "../../etc/passwd"); err == nil {
		t.Fatal("expected error for path outside root")
	}
}
func TestOpenPathArgv(t *testing.T) {
	// abs 路径作独立 argv，不拼 shell
	argv := openPathArgv(`C:\a b\x.docx`)
	if len(argv) < 3 || argv[0] != "cmd" || argv[len(argv)-1] != `C:\a b\x.docx` {
		t.Fatalf("got %#v", argv)
	}
}
```
（先读 `app_workspace.go` 确认 `resolveInRoot` 签名 + `NewApp`；根限定测试须真触发 resolveInRoot 的越权分支——用存在的 root TempDir + `..` relPath。）

- [ ] **Step 2: 确认失败** `go test ./... -run 'OpenPath|SaveGeneratedFile'`（在 legionAgentGUI/）。

- [ ] **Step 3: 实现** — `app_workspace.go` 追加（`os/exec`/`io`/`os` 按需 import；`runtime` 是 `github.com/wailsapp/wails/v2/pkg/runtime`，app.go 已 import）：
```go
// openPathArgv builds the argv to open a file with its OS-default program on
// Windows via `cmd /c start "" <path>` — the empty "" is start's title slot so a
// quoted path is not mistaken for a window title. abs is passed as ONE argv
// element (no shell string building), so path metacharacters cannot inject.
func openPathArgv(abs string) []string {
	return []string{"cmd", "/c", "start", "", abs}
}

// OpenPath opens a generated file with the OS-default program (Word/Excel/…),
// confined to the session workspace root. Fail-loud on out-of-root / launch err.
func (a *App) OpenPath(root, relPath string) error {
	abs, err := resolveInRoot(root, relPath)
	if err != nil {
		return fmt.Errorf("open path: %w", err)
	}
	argv := openPathArgv(abs)
	if err := exec.Command(argv[0], argv[1:]...).Start(); err != nil {
		return fmt.Errorf("open %q with default program: %w", abs, err)
	}
	return nil
}

// SaveGeneratedFile prompts for a destination and copies the generated file
// there (Save As), confined to the workspace root. A cancelled dialog (empty
// path) is a legitimate optional, not an error. Fail-loud on out-of-root / IO.
func (a *App) SaveGeneratedFile(root, relPath string) error {
	abs, err := resolveInRoot(root, relPath)
	if err != nil {
		return fmt.Errorf("save file: %w", err)
	}
	dest, err := runtime.SaveFileDialog(a.ctx, runtime.SaveDialogOptions{
		DefaultFilename: filepath.Base(abs),
		Title:           "保存文件",
	})
	if err != nil {
		return fmt.Errorf("save dialog for %q: %w", abs, err)
	}
	if dest == "" {
		return nil // user cancelled — legitimate optional
	}
	src, err := os.Open(abs)
	if err != nil {
		return fmt.Errorf("open source %q: %w", abs, err)
	}
	defer src.Close()
	out, err := os.Create(dest)
	if err != nil {
		return fmt.Errorf("create dest %q: %w", dest, err)
	}
	defer out.Close()
	if _, err := io.Copy(out, src); err != nil {
		return fmt.Errorf("copy %q to %q: %w", abs, dest, err)
	}
	return nil
}
```
   若 `resolveInRoot` 签名/行为不同(如返回 abs 已含校验)，对齐使用。

- [ ] **Step 4: 确认通过** `go test ./... -run 'OpenPath|SaveGeneratedFile' -v`；`go build ./...`；`gofmt -l app_workspace.go app_workspace_test.go`（空）。

- [ ] **Step 5: 重生成绑定** `wails generate module` → 确认 `frontend/wailsjs/go/main/App.d.ts` 有 `OpenPath`/`SaveGeneratedFile`。无 CLI 则手动补（仿现有导出）。

- [ ] **Step 6: 提交**
```bash
git add app_workspace.go app_workspace_test.go frontend/wailsjs/go/main/App.js frontend/wailsjs/go/main/App.d.ts
git commit -m "feat(gui): add OpenPath + SaveGeneratedFile bindings for file cards"
```

---

### Task 2: chatStore — Message.generatedFiles + 映射/判定 helper

**Files:** `frontend/src/stores/chatStore.ts`；`frontend/src/lib/generatedFiles.ts` + `.test.ts`

- [ ] **Step 1: 失败测试** `frontend/src/lib/generatedFiles.test.ts`：
```ts
import { describe, it, expect } from 'vitest'
import { mapGeneratedFiles, isPreviewable } from './generatedFiles'

describe('mapGeneratedFiles', () => {
  it('maps backend snake_case to GeneratedFile', () => {
    const out = mapGeneratedFiles([{ path: 'a/b.html', url: '/v1/files?x', download_url: '/v1/files?x&download=1', name: 'b.html' }])
    expect(out).toEqual([{ path: 'a/b.html', url: '/v1/files?x', downloadUrl: '/v1/files?x&download=1', name: 'b.html' }])
  })
  it('returns [] for missing/nonarray', () => {
    expect(mapGeneratedFiles(undefined)).toEqual([])
    expect(mapGeneratedFiles(null)).toEqual([])
  })
})

describe('isPreviewable', () => {
  it('true for html/md/text/code/image', () => {
    for (const n of ['a.html','a.md','a.txt','a.ts','a.png','a.svg']) expect(isPreviewable(n)).toBe(true)
  })
  it('false for office/binary', () => {
    for (const n of ['a.docx','a.xlsx','a.pptx','a.pdf','a.zip']) expect(isPreviewable(n)).toBe(false)
  })
})
```

- [ ] **Step 2: 确认失败** `cd frontend && npx vitest run src/lib/generatedFiles.test.ts`.

- [ ] **Step 3: 实现**
  - `frontend/src/lib/generatedFiles.ts`：
```ts
export interface GeneratedFile {
  path: string
  url: string
  downloadUrl: string
  name: string
}

// mapGeneratedFiles converts the backend's generated_files (snake_case
// download_url) into GeneratedFile[]. Non-array input yields [] — a task that
// wrote no files is a legitimate empty, not an error.
export function mapGeneratedFiles(raw: unknown): GeneratedFile[] {
  if (!Array.isArray(raw)) return []
  return raw.map((r) => {
    const o = r as Record<string, unknown>
    return {
      path: String(o.path ?? ''),
      url: String(o.url ?? ''),
      downloadUrl: String(o.download_url ?? ''),
      name: String(o.name ?? ''),
    }
  })
}

const PREVIEWABLE = new Set([
  'html','htm','md','markdown','txt','log','json','ts','tsx','js','jsx','go','py','sh','css','yaml','yml','toml',
  'png','jpg','jpeg','gif','webp','svg','bmp',
])

// isPreviewable decides whether a file can render in the in-app preview (vs
// download/open-external only). Extension-based; office/binary → false.
export function isPreviewable(name: string): boolean {
  const ext = name.split('.').pop()?.toLowerCase() ?? ''
  return PREVIEWABLE.has(ext)
}
```
  - `chatStore.ts`：`import type { GeneratedFile } from '../lib/generatedFiles'`；`Message` 加：
```ts
  // generatedFiles are files the task produced (write_file), surfaced as cards
  // under the assistant bubble. Absent on user/system messages and history
  // predating the field.
  generatedFiles?: GeneratedFile[]
```

- [ ] **Step 4: 确认通过** `cd frontend && npx vitest run src/lib/generatedFiles.test.ts`；`npx tsc --noEmit`。

- [ ] **Step 5: 提交**
```bash
git add frontend/src/lib/generatedFiles.ts frontend/src/lib/generatedFiles.test.ts frontend/src/stores/chatStore.ts
git commit -m "feat(gui): add GeneratedFile type + mapping/previewable helpers"
```

---

### Task 3: fetchPreview — URL 加载建 PreviewSource

**Files:** `frontend/src/lib/fetchPreview.ts` + `.test.ts`

**Context:** 拿 `card.url`（相对→用 GetBrowserEndpoint 的 baseURL 拼、绝对原样）+ token fetch → 按 content-type/扩展名建 `PreviewSource`（previewStore 的联合类型）。`GetBrowserEndpoint()` 返 `{baseURL, token}`（wailsjs 绑定）。

- [ ] **Step 1: 失败测试** `frontend/src/lib/fetchPreview.test.ts`：
```ts
import { describe, it, expect, vi, beforeEach } from 'vitest'

const appMocks = vi.hoisted(() => ({ GetBrowserEndpoint: vi.fn() }))
vi.mock('../../wailsjs/go/main/App', () => appMocks)

import { fetchPreview } from './fetchPreview'

beforeEach(() => {
  appMocks.GetBrowserEndpoint.mockReset()
  appMocks.GetBrowserEndpoint.mockResolvedValue({ baseURL: 'http://127.0.0.1:9000', token: 'tok' })
  vi.stubGlobal('fetch', vi.fn())
})

it('resolves relative url against baseURL and sends bearer token', async () => {
  ;(fetch as any).mockResolvedValue(new Response('<h1>hi</h1>', { headers: { 'Content-Type': 'text/html' } }))
  const src = await fetchPreview({ path: 'a.html', url: '/v1/files?x', downloadUrl: '', name: 'a.html' })
  expect((fetch as any).mock.calls[0][0]).toBe('http://127.0.0.1:9000/v1/files?x')
  expect((fetch as any).mock.calls[0][1].headers.Authorization).toBe('Bearer tok')
  expect(src).toEqual({ kind: 'html', html: '<h1>hi</h1>', title: 'a.html', sourceUrl: '/v1/files?x' })
})

it('builds markdown source for .md', async () => {
  ;(fetch as any).mockResolvedValue(new Response('# t', { headers: { 'Content-Type': 'text/markdown' } }))
  const src = await fetchPreview({ path: 'd.md', url: '/v1/files?y', downloadUrl: '', name: 'd.md' })
  expect(src.kind).toBe('markdown')
})

it('builds code source for .ts', async () => {
  ;(fetch as any).mockResolvedValue(new Response('const x=1', { headers: { 'Content-Type': 'text/plain' } }))
  const src = await fetchPreview({ path: 'a.ts', url: '/v1/files?z', downloadUrl: '', name: 'a.ts' })
  expect(src.kind).toBe('code')
})

it('throws on non-ok response', async () => {
  ;(fetch as any).mockResolvedValue(new Response('nope', { status: 404 }))
  await expect(fetchPreview({ path: 'a.html', url: '/v1/files?x', downloadUrl: '', name: 'a.html' })).rejects.toThrow()
})
```

- [ ] **Step 2: 确认失败** `cd frontend && npx vitest run src/lib/fetchPreview.test.ts`.

- [ ] **Step 3: 实现** `frontend/src/lib/fetchPreview.ts`：
```ts
import { GetBrowserEndpoint } from '../../wailsjs/go/main/App'
import type { PreviewSource } from '../stores/previewStore'
import type { GeneratedFile } from './generatedFiles'

const LANG_BY_EXT: Record<string, string> = {
  ts: 'typescript', tsx: 'tsx', js: 'javascript', jsx: 'jsx', go: 'go', json: 'json',
  sh: 'bash', py: 'python', css: 'css', yaml: 'yaml', yml: 'yaml', toml: 'toml',
}
const IMAGE_EXT = new Set(['png','jpg','jpeg','gif','webp','svg','bmp'])

function extOf(name: string): string {
  return name.split('.').pop()?.toLowerCase() ?? ''
}

// fetchPreview loads a generated file via the backend /v1/files endpoint (with
// the current loopback token) and maps it to a PreviewSource for PreviewContent.
// Relative urls are resolved against the live baseURL from GetBrowserEndpoint so
// a serve restart (new port) never breaks the link. Throws on non-2xx / network.
export async function fetchPreview(file: GeneratedFile): Promise<PreviewSource> {
  const { baseURL, token } = await GetBrowserEndpoint()
  const full = /^https?:\/\//i.test(file.url) ? file.url : baseURL + file.url
  const resp = await fetch(full, token ? { headers: { Authorization: 'Bearer ' + token } } : {})
  if (!resp.ok) {
    throw new Error(`preview fetch ${file.name} failed: ${resp.status}`)
  }
  const ext = extOf(file.name)
  if (IMAGE_EXT.has(ext)) {
    const blob = await resp.blob()
    const dataUri = await new Promise<string>((resolve, reject) => {
      const fr = new FileReader()
      fr.onload = () => resolve(String(fr.result))
      fr.onerror = () => reject(fr.error)
      fr.readAsDataURL(blob)
    })
    return { kind: 'image', dataUri, title: file.name, path: file.path }
  }
  const text = await resp.text()
  if (ext === 'html' || ext === 'htm') {
    return { kind: 'html', html: text, title: file.name, sourceUrl: file.url }
  }
  if (ext === 'md' || ext === 'markdown') {
    return { kind: 'markdown', text, title: file.name, path: file.path }
  }
  return { kind: 'code', text, lang: LANG_BY_EXT[ext] ?? 'text', title: file.name, path: file.path }
}
```

- [ ] **Step 4: 确认通过** `cd frontend && npx vitest run src/lib/fetchPreview.test.ts`；`npx tsc --noEmit`。

- [ ] **Step 5: 提交**
```bash
git add frontend/src/lib/fetchPreview.ts frontend/src/lib/fetchPreview.test.ts
git commit -m "feat(gui): add fetchPreview building PreviewSource from /v1/files url"
```

---

### Task 4: FileCard + FileCardList 组件

**Files:** `frontend/src/components/FileCard.tsx`、`FileCardList.tsx` + 各 `.test.tsx`

**Context:** 卡片:图标+名+类型+动作。可预览→"预览"(fetchPreview→previewStore.open);全格式→下载(SaveGeneratedFile)/外部打开(OpenPath)/复制链接(clipboard)。root=会话 workingDir（`sessionStore.currentSession.workingDir`）。

- [ ] **Step 1: 失败测试** `FileCard.test.tsx`（mock App 绑定 + previewStore + sessionStore + fetchPreview）：
```tsx
import { describe, it, expect, vi, beforeEach } from 'vitest'
import { render, screen, fireEvent, waitFor } from '@testing-library/react'

const appMocks = vi.hoisted(() => ({ OpenPath: vi.fn(), SaveGeneratedFile: vi.fn(), GetBrowserEndpoint: vi.fn() }))
vi.mock('../../wailsjs/go/main/App', () => appMocks)
const previewMock = vi.hoisted(() => ({ fetchPreview: vi.fn() }))
vi.mock('../lib/fetchPreview', () => previewMock)

import { FileCard } from './FileCard'
import { usePreviewStore } from '../stores/previewStore'
import { useSessionStore } from '../stores/sessionStore'

beforeEach(() => {
  Object.values(appMocks).forEach((m) => m.mockReset())
  previewMock.fetchPreview.mockReset()
  usePreviewStore.getState().close()
  useSessionStore.setState({ currentSessionId: 's1', sessions: [{ id: 's1', project: 'p', title: 't', archived: false, updatedAt: '', workingDir: 'F:/w' }] })
})

const html = { path: 'a.html', url: '/v1/files?x', downloadUrl: '/v1/files?x&download=1', name: 'a.html' }
const docx = { path: 'r.docx', url: '/v1/files?d', downloadUrl: '/v1/files?d&download=1', name: 'r.docx' }

it('shows name + preview action for previewable', () => {
  render(<FileCard file={html} />)
  expect(screen.getByText('a.html')).toBeInTheDocument()
  expect(screen.getByRole('button', { name: '预览' })).toBeInTheDocument()
})

it('hides preview for office', () => {
  render(<FileCard file={docx} />)
  expect(screen.queryByRole('button', { name: '预览' })).toBeNull()
})

it('preview opens PreviewContent via fetchPreview', async () => {
  previewMock.fetchPreview.mockResolvedValue({ kind: 'html', html: '<h1>h</h1>', title: 'a.html' })
  render(<FileCard file={html} />)
  fireEvent.click(screen.getByRole('button', { name: '预览' }))
  await waitFor(() => expect(usePreviewStore.getState().source?.kind).toBe('html'))
})

it('open-external calls OpenPath with workingDir + path', () => {
  render(<FileCard file={docx} />)
  fireEvent.click(screen.getByRole('button', { name: '外部打开' }))
  expect(appMocks.OpenPath).toHaveBeenCalledWith('F:/w', 'r.docx')
})

it('download calls SaveGeneratedFile', () => {
  render(<FileCard file={docx} />)
  fireEvent.click(screen.getByRole('button', { name: '下载' }))
  expect(appMocks.SaveGeneratedFile).toHaveBeenCalledWith('F:/w', 'r.docx')
})

it('copy link writes url to clipboard', async () => {
  const writeText = vi.fn().mockResolvedValue(undefined)
  vi.stubGlobal('navigator', { clipboard: { writeText } })
  render(<FileCard file={html} />)
  fireEvent.click(screen.getByRole('button', { name: '复制链接' }))
  expect(writeText).toHaveBeenCalledWith('/v1/files?x')
})
```
`FileCardList.test.tsx`：渲染多卡片、空数组不渲染。

- [ ] **Step 2: 确认失败** `cd frontend && npx vitest run src/components/FileCard.test.tsx src/components/FileCardList.test.tsx`.

- [ ] **Step 3: 实现**
  - `FileCard.tsx`：读 `useSessionStore` 当前会话 workingDir 作 root；`isPreviewable(file.name)` 决定是否显"预览"。动作按钮：
    - 预览 → `fetchPreview(file).then(usePreviewStore.getState().open).catch(console.error)`
    - 下载 → `SaveGeneratedFile(root, file.path).catch(console.error)`
    - 外部打开 → `OpenPath(root, file.path).catch(console.error)`
    - 复制链接 → `navigator.clipboard.writeText(file.url)`
    - 图标按扩展名(复用/补 icons)；无 workingDir 时下载/打开禁用(root 空)。
  - `FileCardList.tsx`：`files.length===0 → null`；否则 `flex flex-wrap gap` 渲染 `FileCard`。
  （动作可放一行按钮或下拉;测试按 role name 找按钮，保持可访问 aria-label='预览'/'下载'/'外部打开'/'复制链接'。）

- [ ] **Step 4: 确认通过** `cd frontend && npx vitest run src/components/FileCard.test.tsx src/components/FileCardList.test.tsx`；`npx tsc --noEmit`。

- [ ] **Step 5: 提交**
```bash
git add frontend/src/components/FileCard.tsx frontend/src/components/FileCard.test.tsx frontend/src/components/FileCardList.tsx frontend/src/components/FileCardList.test.tsx frontend/src/components/icons.tsx
git commit -m "feat(gui): FileCard + FileCardList with preview/download/open/copy actions"
```

---

### Task 5: MessageBubble 挂 FileCardList

**Files:** `frontend/src/components/MessageBubble.tsx`；`MessageBubble.test.tsx`（追加）

- [ ] **Step 1: 失败测试**（追加）：
```tsx
describe('MessageBubble generated file cards', () => {
  it('renders a card for an assistant message with generatedFiles', () => {
    render(<MessageBubble message={{ id: 'a1', role: 'assistant', content: 'done', generatedFiles: [{ path: 'a.html', url: '/v1/files?x', downloadUrl: '/v1/files?x&download=1', name: 'a.html' }] }} />)
    expect(screen.getByText('a.html')).toBeInTheDocument()
  })
  it('renders no card area when none', () => {
    render(<MessageBubble message={{ id: 'a2', role: 'assistant', content: 'hi' }} />)
    expect(screen.queryByText('a.html')).toBeNull()
  })
})
```
（FileCard 内部会调 App 绑定/sessionStore——测试文件顶部补 mock：`vi.mock('../../wailsjs/go/main/App', ...)` 至少 stub `OpenPath/SaveGeneratedFile/GetBrowserEndpoint`，并 seed sessionStore，避免渲染崩。参照 Task 4 的 mock。）

- [ ] **Step 2: 确认失败** `cd frontend && npx vitest run src/components/MessageBubble.test.tsx`.

- [ ] **Step 3: 实现** — MessageBubble import `FileCardList`；在 assistant 分支的 prose `</div>` 后、meta 行前(与 html 预览按钮同区)插：
```tsx
      {isAssistant && message.generatedFiles && message.generatedFiles.length > 0 && (
        <FileCardList files={message.generatedFiles} />
      )}
```

- [ ] **Step 4: 确认通过** `cd frontend && npx vitest run src/components/MessageBubble.test.tsx`；`npx tsc --noEmit`。

- [ ] **Step 5: 提交**
```bash
git add frontend/src/components/MessageBubble.tsx frontend/src/components/MessageBubble.test.tsx
git commit -m "feat(gui): render generated file cards under assistant bubble"
```

---

### Task 6: ChatPanel 装配（实时 + 历史）

**Files:** `frontend/src/components/ChatPanel.tsx`；`ChatPanel.test.tsx`（追加）

**Context:** 两处挂 generatedFiles 到 Message：① 实时 `GetTaskResult`（~183 settle 载荷 → 最终写 message 处）② 历史 `GetSessionTurns`（~383/518 `addMessage`）。`import { mapGeneratedFiles } from '../lib/generatedFiles'`。

- [ ] **Step 1: 失败测试**（追加，参照现有 ChatPanel.test mock）：
  - 历史：`GetSessionTurns` mock 返回带 `generated_files` 的 assistant turn → 该 message 有 `generatedFiles`（可断言 store 里 message 或渲染出文件名）。
  - 实时：`GetTaskResult` mock 返回 `generated_files` → 完成后的 assistant message 带 `generatedFiles`。
  先 `grep -n "settle\|GetTaskResult\|addMessage\|finalizeMessage\|generatedFiles" src/components/ChatPanel.tsx` 找准写 message 的落点。报告落点。

- [ ] **Step 2: 确认失败**。

- [ ] **Step 3: 实现**
  - 历史（~383 `addMessage({id,role,content,agent})`）加：`generatedFiles: mapGeneratedFiles((turn as any).generated_files)`（同 ~518 若有第二处）。
  - 实时：`GetTaskResult`（~178-186 settle 载荷）加 `generatedFiles: mapGeneratedFiles(res?.generated_files)` 到 settle 对象;在 settle 被消费、最终写/finalize assistant message 的地方（`grep finalizeMessage/appendToken/setMessages`）把 `generatedFiles` 一并写入该 message。若 settle 类型是内联对象，扩其类型定义。
  （保持向后兼容：无 generated_files → `[]` → 不渲染卡片区。）

- [ ] **Step 4: 确认通过** `cd frontend && npx vitest run src/components/ChatPanel.test.tsx`（含回归）；`npx tsc --noEmit`。

- [ ] **Step 5: 提交**
```bash
git add frontend/src/components/ChatPanel.tsx frontend/src/components/ChatPanel.test.tsx
git commit -m "feat(gui): attach generatedFiles to messages from result + history"
```

---

### Task 7: 全量校验 + 手动

- [ ] **Step 1: 前端全量** `cd frontend && npx vitest run && npx tsc --noEmit && npm run build`（全绿）。
- [ ] **Step 2: Go** `go build ./... && go vet ./... && go test ./...`（在 legionAgentGUI/，全绿）;`gofmt -l $(git diff --name-only master..HEAD | grep '\.go$')`（空）。
- [ ] **Step 3: 手动 `wails dev`**：让 agent 生成新 html/md/图片/docx →
  1. assistant 气泡下出现文件卡片
  2. html/md/图片点"预览"→ 右列切「预览」tab、渲染出内容
  3. docx"外部打开"→ Word 起;"下载"→ 另存对话框;"复制链接"→ 剪贴板得 url
  4. 切历史会话 → 卡片随历史重现
  5. office 卡片无"预览"按钮
- [ ] **Step 4: 收尾提交**（如微调）。

---

## 范围外（后续）
- 预览大文件进度/取消。
- 跨平台 OpenPath（当前 Windows `start`；mac `open`/linux `xdg-open`）。
- 卡片图标集完善。

---

## Self-Review

**Spec 覆盖：** Message.generatedFiles + 映射→T2;实时+历史装配→T6;卡片 UI→T4;挂气泡→T5;预览走 /v1/files URL→T3+T4;下载(SaveGeneratedFile)/外部打开(OpenPath)/复制链接→T1+T4;可预览判定→T2;office 无预览→T4。fail-loud(fetch 失败报错、根限定、取消=可选)贯穿。

**类型一致性：** `GeneratedFile`(T2)在 T3(fetchPreview)/T4(FileCard)/T5/T6 一致(downloadUrl 驼峰,后端 download_url 在 mapGeneratedFiles 转);`PreviewSource`(previewStore)由 fetchPreview(T3)产、PreviewContent 消费(已存在);`OpenPath`/`SaveGeneratedFile`(T1 Go)签名 `(root, relPath)` 与 T4 调用一致;root=sessionStore workingDir。

**占位符：** T4/T6 因组件/ChatPanel 复杂，给完整测试(验收)+ 实现契约 + grep 定位;Go 绑定/helper/fetchPreview 含完整代码。无 TBD。
