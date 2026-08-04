---
id: "spec-agent-browser-001"
title: "Agent 内置浏览器子系统 PRD"
aliases: ["Agent Browser PRD", "内置浏览器 PRD", "browser runtime prd"]
type: "spec"
category: "design/architecture"
tags: ["agent", "browser", "runtime", "prd", "tool", "go-rod"]
version: "1.0.0"
created: "2026-08-04"
updated: "2026-08-04"
author: "jxncyjq"
status: "draft"
parent: null
children: []
related_docs:
  - id: "design-agent-browser-runtime-001"
    relation: "related_to"
    path: "../../superpowers/specs/2026-08-04-agent-browser-design.md"
  - id: "design-agent-working-modes-001"
    relation: "depends_on"
    path: "../../superpowers/specs/2026-07-15-agent-working-modes-design.md"
---

# Agent 内置浏览器子系统 PRD

<!-- @section: overview -->
## 概述

本 PRD 定义 Legion Agent 的**内置浏览器子系统（Browser Runtime）**的产品需求。目标：让 Agent 像人一样"看网页、点按钮、填表单、下载文件"，同时把资源占用、并发隔离、可靠性、安全边界控制在生产可用范围内。

技术方案已在架构设计文档 [[agent-browser-design]]（v2.4）锁定：Go 后端 + go-rod（CDP）+ Wails/React 前端 + HTTP POST/SSE 统一传输。本 PRD 只讲**产品视角**（为什么/给谁/做什么/验收标准）；实现级技术 spec 见 [[design-agent-browser-runtime-001|Browser Runtime 技术 spec]]。

<!-- @end-section -->

<!-- @section: background -->
## 背景

### 现状缺口

Legion Agent 当前只有三个 Web 工具（`internal/tool/web.go`、`websearch.go`、`webextract.go`）：

| 工具 | 能力 | 局限 |
|---|---|---|
| `fetch_url` | GET 单个 URL，抽正文 | 无状态、无会话、单次请求 |
| `web_search` | 搜索引擎查询 | 只出链接/摘要 |
| `web_extract` | 结构化抽取 | 无交互 |

三者都是**无状态单次请求**：不能登录、不能点按钮、不能填表单、不能处理多步页面流程、不能下载文件、看不到页面渲染结果。凡是"必须交互才能拿到信息"的任务（登录后台、翻页、点开详情、提交表单）全部做不了。

### 需求来源

Agent 要完成真实的 Web 任务，需要**有状态、可交互、语义级**的浏览能力——这正是内置浏览器子系统要补的缺口。架构选型、并发模型、安全模型、跨平台方案均已在 [[agent-browser-design]] 定案，本期进入产品需求与实现规划阶段。

<!-- @end-section -->

<!-- @section: goals -->
## 目标与非目标

### 目标

1. **语义级动作**：给 Agent 打开/点击/输入/读取/下载等"人类操作"级工具，而非裸 DOM 或任意脚本执行。
2. **token 可控表示**：把页面压成可定位、可点击、token 可控的表示（默认 a11y 语义树 + 稳定 `ref`）。
3. **会话/进程解耦**：逻辑会话与物理 Chromium 进程解耦，进程可复用/回收而不丢登录态。
4. **跨会话并发、同会话串行**：兼顾吞吐与页面状态一致。
5. **生产必需能力**：超时分层、错误规范化、SSRF 防护、沙箱隔离。
6. **跨三桌面端一致**：同一套 Go 上层逻辑在 Windows/macOS/Linux 行为一致，OS 差异收敛到平台抽象层。
7. **API 优先、部署无关**：一套后端核心，支持开发本地 / 独立服务 / Wails 独立 App 三种运行模式；前端能实时观看 Agent 浏览过程。

### 非目标（本期不做）

- 通用爬虫调度平台
- 绕过所有反爬（合规优先）
- `execute_arbitrary_js` 类黑盒工具
- 多租户强隔离与计费（服务模式的会话归属/配额做，但计费不做）

<!-- @end-section -->

<!-- @section: personas -->
## 用户与场景

### 角色

| 角色 | 是谁 | 通过什么用 |
|---|---|---|
| **Agent（主消费者）** | LLM 驱动的任务执行体 | 工具面（§6 九工具），一次一个语义动作 |
| **终端用户** | 用 GUI/TUI 的人 | 观看 Agent 浏览过程（只读视图），必要时接管 |
| **服务接入方** | 服务模式下的其他前端/客户端 | 网络 API（带鉴权） |

### 场景故事

**S1 — Agent 完成登录后台任务**
Agent 收到"查订单 X 状态"任务 → `browser_open` 打开后台 → `browser_read` 拿 a11y 树看到登录框 → `browser_type` 填账号密码并提交 → 页面跳转，自动附带新 observation → `browser_click` 点"订单管理" → 翻页找到订单 → `browser_extract` 按 schema 抽出状态回给模型。全程会话内串行，登录态持久化，任务结束落盘可恢复。

**S2 — 终端用户观看 + 接管**
用户在 GUI 里实时看到 Agent 操作的 Chromium 画面（screencast 帧经 SSE 推流）。遇到验证码时 Agent 报 `BLOCKED_BY_CAPTCHA`，用户切"接管模式"手动过验证码后交还 Agent（接管模式为分叉点，见风险）。

**S3 — 服务模式多前端并发**
后端作为独立服务部署，多个客户端各自带 token 调用，`session_id` 绑定调用方身份互不串号，按调用方配额限流。

<!-- @end-section -->

<!-- @section: functional-requirements -->
## 功能需求（FR）

每条含**验收标准**。工具面对应架构文档 §6，组件对应 §5。

### FR-1 语义工具面（九个工具）

Agent 可用的工具集：

```
browser_open(url, session_id?)               → { session_id, observation }
browser_read(session_id, mode?)              → observation    // 默认 a11y 树
browser_click(session_id, ref)               → observation
browser_type(session_id, ref, text, submit?) → observation
browser_scroll(session_id, direction)        → observation
browser_back(session_id) / browser_forward   → observation
browser_screenshot(session_id)               → image
browser_extract(session_id, schema)          → structured_data
browser_download_list(session_id)            → [file_id, …]
browser_close(session_id)                    → ok
```

**验收**：每个动作执行后自动附带最新 observation（省一次"看一下"往返）；不暴露任意脚本执行；`browser_extract` 在运行时内部完成结构化，长页面不塞回模型。

### FR-2 页面观测（三表示）

| 模式 | 内容 | 定位 | 适用 |
|---|---|---|---|
| A11y 语义树（默认） | 精简可交互元素列表 | 会话内稳定 `ref` | token 省、最稳 |
| DOM/Markdown | 正文抽取 | CSS/文本 | 纯阅读 |
| Screenshot | 截图 + 标注框 | 坐标/set-of-marks | 视觉兜底 |

**验收**：默认输出 a11y 树 + 稳定 `ref`（如 `e12`）；输出前预算裁剪（`max_elements`、按可见性排序、折叠重复项），把 50k token 页面压到 1~3k；`ref` 在会话内稳定，`click(ref)` 命中率高。

### FR-3 会话与登录态

**验收**：Session 绑定 `task_id`，持有 cookies/localStorage/tab 列表/动作历史；同 Session 内动作串行、跨 Session 并行；storageState 持久化到会话目录，App/服务重启可恢复登录态；TTL 空闲回收后 Session 可从磁盘重建（换新 Context）。

### FR-4 流式观测视图

**验收**：前端订阅 `GET /sessions/{id}/stream`（SSE），收到 `observation`/`frame`/`progress` 三类事件；截图帧限帧率（5~10 fps）、按需开关、降质；断线带 `Last-Event-ID` 重连，帧可丢但状态事件不丢（环形缓冲补发）；默认只读展示。

### FR-5 下载隔离

**验收**：下载落到受控缓存目录（按 `session_id` 分区、sha256 去重、LRU+TTL 清理）；返回 Agent 的是 `file_id` + 元信息（名/类型/大小），**不是路径**；大文件超限直接拒绝并给可读原因（`DOWNLOAD_TOO_LARGE`）。

### FR-6 三运行模式

**验收**：同一后端核心，开发本地 / 独立服务 / Wails 独立 App 三模式复用同一套 HTTP POST + SSE 传输；切换只改监听地址/鉴权，不改核心；App 模式进程内 loopback（`127.0.0.1` + 随机端口 + 一次性 token）自连。

### FR-7 安全基线

**验收**：自动化 Chromium 默认独立干净 profile + incognito，绝不接管用户日常浏览器身份；SSRF 防护禁止访问本机/局域网（DNS 解析后校验最终 IP 防 rebinding）；拦截 `file://`/`chrome://`/`data:` 危险协议；下载只落受控目录；服务模式鉴权 + 会话归属 + 配额。

<!-- @end-section -->

<!-- @section: nfr -->
## 质量需求（NFR）

### 资源与并发

- **App/开发模式**：目标"别拖垮用户机器"。`max_contexts` 2~4、`max_pages_per_context` 1~2、`mem_budget` ≤ 系统内存 25% 或绝对上限，触顶排队不扩容；<8GB 内存收紧到 1 context。用 PAL 实测系统内存定预算，不硬编码。
- **独立服务模式**：目标吞吐。进程数 × 每进程 Context 数容量模型（预留 20% headroom），按调用方配额 + 全局队列背压。

### 可靠性

超时分层（导航/动作/networkidle/整体墙钟）任一层超时返回可读错误；自动等待（go-rod 内建 + SPA DOM 稳定检测），避免裸 sleep；错误规范化为 Agent 可自恢复的语义错误码（`ELEMENT_NOT_FOUND`/`NAVIGATION_TIMEOUT`/`BLOCKED_BY_CAPTCHA`/`DOWNLOAD_TOO_LARGE`/`CONTEXT_EVICTED`）；僵死进程 `Reap`，graceful 重启期间新请求路由到其他进程。

### 跨平台一致

Session/Observation/Tool 层完全平台无关，OS 差异全收敛到 PAL（进程定位/终止、内存采样、数据目录、路径转换、沙箱包装）；三平台各一条 CI 跑同一套端到端用例（open/click/type/download/screenshot）。

<!-- @end-section -->

<!-- @section: milestones -->
## 里程碑

映射架构文档 §14 Roadmap 为可交付里程碑：

| M | 内容 | 交付判定 |
|---|---|---|
| **M1 最小闭环** | `open/read/click/type` + a11y observation + 单进程多 Context，开发本地模式跑通 | 四工具端到端可用，a11y `ref` 稳定命中 |
| **M2 流式观测** | SSE 三类事件 + Last-Event-ID 重连 | 前端看见 Agent 浏览过程，断线可续 |
| **M3 会话持久化** | storageState + TTL 回收（复用 SQLite `agent.db`） | 重启恢复登录态；空闲回收后可重建 |
| **M4 App 打包 + 跨平台** | Wails loopback 握手 + PAL 三平台 + Chromium 分发 + CI 矩阵 | 三平台独立 App 打出，CI 全绿 |
| **M5 安全基线** | 独立身份 + SSRF + 三平台原生沙箱；服务模式鉴权/归属/配额 | 安全用例通过 |
| **M6 资源与压测** | 进程池扩缩容 + 两种资源策略压测 | 压测达标 |
| **M7 按需扩展** | screenshot/set-of-marks、接管模式、反检测 | 分叉点按需交付 |

<!-- @end-section -->

<!-- @section: risks -->
## 风险与分叉点

| 分叉点 | 触发条件 | 影响 |
|---|---|---|
| 多租户强隔离/计费 | 服务模式商业化 | 一租户一进程/容器 + 计量埋点，本期不做 |
| 人工接管浏览 | 遇验证码/需人工 | 只读视图上加"接管模式"，回注鼠标/键盘事件 |
| 纯信息抽取 | 不需交互场景 | 砍 Session 锁 + 交互工具，退化为 navigate+render+extract |
| 视觉模型点坐标 | a11y 树不够用 | Observation 默认切 screenshot + set-of-marks |

**工程风险**：App 模式 loopback 端口暴露面（用 127.0.0.1 + 随机端口 + 一次性 token + Origin 校验四件套消化）；Chromium 分发体积（内置固定版 +150~300MB/平台，系统兜底）；Windows 内存偏高（PAL 实测校正）。

<!-- @end-section -->

## 相关文档

- [[agent-browser-design]] — 架构设计文档 v2.4（技术选型/接口/并发/安全/跨平台的定案来源）
- [[design-agent-browser-runtime-001|Browser Runtime 技术 spec]] — 本 PRD 的实现级技术 spec + legionAgent 接线
- [[design-agent-working-modes-001|Agent 工作模式]] — 浏览器工具作为副作用工具接入 Manual/Plan/Auto 审批的依赖
