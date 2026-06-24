---
id: "agent-agents-md-001"
title: "AGENTS.md 使用说明"
type: "guide"
category: "backend/agent"
tags: ["agent", "agents-md", "context"]
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

# AGENTS.md 使用说明

`AGENTS.md` 是给 Codex、Claude Code、OMC 或其他 coding agent 读取的仓库级协作指令文件。它不是业务代码配置，也不会被 `legionAgent` 程序运行时读取；它用于告诉 AI 助手在这个仓库里如何工作、看哪些文档、遵守哪些约定。

## 当前文件位置

当前仓库根目录已有：

```text
F:\source\stardust\Legion\AGENTS.md
```

可以用 PowerShell 查看：

```powershell
Get-Content -Path F:\source\stardust\Legion\AGENTS.md
```

也可以在仓库根目录执行：

```powershell
Get-Content -Path .\AGENTS.md
```

查找所有 `AGENTS.md`：

```powershell
Get-ChildItem -Path F:\source\stardust\Legion -Filter AGENTS.md -Recurse -Force
```

## 放在哪里

推荐放置规则：

| 位置 | 作用范围 | 适用场景 |
|------|----------|----------|
| 仓库根目录 `AGENTS.md` | 整个仓库默认指令 | 通用协作规则、语言要求、文档地图、全局工程约定 |
| 子项目目录 `AGENTS.md` | 该子项目覆盖或补充根目录指令 | 多仓库/多项目混放，例如 `graphify`、`DeepSeek-TUI`、`legion/legionAgent` |
| 用户级目录 | 所有本地项目默认指令 | 个人偏好，不建议写项目专属内容 |

在 Legion 仓库中，如果要给 `legion/legionAgent` 单独定义规则，可以新增：

```text
F:\source\stardust\Legion\legion\legionAgent\AGENTS.md
```

但如果规则适用于整个 `F:\source\stardust\Legion` 工作区，应继续维护根目录：

```text
F:\source\stardust\Legion\AGENTS.md
```

## 文件内容建议

`AGENTS.md` 适合放：

- 固定交流语言，例如“永远使用中文交流”。
- 仓库结构说明和文档入口。
- 构建、测试、格式化、发布命令。
- 代码风格、测试要求、提交要求。
- 禁止事项，例如不要删除用户改动、不要绕过测试。
- 多 agent 协作规则，例如先规划、再实现、验证后再声明完成。

不适合放：

- 真实密钥、token、账号信息。
- 运行时配置，例如 MaaS API key、SQLite 路径，这些应放配置文件或环境变量。
- 易频繁变化的任务进度，任务进度应放 `docs/plans/...`。

## 当前注意事项

当前根目录 `AGENTS.md` 中引用了：

```text
INSTRUCTIONS/
```

但当前 `F:\source\stardust\Legion` 根目录下没有 `INSTRUCTIONS` 目录。因此这些链接现在不可直接打开。后续可以二选一处理：

1. 恢复或创建 `INSTRUCTIONS/` 目录，把仓库规则、开发规范、测试规范放进去。
2. 修改根目录 `AGENTS.md`，把文档地图指向当前实际存在的 `docs/` 路径。

## 推荐维护方式

短期建议：

- 根目录 `AGENTS.md` 继续作为全局 AI 协作入口。
- Agent 项目专项说明放到 `docs/agents/legion-agent/`。
- 开发计划继续放到 `docs/plans/03-agent/`。

如果后续 `legion/legionAgent` 独立成真正单仓库，再在该目录下创建独立的 `AGENTS.md`，写入只属于 Agent 仓库的构建、测试和协作规则。
