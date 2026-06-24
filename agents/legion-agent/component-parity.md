---
id: "agent-component-parity-001"
title: "Agent Component Parity"
type: "reference"
category: "backend/agent"
tags: ["agent", "parity", "components"]
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

# Agent Component Parity

P10 的目标是把已经可运行的 Legion Agent 从 MVP 行为推进到 `agent_components` 设计规格的组件对齐状态。组件状态由 `legion/legionAgent/internal/compat/testdata/component-parity.json` 记录，并由 `TestComponentParityManifest` 作为门禁检查。

## 状态口径

| 状态 | 含义 |
|------|------|
| `done` | 已达到当前规格验收，并有测试守护 |
| `partial` | 已有 MVP 或局部实现，但 P10 仍要补齐高级行为 |
| `planned` | 已列入 P10，尚未开始代码实现 |
| `deferred` | 明确延后，不阻塞 P10 验收 |

## P2-P9 MVP 基线

| 范围 | 已完成能力 |
|------|------------|
| A00-A03 | 上下文组装、阈值型压缩检测、任务执行上下文注入 |
| A10-A12 | 任务状态机、任务锁、后台调度 |
| A20-A23 | 工具注册、基础策略、权限与路径保护、最小 guardrail |
| A30-A32 | 技能加载、安全扫描、安装元数据 |
| A40-A43 | 工作记忆、情景记忆、能力资产检索与冻结 |
| A50-A54 | 学习信号、GEP MVP、演化事件写入 |
| A60-A64 | 质量评审、信任分、Eval、退化治理 |
| A70 | sequence、parallel、approval、condition、wait_event、subworkflow |
| C70/X00-X05 | MaaS 推理端口、事件、审计、路径保护、输出净化、服务 API |

## P10 Spec Parity 范围

| P10 任务 | 组件 | 对齐目标 |
|----------|------|----------|
| AG-P10-001 | Registry/Compat | 建立组件实现状态、测试覆盖、降级策略门禁 |
| AG-P10-002 | A00/A03/C70 | 四层上下文压缩、checkpoint、无 LLM 降级已完成 |
| AG-P10-003 | A20-A23/X04/X05 | descriptor、schema、policy、permission、guardrails、timeout、sanitize 全链路已完成 |
| AG-P10-004 | A30-A32 | 技能 candidate/quarantine/enable/disable/reject 生命周期与审计已完成 |
| AG-P10-005 | A52-A54/A43/X02 | Gene 六元组、固化门控、immutable sealed evolution log 已完成 |
| AG-P10-006 | A60-A64/X00/X02 | Eval、trust、degradation 历史化与诊断摘要已完成 |
| AG-P10-007 | A70/A10/A62/X00 | loop、join、quorum、timeout 与 workflow submit/resume API 已完成 |

## Deferred 口径

当前 P10 不把外部平台级集成作为强制范围，例如外部技能市场、生产级 Prometheus exporter、跨租户 RBAC 和完整 OpenAPI 发布流程。这些能力可以在 P11 以后独立规划；P10 只要求本仓库内组件行为、测试覆盖和降级策略可追踪。

## P10 完成状态

| 组件组 | 完成口径 |
|--------|----------|
| A03 | 四层压缩、checkpoint、C70 缺失降级均有测试覆盖 |
| A20-A23 | 工具 descriptor、schema、policy、permission、guardrails、timeout、sanitize 形成固定流水线 |
| A30-A32 | skill candidate/quarantine/enabled/disabled/rejected 生命周期和审计闭环完成 |
| A52-A54 | Gene 六元组、solidify gate、sealed event log 和 capability asset 写入完成 |
| A60-A64 | eval 趋势、trust snapshot、degradation decision 历史化，diagnostics 只输出摘要 |
| A70 | loop、join、quorum、timeout 和 workflow submit/resume/get API 完成 |

## P11 建议

P11 可以从“组件可运行”推进到“平台可集成”：

| 方向 | 建议内容 |
|------|----------|
| API 契约 | 发布完整 OpenAPI、错误码矩阵、客户端兼容测试 |
| 观测集成 | 增加 Prometheus exporter、trace exporter、外部日志字段约定 |
| 多租户安全 | 补齐 tenant/company 边界、RBAC、审计查询权限 |
| 外部生态 | 对接外部 Skill registry、模型供应商 profile、工具市场签名校验 |
| 数据运维 | schema migration 分版本、数据保留策略、质量历史归档 |
