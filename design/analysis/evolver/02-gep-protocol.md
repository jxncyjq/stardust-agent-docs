---
id: "analysis-evolver-gep-002"
title: "GEP 基因组进化协议分析"
aliases: ["GEP protocol", "Genome Evolution", "基因进化协议"]
type: "analysis"
category: "design/analysis/evolver"
tags: ["evolver", "gep", "genome", "evolution", "signals", "assets"]
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
  - id: "analysis-evolver-datamodels-005"
    relation: "related_to"
    path: "./05-data-models.md"
---

<!-- @section: overview -->
# GEP 基因组进化协议分析

## 系统概述

GEP (Genome Evolution Protocol) 是 Evolver 的核心协议，定义了 AI Agent 如何**从运行时信号中学习、选择进化策略、执行变更、并记录审计轨迹**的完整生命周期。GEP 共 57 个 .js 文件（`src/gep/` 顶层 53 个 + `src/gep/validator/` 4 个）。其中顶层 53 个里：**26 个可读**（约 236KB）+ **27 个被 `javascript-obfuscator` 混淆**（约 2.9MB）；`validator/` 子目录全部可读。

## 一、GEP 三大资产模型

### Gene（基因）— 可复用的进化策略模板

```json
{
  "type": "Gene",
  "id": "gene_gep_repair_from_errors",
  "schema_version": "1.6.0",
  "asset_id": "sha256:...",
  "category": "repair|optimize|innovate",
  "signals_match": ["error", "exception", "failed"],
  "preconditions": ["signals contains error-related indicators"],
  "strategy": [
    "1. Extract structured signals from logs...",
    "2. Select an existing Gene by signals match...",
    "3. Apply repair with minimal blast radius..."
  ],
  "avoid": ["DO NOT edit critical config files..."],
  "constraints": {
    "max_files": 20,
    "forbidden_paths": [".git", "node_modules"]
  },
  "validation": [
    "node scripts/validate-modules.js ./src/evolve",
    "node scripts/validate-suite.js"
  ],
  "summary": "Reusable repair strategy from error signals",
  "anti_patterns": ["tool_bypass"]
}
```

**关键字段**:
- `id`: 格式 `gene_{prefix}_{slug}`，内容寻址
- `category`: 三分类 — `repair`（修复）、`optimize`（优化）、`innovate`（创新）
- `signals_match`: 触发此基因的信号模式列表
- `strategy`: 自然语言步骤（指导 LLM 执行）
- `avoid`: 反模式 / "禁止" 规则
- `validation`: Shell 命令（仅限 `node`、`npm`、`npx`）
- `constraints`: `max_files` 爆炸半径限制、`forbidden_paths` 黑名单

### Capsule（胶囊）— 已证实的成功执行记录

```json
{
  "type": "Capsule",
  "id": "cap_s2g_env_vars_abc12345",
  "gene": "gene_s2g_env_vars",
  "trigger": ["env_files", "vercel_env"],
  "summary": "Applied gene_s2g_env_vars on local skill invocation",
  "confidence": 0.88,
  "blast_radius": { "files": 3, "lines": 45 },
  "outcome": { "status": "success", "score": 0.88 },
  "success_streak": 3,
  "env_fingerprint": { "os": "linux", "node": "v22" },
  "execution_trace": [
    { "step": 1, "cmd": "node --version", "exit": 0 },
    { "step": 2, "cmd": "npm test", "exit": 0 }
  ]
}
```

**Capsule 完整性守卫** (`skill2gep.js::detectForgery()`):
1. 执行轨迹不能为空
2. 爆炸半径必须显示实际变更 (files > 0 OR lines > 0)
3. 至少一条轨迹条目必须有记录的退出码

### EvolutionEvent（进化事件）— 不可变的审计追踪

```json
{
  "type": "EvolutionEvent",
  "schema_version": "1.6.0",
  "id": "evt_1776784060000",
  "parent": null,
  "intent": "optimize",
  "signals": ["skill_distillation"],
  "genes_used": ["gene_skill2gep_gene_distill"],
  "mutation_id": "mut_skill2gep_run3",
  "blast_radius": { "files": 2, "lines": 110 },
  "outcome": { "status": "success", "score": 0.88 },
  "capsule_id": "cap_20260421t150740_420781e4",
  "validation_report_id": "valrpt_1776784060000",
  "env_fingerprint": { "os": "linux-6.1", "node": "22.22.0" },
  "asset_id": "sha256:..."
}
```

**关键字段**:
- `parent`: 链接到前序事件，形成进化链
- `intent`: 映射到 Gene 类别
- `blast_radius`: 本轮变更的文件/行数
- `outcome`: `success` / `failed`
- `capsule_id`: 引用生成的 Capsule

### ValidationReport（验证报告）

```json
{
  "type": "ValidationReport",
  "id": "vr_1712345678000",
  "gene_id": "gene_gep_repair_from_errors",
  "commands": [
    { "command": "node scripts/validate-suite.js", "ok": true }
  ],
  "overall_ok": true,
  "duration_ms": 1234,
  "asset_id": "sha256:..."
}
```

<!-- @end-section -->

<!-- @section: signals -->
## 二、3 层信号提取引擎

`src/gep/signals.js` (662 行，可读) — 这是 GEP 的"感官系统"。

### 第 1 层: 正则模式匹配（确定性，零延迟）

手写规则，匹配已知信号模式：

| 信号类型 | 匹配模式 |
|----------|----------|
| `log_error` | `ERROR`, `FATAL`, `Traceback`, `panic` |
| `errsig:` | 显式错误信号前缀 |
| `tool_bypass` | 代理绕过工具直接操作 |
| `capability_gap` | "I cannot", "not able to", "don't know how" |
| `user_feature_request` | "can you add", "please implement" |
| `recurring_error` | 同一错误 >= 3 次 |
| `perf_bottleneck` | "slow", "timeout", "O(n^2)" |
| `windows_shell_incompatible` | Windows/Unix 命令不兼容 |

支持 4 种语言: EN, ZH-CN, ZH-TW, JA。

### 第 2 层: 加权关键词打分（统计，零延迟）

7 个信号配置文件 (`SIGNAL_PROFILES`):

```javascript
{
  perf_bottleneck: {
    keywords: { "slow": 2, "timeout": 3, "optimize": 1, "O(n": 4 },
    threshold: 5
  },
  capability_gap: {
    keywords: { "cannot": 2, "unable": 2, "missing": 1 },
    threshold: 4
  },
  // ...
}
```

关键词权重累加，超过阈值则触发信号。

### 第 3 层: LLM 语义分析（限速，每 5 轮一次）

将语料摘要发送到 EvoMap Hub 的 `/a2a/signal/analyze` 端点，由 LLM 进行深度语义分析提取信号。

### 后处理

合并三层信号后：
- **优先级排序**: 存在可操作信号时移除表面信号
- **去重**: 抑制 8 轮内出现 >= 3 次的已处理信号
- **历史分析** (`analyzeRecentHistory`): 连续修复计数、空周期计数、失败率、信号频率、基因频率
- **平台期检测**: 触发 `plateau_pivot_required` / `plateau_pivot_suggested`

### 信号类型清单 (20+ 种)

| 类别 | 信号 |
|------|------|
| 用户需求 | `user_feature_request`, `user_improvement_suggestion` |
| 系统限制 | `perf_bottleneck`, `capability_gap` |
| 失败模式 | `recurring_error`, `repair_loop_detected`, `consecutive_failure_streak_N` |
| 生命周期 | `evolution_stagnation_detected`, `evolution_saturation`, `force_steady_state` |
| 效率 | `empty_cycle_loop_detected`, `plateau_pivot_required` |
| 完整性 | `tool_bypass` |
| 自适应 | `force_innovation_after_repair_loop` |
| 基因管理 | `ban_gene:X` |
| 生态 | `hub_search_miss_with_problem` |

<!-- @end-section -->

<!-- @section: lifecycle -->
## 三、进化生命周期

### 完整状态机

```
                    ┌──────────────────────┐
                    │   extractSignals()    │  signals.js
                    │   3 层提取引擎        │
                    └──────────┬───────────┘
                               │
                               ▼
                    ┌──────────────────────┐
                    │   fetchTasks()        │  taskReceiver.js
                    │   拉取 Hub 任务        │
                    └──────────┬───────────┘
                               │
                               ▼
                    ┌──────────────────────┐
                    │   selectGene()        │  selector.js (混淆)
                    │   personality()       │  personality.js (混淆)
                    │   选择意图 + 基因       │
                    └──────────┬───────────┘
                               │
                               ▼
                    ┌──────────────────────┐
                    │   mutate()            │  mutation.js (混淆)
                    │   规划变更             │
                    └──────────┬───────────┘
                               │
                               ▼
                    ┌──────────────────────┐
                    │   policyCheck()       │  policyCheck.js (混淆)
                    │   策略约束检查         │
                    └──────────┬───────────┘
                               │
                               ▼
                    ┌──────────────────────┐
                    │   LLM 执行            │  prompt.js (混淆)
                    │   (外部 Agent)        │  bridge.js → sessions_spawn
                    └──────────┬───────────┘
                               │
                ┌──────────────┼──────────────┐
                │                              │
                ▼                              ▼
        ┌──────────────┐            ┌──────────────────┐
        │ 验证命令执行  │            │  LLM 审查         │
        │ (validator/) │            │  (llmReview.js)   │
        └──────┬───────┘            └────────┬─────────┘
               │                             │
               └─────────────┬───────────────┘
                             │
                             ▼
                    ┌──────────────────────┐
                    │   solidify()         │  solidify.js (混淆)
                    │   1. 提交/回滚         │
                    │   2. 创建 Capsule      │
                    │   3. 追加 Event        │
                    └──────────┬───────────┘
                               │
                               ▼
                    ┌──────────────────────┐
                    │   publish()           │
                    │   hubVerify()         │
                    │   selfPR()            │
                    │   skillPublisher()    │
                    │   issueReporter()     │
                    └──────────┬───────────┘
                               │
                               ▼
                    ┌──────────────────────┐
                    │   reflection()        │  reflection.js (混淆)
                    │   distill skills      │  skillDistiller.js (混淆)
                    └──────────────────────┘
```

### 固化阶段详细流程

`solidify.js` (419KB 混淆):

1. **运行验证命令**: 执行 `Gene.validation[]` 中的命令
2. **金丝雀检查** (`canary.js`): 验证 `index.js` 仍可加载
3. **爆炸半径评估**: 计算变更文件数和行数
4. **Git 操作** (`gitOps.js`):
   - 成功 → `git commit`
   - 失败 → `git rollback` (hard/stash/none 三种模式)
5. **创建 Capsule**: 记录执行轨迹和结果
6. **追加 EvolutionEvent**: JSONL 不可变审计
7. **自动蒸馏**: 每 5 次固化触发一次技能蒸馏

### 自保护机制

- **源文件保护** (`shield.js`): 禁止 Agent 覆盖 evolver 核心代码
- **金丝雀** (`canary.js`): 防止提交损坏的代码
- **Git 回滚** (`gitOps.js`): 失败时自动恢复
- **爆炸半径限制**: `constraints.max_files` + `forbidden_paths`

<!-- @end-section -->

<!-- @section: scheduling -->
## 四、自适应调度

### 空闲调度器 (idleScheduler.js)

根据用户活跃度调节进化强度：

| 空闲时间 | 强度 | 睡眠倍率 | 额外操作 |
|----------|------|----------|----------|
| < 5 分钟 | `normal` | 1.0× | 标准周期 |
| >= 5 分钟 | `aggressive` | 0.5× | +蒸馏 +反思 +探索 |
| >= 30 分钟 | `deep` | 0.25× | +深度进化操作 |

### 守护进程主循环

```javascript
while (true) {
  await evolve.run()           // 执行一个进化周期
  adaptiveSleep()              // 自适应退避
  ommlsIdleScheduling()        // OMLS 空闲调度
  checkSuicide()               // 内存/周期数 → 重生
  checkSaturation()            // 饱和度 → 减缓
}
```

**自杀检查**: 超过 `MAX_CYCLES` (100) 或 RSS > `MAX_RSS_MB` (500) → 重生成自身

<!-- @end-section -->

<!-- @section: skill2gep -->
## 五、Skill ↔ GEP 双向蒸馏

### 正向蒸馏: Capsule 流 → Gene

`skillDistiller.js` (413KB 混淆):
- 从成功的 Capsule 流中提取模式
- 生成新的 Gene 定义
- 自动合并相似的 Gene

### 反向蒸馏: SKILL.md → Gene + Capsule

`skill2gep.js` (645 行，可读):

```
SKILL.md (Markdown)
    │
    ▼
parseSkillMd() → { signals_match, strategy, avoid, validation }
    │
    ▼
synthesizeGene() → 草稿 Gene
    │
    ▼
validateSynthesizedGene() → 通过 skillDistiller 验证
    │
    ▼
assembleCapsule() → 交叉引用 Gene.validation vs execution.trace
    │
    ▼
upsertGene() + appendCapsule()
```

**防伪造检查** (`detectForgery`):
1. 执行轨迹非空
2. 爆炸半径 > 0
3. 至少一条轨迹有退出码

### Gene → SKILL.md 发布

`skillPublisher.js` (352 行，可读):
- 将本地 Gene 转换为 SKILL.md 格式
- 发布到 EvoMap Hub Skill Store (`/a2a/skill/store/publish`)

<!-- @end-section -->

<!-- @section: modules -->
## 六、GEP 模块清单

### 可读模块 (26 个)

| 模块 | 行数 | 用途 |
|------|------|------|
| `signals.js` | 662 | 3 层信号提取 |
| `assetStore.js` | 448 | JSONL 资产存储 + 文件锁 |
| `skill2gep.js` | 645 | SKILL.md → Gene 反向蒸馏 |
| `taskReceiver.js` | 565 | Hub 任务拉取/声明/完成 |
| `questionGenerator.js` | 415 | 主动问题生成 |
| `selfPR.js` | 404 | 自动 GitHub PR |
| `skillPublisher.js` | 352 | Gene → SKILL.md 发布 |
| `issueReporter.js` | 329 | 自动 GitHub Issue |
| `executionTrace.js` | 291 | 结构化执行轨迹 |
| `localStateAwareness.js` | 244 | 本地状态捕获 |
| `gitOps.js` | 234 | Git 操作/回滚 |
| `paths.js` | 218 | 路径解析 |
| `privacyClient.js` | 216 | 隐私计算 |
| `sanitize.js` | 179 | 泄漏扫描 |
| `a2a.js` | 173 | A2A 广播 |
| `idleScheduler.js` | 168 | 空闲调度 |
| `assetCallLog.js` | 130 | 资产调用日志 |
| `claimNudge.js` | 121 | 控制台提醒 |
| `featureFlags.js` | 114 | 特性开关 |
| `directoryClient.js` | 110 | 目录客户端 |
| `llmReview.js` | 92 | LLM 代码审查 |
| `mailboxTransport.js` | 82 | 邮箱传输 |
| `bridge.js` | 71 | Prompt 桥接 |
| `validationReport.js` | 55 | 验证报告 |
| `assets.js` | 36 | 资产格式化 |
| `analyzer.js` | 35 | 自我修正分析 |

### 混淆模块 (27 个，全部位于 `src/gep/` 顶层)

| 模块 | 大小 | 用途 |
|------|------|------|
| `solidify.js` | 409KB | 固化引擎 |
| `skillDistiller.js` | 404KB | 正向蒸馏 |
| `a2aProtocol.js` | 317KB | A2A 核心协议 |
| `prompt.js` | 243KB | GEP Prompt 组装 |
| `memoryGraph.js` | 239KB | 记忆图 |
| `policyCheck.js` | 186KB | 策略约束检查 |
| `selector.js` | 135KB | 基因选择器 |
| `personality.js` | 123KB | 人格状态 |
| `hubSearch.js` | 87KB | Hub 语义搜索 |
| `candidates.js` | 77KB | 候选管理 |
| `hubVerify.js` | 70KB | Hub 资产验证 |
| `explore.js` | 66KB | 探索式信号发现 |
| `mutation.js` | 66KB | 变异生成 |
| `reflection.js` | 64KB | 自反思 |
| `hubReview.js` | 62KB | Hub 审查 |
| `curriculum.js` | 55KB | 学习课程 |
| `deviceId.js` | 52KB | 设备指纹 |
| `memoryGraphAdapter.js` | 37KB | 记忆图 I/O |
| `strategy.js` | 32KB | 策略预设（公开 6 个名称：balanced / innovate / harden / repair-only / early-stabilize / steady-state） |
| `narrativeMemory.js` | 31KB | 叙述性记忆 |
| `learningSignals.js` | 28KB | 学习信号 |
| `integrityCheck.js` | 27KB | 完整性验证 |
| `candidateEval.js` | 25KB | 候选评估（顶层调用 `_0x...` 风格的混淆代码） |
| `envFingerprint.js` | 23KB | 环境指纹 |
| `shield.js` | 23KB | 源文件保护 |
| `crypto.js` | 23KB | AES-256-GCM |
| `contentHash.js` | 20KB | SHA-256 资产哈希 |

> 大小按 `du -k` 截断的 KiB（1024 进制），与 `wc -c` 字节数除 1024 一致；与历史文档相比修正了向上取整偏差。
>
> **不在此表内但同样混淆**：`src/evolve.js`（约 747KB，主进化循环编排器）位于 `src/` 顶层而非 `src/gep/` 子包，按目录归类不并入此表。如把它一并算进"核心引擎混淆资产"，则混淆文件总数为 28 个、约 3.6MB。

<!-- @end-section -->

<!-- @section: related -->
## 相关文档

- [[01-overview|Evolver 项目架构总览]]
- [[03-atp-protocol|ATP Agent 交易协议分析]]
- [[05-data-models|数据模型与资产系统分析]]

<!-- @end-section -->
