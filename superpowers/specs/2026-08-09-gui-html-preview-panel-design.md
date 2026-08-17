---
id: "design-gui-html-preview-panel-001"
title: "GUI HTML 预览面板技术 spec（iframe srcdoc 静态渲染）"
aliases: ["HTML 预览面板", "WebPreviewPanel", "html preview panel", "webview 面板"]
type: "design"
category: "superpowers/specs"
tags: ["gui", "react", "iframe", "webview", "html", "preview", "wails", "spec"]
version: "1.0.0"
created: "2026-08-09"
updated: "2026-08-09"
author: "jxncyjq"
status: "draft"
parent: null
children: []
related_docs:
  - id: "design-agent-browser-view-ui-001"
    relation: "related"
    path: "./2026-08-08-agent-browser-view-ui-design.md"
---

# GUI HTML 预览面板技术 spec

> 在 Wails GUI 右侧分栏内，用 `<iframe srcdoc>` 静态渲染纯 HTML 内容（agent 富 HTML、markdown 工具转出的完整 HTML、本地 .html 文件）。远程 http(s) URL 不进面板，统一丢系统浏览器。**零脚本、零交互、无爬虫**——`sandbox=""` 全静态。参考 hermes-desktop 的 WebPreviewPanel，但去掉其 Electron `<webview>` 专属的导航/前进后退/元素审查/地址栏。

<!-- @section: overview -->
## 概述

### 目标
让 GUI 能就地展示"纯 HTML 页面"——典型如 markdown 工具转出的完整 HTML 文档、agent 返回的富 HTML 片段、磁盘上的 .html 报告。只读展示，不需要用户在页内交互。

### 非目标（明确砍掉）
- ❌ 内嵌浏览远程网站（http(s) URL 一律外部浏览器打开，不在面板内渲染）
- ❌ 导航栏 / 前进后退 / 刷新历史 / 手动地址栏输入
- ❌ 元素审查 / 点选 / 与页面交互
- ❌ 爬虫 / 抓取 / 服务端代理拉取
- ❌ 独立原生窗口（不引入 go-webview2）

<!-- @section: background -->
## 背景与选型依据

### 为什么不用 go-webview2 / 原生 webview
`go-webview2` 是 Wails 官方 fork 的 WebView2 绑定，**本身就是 Wails v2 创建主窗口的底层库**，README 明确 "not intended to be used as a standalone package"。用它另开独立 webview 的问题：

| 维度 | go-webview2 独立窗口 | iframe 面板（本方案） |
|---|---|---|
| 平台 | 仅 Windows（需 WebView2 运行时 + DLL） | 跟随 Wails，Win/Mac/Linux 全通 |
| 布局 | 只能开独立 OS 窗口，无法嵌入主窗口做"右侧分栏" | 主窗口内一块 React 区域 |
| 生命周期 | 自己管 HWND / 消息循环 / 定位，与 Wails 运行时争控制 | React 组件挂载即可 |
| 多窗口 | Wails v2 声明式 API 不支持多窗口（v3 才规划） | 无关 |

Wails 官方讨论结论：单窗口内嵌多 webview "the only way is iframe"（[Discussion #1163](https://github.com/wailsapp/wails/discussions/1163)）。

### iframe 的真限制及其规避
`<iframe src="远程URL">` 会被大量站点的 `X-Frame-Options` / CSP `frame-ancestors` 拒绝（GitHub、Google、多数文档站禁止被嵌），导致白屏。hermes-desktop 能浏览任意 URL，是因为它是 **Electron**，用原生 `<webview>`（独立 web contents，无视 X-Frame-Options）——这是 Wails 不具备的能力。

**规避方式**：面板只渲染"自己的 HTML"（走 `srcdoc`，不受 X-Frame-Options 约束）；远程 URL 改由 Wails `BrowserOpenURL` 丢系统浏览器。X-Frame-Options 问题与"远程脚本安全"问题一并消失。

<!-- @section: architecture -->
## 架构

单 webview 内右侧可调宽分栏。纯前端 React 组件 + 少量 Go 绑定（仅"读本地文件"与"开外部浏览器"）。

```
AppLayout (flex row)
├── ChatPanel               (主区, flex-1)
└── WebPreviewPanel         (右侧, 条件渲染, 可拖宽, 宽度持久化)
      └── <iframe srcdoc sandbox="">
```

状态收敛到 zustand `previewStore`（单例面板，新预览替换旧的）：

```ts
interface PreviewStore {
  source: PreviewSource | null;
  isOpen: boolean;
  widthPx: number;              // 持久化到 localStorage
  open(src: PreviewSource): void;
  close(): void;
  setWidth(px: number): void;
}
```

<!-- @section: components -->
## 核心组件与接口

所有进面板的内容最终都收敛成 **HTML 字符串 → `<iframe srcdoc={html}>`**。

```ts
type PreviewSource =
  | { kind: "html";      html: string; title?: string; sourceUrl?: string }
  | { kind: "localFile"; path: string };   // 经 Go 后端 ReadHTMLFile 解析为 html 字符串后再 srcdoc
```

### 组件清单
| 组件/文件 | 职责 |
|---|---|
| `src/components/WebPreviewPanel.tsx` | 面板容器：工具条 + iframe + 拖宽手柄 |
| `src/stores/previewStore.ts` | zustand 状态：source / isOpen / widthPx |
| `src/lib/openLink.ts` | 链接分流工具：http(s) → `BrowserOpenURL`；其他不处理 |
| `MessageBubble.tsx`（改） | ① 渲染的 http 链接点击走 `openLink` ② html 代码块加"预览"按钮 |
| Go: `app.go` / 绑定 | `ReadHTMLFile(path) (string, error)`；监听/发送 `preview:open` event |

### 面板工具条（无地址栏）
- 左：标题（`source.title` 或文件名，缺省"HTML 预览"）
- 右：`外部浏览器打开`按钮（仅当 `source.sourceUrl` 存在时可见）+ `关闭`按钮

<!-- @section: data-flow -->
## 数据流（三条触发链）

| 触发 | 路径 |
|---|---|
| 点聊天里的 http(s) 链接 | `MessageBubble` 链接点击 → `openLink(url)` → `BrowserOpenURL(url)`（**不进面板**） |
| HTML 代码块"预览"按钮 | 代码块 lang=`html` → 渲染"预览"按钮 → `previewStore.open({ kind:"html", html })` |
| agent 工具主动推送 | Go 侧 `runtime.EventsEmit(ctx, "preview:open", payload)` → 前端 `EventsOn("preview:open")` → `open(source)` |
| 本地 .html 文件 | `open({ kind:"localFile", path })` → 调 Go `ReadHTMLFile(path)` → 得字符串 → 转成 `kind:"html"` 灌 srcdoc |

<!-- @section: security -->
## 安全

- **iframe `sandbox=""`**：不加 `allow-scripts`、不加 `allow-same-origin`。静态渲染、零脚本执行、零 app/父页访问。完全贴合"纯 HTML、不需交互"。
  - 后果：agent 推送的 HTML 即便含 `<script>` 也不执行 → 防注入。
  - 已知取舍：本地 HTML 内的 JS 图表/脚本同样不会跑。当前需求无此项，**默认最安全**；若后续出现"带 JS 图表的报告"需求，再对 `localFile` 单独放开 `allow-scripts`（需独立评审）。
- **`ReadHTMLFile` 后端校验**：仅允许 `.html` / `.htm` 扩展名；拒绝路径穿越（`..`）；限定在允许的工作目录根内（复用现有 working_dir 解析逻辑）。
- **链接分流**：只对 `http:` / `https:` 协议调 `BrowserOpenURL`；`file:` / `javascript:` 等协议一律忽略，不打开。

<!-- @section: testing -->
## 测试

### 前端（vitest + RTL）
- srcdoc 正确注入传入的 html 字符串
- iframe 带 `sandbox` 属性且**不含** `allow-scripts`
- `close()` 清空 source 且面板不渲染
- 点 http 链接调用 `BrowserOpenURL`（mock），且**不**触发 `previewStore.open`
- html 代码块渲染出"预览"按钮，点击后 store.source.kind === "html"
- 宽度拖拽后持久化到 localStorage

### 后端（Go）
- `ReadHTMLFile` 正常读 `.html` 返回内容
- 拒绝非 `.html/.htm` 扩展
- 拒绝含 `..` 的路径 / 越出根目录的路径

### 手动（wails dev）
- 三条触发链各跑一遍：代码块预览按钮、agent event 推送、本地文件打开
- http 链接点击确认弹系统浏览器、面板不动

<!-- @section: scope -->
## 范围与工作量

**新增**：`WebPreviewPanel.tsx`、`previewStore.ts`、`openLink.ts`、Go `ReadHTMLFile` 绑定 + `preview:open` event 通道。
**改动**：`MessageBubble.tsx`（链接分流 + html 代码块预览按钮）、`AppLayout`（挂载右侧面板 + flex 布局）。
**依赖**：无新增 npm 包（React/zustand/Tailwind 已有）；无新增 Go 依赖。

<!-- @section: open-questions -->
## 待确认 / 后续

- 本地 HTML 带 JS 图表需求：暂无 → `sandbox=""` 默认。出现后再评审放开 `localFile` 脚本。
- `preview:open` event payload 的具体字段（title/html/path）与 agent 侧工具如何产出，落实现计划时对齐后端。
