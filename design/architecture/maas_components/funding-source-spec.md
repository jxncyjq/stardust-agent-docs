---
id: "spec-component-funding-source-032"
title: "FundingSource 组件规范"
aliases: ["FundingSource规范", "资金来源", "funding-source-spec"]
type: "spec"
category: "design/architecture/components"
tags: ["component-spec", "billing", "wallet", "subscription", "maas"]
version: "0.1.0"
created: "2026-05-09"
updated: "2026-05-09"
author: "jxncyjq"
status: "draft"
parent: "spec-component-registry-000"

component_id: "C32"
layer: "L4"
depends_on: []
optional_deps: []
conflicts_with: []
required_by:
  - "C31"   # BillingSession 通过 FundingSource 执行实际扣款
assembly_profiles:
  - minimal
  - standard
  - enterprise
  - embedded
---

<!-- @section: overview -->
# FundingSource 组件规范

## 1. 组件定位

`FundingSource` 是计费层的**资金来源抽象**，将实际的扣款操作（钱包、订阅、免费额度）统一为一个接口，使 `BillingSession` 与具体存储方案解耦。

框架提供三种内置实现，`BillingSessionFactory` 根据用户的计费偏好（`BillingPreference`）动态选择。

```
BillingSession
        │
        │ 委托扣款
        ▼
  FundingSource（接口）
  ┌──────────────────────┬─────────────────────────┬──────────────────────┐
  │ WalletFundingSource  │SubscriptionFundingSource │ FreeTierFundingSource│
  │ 直接操作配额钱包      │ 操作订阅 amount_used     │ 免费额度桶            │
  └──────────────────────┴─────────────────────────┴──────────────────────┘
```

**读者**：实现新资金来源的工程师、理解计费流程的开发者。

<!-- @end-section -->

<!-- @section: interface -->
---

## 2. 接口定义

### 2.1 FundingSource 接口

```go
// FundingSource 抽象实际的资金来源（钱包 / 订阅 / 免费额度）。
// 由 BillingSessionFactory 根据用户的 BillingPreference 在请求开始时选择。
// 所有写方法必须以 requestID 为幂等键。
type FundingSource interface {
    // SourceType 返回资金来源类型标识（用于日志和审计）。
    SourceType() string // "wallet" | "subscription" | "free_tier"

    // Balance 返回当前可用配额余额。
    // 不要求强一致（允许略旧），仅用于信任旁路判断。
    // 失败时返回 0 和错误，调用方应跳过信任旁路（按保守处理）。
    Balance(ctx context.Context) (int64, error)

    // PreConsume 预扣指定配额量。
    // requestID 是幂等键；相同 requestID 的重复调用视为成功（不重复扣款）。
    // 余额不足时返回 ErrInsufficientBalance（不做部分扣款）。
    PreConsume(ctx context.Context, requestID string, amount int64) error

    // Settle 结算差额（delta = actual - preConsumed）。
    // delta > 0：补扣；delta < 0：退还。delta == 0：无操作（但仍需记录）。
    // 必须在 PreConsume 成功后才能调用（框架保证调用顺序）。
    Settle(ctx context.Context, requestID string, delta int64) error

    // Refund 退还全部预扣配额。
    // 幂等：相同 requestID 多次调用只退款一次。
    // 若 PreConsume 未被调用（信任旁路跳过），Refund 应为无操作。
    Refund(ctx context.Context, requestID string) error
}

// FundingSourceFactory 根据用户上下文创建合适的 FundingSource。
// 框架在 BillingSessionFactory.New() 内部调用。
type FundingSourceFactory interface {
    // Select 根据用户计费偏好和当前余额情况，选择最合适的资金来源。
    // 内部实现计费偏好策略（subscription_first / wallet_first 等）。
    Select(ctx context.Context, userID string, pref BillingPreference) (FundingSource, error)
}
```

### 2.2 计费偏好

```go
// BillingPreference 决定 FundingSourceFactory 如何选择和回退资金来源。
type BillingPreference string

const (
    // BillingPrefSubscriptionFirst 先用订阅额度，耗尽后回退到钱包（默认）。
    BillingPrefSubscriptionFirst BillingPreference = "subscription_first"

    // BillingPrefWalletFirst 先用钱包，不足时回退到订阅。
    BillingPrefWalletFirst BillingPreference = "wallet_first"

    // BillingPrefSubscriptionOnly 仅使用订阅，不足则拒绝请求。
    BillingPrefSubscriptionOnly BillingPreference = "subscription_only"

    // BillingPrefWalletOnly 仅使用钱包，不足则拒绝请求。
    BillingPrefWalletOnly BillingPreference = "wallet_only"
)
```

<!-- @end-section -->

<!-- @section: implementations -->
---

## 3. 内置实现

### 3.1 WalletFundingSource

直接操作用户配额钱包（`users.quota` 字段）。

```go
// WalletFundingSource 实现 FundingSource，操作 users 表中的 quota 字段。
// 使用数据库行锁（SELECT FOR UPDATE）保证扣款原子性。
//
// 数据库操作：
//   PreConsume: UPDATE users SET quota = quota - amount WHERE id = ? AND quota >= amount
//   Settle:     UPDATE users SET quota = quota - delta WHERE id = ?  (delta 可负)
//   Refund:     UPDATE users SET quota = quota + preConsumed WHERE id = ?
//              （通过 billing_records 表查询 preConsumed，避免重复退款）
//
// 幂等实现：
//   billing_records 表存储 (request_id, state, pre_consumed, settled)
//   PreConsume: INSERT ... ON CONFLICT (request_id) DO NOTHING
//   Refund:     UPDATE ... WHERE state != 'refunded'（CAS 防重）
type WalletFundingSource struct {
    db     *sql.DB
    userID string
}

func (w *WalletFundingSource) SourceType() string { return "wallet" }
```

**表结构（参考）**：

```sql
-- 幂等记录表（所有 FundingSource 共用）
CREATE TABLE billing_records (
    request_id   VARCHAR(64) PRIMARY KEY,
    user_id      BIGINT      NOT NULL,
    source_type  VARCHAR(20) NOT NULL,
    pre_consumed BIGINT      NOT NULL DEFAULT 0,
    settled      BIGINT      NOT NULL DEFAULT 0,
    state        VARCHAR(20) NOT NULL DEFAULT 'pending', -- pending|presumed|settled|refunded
    created_at   TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    settled_at   TIMESTAMPTZ
);
```

### 3.2 SubscriptionFundingSource

操作用户订阅的已用量（`subscriptions.amount_used`），使用行锁防止超扣。

```go
// SubscriptionFundingSource 操作 subscriptions 表中的 amount_used 字段。
//
// 约束检查：
//   PreConsume: amount_used + amount <= total_amount（订阅总额）
//   超出订阅额度时返回 ErrInsufficientBalance
//
// 注意：订阅资金来源不支持信任旁路（Balance() 返回精确值，
// 但 BillingSessionFactory 对订阅类型不跳过预扣）。
type SubscriptionFundingSource struct {
    db             *sql.DB
    userID         string
    subscriptionID int64
}

func (s *SubscriptionFundingSource) SourceType() string { return "subscription" }
```

### 3.3 FreeTierFundingSource

管理免费额度桶，耗尽后**不自动回退**（由 `FundingSourceFactory` 的策略层处理回退）。

```go
// FreeTierFundingSource 操作 free_tier_buckets 表。
// 耗尽时返回 ErrInsufficientBalance，由 Select 策略层决定是否回退到钱包。
//
// 配置：每个用户每月的免费额度由 TenantConfig 决定。
// 重置：月初自动重置（后台 cron job，非本组件责任）。
type FreeTierFundingSource struct {
    db     *sql.DB
    userID string
}

func (f *FreeTierFundingSource) SourceType() string { return "free_tier" }
```

### 3.4 FallbackFundingSource（组合器）

组合两个 FundingSource，实现自动回退：

```go
// FallbackFundingSource 先尝试主资金来源，余额不足时切换到备用来源。
// 用于实现 subscription_first 和 wallet_first 策略。
//
// 注意：回退发生在 PreConsume 时（余额不足触发），
// Settle / Refund 始终向最终使用的资金来源操作。
type FallbackFundingSource struct {
    primary  FundingSource
    fallback FundingSource
    used     FundingSource // PreConsume 后确定
}
```

<!-- @end-section -->

<!-- @section: selection-strategy -->
---

## 4. 资金来源选择策略

`FundingSourceFactory.Select()` 根据 `BillingPreference` 构建资金来源链：

| 偏好 | 实现 | 行为 |
|------|------|------|
| `subscription_first`（默认） | `FallbackFundingSource(sub, wallet)` | 先扣订阅，不足切钱包 |
| `wallet_first` | `FallbackFundingSource(wallet, sub)` | 先扣钱包，不足切订阅 |
| `subscription_only` | `SubscriptionFundingSource` | 不足直接返回 402 |
| `wallet_only` | `WalletFundingSource` | 不足直接返回 402 |

**无可用资金来源时**（用户无订阅且钱包余额为零）：`Select()` 返回 `ErrNoFundingSource`，框架拒绝请求并返回 402。

<!-- @end-section -->

<!-- @section: behavior-contracts -->
---

## 5. 行为契约

| 契约 | 说明 |
|------|------|
| **幂等性** | 所有写方法以 `requestID` 为幂等键，重复调用不重复操作 |
| **不做部分扣款** | 余额不足时原子拒绝，不扣部分金额 |
| **Balance 允许略旧** | `Balance()` 可使用副本或缓存，不要求实时一致 |
| **Refund 为无操作（无预扣时）** | 信任旁路跳过了 PreConsume 时，Refund 必须安全返回 nil |
| **Settle 允许负 delta** | 负 delta 表示退还超扣部分，实现方不得拒绝 |
| **数据库操作用行锁** | PreConsume 必须使用 `SELECT FOR UPDATE` 或等效机制防止超扣 |
| **Refund 异步安全** | Refund 在独立 goroutine 中执行，实现方不得持有请求级锁 |

<!-- @end-section -->

<!-- @section: error-types -->
---

## 6. 错误类型

```go
// FundingSourceError 是 FundingSource 方法返回的错误类型。
type FundingSourceError struct {
    Code    FundingSourceErrorCode
    Message string
    Cause   error
}

type FundingSourceErrorCode string

const (
    // ErrInsufficientBalance 余额不足。调用方（BillingSession）返回 402。
    ErrInsufficientBalance FundingSourceErrorCode = "INSUFFICIENT_BALANCE"

    // ErrNoFundingSource 用户没有可用的资金来源（无订阅且无钱包余额）。
    ErrNoFundingSource FundingSourceErrorCode = "NO_FUNDING_SOURCE"

    // ErrFundingSourceUnavailable 资金来源暂时不可用（DB 超时等）。
    // 调用方可重试（最多 2 次）。
    ErrFundingSourceUnavailable FundingSourceErrorCode = "FUNDING_SOURCE_UNAVAILABLE"

    // ErrRecordNotFound 结算或退款时找不到对应的 billing_record。
    // 通常因 PreConsume 未成功就调用 Settle，属于框架 bug。
    ErrRecordNotFound FundingSourceErrorCode = "RECORD_NOT_FOUND"
)
```

<!-- @end-section -->

<!-- @section: testing -->
---

## 7. 测试支持

### 7.1 契约测试

```go
// RunFundingSourceContractTests 验证 FundingSource 实现的幂等性和正确性。
func RunFundingSourceContractTests(t *testing.T, source FundingSource) {
    t.Run("PreConsume/InsufficientBalanceReturnsError", ...)
    t.Run("PreConsume/Idempotent/SameRequestIDNoDoubleDeduct", ...)
    t.Run("Settle/PositiveDeltaDeductsExtra", ...)
    t.Run("Settle/NegativeDeltaRefundsExtra", ...)
    t.Run("Refund/Idempotent/MultipleCallsOnlyRefundOnce", ...)
    t.Run("Refund/NoPriorPreConsume/ReturnsNil", ...)
    t.Run("Balance/ReturnsNonNegative", ...)
}
```

### 7.2 内存 Mock 实现

```go
// MemoryFundingSource 基于内存的 FundingSource 实现，用于单元测试。
type MemoryFundingSource struct {
    mu      sync.Mutex
    balance int64
    records map[string]*billingRecord // requestID → record
}

func NewMemoryFundingSource(initialBalance int64) *MemoryFundingSource {
    return &MemoryFundingSource{
        balance: initialBalance,
        records: make(map[string]*billingRecord),
    }
}

func (m *MemoryFundingSource) SourceType() string { return "memory" }

func (m *MemoryFundingSource) PreConsume(_ context.Context, requestID string, amount int64) error {
    m.mu.Lock()
    defer m.mu.Unlock()
    if _, exists := m.records[requestID]; exists {
        return nil // 幂等
    }
    if m.balance < amount {
        return &FundingSourceError{Code: ErrInsufficientBalance}
    }
    m.balance -= amount
    m.records[requestID] = &billingRecord{preConsumed: amount, state: "presumed"}
    return nil
}
```

<!-- @end-section -->

<!-- @section: checklist -->
---

## 8. 实现检查清单

```
FundingSource
  ☐ PreConsume：余额不足原子拒绝（不部分扣款），行锁防超扣
  ☐ PreConsume：以 requestID 为幂等键，重复调用返回 nil
  ☐ Settle：支持正/负 delta；负 delta 退还超扣
  ☐ Settle：必须验证 billing_record 存在（防止跳过 PreConsume）
  ☐ Refund：以 requestID 为幂等键，多次调用只退款一次
  ☐ Refund：无预扣记录时安全返回 nil（信任旁路场景）
  ☐ Balance：允许使用略旧数据，失败时返回 0 + error

WalletFundingSource
  ☐ 使用 billing_records 表存储幂等状态
  ☐ PreConsume 使用 SELECT FOR UPDATE 或 CAS UPDATE 防超扣

SubscriptionFundingSource
  ☐ 检查 amount_used + amount <= total_amount
  ☐ 订阅过期时返回 ErrInsufficientBalance

FreeTierFundingSource
  ☐ 耗尽时返回 ErrInsufficientBalance（不自动回退）
  ☐ 月度重置由外部 cron 负责

FallbackFundingSource
  ☐ PreConsume 失败（余额不足）时切换到 fallback
  ☐ Settle / Refund 对最终使用的资金来源操作

FundingSourceFactory
  ☐ 支持四种计费偏好策略
  ☐ 无可用来源时返回 ErrNoFundingSource

测试
  ☐ 运行 RunFundingSourceContractTests（全部通过）
  ☐ 并发 PreConsume 不超扣（竞态验证）
  ☐ 负 delta Settle 场景
  ☐ 幂等场景（PreConsume / Refund 重复调用）
```

<!-- @end-section -->

<!-- @section: related -->
---

## 相关文档

- [[component-registry|组件注册表与依赖关系图]]（C32 在依赖图中的位置）
- billing-session-spec.md（C31，主要消费者）
- pricing-engine-spec.md（C30，与 BillingSession 协同工作）
- quota-budget-manager-spec.md（C33，可选的多级配额管理）

<!-- @end-section -->
