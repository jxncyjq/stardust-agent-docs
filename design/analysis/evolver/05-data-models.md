---
id: "analysis-evolver-datamodels-005"
title: "数据模型与资产系统分析"
aliases: ["evolver data models", "asset system", "JSONL storage"]
type: "analysis"
category: "design/analysis/evolver"
tags: ["evolver", "data-model", "JSONL", "assets", "configuration"]
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
# 数据模型与资产系统分析

## 系统概述

Evolver 的数据存储完全基于**文件系统**（JSON + JSONL），不使用任何数据库。核心资产存储在 `assets/gep/` 下，采用内容寻址（SHA-256）确保完整性。与 new-api 和 Hermes Agent 不同，Evolver 没有任何 SQL 依赖。

## 一、资产存储架构

### 存储布局

```
assets/gep/                      (EVOLVER_REPO_ROOT + assets/gep/)
├── genes.json                   ← Gene 定义 (JSON 对象)
├── genes.jsonl                  ← 额外 Gene (仅追加)
├── capsules.json                ← 成功 Capsule (JSON 对象)
├── capsules.jsonl               ← 额外 Capsule (仅追加)
├── events.jsonl                 ← EvolutionEvent (仅追加, 不可变)
├── candidates.jsonl             ← 候选选择记录
├── external_candidates.jsonl    ← 外部候选跟踪
├── failed_capsules.json         ← 失败 Capsule (上限 200)
└── .integrity                   ← 完整性校验和
```

### 文件锁机制

`assetStore.js` — 使用建议性 O_EXCL 锁文件 (`*.lock`):
- 读-改-写原子性
- 僵尸锁检测（死 PID）→ 回收
- 锁超时: 5 秒，重试间隔 20ms
- 防止多进程竞态条件

### 内容寻址

所有资产携带 `asset_id: "sha256:..."`：
- `contentHash.js` (混淆) 计算
- SHA-256 over JSON 规范形式
- 排除 `asset_id` 字段自身
- 模式版本: `1.6.0`

## 二、数据目录全景

```
~/.evolver/                       (数据根目录)
├── settings.json                 ← 代理运行时设置
│
├── memory/                       (进化记忆)
│   ├── graph.jsonl               ← 记忆图
│   ├── narrative.jsonl           ← 叙述性记忆
│   ├── atp-autobuyer-ledger.json ← ATP 自动买家账本
│   └── ...                       ← 其他记忆文件
│
├── skills/                       (已蒸馏技能)
│   └── <skill-name>/
│       └── SKILL.md
│
├── validator_stake_state.json    ← 验证器质押状态
│
└── <repo>/                       (进化仓库)
    └── assets/gep/               ← GEP 资产 (见上)
```

## 三、核心数据模型

### Gene（基因）

**schema_version**: `1.6.0`

| 字段 | 类型 | 说明 |
|------|------|------|
| `type` | `"Gene"` | 资产类型 |
| `id` | string | 格式 `gene_{prefix}_{slug}` |
| `asset_id` | `"sha256:..."` | 内容哈希 |
| `category` | `repair\|optimize\|innovate` | 三分类 |
| `signals_match` | string[] | 触发信号列表 |
| `preconditions` | string[] | 前置条件 |
| `strategy` | string[] | 自然语言步骤 |
| `avoid` | string[] | 禁止事项 |
| `constraints` | object | `max_files`, `forbidden_paths` |
| `validation` | string[] | Shell 验证命令 |
| `summary` | string | 摘要描述 |
| `_source` | object | 来源追踪 (skill2gep 等) |
| `anti_patterns` | string[] | 反模式 |

默认基因（3 个）:
1. `gene_gep_repair_from_errors` — 从错误中修复
2. `gene_gep_optimize_prompt_and_assets` — 优化 Prompt 和资产
3. `gene_tool_integrity` — 工具完整性检查

### Capsule（胶囊）

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | `"cap_..."` | 胶囊 ID |
| `gene` | string | 来源 Gene ID |
| `trigger` | string[] | 触发条件 |
| `summary` | string | 摘要 |
| `confidence` | number | 置信度 0-1 |
| `blast_radius` | `{files, lines}` | 爆炸半径 |
| `outcome` | `{status, score}` | 结果状态和评分 |
| `success_streak` | int | 连续成功次数 |
| `execution_trace` | array | 执行轨迹 |
| `env_fingerprint` | object | 环境指纹 |
| `a2a` | object | 外部来源标记 |

**发布阈值**: `confidence >= MIN_PUBLISH_SCORE` (0.78)

### EvolutionEvent（进化事件）

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | `"evt_..."` | 事件 ID |
| `parent` | string? | 父事件 ID (进化链) |
| `intent` | string | 意图类别 |
| `signals` | string[] | 触发信号 |
| `genes_used` | string[] | 使用的基因 |
| `mutation_id` | string | 变异 ID |
| `blast_radius` | `{files, lines}` | 爆炸半径 |
| `outcome` | `{status, score}` | 结果 |
| `capsule_id` | string? | 关联胶囊 |
| `validation_report_id` | string? | 验证报告 ID |
| `asset_id` | `"sha256:..."` | 内容哈希 |

### ValidationReport（验证报告）

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | `"vr_..."` | 报告 ID |
| `gene_id` | string | 关联基因 |
| `commands` | array | 验证命令列表 |
| `overall_ok` | bool | 全部通过? |
| `duration_ms` | int | 耗时 |
| `asset_id` | `"sha256:..."` | 内容哈希 |

## 四、配置系统

### 配置源优先级

```
环境变量 (最高优先级)
    ↓ 覆盖
.env 文件
    ↓ 覆盖
config.js 默认值 (最低优先级)
```

### 关键配置项 (~40 个)

**网络 & A2A**:
| 变量 | 默认值 | 说明 |
|------|--------|------|
| `A2A_HUB_URL` | `https://evomap.ai` | Hub API URL (4 级回退) |
| `A2A_NODE_ID` | (必需) | 节点身份 |
| `HEARTBEAT_INTERVAL_MS` | 360000 (6 分钟) | 心跳间隔 |
| `HTTP_TRANSPORT_TIMEOUT_MS` | 15000 | HTTP 超时 |

**固化 & 验证**:
| 变量 | 默认值 | 说明 |
|------|--------|------|
| `VALIDATION_TIMEOUT_MS` | 180000 (3 分钟) | 验证超时 |
| `CANARY_TIMEOUT_MS` | 30000 | 金丝雀超时 |
| `MIN_PUBLISH_SCORE` | 0.78 | 最低发布评分 |

**进化循环**:
| 变量 | 默认值 | 说明 |
|------|--------|------|
| `EVOLVE_STRATEGY` | `balanced` | 策略预设 |
| `REPAIR_LOOP_THRESHOLD` | 3 | 修复循环阈值 |
| `PROMPT_MAX_CHARS` | 24000 | Prompt 最大长度 |
| `EVOLVER_MIN_SLEEP_MS` | 2000 | 最小睡眠 |
| `EVOLVER_MAX_SLEEP_MS` | 300000 (5 分钟) | 最大睡眠 |
| `EVOLVER_MAX_CYCLES_PER_PROCESS` | 100 | 重生周期 |
| `EVOLVER_MAX_RSS_MB` | 500 | 内存上限 |

**Self-PR**:
| 变量 | 默认值 | 说明 |
|------|--------|------|
| `SELF_PR_MIN_SCORE` | 0.85 | 最低评分 |
| `SELF_PR_MIN_STREAK` | 3 | 最低连续成功 |
| `SELF_PR_COOLDOWN_MS` | 86400000 (24 小时) | 冷却时间 |

**安全**:
| 变量 | 默认值 | 说明 |
|------|--------|------|
| `EVOLVER_ROLLBACK_MODE` | `hard` | 回滚模式 |
| `EVOLVE_ALLOW_SELF_MODIFY` | false | 允许自修改 |
| `LEAK_CHECK_MODE` | `strict` | 泄漏检查模式 |

## 五、安全机制

### 泄漏扫描 (sanitize.js)

两种互补检测:
1. **模式扫描**: 27+ 正则模式覆盖 token、密钥、密码、私钥、数据库 URL
2. **环境值反向检测**: 检查 `process.env` 实际值是否原样出现在内容中

### 文件安全

- **源文件保护** (`shield.js`): 禁止覆盖 evolver 核心代码
- **文件安全** (`fileSafety`): 禁止写入 `.git`、`node_modules` 等关键路径
- **金丝雀** (`canary.js`): 固化前验证 `index.js` 可正常加载

### 加密

`crypto.js` (混淆): AES-256-GCM 加密/解密，用于隐私计算和 Hub 通信

### 隐私计算

`privacyClient.js`:
- 加密 Blob 上传到 Hub
- 密封工具执行（不解密直接在 Hub 执行）
- 结果加密返回

## 六、与其他项目的存储对比

| 方面 | new-api | Hermes Agent | Evolver |
|------|---------|-------------|---------|
| 存储引擎 | SQLite/MySQL/PG | SQLite + WAL | JSONL 文件 |
| 全文搜索 | 无内置 | FTS5 (双分词器) | 无 |
| 内容寻址 | 无 | 无 | SHA-256 |
| 资产版本 | 无 | 无 | schema_version |
| 审计追踪 | Log 表 | sessions 表 | events.jsonl |
| 文件锁 | 数据库事务 | WAL + 重试 | O_EXCL 锁文件 |
| 并发策略 | 连接池 | 15 次重试 | 5 秒锁超时 |
| 缓存 | Redis + 内存 | 无 | 内存索引 |

<!-- @end-section -->

<!-- @section: related -->
## 相关文档

- [[01-overview|Evolver 项目架构总览]]
- [[02-gep-protocol|GEP 基因组进化协议分析]]
- [[06-evolver-insights|Evolver 洞察与 Legion 参考]]

<!-- @end-section -->
