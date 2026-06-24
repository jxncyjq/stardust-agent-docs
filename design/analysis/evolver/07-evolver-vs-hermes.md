---
id: "analysis-evolver-vs-hermes-007"
title: "Evolver vs Hermes 深度对比 — Evolver 的差异化优势"
aliases: ["evolver vs hermes", "Evolver对比Hermes", "Evolver优势分析"]
type: "analysis"
category: "design/analysis/evolver"
tags: ["evolver", "hermes-agent", "comparison", "differentiation", "advantage"]
version: "1.0.0"
created: "2026-05-04"
updated: "2026-05-04"
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
  - id: "analysis-evolver-atp-003"
    relation: "related_to"
    path: "./03-atp-protocol.md"
  - id: "analysis-evolver-adapters-004"
    relation: "related_to"
    path: "./04-adapters-integration.md"
  - id: "analysis-evolver-datamodels-005"
    relation: "related_to"
    path: "./05-data-models.md"
  - id: "analysis-evolver-insights-006"
    relation: "related_to"
    path: "./06-evolver-insights.md"
  - id: "analysis-hermes-insights-006"
    relation: "compared_with"
    path: "../hermes/06-hermes-insights.md"
---

<!-- @section: overview -->
# Evolver vs Hermes 深度对比 — Evolver 的差异化优势

## 文档目的

本文档从 **Evolver** 的视角出发，深度对比 Evolver 与 Hermes Agent 的设计哲学、架构决策和能力差异，明确 Evolver 相对于 Hermes 的独特优势。Evolver 是目前分析的 4 个参考项目中**最独特的**——它不关注 Agent 如何工作，而是关注 Agent 如何**持续变好**。这一维度是 Hermes 完全空白的领域。

## 一、根本定位差异

| 维度 | Evolver | Hermes Agent |
|------|---------|-------------|
| **本质** | Agent 自我进化引擎 | Agent 操作系统 |
| **核心问题** | "Agent 如何从经验中学习？" | "Agent 如何完成各种任务？" |
| **用户** | Agent 自身 | 终端用户、开发者 |
| **产出** | 进化资产 (Gene/Capsule/Event) | 任务执行结果 |
| **运行模式** | 后台进化循环 (`--loop`) | 交互式 REPL / 网关守护进程 |
| **设计范式** | 协议驱动 (GEP + ATP) | 平台驱动 (工具 + 技能 + 网关) |
| **创新层级** | 元认知层 (让 Agent 反思和改进) | 执行层 (让 Agent 做更多事) |

**关键洞察**: Hermes 让 Agent **能做更多事**，Evolver 让 Agent **能把事做得更好**。两者的关系不是竞争，而是**能力维度上的互补**。

<!-- @end-section -->

<!-- @section: advantage-evolution -->
## 二、Evolver 的核心优势

### 2.1 GEP 进化协议 — 形式化的自我改进

这是 Evolver 最核心的差异化能力，也是 Hermes **完全不具备**的能力：

```
Evolver GEP 进化循环:
  日志/输出
    │
    ▼
  信号提取 (3 层)
    │
    ▼
  策略匹配 (Gene 选择)
    │
    ├── repair:   修复已知错误
    ├── optimize: 优化 Prompt 和资产
    └── innovate: 探索新的解决方案
    │
    ▼
  变异执行 (LLM 按 Gene.strategy[] 行动)
    │
    ▼
  固化 (验证 → 金丝雀 → 审计 → 回滚)
    │
    ▼
  Capsule 创建 (执行证明) + EvolutionEvent 追加
    │
    ▼
  发布到 Hub (可选, 自 PR)
```

```
Hermes 的"改进"方式:
  → 用户手动输入 /memory save
  → 用户手动编辑 MEMORY.md
  → 用户手动安装新技能
  → 无自动进化、无信号检测、无策略匹配
```

**Evolver 的优势**: Hermes 的 Agent 每次都是从"零知识"开始（除了 MEMORY.md 和 USER.md），而 Evolver 的 Agent 会从**每一次失败中学习**，并将经验固化为可复用的进化资产。这是一个"手动档"和"自动档"的区别。

### 2.2 3 层信号提取引擎 — Agent 的感官系统

Evolver 拥有一个完整的信号感知系统，这在 Hermes 中完全不存在：

```
Evolver 3 层信号引擎:

第 1 层: 正则匹配 (确定性, 0ms, 零成本)
  → 已知错误模式 (stack traces, syntax errors, timeout patterns)
  → 显式信号前缀 (EVOLVER_SIGNAL: ...)
  → 支持多语言 (EN/ZH-CN/ZH-TW/JA)

第 2 层: 关键词打分 (统计, 0ms, 零成本)
  → 模糊/分散的隐含信号
  → 加权计分 → 超过阈值触发进化
  → 上下文感知 (文件路径、命令类型)

第 3 层: LLM 语义分析 (限速, 每 N 轮, 有成本)
  → 深度语义理解
  → 新信号类型发现
  → 复杂因果关系提取

Hermes:
  无信号系统
  → 仅 /memory save 手动保存
  → Agent 无法感知自己的失败模式
```

**Evolver 的优势**: 3 层信号引擎的分级设计极其务实。第 1、2 层零成本、零延迟处理大多数情况，第 3 层仅在必要时动用 LLM。这是"感知-决策-行动"循环中的**感知层**，是任何自我改进系统的前提。

### 2.3 信号后处理管线 — 防止进化陷阱

Evolver 的信号处理不仅仅是检测，还有完整的后处理逻辑，防止 Agent 陷入**自我改进的死循环**：

```
Evolver 信号后处理:

  去重与抑制:
    8 轮内出现 >= 3 次的信号 → 自动抑制
    → 防止同一错误反复触发进化

  优先级排序:
    可操作信号 > 表面信号
    → 优先处理有明确修复路径的问题

  修复循环检测:
    同一策略连续失败 N 次 → 强制创新
    → 避免"用同样的方法反复尝试"

  平台期检测:
    长时间无进展 → 建议转向
    → 避免陷入局部最优

Hermes:
  无信号后处理
  → 工具看门狗仅检测工具级别的重复失败
  → 不支持跨会话的进化模式分析
```

**Evolver 的优势**: 这些机制回答了"自我改进系统如何避免自我毁灭"这个关键问题。Hermes 的工具看门狗（`tool_guardrails.py`）只在单次会话内工作，不追踪跨会话的改进模式。

### 2.4 Gene/Capsule/Event 三元资产模型 — 进化资产管理

Evolver 将进化经验**资产化**，而 Hermes 的经验仅存在于会话日志中：

```
Evolver 三元资产模型:

  Gene (基因)
    "遇到 X 类型的问题，按 Y 步骤解决"
    → 可复用的策略模板
    → 自然语言步骤 (非代码补丁)
    → 携带约束 (max_files, forbidden_paths)
    → 携带验证 (Shell 验证命令)

  Capsule (胶囊)
    "策略 Z 在环境 W 下成功 N 次"
    → 带执行证明的成功记录
    → 完整的 execution_trace (命令、退出码、输出)
    → 置信度评分 (0-1)
    → 连续成功次数 (streak)

  EvolutionEvent (进化事件)
    "谁在何时做了什么，结果如何"
    → 不可变的审计追踪
    → parent 字段链接进化链
    → 仅追加、永不删除
    → SHA-256 内容寻址

Hermes:
  会话历史 (sessions + messages 表)
  → 原生数据，未结构化为可复用资产
  → MEMORY.md: 非结构化自由文本
  → 技能: 静态知识，非从经验中学习
```

**Evolver 的优势**: Gene 是"知道做什么"，Capsule 是"证明这样做有效"，EvolutionEvent 是"记录何时做的"。三者合在一起，形成了一个完整的**经验学习闭环**。Hermes 的 MEMORY.md 和会话历史缺乏这种结构化。

### 2.5 固化安全网 — Agent 变更的防护体系

Evolver 的固化（Solidify）流程是一个**多层安全网**，确保 Agent 的自我修改不会破坏系统：

```
Evolver 固化流程:

  变更完成
    │
    ├── 1. 验证命令执行
    │     Gene.validation[] → 运行预定义的验证脚本
    │     VALIDATION_TIMEOUT_MS = 180000 (3 分钟)
    │
    ├── 2. 金丝雀检查
    │     index.js 可正常加载?
    │     CANARY_TIMEOUT_MS = 30000
    │     → 失败 → 自动回滚
    │
    ├── 3. 爆炸半径评估
    │     Gene.constraints.max_files (最多修改多少文件)
    │     Gene.constraints.forbidden_paths (禁止修改的路径)
    │     → 超限 → 拒绝
    │
    ├── 4. 源文件保护
    │     shield.js → 核心文件不可被覆盖
    │     → 防止 Agent 自毁
    │
    ├── 5. Git 提交 / 回滚
    │     所有操作基于 Git
    │     EVOLVER_ROLLBACK_MODE = hard
    │     → 失败自动 git reset --hard
    │
    ├── 6. Capsule 创建
    │     执行证明 + 环境指纹
    │     → 记录可复现的成功条件
    │
    └── 7. EvolutionEvent 追加
          不可变的审计追踪
          → 完整的进化历史
```

```
Hermes 的安全机制:
  - 文件安全黑名单 (file_safety.py)
  - 工具看门狗 (tool_guardrails.py)
  - 技能安全扫描 (skills_guard.py)
  → 这些是"防御性"的（阻止危险操作）
  → 不是"验证性"的（操作后确认正确性）
```

**Evolver 的优势**: Hermes 的安全机制是"事前阻止"（不要做危险的事），Evolver 的固化是"事后验证"（做完了确认没问题，有问题自动回滚）。两者结合才是最完整的防护体系。

### 2.6 内容寻址 (SHA-256) — 资产完整性保证

Evolver 的所有进化资产都携带 SHA-256 哈希：

```
Evolver 内容寻址:
  asset_id: "sha256:abc123def456..."
  → 所有 Gene/Capsule/Event 携带
  → SHA-256 over JSON 规范形式
  → 排除 asset_id 字段自身
  → 模式版本: 1.6.0

  用途:
  - 去重: 相同内容的资产自动识别
  - 完整性: 检测篡改和损坏
  - P2P 共享: 可信的内容交换
  - 审计: 不可否认的资产引用

Hermes:
  无内容寻址
  → 会话 ID 是随机 UUID
  → 消息无哈希
  → 技能无版本哈希
```

**Evolver 的优势**: 内容寻址是去中心化系统的基石。它为未来的 P2P Agent 协作、跨节点经验共享提供了基础设施。Hermes 使用随机 ID，无法验证资产完整性。

### 2.7 ATP Agent 交易协议 — Agent 间经济层

Evolver 的 ATP 创造了一个 Agent 之间的**自动化服务市场**：

```
Evolver ATP 交易:

  Merchant (商户)           Hub 市场            Consumer (消费者)
      │                        │                      │
      │ 注册服务                │                      │
      │──publishService()──→   │                      │
      │                        │                      │
      │                        │   下单                │
      │                        │←──placeOrder()────   │
      │                        │   capabilities       │
      │                        │   budget: 50 credits  │
      │                        │   routing: fastest    │
      │                        │                      │
      │  订单通知               │                      │
      │←──onOrder()────────   │                      │
      │                        │                      │
      │  提交交付               │                      │
      │──submitDelivery()──→  │                      │
      │                        │                      │
      │                        │   验证交付             │
      │                        │──verifyDelivery()──→  │
      │                        │                      │
      │  结算                   │                      │
      │←──settled/split────   │                      │
```

```
Hermes:
  无 Agent 间交易
  → delegate_task: 单方面委派子任务
  → 无服务注册、无定价、无经济激励
  → 无信用系统
```

**Evolver 的优势**: ATP 的四种路由模式（fastest/cheapest/auction/swarm）和三种验证模式（auto/ai_judge/bilateral）构成了一个完整的 Agent 经济层。这是 Hermes 完全没有涉及的领域。

### 2.8 自动买家 — 能力缺口自动填补

```
Evolver AutoBuyer:
  检测能力缺口 → 自动搜索商户 → 自动下单 → 自动验证

  安全机制:
  - 每日信用额度上限 (ATP_AUTOBUY_DAILY_CAP_CREDITS = 50)
  - 每订单额度上限 (ATP_AUTOBUY_PER_ORDER_CAP_CREDITS = 10)
  - 冷启动窗口 (前 5 分钟半上限)
  - 24 小时同问题去重 (SHA-256)
  - 每订单 3 秒超时竞赛 (永不阻塞主循环)

Hermes:
  无能力缺口检测
  → 用户手动安装技能
  → 用户手动配置工具
  → 无自动"按需扩展"
```

**Evolver 的优势**: 自动买家让 Agent 具备了"认识到自己能力不足→主动寻求帮助"的能力。这是迈向真正自主 Agent 的关键一步。

### 2.9 代理邮箱模式 — 离线优先设计

Evolver 的本地代理是一个精心设计的离线优先通信层：

```
Evolver Proxy 架构:

  AI Agent
    │ HTTP (localhost:19820)
    ▼
  ┌─────────────────────────────┐
  │        EvoMapProxy           │
  │                              │
  │  MailboxStore (JSONL)        │
  │  → 消息持久化                │
  │  → UUID v7 时间排序          │
  │  → _op: "update" 补丁行      │
  │                              │
  │  SyncEngine                  │
  │  → 自适应轮询 (空闲/活跃)    │
  │  → 出站刷新 + 入站拉取       │
  │  → 指数退避重试              │
  │                              │
  │  LifecycleManager            │
  │  → hello + heartbeat (360s)  │
  │  → 重新认证 (401/403)        │
  └─────────────────────────────┘
         │ HTTPS
         ▼
    EvoMap Hub

  核心优势:
  - 离线可用: 所有核心功能本地运行
  - 消息持久化: 确保不可否认性
  - 认证隔离: Agent 不需要 Hub 凭证
```

```
Hermes:
  - 直接与 LLM API 通信 (无代理层)
  - 网关直接与消息平台通信
  - 无离线消息队列
  - 无自动同步/重试
```

**Evolver 的优势**: 代理邮箱模式实现了认证隔离（Agent 不需要知道 Hub 凭证）、离线缓冲（断网时消息排队）和自动同步（恢复网络后自动刷新）。这是微服务架构中常见的 Sidecar 模式在 Agent 系统中的优雅应用。

### 2.10 泄漏扫描 — 深度安全防护

Evolver 的泄漏扫描比 Hermes 的日志脱敏更深层：

```
Evolver 泄漏扫描 (sanitize.js):

  1. 模式扫描: 27+ 正则模式
     - API Key (各种前缀)
     - Token (Bearer, JWT, OAuth)
     - 密码 (各种格式)
     - 私钥 (PEM, SSH)
     - 数据库 URL (含凭证)
     - 云服务凭证 (AWS, GCP, Azure)

  2. 环境值反向检测:
     - 检查 process.env 中的实际值
     - 是否原样出现在输出内容中
     - → 捕获非标准格式的泄漏
     - → 这是 Hermes 没有的检测维度

  3. 配置:
     - LEAK_CHECK_MODE = strict

Hermes:
  日志脱敏 (redact.py):
  - 95+ API Key 前缀模式
  - 仅作用于日志输出
  - 不检查实际环境变量值
```

**Evolver 的优势**: 环境值反向检测是 Evolver 泄漏扫描的杀手级特性。它不依赖"凭证长什么样"的模式匹配，而是直接检查"已知的秘密是否出现在输出中"。这能捕获模式匹配遗漏的任何格式的凭证泄漏。

### 2.11 自适应空闲调度 — 资源高效利用

```
Evolver 自适应调度 (idleScheduler.js):

  活跃期:
    短睡眠间隔 (EVOLVER_MIN_SLEEP_MS = 2000)
    → 快速响应进化信号

  空闲期:
    长睡眠间隔 (EVOLVER_MAX_SLEEP_MS = 300000, 5 分钟)
    → 节省资源

  饱和度检测:
    连续无进化信号 → 逐渐延长睡眠
    检测到信号 → 立即缩短睡眠

  自杀检查:
    MAX_CYCLES_PER_PROCESS = 100 (防止内存泄漏)
    MAX_RSS_MB = 500 (防止内存溢出)
    → 自动重启进程

Hermes:
  - Gateway: 事件驱动 (无主动轮询)
  - CLI: 用户交互驱动 (无后台循环)
  - Cron: 定时调度 (固定间隔)
  → 无自适应调度机制
```

**Evolver 的优势**: 自适应空闲调度让 Evolver 可以在后台持续运行而不过度消耗资源。自杀检查机制（MAX_CYCLES + MAX_RSS）是一个务实的工程决策——与其修复内存泄漏，不如在达到阈值时自动重启进程。

### 2.12 策略预设 — 进化行为的可配置性

Evolver 提供 6 种策略预设，让用户可以根据项目阶段调节进化行为：

```
Evolver 策略预设 (EVOLVE_STRATEGY，由 src/gep/strategy.js::getStrategyNames() 返回):

  balanced (默认):
    → 创新 / 优化 / 修复混合，是日常运行的稳态选择

  innovate:
    → 提高 innovate 类基因权重，主动探索新策略

  harden:
    → 提高 optimize / repair 权重，强化稳定性与回归修复

  repair-only:
    → 仅选取 repair 类基因，进入"只修复不创新"模式

  early-stabilize:
    → 项目早期降低创新比例，先把基线打牢

  steady-state:
    → 稳态运行，最低强度，仅必要时演化

Hermes:
  无进化行为配置
  → yolo 模式仅影响工具执行审批
  → "迭代预算"仅影响单次会话
```

> 各预设的具体权重与触发条件实现在 `src/gep/strategy.js`（混淆），公开 API 仅暴露 `STRATEGIES` / `resolveStrategy(name)` / `getStrategyNames()`；本文档不再列具体百分比，避免与代码漂移。

**Evolver 的优势**: 策略预设让用户可以根据项目阶段灵活调整进化行为（早期 `early-stabilize`、稳态 `steady-state`、日常 `balanced`、创新冲刺 `innovate`、只修不改 `repair-only`、加固阶段 `harden`）。Hermes 没有类似的行为调节机制。

<!-- @end-section -->

<!-- @section: philosophy -->
## 三、设计哲学的深层对比

### 3.1 Evolver: "协议思维" vs Hermes: "平台思维"

```
Evolver 的设计哲学:
  "定义一套协议，让 Agent 按照协议自我改进"
  → GEP: 进化协议 (如何学习)
  → ATP: 交易协议 (如何交换)
  → 协议是抽象的、可验证的、与实现无关的

Hermes 的设计哲学:
  "提供一个完整的平台，让 Agent 在其中自由工作"
  → 工具: 原子操作
  → 技能: 领域知识
  → 网关: 多平台入口
  → 平台是具体的、功能丰富的、开箱即用的
```

### 3.2 Evolver: "深度优先" vs Hermes: "宽度优先"

```
Evolver 的深度:
  └── 进化 (单一维度, 极致深入)
        ├── 信号提取 (3 层, 20+ 种信号类型)
        ├── 策略选择 (6 种预设, 3 类基因 repair/optimize/innovate)
        ├── 变异执行 (自然语言策略步骤)
        ├── 固化验证 (7 层安全检查)
        ├── 资产化 (3 种资产类型, 内容寻址)
        └── 发布 (自 PR, Hub 共享)

Hermes 的宽度:
  运行时 + 工具 + 技能 + 网关 + Web + TUI + ACP + Cron + ...
  → 每个维度都有, 但都不如 Evolver 在进化维度上的深度
```

**Evolver 在"自我进化"这一个维度上的深度，超过了 Hermes 任何单一维度的深度。**

### 3.3 Evolver: "Agent 中心" vs Hermes: "用户中心"

```
Evolver 的 Agent 中心设计:
  - 用户安装 Evolver 后, 由 Agent 自主使用
  - CLI 命令 (solidify/review/distill) 是 Agent 调用的
  - 钩子脚本在 Agent 工作时自动触发
  - 进化循环在后台静默运行
  → 用户很少直接与 Evolver 交互

Hermes 的用户中心设计:
  - 用户通过 CLI/TUI/Web 直接与 Agent 对话
  - 用户通过 Gateway 在消息平台接收 Agent 消息
  - 用户手动安装技能、配置模型
  → Agent 是用户的工具
```

**Evolver 的"Agent 中心"设计意味着它与 Hermes 可以在不同层面上配合**：Hermes 作为用户交互的 Agent，Evolver 作为在后台默默改进 Hermes 的元 Agent。

### 3.4 Evolver: "寄生式架构" vs Hermes: "自包含架构"

```
Evolver 的寄生式:
  Evolver → 钩子注入 → Claude Code / Cursor / Codex
  → 不需要自己的 Agent 运行时
  → 可以附着在任何 Agent 上
  → 轻量、无侵入

Hermes 的自包含:
  Hermes → 自己就是 Agent
  → 完整的运行时
  → 不依赖任何外部 Agent
```

**Evolver 寄生式架构的独特优势**: 它可以增强**任何** Agent（Claude Code、Cursor、Codex、甚至 Hermes），而不需要替换它们。这意味着 Evolver 的理念可以应用到 Legion 的 Agent 运行时中，而无需重新发明 Agent。

<!-- @end-section -->

<!-- @section: gap-analysis -->
## 四、Evolver 缺少而 Hermes 拥有的能力

客观地说，Hermes 在以下方面超越了 Evolver：

| Hermes 独有能力 | 说明 | Evolver 的缺口 |
|---------------|------|---------------|
| **Agent 运行时** | AIAgent 主循环 + 传输层 | 无 Agent 运行时（依赖外部） |
| **工具系统** | 70+ 工具 + 自注册 + 看门狗 | 无工具系统 |
| **技能市场** | 200+ 技能 + 安全扫描 | 仅有技能蒸馏（单向） |
| **多平台网关** | 20+ 消息平台 | 无消息网关 |
| **Web 仪表盘** | React 19 + 12 个页面 | 无 Web UI |
| **TUI** | React/Ink 终端 UI | 无 TUI |
| **上下文压缩** | 辅助 LLM 摘要 + 裁剪 | 无上下文压缩 |
| **插件系统** | 4 种插件类型 | 无插件系统 |
| **会话持久化** | SQLite + FTS5 | JSONL 文件 |
| **RL 训练** | Atropos 框架 + 轨迹压缩 | 无训练基础设施 |
| **开源程度** | 完全开源 (200+ 文件) | 核心混淆 (25 个模块) |

这些 Hermes 的独特优势详见 [[../hermes/07-hermes-vs-evolver|Hermes vs Evolver 深度对比]]。

<!-- @end-section -->

<!-- @section: legion-implications -->
## 五、对 Legion 设计的启示

### 5.1 Evolver 不可替代的价值

Evolver 解决了一个 Hermes 完全没触及的问题：**Agent 的持续自我改进**。在 Legion 的设计中，这意味着：

```
Legion 的两个维度:

  执行维度 (参考 Hermes):
    Agent 能做什么?
    → 工具、技能、网关、多平台

  进化维度 (参考 Evolver):
    Agent 如何变好?
    → 信号感知、策略匹配、经验固化、审计追踪
```

**一个只有执行能力而没有学习能力的 Agent 系统，会反复犯同样的错误。一个只有学习能力而没有执行能力的 Agent 系统，没有东西可学。**

### 5.2 Legion 应该从 Evolver 继承什么

| Evolver 能力 | Legion 继承方式 | 优先级 |
|-------------|----------------|--------|
| 3 层信号提取 | Legion SignalDetector — Agent 感官系统 | P0 |
| Gene/Capsule/Event 三元资产 | Legion EvolutionAssets — Wiki 知识条目 | P0 |
| 固化安全网 | Legion ChangeGuard — 变更验证+回滚 | P0 |
| 仅追加审计日志 | Legion AuditLog — 不可变操作追踪 | P1 |
| 内容寻址 | Legion ContentHash — 知识完整性 | P1 |
| 泄漏扫描 | Legion LeakScanner — 安全模块 | P1 |
| ATP 交易协议 | Legion AgentMarket — Agent 间服务交换 | P2 |
| 代理邮箱模式 | Legion Sidecar — 组件间通信代理 | P2 |
| 自适应空闲调度 | Legion ResourceManager — 资源管理 | P2 |
| 策略预设 | Legion EvolutionPolicy — 进化行为配置 | P2 |

### 5.3 Evolver 的局限性 — Legion 需要改进的地方

| 方面 | Evolver 现状 | Legion 改进方向 |
|------|------------|----------------|
| 核心实现 | JavaScript (Node.js) | Go/Rust (性能 + 类型安全) |
| 代码透明度 | 核心混淆 (~3MB) | 完全开源 |
| 信号提取 | 正则 + 关键词 (脆弱) | 结构化 Schema + Embedding |
| 知识存储 | JSONL 文件 (线性搜索) | 数据库 + 向量检索 |
| 进化验证 | 启发式 (置信度评分) | 统计学显著性检验 + A/B |
| 进化范围 | 单 Agent | 多 Agent 协作进化 |
| 可视化 | CLI 文本 | Web 进化仪表盘 |
| 审计 | JSONL 文件 | 区块链式完整性链 |
| Hub 依赖 | 单点 EvoMap Hub | 去中心化 P2P |
| Git 依赖 | 强依赖 Git | 可选 Git + 数据库回滚 |

### 5.4 核心设计原则

```
理想的 Legion Agent:

  ┌─────────────────────────────────────────┐
  │            Legion Agent 运行时            │
  │                                          │
  │   ┌─────────────┐  ┌─────────────────┐  │
  │   │  Hermes 层   │  │   Evolver 层     │  │
  │   │  (工作能力)  │  │   (学习能力)     │  │
  │   │             │  │                  │  │
  │   │ • 工具执行   │  │ • 信号感知       │  │
  │   │ • 技能调用   │  │ • 策略匹配       │  │
  │   │ • 多平台交互 │  │ • 经验固化       │  │
  │   │ • 上下文管理 │  │ • 变更验证       │  │
  │   │ • 会话持久化 │  │ • 审计追踪       │  │
  │   └──────┬──────┘  └────────┬────────┘  │
  │          │                  │            │
  │          └────────┬─────────┘            │
  │                   ▼                      │
  │         Legion Wiki 知识引擎             │
  │     (Gene/Capsule 作为结构化 Wiki 条目)   │
  └─────────────────────────────────────────┘
```

<!-- @end-section -->

<!-- @section: related -->
## 相关文档

- [[01-overview|Evolver 项目架构总览]]
- [[02-gep-protocol|GEP 基因组进化协议分析]]
- [[06-evolver-insights|Evolver 洞察与 Legion 参考]]
- [[../hermes/07-hermes-vs-evolver|Hermes vs Evolver 深度对比]]
- [[../hermes/index|Hermes Agent 分析索引]]

<!-- @end-section -->
