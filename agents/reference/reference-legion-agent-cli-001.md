---
id: "reference-legion-agent-cli-001"
title: "Legion Agent CLI 命令速查"
aliases: ["CLI 命令", "agent run", "agent serve", "agent plugins"]
type: "reference"
category: "agents/reference"
tags: ["agent", "cli", "commands", "backup", "data", "plugins"]
version: "2.0.0"
created: "2026-05-19"
updated: "2026-08-27"
author: "jxncyjq"
status: "published"
parent: "reference-legion-agent-user-manual-001"
children: []
related_docs:
  - id: "reference-legion-agent-http-service-001"
    relation: "related_to"
    path: "./reference-legion-agent-http-service-001.md"
  - id: "reference-legion-agent-auth-001"
    relation: "related_to"
    path: "./reference-legion-agent-auth-001.md"
  - id: "agent-storage-ops-001"
    relation: "related_to"
    path: "../legion-agent/storage-ops.md"
---

# Legion Agent CLI 命令速查

源码运行时命令前缀是 `go run ./cmd/agent --`（注意是 `./cmd/agent`，不是 `./cmd`）。构建出二进制后把前缀换成 `agent` 即可：

```powershell
go build ./cmd/agent
```

网关是另一个二进制：`./cmd/gateway`，见 [[reference-legion-gateway-001|Legion Gateway IM 网关参考]]。

<!-- @section: command-table -->
## 命令表

| 命令 | 用途 |
|------|------|
| `agent run --demo --plain` | 运行 demo 任务，不调真实模型 |
| `agent run --plain --config agent.json --prompt "..."` | 执行一次真实任务 |
| `agent tui --config agent.json` | 启动交互式 TUI |
| `agent serve --config agent.json --addr :8080` | 启动 HTTP 服务 |
| `agent plugins status` | 列出部署清单里每个插件及其结局 |
| `agent plugins reload` | 重读清单并让运行中的插件向它收敛 |
| `agent plugins install <url> --digest sha256:<hex>` | 拉取校验并注册远程插件包 |
| `agent plugins grant <name> --capabilities ...` | 授权已注册插件按声明能力运行 |
| `agent plugins deny <name>` | 撤销授权，保留注册 |
| `agent plugins keygen --key-id <id> --private-key <path>` | 生成 Ed25519 签名密钥对 |
| `agent plugins sign <package dir> --private-key <path>` | 给插件包的 `plugin.json` 签名 |
| `agent backup --config agent.json --out agent.db.bak` | 备份 SQLite 数据库 |
| `agent restore --config agent.json --in agent.db.bak` | 从备份恢复 |
| `agent data retention --config agent.json ...` | 预览/执行数据保留清理 |
| `agent data export --config agent.json --out snapshot.json` | 导出审计与事件快照 |
| `agent skill sync --config agent.json` | 从 registry 同步技能 |
| `agent version --plain` | 输出版本信息 |
| `agent doctor` | 检查本地 Agent 设置 |

<!-- @end-section -->

<!-- @section: run -->
## run：单次任务

适合脚本、CI、一次性问答，执行完立即退出。

```powershell
go run ./cmd/agent -- run --plain --config .\agent.json --prompt "总结当前 Agent 的模块"
go run ./cmd/agent -- run --plain --config .\agent.json --maas-profile review --prompt "审查当前变更风险"
go run ./cmd/agent -- run --plain --config .\agent.json --no-context-files --prompt "只回答 hello"
go run ./cmd/agent -- run --demo --plain
```

<!-- @end-section -->

<!-- @section: tui -->
## tui：交互式工作台

```powershell
go run ./cmd/agent -- tui --config .\agent.json
go run ./cmd/agent -- tui --config .\agent.json --maas-url https://api.example.com --maas-api-key sk-...
```

按键、slash 命令、`@agent` 见 [[reference-legion-agent-tui-001|TUI 使用]]。

<!-- @end-section -->

<!-- @section: serve -->
## serve：HTTP 服务

```powershell
go run ./cmd/agent -- serve --config .\agent.json --addr :8080
curl http://127.0.0.1:8080/healthz
```

只有两个 flag：`--config` 与 `--addr`。其余全部走配置文件或环境变量。

- 不传 `--addr` 且配置里没有 `server.listen_addr` → 绑 `127.0.0.1:0`（随机端口）并自动进入 loopback 加固：铸一次性 token、写握手文件、套 Origin 守卫。
- 显式 `--addr 127.0.0.1:9000` **不会**自动加固，需要就把 `server.loopback_hardening` 设为 true。

<!-- @end-section -->

<!-- @section: plugins -->
## plugins：WASM 插件

「安装 ≠ 授权」，两步是刻意分开的：

```powershell
# 注册（不授权）
go run ./cmd/agent -- plugins install https://example.com/p.tar.gz --digest sha256:<hex> --config .\agent.json

# 授权：--capabilities 必须与 plugin.json 声明的集合完全一致（不能子集也不能超集）
go run ./cmd/agent -- plugins grant my-plugin --capabilities http,fs `
  --allowed-hosts api.example.com --allowed-paths ./data --config .\agent.json

# 撤销授权（保留注册）
go run ./cmd/agent -- plugins deny my-plugin --config .\agent.json

# 查看与收敛
go run ./cmd/agent -- plugins status --config .\agent.json
go run ./cmd/agent -- plugins reload --config .\agent.json
```

要点：

- `--digest` 是必填的：远程包永远不会未经校验就装上。
- `install --grant <caps>` 可以在注册的同时完成授权，但同样要求**完全**列出插件声明的能力；host/path 允许清单仍要用 `plugins grant` 设。
- `--allowed-hosts` / `--allowed-paths` 允许**收窄**到插件声明的子集，但不能出现插件没声明过的名字。
- `install` 与 `grant` 都**不会**自动 reload 运行中的服务，改完执行 `plugins reload`。
- 签名：`plugins keygen` 生成密钥对（私钥文件已存在时拒绝覆盖），`plugins sign <dir>` 对 `plugin.json` 原始字节签名生成 `plugin.sig`。部署侧 `plugins.require_signature=true` 时才验签。

同一套授权也可以走 HTTP：`GET /v1/plugins`、`POST /v1/plugins/{name}/grant|deny`（仅 admin 角色）。

<!-- @end-section -->

<!-- @section: data -->
## backup / restore / data

只支持 SQLite 存储，先确认 `storage.driver=sqlite`。

```powershell
go run ./cmd/agent -- backup --config .\agent.json --out .\backups\agent.db.bak
go run ./cmd/agent -- restore --config .\agent.json --in .\backups\agent.db.bak
```

restore 会保留恢复前的数据库副本，误覆盖可回退。

```powershell
# 导出审计与事件快照
go run ./cmd/agent -- data export --config .\agent.json --out .\agent-export.json

# 预览保留清理（不加 --apply 就是 dry-run）
go run ./cmd/agent -- data retention --config .\agent.json --audit-days 30 --runtime-days 30 --quality-days 30 --episodic-days 30

# 执行
go run ./cmd/agent -- data retention --config .\agent.json --audit-days 30 --runtime-days 30 --quality-days 30 --episodic-days 30 --apply
```

`--episodic-days` 清理情景记忆。注意：情景记忆同时存在 FTS 索引表，清理走命令而不是手删表。

<!-- @end-section -->

<!-- @section: skill -->
## skill：同步技能

```powershell
go run ./cmd/agent -- skill sync --config .\agent.json
go run ./cmd/agent -- skill sync --config .\agent.json --agent writer
go run ./cmd/agent -- skill sync --registry-url https://example.com/skills/index.json --install-root .\skills
```

`--agent writer` 优先使用 writer 自己的 `skills.registry_url` / `skills.install_root`，未配置则回退根配置。

<!-- @end-section -->

<!-- @section: flags -->
## 常用覆盖参数

| 参数 | 适用命令 | 作用 |
|------|----------|------|
| `--config` | 全部 | 指定配置文件 |
| `--prompt` | `run` | 单次任务内容 |
| `--demo` | `run` | 走 demo 响应，不调模型 |
| `--plain` | `run`、`version` | 机器可读输出 |
| `--maas-profile` | `run`、`tui` | 选择 `maas.profiles` 中的 profile |
| `--maas-url` | `run`、`tui` | 临时覆盖 base URL，并绕过 profile |
| `--maas-api-key` | `run`、`tui` | 临时覆盖 API key |
| `--no-context-files` | `run`、`tui` | 本次不加载 AGENTS/SOUL/TOOLS/USER/MEMORY |
| `--addr` | `serve` | HTTP 监听地址 |

配置项也可以用 `LEGION_AGENT_*` 环境变量覆盖，见 [[reference-legion-agent-config-context-001|配置与上下文文件]]。

<!-- @end-section -->

## 相关文档

- [[reference-legion-agent-config-context-001|配置与上下文文件]] — 配置字段与环境变量
- [[reference-legion-agent-http-service-001|HTTP 服务]] — serve 起来之后调什么
- [[reference-legion-agent-auth-001|鉴权与授权参考]] — token / 插件授权
- [[reference-legion-gateway-001|Legion Gateway IM 网关参考]] — `cmd/gateway`
- [[agent-storage-ops-001|存储运维]] — 备份恢复与保留策略细则
