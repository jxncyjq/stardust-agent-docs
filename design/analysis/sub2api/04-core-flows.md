---
id: "analysis-sub2api-core-flows-001"
title: "Sub2API 核心业务流程"
type: "analysis"
category: "design/analysis"
tags: ["sub2api", "flows", "business"]
version: "1.0.0"
created: "2026-06-24"
updated: "2026-06-24"
author: "jxncyjq"
status: "published"
parent: null
children: []
related_docs:
  - id: "analysis-sub2api-index-001"
    relation: "related_to"
    path: "./README.md"
---

# Sub2API 核心业务流程

## 1. 订阅转 API 核心流程

```
用户请求
    │
    ▼
┌─────────────────────────────────────────┐
│ 1. API Key 认证 (APIKeyAuthMiddleware)  │
│    从请求头提取 API Key                  │
│    查询 APIKey 表，验证有效性            │
│    检查过期、禁用、IP 白名单等           │
└─────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────┐
│ 2. 用户和订阅验证                        │
│    获取 API Key 关联的用户               │
│    检查用户余额和并发数                  │
│    验证订阅状态（如果绑定了分组）        │
└─────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────┐
│ 3. 请求规范化                           │
│    规范化 API 端点路径                   │
│    识别目标平台（Claude/OpenAI/Gemini） │
│    提取请求体中的模型、消息等            │
└─────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────┐
│ 4. 账号选择和调度                        │
│    获取用户允许的账号列表                │
│    应用粘性会话（sticky session）        │
│    负载感知调度（load-aware）            │
│    选择最优账号                          │
└─────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────┐
│ 5. 请求转发                             │
│    构建上游请求                          │
│    注入账号凭证（API Key/OAuth Token）  │
│    转发到上游 API                        │
│    处理流式响应（SSE）                   │
└─────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────┐
│ 6. 响应处理和计费                        │
│    解析响应中的 Token 使用量             │
│    计算成本（基于定价表）                │
│    记录用量日志                          │
│    扣除用户余额                          │
│    返回响应给客户端                      │
└─────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────┐
│ 7. 故障转移（Failover）                 │
│    如果请求失败，尝试其他账号            │
│    同账号重试（最多 3 次，500ms 间隔）  │
│    切换账号（最多 N 次，可配置）         │
│    临时封禁失败账号                      │
└─────────────────────────────────────────┘
```

## 2. 故障转移机制

**文件**：`backend/internal/handler/failover_loop.go`

### 关键常量

```go
maxSameAccountRetries  = 3       // 同账号重试上限
sameAccountRetryDelay  = 500ms   // 同账号重试间隔
singleAccountBackoffDelay = 2s   // 单账号分组 503 退避延时
```

### FailoverState 结构

```go
SwitchCount          int         // 当前切换次数
MaxSwitches          int         // 最大切换次数（可配置）
FailedAccountIDs     map[int]bool // 失败账号集合
SameAccountRetryCount map[int]int // 各账号重试计数
ForceCacheBilling    bool        // 是否强制缓存计费
hasBoundSession      bool        // 是否有绑定会话
```

### 故障转移策略

1. **同账号重试**：最多 3 次，间隔 500ms
2. **临时封禁**：同账号重试耗尽后，调用 `TempUnscheduleRetryableError` 临时封禁该账号
3. **账号切换**：支持最多 N 次切换（可配置）
4. **Antigravity 延时**：每次切换递增延时（第 1 次 0s，第 2 次 1s，第 3 次 2s...）
5. **单账号退避**：503 错误时，清除排除列表后等待 2 秒重试

## 3. 认证授权流程

### 邮箱/密码认证

```
用户输入邮箱密码
  → 验证邮箱和密码
  → 生成 JWT Token
  → 返回 Token 给前端
  → 前端存储 Token（localStorage）
  → 后续请求在 Authorization 头中携带 Token
```

### OAuth 认证

支持的提供商：
- Email OAuth（通用）
- LinuxDo OAuth
- WeChat OAuth（微信扫码）
- OIDC OAuth（OpenID Connect）
- GitHub OAuth
- Google OAuth

```
用户点击 OAuth 登录
  → 重定向到 OAuth 提供商
  → 用户授权
  → 提供商回调到 Sub2API
  → 交换授权码获取 Access Token
  → 获取用户信息
  → 创建或更新本地用户
  → 生成 JWT Token
  → 重定向回前端，携带 Token
```

## 4. 支付流程

支持的支付方式：支付宝、微信支付、Stripe、EasyPay

```
用户选择充值金额
  → 选择支付方式
  → 创建支付订单（PaymentOrder）
  → 调用支付提供商 API
  → 获取支付链接/二维码
  → 用户完成支付
  → 支付提供商回调 Webhook
  → 验证签名
  → 更新订单状态为 PAID
  → 增加用户余额
  → 记录审计日志
```

## 5. 用量计费流程

```
请求到达网关
  → 转发到上游 API
  → 获取响应中的 Token 使用量
  → 查询定价表（基于平台、模型、Token 类型）
  → 计算成本 = input_tokens × input_price + output_tokens × output_price
  → 应用账号级别的计费倍数（billing_rate_multiplier）
  → 应用用户级别的计费倍数
  → 记录 UsageLog
  → 异步扣除用户余额
  → 更新 API Key 的 quota_used
```

## 6. 并发控制

**文件**：`backend/internal/repository/concurrency_cache.go`

使用 Redis 有序集合（Sorted Set）实现：

- **O(1) 计数**：使用 `ZCARD` 原子获取并发数
- **自动过期清理**：`ZREMRANGEBYSCORE` 清理过期槽位
- **多实例时钟同步**：使用 Redis `TIME` 命令而非本地时间
- **支持重试刷新**：已存在的 requestID 可刷新时间戳

### Redis 键格式

```
concurrency:account:{accountID}   账号级并发槽位
concurrency:user:{userID}         用户级并发槽位
concurrency:wait:{userID}         等待队列计数
wait:account:{accountID}          账号级等待队列
```

### Lua 脚本

- `acquireScript`：原子获取槽位（检查上限、清理过期、添加新槽位）
- `getCountScript`：统计并清理过期条目
- `incrementWaitScript`：增加等待队列计数
- `decrementWaitScript`：减少等待队列计数

### 图片生成并发限制

使用内存实现（`sync.Mutex` + 通知通道），支持等待模式和拒绝模式。
