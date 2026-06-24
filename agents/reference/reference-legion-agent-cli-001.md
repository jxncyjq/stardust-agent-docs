---
id: "reference-legion-agent-cli-001"
title: "Legion Agent CLI 命令速查"
aliases: ["CLI 命令", "agent run", "agent serve"]
type: "reference"
category: "agents/reference"
tags: ["agent", "cli", "commands", "backup", "data"]
version: "1.1.0"
created: "2026-05-19"
updated: "2026-05-25"
author: "jxncyjq"
status: "published"
parent: "reference-legion-agent-user-manual-001"
children: []
---

# Legion Agent CLI 命令速查

源码运行时，所有命令前都加 `go run ./cmd --`。如果你已经构建出 `agent.exe` 或把二进制加入 PATH，可以把示例里的 `go run ./cmd --` 替换成 `agent`。

## 命令表

| 命令 | 用途 |
|------|------|
| `agent run --demo --plain` | 运行 demo 任务 |
| `agent run --plain --config agent.json --prompt "..."` | 执行一次真实任务 |
| `agent tui --config agent.json` | 启动交互式 TUI |
| `agent serve --config agent.json --addr :8080` | 启动 HTTP 服务 |
| `agent backup --config agent.json --out agent.db.bak` | 备份 SQLite 数据库 |
| `agent restore --config agent.json --in agent.db.bak` | 从备份恢复 SQLite 数据库 |
| `agent data retention --config agent.json --quality-days 30` | 预览数据保留清理 |
| `agent data retention --config agent.json --quality-days 30 --apply` | 执行数据保留清理 |
| `agent data export --config agent.json --out agent-export.json` | 导出审计和事件快照 |
| `agent skill sync --config agent.json` | 从 skill registry 同步技能 |
| `agent version --plain` | 输出版本信息 |
| `agent doctor` | 检查本地 Agent 设置 |

## 源码运行模板

```powershell
go run ./cmd --
```

例如：

```powershell
go run ./cmd -- run --plain --config .\agent.json --prompt "总结当前实现"
```

## run：单次任务

适合脚本、CI、一次性问答。`run` 执行完成后立即退出，不进入交互界面。

```powershell
go run ./cmd -- run --plain --config .\agent.json --prompt "总结当前 Agent 的模块"
```

临时切换模型 profile：

```powershell
go run ./cmd -- run --plain --config .\agent.json --maas-profile review --prompt "审查当前变更风险"
```

临时关闭上下文文件：

```powershell
go run ./cmd -- run --plain --config .\agent.json --no-context-files --prompt "只回答 hello"
```

运行 demo，不调用真实模型：

```powershell
go run ./cmd -- run --demo --plain
```

## tui：交互式工作台

适合日常使用。TUI 会保留界面、支持多轮 session、slash 命令、`@agent` 选择、鼠标滚动输出区。

```powershell
go run ./cmd -- tui --config .\agent.json
```

临时覆盖 MaaS 地址和 key：

```powershell
go run ./cmd -- tui --config .\agent.json --maas-url https://api.example.com --maas-api-key sk-...
```

## serve：HTTP 服务

适合外部系统集成、workflow、多 Agent 编排、查询 session/message/task。

```powershell
go run ./cmd -- serve --config .\agent.json --addr :8080
```

启动后验证：

```powershell
curl http://127.0.0.1:8080/healthz
curl http://127.0.0.1:8080/readyz
```

## backup / restore：数据库备份恢复

只支持 SQLite 存储。备份前确认 `storage.driver=sqlite`。

```powershell
go run ./cmd -- backup --config .\agent.json --out .\backups\agent.db.bak
go run ./cmd -- restore --config .\agent.json --in .\backups\agent.db.bak
```

restore 会保留一份恢复前数据库副本，避免误覆盖后无法回退。

## data：导出与保留清理

导出审计和事件快照：

```powershell
go run ./cmd -- data export --config .\agent.json --out .\agent-export.json
```

预览数据保留清理：

```powershell
go run ./cmd -- data retention --config .\agent.json --audit-days 30 --runtime-days 30 --quality-days 30
```

确认后执行：

```powershell
go run ./cmd -- data retention --config .\agent.json --audit-days 30 --runtime-days 30 --quality-days 30 --apply
```

## skill：同步技能

从配置的 registry 同步：

```powershell
go run ./cmd -- skill sync --config .\agent.json
```

同步指定子 Agent 的技能目录：

```powershell
go run ./cmd -- skill sync --config .\agent.json --agent writer
```

`--agent writer` 会读取 `agent.json` 中注册的 writer 配置，并优先使用 writer 的 `skills.registry_url` 和 `skills.install_root`。如果子 Agent 未配置这些字段，则回退到根配置。

临时指定 registry 和安装目录：

```powershell
go run ./cmd -- skill sync --registry-url https://example.com/skills/index.json --install-root .\skills
```

## 常用覆盖参数

| 参数 | 作用 |
|------|------|
| `--config` | 指定配置文件 |
| `--maas-profile` | 选择 `maas.profiles` 中的模型 profile |
| `--maas-url` | 临时覆盖 MaaS base URL，并绕过 profile |
| `--maas-api-key` | 临时覆盖 MaaS API key |
| `--no-context-files` | 本次运行不加载上下文文件 |
| `--plain` | 输出机器可读文本，适合脚本和 CI |
