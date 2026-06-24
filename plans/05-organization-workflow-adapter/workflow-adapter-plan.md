---
id: "plans-workflow-adapter-001"
title: "工作流与 Adapter 计划"
type: "plan"
category: "plans/organization-workflow-adapter"
tags: ["plan", "workflow", "adapter", "dsl"]
version: "0.1.0"
created: "2026-05-10"
updated: "2026-05-10"
status: "draft"
---

# 工作流与 Adapter 计划

## 工作流目标

以 DSL 先行，后续再同步可视化画布。第一阶段重点是让流程原语可执行、可暂停、可审计。

## 8 种流程原语

| 原语 | 计划 |
|------|------|
| Sequence | 顺序执行 |
| Parallel | 并行执行和失败策略 |
| Condition | 条件分支 |
| Loop | 循环迭代和退出条件 |
| Join | 汇聚等待 |
| ApprovalGate | 调用 A62 创建审批工单 |
| WaitEvent | 订阅 X00 事件继续 |
| ErrorHandler | 重试、回退、升级 |

## Adapter 目标

控制平面不直接运行 Agent，通过 Adapter 接入不同执行载体。

| Adapter | 优先级 | 说明 |
|---------|--------|------|
| LLM API Adapter | P0 | 通过 C70 调模型 |
| CLI Agent Adapter | P1 | 本地 CLI 编程/运维 Agent |
| MCP Adapter | P1 | 连接外部工具服务 |
| HTTP Webhook Adapter | P2 | 企业内部 API/SaaS |
| Process Adapter | P2 | 本地脚本和批处理 |
| Composite Adapter | P3 | 组合多执行方式 |

## 验收

- DSL 与运行状态可序列化。
- ApprovalGate 能暂停和恢复工作流。
- Adapter 输出能解析 usage 和 run result。
- 每个节点都有输入、输出、状态和审计记录。

