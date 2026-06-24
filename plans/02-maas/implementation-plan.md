---
id: "plans-maas-implementation-001"
title: "MaaS 模型调度层实施计划"
type: "plan"
category: "plans/maas"
tags: ["plan", "maas", "routing", "billing", "observability"]
version: "0.1.0"
created: "2026-05-10"
updated: "2026-05-10"
status: "deferred"
---

# MaaS 模型调度层实施计划

## 目标

将模型选择从 Agent 定义中解耦，统一纳管多厂商、多模型、多预算、多路由策略。

## 执行状态标记

当前暂不启动 MaaS 网关服务端编码，原因是项目近期第一目标为 **优化 Agent**。

本计划保留为后续 MaaS 服务端实施蓝图：

- 当前不新建 `legion-maas` 网关服务端仓库/目录。
- 当前不推进 C01/C03/C14/C30/C50/C62/C70 的服务端实现。
- Agent 侧已完成 C70 HTTP 客户端、MaaS profile、内部 `/v1/inference/generate` 协议调用和 OpenAI-compatible 直连接入，可作为过渡方案。
- 当 Agent 运行时、TUI、配置、上下文和本地开发体验稳定后，再恢复本计划并拆分 MaaS M1-M6 的编码任务。

## 阶段

| 阶段 | 组件 | 交付 |
|------|------|------|
| M1 Provider 基线 | C01/C03/C04 | 接入至少一个 OpenAI-compatible provider，完成模型别名映射 |
| M2 路由基线 | C14/C20-C24 | role/task/cost/health/pinned 策略链 |
| M3 计费配额 | C30-C34 | pricing、billing session、quota、circuit breaker |
| M4 可靠性 | C40/C41/C50 | failover、retry、stream proxy |
| M5 观测审计 | C19/C60/C61/C62/C63 | trace、metrics、audit、cost attribution |
| M6 平台推理端口 | C70 | Agent/Know/Aegis 统一调用入口 |

## Profile

| Profile | 用途 | 必需组件 |
|---------|------|----------|
| minimal | 本地开发和 MVP | C01/C03/C04/C14/C30/C31/C32/C50/C62/C70 |
| standard | SaaS 多租户 | minimal + C05/C06/C10/C11/C12/C13/C33/C40/C41/C61 |
| enterprise | 合规和大规模 | standard + C16/C17/C19/C34/C60/C63 |

## 验收

- 同一请求能输出路由原因、provider、model、token、cost。
- budget 剩余不足时可降级或拒绝。
- provider 失败后能按策略重试或 failover。
- C70 调用不泄漏 provider SDK 细节给上层。
