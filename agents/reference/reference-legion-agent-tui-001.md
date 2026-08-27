---
id: "reference-legion-agent-tui-001"
title: "Legion Agent TUI 使用"
aliases: ["TUI 使用", "agent tui", "Bubble Tea TUI"]
type: "reference"
category: "agents/reference"
tags: ["agent", "tui", "bubble-tea", "commands", "composer"]
version: "1.2.0"
created: "2026-05-19"
updated: "2026-08-27"
author: "jxncyjq"
status: "published"
parent: "reference-legion-agent-user-manual-001"
children: []
related_docs:
  - id: "reference-legion-agent-session-001"
    relation: "related_to"
    path: "./reference-legion-agent-session-001.md"
  - id: "reference-legion-agent-multi-agent-usage-001"
    relation: "related_to"
    path: "./reference-legion-agent-multi-agent-usage-001.md"
  - id: "reference-legion-agent-tools-001"
    relation: "related_to"
    path: "./reference-legion-agent-tools-001.md"
---

# Legion Agent TUI 使用

## 启动

```powershell
go run ./cmd/agent -- tui --config .\agent.json
```

界面进入后，底部 Composer 输入任务，按 Enter 发送。运行期间底部会显示工作状态和动态进度条；长输出可以在主输出区用鼠标滚轮上下滚动。

## 界面区域

TUI 采用全屏 Bubble Tea 工作台：

| 区域 | 作用 |
|------|------|
| 顶部状态 | 显示 Agent 名称、当前模型 profile 和模型名 |
| 主输出区 | 显示用户输入、thinking、模型输出、命令结果 |
| 右侧 Plan 区 | 预留计划/目标/循环信息展示 |
| Composer | 底部输入框，输入自然语言任务、slash 命令或 `@agent` |
| Footer | 显示当前 Agent、模型、工作状态、快捷键和动态进度条 |

空态时，主区域居中显示 `Legion Agent TUI` 和当前 `role · model`。开始对话后，主输出区切换为对话流。

## 常用按键

| 操作 | 说明 |
|------|------|
| `Enter` | 提交当前输入 |
| `Backspace` | 编辑输入 |
| `Up/Down` | 普通输入时浏览历史；命令提示时选择命令 |
| `/` | 弹出 slash 命令提示 |
| `@` | 弹出已注册 Agent 选择 |
| `Esc` 或 `Ctrl+C` | 退出 TUI |

## 输入模式

直接输入自然语言会交给当前 Agent：

```text
总结一下当前项目的 Agent 能力
```

输入 `/` 会进入 slash 命令选择。可以继续输入过滤，也可以用上下方向键选择命令后按 Enter。

输入 `@` 会进入 Agent 选择。选择后补充任务内容：

```text
@researcher 调研当前 session/cache 实现
```

普通输入状态下，上下方向键用于查看上一条/下一条输入历史。

## Slash 命令

| 命令 | 作用 |
|------|------|
| `/history` | 显示完整对话历史 |
| `/audit` | 显示最近一次任务的审计动作 |
| `/event` | 显示最近一次任务的事件流 |
| `/tasks` | 显示 TaskLedger 活跃任务投影 |
| `/task <id>` | 查看指定 TaskLedger 任务详情 |
| `/handoff <agent> <task_id> <summary>` | 向指定 Agent 追加任务交接消息 |
| `/send <agent> <message>` | 向指定 Agent 发送 inbox 消息 |
| `/inbox` | 查看当前 Agent 未读 inbox |
| `/new` | 创建新会话 |
| `/sessions` | 列出当前 Agent 的会话 |
| `/switch <session_id>` | 切换到指定会话 |
| `/clear-session` | 清空当前会话 |
| `/skill install <source>` | 安装技能 |
| `/skill update <name>` | 更新技能 |
| `/skill uninstall <name>` | 卸载技能 |
| `/mode <manual\|plan\|auto>` | 设置当前会话工作模式 |
| `/cwd <path>` | 设置当前会话工作目录 |

`/skill` 默认操作主 Agent 的 `skills.install_root`。如果要操作子 Agent 的技能目录，可以追加 `--agent <name>`：

```text
/skill install github:owner/writer-style --agent writer
/skill update writer-style --agent writer
/skill uninstall writer-style --agent writer
```

TUI 会使用已注册 Agent 的配置解析目标目录。writer 配置了 `skills.install_root` 时写入 writer 目录；未配置时继承根配置。

### 工作模式与工作目录

```text
/mode manual
/cwd F:\work\demo
```

`/mode` 三档：`auto` 不受限（默认）、`plan` 只给只读工具产出计划、`manual` 把有副作用的工具挡在人工审批之后。模式存在会话上，之后该会话派生的任务都继承它。

`/cwd` 设定会话工作目录，它同时决定工具沙箱根与 `agents.md` 项目根。**一次性可设**：从空设成某目录可以，重设同值可以，改成另一个目录会被拒——会话磁盘状态按写入时的目录归档，改指向会遗弃已有状态。

## 常用操作示例

新建一个干净会话：

```text
/new
```

查看历史 session 并切换：

```text
/sessions
/switch session-1770000000000000000
```

让 researcher 和 writer 围绕同一任务协作：

```text
@researcher --task TASK-20260525-001 调研当前实现
@writer --task TASK-20260525-001 根据 researcher 的结果整理说明
```

人工发送消息给 writer，再让 writer 消费 inbox：

```text
/send writer researcher 已完成调研，请整理成用户文档
@writer --inbox 根据未读消息继续处理
```

查看最近一次模型运行的事件和审计：

```text
/event
/audit
```

## Prompt 与 Thinking

TUI 默认显示用户输入、thinking 和模型输出。`thinking` 优先显示 MaaS/OpenAI-compatible 响应中明确返回的公开 reasoning 字段；如果上游没有返回公开 reasoning 字段，则回落为 Agent 可观测流程摘要。

关闭显示：

```json
{
  "tui": {
    "show_prompt": false,
    "show_thinking": false
  }
}
```

`show_thinking=true` 只展示模型服务明确返回的公开 reasoning 或 Agent 可观测流程摘要，不会凭空生成隐藏思维链。

## 颜色能力

如果终端颜色显示异常，可以降低颜色能力：

```powershell
$env:LEGION_AGENT_TUI_COLOR_PROFILE = "ansi256"
go run ./cmd/agent -- tui --config .\agent.json
```

或在配置中设置：

```json
{
  "tui": {
    "color_profile": "ansi256"
  }
}
```

## 相关文档

- [[reference-legion-agent-session-001|会话连续性]] — 会话命令背后的持久化与工作模式
- [[reference-legion-agent-multi-agent-usage-001|多 Agent 调用]] — `@agent`、`--task`、`--inbox`
- [[reference-legion-agent-tools-001|工具能力]] — `/event` 里看到的工具事件
- [[reference-legion-agent-config-context-001|配置与上下文文件]] — `tui.*` 配置
- [[reference-legion-agent-cli-001|CLI 命令速查]] — 启动参数
