---
id: "reference-legion-agent-backend-api-001"
title: "Legion Agent 后端系统调用参考"
aliases: ["后端 API", "系统调用", "legionAgent HTTP 端点", "agent serve API"]
type: "reference"
category: "agents/reference"
tags: ["agent", "backend", "http", "api", "sse", "task", "session"]
version: "1.0.0"
created: "2026-08-27"
updated: "2026-08-27"
author: "jxncyjq"
status: "published"
parent: "reference-legion-agent-user-manual-001"
children: []
related_docs:
  - id: "reference-legion-agent-auth-001"
    relation: "depends_on"
    path: "./reference-legion-agent-auth-001.md"
  - id: "reference-legion-agent-integration-001"
    relation: "related_to"
    path: "./reference-legion-agent-integration-001.md"
  - id: "reference-legion-agent-http-service-001"
    relation: "related_to"
    path: "./reference-legion-agent-http-service-001.md"
  - id: "agent-http-api-001"
    relation: "related_to"
    path: "../legion-agent/http-api.md"
---

# Legion Agent 后端系统调用参考

本文是 `legionAgent`（`cmd/agent serve`）后端 HTTP 面的**全量端点手册**：路由、方法、鉴权口径、请求体、响应体、错误码，以及一次任务从提交到出结果的内部调用链。鉴权细节见 [[reference-legion-agent-auth-001|鉴权参考]]；客户端怎么接见 [[reference-legion-agent-integration-001|接入参考]]。

事实基准：`internal/server/http.go`（路由表与 handler）、`internal/server/openapi.go`（契约）、`internal/runtime`（任务执行链）。

<!-- @section: routing-model -->
## 路由模型

服务端不是 `http.ServeMux`，而是 `HTTPServer.ServeHTTP` 里一条 `switch` 显式匹配「方法 + 路径」。含义：

- 路径匹配是前缀/后缀判定，不做参数解析框架；未命中任何 case 返回 `404 {"error":"not found"}`。
- 每个请求先过 `authorized(r)`，未通过返回 `401 {"error":"unauthorized"}`，不进业务分支。
- 每个响应都带 `X-Request-ID`（请求头同名字段透传，缺失时服务端生成 `req-<hex>`；头名由 `server.request_id_header` 配置）。
- 每个请求出口都记 `http request handled`（method/path/status/elapsed_ms）并计入 `/metrics` 的状态码计数。
- 错误统一是 `{"error":"<message>"}`（`ErrorResponse`），不是纯文本。

<!-- @end-section -->

<!-- @section: endpoint-table -->
## 端点全表

「Token」列指是否需要 `Authorization: Bearer <admin_token>`（仅在配置了 token 时生效）。「RBAC」列指是否额外经 `X-Role` / `X-Company-ID` 判定。

### 健康、契约与观测

| 方法 | 路径 | Token | RBAC | 说明 |
|------|------|-------|------|------|
| GET | `/healthz` | 可豁免 | — | 存活检查，`server.public_health_enabled=true` 时匿名可访问 |
| GET | `/readyz` | 可豁免 | — | 依赖可用性（storage Ping），同上可匿名 |
| GET | `/metrics` | 是 | — | 进程内指标快照，`?format=prometheus` 输出 Prometheus 文本 |
| GET | `/debug/diagnostics` | 是 | — | 脱敏诊断：storage 驱动/路径、MaaS 地址、token 是否配置、scheduler 状态 |
| GET | `/debug/traces` | 是 | — | 脱敏 trace 快照 |
| GET | `/openapi.json` | 否（恒公开） | — | OpenAPI 3.1 契约 |

### 任务

| 方法 | 路径 | Token | RBAC | 说明 |
|------|------|-------|------|------|
| POST | `/v1/tasks` | 是 | — | 提交任务 |
| GET | `/v1/tasks` | 是 | — | 列出任务（按创建顺序，最新在后） |
| GET | `/v1/tasks/{id}` | 是 | 公司隔离 | 任务状态 |
| GET | `/v1/tasks/{id}/result` | 是 | — | 状态 + 答案文本 + token 用量 + 生成文件 |
| POST | `/v1/tasks/{id}/interrupt` | 是 | — | 中断正在运行的任务；未在运行返回 404 |
| POST | `/v1/tasks/{id}/approvals/{ticketID}` | 是 | — | Manual 模式工具审批裁决 |

### 会话与对话

| 方法 | 路径 | Token | RBAC | 说明 |
|------|------|-------|------|------|
| POST | `/v1/sessions` | 是 | — | 新建会话（id 由服务端生成） |
| GET | `/v1/sessions` | 是 | — | 会话列表，支持 `company_id`、`agent_id` 过滤 |
| PATCH | `/v1/sessions/{id}` | 是 | — | 改 title / project / archived / mode / working_dir |
| DELETE | `/v1/sessions/{id}` | 是 | — | 删除会话行 + 其磁盘会话目录 |
| GET | `/v1/sessions/{id}/turns` | 是 | — | 会话 turns，`limit=0` 或省略即全量 |

### Agent 与消息

| 方法 | 路径 | Token | RBAC | 说明 |
|------|------|-------|------|------|
| GET | `/v1/agents` | 是 | — | 已注册子 Agent 名单（配置 `agents` 的 key；默认 Agent 不在列，用空 `agent_id` 提交任务即可命中） |
| GET | `/v1/agents/{id}/messages` | 是 | 公司隔离 | 查询 inbox/outbox，支持 `company_id`、`status`、`limit`、`mark_read` |
| POST | `/v1/agents/{id}/messages` | 是 | 公司隔离 | 发送 AgentMessage |

### Workflow

| 方法 | 路径 | Token | RBAC | 说明 |
|------|------|-------|------|------|
| POST | `/v1/workflows` | 是 | — | 提交 workflow definition |
| GET | `/v1/workflows/{id}` | 是 | — | 查询 workflow 状态 |
| POST | `/v1/workflows/{id}/events` | 是 | — | 投递恢复事件（唤醒 waiting 节点） |
| GET | `/v1/workflows/waiting` | 是 | — | 列出 waiting workflow |

### 治理、审批与质量

| 方法 | 路径 | Token | RBAC | 说明 |
|------|------|-------|------|------|
| GET | `/v1/audit-events` | 是 | `read_audit`（admin/operator） | 审计事件 |
| GET | `/v1/runtime-events` | 是 | — | 最近运行时事件，最多 200 条，时间顺序 |
| GET | `/v1/quality/evals` | 是 | `read_quality` | 质量评估，支持 `agent_id`、`task_id`、`component` |
| GET | `/v1/approvals` | 是 | — | 未决审批工单；仅支持 `status=pending`，其他值 400 |

### 插件（WASM）

| 方法 | 路径 | Token | RBAC | 说明 |
|------|------|-------|------|------|
| GET | `/v1/plugins` | 是 | `read_plugin`（admin/operator） | 部署清单内插件及其声明/已授权能力 |
| POST | `/v1/plugins/{name}/grant` | 是 | `write_plugin`（**仅 admin**） | 授权插件能力 / host / path |
| POST | `/v1/plugins/{name}/deny` | 是 | `write_plugin`（**仅 admin**） | 撤销授权，保留注册 |

未配置 `plugins.manifest` 的进程，这三个端点返回 **404**（「本进程未装配插件加载器」），而不是空列表——「没装插件」和「这套部署压根没开插件」是两件事。

### 技能

| 方法 | 路径 | Token | RBAC | 说明 |
|------|------|-------|------|------|
| POST | `/v1/skills/install` | 是 | — | body `{"source":"github:owner/repo"}` |
| POST | `/v1/skills/update` | 是 | — | body `{"name":"..."}` |
| POST | `/v1/skills/uninstall` | 是 | — | body `{"name":"..."}` |

未装配 SkillManager 时返回 503。

### 事件流与文件

| 方法 | 路径 | Token | RBAC | 说明 |
|------|------|-------|------|------|
| GET | `/v1/events` | 是 | — | 平台事件 SSE，`?type=` 按事件类型过滤 |
| GET | `/v1/files` | 是 | — | 流式返回会话工作目录内的文件，参数 `session_id`、`path`、`download=1` |

### 内置浏览器

| 方法 | 路径 | Token | RBAC | 说明 |
|------|------|-------|------|------|
| GET | `/v1/browser/sessions/{id}/stream` | 是 | — | 浏览器会话 SSE（screencast 帧 + status），支持 `Last-Event-ID` 补发 status |
| POST | `/v1/browser/sessions/{id}/takeover` | 是 | — | body `{"enabled":true}` / `{"enabled":false}`，置/清人工接管标志 |
| POST | `/v1/browser/sessions/{id}/viewport` | 是 | — | body `{"width":1280,"height":800}`，设视口消除 letterbox |
| POST | `/v1/browser/sessions/{id}/input` | 是 | — | body `{"events":[...]}`，注入归一化输入事件 |

> 契约缺口（当前状态，非笔误）：`/openapi.json` **未收录** 4 个 `/v1/browser/...` 端点与 `/v1/tasks/{id}/interrupt`。按 OpenAPI 生成的客户端拿不到这些路由，需手写调用。

<!-- @end-section -->

<!-- @section: task-lifecycle -->
## 任务调用链

### 提交

```http
POST /v1/tasks
Authorization: Bearer <admin-token>
Content-Type: application/json

{"id":"task-1","company_id":"company-1","agent_id":"writer","session_id":"session-177...","input":"整理说明","images":[]}
```

`id` 与 `input` 必填（缺失 400）。`id` 冲突返回 **409**。响应 `201` 返回完整 `Task`。

带 `session_id` 时服务端会：

1. 加载会话；不存在返回 404。
2. 用会话的 `mode` 作为任务 mode；库里存了非法 mode 返回 500（不静默改成 auto）。
3. 继承会话 `working_dir`；非空但目录不存在返回 400。
4. 先落一条 user turn（id 固定为 `<taskID>:user`，重复提交不会重复写），再入队。

不带 `session_id` 的一次性任务，mode 为 `auto`。`images` 是 base64 图片数组，落库时不存原图，只在 turn 里追加 `[附图 N 张]` 标记。

### 执行链

```text
POST /v1/tasks -> TaskStore.Add（pending）
  -> coordinator 取任务（runtime.max_concurrent_tasks 并发上限，默认 4）
  -> AgentRuntimeResolver 按 agent_id 解析子 Agent 配置 / 模型 profile / 工具集
  -> Runtime 组装 prompt（上下文文件 + 最近 N 轮 turn + 能力目录）
  -> tool loop：模型返回 tool_calls -> Registry.Execute -> 结果作为 tool 消息追加
  -> 产出最终回答 -> 记 assistant turn -> task_completed 事件（含 token 用量）
```

工具循环的关键约束：

| 约束 | 配置 | 默认 |
|------|------|------|
| 最大连续工具轮数 | `runtime.max_tool_rounds` | 4 |
| 并发任务数 | `runtime.max_concurrent_tasks` | 4 |
| 会话压缩阈值 | `runtime.compact_token_threshold` | 0（关闭），单任务最多压缩 3 次 |
| 审批超时 | `runtime.approval_timeout_seconds` | 300 秒，超时按拒绝处理 |

### 任务状态

`pending` → `assigned` → `running` → `quality_review` → `done` / `failed` / `suspended` / `cancelled`。

`GET /v1/tasks/{id}/result` 只在终态才有答案文本，响应形如：

```json
{
  "task_id": "task-1",
  "status": "done",
  "result": "……模型答案……",
  "prompt_tokens": 12000,
  "completion_tokens": 800,
  "cached_tokens": 2048,
  "total_tokens": 12800,
  "elapsed_ms": 15230,
  "generated_files": [
    {
      "path": "docs/report.md",
      "name": "report.md",
      "url": "/v1/files?path=docs%2Freport.md&session_id=session-1",
      "download_url": "/v1/files?download=1&path=docs%2Freport.md&session_id=session-1"
    }
  ]
}
```

`generated_files` 的链接是**每次响应现拼**的：库里只存工作区相对路径，改了 `server.file_base_url` 立刻生效，无需数据迁移。数组为空表示这次任务没有成功写出新文件——`write_file` 默认 `overwrite=false`，写已存在文件会失败并被正确跳过，这是预期行为不是缺陷。

### 中断

```http
POST /v1/tasks/task-1/interrupt
```

成功返回 `204`；任务不在运行（已结束或不存在）返回 `404`——「没在跑」绝不会被报成「中断成功」。

<!-- @end-section -->

<!-- @section: sse -->
## 事件流

`GET /v1/events` 是 `text/event-stream`，连接建立即先刷 200 响应头（空闲总线也不会看起来像挂死）。帧格式：

```text
event: task_completed
id: <event-id>
: subject_id=<subject>
data: {"task_id":"task-1","message":"...","total_tokens":12800}
```

常见事件类型（与 `RuntimeEvent.Type` 同名，下划线形式）：

| 类型 | 含义 |
|------|------|
| `task_started` | 任务开始执行 |
| `inference_completed` | 一次模型推理完成 |
| `tool_call_requested` | 模型请求调用工具 |
| `tool_executed` / `tool_failed` | 工具执行成功 / 失败 |
| `tool_result` | 工具结果回填 |
| `tool_loop_broken` | 触发熔断（重复调用、超轮数） |
| `subtask_completed` | 子 Agent 委派任务完成 |
| `task_completed` | 任务结束，带 token 与耗时 |
| `task_cancelled` | 任务被 `/interrupt` 中断，状态落 `cancelled` |

脱敏规则：`prompt`、`input`、`secret`、`api_key`、`token` 等敏感键在出站前剔除，递归到嵌套 map；任何单个字符串值超过 512 字符会被截断。SSE 是**尽力而为**的通知层——事件总线内部快照才是权威，publish 失败只记 Warn，不影响任务本身。

浏览器会话流 `/v1/browser/sessions/{id}/stream` 是另一条 SSE：断线重连带 `Last-Event-ID` 时只补发 status 事件，**画面帧不补发**。

<!-- @end-section -->

<!-- @section: errors -->
## 错误码口径

| 状态 | 触发 |
|------|------|
| 400 | 请求体解析失败、必填缺失、参数非法（`mode` 非 manual/plan/auto、`limit` 非法、审批 decision 非 approve/deny、viewport 越界） |
| 401 | 配置了 admin token 但请求未带或带错 Bearer |
| 403 | 跨公司访问被拒（写审计事件 `access_denied.cross_company`）、RBAC 角色不足、`/v1/files` 路径越界、Origin 守卫拒绝 |
| 404 | 路由未命中、资源不存在、任务不在运行、插件未启用、浏览器会话不存在 |
| 409 | 任务 id 冲突、审批工单已裁决、插件部署清单被并发改动 |
| 500 | 存储读写失败、会话磁盘目录删除失败等服务端内部错误 |
| 503 | 依赖未装配（session store / message store / 事件总线 / 浏览器运行时 / SkillManager / 审批存储缺失） |

`POST /v1/sessions` 和 `PATCH /v1/sessions/{id}` 使用 `DisallowUnknownFields`：传了服务端不认识的字段会 400，而不是被静默丢弃（典型如客户端自带 `id`——会话 id 恒由服务端生成）。

`PATCH /v1/sessions/{id}` 的 `working_dir` 是**一次性可设**语义：从空设成某目录允许，重复设同值允许，改成另一个目录直接拒绝——会话磁盘状态按写入时的 working_dir 归档，改指向会遗弃已有状态。

<!-- @end-section -->

## 相关文档

- [[reference-legion-agent-auth-001|Legion Agent 鉴权参考]] — token、握手、RBAC、工具授权
- [[reference-legion-agent-integration-001|Legion Agent 接入参考]] — GUI / 第三方 / SSE 客户端怎么接
- [[reference-legion-gateway-001|Legion Gateway IM 网关参考]] — IM 平台接入
- [[reference-legion-agent-http-service-001|HTTP 服务]] — 面向使用者的 curl 速查
- [[reference-legion-agent-session-001|会话连续性]] — session/turn 语义
- [[agent-http-api-001|Legion Agent HTTP API]] — legion-agent 目录下的 API 说明
