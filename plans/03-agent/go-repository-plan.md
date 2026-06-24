---
id: "plans-agent-go-repository-001"
title: "Agent 独立 Go 仓库计划"
type: "plan"
category: "plans/agent"
tags: ["plan", "agent", "golang", "repository"]
version: "0.1.0"
created: "2026-05-10"
updated: "2026-05-10"
status: "draft"
---

# Agent 独立 Go 仓库计划

## 目标

Agent 作为独立仓库开发，目录名固定为 `agent`，服务入口固定为 `agent/cmd/main.go`，Go 版本固定为 `1.26.0`。

该仓库只实现 Agent Engine。MaaS、Know、Common 的生产实现通过端口接口接入，不在本仓库内展开完整实现。

## 仓库结构

```text
agent/
  go.mod
  go.sum
  README.md
  Makefile
  cmd/
    main.go
  configs/
    agent.example.yaml
  internal/
    app/
    cli/
    config/
    domain/
    runtime/
    cognitive/
    task/
    tool/
    skill/
    quality/
    approval/
    memory/
    evolution/
    workflow/
    port/
    adapter/
    observability/
    storage/
    tui/
```

Agent 相关文档统一放在主仓库 `docs/agents/legion-agent/` 下，不放入 `legion/legionAgent/docs`。

## go.mod

```go
module github.com/stardust/legion-agent

go 1.26.0
```

实际 module path 可在建仓时按组织名调整，但 `go 1.26.0` 不变。

## 包边界

| 包 | 职责 |
|----|------|
| `cmd` | 唯一进程入口，使用 `run() error` 模式 |
| `internal/app` | 应用装配、生命周期、依赖注入 |
| `internal/cli` | Cobra 命令定义、参数解析、命令到应用服务的转换 |
| `internal/config` | 配置加载和校验 |
| `internal/domain` | 领域类型：Agent、Task、ToolCall、ApprovalTicket、AuditEvent |
| `internal/runtime` | A01 AgentRuntime 主循环 |
| `internal/cognitive` | A00 CognitiveCore、A03 ContextCompressor |
| `internal/task` | A10 TaskScheduler、A11 TaskLock |
| `internal/tool` | A20-A23 工具注册、策略、权限和 guardrails |
| `internal/skill` | A30-A32 技能加载、扫描、安装和运行时注入 |
| `internal/quality` | A60 AegisReviewer、A63 EvalEngine-lite |
| `internal/approval` | A62-lite |
| `internal/memory` | A40-A43 的接口和 Noop，P5 再扩展 |
| `internal/evolution` | A50-A54 信号提取、GEP、蒸馏、固化和进化事件 |
| `internal/workflow` | A70 工作流 DSL 解析、状态机和节点执行 |
| `internal/port` | 外部端口接口：MaaS、Know、EventBus、AuditLog |
| `internal/adapter` | fake/inmemory/http adapter |
| `internal/observability` | 日志、事件、审计实现 |
| `internal/storage` | repository 接口与内存/SQLite 实现 |
| `internal/tui` | Bubble Tea 终端 UI，展示 Agent 运行介绍、任务状态、工具调用和审批暂停 |

## 组件到包映射

| 能力域 | 组件 | 包 |
|--------|------|----|
| L0 核心编排 | A00/A03 | `internal/cognitive` |
| L0 核心运行 | A01/A02 | `internal/runtime` |
| L1 任务管理 | A10/A11/A12 | `internal/task` |
| L2 工具执行 | A20/A21/A22/A23 | `internal/tool` |
| L3 技能系统 | A30/A31/A32 | `internal/skill` |
| L4 记忆系统 | A40/A41/A42/A43 | `internal/memory` |
| L5 学习进化 | A50/A51/A52/A53/A54 | `internal/evolution` |
| L6 质量治理 | A60/A61/A63/A64 | `internal/quality` |
| L6 审批治理 | A62 | `internal/approval` |
| L7 工作流 | A70 | `internal/workflow` |

## Go 约定

- 不使用 `util`、`helper`、`common` 泛包名。
- `cmd/main.go` 只做启动，业务逻辑放入 `internal/app`。
- 库包不调用 `os.Exit`、`log.Fatal`。
- 避免 `init()` 做 I/O、注册或读取环境。
- flags 只在 `cmd` 中定义，并传入配置对象。
- 接口优先放在消费者侧；跨模块稳定端口集中放在 `internal/port`。

## 首轮不做

- 不接真实 MaaS provider。
- 不接真实 Know 服务。
- 不实现完整组织树和工作流 DSL。
- 不实现完整学习进化链路。
- 不引入复杂数据库迁移框架；先以内存和轻量持久化支撑测试。
