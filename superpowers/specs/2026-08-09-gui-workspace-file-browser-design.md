---
id: "design-gui-workspace-file-browser-001"
title: "GUI 工作目录文件浏览器技术 spec（文件树 + 类型分派预览 + 外部编辑器）"
aliases: ["文件浏览器", "WorkspaceFilePanel", "workspace file browser", "文件 tab"]
type: "design"
category: "superpowers/specs"
tags: ["gui", "react", "wails", "file-browser", "preview", "shiki", "markdown", "spec"]
version: "1.0.0"
created: "2026-08-09"
updated: "2026-08-09"
author: "jxncyjq"
status: "draft"
parent: null
children: []
related_docs:
  - id: "design-gui-html-preview-panel-001"
    relation: "extends"
    path: "./2026-08-09-gui-html-preview-panel-design.md"
---

# GUI 工作目录文件浏览器技术 spec

> 在 Wails GUI 右列新增「文件」tab：浏览当前会话 `workingDir` 的懒加载文件树，选中文件按类型分派**只读预览**（代码高亮 / markdown 渲染含 front-matter 属性表 / 图片 / HTML sandboxed iframe），可复制路径、在资源管理器显示、用**外部配置的编辑器**打开。文件名过滤 + `?text` 内容搜索。停靠右列（上树下预览），可最大化成宽的 Files+File 双面板遮罩。**纯 GUI-local**：新增 Go 绑定直接读本机文件系统，不碰 legionAgent 后端、无 HTTP/proto 改动。参考 Obsidian 双面板结构。

<!-- @section: overview -->
## 概述

### 目标
让用户在 GUI 内浏览会话工作目录的文件、只读预览内容、并一键用外部编辑器打开——无需切到系统文件管理器。

### 非目标（明确砍掉）
- ❌ 编辑 / 保存文件（只读预览）
- ❌ 内嵌浏览远程目录 / 网络文件系统
- ❌ git 状态标记、diff、多选、拖拽、重命名 / 删除
- ❌ 修改 legionAgent 后端 / HTTP API / proto
- ❌ 通过内嵌 serve 的 HTTP 中转（文件操作走 GUI-local Go 直连文件系统）

### 与已合入的 HTML 预览面板的关系
本功能扩展 [[design-gui-html-preview-panel-001]]（PR #16 已合 master）：把只渲染 HTML 的 `WebPreviewPanel` 泛化为按 kind 分派的共享 `PreviewContent` 渲染器，「预览」tab（聊天触发的 HTML 预览）与「文件」tab 底部预览区共用它。`previewStore.PreviewSource` 从单一 `{kind:'html'}` 扩成联合类型。

<!-- @section: architecture -->
## 架构

单 Wails webview 内右列新增第 4 个 tab「文件」（状态 / 浏览器 / 预览 / **文件**）。纯前端 React + 5 个 GUI-local Go 绑定（直接 `os.ReadDir` / `os.ReadFile` / `os/exec`，不经内嵌 serve）。根 = 当前会话 `workingDir`（前端 `sessionStore` 已有，set-once 绑定）。

```
RightPanel (tabs)
└── 「文件」tab = WorkspaceFilePanel
      ├── 顶部：过滤栏（文件名过滤 / ?text 内容搜索）+ 最大化按钮
      ├── 上：FileTree（懒加载，占比可调）
      └── 下：FilePreview（路径面包屑 + 工具条 + PreviewContent）

最大化 → 宽遮罩：同样 FileTree + FilePreview，换大容器（Files 面板 / File 面板）
```

**GUI-local vs 后端**：`NewSession`/`ListSessions` 等经内嵌 legion-agent HTTP 服务；本功能是纯本机文件操作，直连文件系统（与 `PickDirectory`、`ReadHTMLFile` 同层），legionAgent 服务零参与。

<!-- @section: components -->
## 组件与接口

### 前端（新增 `src/components/workspace/`）
| 组件 / 文件 | 职责 |
|---|---|
| `WorkspaceFilePanel.tsx` | 「文件」tab 容器：过滤栏 + FileTree + FilePreview + 最大化开关；空态（无 workingDir） |
| `FileTree.tsx` | 懒加载文件树；文件名过滤；`?` 内容搜索结果态 |
| `FilePreview.tsx` | 路径面包屑 + 工具条（源码切换 / 复制路径 / 资源管理器显示 / 用编辑器打开）+ `PreviewContent` |
| `PreviewContent.tsx` | **共享** kind 分派渲染器（见下）；被 FilePreview 与重构后的 WebPreviewPanel 共用 |
| `WorkspaceOverlay.tsx` | 最大化遮罩：宽版 Files+File 双面板，复用同组件 |

### store
- `src/stores/workspaceStore.ts`（新建）：`{ rootDir, tree(展开态/子节点缓存), selected, filter, searchQuery, searchHits, expanded(Set), maximized }` + actions（`loadDir`/`toggleDir`/`select`/`setFilter`/`runSearch`/`setMaximized`）。
- `src/stores/previewStore.ts`（改）：`PreviewSource` 扩成联合类型。

### PreviewSource 联合类型（previewStore）
```ts
export type PreviewSource =
  | { kind: 'html';     html: string;    title?: string; sourceUrl?: string }
  | { kind: 'code';     text: string;    lang: string;   title?: string; path?: string }
  | { kind: 'markdown'; text: string;    title?: string; path?: string }
  | { kind: 'image';    dataUri: string; title?: string; path?: string }
  | { kind: 'binary';   title?: string;  path?: string } // 占位："不支持预览"
```

### PreviewContent 分派
- `html` → `sandbox=""` iframe srcdoc（沿用已合入实现）
- `code` → shiki 高亮（复用 `src/lib/highlighter.ts`；lang 未知回退纯文本）
- `markdown` → 顶部 front-matter **属性表**（解析 YAML 头 → Id/Title/Tags... 键值表）+ 下方 react-markdown 正文（复用 MessageBubble 同款 remark-gfm + shiki）
- `image` → `<img src={dataUri}>`
- `binary` → 占位提示

### 工具条动作（FilePreview 右上）
源码/渲染切换（md·html：渲染 ↔ raw `<pre>`）、复制路径、在资源管理器中显示、用编辑器打开。

### Go 绑定（新增 `app_workspace.go`，全部限定在 `root` 内，fail-loud）
```go
type WorkspaceEntry struct { Name string; IsDir bool; Size int64 }
type WorkspaceFile  struct { Kind string; Text string; DataURI string; Lang string } // Kind: code|markdown|html|image|binary
type SearchHit      struct { Path string; Line int; Snippet string }

func (a *App) ListWorkspaceDir(root, sub string) ([]WorkspaceEntry, error)
func (a *App) ReadWorkspaceFile(root, path string) (WorkspaceFile, error)
func (a *App) SearchWorkspaceContent(root, query string) ([]SearchHit, error)
func (a *App) OpenInEditor(template, path string) error
func (a *App) RevealInExplorer(path string) error
```

### 外部编辑器配置
命令模板（如 `code "{path}"`）是**机器相关的 GUI 偏好** → 存 `localStorage`（与主题、面板宽度同款），不进 agent.json、不落后端。设置页新增一个字段编辑它。每次「用编辑器打开」读 localStorage 模板传给 `OpenInEditor`。

<!-- @section: data-flow -->
## 数据流

| 动作 | 路径 |
|---|---|
| 打开「文件」tab | 读 `sessionStore` 当前会话 `workingDir` → `workspaceStore.rootDir`；空 → 空态"未绑定工作目录" |
| 展开文件夹 | `ListWorkspaceDir(root, sub)` 懒加载该层 → 缓存进 store 树 |
| 文件名过滤 | 纯前端过滤已加载节点 |
| `?text` 内容搜索 | `SearchWorkspaceContent(root, text)` → 结果列表态；点结果 → 选中该文件 |
| 选中文件 | `ReadWorkspaceFile(root, path)` → 按 Kind 建 `PreviewSource` → 灌 FilePreview |
| 源码/渲染切换 | 前端本地 state 在渲染 ↔ raw `<pre>` 间切 |
| 复制路径 | `navigator.clipboard.writeText(absPath)` |
| 资源管理器显示 | `RevealInExplorer(absPath)` |
| 用编辑器打开 | 读 localStorage 模板 → `OpenInEditor(template, absPath)` |
| 最大化 | `workspaceStore.maximized=true` → 渲染 `WorkspaceOverlay` |

<!-- @section: security -->
## 安全与错误（fail-loud，守 legionAgentGUI CLAUDE.md 铁律）

- **根限定**：所有绑定首参 `root`；目标路径 `filepath.Clean` 后用 `filepath.Rel(root, target)` 校验结果不以 `..` 开头、不越出 root，否则返回 wrapped error（`fmt.Errorf("<动作> %q: outside workspace root", path)`）。防越权读盘 / 越权 exec。
- **二进制守卫**：`ReadWorkspaceFile` 读前若检测 NUL 字节 / 非法 UTF-8 → 返回 `Kind:"binary"` 占位，不把二进制塞进文本渲染器。
- **大小上限**：预览读取封顶（2 MB）；超限返回 error（"文件过大，不预览"）。搜索单文件同样限大小。
- **exec 注入防护**：`OpenInEditor` 把 `template` 按 argv **引号感知 split**，`{path}` 替换为**单个 argv 元素**（绝不 shell 字符串拼接、不 `sh -c`）；找不到 editor / 退出非零 → 返回 error。模板缺 `{path}` 时把 path 追加为末尾 argv。
- **搜索封顶**：`SearchWorkspaceContent` 限遍历文件数、单文件大小、命中结果数；跳过二进制与超大文件；每项都不静默吞错（跳过要能解释）。
- **RevealInExplorer**：仅接受 root 内路径；Windows `explorer /select,<abs>`。

> 唯一豁免：contract 显式可选项（如 localStorage 无编辑器模板时「用编辑器打开」按钮禁用并提示去设置，属正当可选，非兜底）。

<!-- @section: testing -->
## 测试

### 前端（vitest + RTL，须在 `frontend/` 目录跑）
- 空态：无 workingDir 时提示、无树
- 树懒加载：展开目录调 `ListWorkspaceDir`（mock）并渲染子节点
- 文件名过滤：输入过滤已加载节点
- `?` 内容搜索：切换到搜索结果态、点结果选中文件
- 选文件 → `PreviewContent` 按 kind 分派（code/markdown/image/html/binary 各断言渲染物）
- front-matter 属性表：markdown 头解析成键值表 + 正文渲染
- 工具条：复制路径 / RevealInExplorer / OpenInEditor 调对应绑定（mock）；无模板时按钮禁用
- 源码/渲染切换：md 在渲染与 raw `<pre>` 间切
- `WebPreviewPanel` 重构后仍通过原 HTML 预览测试（回归）

### Go（`app_workspace_test.go`）
- `ListWorkspaceDir`：正常列一层；拒绝 root 外 / `..`
- `ReadWorkspaceFile`：文本→code/markdown/html 正确 Kind + lang；图片→image data URI；二进制→binary 占位；超大→error；root 外→error
- `SearchWorkspaceContent`：命中返回 path/line/snippet；跳过二进制；数量上限；root 外→error
- `OpenInEditor`：模板 argv 解析（含带空格路径用引号）；`{path}` 单 argv 替换；拒 shell 元字符注入（用假 editor 断言 argv）；缺模板→error
- `RevealInExplorer`：root 外→error

<!-- @section: scope -->
## 范围与工作量

**新增**：`WorkspaceFilePanel` / `FileTree` / `FilePreview` / `PreviewContent` / `WorkspaceOverlay` / `workspaceStore` / `app_workspace.go` + 设置页编辑器模板字段。
**改动**：`previewStore`（PreviewSource 联合）、`WebPreviewPanel`（改用 `PreviewContent`）、`App.tsx`（「文件」tab）、`uiStore`（rightView 加 `'files'`）、SettingsModal（编辑器字段）。
**依赖**：无新增 npm 包（shiki / react-markdown / zustand / Tailwind 已有）；无新增 Go 依赖。
**仓库**：仅 legionAgentGUI（Go + 前端）；legionAgent 后端零改动。

<!-- @section: open-questions -->
## 待确认 / 后续

- 工具条「文件内搜索」（当前文件内高亮定位）暂未纳入 v1，如需再加。
- 大型目录树的虚拟滚动 / 分页暂不做；若单目录条目极多再优化。
- front-matter 属性表仅展示（不可编辑），与 Obsidian 的可编辑属性不同——本 spec 只读。
