---
id: "plans-common-implementation-001"
title: "Common 公共组件实施计划"
type: "plan"
category: "plans/common"
tags: ["plan", "common", "event-bus", "audit", "embedding", "security"]
version: "0.1.0"
created: "2026-05-10"
updated: "2026-05-10"
status: "draft"
---

# Common 公共组件实施计划

## 目标

Common 是 A/C/K 的基础设施层，优先交付可观测和风险边界能力。

## 组件计划

| 组件 | 优先级 | 交付内容 | 验收 |
|------|--------|----------|------|
| X00 EventBus | P0 | 本地内存 pub/sub、事件类型、订阅过滤、Noop | Agent token、任务状态、审批事件可发布 |
| X02 ImmutableAuditLog | P0 | append-only 写入、hash、request_id、实体引用 | 任务、知识、推理审计可关联 |
| X04 PathGuard | P0 | 路径规范化、workspace 限定、敏感文件识别 | Know 扫描和 Agent 工具写入不能越界 |
| X05 OutputSanitizer | P0 | Markdown/HTML/YAML/MCP 输出净化 | 报告和 MCP 输出不产生注入风险 |
| X01 EmbeddingProvider | P1 | 嵌入端口、批量 embed、缓存、Noop | K32/A42/C16 可复用 |
| X03 SafeFetcher | P1 | scheme 限制、SSRF 防护、redirect 复检、大小限制 | K11 外部 URL 接入可控 |

## 实施步骤

1. 定义公共接口和错误码。
2. 提供内存/本地文件/Noop 实现，支撑测试。
3. 接入 MaaS/Agent/Know 的最小链路。
4. 增加生产实现：数据库审计、分布式事件、嵌入 provider。
5. 加契约测试，保证各模块依赖 X 组件时行为一致。

