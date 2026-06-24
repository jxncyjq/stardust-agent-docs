---
id: "spec-agent-workflow-engine-070"
title: "WorkflowEngine 组件规范"
aliases: ["WorkflowEngine规范", "工作流DSL执行引擎", "workflow-engine-spec"]
type: "spec"
category: "design/architecture/agent_components"
tags: ["component-spec", "agent-engine", "workflow", "dsl"]
version: "0.1.0"
created: "2026-05-09"
updated: "2026-05-09"
author: "jxncyjq"
status: "draft"
parent: "spec-agent-component-registry-000"

component_id: "A70"
layer: "L7"
depends_on:
  - "A10"
  - "A62"
  - "X00"
optional_deps: []
conflicts_with: []
required_by: []
assembly_profiles:
  - enterprise
---

<!-- @section: overview -->
# WorkflowEngine 组件规范

## 1. 组件定位

`WorkflowEngine` 执行多 Agent 协作 DSL，将流程原语转换为任务调度、审批等待、事件等待和错误处理。

<!-- @end-section -->

<!-- @section: primitives -->
---

## 2. 八种原语

| 原语 | 语义 |
|------|------|
| Sequence | 顺序执行单步骤 |
| Parallel | 并行执行分支 |
| Branch | 条件路由 |
| Loop | 有最大迭代数的受控循环 |
| Join | 等待指定步骤完成 |
| ApprovalGate | 暂停并等待人类审批 |
| EventWait | 等待外部事件 |
| ErrorHandler | 包裹失败处理 |

<!-- @end-section -->

<!-- @section: failure-policy -->
---

## 3. 失败策略

| 策略 | 语义 |
|------|------|
| FailFast | 任意分支失败即取消其余 |
| CollectAll | 等待所有分支并收集结果 |
| Quorum | 至少 N 个分支成功即成功 |

`Loop`、`Join`、`EventWait` 超时均视为失败，交由最近的 `ErrorHandler` 处理。

<!-- @end-section -->

<!-- @section: contracts -->
---

## 4. 行为契约

- 工作流节点不得绕过 TaskScheduler 直接执行 AgentRuntime。
- ApprovalGate 必须通过 ApprovalService 创建工单。
- 每个节点状态变更通过 EventBus 广播。
- 工作流恢复必须基于持久化 checkpoint。

<!-- @end-section -->
