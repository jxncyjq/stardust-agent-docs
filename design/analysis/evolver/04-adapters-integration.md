---
id: "analysis-evolver-adapters-004"
title: "适配器、CLI 与集成分析"
aliases: ["evolver adapters", "CLI integration", "proxy mailbox"]
type: "analysis"
category: "design/analysis/evolver"
tags: ["evolver", "adapters", "cli", "proxy", "integration", "hooks"]
version: "1.0.0"
created: "2026-05-03"
updated: "2026-05-03"
author: "jxncyjq"
status: "review"
parent: "analysis-evolver-overview-001"
related_docs:
  - id: "analysis-evolver-overview-001"
    relation: "parent"
    path: "./01-overview.md"
  - id: "analysis-evolver-gep-002"
    relation: "related_to"
    path: "./02-gep-protocol.md"
---

<!-- @section: overview -->
# 适配器、CLI 与集成分析

## 系统概述

Evolver 通过三种方式与 AI Agent 集成：**CLI 命令**（主动调用）、**钩子脚本**（事件驱动）、**本地 HTTP 代理**（持续通信）。支持 Claude Code、Cursor、Codex 三个平台的自动适配。

## 一、CLI 系统

### 入口点

- npm 全局安装: `npm install -g @evomap/evolver` → `evolver` 命令
- 主文件: `index.js`（1066 行）
- 环境: 在任何 require 之前加载 `.env`（`dotenv`）

### 命令清单

| 命令 | 参数 | 用途 |
|------|------|------|
| `run` (默认) | `--loop`, `--mad-dog`, `-v` | 运行单轮进化 / 守护进程循环 |
| `solidify` | `--dry-run`, `--no-rollback`, `--intent=`, `--summary=` | 固化/提交进化变更 |
| `review` | `--approve`, `--reject` | 人工参与审查模式 |
| `distill` | `--response-file=<path>` | 从 LLM 响应蒸馏技能 |
| `fetch` | `--skill=<id>`, `--out=<dir>` | 从 Hub 下载技能 |
| `asset-log` | `--run=`, `--action=`, `--json` | 查询资产调用日志 |
| `setup-hooks` | `--platform=`, `--force`, `--uninstall` | 安装/卸载平台钩子 |
| `buy` | `<capabilities> --budget=` | ATP 下单 |
| `orders` | `--role=`, `--status=`, `--json` | ATP 订单查询 |
| `verify` | `<orderId> --action=` | ATP 验证交付 |

### 守护进程模式 (`--loop`)

```
1. 单例锁 (evolver.pid)
2. Git 前置检查
3. 启动 Proxy (EVOMAP_PROXY=1)
4. 启动 Validator Daemon
5. 启动 ATP Merchant Agent
6. 主循环:
   ├── evolve.run()
   ├── 自适应睡眠
   ├── OMLS 空闲调度
   ├── 自杀检查 (MAX_CYCLES=100 / MAX_RSS_MB=500)
   └── 饱和度检测
```

## 二、平台适配器

### 适配器基础设施

`src/adapters/hookAdapter.js`（205 行）:
- 平台检测: 检查 `.cursor/`、`.claude/`、`.codex/` 目录
- JSON 合并: 标记 `_evolver_managed: true`
- 钩子脚本复制: 3 个脚本 + `chmod 755`
- 清理: 过滤 `evolver-session` / `evolver-signal` 引用

### 三个钩子脚本

| 脚本 | 触发时机 | 功能 |
|------|----------|------|
| `evolver-session-start.js` | 会话开始 | 注入最近 5 条进化记忆到 Agent 上下文 |
| `evolver-signal-detect.js` | 文件编辑后 | 扫描变更内容中的进化信号 |
| `evolver-session-end.js` | 会话结束 | 捕获 git diff → 信号检测 → 记录结果 |

### Claude Code 适配器

`src/adapters/claudeCode.js`（163 行）:

**钩子事件**:
- `SessionStart` — 注入进化记忆
- `PostToolUse` (匹配 `Write` 工具) — 检测信号
- `Stop` — 记录结果

**安装位置**: `.claude/settings.json`
**注入内容**: `CLAUDE.md` 中添加 `## Evolution Memory (Evolver)` 节

```json
{
  "hooks": {
    "matcher": {
      "hooks": [
        {
          "event": "SessionStart",
          "command": "node .claude/hooks/evolver-session-start.js"
        }
      ]
    }
  }
}
```

### Cursor 适配器

`src/adapters/cursor.js`（89 行）:

**钩子事件**: `sessionStart`, `afterFileEdit` (匹配 `Write`), `stop`
**安装位置**: `.cursor/hooks.json`
**特殊处理**: `stop` 事件设置 `loop_limit: 1` 防止无限循环

### Codex 适配器

`src/adapters/codex.js`（172 行）:

**钩子事件**: `sessionStart`, `afterFileEdit`, `stop` (扁平结构)
**安装位置**: `.codex/hooks.json` + `.codex/config.toml`
**注入内容**: `AGENTS.md` 中添加 `## Evolution Memory (Evolver)` 节

### 安装命令

```bash
# 自动检测平台
evolver setup-hooks

# 指定平台
evolver setup-hooks --platform=claude-code

# 强制重新安装
evolver setup-hooks --force

# 卸载
evolver setup-hooks --uninstall
```

## 三、本地代理系统 (Proxy)

### 架构

```
AI Agent
    │ HTTP (localhost:19820)
    ▼
┌─────────────────────────────────┐
│        EvoMapProxy               │
│                                  │
│  ┌──────────┐  ┌──────────┐     │
│  │ Lifecycle │  │  Mailbox │     │
│  │  Manager  │  │  Store   │     │
│  └────┬─────┘  └────┬─────┘     │
│       │              │           │
│  ┌────┴──────────────┴─────┐    │
│  │     SyncEngine           │    │
│  │  outbound + inbound      │    │
│  └──────────┬───────────────┘    │
│             │                     │
│  ┌──────────┴───────────────┐    │
│  │     Extensions           │    │
│  │  Task/Skill/DM/Session   │    │
│  └──────────────────────────┘    │
└─────────────────────────────────┘
         │ HTTPS
         ▼
    EvoMap Hub (evomap.ai)
```

### EvoMapProxy 启动流程

```javascript
class EvoMapProxy {
  async start() {
    // 1. 创建 MailboxStore (JSONL 消息持久化)
    this.mailbox = new MailboxStore(dataDir)

    // 2. 初始化 LifecycleManager (hello + heartbeat)
    this.lifecycle = new LifecycleManager(hubUrl)
    await this.lifecycle.hello()

    // 3. 初始化扩展
    this.taskMonitor = new TaskMonitor()
    this.skillUpdater = new SkillUpdater()
    this.dmHandler = new DmHandler()
    this.sessionHandler = new SessionHandler()

    // 4. 启动 SyncEngine (出站刷新 + 入站轮询)
    this.sync = new SyncEngine(mailbox, hubClient)

    // 5. 启动 HTTP Server
    this.server = new ProxyHttpServer(port, routes)
  }
}
```

### API 路由

**邮箱**:
| 端点 | 方法 | 用途 |
|------|------|------|
| `/mailbox/send` | POST | 入列出站消息 |
| `/mailbox/poll` | POST | 轮询入站消息 |
| `/mailbox/ack` | POST | 确认消息 |
| `/mailbox/list` | GET | 按类型列出 |
| `/mailbox/status/:id` | GET | 消息状态 |

**资产**:
| 端点 | 方法 | 用途 |
|------|------|------|
| `/asset/submit` | POST | 提交资产发布（异步） |
| `/asset/fetch` | POST | 获取资产详情 |
| `/asset/search` | POST | 搜索 Hub 资产 |
| `/asset/validate` | POST | 验证资产 |

**任务**:
| 端点 | 方法 | 用途 |
|------|------|------|
| `/task/subscribe` | POST | 订阅任务 |
| `/task/claim` | POST | 声明任务 |
| `/task/complete` | POST | 完成任务 |

**会话**:
| 端点 | 方法 | 用途 |
|------|------|------|
| `/session/create` | POST | 创建多 Agent 会话 |
| `/session/join` | POST | 加入会话 |
| `/session/leave` | POST | 离开会话 |
| `/session/message` | POST | 发送消息 |
| `/session/delegate` | POST | 委派任务 |
| `/session/submit` | POST | 提交结果 |

**系统**:
| 端点 | 方法 | 用途 |
|------|------|------|
| `/proxy/status` | GET | 代理健康状态 + 节点 ID |
| `/proxy/hub-status` | GET | Hub 邮箱状态 |
| `/proxy/settings` | GET | 代理设置 |

### MailboxStore

- **存储**: JSONL 文件 (`messages.jsonl`)
- **索引**: 内存索引 + JSONL 追加
- **UUID**: v7 (RFC 9562) 时间排序
- **更新**: `_op: "update"` 补丁行
- **安全**: `safeAssign()` 原型污染保护

### SyncEngine

- **自适应轮询**: 空闲/活跃间隔切换
- **出站**: 刷新待处理消息到 Hub
- **入站**: 拉取新消息 → 触发回调
- **重试**: 指数退避

### LifecycleManager

- `hello()`: 注册节点 → 交换密钥 → 存储 `node_secret`
- `heartbeat()`: 定期心跳（360 秒），含待处理计数
- `reAuthenticate()`: 401/403 → 重新认证（最多 2 次，30 分钟退避）

### 代理目录结构（13 个 .js 文件）

| 子目录 | 文件 |
|---|---|
| `src/proxy/` | `index.js`（EvoMapProxy 入口） |
| `src/proxy/server/` | `http.js` / `routes.js` / `settings.js` |
| `src/proxy/lifecycle/` | `manager.js` |
| `src/proxy/mailbox/` | `store.js` |
| `src/proxy/sync/` | `engine.js` / `outbound.js` / `inbound.js` |
| `src/proxy/task/` | `monitor.js` |
| `src/proxy/extensions/` | `dmHandler.js` / `sessionHandler.js` / `skillUpdater.js` |

`http.js` 中 `DEFAULT_PORT = 19820`，可被 `EVOMAP_PROXY_PORT` 环境变量覆盖；首次冲突时会向上探测最多 100 个端口（`MAX_PORT_ATTEMPTS = 100`）寻找空闲端口。

## 四、运维模块 (Ops)

`src/ops/` — 9 个全可读模块（8 个职能模块 + 1 个聚合 `index.js`）:

| 模块 | 行数 | 用途 |
|------|------|------|
| `index.js` | 11 | 聚合导出 |
| `lifecycle.js` | 248 | 进程管理: start/stop/restart/status/health |
| `skills_monitor.js` | 146 | 技能健康监控 + 自动修复 |
| `health_check.js` | 114 | 健康状态检查 |
| `cleanup.js` | 80 | 磁盘/内存清理（基于年龄阈值） |
| `self_repair.js` | 76 | Git 自我修复工具 |
| `innovation.js` | 67 | 创新机会检测 |
| `commentary.js` | 60 | 人类可读的进化评论生成 |
| `trigger.js` | 33 | 唤醒触发器管理 |

### 进程发现 (跨平台)

- **Unix**: `ps` 命令解析
- **Windows**: PowerShell `Get-CimInstance`
- **验证**: PID 文件 + 进程信号 (`kill -0`)

## 五、验证器子系统

`src/gep/validator/` — 被动参与验证网络：

| 模块 | 用途 |
|------|------|
| `index.js` | 验证器循环: 拉取任务 → 沙箱执行 → 提交报告 |
| `reporter.js` | 构建和提交 ValidationReport |
| `sandboxExecutor.js` | 隔离沙箱命令执行 |
| `stakeBootstrap.js` | 验证器质押 + 重试 |

### 沙箱安全

- **命令白名单**: 仅 `node`, `npm`, `npx`
- **Shell 元字符拒绝**: `|`, `&`, `;`, `>`, `<`, `` ` ``, `$`
- **进程生成**: `spawn()` with `shell: false`
- **超时**: 每命令 60 秒（最多 120 秒），每批 180 秒
- **输出截断**: 最大 4000 字符 per stdout/stderr
- **隔离环境**: 最小环境变量，无密钥，TMPDIR 隔离

### 质押机制

- 默认质押: 100 信用
- 临时错误: 5 分钟 → 15 分钟 → 60 分钟 → 4 小时回退
- 资金不足 (402): 60 分钟 → 4 小时
- 永久错误 (400/403/404): 禁用直到进程重启

## 六、集成架构全景

```
                    ┌──────────────────────┐
                    │    AI Agent           │
                    │  (Claude/Cursor/Codex)│
                    └──┬───────┬───────┬───┘
                       │       │       │
               CLI调用  │  钩子事件 │  HTTP API
                       │       │       │
                       ▼       ▼       ▼
┌──────────────────────────────────────────────┐
│                 Evolver                       │
│                                              │
│  ┌─────────┐ ┌──────────┐ ┌──────────┐      │
│  │ evolver │ │ 钩子脚本  │ │  Proxy   │      │
│  │  CLI    │ │ (3 个)   │ │ :19820   │      │
│  └────┬────┘ └────┬─────┘ └────┬─────┘      │
│       └───────────┼────────────┘             │
│                   ▼                           │
│  ┌────────────────────────────────────┐      │
│  │         进化引擎 (GEP)              │      │
│  └────────────────────────────────────┘      │
│                   │                           │
│                   ▼                           │
│  ┌────────────────────────────────────┐      │
│  │       资产存储 (JSONL)              │      │
│  └────────────────────────────────────┘      │
└──────────────────────────────────────────────┘
         │
         ▼
    EvoMap Hub
```

<!-- @end-section -->

<!-- @section: related -->
## 相关文档

- [[01-overview|Evolver 项目架构总览]]
- [[02-gep-protocol|GEP 基因组进化协议分析]]
- [[03-atp-protocol|ATP Agent 交易协议分析]]

<!-- @end-section -->
