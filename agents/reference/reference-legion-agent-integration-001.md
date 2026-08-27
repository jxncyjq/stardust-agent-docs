---
id: "reference-legion-agent-integration-001"
title: "Legion Agent 接入参考"
aliases: ["接入", "客户端接入", "GUI 接入", "第三方接入", "integration"]
type: "reference"
category: "agents/reference"
tags: ["agent", "integration", "gui", "sse", "client", "backend"]
version: "1.0.0"
created: "2026-08-27"
updated: "2026-08-27"
author: "jxncyjq"
status: "published"
parent: "reference-legion-agent-user-manual-001"
children: []
related_docs:
  - id: "reference-legion-agent-backend-api-001"
    relation: "depends_on"
    path: "./reference-legion-agent-backend-api-001.md"
  - id: "reference-legion-agent-auth-001"
    relation: "depends_on"
    path: "./reference-legion-agent-auth-001.md"
  - id: "reference-legion-gateway-001"
    relation: "related_to"
    path: "./reference-legion-gateway-001.md"
---

# Legion Agent 接入参考

本文讲**怎么把 legionAgent 接进别的东西**：三种接入形态、各自的凭据获取方式、最小调用序列、以及真机踩过的坑。端点细节见 [[reference-legion-agent-backend-api-001|后端系统调用参考]]，凭据规则见 [[reference-legion-agent-auth-001|鉴权参考]]。

<!-- @section: shapes -->
## 三种接入形态

| 形态 | 谁在用 | 进程关系 | 凭据来源 |
|------|--------|----------|----------|
| 内嵌 App | legionAgentGUI（Wails 桌面端） | serve 跑在 GUI 进程内 | 进程内直接拿 `ServeResult.Token` |
| 本机自连 | 独立前端 / 本机脚本连本机 serve | 两个进程，同一台机器 | 握手文件 `{baseURL, token}` |
| 远端服务 | 第三方后端、CI、IM 网关 | 跨机 | 预共享 `server.admin_token`（+ 网关注入身份头） |

<!-- @end-section -->

<!-- @section: embedded -->
## 形态一：内嵌 App（GUI）

`legionAgentGUI` 不是「起个子进程再连」，而是**在自己进程里装配 serve**：调 `cli.BuildServeService`，拿回 `ServeResult{Service, Listener, Token, BaseURL, Close}`。

要点：

1. **端口与 token 都会变**。未配 `listen_addr` 时绑 `127.0.0.1:0`（随机端口）；保存配置触发 `ServeManager.Restart` 会重新绑定端口**并重铸 token**。所以任何长连接/请求都必须**每次读取**当前 baseURL 和 token，不能在启动时捕获一次。
2. **SSE 走 Go 侧桥接**。WebView2 里前端直接消费长连 SSE 不可靠，GUI 的做法是 Go 侧 `StartSSEBridge` 保持连接，再用 Wails `EventsEmit` 把事件转发给 React。断线按固定间隔重连（重连前重新读 baseURL/token）。
3. **写操作走 Go binding，不要前端裸 fetch**。跨域预检 `OPTIONS` 在本服务上没有路由，会 404 导致请求失败；GUI 侧把 takeover 这类 POST 封成 Go 方法暴露给前端。
4. **加固改动必须回归所有 in-process 消费者**。历史事故：只改后端 token 校验、没同步 GUI 侧取 token 的路径，合进 master 当天 GUI 全线 403。

<!-- @end-section -->

<!-- @section: loopback-client -->
## 形态二：本机自连（握手文件）

适用于独立前端或本机脚本连一个已经在跑的 `agent serve`。

前提：serve 端开了 loopback 加固（`server.loopback_hardening=true`，或 GUI 默认的随机 loopback 绑定）。

接入步骤：

1. 读握手文件（`server.handshake_file`，缺省是平台 AppData 目录下的 `handshake.json`，权限 0600）：

   ```json
   {"baseURL":"http://127.0.0.1:53124","token":"a3f1…（64 hex）"}
   ```

2. 所有请求带 `Authorization: Bearer <token>`。
3. 浏览器类客户端注意 Origin 守卫：`Origin` 缺失或等于 `baseURL` 才放行，其他值 403。
4. token **每次 serve 启动都换**：进程重启后必须重读握手文件，不要缓存到磁盘。

握手文件写失败时 serve 仍会启动（只记 Warn），此时客户端读不到文件 = 服务在跑但自连不了，去日志找 `write loopback handshake`。

<!-- @end-section -->

<!-- @section: remote -->
## 形态三：远端服务接入

```powershell
go run ./cmd/agent -- serve --config .\agent.json --addr :8080
```

必配项：

```json
{
  "server": {
    "listen_addr": ":8080",
    "admin_token": "<强随机值，来自 secret 管理>",
    "require_identity": true,
    "public_health_enabled": false,
    "file_base_url": "https://agent.example.com"
  },
  "storage": { "driver": "sqlite", "path": "agent.db" }
}
```

- `require_identity=true` 后，调用方必须带 `X-Role` 与 `X-Company-ID`；这两个头是自述值，**必须由前置网关注入并剥掉客户端自带的同名头**才有边界意义。
- `file_base_url` 决定 `generated_files` 里返回绝对 URL 还是相对路径。跨机接入必须配，否则客户端拿到 `/v1/files?...` 需要自己拼域名。
- 只用 SQLite 驱动才有持久化（task / event / audit / session / message）；`memory` 驱动重启即失忆。

<!-- @end-section -->

<!-- @section: minimal-flow -->
## 最小接入序列

一个「建会话 → 提任务 → 等结果 → 取文件」的完整闭环：

```powershell
# 1. 建会话（可选，但要多轮上下文就必须建）
curl -X POST "http://127.0.0.1:8080/v1/sessions" `
  -H "Authorization: Bearer change-me" -H "Content-Type: application/json" `
  -d "{\"company_id\":\"company-1\",\"agent_id\":\"writer\",\"project\":\"docs\",\"title\":\"接入验证\",\"mode\":\"auto\",\"working_dir\":\"F:/work/demo\"}"
# -> 201 {"id":"session-1770000000000000000", ...}

# 2. 提任务（id 由调用方铸造，冲突 409）
curl -X POST "http://127.0.0.1:8080/v1/tasks" `
  -H "Authorization: Bearer change-me" -H "Content-Type: application/json" `
  -d "{\"id\":\"task-1\",\"company_id\":\"company-1\",\"agent_id\":\"writer\",\"session_id\":\"session-1770000000000000000\",\"input\":\"写一份接入说明到 docs/intro.md\"}"

# 3. 等结果：订阅 SSE（推荐）或轮询 result
curl -N -H "Authorization: Bearer change-me" "http://127.0.0.1:8080/v1/events?type=task_completed"
curl -H "Authorization: Bearer change-me" "http://127.0.0.1:8080/v1/tasks/task-1/result"

# 4. 取生成文件（URL 来自 result.generated_files）
curl -H "Authorization: Bearer change-me" "http://127.0.0.1:8080/v1/files?session_id=session-1770000000000000000&path=docs%2Fintro.md&download=1" -o intro.md
```

两种等结果方式的取舍：

| 方式 | 适合 | 注意 |
|------|------|------|
| SSE `/v1/events` | 实时 UI、需要中间过程（工具调用、推理完成） | 尽力而为，可能丢事件；断线要能重连并回补状态 |
| 轮询 `/v1/tasks/{id}/result` | 脚本、网关、批处理 | 只在终态（done/failed/suspended）才有答案；未知状态要当错误处理，别当「还没完成」无限重试 |

<!-- @end-section -->

<!-- @section: pitfalls -->
## 接入常见坑

| 现象 | 原因 | 处理 |
|------|------|------|
| 401 unauthorized | 配了 `admin_token` 但没带 Bearer；或 GUI 重启后 token 换了 | 每次请求现读 token |
| 403 forbidden origin | 浏览器客户端 `Origin` 与 serve baseURL 不一致 | 同源加载，或从服务端侧调用 |
| 403 company access denied | `require_identity=true` 却没注入 `X-Company-ID` | 网关注入身份头 |
| 预检 OPTIONS 404 | 服务端无 CORS 路由 | 走宿主进程（Go binding / 后端代理）发请求，别在浏览器里跨域直连 |
| `generated_files` 为空 | `write_file` 默认 `overwrite=false`，目标文件已存在时写入失败 | 验证时换新文件名；需要覆盖就显式传 `overwrite=true` |
| 文件 404 / 403 | `session_id` 没有 `working_dir`；或 `path` 越出工作目录 | 建会话时设 `working_dir`；路径只能是工作目录内相对路径 |
| 任务卡在 pending | 并发已满（`runtime.max_concurrent_tasks` 默认 4）或调度未启动 | 看 `/debug/diagnostics` 的 scheduler 状态 |
| SSE 连上无输出 | 总线空闲，非故障（连接建立即已刷 200 头） | 用 `/v1/runtime-events` 拉最近事件对照 |
| 中断返回 404 | 任务已结束或从未运行 | 这是契约：不在运行绝不报成功 |
| 插件端点 404 | 该进程没配 `plugins.manifest` | 与「没有插件」区分开 |

<!-- @end-section -->

<!-- @section: checklist -->
## 接入自检

1. 凭据是否**每次现读**（token 会随重启轮换）。
2. 是否处理了 401/403/404/409/503 五类响应，而不是只看 200。
3. 长任务是否有中断入口（`POST /v1/tasks/{id}/interrupt`）。
4. Manual 模式下是否有消费 `/v1/approvals` 并回裁的通道，否则任务会挂到审批超时（默认 300 秒）后被拒。
5. 文件链接是否用响应里的 `url` / `download_url`，而不是自己缓存拼接。
6. 跨机部署是否配了 `file_base_url`、`require_identity`，是否关掉了匿名健康检查。

<!-- @end-section -->

## 相关文档

- [[reference-legion-agent-backend-api-001|后端系统调用参考]] — 端点、请求体、错误码
- [[reference-legion-agent-auth-001|鉴权与授权参考]] — token / 握手 / RBAC / 工具授权
- [[reference-legion-gateway-001|Legion Gateway IM 网关参考]] — IM 平台如何接入
- [[reference-legion-agent-http-service-001|HTTP 服务]] — curl 速查
- [[reference-legion-agent-cli-001|CLI 命令速查]] — serve 启动参数
