---
id: "plans-platform-roadmap-001"
title: "Legion 全局路线图"
type: "plan"
category: "plans/platform"
tags: ["plan", "roadmap", "milestone"]
version: "0.1.0"
created: "2026-05-10"
updated: "2026-05-10"
status: "draft"
---

# Legion 全局路线图

## 目标

Legion 的产品目标是让用户“管理一支智能体团队”，而不是使用单点 AI 工具。路线图必须同时推进三条主线：

1. 模型资源统一调度：MaaS。
2. 智能体可运行、可学习、可治理：Agent Engine。
3. 人机共建知识基座：Know / LLM Wiki。

## 阶段计划

| 阶段 | 时间口径 | 关键结果 | 依赖 |
|------|----------|----------|------|
| P0 架构基线 | 第 1 轮 | A/C/K/X 组件边界稳定；所有模块有最小 Profile | 已完成设计文档初稿 |
| P1 平台底座 | 第 2 轮 | Common 与 MaaS minimal 可独立启动；C70 可被上层调用 | X00-X05、C01/C03/C04/C14/C50/C62/C70 |
| P2 Agent MVP | 第 3 轮 | 单 Agent 完整心跳：最小组织身份、锁任务、调 MaaS、执行工具、写审计、产出结果、A62-lite 异常恢复 | P1 |
| P3 Know MVP | 第 4 轮 | 代码/文档进入知识图谱；Agent 可通过 K61-lite 查询知识；K50-lite + KnowledgeReview 跑通最小审核链 | P1/P2 |
| P4 协作控制平面 | 第 5 轮 | 公司/部门/Agent/项目组完整模型、通讯、A62-full、工作流 DSL 跑通 | P2/P3 |
| P5 学习进化 | 第 6 轮 | 情景记忆、能力记忆、GEP、Aegis/Eval/退化检测闭环 | P2/P4 |
| P6 场景模板 | 第 7 轮 | 一个标杆场景完整产品化；其余场景提供骨架模板 | P4/P5 |

## MVP 切片

第一版 MVP 不追求完整产品，而追求一条可演示闭环：

```text
创建公司和 Agent
→ 分配任务
→ Agent 通过 C70 调用 MaaS
→ 执行工具
→ 写入任务审计
→ 产出结果
→ Aegis 审核
→ 关键总结进入 Know 待审核
→ 人类审批后成为可检索知识
```

该闭环依赖四个前置 lite 能力，必须在 P2/P3 完成：

| 能力 | 阶段 | 说明 |
|------|------|------|
| Company/Agent-lite | P2 | 只保留 company_id、agent_id、role、budget、permission、status，支撑任务身份和审计 |
| A62-lite | P2 | 只支持创建工单、记录审批结果、恢复 suspended task |
| K50-lite | P3 | 只支持 PendingReview/Approved/Rejected 和审计，不做完整冲突裁决 |
| K61-lite | P3 | 只支持 search_knowledge/get_node，供 Agent 查询知识 |

## 非目标

- P1-P3 不做完整交易市场，只保留技能/模型/知识资产的注册边界。
- P1-P3 不做复杂可视化画布，只先落 DSL 与运行时。
- P1-P3 不做跨公司知识合并，只保证隔离模型正确。
- P1-P3 不做完整审批工作台，只做 `A62-lite` 支撑 MVP 闭环。
- P1-P3 不做完整组织管理 UI，只做 `Company/Agent-lite` 数据模型。
