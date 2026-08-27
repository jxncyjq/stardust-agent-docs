---
id: "reference-legion-gateway-001"
title: "Legion Gateway IM 网关参考"
aliases: ["legion-gateway", "IM 网关", "Telegram 接入", "gateway"]
type: "reference"
category: "agents/reference"
tags: ["agent", "gateway", "im", "telegram", "integration", "backend"]
version: "1.0.0"
created: "2026-08-27"
updated: "2026-08-27"
author: "jxncyjq"
status: "published"
parent: "reference-legion-agent-user-manual-001"
children: []
related_docs:
  - id: "reference-legion-agent-integration-001"
    relation: "related_to"
    path: "./reference-legion-agent-integration-001.md"
  - id: "reference-legion-agent-backend-api-001"
    relation: "depends_on"
    path: "./reference-legion-agent-backend-api-001.md"
  - id: "reference-legion-agent-auth-001"
    relation: "depends_on"
    path: "./reference-legion-agent-auth-001.md"
---

# Legion Gateway IM 网关参考

`legion-gateway`（`cmd/gateway`）把 IM 平台桥到一个已在运行的 Legion core 上。它是**独立进程、纯出站**：靠长轮询拉消息、靠 HTTP 调 core，不需要公网回调地址，也不需要给 core 开新端口。

事实基准：`cmd/gateway/main.go`、`internal/gateway/*`、`configs/gateway.example.json`。

<!-- @section: architecture -->
## 结构

```text
IM 平台 ──long poll──> ChannelAdapter ──inbound chan──> GatewayRunner
                                                          │
                                        SessionBinder(SQLite) 解析/新建绑定
                                                          │
                                        CoreClient ──HTTP──> Legion core
                                          POST /v1/sessions
                                          POST /v1/tasks
                                          GET  /v1/tasks/{id}/result（2s 轮询）
                                                          │
                                        DeliveryTracker + DeliveryRouter
                                                          │
                                   ChannelAdapter.Send ──> IM 平台
```

四个组件各管一段：

| 组件 | 职责 |
|------|------|
| `PlatformRegistry` | 平台名 → 工厂 + 元数据（最大消息长度、是否 PII 安全）。加平台 = 一次 `Register`，不动核心逻辑；重名或缺工厂直接报错 |
| `ChannelAdapter` | 一个平台的接入实现：`Start` 自驱动入站循环、`Send` 出站、`Close` 释放 |
| `SessionBinder` | `"<platform>:<chatID>"` → Legion `session_id` + 原始 chat id，SQLite 持久化，重启不丢会话连续性 |
| `DeliveryTracker` / `DeliveryRouter` | 记录在途任务的投递目标；按平台上限切分长消息后发出。tracker 是**进程内**的，重启会丢在途条目（已知取舍） |

三条并发循环由 `GatewayRunner.Run` 拉起：每个 adapter 的入站循环、一个 inbound worker、一个投递轮询循环。任一循环致命错误会取消其余并返回。

<!-- @end-section -->

<!-- @section: privacy -->
## PII 规则（硬约束）

原始 chat / user id **只留在网关的绑定库里**，其余任何地方都用 `HashID`（sha256 前 16 hex）：

- 提交给 core 的 `task_id` 形如 `telegram-<hash>-<seq>`；
- 新建会话的 `title` 是 `telegram:<hash>`，`project` 是平台名；
- 所有日志字段用 hash 形式。

出站投递需要真实 chat id，所以它只从 binder 取，不进 core、不进日志。

<!-- @end-section -->

<!-- @section: config -->
## 配置

配置路径来自 `LEGION_GATEWAY_CONFIG` 或第一个命令行参数，二者都没有则启动失败。

```json
{
  "core":     { "base_url": "http://127.0.0.1:8080", "token_env": "LEGION_ADMIN_TOKEN" },
  "identity": { "agent_id": "im-agent", "company_id": "default-company" },
  "binding":  { "sqlite_path": "/data/gateway.db" },
  "delivery": { "retries": 3, "backoff_ms": 500 },
  "platforms": {
    "telegram": { "enabled": true, "token_env": "LEGION_TELEGRAM_BOT_TOKEN", "poll_timeout_s": 30 }
  }
}
```

| 字段 | 说明 |
|------|------|
| `core.base_url` | Legion core 地址，必填 |
| `core.token_env` | **环境变量名**，值是 core 的 admin token。配置文件里从不写明文 secret |
| `identity.agent_id` / `company_id` | 网关提交任务时用的身份，两者必填 |
| `binding.sqlite_path` | 绑定库路径（`channel_bindings` 表自动建） |
| `delivery.retries` | 投递最大尝试次数，缺省/非法时为 3 |
| `delivery.backoff_ms` | 重试间隔，缺省/非法时为 500ms |
| `platforms.<name>.enabled` | 关掉的平台不解析 token、不建 adapter |
| `platforms.<name>.token_env` | 该平台凭据的环境变量名 |
| `platforms.<name>.poll_timeout_s` | 长轮询超时；Telegram 缺省 30 秒 |

**fail-loud 点**：配置文件读不到、JSON 解析失败、`core.base_url` 或 identity 为空、`token_env` 为空、指向的环境变量未设或为空值——全部是启动期致命错误，不会用空 token 摸黑跑。启用了但未注册的平台同样直接报错。

启动：

```bash
LEGION_ADMIN_TOKEN=xxx LEGION_TELEGRAM_BOT_TOKEN=yyy legion-gateway /etc/legion/gateway.json
```

容器化参考 `deploy/docker-compose.gateway.yml`：镜像里跑 `legion-gateway /etc/legion/gateway.json`，配置只读挂载，绑定库放在具名卷上。

<!-- @end-section -->

<!-- @section: flow -->
## 消息流

### 入站

1. adapter 推 `InboundMessage{Platform, ChatID, UserID, Text, Images, ReceivedAt}`。
2. `HandleInbound` 用 `"<platform>:<chatID>"` 查绑定：
   - 命中 → 复用既有 `session_id`；
   - 未命中（首次对话，合法状态） → `POST /v1/sessions` 新建，再 `Bind` 落库。
3. 铸 `task_id`（平台 + hash + 进程内自增序号，永不碰撞），`POST /v1/tasks` 提交。
4. `tracker.Track(taskID, target)` 记下回投目标。

### 出站

`pollLoop` 每 **2 秒**跑一次 `PollOnce`，对每个在途任务：

- 还在退避窗口内 → 跳过；
- `GET /v1/tasks/{id}/result`：
  - `done` / `failed` / `suspended` → 终态；
  - `pending` / `running` / `assigned` → 未完成，下轮再来；
  - **其他任何状态 → 报错**（版本漂移或字段改名不会被当成「还没跑完」而无限静默重试）；
- 终态但答案为空（如 failed）→ 直接从 tracker 摘除，不投递；
- 有答案 → `DeliveryRouter.Route` 按平台上限切分后发送。失败则退避重试，用尽 `delivery.retries` 后摘除并记 **Error**（丢失必须留痕）。

消息切分优先在块内最后一个换行处断开，避免把回复截在半行；边界换行不会再输出成空块。

<!-- @end-section -->

<!-- @section: telegram -->
## Telegram 适配器

| 项 | 值 |
|----|-----|
| 入站 | `getUpdates` 长轮询，`timeout` 取 `poll_timeout_s`（缺省 30 秒），带 `offset` 递进 |
| 出站 | `sendMessage` |
| 最大消息长度 | 4096（注册进 registry，路由据此切分） |
| PII 安全标记 | true |
| 失败处理 | `getUpdates` 出错记 Warn 并退避重试，不终止进程 |

<!-- @end-section -->

<!-- @section: add-platform -->
## 新增一个平台

1. 在 `internal/gateway/platforms/<name>/` 实现 `ChannelAdapter`（`Platform` / `Start` / `Send` / `Close`）。
2. 在 `cmd/gateway/main.go` 里 `reg.Register(gateway.PlatformEntry{Platform: "<name>", Factory: <name>.New, MaxMessageLength: N, PIISafe: true})`。
3. 配置里加一节 `platforms.<name>`，含 `enabled` 与 `token_env`。
4. 遵守 PII 规则：任何交给 core 或写日志的标识都要过 `HashID`。

<!-- @end-section -->

<!-- @section: ops -->
## 运维注意

| 事项 | 说明 |
|------|------|
| 在途任务不持久 | tracker 在内存里，网关重启后未投递的答案不会补发；绑定关系在 SQLite 里，不受影响 |
| 绑定库单写 | 连接池上限为 1，`busy_timeout=5s`，多平台 goroutine 串行写而不是撞 SQLITE_BUSY |
| core 侧凭据 | 网关用的是 core 的 `admin_token`，权限等同管理员。core 若开了 `require_identity`，需要在网关与 core 之间的前置层注入身份头 |
| 一个网关 = 一个身份 | 所有 IM 任务都以 `identity.agent_id` / `company_id` 提交，多租户请起多个网关实例 |
| 无审批消费 | 网关不消费 `/v1/approvals`。若 core 的会话是 Manual 模式，任务会挂到审批超时后被拒——IM 场景建议保持 `auto` |

<!-- @end-section -->

## 相关文档

- [[reference-legion-agent-integration-001|接入参考]] — 其他接入形态
- [[reference-legion-agent-backend-api-001|后端系统调用参考]] — 网关调用的三个 core 端点
- [[reference-legion-agent-auth-001|鉴权与授权参考]] — core 侧 token 与身份头
- [[reference-legion-agent-http-service-001|HTTP 服务]] — curl 速查
