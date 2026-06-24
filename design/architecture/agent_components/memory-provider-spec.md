---
id: "spec-agent-memory-provider-040"
title: "MemoryProvider 组件规范"
aliases: ["MemoryProvider规范", "三层记忆入口", "memory-provider-spec"]
type: "spec"
category: "design/architecture/agent_components"
tags: ["component-spec", "agent-engine", "memory", "context"]
version: "0.1.0"
created: "2026-05-09"
updated: "2026-05-09"
author: "jxncyjq"
status: "draft"
parent: "spec-agent-component-registry-000"

component_id: "A40"
layer: "L4"
depends_on:
  - "A42"
optional_deps:
  - "A43"
  - "X01"
conflicts_with: []
required_by:
  - "A00"
  - "A01"
assembly_profiles:
  - standard
  - enterprise
---

<!-- @section: overview -->
# MemoryProvider 组件规范

## 1. 组件定位

`MemoryProvider` 是三层记忆系统的统一入口，向 `CognitiveCore` 提供可注入上下文，向 `AgentRuntime` 提供任务结束后的同步写入。

三层记忆：`WorkingMemory`（单任务草稿）、`EpisodicMemoryStore`（跨任务情景经验）、`CapabilityMemoryStore`（Gene/Capsule 能力资产）。

<!-- @end-section -->

<!-- @section: interface -->
---

## 2. 接口定义

```go
type MemoryProvider interface {
    SystemPromptBlock(ctx context.Context, req MemoryQuery) (string, error)
    Prefetch(ctx context.Context, req MemoryQuery) (MemoryBundle, error)
    SyncAfterTurn(ctx context.Context, event TurnMemoryEvent) error
    SyncAfterTask(ctx context.Context, event TaskMemoryEvent) error
}
```

<!-- @end-section -->

<!-- @section: retrieval -->
---

## 3. 检索策略

| 来源 | 默认数量 | 注入优先级 | 说明 |
|------|----------|------------|------|
| WorkingMemory | 当前任务全部有效片段 | P4 | 草稿、约束、用户偏好 |
| EpisodicMemoryStore | TopK=5 | P5 | 相似任务经验，余弦相似度 > 0.7 |
| CapabilityMemoryStore | TopK=3 | P6 | 与任务类型匹配的 Gene/Capsule |

EmbeddingProvider 缺失时，情景记忆退化为 FTS5 检索，能力记忆按成功率与标签匹配排序。

<!-- @end-section -->

<!-- @section: contracts -->
---

## 4. 行为契约

- 不把未经审核的 Gene 注入系统提示。
- P5/P6 总 token 预算由 `CognitiveCore` 统一裁剪。
- 任务失败也应写入轻量失败记忆，供 GEP 失败扫描使用。
- 记忆注入必须记录来源 ID，便于回溯。

<!-- @end-section -->
