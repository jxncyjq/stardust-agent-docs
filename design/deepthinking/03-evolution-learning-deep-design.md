---
id: "deepthinking-evolution-003"
title: "进化学习系统深度设计"
aliases: ["evolution learning deep design", "进化系统设计", "GEP legion design"]
type: "deepthinking"
category: "design/deepthinking"
tags: ["evolution", "learning", "GEP", "self-improvement", "deep-design"]
version: "1.0.0"
created: "2026-05-04"
updated: "2026-05-04"
author: "jxncyjq"
status: "review"
parent: null
related_docs:
  - id: "analysis-evolver-gep-002"
    relation: "reference"
    path: "../analysis/evolver/02-gep-protocol.md"
  - id: "analysis-evolver-insights-006"
    relation: "reference"
    path: "../analysis/evolver/06-evolver-insights.md"
  - id: "analysis-evolver-vs-hermes-007"
    relation: "reference"
    path: "../analysis/evolver/07-evolver-vs-hermes.md"
  - id: "architecture-legion"
    relation: "parent_design"
    path: "../architecture/Legion.md"
---

<!-- @section: overview -->
# 进化学习系统深度设计

## 核心命题

> 如何将 Evolver 的 GEP 协议"内建"到 Legion 的 Agent 引擎中，而非作为外部"寄生"层？

答案：**信号感知 + 策略匹配 + 经验固化 + 进化评估**四阶段闭环，原生集成在 Agent 运行时中，而非外挂。

## 一、Evolver 的核心贡献 — 和它的局限性

### 1.1 Evolver 贡献了什么

Evolver 证明了一个关键命题：**Agent 的自我改进可以被形式化为协议**。

```
Evolver 的四大贡献:
  1. 3 层信号提取 — Agent 的感官系统
  2. Gene 策略模板 — 可复用的"知道做什么"
  3. Capsule 执行证明 — 可验证的"证明这样做有效"
  4. 固化安全网 — Agent 自我修改的安全保障
```

### 1.2 Evolver 的局限性 — Legion 需要改进的地方

| Evolver 局限 | 为什么是问题 | Legion 如何改进 |
|-------------|------------|---------------|
| 寄生式架构 | 依赖外部 Agent，无法自主执行 | **内建式** — Agent 引擎原生进化 |
| 正则信号提取 | 脆弱、难维护 | **结构化 Schema** — 类型安全的信号定义 |
| JSONL 存储 | 查询性能差 | **数据库 + 向量检索** |
| JavaScript/混淆 | 不可审计 | **Go 实现 + 完全开源** |
| 单 Agent 进化 | 经验无法共享 | **多 Agent 协作进化** |
| 无统计学验证 | 置信度评分粗糙 | **A/B 对照 + 显著性检验** |
| Git 强依赖 | 非 Git 环境不可用 | **数据库回滚 + Git 可选** |
| Hub 单点依赖 | 服务中断风险 | **P2P 经验共享** |

<!-- @end-section -->

<!-- @section: four-phase-loop -->
## 二、进化闭环 — 四阶段架构

### 2.1 总览

```
┌─────────────────────────────────────────────────────────┐
│                   Legion 进化学习系统                      │
│                                                          │
│  ┌──────────┐   ┌──────────┐   ┌──────────┐   ┌───────┐│
│  │ 1.信号感知│ → │2.策略匹配│ → │3.经验固化│ → │4.进化 ││
│  │ Signal   │   │ Strategy │   │Solidify  │   │评估   ││
│  │ Detector │   │ Selector │   │ Engine   │   │Eval   ││
│  └──────────┘   └──────────┘   └──────────┘   └───────┘│
│       │              │              │              │     │
│       ▼              ▼              ▼              ▼     │
│  Agent 日志       Gene 库        Capsule 库    进化报告  │
│  工具调用结果     策略模板       执行证明       能力雷达  │
│  任务反馈         约束条件       审计事件       退化预警  │
└─────────────────────────────────────────────────────────┘
```

### 2.2 与 Evolver GEP 的对应关系

| Evolver GEP | Legion 进化系统 | 改进 |
|------------|---------------|------|
| signals.js (混淆) | SignalDetector (Go) | 结构化 Schema，类型安全 |
| genes.json | GeneRegistry (PostgreSQL) | 结构化查询，版本管理 |
| capsules.json | CapsuleStore (PostgreSQL) | 统计学验证 |
| events.jsonl | AuditLog (不可变追加表) | 数据库级完整性 |
| solidify.js (混淆) | SolidifyEngine (Go) | 完全可审计 |
| validator/ | ValidationPipeline | 可扩展验证策略 |
| PROMPT_MAX_CHARS=24000 | 上下文管理统一处理 | Agent 引擎原生压缩 |

<!-- @end-section -->

<!-- @section: signal-detector -->
## 三、信号感知层 — Agent 的感官系统

### 3.1 结构化信号 Schema

Evolver 用正则表达式匹配信号，维护成本高。Legion 使用结构化的信号类型：

```go
// SignalType 定义可识别的信号类型
type SignalType string

const (
    SignalTaskSuccess    SignalType = "task_success"     // 任务成功
    SignalTaskFailure    SignalType = "task_failure"     // 任务失败
    SignalErrorPattern   SignalType = "error_pattern"    // 重复出现的错误
    SignalHumanFeedback  SignalType = "human_feedback"   // 人类反馈
    SignalPeerFeedback   SignalType = "peer_feedback"    // 同伴反馈
    SignalInefficiency   SignalType = "inefficiency"     // 效率低下
    SignalNovelSituation SignalType = "novel_situation"  // 遇到新情况
    SignalBetterWay      SignalType = "better_way"       // 发现了更好的方法
)

// Signal 结构化的信号定义
type Signal struct {
    ID          string      `json:"id"`
    Type        SignalType  `json:"type"`
    Source      SignalSource `json:"source"`  // 来源: 哪个 Agent 的哪个任务
    Severity    float64     `json:"severity"` // 0.0 ~ 1.0
    Context     SignalContext `json:"context"`
    Evidence    string      `json:"evidence"` // 支撑证据
    Timestamp   time.Time   `json:"timestamp"`
    Extraction  ExtractionMethod `json:"extraction"`
}

type SignalContext struct {
    TaskID      string      `json:"task_id"`
    TaskType    string      `json:"task_type"`
    AgentID     string      `json:"agent_id"`
    AgentRole   string      `json:"agent_role"`
    ModelUsed   string      `json:"model_used"`
    InputSummary string     `json:"input_summary"`
    OutputSummary string    `json:"output_summary"`
    ErrorType   string      `json:"error_type,omitempty"`
    Feedback    string      `json:"feedback,omitempty"`
}

type ExtractionMethod string
const (
    ExtractionRule   ExtractionMethod = "rule"    // 确定性规则
    ExtractionHeuristic ExtractionMethod = "heuristic" // 启发式
    ExtractionLLM    ExtractionMethod = "llm"     // LLM 语义分析
)
```

### 3.2 三层提取策略 (继承并改进 Evolver)

```
第 1 层: 确定性规则 (零成本, 即时)
  → 明确错误模式 (编译错误、测试失败、API 错误码)
  → 显式反馈信号 (人类评分、审批拒绝)
  → 阈值触发 (Token 超预算、执行超时)
  → 覆盖 60-70% 的信号

第 2 层: 启发式打分 (零成本, 即时)
  → 关键词频率异常
  → 工具调用模式重复
  → 任务完成时间趋势
  → 覆盖 15-25% 的信号

第 3 层: LLM 语义分析 (有成本, 每 N 轮或每 M 个任务)
  → 复杂因果分析 ("为什么这个 PR 被反复拒绝？")
  → 新模式发现 ("Agent 在处理 X 类问题时总是先做 Y")
  → 跨任务的抽象洞察
  → 覆盖 5-15% 的信号
```

### 3.3 信号后处理管线 (借鉴 Evolver 的防循环机制)

```go
type SignalPipeline struct {
    Deduplicator  *Deduplicator   // N 轮内出现 >= 3 次 → 抑制
    Prioritizer   *Prioritizer     // 可操作 > 表面
    CycleDetector *CycleDetector  // 修复循环检测 → 强制创新
    PlateauDetector *PlateauDetector // 平台期检测 → 建议转向
}

func (p *SignalPipeline) Process(rawSignals []Signal) []Signal {
    signals := p.Deduplicator.Dedup(rawSignals)
    signals = p.Prioritizer.Prioritize(signals)
    signals = p.CycleDetector.Filter(signals)
    signals = p.PlateauDetector.Enhance(signals)
    return signals
}
```

<!-- @end-section -->

<!-- @section: gene-system -->
## 四、Gene 策略模板系统

### 4.1 Gene — 可复用的策略资产

Evolver 的 Gene 是自然语言 Prompt。Legion 的 Gene 升级为结构化策略：

```go
type Gene struct {
    ID            string        `json:"id"`            // gene_{category}_{slug}
    Version       int           `json:"version"`       // 版本号
    Category      GeneCategory  `json:"category"`      // repair | optimize | innovate
    SignalsMatch  []SignalType  `json:"signals_match"` // 匹配的信号类型
    Preconditions []Condition   `json:"preconditions"` // 前置条件
    Strategy      []Strategy    `json:"strategy"`      // 策略步骤
    Avoid         []string      `json:"avoid"`         // 禁止事项
    Constraints   Constraints   `json:"constraints"`   // 约束条件
    Validation    []Validation  `json:"validation"`    // 验证步骤
    Performance   Performance   `json:"performance"`   // 历史表现数据
    ParentGeneID  string        `json:"parent_gene_id"` // 派生来源 (A/B 测试)
    Status        GeneStatus    `json:"status"`        // active | deprecated | testing
    CreatedAt     time.Time     `json:"created_at"`
    UpdatedAt     time.Time     `json:"updated_at"`
}

type Strategy struct {
    Order    int               `json:"order"`
    Action   string            `json:"action"`    // 自然语言描述
    Rationale string           `json:"rationale"` // 为什么这样做
    Evidence []EvidenceRef     `json:"evidence"`  // 支撑证据引用
}

type Constraints struct {
    MaxFiles       int      `json:"max_files"`        // 最大修改文件数
    ForbiddenPaths []string `json:"forbidden_paths"`  // 禁止修改的路径
    MaxTokens      int      `json:"max_tokens"`       // 最大 Token 消耗
    RequiresApproval bool   `json:"requires_approval"` // 是否需要人工审批
}

type Validation struct {
    Command  string `json:"command"`   // 验证命令
    Expected string `json:"expected"`  // 预期结果
    TimeoutMs int   `json:"timeout_ms"`
}

type Performance struct {
    SuccessCount   int     `json:"success_count"`
    FailureCount   int     `json:"failure_count"`
    AvgScore       float64 `json:"avg_score"`        // 平均结果评分
    AvgCostTokens  int     `json:"avg_cost_tokens"`  // 平均 Token 成本
    LastUsedAt     time.Time `json:"last_used_at"`
    Effectiveness  float64 `json:"effectiveness"`    // 综合有效性 (0-1)
}
```

### 4.2 Gene 的生命周期

```
创建 (created)
  │ 新信号类型 → 创建实验性 Gene
  │ 状态: testing
  │
  ├── A/B 测试
  │     对照组 A: 使用旧 Gene (或无 Gene)
  │     实验组 B: 使用新 Gene
  │     → 统计学显著性检验
  │
  ├── 验证通过 → 状态: active
  │     成功率 >= 阈值
  │     成本可控
  │     无安全事故
  │
  ├── 持续监控
  │     Performance 持续更新
  │     有效性下降 → 标记为 degraded
  │     长时间未使用 → 标记为 stale
  │
  └── 废弃
       状态: deprecated
       保留审计记录，不再推荐使用
```

### 4.3 Gene 的生成方式

| 来源 | 方式 | 示例 |
|------|------|------|
| 系统预设 | 内置基础 Gene | "从编译错误中修复" |
| 人类编写 | 专家定义策略模板 | "代码审查最佳实践" |
| Agent 蒸馏 | 从成功 Capsule 中提炼 | "这 10 次成功任务共同使用了 X 方法" |
| A/B 实验 | 变异生成新策略 | 对现有 Gene 做小幅变异 |
| 协作共享 | 从其他 Agent 导入 | "Agent B 在处理这类问题时的方法" |

<!-- @end-section -->

<!-- @section: capsule -->
## 五、Capsule — 执行证明与经验资产

### 5.1 Capsule 结构

```go
type Capsule struct {
    ID              string        `json:"id"`
    GeneID          string        `json:"gene_id"`       // 使用的 Gene
    AgentID         string        `json:"agent_id"`
    TaskID          string        `json:"task_id"`
    TriggerSignals  []SignalType  `json:"trigger_signals"`
    Outcome         Outcome       `json:"outcome"`
    ExecutionTrace  []TraceEntry  `json:"execution_trace"` // 完整执行轨迹
    CostBreakdown   CostBreakdown `json:"cost_breakdown"`
    BlastRadius     BlastRadius   `json:"blast_radius"`
    EnvFingerprint  EnvFingerprint `json:"env_fingerprint"` // 环境指纹
    ConfidenceScore float64       `json:"confidence_score"` // 置信度 0-1
    ContentHash     string        `json:"content_hash"`    // SHA-256
    CreatedAt       time.Time     `json:"created_at"`
}

type TraceEntry struct {
    Step      int       `json:"step"`
    Action    string    `json:"action"`    // 自然语言描述
    Command   string    `json:"command,omitempty"`
    ExitCode  int       `json:"exit_code,omitempty"`
    Output    string    `json:"output,omitempty"`
    DurationMs int      `json:"duration_ms"`
}

type Outcome struct {
    Status      string  `json:"status"`  // success | partial | failure
    Score       float64 `json:"score"`   // 0-100
    HumanRating *float64 `json:"human_rating,omitempty"`
    PeerRating  *float64 `json:"peer_rating,omitempty"`
    AutoRating  float64 `json:"auto_rating"` // 自动化指标
}
```

### 5.2 Capsule 的统计学验证

Evolver 的 Capsule 只有一个粗糙的 `confidence` 分数。Legion 需要统计学的严谨性：

```go
type StatisticalValidation struct {
    CapsuleID       string
    SampleSize      int     // A/B 测试样本数
    ControlMean     float64 // 对照组 (无 Gene) 平均结果
    ExperimentMean  float64 // 实验组 (有 Gene) 平均结果
    PValue          float64 // 统计学显著性
    EffectSize      float64 // Cohen's d
    ConfidenceInterval struct {
        Lower float64
        Upper float64
    }
    RecommendedAt   time.Time
}
```

**发布阈值**：只有当 Gene 的 `effectiveness >= 0.78`（Evolver 的 MIN_PUBLISH_SCORE）且 A/B 测试的 `p_value < 0.05` 时，Gene 才从 `testing` 转为 `active`。

<!-- @end-section -->

<!-- @section: solidify -->
## 六、固化安全网 — Agent 自我修改的防护

### 6.1 固化流程 (继承并增强 Evolver)

```
Agent 完成自我改进操作后:

  ┌─────────────────────────────────────────┐
  │          SolidifyEngine.Solidify()        │
  │                                          │
  │  1. 修改范围检查                          │
  │     ├── 修改文件数 > max_files? → 拒绝    │
  │     ├── 触及 forbidden_paths? → 拒绝      │
  │     └── 修改超过 explosion_radius? → 拒绝 │
  │                                          │
  │  2. 安全检查                              │
  │     ├── 泄漏扫描 (27+ 模式)               │
  │     ├── 环境值反向检测                    │
  │     └── 敏感路径保护                     │
  │                                          │
  │  3. 金丝雀检查                            │
  │     ├── 系统可启动? (health check)        │
  │     ├── 关键 API 可调用?                  │
  │     └── 数据库连接正常?                   │
  │                                          │
  │  4. 验证命令执行                          │
  │     └── Gene.Validation[] → 逐一运行      │
  │                                          │
  │  5. 决策                                  │
  │     ├── 全部通过 → 提交 + 创建 Capsule    │
  │     └── 任一失败 → 自动回滚 + 记录失败     │
  │                                          │
  │  6. 审计事件追加                          │
  │     └── EvolutionEvent → 不可变审计日志    │
  └─────────────────────────────────────────┘
```

### 6.2 回滚机制 (突破 Git 强依赖)

Evolver 强依赖 Git 做回滚。Legion 支持多种回滚方式：

| 修改类型 | 回滚方式 |
|---------|---------|
| 代码修改 | Git revert (优先) / 数据库快照回滚 |
| 配置修改 | 配置版本管理 (数据库中存储历史版本) |
| 知识库修改 | Wiki 版本回滚 |
| 数据库记录 | 事务回滚 / 快照恢复 |
| 外部系统 | Saga 补偿事务 |

<!-- @end-section -->

<!-- @section: evolution-evaluator -->
## 七、进化评估器 — 量化 Agent 成长

### 7.1 评估维度

```go
type EvolutionReport struct {
    AgentID     string
    Period      TimeRange
    GeneratedAt time.Time

    TaskQuality     QualityMetrics      // 任务质量
    DecisionEfficiency EfficiencyMetrics // 决策效率
    LearningEffect  LearningMetrics     // 学习效果
    Collaboration   CollaborationMetrics // 协作能力
    RiskPerformance RiskMetrics         // 风险表现

    RadarChart      RadarData           // 能力雷达图
    Trends          []TrendLine         // 趋势分析
    Alerts          []Alert             // 退化预警
    Recommendations []Recommendation    // 改进建议
}

type QualityMetrics struct {
    FirstPassRate   float64  // 一次通过率
    ReworkRate      float64  // 返工率
    HumanScoreAvg   float64  // 人类评分均值
    HumanScoreTrend TrendLine // 评分趋势
}

type LearningMetrics struct {
    NewGenesCreated    int
    CapsulesPublished  int
    SameTaskTypeTrend  TrendLine  // 同类任务表现趋势
    SkillsAcquired     []string   // 新获得的能力
}
```

### 7.2 退化检测

借鉴 Evolver 的平台期检测 + 增强的退化监控：

```go
type DegradationDetector struct {
    Thresholds DegradationThresholds
}

func (d *DegradationDetector) Check(report EvolutionReport) []Alert {
    var alerts []Alert

    // 连续 N 期质量下降
    if report.TaskQuality.HumanScoreTrend.Slope < d.Thresholds.MinSlope {
        alerts = append(alerts, Alert{
            Level:   Warning,
            Message: "Agent 任务质量连续下降，建议审查最近的学习记录",
        })
    }

    // 返工率异常上升
    if report.TaskQuality.ReworkRate > d.Thresholds.MaxReworkRate {
        alerts = append(alerts, Alert{
            Level:   Critical,
            Message: "返工率超出阈值，可能学到不当的工作模式",
        })
    }

    return alerts
}
```

### 7.3 进化策略预设 (借鉴 Evolver 的 6 种模式)

```yaml
evolution_presets:
  conservative:     # 生产环境
    auto_apply: false
    requires_approval: true
    max_blast_radius: { files: 1, lines: 20 }
    signal_threshold: high
    gene_tier: [active]  # 仅使用已验证的策略

  balanced:         # 默认
    auto_apply: true
    requires_approval: { when: "blast_radius.files > 3" }
    max_blast_radius: { files: 5, lines: 100 }
    signal_threshold: medium

  aggressive:       # 开发环境
    auto_apply: true
    requires_approval: { when: "blast_radius.files > 10" }
    max_blast_radius: { files: 15, lines: 300 }
    signal_threshold: low
    gene_tier: [active, testing]  # 允许实验性策略

  repair_only:      # 仅修复
    gene_category: [repair]

  optimize_only:    # 仅优化
    gene_category: [optimize]

  full_exploration: # 全探索
    gene_category: [repair, optimize, innovate]
    auto_create_genes: true  # 允许 Agent 自创新策略
```

<!-- @end-section -->

<!-- @section: sharing -->
## 八、多 Agent 进化 — 经验共享与协作学习

这是 Evolver 完全不具备的能力。

### 8.1 经验共享模型

```
Agent A                           Agent B
  │                                 │
  ├── 执行任务                       │
  ├── 创建 Capsule                  │
  ├── 提炼 Gene                     │
  │                                 │
  │   Gene 发布到 LLM Wiki ────────→ │
  │                                 ├── 发现相关 Gene
  │                                 ├── 在自己的任务中尝试
  │                                 ├── 成功 → 反馈验证
  │                                 └── 失败 → 反馈不适用原因
  │                                     │
  └── 收到反馈 ←──────────────────────┘
```

### 8.2 协作学习路径

Legion 架构方案中定义的四种学习路径在进化系统中的实现：

| 路径 | 实现方式 |
|------|---------|
| 任务反馈学习 | 任务完成 → 信号提取 → Gene 匹配 → Capsule 创建 |
| 自我反思学习 | 任务周期结束 → 自动触发反思 → 提取经验 → 更新行为 |
| 协作观察学习 | 上下游 Agent 的输出质量 → 间接反馈 → 提取最佳实践 |
| 知识库驱动学习 | 心跳感知 Wiki 更新 → 检索相关 Gene → 更新本地策略缓存 |

<!-- @end-section -->

<!-- @section: design-decisions -->
## 九、关键设计决策

| 决策 | 选择 | vs Evolver |
|------|------|-----------|
| 架构模式 | 内建式 (Agent 引擎内部) | Evolver 是寄生式 |
| 信号定义 | 结构化 Schema (Go struct) | Evolver 是正则表达式 |
| Gene 存储 | PostgreSQL + 版本管理 | Evolver 是 JSON 文件 |
| Capsule 验证 | 统计学显著性 + A/B | Evolver 是简单置信度 |
| 回滚机制 | 数据库回滚 + Git (可选) | Evolver 强依赖 Git |
| Agent 数量 | 多 Agent 协作进化 | Evolver 仅单 Agent |
| 代码透明度 | 完全开源 | Evolver 核心混淆 |
| 评估 | 多维度雷达图 + 退化检测 | Evolver 无评估系统 |
| 策略预设 | 6 种模式 (同 Evolver) | 同 Evolver |

### 不做什么

1. **不做寄生式架构** — 进化是 Agent 引擎的核心能力，不是外挂
2. **不依赖 Git** — 支持非 Git 环境的回滚
3. **不混淆代码** — 完全开源可审计
4. **不做单 Agent 封闭进化** — 经验通过 Wiki 在 Agent 间共享

<!-- @end-section -->

<!-- @section: related -->
## 相关文档

- [[../analysis/evolver/02-gep-protocol|Evolver GEP 协议分析]]
- [[../analysis/evolver/07-evolver-vs-hermes|Evolver vs Hermes 对比]]
- [[../architecture/Legion|Legion 项目方案 — Agent 引擎与学习引擎]]
- [[02-agent-runtime-deep-design|Agent 运行时深度设计]]
- [[04-wiki-knowledge-deep-design|LLM Wiki 知识引擎深度设计]]
- [[05-multi-agent-collaboration|多智能体协作深度设计]]

<!-- @end-section -->
