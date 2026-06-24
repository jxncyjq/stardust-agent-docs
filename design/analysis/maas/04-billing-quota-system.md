---
id: "analysis-newapi-billing-004"
title: "计费与配额系统分析"
aliases: ["new-api billing", "计费系统", "quota system", "billing expression"]
type: "analysis"
category: "design/analysis/maas"
tags: ["new-api", "billing", "quota", "pricing", "subscription", "maas"]
version: "1.0.0"
created: "2026-05-03"
updated: "2026-05-03"
author: "jxncyjq"
status: "review"
parent: "analysis-newapi-overview-001"
related_docs:
  - id: "analysis-newapi-overview-001"
    relation: "parent"
    path: "./01-overview.md"
  - id: "analysis-newapi-modules-002"
    relation: "related_to"
    path: "./02-module-analysis.md"
  - id: "analysis-newapi-channel-003"
    relation: "related_to"
    path: "./03-channel-adapter-system.md"
---

<!-- @section: overview -->
# 计费与配额系统分析

## 系统总览

New API 的计费体系由**三个核心层**组成，呈正交结构：

```
定价引擎 → 计费会话 → 消费记录
(Price)    (Session)   (Log)
```

1. **定价引擎** — 定义每个模型的单价/倍率/表达式（支持三种模式）
2. **计费会话** — 管理单次请求的 预扣→结算→退款 生命周期，抽象资金来源（钱包 vs 订阅）
3. **消费记录** — 持久化日志 + 通知 + 统计

核心单位换算：**`1 USD = 500,000 配额单位`**（`QuotaPerUnit = 500 * 1000.0 = 500_000`，等价于 `$0.002 / 1K tokens`，定义在 `common/constants.go:41`）。

<!-- @end-section -->

<!-- @section: pricing-modes -->
## 三种定价模式

### 模式对比

| 模式 | 配置键 | 适用场景 |
|------|--------|----------|
| `ratio`（倍率） | `model_ratio` | 传统按 token 计费模型 |
| `price`（固定价格） | `model_price` | 固定单价的模型 |
| `tiered_expr`（阶梯表达式） | `billing_expr` | 复杂动态定价（缓存、分时段） |

系统通过 `billing_setting.GetBillingMode(model)` 决定为每个模型激活哪个模式。

### 模式 1: 倍率系统 (ratio)

**核心计算公式**:
```
配额 = (inputTextTokens +
        outputTextTokens × completionRatio +
        inputAudioTokens × audioRatio +
        outputAudioTokens × audioRatio × audioCompletionRatio)
       × modelRatio × groupRatio
```

**倍率层级**:

| 倍率 | 作用域 | 默认值 | 说明 |
|------|--------|--------|------|
| `model_ratio` | 模型 | 配置值 | 输入 Token 基础倍率 |
| `completion_ratio` | 模型 | 硬编码回退 | 输出 Token 完成倍率 |
| `cache_ratio` | 模型 | 配置值 | 缓存命中折价 |
| `create_cache_ratio` | 模型 | 配置值 | 缓存创建溢价 |
| `image_ratio` | 模型 | 配置值 | 图片 Token 倍率 |
| `audio_ratio` | 模型 | 配置值 | 音频 Token 倍率 |
| `group_ratio` | 用户分组 | 1.0 | 分组折扣/溢价 |

**完成倍率硬编码回退** (`getHardcodedCompletionModelRatio()`):
- `gpt-4o` 系列 → 4×
- `claude-3`/`claude-4` 系列 → 5×
- `gemini-1.5` 系列 → 4×

**分组倍率三层结构**:
1. 基础分组倍率 `GetGroupRatio(group)` — `"default"`/`"vip"`/`"svip"` 默认 1.0
2. 组对组倍率 `GetGroupGroupRatio(userGroup, usingGroup)`
3. 特殊可用分组覆盖

### 模式 2: 固定价格 (price)

```
配额 = modelPrice × QuotaPerUnit × groupRatio
```

适用于按次计费或固定单价的模型。

### 模式 3: 阶梯表达式 (tiered_expr)

**设计理念**: 一个表达式 = 完整计费逻辑。定价、阶梯条件、缓存/图片/音频差异化、基于时间的折扣 — 一行搞定。

**Token 变量**:

| 变量 | 含义 | 自动排除规则 |
|------|------|-------------|
| `p` | 提示 Token | 排除 `cr`, `img`, `ai` 等单独定价的子类 |
| `c` | 完成 Token | 排除单独定价的子类 |
| `len` | 输入上下文长度 | **从不缩减**（仅阶梯条件） |
| `cr` | 缓存读取命中 | 独立计费 |
| `cc` | 缓存创建（5min TTL） | 独立计费 |
| `cc1h` | 缓存创建（1h TTL） | Claude 专用 |
| `img` | 图片输入 Token | 独立计费 |
| `img_o` | 图片输出 Token | 独立计费 |
| `ai` | 音频输入 Token | 独立计费 |
| `ao` | 音频输出 Token | 独立计费 |

**内置函数**: `tier(name, value)`, `param(path)`, `header(key)`, `has(source, substr)`, `hour(tz)`, `minute(tz)`, `weekday(tz)`, `month(tz)`, `day(tz)`, `max`, `min`, `abs`, `ceil`, `floor`

**自动排除机制**: 通过 AST 自省（`UsedVars()`）检测表达式引用的变量，自动从 `p` 和 `c` 中减去单独定价的子类别，避免重复计费。

**配额转换公式**:
```
内部配额 = (表达式输出 / 1,000,000) × QuotaPerUnit × groupRatio
```

**表达式缓存**: 按 SHA-256 哈希缓存编译结果，最多 256 条目。`InvalidateCache()` 在规则更新时清除。

**表达式示例**:
```
# 基础定价: $2.50/M 输入 + $10/M 输出
p * 2.5 + c * 10

# 带阶梯: 前 128K 上下文 $2.50, 超出 $5.00
tier(len <= 128000, p * 2.5) + tier(len > 128000, p * 5) + c * 10

# 带缓存折扣: 缓存命中 $0.50/M
(p - cr) * 2.5 + c * 10 + cr * 0.5

# 带时段折扣: 0-8点 8 折
p * 2.5 * tier(hour('Asia/Shanghai') >= 0 && hour('Asia/Shanghai') < 8, 0.8, 1) + c * 10
```

<!-- @end-section -->

<!-- @section: billing-session -->
## 计费会话 (BillingSession)

### 资金来源抽象

```go
type FundingSource interface {
    Source() string
    PreConsume(amount int) error
    Settle(delta int) error
    Refund() error
}
```

**两种实现**:

| 实现 | 说明 |
|------|------|
| `WalletFunding` | 用户配额钱包，直接扣减 `User.Quota` |
| `SubscriptionFunding` | 订阅套餐，委托 `model.PreConsumeUserSubscription()` |

### 计费偏好策略

用户可配置 `BillingPreference` 决定资金来源优先级：

| 策略 | 行为 |
|------|------|
| `subscription_only` | 仅使用订阅 |
| `wallet_only` | 仅使用钱包 |
| `wallet_first` | 先钱包，不足时回退到订阅 |
| `subscription_first`（默认） | 先检查是否有活跃订阅，有则用订阅；不足时回退钱包 |

### BillingSession 生命周期

```
请求到达
  │
  ▼
NewBillingSession(relayInfo)
  ├── 读用户 BillingPreference
  ├── 选择 FundingSource (钱包/订阅)
  └── 检查信任额度旁路
  │
  ▼
preConsume(quota)
  ├── shouldTrust()? → 是 → 跳过实际预扣（高余额用户）
  ├── DecreaseTokenQuota (令牌额度扣减)
  ├── funding.PreConsume()
  │   ├── Wallet: DecreaseUserQuota (用户钱包扣减)
  │   └── Subscription: PreConsumeUserSubscription (幂等)
  └── 任一步失败 → 回滚已完成步骤
  │
  ▼
上游 API 调用成功
  │
  ▼
Settle(actualQuota)
  ├── delta = actualQuota - preConsumedQuota
  ├── delta > 0 → 补扣
  ├── delta < 0 → 退还多余预扣
  └── 触发额度通知
  │
  ▼
RecordConsumeLog() → 持久化日志

====== 如果上游失败 ======
  │
  ▼
Refund()
  ├── 异步退还预扣的 Token 配额
  ├── 异步退还资金来源预扣
  └── 幂等保护（防止重复退款）
```

### 信任额度旁路

当用户余额远高于 `TrustQuota`（= `10 × QuotaPerUnit` = 5,000,000 配额单位 = 折合 10 USD 等值的内部配额，源自 `common/quota.go::GetTrustQuota`）时，跳过预扣费以避免高余额用户的锁争用。**订阅计费不启用信任旁路**。

### 流式传输中途追加

对于流式请求，如果输出 Token 超过预估，可调用 `Reserve(targetQuota)` 追加保留配额。

<!-- @end-section -->

<!-- @section: tiered-settlement -->
## 阶梯计费结算流程

### 预扣费阶段

`modelPriceHelperTiered()`:
1. 从 `billing_setting.GetBillingExpr(model)` 加载表达式
2. 构建 `RequestInput`（headers + body 供 `param()`/`header()` 使用）
3. 用估算 Token 运行表达式: `RunExprWithRequest(expr, {P: estimatedInput, C: estimatedOutput}, requestInput)`
4. 配额转换: `rawCost / 1_000_000 × QuotaPerUnit`
5. 创建 `BillingSnapshot`（冻结状态）存储到 `RelayInfo.TieredBillingSnapshot`

### 结算阶段

`TryTieredSettle()`:
1. 用实际 usage 构建 `TokenParams`（`BuildTieredTokenParams(usage, isClaudeSemantic, usedVars)`）
2. 用冻结的 `BillingSnapshot` 重新运行表达式
3. 配额转换并返回 `TieredResult`

<!-- @end-section -->

<!-- @section: subscription -->
## 订阅系统

### 数据模型

| 模型 | 职责 |
|------|------|
| `SubscriptionPlan` | 套餐定义：标题、价格、时长、总配额、重置周期、升级分组 |
| `SubscriptionOrder` | 支付订单：用户、套餐、金额、交易单号、支付方式 |
| `UserSubscription` | 活跃订阅实例：总量/已用、时间范围、重置配置 |
| `SubscriptionPreConsumeRecord` | 按请求的幂等预扣费记录 |

### 订阅生命周期

```
购买 → 订单创建 → 支付回调 → 创建 UserSubscription
  ├── 设置分组升级 (UpgradeGroup → User.Group)
  └── 保存旧分组 (PrevUserGroup)

消费:
  RequestId → PreConsumeUserSubscription (幂等)
  ├── FOR UPDATE 行锁
  ├── 检查预扣记录是否已存在
  └── 原子扣减 AmountUsed

结算: PostConsumeUserSubscriptionDelta(delta)
  └── FOR UPDATE → AmountUsed += delta

重置: maybeResetUserSubscriptionWithPlanTx
  └── NextResetTime 已过 → AmountUsed = 0 → 推进窗口

过期: ExpireDueSubscriptions
  ├── Status → "expired"
  ├── 分组降级 (UpgradeGroup → PrevUserGroup)
  └── 检查是否还有其他活跃升级订阅

退款: RefundSubscriptionPreConsume
  └── RequestId 幂等 → 3 次重试归还 AmountUsed
```

### 配额重置周期

| 周期 | 重置时间 |
|------|----------|
| `daily` | 次日 00:00 |
| `weekly` | 下周一 00:00 |
| `monthly` | 下月 1 日 00:00 |
| `custom` | 按秒计时间隔 |

<!-- @end-section -->

<!-- @section: consumption-log -->
## 消费日志系统

### Log 类型

| 类型 | 值 | 说明 |
|------|-----|------|
| `LogTypeUnknown` | 0 | 未分类 |
| `LogTypeTopup` | 1 | 充值 |
| `LogTypeConsume` | 2 | 消费 |
| `LogTypeManage` | 3 | 管理操作 |
| `LogTypeSystem` | 4 | 系统操作 |
| `LogTypeError` | 5 | 错误 |
| `LogTypeRefund` | 6 | 退款 |

### Log 字段

```go
type Log struct {
    UserId, TokenId, ChannelId   int      // 关联 ID
    ModelName, Username, TokenName string  // 名称
    PromptTokens, CompletionTokens int     // Token 统计
    Quota                        int      // 消耗配额
    UseTime                      int      // 耗时(ms)
    IsStream                     bool     // 是否流式
    Group, Ip, RequestId         string   // 上下文
    Other                        string   // JSON 扩展（含计费明细）
}
```

### 计费信息注入

`InjectTieredBillingInfo()` 将 `billing_mode`, `expr_b64`, `matched_tier` 写入 Log.Other，供前端展示计费明细。

<!-- @end-section -->

<!-- @section: channel-balance -->
## 渠道余额监控

`controller/channel-billing.go` 实现了对多个提供商的余额自动查询：

| 提供商 | 查询接口 | 余额计算 |
|--------|----------|----------|
| OpenAI | `/v1/dashboard/billing/subscription` | `hardLimitUSD - usageUSD` |
| SiliconFlow | `/v1/user/info` | `totalBalance` |
| DeepSeek | `/user/balance` | CNY → USD 转换 |
| OpenRouter | `/v1/credits` | `totalCredits - totalUsage` |
| Moonshot | 自定义 | CNY → USD (`operation_setting.Price`) |

`AutomaticallyUpdateChannels(frequency)` 定时刷新所有渠道余额，余额为零或负数的渠道自动禁用。

<!-- @end-section -->

<!-- @section: openai-compat -->
## OpenAI 兼容计费端点

提供 `/v1/dashboard/billing/subscription` 和 `/v1/dashboard/billing/usage` 端点，与 OpenAI API 完全兼容：

- **subscription**: 返回配额作为 `soft_limit_usd`/`hard_limit_usd`，支持 USD/CNY/TOKENS 三种显示模式
- **usage**: 返回总使用量 `total_usage × 100`（美分）

<!-- @end-section -->

<!-- @section: violations -->
## 违规罚款机制

当上游返回 CSAM (Child Safety) 违规标记时：
- 正常预扣费**退款**后执行**额外罚款**
- 罚款金额 = `GrokSettings.ViolationDeductionAmount × groupRatio`
- 与正常计费流程隔离，不混入计费会话

<!-- @end-section -->

<!-- @section: data-flow -->
## 整体计费数据流

```
请求到达
  │
  ▼
ModelPriceHelper() [relay/helper/price.go]
  ├── tiered_expr: 加载表达式 → 估算运行 → 冻结 BillingSnapshot
  └── ratio/price: 获取倍率 → 计算 preConsumedQuota
  │
  ▼
PreConsumeBilling() [service/billing.go]
  ├── NewBillingSession() — 选资金来源 (钱包/订阅)
  └── session.preConsume() — 信任检查 → 预扣
  │
  ▼
上游 API 调用
  │
  ▼
SettleBilling()
  ├── tiered: TryTieredSettle() — 实际 Token 重新运行表达式
  └── BillingSession.Settle(actualQuota)
      ├── 差额调整资金来源
      ├── 差额调整 Token 配额
      └── 额度通知
  │
  ▼
RecordConsumeLog() → DB
  │
  ▼
（失败时）BillingSession.Refund() → 异步全额退还
```

<!-- @end-section -->

<!-- @section: related -->
## 相关文档

- [[01-overview|New API 项目架构总览]]
- [[02-module-analysis|Go 模块功能分析]]
- [[03-channel-adapter-system|渠道适配器系统分析]]
- [[05-middleware-and-flow|中间件与请求流程分析]]

<!-- @end-section -->
