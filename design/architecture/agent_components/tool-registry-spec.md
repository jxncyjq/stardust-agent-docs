---
id: "spec-agent-tool-registry-020"
title: "ToolRegistry 组件规范"
aliases: ["ToolRegistry规范", "工具注册表", "tool-registry-spec"]
type: "spec"
category: "design/architecture/agent_components"
tags: ["component-spec", "agent-engine", "tool", "mcp"]
version: "0.1.0"
created: "2026-05-09"
updated: "2026-05-09"
author: "jxncyjq"
status: "draft"
parent: "spec-agent-component-registry-000"

component_id: "A20"
layer: "L2"
depends_on: []
optional_deps:
  - "A21"
  - "A22"
  - "A23"
conflicts_with: []
required_by:
  - "A01"
assembly_profiles:
  - minimal
  - standard
  - enterprise
---

<!-- @section: overview -->
# ToolRegistry 组件规范

## 1. 组件定位

`ToolRegistry` 管理 Agent 可调用工具：本地工具、MCP 代理工具、平台内置工具与技能暴露工具。它负责注册、发现、Schema 校验与执行分发。

<!-- @end-section -->

<!-- @section: interface -->
---

## 2. 接口定义

```go
type ToolRegistry interface {
    Register(tool Tool) error
    List(ctx ToolListContext) ([]ToolDescriptor, error)
    Execute(ctx context.Context, call ToolCall) (ToolResult, error)
}

type Tool interface {
    Name() string
    Descriptor() ToolDescriptor
    Execute(ctx context.Context, args json.RawMessage) (ToolResult, error)
}
```

<!-- @end-section -->

<!-- @section: pipeline -->
---

## 3. 执行流水线

1. 解析工具名并定位实现。
2. JSON Schema 校验入参。
3. 调用 `PermissionEnforcer` 做批量权限检查。
4. 调用 `ExecutionPolicy` 判断自动允许、询问或拒绝。
5. 调用 `ToolGuardrails.BeforeCall`。
6. 执行工具并捕获超时、stderr、退出码。
7. 调用 `ToolGuardrails.AfterCall`。
8. 返回结构化 `ToolResult` 并写入工具调用日志。

<!-- @end-section -->

<!-- @section: contracts -->
---

## 4. 行为契约

- 工具名全局唯一，格式为 `domain.action`。
- 工具描述必须稳定，避免破坏 Prompt Cache。
- 同一任务内工具列表上限由 CognitiveCore 控制。
- 工具执行必须有 timeout，默认 120s。

<!-- @end-section -->
