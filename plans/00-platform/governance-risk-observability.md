---
id: "plans-platform-governance-risk-observability-001"
title: "三原则落地计划"
type: "plan"
category: "plans/platform"
tags: ["plan", "observability", "governance", "risk"]
version: "0.1.0"
created: "2026-05-10"
updated: "2026-05-10"
status: "draft"
---

# 三原则落地计划

## 可观测

| 范围 | 落地组件 | 交付 |
|------|----------|------|
| 模型调用 | C19/C60/C61/C62/C70 | request_id、agent_id、model、provider、token、cost、latency |
| Agent 任务 | A02/A10/A11/X02 | 心跳链路、任务状态、锁定记录、异常原因 |
| 工具执行 | A20/A21/A22/A23 | tool_call_log、权限判定、沙盒结果 |
| 知识变更 | K30/K50/X02 | chunk 来源、版本、状态转换、裁决记录 |
| 工作流 | A70/A62/X00 | 节点状态、审批记录、失败重试 |

## 可治理

| 场景 | 机制 |
|------|------|
| 模型选择 | MaaS 路由策略可配置，支持 pinned route |
| 预算超限 | 配额检查、熔断、模型降级、审批 |
| Agent 异常 | HardLoop 暂停、A62 审批恢复、TrustScore 降权 |
| 知识写入 | AI 知识进入 PendingReview，K50 状态机控制进入 Approved |
| 工作流高风险节点 | ApprovalGate、人工暂停、跳过、回退 |

## 可控风险

| 风险 | 默认边界 |
|------|----------|
| 预算失控 | 公司/部门/Agent/模型多级预算 |
| 权限越界 | ExecutionPolicy + PermissionEnforcer + PathGuard |
| 外部抓取风险 | SafeFetcher SSRF 防护、大小限制、redirect 复检 |
| 输出注入 | OutputSanitizer 清理 HTML/YAML/Markdown/MCP 文本 |
| 错误知识传播 | K50 审核状态机、冲突检测、版本回滚 |
| Agent 学坏 | SkillSecurityScanner、Aegis、Eval、EvolutionEvaluator |

