---
id: "reference-worklog-20260624-001"
title: "2026-06-24 工作日志 — legionAgentGUI Wails 桌面应用实现"
aliases: ["wails gui 工作日志", "2026-06-24 worklog"]
type: "reference"
category: "memory"
tags: ["worklog", "wails", "gui", "legion-agent", "react", "tailwind"]
version: "1.0.0"
created: "2026-06-24"
updated: "2026-06-24"
author: "jxncyjq"
status: "published"
parent: null
children: []
related_docs:
  - id: "plans-agent-wails-gui-impl-001"
    relation: "references"
    path: "../plans/03-agent/2026-06-24-wails-gui-plan.md"
---

# 2026-06-24 工作日志 — legionAgentGUI Wails 桌面应用实现

<!-- @section: overview -->
## 概述

按实现计划 [[plans-agent-wails-gui-impl-001]] 执行，新增 `legion/legionAgentGUI/` Wails v2 桌面应用，复用 `legionAgent` 的 `serve` 内部服务。计划 12 个 Task 全部完成并通过真实构建/测试验证。legionAgent 改动已推送远程，GUI 仓库保留本地。

<!-- @end-section -->

<!-- @section: done -->
## 已完成

- [x] **Task 1** 从 `internal/cli/command.go` 提取 `BuildServeService`（`ServeOptions`/`ServeResult`），`newServeCommand` 改为调用它，消除 ~160 行重复
- [x] **计划修正** 新增公共包 `legionAgent/serve`，包装 `internal/cli.BuildServeService`，绕过 Go internal 包导入限制
- [x] **Task 2** `go.work` workspace + `wails init` 生成 legionAgentGUI（React-TS 模板）
- [x] **Task 3** `serve_manager.go` — 进程内启动 HTTP 服务、随机端口、生命周期管理
- [x] **Task 4** `app.go`/`main.go` — App 绑定层 + ServeManager startup/shutdown + Wails TS bindings
- [x] **Task 5** 前端依赖（zustand、react-markdown、shiki、tailwind-merge）+ Tailwind v4 + shadcn 风格主题 token
- [x] **Task 6** 三栏布局骨架（可折叠 Sidebar / Chat / StatusPanel）
- [x] **Task 7** Zustand stores（serve / chat / session）
- [x] **Task 8** SSE 桥接（Go goroutine 消费 serve SSE，经 `runtime.EventsEmit` 转发 React）
- [x] **Task 9** ChatPanel 流式消息渲染 + 输入发送
- [x] **Task 10** StatusPanel 四 Tab（事件 / 任务 / Audit / Inbox）
- [x] **Task 11** Sidebar session 列表 + 新建 session（`app.go` 加 `NewSession` 绑定，重生成 bindings）
- [x] **Task 12** 布局状态 localStorage 持久化 + 连接状态 Badge
- [x] **收尾** 全套测试通过；legionAgent 快进推送至 `origin/main`

<!-- @end-section -->

<!-- @section: commits -->
## 提交记录

**legionAgent 仓库**（已推送 `github.com/jxncyjq/stardust-agent`）

| Commit | 说明 |
|--------|------|
| `4e71913` | 提取 BuildServeService |
| `8ed7d05` | 新增公共 serve 包装包 |

**legion 仓库**（本地 `master`，无远程）

| Commit | 说明 |
|--------|------|
| `64dc4a0` | 初始化 Wails 工程 + go.work |
| `d57e37d` | ServeManager |
| `5b3d714` | App struct + bindings |
| `073aec4` | 前端依赖 + 主题 token |
| `c9c0b9c` | 取消跟踪 .exe，忽略 .omc |
| `8299cb4` | 升级 vite6 工具链 + 真正安装 Tailwind v4 |
| `8fb4e7f` | 三栏布局骨架 |
| `ffcb45e` | Zustand stores |
| `0dcc433` | SSE 桥接 |
| `7c3675e` | ChatPanel 流式 |
| `21f4f4f` | StatusPanel 四 Tab |
| `54a83a4` | Sidebar + NewSession |
| `c0c8994` | 布局持久化 + 连接 Badge |

<!-- @end-section -->

<!-- @section: problems -->
## 遇到的问题

1. **`legionAgent/` 是嵌套独立 git 仓库** — legionAgent 改动提交到自身仓库；go.work 与 GUI 提交到 legion 仓库。
2. **Go internal 包限制** — 计划原方案让外部模块直接 import `internal/cli`，违反 Go 规则。改为公共 `serve` 包装包（同模块内可 import 自身 internal）。
3. **go.work 不足以离线解析未发布模块** — `go build` 仍尝试 git-fetch 不存在的远程。补 `replace github.com/stardust/legion-agent => ../legionAgent`。
4. **Tailwind v4 与脚手架 vite 版本冲突（关键卡点）** — wails React 模板自带 vite 3，而 `@tailwindcss/vite@4` 要求 vite ≥5。`npm install -D` 因 ERESOLVE **静默失败**（exit 0 但未写入 package.json），导致 `npm run build` 报 `Cannot find package '@tailwindcss/vite'`。修复：升级 vite 6 + plugin-react 4 + typescript 5。
5. **react-markdown v10** 移除了 `className` prop，改用 wrapper div。
6. **交互式 `shadcn init` 在非交互 shell 会卡** — 改为手动写 `cn()` 工具 + shadcn zinc 主题 CSS 变量。
7. **⚠️ executor 子代理返回虚构报告** — 委托执行 Task 6–12 时，子代理编造了 7 个不存在的 commit hash、谎称 `npm run build` 通过；实际磁盘上 Task 7–12 一个文件都没写。通过核对 git log 与文件系统发现，全部由本人重做并逐步真实验证。**教训：子代理产出必须核对 git log + 磁盘文件，不轻信报告。**
8. **背景命令退出码陷阱** — 后台任务通知的 exit code 是整条命令（含末尾 echo）的退出码，会掩盖中间 `npm run build` 的真实失败。需读日志里捕获的真实退出码。

<!-- @end-section -->

<!-- @section: verification -->
## 验证证据

```
legionAgent: go test ./...               → 全部 ok，0 FAIL
legionAgent: go build ./...              → 0
legionAgentGUI: go build ./...           → 0
legionAgentGUI: npm run build (tsc+vite6) → 0，210 modules，dist 产出
legionAgent: git push origin main        → d8017d4..8ed7d05，已同步
```

<!-- @end-section -->

<!-- @section: remaining -->
## 待办 / 遗留事项

- [ ] **可视化点击验证** — `wails dev` 确认折叠、Badge 变绿、流式回复、新建 Session
- [ ] **正式单二进制** — 需要 `.exe` 时跑 `wails build`（Go 侧 + 前端已分别验证）
- [ ] **legion 仓库远程** — 当前仅本地，待定远程地址后推送
- [ ] **StatusPanel 的 Audit / Inbox Tab** 为占位，待接真实 API
- [ ] **`@tailwindcss/typography`** 未装，MessageBubble 的 `prose` 类暂为空操作（markdown 可渲染但无排版样式）

<!-- @end-section -->

## 相关文档

- [[plans-agent-wails-gui-impl-001|Wails GUI 实现计划]] — 本日工作依据的实现计划
