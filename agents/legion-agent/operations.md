---
id: "agent-operations-001"
title: "Legion Agent 运维 Runbook"
type: "guide"
category: "backend/agent"
tags: ["agent", "operations", "runbook"]
version: "1.0.0"
created: "2026-06-24"
updated: "2026-06-24"
author: "jxncyjq"
status: "published"
parent: null
children: []
related_docs:
  - id: "agent-index-001"
    relation: "related_to"
    path: "./index.md"
---

# Legion Agent 运维 Runbook

本文档用于 Legion Agent 独立服务的本地部署、检查、排障、备份、恢复和回滚。

## 本地启动

准备配置文件：

```json
{
  "storage": {
    "driver": "sqlite",
    "path": "data/agent.db"
  },
  "server": {
    "listen_addr": ":8080",
    "admin_token": "change-me",
    "public_health_enabled": true,
    "request_id_header": "X-Request-ID"
  },
  "service": {
    "background_interval": "1s"
  }
}
```

启动服务：

```powershell
go run ./cmd -- serve --config .\configs\local.json --addr :8080
```

生产环境必须设置 `server.admin_token`，并建议把配置文件权限限制在服务账号可读范围内。

## 健康检查

`/healthz` 只表示进程和 HTTP handler 可响应。

```powershell
curl http://127.0.0.1:8080/healthz
```

响应：

```json
{"status":"ok"}
```

如果 `server.public_health_enabled=false`，需要携带管理 token。

## Readiness 检查

`/readyz` 检查依赖可用性。当前会检查 storage；SQLite 模式会执行轻量 ping。

```powershell
curl http://127.0.0.1:8080/readyz
```

正常：

```json
{
  "status": "ok",
  "checks": {
    "storage": "ok"
  }
}
```

异常时返回 `503`：

```json
{
  "status": "unavailable",
  "reason": "storage_unavailable",
  "checks": {
    "storage": "unavailable"
  }
}
```

## Metrics

`/metrics` 返回当前进程内存指标快照，属于管理接口。

```powershell
curl -H "Authorization: Bearer change-me" http://127.0.0.1:8080/metrics
```

重点字段：

| 字段 | 说明 |
|------|------|
| `tasks` | task submitted/running/done/failed 计数 |
| `http_status` | HTTP 状态码计数 |
| `model_calls` | 模型调用 success/failed 计数 |
| `approvals` | 审批计数 |
| `workflow_runs` | workflow 状态计数 |

指标是单进程内存快照，进程重启后重新计数。

## Diagnostics

`/debug/diagnostics` 返回脱敏诊断快照，属于管理接口。

```powershell
curl -H "Authorization: Bearer change-me" http://127.0.0.1:8080/debug/diagnostics
```

诊断内容包含版本、uptime、配置轮廓、scheduler 状态和 metrics。不会输出 MaaS API key、admin token、完整 prompt 或完整 storage path；敏感字段显示为 `[redacted]`，storage path 只输出 hash。

## 日志字段

Legion Agent 使用 JSON 结构化日志。关键字段：

| 字段 | 说明 |
|------|------|
| `time` | 日志时间 |
| `level` | `INFO`、`WARN`、`ERROR` |
| `msg` | 静态日志消息 |
| `component` | `app`、`server`、`service` 等组件 |
| `request_id` | HTTP 请求追踪 ID |
| `task_id` | task 追踪 ID |
| `method` / `path` / `status` | HTTP 请求摘要 |

日志不应包含 admin token、MaaS API key 或完整 prompt。

## SQLite 备份

备份前建议停止 `agent serve`。

```powershell
go run ./cmd -- backup --config .\configs\local.json --out .\backups\agent.db.bak
```

备份会生成：

| 文件 | 说明 |
|------|------|
| `agent.db.bak` | SQLite 数据库备份 |
| `agent.db.bak.sha256` | SHA-256 校验和 |

## SQLite 恢复

恢复前必须停止 `agent serve`。

```powershell
go run ./cmd -- restore --config .\configs\local.json --in .\backups\agent.db.bak
```

恢复流程：

1. 读取 `.sha256`。
2. 校验备份文件 checksum。
3. 为当前目标库创建 `.pre-restore-*` 保护备份。
4. 覆盖目标库。
5. 启动服务并检查 `/readyz`。

checksum 不匹配时 restore 会失败，且不会覆盖目标库。

## 发布产物选择

本地发布构建：

```powershell
.\scripts\release.ps1 -Version 0.1.0 -Commit <git-sha> -OutDir .\dist
```

选择规则：

| 产物 | 使用场景 |
|------|----------|
| `legion-agent-windows-amd64.exe` | Windows x64 |
| `legion-agent-linux-amd64` | Linux x64 |
| `legion-agent-linux-arm64` | Linux ARM64 |

检查版本：

```powershell
.\dist\legion-agent-windows-amd64.exe version --plain
```

## 回滚步骤

1. 停止当前 `agent serve`。
2. 备份当前 SQLite 数据库。
3. 替换为上一版二进制。
4. 执行 `version --plain` 确认版本。
5. 启动服务。
6. 检查 `/healthz`、`/readyz`、`/metrics`。
7. 如数据异常，使用 `restore` 回滚数据库备份。

## 总验收命令

```powershell
go test ./...
go vet ./...
go build -o NUL ./cmd
.\scripts\smoke.ps1
.\scripts\release.ps1 -Version 0.1.0-local -Commit local-test -OutDir .\dist
```
