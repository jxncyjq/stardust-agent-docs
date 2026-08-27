---
id: "reference-legion-agent-auth-001"
title: "Legion Agent 鉴权与授权参考"
aliases: ["鉴权", "admin token", "RBAC", "loopback handshake", "工具授权"]
type: "reference"
category: "agents/reference"
tags: ["agent", "auth", "rbac", "security", "token", "approval"]
version: "1.0.0"
created: "2026-08-27"
updated: "2026-08-27"
author: "jxncyjq"
status: "published"
parent: "reference-legion-agent-user-manual-001"
children: []
related_docs:
  - id: "reference-legion-agent-backend-api-001"
    relation: "related_to"
    path: "./reference-legion-agent-backend-api-001.md"
  - id: "reference-legion-agent-integration-001"
    relation: "related_to"
    path: "./reference-legion-agent-integration-001.md"
  - id: "agent-security-tenancy-001"
    relation: "related_to"
    path: "../legion-agent/security-tenancy.md"
  - id: "agent-governance-rbac-001"
    relation: "related_to"
    path: "../legion-agent/governance-rbac.md"
---

# Legion Agent 鉴权与授权参考

Legion Agent 的鉴权分**五层**，各管一段，互不替代：

| 层 | 管什么 | 代码位置 |
|----|--------|----------|
| 1. 传输鉴权 | 谁能调这个 HTTP 端口 | `internal/server/http.go: authorized` |
| 2. Loopback 加固 | 本机 App/GUI 自连的握手与 Origin 守卫 | `internal/server/loopback_auth.go` |
| 3. 身份与租户 | 请求代表哪个公司、哪个角色 | `internal/security` |
| 4. 工具授权 | 这个 Agent 能调哪些工具 | `internal/tool/permission.go`、`internal/toolauth` |
| 5. 人工闸门 | 有副作用的动作要不要人批 | `internal/manualgate`、`internal/approval`、插件同意流 |

<!-- @section: transport-auth -->
## 一、传输鉴权：admin token

```json
{
  "server": {
    "listen_addr": ":8080",
    "admin_token": "change-me",
    "public_health_enabled": true
  }
}
```

规则（`HTTPServer.authorized`）：

- `admin_token` 为空 → **所有端点放行**。这是单机默认形态；把这样的进程绑到 `0.0.0.0` 等于开放全部管理面。
- `admin_token` 非空 → 除下列豁免外，一律要求 `Authorization: Bearer <admin_token>`，否则 401：
  - `GET /openapi.json` 恒公开（外部生成客户端用）；
  - `GET /healthz`、`GET /readyz` 在 `public_health_enabled=true` 时公开。
- token 比较是整串相等，没有多 token、没有过期、没有吊销接口。轮换 = 改配置 + 重启。

环境变量覆盖：`LEGION_AGENT_ADMIN_TOKEN`、`LEGION_AGENT_PUBLIC_HEALTH`。

```powershell
curl -H "Authorization: Bearer change-me" http://127.0.0.1:8080/metrics
```

<!-- @end-section -->

<!-- @section: loopback -->
## 二、Loopback 加固（App / GUI 模式）

GUI（Wails）与后端同机通信，没有预共享密钥，所以走**每次启动新铸一次性 token + 握手文件**的路子。

触发条件（`BuildServeService`）：

```text
loopbackHardening = server.loopback_hardening
                 || (未配置 listen_addr 且默认绑到 127.0.0.1:0)
```

即：显式打开开关，**或** GUI 模式下没配监听地址（默认随机 loopback 端口）。注意 `serve --addr 127.0.0.1:9000` 这类显式 loopback 绑定**不会**自动加固——要加固就显式配 `loopback_hardening: true`。

加固打开后：

1. `admin_token` 为空时，用 `crypto/rand` 生成 32 字节、hex 编码的 64 字符一次性 token（每次启动都换）。显式配了 `admin_token` 则一律沿用配置值。
2. 监听绑定拿到真实端口后，把 `{"baseURL":"http://127.0.0.1:<port>","token":"<token>"}` 写进握手文件，权限 `0600`（同机其他用户读不到）。
   - 路径取 `server.handshake_file`；为空则用平台 AppData 目录下的 `handshake.json`。
   - 写失败**不致命**：服务照跑，只是前端自动连不上，日志记 Warn。
3. 整个 mux 外面套 `LoopbackOriginGuard`：
   - 无 `Origin` 头 → 放行（webview 同源 fetch/EventSource、服务器到服务器客户端通常不带）；
   - `Origin` 等于本服务 baseURL → 放行；
   - `Origin` 存在但不匹配 → **403 forbidden origin**。
   - 守卫在最外层，SSE 端点（`/v1/events`、`/v1/browser/sessions/{id}/stream`）同样被覆盖。

Origin 守卫是纵深防御，**Bearer token 才是主鉴权**。

> 排障经验：`handshake.json` 是 loopback 自连凭据，与「浏览器功能是否开启」无关，别拿它当浏览器能力的证据。

<!-- @end-section -->

<!-- @section: identity -->
## 三、身份、租户与 RBAC

身份来自三个**调用方自述**的请求头（`security.PrincipalFromRequest`）：

| 请求头 | 含义 |
|--------|------|
| `X-Company-ID` | 租户/公司 ID |
| `X-Subject-ID` | 调用主体 ID，用于审计 |
| `X-Role` | 角色：`admin` / `operator` / `viewer` |

### 两种部署契约：`server.require_identity`

| 取值 | 语义 |
|------|------|
| `false`（默认） | 缺身份是**契约声明的合法状态**：空 `X-Role` 视为 `admin`，空 `X-Company-ID` 匹配所有公司。单机 / loopback 形态。装配时服务端会主动打一条日志声明这一点，不是静默兜底。 |
| `true` | 缺身份即拒：空角色落到 deny 分支，空 company 一律 `CanAccessCompany=false`，返回 403 + 审计事件。多租户 / 对网暴露必开。 |

环境变量 `LEGION_AGENT_REQUIRE_IDENTITY` 只接受 `strconv.ParseBool` 能解析的值，其他值直接让配置加载失败。

**关键边界**：`require_identity` 不是通用认证开关。它只覆盖会查 Policy 的端点（audit / quality / plugins / 公司隔离资源），不覆盖 `/v1/tasks` 列表这类未做租户判定的端点；而且 Role/CompanyID 是调用方自述的头——只有在**网关会注入这些头并剥掉客户端自带值**时才构成真正的边界。

### 角色能力矩阵

| Action | admin | operator | viewer |
|--------|-------|----------|--------|
| `read_audit` | ✅ | ✅ | ❌ |
| `read_quality` | ✅ | ✅ | ✅ |
| `read_task` | ✅ | ✅ | ✅ |
| `read_workflow` | ✅ | ✅ | ✅ |
| `read_plugin` | ✅ | ✅ | ❌ |
| `write_plugin`（grant/deny） | ✅ | ❌ | ❌ |

未识别的角色一律拒绝。`write_plugin` 只给 admin 是**刻意的保守默认**（改的是「什么代码能带什么能力跑」），不是漏配。

### 租户隔离

`requireCompanyAccess` 在任务查询、Agent 消息等端点上比对 `X-Company-ID` 与资源归属，拒绝时：

1. 返回 `403 company access denied`；
2. 写审计事件 `access_denied.cross_company`，资源标识以 sha256 摘要形式记录（不落原文）；
3. 审计写失败会记 Error 日志——安全取证链断了必须看得见。

<!-- @end-section -->

<!-- @section: tool-auth -->
## 四、工具授权

三道独立的闸，全部要过：

### 4.1 角色允许表（`BatchRolePermissionEnforcer`）

工具注册时带一张 `role:tool` 允许表，当前生产工具全部挂在 `developer:*` 能力域下。非 `developer` 角色会回退查 `developer:<tool>` 的许可；显式 override 优先于表。

**加固教训**：新增工具（例如 `browser_*`）必须同步登记进这张表，否则运行期一律 `permission denied`——enforcer 跑在策略判定之前，配置层怎么开都没用。

### 4.2 per-agent 禁用清单（`runtime.disabled_tools`）

```json
{
  "runtime": {
    "disabled_tools": ["write_file", "browser_open"]
  }
}
```

- 是**deny-list**：不写 / 写 null / 写空数组都表示「一个都不禁」。
- 每个名字必须是 `toolauth` 目录里登记的 gateable 工具，装配期校验，写错名字直接失败。
- 元工具 `call_tool`、`load_capabilities` 永远常驻，不可禁用、不在清单里。
- 当前 gateable 集合（`internal/toolauth/catalog.go`）：文件类 `read_file` / `write_file` / `list_files` / `search_content`，任务台账 `create_task` / `claim_task` / `update_task` / `append_task_message` / `read_task` / `rebuild_tasks`，消息 `send_message` / `read_messages`，网络 `fetch_url` / `web_search` / `web_extract`，编排者专属 `delegate_task` / `moa_consult` / `session_search`，浏览器 `browser_open` / `browser_read` / `browser_click` / `browser_type` / `browser_close`，外加运行期贡献的插件工具。
- 新增任何生产工具都必须登记到 gateable，drift-guard 测试会因为漏登记而失败。

### 4.3 执行管线（`Registry.Execute`）

一次工具调用按序过：查 handler → 按 `InputSchema` 校验参数 → 角色权限 → 策略判定（高风险且未自动允许则拒） → guardrails → 按工具超时建 ctx → 执行 → 结果 guardrails → 输出脱敏与控制字符清理 → 写审计。

工作区边界：文件工具的路径必须落在该任务的 ToolRoot 内（会话 `working_dir` 优先，否则回退配置根），越界路径被拒。

<!-- @end-section -->

<!-- @section: human-gate -->
## 五、人工闸门

### 5.1 工作模式

会话/任务的 `mode` 决定副作用怎么处理：

| 模式 | 行为 |
|------|------|
| `auto`（默认） | 不受限执行 |
| `plan` | 只提供只读工具，产出计划，无副作用 |
| `manual` | 标记为 `Sensitive` 的工具被挡在人工审批之后 |

标记为 `Sensitive` 的工具：`write_file`、`fetch_url`、`web_search`、`web_extract`、`delegate_task`、`moa_consult`、`browser_open` / `browser_click` / `browser_type`。

Manual 模式下如果审批依赖（toolGate / checkpoints）没装配，运行时直接失败——不会「无人可批就当批了」。

### 5.2 审批闭环

```text
模型请求 Sensitive 工具
  -> 开审批工单（audit: approval_opened）
  -> SSE 推 pending 事件 / GET /v1/approvals 拉取未决工单
  -> POST /v1/tasks/{taskID}/approvals/{ticketID}  body {"decision":"approve"|"deny"}
  -> 批准则继续工具循环，拒绝则把 reject 结果回给模型
```

- 工单超过 `runtime.approval_timeout_seconds`（默认 300 秒）由后台扫描自动拒绝，这是**契约结果**（给模型一个 reject），不是静默丢弃。
- 重复裁决同一工单返回 409；工单不存在返回 404。
- `GET /v1/approvals` 的 `arguments` 已做脱敏与长度截断。

### 5.3 插件同意流

「安装 ≠ 授权」：注册进部署清单的插件，在被 `grant` 之前不能带能力运行。

- `GET /v1/plugins` 看声明能力与已授权能力；
- `POST /v1/plugins/{name}/grant`，body `{"capabilities":[],"allowed_hosts":[],"allowed_paths":[]}`；
- `POST /v1/plugins/{name}/deny` 撤销授权、保留注册。
- 服务端只做错误分类不做二次判断：未知插件 404、清单被并发改动 409、磁盘写失败 500、加载器不可用 503、其余 400。
- CLI 侧对应 `agent plugins install/grant/deny/status/reload`，以及 `keygen` / `sign`（Ed25519 包签名）。

<!-- @end-section -->

<!-- @section: checklist -->
## 部署自检清单

对外暴露（非纯本机）时逐项确认：

1. `server.admin_token` 已配且不是示例值；密钥不进可提交的配置文件。
2. `server.require_identity=true`，并且前置网关会注入 `X-Role` / `X-Company-ID` 且**剥掉客户端自带的同名头**。
3. `server.public_health_enabled` 按需——不需要匿名探活就关掉。
4. 只对本机 GUI 服务时，用 `loopback_hardening` + 握手文件，不要手配长期 token。
5. `runtime.disabled_tools` 按角色最小化；对外角色不要留 `write_file`、`browser_*`、`delegate_task`。
6. 默认工作模式对高风险场景设 `manual`，确认审批端点有人/有 UI 消费。
7. 插件部署确认每个插件的 grant 范围（能力 / host / path）是最小集。
8. `/v1/audit-events` 有人定期看，尤其 `access_denied.cross_company`。

<!-- @end-section -->

## 相关文档

- [[reference-legion-agent-backend-api-001|后端系统调用参考]] — 端点全表与错误码
- [[reference-legion-agent-integration-001|接入参考]] — 客户端如何携带凭据
- [[reference-legion-gateway-001|Legion Gateway IM 网关参考]] — 网关侧凭据与身份注入
- [[reference-legion-agent-tools-001|工具能力]] — 工具清单与执行管线
- [[agent-security-tenancy-001|安全与多租户]]
- [[agent-governance-rbac-001|Governance RBAC]]
