---
id: "spec-component-content-filter-017"
title: "ContentFilter 组件规范"
aliases: ["ContentFilter规范", "内容过滤器", "content-filter-spec"]
type: "spec"
category: "design/architecture/components"
tags: ["component-spec", "safety", "pipeline", "content-moderation", "maas"]
version: "0.1.0"
created: "2026-05-09"
updated: "2026-05-09"
author: "jxncyjq"
status: "draft"
parent: "spec-component-registry-000"

component_id: "C17"
layer: "L2"
depends_on: []
optional_deps: []
conflicts_with: []
required_by: []
assembly_profiles:
  - enterprise
---

<!-- @section: overview -->
# ContentFilter 组件规范

## 1. 组件定位

`ContentFilter` 在请求进入上游 AI 服务前（以及响应返回客户端前）对内容进行安全检查，拦截违规的输入或输出。

```
ModelRouter → ContentFilter(input) → ProviderExecutor
                   │
                   ├── 违规 → 403，终止请求
                   └── 通过 → 继续

ProviderExecutor → ContentFilter(output) → 响应客户端
                         │
                         ├── 违规 → 过滤/替换内容
                         └── 通过 → 原样返回
```

**Noop 行为**：C17 未注册时，`NoopContentFilter` 始终通过，不过滤任何内容。

<!-- @end-section -->

<!-- @section: interface -->
---

## 2. 接口定义

```go
// ContentFilter 对请求输入和响应输出进行安全检查。
// 并发安全：同一实例可在多个 goroutine 中并发调用。
type ContentFilter interface {
    // FilterInput 检查请求输入（messages + system prompt）。
    // 通过时返回 nil；违规时返回 *ContentFilterError。
    FilterInput(ctx context.Context, input *ModelInput) error

    // FilterOutput 检查响应输出内容。
    // 通过时返回原始 output；违规时可替换内容或返回错误。
    FilterOutput(ctx context.Context, output *ModelOutput) (*ModelOutput, error)
}

// ContentFilterError 内容违规时返回的错误。
type ContentFilterError struct {
    Category    string   // 违规类别，如 "violence" | "hate_speech" | "pii"
    Confidence  float64  // 违规置信度 [0, 1]
    Message     string   // 人类可读描述
}
```

<!-- @end-section -->

<!-- @section: filter-categories -->
---

## 3. 过滤类别

| 类别 | 说明 | 默认行为 |
|------|------|----------|
| `violence` | 暴力内容 | 拦截（403） |
| `hate_speech` | 仇恨言论 | 拦截 |
| `self_harm` | 自伤内容 | 拦截 |
| `sexual_explicit` | 显式性内容 | 拦截（可按租户配置允许） |
| `pii` | 个人隐私信息（手机号、身份证等）| 脱敏替换（不拦截）|
| `prompt_injection` | 提示注入攻击 | 拦截 |

<!-- @end-section -->

<!-- @section: config -->
---

## 4. 配置 Schema

```go
// ContentFilterConfig 是 ContentFilter 的配置。
type ContentFilterConfig struct {
    // FilterInput 是否检查输入（默认开启）。
    FilterInput bool `default:"true"`
    // FilterOutput 是否检查输出（默认开启）。
    FilterOutput bool `default:"true"`

    // Backend 过滤后端。
    Backend string `default:"builtin" validate:"oneof=builtin openai_moderation azure_content_safety"`

    // Categories 各类别的配置（阈值 + 动作）。
    Categories map[string]CategoryConfig
}

type CategoryConfig struct {
    Enabled    bool    `default:"true"`
    Threshold  float64 `default:"0.8"`
    Action     string  `default:"block" validate:"oneof=block redact warn"`
}
```

<!-- @end-section -->

<!-- @section: behavior-contracts -->
---

## 5. 行为契约

| 契约 | 说明 |
|------|------|
| **FilterInput 失败阻断** | 后端不可用时可配置为允许（宽松）或拒绝（保守）|
| **FilterOutput 失败替换** | 输出过滤失败时返回通用错误响应（不暴露原始违规内容）|
| **Noop 始终通过** | C17 未注册时透传，不过滤任何内容 |
| **PII 脱敏不拦截** | PII 类别执行替换（如 `138****1234`），不返回 403 |

<!-- @end-section -->

<!-- @section: checklist -->
---

## 6. 实现检查清单

```
ContentFilter
  ☐ FilterInput：检查 messages + system prompt
  ☐ FilterOutput：检查生成内容
  ☐ 各类别独立配置（阈值 + 动作）
  ☐ PII 类别执行脱敏替换（非拦截）
  ☐ 后端不可用时的降级策略（可配置）

Noop 实现
  ☐ FilterInput 返回 nil
  ☐ FilterOutput 原样返回 output
```

<!-- @end-section -->

<!-- @section: related -->
---

## 相关文档

- [[component-registry|组件注册表与依赖关系图]]（C17 在依赖图中的位置）
- audit-logger-spec.md（C62，记录内容违规事件）

<!-- @end-section -->
