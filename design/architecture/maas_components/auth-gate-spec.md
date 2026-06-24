---
id: "spec-component-auth-gate-010"
title: "AuthGate 组件规范"
aliases: ["AuthGate规范", "认证网关", "auth-gate-spec"]
type: "spec"
category: "design/architecture/components"
tags: ["component-spec", "auth", "pipeline", "security", "maas"]
version: "0.1.0"
created: "2026-05-09"
updated: "2026-05-09"
author: "jxncyjq"
status: "draft"
parent: "spec-component-registry-000"

component_id: "C10"
layer: "L2"
depends_on: []
optional_deps: []
conflicts_with:
  - "embedded"  # embedded 装配方案中禁用（调用方自行处理认证）
required_by: []
assembly_profiles:
  - standard
  - enterprise
---

<!-- @section: overview -->
# AuthGate 组件规范

## 1. 组件定位

`AuthGate` 是请求管道的**认证守门人**，在请求进入业务逻辑前验证调用方身份，并将身份信息填充到 `RequestContext` 中供后续节点使用。

它是请求管道的第一个节点，认证失败立即终止请求（返回 401/403），不进入后续流程。

```
HTTP 请求
    │
    ▼
AuthGate（第一个管道节点）
    │ 验证 API Key / Token
    ├── 认证失败 → 401/403，终止
    │
    ▼ 认证成功
    │ 填充 RequestContext.CallerIdentity
    │
    ▼
TenantContextLoader → RateLimiter → ... → ProviderExecutor
```

**读者**：实现认证逻辑的安全工程师、集成 AuthGate 的管道开发者。

<!-- @end-section -->

<!-- @section: interface -->
---

## 2. 接口定义

```go
// AuthGate 验证请求方身份并填充调用者信息到 RequestContext。
// 并发安全：同一实例可在多个 goroutine 中并发调用。
type AuthGate interface {
    // Authenticate 验证请求携带的凭据。
    // 成功时返回 CallerIdentity，框架将其填入 RequestContext。
    // 失败时返回 *AuthError（含错误类型和原因）。
    Authenticate(ctx context.Context, req *IncomingRequest) (*CallerIdentity, error)
}

// IncomingRequest 是 AuthGate 接收的请求摘要（框架从 HTTP 请求中提取）。
type IncomingRequest struct {
    // APIKey 来自 Authorization: Bearer <key> 或 x-api-key 头部
    APIKey string
    // Token 来自 Authorization: Bearer <token>（JWT 场景）
    Token string
    // ClientIP 原始客户端 IP（经过代理时从 X-Forwarded-For 解析）
    ClientIP string
    // Path 请求路径（用于路径级权限检查）
    Path string
    // Method HTTP 方法
    Method string
}

// CallerIdentity 认证成功后的调用者身份信息。
type CallerIdentity struct {
    UserID     string   // 用户唯一 ID
    TenantID   string   // 租户 ID（多租户场景）
    AgentID    string   // Agent ID（可选，特定 Agent 的 API Key）
    Scopes     []string // 权限范围，如 ["chat", "embeddings"]
    KeyID      string   // API Key ID（用于审计日志）
    ExpiresAt  *time.Time // nil 表示永不过期
}
```

<!-- @end-section -->

<!-- @section: auth-methods -->
---

## 3. 认证方式

### 3.1 API Key 认证（主要方式）

```go
// APIKeyAuthGate 验证数据库中存储的 API Key。
// API Key 格式：sk-{base62_random_32chars}（类似 OpenAI 格式）
//
// 验证流程：
//   1. 从请求头提取 API Key
//   2. 计算 Key 的 SHA-256 哈希
//   3. 查询数据库（使用缓存，TTL=5min）
//   4. 验证 Key 未过期、未吊销
//   5. 返回 CallerIdentity
type APIKeyAuthGate struct {
    db    *sql.DB
    cache KeyCache // 内存缓存，减少 DB 查询
}
```

**数据库表**：

```sql
CREATE TABLE api_keys (
    id          VARCHAR(32)  PRIMARY KEY,     -- Key ID（前缀部分）
    key_hash    VARCHAR(64)  NOT NULL UNIQUE, -- SHA-256(api_key)
    user_id     VARCHAR(64)  NOT NULL,
    tenant_id   VARCHAR(64)  NOT NULL,
    agent_id    VARCHAR(64),                  -- 可选，绑定到特定 Agent
    scopes      TEXT[]       NOT NULL DEFAULT '{chat}',
    expires_at  TIMESTAMPTZ,                  -- NULL = 永不过期
    revoked     BOOLEAN      NOT NULL DEFAULT FALSE,
    created_at  TIMESTAMPTZ  NOT NULL DEFAULT NOW(),
    last_used_at TIMESTAMPTZ
);
```

### 3.2 JWT 认证（可选扩展）

```go
// JWTAuthGate 验证 JWT Token（如 OAuth2 access token）。
// 用于与外部 IdP 集成的场景。
type JWTAuthGate struct {
    jwks    JWKSClient // JSON Web Key Set 客户端
    issuer  string
    audience []string
}
```

### 3.3 组合认证（多方式支持）

```go
// CompositeAuthGate 按顺序尝试多种认证方式，第一个成功即返回。
// 用于同时支持 API Key 和 JWT 的场景。
type CompositeAuthGate struct {
    gates []AuthGate
}

func (c *CompositeAuthGate) Authenticate(ctx context.Context, req *IncomingRequest) (*CallerIdentity, error) {
    var lastErr error
    for _, gate := range c.gates {
        identity, err := gate.Authenticate(ctx, req)
        if err == nil {
            return identity, nil
        }
        lastErr = err
    }
    return nil, lastErr
}
```

<!-- @end-section -->

<!-- @section: config -->
---

## 4. 配置 Schema

```go
// AuthGateConfig 是 AuthGate 的配置。
type AuthGateConfig struct {
    // Method 认证方式。
    Method string `default:"api_key" validate:"oneof=api_key jwt composite"`

    // APIKey 仅 method=api_key 时有效。
    APIKey *APIKeyConfig

    // JWT 仅 method=jwt 时有效。
    JWT *JWTConfig

    // Cache API Key 缓存配置（减少 DB 查询）。
    Cache AuthCacheConfig
}

type APIKeyConfig struct {
    // KeyPrefix 期望的 Key 前缀，不匹配则跳过（CompositeAuthGate 场景）。
    KeyPrefix string `default:"sk-"`
}

type JWTConfig struct {
    JWKSURL  string
    Issuer   string
    Audience []string
}

type AuthCacheConfig struct {
    Enabled bool          `default:"true"`
    TTL     time.Duration `default:"5m"`
    MaxSize int           `default:"10000"` // 最多缓存条目数（LRU）
}
```

<!-- @end-section -->

<!-- @section: behavior-contracts -->
---

## 5. 行为契约

| 契约 | 说明 |
|------|------|
| **认证失败立即终止** | Authenticate 返回 error 时，框架不继续执行后续管道节点 |
| **Key Hash 不暴露明文** | 数据库只存储 SHA-256(api_key)，不存储明文 |
| **缓存不缓存吊销状态** | 吊销的 Key 在 TTL 内仍可能通过缓存认证；生产建议短 TTL（≤5min）或实时查询 |
| **last_used_at 异步更新** | 更新最后使用时间是异步操作，不阻塞认证响应 |
| **IP 不作为认证凭据** | ClientIP 仅用于审计和告警，不作为认证决策的依据 |
| **CallerIdentity 不可变** | 返回的 CallerIdentity 填入 RequestContext 后不得修改 |

<!-- @end-section -->

<!-- @section: error-types -->
---

## 6. 错误类型

```go
type AuthError struct {
    Code    AuthErrorCode
    Message string
}

type AuthErrorCode string

const (
    // ErrMissingCredentials 请求未携带任何凭据（无 Authorization 头）。
    // 框架返回 401。
    ErrMissingCredentials AuthErrorCode = "MISSING_CREDENTIALS"

    // ErrInvalidCredentials 凭据格式错误或签名无效。
    // 框架返回 401。
    ErrInvalidCredentials AuthErrorCode = "INVALID_CREDENTIALS"

    // ErrKeyRevoked API Key 已被吊销。
    // 框架返回 401。
    ErrKeyRevoked AuthErrorCode = "KEY_REVOKED"

    // ErrKeyExpired API Key 已过期。
    // 框架返回 401。
    ErrKeyExpired AuthErrorCode = "KEY_EXPIRED"

    // ErrInsufficientScope 权限范围不足（如用 chat Key 访问 embeddings）。
    // 框架返回 403。
    ErrInsufficientScope AuthErrorCode = "INSUFFICIENT_SCOPE"
)
```

<!-- @end-section -->

<!-- @section: testing -->
---

## 7. 测试支持

```go
// RunAuthGateContractTests 验证 AuthGate 实现的行为契约。
func RunAuthGateContractTests(t *testing.T, gate AuthGate) {
    t.Run("Authenticate/ValidKey/ReturnsIdentity", ...)
    t.Run("Authenticate/MissingKey/Returns401", ...)
    t.Run("Authenticate/InvalidKey/Returns401", ...)
    t.Run("Authenticate/RevokedKey/Returns401", ...)
    t.Run("Authenticate/ExpiredKey/Returns401", ...)
    t.Run("Authenticate/InsufficientScope/Returns403", ...)
    t.Run("ConcurrencySafety/ParallelAuthenticate", ...)
}

// MockAuthGate 用于管道单元测试，跳过真实认证。
type MockAuthGate struct {
    Identity *CallerIdentity
    Err      error
}

func (m *MockAuthGate) Authenticate(_ context.Context, _ *IncomingRequest) (*CallerIdentity, error) {
    return m.Identity, m.Err
}
```

<!-- @end-section -->

<!-- @section: checklist -->
---

## 8. 实现检查清单

```
AuthGate
  ☐ 验证 API Key 格式（前缀 + 长度）
  ☐ 存储 SHA-256(key)，不存储明文
  ☐ 检查 revoked / expires_at
  ☐ 检查 scopes 与请求路径匹配
  ☐ last_used_at 异步更新（不阻塞）

缓存
  ☐ LRU 缓存（大小可配置）
  ☐ TTL 过期（可配置，建议 ≤5min）
  ☐ 吊销时主动失效缓存条目（可选但推荐）

JWT（如实现）
  ☐ 从 JWKS URL 动态获取公钥（带缓存）
  ☐ 验证 issuer、audience、exp 声明

测试
  ☐ 运行 RunAuthGateContractTests（全部通过）
  ☐ 缓存命中 vs 缓存未命中的认证路径
  ☐ 并发认证安全验证
```

<!-- @end-section -->

<!-- @section: related -->
---

## 相关文档

- [[component-registry|组件注册表与依赖关系图]]（C10 在依赖图中的位置）
- tenant-context-loader-spec.md（C11，AuthGate 之后加载租户上下文）
- audit-logger-spec.md（C62，记录认证事件）

<!-- @end-section -->
