---
id: "analysis-sub2api-configuration-001"
title: "Sub2API 配置系统"
type: "analysis"
category: "design/analysis"
tags: ["sub2api", "configuration"]
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

# Sub2API 配置系统

## 配置文件

**文件**：`backend/internal/config/config.go`（2766 行）

支持 YAML 配置文件和环境变量两种方式。

## 主要配置模块

| 配置类 | 用途 |
|------|------|
| `ServerConfig` | 服务器绑定地址、端口、H2C 配置 |
| `LogConfig` | 日志级别、格式、输出、轮转、采样 |
| `DatabaseConfig` | PostgreSQL 连接池参数 |
| `RedisConfig` | Redis 连接池、密码、TLS |
| `GatewayConfig` | 网关超时、连接池隔离、响应体限制 |
| `ConcurrencyConfig` | 并发限制配置 |
| `ImageConcurrencyConfig` | 图片生成并发隔离 |
| `GatewaySchedulingConfig` | 调度缓存、粘性会话、受控回源 |
| `BillingConfig` | 计费相关配置 |
| `JWTConfig` | JWT 密钥、过期时间 |
| `TotpConfig` | 2FA 加密密钥 |
| `GeminiConfig` | Gemini OAuth、配额策略 |
| `DashboardAggregationConfig` | 仪表盘预聚合 |
| `UsageCleanupConfig` | 使用记录清理 |

## 关键配置项

### 服务器配置

```bash
SERVER_PORT=8080
SERVER_MODE=release
SERVER_H2C_ENABLED=true
SERVER_H2C_MAX_CONCURRENT_STREAMS=50
```

### 数据库配置

```bash
DATABASE_MAX_OPEN_CONNS=256
DATABASE_MAX_IDLE_CONNS=128
DATABASE_CONN_MAX_LIFETIME_MINUTES=30
DATABASE_CONN_MAX_IDLE_TIME_MINUTES=5
```

### Redis 配置

```bash
REDIS_POOL_SIZE=4096
REDIS_MIN_IDLE_CONNS=256
```

### 网关配置

```bash
GATEWAY_MAX_CONNS_PER_HOST=2048
GATEWAY_MAX_IDLE_CONNS=8192
GATEWAY_MAX_IDLE_CONNS_PER_HOST=4096
```

### 网关调度配置

```bash
# 粘性会话
GATEWAY_SCHEDULING_STICKY_SESSION_MAX_WAITING=3
GATEWAY_SCHEDULING_STICKY_SESSION_WAIT_TIMEOUT=120s

# 兜底回源
GATEWAY_SCHEDULING_FALLBACK_WAIT_TIMEOUT=30s
GATEWAY_SCHEDULING_FALLBACK_MAX_WAITING=100
GATEWAY_SCHEDULING_DB_FALLBACK_ENABLED=true
GATEWAY_SCHEDULING_DB_FALLBACK_TIMEOUT_SECONDS=0
GATEWAY_SCHEDULING_DB_FALLBACK_MAX_QPS=0

# Outbox 轮询
GATEWAY_SCHEDULING_OUTBOX_POLL_INTERVAL_SECONDS=1
GATEWAY_SCHEDULING_OUTBOX_LAG_WARN_SECONDS=5
GATEWAY_SCHEDULING_OUTBOX_LAG_REBUILD_SECONDS=10
GATEWAY_SCHEDULING_OUTBOX_LAG_REBUILD_FAILURES=3
GATEWAY_SCHEDULING_OUTBOX_BACKLOG_REBUILD_ROWS=10000

# 全量重建
GATEWAY_SCHEDULING_FULL_REBUILD_INTERVAL_SECONDS=300
```

### 图片生成并发配置

```bash
GATEWAY_IMAGE_CONCURRENCY_ENABLED=false
GATEWAY_IMAGE_CONCURRENCY_MAX_CONCURRENT_REQUESTS=0   # 0=无限
GATEWAY_IMAGE_CONCURRENCY_OVERFLOW_MODE=reject         # reject / wait
GATEWAY_IMAGE_CONCURRENCY_WAIT_TIMEOUT_SECONDS=30
GATEWAY_IMAGE_CONCURRENCY_MAX_WAITING_REQUESTS=100
```

### 仪表盘聚合配置

```bash
DASHBOARD_AGGREGATION_ENABLED=true
DASHBOARD_AGGREGATION_INTERVAL_SECONDS=60
DASHBOARD_AGGREGATION_LOOKBACK_SECONDS=120
DASHBOARD_AGGREGATION_RECOMPUTE_DAYS=2
DASHBOARD_AGGREGATION_RETENTION_USAGE_LOGS_DAYS=90
DASHBOARD_AGGREGATION_RETENTION_HOURLY_DAYS=180
DASHBOARD_AGGREGATION_RETENTION_DAILY_DAYS=730
```

### 安全配置

```bash
JWT_SECRET=<随机字符串>
TOTP_ENCRYPTION_KEY=<32字节密钥>
```

## 部署配置（Docker Compose）

**文件**：`deploy/docker-compose.yml`

### 多阶段构建（Dockerfile）

```
Stage 1: Frontend Builder (Node 24)
  安装 pnpm → 安装依赖 → 构建前端

Stage 2: Backend Builder (Go 1.26.3)
  下载 Go 依赖 → 复制前端 dist → 编译二进制（embed 前端）

Stage 3: PostgreSQL Client
  复制 pg_dump/psql

Stage 4: Final Runtime (Alpine 3.21)
  安装运行时依赖 → 创建非 root 用户 → 复制二进制和资源 → 健康检查
```

### Docker Compose 服务

**Sub2API 应用**：
- 镜像：`weishaw/sub2api:latest`
- 端口：8080
- 卷：`/app/data`（数据持久化）
- 依赖：PostgreSQL、Redis

**PostgreSQL 18**：
- 端口：5432
- 卷：`postgres_data`
- 健康检查：`pg_isready`

**Redis 8**：
- 端口：6379
- 卷：`redis_data`
- 持久化：RDB + AOF
- 健康检查：`redis-cli ping`

## 生产环境建议

**资源配置**：
- PostgreSQL：`max_connections=1024`，`shared_buffers=1GB`
- Redis：`maxclients=50000`，`pool_size=4096`
- 应用：`DATABASE_MAX_OPEN_CONNS=256`，`REDIS_MIN_IDLE_CONNS=256`

**安全配置**：
- 设置固定 `JWT_SECRET` 和 `TOTP_ENCRYPTION_KEY`
- 启用 URL 白名单验证（生产环境）
- 配置 CSP 和安全头
- 使用 HTTPS（生产环境）
