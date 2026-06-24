---
id: "spec-component-audit-logger-062"
title: "AuditLogger 组件规范"
aliases: ["AuditLogger规范", "审计日志", "audit-logger-spec"]
type: "spec"
category: "design/architecture/components"
tags: ["component-spec", "observability", "audit", "compliance", "maas"]
version: "0.1.0"
created: "2026-05-09"
updated: "2026-05-09"
author: "jxncyjq"
status: "draft"
parent: "spec-component-registry-000"

component_id: "C62"
layer: "L7"
depends_on: []
optional_deps: []
conflicts_with: []
required_by: []
assembly_profiles:
  - minimal
  - standard
  - enterprise
  - embedded
---

<!-- @section: overview -->
# AuditLogger 组件规范

## 1. 组件定位

`AuditLogger` 记录每次推理请求的**完整审计日志**，用于合规、计费核对、用量分析和问题排查。它是所有装配方案（包括 `minimal`）的必须组件。

```
请求完成（成功或失败）
        │
        ▼
AuditLogger.Log(event)
        │
        ├── PostgreSQL / ClickHouse（持久化存储）
        └── stdout（开发模式）
```

<!-- @end-section -->

<!-- @section: interface -->
---

## 2. 接口定义

```go
// AuditLogger 记录请求的完整审计事件。
// 并发安全：所有方法可被多个 goroutine 并发调用。
type AuditLogger interface {
    // Log 记录一条审计事件（异步写入，不阻塞请求响应）。
    Log(ctx context.Context, event *AuditEvent)
}

// AuditEvent 单次请求的完整审计记录。
type AuditEvent struct {
    // 身份
    RequestID   string
    UserID      string
    TenantID    string
    AgentID     string
    KeyID       string       // API Key ID

    // 路由结果
    ProviderID  string
    ModelID     string       // 框架规范名
    Strategy    string       // 命中的路由策略

    // 用量
    Usage       TokenUsage
    Cost        int64        // 实际消耗配额单位（来自 BillingSession.Snapshot）

    // 时序
    RequestAt   time.Time
    ResponseAt  time.Time
    Latency     time.Duration

    // 状态
    StatusCode  int
    ErrorCode   string       // 空字符串表示成功
    FailoverCount int

    // 内容（按合规要求配置是否记录）
    PromptHash  string       // SHA-256(prompt)，默认不存明文
    OutputHash  string       // SHA-256(output)
}
```

<!-- @end-section -->

<!-- @section: storage-backends -->
---

## 3. 存储后端

| 后端 | 适用场景 | 说明 |
|------|----------|------|
| `postgres` | 标准生产 | 写入 `inference_logs` 表，支持精确查询 |
| `clickhouse` | 大规模分析 | 列存储，适合亿级日志的聚合分析 |
| `stdout` | 开发/调试 | JSON 格式打印，不持久化 |

**PostgreSQL 表结构**：

```sql
CREATE TABLE inference_logs (
    id            BIGSERIAL    PRIMARY KEY,
    request_id    VARCHAR(64)  NOT NULL UNIQUE,
    user_id       VARCHAR(64)  NOT NULL,
    tenant_id     VARCHAR(64)  NOT NULL,
    provider_id   VARCHAR(64)  NOT NULL,
    model_id      VARCHAR(128) NOT NULL,
    input_tokens  INT          NOT NULL DEFAULT 0,
    output_tokens INT          NOT NULL DEFAULT 0,
    cost_units    BIGINT       NOT NULL DEFAULT 0,
    status_code   INT          NOT NULL,
    error_code    VARCHAR(64),
    latency_ms    INT          NOT NULL,
    failover_count INT         NOT NULL DEFAULT 0,
    request_at    TIMESTAMPTZ  NOT NULL,
    created_at    TIMESTAMPTZ  NOT NULL DEFAULT NOW()
);

CREATE INDEX ON inference_logs (user_id, request_at DESC);
CREATE INDEX ON inference_logs (tenant_id, request_at DESC);
```

<!-- @end-section -->

<!-- @section: behavior-contracts -->
---

## 4. 行为契约

| 契约 | 说明 |
|------|------|
| **异步写入** | Log 立即返回，写入在后台 goroutine 执行 |
| **写入失败不影响请求** | 后端不可用时记录 warn 日志，不向调用方传播错误 |
| **默认不存明文内容** | prompt/output 默认只存 SHA-256 哈希；存明文需显式配置 |
| **幂等写入** | 以 request_id 为唯一键，重复写入同一 request_id 安全（ON CONFLICT DO NOTHING）|

<!-- @end-section -->

<!-- @section: checklist -->
---

## 5. 实现检查清单

```
AuditLogger
  ☐ Log：异步写入（不阻塞请求响应）
  ☐ 写入失败静默（记录 warn，不传播错误）
  ☐ 以 request_id 为幂等键
  ☐ 默认不存明文 prompt/output（只存哈希）
  ☐ 支持 postgres / clickhouse / stdout 三种后端

PostgreSQL 后端
  ☐ 批量写入（减少 DB 连接开销）
  ☐ 写入队列满时丢弃或阻塞（可配置）

测试
  ☐ 幂等写入（重复 request_id 安全）
  ☐ 后端不可用时静默失败
  ☐ 并发安全
```

<!-- @end-section -->

<!-- @section: related -->
---

## 相关文档

- [[component-registry|组件注册表与依赖关系图]]（C62 在依赖图中的位置）
- billing-session-spec.md（C31，提供 BillingSnapshot 用于 cost 字段）
- stream-proxy-spec.md（C50，提供 FullContent 用于内容审计）

<!-- @end-section -->
