---
id: "index-components"
title: "组件规范索引"
aliases: ["组件索引", "components-index"]
type: "index"
category: "design/architecture/components"
tags: ["index", "component-spec", "maas"]
version: "0.1.0"
created: "2026-05-09"
updated: "2026-05-09"
author: "jxncyjq"
status: "active"
---

# Legion 组件规范索引

本目录收录 Legion LLM 基础设施抽象层的**全部组件规范文档**。

依赖关系总览、装配方案（Profile）、Noop 行为表请查阅主表文档：
→ **[[component-registry|组件注册表与依赖关系图]]**

全平台 A/C/K/X 跨模块依赖请查阅：
→ **[[../platform-component-registry|平台组件注册表与跨模块依赖]]**

---

## 框架层（L0）

| ID | 组件 | 规范文档 | 说明 |
|----|------|----------|------|
| C00 | ComponentRegistry | [component-registry-impl-spec.md](./component-registry-impl-spec.md) | 注册中心、依赖校验、Noop 注入、生命周期管理 |

> 依赖关系主表：[component-registry.md](./component-registry.md)

---

## 提供商层（L1）

| ID | 组件 | 规范文档 | 说明 |
|----|------|----------|------|
| C01 | ModelProvider | [model-provider-spec.md](./model-provider-spec.md) | 同步推理提供商抽象（构建请求、解析响应、流式） |
| C02 | AsyncTaskProvider | [async-task-provider-spec.md](./async-task-provider-spec.md) | 异步任务提供商（视频/音频/图像生成，轮询调度） |
| C03 | ProviderRegistry | [provider-registry-spec.md](./provider-registry-spec.md) | 管理所有 ModelProvider 实例，支持健康过滤和模型查询 |
| C04 | ModelMapper | [model-mapper-spec.md](./model-mapper-spec.md) | 统一模型名到提供商模型 ID 的别名映射 |
| C05 | ProviderHealthMonitor | [provider-health-monitor-spec.md](./provider-health-monitor-spec.md) | 健康评分（1 - 错误率）、熔断三态、主动探针 |
| C06 | ConcurrencyLimiter | [concurrency-limiter-spec.md](./concurrency-limiter-spec.md) | 提供商并发飞行请求数控制（内存/Redis 信号量） |

---

## 管道层（L2）

| ID | 组件 | 规范文档 | 说明 |
|----|------|----------|------|
| C10 | AuthGate | [auth-gate-spec.md](./auth-gate-spec.md) | API Key / JWT 认证，LRU 缓存，异步 last_used_at |
| C11 | TenantContextLoader | [tenant-context-loader-spec.md](./tenant-context-loader-spec.md) | 租户上下文（计费偏好、路由覆盖、功能开关）加载 |
| C12 | RateLimiter | [rate-limiter-spec.md](./rate-limiter-spec.md) | 固定窗口/滑动窗口/令牌桶限流，Redis Lua 原子 |
| C13 | QuotaChecker | [quota-checker-spec.md](./quota-checker-spec.md) | 请求前只读配额预检（包装 QuotaBudgetManager.Check） |
| C14 | ModelRouter | [model-router-spec.md](./model-router-spec.md) | 策略链路由：角色/任务类型/成本感知/健康感知/固定 |
| C16 | SemanticCache | [semantic-cache-spec.md](./semantic-cache-spec.md) | 向量相似度语义缓存（temperature=0 时生效） |
| C17 | ContentFilter | [content-filter-spec.md](./content-filter-spec.md) | 输入/输出内容安全检查，PII 脱敏，多后端 |
| C19 | TelemetryEmitter | [telemetry-emitter-spec.md](./telemetry-emitter-spec.md) | 请求结束后发射 7 项标准指标到 TraceCollector/MetricsRecorder |

> C15 RequestTransformer、C18 ResponseNormalizer 内置于 ModelProvider，无独立规范。

---

## 路由策略层（L3）

> 所有策略均内置于 ModelRouter（C14），通过策略链组合，无独立规范文档。

| ID | 组件 | 实现位置 | 说明 |
|----|------|----------|------|
| C20 | RoleBasedRouter | 内置于 C14 | 按用户角色选择模型 |
| C21 | TaskTypeRouter | 内置于 C14 | 按任务类型（代码/摘要/翻译）选择模型 |
| C22 | CostAwareRouter | 内置于 C14 | 按成本权重选择最低费用提供商 |
| C23 | HealthAwareRouter | 内置于 C14 | 过滤健康分低于阈值的提供商 |
| C24 | PinnedModelRouter | 内置于 C14 | 租户/用户固定路由到指定模型 |

---

## 计费层（L4）

| ID | 组件 | 规范文档 | 说明 |
|----|------|----------|------|
| C30 | PricingEngine | [pricing-engine-spec.md](./pricing-engine-spec.md) | 三种计价模式（ratio/fixed/tiered_expr），快照一致性 |
| C31 | BillingSession | [billing-session-spec.md](./billing-session-spec.md) | 预扣/结算/退款三阶段，幂等 requestID |
| C32 | FundingSource | [funding-source-spec.md](./funding-source-spec.md) | 资金来源抽象（钱包/订阅/免费额），FallbackFundingSource 组合 |
| C33 | QuotaBudgetManager | [quota-budget-manager-spec.md](./quota-budget-manager-spec.md) | 多级配额（用户→组→租户→全局），RPM via Redis |
| C34 | CircuitBreaker | [circuit-breaker-spec.md](./circuit-breaker-spec.md) | 配额熔断器（CLOSED/OPEN/HALF_OPEN），被 QuotaBudgetManager 使用 |

---

## 可靠性层（L5）

| ID | 组件 | 规范文档 | 说明 |
|----|------|----------|------|
| C40 | FailoverManager | [failover-manager-spec.md](./failover-manager-spec.md) | 故障转移协调，依赖 RetryScheduler，可选 HealthMonitor |
| C41 | RetryScheduler | [retry-scheduler-spec.md](./retry-scheduler-spec.md) | 指数退避+抖动，ProviderHint 读 Retry-After，ErrorClass 映射 |

> C42 ProviderErrorClassifier 内置于 ModelProvider，无独立规范。

---

## 流式处理层（L6）

| ID | 组件 | 规范文档 | 说明 |
|----|------|----------|------|
| C50 | StreamProxy | [stream-proxy-spec.md](./stream-proxy-spec.md) | 双 goroutine SSE 代理（Reader 解析 / Writer 聚合转发） |

> C51 StreamAggregator 内置于 StreamProxy；C52 StreamUsageExtractor 内置于 ModelProvider。

---

## 可观测层（L7）

| ID | 组件 | 规范文档 | 说明 |
|----|------|----------|------|
| C60 | TraceCollector | [trace-collector-spec.md](./trace-collector-spec.md) | 分布式追踪出口（OTLP/Jaeger/Noop），异步批量发送 |
| C61 | MetricsRecorder | [metrics-recorder-spec.md](./metrics-recorder-spec.md) | 指标记录出口（Prometheus/OTLP/Noop），8 项标准指标 |
| C62 | AuditLogger | [audit-logger-spec.md](./audit-logger-spec.md) | 审计日志（postgres/clickhouse/stdout），SHA-256 内容哈希，幂等 |
| C63 | CostAttributor | [cost-attributor-spec.md](./cost-attributor-spec.md) | 成本按维度归因（用户/租户/Agent/模型），小时聚合，预算告警 |

---

## 平台推理端口（L8）

| ID | 组件 | 规范文档 | 说明 |
|----|------|----------|------|
| C70 | MaasInferenceClient | [maas-inference-client-spec.md](./maas-inference-client-spec.md) | Agent/Know/Aegis 调用 MaaS 的稳定推理端口，内部组合路由、provider、流式、过滤、遥测、审计 |

> Agent、Know、Aegis、SemanticExtractor 不直接调用 C14 ModelRouter 或 C01 ModelProvider，统一通过 C70 提交推理请求。

---

## 通用组件（在 common_components 目录）

以下组件被 MaaS、Agent、Know 共同依赖，规范在独立目录维护：

| ID | 组件 | 规范文档 | MaaS 使用方 |
|----|------|----------|-------------|
| X00 | EventBus | [../common_components/event-bus-spec.md](../common_components/event-bus-spec.md) | TelemetryEmitter、StreamProxy 可发布流式/指标事件 |
| X01 | EmbeddingProvider | [../common_components/embedding-provider-spec.md](../common_components/embedding-provider-spec.md) | SemanticCache 可复用统一嵌入端口 |
| X02 | ImmutableAuditLog | [../common_components/immutable-audit-log-spec.md](../common_components/immutable-audit-log-spec.md) | AuditLogger 可把推理审计摘要镜像到不可变证据链 |
| X03 | SafeFetcher | [../common_components/safe-fetcher-spec.md](../common_components/safe-fetcher-spec.md) | MaaS 不直接抓取外部 URL，供上层接入侧使用 |
| X04 | PathGuard | [../common_components/path-guard-spec.md](../common_components/path-guard-spec.md) | MaaS 文件型 provider/批处理适配器可复用路径沙盒 |
| X05 | OutputSanitizer | [../common_components/output-sanitizer-spec.md](../common_components/output-sanitizer-spec.md) | ResponseNormalizer/下游导出适配器可复用输出净化 |

---

## 文档统计

| 层次 | 组件数 | 独立规范数 | 内置（无独立规范）|
|------|--------|-----------|-----------------|
| L0 框架层 | 1 | 1 | 0 |
| L1 提供商层 | 6 | 6 | 0 |
| L2 管道层 | 8 | 8 | 2（C15/C18）|
| L3 路由策略层 | 5 | 0 | 5（均内置于 C14）|
| L4 计费层 | 5 | 5 | 0 |
| L5 可靠性层 | 3 | 2 | 1（C42）|
| L6 流式处理层 | 3 | 1 | 2（C51/C52）|
| L7 可观测层 | 4 | 4 | 0 |
| L8 平台推理端口 | 1 | 1 | 0 |
| **合计** | **36** | **28** | **10** |

---

## 快速入口

| 需求 | 文档 |
|------|------|
| 了解完整依赖关系图 | [component-registry.md](./component-registry.md) |
| 如何注册和启动框架 | [component-registry-impl-spec.md](./component-registry-impl-spec.md) |
| 接入新的 AI 提供商 | [model-provider-spec.md](./model-provider-spec.md) |
| 接入异步生成任务 | [async-task-provider-spec.md](./async-task-provider-spec.md) |
| 配置计费与配额 | [billing-session-spec.md](./billing-session-spec.md) · [pricing-engine-spec.md](./pricing-engine-spec.md) · [quota-budget-manager-spec.md](./quota-budget-manager-spec.md) |
| 配置路由策略 | [model-router-spec.md](./model-router-spec.md) |
| 配置可靠性（重试/故障转移） | [failover-manager-spec.md](./failover-manager-spec.md) · [retry-scheduler-spec.md](./retry-scheduler-spec.md) |
| 配置可观测性 | [telemetry-emitter-spec.md](./telemetry-emitter-spec.md) · [trace-collector-spec.md](./trace-collector-spec.md) · [metrics-recorder-spec.md](./metrics-recorder-spec.md) |
| 配置安全（认证/内容过滤） | [auth-gate-spec.md](./auth-gate-spec.md) · [content-filter-spec.md](./content-filter-spec.md) |
| 给 Agent/Know 调用推理 | [maas-inference-client-spec.md](./maas-inference-client-spec.md) |
| 了解跨模块依赖 | [../platform-component-registry.md](../platform-component-registry.md) |
