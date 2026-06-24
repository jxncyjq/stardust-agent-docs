---
id: "spec-component-billing-session-031"
title: "BillingSession 组件规范"
aliases: ["BillingSession规范", "计费会话", "billing-session-spec"]
type: "spec"
category: "design/architecture/components"
tags: ["component-spec", "billing", "maas", "interface", "contract"]
version: "0.1.0"
created: "2026-05-09"
updated: "2026-05-09"
author: "jxncyjq"
status: "draft"
parent: "spec-component-registry-000"

component_id: "C31"
layer: "L4"
depends_on:
  - "C30"   # PricingEngine — 计算本次调用成本
  - "C32"   # FundingSource — 实际扣款对象（钱包/订阅）
optional_deps:
  - "C33"   # QuotaBudgetManager — 缺失时不做多级配额检查，只扣资金来源
conflicts_with: []
required_by:
  - "C63"   # CostAttributor 依赖 BillingSession 获取结算数据
assembly_profiles:
  - minimal
  - standard
  - enterprise
  - embedded
---

<!-- @section: overview -->
# BillingSession 组件规范

## 1. 组件定位

`BillingSession` 管理**单次推理请求**的完整计费生命周期：预扣 → 执行 → 结算 / 退款。

它是计费层的**编排者**，本身不计算价格（委托给 `PricingEngine`），也不持有余额（委托给 `FundingSource`），只负责把两者串联成原子操作并保证幂等。

```
调用方（RequestPipeline）
        │
        ▼
  BillingSession
  ┌──────────────────────────────────┐
  │  PreConsume(estimate)            │
  │    ├── PricingEngine.Estimate()  │
  │    ├── FundingSource.PreConsume()│
  │    └── QuotaBudget.Consume()     │ ← 可选
  │                                  │
  │  [上游 API 调用]                 │
  │                                  │
  │  Settle(actual) / Refund()       │
  │    ├── PricingEngine.Calculate() │
  │    ├── FundingSource.Settle()    │
  │    └── QuotaBudget.Adjust()      │ ← 可选
  └──────────────────────────────────┘
```

**读者**：实现计费逻辑的工程师、集成 BillingSession 的管道开发者。

<!-- @end-section -->

<!-- @section: interface -->
---

## 2. 接口定义

### 2.1 BillingSession 接口

```go
// BillingSession 管理单次请求的计费生命周期。
// 每次请求由 BillingSessionFactory 创建一个新实例，实例不得跨请求复用。
// 所有方法均非并发安全（单请求单实例，无需并发）。
type BillingSession interface {
    // PreConsume 在上游调用前执行预扣。
    // estimate 是 PricingEngine 基于估算 token 数计算的配额，用于预扣。
    // 信任旁路：若 FundingSource.Balance() 远高于 TrustThreshold，跳过实际预扣
    // 以减少数据库锁争用（仍记录 shouldTrust=true 标记，Settle 时补扣）。
    // 失败时返回 *BillingError，调用方应立即终止请求。
    PreConsume(ctx context.Context, estimate QuotaEstimate) error

    // Settle 在上游调用成功后结算实际消耗。
    // actual 是 PricingEngine 基于真实 token 用量计算的配额。
    // delta = actual - preConsumed：正值补扣，负值退还。
    // Settle 必须在 PreConsume 成功后才能调用。
    Settle(ctx context.Context, actual QuotaResult) error

    // Refund 在上游调用失败时退还预扣的全部配额。
    // 幂等：同一 RequestID 多次调用只执行一次退款。
    // 应在 goroutine 中异步调用，不阻塞错误响应返回。
    Refund(ctx context.Context) error

    // Snapshot 返回当前计费状态快照（用于日志和遥测）。
    Snapshot() BillingSnapshot
}

// BillingSessionFactory 创建 BillingSession 实例。
// 框架在每个请求开始时调用，传入请求上下文。
type BillingSessionFactory interface {
    // New 根据请求上下文创建计费会话。
    // 内部完成：选择 FundingSource、加载计费偏好、初始化幂等 key。
    New(ctx *RequestContext) (BillingSession, error)
}
```

### 2.2 相关类型

```go
// QuotaEstimate 预扣阶段的估算结果（来自 PricingEngine）。
type QuotaEstimate struct {
    Units       int64       // 预扣配额单位
    BillingMode BillingMode // ratio | fixed | tiered_expr
    ModelID     string
    Snapshot    any         // 阶梯计费模式下冻结的表达式快照
}

// QuotaResult 结算阶段的实际结果（来自 PricingEngine）。
type QuotaResult struct {
    Units   int64
    Usage   TokenUsage
}

// BillingSnapshot 计费状态快照，用于日志、审计和遥测。
type BillingSnapshot struct {
    RequestID      string
    FundingSource  string      // "wallet" | "subscription" | "free_tier"
    BillingMode    BillingMode
    PreConsumed    int64
    ActualConsumed int64
    Delta          int64       // actual - pre（结算后才有值）
    TrustBypassed  bool        // 是否触发了信任旁路
    Status         SessionStatus
}

// SessionStatus 会话状态机。
type SessionStatus int

const (
    SessionPending   SessionStatus = iota // 已创建，未预扣
    SessionPresumed                       // 已预扣，等待结算
    SessionSettled                        // 已结算
    SessionRefunded                       // 已退款
)
```

<!-- @end-section -->

<!-- @section: funding-source -->
---

## 3. FundingSource 接口（C32）

`FundingSource` 是 `BillingSession` 的必须依赖，抽象实际的资金来源。

```go
// FundingSource 抽象资金来源（钱包、订阅、免费额度）。
// 由 BillingSessionFactory 根据用户的 BillingPreference 选择。
type FundingSource interface {
    // SourceType 返回资金来源类型标识。
    SourceType() string // "wallet" | "subscription" | "free_tier"

    // Balance 返回当前可用额度（配额单位）。
    // 用于信任旁路判断，不要求强一致（允许略旧）。
    Balance(ctx context.Context) (int64, error)

    // PreConsume 预扣指定额度。幂等键为 requestID。
    // 余额不足时返回 ErrInsufficientBalance。
    PreConsume(ctx context.Context, requestID string, amount int64) error

    // Settle 结算差额（可正可负）。
    // delta > 0：补扣；delta < 0：退还。
    Settle(ctx context.Context, requestID string, delta int64) error

    // Refund 退还全部预扣额度。幂等，多次调用安全。
    Refund(ctx context.Context, requestID string) error
}
```

**内置实现**：

| 实现 | 说明 |
|------|------|
| `WalletFundingSource` | 直接操作用户配额钱包（`users.quota` 字段） |
| `SubscriptionFundingSource` | 操作用户订阅的 `amount_used`，行锁 + FOR UPDATE |
| `FreeTierFundingSource` | 免费额度桶，到零后自动切换到钱包 |

**计费偏好（BillingPreference）** — 决定 Factory 选哪个实现：

| 策略 | 行为 |
|------|------|
| `subscription_first`（默认） | 先用订阅；订阅耗尽回退钱包 |
| `wallet_first` | 先用钱包；不足回退订阅 |
| `subscription_only` | 仅订阅；不足直接拒绝 |
| `wallet_only` | 仅钱包；不足直接拒绝 |

<!-- @end-section -->

<!-- @section: pricing-engine -->
---

## 4. PricingEngine 接口（C30）

`PricingEngine` 是 `BillingSession` 的另一个必须依赖，负责价格计算。

```go
// PricingEngine 根据模型和 token 用量计算配额消耗。
type PricingEngine interface {
    // Estimate 在请求前估算配额（用于预扣）。
    // estimatedInput / estimatedOutput 由框架基于上下文长度估算。
    Estimate(ctx context.Context, modelID string,
        estimatedInput, estimatedOutput int) (QuotaEstimate, error)

    // Calculate 根据真实 token 用量计算最终配额。
    Calculate(ctx context.Context, modelID string,
        usage TokenUsage, snapshot QuotaEstimate) (QuotaResult, error)
}
```

**三种定价模式**（在 `QuotaEstimate.BillingMode` 中标识）：

| 模式 | 计算公式 | 适用场景 |
|------|----------|----------|
| `ratio` | `tokens × model_ratio × group_ratio` | 主流按 token 计费模型 |
| `fixed` | `model_price × QuotaPerUnit` | 按次计费 |
| `tiered_expr` | 表达式引擎（含缓存折扣、分时段等） | 复杂动态定价 |

<!-- @end-section -->

<!-- @section: state-machine -->
---

## 5. 状态机与行为契约

### 5.1 合法状态转换

```
Pending ──PreConsume()──► Presumed ──Settle()──► Settled
                               │
                               └──Refund()──► Refunded
```

非法调用（框架返回 `ErrInvalidSessionState`）：

| 当前状态 | 非法调用 |
|----------|----------|
| `Pending` | `Settle()`、`Refund()` |
| `Presumed` | `PreConsume()`（重复预扣） |
| `Settled` | `Settle()`、`Refund()`、`PreConsume()` |
| `Refunded` | 所有写操作 |

### 5.2 行为契约

| 契约 | 说明 |
|------|------|
| **原子性** | `PreConsume` 内的 token 配额扣减和资金来源扣减，任一失败则全部回滚 |
| **幂等性** | `Refund` 以 `RequestID` 为幂等键，多次调用只退款一次 |
| **信任旁路不影响结算** | 信任旁路跳过预扣，但 `Settle` 仍执行完整结算（包括补扣） |
| **Delta 精度** | `delta` 可为负数（预估偏高时），负 delta 必须退还而非丢弃 |
| **Settle 不阻塞响应** | 框架在响应已写回客户端后异步执行 Settle，Settle 失败不影响用户 |
| **Refund 异步** | Refund 在独立 goroutine 中执行，失败时入重试队列，最终一致 |

### 5.3 信任旁路阈值

```go
// TrustThreshold 当用户余额 >= 此值时，跳过预扣（减少 DB 锁争用）。
// 默认值 = 10 × QuotaPerUnit（约合 10 USD 等值的内部配额）。
// 可通过配置覆盖。订阅类资金来源不适用信任旁路。
const DefaultTrustThreshold = 10 * QuotaPerUnit
```

<!-- @end-section -->

<!-- @section: error-types -->
---

## 6. 错误类型

```go
// BillingError 是 BillingSession 所有方法返回的错误类型。
type BillingError struct {
    Code    BillingErrorCode
    Message string
    Cause   error // 原始错误（DB 错误等）
}

type BillingErrorCode string

const (
    // ErrInsufficientBalance 余额不足，调用方应拒绝请求并返回 402。
    ErrInsufficientBalance BillingErrorCode = "INSUFFICIENT_BALANCE"

    // ErrInvalidSessionState 状态机非法转换（编程错误）。
    ErrInvalidSessionState BillingErrorCode = "INVALID_SESSION_STATE"

    // ErrFundingSourceUnavailable 资金来源暂时不可用（DB 超时等），可重试。
    ErrFundingSourceUnavailable BillingErrorCode = "FUNDING_SOURCE_UNAVAILABLE"

    // ErrPricingModelNotFound 未找到该模型的定价配置。
    ErrPricingModelNotFound BillingErrorCode = "PRICING_MODEL_NOT_FOUND"
)
```

**调用方处理指南**：

| 错误码 | HTTP 状态码 | 处理方式 |
|--------|-------------|----------|
| `INSUFFICIENT_BALANCE` | 402 | 拒绝请求，提示用户充值 |
| `INVALID_SESSION_STATE` | 500 | 记录告警，属于框架 bug |
| `FUNDING_SOURCE_UNAVAILABLE` | 503 | 可重试（最多 2 次） |
| `PRICING_MODEL_NOT_FOUND` | 400 | 拒绝请求，提示模型配置缺失 |

<!-- @end-section -->

<!-- @section: lifecycle -->
---

## 7. 生命周期与配置

### 7.1 配置 Schema

```go
// BillingSessionConfig 由 BillingSessionFactory 使用。
type BillingSessionConfig struct {
    // TrustThreshold 信任旁路阈值（配额单位）。
    // 0 = 禁用信任旁路（所有请求都预扣）。
    TrustThreshold int64 `default:"5000000"` // 10 USD 等值

    // RefundRetryMax 退款失败时的最大重试次数。
    RefundRetryMax int `default:"5"`

    // RefundRetryBackoff 退款重试退避基数。
    RefundRetryBackoff time.Duration `default:"10s"`

    // AsyncSettle 是否异步执行 Settle（true = 响应先返回，Settle 后台执行）。
    AsyncSettle bool `default:"true"`
}
```

### 7.2 框架集成点

`BillingSession` 由请求管道的两个节点调用：

```
QuotaChecker（C13）    → 调用 PricingEngine.Estimate()，快速预检（不预扣）
      ↓
ProviderExecutor       → 调用 BillingSession.PreConsume()
      ↓
[上游 API 调用]
      ↓
BillingSettler（管道尾）→ 调用 BillingSession.Settle() 或 Refund()
      ↓
AuditLogger（C62）     → 调用 BillingSession.Snapshot() 写审计日志
```

<!-- @end-section -->

<!-- @section: testing -->
---

## 8. 测试支持

### 8.1 Mock 实现

```go
// MockBillingSession 用于管道和控制器的单元测试。
type MockBillingSession struct {
    PreConsumeErr error
    SettleErr     error
    RefundErr     error
    Calls         []string // 记录调用顺序，用于断言状态机
}

func (m *MockBillingSession) PreConsume(_ context.Context, _ QuotaEstimate) error {
    m.Calls = append(m.Calls, "PreConsume")
    return m.PreConsumeErr
}
// ... Settle, Refund, Snapshot 同上
```

### 8.2 契约测试

```go
// RunBillingSessionContractTests 验证 BillingSession 实现的状态机和幂等性。
//
//   func TestMyBillingSession(t *testing.T) {
//       session := myimpl.NewSession(testFactory())
//       billing_testing.RunBillingSessionContractTests(t, session)
//   }
func RunBillingSessionContractTests(t *testing.T, factory BillingSessionFactory) {
    t.Run("StateMachine/SettleBeforePreConsumeFails", ...)
    t.Run("StateMachine/DoublePreConsumeFails", ...)
    t.Run("StateMachine/RefundAfterSettleFails", ...)
    t.Run("Idempotency/MultipleRefundOnlyOnce", ...)
    t.Run("TrustBypass/HighBalanceSkipsPreConsume", ...)
    t.Run("Delta/NegativeDeltaIsRefunded", ...)
}
```

<!-- @end-section -->

<!-- @section: checklist -->
---

## 9. 实现检查清单

```
BillingSessionFactory
  ☐ 根据 BillingPreference 正确选择 FundingSource 实现
  ☐ 支持四种计费偏好策略（subscription_first 为默认）
  ☐ 初始化时验证 PricingEngine 和 FundingSource 均非 nil

BillingSession
  ☐ 状态机：非法转换返回 ErrInvalidSessionState
  ☐ PreConsume：token 配额和资金来源原子扣减（任一失败全回滚）
  ☐ 信任旁路：余额 >= TrustThreshold 时跳过预扣，标记 shouldTrust
  ☐ Settle：正确计算 delta，正值补扣，负值退还
  ☐ Refund：RequestID 幂等，多次调用只执行一次
  ☐ Snapshot：在任意状态下均可调用，返回当前快照

测试
  ☐ 运行 RunBillingSessionContractTests（全部通过）
  ☐ 覆盖信任旁路场景
  ☐ 覆盖负 delta 退款场景
  ☐ 覆盖 Refund 幂等场景
```

<!-- @end-section -->

<!-- @section: related -->
---

## 相关文档

- [[component-registry|组件注册表与依赖关系图]]（C31 在依赖图中的位置）
- pricing-engine-spec.md（C30，必须依赖）
- funding-source-spec.md（C32，必须依赖）
- quota-budget-manager-spec.md（C33，可选依赖）
- [[model-provider-spec|ModelProvider 组件规范]]（C01）

<!-- @end-section -->
