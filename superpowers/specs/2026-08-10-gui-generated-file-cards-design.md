---
id: "design-gui-generated-file-cards-001"
title: "GUI 对话生成文件卡片技术 spec（子项目 B：卡片 + 预览/下载/外部打开/复制链接）"
aliases: ["生成文件卡片", "generated file cards", "对话文件卡片", "FileCard"]
type: "design"
category: "superpowers/specs"
tags: ["gui", "react", "wails", "generated-files", "file-card", "preview", "download", "spec"]
version: "1.0.0"
created: "2026-08-10"
updated: "2026-08-10"
author: "jxncyjq"
status: "draft"
parent: null
children: []
related_docs:
  - id: "design-generated-files-capture-001"
    relation: "extends"
    path: "../../../legion/legionAgent/docs/superpowers/specs/2026-08-10-generated-files-capture-design.md"
  - id: "design-gui-workspace-file-browser-001"
    relation: "extends"
    path: "./2026-08-09-gui-workspace-file-browser-design.md"
---

# GUI 对话生成文件卡片技术 spec（子项目 B）

> 消费子项目 A（PR #76）吐出的 `generated_files`，在 assistant 消息气泡下渲染可点文件卡片：可预览格式（html/md/文本/代码/图片）经 `/v1/files` URL 加载进「预览」tab（复用 PreviewContent），office/二进制走下载 + 外部程序打开，并支持复制链接地址。参考 hermes-desktop（`shell.openPath` 外部打开、`showSaveDialog`+拷贝下载）。**纯 legionAgentGUI 仓**：前端卡片 + 2 个 GUI-local Go 绑定，legionAgent 后端零改动（A 已提供数据）。

<!-- @section: overview -->
## 概述

### 背景
子项目 A（[[design-generated-files-capture-001]]）已让后端：捕获 write_file 产物、`taskResultResponse.generated_files` + 历史 turns 端点都返回 `[{path,url,download_url,name}]`、新增 `GET /v1/files` 文件端点。本子项目在 GUI 把这些数据渲染成卡片并接上动作。

### 目标
assistant 消息下显示生成文件卡片；预览（可预览格式）、下载、外部打开、复制链接。

### 非目标
- ❌ office（docx/xlsx/pptx）in-app 渲染（走下载/外部打开）
- ❌ 改 legionAgent 后端（A 已提供 `generated_files` + `/v1/files`）
- ❌ 拖拽、批量下载、卡片内编辑

<!-- @section: data-flow -->
## 数据流

- `chatStore.Message` 加 `generatedFiles?: GeneratedFile[]`（`{path,url,download_url,name}`）。
- 装配两路（`GetTaskResult`/`GetSessionTurns` 都返 `map[string]any`/`[]map[string]any`，`generated_files` 自动透传，无需改 Go）：
  - **实时**：任务完成 `GetTaskResult(taskID)`（ChatPanel ~183，已读 `prompt_tokens` 等旁）读 `res.generated_files` → 挂到对应 assistant Message。
  - **历史**：`GetSessionTurns`（ChatPanel ~383/518）每个 turn 读 `turn.generated_files` → 挂到重建的 Message。
- `MessageBubble`（assistant 分支）在正文/meta 之间渲染 `<FileCardList files={message.generatedFiles} />`。

```
后端 A: generated_files:[{path,url,download_url,name}]
   ├─ GetTaskResult(map 透传) ──┐
   └─ GetSessionTurns(map 透传) ─┤
                                 ▼
              Message.generatedFiles ──> MessageBubble ──> FileCardList ──> FileCard×N
```

<!-- @section: components -->
## 组件与接口

| 文件 | 职责 | 动作 |
|---|---|---|
| `frontend/src/components/FileCard.tsx` | 单卡片：图标(按扩展名)+名称+类型+动作菜单 | 新建 |
| `frontend/src/components/FileCardList.tsx` | assistant 气泡下的卡片列表 | 新建 |
| `frontend/src/lib/fetchPreview.ts` | 拿 card.url + token fetch → 建 PreviewSource | 新建 |
| `frontend/src/stores/chatStore.ts` | `Message.generatedFiles` + 类型 `GeneratedFile` | 改 |
| `frontend/src/components/MessageBubble.tsx` | assistant 气泡挂 `FileCardList` | 改 |
| `frontend/src/components/ChatPanel.tsx` | 实时/历史装配 `generatedFiles` 进 Message | 改 |
| `app_workspace.go` | `OpenPath` + `SaveGeneratedFile` 绑定 | 改 |

### GeneratedFile 类型（chatStore）
```ts
export interface GeneratedFile {
  path: string          // workspace 相对
  url: string           // 预览链接(相对 /v1/files?... 或配了域名的绝对)
  downloadUrl: string   // 下载链接(?download=1)
  name: string
}
```
（后端 JSON 是 `download_url`；装配时映射到 `downloadUrl`。）

### 可预览判定（按扩展名）
`html/htm/md/markdown/txt/log/json/代码类/png/jpg/jpeg/gif/webp/svg` → 可预览;其余(docx/xlsx/pptx/pdf/zip...) → 仅下载/外部打开。

<!-- @section: actions -->
## 四种动作

### 预览（可预览格式）— 走 /v1/files URL
`fetchPreview.ts`：
1. `GetBrowserEndpoint()` 拿 `{baseURL, token}`（现有绑定，浏览器 SSE 已用）。
2. 解析 `card.url`：相对(`/v1/files?...`)→ `baseURL + url`;绝对→原样。
3. `fetch(fullURL, { headers: { Authorization: 'Bearer ' + token } })`。
4. 按响应 `Content-Type`/扩展名建 `PreviewSource`：html→`{kind:'html',html:text}`、md→`{kind:'markdown',text}`、文本/代码→`{kind:'code',text,lang}`、图片→`{kind:'image',dataUri}`(blob→base64)。
5. `previewStore.open(source)` → 右列自动切「预览」tab（机制已存在）。
失败(4xx/网络)→ `console.error` + 卡片提示，不静默。

### 下载（全格式）— Save As 拷贝
新 Go 绑定 `SaveGeneratedFile(root, relPath)`（仿 hermes `saveMedia`）：`runtime.SaveFileDialog(a.ctx, {DefaultFilename: base})` → 用户选目标 → 根限定解析源 → `io.Copy` 拷贝。取消(空路径)= 正当可选。

### 外部打开（office 主动作/全格式）— OS 默认程序
新 Go 绑定 `OpenPath(root, relPath)`（仿 hermes `shell.openPath`）：根限定解析 abs → Windows `exec.Command("cmd", "/c", "start", "", abs)`（**不拼 shell 字符串**，abs 作独立 argv；`start` 首个 `""` 是标题占位）。office 用 Word/Excel 打开。

### 复制链接
`navigator.clipboard.writeText(card.url)`（相对或配了域名的绝对，按 A 的出口）。

<!-- @section: bindings -->
## 新增 Go 绑定（app_workspace.go，GUI-local，根限定 fail-loud）

复用现有 `resolveInRoot`(app_preview.go/app_workspace.go 已有根限定 helper)：
```go
func (a *App) OpenPath(root, relPath string) error          // ShellExecute 默认程序
func (a *App) SaveGeneratedFile(root, relPath string) error // SaveFileDialog + io.Copy
```
- root = 会话 workingDir（前端 `sessionStore.currentSession.workingDir` 已有）。
- 根限定：目标越出 root → wrapped error（不打开/拷贝越权文件）。
- exec 不走 shell（abs 作独立 argv）。SaveFileDialog 取消返回空路径 = 正当可选(不报错)。

<!-- @section: failloud -->
## 安全与错误（守 legionAgentGUI CLAUDE.md 铁律）

- 预览 fetch 失败 → `console.error` + 卡片可见提示，不静默吞。
- `OpenPath`/`SaveGeneratedFile` 根限定失败、拷贝/exec 失败 → 返回 wrapped error;前端 `.catch(console.error)`。
- SaveFileDialog 取消 = 契约可选（返回空、不报错）。
- 预览 token 从 `GetBrowserEndpoint` 活取（不缓存），与浏览器 SSE 一致——serve 重启换 token 不影响。
- 相对 `card.url` 必须用**当前** baseURL 拼(不缓存旧 baseURL)。

<!-- @section: testing -->
## 测试

### 前端（vitest + RTL，须在 frontend/ 跑）
- `Message.generatedFiles` 装配：GetTaskResult mock 带 generated_files → Message 有卡片;GetSessionTurns 历史同理。
- `FileCard`：可预览格式显"预览"按钮、office 不显;点下载/外部打开/复制链接调对应绑定/clipboard(mock)。
- `fetchPreview`：mock GetBrowserEndpoint + fetch → 各 content-type 建对应 PreviewSource kind;失败 → 不 open + console.error。
- 相对 url 用 baseURL 拼、绝对 url 原样。
- 空 generatedFiles → 不渲染卡片区。

### Go（app_workspace_test.go）
- `OpenPath`/`SaveGeneratedFile` 根限定：越权 relPath → error;正常路径构造正确(OpenPath 用假命令断言 argv 不拼 shell、abs 独立;SaveGeneratedFile 拷贝逻辑抽纯函数测)。

### 手动（wails dev）
- 让 agent 生成新 html/md/图片/docx → 气泡下出卡片;点 html/md/图片"预览"→ 预览 tab 显示;docx"外部打开"→ Word 起;"下载"→ 另存;"复制链接"→ 剪贴板得 url。

<!-- @section: scope -->
## 范围与工作量
**新增**：FileCard / FileCardList / fetchPreview；Go `OpenPath` + `SaveGeneratedFile`。
**改动**：chatStore(Message.generatedFiles)、MessageBubble(挂卡片)、ChatPanel(实时/历史装配)。
**依赖**：无新增 npm/Go 包(PreviewContent/previewStore/GetBrowserEndpoint/SaveFileDialog/exec 均已有)。
**仓库**：仅 legionAgentGUI。

<!-- @section: open-questions -->
## 待确认 / 后续
- 预览大文件/超时的取消与进度（v1 直 fetch，够用）。
- 卡片图标集：按扩展名映射一组 icon（复用/补 icons.tsx）。
- 跨平台 OpenPath（当前 Windows `start`；mac `open`/linux `xdg-open` 留部署时补）。
