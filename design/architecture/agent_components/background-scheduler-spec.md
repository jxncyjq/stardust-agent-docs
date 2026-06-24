---
id: "spec-agent-background-scheduler-012"
title: "BackgroundScheduler 组件规范"
aliases: ["BackgroundScheduler规范", "后台调度器", "background-scheduler-spec"]
type: "spec"
category: "design/architecture/agent_components"
tags: ["component-spec", "agent-engine", "scheduler", "background"]
version: "0.1.0"
created: "2026-05-09"
updated: "2026-05-09"
author: "jxncyjq"
status: "draft"
parent: "spec-agent-component-registry-000"

component_id: "A12"
layer: "L1"
depends_on: []
optional_deps: []
conflicts_with: []
required_by: []
assembly_profiles:
  - standard
  - enterprise
---

<!-- @section: overview -->
# BackgroundScheduler 组件规范

## 1. 组件定位

`BackgroundScheduler` 运行 Agent Engine 的系统维护任务，包括 stale task 回收、Aegis 批量审核、GEP 失败扫描、信任分冷却刷新、Eval 漂移检测。

它是平台内部运维调度器，不等同于用户可配置的业务定时工作流。

<!-- @end-section -->

<!-- @section: jobs -->
---

## 2. 标准任务

| Job | 间隔 | 抖动 | 职责 |
|-----|------|------|------|
| `stale_task_requeue` | 60s | 10s | 回收过期锁任务 |
| `quality_review` | 60s | 30s | 批量执行 Aegis 审核 |
| `gep_failure_scan` | 5min | 60s | 扫描失败学习事件并触发 GEP |
| `trust_cooldown_refresh` | 10min | 60s | 懒惰刷新信任冷却状态 |
| `eval_drift_detection` | 1h | 5min | 生成 Layer 4 漂移评估 |

<!-- @end-section -->

<!-- @section: behavior -->
---

## 3. 行为契约

| 契约 | 说明 |
|------|------|
| 防重入 | 每个 Job 使用 AtomicBool 或分布式锁防止重叠执行 |
| 动态开关 | 管理 UI 可启停单个 Job，配置热加载 |
| 分散启动 | 服务启动后按 Job 抖动延迟启动，避免雪崩 |
| 失败可观测 | 记录 last_run_at、last_status、duration、error |
| 不阻塞启动 | 单个 Job 初始化失败不阻止核心 AgentRuntime 启动 |

<!-- @end-section -->
