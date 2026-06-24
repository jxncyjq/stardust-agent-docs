---
id: "analysis-newapi-modules-002"
title: "New API Go 模块功能分析"
aliases: ["new-api modules", "新API模块分析"]
type: "analysis"
category: "design/analysis/maas"
tags: ["new-api", "go", "modules", "architecture", "maas"]
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
  - id: "analysis-newapi-channel-003"
    relation: "related_to"
    path: "./03-channel-adapter-system.md"
  - id: "analysis-newapi-billing-004"
    relation: "related_to"
    path: "./04-billing-quota-system.md"
---

<!-- @section: modules -->
# New API Go 模块功能分析

## 模块目录总览

New API 后端按职责分为 15 个核心 Go 包，遵循分层架构：路由层 → 中间件层 → 控制器层 → 服务层 → 模型层。

```
new-api/
├── main.go              ← 入口：Gin 初始化、中间件注册、数据库连接
├── router/              ← HTTP 路由定义（6 个路由模块）
├── middleware/           ← 中间件链（认证、限流、分发、性能检查等，21 个 .go 文件）
├── controller/          ← 请求处理器（67 个文件，relay、user、channel、billing 等）
├── service/             ← 业务逻辑（59 个文件，渠道选择、计费、配额、格式转换）
├── model/               ← 数据模型 + DB 访问（GORM，36 个文件，25 张表）
├── relay/               ← AI API 中继层（核心，196 个文件）
│   ├── channel/         ← 37 个提供商适配器
│   ├── common/          ← RelayInfo 共享上下文
│   ├── helper/          ← 流处理、价格计算、模型映射
│   └── constant/        ← RelayMode 常量定义
├── dto/                 ← 数据传输对象（29 个文件，请求/响应结构体）
├── constant/            ← 全局常量（13 个文件，渠道类型、API 类型、Context Key）
├── setting/             ← 配置管理（48 个文件，比率、模型、操作、系统、性能）
├── common/              ← 共享工具（46 个文件，JSON、加密、Redis、限流、BodyStorage）
├── types/               ← 类型定义（9 个文件，错误、RelayFormat、文件源）
├── logger/              ← 统一日志封装（`SysLog` / `LogError` 等）
├── i18n/                ← 后端国际化（go-i18n，中/英）
├── oauth/               ← OAuth 提供商实现（GitHub、Discord、OIDC 等）
└── pkg/                 ← 内部包（cachex 缓存、ionet 网络、billingexpr 计费表达式）
```

## 各包职责详解

### 1. main.go — 应用入口

**文件**: `new-api/main.go`

**核心流程**:
1. 加载环境变量和配置
2. 初始化数据库（SQLite/MySQL/PostgreSQL 三选一）
3. 初始化 Redis（可选）
4. 初始化 Channel 缓存、Option 配置映射
5. 注册 Gin 全局中间件：`CustomRecovery → RequestId → PoweredBy → I18n → Logger → Session`
6. 调用 `router.SetRouter()` 组装路由
7. 启动 HTTP/HTTPS 服务

**关键设计**:
- 支持主从节点模式（`IS_MASTER_NODE`），从节点跳过 Option 同步和定时任务
- 前端嵌入二进制（Go `embed`），单文件部署
- Session 存储（Cookie 实现，30天有效期）

### 2. router/ — HTTP 路由定义

**文件**:
- `main.go` — 路由组合入口 `SetRouter()`
- `api-router.go` — `/api/*` 管理 API 路由
- `relay-router.go` — `/v1/*` AI 代理路由
- `dashboard.go` — `/v1/dashboard/billing/*` 计费查询路由
- `video-router.go` — 视频生成路由（含即梦/Kling 适配器）
- `web-router.go` — 前端静态资源路由

**路由分组**:
| 路由组 | 前缀 | 用途 |
|--------|------|------|
| API | `/api/*` | 管理后台 CRUD（用户、渠道、令牌、日志） |
| Relay | `/v1/*`, `/v1beta/*` | AI API 代理（OpenAI 兼容接口） |
| Dashboard | `/v1/dashboard/billing/*` | OpenAI 兼容计费查询 |
| Video | `/v1/video/*`, `/v1/videos/*` | 视频生成代理 |
| Web | `/` | 前端 SPA 静态资源 |

**Relay 路由的中间件链** (最核心):
```
CORS → DecompressRequest → BodyStorageCleanup → Stats → RouteTag("relay")
  → SystemPerformanceCheck → TokenAuth → ModelRequestRateLimit → Distribute
  → controller.Relay
```

### 3. middleware/ — 中间件层

**文件清单** (21 个文件):

| 中间件 | 文件 | 功能 |
|--------|------|------|
| `auth.go` | TokenAuth, UserAuth, AdminAuth, RootAuth | 多级认证（Token/Session/OAuth） |
| `distributor.go` | Distribute | 渠道选择与分配 |
| `rate-limit.go` | GlobalWebRateLimit, GlobalAPIRateLimit, CriticalRateLimit 等 | IP 级别限流 |
| `model-rate-limit.go` | ModelRequestRateLimit | 模型级请求频率限制（按用户） |
| `performance.go` | SystemPerformanceCheck | CPU/内存/磁盘过载保护 |
| `gzip.go` | DecompressRequestMiddleware | 请求体解压（gzip/brotli） |
| `body_cleanup.go` | BodyStorageCleanup | 请求体临时文件清理 |
| `stats.go` | StatsMiddleware | 活跃连接计数 |
| `logger.go` | SetUpLogger, RouteTag | 请求日志与路由标签 |
| `recover.go` | RelayPanicRecover | Relay 专用 panic 恢复 |
| `request-id.go` | RequestId | 唯一请求 ID 生成 |
| `i18n.go` | I18n | 语言检测 |
| `cors.go` | CORS, PoweredBy | 跨域配置 |
| `cache.go` | CacheControl | 静态资源缓存头 |
| `disable-cache.go` | DisableCache | API 禁用缓存 |
| `secure_verification.go` | SecureVerificationRequired | 二次安全验证 |
| `turnstile-check.go` | TurnstileCheck | Cloudflare 人机验证 |
| `email-verification-rate-limit.go` | EmailVerificationRateLimit | 邮件发送限流 |
| `jimeng_adapter.go` | JimengAdapter | 即梦 API 适配 |
| `kling_adapter.go` | KlingAdapter | Kling API 适配 |
| `utils.go` | abortWithOpenAiMessage 等 | 工具函数 |

**中间件数据传递**: 通过 `gin.Context.Set()` / `common.SetContextKey()` 在中间件间传递数据，下游通过 `c.Get()` / `common.GetContextKey()` 读取。

### 4. controller/ — 请求处理器

**核心文件**:
- `relay.go` — 核心 Relay 处理逻辑（含重试循环、预扣费、结算）
- `billing.go` — OpenAI 兼容计费端点（subscription/usage）
- `channel-billing.go` — 渠道余额自动更新（OpenAI/SiliconFlow/DeepSeek 等）
- `user.go`, `token.go`, `channel.go`, `log.go`, `option.go` — 管理 CRUD

**Relay 核心流程** (`Relay()`):
```
1. 解析请求 → 2. 生成 RelayInfo → 3. 敏感词检查
  → 4. 估算 Token → 5. 计算价格 → 6. 预扣费
  → 7. 重试循环: 选渠道 → 执行 Relay → 处理错误
  → 8. 结算 → 9. 记录日志
```

### 5. service/ — 业务逻辑

**核心文件**:
- `channel_select.go` — 渠道选择（含 auto 分组跨分组重试）
- `billing.go` — 计费编排（PreConsumeBilling / SettleBilling）
- `billing_session.go` — 计费会话（预扣费-结算-退款生命周期）
- `funding_source.go` — 资金来源抽象（钱包 vs 订阅）
- `quota.go` — 配额计算与 Token 扣减
- `tiered_settle.go` — 阶梯计费结算
- `violation_fee.go` — CSAM 违规罚款
- 格式转换: `openai_to_claude.go`, `openai_to_gemini.go` 等

### 6. model/ — 数据模型

**核心模型** (36 个文件，25 张表):

| 模型 | 文件 | 说明 |
|------|------|------|
| User | user.go | 用户（含 OAuth 绑定、邀请系统） |
| Token | token.go | API 令牌（含模型限制、IP 白名单） |
| Channel | channel.go | 渠道（含多 Key 管理） |
| Ability | ability.go | 渠道能力（渠道×模型×分组交叉表） |
| Log | log.go | 消费/错误日志 |
| Option | option.go | 系统配置键值对（~150 项） |
| Subscription | subscription.go | 订阅计划/订单/实例 |
| Task | task.go | 异步任务（视频/音频生成） |
| Pricing | pricing.go | 定价缓存聚合 |

**数据库兼容**: 通过 `commonGroupCol`/`commonKeyCol`/`commonTrueVal` 等变量兼容 SQLite/MySQL/PostgreSQL 三种数据库。

### 7. relay/ — AI API 中继层

这是整个系统的核心，详细分析见 [[03-channel-adapter-system|渠道适配器系统分析]]。

**顶层文件**:
- `compatible_handler.go` — TextHelper、ImageHelper、AudioHelper 等主处理函数
- `relay_adaptor.go` — 适配器工厂 `GetAdaptor()` / `GetTaskAdaptor()`
- `relay_task.go` — 异步任务提交流程

**子目录**:
- `channel/` — 37 个提供商适配器（每个一个子目录）
- `common/` — RelayInfo（顶层 61 个字段 + 多个嵌入式子结构 `ChannelMeta` / `*TaskRelayInfo` / `ThinkingContentInfo` / `*ClaudeConvertInfo` / `*RerankerInfo` / `*ResponsesUsageInfo` / `PriceData` / `StreamStatus` / `RequestConversionChain`，定义在 `relay/common/relay_info.go:87-181`） |
- `helper/` — 流处理（SSE Scanner）、价格计算、模型映射
- `constant/` — RelayMode 常量（30+ 模式）

### 8. dto/ — 数据传输对象

定义前端与后端之间的请求/响应结构体：
- `GeneralOpenAIRequest`, `ChatCompletionsRequest` 等 OpenAI 格式
- `ClaudeRequest`, `GeminiChatRequest` 等各渠道格式
- `UserSettings`, `ChannelSettings` 等配置结构

### 9. constant/ — 常量定义

- `channel.go` — 48 个 `ChannelType*` 渠道类型常量（OpenAI=1, Anthropic=14, Gemini=24, ...，部分 ID 段位为兼容历史预留空缺）
- `context_key.go` — 46 个 `ContextKey*` 常量（按 `grep -cE '^\s*ContextKey[A-Z]' constant/context_key.go` 统计）
- `api_type.go` — API 类型常量
- `task.go` — 异步任务平台常量

### 10. setting/ — 配置管理

**子包**:
- `ratio_setting/` — 模型倍率、分组倍率、缓存倍率配置
- `billing_setting/` — 阶梯计费表达式存储和冒烟测试
- `model_setting/` — 模型列表配置
- `operation_setting/` — 运营配置（重试策略、错误日志等）
- `system_setting/` — 系统配置（前端 URL、通知等）
- `performance_setting/` — 性能监控阈值

**配置加载**: 环境变量 `.env` → Option 表 → 内存 Map，三层覆盖。

### 11. common/ — 共享工具

- `json.go` — JSON 序列化/反序列化封装（统一入口）
- `redis.go` — Redis 客户端初始化与操作
- `rate-limit.go` — 内存限流器实现（滑动窗口）
- `body_storage.go` — 请求体存储（内存/磁盘双模式，支持多次读取）
- `gin.go` — Context 读写辅助 + 请求体解析
- `crypto.go`, `email.go`, `verification.go` — 加密、邮件、验证码

### 12. pkg/ — 内部通用包

- `cachex/` — 通用缓存抽象（Redis + 内存 LRU 双层缓存 HybridCache）
- `ionet/` — HTTP 代理客户端封装
- `billingexpr/` — 阶梯计费表达式引擎（编译、执行、结算）

### 13. i18n/ — 国际化

使用 `nicksnyder/go-i18n/v2`，支持中文/英文。
语言检测优先级：用户设置 > Accept-Language 请求头 > 默认语言。

### 14. oauth/ — OAuth 提供商

支持 7 种 OAuth 登录方式：
GitHub、Discord、OIDC、微信、Telegram、LinuxDO、自定义 OAuth Provider。

## 包依赖关系

```
main.go
  ├── router/        → middleware/, controller/
  ├── middleware/     → model/, service/, common/, constant/
  ├── controller/    → service/, model/, relay/, common/, dto/
  ├── service/       → model/, common/, constant/, setting/
  ├── relay/         → service/, model/, common/, dto/, constant/
  │   └── channel/   → relay/common/, dto/
  ├── model/         → common/, constant/, dto/
  └── setting/       → common/, model/
```

## 关键设计模式

1. **分层架构**: Router → Middleware → Controller → Service → Model，单向依赖
2. **策略模式 + 工厂模式**: `Adaptor` 接口 + `GetAdaptor()` 工厂实现渠道可插拔
3. **中间件管道**: Gin 中间件链，数据通过 Context 传递
4. **Cache-Aside**: Redis 缓存 + DB 回退 + 异步回填
5. **适配器模式**: 37 个渠道适配器将各异构 API 统一为 OpenAI 格式
6. **模板方法**: Relay 框架定义流程骨架，各渠道适配器实现具体步骤

<!-- @end-section -->

<!-- @section: related -->
## 相关文档

- [[01-overview|New API 项目架构总览]]
- [[03-channel-adapter-system|渠道适配器系统分析]]
- [[04-billing-quota-system|计费与配额系统分析]]
- [[05-middleware-and-flow|中间件与请求流程分析]]
- [[06-data-models|核心数据模型分析]]

<!-- @end-section -->
