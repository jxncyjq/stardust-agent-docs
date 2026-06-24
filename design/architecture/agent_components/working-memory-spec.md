---
id: "spec-agent-working-memory-041"
title: "WorkingMemory 组件规范"
aliases: ["WorkingMemory规范", "工作记忆", "working-memory-spec"]
type: "spec"
category: "design/architecture/agent_components"
tags: ["component-spec", "agent-engine", "memory", "working-memory"]
version: "0.1.0"
created: "2026-05-09"
updated: "2026-05-09"
author: "jxncyjq"
status: "draft"
parent: "spec-agent-component-registry-000"

component_id: "A41"
layer: "L4"
depends_on: []
optional_deps: []
conflicts_with: []
required_by:
  - "A40"
assembly_profiles:
  - standard
  - enterprise
---

<!-- @section: overview -->
# WorkingMemory 组件规范

## 1. 组件定位

`WorkingMemory` 是单次任务内的短生命周期草稿本，用于保存中间假设、用户偏好、已验证约束和临时计划。任务结束后由 `MemoryProvider` 选择性蒸馏为情景记忆。

<!-- @end-section -->

<!-- @section: limits -->
---

## 2. 容量与生命周期

| 项 | 默认值 |
|----|--------|
| 最大容量 | 64KB / task |
| 写入模式 | append-only |
| 生命周期 | task start → task terminal |
| 淘汰策略 | 优先淘汰低优先级草稿 |

<!-- @end-section -->

<!-- @section: interface -->
---

## 3. 接口定义

```go
type WorkingMemory interface {
    Append(ctx context.Context, item WorkingMemoryItem) error
    List(ctx context.Context, taskID string) ([]WorkingMemoryItem, error)
    Compact(ctx context.Context, taskID string) (CompactResult, error)
    Clear(ctx context.Context, taskID string) error
}
```

<!-- @end-section -->

<!-- @section: contracts -->
---

## 4. 行为契约

- 不保存密钥、token、密码等敏感值。
- `Compact` 只压缩草稿，不改变审计日志。
- 写入项必须带 `source`，区分用户、工具、LLM、系统。

<!-- @end-section -->
