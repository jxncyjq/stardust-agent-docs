---
id: "plans-agent-cli-tui-001"
title: "Agent CLI 与终端 UI 计划"
type: "plan"
category: "plans/agent"
tags: ["plan", "agent", "cli", "tui", "bubble-tea"]
version: "0.1.0"
created: "2026-05-10"
updated: "2026-05-10"
status: "draft"
---

# Agent CLI 与终端 UI 计划

## 目标

Agent 独立仓库需要提供一个可演示、可调试、可运维的终端入口。CLI 负责命令、参数和脚本化调用；终端 UI 负责展示 Agent 运行介绍、任务执行过程、工具调用、审批暂停、质量检查和审计事件。

终端 UI 选型固定为 `Bubble Tea`，配套使用 `Bubbles` 和 `Lip Gloss`。命令入口推荐使用 `Cobra`。

## 技术选型

| 组件 | 用途 | 说明 |
|------|------|------|
| `Cobra` | CLI 命令框架 | 管理 `run`、`inspect`、`replay`、`doctor` 等命令 |
| `Bubble Tea` | TUI 状态机 | 展示 Agent 运行过程，处理键盘交互和刷新 |
| `Bubbles` | TUI 组件库 | 使用 spinner、progress、viewport、table 等基础组件 |
| `Lip Gloss` | TUI 样式 | 统一颜色、边框、布局和状态标签 |

## 命令规划

```text
agent run --task-id <task_id>
agent run --demo
agent inspect task --task-id <task_id>
agent inspect agent --agent-id <agent_id>
agent replay --run-id <run_id>
agent doctor
```

## TUI 展示内容

`agent run` 默认进入 Bubble Tea TUI，展示以下区域：

| 区域 | 内容 |
|------|------|
| Agent 概览 | Agent 名称、角色、当前状态、运行轮次、模型端口 |
| 任务状态 | Task/TaskRun 状态机、锁状态、开始时间、耗时 |
| 推理过程 | C70 MaaS 推理请求摘要、token 用量、延迟、错误 |
| 工具调用 | 工具名、输入摘要、权限判断、执行结果、耗时 |
| 质量检查 | AegisReviewer 结论、风险等级、是否需要审批 |
| 审批暂停 | A62-lite 工单状态、恢复入口、拒绝原因 |
| 审计事件 | EventBus/AuditLog 最近事件流 |

## 包结构补充

```text
agent/
  internal/
    cli/
      command.go
      run.go
      inspect.go
      replay.go
      doctor.go
    tui/
      model.go
      update.go
      view.go
      style.go
      event.go
```

包边界：

| 包 | 职责 |
|----|------|
| `internal/cli` | Cobra 命令定义、参数解析、命令到 app service 的转换 |
| `internal/tui` | Bubble Tea model/update/view、TUI 事件订阅和渲染 |
| `internal/app` | 暴露 CLI/TUI 可调用的应用服务，不依赖 Bubble Tea |
| `internal/runtime` | 只发布运行事件，不直接处理终端显示 |

## MVP 分期

### P2 Scaffold

- `agent run --demo` 可启动假任务。
- Bubble Tea TUI 展示 Agent 概览、任务状态、工具调用和审计事件。
- `agent doctor` 检查配置、工作目录、外部端口 fake adapter。

### P3 可观测运行

- TUI 接入真实 TaskRun 事件流。
- 展示 C70 推理端口调用摘要。
- 展示 A62-lite 审批暂停和恢复状态。

### P4 运维调试

- `agent inspect task` 查看任务详情。
- `agent replay` 基于审计事件重放运行过程。
- TUI 增加过滤、暂停滚动、错误详情展开。

## 实现约束

- TUI 只消费领域事件和查询接口，不直接访问 repository。
- 终端展示不得成为业务状态来源；业务状态以 runtime、task、approval、audit 为准。
- 所有 TUI 状态都可从 TaskRun 和 AuditEvent 重建。
- 非交互环境下，`agent run --task-id` 支持 `--plain` 输出结构化日志，便于 CI 和脚本使用。
