---
id: "agent-storage-ops-001"
title: "Legion Agent SQLite 运维"
type: "guide"
category: "backend/agent"
tags: ["agent", "sqlite", "storage", "ops"]
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

# Legion Agent SQLite 运维

Legion Agent 在 `storage.driver=sqlite` 时，会把 task、runtime event、audit、workflow waiting state、skill、memory 和 evolution 数据写入 `storage.path`。

## Schema Version

SQLite 初始化会创建 `schema_migrations` 表，并记录当前 schema 版本。

当前版本：

```text
1
```

重复启动或重复迁移是幂等的，不会重复写入相同版本。

## 备份

备份前建议停止正在运行的 `agent serve`，避免复制进行中的写入。

```powershell
go run ./cmd -- backup --config .\configs\local.json --out .\backups\agent.db.bak
```

输出示例：

```text
backup=.\backups\agent.db.bak checksum=<sha256>
```

备份会生成两个文件：

| 文件 | 说明 |
|------|------|
| `agent.db.bak` | SQLite 数据库备份 |
| `agent.db.bak.sha256` | 备份文件 SHA-256 校验和 |

## 恢复

恢复前必须停止 `agent serve`。恢复命令会先校验 `.sha256`，再为当前目标库创建 `.pre-restore-*` 保护备份，最后覆盖目标库。

```powershell
go run ./cmd -- restore --config .\configs\local.json --in .\backups\agent.db.bak
```

输出示例：

```text
restored=data/agent.db pre_restore=data/agent.db.pre-restore-20260514123000 checksum=<sha256>
```

## 失败处理

| 失败 | 行为 |
|------|------|
| checksum 不匹配 | restore 直接失败，不覆盖目标库 |
| 备份文件不存在 | restore 失败，不覆盖目标库 |
| 目标库不存在 | restore 直接从备份创建目标库 |
| 目标库存在 | restore 前创建 `.pre-restore-*` 保护备份 |

## 验证

恢复后运行：

```powershell
go test ./internal/storage -run TestSQLiteRepositoryPing
go run ./cmd -- serve --config .\configs\local.json --addr :8080
```

然后检查：

```powershell
curl http://127.0.0.1:8080/readyz
curl -H "Authorization: Bearer <token>" http://127.0.0.1:8080/debug/diagnostics
```
