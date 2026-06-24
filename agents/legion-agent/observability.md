---
id: "agent-observability-001"
title: "Legion Agent 外部观测接口"
type: "guide"
category: "agents/legion-agent"
tags: ["agent", "observability", "metrics", "prometheus", "p11"]
version: "0.1.0"
created: "2026-05-16"
updated: "2026-05-16"
status: "draft"
related_docs:
  - path: "../../plans/03-agent/p11-platform-integration-plan.md"
    relation: "implements"
---

# Legion Agent 外部观测接口

Agent 的 `/metrics` 接口支持两种格式：

| 请求 | 格式 | 用途 |
|------|------|------|
| `GET /metrics` | JSON | 调试、诊断页面、脚本读取 |
| `GET /metrics?format=prometheus` | Prometheus text format | 外部监控系统抓取 |

两种格式都需要管理令牌。

## Prometheus 指标

| 指标 | 标签 | 说明 |
|------|------|------|
| `legion_agent_http_requests_total` | `status` | HTTP 响应状态计数 |
| `legion_agent_tasks_total` | `status` | task 状态计数 |
| `legion_agent_model_calls_total` | `status` | model 调用结果计数 |
| `legion_agent_approvals_total` | `status` | approval 结果计数 |
| `legion_agent_workflows_total` | `status` | workflow 状态计数 |

## 敏感信息规则

Prometheus 输出只包含低基数聚合计数，不输出 prompt、tool input、secret、api key、request body 或模型响应正文。

## 验证命令

```powershell
go test ./internal/server -run TestMetricsPrometheusFormat -count=1
```
