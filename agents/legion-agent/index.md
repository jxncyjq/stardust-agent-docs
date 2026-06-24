---
id: "agent-index-001"
title: "Legion Agent 文档索引"
type: "index"
category: "backend/agent"
tags: ["agent", "index", "navigation"]
version: "1.0.0"
created: "2026-06-24"
updated: "2026-06-24"
author: "jxncyjq"
status: "published"
parent: null
children: []
related_docs:
  - id: "agent-http-api-001"
    relation: "related_to"
    path: "./http-api.md"
---

# Legion Agent 文档索引

| 文档 | 说明 |
|------|------|
| [agents-md.md](./agents-md.md) | `AGENTS.md` 的作用、查看方式、放置位置和当前注意事项 |
| [mvp.md](./mvp.md) | Sprint 01 MVP 能力说明 |
| [package-structure.md](./package-structure.md) | Legion Agent Go 包结构说明 |
| [e2e-smoke.md](./e2e-smoke.md) | 端到端 smoke 验收命令 |
| [configuration.md](./configuration.md) | JSON 配置文件与环境变量覆盖说明 |
| [http-api.md](./http-api.md) | `agent serve` HTTP API 说明 |
| [openapi.md](./openapi.md) | P11 OpenAPI 3.1 契约、兼容性门禁和敏感信息规则 |
| [persona-files.md](./persona-files.md) | `AGENTS.md`、`SOUL.md`、`TOOLS.md`、`USER.md`、`MEMORY.md` 的运行时加载、扫描、截断与注入说明 |
| [events.md](./events.md) | P11 平台 EventBus 与 `/v1/events` SSE 订阅说明 |
| [security-tenancy.md](./security-tenancy.md) | P11 tenant/company 访问边界与跨 company 拒绝审计说明 |
| [observability.md](./observability.md) | P11 `/metrics` JSON 与 Prometheus text format 外部观测说明 |
| [data-retention.md](./data-retention.md) | P11 SQLite 数据保留、质量历史清理与运行/审计快照导出说明 |
| [governance-rbac.md](./governance-rbac.md) | P12 RBAC、company 过滤、audit/quality 查询边界说明 |
| [skill-registry.md](./skill-registry.md) | P12 远端 Skill registry 同步、hash 校验、扫描与 quarantine 说明 |
| [model-profiles.md](./model-profiles.md) | MaaS 多模型 profile 配置、启动时 profile 选择优先级与路由规则参考手册 |
| [traces.md](./traces.md) | P12 trace snapshot、脱敏与调试接口说明 |
| [api-errors.md](./api-errors.md) | P12 OpenAPI 错误响应矩阵与客户端兼容说明 |
| [ci.md](./ci.md) | GitHub Actions CI 流水线说明 |
| [storage-ops.md](./storage-ops.md) | SQLite schema、备份和恢复运维说明 |
| [sqlite-schema.md](./sqlite-schema.md) | `agent.db` SQLite 表结构、字段含义和主要数据流说明 |
| [release.md](./release.md) | 版本命令、发布构建产物和回滚说明 |
| [operations.md](./operations.md) | 部署、健康检查、诊断、备份恢复和回滚 Runbook |
| [component-parity.md](./component-parity.md) | P10 组件实现状态、测试覆盖和降级策略对齐说明 |
| [../specs/2026-05-18-multi-agent-implementation-clarification.md](../specs/2026-05-18-multi-agent-implementation-clarification.md) | 多 Agent 代码实现澄清：当前边界、关键缺口和 per-agent runtime routing 落地顺序 |
