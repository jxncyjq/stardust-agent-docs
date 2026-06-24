---
id: "analysis-clawcode-checklist-004"
title: "Claw Code 系统化需求清单"
aliases: ["需求清单", "claw-code requirements", "claw 功能清单"]
type: "analysis"
category: "design/analysis/claude"
tags: ["claw-code", "requirements", "checklist", "completeness"]
version: "1.0.0"
created: "2026-05-03"
updated: "2026-05-03"
author: "jxncyjq"
status: "review"
parent: "analysis-clawcode-overview-001"
children: []
related_docs:
  - id: "analysis-clawcode-overview-001"
    relation: "depends_on"
    path: "./01-overview.md"
  - id: "analysis-clawcode-rust-002"
    relation: "references"
    path: "./02-rust-crates-analysis.md"
  - id: "analysis-clawcode-python-003"
    relation: "references"
    path: "./03-python-subsystems-analysis.md"
  - id: "analysis-clawcode-gap-007"
    relation: "related_to"
    path: "./07-gap-analysis.md"
---

<!-- @section: overview -->
# Claw Code 系统化需求清单

> 基于 `rust/crates/` 和 `src/` 所有模块的功能分析，按功能领域系统化整理。

---

## 1. CLI 入口与用户交互

| ID | 需求项 | 对应模块 | 状态 |
|----|--------|----------|------|
| CLI-001 | 命令行参数解析（--model / --permission-mode / --resume / --output-format 等） | `rusty-claude-cli/main.rs` | ✅ 已实现 |
| CLI-002 | 交互式 REPL 模式（支持多行输入、斜杠命令补全） | `rusty-claude-cli/main.rs` + `input.rs` | ✅ 已实现 |
| CLI-003 | 一次性 prompt 模式（非交互式） | `rusty-claude-cli/main.rs` | ✅ 已实现 |
| CLI-004 | JSON 输出格式支持（用于脚本化） | `rusty-claude-cli/main.rs` | ✅ 已实现 |
| CLI-005 | Markdown 到 ANSI 终端渲染（含语法高亮） | `rusty-claude-cli/render.rs` | ✅ 已实现 |
| CLI-006 | 终端颜色主题支持 | `rusty-claude-cli/render.rs` | ✅ 已实现 |
| CLI-007 | 旋转动画指示器（Braille spinner） | `rusty-claude-cli/render.rs` | ✅ 已实现 |
| CLI-008 | 斜杠命令 Tab 补全和语法高亮 | `rusty-claude-cli/input.rs` | ✅ 已实现 |
| CLI-009 | 仓库初始化向导（.claw/ 目录、CLAUDE.md 生成） | `rusty-claude-cli/init.rs` | ✅ 已实现 |
| CLI-010 | 技术栈自动检测 | `rusty-claude-cli/init.rs` | ✅ 已实现 |
| CLI-011 | Ctrl+J / Shift+Enter 多行输入支持 | `rusty-claude-cli/input.rs` | ✅ 已实现 |

---

## 2. 配置管理

| ID | 需求项 | 对应模块 | 状态 |
|----|--------|----------|------|
| CFG-001 | 三层配置加载与合并（User > Project > Local） | `config.rs` | ✅ 已实现 |
| CFG-002 | 配置文件发现（~/.claw.json / .claw/settings.json 等） | `config.rs` | ✅ 已实现 |
| CFG-003 | 配置文件校验（未知键/类型错误/弃用字段检测） | `config_validate.rs` | ✅ 已实现 |
| CFG-004 | 模型别名解析（opus/sonnet/haiku 等别名） | `api/providers/mod.rs` | ✅ 已实现 |
| CFG-005 | 用户自定义别名（settings.json 中定义） | `api/providers/mod.rs` | ✅ 需验证 |
| CFG-006 | JSON 配置文件解析（不依赖 serde_json） | `json.rs` | ✅ 已实现 |
| CFG-007 | 功能开关配置（hooks/plugins/MCP/OAuth/model/permissions/sandbox） | `config.rs` | ✅ 已实现 |
| CFG-008 | 工作区指纹化配置隔离 | `session_control.rs` | ✅ 已实现 |

---

## 3. AI 提供商集成

| ID | 需求项 | 对应模块 | 状态 |
|----|--------|----------|------|
| PRV-001 | Anthropic Messages API 客户端 | `api/providers/anthropic.rs` | ✅ 已实现 |
| PRV-002 | OpenAI Chat Completions 兼容客户端 | `api/providers/openai_compat.rs` | ✅ 已实现 |
| PRV-003 | xAI (Grok) 提供商支持 | `api/providers/openai_compat.rs` | ✅ 已实现 |
| PRV-004 | DashScope (阿里云/通义千问) 提供商支持 | `api/providers/openai_compat.rs` | ✅ 已实现 |
| PRV-005 | OpenRouter 代理支持 | `api/providers/openai_compat.rs` | ✅ 已实现 |
| PRV-006 | Ollama 本地模型支持 | `api/providers/openai_compat.rs` | ✅ 已实现 |
| PRV-007 | 模型名前缀路由（根据前缀自动选提供商） | `api/providers/mod.rs` | ✅ 已实现 |
| PRV-008 | API Key 认证（x-api-key header） | `api/providers/anthropic.rs` | ✅ 已实现 |
| PRV-009 | Bearer Token 认证（Authorization header） | `api/providers/anthropic.rs` | ✅ 已实现 |
| PRV-010 | 凭证检测与错误提示（sk-ant-* 误放入 Bearer 槽） | `api/client.rs` | ✅ 已实现 |
| PRV-011 | 请求格式转换（Anthropic ↔ OpenAI 格式） | `api/providers/openai_compat.rs` | ✅ 已实现 |
| PRV-012 | 指数退避自动重试（最多 8 次） | `api/providers/anthropic.rs` | ✅ 已实现 |
| PRV-013 | 飞行前 token 估算检查（防超上下文窗口） | `api/providers/mod.rs` | ✅ 已实现 |
| PRV-014 | 推理模型参数自动清理（temperature/top_p 等） | `api/providers/openai_compat.rs` | ✅ 已实现 |

---

## 4. 会话管理

| ID | 需求项 | 对应模块 | 状态 |
|----|--------|----------|------|
| SES-001 | 对话消息持久化（ConversationMessage + ContentBlock） | `session.rs` | ✅ 已实现 |
| SES-002 | 会话版本化序列化 | `session.rs` | ✅ 已实现 |
| SES-003 | 会话文件轮转（>256KB 自动轮转，保留最近 3 个备份） | `session.rs` | ✅ 已实现 |
| SES-004 | 工作区哈希命名空间隔离 | `session_control.rs` | ✅ 已实现 |
| SES-005 | 会话列出/删除/修剪 | `session_control.rs` | ✅ 已实现 |
| SES-006 | 会话恢复（--resume latest） | `main.rs` + `session.rs` | ✅ 已实现 |
| SES-007 | 会话压缩元数据记录 | `session.rs` | ✅ 已实现 |
| SES-008 | 会话分叉溯源 | `session.rs` | ✅ 已实现 |
| SES-009 | 会话导出 | `main.rs` | ✅ 已实现 |
| SES-010 | token 用量聚合统计（session 级别） | `usage.rs` | ✅ 已实现 |

---

## 5. 对话引擎

| ID | 需求项 | 对应模块 | 状态 |
|----|--------|----------|------|
| CNV-001 | 对话循环编排（请求组装→流式解析→工具调用→结果回传） | `conversation.rs` | ✅ 已实现 |
| CNV-002 | 系统提示组装（cwd/日期/git/shell/指令文件） | `prompt.rs` | ✅ 已实现 |
| CNV-003 | CLAUDE.md 指令文件发现与加载 | `prompt.rs` | ✅ 已实现 |
| CNV-004 | Git 上下文自动收集（分支名/最近提交/暂存文件） | `git_context.rs` | ✅ 已实现 |
| CNV-005 | SSE 流式事件解析 | `sse.rs` | ✅ 已实现 |
| CNV-006 | 流式 Markdown 渲染状态管理 | `rusty-claude-cli/render.rs` | ✅ 已实现 |
| CNV-007 | 会话自动压缩（token 超出预算时触发） | `compact.rs` | ⚠️ 部分实现 |
| CNV-008 | 摘要文本压缩（按字符/行/行宽预算） | `summary_compression.rs` | ✅ 已实现 |
| CNV-009 | 提示缓存（请求指纹/Cache 命中/Cache 断裂检测） | `api/prompt_cache.rs` | ✅ 已实现 |
| CNV-010 | token 使用量实时跟踪 | `usage.rs` | ✅ 已实现 |
| CNV-011 | token 费用估算（按模型定价表） | `usage.rs` | ✅ 已实现 |

---

## 6. 权限与安全

| ID | 需求项 | 对应模块 | 状态 |
|----|--------|----------|------|
| PRM-001 | 五级权限模式（ReadOnly / WorkspaceWrite / DangerFullAccess / Prompt / Allow） | `permissions.rs` | ✅ 已实现 |
| PRM-002 | 基于工具到模式映射表的授权决策 | `permissions.rs` | ✅ 已实现 |
| PRM-003 | 权限执行门控（PermissionEnforcer.check） | `permission_enforcer.rs` | ✅ 已实现 |
| PRM-004 | 文件写入工作区边界验证 | `permission_enforcer.rs` + `file_ops.rs` | ✅ 已实现 |
| PRM-005 | 只读模式下 bash 写入命令拦截 | `permission_enforcer.rs` + `bash_validation.rs` | ✅ 已实现 |
| PRM-006 | Prompt 模式自动放行（委托给交互式提示） | `permission_enforcer.rs` | ✅ 已实现 |
| PRM-007 | 权限覆盖（Allow/Deny/Ask）支持 | `permissions.rs` | ✅ 已实现 |
| PRM-008 | 目录信任检查与自动白名单 | `trust_resolver.rs` | ✅ 已实现 |
| PRM-009 | 信任策略（AutoTrust / RequireApproval / Deny） | `trust_resolver.rs` | ✅ 已实现 |

---

## 7. 工具系统

> 本节按"功能性需求"维度组织，共 45 项；它们对应 `tools/src/lib.rs::mvp_tool_specs()` 中的 50 个工具规范——`Worker*` 编排工具族（9 项）以单条 ORC 范畴需求覆盖（见 §15），`Brief` 是别名不计需求项，因此 45 ≠ 50。

### 7.1 文件操作工具

| ID | 需求项 | 对应模块 | 状态 |
|----|--------|----------|------|
| TOL-001 | 读取文件（二进制检测、10MB 限制） | `file_ops.rs` | ✅ 已实现 |
| TOL-002 | 写入文件（10MB 限制、工作区边界检查） | `file_ops.rs` | ✅ 已实现 |
| TOL-003 | 编辑文件（结构化 patch 生成） | `file_ops.rs` | ✅ 已实现 |
| TOL-004 | Glob 模式文件搜索 | `file_ops.rs` | ✅ 已实现 |
| TOL-005 | 正则表达式内容搜索（含区块组装） | `file_ops.rs` | ✅ 已实现 |
| TOL-006 | 路径遍历防护（符号链接、../ 逃逸） | `file_ops.rs` | ✅ 已实现 |
| TOL-007 | 符号链接逃逸检测 | `file_ops.rs` | ✅ 已实现 |

### 7.2 命令执行工具

| ID | 需求项 | 对应模块 | 状态 |
|----|--------|----------|------|
| TOL-008 | Bash 命令执行（超时/后台/沙箱控制） | `bash.rs` | ✅ 已实现 |
| TOL-009 | Bash 命令安全校验管线（6 类校验） | `bash_validation.rs` | ✅ 已实现 |
| TOL-010 | 破坏性命令警告 | `bash_validation.rs` | ✅ 已实现 |
| TOL-011 | sed 表达式校验 | `bash_validation.rs` | ✅ 已实现 |
| TOL-012 | 可疑路径检测 | `bash_validation.rs` | ✅ 已实现 |
| TOL-013 | 命令语义分类（ReadOnly/Write/Destructive/Network 等） | `bash_validation.rs` | ✅ 已实现 |
| TOL-014 | PowerShell 命令执行 | `tools/lib.rs` | ✅ 已实现 |

### 7.3 任务管理工具

| ID | 需求项 | 对应模块 | 状态 |
|----|--------|----------|------|
| TOL-015 | 创建任务 | `task_registry.rs` + `tools/lib.rs` | ✅ 已实现 |
| TOL-016 | 获取任务详情 | `task_registry.rs` | ✅ 已实现 |
| TOL-017 | 列出所有任务 | `task_registry.rs` | ✅ 已实现 |
| TOL-018 | 停止任务 | `task_registry.rs` | ✅ 已实现 |
| TOL-019 | 更新任务状态 | `task_registry.rs` | ✅ 已实现 |
| TOL-020 | 获取任务输出 | `task_registry.rs` | ✅ 已实现 |
| TOL-021 | 任务包定义与校验 | `task_packet.rs` | ✅ 已实现 |
| TOL-022 | 任务线程安全存储 | `task_registry.rs` | ✅ 已实现 |

### 7.4 团队与定时任务工具

| ID | 需求项 | 对应模块 | 状态 |
|----|--------|----------|------|
| TOL-023 | 创建团队 | `team_cron_registry.rs` | ✅ 已实现 |
| TOL-024 | 删除团队 | `team_cron_registry.rs` | ✅ 已实现 |
| TOL-025 | 创建定时任务 | `team_cron_registry.rs` | ✅ 已实现 |
| TOL-026 | 删除定时任务 | `team_cron_registry.rs` | ✅ 已实现 |
| TOL-027 | 列出定时任务 | `team_cron_registry.rs` | ✅ 已实现 |

### 7.5 网络工具

| ID | 需求项 | 对应模块 | 状态 |
|----|--------|----------|------|
| TOL-028 | Web 页面抓取 | `tools/lib.rs` | ✅ 已实现 |
| TOL-029 | Web 搜索 | `tools/lib.rs` | ✅ 已实现 |
| TOL-030 | 代理配置（HTTP_PROXY / HTTPS_PROXY / NO_PROXY） | `api/http_client.rs` | ✅ 已实现 |

### 7.6 AI 辅助工具

| ID | 需求项 | 对应模块 | 状态 |
|----|--------|----------|------|
| TOL-031 | 技能调用 | `tools/lib.rs` | ✅ 已实现 |
| TOL-032 | 子代理调度 | `tools/lib.rs` | ✅ 已实现 |
| TOL-033 | TodoWrite（任务清单） | `tools/lib.rs` | ✅ 已实现 |
| TOL-034 | ToolSearch（工具搜索） | `tools/lib.rs` | ✅ 已实现 |

### 7.7 交互工具

| ID | 需求项 | 对应模块 | 状态 |
|----|--------|----------|------|
| TOL-035 | AskUserQuestion | `tools/lib.rs` | ⚠️ 桩实现 |
| TOL-036 | SendUserMessage | `tools/lib.rs` | ✅ 已实现 |
| TOL-037 | NotebookEdit | `tools/lib.rs` | ✅ 已实现 |

### 7.8 辅助工具

| ID | 需求项 | 对应模块 | 状态 |
|----|--------|----------|------|
| TOL-038 | PDF 文本提取 | `tools/pdf_extract.rs` | ✅ 已实现 |
| TOL-039 | 车道补全 | `tools/lane_completion.rs` | ✅ 已实现 |
| TOL-040 | Sleep（延迟等待） | `tools/lib.rs` | ✅ 已实现 |
| TOL-041 | REPL 模式 | `tools/lib.rs` | ✅ 已实现 |
| TOL-042 | 计划模式（EnterPlanMode / ExitPlanMode） | `tools/lib.rs` | ✅ 已实现 |
| TOL-043 | StructuredOutput | `tools/lib.rs` | ✅ 已实现 |
| TOL-044 | RemoteTrigger | `tools/lib.rs` | ⚠️ 桩实现 |
| TOL-045 | Config | `tools/lib.rs` | ✅ 已实现 |

---

## 8. 斜杠命令系统

| ID | 需求项 | 对应模块 | 状态 |
|----|--------|----------|------|
| CMD-001 | 命令注册表（`SLASH_COMMAND_SPECS` 共 139 项斜杠命令；本表覆盖其中 30 项核心命令的功能性需求） | `commands/lib.rs` | ✅ 已实现 |
| CMD-002 | 命令来源分类（Builtin / InternalOnly / FeatureGated） | `commands/lib.rs` | ✅ 已实现 |
| CMD-003 | /help 帮助系统 | `commands/lib.rs` | ✅ 已实现 |
| CMD-004 | /doctor 健康检查 | `commands/lib.rs` | ✅ 已实现 |
| CMD-005 | /status 工作空间状态 | `commands/lib.rs` | ✅ 已实现 |
| CMD-006 | /model 模型切换 | `commands/lib.rs` | ✅ 已实现 |
| CMD-007 | /config 配置管理 | `commands/lib.rs` | ✅ 已实现 |
| CMD-008 | /cost 费用统计 | `commands/lib.rs` | ✅ 已实现 |
| CMD-009 | /compact 会话压缩 | `commands/lib.rs` | ✅ 已实现 |
| CMD-010 | /clear 清除历史 | `commands/lib.rs` | ✅ 已实现 |
| CMD-011 | /diff 代码差异 | `commands/lib.rs` | ✅ 已实现 |
| CMD-012 | /init 仓库初始化 | `commands/lib.rs` | ✅ 已实现 |
| CMD-013 | /export 会话导出 | `commands/lib.rs` | ✅ 已实现 |
| CMD-014 | /session 会话管理 | `commands/lib.rs` | ✅ 已实现 |
| CMD-015 | /plugin 插件管理（安装/启用/禁用/卸载） | `commands/lib.rs` | ✅ 已实现 |
| CMD-016 | /mcp MCP 服务器管理 | `commands/lib.rs` | ✅ 已实现 |
| CMD-017 | /agents 代理管理 | `commands/lib.rs` | ✅ 已实现 |
| CMD-018 | /skills 技能管理 | `commands/lib.rs` | ✅ 已实现 |
| CMD-019 | /plan 规划模式 | `commands/lib.rs` | ✅ 已实现 |
| CMD-020 | /review 代码审查 | `commands/lib.rs` | ✅ 已实现 |
| CMD-021 | /tasks 任务管理 | `commands/lib.rs` | ✅ 已实现 |
| CMD-022 | /bughunter 缺陷扫描 | `commands/lib.rs` | ✅ 已实现 |
| CMD-023 | /teleport 符号跳转 | `commands/lib.rs` | ✅ 已实现 |
| CMD-024 | /ultraplan 深度规划 | `commands/lib.rs` | ✅ 已实现 |
| CMD-025 | /commit 创建提交 | `commands/lib.rs` | ✅ 已实现 |
| CMD-026 | /pr 创建 PR | `commands/lib.rs` | ✅ 已实现 |
| CMD-027 | /issue 创建 Issue | `commands/lib.rs` | ✅ 已实现 |
| CMD-028 | /vim Vim 模式切换 | `commands/lib.rs` | ✅ 已实现 |
| CMD-029 | /theme 主题切换 | `commands/lib.rs` | ✅ 已实现 |
| CMD-030 | /version 版本信息 | `commands/lib.rs` | ✅ 已实现 |
| CMD-031 | Skill 斜杠分发（本地/远程） | `commands/lib.rs` | ✅ 已实现 |

---

## 9. MCP 协议

| ID | 需求项 | 对应模块 | 状态 |
|----|--------|----------|------|
| MCP-001 | MCP 客户端多传输协议抽象（Stdio/Sse/Http/WebSocket/Sdk/ManagedProxy） | `mcp_client.rs` | ✅ 已实现 |
| MCP-002 | MCP 服务器进程管理（spawn + initialize 握手） | `mcp_stdio.rs` | ✅ 已实现 |
| MCP-003 | stdio JSON-RPC MCP 服务端实现 | `mcp_server.rs` | ✅ 已实现 |
| MCP-004 | 工具/资源发现 | `mcp_stdio.rs` | ✅ 已实现 |
| MCP-005 | 工具调用 | `mcp_stdio.rs` | ✅ 已实现 |
| MCP-006 | MCP 工具注册表桥接 | `mcp_tool_bridge.rs` | ✅ 已实现 |
| MCP-007 | MCP 服务器连通性跟踪 | `mcp_tool_bridge.rs` | ✅ 已实现 |
| MCP-008 | MCP 生命周期硬化（11 阶段、错误隔离） | `mcp_lifecycle_hardened.rs` | ✅ 已实现 |
| MCP-009 | MCP 工具名标准化（mcp__server__tool 格式） | `mcp.rs` | ✅ 已实现 |
| MCP-010 | 服务器签名去重 | `mcp.rs` | ✅ 已实现 |
| MCP-011 | OAuth 认证支持 | `mcp_stdio.rs` | ✅ 已实现 |
| MCP-012 | 超时控制 | `mcp_stdio.rs` | ✅ 已实现 |
| MCP-013 | 端到端 MCP 运行时集成 | `mcp.rs` + `mcp_client.rs` | ⚠️ 部分实现 |

---

## 10. MCP 服务端能力

| ID | 需求项 | 对应模块 | 状态 |
|----|--------|----------|------|
| MCP-SRV-001 | JSON-RPC initialize 握手（协议版本协商） | `mcp_server.rs` | ✅ 已实现 |
| MCP-SRV-002 | tools/list（列出注册工具描述符） | `mcp_server.rs` | ✅ 已实现 |
| MCP-SRV-003 | tools/call（调用工具并返回结果） | `mcp_server.rs` | ✅ 已实现 |
| MCP-SRV-004 | 换行分帧极简协议 | `mcp_server.rs` | ✅ 已实现 |

---

## 11. LSP 客户端

| ID | 需求项 | 对应模块 | 状态 |
|----|--------|----------|------|
| LSP-001 | diagnostics 诊断操作 | `lsp_client.rs` | ✅ 已实现 |
| LSP-002 | hover 悬停信息 | `lsp_client.rs` | ✅ 已实现 |
| LSP-003 | definition 定义跳转 | `lsp_client.rs` | ✅ 已实现 |
| LSP-004 | references 引用查找 | `lsp_client.rs` | ✅ 已实现 |
| LSP-005 | completion 代码补全 | `lsp_client.rs` | ⚠️ 注册表支持，工具暴露不充分 |
| LSP-006 | symbols 符号查询 | `lsp_client.rs` | ✅ 已实现 |
| LSP-007 | format 格式化 | `lsp_client.rs` | ⚠️ 注册表支持，工具暴露不充分 |
| LSP-008 | 有状态 LSP 注册表 | `lsp_client.rs` | ✅ 已实现 |

---

## 12. 插件系统

| ID | 需求项 | 对应模块 | 状态 |
|----|--------|----------|------|
| PLG-001 | 三种插件类型（Builtin / Bundled / External） | `plugins/lib.rs` | ✅ 已实现 |
| PLG-002 | 插件清单定义（名称/版本/权限/钩子/工具/命令） | `plugins/lib.rs` | ✅ 已实现 |
| PLG-003 | 插件安装（本地路径 / Git URL） | `plugins/lib.rs` | ✅ 已实现 |
| PLG-004 | 插件启用/禁用/卸载/更新 | `plugins/lib.rs` | ✅ 已实现 |
| PLG-005 | 插件注册表持久化 | `plugins/lib.rs` | ✅ 已实现 |
| PLG-006 | 通过子进程执行插件工具（stdin/stdout 通信） | `plugins/lib.rs` | ✅ 已实现 |
| PLG-007 | 环境变量传递上下文 | `plugins/lib.rs` | ✅ 已实现 |
| PLG-008 | 钩子执行（PreToolUse / PostToolUse / PostToolUseFailure） | `plugins/hooks.rs` | ✅ 已实现 |
| PLG-009 | 钩子按 glob 模式匹配文件 | `plugins/hooks.rs` | ✅ 已实现 |
| PLG-010 | 插件状态机（Unconfigured → Validated → Healthy → … → Stopped） | `plugin_lifecycle.rs` | ✅ 已实现 |
| PLG-011 | 测试隔离（EnvLock 独立 HOME 目录） | `plugins/test_isolation.rs` | ✅ 已实现 |

---

## 13. OAuth 认证

| ID | 需求项 | 对应模块 | 状态 |
|----|--------|----------|------|
| OAU-001 | OAuth 2.0 + PKCE 授权码流程 | `oauth.rs` | ✅ 已实现 |
| OAU-002 | PKCE S256 挑战码生成 | `oauth.rs` | ✅ 已实现 |
| OAU-003 | 回调参数解析 | `oauth.rs` | ✅ 已实现 |
| OAU-004 | 令牌持久化与加载 | `oauth.rs` | ✅ 已实现 |
| OAU-005 | 令牌刷新 | `oauth.rs` | ✅ 已实现 |

---

## 14. 沙箱隔离

| ID | 需求项 | 对应模块 | 状态 |
|----|--------|----------|------|
| SBX-001 | 文件系统隔离级别（Off / WorkspaceOnly / AllowList） | `sandbox.rs` | ✅ 已实现 |
| SBX-002 | 沙箱配置管理 | `sandbox.rs` | ✅ 已实现 |
| SBX-003 | Linux 沙箱命令封装（unshare） | `sandbox.rs` | ✅ 已实现 |
| SBX-004 | 容器环境自动检测 | `sandbox.rs` | ✅ 已实现 |
| SBX-005 | 沙箱能力探测（替代二进制存在性检查） | `sandbox.rs` | ✅ 已实现 |

---

## 15. 任务编排与自动化

| ID | 需求项 | 对应模块 | 状态 |
|----|--------|----------|------|
| ORC-001 | 策略引擎（PolicyRule + PolicyCondition + PolicyAction） | `policy_engine.rs` | ✅ 已实现 |
| ORC-002 | 策略条件（And/Or/绿色等级/分支过期/启动阻塞/审查通过） | `policy_engine.rs` | ✅ 已实现 |
| ORC-003 | 策略动作（合并/前向合并/恢复/升级/关闭 lane/清理会话） | `policy_engine.rs` | ✅ 已实现 |
| ORC-004 | 故障恢复配方（6 种场景） | `recovery_recipes.rs` | ✅ 已实现 |
| ORC-005 | 绿色契约（TargetedTests / Package / Workspace / MergeReady） | `green_contract.rs` | ✅ 已实现 |
| ORC-006 | 分支锁定碰撞检测 | `branch_lock.rs` | ✅ 已实现 |
| ORC-007 | Lane 事件系统（21 种事件） | `lane_events.rs` | ✅ 已实现 |

---

## 16. 分支与版本管理

| ID | 需求项 | 对应模块 | 状态 |
|----|--------|----------|------|
| BRN-001 | 基准提交验证（HEAD vs 预期基准） | `stale_base.rs` | ✅ 已实现 |
| BRN-002 | 过期分支检测（Fresh / Stale / Diverged） | `stale_branch.rs` | ✅ 已实现 |
| BRN-003 | 过期分支策略应用（AutoRebase / AutoMergeForward / WarnOnly / Block） | `stale_branch.rs` | ✅ 已实现 |
| BRN-004 | Worker 启动状态机（6 种状态、失败分类） | `worker_boot.rs` | ✅ 已实现 |

---

## 17. 遥测与追踪

| ID | 需求项 | 对应模块 | 状态 |
|----|--------|----------|------|
| TEL-001 | 客户端身份定义（应用名/版本/运行时） | `telemetry/lib.rs` | ✅ 已实现 |
| TEL-002 | API 请求 profile（版本/Beta/额外请求体） | `telemetry/lib.rs` | ✅ 已实现 |
| TEL-003 | HTTP 请求生命周期事件追踪 | `telemetry/lib.rs` | ✅ 已实现 |
| TEL-004 | 会话追踪记录 | `telemetry/lib.rs` | ✅ 已实现 |
| TEL-005 | JSONL 文件持久化遥测接收器 | `telemetry/lib.rs` | ✅ 已实现 |
| TEL-006 | 内存遥测接收器（测试用） | `telemetry/lib.rs` | ✅ 已实现 |

---

## 18. 远程与代理

| ID | 需求项 | 对应模块 | 状态 |
|----|--------|----------|------|
| RMT-001 | 远程会话上下文检测（环境变量） | `remote.rs` | ✅ 已实现 |
| RMT-002 | 上游代理管理（URL/令牌/CA 证书） | `remote.rs` | ✅ 已实现 |
| RMT-003 | NO_PROXY 默认列表 | `remote.rs` | ✅ 已实现 |
| RMT-004 | HTTP 代理支持（HTTP_PROXY / HTTPS_PROXY） | `api/http_client.rs` | ✅ 已实现 |
| RMT-005 | 统一代理 URL 配置（proxy_url 字段） | `api/http_client.rs` | ✅ 已实现 |

---

## 19. 测试与质量

| ID | 需求项 | 对应模块 | 状态 |
|----|--------|----------|------|
| TST-001 | Mock Anthropic 服务（12 种测试场景） | `mock-anthropic-service` | ✅ 已实现 |
| TST-002 | 确定性测试场景（StreamingText / ReadFileRoundtrip / Bash 等） | `mock-anthropic-service` | ✅ 已实现 |
| TST-003 | Mock 一致性校验工具 | `compat-harness` | ✅ 已实现 |
| TST-004 | 上游 TypeScript 清单提取 | `compat-harness` | ✅ 已实现 |
| TST-005 | 无 `#[ignore]` 隐藏失败测试 | — | ✅ 已验证 |
| TST-006 | CI 绿色构建 | — | ❌ 待完善 |

---

## 20. 兼容性检查（Python 端）

| ID | 需求项 | 对应模块 | 状态 |
|----|--------|----------|------|
| CMP-001 | 一致性审计（Python 端口 vs TypeScript 归档） | `parity_audit.py` | ✅ 已实现 |
| CMP-002 | 工作空间清单生成 | `port_manifest.py` | ✅ 已实现 |
| CMP-003 | 命令快照加载（207 条） | `commands.py` | ✅ 已实现 |
| CMP-004 | 工具快照加载（184 条） | `tools.py` | ✅ 已实现 |
| CMP-005 | 28 个子系统归档元数据 | `_archive_helper.py` + JSON 文件 | ✅ 已实现 |

---

## 状态图例

| 符号 | 含义 |
|------|------|
| ✅ 已实现 | 功能已完整实现并通过测试验证 |
| ⚠️ 部分实现 | 基本框架已实现，但细节或端到端集成不完整 |
| ❌ 待完善 | 尚未实现或持续失败 |

---

## 统计总览

| 领域 | 需求项数 | 已实现 | 部分实现 | 待完善 |
|------|----------|--------|----------|--------|
| CLI 入口与用户交互 | 11 | 11 | 0 | 0 |
| 配置管理 | 8 | 8 | 0 | 0 |
| AI 提供商集成 | 14 | 14 | 0 | 0 |
| 会话管理 | 10 | 10 | 0 | 0 |
| 对话引擎 | 11 | 10 | 1 | 0 |
| 权限与安全 | 9 | 9 | 0 | 0 |
| 工具系统 | 45 | 43 | 2 | 0 |
| 斜杠命令系统 | 31 | 31 | 0 | 0 |
| MCP 协议 | 13 | 12 | 1 | 0 |
| MCP 服务端 | 4 | 4 | 0 | 0 |
| LSP 客户端 | 8 | 6 | 2 | 0 |
| 插件系统 | 11 | 11 | 0 | 0 |
| OAuth 认证 | 5 | 5 | 0 | 0 |
| 沙箱隔离 | 5 | 5 | 0 | 0 |
| 任务编排与自动化 | 7 | 7 | 0 | 0 |
| 分支与版本管理 | 4 | 4 | 0 | 0 |
| 遥测与追踪 | 6 | 6 | 0 | 0 |
| 远程与代理 | 5 | 5 | 0 | 0 |
| 测试与质量 | 6 | 5 | 0 | 1 |
| 兼容性检查 | 5 | 5 | 0 | 0 |
| **总计** | **218** | **211** | **6** | **1** |
| **完成率** | — | **96.8%** | **2.8%** | **0.5%** |
<!-- @end-section -->

<!-- @section: related -->
## 相关文档

- [[01-overview|项目架构总览]]
- [[02-rust-crates-analysis|Rust Crate 功能模块分析]]
- [[07-gap-analysis|差距分析]]
<!-- @end-section -->
