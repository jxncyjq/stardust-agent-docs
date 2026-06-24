---
id: "analysis-newapi-datamodels-006"
title: "核心数据模型分析"
aliases: ["new-api data models", "数据模型", "GORM", "database schema"]
type: "analysis"
category: "design/analysis/maas"
tags: ["new-api", "data-model", "GORM", "MySQL", "PostgreSQL", "SQLite", "maas"]
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
  - id: "analysis-newapi-billing-004"
    relation: "related_to"
    path: "./04-billing-quota-system.md"
---

<!-- @section: overview -->
# 核心数据模型分析

## 数据库架构

### 双数据库实例

```go
var DB *gorm.DB     // 主数据库
var LOG_DB *gorm.DB // 日志数据库（可独立部署）
```

日志数据库可通过 `LOG_SQL_DSN` 环境变量独立配置，默认复用主库。

### 三数据库兼容

| 数据库 | 识别方式 | 连接配置 |
|--------|----------|----------|
| SQLite | DSN 前缀 `local` 或 DSN 为空 | 文件路径 |
| MySQL | 其他非 `postgres://` 前缀 | 自动追加 `parseTime=true` |
| PostgreSQL | DSN 前缀 `postgres://` 或 `postgresql://` | 双引号引用列名 |

### 跨数据库兼容机制

```go
var commonGroupCol string  // `group` (MySQL/SQLite) 或 "group" (PostgreSQL)
var commonKeyCol string    // `key` 或 "key"
var commonTrueVal string   // 1 或 true
var commonFalseVal string  // 0 或 false
```

### 连接池配置

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `SQL_MAX_IDLE_CONNS` | 100 | 最大空闲连接 |
| `SQL_MAX_OPEN_CONNS` | 1000 | 最大打开连接 |
| `SQL_MAX_LIFETIME` | 60s | 连接最大存活时间 |

### 数据库迁移

使用 GORM `AutoMigrate` 自动迁移共 **25 张表**：24 张通过 `migrateDBFast()` 并发执行加速，另 1 张 `SubscriptionPlan` 因 SQLite 兼容（`ensureSubscriptionPlanTableSQLite`）需要在 SQLite 路径上单独迁移。

特殊迁移步骤：
1. `price_amount` 列从 float 迁移到 decimal(10,6)
2. `model_limits` 列从 varchar(1024) 迁移到 text
3. SQLite 手动建表和逐列 ADD COLUMN
4. MySQL 中文字符集校验（utf8mb4/utf8/gbk/big5/gb18030）

<!-- @end-section -->

<!-- @section: core-models -->
## 核心数据模型

### 1. User（用户）

**表**: `users`

**核心字段**:
```go
User {
    Id, Username, Password, DisplayName, Email  // 身份
    Role, Status                                 // 权限 (admin/common, enabled/disabled)
    Quota, UsedQuota, RequestCount               // 额度
    Group                                        // 分组 (default/vip/svip)
    GitHubId, DiscordId, OidcId, WeChatId...    // OAuth 绑定 (6 种)
    AffCode, AffCount, AffQuota, InviterId       // 邀请系统
    Setting                                      // JSON 用户设置
    AccessToken                                  // 系统管理 Token (char(32))
    DeletedAt                                    // 软删除
}
```

**关键方法**:
- `Insert/Update/Delete` — CRUD（Insert 含邀请奖励逻辑）
- `ValidateAndFill` — 密码验证 + 状态检查
- `IncreaseUserQuota/DecreaseUserQuota` — 额度变更
- `ClearBinding` — OAuth 解绑
- `ToBaseUser()` — 转换为缓存对象

**用户设置 (Setting JSON)**:
- 通知配置、Webhook URL、语言偏好
- 计费偏好（subscription_first / wallet_first 等）
- 侧边栏模块配置

### 2. UserBase（用户缓存对象）

**Redis Key**: `user:{userId}`，Hash 结构

**缓存字段**:
```go
UserBase {
    Id, Group, Email, Quota, Status, Username, Setting
}
```

**缓存策略**:
- Cache-Aside: Redis → DB → 异步回填
- 原子操作: `cacheIncrUserQuota` / `cacheDecrUserQuota` (Redis HIncrBy)
- 写穿透: Update 后异步写 Redis (`gopool.Go`)

### 3. Token（API 令牌）

**表**: `tokens`

**核心字段**:
```go
Token {
    Id, UserId, Key                       // 主键 + 外键
    Status, Name                           // 状态 + 名称
    ExpiredTime (-1=永不过期)               // 过期时间
    RemainQuota, UnlimitedQuota, UsedQuota // 额度
    ModelLimitsEnabled, ModelLimits        // 模型限制
    AllowIps                               // IP 白名单 (CIDR)
    Group, CrossGroupRetry                 // 分组 + 跨分组重试
    DeletedAt                              // 软删除
}
```

**与 User 关系**: `UserId` 外键指向 `User.Id`

**验证流程** (`ValidateUserToken`):
1. Redis 缓存命中 → 检查状态/过期/额度
2. 缓存未命中 → DB 查询 → 异步回填缓存
3. 过期或额度耗尽 → 自动标记为禁用

**Token 缓存**:
- Redis Key: `token:{HMAC(key)}`（HMAC 哈希保护原始 Key）
- 原子操作: `cacheIncrTokenQuota` / `cacheDecrTokenQuota`

### 4. Channel（渠道）

**表**: `channels`

**核心字段**:
```go
Channel {
    Id, Type, Key, Name, Status            // 基本信息
    BaseURL                                // 自定义 API 端点
    Models, Group                          // 支持的模型和分组（逗号分隔）
    Weight, Priority                       // 负载权重 + 优先级
    Balance, BalanceUpdatedTime            // USD 余额
    ModelMapping                           // 模型名映射 JSON
    ParamOverride, HeaderOverride          // 参数/请求头覆盖 JSON
    Setting, OtherSettings                 // 渠道配置 JSON
    AutoBan                                // 是否自动禁用
    ChannelInfo                            // 多 Key 管理信息 JSON
}
```

**多 Key 模式** (`ChannelInfo`):
```go
ChannelInfo {
    IsMultiKey             bool              // 是否启用多 Key
    MultiKeySize           int               // Key 数量
    MultiKeyStatusList     map[int]int       // 各 Key 状态
    MultiKeyMode           string            // random/polling
    MultiKeyPollingIndex   int               // 轮询当前位置
}
```

**多 Key 策略**:
- `Random`: 随机选择启用的 Key
- `Polling`: 轮询选择（channel 级别线程安全锁）

**关键方法**:
- `Insert()` → 创建渠道 + 自动生成 Ability 记录
- `Update()` → 更新渠道 + 事务内重建 Ability
- `GetNextEnabledKey()` → 获取下一个可用 API Key

### 5. Ability（渠道能力表）

**表**: `abilities`

**复合主键**: `(group, model, channel_id)`

```go
Ability {
    Group, Model, ChannelId    // 复合主键
    Enabled                    // 是否启用
    Priority, Weight           // 优先级 + 权重
    Tag                        // 标签
}
```

**与 Channel 关系**:
- `Channel.AddAbilities(tx)`: Models × Groups 笛卡尔积生成 Ability
- `Channel.UpdateAbilities(tx)`: 先删后插（事务内）
- `Channel.DeleteAbilities()`: 级联删除

**渠道选择逻辑**:
1. 按 `(group, model, enabled=true)` 查询
2. 按 Priority 降序取最高优先级集合
3. 同优先级按 Weight 加权随机选择
4. 失败重试: `retry` 参数让选择下一优先级

### 6. Channel 内存缓存

**双缓存结构**:
```go
group2model2channels: map[string]map[string][]int  // group→model→[]channelId
channelsIDM: map[int]*Channel                       // channelId→*Channel
```

**缓存操作**:
- `InitChannelCache`: 全量加载 → 按 Priority 排序
- `SyncChannelCache(frequency)`: 定期 DB 全量重载
- `CacheUpdateChannelStatus`: 更新状态 + 维护 group2model2channels
- `GetRandomSatisfiedChannel`: 按 group+model+priority 加权随机选择

### 7. Log（日志）

**表**: `logs`（可使用独立 LOG_DB）

```go
Log {
    Id, UserId, TokenId, ChannelId                     // 关联 ID
    Type                                                // 0-6 类型
    ModelName, Username, TokenName, ChannelName         // 名称
    PromptTokens, CompletionTokens                      // Token 统计
    Quota                                               // 消耗配额
    UseTime, IsStream                                   // 耗时 + 流式标记
    Group, Ip, RequestId                                // 上下文
    Other                                               // JSON 扩展
}
```

**日志类型**:
| 类型 | 值 | 说明 |
|------|-----|------|
| Unknown | 0 | 未分类 |
| Topup | 1 | 充值 |
| Consume | 2 | 消费 |
| Manage | 3 | 管理操作 |
| System | 4 | 系统操作 |
| Error | 5 | 错误 |
| Refund | 6 | 退款 |

**关键方法**:
- `RecordConsumeLog` — 消费日志 + 触发数据看板
- `RecordErrorLog` — 错误日志（支持用户 IP 记录偏好）
- `GetAllLogs/GetUserLogs` — 多条件查询 + 统计
- `SumUsedQuota` — 聚合配额/RPM/TPM
- `DeleteOldLog` — 分批删除旧日志

### 8. Option（系统配置）

**表**: `options`，Key-Value 结构

```go
Option {
    Key   string  // 主键
    Value string  // 配置值
}
```

**双层配置架构**:
1. **扁平键值对**: 约 150+ 配置项，内存 `OptionMap`
2. **分层配置**: `config.GlobalConfig`，通过点号分隔键名识别

**配置加载**: 环境变量 → 默认值 → DB 覆盖

<!-- @end-section -->

<!-- @section: billing-models -->
## 计费相关模型

### SubscriptionPlan（订阅计划）

```go
SubscriptionPlan {
    Id, Title, Subtitle                         // 基本信息
    PriceAmount (decimal(10,6)), Currency       // 价格
    DurationUnit, DurationValue                 // 时长 (year/month/day/hour/custom)
    TotalAmount                                 // 总配额量
    QuotaResetPeriod                            // 重置周期 (never/daily/weekly/monthly/custom)
    UpgradeGroup                                // 购买后升级的分组
    StripePriceId, CreemProductId              // 支付集成
}
```

### SubscriptionOrder（订阅订单）

```go
SubscriptionOrder {
    Id, UserId, PlanId                         // 关联
    Money, TradeNo                              // 金额 + 交易单号
    PaymentMethod, PaymentProvider, Status      // 支付信息
}
```

### UserSubscription（用户订阅实例）

```go
UserSubscription {
    Id, UserId, PlanId                         // 关联
    AmountTotal, AmountUsed                     // 总量/已用
    StartTime, EndTime                          // 有效时间
    Status (active/expired/cancelled)           // 状态
    LastResetTime, NextResetTime                // 重置时间
    UpgradeGroup, PrevUserGroup                 // 分组变更追踪
}
```

**分组升级/降级**:
- 购买时: `User.Group → UpgradeGroup`, 保存 `PrevUserGroup`
- 过期时: `UpgradeGroup → PrevUserGroup`（检查是否还有其他活跃订阅）

### SubscriptionPreConsumeRecord（预消费记录）

```go
SubscriptionPreConsumeRecord {
    Id, RequestId (uniqueIndex)                // 幂等保护
    UserId, UserSubscriptionId                 // 关联
    PreConsumed                                // 预扣量
    Status (consumed/refunded)                 // 状态
}
```

**幂等预扣费**: 通过 `RequestId` 唯一索引防止重复扣费。

### Pricing（定价缓存）

```go
Pricing {
    ModelName, ModelRatio, ModelPrice          // 基础定价
    CompletionRatio                             // 完成倍率
    CacheRatio, CreateCacheRatio               // 缓存折价/溢价
    ImageRatio, AudioRatio                     // 多媒体倍率
    BillingMode, BillingExpr                   // 阶梯计费
    EnableGroup                                // 可用分组
    SupportedEndpointTypes                     // 支持端点
}
```

**缓存刷新**: 1 分钟 TTL，`RefreshPricing()` 可强制立即刷新。

<!-- @end-section -->

<!-- @section: task-models -->
## 任务相关模型

### Task（异步任务）

```go
Task {
    ID, TaskID (task_xxxx)                     // 主键 + 对外 ID
    Platform, Action                            // 平台 + 动作
    UserId, Group, ChannelId                    // 关联
    Quota                                       // 消耗配额
    Status (NOT_START→SUBMITTED→IN_PROGRESS→SUCCESS/FAILURE)
    Properties (JSON)                           // 输入参数 + 模型名
    PrivateData (JSON)                          // 隐私数据（Key, UpstreamTaskID）
    Data (JSON)                                 // 灵活结果数据
    Progress, FailReason                        // 进度 + 失败原因
}
```

**隐私数据** (`TaskPrivateData`):
```go
TaskPrivateData {
    Key, UpstreamTaskID, ResultURL             // 上游信息
    BillingSource (wallet/subscription)         // 计费来源
    BillingContext                              // 计费参数快照
}
```

**CAS 更新**: `UpdateWithStatus(fromStatus)` 实现 Compare-And-Swap，防止并发覆盖计费状态。

### Midjourney（Midjourney 任务）

```go
Midjourney {
    Id, UserId, ChannelId                      // 关联
    MjId, Action, Prompt                       // Midjourney 特有
    State, Status, Progress                    // 状态跟踪
    ImageUrl, VideoUrl                         // 结果
    Quota                                       // 消耗配额
    Buttons, Properties                        // 交互数据
}
```

<!-- @end-section -->

<!-- @section: other-models -->
## 辅助模型

### Redemption（兑换码）

```go
Redemption {
    Id, UserId, Key (char(32))                 // 创建者 + 唯一码
    Quota, Count                                // 面额 + 数量
    Status, UsedUserId                          // 状态 + 使用者
    ExpiredTime (0=永不过期)                    // 过期时间
}
```

**安全兑换**: `FOR UPDATE` 行锁 + `common.RandomSleep()` 防时序攻击。

### TopUp（充值订单）

```go
TopUp {
    Id, UserId                                  // 关联
    Amount, Money                               // 配额量 + 金额
    TradeNo (unique)                            // 交易单号
    PaymentMethod, PaymentProvider, Status      // 支付信息
}
```

**支持的支付方式**: stripe, creem, waffo, waffo_pancake

### QuotaData（数据看板）

```go
QuotaData {
    Id, UserID, Username, ModelName            // 维度
    CreatedAt (按小时粒度)                      // 时间
    TokenUsed, Count, Quota                     // 统计
}
```

**缓存写入**: 内存聚合 → 定时刷 DB（按 `userId-modelName-createdAt` 做 Upsert）

### Model / Vendor（模型元数据）

```go
Model {
    Id, ModelName, Description, Icon, Tags      // 基本信息
    VendorID                                     // → Vendor
    NameRule (Exact/Prefix/Contains/Suffix)     // 匹配规则
    Endpoints (JSON)                             // 自定义端点
}
```

```go
Vendor {
    Id, Name, Description, Icon                  // 供应商信息
}
```

**Model-Vendor 关系**: N:1。`NameRule` 支持四种匹配模式用于定价自动匹配。

### PasskeyCredential（WebAuthn）

每个用户仅一个 Passkey:
```go
PasskeyCredential {
    UserID (uniqueIndex), CredentialID           // 1:1 User
    PublicKey, SignCount                         // WebAuthn 凭证
    CloneWarning, BackupState                    // 安全属性
}
```

### TwoFA（双因素认证）

```go
TwoFA {
    UserId (unique), Secret                     // 1:1 User
    IsEnabled, FailedAttempts                    // 状态
    LockedUntil                                  // 锁定时间
}
```

### Checkin（签到）

```go
Checkin {
    UserId, CheckinDate (YYYY-MM-DD)            // 复合唯一索引
    QuotaAwarded                                 // 随机奖励
}
```

### PrefillGroup（预填组）

```go
PrefillGroup {
    Name, Type (model/tag/endpoint)             // 分组信息
    Items (JSON)                                 // 预填项列表
}
```

<!-- @end-section -->

<!-- @section: entity-relationships -->
## 实体关系图

```
User (1) ──< (N) Token
User (1) ──< (N) TopUp
User (1) ──< (N) Redemption
User (1) ──< (N) Midjourney
User (1) ──< (N) Task
User (1) ──< (N) Log
User (1) ──< (N) SubscriptionOrder
User (1) ──< (N) UserSubscription
User (1) ──< (N) Checkin
User (1) ──< (N) UserOAuthBinding
User (1) ── (1) TwoFA
User (1) ── (1) PasskeyCredential

Channel (1) ──< (N) Ability
Channel (1) ──< (N) Log
Channel (1) ──< (N) Midjourney
Channel (1) ──< (N) Task

Ability: 复合主键 (Group, Model, ChannelId)
  = Channel.Models × Channel.Groups 笛卡尔积

Model (N) >── (1) Vendor

SubscriptionPlan (1) ──< (N) SubscriptionOrder
SubscriptionPlan (1) ──< (N) UserSubscription
UserSubscription (1) ──< (N) SubscriptionPreConsumeRecord

CustomOAuthProvider (1) ──< (N) UserOAuthBinding

Option: 独立键值对表，无外键
Setup: 独立表，仅一条记录
PrefillGroup: 独立表，无外键
QuotaData: 独立统计表，无外键
```

<!-- @end-section -->

<!-- @section: cache-strategy -->
## 缓存策略总结

### Redis 缓存

| 缓存对象 | Key 格式 | 类型 | 模式 |
|----------|----------|------|------|
| UserBase | `user:{id}` | Hash | Cache-Aside + 异步回填 |
| Token | `token:{HMAC(key)}` | Hash | Cache-Aside + 原子增减 |
| SubscriptionPlan | `{planId}` | HybridCache | Redis + LRU 双层 |

### 内存缓存

| 缓存对象 | 数据结构 | 同步 |
|----------|----------|------|
| channelsIDM | `map[int]*Channel` | 定期 DB 全量重载 |
| group2model2channels | `map[string]map[string][]int` | 定期 DB 全量重载 |
| pricingMap | `[]Pricing` | 1 分钟 TTL 自动刷新 |
| CacheQuotaData | `map[string]*QuotaData` | 内存聚合 → 定时刷 DB |

### 批量更新

当 `BatchUpdateEnabled=true` 时，以下操作不立即写 DB：
- `UserQuota`, `TokenQuota`, `UsedQuota`, `ChannelUsedQuota`, `RequestCount`

先写入内存 map，定期批量刷新到 DB。

<!-- @end-section -->

<!-- @section: related -->
## 相关文档

- [[01-overview|New API 项目架构总览]]
- [[02-module-analysis|Go 模块功能分析]]
- [[04-billing-quota-system|计费与配额系统分析]]
- [[05-middleware-and-flow|中间件与请求流程分析]]
- [[07-maas-insights|MaaS 洞察与 Legion 参考]]

<!-- @end-section -->
