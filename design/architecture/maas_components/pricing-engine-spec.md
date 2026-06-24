---
id: "spec-component-pricing-engine-030"
title: "PricingEngine 组件规范"
aliases: ["PricingEngine规范", "定价引擎", "pricing-engine-spec"]
type: "spec"
category: "design/architecture/components"
tags: ["component-spec", "billing", "pricing", "maas", "interface"]
version: "0.1.0"
created: "2026-05-09"
updated: "2026-05-09"
author: "jxncyjq"
status: "draft"
parent: "spec-component-registry-000"

component_id: "C30"
layer: "L4"
depends_on: []
optional_deps: []
conflicts_with: []
required_by:
  - "C31"   # BillingSession 使用 PricingEngine 计算预扣和结算金额
  - "C22"   # CostAwareStrategy 使用 PricingEngine 比较提供商成本
assembly_profiles:
  - minimal
  - standard
  - enterprise
  - embedded
---

<!-- @section: overview -->
# PricingEngine 组件规范

## 1. 组件定位

`PricingEngine` 是计费层的**价格计算引擎**，负责将"模型 ID + Token 用量"转换为"内部配额单位"。

它支持三种定价模式，覆盖从简单按 token 计费到复杂动态定价的所有场景，是 `BillingSession` 和 `CostAwareStrategy` 的必须依赖。

```
BillingSession / CostAwareStrategy
        │
        ▼
  PricingEngine
  ┌──────────────────────────────────────────────────────┐
  │  Estimate(modelID, estimatedInput, estimatedOutput)  │ ← 请求前预估
  │  Calculate(modelID, actualUsage, snapshot)           │ ← 调用后精确计算
  │                                                      │
  │  内置三种定价模式：                                   │
  │    ratio      → tokens × ratio × group_ratio         │
  │    fixed      → count × fixed_price                  │
  │    tiered_expr→ AST 表达式引擎                        │
  └──────────────────────────────────────────────────────┘
```

**读者**：实现定价逻辑的工程师、配置模型价格的运营人员。

<!-- @end-section -->

<!-- @section: interface -->
---

## 2. 接口定义

### 2.1 PricingEngine 接口

```go
// PricingEngine 根据模型和 token 用量计算配额消耗。
// 并发安全：同一实例可在多个 goroutine 中并发调用。
// 不做 IO：价格数据在初始化时加载到内存，运行期只读。
type PricingEngine interface {
    // Estimate 在请求前基于估算 token 数计算预扣配额。
    // estimatedInput / estimatedOutput 由框架基于消息长度估算，可能偏高。
    // 返回的 QuotaEstimate 包含阶梯计费的快照，传递给 Calculate 保证一致性。
    Estimate(ctx context.Context, modelID string,
        estimatedInput, estimatedOutput int) (QuotaEstimate, error)

    // Calculate 基于真实 token 用量计算最终配额。
    // snapshot 是 Estimate 返回的快照，用于 tiered_expr 模式下的表达式一致性。
    // 在 ratio / fixed 模式下 snapshot 可忽略（传入零值亦可）。
    Calculate(ctx context.Context, modelID string,
        usage TokenUsage, snapshot QuotaEstimate) (QuotaResult, error)

    // ModelPrice 返回模型的定价信息（用于 CostAwareStrategy 比较成本）。
    // 返回的 InputPrice / OutputPrice 是 USD per 1M tokens。
    // 若模型不存在，返回 ErrPricingModelNotFound。
    ModelPrice(modelID string) (ModelPriceInfo, error)

    // Reload 重新从配置源加载价格数据（热更新）。
    // 框架在检测到价格配置变更时调用，实现方保证原子替换。
    Reload(ctx context.Context) error
}
```

### 2.2 相关类型

```go
// QuotaEstimate 预扣阶段的估算结果。
type QuotaEstimate struct {
    Units       int64       // 预扣配额单位数（内部计量单位）
    BillingMode BillingMode // ratio | fixed | tiered_expr
    ModelID     string
    // Snapshot 仅 tiered_expr 模式有值，冻结当前计费表达式，
    // 防止计费期间价格更新导致预扣和结算不一致。
    Snapshot    *TieredExprSnapshot
}

// QuotaResult 结算阶段的精确结果。
type QuotaResult struct {
    Units   int64      // 实际消耗的配额单位数
    Usage   TokenUsage // 用于审计的 token 明细
}

// ModelPriceInfo 用于路由决策的价格信息（不含内部配额换算）。
type ModelPriceInfo struct {
    ModelID      string
    BillingMode  BillingMode
    InputPrice   float64  // USD per 1M input tokens
    OutputPrice  float64  // USD per 1M output tokens
    FixedPrice   float64  // 仅 fixed 模式有值（USD per request）
}

// BillingMode 定价模式。
type BillingMode string

const (
    BillingModeRatio      BillingMode = "ratio"
    BillingModeFixed      BillingMode = "fixed"
    BillingModeTieredExpr BillingMode = "tiered_expr"
)
```

<!-- @end-section -->

<!-- @section: pricing-modes -->
---

## 3. 定价模式详解

### 3.1 ratio 模式（按比例）

适用于：主流按 token 计费模型（Claude、GPT-4 等）。

```
配额 = (InputTokens × InputRatio + OutputTokens × OutputRatio) × GroupRatio
```

**参数说明**：
- `InputRatio`：输入 token 的配额比率（默认 1.0，即 1 token = 1 配额单位）
- `OutputRatio`：输出 token 的配额比率（通常 = InputRatio × 3，反映成本差异）
- `GroupRatio`：用户组系数（VIP 用户 0.8 → 8 折；免费用户 1.5 → 溢价）

**配置示例**（YAML）：

```yaml
model_pricing:
  claude-sonnet-4-6:
    mode: ratio
    input_ratio: 1.0
    output_ratio: 3.0
    # USD 参考价（供 CostAwareStrategy 使用）
    input_price_usd_per_1m: 3.00
    output_price_usd_per_1m: 15.00
```

**缓存 token 处理**（Anthropic prompt cache）：

```go
// CacheReadTokens 按折扣比率计算（默认 0.1 × input_ratio）
// CacheWriteTokens 按溢价计算（默认 1.25 × input_ratio）
// 框架内置此逻辑，实现方无需单独处理
```

### 3.2 fixed 模式（按次计费）

适用于：图像生成、视频生成等按请求计费的模型。

```
配额 = FixedUnitsPerRequest × GroupRatio
```

**配置示例**：

```yaml
model_pricing:
  dall-e-3:
    mode: fixed
    fixed_units: 10000    # 每次请求消耗 10000 配额单位（约 1 USD）
```

### 3.3 tiered_expr 模式（表达式引擎）

适用于：分时段折扣、阶梯价格、复杂动态定价场景。

定价规则通过 AST 表达式字符串配置，支持变量引用和条件分支：

```
expr: "(input_tokens * 0.003 + output_tokens * 0.015) * group_ratio * time_factor"

variables:
  time_factor:
    type: time_range
    ranges:
      - { from: "00:00", to: "08:00", value: 0.5 }  # 低峰折扣
      - { from: "08:00", to: "24:00", value: 1.0 }
```

**快照机制**：

预扣时冻结当前表达式快照（`TieredExprSnapshot`），结算时使用相同快照计算，
避免价格更新导致预扣与结算不一致。

```go
// TieredExprSnapshot 冻结的表达式快照。
type TieredExprSnapshot struct {
    ExprVersion string  // 价格配置的版本号
    Variables   map[string]float64  // 预扣时刻的变量取值（已求值）
    CompiledAt  time.Time
}
```

<!-- @end-section -->

<!-- @section: config -->
---

## 4. 配置 Schema

```go
// PricingEngineConfig 是 PricingEngine 的配置。
type PricingEngineConfig struct {
    // Source 价格数据来源。
    Source PricingSource `validate:"required"`

    // QuotaPerUnit 一个配额单位对应的 USD 价值（用于配额→USD 换算）。
    // 默认 0.000001 USD/unit，即 1M 配额单位 = 1 USD。
    QuotaPerUnit float64 `default:"0.000001" validate:"min=0.0000001"`

    // DefaultGroupRatio 未配置用户组比率时的默认值。
    DefaultGroupRatio float64 `default:"1.0" validate:"min=0.01,max=100"`

    // CacheReadDiscount CacheReadTokens 相对于 InputTokens 的折扣系数。
    CacheReadDiscount float64 `default:"0.1" validate:"min=0,max=1"`

    // CacheWritePremium CacheWriteTokens 相对于 InputTokens 的溢价系数。
    CacheWritePremium float64 `default:"1.25" validate:"min=1"`

    // ReloadInterval 自动重新加载价格的间隔（0 = 禁用自动重载）。
    ReloadInterval time.Duration `default:"5m"`
}

// PricingSource 价格数据来源。
type PricingSource struct {
    Type    string // "yaml" | "database" | "remote"
    // YAML 来源
    FilePath string
    // Database 来源
    TableName string
    // Remote 来源
    URL     string
    Headers map[string]string
}
```

<!-- @end-section -->

<!-- @section: quota-unit -->
---

## 5. 配额单位制

框架使用**内部配额单位（Quota Unit）**作为统一计量货币：

```
1 配额单位 = QuotaPerUnit USD = 0.000001 USD（默认）
1M 配额单位 = 1 USD

换算示例（claude-sonnet-4-6，ratio 模式）：
  InputRatio = 1.0，OutputRatio = 3.0
  1 input token  = 1 配额单位 = 0.000001 USD
  1 output token = 3 配额单位 = 0.000003 USD
  参考价：$3.00/1M input → 3.00/1 = 3.0 ratio ✓
```

**GroupRatio 示例**：

| 用户组 | GroupRatio | 含义 |
|--------|-----------|------|
| free   | 1.5       | 免费用户需多消耗 50% 配额 |
| basic  | 1.0       | 标准定价 |
| pro    | 0.9       | 9 折优惠 |
| vip    | 0.7       | 7 折优惠 |

<!-- @end-section -->

<!-- @section: behavior-contracts -->
---

## 6. 行为契约

| 契约 | 说明 |
|------|------|
| **不做 IO** | `Estimate()` 和 `Calculate()` 是纯内存操作，不查询 DB |
| **并发安全** | 同一实例可被多个 goroutine 并发调用 |
| **Units 始终 >= 0** | 不得返回负值配额（计算结果向上取整） |
| **Estimate >= 实际用量（软约束）** | 预估应保守（偏高），避免账户余额提前耗尽，但不强制 |
| **tiered_expr 快照一致性** | Estimate 返回的 Snapshot 必须在 Calculate 中复现相同结果 |
| **ModelNotFound 立即返回** | 模型 ID 不存在时，不得返回 0 值，必须返回 ErrPricingModelNotFound |
| **Reload 原子替换** | 价格热更新期间，飞行中的请求使用旧版本完成，新版本对新请求生效 |

<!-- @end-section -->

<!-- @section: error-types -->
---

## 7. 错误类型

```go
// PricingError 是 PricingEngine 返回的错误类型。
type PricingError struct {
    Code    PricingErrorCode
    Message string
    Cause   error
}

type PricingErrorCode string

const (
    // ErrPricingModelNotFound 未找到模型的定价配置。
    // 调用方应返回 400（请求的模型未配置价格）。
    ErrPricingModelNotFound PricingErrorCode = "PRICING_MODEL_NOT_FOUND"

    // ErrExprEvalFailed 表达式引擎求值失败（配置错误）。
    // 调用方应告警并降级到 ratio 模式（如果有配置）。
    ErrExprEvalFailed PricingErrorCode = "EXPR_EVAL_FAILED"

    // ErrSnapshotMismatch 结算时 Snapshot 版本与当前价格不兼容。
    // 极罕见，通常因价格配置删除引发。使用当前版本重新计算。
    ErrSnapshotMismatch PricingErrorCode = "SNAPSHOT_MISMATCH"
)
```

<!-- @end-section -->

<!-- @section: testing -->
---

## 8. 测试支持

### 8.1 契约测试

```go
// RunPricingEngineContractTests 验证 PricingEngine 实现的计算正确性和行为契约。
func RunPricingEngineContractTests(t *testing.T, engine PricingEngine) {
    t.Run("Estimate/UnitsNonNegative", ...)
    t.Run("Calculate/RatioMode/CorrectFormula", ...)
    t.Run("Calculate/FixedMode/CorrectFormula", ...)
    t.Run("Calculate/TieredExpr/SnapshotConsistency", ...)
    t.Run("ModelPrice/NotFoundReturnsError", ...)
    t.Run("ConcurrencySafety/ParallelEstimate", ...)
    t.Run("Reload/InFlightRequestsUseOldVersion", ...)
}
```

### 8.2 固定价格 Mock

```go
// FixedPricingEngine 用于单元测试，对所有模型返回固定价格。
type FixedPricingEngine struct {
    UnitsPerInputToken  int64
    UnitsPerOutputToken int64
}

func (f *FixedPricingEngine) Estimate(_ context.Context, modelID string,
    estimatedInput, estimatedOutput int) (QuotaEstimate, error) {
    return QuotaEstimate{
        Units:       int64(estimatedInput)*f.UnitsPerInputToken + int64(estimatedOutput)*f.UnitsPerOutputToken,
        BillingMode: BillingModeRatio,
        ModelID:     modelID,
    }, nil
}
```

<!-- @end-section -->

<!-- @section: checklist -->
---

## 9. 实现检查清单

```
PricingEngine
  ☐ 支持三种定价模式：ratio / fixed / tiered_expr
  ☐ Estimate：保守估算（≥ 实际消耗），不做 IO
  ☐ Calculate：精确计算，Units >= 0，向上取整
  ☐ CacheReadTokens / CacheWriteTokens 按折扣/溢价系数处理
  ☐ GroupRatio 从 RequestContext.TenantContext 获取
  ☐ ModelPrice：仅读取内存缓存，不查 DB
  ☐ Reload：原子替换价格数据，飞行中请求不受影响
  ☐ 模型不存在时返回 ErrPricingModelNotFound（不返回 0）

tiered_expr 模式
  ☐ Estimate 时求值所有时间相关变量，冻结为 Snapshot
  ☐ Calculate 时使用 Snapshot 中的变量值（不重新求值）
  ☐ 表达式求值失败时返回 ErrExprEvalFailed

测试
  ☐ 运行 RunPricingEngineContractTests（全部通过）
  ☐ 三种模式各有公式验证用例
  ☐ GroupRatio 影响验证
  ☐ 并发安全验证
```

<!-- @end-section -->

<!-- @section: related -->
---

## 相关文档

- [[component-registry|组件注册表与依赖关系图]]（C30 在依赖图中的位置）
- billing-session-spec.md（C31，主要消费者）
- model-router-spec.md（C14，CostAwareStrategy C22 消费 ModelPrice）
- funding-source-spec.md（C32，与 BillingSession 协同工作）

<!-- @end-section -->
