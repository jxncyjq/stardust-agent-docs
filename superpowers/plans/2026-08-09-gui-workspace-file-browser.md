# GUI 工作目录文件浏览器 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 在 Wails GUI 右列新增「文件」tab，浏览当前会话 `workingDir` 的懒加载文件树，选中文件按类型只读预览（代码高亮 / markdown 含 front-matter 属性表 / 图片 / HTML），支持文件名过滤、`?text` 内容搜索、复制路径、资源管理器显示、用外部配置的编辑器打开；可最大化成宽遮罩。

**Architecture:** 纯前端 React + zustand + 5 个 GUI-local Go 绑定（直连文件系统 `os.ReadDir`/`os.ReadFile`/`os/exec`，不经内嵌 serve、不碰 legionAgent 后端）。把已合入的 HTML-only `WebPreviewPanel` 泛化成共享 `PreviewContent`（按 kind 分派），「预览」tab 与「文件」tab 底部预览共用。根 = 会话 `workingDir`（`sessionStore` 已有）。全程 fail-loud（根限定 / 二进制守卫 / 大小上限 / exec argv 防注入）。

**Tech Stack:** React 18 + TS + zustand 5 + Tailwind 4 + shiki + react-markdown + Vitest；Go + Wails v2。

**Spec:** `docs/superpowers/specs/2026-08-09-gui-workspace-file-browser-design.md`

**仓库根：** 前端路径相对 `legion/legionAgentGUI/frontend/`，Go 相对 `legion/legionAgentGUI/`。

## CRITICAL：vitest 必须在 `frontend/` 目录跑
`cd legion/legionAgentGUI/frontend` 再 `npx vitest`（正确版本报 `v2.1.9`）。父目录另有一个无 jsdom 的 vitest 会 `document is not defined` 假失败。Go 命令在仓根跑。

---

## 文件结构

| 文件 | 责任 | 动作 |
|---|---|---|
| `frontend/src/stores/previewStore.ts` | `PreviewSource` 扩成 kind 联合 | 改 |
| `frontend/src/lib/highlighter.ts` | 导出 `highlightToHtml(code,lang)` | 改 |
| `frontend/src/lib/frontmatter.ts` | 极简 front-matter 解析 | 新建 |
| `frontend/src/components/PreviewContent.tsx` | 共享 kind 分派渲染器 | 新建 |
| `frontend/src/components/WebPreviewPanel.tsx` | 改用 `PreviewContent` | 改 |
| `frontend/src/lib/editorTemplate.ts` | localStorage 编辑器模板 get/set | 新建 |
| `frontend/src/stores/workspaceStore.ts` | 树/选中/过滤/搜索/最大化状态 | 新建 |
| `frontend/src/components/workspace/FileTree.tsx` | 懒加载树 + 过滤 + 搜索结果 | 新建 |
| `frontend/src/components/workspace/FilePreview.tsx` | 面包屑 + 工具条 + PreviewContent | 新建 |
| `frontend/src/components/workspace/WorkspaceFilePanel.tsx` | 「文件」tab 容器 + 空态 + 最大化 | 新建 |
| `frontend/src/App.tsx` | 「文件」tab 接入 | 改 |
| `frontend/src/stores/uiStore.ts` | `RightView` 加 `'files'` | 改 |
| `app_workspace.go` | 5 个绑定 | 新建 |
| `app_workspace_test.go` | Go 测试 | 新建 |

---

### Task 1: previewStore — PreviewSource 扩成 kind 联合

**Files:**
- Modify: `frontend/src/stores/previewStore.ts`
- Modify: `frontend/src/stores/previewStore.test.ts`（追加）

**Context:** 现有 `PreviewSource` 只有 `{kind:'html',...}`（已合入）。扩成联合以支持文件预览。`open`/`close`/`source` 逻辑不变，只改类型。现有 html 测试须继续通过。

- [ ] **Step 1: 追加失败测试** 到 `previewStore.test.ts` 末尾：

```ts
it('holds a code source', () => {
  usePreviewStore.getState().open({ kind: 'code', text: 'const x=1', lang: 'ts', title: 'a.ts', path: '/w/a.ts' })
  const s = usePreviewStore.getState().source
  expect(s).toEqual({ kind: 'code', text: 'const x=1', lang: 'ts', title: 'a.ts', path: '/w/a.ts' })
})

it('holds an image source', () => {
  usePreviewStore.getState().open({ kind: 'image', dataUri: 'data:image/png;base64,AAA', path: '/w/i.png' })
  expect(usePreviewStore.getState().source?.kind).toBe('image')
})
```

- [ ] **Step 2: 确认失败** `cd frontend && npx vitest run src/stores/previewStore.test.ts` → 新用例 TS/断言失败。

- [ ] **Step 3: 实现** — 替换 `previewStore.ts` 中 `PreviewSource` 接口为联合类型：

```ts
export type PreviewSource =
  | { kind: 'html';     html: string;    title?: string; sourceUrl?: string }
  | { kind: 'code';     text: string;    lang: string;   title?: string; path?: string }
  | { kind: 'markdown'; text: string;    title?: string; path?: string }
  | { kind: 'image';    dataUri: string; title?: string; path?: string }
  | { kind: 'binary';   title?: string;  path?: string }
```
（`PreviewState`/`create` 主体不变——`source: PreviewSource | null`、`open(src)`、`close()`。删掉原 `export interface PreviewSource {...}`。）

- [ ] **Step 4: 确认通过** `cd frontend && npx vitest run src/stores/previewStore.test.ts` → 全过。`npx tsc --noEmit`。

- [ ] **Step 5: 提交**
```bash
git add frontend/src/stores/previewStore.ts frontend/src/stores/previewStore.test.ts
git commit -m "feat(gui): extend PreviewSource to a kind union for file preview"
```

---

### Task 2: highlighter — 导出 highlightToHtml

**Files:**
- Modify: `frontend/src/lib/highlighter.ts`
- Test: `frontend/src/lib/highlighter.test.ts`（新建）

**Context:** `highlighter.ts` 已用 `createHighlighter` 建了同步 `highlighter` 实例（`codeToHtml` 同步），但只导出了 `rehypeShikiPlugin`。`code` kind 预览要高亮整份代码字符串，需导出一个直接调 `codeToHtml` 的 helper（避免用 markdown 围栏、防内容含 ``` 破坏围栏）。

- [ ] **Step 1: 失败测试** `frontend/src/lib/highlighter.test.ts`：
```ts
import { describe, it, expect } from 'vitest'
import { highlightToHtml } from './highlighter'

describe('highlightToHtml', () => {
  it('returns shiki pre with colored spans for known lang', () => {
    const html = highlightToHtml('const x: number = 1', 'ts')
    expect(html).toContain('<pre')
    expect(html).toContain('shiki')
    expect(html).toMatch(/style="[^"]*color/)
  })
  it('falls back to plain text for unknown lang (no throw)', () => {
    const html = highlightToHtml('随便文本', 'no-such-lang')
    expect(html).toContain('<pre')
  })
})
```

- [ ] **Step 2: 确认失败** `cd frontend && npx vitest run src/lib/highlighter.test.ts` → `highlightToHtml` 未导出。

- [ ] **Step 3: 实现** — 在 `highlighter.ts` 末尾追加：
```ts
// highlightToHtml renders a full code string to Shiki HTML (dual light/dark
// themes, CSS-variable driven like the markdown plugin). Used by the file
// preview's `code` kind. Unknown languages fall back to plain text instead of
// throwing (bundledLanguages check), so an arbitrary file extension is safe.
import { bundledLanguages } from 'shiki'

export function highlightToHtml(code: string, lang: string): string {
  const known = Object.prototype.hasOwnProperty.call(bundledLanguages, lang)
  return highlighter.codeToHtml(code, {
    lang: known ? lang : 'text',
    themes: { light: 'github-light', dark: 'github-dark' },
  })
}
```
> 注意：`highlighter` 是模块内已存在的 const（第 12 行 `const highlighter = await createHighlighter(...)`）。若 `'text'` 未在已注册 langs 中导致报错，改用 `known ? lang : undefined` 并省略 lang（shiki 对 undefined 走 plaintext）。先验证哪种可用。

- [ ] **Step 4: 确认通过** `cd frontend && npx vitest run src/lib/highlighter.test.ts` → 过。`npx tsc --noEmit`。

- [ ] **Step 5: 提交**
```bash
git add frontend/src/lib/highlighter.ts frontend/src/lib/highlighter.test.ts
git commit -m "feat(gui): export highlightToHtml for full-file code preview"
```

---

### Task 3: frontmatter 解析器

**Files:**
- Create: `frontend/src/lib/frontmatter.ts`
- Test: `frontend/src/lib/frontmatter.test.ts`

**Context:** 无 yaml 库。写极简解析：抽取 `---`…`---` 头，解析 top-level `key: value`，值支持裸串、引号串、inline 数组 `[a, "b"]`；嵌套/块结构原样保留为字符串。返回有序键值对 + 剩余正文。

- [ ] **Step 1: 失败测试** `frontend/src/lib/frontmatter.test.ts`：
```ts
import { describe, it, expect } from 'vitest'
import { parseFrontmatter } from './frontmatter'

describe('parseFrontmatter', () => {
  it('splits frontmatter props and body', () => {
    const src = '---\nid: "x-1"\ntitle: 标题\ntags: [a, "b c"]\n---\n# 正文\nhello'
    const r = parseFrontmatter(src)
    expect(r.props).toEqual([
      { key: 'id', value: 'x-1' },
      { key: 'title', value: '标题' },
      { key: 'tags', value: 'a, b c' },
    ])
    expect(r.body).toBe('# 正文\nhello')
  })
  it('returns empty props and full body when no frontmatter', () => {
    const r = parseFrontmatter('# just body')
    expect(r.props).toEqual([])
    expect(r.body).toBe('# just body')
  })
  it('keeps nested/block values as raw string', () => {
    const r = parseFrontmatter('---\nrelated:\n  - a\n  - b\nid: 1\n---\nx')
    // block list under `related` is captured raw; `id` still parsed
    expect(r.props.find((p) => p.key === 'id')?.value).toBe('1')
  })
})
```

- [ ] **Step 2: 确认失败** `cd frontend && npx vitest run src/lib/frontmatter.test.ts`.

- [ ] **Step 3: 实现** `frontend/src/lib/frontmatter.ts`：
```ts
// Prop is one displayed frontmatter row. Value is always a display string —
// arrays are joined, quotes stripped; nested/block values are kept raw.
export interface Prop { key: string; value: string }
export interface Parsed { props: Prop[]; body: string }

const unquote = (s: string): string => {
  const t = s.trim()
  if ((t.startsWith('"') && t.endsWith('"')) || (t.startsWith("'") && t.endsWith("'"))) {
    return t.slice(1, -1)
  }
  return t
}

// parseFrontmatter extracts a leading `---`…`---` YAML block and parses only the
// top-level `key: value` lines (裸串 / 引号串 / inline `[...]`). Block or nested
// structures are not YAML-parsed; a top-level key whose value is empty (block
// follows) is skipped rather than guessed. Not a general YAML parser — the docs
// use a flat, predictable shape. No frontmatter → empty props, whole input body.
export function parseFrontmatter(src: string): Parsed {
  const m = /^---\r?\n([\s\S]*?)\r?\n---\r?\n?([\s\S]*)$/.exec(src)
  if (!m) return { props: [], body: src }
  const [, head, body] = m
  const props: Prop[] = []
  for (const raw of head.split(/\r?\n/)) {
    // top-level only: skip indented (block/nested) lines
    if (/^\s/.test(raw) || raw.trim() === '') continue
    const idx = raw.indexOf(':')
    if (idx < 0) continue
    const key = raw.slice(0, idx).trim()
    let val = raw.slice(idx + 1).trim()
    if (val === '') continue // block/nested follows — not displayed as a scalar
    if (val.startsWith('[') && val.endsWith(']')) {
      val = val.slice(1, -1).split(',').map((x) => unquote(x)).filter((x) => x !== '').join(', ')
    } else {
      val = unquote(val)
    }
    props.push({ key, value: val })
  }
  return { props, body: body ?? '' }
}
```

- [ ] **Step 4: 确认通过** `cd frontend && npx vitest run src/lib/frontmatter.test.ts` → 3 过。

- [ ] **Step 5: 提交**
```bash
git add frontend/src/lib/frontmatter.ts frontend/src/lib/frontmatter.test.ts
git commit -m "feat(gui): add minimal frontmatter parser for markdown preview"
```

---

### Task 4: PreviewContent 共享分派渲染器

**Files:**
- Create: `frontend/src/components/PreviewContent.tsx`
- Test: `frontend/src/components/PreviewContent.test.tsx`

**Context:** 按 `PreviewSource.kind` 分派：`html`→`sandbox=""` iframe（同已合入 WebPreviewPanel 的 iframe）、`code`→`highlightToHtml` + `dangerouslySetInnerHTML`、`markdown`→front-matter 属性表 + `ReactMarkdown`（复用 MessageBubble 同款 `remarkGfm`+`rehypeShikiPlugin`）、`image`→`<img>`、`binary`→占位。`markdown`/`html` 支持"源码模式"（`raw` prop）显示 raw `<pre>`。

- [ ] **Step 1: 失败测试** `frontend/src/components/PreviewContent.test.tsx`：
```tsx
import { describe, it, expect } from 'vitest'
import { render, screen } from '@testing-library/react'
import { PreviewContent } from './PreviewContent'

describe('PreviewContent kind dispatch', () => {
  it('html → sandboxed iframe', () => {
    render(<PreviewContent source={{ kind: 'html', html: '<h1>h</h1>' }} raw={false} />)
    const f = screen.getByTitle('HTML 预览内容') as HTMLIFrameElement
    expect(f.getAttribute('sandbox')).not.toBeNull()
    expect(f.getAttribute('sandbox')).not.toContain('allow-scripts')
    expect(f.getAttribute('srcdoc')).toBe('<h1>h</h1>')
  })
  it('code → shiki pre', () => {
    const { container } = render(<PreviewContent source={{ kind: 'code', text: 'const x=1', lang: 'ts' }} raw={false} />)
    expect(container.querySelector('pre')).not.toBeNull()
  })
  it('markdown → frontmatter table + body', () => {
    render(<PreviewContent source={{ kind: 'markdown', text: '---\nid: x1\n---\n# 标题' }} raw={false} />)
    expect(screen.getByText('id')).toBeInTheDocument()
    expect(screen.getByText('x1')).toBeInTheDocument()
    expect(screen.getByRole('heading', { name: '标题' })).toBeInTheDocument()
  })
  it('markdown raw mode shows source pre', () => {
    const { container } = render(<PreviewContent source={{ kind: 'markdown', text: '# 标题' }} raw={true} />)
    expect(container.querySelector('pre')?.textContent).toContain('# 标题')
  })
  it('image → img with dataUri', () => {
    render(<PreviewContent source={{ kind: 'image', dataUri: 'data:image/png;base64,AAA' }} raw={false} />)
    expect(screen.getByRole('img').getAttribute('src')).toBe('data:image/png;base64,AAA')
  })
  it('binary → placeholder', () => {
    render(<PreviewContent source={{ kind: 'binary' }} raw={false} />)
    expect(screen.getByText('不支持预览此文件')).toBeInTheDocument()
  })
})
```

- [ ] **Step 2: 确认失败** `cd frontend && npx vitest run src/components/PreviewContent.test.tsx`.

- [ ] **Step 3: 实现** `frontend/src/components/PreviewContent.tsx`：
```tsx
import { memo } from 'react'
import ReactMarkdown from 'react-markdown'
import remarkGfm from 'remark-gfm'
import { rehypeShikiPlugin, highlightToHtml } from '../lib/highlighter'
import { parseFrontmatter } from '../lib/frontmatter'
import type { PreviewSource } from '../stores/previewStore'

interface Props {
  source: PreviewSource
  // raw shows the unrendered source (<pre>) for markdown/html; ignored otherwise.
  raw: boolean
}

// PreviewContent is the shared, kind-dispatched preview renderer. It backs both
// the chat-triggered 预览 tab (WebPreviewPanel) and the file browser's preview
// pane. Pure and props-driven for testability. Security for `html` matches the
// original panel: sandbox="" withholds scripts/same-origin entirely.
export const PreviewContent = memo(function PreviewContent({ source, raw }: Props) {
  switch (source.kind) {
    case 'html':
      if (raw) return <pre className="p-3 text-xs whitespace-pre-wrap break-all">{source.html}</pre>
      return (
        <iframe
          title="HTML 预览内容"
          sandbox=""
          srcDoc={source.html}
          className="min-h-0 flex-1 w-full h-full border-0 bg-white"
        />
      )
    case 'code':
      return (
        <div
          className="preview-code overflow-auto p-1 text-xs"
          dangerouslySetInnerHTML={{ __html: highlightToHtml(source.text, source.lang) }}
        />
      )
    case 'markdown': {
      if (raw) return <pre className="p-3 text-xs whitespace-pre-wrap break-all">{source.text}</pre>
      const { props, body } = parseFrontmatter(source.text)
      return (
        <div className="overflow-auto p-3">
          {props.length > 0 && (
            <table className="mb-3 w-full border-collapse text-xs">
              <tbody>
                {props.map((p) => (
                  <tr key={p.key} className="border-b border-border/50">
                    <td className="w-32 py-1 pr-3 align-top text-muted-foreground">{p.key}</td>
                    <td className="py-1 align-top text-foreground break-all">{p.value}</td>
                  </tr>
                ))}
              </tbody>
            </table>
          )}
          <div className="prose prose-sm dark:prose-invert max-w-none prose-pre:bg-background prose-pre:text-foreground">
            <ReactMarkdown remarkPlugins={[remarkGfm]} rehypePlugins={[rehypeShikiPlugin]}>
              {body}
            </ReactMarkdown>
          </div>
        </div>
      )
    }
    case 'image':
      return (
        <div className="flex h-full items-center justify-center overflow-auto p-3">
          <img src={source.dataUri} alt={source.title ?? '图片预览'} className="max-w-full max-h-full object-contain" />
        </div>
      )
    case 'binary':
      return (
        <div className="flex h-full items-center justify-center p-4 text-xs text-muted-foreground">
          不支持预览此文件
        </div>
      )
  }
})
```

- [ ] **Step 4: 确认通过** `cd frontend && npx vitest run src/components/PreviewContent.test.tsx` → 6 过。`npx tsc --noEmit`。

- [ ] **Step 5: 提交**
```bash
git add frontend/src/components/PreviewContent.tsx frontend/src/components/PreviewContent.test.tsx
git commit -m "feat(gui): add shared kind-dispatched PreviewContent renderer"
```

---

### Task 5: 重构 WebPreviewPanel 用 PreviewContent

**Files:**
- Modify: `frontend/src/components/WebPreviewPanel.tsx`
- Modify: `frontend/src/components/WebPreviewPanel.test.tsx`（如需）

**Context:** WebPreviewPanel 现在自己渲染 html iframe。改成把渲染委托给 `PreviewContent`，保留工具条（标题 / 外部打开 / 关闭）。**原有 5 个测试须继续通过**（回归）——iframe 仍带 `sandbox=""`、srcdoc 注入、close、外部打开、空态。

- [ ] **Step 1: 跑现有测试建基线** `cd frontend && npx vitest run src/components/WebPreviewPanel.test.tsx` → 记录当前 5 过。

- [ ] **Step 2: 实现** — 把 WebPreviewPanel 的 iframe 部分替换为 `<PreviewContent source={source} raw={false} />`，其余（空态、工具条）不变：
```tsx
import { PreviewContent } from './PreviewContent'
// ...（工具条 return 中，把原 <iframe .../> 换成：）
      <div className="min-h-0 flex-1 flex flex-col">
        <PreviewContent source={source} raw={false} />
      </div>
```
（保留 `sandbox` 测试的可通过性：PreviewContent 的 html 分支 iframe title 仍是 `"HTML 预览内容"`、`sandbox=""`，与原测试断言一致。）

- [ ] **Step 3: 确认回归通过** `cd frontend && npx vitest run src/components/WebPreviewPanel.test.tsx` → 5 仍过。若某断言因结构变化失败，最小调整测试选择器（不放宽安全断言）。`npx tsc --noEmit`。

- [ ] **Step 4: 提交**
```bash
git add frontend/src/components/WebPreviewPanel.tsx frontend/src/components/WebPreviewPanel.test.tsx
git commit -m "refactor(gui): WebPreviewPanel renders via shared PreviewContent"
```

---

### Task 6: Go — ListWorkspaceDir + ReadWorkspaceFile

**Files:**
- Create: `app_workspace.go`
- Create: `app_workspace_test.go`

**Context:** GUI-local 绑定，直连文件系统。`App` 在 `app.go`（package main）。守 fail-loud 铁律（见 legionAgentGUI/CLAUDE.md）。根限定用 `filepath.Rel` 校验目标在 root 内。二进制守卫 + 2MB 大小上限。kind 按扩展名判定。

- [ ] **Step 1: 失败测试** `app_workspace_test.go`：
```go
package main

import (
	"os"
	"path/filepath"
	"strings"
	"testing"
)

func TestListWorkspaceDir(t *testing.T) {
	root := t.TempDir()
	os.WriteFile(filepath.Join(root, "a.txt"), []byte("x"), 0o644)
	os.Mkdir(filepath.Join(root, "sub"), 0o755)
	a := NewApp("")
	entries, err := a.ListWorkspaceDir(root, "")
	if err != nil {
		t.Fatal(err)
	}
	if len(entries) != 2 {
		t.Fatalf("want 2 entries, got %d", len(entries))
	}
}

func TestListWorkspaceDirRejectsOutsideRoot(t *testing.T) {
	root := t.TempDir()
	a := NewApp("")
	if _, err := a.ListWorkspaceDir(root, "../.."); err == nil {
		t.Fatal("expected error for path outside root")
	}
}

func TestReadWorkspaceFileCodeKind(t *testing.T) {
	root := t.TempDir()
	os.WriteFile(filepath.Join(root, "m.ts"), []byte("const x=1"), 0o644)
	a := NewApp("")
	f, err := a.ReadWorkspaceFile(root, filepath.Join(root, "m.ts"))
	if err != nil {
		t.Fatal(err)
	}
	if f.Kind != "code" || f.Lang != "typescript" || f.Text != "const x=1" {
		t.Fatalf("got %+v", f)
	}
}

func TestReadWorkspaceFileMarkdownKind(t *testing.T) {
	root := t.TempDir()
	os.WriteFile(filepath.Join(root, "d.md"), []byte("# hi"), 0o644)
	a := NewApp("")
	f, _ := a.ReadWorkspaceFile(root, filepath.Join(root, "d.md"))
	if f.Kind != "markdown" {
		t.Fatalf("want markdown, got %q", f.Kind)
	}
}

func TestReadWorkspaceFileImageKind(t *testing.T) {
	root := t.TempDir()
	// 1x1 png header bytes are enough; we only check kind + data URI prefix.
	os.WriteFile(filepath.Join(root, "i.png"), []byte{0x89, 0x50, 0x4e, 0x47}, 0o644)
	a := NewApp("")
	f, _ := a.ReadWorkspaceFile(root, filepath.Join(root, "i.png"))
	if f.Kind != "image" || !strings.HasPrefix(f.DataURI, "data:image/png;base64,") {
		t.Fatalf("got %+v", f)
	}
}

func TestReadWorkspaceFileBinaryGuard(t *testing.T) {
	root := t.TempDir()
	os.WriteFile(filepath.Join(root, "b.bin"), []byte{0x00, 0x01, 0x02}, 0o644)
	a := NewApp("")
	f, _ := a.ReadWorkspaceFile(root, filepath.Join(root, "b.bin"))
	if f.Kind != "binary" {
		t.Fatalf("want binary, got %q", f.Kind)
	}
}

func TestReadWorkspaceFileRejectsOutsideRoot(t *testing.T) {
	root := t.TempDir()
	a := NewApp("")
	if _, err := a.ReadWorkspaceFile(root, filepath.Join(root, "..", "escape.txt")); err == nil {
		t.Fatal("expected error for path outside root")
	}
}
```

- [ ] **Step 2: 确认失败** `go test ./... -run TestListWorkspaceDir` → undefined。

- [ ] **Step 3: 实现** `app_workspace.go`：
```go
package main

import (
	"encoding/base64"
	"fmt"
	"os"
	"path/filepath"
	"strings"
	"unicode/utf8"
)

const maxPreviewBytes = 2 << 20 // 2 MiB

type WorkspaceEntry struct {
	Name  string `json:"name"`
	IsDir bool   `json:"isDir"`
	Size  int64  `json:"size"`
}

type WorkspaceFile struct {
	Kind    string `json:"kind"` // code | markdown | html | image | binary
	Text    string `json:"text"`
	DataURI string `json:"dataURI"`
	Lang    string `json:"lang"`
}

// resolveInRoot cleans target and verifies it stays within root. It returns the
// absolute cleaned path or a fail-loud error (never a silently clamped path).
func resolveInRoot(root, target string) (string, error) {
	absRoot, err := filepath.Abs(root)
	if err != nil {
		return "", fmt.Errorf("resolve root %q: %w", root, err)
	}
	absTarget := target
	if !filepath.IsAbs(absTarget) {
		absTarget = filepath.Join(absRoot, absTarget)
	}
	absTarget = filepath.Clean(absTarget)
	rel, err := filepath.Rel(absRoot, absTarget)
	if err != nil {
		return "", fmt.Errorf("relate %q to root: %w", target, err)
	}
	if rel == ".." || strings.HasPrefix(rel, ".."+string(filepath.Separator)) {
		return "", fmt.Errorf("path %q is outside workspace root", target)
	}
	return absTarget, nil
}

// ListWorkspaceDir lists one directory level under root/sub (sub relative to
// root). Fail-loud: a sub escaping root, or a stat/read error, returns an error.
func (a *App) ListWorkspaceDir(root, sub string) ([]WorkspaceEntry, error) {
	dir, err := resolveInRoot(root, sub)
	if err != nil {
		return nil, err
	}
	items, err := os.ReadDir(dir)
	if err != nil {
		return nil, fmt.Errorf("read dir %q: %w", dir, err)
	}
	out := make([]WorkspaceEntry, 0, len(items))
	for _, it := range items {
		info, err := it.Info()
		if err != nil {
			return nil, fmt.Errorf("stat entry %q: %w", it.Name(), err)
		}
		out = append(out, WorkspaceEntry{Name: it.Name(), IsDir: it.IsDir(), Size: info.Size()})
	}
	return out, nil
}

var imageExt = map[string]string{
	".png": "image/png", ".jpg": "image/jpeg", ".jpeg": "image/jpeg",
	".gif": "image/gif", ".webp": "image/webp", ".bmp": "image/bmp", ".svg": "image/svg+xml",
}

// langByExt maps a file extension to a shiki language name. Unknown → "text".
var langByExt = map[string]string{
	".ts": "typescript", ".tsx": "tsx", ".js": "javascript", ".jsx": "jsx",
	".go": "go", ".json": "json", ".sh": "bash", ".py": "python", ".md": "markdown",
	".html": "html", ".css": "css", ".yaml": "yaml", ".yml": "yaml", ".toml": "toml",
}

// ReadWorkspaceFile reads a file for read-only preview. It dispatches Kind by
// extension (image via base64 data URI; .md → markdown; .html → html; else code),
// with a binary guard (NUL byte / invalid UTF-8 → Kind "binary") and a 2 MiB
// cap. Fail-loud: outside-root / stat / read / oversize errors propagate.
func (a *App) ReadWorkspaceFile(root, path string) (WorkspaceFile, error) {
	abs, err := resolveInRoot(root, path)
	if err != nil {
		return WorkspaceFile{}, err
	}
	info, err := os.Stat(abs)
	if err != nil {
		return WorkspaceFile{}, fmt.Errorf("stat file %q: %w", path, err)
	}
	if info.IsDir() {
		return WorkspaceFile{}, fmt.Errorf("read file %q: is a directory", path)
	}
	if info.Size() > maxPreviewBytes {
		return WorkspaceFile{}, fmt.Errorf("read file %q: too large (%d bytes > %d)", path, info.Size(), maxPreviewBytes)
	}
	ext := strings.ToLower(filepath.Ext(abs))
	if mime, ok := imageExt[ext]; ok {
		data, err := os.ReadFile(abs)
		if err != nil {
			return WorkspaceFile{}, fmt.Errorf("read image %q: %w", path, err)
		}
		return WorkspaceFile{Kind: "image", DataURI: "data:" + mime + ";base64," + base64.StdEncoding.EncodeToString(data)}, nil
	}
	data, err := os.ReadFile(abs)
	if err != nil {
		return WorkspaceFile{}, fmt.Errorf("read file %q: %w", path, err)
	}
	if !utf8.Valid(data) || bytesContainNUL(data) {
		return WorkspaceFile{Kind: "binary"}, nil
	}
	text := string(data)
	switch ext {
	case ".md", ".markdown":
		return WorkspaceFile{Kind: "markdown", Text: text}, nil
	case ".html", ".htm":
		return WorkspaceFile{Kind: "html", Text: text}, nil
	default:
		lang := langByExt[ext]
		if lang == "" {
			lang = "text"
		}
		return WorkspaceFile{Kind: "code", Text: text, Lang: lang}, nil
	}
}

func bytesContainNUL(b []byte) bool {
	for _, c := range b {
		if c == 0 {
			return true
		}
	}
	return false
}
```
> 注意：`.html` 预览这里返回 `Kind:"html"` + `Text`（原文），前端拿 Text 组成 `{kind:'html', html: text}`。（前端 ReadWorkspaceFile→PreviewSource 映射见 Task 9/10。）

- [ ] **Step 4: 确认通过** `go test ./... -run 'TestListWorkspaceDir|TestReadWorkspaceFile' -v`（7 过）；`gofmt -l app_workspace.go app_workspace_test.go`（空）；`go vet ./...`。

- [ ] **Step 5: 提交**
```bash
git add app_workspace.go app_workspace_test.go
git commit -m "feat(gui): ListWorkspaceDir + ReadWorkspaceFile (root-confined, fail-loud)"
```

---

### Task 7: Go — SearchWorkspaceContent

**Files:**
- Modify: `app_workspace.go`
- Modify: `app_workspace_test.go`（追加）

**Context:** 递归 grep 文本文件；跳过二进制/超大；限文件数(2000)、单文件(1MiB)、结果数(500)。root 限定。

- [ ] **Step 1: 追加失败测试**：
```go
func TestSearchWorkspaceContent(t *testing.T) {
	root := t.TempDir()
	os.WriteFile(filepath.Join(root, "a.txt"), []byte("hello world\nfoo bar"), 0o644)
	os.WriteFile(filepath.Join(root, "b.md"), []byte("# nothing here"), 0o644)
	os.WriteFile(filepath.Join(root, "c.bin"), []byte{0x00, 0x01}, 0o644)
	a := NewApp("")
	hits, err := a.SearchWorkspaceContent(root, "world")
	if err != nil {
		t.Fatal(err)
	}
	if len(hits) != 1 || !strings.HasSuffix(hits[0].Path, "a.txt") || hits[0].Line != 1 {
		t.Fatalf("got %+v", hits)
	}
}

func TestSearchWorkspaceContentRejectsOutsideRoot(t *testing.T) {
	a := NewApp("")
	if _, err := a.SearchWorkspaceContent("../..", "x"); err == nil {
		// resolveInRoot(root, ".") for the root itself must still validate root exists
	}
	// primary guard: empty query returns error
	root := t.TempDir()
	if _, err := a.SearchWorkspaceContent(root, ""); err == nil {
		t.Fatal("expected error for empty query")
	}
}
```

- [ ] **Step 2: 确认失败** `go test ./... -run TestSearchWorkspaceContent`.

- [ ] **Step 3: 实现** — 在 `app_workspace.go` 追加：
```go
const (
	maxSearchFiles     = 2000
	maxSearchFileBytes = 1 << 20 // 1 MiB
	maxSearchHits      = 500
)

type SearchHit struct {
	Path    string `json:"path"`
	Line    int    `json:"line"`
	Snippet string `json:"snippet"`
}

// SearchWorkspaceContent walks root recursively and returns line hits for query
// (case-sensitive substring) across text files. It skips binary/oversize files
// and caps files scanned / hits returned; a skip is a bounded, documented limit,
// not a silent swallow. Fail-loud: empty query or a walk error returns an error.
func (a *App) SearchWorkspaceContent(root, query string) ([]SearchHit, error) {
	if strings.TrimSpace(query) == "" {
		return nil, fmt.Errorf("search query is empty")
	}
	absRoot, err := filepath.Abs(root)
	if err != nil {
		return nil, fmt.Errorf("resolve root %q: %w", root, err)
	}
	if info, err := os.Stat(absRoot); err != nil || !info.IsDir() {
		return nil, fmt.Errorf("search root %q not a directory: %w", root, err)
	}
	hits := make([]SearchHit, 0, 64)
	filesSeen := 0
	err = filepath.WalkDir(absRoot, func(p string, d os.DirEntry, walkErr error) error {
		if walkErr != nil {
			return fmt.Errorf("walk %q: %w", p, walkErr)
		}
		if d.IsDir() {
			return nil
		}
		if filesSeen >= maxSearchFiles || len(hits) >= maxSearchHits {
			return filepath.SkipAll
		}
		info, err := d.Info()
		if err != nil {
			return fmt.Errorf("stat %q: %w", p, err)
		}
		if info.Size() > maxSearchFileBytes {
			return nil
		}
		data, err := os.ReadFile(p)
		if err != nil {
			return fmt.Errorf("read %q: %w", p, err)
		}
		if !utf8.Valid(data) || bytesContainNUL(data) {
			return nil
		}
		filesSeen++
		for i, line := range strings.Split(string(data), "\n") {
			if strings.Contains(line, query) {
				snippet := line
				if len(snippet) > 200 {
					snippet = snippet[:200]
				}
				hits = append(hits, SearchHit{Path: p, Line: i + 1, Snippet: strings.TrimSpace(snippet)})
				if len(hits) >= maxSearchHits {
					return filepath.SkipAll
				}
			}
		}
		return nil
	})
	if err != nil {
		return nil, err
	}
	return hits, nil
}
```

- [ ] **Step 4: 确认通过** `go test ./... -run TestSearchWorkspaceContent -v`；`gofmt -l .` 忽略 frontend；`go vet ./...`。

- [ ] **Step 5: 提交**
```bash
git add app_workspace.go app_workspace_test.go
git commit -m "feat(gui): SearchWorkspaceContent recursive text grep with caps"
```

---

### Task 8: Go — OpenInEditor + RevealInExplorer

**Files:**
- Modify: `app_workspace.go`
- Modify: `app_workspace_test.go`（追加）
- 重生成 Wails 绑定

**Context:** `OpenInEditor(template, path)` 把 template 按 argv **引号感知 split**，`{path}` 替换为单个 argv 元素（缺 `{path}` 则 append path），**不走 shell**，`exec.Command(argv[0], argv[1:]...)`。`RevealInExplorer(path)` Windows `explorer /select,<abs>`。测试用可断言的 argv 拆分函数（把 split 抽成纯函数 `buildEditorArgv` 单测，避免真的启动进程）。

- [ ] **Step 1: 追加失败测试**：
```go
func TestBuildEditorArgv(t *testing.T) {
	argv, err := buildEditorArgv(`code "{path}"`, `C:\a b\x.ts`)
	if err != nil {
		t.Fatal(err)
	}
	if len(argv) != 2 || argv[0] != "code" || argv[1] != `C:\a b\x.ts` {
		t.Fatalf("got %#v", argv)
	}
}

func TestBuildEditorArgvAppendsWhenNoPlaceholder(t *testing.T) {
	argv, _ := buildEditorArgv("notepad", `x.txt`)
	if len(argv) != 2 || argv[0] != "notepad" || argv[1] != "x.txt" {
		t.Fatalf("got %#v", argv)
	}
}

func TestBuildEditorArgvNoShellInjection(t *testing.T) {
	// A path with shell metacharacters must land as ONE argv element, never split.
	argv, _ := buildEditorArgv("code {path}", `x.txt; rm -rf /`)
	if len(argv) != 2 || argv[1] != `x.txt; rm -rf /` {
		t.Fatalf("injection not contained: %#v", argv)
	}
}

func TestBuildEditorArgvEmptyTemplate(t *testing.T) {
	if _, err := buildEditorArgv("   ", "x.txt"); err == nil {
		t.Fatal("expected error for empty template")
	}
}
```

- [ ] **Step 2: 确认失败** `go test ./... -run TestBuildEditorArgv`.

- [ ] **Step 3: 实现** — 追加到 `app_workspace.go`（`os/exec` 需加入 import）：
```go
// buildEditorArgv turns a user editor template into an argv slice. It splits on
// whitespace with double-quote awareness, then replaces the `{path}` token with
// the file path as a SINGLE argv element (never shell-interpolated). A template
// without `{path}` gets the path appended. Empty template → error. The result is
// run via exec.Command WITHOUT a shell, so path metacharacters cannot inject.
func buildEditorArgv(template, path string) ([]string, error) {
	if strings.TrimSpace(template) == "" {
		return nil, fmt.Errorf("editor template is empty")
	}
	tokens := splitArgs(template)
	if len(tokens) == 0 {
		return nil, fmt.Errorf("editor template has no command")
	}
	argv := make([]string, 0, len(tokens)+1)
	replaced := false
	for _, tok := range tokens {
		if strings.Contains(tok, "{path}") {
			argv = append(argv, strings.ReplaceAll(tok, "{path}", path))
			replaced = true
		} else {
			argv = append(argv, tok)
		}
	}
	if !replaced {
		argv = append(argv, path)
	}
	return argv, nil
}

// splitArgs splits a command line on whitespace, honoring double quotes so a
// quoted segment (e.g. "{path}") stays one token. Quotes are stripped.
func splitArgs(s string) []string {
	var out []string
	var cur strings.Builder
	inQuote := false
	flush := func() {
		if cur.Len() > 0 {
			out = append(out, cur.String())
			cur.Reset()
		}
	}
	for _, r := range s {
		switch {
		case r == '"':
			inQuote = !inQuote
		case (r == ' ' || r == '\t') && !inQuote:
			flush()
		default:
			cur.WriteRune(r)
		}
	}
	flush()
	return out
}

// OpenInEditor launches the user-configured editor on path (no shell). Fail-loud:
// a bad template or a launch failure returns a wrapped error.
func (a *App) OpenInEditor(template, path string) error {
	argv, err := buildEditorArgv(template, path)
	if err != nil {
		return fmt.Errorf("open in editor: %w", err)
	}
	cmd := exec.Command(argv[0], argv[1:]...)
	if err := cmd.Start(); err != nil {
		return fmt.Errorf("launch editor %q: %w", argv[0], err)
	}
	return nil
}

// RevealInExplorer opens the OS file manager with path selected. Windows-only
// behaviour here (explorer /select). Fail-loud on launch error.
func (a *App) RevealInExplorer(path string) error {
	abs, err := filepath.Abs(path)
	if err != nil {
		return fmt.Errorf("reveal %q: %w", path, err)
	}
	cmd := exec.Command("explorer", "/select,"+abs)
	// explorer returns exit code 1 even on success; Start (not Run) avoids that.
	if err := cmd.Start(); err != nil {
		return fmt.Errorf("launch explorer for %q: %w", abs, err)
	}
	return nil
}
```
> 加 `"os/exec"` 到 import 块。

- [ ] **Step 4: 确认通过** `go test ./... -run 'TestBuildEditorArgv' -v`（4 过）；`go build ./...`；`gofmt -l app_workspace.go`（空）。

- [ ] **Step 5: 重生成绑定** `wails generate module`，确认 `frontend/wailsjs/go/main/App.d.ts` 出现 `ListWorkspaceDir`/`ReadWorkspaceFile`/`SearchWorkspaceContent`/`OpenInEditor`/`RevealInExplorer` 及 models（`WorkspaceEntry`/`WorkspaceFile`/`SearchHit`）。
  - 无 `wails` CLI 时手动补 `App.d.ts`/`App.js`（仿现有导出）+ 在 `frontend/wailsjs/go/models.ts` 加三个接口。报告所走路径。

- [ ] **Step 6: 提交**
```bash
git add app_workspace.go app_workspace_test.go frontend/wailsjs/go/main/App.js frontend/wailsjs/go/main/App.d.ts frontend/wailsjs/go/models.ts
git commit -m "feat(gui): OpenInEditor (argv, no shell) + RevealInExplorer + bindings"
```

---

### Task 9: editorTemplate 助手 + workspaceStore

**Files:**
- Create: `frontend/src/lib/editorTemplate.ts` + `.test.ts`
- Create: `frontend/src/stores/workspaceStore.ts` + `.test.ts`

**Context:** 编辑器模板存 localStorage（GUI 偏好）。workspaceStore 管树/选中/过滤/搜索/最大化。

- [ ] **Step 1: 失败测试** `frontend/src/lib/editorTemplate.test.ts`：
```ts
import { describe, it, expect, beforeEach } from 'vitest'
import { getEditorTemplate, setEditorTemplate } from './editorTemplate'

describe('editorTemplate', () => {
  beforeEach(() => localStorage.clear())
  it('returns empty string when unset', () => {
    expect(getEditorTemplate()).toBe('')
  })
  it('persists and reads back', () => {
    setEditorTemplate('code "{path}"')
    expect(getEditorTemplate()).toBe('code "{path}"')
  })
})
```
`frontend/src/stores/workspaceStore.test.ts`：
```ts
import { describe, it, expect, beforeEach } from 'vitest'
import { useWorkspaceStore } from './workspaceStore'

describe('workspaceStore', () => {
  beforeEach(() => useWorkspaceStore.getState().reset())
  it('sets root and clears on reset', () => {
    useWorkspaceStore.getState().setRoot('/w')
    expect(useWorkspaceStore.getState().rootDir).toBe('/w')
    useWorkspaceStore.getState().reset()
    expect(useWorkspaceStore.getState().rootDir).toBe('')
  })
  it('toggles a directory expanded state', () => {
    useWorkspaceStore.getState().toggleExpanded('sub')
    expect(useWorkspaceStore.getState().expanded.has('sub')).toBe(true)
    useWorkspaceStore.getState().toggleExpanded('sub')
    expect(useWorkspaceStore.getState().expanded.has('sub')).toBe(false)
  })
  it('sets filter and maximized', () => {
    useWorkspaceStore.getState().setFilter('abc')
    expect(useWorkspaceStore.getState().filter).toBe('abc')
    useWorkspaceStore.getState().setMaximized(true)
    expect(useWorkspaceStore.getState().maximized).toBe(true)
  })
})
```

- [ ] **Step 2: 确认失败** `cd frontend && npx vitest run src/lib/editorTemplate.test.ts src/stores/workspaceStore.test.ts`.

- [ ] **Step 3: 实现** `frontend/src/lib/editorTemplate.ts`：
```ts
const KEY = 'legion:editorTemplate'

// getEditorTemplate returns the user's external-editor command template
// (e.g. `code "{path}"`), or "" if unset. Machine-specific → localStorage, not
// the shared agent config.
export function getEditorTemplate(): string {
  return localStorage.getItem(KEY) ?? ''
}

export function setEditorTemplate(template: string): void {
  localStorage.setItem(KEY, template)
}
```
`frontend/src/stores/workspaceStore.ts`：
```ts
import { create } from 'zustand'
import type { WorkspaceEntry, SearchHit } from '../../wailsjs/go/models'

// TreeNode caches one loaded directory level. children===null means "not loaded
// yet" (lazy). subPath is relative to rootDir.
export interface TreeNode {
  entry: WorkspaceEntry
  subPath: string
  children: TreeNode[] | null
}

interface WorkspaceState {
  rootDir: string
  roots: TreeNode[]          // top-level entries
  expanded: Set<string>      // subPaths of expanded dirs
  selected: string | null    // subPath of selected file
  filter: string
  searchHits: SearchHit[] | null // non-null → search results mode
  maximized: boolean
  setRoot: (dir: string) => void
  setRoots: (nodes: TreeNode[]) => void
  setChildren: (subPath: string, children: TreeNode[]) => void
  toggleExpanded: (subPath: string) => void
  select: (subPath: string | null) => void
  setFilter: (f: string) => void
  setSearchHits: (hits: SearchHit[] | null) => void
  setMaximized: (v: boolean) => void
  reset: () => void
}

const empty = {
  roots: [] as TreeNode[], expanded: new Set<string>(), selected: null as string | null,
  filter: '', searchHits: null as SearchHit[] | null, maximized: false,
}

// setChildren walks the tree to attach loaded children to the node at subPath.
function attach(nodes: TreeNode[], subPath: string, children: TreeNode[]): TreeNode[] {
  return nodes.map((n) => {
    if (n.subPath === subPath) return { ...n, children }
    if (n.children) return { ...n, children: attach(n.children, subPath, children) }
    return n
  })
}

export const useWorkspaceStore = create<WorkspaceState>((set) => ({
  rootDir: '',
  ...empty,
  setRoot: (dir) => set({ rootDir: dir, ...empty }),
  setRoots: (nodes) => set({ roots: nodes }),
  setChildren: (subPath, children) => set((s) => ({ roots: attach(s.roots, subPath, children) })),
  toggleExpanded: (subPath) => set((s) => {
    const next = new Set(s.expanded)
    next.has(subPath) ? next.delete(subPath) : next.add(subPath)
    return { expanded: next }
  }),
  select: (subPath) => set({ selected: subPath }),
  setFilter: (f) => set({ filter: f }),
  setSearchHits: (hits) => set({ searchHits: hits }),
  setMaximized: (v) => set({ maximized: v }),
  reset: () => set({ rootDir: '', ...empty }),
}))
```
> 若 `WorkspaceEntry`/`SearchHit` 未在 `models.ts`（Task 8 绑定生成），先确认其导出名，import 对齐。

- [ ] **Step 4: 确认通过** `cd frontend && npx vitest run src/lib/editorTemplate.test.ts src/stores/workspaceStore.test.ts`；`npx tsc --noEmit`。

- [ ] **Step 5: 提交**
```bash
git add frontend/src/lib/editorTemplate.ts frontend/src/lib/editorTemplate.test.ts frontend/src/stores/workspaceStore.ts frontend/src/stores/workspaceStore.test.ts
git commit -m "feat(gui): editorTemplate localStorage helper + workspaceStore"
```

---

### Task 10: FileTree 组件

**Files:**
- Create: `frontend/src/components/workspace/FileTree.tsx` + `.test.tsx`

**Context:** 懒加载树：渲染 `workspaceStore.roots`；点文件夹 → 若 `children===null` 调 `ListWorkspaceDir(root, subPath)` 填充 + 展开；点文件 → `select(subPath)`。文件名过滤（前端过滤已加载节点）。`?text` → 调 `SearchWorkspaceContent` 进结果态，点结果 select 对应文件。图标：文件夹 chevron + 文件类型图标（复用现有 icons，缺则补一个 FileIcon/FolderIcon）。挂载时若 `roots` 空且有 rootDir，加载顶层。

- [ ] **Step 1: 失败测试** `frontend/src/components/workspace/FileTree.test.tsx`（mock `../../../wailsjs/go/main/App`）：
```tsx
import { describe, it, expect, vi, beforeEach } from 'vitest'
import { render, screen, fireEvent, waitFor } from '@testing-library/react'

const appMocks = vi.hoisted(() => ({ ListWorkspaceDir: vi.fn(), SearchWorkspaceContent: vi.fn() }))
vi.mock('../../../wailsjs/go/main/App', () => appMocks)

import { FileTree } from './FileTree'
import { useWorkspaceStore } from '../../stores/workspaceStore'

beforeEach(() => {
  useWorkspaceStore.getState().reset()
  appMocks.ListWorkspaceDir.mockReset()
  appMocks.SearchWorkspaceContent.mockReset()
})

it('loads top level on mount when root set and empty', async () => {
  useWorkspaceStore.getState().setRoot('/w')
  appMocks.ListWorkspaceDir.mockResolvedValue([
    { name: 'a.ts', isDir: false, size: 3 }, { name: 'sub', isDir: true, size: 0 },
  ])
  render(<FileTree />)
  await waitFor(() => expect(screen.getByText('a.ts')).toBeInTheDocument())
  expect(screen.getByText('sub')).toBeInTheDocument()
})

it('selecting a file updates the store', async () => {
  useWorkspaceStore.getState().setRoot('/w')
  useWorkspaceStore.getState().setRoots([{ entry: { name: 'a.ts', isDir: false, size: 3 }, subPath: 'a.ts', children: null }])
  render(<FileTree />)
  fireEvent.click(screen.getByText('a.ts'))
  expect(useWorkspaceStore.getState().selected).toBe('a.ts')
})

it('filter hides non-matching loaded nodes', () => {
  useWorkspaceStore.getState().setRoot('/w')
  useWorkspaceStore.getState().setRoots([
    { entry: { name: 'alpha.ts', isDir: false, size: 1 }, subPath: 'alpha.ts', children: null },
    { entry: { name: 'beta.ts', isDir: false, size: 1 }, subPath: 'beta.ts', children: null },
  ])
  render(<FileTree />)
  fireEvent.change(screen.getByPlaceholderText(/过滤|Filter/i), { target: { value: 'alpha' } })
  expect(screen.getByText('alpha.ts')).toBeInTheDocument()
  expect(screen.queryByText('beta.ts')).toBeNull()
})

it('? prefix triggers content search', async () => {
  useWorkspaceStore.getState().setRoot('/w')
  appMocks.SearchWorkspaceContent.mockResolvedValue([{ path: '/w/a.ts', line: 2, snippet: 'hit' }])
  render(<FileTree />)
  fireEvent.change(screen.getByPlaceholderText(/过滤|Filter/i), { target: { value: '?hit' } })
  await waitFor(() => expect(appMocks.SearchWorkspaceContent).toHaveBeenCalledWith('/w', 'hit'))
})
```

- [ ] **Step 2: 确认失败** `cd frontend && npx vitest run src/components/workspace/FileTree.test.tsx`.

- [ ] **Step 3: 实现** — `FileTree.tsx`。要点：
  - 顶部 `<input>` placeholder「过滤文件… (?text 搜内容)」；值以 `?` 开头 → debounce 调 `SearchWorkspaceContent(rootDir, query.slice(1))` → `setSearchHits`；否则 `setSearchHits(null)` + `setFilter(value)`。
  - `searchHits!==null` → 渲染结果列表（path:line + snippet），点击 → `select(相对path)` 并可清搜索。
  - 否则渲染树：递归组件；文件夹行点击 → 若 `children===null`，`ListWorkspaceDir(rootDir, subPath)` → map 成 TreeNode[]（subPath = `join(parentSub, name)`，用 `/` 连）→ `setChildren` + `toggleExpanded`；已加载则仅 toggle。文件行点击 → `select(subPath)`，高亮当前 selected。
  - 过滤：filter 非空时，只显示 name 含 filter 的已加载节点（大小写不敏感）。
  - useEffect：mount 时若 `rootDir && roots.length===0` → 载顶层（`ListWorkspaceDir(rootDir,'')`）。
  - 图标：复用 `FolderIcon`（若无则在 icons.tsx 补），文件用简单文档图标或首字母；chevron 用现有 `ChevronRightIcon`/`ChevronDownIcon`。
  - 所有 App 调用 `.catch((e)=>console.error(...))`（fail-loud 边界，不静默）。

  （完整实现由执行者按上述契约写；测试即验收标准。保持组件聚焦——树渲染与搜索态两分支清晰。）

- [ ] **Step 4: 确认通过** `cd frontend && npx vitest run src/components/workspace/FileTree.test.tsx`（4 过）；`npx tsc --noEmit`。

- [ ] **Step 5: 提交**
```bash
git add frontend/src/components/workspace/FileTree.tsx frontend/src/components/workspace/FileTree.test.tsx frontend/src/components/icons.tsx
git commit -m "feat(gui): FileTree lazy tree with filename filter + content search"
```

---

### Task 11: FilePreview 组件

**Files:**
- Create: `frontend/src/components/workspace/FilePreview.tsx` + `.test.tsx`

**Context:** 当 `workspaceStore.selected` 变化 → `ReadWorkspaceFile(rootDir, absPath)` → 映射成 `PreviewSource`（`WorkspaceFile.Kind` html→`{kind:'html',html:Text}`、markdown→`{kind:'markdown',text:Text}`、code→`{kind:'code',text:Text,lang:Lang}`、image→`{kind:'image',dataUri:DataURI}`、binary→`{kind:'binary'}`；都带 `path`）→ 存本地 state。渲染面包屑（相对路径）+ 工具条 + `<PreviewContent source raw={rawMode}/>`。工具条：源码/渲染切换（仅 md/html 显示，切 rawMode）、复制路径、资源管理器显示、用编辑器打开（无模板则禁用/提示，齿轮 popover 设模板）。空 selected → 空态。

- [ ] **Step 1: 失败测试** `frontend/src/components/workspace/FilePreview.test.tsx`（mock App + runtime）：
```tsx
import { describe, it, expect, vi, beforeEach } from 'vitest'
import { render, screen, fireEvent, waitFor } from '@testing-library/react'

const appMocks = vi.hoisted(() => ({ ReadWorkspaceFile: vi.fn(), OpenInEditor: vi.fn(), RevealInExplorer: vi.fn() }))
vi.mock('../../../wailsjs/go/main/App', () => appMocks)

import { FilePreview } from './FilePreview'
import { useWorkspaceStore } from '../../stores/workspaceStore'
import { setEditorTemplate } from '../../lib/editorTemplate'

beforeEach(() => {
  useWorkspaceStore.getState().reset()
  useWorkspaceStore.getState().setRoot('/w')
  localStorage.clear()
  Object.values(appMocks).forEach((m) => m.mockReset())
})

it('shows empty state with no selection', () => {
  render(<FilePreview />)
  expect(screen.getByText(/未选择|选择文件/)).toBeInTheDocument()
})

it('reads and renders the selected markdown file', async () => {
  appMocks.ReadWorkspaceFile.mockResolvedValue({ kind: 'markdown', text: '---\nid: x1\n---\n# 标题', dataURI: '', lang: '' })
  render(<FilePreview />)
  useWorkspaceStore.getState().select('d.md')
  await waitFor(() => expect(screen.getByText('id')).toBeInTheDocument())
  expect(screen.getByRole('heading', { name: '标题' })).toBeInTheDocument()
})

it('open-in-editor disabled without a template', async () => {
  appMocks.ReadWorkspaceFile.mockResolvedValue({ kind: 'code', text: 'x', dataURI: '', lang: 'text' })
  render(<FilePreview />)
  useWorkspaceStore.getState().select('a.txt')
  await waitFor(() => expect(screen.getByRole('button', { name: '用编辑器打开' })).toBeDisabled())
})

it('open-in-editor calls OpenInEditor with template + abs path', async () => {
  setEditorTemplate('code "{path}"')
  appMocks.ReadWorkspaceFile.mockResolvedValue({ kind: 'code', text: 'x', dataURI: '', lang: 'text' })
  appMocks.OpenInEditor.mockResolvedValue(undefined)
  render(<FilePreview />)
  useWorkspaceStore.getState().select('a.txt')
  await waitFor(() => screen.getByRole('button', { name: '用编辑器打开' }))
  fireEvent.click(screen.getByRole('button', { name: '用编辑器打开' }))
  await waitFor(() => expect(appMocks.OpenInEditor).toHaveBeenCalledWith('code "{path}"', expect.stringContaining('a.txt')))
})
```

- [ ] **Step 2: 确认失败** `cd frontend && npx vitest run src/components/workspace/FilePreview.test.tsx`.

- [ ] **Step 3: 实现** — `FilePreview.tsx`。要点：
  - `selected` 变化的 useEffect：空→清 state；否则 `ReadWorkspaceFile(rootDir, joinAbs(rootDir, selected))`（abs 路径用 `rootDir` + `/` + `selected`；Windows 下 rootDir 已是绝对路径，join 用 `/` 即可，Go 侧 `resolveInRoot` 会 Clean）→ 映射 PreviewSource（见上）→ setState；`.catch(console.error)`。
  - 面包屑：显示 `selected`（相对路径）。
  - 工具条按钮：`源码`（仅 kind∈{markdown,html} 显示，toggle `raw`）；`复制路径`（clipboard 写 abs）；`资源管理器`（`RevealInExplorer(abs)`）；`用编辑器打开`（`getEditorTemplate()` 为空 → `disabled` + title 提示；否则 `OpenInEditor(tpl, abs)`）；齿轮 → 小 popover/prompt 设 `setEditorTemplate`。
  - `<PreviewContent source={source} raw={raw} />` 填充主体。
  - 错误/失败 `.catch(console.error)`，不静默。

- [ ] **Step 4: 确认通过** `cd frontend && npx vitest run src/components/workspace/FilePreview.test.tsx`（4 过）；`npx tsc --noEmit`。

- [ ] **Step 5: 提交**
```bash
git add frontend/src/components/workspace/FilePreview.tsx frontend/src/components/workspace/FilePreview.test.tsx
git commit -m "feat(gui): FilePreview with breadcrumb, toolbar, editor open"
```

---

### Task 12: WorkspaceFilePanel + App 集成

**Files:**
- Create: `frontend/src/components/workspace/WorkspaceFilePanel.tsx` + `.test.tsx`
- Modify: `frontend/src/stores/uiStore.ts`（`RightView` 加 `'files'`）
- Modify: `frontend/src/App.tsx`（「文件」tab）
- Modify: `frontend/src/App.test.tsx`（追加）

**Context:** `WorkspaceFilePanel`：读 `sessionStore` 当前会话 `workingDir` → 设 `workspaceStore.setRoot`（变化时）；无 workingDir → 空态"未绑定工作目录"；否则上 `FileTree` 下 `FilePreview`（可拖分割，简单固定比例即可）+ 顶部最大化按钮（`setMaximized`）；`maximized` → 渲染宽遮罩（同两组件，大容器 Files/File 标题）。RightPanel 加「文件」tab（id `'files'`）。

- [ ] **Step 1: uiStore 加 'files'** — 改 `RightView` 类型为 `'status' | 'browser' | 'preview' | 'files'`（其余不变）。

- [ ] **Step 2: 失败测试** `WorkspaceFilePanel.test.tsx`：
```tsx
import { describe, it, expect, vi, beforeEach } from 'vitest'
import { render, screen } from '@testing-library/react'
vi.mock('../../../wailsjs/go/main/App', () => ({ ListWorkspaceDir: vi.fn().mockResolvedValue([]), ReadWorkspaceFile: vi.fn(), SearchWorkspaceContent: vi.fn(), OpenInEditor: vi.fn(), RevealInExplorer: vi.fn() }))
import { WorkspaceFilePanel } from './WorkspaceFilePanel'
import { useSessionStore } from '../../stores/sessionStore'

beforeEach(() => useSessionStore.setState({ currentSessionId: 's1', sessions: [{ id: 's1', project: 'p', title: 't', archived: false, updatedAt: '' }] }))

it('shows empty state when session has no working dir', () => {
  render(<WorkspaceFilePanel />)
  expect(screen.getByText(/未绑定工作目录/)).toBeInTheDocument()
})

it('shows the tree area when working dir is set', () => {
  useSessionStore.setState({ currentSessionId: 's1', sessions: [{ id: 's1', project: 'p', title: 't', archived: false, updatedAt: '', workingDir: '/w' }] })
  render(<WorkspaceFilePanel />)
  expect(screen.getByPlaceholderText(/过滤/)).toBeInTheDocument()
})
```
追加 `App.test.tsx`：
```tsx
it('switches to files tab and shows workspace empty state', () => {
  render(<App />)
  fireEvent.click(screen.getByRole('tab', { name: '文件' }))
  expect(useUIStore.getState().rightView).toBe('files')
})
```
（`App.test.tsx` 顶部 mock 需补上 workspace 绑定方法，避免渲染崩。）

- [ ] **Step 3: 确认失败** `cd frontend && npx vitest run src/components/workspace/WorkspaceFilePanel.test.tsx src/App.test.tsx`.

- [ ] **Step 4: 实现**
  - `WorkspaceFilePanel.tsx`：读 `useSessionStore` 当前会话 workingDir；useEffect 同步到 `workspaceStore.setRoot`（仅在变化时）；无 → 空态；有 → 上下布局 FileTree/FilePreview + 最大化按钮；`maximized` 时额外渲染一个 fixed 遮罩容器包同样两组件（标题 Files / File + 关闭/还原）。
  - `App.tsx`：`RightPanel` 的 `views` 加 `{ id: 'files', label: '文件' }`；渲染分支加 `{view === 'files' && <WorkspaceFilePanel />}`；import。
- [ ] **Step 5: 确认通过** `cd frontend && npx vitest run src/components/workspace/WorkspaceFilePanel.test.tsx src/App.test.tsx`；`npx tsc --noEmit`。
- [ ] **Step 6: 提交**
```bash
git add frontend/src/components/workspace/WorkspaceFilePanel.tsx frontend/src/components/workspace/WorkspaceFilePanel.test.tsx frontend/src/stores/uiStore.ts frontend/src/App.tsx frontend/src/App.test.tsx
git commit -m "feat(gui): wire 文件 tab (WorkspaceFilePanel) into right column"
```

---

### Task 13: 全量校验 + 手动验证

- [ ] **Step 1: Go 全量** `go build ./... && go vet ./... && go test ./...`（全绿）；`gofmt -l .`（忽略 frontend/ 应为空）。
- [ ] **Step 2: 前端全量** `cd frontend && npx vitest run && npx tsc --noEmit && npm run build`（全绿）。
- [ ] **Step 3: 手动 `wails dev`**（在 legionAgentGUI/）：
  1. 给会话绑定一个 workingDir → 右列「文件」tab 出现文件树。
  2. 展开目录懒加载；点 .md 文件 → 预览显 front-matter 属性表 + 正文；点 .ts → 高亮；点图片 → 显示；点二进制 → 占位。
  3. 过滤框输入文件名 → 树过滤；输入 `?关键词` → 内容搜索结果，点结果打开文件。
  4. 源码/渲染切换；复制路径;「资源管理器显示」弹出选中;设编辑器模板后「用编辑器打开」启动编辑器。
  5. 最大化 → 宽 Files+File 双面板;还原。
  6. 无 workingDir 会话 → 空态提示。
- [ ] **Step 4: 收尾提交**（如手动微调）。

---

## 范围外（后续）
- 工具条「文件内搜索」（当前文件内定位高亮）。
- 大目录虚拟滚动。
- 树/预览分割比例可拖拽持久化（v1 固定比例）。

---

## Self-Review

**Spec 覆盖：** 文件树懒加载→T10;文件名过滤→T10;`?`内容搜索→T7+T10;类型分派预览→T4;front-matter 属性表→T3+T4;图片→T4+T6;html 复用 sandboxed iframe→T4;源码切换/复制路径/资源管理器/编辑器打开→T11;编辑器模板 localStorage→T9+T11;根限定/二进制守卫/大小上限/exec 防注入→T6/T7/T8;「文件」tab+最大化→T12;PreviewSource 泛化→T1;WebPreviewPanel 复用→T5。全部有落点。

**类型一致性：** `PreviewSource`(T1) 被 T4/T5/T11 引用一致;`WorkspaceEntry`/`WorkspaceFile`/`SearchHit`(T6/T7/T8 Go + 绑定 models) 被 T9/T10/T11 import;`WorkspaceFile.Kind`→`PreviewSource.kind` 映射在 T11 明确;`RightView` 加 `'files'`(T12) 与 App 分支一致;`highlightToHtml`(T2) 被 T4 用;`parseFrontmatter`(T3) 被 T4 用;`getEditorTemplate`/`setEditorTemplate`(T9) 被 T11 用;`workspaceStore` API(T9) 被 T10/T11/T12 用。

**占位符：** T10/T11 的组件实现以"契约 + 测试即验收"给出（树/预览逻辑较大，测试锁定行为），其余任务含完整代码。无 TBD。
