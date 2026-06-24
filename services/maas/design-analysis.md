---
id: "design-maas-001"
title: "MaaS 模型调度层设计分析"
aliases: ["MaaS设计", "模型调度层", "Model as a Service"]
type: "design"
category: "services/maas"
tags: ["maas", "模型调度", "智能路由", "配额管控", "架构设计"]
version: "1.0.0"
created: "2026-05-03"
updated: "2026-05-03"
author: "jxncyjq"
status: "draft"
parent: "[[Legion]]"
related_docs: ["[[design-app-整体架构-001]]"]
---

# MaaS 模型调度层设计分析

> 基于 [[Legion]] 项目方案 V3.0 第 3.1 节的分析展开

---

## 一、在系统中的定位

MaaS（Model as a Service）是 Legion 三大自研引擎中的**基础设施层**，位于 Agent 引擎和 LLM Wiki 之下。核心使命：**将"模型选择"从 Agent 定义中解耦**，由平台统一调度。

```
Agent 引擎（上层消费者）
    ↓ 请求推理资源
MaaS 模型调度层（中间调度）
    ↓ 路由 + 配额 + 追踪
模型注册中心（底层资源池）
```

---

## 二、设计目标

解决的核心矛盾：**不同智能体执行不同任务时，对模型能力的需求差异巨大**。

| 场景 | 需求 | MaaS 策略 |
|------|------|-----------|
| CEO Agent 做战略决策 | 最强推理能力 | 路由到 Opus 级模型 |
| 客服 Agent 日常回复 | 低成本、低延迟 | 路由到轻量模型 |
| 代码生成 | 专用模型能力 | 路由到代码特化模型 |

关键设计意图：**成本最优 + 能力最优**的智能分配，两者同时追求。

---

## 三、模型注册中心

### 3.1 模型能力标签

每个注册模型携带标准化标签：

- 推理强度等级
- 上下文窗口大小
- 支持的模态（文本/图像/代码）
- 每 Token 单价
- 平均延迟

### 3.2 四类模型来源

| 来源 | 接入方式 | 典型场景 |
|------|----------|----------|
| 云端商业模型 | API Key + Endpoint 注册 | OpenAI / Anthropic / DeepSeek / 文心 / 通义 |
| 企业私有部署 | 内网 Endpoint 注册 | vLLM / TGI 部署的企业专属模型 |
| 本地推理 | 本机进程自动发现 | Ollama / llama.cpp / LM Studio |
| 特化模型 | 模型仓库拉取 | 代码生成专用、图像理解、嵌入模型 |

设计要点：
- 四类来源覆盖从公有云到本地到专用的完整谱系
- 本地推理采用"自动发现"降低接入成本
- 特化模型单独分类，应对通用模型在特定领域的不足

---

## 四、智能路由引擎

支持三种路由策略：

### 策略一：按角色匹配

```yaml
role_based:
  ceo:               { prefer: "claude-opus-4",   fallback: "gpt-4o" }
  developer:         { prefer: "claude-sonnet-4", fallback: "deepseek-coder" }
  customer_service:  { prefer: "deepseek-chat",   fallback: "qwen-turbo" }
```

每个角色绑定首选模型 + 降级模型。

### 策略二：按任务类型匹配

```yaml
task_based:
  strategic_planning: { min_reasoning_level: "high",  prefer_cost: false }
  code_generation:    { require_capability: "code",    prefer_cost: true }
  content_writing:    { min_reasoning_level: "medium", prefer_cost: true }
  data_extraction:    { min_reasoning_level: "low",    prefer_cost: true }
```

引入两个维度：**能力门槛**（min_reasoning_level / require_capability）和 **成本偏好**（prefer_cost）。

### 策略三：按预算约束降级

```yaml
budget_aware:
  when_budget_remaining < 20%: { force_downgrade: true, max_tier: "medium" }
  when_budget_remaining < 5%:  { force_downgrade: true, max_tier: "low" }
```

兜底策略，预算紧张时的全局降级规则。

### 策略冲突解决（推断）

1. **预算约束最高优** — 预算耗尽时强制降级
2. **任务类型优先于角色** — 具体任务需求比身份标签更精确
3. **角色为兜底匹配** — 任务类型未声明时使用角色默认配置

---

## 五、配额管控引擎

多维度、带弹性的管控体系：

| 层级 | 维度 | 说明 |
|------|------|------|
| 公司级 | 全公司月预算 | 顶层总额 |
| 部门级 | 按部门月预算上限 | 组织维度分配 |
| Agent 级 | 月度 / 单次调用上限 | 个体粒度控制 |
| 模型级 | 高成本模型频次限制 | 如 Opus 级每日 N 次 |
| 弹性 | 三级熔断 | 告警 → 自动降级 → 超支冻结 |

五层维度形成"金字塔"管控结构，从粗到细。三级熔断呈递进式，给业务留缓冲而非直接切断。

---

## 六、三原则落地

### 可观测

- 模型调用链路全程追踪
- 哪个 Agent、执行什么任务、路由到哪个模型
- 消耗多少 Token、响应时间多少
- 全部可查

### 可治理

- 路由策略可随时调整
- 管理员可手动指定某 Agent 必须使用某模型
- 管理员可手动禁止某 Agent 使用某模型

### 可控风险

- 三级熔断：告警 → 自动降级 → 超支冻结
- 模型降级不中断业务（只降低推理质量）
- 预算不会失控

关键设计思路：**降级不中断** — 宁可用弱模型继续干活，也不能断业务。

---

## 七、模块间数据流

```
Agent 引擎 → MaaS
  请求: { agent_role, task_type, budget_context }

MaaS 内部:
  1. 查模型注册中心 → 候选模型列表
  2. 跑路由策略 → 择优选择
  3. 查配额管控 → 是否允许/需要降级
  4. 生成调用记录 → 写入审计日志

MaaS → Agent 引擎
  响应: { selected_model, endpoint, cost_estimate }

MaaS → 计量系统
  写入: { agent_id, model_id, tokens_used, cost, latency }
```

---

## 八、待明确的设计问题

| 问题 | 说明 |
|------|------|
| 路由策略优先级 | 三种策略冲突时的明确优先顺序 |
| 模型注册标签的扩展性 | 能力标签体系是否支持自定义标签 |
| 多模型并发调用 | 一个 Agent 是否可以同时使用多个模型 |
| 模型热替换 | Agent 运行中能否不中断地切换模型 |
| 与 Agent 引擎的记忆联动 | MaaS 的路由决策是否会参考 Agent 的历史表现 |
| 计量数据的存储策略 | 调用日志的保留周期、聚合策略 |

---

## 九、与现有代码的对应关系

现有 `legion/maas/` 目录下已包含 7 个微服务：

| 服务 | 推测职责 |
|------|----------|
| TopModelsLogin | 用户认证与登录 |
| TopModelsPlatform | 平台管理后台（模型 CRUD、供应商管理） |
| TopModelsNode | 节点服务（模型部署节点管理） |
| TopModelsService | 核心调度服务（路由 + 调用） |
| TopModelsBilling | 计费与配额管理 |
| TopModelsLogs | 调用日志与审计 |
| TopModelsStat | 统计分析（用量、趋势） |

核心数据模型（`models_info`、`models_provider`、`user_models_infos`）已定义，使用 xorm 作为 ORM，支持分表。
