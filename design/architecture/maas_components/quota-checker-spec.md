---
id: "spec-component-quota-checker-013"
title: "QuotaChecker 组件规范"
aliases: ["QuotaChecker规范", "配额检查器", "quota-checker-spec"]
type: "spec"
category: "design/architecture/components"
tags: ["component-spec", "quota", "pipeline", "billing", "maas"]
version: "0.1.0"
created: "2026-05-09"
updated: "2026-05-09"
author: "jxncyjq"
status: "draft"
parent: "spec-component-registry-000"

component_id: "C13"
layer: "L2"
depends_on:
  - "C33"   # QuotaBudgetManager — 执行实际配额检查
optional_deps: []
conflicts_with: []
required_by: []
assembly_profiles:
  - standard
  - enterprise
---

<!-- @section: overview -->
# QuotaChecker 组件规范

## 1. 组件定位

`QuotaChecker` 是请求管道中的**快速配额预检节点**，在请求进入 `ProviderExecutor` 之前做一次轻量检查：用户是否有足够的配额余额发起请求。

它只做**只读预检**（不扣款），将真正的扣款留给 `BillingSession.PreConsume()`，避免在请求必然失败（余额为零）时浪费路由和提供商调用开销。

```
RateLimiter → QuotaChecker → ModelRouter → ProviderExecutor
                  │
                  │ QuotaBudgetManager.Check(ctx, estimatedUnits=0)
                  │ 仅检查账户是否有余额，不估算 token 数
                  │
                  ├── 余额为零 → 402，终止（避免无意义的后续处理）
                  └── 余额充足 → 继续
```

**读者**：集成配额管理的管道开发者。

<!-- @end-section -->

<!-- @section: interface -->
---

## 2. 接口定义

```go
// QuotaChecker 在请求入口做快速配额余额预检。
// 并发安全：同一实例可在多个 goroutine 中并发调用。
type QuotaChecker interface {
    // Check 验证调用方是否有足够余额发起请求。
    // 此时尚未估算 token 数，仅检查账户余额 > 0。
    // 返回 nil 表示允许继续；返回 error 表示拒绝（余额不足或后端不可用）。
    Check(ctx *RequestContext) error
}
```

**实现说明**：

`QuotaChecker` 是一个**薄封装层**，核心逻辑委托给 `QuotaBudgetManager.Check(ctx, estimatedUnits=0)`：

```go
func (c *quotaChecker) Check(ctx *RequestContext) error {
    // estimatedUnits=0 表示仅检查余额是否 > 0，不估算具体消耗
    return c.budgetMgr.Check(ctx, 0)
}
```

<!-- @end-section -->

<!-- @section: behavior-contracts -->
---

## 3. 行为契约

| 契约 | 说明 |
|------|------|
| **只读操作** | Check 不扣减任何配额，不产生任何副作用 |
| **余额为零才拒绝** | 有任何余额（哪怕只剩 1 单位）即允许通过 |
| **后端不可用时降级** | QuotaBudgetManager 不可用时，可配置为允许（乐观）或拒绝（保守） |

<!-- @end-section -->

<!-- @section: checklist -->
---

## 4. 实现检查清单

```
QuotaChecker
  ☐ 委托 QuotaBudgetManager.Check(ctx, 0) 实现
  ☐ 余额 > 0 时允许通过
  ☐ 后端不可用时的降级策略（可配置：允许 / 拒绝）
  ☐ 并发安全
```

<!-- @end-section -->

<!-- @section: related -->
---

## 相关文档

- [[component-registry|组件注册表与依赖关系图]]（C13 在依赖图中的位置）
- quota-budget-manager-spec.md（C33，必须依赖）
- billing-session-spec.md（C31，真正扣款的位置）

<!-- @end-section -->
