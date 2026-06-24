---
id: "agent-persona-files-001"
title: "Legion Agent 运行时上下文文件"
type: "guide"
category: "backend/agent"
tags: ["agent", "persona", "context", "runtime"]
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

# Legion Agent 运行时上下文文件

Legion Agent 现在支持在 `agent run` 时加载 `AGENTS.md`、`SOUL.md`、`TOOLS.md`、`USER.md`、`MEMORY.md`，用于组装稳定的身份、项目规则、工具策略和记忆快照。

## 文件分工

| 文件 | 默认路径 | Prompt slot | 定位 |
|------|----------|-------------|------|
| `AGENTS.md` | `AGENTS.md` | Project instructions | 当前项目规则、构建验证、文档目标 |
| `SOUL.md` | `configs/persona/SOUL.md` | Agent identity | Agent 身份、人格、价值观、行为风格 |
| `TOOLS.md` | `configs/persona/TOOLS.md` | Tool policy | 工具使用原则、风险边界、审批触发 |
| `USER.md` | `configs/persona/USER.md` | User profile | 用户偏好、长期协作习惯 |
| `MEMORY.md` | `configs/persona/MEMORY.md` | Agent memory | Agent 长期经验和可复用记忆摘要 |

加载顺序固定为 `SOUL.md`、`AGENTS.md`、`TOOLS.md`、`USER.md`、`MEMORY.md`。缺失文件会降级为空，不阻断运行。

`TOOLS.md` 还承担伪工具调用约束：当 Runtime 没有真实工具执行协议时，模型不得把 `search_content(...)`、`read_file(...)`、`run_shell(...)` 这类调用当作普通文本输出。输出净化层会兜底替换这类内容，避免用户误以为 Agent 已经搜索或执行过工具。

## 配置与关闭

配置位于 `context_files`：

```json
{
  "context_files": {
    "enabled": true,
    "root": ".",
    "agents_path": "AGENTS.md",
    "soul_path": "configs/persona/SOUL.md",
    "tools_path": "configs/persona/TOOLS.md",
    "user_path": "configs/persona/USER.md",
    "memory_path": "configs/persona/MEMORY.md",
    "max_file_chars": 20000
  }
}
```

可通过两种方式关闭：

```powershell
go run ./cmd -- run --plain --config .\agent.json --prompt "..." --no-context-files
```

或设置：

```json
{ "context_files": { "enabled": false } }
```

## 安全边界

- 所有文件路径必须位于 `context_files.root` 内。
- 单文件按 `max_file_chars` 限制，超出后保留 head/tail 并加入截断标记。
- 加载器会阻断明显 prompt injection、泄密诱导、凭证模式和不可见方向控制字符。
- 被阻断的文件不会注入原文，只会渲染 blocked marker，方便排查。
- `AGENTS.md` 只能追加项目规则，不能覆盖系统安全策略。

## docs 与 memory 目录

`workspace.docs_root` 和 `workspace.memory_root` 约定运行时写入边界：

| 目录 | 用途 |
|------|------|
| `docs/` | Agent 输出给人阅读的经验文档、报告、runbook、决策记录 |
| `memory/` | Agent 输出给后续检索和注入使用的记忆材料 |

`configs/persona/USER.md` 和 `configs/persona/MEMORY.md` 是启动时注入的快照；`memory/` 保存更细粒度的材料，后续可以人工归并回快照文件。
