---
id: "analysis-evolver-overview-001"
title: "Evolver 项目架构总览"
aliases: ["evolver overview", "Evolver架构概览", "GEP引擎"]
type: "analysis"
category: "design/analysis/evolver"
tags: ["evolver", "gep", "evolution", "agent", "javascript", "self-improvement"]
version: "1.0.0"
created: "2026-05-03"
updated: "2026-05-03"
author: "jxncyjq"
status: "review"
parent: null
children:
  - "analysis-evolver-gep-002"
  - "analysis-evolver-atp-003"
  - "analysis-evolver-adapters-004"
  - "analysis-evolver-datamodels-005"
  - "analysis-evolver-insights-006"
  - "analysis-evolver-vs-hermes-007"
related_docs:
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
  - id: "analysis-evolver-vs-hermes-007"
    relation: "related_to"
    path: "./07-evolver-vs-hermes.md"
---

<!-- @section: overview -->
# Evolver 项目架构总览

## 项目概述

Evolver 是由 **EvoMap** 开发的基于 **GEP (Genome Evolution Protocol)** 的 AI Agent 自我进化引擎。它不是代码补丁工具，而是一个**协议约束的 Prompt 生成器**——将临时性的 Prompt 调整转化为可审计、可复用的"进化资产"。核心思想是：从运行时日志中提取信号，选择匹配的进化策略（Gene），生成结构化的 GEP Prompt 指导 Agent 自我改进，最后固化变更并记录审计轨迹。

- 仓库地址：`EvoMap/evolver`
- npm 包：`@evomap/evolver` v1.69.20
- 主要语言：JavaScript (Node.js >= 18)
- 运行时依赖：仅 `dotenv`（零外部依赖）
- 构建依赖：`javascript-obfuscator`（核心引擎混淆）
- 许可证：GPL-3.0-or-later（正在向 source-available 过渡）

## 系统定位

```
AI Agent (Claude Code / Cursor / Codex)
    │  CLI 命令 / 钩子脚本 / HTTP API
    ▼
┌──────────────────────────────────────────────────────────┐
│                     Evolver 进化引擎                       │
│                                                          │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐              │
│  │ 守护进程  │  │  代理    │  │ 适配器   │  入口层       │
│  │ (--loop) │  │(19820)  │  │(3 平台) │              │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘              │
│       └──────────────┼─────────────┘                    │
│                      ▼                                   │
│  ┌──────────────────────────────────────────┐           │
│  │         GEP 进化核心                       │           │
│  │                                          │           │
│  │  Signal → Select → Mutate → Execute      │           │
│  │     ↓        ↓        ↓         ↓        │           │
│  │  3层信号  策略选择  策略约束  LLM Prompt  │           │
│  └──────────────────────────────────────────┘           │
│                      │                                   │
│       ┌──────────────┼──────────────┐                   │
│       ▼              ▼              ▼                   │
│  ┌─────────┐  ┌──────────┐  ┌──────────┐              │
│  │  ATP    │  │  Proxy   │  │   Ops    │  服务层       │
│  │(Agent交易)│  │(邮箱代理)│  │(运维管理)│              │
│  └─────────┘  └──────────┘  └──────────┘              │
│                      │                                   │
│                      ▼                                   │
│  ┌──────────────────────────────────────────┐           │
│  │  Asset Store (JSONL + SHA-256 哈希)      │  资产层    │
│  │  Genes / Capsules / EvolutionEvents      │           │
│  └──────────────────────────────────────────┘           │
└──────────────────────────────────────────────────────────┘
         │
         ▼
    EvoMap Hub (https://evomap.ai)
    ├── 技能市场
    ├── 验证器网络
    ├── ATP 交易市场
    └── 排行榜
```

## 技术栈

| 层级 | 技术 | 说明 |
|------|------|------|
| 运行时 | Node.js >= 18 | JavaScript 引擎 |
| 运行时依赖 | dotenv | 唯一运行时依赖 |
| CLI | 自研 (argparse 风格) | 8 个主命令处理器（run / solidify / distill / review / fetch / asset-log / setup-hooks / atp 子命令组），共 10 个可调用命令名 |
| 状态存储 | JSONL + JSON 文件 | 追加式日志存储 |
| 网络协议 | HTTP/JSON (A2A Protocol) | Hub 通信 |
| 本地代理 | HTTP Server (自研) | 端口 19820 |
| 平台集成 | 钩子脚本 (Node.js) | Claude Code / Cursor / Codex |
| 安全 | AES-256-GCM, SHA-256 | 资产哈希、隐私计算 |
| 混淆 | javascript-obfuscator | 核心引擎保护 |

## 分层架构

```
入口层 (Entry Points)
  ├── index.js           — CLI 守护进程 (10 个命令, --loop 模式)
  ├── src/proxy/         — 本地 HTTP 邮箱代理 (端口 19820，可被 `EVOMAP_PROXY_PORT` 覆盖)
  └── src/adapters/      — 平台钩子适配器 (3 平台 + 1 基础设施模块 hookAdapter.js)
       │
       ▼
核心引擎层 (Core Engine) [混淆]
  ├── src/evolve.js      — 主进化循环 (765KB 混淆)
  ├── src/gep/signals.js  — 3 层信号提取引擎 (可读)
  ├── src/gep/selector.js — 基因选择器 (混淆)
  ├── src/gep/prompt.js   — GEP Prompt 组装 (混淆)
  ├── src/gep/solidify.js — 固化引擎 (419KB 混淆)
  └── src/gep/strategy.js — 策略预设 (混淆)
       │
       ▼
协议层 (Protocols)
  ├── src/gep/            — Genome Evolution Protocol (53 顶层 + 4 个 validator/ = 57 文件)
  └── src/atp/            — Agent Transaction Protocol (8 文件, 全可读)
       │
       ▼
资产层 (Assets)
  ├── src/gep/assetStore.js — JSONL 文件存储 + 文件锁
  └── assets/gep/           — genes.json, capsules.json, events.jsonl
```

## 核心概念

### GEP 三大资产

| 资产类型 | 文件 | 用途 |
|----------|------|------|
| **Gene** (基因) | `genes.json` | 可复用的进化策略模板 |
| **Capsule** (胶囊) | `capsules.json` | 已证实的成功执行记录 |
| **EvolutionEvent** (进化事件) | `events.jsonl` | 不可变的审计追踪 |

### 进化生命周期

```
提取信号 → 选择基因 → 变异规划 → 策略检查 → 执行 → 固化 → 发布
(3层引擎)  (策略匹配)  (约束约束)  (安全检查) (LLM) (审计+回滚) (Hub/PR)
```

### 策略预设

| 策略 | 用途 |
|------|------|
| `balanced` | 默认均衡（创新/优化/修复混合） |
| `innovate` | 积极创新（提高 innovate 类基因权重） |
| `harden` | 稳定加固（提高优化与修复权重） |
| `repair-only` | 纯修复模式（只选 repair 类基因） |
| `early-stabilize` | 早期稳定（项目初期降低创新比例，先把基线打牢） |
| `steady-state` | 稳态运行（最低强度，仅必要时演化） |

> 6 项预设由 `src/gep/strategy.js::getStrategyNames()` 返回。具体的创新/优化/修复权重在混淆的 `strategy.js` 内部，文档不再列具体百分比，避免与代码漂移。

## 核心数据流

```
Agent 会话日志
  │
  ▼
3 层信号提取 (signals.js)
  ├── 正则匹配 → 确定性模式
  ├── 关键词打分 → 统计模式
  └── LLM 语义 → 深度分析 (每5轮限速)
  │
  ▼
去重与优先级排序
  ├── 移除重复信号 (8轮内出现3次+)
  ├── 修复循环检测
  └── 平台期检测
  │
  ▼
基因选择 (selector.js)
  ├── 信号匹配 → Gene.signals_match
  ├── 策略权重 → 创新/优化/修复比例
  └── 人格状态 → RIGOR/CREATIVITY/RISK_TOLERANCE
  │
  ▼
GEP Prompt 生成 (prompt.js)
  ├── 嵌入选中的基因 + 胶囊
  ├── 注入策略约束 + 变异参数
  └── 注入人格状态 + 记忆上下文
  │
  ▼
Agent 执行 (外部 LLM)
  │
  ▼
固化 (solidify.js)
  ├── 运行验证命令 (node/npm/npx)
  ├── 金丝雀检查 (index.js 可加载?)
  ├── 爆炸半径评估
  ├── Git 提交 / 回滚
  └── 追加 EvolutionEvent + Capsule
  │
  ▼
发布 (可选)
  ├── Hub 验证 → /a2a/verify
  ├── 自我 PR → GitHub
  └── 技能发布 → Hub 技能商店
```

## 关键设计原则

1. **协议约束进化**: 每个变更必须遵循 信号→基因→Prompt→固化→事件 的完整链条
2. **本地优先 + 可选 Hub**: 所有核心功能离线可用，Hub 连接启用网络功能
3. **代理邮箱模式**: Agent 永不直接调用 Hub API，通过本地 HTTP 代理中转
4. **仅追加审计**: EvolutionEvents 是 JSONL 仅追加日志，不可变
5. **内容寻址**: 所有资产携带 `sha256:` asset_id，确保完整性
6. **自保护**: 保护关键源文件不被 Agent 覆盖；金丝雀检查防止提交损坏代码
7. **自适应调度**: OMLS 风格的空闲调度，根据用户活跃度调整进化强度

## 混淆策略

Evolver 采用**分层混淆**策略：
- **核心引擎**（27 个文件，约 2.9MB / 2,925KB）：`javascript-obfuscator` 压缩为单行、`_0xXXXX` 风格变量名 + 字符串数组重排的混淆代码
- **基础设施代码**（适配器、代理、ATP、ops、配置、`gep/validator/`）：完全可读
- GEP 顶层 53 个 .js 中：26 个可读（约 236KB）+ 27 个混淆（约 2.9MB）
- 这与项目向"source-available"许可模式的过渡一致

## 文档索引

- [[02-gep-protocol|GEP 基因组进化协议分析]]
- [[03-atp-protocol|ATP Agent 交易协议分析]]
- [[04-adapters-integration|适配器、CLI 与集成分析]]
- [[05-data-models|数据模型与资产系统分析]]
- [[06-evolver-insights|Evolver 洞察与 Legion 参考]]

<!-- @end-section -->

<!-- @section: related -->
## 相关文档

- [[02-gep-protocol|GEP 基因组进化协议分析]]
- [[03-atp-protocol|ATP Agent 交易协议分析]]
- [[04-adapters-integration|适配器、CLI 与集成分析]]
- [[05-data-models|数据模型与资产系统分析]]
- [[06-evolver-insights|Evolver 洞察与 Legion 参考]]

<!-- @end-section -->
