---
id: "agent-traces-001"
title: "Legion Agent Trace Snapshot"
type: "reference"
category: "backend/agent"
tags: ["agent", "trace", "snapshot"]
version: "1.0.0"
created: "2026-06-24"
updated: "2026-06-24"
author: "jxncyjq"
status: "published"
parent: null
children: []
related_docs:
  - id: "agent-index-001"
    relation: "related_to"
    path: "./index.md"
---

# Legion Agent Trace Snapshot

P12 增加轻量级 `TraceRecorder` 和 `/debug/traces` 调试出口，用于在没有外部 OTel 后端时查看最近一批 Agent 内部 span 摘要。

## HTTP 调试接口

```powershell
agent serve --config agent.json
```

```powershell
Invoke-RestMethod `
  -Uri http://127.0.0.1:8080/debug/traces `
  -Headers @{ Authorization = "Bearer <admin-token>" }
```

返回结构：

```json
{
  "spans": [
    {
      "trace_id": "trace-1",
      "span_id": "span-1",
      "name": "model.generate",
      "started_at": "2026-05-16T10:00:00Z",
      "ended_at": "2026-05-16T10:00:00.020Z",
      "attributes": {
        "component": "runtime",
        "api_key": "[redacted]"
      }
    }
  ]
}
```

## 安全边界

- `/debug/traces` 复用 `AdminToken`，默认不作为公开健康接口暴露。
- trace 在 `Record` 写入时净化敏感属性，`prompt`、`input`、`secret`、`token`、`api_key`、`authorization`、`credential` 等字段不会原样进入 snapshot。
- `Snapshot` 返回 span 与属性 map 的副本，调用方修改返回值不会污染 recorder 内部状态。
- recorder 只保留最近 `MaxSpans` 条 span，默认 100 条，避免调试数据无界增长。

## 适用场景

当前实现定位为 Agent 本地和平台接入前的结构化调试面，适合排查 task、workflow、model call、tool call 的短链路摘要。生产级分布式追踪仍应由后续平台侧 OTel collector 或统一 X00 观测服务承接。
