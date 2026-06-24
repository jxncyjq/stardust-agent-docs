---
id: "plans-maas-c70-inference-port-001"
title: "C70 MaasInferenceClient 专项计划"
type: "plan"
category: "plans/maas"
tags: ["plan", "maas", "c70", "inference-port"]
version: "0.1.0"
created: "2026-05-10"
updated: "2026-05-10"
status: "partial"
---

# C70 MaasInferenceClient 专项计划

## 目标

C70 是 Agent、Know、Aegis 调用 MaaS 的唯一稳定端口。它屏蔽 `ModelRouter`、`ModelProvider`、流式代理、过滤、缓存、审计等内部实现。

## 当前状态

C70 当前采用 **Agent 侧客户端先行、MaaS 服务端后续** 的策略：

- `legion/legionAgent` 已有 `MaasInferenceClient` 端口、Recording/Fake 实现、HTTP 客户端和 profile 装配。
- Agent 侧 HTTP 客户端已支持内部 MaaS 协议 `/v1/inference/generate`，也支持 OpenAI-compatible `/chat/completions` 作为过渡直连。
- MaaS 网关服务端内部的 `ModelRouter`、`ModelProvider`、计费、流式代理、过滤、遥测、审计组合尚未开始正式实现。
- 下一阶段继续优先优化 Agent；MaaS 服务端 C70 门面待后续恢复 MaaS M1-M6 时实现。

## 调用场景

| Purpose | 使用方 | 说明 |
|---------|--------|------|
| agent_loop | A01 AgentRuntime | 主循环，通常使用 stream |
| context_summary | A03 ContextCompressor | 摘要与循环检查点 |
| aegis_review | A60 AegisReviewer | 质量评审，默认 Advanced Tier |
| semantic_extract | K13 SemanticExtractor | 知识节点/关系抽取 |
| entity_dedup | K21 EntityDeduplicator | LLM tiebreak |
| conflict_judgment | K51 ConflictDetector | 轻量冲突判断 |

## 实施步骤

1. 定义 `InferenceRequest / InferenceResponse / StreamSink`。
2. 将 role/task/tier/model_hint 转换为 MaaS RequestContext。
3. 内部调用 C14 选择 provider/model。
4. 同步调用走 C01，流式调用走 C50。
5. 可选接入 C16/C17/C19/C40/C62。
6. 提供 Fake/Recording 实现，支撑 Agent/Know 测试。

## 验收

- 上层代码中不出现 provider SDK 依赖。
- C70 失败返回统一错误码。
- 所有调用都有 request_id 和可追踪审计。
