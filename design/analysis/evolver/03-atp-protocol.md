---
id: "analysis-evolver-atp-003"
title: "ATP Agent 交易协议分析"
aliases: ["ATP protocol", "Agent Transaction", "代理交易协议"]
type: "analysis"
category: "design/analysis/evolver"
tags: ["evolver", "atp", "agent-transaction", "marketplace", "protocol"]
version: "1.0.0"
created: "2026-05-03"
updated: "2026-05-03"
author: "jxncyjq"
status: "review"
parent: "analysis-evolver-overview-001"
related_docs:
  - id: "analysis-evolver-overview-001"
    relation: "parent"
    path: "./01-overview.md"
  - id: "analysis-evolver-gep-002"
    relation: "related_to"
    path: "./02-gep-protocol.md"
---

<!-- @section: overview -->
# ATP Agent 交易协议分析

## 系统概述

ATP (Agent Transaction Protocol) 是 Evolver 的低佣金 Agent-to-Agent 交易网络，为自动化服务交换提供经济层。与 GEP 不同，**ATP 的 8 个模块全部可读**（无混淆）。

ATP 由三个角色组成：
- **Merchant**（商户）: 注册服务、处理订单、提交交付证明
- **Consumer**（消费者）: 下单、验证交付、结算订单
- **AutoBuyer**（自动买家）: 检测能力缺口后自动下单（可选）

## 一、ATP 模块清单

| 模块 | 行数 | 用途 |
|------|------|------|
| `index.js` | 29 | 模块聚合导出 |
| `hubClient.js` | 255 | Hub ATP API 客户端（双传输） |
| `merchantAgent.js` | 118 | 商户代理模板 |
| `consumerAgent.js` | 157 | 消费者代理模板 |
| `serviceHelper.js` | 99 | 服务发布/更新助手 |
| `defaultHandler.js` | 69 | 默认订单处理 + ATP 配置 |
| `autoBuyer.js` | 242 | 可选能力缺口自动买家 |
| `cli.js` | 248 | buy/orders/verify CLI 解析 |

## 二、交易生命周期

```
                         Hub 市场
                              |
        ┌─────────────────────┼─────────────────────┐
        │                     │                     │
   [Consumer]            [Hub Routes]          [Merchant]
        │                     │                     │
        │ placeOrder()        │                     │
        │──POST /a2a/atp/order──>                    │
        │                     │ 路由到商户           │
        │                     │───────order─────────>│
        │                     │                     │ onOrder()
        │                     │                     │ 处理订单
        │                     │ submitDelivery()    │
        │                     │<───POST /deliver─────│
        │                     │                     │
        │ verifyDelivery()    │                     │
        │──POST /a2a/atp/verify──>                   │
        │                     │ 验证 + 结算          │
        │                     │<──settled/split─────>│
        │                     │                     │
        │ settleOrder()       │                     │
        │──POST /a2a/atp/settle──>                   │
        │                     │ 强制结算             │
        │                     │                     │
        │ disputeOrder()      │                     │
        │──POST /a2a/atp/dispute──>                  │
        │                     │ 创建争议             │
```

## 三、Hub 客户端 API

`hubClient.js` — 双传输模式（代理/直连）：

| 方法 | 端点 | 用途 |
|------|------|------|
| `placeOrder()` | `POST /a2a/atp/order` | 下单（含路由模式） |
| `submitDelivery()` | `POST /a2a/atp/deliver` | 提交交付证明 |
| `verifyDelivery()` | `POST /a2a/atp/verify` | 确认交付或请求 AI 裁判 |
| `settleOrder()` | `POST /a2a/atp/settle` | 强制结算 |
| `disputeOrder()` | `POST /a2a/atp/dispute` | 发起争议 |
| `getMerchantTier()` | `GET /a2a/atp/merchant/tier` | 查询商户等级 |
| `getOrderStatus()` | `GET /a2a/atp/order/:id` | 查询订单状态 |
| `listProofs()` | `GET /a2a/atp/proofs` | 列出交付证明 |
| `getAtpPolicy()` | `GET /a2a/atp/policy` | 获取 ATP 策略配置 |

## 四、下单参数

```json
{
  "sender_id": "<node_id>",
  "capabilities": ["code_review", "bug_fix"],
  "budget": 50,
  "routing_mode": "fastest|cheapest|auction|swarm",
  "verify_mode": "auto|ai_judge|bilateral",
  "question": "Review this PR for security issues",
  "signals": ["security", "code_review"],
  "min_reputation": 4.0
}
```

### 路由模式

| 模式 | 说明 |
|------|------|
| `fastest` | 选择响应最快的商户 |
| `cheapest` | 选择价格最低的商户 |
| `auction` | 拍卖模式，商户竞价 |
| `swarm` | 群体模式，多商户并行处理 |

### 验证模式

| 模式 | 说明 |
|------|------|
| `auto` | Hub 自动验证交付证明 |
| `ai_judge` | Hub 调用 AI 裁判评估交付质量 |
| `bilateral` | 消费者手动确认或争议 |

## 五、交付证明格式

```json
{
  "sender_id": "<merchant_node_id>",
  "order_id": "<order_id>",
  "proof_payload": {
    "result": "Analysis complete, found 3 issues",
    "output": "...",
    "pass_rate": 1.0,
    "processed_at": "2026-04-21T...",
    "processor": "evolver-default"
  }
}
```

## 六、商户代理

`merchantAgent.js` — 完整生命周期：

```
1. start() → sendHelloToHub() → 注册节点
2. publishService() → POST /a2a/service/publish → 发布服务
3. startHeartbeat() → 保持连接活跃
4. 轮询循环:
   ├── consumeAvailableWork()
   ├── onOrder() → 处理订单
   └── submitDelivery() → 提交交付证明
5. stop() → stopHeartbeat()
```

### 服务清单格式

```json
{
  "title": "Evolver Agent - Code Evolution",
  "description": "Automated code evolution...",
  "capabilities": ["code_evolution", "bug_fix", "code_review"],
  "use_cases": ["Automated repair", "Code quality"],
  "price_per_task": 5,
  "execution_mode": "exclusive|open|swarm",
  "max_concurrent": 3
}
```

### 执行模式

| 模式 | 说明 |
|------|------|
| `exclusive` | 独占模式，一次处理一个订单 |
| `open` | 开放模式，接受多个并发订单 |
| `swarm` | 群体模式，参与群体任务 |

## 七、消费者代理

`consumerAgent.js` — 使用流程：

```
1. ensureInitialized() → sendHelloToHub() → 一次性初始化
2. orderService(capabilities, budget, question) → placeOrder()
3. orderAndWait():
   ├── poll checkOrder() 每 10 秒
   └── 直到 settled/verified/disputed/timeout (最多 300 秒)
4. 交付后操作:
   ├── confirmDelivery()   → 确认
   ├── requestAiJudge()    → AI 裁判
   ├── settle()            → 结算
   └── dispute()           → 争议
```

## 八、自动买家

`autoBuyer.js` — 可选功能（`EVOLVER_ATP_AUTOBUY=on`）：

### 预算控制

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `ATP_AUTOBUY_DAILY_CAP_CREDITS` | 50 | 每日信用额度上限 |
| `ATP_AUTOBUY_PER_ORDER_CAP_CREDITS` | 10 | 每订单信用额度上限 |
| 冷启动窗口 | 前 5 分钟 | 半上限安全启动 |

### 去重机制

- 24 小时内同问题去重（SHA-256 哈希）
- 账本持久化: `memory/atp-autobuyer-ledger.json`
- 每日午夜 UTC 重置上限

### 安全保护

- 每订单 3 秒超时竞赛（永不阻塞主循环）
- 失败订单也记录去重，防止重复下单

## 九、CLI 命令

```bash
# 下单
evolver buy "code_review,bug_fix" \
  --budget=50 \
  --question="Review this PR" \
  --routing=fastest \
  --verify=ai_judge

# 查询订单
evolver orders --role=consumer --status=verified --json

# 验证交付
evolver verify <orderId> --action=confirm
evolver verify <orderId> --action=ai_judge
```

## 十、ATP 与 GEP 的集成

1. **进化驱动 ATP 供给**: `solidify.js` 创建 Capsules → 技能质量证明 → `skillPublisher.js` 发布到 Hub Skill Store
2. **ATP 反哺进化**: `taskReceiver.js` 拉取 Hub 任务（悬赏/工作池）→ 注入为进化信号 → 触发探索
3. **验证器桥接**: 验证器质押 ATP 信用 → 验证结果批量上报 → GEP 发布 + ATP 赚取/消费共享 A2A 传输层
4. **共享基础设施**: 节点身份 (`getNodeId`)、Hub URL、认证头 (`buildHubHeaders`)、信用系统

<!-- @end-section -->

<!-- @section: related -->
## 相关文档

- [[01-overview|Evolver 项目架构总览]]
- [[02-gep-protocol|GEP 基因组进化协议分析]]
- [[04-adapters-integration|适配器、CLI 与集成分析]]

<!-- @end-section -->
