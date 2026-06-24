---
id: "plans-agent-workflow-engine-001"
title: "Agent 工作流引擎计划"
type: "plan"
category: "plans/agent"
tags: ["plan", "agent", "workflow", "dsl"]
version: "0.1.0"
created: "2026-05-10"
updated: "2026-05-10"
status: "draft"
related_docs:
  - path: "../../design/architecture/agent_components/workflow-engine-spec.md"
    relation: "implements"
---

# Agent 工作流引擎计划

## 目标

实现 A70 WorkflowEngine，让多个 Agent 任务可以按 DSL 编排，同时把人工审批、错误处理、并行执行和审计纳入统一运行链。

## MVP DSL 原语

| 原语 | 阶段 | 语义 |
|------|------|------|
| `sequence` | P4 | 按顺序执行多个节点 |
| `parallel` | P4 | 并行执行多个节点，支持 fail_fast / continue |
| `agent_task` | P4 | 创建 Agent Task 并交给 A10/A02 执行 |
| `approval` | P4 | 创建 A62 人工审批节点 |
| `condition` | P4 | 根据上游输出选择分支 |
| `error_handler` | P4 | 捕获失败并执行补偿节点 |
| `wait_event` | P5 | 等待 X00 事件恢复 |
| `subworkflow` | P5 | 调用另一个工作流定义 |

## 包结构

```text
agent/
  internal/
    workflow/
      definition.go
      parser.go
      engine.go
      state.go
      executor.go
      repository.go
```

## 状态机

```text
created -> running -> completed
running -> waiting_approval
running -> failed
waiting_approval -> running
waiting_approval -> canceled
failed -> compensated
```

## 与其他组件关系

| 组件 | 调用关系 |
|------|----------|
| A10 TaskScheduler | `agent_task` 节点创建或派遣任务 |
| A62 ApprovalService | `approval` 节点创建工单并等待决策 |
| X00 EventBus | 发布 workflow_started/node_completed/workflow_failed |
| X02 AuditLog | 记录工作流状态转换和节点结果 |
| A61 TrustScoreManager | 可选影响 agent_task 派遣建议 |

## 验收

- 一个 `sequence` 工作流能连续执行两个 fake agent_task。
- 一个 `parallel` 工作流在 fail_fast 下能取消未完成节点。
- 一个 `approval` 节点能暂停工作流，并在批准后恢复。
- 工作流每个节点都有可追踪审计事件。
