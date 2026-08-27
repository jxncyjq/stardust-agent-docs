---
id: "reference-legion-agent-tools-001"
title: "Legion Agent 工具能力"
aliases: ["Agent tools", "内置工具", "工具调用", "lazy tools"]
type: "reference"
category: "agents/reference"
tags: ["agent", "tools", "taskledger", "message", "browser", "web", "toolauth"]
version: "2.0.0"
created: "2026-05-25"
updated: "2026-08-27"
author: "jxncyjq"
status: "published"
parent: "reference-legion-agent-user-manual-001"
children: []
related_docs:
  - id: "reference-legion-agent-config-context-001"
    relation: "related_to"
    path: "./reference-legion-agent-config-context-001.md"
  - id: "reference-legion-agent-tasks-md-001"
    relation: "related_to"
    path: "./reference-legion-agent-tasks-md-001.md"
  - id: "reference-legion-agent-auth-001"
    relation: "related_to"
    path: "./reference-legion-agent-auth-001.md"
---

# Legion Agent 工具能力

本文说明模型运行时可以调用的内置工具、工具怎么被暴露给模型、以及执行链路上的每道闸。普通用户不需要在 TUI 里手写 `read_file({...})` 这类伪调用——正确方式是用自然语言提任务，由模型按 schema 发起真实工具调用。

<!-- @section: exposure -->
## 工具怎么暴露给模型：lazy 协议

默认（`runtime.lazy_tools=true`，默认开）模型每轮**只看到两个元工具**，而不是全部工具的完整 schema：

| 元工具 | 作用 |
|--------|------|
| `load_capabilities` | 按名字加载能力全文：工具的参数 schema，或技能的完整说明。一次最多 5 个 |
| `call_tool` | 按名字调用一个真实工具，参数用 `arguments_json` 字符串传 |

可用能力的名字与一句话说明放在 prompt 的 `<available_capabilities>` 清单里，模型先看清单、按需 `load_capabilities` 加载、再 `call_tool` 调用。这样一次不需要工具的普通对话只付两个小 schema 的开销，而不是全量工具 schema（约 1800 token）。

元工具**永远常驻**，不参与禁用清单。把 `runtime.lazy_tools` 设为 `false` 会回退到「每轮下发完整原生工具 schema」的旧行为（安全回滚开关）。

<!-- @end-section -->

<!-- @section: loop -->
## 工具循环：多轮消息数组

```text
Descriptor + Handler 注册
  -> Runtime 组装 InferenceTool（lazy 下是两个元工具）
  -> 模型返回 tool_calls
  -> Registry.Execute 真正执行
  -> 结果作为独立的 tool 消息追加进会话数组
  -> 下一轮推理，直到模型给出最终回答或触发上限
```

会话是**追加式多轮消息数组**，不是「把工具结果拼回同一条 user 消息」。这条是有事故背景的硬约束：早期实现每轮重发一条去重后的 user 消息，模型看到的 prompt 逐轮字节相同，于是反复发同一个调用——一次任务 554 秒里读了同一个文件 152 次。保持每轮独立，重复才对模型可见。

循环上限：

| 限制 | 值 | 触发后 |
|------|-----|--------|
| `runtime.max_tool_rounds` | 默认 4 | 超轮数仍要工具，任务失败并提示 |
| 单工具调用次数上限 `toolLoopCap` | 30（按工具名计，忽略参数） | 发 `tool_loop_broken` 事件，停止工具循环 |
| `runtime.compact_token_threshold` | 默认 0（关闭） | 超阈值压缩历史，单任务最多 3 次；首条消息（稳定缓存前缀）永不压缩 |

单工具计数是**任务级共享预算**：插件通过宿主 `call_tool` 发起的调用记在同一个计数器上，不能绕过。

<!-- @end-section -->

<!-- @section: pipeline -->
## Registry 执行管线

`Registry.Execute` 是工具安全边界的中心，一次调用依次过：

1. 查 handler。
2. 按 `Descriptor.InputSchema` 校验必填与基本类型。
3. 角色权限（`role:tool` 允许表，当前生产工具都在 `developer:*` 域）。
4. 执行策略：高风险且未自动允许则拒。
5. guardrails（前置）。
6. 按工具 `Timeout` 建执行上下文。
7. 调真实 handler。
8. guardrails（后置）。
9. 输出脱敏 + 控制字符清理。
10. 写审计。

「模型说自己调用了工具」和「工具真的被执行」是两件事：只有返回结构化 `tool_calls` 并通过 Registry 执行成功，才算真实调用。

<!-- @end-section -->

<!-- @section: file-tools -->
## 工作区文件工具

| 工具 | 参数 | 说明 |
|------|------|------|
| `read_file` | `path`、`offset` 可选、`limit` 可选 | 读工作区内 UTF-8 文本。**长文件分页返回**，用上一页末尾给出的 offset 继续读 |
| `write_file` | `path`、`content`、`overwrite` 可选 | 写文件；已存在且未显式 `overwrite=true` 时失败（Sensitive） |
| `search_content` | `pattern`、`directory` 可选、`file_types` 可选 | 工作区内文本搜索 |
| `list_files` | `directory` 可选 | 列目录 |

路径可以是相对工作区根的路径，也可以是仍在根内的绝对路径；越界路径被拒。工作区根（ToolRoot）按 **会话 `working_dir` 优先、否则回退配置根** 解析，`agents.md` 分层注入用的 projectRoot 与它同源。

`write_file` 成功写出的相对路径会被任务捕获，出现在 `GET /v1/tasks/{id}/result` 的 `generated_files` 里。

<!-- @end-section -->

<!-- @section: other-tools -->
## 其余内置工具

### 任务台账（TaskLedger）

| 工具 | 说明 |
|------|------|
| `create_task` / `claim_task` / `update_task` | 建/认领/更新共享任务 |
| `append_task_message` | 向任务追加协作消息 |
| `read_task` | 读单个任务详情 |
| `rebuild_tasks` | 从结构化状态重建 `tasks.md` 投影 |

### Agent 消息

| 工具 | 说明 |
|------|------|
| `send_message` | 向目标 Agent 发消息，支持 `task_id`、`type`、`summary`、`artifact` |
| `read_messages` | 读当前 Agent 消息，可按状态过滤并标记已读 |

TUI 的 `/send`、`/inbox`、`@agent --inbox` 与 HTTP `/v1/agents/{id}/messages` 读写的是同一份数据。

### 网络

| 工具 | 说明 | 启用条件 |
|------|------|----------|
| `fetch_url` | 抓取 URL 内容（Sensitive） | `web.enabled` |
| `web_extract` | 抓网页正文（Sensitive） | `web.enabled` |
| `web_search` | 经 SearXNG 搜索（Sensitive） | `web.searxng_url` 非空，否则**根本不注册** |

`web` 配置还控制私网访问（`allow_private_hosts`）、超时、响应上限、域名 allowlist、搜索引擎与默认条数。

### 内置浏览器

| 工具 | 说明 |
|------|------|
| `browser_open` | 打开 URL，返回 `session_id` 和带稳定 ref 的可访问性树（Sensitive） |
| `browser_read` | 读当前页可访问性树，只读 |
| `browser_click` | 按 ref 点击（Sensitive） |
| `browser_type` | 按 ref 输入文本，`submit=true` 时回车（Sensitive） |
| `browser_close` | 关闭会话释放上下文 |

默认关闭（`browser.enabled=false`），开启需要运行环境有可用 Chromium；`browser.bin_path` 可指向系统 Chrome/Edge 绕开自动下载。观测超过 `snapshot_rune_threshold` 会走三级降级：全文落盘去重 + 任务导向抽取 + 按行截断，模型再用 `read_file` 翻页读落盘快照。

### 编排者专属

| 工具 | 说明 |
|------|------|
| `delegate_task` | 委派子任务给其他 Agent（Sensitive），子角色 `leaf`（默认，不能再委派）或 `orchestrator`（可嵌套至深度上限） |
| `moa_consult` | 向多个模型发起 MoA 咨询（Sensitive，高成本） |
| `session_search` | 跨会话检索历史 |

<!-- @end-section -->

<!-- @section: agent-diff -->
## 主 Agent 与子 Agent 的工具差异

| 运行时 | 工具集 |
|--------|--------|
| 默认（根编排者） | 读写文件 + TaskLedger + AgentMessage + web +（开启时）browser + `delegate_task` + `moa_consult` + `session_search` |
| 子 Agent（worker） | 读写文件 + TaskLedger + AgentMessage + web +（开启时）browser |

子 Agent 现在**有写文件能力**（读写工作区 registry），旧版「子 Agent 只读」的说法已过时。

差异只在三个编排者层能力上，且是刻意设计（有测试锁定）：

- `delegate_task`：worker 再派 worker 会让委派树无界；
- `session_search`：会跨公司/跨 Agent 读历史，越过 worker 被限定的沙箱与任务简报；
- `moa_consult`：会绕过该 worker 被指派的模型 profile 并放大成本。

在此之上，`runtime.disabled_tools` 可以按 Agent 再做减法，详见 [[reference-legion-agent-auth-001|鉴权与授权参考]]。

<!-- @end-section -->

<!-- @section: modes -->
## 工作模式与审批

| 模式 | 工具行为 |
|------|----------|
| `auto` | 不受限 |
| `plan` | 只给只读工具，产出计划无副作用 |
| `manual` | Sensitive 工具挡在人工审批后；审批依赖未装配时运行时直接失败 |

Sensitive 工具当前是：`write_file`、`fetch_url`、`web_search`、`web_extract`、`delegate_task`、`moa_consult`、`browser_open` / `browser_click` / `browser_type`。

审批通过 `GET /v1/approvals` 拉取、`POST /v1/tasks/{taskID}/approvals/{ticketID}` 裁决，超时（默认 300 秒）自动按拒绝返回给模型。

<!-- @end-section -->

<!-- @section: troubleshoot -->
## 常见排查

| 现象 | 判断方向 |
|------|----------|
| 模型输出 `search_content({...})` 文本 | 伪工具调用：模型没返回结构化 `tool_calls`，先查模型兼容性 |
| 同一工具被反复调用后中断 | 触发 `toolLoopCap`（30 次/工具），看 `tool_loop_broken` 事件与前因 |
| 工具结果没进最终回答 | 查 `runtime.max_tool_rounds` 是否用尽 |
| 工具压根不在清单里 | 该工具未启用（`web.searxng_url` 为空 / `browser.enabled=false`），或被 `runtime.disabled_tools` 禁了 |
| `permission denied` | 工具没登记进角色允许表（新增工具必须同步登记，enforcer 先于策略生效） |
| 配置里写了工具名但装配失败 | 名字不在 `toolauth` gateable 目录里 |
| 路径被拒 | 路径必须落在该任务 ToolRoot（会话 `working_dir` 优先）内 |
| `generated_files` 为空 | `write_file` 遇到已存在文件且没传 `overwrite=true`，被正确跳过 |
| TUI 看不到工具过程 | 用 `/event` 看事件；也可订阅 `/v1/events` |

<!-- @end-section -->

## 相关文档

- [[reference-legion-agent-auth-001|鉴权与授权参考]] — 角色允许表、`disabled_tools`、审批闸门
- [[reference-legion-agent-config-context-001|配置与上下文文件]] — `runtime` / `web` / `browser` 配置块
- [[reference-legion-agent-tasks-md-001|tasks.md 协作规范]] — TaskLedger 协议
- [[reference-legion-agent-backend-api-001|后端系统调用参考]] — 审批与事件端点
- [[reference-legion-agent-troubleshooting-001|常见问题排查]]
