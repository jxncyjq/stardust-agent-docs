---
id: "deepthinking-wiki-004"
title: "LLM Wiki 知识引擎深度设计"
aliases: ["wiki knowledge deep design", "Wiki知识引擎设计", "knowledge graph design"]
type: "deepthinking"
category: "design/deepthinking"
tags: ["wiki", "knowledge", "graph", "rag", "deep-design"]
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
  - id: "analysis-hermes-tools-003"
    relation: "reference"
    path: "../analysis/hermes/03-tools-skills-plugins.md"
  - id: "analysis-hermes-datamodels-005"
    relation: "reference"
    path: "../analysis/hermes/05-data-models.md"
  - id: "architecture-legion"
    relation: "parent_design"
    path: "../architecture/Legion.md"
---

<!-- @section: overview -->
# LLM Wiki 知识引擎深度设计

## 核心命题

> LLM Wiki 应该是"文档库 + RAG"还是"人机共建的知识图谱"？

答案：**人机共建的知识图谱**。hermes 的技能系统证明了文件级别的知识管理可以做到 200+ 技能的规模。Evolver 的 Gene/Capsule 模型证明了结构化知识资产的价值。Legion 的 Wiki 应该在两者基础上更进一步——用知识图谱取代文件系统，用结构化的知识条目取代 Markdown 文件。

## 一、从参考项目中提炼的 Wiki 设计原则

### 1.1 hermes 技能系统的启示

| 做对了什么 | 为什么有效 | Legion 如何继承 |
|-----------|----------|---------------|
| 渐进式加载 | 200+ 技能不能全塞进系统提示 | 元数据索引 + 按需检索 |
| SKILL.md 格式 | 简单、统一的知识载体 | 知识条目统一 Schema |
| 安全扫描 488+ 模式 | 社区技能的信任问题 | 知识审核流程 + 安全扫描 |
| 信任级别 | builtin > trusted > community | 知识来源可信度标注 |
| 技能策展 | 活跃 → 陈旧 → 归档 | 知识生命周期管理 |

| 没做好什么 | 为什么不够 | Legion 如何改进 |
|-----------|----------|---------------|
| 文件系统存储 | 无结构化查询 | 知识图谱 + 全文索引 + 向量检索 |
| 静态知识 | 无法自动更新 | Agent 可写入、更新知识 |
| 无版本管理 | 技能版本靠文件名 | Git 级别的版本管理 |
| 无关联关系 | 技能之间无显式链接 | 知识图谱的关联编织 |
| 人写 AI 读 | 单向流动 | 人机共建共享 |

### 1.2 Evolver 资产模型的启示

Evolver 的 Gene/Capsule/Event 三元模型天然适合 Wiki：

| Evolver 资产 | Wiki 映射 | 用途 |
|-------------|----------|------|
| Gene | 策略模板条目 | "遇到 X 类问题，按 Y 步骤解决" |
| Capsule | 案例条目 | "策略 Z 在环境 W 下成功 N 次" |
| EvolutionEvent | 审计日志 | "谁在何时做了什么，结果如何" |
| 内容寻址 (SHA-256) | 知识完整性 | 检测篡改、去重、可信共享 |

<!-- @end-section -->

<!-- @section: knowledge-model -->
## 二、知识模型 — Wiki 的核心数据结构

### 2.1 知识条目 (Knowledge Entry)

Wiki 的核心单元是知识条目，而非 Markdown 文件：

```go
type KnowledgeEntry struct {
    ID              string          `json:"id"`
    Type            EntryType       `json:"type"`
    Title           string          `json:"title"`
    Content         ContentBlock    `json:"content"`     // 结构化内容
    Status          EntryStatus     `json:"status"`
    Visibility      Visibility      `json:"visibility"`
    Version         int             `json:"version"`
    ContentHash     string          `json:"content_hash"` // SHA-256

    // 溯源
    Author          Author          `json:"author"`       // 人类 / Agent
    SourceTaskID    string          `json:"source_task_id,omitempty"`
    SourceCapsuleID string          `json:"source_capsule_id,omitempty"`

    // 知识图谱
    Tags            []string        `json:"tags"`
    Domain          string          `json:"domain"`       // 知识域
    Relations       []Relation      `json:"relations"`    // 关联关系
    Embedding       []float32       `json:"embedding"`    // 向量嵌入

    // 治理
    ReviewStatus    ReviewStatus    `json:"review_status"`
    QualityScore    float64         `json:"quality_score"`
    ExpiresAt       *time.Time      `json:"expires_at,omitempty"`
    ViewCount       int             `json:"view_count"`
    UsefulCount     int             `json:"useful_count"`

    CreatedAt       time.Time       `json:"created_at"`
    UpdatedAt       time.Time       `json:"updated_at"`
}

type EntryType string
const (
    EntryDocument     EntryType = "document"     // 文档/指南
    EntryStrategy     EntryType = "strategy"     // Gene — 策略模板
    EntryCase         EntryType = "case"         // Capsule — 成功案例
    EntryBestPractice EntryType = "best_practice" // 最佳实践
    EntryFAQ          EntryType = "faq"          // 常见问题
    EntryAPIDoc       EntryType = "api_doc"      // API 文档
    EntryDecision     EntryType = "decision"     // 决策记录
    EntryResearch     EntryType = "research"     // 研究报告
)

type EntryStatus string
const (
    StatusDraft     EntryStatus = "draft"
    StatusReview    EntryStatus = "review"      // 待审核
    StatusPublished EntryStatus = "published"
    StatusArchived  EntryStatus = "archived"
    StatusDeprecated EntryStatus = "deprecated"
)

type Author struct {
    Type   AuthorType `json:"type"`   // human | agent
    ID     string     `json:"id"`     // 用户 ID 或 Agent ID
    Name   string     `json:"name"`
}

type ReviewStatus string
const (
    ReviewNone       ReviewStatus = "none"        // 不需要审核 (人类创建)
    ReviewPending    ReviewStatus = "pending"     // 待审核 (Agent 创建)
    ReviewApproved   ReviewStatus = "approved"    // 已审核通过
    ReviewRejected   ReviewStatus = "rejected"    // 已驳回
    ReviewCrossChecked ReviewStatus = "cross_checked" // 交叉验证通过
)
```

### 2.2 知识关系 (Knowledge Relations)

知识图谱的核心是关系：

```go
type Relation struct {
    SourceID    string        `json:"source_id"`    // 源条目 ID
    TargetID    string        `json:"target_id"`    // 目标条目 ID
    Type        RelationType  `json:"type"`
    Weight      float64       `json:"weight"`       // 关系强度 0-1
    Evidence    string        `json:"evidence"`     // 关系依据
}

type RelationType string
const (
    RelDerivesFrom     RelationType = "derives_from"     // 派生自
    RelGeneralizes     RelationType = "generalizes"      // 泛化
    RelSpecializes     RelationType = "specializes"      // 特化
    RelDependsOn       RelationType = "depends_on"       // 依赖
    RelConflicts       RelationType = "conflicts_with"   // 矛盾
    RelSupports        RelationType = "supports"         // 支持
    RelReplaces        RelationType = "replaces"         // 取代
    RelRelatesTo       RelationType = "relates_to"       // 关联
    RelExample         RelationType = "example_of"       // 是...的示例
    RelPreviousVersion RelationType = "previous_version" // 前一个版本
)
```

### 2.3 知识域 (Knowledge Domain)

借鉴 Legion 架构方案中的组织架构：

```yaml
knowledge_domains:
  - id: "global"
    name: "公司级知识"
    visibility: company
    examples: ["企业制度", "产品文档", "技术规范"]

  - id: "engineering"
    name: "工程部知识"
    visibility: department
    parent: global

  - id: "product"
    name: "产品部知识"
    visibility: department
    parent: global

  - id: "agent_private"
    name: "Agent 私有知识"
    visibility: agent_only
    examples: ["经验记忆", "学习笔记"]
```

<!-- @end-section -->

<!-- @section: knowledge-pipeline -->
## 三、知识加工管线 — 从原始内容到结构化知识

### 3.1 五步加工流程 (继承架构方案的 5 步加工)

```
原始内容 (文档 / Agent 产出 / 对话)
  │
  ▼
① 智能切片 (Chunking)
  ├── 语义分块 (按段落/章节/主题)
  ├── 保持知识点完整性 (不切断逻辑)
  └── 生成切片摘要
  │
  ▼
② 结构化抽取 (Extraction)
  ├── 实体提取 (人名、术语、API、版本号)
  ├── 关系提取 (依赖、调用、继承)
  ├── 属性提取 (参数、配置、约束)
  └── 生成结构化标签
  │
  ▼
③ 多模态嵌入 (Embedding)
  ├── 文本 → text-embedding-3-large
  ├── 代码 → code-embedding (专用模型)
  ├── 图表 → 描述文本 → text-embedding
  └── 存储向量索引
  │
  ▼
④ 关联编织 (Linking)
  ├── 自动发现与已有知识的关联
  ├── 语义相似度 → 关联建议
  ├── 图谱遍历 → 间接关联
  └── 人工确认关键关联
  │
  ▼
⑤ 质量评分 (Scoring)
  ├── 完整性: 是否包含必要字段
  ├── 一致性: 是否与已有知识矛盾
  ├── 时效性: 是否过期
  ├── 引用度: 被其他条目引用的频率
  └── 更新频率: 是否长期未更新
```

### 3.2 矛盾检测与冲突裁决

```go
type ConflictDetector struct {
    EmbeddingThreshold float64  // 语义相似度阈值
}

func (d *ConflictDetector) Detect(newEntry KnowledgeEntry, existing []KnowledgeEntry) []Conflict {
    var conflicts []Conflict

    for _, existing := range existing {
        // 语义高度相似但结论不同 → 可能是矛盾
        if cosineSimilarity(newEntry.Embedding, existing.Embedding) > d.EmbeddingThreshold {
            if newEntry.hasContradictoryClaim(existing) {
                conflicts = append(conflicts, Conflict{
                    EntryA: existing,
                    EntryB: newEntry,
                    Type:   "contradictory_claim",
                })
            }
        }
    }

    return conflicts
}

// 冲突裁决流程
// 新知识写入 → 检测到矛盾 → 标记冲突
//   → 低优先级: 两个条目都保留 + 标注冲突关系
//   → 高优先级: 提交裁决 (人类 / 权威 Agent)
//     → 裁决结果: 保留 A / 保留 B / 合并 / 创建新版本
```

<!-- @end-section -->

<!-- @section: retrieval -->
## 四、知识检索 — 三路混合检索

### 4.1 检索架构

```
用户 / Agent 查询
  │
  ├── ① 向量语义检索 (pgvector)
  │      embedding ← text-embedding-3-large(query)
  │      SELECT * FROM entries
  │      ORDER BY embedding <=> query_embedding
  │      LIMIT 20
  │
  ├── ② 关键词全文检索 (PostgreSQL FTS)
  │      使用中文分词 (zhparser/jieba)
  │      SELECT * FROM entries
  │      WHERE to_tsvector('chinese', content) @@ to_tsquery('chinese', query)
  │      LIMIT 20
  │
  ├── ③ 知识图谱遍历 (Cypher / 图查询)
  │      MATCH (e:Entry)-[:RELATES_TO*1..3]-(related)
  │      WHERE e.id IN (matched_ids)
  │      RETURN related
  │
  └── 混合排序 (Fusion)
       └── Reciprocal Rank Fusion (RRF) 融合三路结果
           └── 最终 Top-K
```

### 4.2 混合排序算法

```go
func HybridSearch(ctx context.Context, query string, k int) ([]SearchResult, error) {
    // 并行三路检索
    var (
        semanticResults []SearchResult
        keywordResults  []SearchResult
        graphResults    []SearchResult
    )

    // 并行执行
    g := errgroup.Group{}
    g.Go(func() error { semanticResults = semanticSearch(query, 20); return nil })
    g.Go(func() error { keywordResults = keywordSearch(query, 20); return nil })
    g.Go(func() error { graphResults = graphTraverse(semanticResults, 2); return nil })
    g.Wait()

    // Reciprocal Rank Fusion
    fused := rrf([]ResultSet{
        {Results: semanticResults, Weight: 0.5},
        {Results: keywordResults, Weight: 0.3},
        {Results: graphResults, Weight: 0.2},
    }, k)

    // 重排序 (可选的 LLM 重排序)
    if len(fused) > k {
        fused = llmRerank(query, fused, k)
    }

    return fused, nil
}
```

### 4.3 差异化检索 (人类 vs Agent)

| 使用者 | 检索偏向 | 返回内容 |
|--------|---------|---------|
| 人类用户 | 可读性优先，支持图谱可视化 | 结构化文档 + 关系图 + 自然语言摘要 |
| Agent | 精确性优先，最小化上下文占用 | 结构化片段 + 引用溯源 + 策略步骤 |
| 工作流节点 | 上下文相关性优先 | 自动附加到 Agent 认知内核 |

<!-- @end-section -->

<!-- @section: governance -->
## 五、知识治理 — 质量保证与审核流程

### 5.1 知识生命周期

```
┌─────────────────────────────────────────────┐
│              知识生命周期                      │
│                                              │
│  创建 → 审核 → 发布 → 活跃 → 过期标记 → 归档   │
│    │      │      │      │         │        │
│    │      └──────┼──────┼─────────┼────────┤
│    │    驳回     │      │         │        │
│    └────────────┘      │         │        │
│                        │         │        │
│                  更新 ──┘   质量下降│        │
│                        │         │        │
│                   版本回滚 ←── 发现错误│     │
│                                              │
│  AI 创建: 默认进入"待审核"状态                 │
│  人类创建: 可选直接发布或待审核                 │
│  高权限 Agent: 可审核低权限 Agent 的产出       │
└─────────────────────────────────────────────┘
```

### 5.2 审核流程

```go
type ReviewWorkflow struct {
    Steps []ReviewStep
}

type ReviewStep struct {
    Type     ReviewType   // human | agent | automatic
    Reviewer string       // 审核者 ID
    Criteria []string     // 审核标准
    Timeout  time.Duration
}

// AI 知识的标准审核流程:
// Step 1: 自动检查 (安全扫描 + 格式校验) — 即时
// Step 2: 低权限 Agent 交叉验证 — < 5 分钟
// Step 3: 人类抽检 (随机 10%) — < 24 小时
// Step 4: 发布后 30 天: 质量复评 (基于引用/反馈)
```

### 5.3 版本管理

每次知识变更生成新版本，而非原地修改：

```go
type EntryVersion struct {
    EntryID     string
    Version     int
    Content     ContentBlock
    ContentHash string       // SHA-256
    DiffFromPrev string      // 与上一版本的差异
    ChangeReason string      // 变更原因
    ChangedBy   Author
    CreatedAt   time.Time
}

// 版本操作:
// - 查看: 任意历史版本
// - 对比: 任意两个版本的差异
// - 回滚: 将指定版本设为当前版本 (保留回滚记录)
// - 分支: 创建知识分支 (实验性修改)
// - 合并: 分支合并回主干
```

<!-- @end-section -->

<!-- @section: wiki-for-agent -->
## 六、Wiki 如何服务于 Agent

### 6.1 Agent 读取 Wiki

```
Agent 心跳 / 任务开始时:
  1. 认知内核向 Wiki 发起知识请求
     请求包含: 任务描述、角色、当前上下文、需要什么类型的知识
  2. Wiki 混合检索 → 返回最相关的知识条目
  3. 知识以结构化片段的形式注入到认知内核的"相关知识"区块
  4. Agent 引用 Wiki 知识时带有来源引用 (source: wiki:{entry_id}:v{version})

Agent 遇到未知领域:
  1. Agent 主动调用 wiki_search 工具
  2. Wiki 返回相关知识 + 学习路径推荐
  3. Agent 吸收知识后继续执行任务
```

### 6.2 Agent 写入 Wiki

```
任务完成后:
  1. 情景记忆提取 → 生成 Capsule
  2. Capsule 中的关键经验 → 提炼为知识条目草稿
  3. 知识条目提交到 Wiki → 进入审核流程
  4. 审核通过 → 正式入库 → 其他 Agent 可检索使用

定期蒸馏:
  1. 学习引擎定期聚类多条 Capsule
  2. 提取共性模式 → 生成 Gene (策略模板)
  3. Gene 存入 Wiki → 所有 Agent 可见
  4. A/B 测试验证 → 高效果 Gene 提升权重
```

### 6.3 Wiki 作为 Agent 的"集体大脑"

```
             ┌─ Agent A: 写入经验 ──┐
             │                      │
             ▼                      ▼
         ┌─────────────────────────────┐
         │        LLM Wiki              │
         │                              │
         │  策略模板 (Gene)              │
         │  成功案例 (Capsule)           │
         │  最佳实践                     │
         │  领域知识                     │
         │  FAQ                        │
         │  决策记录                     │
         │                              │
         └─────────────────────────────┘
             │                      │
             ▼                      ▼
         Agent B: 读取学习    Agent C: 读取应用
```

<!-- @end-section -->

<!-- @section: design-decisions -->
## 七、关键设计决策

| 决策 | 选择 | 理由 |
|------|------|------|
| 数据模型 | 结构化知识条目 | 比 Markdown 文件更可查询 |
| 存储引擎 | PostgreSQL + pgvector | 结构化 + 向量 + 全文 |
| 检索策略 | 三路混合检索 (向量+关键词+图谱) | 覆盖语义/精确/关联三维 |
| 知识关系 | 显式关系图谱 | 比文件系统的隐含关系更可计算 |
| 版本控制 | 数据库级版本管理 | 类 Git 体验 + 数据库性能 |
| 矛盾处理 | 冲突检测 + 裁决机制 | 保证知识库一致性 |
| AI 产出 | 全量审核流程 | 三原则"可治理"落地 |
| 人类角色 | 最终裁决者 | 三原则"可治理"落地 |

### 与 hermes 技能系统的关键差异

| 维度 | hermes Skills | Legion Wiki |
|------|-------------|------------|
| 存储 | 文件系统 (SKILL.md) | 数据库 (知识条目) |
| 检索 | 名称/标签查找 | 混合检索 (向量+FTS+图谱) |
| 更新 | 用户手动安装新版本 | Agent 自动建议更新 + 人类审核 |
| 关联 | 无 (扁平列表) | 知识图谱显式关系 |
| 创作 | 仅人类 | 人 + Agent 共建 |
| 治理 | 安全扫描 | 安全 + 审核 + 版本 + 质量评分 |

### 不做什么

1. **不做纯文件系统** — hermes 的 SKILL.md 规模上限已验证
2. **不做仅人类可写** — 人机共建是核心定位
3. **不做无版本控制** — 知识变更必须可追溯、可回滚
4. **不做无审核的 Agent 写入** — 三原则要求 AI 产出必须审核

<!-- @end-section -->

<!-- @section: related -->
## 相关文档

- [[../analysis/hermes/03-tools-skills-plugins|hermes 技能系统分析]]
- [[../analysis/evolver/02-gep-protocol|Evolver GEP 资产模型分析]]
- [[../architecture/Legion|Legion 项目方案 — LLM Wiki 引擎]]
- [[02-agent-runtime-deep-design|Agent 运行时深度设计]]
- [[03-evolution-learning-deep-design|进化学习系统深度设计]]
- [[07-architecture-integration|系统集成架构深度设计]]

<!-- @end-section -->
