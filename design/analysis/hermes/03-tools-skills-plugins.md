---
id: "analysis-hermes-tools-003"
title: "工具、技能与插件系统分析"
aliases: ["hermes tools", "skills plugins", "工具技能插件"]
type: "analysis"
category: "design/analysis/hermes"
tags: ["hermes-agent", "tools", "skills", "plugins", "mcp", "extensibility"]
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
  - id: "analysis-hermes-runtime-002"
    relation: "related_to"
    path: "./02-agent-runtime.md"
---

<!-- @section: overview -->
# 工具、技能与插件系统分析

## 系统概述

Hermes Agent 的能力扩展通过三个互补的子系统实现：

| 系统 | 粒度 | 加载时机 | 用途 |
|------|------|----------|------|
| **Tools** | 函数级 | 导入时自注册 | Agent 可调用的原子操作 |
| **Skills** | 目录级 | 按需加载 | 可复用的领域知识程序包 |
| **Plugins** | 模块级 | 启动时注册 | 扩展核心能力（记忆/上下文/钩子） |

## 一、工具系统 (Tools)

### ToolRegistry — 中央注册表

`tools/registry.py` — **单例模式**：

```python
class ToolRegistry:
    _tools: Dict[str, ToolEntry]     # name → entry
    _lock: threading.RLock            # 线程安全
    _generation: int                  # 缓存失效计数器

    def register(name, toolset, schema, handler, check_fn, ...)
    def deregister(name)              # MCP 动态工具刷新
    def get_definitions(tool_names)   # → OpenAI 兼容 schema 列表
    def dispatch(name, args)          # → 工具执行结果
```

### ToolEntry 结构

```python
@dataclass
class ToolEntry:
    name: str                    # 工具名称
    toolset: str                 # 所属工具集
    schema: Dict                 # OpenAI function schema
    handler: Callable            # 执行函数
    check_fn: Optional[Callable] # 可用性检查
    requires_env: List[str]      # 需要的环境变量
    is_async: bool               # 是否异步
    description: str             # 描述
    emoji: str                   # 图标
    max_result_size_chars: int   # 结果大小上限
```

### 工具自动发现

`model_tools.py::discover_builtin_tools()`:
1. 扫描 `tools/*.py` 文件
2. **AST 解析** — 找顶层的 `registry.register(...)` 调用
3. 跳过 `mcp_tool.py`（昂贵的 MCP SDK 导入）
4. 导入每个文件 → 触发 `register()` 自注册
5. 无需维护显式导入列表

### 内置工具分类（68 个静态注册的工具规范）

来源：`grep -rn "^registry\.register(" tools/*.py` 共 68 处顶层调用（不含 MCP 在运行时通过 `tools/list_changed` 动态注册的工具，也不含通过 `PluginContext.register_tool` 注入的插件工具）。`tools/` 目录下另有 86 个 .py 文件，剩余 18 个是 helper / abstract base / parser，不直接 `register()`。

| 类别 | 工具 | 数量 |
|------|------|------|
| **终端** | terminal_tool, code_execution_tool | 2 |
| **环境** | local, docker, ssh, modal, daytona, vercel, singularity | 7 |
| **浏览器** | browser_tool, browser_cdp, browser_dialog, browser_supervisor, camofox | 5 |
| **文件** | file_tools, file_operations, file_state | 3 |
| **网络** | web_search, web_extract, vision_tools | 3 |
| **消息** | send_message, discord, feishu, homeassistant, yuanbao | 5 |
| **多媒体** | image_generation, tts, transcription, voice_mode | 4 |
| **Agent 管理** | delegate_task, kanban, clarify, todo, memory, session_search | 6 |
| **技能** | skills_list, skill_view, skill_manager | 3 |
| **MCP** | mcp_tool（MCP 服务器客户端，运行时动态扩展） | 1 |
| **安全** | approval, interrupt, slash_confirm | 3 |
| **其他**（cronjob/rl_training/checkpoint_manager 等） | 见 `tools/*.py` | 26 |

### 工具执行流程

```
AIAgent._invoke_tool(name, args)
  ├── 1. pre_tool_call 插件钩子检查
  ├── 2. 内置代理级工具 (todo/memory/delegate/clarify/session_search)
  ├── 3. 记忆管理器工具 (外部记忆提供者)
  └── 4. handle_function_call() → model_tools.py
       ├── 参数类型强制转换
       ├── registry.dispatch(name, args)
       ├── post_tool_call / transform_tool_result 钩子
       └── 返回 JSON 字符串结果
```

### 并发工具执行

- `_should_parallelize_tool_batch()` 检查工具调用的独立性
- `_execute_tool_calls_concurrent()` 使用 `ThreadPoolExecutor` 并行执行
- 结果按原始调用顺序收集

### 工具看门狗

`agent/tool_guardrails.py`:
- 每回合跟踪工具调用模式
- 检测精确重复失败、同工具故障、幂等无进展
- 决策: `allow` → `warn` → `block` → `halt`

### 环境后端

`tools/environments/` — 终端执行后端（`BaseEnvironment` ABC）:

```
BaseEnvironment
  ├── Local         — 本地 Shell
  ├── Docker        — Docker 容器
  ├── SSH           — 远程 SSH
  ├── Daytona       — Daytona 开发环境
  ├── Modal         — Modal 无服务器
  ├── Singularity   — HPC 容器
  └── VercelSandbox — Vercel 沙箱
```

## 二、技能系统 (Skills)

### 技能格式

每个技能是一个包含 `SKILL.md` 的目录（遵循 `agentskills.io` 开放标准）：

```yaml
---
name: skill-name             # 必需, 最多 64 字符
description: Brief desc      # 必需, 最多 1024 字符
version: 1.0.0
license: MIT
platforms: [macos, linux]
prerequisites:
  env_vars: [API_KEY]
  commands: [curl, jq]
metadata:
  hermes:
    tags: [fine-tuning, llm]
---
# 技能内容 (Markdown)...
```

可选子目录: `references/`, `templates/`, `scripts/`, `assets/`

### 渐进式加载

1. **元数据扫描** (`skill_utils.py`): 解析 Front Matter → 名称、描述、标签
2. **技能索引注入**: 系统提示中包含所有技能的触发条件和描述
3. **按需加载**: Agent 使用 `skill_view` 工具查看完整内容
4. **模板变量**: `${HERMES_SKILL_DIR}` 和内联 Shell `!<cmd>` 预处理

### 技能管理

- **`skill_commands.py`** — `/skill-name` 斜杠命令处理
- **`skill_manager_tool.py`** — Agent 自主创建/编辑/删除技能
- **`skills_sync.py`** — 捆绑技能清单同步到 `~/.hermes/skills/`
- **`skills_hub.py`** — 技能市场：`SkillSource` ABC、GitHub 源、可选技能源

### 安全扫描

`tools/skills_guard.py`（932 行，~520 条字符串模式）— 外部来源技能的威胁模式扫描：

| 类别 | 检测内容 |
|------|----------|
| 数据泄露 | 敏感文件读取、网络外传 |
| 提示注入 | "ignore previous instructions" 模式 |
| 破坏性操作 | `rm -rf`, `format`, `dd` |
| 持久化 | crontab 修改, systemd 服务 |
| 混淆 | base64 编码, Unicode 隐藏字符 |
| 供应链 | 可疑 URL、依赖下载 |
| 凭据暴露 | API Key 模式 |

**信任级别**: `builtin` > `trusted` > `community` > `agent-created`

### 技能策展

`agent/curator.py` (69KB) — 后台维护：
- 定期审查 Agent 创建的技能
- 生命周期: 活跃 → 陈旧 → 已归档
- 仅管理 Agent 创建的技能，永不自动删除

### 内置技能目录

`skills/` 下共 89 个 SKILL.md（按 `find skills -name SKILL.md` 统计），分布在 25 个分类目录中。`optional-skills/` 另有 60 个 SKILL.md（15 个分类）。

`skills/` 实际分类（25 个）：

| 分类目录 | 主题 |
|---------|------|
| `software-development/` | git-workflow / code-review / refactoring（11 个技能） |
| `creative/` | architecture-diagram / ascii-art / meme-generation 等（19 个技能，最大类别） |
| `productivity/` | meeting-notes / calendar 等（8 个） |
| `github/` | issue-management / pr-review（6 个） |
| `research/` | literature-review / data-analysis（5 个） |
| `media/` | 多媒体处理（5 个） |
| `autonomous-ai-agents/` | claude-code / codex / hermes-agent（4 个） |
| `apple/` | apple-notes / imessage / findmy / apple-reminders（4 个） |
| `devops/` | docker / cli / k8s-debug（3 个） |
| `mlops/` | training / inference / models / evaluation（11 个分布在 4 个子目录） |
| `gaming/` | 游戏相关（2 个） |
| `diagramming` / `dogfood` / `domain` / `email` / `gifs` / `index-cache` / `inference-sh` / `mcp` / `note-taking` / `red-teaming` / `smart-home` / `social-media` / `yuanbao` | 其余分类（每类 1–2 个） |

## 三、插件系统 (Plugins)

### 插件类型

| 类型 | 目录 | 注册方式 |
|------|------|----------|
| 记忆提供者 | `plugins/memory/` | `register_memory_provider()` |
| 上下文引擎 | `plugins/context_engine/` | `ContextEngine` 子类 |
| 图像生成 | `plugins/image_gen/` | `register_image_gen_provider()` |
| 通用钩子 | `plugins/<name>/` | `register(PluginContext)` |
| 仪表盘 | `plugins/<name>/` | Web 插件 SDK |

### PluginContext 接口

```python
class PluginContext:
    def register_tool(name, toolset, schema, handler, ...)
    def register_hook(event, callback)       # 钩子注册
    def register_memory_provider(provider)
    def register_context_engine(engine)
    def register_cli_command(name, handler)
    def register_platform(platform_entry)
```

### 插件钩子系统

`plugins/` 目录下的通用插件可以注册以下钩子：

| 钩子 | 触发时机 |
|------|----------|
| `pre_tool_call` | 工具调用前（可阻止） |
| `post_tool_call` | 工具调用后 |
| `pre_llm_call` | LLM API 调用前 |
| `post_llm_call` | LLM API 调用后 |
| `on_session_start` | 会话开始 |
| `on_session_end` | 会话结束 |

### 发现机制

1. 扫描 `plugins/<name>/plugin.yaml`
2. 加载 `plugins/<name>/__init__.py` 中的 `register(ctx)` 函数
3. 捆绑的插件优先级高于用户安装的插件
4. 假 `PluginContext` 收集器用于注册捕获

### 已内置的插件

| 插件 | 类型 | 功能 |
|------|------|------|
| memory/honcho | 记忆提供者 | 对话记忆 |
| memory/mem0 | 记忆提供者 | 向量记忆 |
| memory/holographic | 记忆提供者 | SQLite + FTS5 + HRR |
| context_engine/ | 上下文引擎 | 可插拔压缩引擎 |
| observability/langfuse | 可观测性 | 追踪/监控集成 |
| hermes-achievements | 仪表盘 | 成就系统 |
| kanban | 仪表盘 + 工具 | 看板项目管理 |
| disk-cleanup | 系统 | 磁盘清理 |
| google_meet | 集成 | Google Meet 机器人 |
| spotify | 集成 | Spotify 控制 |

## 四、MCP 集成

### MCP 客户端

`tools/mcp_tool.py`（3,145 行）：

**架构**:
- 专用后台 `asyncio.Task` 运行每个 MCP 服务器
- 支持 **stdio**（命令 + 参数）和 **HTTP/StreamableHTTP**（url）传输
- 工具调用通过 `run_coroutine_threadsafe()` 调度

**关键功能**:
1. **工具发现**: 连接时 `list_tools()`，自动注册到 `ToolRegistry`
2. **动态刷新**: 处理 `tools/list_changed` 通知
3. **采样支持**: MCP 服务器可通过 `sampling/createMessage` 请求 LLM 完成
4. **断路器**: 连续失败 → 冷却时间内短路调用
5. **自动重连**: 指数退避，最多 5 次
6. **安全**: 子进程环境变量过滤、提示注入扫描

### MCP 服务器

`mcp_serve.py` — 将 Hermes 暴露为 MCP 工具：

```bash
hermes mcp serve
```

暴露的工具（匹配 OpenClaw 的 9 工具 MCP 桥接表面）：
- `conversations_list`, `conversation_get`, `messages_read`
- `messages_send`, `attachments_fetch`
- `events_poll`, `events_wait`
- `permissions_list_open`, `permissions_respond`
- `channels_list`（Hermes 特有）

## 五、工具集系统

### 工具集定义

`toolsets.py`:

```python
_HERMES_CORE_TOOLS = [
    # 约 66 个 CLI 和消息平台共享的核心工具
    "web_search", "terminal", "file_*", "browser_*",
    "skills_*", "vision", "image_generation", "tts",
    "todo", "memory", "session_search", "clarify",
    "execute_code", "delegate_task", "cronjob",
    "send_message", "kanban_*", ...
]

TOOLSETS = {
    "web": {"description": "...", "tools": ["web_search", "web_extract"]},
    "terminal": {"description": "...", "tools": ["terminal", "code_execution"]},
    "research": {"includes": ["web", "terminal", "file", "vision", "skills"]},
    "development": {"includes": ["terminal", "file", "web", "skills"]},
    "full_stack": {"tools": [...all...]},
    "minimal": {"tools": ["terminal", "todo"]},
    ...
}
```

**特点**:
- 可组合: 工具集可通过 `includes` 引用其他工具集
- 递归解析: `resolve_toolset()` 扁平化为最终工具列表

### 工具集分布

`toolset_distributions.py` — 用于数据生成的概率分布：

```python
DISTRIBUTIONS = {
    "default": {"toolsets": {"web": 30, "file": 40, "terminal": 40, ...}},
    "image_gen": {"toolsets": {"image_gen": 60, "vision": 40, ...}},
    "browser_tasks": {"toolsets": {"browser": 50, "file": 30, ...}},
    ...
}
```

`sample_toolsets_from_distribution(name)` — 按概率独立掷骰子选择工具集

<!-- @end-section -->

<!-- @section: related -->
## 相关文档

- [[01-overview|Hermes Agent 项目架构总览]]
- [[02-agent-runtime|Agent 运行时引擎分析]]
- [[04-gateway-cli-deployment|网关、CLI 与部署分析]]

<!-- @end-section -->
