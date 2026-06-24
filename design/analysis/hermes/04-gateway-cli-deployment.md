---
id: "analysis-hermes-gateway-004"
title: "网关、CLI 与部署分析"
aliases: ["hermes gateway", "CLI deployment", "网关CLI部署"]
type: "analysis"
category: "design/analysis/hermes"
tags: ["hermes-agent", "gateway", "cli", "web", "deployment", "tui"]
version: "1.0.0"
created: "2026-05-03"
updated: "2026-05-03"
author: "jxncyjq"
status: "review"
parent: "analysis-hermes-overview-001"
related_docs:
  - id: "analysis-hermes-overview-001"
    relation: "parent"
    path: "./01-overview.md"
  - id: "analysis-hermes-tools-003"
    relation: "related_to"
    path: "./03-tools-skills-plugins.md"
---

<!-- @section: overview -->
# 网关、CLI 与部署分析

## 系统概述

Hermes Agent 提供多种入口方式和用户界面，从命令行到多平台消息网关，从终端 UI 到 Web 仪表盘。这些组件共享相同的核心 `AIAgent` 引擎，但各自面向不同的使用场景。

## 一、CLI 架构

### 入口设计

三个公开入口点（`pyproject.toml`）:
- `hermes` → `hermes_cli.main:main`
- `hermes-agent` → `run_agent:main`
- `hermes-acp` → `acp_adapter.entry:main`

### 子命令系统

`hermes_cli/main.py` — 35 个子命令（按 `subparsers.add_parser` 实际计数）：

| 类别 | 子命令 |
|------|--------|
| **核心** | chat, model, fallback, setup |
| **网关** | gateway (run/start/stop/restart/status/install/setup) |
| **会话** | sessions (list/export/delete/prune/stats/rename/browse) |
| **技能** | skills (browse/search/install/inspect/update/audit/publish) |
| **插件** | plugins (install/update/remove/list/enable/disable) |
| **定时** | cron (list/create/edit/pause/resume/run/remove) |
| **数据** | backup, import, dump, logs |
| **配置** | config (show/edit/set/path/migrate), env, profiles |
| **诊断** | doctor, status, debug |
| **集成** | auth, login, webhook, mcp, acp, dashboard |
| **其他** | version, update, uninstall, completion |

### 斜杠命令注册表

`hermes_cli/commands.py` — `COMMAND_REGISTRY`：
- 单一 Truth Source 驱动 CLI 自动补全、网关分发、Telegram 菜单、Slack 路由
- `CommandDef` 数据类：名称、描述、处理函数、参数定义
- 无重复定义

### 交互式 REPL

`cli.py`（12,043 行）— `HermesCLI` 类：
- **prompt_toolkit** 固定输入区域 TUI
- 自动补全、历史记录、多行编辑
- **Rich** 横幅/面板
- **皮肤引擎** (`hermes_cli/skin_engine.py`): 4 内置皮肤 (default, Ares, Mono, Slate) + 用户 YAML 皮肤
- 斜杠命令自动补全基于 `COMMAND_REGISTRY`

### 配置系统

`hermes_cli/config.py`:
- `DEFAULT_CONFIG` — 默认值
- YAML 合并（深度合并）
- `cfg_get()` 统一访问
- 三种加载路径: CLI 交互、子命令、网关

`hermes_cli/env_loader.py`:
- 加载 `~/.hermes/.env` → 项目 `.env` 备用

## 二、消息网关

### GatewayRunner

`gateway/run.py` — 网关核心：
- 管理网关生命周期
- 为所有已配置平台启动异步适配器
- LRU Agent 缓存（最大 128，空闲 TTL 1 小时）
- 内置 cron 调度器

### 平台适配器

`gateway/platforms/` — 19 个平台适配器（仓库当前快照统计）：

```
BasePlatformAdapter
  ├── Telegram     (telegram.py + telegram_network.py)
  ├── Discord      (discord.py)
  ├── Slack        (slack.py)
  ├── WhatsApp     (whatsapp.py)
  ├── Signal       (signal.py + signal_rate_limit.py)
  ├── Matrix       (matrix.py)
  ├── Mattermost   (mattermost.py)
  ├── Email        (email.py)
  ├── SMS          (sms.py)
  ├── HomeAssistant (homeassistant.py)
  ├── DingTalk     (dingtalk.py)
  ├── WeCom        (wecom.py + wecom_callback.py + wecom_crypto.py)
  ├── WeChat (微信) (weixin.py)
  ├── Feishu       (feishu.py + feishu_comment.py + feishu_comment_rules.py)
  ├── BlueBubbles  (bluebubbles.py)
  ├── Webhook      (webhook.py)
  ├── API Server   (api_server.py)  # 通用 HTTP 入口，不是社交平台
  └── Yuanbao      (yuanbao.py + yuanbao_media.py + yuanbao_proto.py + yuanbao_sticker.py)
```

> 历史文档曾提"QQ Bot (qqbot.py)"，当前仓库快照中没有该文件；如需 QQ 接入，应通过 `webhook.py` 或 `api_server.py` 桥接。`gateway/platforms/` 下所有 .py 文件（含 helper / rate-limit / proto 子模块）合计 30 个，平台核心适配器仅 19 个。

### 平台注册表

`gateway/platform_registry.py`:
- `PlatformRegistry` 单例 + `PlatformEntry` 数据类
- 工厂模式: 插件平台适配器优先于内置 if/elif 链
- `PlatformEntry`: adapter_factory, check_fn, validate_config, required_env, install_hint, max_message_length, pii_safe, emoji, platform_hint

### 投递路由

`gateway/delivery.py` — `DeliveryRouter` + `DeliveryTarget`:
- 支持语法: `"origin"`, `"local"`, `"telegram"`, `"telegram:123456"`, `"telegram:123456:thread_id"`
- 定时任务输出可投递到任意已配置平台

### 会话管理

`gateway/session.py`:
- `SessionStore` — 跨平台会话管理
- `SessionResetPolicy` — 会话重置策略
- PII 脱敏: `_hash_id`, `_hash_sender_id`

### 并发安全

`gateway/session_context.py`:
- 使用 `contextvars.ContextVar` 而非 `os.environ` 管理会话状态
- 确保 asyncio 任务间的并发安全

### 钩子系统

`gateway/hooks.py` — `HookRegistry`:
- 生命周期事件: `gateway:startup`, `session:start/end/reset`
- Agent 事件: `agent:start/step/end`
- 命令事件: `command:*`

## 三、终端 UI (TUI)

### 进程模型

```
hermes --tui
  └─ Node (Ink/React) ──stdio JSON-RPC── Python (tui_gateway)
       │                                      └─ AIAgent + tools + sessions
       └─ 渲染对话记录、编辑器、提示、活动
```

### 前端 (ui-tui/)

- **框架**: React 19 + Ink v6 + TypeScript
- **自定义 Ink**: `packages/hermes-ink/` — Ink 优化分支
- **组件**: transcript, composer, session picker, thinking, markdown
- **状态管理**: @nanostores/react

### 后端 (tui_gateway/)

- `server.py` — JSON-RPC 2.0 over stdio 核心分发器
- `transport.py` — `TeeTransport` 事件双写
- `event_publisher.py` — WebSocket 侧车发布（仪表盘）
- `slash_worker.py` — 斜杠命令工作器
- `render.py` — 消息渲染

## 四、Web 仪表盘

### 前端 (web/)

**框架**: Vite + React 19 + TypeScript + Tailwind CSS v4

**12 个页面**:

| 页面 | 路由 | 功能 |
|------|------|------|
| Sessions | `/sessions` | 活动/最近 Agent 会话 |
| Chat | `/chat` | 嵌入式终端 (xterm.js + PTY) |
| Analytics | `/analytics` | 使用统计和图表 |
| Models | `/models` | 模型选择和配置 |
| Config | `/config` | 模式驱动的动态配置 |
| Cron | `/cron` | 定时作业管理 |
| Skills | `/skills` | 技能安装/切换 |
| Plugins | `/plugins` | 仪表盘插件管理 |
| Profiles | `/profiles` | 多配置文件管理 |
| Logs | `/logs` | 实时日志查看器 |
| Env | `/env` | API Key/环境变量 |
| Docs | `/docs` | 应用内文档 |

### 后端 (hermes_cli/web_server.py)

- **FastAPI** + WebSocket 在 `127.0.0.1:9119` 提供服务
- 会话 Token 认证（敏感端点）
- DNS 重绑定保护的 Host 头验证
- 速率受限的密钥揭示端点
- 提供 Vite 构建的 `web_dist/` 静态 SPA
- 类型化 REST + WebSocket 端点 68 个（按 `hermes_cli/web_server.py` 中 `@app.{get,post,put,delete,patch,websocket}` 装饰器统计）

### WebSocket 客户端

`web/src/lib/gatewayClient.ts` — JSON-RPC 2.0 over WebSocket:
- 事件: `gateway.ready`, `session.info`, `message.*`, `thinking.*`, `tool.*`, `clarify.*`, `approval.*`, `sudo.*`, `error`

### 仪表盘插件系统

`web/src/plugins/types.ts`:
- 侧边栏选项卡注册 (位置: end, after:X, before:X)
- 内置路由覆盖
- 命名插槽: `backdrop`, `header-banner`, `header-left/right`, `pre-main`, `post-main`, `overlay`

### i18n

完整的国际化：英文 + 中文翻译（`web/src/i18n/en.ts`, `zh.ts`）

### 主题

可自定义布局变体，深色主题，亮/暗切换。

## 五、ACP 适配器 (编辑器集成)

### 架构

```
VS Code / Zed / JetBrains
       │ (Agent Client Protocol)
       ▼
acp_adapter/  (Python ACP 服务器，9 个 .py 文件)
  ├── __main__.py    — `python -m acp_adapter` 入口
  ├── entry.py       — `hermes-acp` 控制台脚本入口
  ├── server.py      — AcpAgent: initialize, new_session, prompt, cancel...
  ├── session.py     — SessionManager: ACP session ↔ AIAgent 映射
  ├── auth.py        — detect_provider()
  ├── events.py      — 事件回调适配
  ├── permissions.py — ACP 权限请求/响应桥接
  └── tools.py       — 工具调用事件构建
```

### 关键特性

- 通过 ACP 协议将 Hermes 暴露给编辑器
- 会话持久化到共享 SessionDB（`~/.hermes/state.db`）
- `ThreadPoolExecutor` (max 4 workers) 运行多个 AIAgent
- 支持 Windows → WSL cwd 转换
- 支持会话 fork/resume/load

## 六、定时任务系统

### 存储

`~/.hermes/cron/jobs.json`，原子替换写入

### 调度类型

| 类型 | 语法 | 用途 |
|------|------|------|
| `once` | `"30m"`, `"2h"`, `"1d"`, ISO 时间戳 | 一次性执行 |
| `interval` | `"every 30m"`, `"every 2h"` | 递归间隔 |
| `cron` | 标准 cron 表达式 | 定时调度 |

### 作业功能

- **模型覆盖**: 每个作业独立配置模型/提供者
- **技能注入**: 单个或多个，有序
- **上下文链**: `context_from` — 作业链输出→输入
- **脚本预处理**: `script` — 每次运行前执行数据收集
- **工具集限制**: `enabled_toolsets` 限制可用工具
- **工作目录**: `workdir` 注入 AGENTS.md/CLAUDE.md
- **投递**: `deliver` 目标（origin/local/telegram...）
- **"最多一次"语义**: 执行前推进 `next_run` 防止崩溃循环
- **宽限期**: 错过运行的优雅期

### 调度器

`cron/scheduler.py`:
- 网关守护进程每 60 秒调用 `tick()`
- 基于文件的锁（`fcntl`/`msvcrt`）防止重复执行

## 七、部署架构

### Docker

**基础镜像**: `debian:13.4` + `uv` + `gosu`

**关键特性**:
- 非 root 用户 `hermes` (UID 10000)
- 预装 Node.js, Python 3.13, Playwright Chromium
- Web UI + TUI 在构建时编译
- `pip install -e ".[all]"` 安装所有可选依赖
- 入口点使用 `tini` 防止僵尸进程
- 数据卷 `/opt/data` → `~/.hermes`

### Docker Compose

两个服务：

```yaml
gateway:
  network_mode: host
  command: ["gateway", "run"]
  restart: unless-stopped

dashboard:
  network_mode: host
  command: ["dashboard", "--host", "127.0.0.1", "--no-open"]
  depends_on: [gateway]
```

### 数据目录

`~/.hermes/` 下的所有数据：

```
~/.hermes/
├── state.db          # SQLite 会话数据库 (WAL + FTS5)
├── config.yaml        # 用户配置
├── .env              # API 密钥
├── SOUL.md           # 自定义角色
├── skills/           # 已安装技能
├── plugins/          # 用户安装插件
├── memories/         # Agent 记忆 (MEMORY.md, USER.md)
├── plans/            # Agent 计划
├── cron/
│   ├── jobs.json     # 定时作业
│   └── output/       # 作业输出
├── logs/             # 应用日志
├── skins/            # UI 皮肤
└── sessions/         # 遗留会话 (JSONL, 正在迁移)
```

## 八、RL 训练集成

### environments/

`environments/` — Atropos RL 训练框架集成：

| 环境 | 用途 |
|------|------|
| `hermes_base_env.py` | 抽象基类（服务器管理、工具解析、Agent 循环） |
| `agent_loop.py` | 可重用多轮 Agent 引擎 |
| `terminal_test_env/` | 全栈验证环境 |
| `hermes_swe_env/` | SWE-bench 风格训练 |
| `benchmarks/terminalbench_2/` | 89 个终端自动化任务 |
| `benchmarks/tblite/` | 100 个校准任务 |
| `benchmarks/yc_bench/` | 长周期战略基准测试 |

### 工具调用解析器

`environments/tool_call_parsers/` — 11 个解析器从原始模型输出中提取结构化工具调用：
`hermes`, `mistral`, `llama`, `qwen`, `deepseek_v3`, `kimi_k2`, `longcat`, `glm45`, `glm47`

### 数据生成

`batch_runner.py` + `trajectory_compressor.py`:
- 多进程并行 Agent 执行
- 轨迹压缩（目标 16K-29K tokens）
- ShareGPT 格式输出
- 工具集分布采样

<!-- @end-section -->

<!-- @section: related -->
## 相关文档

- [[01-overview|Hermes Agent 项目架构总览]]
- [[03-tools-skills-plugins|工具、技能与插件系统分析]]
- [[05-data-models|状态持久化与数据模型分析]]

<!-- @end-section -->
