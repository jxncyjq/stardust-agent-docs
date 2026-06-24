---
id: "architecture-agent-engine-design-001"
title: "Legion Agent 引擎设计方案：五项目精华融合"
aliases: ["agent engine design", "Legion引擎设计", "取长补短设计"]
type: "architecture"
category: "design/architecture"
tags: ["legion", "agent-engine", "maas", "llm-wiki", "adapter", "design", "architecture", "quality-gate", "trust-score", "eval"]
version: "2.0.0"
created: "2026-05-08"
updated: "2026-05-08"
author: "jxncyjq"
status: "draft"
related_docs:
  - id: "architecture-legion-001"
    relation: "parent"
    path: "./Legion.md"
  - id: "analysis-clawcode-overview-001"
    relation: "reference"
    path: "../analysis/claude/01-overview.md"
  - id: "analysis-deepseek-tui-overview-001"
    relation: "reference"
    path: "../analysis/deepseek-tui/01-overview.md"
  - id: "analysis-evolver-overview-001"
    relation: "reference"
    path: "../analysis/evolver/01-overview.md"
  - id: "analysis-hermes-overview-001"
    relation: "reference"
    path: "../analysis/hermes/01-overview.md"
  - id: "analysis-mission-control-index"
    relation: "reference"
    path: "../analysis/mission-control/index.md"
---

<!-- @section: overview -->
# Legion Agent 引擎设计方案：五项目精华融合

## 文档说明

本文档是 **Legion V3.0 架构**（[[Legion|Legion.md]]）的落地设计方案，以三原则（可观测·可治理·可控风险）为顶层约束，系统性地从五个参考项目中提取最优设计，映射到 Legion 的三大引擎（MaaS / Agent / LLM Wiki）与 Adapter 层。

### 五个参考项目对 Legion 的贡献定位

| 项目 | 语言 | 最大贡献领域 | 核心借鉴点 |
|------|------|------------|----------|
| **Claw Code** | Rust | 权限执行框架 | permission_enforcer + bootstrap 12阶段 + LSP 集成 |
| **DeepSeek-TUI** | Rust | 生产级流式管道 | SSE 管道 + 审批策略引擎 + 循环检查点 + Crate 分层 |
| **Hermes Agent** | Python | Agent 运行时完整度 | NormalizedResponse + 错误分类学 + 上下文压缩 + 工具注册 |
| **Evolver** | JavaScript | 自我进化形式化 | GEP 三层信号 + Gene/Capsule/Event 资产 + 固化流程 |
| **Mission Control** | TypeScript | Agent 舰队编排控制 | Aegis 质量门控 + Trust Score + 四层 Eval + 调度器架构 + 技能安全扫描 |

### 设计方法论

Legion.md 的核心思想是：**不让 AI 工具运作，而让 AI 团队运作**。这要求每一个组件设计都必须同时满足：

- `[可观测]`：每个决策、每次调用、每条记忆都可追溯
- `[可治理]`：人类在任何时刻都能介入、审核、纠偏
- `[可控风险]`：自主行为始终在预设安全边界内

<!-- @end-section -->

---

<!-- @section: maas-design -->
## 一、MaaS 模型调度层设计

### 1.1 设计目标回顾（来自 Legion.md §3.1）

MaaS 层的职责是将"模型选择"从 Agent 定义中解耦，由平台统一调度。关键需求：
- 按角色/任务/预算三维路由策略
- 五级配额（公司→部门→团队→Agent→模型）
- 三级预算熔断（告警→降级→冻结）
- 完整的调用链路追踪

### 1.2 传输规范化层 —— 来自 Hermes

Hermes 最优雅的设计是 `ProviderTransport` ABC + `NormalizedResponse`：

```
Agent 循环 ← NormalizedResponse ← ProviderTransport ← [OpenAI|Anthropic|DeepSeek|...]
```

**Legion 采纳方案**：定义 `ModelProvider` trait（Rust）/ 接口（Go），将所有 LLM API 差异在传输层消除：

```rust
// 直接参考 Hermes 的 NormalizedResponse
pub trait ModelProvider: Send + Sync {
    async fn call(&self, req: ModelRequest) -> Result<ModelResponse, LlmError>;
    async fn stream(&self, req: ModelRequest) -> Result<StreamHandle, LlmError>;
    fn capabilities(&self) -> ProviderCapabilities;
    fn health(&self) -> ConnectionHealth;
}

// 统一响应结构 — 消除提供商差异
pub struct ModelResponse {
    pub content: Vec<ContentBlock>,     // 统一内容块
    pub tool_calls: Vec<ToolCall>,      // 统一工具调用格式
    pub usage: TokenUsage,              // 统一用量统计
    pub model_used: String,             // 实际使用的模型（路由后）
    pub latency_ms: u64,                // 响应延迟
    pub provider_id: String,            // 路由溯源
}
```

`[可观测]` `model_used` + `provider_id` 字段确保每次调用都记录了实际路由决策，而非计划路由。

### 1.3 SSE 流式管道 —— 来自 DeepSeek-TUI

DeepSeek-TUI 的 SSE 管道是目前分析的四个项目中最完整的生产实现：

```
字节流 → 缓冲(8MB限) → 行解析 → JSON解析 → 类型安全事件
   ↑            ↑          ↑          ↑
 背压控制    分批处理    CRLF兼容   状态追踪
(高水位警戒) (256行/批)  处理      (content_index)
```

**Legion 采纳方案**：SSE 解析器封装为状态对象（修正了 DeepSeek-TUI 的函数签名问题）：

```rust
// 修正 DeepSeek-TUI parse_sse_chunk 的 7 个 &mut 参数问题
pub struct SseParser {
    content_index: usize,
    text_started: bool,
    thinking_started: bool,
    tool_indices: HashMap<String, usize>,
    high_watermark: usize,          // 8MB 高水位
    batch_size: usize,              // 256行/批
}

impl SseParser {
    pub fn parse(&mut self, chunk: &Value) -> Vec<StreamEvent> { ... }
    pub fn is_above_watermark(&self) -> bool { ... }
}
```

**关键机制保留**：
- 高水位背压控制（8MB → 暂停消费）
- 分批行处理（256行/批，防止 UI 卡顿）
- 空闲超时（5分钟无数据 → 断开重连）
- 流停滞透明重试（最多 3 次，用户无感知）

### 1.4 错误分类与自动恢复 —— 来自 Hermes

Hermes 的 `FailoverReason` 枚举 + 恢复处理是最完整的错误分类实践：

```rust
// 参考 Hermes error_classifier.py 的分类学
pub enum LlmError {
    // 可重试
    RateLimit { retry_after_secs: u64 },
    Timeout { elapsed_ms: u64 },
    ServiceUnavailable,

    // 触发上下文压缩
    ContextWindowExceeded { current_tokens: u64, max_tokens: u64 },

    // 触发凭证轮换
    AuthFailed { hint: String },
    QuotaExceeded { scope: QuotaScope },

    // 触发故障转移
    ModelNotAvailable { model_id: String },
    ProviderError { code: u16, message: String },

    // 不可恢复
    InvalidRequest { message: String },
    ContentFiltered { reason: String },
}

// 错误处理决策树 — 参考 Hermes 的 FailoverReason 模式
impl LlmError {
    pub fn recovery_action(&self) -> RecoveryAction {
        match self {
            LlmError::RateLimit { .. } => RecoveryAction::RetryWithBackoff,
            LlmError::ContextWindowExceeded { .. } => RecoveryAction::CompressContext,
            LlmError::AuthFailed { .. } => RecoveryAction::RotateCredential,
            LlmError::ModelNotAvailable { .. } => RecoveryAction::Fallback,
            _ => RecoveryAction::Fail,
        }
    }
}
```

`[可治理]` 错误分类后的恢复动作（降级到哪个模型、如何压缩）均可在配置中调整，管理员可以覆盖默认行为。

### 1.5 连接健康状态机 —— 来自 DeepSeek-TUI

DeepSeek-TUI 的三态健康状态机直接对应 Legion.md 的三级熔断需求：

```
Healthy (正常)
   │ (连续 2 次失败)
   ▼
Degraded (降级)
   │ (15秒后，发送探针)
   ▼
Recovering (恢复中)
   ├── (探针成功) → Healthy
   └── (探针失败) → Degraded (重计时器)
```

**与 Legion 配额熔断的映射**：

| DeepSeek-TUI 状态 | Legion 配额语义 | 触发条件 |
|-----------------|--------------|---------|
| Healthy | 正常路由 | 默认状态 |
| Degraded | 预算告警 → 强制降级 | budget_remaining < 20% 或连续2次失败 |
| Recovering | 探针验证 | 等待 15s 后发送轻量探针请求 |
| (扩展) Frozen | 预算冻结 | budget_remaining < 0 |

`[可控风险]` 状态机确保模型不可用时自动降级而非中断业务，同时给予自动恢复机会。

### 1.6 速率限制 —— 来自 DeepSeek-TUI

DeepSeek-TUI 的 `TokenBucket` 令牌桶直接用于 Legion 的每提供商限速：

```rust
// 直接参考 DeepSeek-TUI 的 TokenBucket 实现
pub struct TokenBucket {
    capacity: f64,          // 桶容量
    tokens: f64,            // 当前令牌数
    refill_rate: f64,       // 补充速率 (tokens/second)
    last_refill: Instant,
}

// Legion 扩展：多维度令牌桶
pub struct MultiDimensionalRateLimit {
    per_provider: HashMap<String, TokenBucket>,  // 按提供商
    per_agent: HashMap<String, TokenBucket>,      // 按 Agent
    per_company: HashMap<String, TokenBucket>,    // 按公司 (配额管控)
}
```

### 1.7 智能路由引擎实现

结合 Legion.md 的三维路由策略（角色/任务/预算）与四个项目的实现经验：

```rust
pub struct IntelligentRouter {
    model_registry: Arc<ModelRegistry>,
    rate_limiter: Arc<MultiDimensionalRateLimit>,
    health_monitor: Arc<HealthMonitor>,
    routing_config: RoutingConfig,
}

impl IntelligentRouter {
    /// 统一路由入口 — 所有路由调用都走此方法
    ///
    /// - 有角色上下文（正常 Agent 任务）→ 委托 route_with_role()（§1.8.5，10步完整路径）
    /// - 无角色上下文（系统任务、预热请求）→ 简化路径（Standard Tier + 健康过滤）
    pub async fn route(&self, ctx: &RoutingContext) -> Result<RoutingDecision, RouterError> {
        match (&ctx.agent, &ctx.task) {
            (Some(agent), Some(task)) => {
                // 有角色上下文 → 完整角色感知路由（见 §1.8.5）
                self.route_with_role(agent, task, &ctx.budget).await
            }
            _ => {
                // 无角色上下文 → 简化路径：Standard + 健康 + 速率限制
                let candidates = self.model_registry.find(ModelTier::Standard, &HashSet::new());
                let healthy = candidates.into_iter()
                    .filter(|m| self.health_monitor.is_healthy(&m.provider_id))
                    .collect::<Vec<_>>();
                let available = self.check_rate_limits(&healthy, ctx.agent_id.as_deref()).await?;
                let selected = self.select_optimal(&available, &ctx.optimization_goal)?;
                Ok(RoutingDecision {
                    selected_model: selected,
                    decision_trace: RoutingTrace::system_path(),
                })
            }
        }
    }
}
```

`[可观测]` 每一步路由决策记录到追踪日志：候选集合、过滤原因、最终选择、选择依据。角色感知路径的完整 10 步追踪见 §1.8.5 `RoutingTrace`。

---

### 1.8 角色-模型路由矩阵 —— Legion 核心差异化

#### 1.8.1 设计背景

Legion.md §3.1.1 明确指出："CEO Agent 做战略决策需要最强推理模型，客服 Agent 做日常回复用轻量模型即可。" 这不是一句话的设计原则——它要求 MaaS 层为 Legion.md §4.2.1 中定义的每一类角色，都建立精细的**模型路由档案**。

四个参考项目都回避了这个问题的深度：
- Hermes 是"静态选择 + 故障转移"，没有角色感知
- DeepSeek-TUI 是手动配置单个模型，不支持角色路由
- Claw Code 是用户手动切换模型
- Evolver 不涉及多模型

Legion 在此做出真正的差异化：**每个角色的每类任务，都有精确的模型分配策略**。

#### 1.8.2 模型等级定义

在路由策略中使用**能力等级（Tier）**而非具体模型名称，解耦路由逻辑与具体模型版本：

```rust
pub enum ModelTier {
    /// 轻量级 — 低延迟、低成本
    /// 适用：格式转换、简单分类、模板填充、简短回复
    /// 参考：deepseek-chat / qwen-turbo / gpt-4o-mini
    Light,

    /// 标准级 — 均衡能力
    /// 适用：内容创作、代码实现、数据分析、日常推理
    /// 参考：claude-sonnet-4 / deepseek-coder / gpt-4o
    Standard,

    /// 高级 — 强推理能力
    /// 适用：复杂架构设计、深度分析、跨文档综合判断
    /// 参考：claude-sonnet-4-extended / gpt-4o-reasoner
    Advanced,

    /// 旗舰级 — 最强推理，成本最高
    /// 适用：战略决策、创新性问题、高风险评审
    /// 参考：claude-opus-4 / gpt-4o / deepseek-reasoner
    Flagship,
}

pub struct TierSpec {
    pub tier: ModelTier,
    pub min_context_window: u32,      // 最小上下文窗口要求
    pub require_tool_use: bool,       // 是否需要工具调用能力
    pub require_structured_output: bool,
    pub latency_budget_ms: Option<u64>, // 延迟预算（客服场景严格）
}
```

#### 1.8.3 角色路由档案

Legion.md §4.2.1 定义了 6 类 Agent 角色，每类角色的模型需求差异显著：

```yaml
# 角色路由档案 — 完整配置示例
role_routing_profiles:

  # ── 管理层 ──────────────────────────────────────────────
  ceo:
    description: "战略规划、目标制定、重大决策"
    default_tier: flagship
    task_overrides:
      status_review: standard          # 看日报用标准级即可
      strategic_planning: flagship     # 战略规划用旗舰级
      resource_allocation: advanced    # 资源分配用高级
      routine_approval: standard       # 常规审批用标准级
    fallback_chain: [flagship, advanced, standard]
    cost_sensitivity: low              # 对成本不敏感

  cto:
    description: "技术决策、架构评审、团队管理"
    default_tier: advanced
    task_overrides:
      architecture_review: flagship    # 架构评审需要旗舰
      code_review: advanced
      tech_roadmap: flagship
      daily_standup: standard
    fallback_chain: [advanced, standard, light]
    cost_sensitivity: medium

  project_manager:
    description: "任务拆解、进度跟踪、风险管理"
    default_tier: standard
    task_overrides:
      risk_assessment: advanced        # 风险评估需要高级
      task_decomposition: standard
      progress_report: light           # 进度报告用轻量
      stakeholder_communication: standard
    fallback_chain: [standard, light]
    cost_sensitivity: medium

  # ── 研发类 ──────────────────────────────────────────────
  architect:
    description: "系统架构设计、技术选型、性能优化"
    default_tier: flagship
    task_overrides:
      system_design: flagship
      performance_analysis: advanced
      code_generation: advanced        # 架构师写代码用高级
      documentation: standard
    fallback_chain: [flagship, advanced, standard]
    cost_sensitivity: low
    special_capabilities:
      - code                           # 必须支持代码能力
      - structured_output

  backend_developer:
    description: "后端服务开发、API 设计、数据库"
    default_tier: standard
    task_overrides:
      complex_algorithm: advanced      # 复杂算法用高级
      routine_crud: light              # CRUD 代码用轻量
      bug_fix: standard
      code_review: advanced
      unit_test_generation: light      # 生成测试用轻量
      api_design: standard
    fallback_chain: [standard, light]
    cost_sensitivity: high
    special_capabilities: [code]

  frontend_developer:
    description: "前端界面开发、组件实现、性能优化"
    default_tier: standard
    task_overrides:
      complex_ui_logic: advanced
      component_generation: light      # 生成简单组件用轻量
      css_styling: light
      performance_debugging: advanced
    fallback_chain: [standard, light]
    cost_sensitivity: high
    special_capabilities: [code]

  devops:
    description: "基础设施、CI/CD、监控告警"
    default_tier: standard
    task_overrides:
      incident_analysis: advanced      # 故障分析需要高级
      pipeline_setup: standard
      routine_deployment: light
      security_audit: advanced
    fallback_chain: [standard, light]
    cost_sensitivity: high

  # ── 产品类 ──────────────────────────────────────────────
  product_manager:
    description: "需求分析、产品规划、用户调研"
    default_tier: standard
    task_overrides:
      market_research: advanced        # 市场调研需要深度分析
      requirement_analysis: standard
      prd_writing: standard
      competitive_analysis: advanced
      wireframe_description: light
    fallback_chain: [standard, light]
    cost_sensitivity: medium

  ui_designer:
    description: "视觉设计、交互设计、设计系统"
    default_tier: standard
    task_overrides:
      design_concept: advanced         # 创意概念需要高级
      design_review: standard
      asset_description: light
    fallback_chain: [standard, light]
    cost_sensitivity: medium
    special_capabilities: [vision]     # 需要视觉能力（图片理解）

  ux_researcher:
    description: "用户调研、数据分析、洞察提炼"
    default_tier: standard
    task_overrides:
      user_insight_synthesis: advanced
      survey_design: standard
      report_writing: standard
      data_coding: light
    fallback_chain: [standard, light]
    cost_sensitivity: medium

  # ── 运营类 ──────────────────────────────────────────────
  content_editor:
    description: "内容创作、文章撰写、内容审核"
    default_tier: standard
    task_overrides:
      long_form_article: advanced      # 长篇文章需要高级
      short_copy: light
      content_review: light
      seo_optimization: light
      headline_generation: light
    fallback_chain: [standard, light]
    cost_sensitivity: high             # 内容团队对成本敏感

  customer_service:
    description: "用户咨询、投诉处理、满意度维护"
    default_tier: light                # 客服默认用轻量级
    task_overrides:
      complex_complaint: standard      # 复杂投诉升级
      routine_faq: light
      refund_decision: standard        # 退款决策需要标准
      escalation: advanced             # 升级案例用高级
    fallback_chain: [light, standard]
    cost_sensitivity: very_high        # 客服对延迟和成本极敏感
    latency_budget_ms: 2000            # 严格的延迟预算

  social_media_operator:
    description: "社交媒体运营、粉丝互动、内容发布"
    default_tier: light
    task_overrides:
      campaign_strategy: standard
      post_generation: light
      comment_reply: light
      crisis_response: advanced        # 危机公关升级
    fallback_chain: [light, standard]
    cost_sensitivity: very_high

  # ── 质量类 ──────────────────────────────────────────────
  qa_engineer:
    description: "测试执行、Bug 管理、质量保证"
    default_tier: standard
    task_overrides:
      test_case_generation: light      # 生成测试用例用轻量
      bug_analysis: standard
      security_testing: advanced
      regression_test: light
      test_report: standard
    fallback_chain: [standard, light]
    cost_sensitivity: high
    special_capabilities: [code]

  security_auditor:
    description: "安全审计、漏洞检测、合规检查"
    default_tier: advanced             # 安全默认高级
    task_overrides:
      vulnerability_analysis: flagship
      compliance_check: advanced
      routine_scan: standard
      penetration_test: flagship
    fallback_chain: [advanced, standard]
    cost_sensitivity: low              # 安全不计成本

  # ── 数据类 ──────────────────────────────────────────────
  data_analyst:
    description: "数据清洗、报表生成、趋势分析"
    default_tier: standard
    task_overrides:
      complex_statistical_analysis: advanced
      dashboard_generation: light
      data_cleaning: light
      insight_report: standard
      anomaly_detection: advanced
    fallback_chain: [standard, light]
    cost_sensitivity: medium
    special_capabilities: [code, structured_output]

  bi_engineer:
    description: "数据仓库、ETL 流程、BI 系统"
    default_tier: standard
    task_overrides:
      data_modeling: advanced
      etl_pipeline: standard
      query_optimization: advanced
      routine_reporting: light
    fallback_chain: [standard, light]
    cost_sensitivity: high
    special_capabilities: [code]
```

#### 1.8.4 任务复杂度动态评估

同一角色执行相同类型的任务，复杂度不同时应动态调整模型选择：

```rust
pub struct TaskComplexityScorer;

impl TaskComplexityScorer {
    pub fn score(&self, task: &Task, agent: &AgentDef) -> ComplexityScore {
        let mut score = 0u32;

        // 维度1：输入规模
        score += match task.input_tokens_estimate {
            0..=1_000   => 0,
            1_001..=8_000  => 1,
            8_001..=32_000 => 2,
            _              => 3,
        };

        // 维度2：工具调用深度（需要多少轮工具调用）
        score += match task.expected_tool_rounds {
            0..=1  => 0,
            2..=5  => 1,
            6..=15 => 2,
            _      => 3,
        };

        // 维度3：跨文档综合（需要阅读多少个上下文文档）
        score += match task.context_doc_count {
            0..=2  => 0,
            3..=10 => 1,
            _      => 2,
        };

        // 维度4：决策风险等级（来自任务元数据）
        score += match task.risk_level {
            RiskLevel::Low    => 0,
            RiskLevel::Medium => 1,
            RiskLevel::High   => 2,
            RiskLevel::Critical => 3,
        };

        // 维度5：需要创意/开放性推理
        score += if task.requires_creative_reasoning { 2 } else { 0 };

        ComplexityScore {
            total: score,
            tier_recommendation: match score {
                0..=2  => ModelTier::Light,
                3..=5  => ModelTier::Standard,
                6..=8  => ModelTier::Advanced,
                _      => ModelTier::Flagship,
            }
        }
    }
}
```

#### 1.8.5 完整路由决策流水线

将角色档案、任务复杂度、预算约束、健康状态全部融合为一条决策流水线：

```rust
impl IntelligentRouter {
    pub async fn route_with_role(
        &self,
        agent: &AgentDef,
        task: &Task,
        budget_ctx: &BudgetContext,
    ) -> Result<RoutingDecision, RouterError> {

        // ── 步骤 1: 确定基础 Tier（角色档案 × 任务类型）────────────
        let role_profile = self.load_role_profile(&agent.role);
        let base_tier = role_profile
            .task_overrides
            .get(&task.task_type)
            .cloned()
            .unwrap_or(role_profile.default_tier);

        // ── 步骤 2: 任务复杂度动态修正 ────────────────────────────
        let complexity = self.complexity_scorer.score(task, agent);
        let complexity_tier = complexity.tier_recommendation;

        // 取两者的较高 Tier（角色优先，但复杂度可以向上推升）
        let required_tier = base_tier.max(complexity_tier);

        // ── 步骤 3: Agent 定义中的显式约束 ───────────────────────
        // Agent YAML 中 model_policy.max_model_tier 是硬上限
        let capped_tier = required_tier.min(agent.model_policy.max_model_tier);

        // ── 步骤 4: 预算感知降级（Legion.md §3.1.3 budget_aware）──
        let budget_tier = self.apply_budget_pressure(capped_tier, budget_ctx);

        // ── 步骤 5: 筛选满足能力要求的候选模型 ────────────────────
        let special_caps = role_profile.special_capabilities
            .iter()
            .chain(task.required_capabilities.iter())
            .collect::<HashSet<_>>();
        let candidates = self.model_registry.find(budget_tier, &special_caps);

        // ── 步骤 6: 延迟预算过滤（客服类严格限制）─────────────────
        let latency_candidates = if let Some(budget_ms) = role_profile.latency_budget_ms {
            candidates.into_iter()
                .filter(|m| m.p95_latency_ms <= budget_ms)
                .collect()
        } else { candidates };

        // ── 步骤 7: 健康检查过滤 ──────────────────────────────────
        let healthy_candidates = latency_candidates.into_iter()
            .filter(|m| self.health_monitor.is_healthy(&m.provider_id))
            .collect::<Vec<_>>();

        // ── 步骤 8: 没有健康候选 → 按 fallback_chain 逐级降级 ─────
        let final_candidates = if healthy_candidates.is_empty() {
            self.fallback_to_chain(&role_profile.fallback_chain, &special_caps).await?
        } else { healthy_candidates };

        // ── 步骤 9: 成本敏感度决定最终选择策略 ────────────────────
        let selected = match role_profile.cost_sensitivity {
            CostSensitivity::VeryHigh => self.select_cheapest(&final_candidates),
            CostSensitivity::High     => self.select_cheapest_in_tier(&final_candidates),
            CostSensitivity::Medium   => self.select_balanced(&final_candidates),
            CostSensitivity::Low      => self.select_highest_quality(&final_candidates),
        };

        // ── 步骤 10: 构建可观测的路由决策记录 ────────────────────
        Ok(RoutingDecision {
            selected_model: selected,
            decision_trace: RoutingTrace {
                agent_id: agent.id.clone(),
                agent_role: agent.role.clone(),
                task_type: task.task_type.clone(),
                base_tier_from_role: base_tier,
                complexity_score: complexity.total,
                complexity_tier,
                required_tier,
                capped_by_agent_policy: required_tier != capped_tier,
                budget_degraded: capped_tier != budget_tier,
                fallback_used: false,
                candidates_count: final_candidates.len(),
                selection_reason: format!(
                    "角色={}, 任务类型={}, 复杂度={}, 最终Tier={:?}, 成本敏感度={:?}",
                    agent.role, task.task_type, complexity.total,
                    budget_tier, role_profile.cost_sensitivity
                ),
            },
        })
    }
}
```

`[可观测]` `RoutingTrace` 记录了路由决策的每一个中间状态——基础 Tier 来自哪里、是否被复杂度推升、是否被预算压降、是否触发了 fallback——管理者可在仪表盘中精确查看每次调用为何使用了该模型。

`[可治理]` 三处硬约束均可被管理员覆盖：
1. `model_policy.max_model_tier` — Agent 级上限
2. `routing_config.role_overrides` — 临时覆盖某角色的路由策略
3. `routing_config.force_bind[agent_id]` — 强制某 Agent 使用指定模型

#### 1.8.6 工作流中的多 Agent 模型分工

在 Legion 的工作流场景下，一个流程中多个不同角色的 Agent 协作时，各自的模型选择策略形成互补的**模型分工格局**：

```
需求到上线全流程 — 模型分工示意

┌─────────────────────────────────────────────────────────────────┐
│  工作流节点          Agent 角色          路由 Tier    模型示例     │
├─────────────────────────────────────────────────────────────────┤
│  收集需求            product_manager     Standard    sonnet-4    │
│  ─────────────────── ✋ 高影响力需求审批门控 ───────────────────  │
│  技术方案设计        architect           Flagship    opus-4      │ ← 最贵
│  UI 原型设计         ui_designer         Standard    sonnet-4    │
│  ─────────────────── ⊕ 并行汇聚等待 ─────────────────────────── │
│  后端开发            backend_developer   Standard    deepseek    │
│    - 复杂算法模块                        Advanced    sonnet-ext  │ ← 动态升级
│    - CRUD 接口                           Light       qwen-turbo  │ ← 动态降级
│  前端开发            frontend_developer  Standard    sonnet-4    │
│  ─────────────────── ✋ 预算门控 ─────────────────────────────── │
│  测试执行            qa_engineer         Light       qwen-turbo  │ ← 批量测试轻量
│    - 安全测试                            Advanced    sonnet-ext  │ ← 安全升级
│  部署上线            devops              Standard    deepseek    │
└─────────────────────────────────────────────────────────────────┘

整个流程：旗舰级(1次) + 高级(3次) + 标准级(n次) + 轻量级(m次)
成本优化效果：相比全程使用旗舰级，预计节省 60-75% 的模型成本
```

#### 1.8.7 角色路由的运行时覆盖接口

`[可治理]` 管理员可以在不修改配置文件的情况下，通过管理 API 实时调整路由策略：

```rust
// 路由策略运行时覆盖（管理员操作接口）
pub enum RoutingOverride {
    // 强制某 Agent 始终使用指定模型（调试或特殊场景）
    ForceModel {
        agent_id: AgentId,
        model_id: String,
        expires_at: Option<DateTime<Utc>>,
        reason: String,         // 必填，写入审计日志
    },

    // 临时调整某角色的默认 Tier（如预算紧张期全员降级）
    AdjustRoleTier {
        role: AgentRole,
        tier_delta: i32,        // -1 降一级, +1 升一级
        scope: OverrideScope,   // 覆盖范围（公司级 / 部门级 / 团队级）
        expires_at: DateTime<Utc>,
        reason: String,
    },

    // 禁止使用某个模型（如模型下线或安全事件）
    BanModel {
        model_id: String,
        fallback_model: String,
        reason: String,
    },
}

/// 路由覆盖生效范围（对应五级组织层次中的上三层）
/// Agent 级和模型级覆盖通过 ForceModel / BanModel 实现，此处只需上层范围
pub enum OverrideScope {
    /// 公司全局生效（影响所有部门、团队、Agent）
    Company { company_id: String },
    /// 部门范围生效（影响该部门下所有团队和 Agent）
    Department { department_id: String },
    /// 团队范围生效（影响该团队下所有 Agent）
    Team { team_id: String },
}
```

`[可控风险]` 所有运行时覆盖操作写入不可篡改的审计日志，包含操作者身份、理由、生效时间范围，保证可追溯且防止滥用。

<!-- @end-section -->

---

<!-- @section: agent-engine-design -->
## 二、AI Agent 引擎设计

### 2.1 设计目标回顾（来自 Legion.md §3.2）

Legion 的 Agent 引擎的核心差异化在于**自然进化**：Agent 在工作中积累经验、从反馈中修正行为、越用越强。设计关键：
- 认知内核：7 要素上下文组装
- 三层记忆：工作/情景/能力
- 四路学习：反馈/反思/协作观察/知识库驱动
- 进化评估器：5 维度量化追踪

### 2.2 Crate 分层架构 —— 来自 DeepSeek-TUI

DeepSeek-TUI 的 Cargo 工作区分层是整个分析中**最重要的架构决策**，Legion 直接采纳：

```
legion-protocol      [Layer 0] 纯数据结构：Message/Tool/Event/Request/Response
legion-config        [Layer 0] 配置加载：TOML + 环境变量 + 优先级合并
legion-model         [Layer 0] 数据库模型：SQLite schema + 查询
legion-state         [Layer 0] 运行时状态：Agent状态机 + 任务状态

legion-tools         [Layer 1] 工具原始类型：ToolSpec + ToolInvocation
legion-skills        [Layer 1] 技能系统：可插拔技能 + 运行时注入
legion-execpolicy    [Layer 1] 执行策略：ApprovalPolicy + SandboxMode
legion-hooks         [Layer 1] 生命周期钩子：pre-turn/post-turn
legion-mcp           [Layer 1] MCP 客户端：stdio/HTTP 协议

legion-memory        [Layer 2] 记忆系统：工作/情景/能力三层
legion-learning      [Layer 2] 学习引擎：GEP信号提取 + Gene资产
legion-evolution     [Layer 2] 进化评估：5维度指标 + 成长报告

legion-maas          [Layer 3] MaaS 调度：路由 + 配额 + 熔断
legion-core          [Layer 3] Agent 引擎：认知内核 + 执行循环

legion-wiki          [Layer 4] LLM Wiki：知识管道 + 混合检索
legion-gateway       [Layer 4] HTTP API：REST + WebSocket + SSE

legion-app           [Layer 5] Web 应用：React 前端服务
legion-cli           [Layer 5] CLI 工具：开发者调试命令
```

**关键约束**：每层只能依赖下层或同层，绝对禁止循环依赖。`legion-protocol` 作为零依赖的纯数据层，所有其他 crate 共享其类型定义。

**同层单向依赖规则**：同一 Layer 内允许单向依赖，但反向禁止。唯一受认可的同层依赖：
- `legion-core`（Layer 3）→ `legion-maas`（Layer 3）：Agent 引擎通过 MaaS 层发起 LLM 调用
- 反向严格禁止：`legion-maas` 不得感知 `legion-core` 的存在，MaaS 层只提供路由/配额/熔断服务，不引用具体 Agent 类型

### 2.3 认知内核（Cognitive Core）—— Legion 原创 + 四项目融合

Legion.md §3.2.2 定义了认知内核的 7 要素公式：

```
执行上下文 = 角色人设
           + 当前任务指令
           + 目标传导链（为什么做）
           + 相关经验记忆（以前类似任务怎么做的）
           + 挂载技能（能用什么工具）
           + 组织上下文（我在团队中的位置、协作者是谁）
           + 约束规则（预算限制、权限边界、安全红线）
```

**设计关键**：这不是 Prompt 拼接，而是有优先级的上下文编排。借鉴四个项目：

```rust
pub struct CognitiveCore {
    // ── Agent 身份与能力（构造时注入，对话期间不可变）────────────
    /// Agent 定义：角色人设（P1）、约束规则（P0）、能力声明
    agent_def: Arc<AgentDef>,

    // ── 记忆与技能（通过 legion-memory / legion-skills 注入）────────
    /// 三层记忆系统（§2.8）：检索任务相关的情景记忆和能力记忆（P5）
    memory: Arc<MemorySystem>,
    /// 技能加载器（§4.5 / legion-skills）：运行时按任务注入可用技能（P6）
    skills: Arc<SkillSystem>,

    // ── 上下文压缩器（§2.6）────────────────────────────────────
    /// ContextCompressor 的所有权在此——cycle_count 由它内部维护和重置
    /// max_cycle_length 在构造 CognitiveCore 时传入 ContextCompressor::new(max_cycle_length)
    context_compressor: ContextCompressor,

    // ── 来自 Hermes：提示缓存稳定性原则 ─────────────────────────
    // "对话中期永不修改系统提示/工具集，避免破坏 Anthropic 提示缓存"
    // AtomicBool 允许通过 &self 修改（assemble_context 不需要 &mut self）
    system_prompt_locked: AtomicBool,
    system_prompt_hash: u64,

    // ── 来自 Claw Code：bootstrap 阶段的结构化初始化 ─────────────
    init_phase: InitPhase,
}

impl CognitiveCore {
    /// 组装 7 要素执行上下文（按优先级编排，P0 安全红线不可覆盖）
    /// 使用 &self（AtomicBool 允许内部可变地锁定系统提示，不需要 &mut self）
    pub fn assemble_context(&self, req: &TaskRequest) -> ExecutionContext {
        let mut ctx = ExecutionContext::new();

        // 优先级 P0: 约束规则（安全红线 — 永远最高优先级）
        ctx.push_safety_rules(&self.agent_def.constraints, Priority::P0_Safety);

        // 优先级 P1: 角色人设（对话期间锁定，参考 Hermes 提示缓存稳定性）
        // compare_exchange 保证仅第一次 assemble 时写入系统提示（对话期间幂等）
        if self.system_prompt_locked
            .compare_exchange(false, true, Ordering::SeqCst, Ordering::SeqCst)
            .is_ok()
        {
            ctx.push_persona(&self.agent_def.persona, Priority::P1_Persona);
        }

        // 优先级 P2: 组织上下文
        ctx.push_org_context(&req.org_context, Priority::P2_Organization);

        // 优先级 P3: 目标传导链（为什么做）
        ctx.push_goal_chain(&req.goal_chain, Priority::P3_Goals);

        // 优先级 P4: 当前任务指令
        ctx.push_task_instruction(&req.task, Priority::P4_Task);

        // 优先级 P5: 相关经验记忆（来自 legion-memory，按相关度检索）
        let memories = self.memory.retrieve_relevant(&req.task, TopK(5));
        ctx.push_memories(&memories, Priority::P5_Memory);

        // 优先级 P6: 挂载技能（来自 legion-skills，运行时注入）
        let skills = self.skills.load_for_task(&req.task);
        ctx.push_skills(&skills, Priority::P6_Skills);

        // 冲突解决：P0 的安全红线始终覆盖其他层
        ctx.resolve_conflicts_by_priority();

        ctx
    }

    /// SoftLoop 触发时调用：强制执行一次循环检查点（委托给 ContextCompressor）
    /// ctx 需要 &mut 是因为检查点会替换消息历史为摘要
    pub async fn force_cycle_checkpoint(
        &mut self,
        ctx: &mut ExecutionContext,
    ) -> Result<(), CompressError> {
        // 强制触发 CycleCheckpoint 分支（忽略 80% 阈值，立即执行）
        self.context_compressor.force_checkpoint(&mut ctx.messages).await
    }
}
```

`[可控风险]` 安全红线（P0）通过优先级系统设计上保证不可被任务指令覆盖，而非依赖 LLM 的理解。

### 2.4 Agent 主循环 —— 来自 Hermes

**Crate 归属**：`legion-core`（Layer 3）。AgentRuntime 是**单次任务执行引擎**，只负责 LLM 调用循环本身——不关心任务来源、不持有锁、不写审计链，这些由 §5.2 的 `AgentCoordinator` 负责。两者的边界：

```
AgentCoordinator（任务编排层）
  │  负责：心跳协议、原子锁、任务状态流转、审计链写入、路由决策
  │  调用 ↓（传入 Task + RoutingDecision，路由已在此层完成）
AgentRuntime（执行层）
  │  负责：LLM 循环、工具执行、上下文组装、流式推送
  │  输入：Task（已锁定）+ RoutingDecision（已选定模型）  输出：TaskResult
```

Hermes 的 Agent 循环经过实战检验，Legion 直接采纳其核心防护机制：

```rust
// legion-core（Layer 3）：单次任务执行引擎
// 职责边界：仅负责 LLM 调用循环。任务调度、原子锁、审计链不在此处理（见 §5.2 AgentCoordinator）
pub struct AgentRuntime {
    cognitive_core: CognitiveCore,
    tool_registry: Arc<ToolRegistry>,
    tool_guardrails: ToolGuardrails,    // 来自 Hermes
    memory: Arc<MemorySystem>,
    maas_client: Arc<MaasClient>,       // 仅用于 LLM 流式调用，不做路由决策（路由在 AgentCoordinator）
    eval_engine: Arc<EvalEngine>,       // Layer 2 评估：收敛比检测（§2.12）
    permission_enforcer: Arc<PermissionEnforcer>,  // 来自 Claw Code
    event_bus: Arc<EventBus>,           // 流式 token 推送到 UI
}

impl AgentRuntime {
    /// 执行单个已分配任务。
    /// `routing` 由 AgentCoordinator 在调用前完成路由决策（见 §5.2），
    /// Runtime 直接使用 routing.selected_model，不再自行调用 MaaS 路由接口。
    ///
    /// ⚠️ `&mut self`：ContextCompressor 的 cycle_count 需要跨迭代持久化
    /// （SoftLoop 时重置计数器，需要可变访问 cognitive_core）
    pub async fn run_task(&mut self, task: Task, routing: RoutingDecision) -> TaskResult {
        let mut iteration = 0;
        let max_iterations = task.budget.max_iterations;

        loop {
            // 防护1: 迭代预算（来自 Hermes iteration_budget）
            if iteration >= max_iterations {
                return TaskResult::BudgetExhausted;
            }

            // 防护2: 中断检查（用户可随时停止）
            if self.check_interrupt().await? {
                return TaskResult::Interrupted;
            }

            // 组装执行上下文（mut：后续 force_cycle_checkpoint 需要修改消息历史）
            let mut ctx = self.cognitive_core.assemble_context(&task);

            // 流式调用 LLM（统一传输层，参考 Hermes NormalizedResponse + DeepSeek-TUI SSE）
            // 使用路由决策中已确定的模型，不在此处重新路由
            // 使用 stream() 而非 call()：支持实时向 UI 推送 token、减少首字节延迟
            let mut stream = self.maas_client.stream(routing.selected_model.clone(), &ctx.messages).await?;
            let mut response = NormalizedResponse::default();
            while let Some(chunk) = stream.next().await {
                let chunk = chunk?;
                response.merge_chunk(chunk);
                // 每个 delta 推送到事件总线（UI 实时渲染）
                self.event_bus.publish(AgentEvent::TokenDelta(chunk)).await;
            }

            // 无工具调用 → 任务完成
            if response.tool_calls.is_empty() {
                self.post_task_learning(&task, &response).await;
                return TaskResult::Completed(response);
            }

            // 工具执行（含权限检查 + 看门狗）
            let results = self.execute_tools(&response.tool_calls, &task).await?;

            // Layer 2 收敛比检测（§2.12）：内循环实时检测，不依赖后台调度器
            // 在每次工具执行完成后立即评估，发现异常后直接响应，无需等待下一个调度 tick
            // 使用 §2.12 定义的权威类型：eval_trace() → TraceEval.looping_status: LoopingStatus
            let trace_eval = self.eval_engine.eval_trace(&task.agent_id).await;
            match trace_eval.looping_status {
                LoopingStatus::SoftLoop => {
                    // ratio 5~10：触发循环检查点（§2.6），压缩上下文后继续执行
                    self.cognitive_core.force_cycle_checkpoint(&mut ctx).await?;
                }
                LoopingStatus::HardLoop => {
                    // ratio > 10：强制中断，由 AgentCoordinator 触发 §6.2 AnomalyEscalation 审批流程
                    return TaskResult::AnomalyDetected {
                        reason: format!("HardLoop detected at iteration {}", iteration),
                    };
                }
                LoopingStatus::Normal => {}
            }

            // 工作记忆更新（参考 DeepSeek-TUI 循环检查点）
            self.update_working_memory(&results, &mut iteration).await?;

            iteration += 1;
        }
    }

    async fn execute_tools(
        &self,
        calls: &[ToolCall],
        task: &Task,
    ) -> Result<Vec<ToolResult>, ToolError> {
        // 工具看门狗 before_call（来自 Hermes ToolGuardrails）
        for call in calls {
            match self.tool_guardrails.before_call(&call.name, &call.args) {
                Decision::Block => return Err(ToolError::Blocked(call.name.clone())),
                Decision::Halt => return Err(ToolError::Halt),
                _ => {}
            }
        }

        // 权限执行器（来自 Claw Code permission_enforcer）
        self.permission_enforcer.check_batch(calls, &task.permissions)?;

        // 批准策略检查（来自 DeepSeek-TUI ApprovalPolicy × SandboxMode）
        let approved = self.check_approval(calls, &task.approval_policy).await?;

        // 并发执行（来自 Hermes ThreadPoolExecutor 模式）
        let results = futures::future::join_all(
            approved.iter().map(|call| self.tool_registry.execute(call))
        ).await;

        // 工具看门狗 after_call
        for (call, result) in calls.iter().zip(results.iter()) {
            self.tool_guardrails.after_call(&call.name, result);
        }

        results.into_iter().collect()
    }
}
```

### 2.5 批准策略引擎 —— 来自 DeepSeek-TUI

DeepSeek-TUI 的 `ApprovalPolicy × SandboxMode` 正交设计直接满足 Legion 的工具执行风险管控：

```rust
// 来自 DeepSeek-TUI execpolicy crate
pub enum ApprovalPolicy {
    AutoAllow,     // 所有工具自动允许（受 auto_allow 列表约束）
    OnRequest,     // 默认允许，但特定工具需批准
    Untrusted,     // 默认拒绝，每次需要显式批准
    Never,         // 禁止所有工具执行
}

pub enum SandboxMode {
    ReadOnly,           // 只读模式：禁止所有写操作
    WorkspaceWrite,     // 工作区写：只能修改项目目录内文件
    DangerFullAccess,   // 完全访问：所有操作（含系统命令）
    ExternalSandbox,    // 外部沙盒：在隔离容器中执行
}

// Legion 扩展：RBAC 集成
pub struct ExecutionPolicy {
    approval: ApprovalPolicy,
    sandbox: SandboxMode,
    auto_allow_prefixes: Vec<String>,  // 配置驱动，无需代码修改
    role_overrides: HashMap<AgentRole, ApprovalPolicy>,  // RBAC 扩展
    audit_all: bool,                    // 是否记录每次工具执行审计日志
}
```

`[可治理]` 不同角色的 Agent 拥有不同的默认批准策略：CEO Agent 可以自动允许更多工具，新入职 Agent 的沙盒更严格。

`[可控风险]` `auto_allow_prefixes` 配置驱动，管理员无需发布新版本即可调整允许范围；`ExternalSandbox` 将危险操作完全隔离到容器中。

### 2.6 上下文压缩 —— 来自 Hermes + DeepSeek-TUI

综合 Hermes 的 4 层压缩算法和 DeepSeek-TUI 的循环检查点机制：

```rust
pub struct ContextCompressor {
    // 来自 Hermes：4 层压缩策略
    strategy: CompressionStrategy,
    // 来自 DeepSeek-TUI：循环检查点
    // cycle_count 由 ContextCompressor 唯一持有（CognitiveCore 不持有，见 §2.3）
    cycle_length: u32,     // 最大检查点轮次（构造时由 CognitiveCore.max_cycle_length 传入）
    cycle_count: u32,      // 当前轮次计数（由 compress() 内部维护，到达 cycle_length 后重置）
    cycle_briefings: Vec<CycleBriefing>,
}

pub enum CompressionStrategy {
    // 第 1 层：裁剪旧工具输出（零 LLM 调用，零成本）
    TrimOldToolOutputs { keep_last_n: usize },
    // 第 2 层：保护首尾消息（关键上下文不丢失）
    ProtectHeadTail { head_n: usize, tail_n: usize },
    // 第 3 层：辅助模型摘要（轻量模型，控制成本）
    AuxiliarySummary { model: String },
    // 第 4 层：DeepSeek-TUI 循环检查点（超长任务适用）
    CycleCheckpoint { cycle_length: u32 },
}

impl ContextCompressor {
    pub fn should_compress(&self, ctx: &ExecutionContext) -> bool {
        // 来自 DeepSeek-TUI：80% 窗口利用率触发压缩
        let utilization = ctx.token_count as f64 / ctx.max_tokens as f64;
        utilization > 0.80
    }

    pub async fn compress(&mut self, messages: &mut Vec<Message>) -> CompressResult {
        // 第 1 层：零成本裁剪（始终优先，无论 strategy 如何）
        let still_oversized = self.trim_old_tool_outputs(messages);
        if !still_oversized {
            return CompressResult::Ok;
        }

        // 第 2+ 层：依据 strategy 选择后续压缩路径
        match &self.strategy {
            // CycleCheckpoint: 超长任务专用，按轮次周期替换为摘要
            CompressionStrategy::CycleCheckpoint { cycle_length } => {
                if self.cycle_count >= *cycle_length {
                    let briefing = self.generate_cycle_briefing(messages).await;
                    self.cycle_briefings.push(briefing);
                    self.replace_with_briefing(messages);
                    self.cycle_count = 0;
                } else {
                    // 未到检查点轮次，回退到辅助模型摘要
                    self.auxiliary_summary(messages).await;
                }
            }
            // AuxiliarySummary: 默认策略，调用轻量辅助模型
            CompressionStrategy::AuxiliarySummary { .. } => {
                self.auxiliary_summary(messages).await;
            }
            // ProtectHeadTail: 零成本保护，已在 trim 阶段隐式处理
            CompressionStrategy::ProtectHeadTail { head_n, tail_n } => {
                self.trim_with_protection(messages, *head_n, *tail_n);
            }
            // TrimOldToolOutputs: 纯裁剪，已在第 1 层完成
            CompressionStrategy::TrimOldToolOutputs { .. } => {}
        }

        CompressResult::Ok
    }
}
```

`[可观测]` 每次压缩记录：压缩策略、压缩前后 Token 数、摘要内容、压缩原因，管理者可查看 Agent 的上下文管理历史。

### 2.7 学习引擎 —— 来自 Evolver GEP

Evolver 的 GEP（Genome Evolution Protocol）正是 Legion 学习引擎的核心实现参考，直接映射到 Legion.md §3.2.4 的四路学习路径。其形式化理论基础来自论文《From Procedural Skills to Strategy Genes: Towards Experience-Driven Test-Time Evolution》（Wang et al., 2026，arXiv:2604.15097），在 4,590 次实验中验证了 Strategy Gene 表示方法的有效性：**单个 Gene（~230 tokens）比 baseline 提升 +3.0pp，而过于详细的 Procedural Skill（~2,500 tokens）反而下降 -1.1pp**。

**2.7.0 形式化定义 — Strategy Gene 六元组**

**Gene 六元组 g = (m, u, π, α, c, v)**（论文 §3 核心定义）：

| 符号 | 字段名 | 含义 | 实验贡献 |
|------|--------|------|---------|
| m | matching signals | 任务识别触发关键词 | 任务召回精度 |
| u | compact summary | 高层意图描述（~50 tokens） | 上下文压缩 |
| π | strategy steps | 有序过程指导（核心控制逻辑） | 主要提升来源 |
| **α** | **AVOID cues** | **失败模式编码，禁止动作列表** | **+4.6pp（最高单字段贡献）** |
| c | constraints | 可选执行边界（爆炸半径限制） | 安全隔离 |
| v | validation hooks | 可选可执行验证检查 | 质量门控 |

> ⚠️ **AVOID 线索（α）是 Gene 中价值最高的字段**：编码「不应该做什么」比编码「应该做什么」对性能提升更大（+4.6pp）。失败模式的泛化能力远强于成功模式，因此 Gene 模板设计时 α 字段应强制非空。

**Capsule 六元组 κ = (q, Gκ, Tκ, oκ, Vκ, ℓκ)**（执行证明记录）：

| 符号 | 含义 |
|------|------|
| q | 任务签名（task signature，用于检索匹配） |
| Gκ | 本次执行使用的 Gene 集合 |
| Tκ | 完整执行轨迹（execution trace） |
| oκ | 执行结果（outcome） |
| Vκ | 验证记录（validation record） |
| ℓκ | 血统指针（lineage pointer → 父 Capsule） |

**进化事件九元组 e = (t, ρ, a_src, a_dst, σ, ι, Δ, ν, τ)**（不可变审计记录）：

| 符号 | 含义 |
|------|------|
| t | 事件类型（distill / mutate / crossover / repair） |
| ρ | 父运行 ID（parent run ID，溯源链） |
| a_src / a_dst | 源资产 / 目标资产 ID（变更前后） |
| σ | 触发信号（trigger signal） |
| ι | 变更意图描述（intent） |
| Δ | 内容差异（diff，JSON patch 格式） |
| ν | 验证结果（validation result） |
| τ | 时间戳 |

**2.7.1 三层信号提取（对应路径一：任务反馈学习）**

```rust
// 来自 Evolver signals.js 的 3 层架构
pub struct SignalExtractor {
    // 第 1 层：正则匹配（确定性，0ms）
    regex_patterns: Vec<SignalPattern>,
    // 第 2 层：关键词打分（统计，0ms）
    keyword_scorer: KeywordScorer,
    // 第 3 层：LLM 语义分析（限速，每 5 轮）
    semantic_analyzer: SemanticAnalyzer,
    llm_rate_limiter: RateLimiter,
}

impl SignalExtractor {
    pub async fn extract(&self, execution_log: &ExecutionLog) -> Vec<Signal> {
        // 层 1: 正则匹配 — 无成本，优先执行
        let mut signals = self.regex_patterns
            .iter()
            .flat_map(|p| p.match_log(execution_log))
            .collect::<Vec<_>>();

        // 层 2: 关键词打分 — 补充正则未覆盖的隐含信号
        let keyword_signals = self.keyword_scorer.score(execution_log);
        signals.extend(keyword_signals);

        // 层 3: LLM 语义分析 — 限速执行，发现新信号
        if self.llm_rate_limiter.allow() {
            let semantic_signals = self.semantic_analyzer.analyze(execution_log).await;
            signals.extend(semantic_signals);
        }

        // 后处理（来自 Evolver 的去重和优先级逻辑）
        self.dedup_and_prioritize(signals)
    }

    fn dedup_and_prioritize(&self, mut signals: Vec<Signal>) -> Vec<Signal> {
        // 8 轮内出现 >= 3 次 → 抑制（防止重复修复循环）
        signals.retain(|s| s.recent_count < 3);
        // 检测修复循环 → 强制切换到创新策略
        if self.detect_fix_loop(&signals) {
            signals.push(Signal::ForceInnovate);
        }
        // 按优先级排序：可操作信号 > 表面信号
        signals.sort_by_key(|s| s.priority);
        signals
    }
}
```

**2.7.2 进化资产三元模型（对应路径二：自我反思学习）**

```rust
// g = (m, u, π, α, c, v) — 论文六元组映射到 Legion §3.2.3 能力记忆
pub struct Gene {
    pub id: String,           // 策略 ID
    pub asset_id: String,     // sha256:... 内容寻址，总长度约束 ≤ 230 tokens

    // 六元组字段（对应论文符号）
    pub m_signals: Vec<String>,         // m: 匹配信号 — 任务识别触发关键词
    pub u_summary: String,              // u: 紧凑摘要 — 高层意图（~50 tokens）
    pub pi_steps: Vec<String>,          // π: 策略步骤 — 有序过程指导（核心控制逻辑）
    pub alpha_avoid: Vec<String>,       // α: ⚠️ AVOID 线索 — 失败模式编码（+4.6pp，最高贡献）
    pub c_constraints: GeneConstraints, // c: 约束条件 — 爆炸半径边界（optional）
    pub v_validation: Vec<ValidationCmd>, // v: 验证钩子 — 可执行完整性检查（optional）

    // 元数据
    pub name: String,
    pub created_by: AgentId,
    pub version: u32,
    pub success_rate: f64,  // 累计成功率（来自 Capsule 统计，影响召回优先级）
    pub usage_count: u32,   // 使用次数（高频 Gene 优先注入，上限 3 个/任务）
}

// κ = (q, Gκ, Tκ, oκ, Vκ, ℓκ) — 论文六元组执行证明记录
pub struct Capsule {
    pub id: String,
    pub asset_id: String,               // sha256: 内容寻址

    // 六元组字段（对应论文符号）
    pub q_task_signature: String,       // q: 任务签名 — 用于后续检索匹配
    pub g_genes_used: Vec<GeneId>,      // Gκ: 本次执行使用的 Gene 集合
    pub t_trace: ExecutionTrace,        // Tκ: 完整执行轨迹
    pub o_outcome: Outcome,             // oκ: 执行结果（success/failure + 输出摘要）
    pub v_record: ValidationRecord,     // Vκ: 验证记录（哪些钩子通过/失败）
    pub l_lineage: Option<CapsuleId>,   // ℓκ: 血统指针 → 父 Capsule ID

    // 额外元数据
    pub success_count: u32,
    pub environment: EnvironmentSnapshot,
    pub created_by: AgentId,
}

// e = (t, ρ, a_src, a_dst, σ, ι, Δ, ν, τ) — 论文九元组不可变审计记录
pub struct EvolutionEvent {
    pub id: String,
    pub asset_id: String,               // sha256: 内容寻址

    // 九元组字段（对应论文符号）
    pub t_type: EvolutionEventType,     // t: 事件类型（distill/mutate/crossover/repair）
    pub rho_parent_run: Option<RunId>,  // ρ: 父运行 ID（事件溯源链）
    pub a_src: Option<AssetId>,         // a_src: 源资产 ID（变更前，None 表示新建）
    pub a_dst: AssetId,                 // a_dst: 目标资产 ID（变更后）
    pub sigma_signal: Vec<String>,      // σ: 触发信号列表
    pub iota_intent: String,            // ι: 变更意图描述
    pub delta_diff: JsonPatch,          // Δ: 内容差异（JSON patch 格式）
    pub nu_validation: ValidationResult, // ν: 验证结果
    pub tau_timestamp: DateTime<Utc>,   // τ: 时间戳

    // Legion 扩展
    pub agent_id: AgentId,
    pub immutable_seal: Ed25519Signature, // Ed25519 签名，防止回溯修改
}
```

`[可观测]` `EvolutionEvent` 的 `rho_parent_run`（ρ 溯源字段）链接形成完整的进化历史链，管理者可以追溯每一步学习决策。

`[可治理]` Gene 的 `v_validation` 字段确保每个学习规则在写入能力记忆前通过验证；管理员可以审核、删除或注入 Gene。

`[可控风险]` `c_constraints` 字段设置爆炸半径（`max_files_changed`, `forbidden_paths`）；金丝雀检查防止损坏代码写入。

**2.7.3 固化流程（对应路径三：协作观察学习）**

```rust
// 来自 Evolver solidify.js 的验证-审计流程
// ⚠️ 架构约束：legion-learning 是 Layer 2，不直接依赖 Git（Layer 1 工具）。
// Git 提交通过 post_solidify_hooks 由 legion-core（Layer 3）注入。
pub struct SolidifyPipeline {
    validator: ChangeValidator,
    blast_radius_assessor: BlastRadiusAssessor,
    // ⚠️ 与 GepState 共享同一个 Arc<CapsuleStore>——solidify_gene() 写入后，
    // GepState.apply() 无需再次写入，避免双写。
    capsule_store: Arc<CapsuleStore>,
    event_log: EvolutionEventLog,
    // 钩子由 legion-core 注入，实现层间解耦（Git、通知等）
    post_solidify_hooks: Vec<Box<dyn PostSolidifyHook>>,
}

/// 代码变更固化（§2.7.3 路径三：协作观察学习）
/// 用于 Agent 的实际代码改动：验证 → 爆炸半径 → Capsule → 审计链
/// 钩子负责 Git commit（由 legion-core 注入，legion-learning 不知道 Git）
impl SolidifyPipeline {
    pub async fn solidify_code_change(
        &self,
        change: &AgentChange,
        gene: &Gene,
    ) -> Result<Capsule, SolidifyError> {
        // 步骤1: 验证钩子执行（v_validation 字段）
        for cmd in &gene.v_validation {
            let result = self.validator.run(cmd).await?;
            if !result.success { return Err(SolidifyError::ValidationFailed); }
        }

        // 步骤2: 金丝雀检查
        self.validator.canary_check().await?;

        // 步骤3: 爆炸半径评估（c_constraints 字段）
        let blast = self.blast_radius_assessor.assess(change);
        if blast.exceeds_limit(&gene.c_constraints) {
            return Err(SolidifyError::BlastRadiusTooLarge(blast));
        }

        // 步骤4: 创建 Capsule（执行证明）
        let capsule = self.capsule_store.create(gene, change).await?;

        // 步骤5: 追加 EvolutionEvent（不可变审计链）
        self.event_log.append(EvolutionEvent::from_code_change(&capsule)).await?;

        // 步骤6: 执行钩子（Git commit 等 Layer 1 操作由此触发）
        for hook in &self.post_solidify_hooks {
            hook.on_solidified(&capsule, change).await?;
        }

        Ok(capsule)
    }

    /// Gene 资产固化（§2.7.5 GEP 周期专用）
    /// 用于将蒸馏出的新 Gene 写入 𝒢，与 solidify_code_change 语义不同。
    ///
    /// ⚠️ 架构约束：此方法负责将 Gene 和 Capsule 持久化到共享 capsule_store（Arc）。
    /// 调用方（GepCycle）后续执行 state.apply(delta) 只需将 gene 推入内存 vec，
    /// 不会再次写 DB——双写问题由此消除。
    pub async fn solidify_gene(
        &self,
        draft: GeneDraft,
        validation: &ValidationResult,
    ) -> Result<GepStateDelta, SolidifyError> {
        // 将草稿提升为正式 Gene（验证已通过）
        let gene = Gene::from_validated_draft(draft, validation);

        // 写入能力记忆（𝒢 持久化，共享 capsule_store）
        self.capsule_store.upsert_gene(&gene).await?;

        // 创建对应 Capsule（𝒞 持久化，ℓκ 指向空 = 新建根节点）
        let capsule = Capsule {
            q_task_signature: validation.task_signature.clone(),
            g_genes_used: vec![gene.id.clone()],
            o_outcome: Outcome::GeneCreated,
            l_lineage: None,  // 新建 Gene，无父 Capsule
            ..Default::default()
        };
        self.capsule_store.insert_capsule(&capsule).await?;

        // 追加 EvolutionEvent（ℰ 更新，不可变）
        let event = EvolutionEvent {
            t_type: EvolutionEventType::Distill,
            a_dst: gene.id.clone().into(),
            nu_validation: validation.result.clone(),
            tau_timestamp: Utc::now(),
            ..Default::default()
        };
        self.event_log.append(event).await?;

        // 只返回 Gene 对象供 GepState 更新内存集合；Capsule 已写入共享 DB，无需再传
        Ok(GepStateDelta::Evolved { new_gene: gene })
    }
}

/// 层间解耦钩子接口（由 legion-core 注入，legion-learning 只知道此接口）
#[async_trait]
pub trait PostSolidifyHook: Send + Sync {
    async fn on_solidified(&self, capsule: &Capsule, change: &AgentChange) -> Result<(), HookError>;
}
```

**2.7.4 知识库驱动学习（对应路径四）**

参见 §三 LLM Wiki 设计，Agent 通过 Wiki API 实时检索相关知识注入认知内核。

**2.7.5 GEP 六阶段形式化周期**

论文定义 GEP 为从状态三元组 **(𝒢, 𝒞, ℰ)** 到 **(𝒢′, 𝒞′, ℰ′)** 的变换（𝒢 = Gene 集合，𝒞 = Capsule 集合，ℰ = EvolutionEvent 审计链），包含六个阶段：

```
(𝒢, 𝒞, ℰ)
    │
    ├─ [1] scan     ← 扫描近期执行历史 ℰ，提取候选经验片段
    │
    ├─ [2] signal   ← 三层信号提取（§2.7.1），输出触发信号 σ
    │
    ├─ [3] intent   ← LLM 推断变更意图 ι（为什么要进化？）
    │
    ├─ [4] mutate   ← 蒸馏算子 ψ 生成 Gene 候选 g′（从 σ+ι 提炼）
    │
    ├─ [5] validate ← 执行验证钩子 v + 金丝雀检查，输出 ν
    │
    └─ [6] solidify ← ν=pass → 写入 𝒢，创建 Capsule κ，追加 Event e
    │
(𝒢′, 𝒞′, ℰ′)
```

| 阶段 | 输入 | 核心操作 | 输出 |
|------|------|---------|------|
| **scan** | 最近执行历史 ℰ | 识别近期失败/成功模式 | 候选经验片段 |
| **signal** | 候选片段 | 三层信号提取（§2.7.1） | 触发信号 σ |
| **intent** | 信号 σ | LLM 推断变更意图 | 意图描述 ι |
| **mutate** | 意图 ι + 现有 𝒢 | 蒸馏算子 ψ 生成草稿 | Gene 候选 g′ |
| **validate** | Gene 草稿 g′ | 执行 v 钩子 + 金丝雀 | 验证结果 ν |
| **solidify** | ν=pass + g′ | 原子写入三元组 | (𝒢′, 𝒞′, ℰ′) |

```rust
/// GEP 三元组状态载体：持有 (𝒢, 𝒞, ℰ) 的引用
/// 对应论文形式化定义：𝒢 = Gene 库，𝒞 = Capsule 集合，ℰ = Evolution Event 日志
///
/// ⚠️ 设计约束：𝒞（Capsule 集合）不预加载全量到内存——
///   90 天窗口可能含数万条记录，全量加载会造成内存压力。
///   改为通过 Arc<CapsuleStore> 懒加载：distillation_op.apply() 按需查询，
///   每次 GEP 周期不强制读取所有 Capsule。
pub struct GepState {
    /// 𝒢 — 当前有效 Gene 库（按 agent_id 分区，常驻内存，数量有限）
    pub genes: Vec<Gene>,
    /// 𝒞 — Capsule 存储的 Arc 句柄（懒加载，按需查询近 90 天记录）
    pub capsule_store: Arc<CapsuleStore>,
    /// ℰ — 进化事件日志（append-only，Ed25519 签名保护）
    pub event_log: Arc<EvolutionEventLog>,
}

impl GepState {
    /// 将 solidify 产出的增量应用到内存 Gene 库（同步）。
    /// Capsule 已由 SolidifyPipeline.solidify_gene() 写入共享 capsule_store（Arc），
    /// 此处只需将新 Gene push 进内存 vec，无需再写 DB。
    pub fn apply(&mut self, delta: GepStateDelta) {
        if let GepStateDelta::Evolved { new_gene } = delta {
            self.genes.push(new_gene);
        }
        // GepStateDelta::NoChange：无需操作
    }
}

/// GEP 周期实现 — 对应论文 Algorithm 1
pub struct GepCycle {
    signal_extractor: SignalExtractor,
    intent_analyzer: IntentAnalyzer,
    distillation_op: Box<dyn DistillationOperator>,  // ψ
    validator: GeneValidator,
    solidify_pipeline: SolidifyPipeline,
}

impl GepCycle {
    /// 运行单次 GEP 周期：(𝒢,𝒞,ℰ) → (𝒢′,𝒞′,ℰ′)
    /// 触发条件：检测到失败模式（修复驱动），而非定时触发
    pub async fn run_cycle(
        &self,
        state: &mut GepState,  // 持有 (𝒢, 𝒞, ℰ) 引用
    ) -> Result<GepStateDelta, GepError> {
        // Phase 1: scan — 扫描近 24h 执行历史，同时提取关联的执行轨迹
        // recent_with_traces() 同时返回事件列表（用于信号提取）和具体执行轨迹（用于蒸馏）
        // 轨迹必须在此阶段提取，在 apply() 时已无法重新获取
        let (history, candidate_traces) = state.event_log
            .recent_with_traces(Duration::hours(24)).await;

        // Phase 2: signal — 三层信号提取
        let signals = self.signal_extractor.extract(&history).await;
        if signals.is_empty() { return Ok(GepStateDelta::NoChange); }

        // Phase 3: intent — 推断变更意图
        let intent = self.intent_analyzer.analyze(&signals).await?;

        // Phase 4: mutate — 蒸馏生成/变异 Gene（token 预算 ≤ 230）
        // 传入 candidate_traces：算子内部据此选择 ψ(s) 单轨迹 或 ψ(ℋ) 多轨迹路径
        let candidate = self.distillation_op
            .apply(&intent, &state.genes, &candidate_traces).await?;

        // Phase 5: validate — 多级验证
        let validation = self.validator.validate(&candidate).await?;
        if !validation.passed { return Err(GepError::ValidationFailed(validation)); }

        // Phase 6: solidify — 原子写入状态（三元组更新，使用 Gene 专用方法）
        // solidify_gene() 已将 Gene + Capsule 写入共享 capsule_store（Arc）
        let delta = self.solidify_pipeline.solidify_gene(candidate, &validation).await?;
        state.apply(delta.clone());  // 同步：仅更新内存 genes vec，DB 写入已在上一步完成
        Ok(delta)
    }
}
```

**2.7.6 蒸馏算子 ψ（Distillation Operator）**

论文定义两种 Gene 来源，对应两种 ψ 形式：

- **`ψ(s)`**：从单次成功执行轨迹 s 中提炼 Gene（快速，单样本）
- **`ψ(ℋ)`**：从历史执行集合 ℋ 中归纳跨任务模式（慢，多样本）

```rust
pub trait DistillationOperator: Send + Sync {
    /// ψ(s): 从单次执行轨迹提炼 Gene（快速路径，单样本蒸馏）
    async fn from_trajectory(&self, s: &ExecutionTrace) -> Result<GeneDraft, DistillError>;

    /// ψ(ℋ): 从历史集合归纳 Gene（慢路径，跨任务模式识别）
    async fn from_history(&self, h: &[ExecutionTrace]) -> Result<GeneDraft, DistillError>;

    /// GEP 周期入口：基于触发信号 σ + 意图 ι + 现有 Gene 库生成候选草稿
    /// candidate_traces: Phase 1（scan）阶段提取的候选轨迹，算子内部据此选择路径：
    ///   - 非空 → 调用 from_trajectory（ψ(s)，快速单样本路径）
    ///   - 为空 → 调用 from_history（ψ(ℋ)，慢路径跨任务归纳）
    async fn apply(
        &self,
        intent: &Intent,
        gene_pool: &[Gene],
        candidate_traces: &[ExecutionTrace],  // 蒸馏源轨迹（ψ路径选择依据）
    ) -> Result<GeneDraft, DistillError>;
}

pub struct LlmDistillationOperator {
    llm_client: Arc<dyn LlmClient>,
    token_budget: u32,  // ≤ 230（论文实验最优值，超过 500 触发自动压缩）
}

impl LlmDistillationOperator {
    fn build_distill_prompt(&self, traces: &[ExecutionTrace]) -> String {
        // 强制要求 AVOID 线索（α）非空
        format!(
            "从以下执行轨迹中提炼策略基因，严格遵守格式：\n\
             - m（匹配信号）：任务识别关键词 3-8 个\n\
             - u（紧凑摘要）：一句话描述策略意图（≤20 词）\n\
             - π（策略步骤）：有序步骤列表，≤5 步\n\
             - α（AVOID线索）：⚠️ 必填！列举已知失败模式和禁止操作\n\
             - c（约束条件）：可选，执行边界说明\n\
             - v（验证钩子）：可选，可执行检查命令\n\
             Token 总预算：≤{} tokens\n\n\
             轨迹：{:#?}",
            self.token_budget,
            traces
        )
    }
}
```

**token 预算约束**（论文实验结论）：

| 表示方式 | 平均长度 | 相对 baseline 效果 |
|---------|---------|-----------------|
| Strategy Gene（本设计）| ~230 tokens | **+3.0pp** |
| Procedural Skill | ~2,500 tokens | **-1.1pp**（性能下降！）|
| 无经验 baseline | — | 0 |

Gene 比 Skill 短 10 倍，效果好 4.1pp。根本原因：**紧凑表示减少上下文噪声**，Skill 的详细步骤反而使模型过拟合特定场景。

**2.7.7 实验验证与 Legion 设计原则**

**论文核心实验数据**（4,590 次试验，45 个科学场景）：

| 指标 | 数值 | 意义 |
|------|------|------|
| Evolver vs base（CritPt 基准） | 27.14% vs 17.7% | +9.44pp 整体提升 |
| 单 Gene vs 多 Gene 组合 | 单 Gene 更优 | 多 Gene 模糊控制焦点 |
| AVOID cues α 单字段贡献 | **+4.6pp** | 失败模式泛化性最强 |
| Gene vs Skill 对比 | +3.0pp vs -1.1pp | 紧凑表示优于详细步骤 |
| Gene 结构鲁棒性 | 步骤顺序敏感 52.8% | 语义比结构更脆弱 |

**Legion 五项设计原则**（来自论文洞见，优先级从高到低）：

1. **表示即算法（Representation > Algorithm）**：学习引擎的核心价值在于将执行经验压缩为正确的知识表示格式（Gene 六元组），而非设计复杂的进化算法。一个好的 Gene 比一段复杂的代码更有价值。

2. **失败优先（Failure-First Encoding）**：α 字段（AVOID 线索）是 Gene 中最重要的字段（+4.6pp），强制非空。设计 Gene 模板时，未填写 α 的草稿应被 validator 拒绝。

3. **单焦点原则（Single Focus）**：每个 Gene 只解决一类问题。每次任务注入的 Gene 数量限制 ≤ 3 个，多 Gene 组合会模糊控制焦点。

4. **Token 预算纪律（Token Budget）**：Gene 总 token 数控制在 ≤ 230。超过 500 tokens 的 Gene 触发自动压缩。禁止将 Procedural Skill 作为 Gene 的主体内容。

5. **修复驱动进化（Repair-Driven Evolution）**：GEP 周期的主要触发条件是检测到失败模式，而非定时触发或探索性重组。进化是收敛行为，不是随机搜索。

`[可观测]` GEP 周期每次运行记录完整的六阶段状态快照：触发信号 σ → 意图 ι → Gene 草稿 → 验证结果 ν → 最终状态。管理者可追溯每个 Gene 的完整来源轨迹（通过 EvolutionEvent 的 ρ/σ/ι/Δ 字段）。

`[可治理]` 管理者可以：①在 validate 阶段前人工审核 Gene 草稿；②直接编辑 Gene 的 α 字段注入已知失败模式，修正 Agent 的错误倾向；③手动限制或解除每任务 Gene 注入数量上限。

`[可控风险]` 三重保障：①token 预算约束（≤ 230 tokens）防止 Gene 膨胀；②validation 钩子（v 字段）在固化前验证，失败直接拒绝；③单焦点原则限制每个 Gene 的影响范围，防止一个错误 Gene 破坏所有任务。

### 2.8 三层记忆系统

映射 Legion.md §3.2.3 到具体实现：

```rust
pub struct MemorySystem {
    // 层1: 工作记忆 — 单次任务，内存中维护
    working: WorkingMemory,

    // 层2: 情景记忆 — 跨任务持久化，SQLite
    episodic: EpisodicMemoryStore,

    // 层3: 能力记忆 — Gene/Capsule 资产，长期沉淀
    capability: CapabilityMemoryStore,
}

// 来自 Hermes MemoryProvider ABC 的可插拔设计
pub trait MemoryProvider: Send + Sync {
    fn system_prompt_block(&self) -> Option<String>;  // 注入系统提示
    async fn prefetch(&self, task: &Task) -> Vec<MemoryChunk>;  // 任务前预取
    async fn sync_after_turn(&self, messages: &[Message]);      // 回合后同步
}

// 内置实现：Legion 本地记忆（与 LLM Wiki 集成）
pub struct LegionMemoryProvider {
    episodic_store: Arc<EpisodicMemoryStore>,
    wiki_client: Arc<WikiClient>,
    embedding_model: Arc<EmbeddingModel>,
}
```

### 2.9 进化评估器

量化追踪 Legion.md §3.2.5 的 5 维度：

```rust
pub struct EvolutionEvaluator {
    metrics_store: Arc<MetricsStore>,
    alert_threshold: EvaluationThresholds,
}

impl EvolutionEvaluator {
    pub async fn evaluate(&self, agent_id: &AgentId, window: TimeWindow) -> AgentReport {
        AgentReport {
            task_quality: self.calc_task_quality(agent_id, window).await,
            decision_efficiency: self.calc_efficiency(agent_id, window).await,
            learning_progress: self.calc_learning_trend(agent_id, window).await,
            collaboration_score: self.calc_collaboration(agent_id, window).await,
            risk_behavior: self.calc_risk_behavior(agent_id, window).await,
        }
    }

    // 检测能力退化 — 自动触发预警
    pub async fn detect_degradation(&self, agent_id: &AgentId) -> Option<DegradationAlert> {
        let trend = self.get_quality_trend(agent_id, last_n_tasks(20)).await;
        if trend.is_declining(0.15) {
            Some(DegradationAlert {
                agent_id: agent_id.clone(),
                dimension: trend.worst_dimension,
                severity: trend.severity(),
                suggested_action: self.suggest_intervention(&trend),
            })
        } else { None }
    }
}
```

`[可观测]` 进化评估器定期生成"Agent 成长报告"，含能力雷达图变化趋势，管理者可随时查看。

`[可治理]` 检测到能力退化自动通知管理者；管理者可以手动触发能力回退、删除错误记忆、注入正确经验。

### 2.10 任务状态机与 Aegis 质量门控 —— 来自 Mission Control

#### 2.10.1 六状态任务状态机

Mission Control 的任务状态机设计是**目前分析的五个项目中最完整的产出质量管控方案**，Legion 直接采纳其核心约束：

```rust
pub enum TaskStatus {
    Inbox,          // 新建待路由
    Assigned,       // 已路由到 Agent，等待执行
    InProgress,     // Agent 正在执行
    QualityReview,  // Agent 完成，等待 Aegis 审核
    Done,           // 已通过质量审核，终态（不可逆）
    Failed,         // 失败（超出重试次数 / 评审 3 次被拒）
    Suspended,      // HardLoop 强制挂起，等待人工审批后恢复（见 §6.2 AnomalyEscalation）
}

impl TaskStatus {
    /// [可治理] 硬约束：只有通过 Aegis 批准才能进入 Done
    /// API 层拒绝任何绕过路径，返回 403
    pub fn can_transition_to_done(&self, reviews: &[QualityReview]) -> bool {
        matches!(self, TaskStatus::QualityReview)
            && reviews.iter().any(|r| r.reviewer == "aegis" && r.verdict == Verdict::Approved)
    }

    /// [可控风险] Suspended 状态只能由 TaskScheduler 在 HardLoop 场景触发；
    /// 恢复（→ InProgress）必须通过 §6.2 审批门控，API 层拒绝直接恢复
    pub fn can_transition_to_suspended(&self) -> bool {
        matches!(self, TaskStatus::InProgress)
    }
}
```

**状态转换图**（七状态，新增 Suspended）：

```
inbox
  ↓ autoRoute()（调度器 task_dispatch，60s）
assigned
  ↓ atomic_claim()（原子 RETURNING UPDATE，无竞态）
in_progress
  ↓ Agent 提交审核
quality_review ──→ Aegis REJECTED（最多 max_cycles 次）──→ in_progress
  ↓ Aegis APPROVED
done（不可逆终态）

in_progress（HardLoop ratio > 10）──→ suspended（等待人工审批）
  ↓ 审批 Approved → in_progress（继续执行）
  ↓ 审批 Denied  → failed

in_progress（卡死超时）──→ stale_requeue ──→ assigned（最多 5 次）──→ failed
```

#### 2.10.2 Aegis 质量评审器

**定位澄清**：Aegis 是**固定的 LLM 评审器**，不是可进化的 Agent。它直接调用 MaaS 层，使用固定的 Prompt 模板，不走 `AgentRuntime` 执行路径，不产生 Gene/Capsule，也不需要被 Aegis 自身审核（无自举问题）。

这个定位选择的理由：
- **稳定性优先**：质量门控自身如果可被进化修改，会引入"守门人被篡改"的安全风险
- **无自举依赖**：Aegis 只依赖 MaaS 层（Phase 1 完成即可运行），不依赖 AgentRuntime
- **可理解性**：固定 Prompt 模板让 Aegis 的判断逻辑完全透明，管理员可以直接审阅和修改

```rust
// ⚠️ Aegis 不是 Agent，不走 AgentRuntime 路径
// 它是一个固定 Prompt 模板的 LLM 调用封装
// 依赖层次：legion-core（直接调用 MaaS，无需 AgentRuntime 初始化）
pub struct AegisReviewer {
    model_client: Arc<MaasClient>,
    max_rejection_cycles: u32,    // 默认 3，超出自动转 failed
    // 固定 Prompt 模板（可配置但不可自动进化）
    prompt_template: AegisPromptTemplate,
}

pub struct QualityReview {
    task_id:   TaskId,
    reviewer:  String,           // "aegis" 或自定义评审 Agent 名
    verdict:   Verdict,
    feedback:  String,           // 驳回时的改进建议
    cycle:     u32,              // 当前是第几轮审核
    created_at: DateTime<Utc>,
}

pub enum Verdict {
    Approved,
    Rejected { reason: String, suggestions: Vec<String> },
}

impl AegisReviewer {
    pub async fn review(&self, task: &Task, output: &AgentOutput) -> QualityReview {
        // 构建审查 prompt：任务描述 + 验收标准 + Agent 输出
        let prompt = self.build_review_prompt(task, output);

        // 通过 MaaS 路由到高质量模型（Aegis 默认使用 Advanced Tier）
        let response = self.model_client
            .call_with_tier(ModelTier::Advanced, &prompt).await?;

        // 解析 "VERDICT: APPROVED/REJECTED" 格式
        self.parse_verdict(&response)
    }
}
```

**与工作流 DSL 集成**：每个 `Sequence` 步骤可配置 `quality_gate`：

```yaml
workflow:
  steps:
    - id: implement_feature
      agent_role: backend_developer
      quality_gate:
        enabled: true
        reviewer: aegis
        max_cycles: 3
        criteria:
          - "代码通过所有单元测试"
          - "无安全漏洞"
          - "API 文档完整"
```

`[可观测]` 每轮审核结果（批准/驳回 + 具体理由）写入 `quality_reviews` 表并广播 SSE 事件，UI 实时反馈。

`[可治理]` Aegis 的审查标准 criteria 可在 YAML 中自定义；`max_cycles` 上限防止无限驳回循环。

`[可控风险]` 超过 `max_rejection_cycles` 自动转 failed 并升级告警——不让 Agent 无限重试消耗预算。

### 2.11 Agent 信任评分系统 —— 来自 Mission Control

#### 2.11.1 与进化评估器的正交关系

进化评估器（§2.9）衡量 **能力成长**（任务质量、学习进度），信任评分衡量 **安全行为**（是否试图越权、泄露秘密、触发注入）。两者正交，共同构成完整的 Agent 评估体系：

```
能力维度（§2.9 进化评估器）  ×  安全维度（§2.11 信任评分）
      ↓                              ↓
  学习引擎驱动                 派遣决策 + 安全态势
```

#### 2.11.2 事件驱动实时重算

```rust
// 来自 Mission Control TRUST_WEIGHTS，Legion 扩展了冷却机制
// 权重设计原则：恢复必须极度困难——50次成功任务才能抵消1次密钥泄露
// 数学验证：0.20 / 0.004 = 50（SecretExposure / TaskSuccess）
const TRUST_WEIGHTS: &[(SecurityEventType, f64)] = &[
    (SecurityEventType::SecretExposure,    -0.20),  // 最严重：密钥泄露（需50次成功任务才能恢复）
    (SecurityEventType::InjectionAttempt,  -0.15),  // 严重：注入攻击（需38次成功任务恢复）
    (SecurityEventType::UnauthorizedAccess,-0.10),  // Legion 扩展：越权访问（需25次恢复）
    (SecurityEventType::AuthFailure,       -0.05),  // 认证失败（需13次恢复）
    (SecurityEventType::RateLimitHit,      -0.03),  // 速率限制（需8次恢复）
    (SecurityEventType::TaskSuccess,       +0.004), // 成功极慢恢复（50次成功 = 1次密钥泄露）
    (SecurityEventType::TaskFailure,       -0.01),  // 任务失败（需3次成功抵消）
];

pub struct AgentTrustProfile {
    score:          f64,           // 0.0 ~ 1.0，初始 1.0
    event_counters: HashMap<SecurityEventType, u32>,
    last_anomaly_at: Option<DateTime<Utc>>,
    cooldown_until:  Option<DateTime<Utc>>,  // Legion 扩展：冷却期
}

impl AgentTrustProfile {
    /// 安全事件触发时实时重算（不用定时批量）
    pub fn record_event(&mut self, event: SecurityEventType) {
        *self.event_counters.entry(event).or_insert(0) += 1;

        let mut score = 1.0f64;
        for (evt, weight) in TRUST_WEIGHTS {
            let count = self.event_counters.get(evt).copied().unwrap_or(0) as f64;
            score += count * weight;
        }
        self.score = score.clamp(0.0, 1.0);

        // 严重事件（|delta| > 0.10）触发 24 小时冷却期
        // 冷却期内：任务成功不能让信任分超过 0.6（Legion 扩展 MC 没有此机制）
        let weight = TRUST_WEIGHTS.iter()
            .find(|(e, _)| *e == event)
            .map(|(_, w)| *w)
            .unwrap_or(0.0);
        if weight < -0.10 {
            self.last_anomaly_at = Some(Utc::now());
            self.cooldown_until = Some(Utc::now() + Duration::hours(24));
        }
    }

    /// 冷却期采用**懒惰求值**：每次查询时实时检查时间戳，无需后台定时任务清理。
    ///
    /// 设计选择：`cooldown_until` 记录绝对时间点，`effective_score()` 在读取时判断是否过期。
    /// 这消除了定时任务与实时查询之间的状态不一致窗口——调度器无需 `trust_score_cooldown` 任务。
    ///
    /// 对比"定时清理"方案的缺点：若调度器 1h 运行一次，冷却期到期后最多有 1h 的延迟，
    /// Agent 被不必要地限制，且两套机制并行会产生短暂的状态矛盾。
    pub fn effective_score(&self) -> f64 {
        if self.cooldown_until.map(|t| t > Utc::now()).unwrap_or(false) {
            self.score.min(0.6)  // 冷却期分数上限 0.6
        } else {
            self.score
        }
    }
}
```

#### 2.11.3 信任分驱动派遣决策

```rust
impl IntelligentRouter {
    /// 信任分过低的 Agent 自动降级处置
    pub fn apply_trust_constraints(
        &self,
        agent: &AgentDef,
        task: &Task,
        trust: &AgentTrustProfile,
    ) -> Result<(), RouterError> {
        let score = trust.effective_score();

        match score {
            s if s < 0.3 => {
                // 极低信任：拒绝派遣，人工介入
                Err(RouterError::TrustTooLow { score: s, threshold: 0.3 })
            }
            s if s < 0.5 => {
                // 低信任：只允许 low/medium 优先级 + Light Tier + 全工具审批
                if task.priority >= Priority::High {
                    return Err(RouterError::TaskTooHighRiskForAgent { score: s });
                }
                Ok(())  // 降级通过，但调用方需附加严格 ApprovalPolicy
            }
            _ => Ok(()),
        }
    }
}
```

`[可观测]` 信任分变更实时广播 SSE 事件，安全审计面板实时更新每个 Agent 的风险状态。

`[可治理]` 管理员可查看安全事件历史，手动重置错误记录（如误报的注入检测）；冷却期可配置。

`[可控风险]` 低信任分 Agent 自动降为严格沙盒，高风险任务绕行，整体安全态势通过信任分均值调制。

### 2.12 四层 Eval 框架 —— 来自 Mission Control

对现有 §2.9 进化评估器（能力维度）的**行为健康补充**，形成两套正交评估。

```rust
impl FourLayerEvalEngine {
    /// Layer 1 — Output Eval（产出质量，7 天窗口）
    pub async fn eval_output(&self, agent_id: &AgentId) -> OutputEval {
        let stats = self.db.task_stats(agent_id, Duration::days(7));
        let completion_rate = stats.done as f64 / stats.total.max(1) as f64;
        let correctness = if stats.rated_count > 0 {
            // 成功率 × 0.6 + 归一化反馈评分 × 0.4
            completion_rate * 0.6 + stats.avg_feedback_normalized * 0.4
        } else { completion_rate };
        OutputEval {
            completion_rate,
            correctness_score: correctness,
            pass: completion_rate >= 0.70 && correctness >= 0.60,
        }
    }

    /// Layer 2 — Trace Eval（工具调用收敛性，24h 窗口）
    ///
    /// 核心洞察（来自 MC）：正常 Agent 使用多样化工具；
    /// 反复调用同一工具 = 陷入循环。三档阈值直接映射到三档行动，无歧义。
    pub async fn eval_trace(&self, agent_id: &AgentId) -> TraceEval {
        let calls = self.mcp_call_log.recent(agent_id, Duration::hours(24));
        let total = calls.len() as f64;
        let unique = calls.iter().map(|c| &c.tool_name)
            .collect::<HashSet<_>>().len() as f64;

        let ratio = if unique > 0.0 { total / unique } else { 1.0 };

        // 三档阈值与三档行动一一对应，消除 bool 标记与实际行动阈值不同步的歧义
        let looping_status = LoopingStatus::from_ratio(ratio);

        TraceEval {
            convergence_ratio: ratio,
            convergence_score: f64::min(1.0, 5.0 / ratio), // 以 SoftLoop 阈值 5.0 归一化
            looping_status,
        }
    }

    /// Layer 3 — Component Eval（工具可靠性）
    pub async fn eval_component(&self, agent_id: &AgentId) -> ComponentEval {
        let stats = self.mcp_call_log.tool_reliability(agent_id);
        ComponentEval {
            tool_success_rate: stats.success_rate,
            pass: stats.success_rate >= 0.80,
            unreliable_tools: stats.tools_below_threshold(0.80),
        }
    }

    /// Layer 4 — Drift Detection（行为漂移，与 4 周前基线对比）
    pub async fn eval_drift(&self, agent_id: &AgentId) -> DriftEval {
        let current  = self.collect_week_metrics(agent_id, 0).await;
        let baseline = self.collect_week_metrics_avg(agent_id, 1..=4).await;

        // 任意指标相对变化 > 10% → 漂移
        let threshold = 0.10;
        let drifts = vec![
            ("avg_tokens", current.avg_tokens, baseline.avg_tokens),
            ("tool_success_rate", current.tool_success_rate, baseline.tool_success_rate),
            ("task_completion_rate", current.completion_rate, baseline.completion_rate),
        ].into_iter()
        .filter(|(_, cur, base)| (cur - base).abs() / base.max(0.001) > threshold)
        .map(|(name, cur, base)| DriftIndicator {
            metric: name.to_string(),
            relative_change: (cur - base) / base,
        })
        .collect::<Vec<_>>();

        DriftEval { drifts: drifts.clone(), is_drifted: !drifts.is_empty() }
    }
}
```

```rust
/// 收敛状态枚举 — 阈值与行动一一对应，无二义性
pub enum LoopingStatus {
    /// ratio ≤ 5.0：正常，工具调用多样化
    Normal,
    /// 5.0 < ratio ≤ 10.0：软循环 → 触发 §2.6 循环检查点（压缩上下文）
    SoftLoop,
    /// ratio > 10.0：硬循环 → 强制中断 + §6.2 AnomalyEscalation 审批门控
    HardLoop,
}

impl LoopingStatus {
    pub fn from_ratio(ratio: f64) -> Self {
        if ratio > 10.0      { Self::HardLoop }
        else if ratio > 5.0  { Self::SoftLoop }
        else                 { Self::Normal }
    }

    pub fn required_action(&self) -> LoopAction {
        match self {
            Self::Normal   => LoopAction::None,
            Self::SoftLoop => LoopAction::TriggerCheckpoint,   // §2.6 循环检查点
            Self::HardLoop => LoopAction::ForceInterruptAndEscalate, // §6.2 审批门控
        }
    }
}
```

**收敛比（Layer 2）与循环检查点（§2.6）的协同**：
- `SoftLoop`（ratio 5~10）：触发 §2.6 的循环检查点——生成本轮执行摘要，压缩上下文，重置循环计数
- `HardLoop`（ratio > 10）：强制中断，触发 §6.2 的 AnomalyEscalation 审批门控

`[可观测]` 四层指标写入 `eval_runs` 表，提供 8 周历史趋势 API，进化评估器可读取漂移数据作为学习触发信号。

### 2.13 后台调度器架构 —— 来自 Mission Control

Legion 引擎的后台维护任务统一由调度器管理，采用 Mission Control 经验证的三原则：**防重入 + 分散初始延迟 + 动态启用**。

```rust
pub struct BackgroundScheduler {
    tasks:         Vec<ScheduledTask>,
    tick_interval: Duration,        // 60s 全局 tick
    config:        Arc<DynamicConfig>,  // 运行时读取，无需重启
}

pub struct ScheduledTask {
    id:                 &'static str,
    interval:           Duration,
    initial_delay:      Duration,  // 分散延迟，防启动时写入争用
    enabled_config_key: &'static str,
    running:            AtomicBool, // 防重入：上次未完成则跳过本次
    next_run:           AtomicI64,
}

impl BackgroundScheduler {
    pub async fn tick(&self) {
        let now = Utc::now().timestamp();
        for task in &self.tasks {
            if task.running.load(Ordering::SeqCst)           { continue; } // 防重入
            if now < task.next_run.load(Ordering::SeqCst)    { continue; } // 未到时间
            if !self.config.is_enabled(task.enabled_config_key) { continue; } // 动态关闭

            task.running.store(true, Ordering::SeqCst);
            // 在独立 task 中运行，不阻塞 tick 循环
            tokio::spawn(Self::run_task(task));
        }
    }
}
```

**Legion 调度任务清单**：

| 任务 ID | 间隔 | 初始延迟 | 职责 |
|---------|------|---------|------|
| `agent_heartbeat` | 5min | 5s | 离线检测 + 僵尸任务触发 |
| `task_dispatch` | 60s | 10s | inbox→assigned→派遣 |
| `quality_review` | 60s | 30s | Aegis 批量审核 |
| `stale_task_requeue` | 60s | 25s | 卡死任务回退 |
| `skill_sync` | 60s | 15s | 技能磁盘↔DB 同步 |
| `memory_fts_update` | 5min | 20s | FTS5 增量索引更新 |
| `eval_drift_detection` | 1h | 60s | Layer 4 漂移定期扫描（⚠️ 仅 Layer 4，Layer 2 收敛比检测为内循环实时，见 §2.4） |
| `evolution_report` | 24h | 120s | 日 Agent 成长报告 |
| `webhook_retry` | 60s | 30s | 失败 Webhook 退避重试 |
| `knowledge_gap_detect` | 24h | 次日 3 点 | Wiki 知识缺口扫描 |
| `auto_backup` | 24h | 次日 4 点 | DB 备份（默认禁用）|

> ⚠️ **`trust_score_cooldown` 任务已移除**：信任分冷却期采用懒惰求值（§2.11.2 `effective_score()` 在每次查询时实时检查 `cooldown_until > now()`），不需要后台任务清理。定时任务方案存在最长 1h 的状态延迟，且与懒惰求值并存会产生状态矛盾。

分散延迟设计：任务首次运行间隔至少 5s 错开，防止服务启动时多个任务同时访问 SQLite。

`[可治理]` `enabled_config_key` 对应管理 UI 中可实时切换的开关，无需重启服务。

`[可控风险]` `running: AtomicBool` 确保慢任务（如漂移检测、知识扫描）不会叠加执行导致内存爆炸。

<!-- @end-section -->

---

<!-- @section: wiki-engine-design -->
## 三、LLM Wiki 知识库引擎设计

### 3.1 设计目标回顾（来自 Legion.md §3.3）

LLM Wiki 是**人机共建、人机共读**的知识系统。关键需求：
- 5 步知识加工管道
- 混合检索（向量 + 全文 + 图谱）
- 严格的知识治理（版本/溯源/审核/冲突裁决）

### 3.2 知识存储引擎 —— 来自 Hermes FTS5 + 内容寻址 + Mission Control BM25/WikiLink

```sql
-- 来自 Hermes WAL + FTS5 模式
PRAGMA journal_mode = WAL;     -- 支持并发读写
PRAGMA synchronous = NORMAL;   -- 性能与安全平衡

-- 核心知识表
CREATE TABLE knowledge_chunks (
    id          TEXT PRIMARY KEY,           -- sha256: 内容寻址（来自 Evolver）
    doc_id      TEXT NOT NULL,
    content     TEXT NOT NULL,
    embedding   BLOB,                       -- 向量嵌入
    created_by  TEXT NOT NULL,              -- 溯源：人类用户 ID 或 Agent ID
    source_type TEXT NOT NULL,              -- 'human' | 'agent_task' | 'agent_learn'
    status      TEXT DEFAULT 'pending',     -- 'pending' | 'approved' | 'archived'
    quality_score REAL,
    created_at  INTEGER NOT NULL,
    version     INTEGER DEFAULT 1
);

-- 全文检索虚拟表定义见 §3.2b（BM25 + porter unicode61，path/title/content 三列）
-- 此处省略，避免与 §3.2b 规范定义冲突

-- 进化资产表（来自 Evolver Gene/Capsule 设计）
CREATE TABLE evolution_assets (
    asset_id    TEXT PRIMARY KEY,           -- sha256:
    asset_type  TEXT NOT NULL,              -- 'gene' | 'capsule' | 'event'
    content     TEXT NOT NULL,              -- JSON
    parent_id   TEXT,                       -- 进化链
    agent_id    TEXT NOT NULL,
    created_at  INTEGER NOT NULL
);

-- 不可变审计日志（来自 Evolver events.jsonl 思想）
CREATE TABLE audit_log (
    id          INTEGER PRIMARY KEY AUTOINCREMENT,
    event_type  TEXT NOT NULL,
    subject_id  TEXT NOT NULL,
    actor_id    TEXT NOT NULL,
    payload     TEXT,                       -- JSON
    created_at  INTEGER NOT NULL
    -- 无 UPDATE/DELETE 权限，仅 INSERT
);
```

### 3.2b FTS5 精细化检索 —— 来自 Mission Control

Mission Control 的 FTS5 实现在 Hermes 的基础上提供了更精细的权重和查询语法：

```sql
-- 来自 Mission Control memory-search.ts（porter unicode61 = 英文词干 + Unicode）
CREATE VIRTUAL TABLE knowledge_fts USING fts5(
    path, title, content,
    content=knowledge_chunks,
    tokenize='porter unicode61'
);
```

**BM25 权重设计**（标题重要性 5x）：

```rust
pub struct KnowledgeFtsEngine {
    db: Arc<Db>,
}

impl KnowledgeFtsEngine {
    pub fn search(&self, query: &str, top_k: usize) -> Vec<FtsResult> {
        let safe_query = self.sanitize_query(query);

        // 标题权重 5.0，内容/路径权重 1.0
        // bm25(knowledge_fts, 1.0, 5.0, 1.0) 对应 (path, title, content)
        self.db.query(
            "SELECT path, title, snippet(knowledge_fts, 2, '<mark>', '</mark>', '...', 40),
                    bm25(knowledge_fts, 1.0, 5.0, 1.0) as rank
             FROM knowledge_fts
             WHERE knowledge_fts MATCH ?
             ORDER BY rank
             LIMIT ?",
            [&safe_query, &top_k.to_string()],
        )
    }

    /// 查询安全化：将普通词转换为前缀匹配，保留 FTS5 操作符
    fn sanitize_query(&self, raw: &str) -> String {
        // 已含 FTS5 操作符（AND/OR/NOT/NEAR/"phrases"）→ 直通
        if raw.contains(" AND ") || raw.contains(" OR ") || raw.contains('"')
            || raw.contains("NEAR") { return raw.to_string(); }

        // 单词转前缀匹配：`legion` → `legion*`
        // 多词转隐式 AND：`legion agent` → `legion* AND agent*`
        raw.split_whitespace()
            .map(|w| format!("{}*", w))
            .collect::<Vec<_>>()
            .join(" AND ")
    }
}
```

**WikiLink 图谱**：知识库文件间通过 `[[目标文件]]` 语法相互链接，构成连接图谱。图谱持久化于 SQLite `wiki_links` 邻接表，支持跨重启的增量维护，避免每次启动全量扫描。

```sql
-- WikiLink 邻接表（SQLite，替代内存 HashMap，支持跨重启增量维护）
CREATE TABLE wiki_links (
    source_path  TEXT NOT NULL,   -- 来源文件路径（MEMORY_PATH 相对路径）
    target_path  TEXT NOT NULL,   -- 目标文件路径（WikiLink 解析后规范化）
    updated_at   INTEGER DEFAULT (unixepoch()),
    PRIMARY KEY (source_path, target_path)
);
CREATE INDEX idx_wiki_links_target ON wiki_links(target_path);
```

```rust
pub struct WikiLinkGraph {
    db: Arc<Db>,  // SQLite 连接（WAL 模式，线程安全）
}

impl WikiLinkGraph {
    /// 提取文件内容中的所有 WikiLink
    pub fn extract_links(content: &str) -> Vec<String> {
        // 匹配 [[链接目标]] 或 [[链接目标|显示文本]]
        let re = Regex::new(r"\[\[([^\]|]+)(?:\|[^\]]*)?\]\]").unwrap();
        re.captures_iter(content)
            .map(|c| c[1].to_string())
            .collect()
    }

    /// 更新文件的出链（保存文件时调用，替换该 source 的全部旧记录）
    pub async fn upsert_links(&self, source: &str, targets: &[String]) -> Result<()> {
        let mut conn = self.db.acquire().await?;
        conn.execute("DELETE FROM wiki_links WHERE source_path = ?1", [source]).await?;
        for target in targets {
            conn.execute(
                "INSERT OR IGNORE INTO wiki_links(source_path, target_path) VALUES(?1, ?2)",
                [source, target.as_str()],
            ).await?;
        }
        Ok(())
    }

    /// gap-detect：检测知识缺口（断链 + 被引用但缺失的目标）
    pub async fn detect_gaps(&self) -> Result<Vec<KnowledgeGap>> {
        // 被引用的 target_path 在 source_path 集合中不存在 → 知识缺口
        let rows = self.db.fetch_all(
            "SELECT target_path, COUNT(*) AS ref_count
             FROM wiki_links
             WHERE target_path NOT IN (SELECT DISTINCT source_path FROM wiki_links)
             GROUP BY target_path
             ORDER BY ref_count DESC",
            [],
        ).await?;
        Ok(rows.iter().map(|r| KnowledgeGap {
            missing:         r.get("target_path"),
            priority_score:  0.3 + 0.15 * r.get::<f64>("ref_count"),
        }).collect())
    }

    /// consolidate：识别 Hub 节点（出度 ≥ 均值 × 2）
    pub async fn find_hubs(&self) -> Result<Vec<HubNode>> {
        let rows = self.db.fetch_all(
            "WITH degree AS (
               SELECT source_path, COUNT(*) AS deg FROM wiki_links GROUP BY source_path
             ),
             avg_deg AS (SELECT AVG(deg) AS avg FROM degree)
             SELECT source_path, deg FROM degree, avg_deg
             WHERE deg >= avg_deg.avg * 2
             ORDER BY deg DESC",
            [],
        ).await?;
        Ok(rows.iter().map(|r| HubNode {
            path:   r.get("source_path"),
            degree: r.get("deg"),
        }).collect())
    }

    /// BFS 邻居遍历（混合检索入口，最多 depth 跳，SQL 递归 CTE）
    pub async fn bfs_neighbors(&self, roots: &[String], depth: u32) -> Result<Vec<String>> {
        // 递归 CTE 实现 BFS，depth 参数限制递归深度
        let placeholders = roots.iter().enumerate()
            .map(|(i, _)| format!("?{}", i + 1))
            .collect::<Vec<_>>().join(", ");
        let sql = format!(
            "WITH RECURSIVE bfs(path, level) AS (
               SELECT source_path, 0 FROM wiki_links WHERE source_path IN ({placeholders})
               UNION
               SELECT wl.target_path, b.level + 1
               FROM bfs b JOIN wiki_links wl ON wl.source_path = b.path
               WHERE b.level < ?{depth_param}
             )
             SELECT DISTINCT path FROM bfs",
            placeholders = placeholders,
            depth_param = roots.len() + 1,
        );
        let mut params: Vec<&str> = roots.iter().map(|s| s.as_str()).collect();
        let depth_str = depth.to_string();
        params.push(&depth_str);
        let rows = self.db.fetch_all(&sql, params).await?;
        Ok(rows.iter().map(|r| r.get("path")).collect())
    }
}
```

`[可观测]` WikiLink 图谱支持知识缺口扫描（gap-detect）和中心节点识别（consolidate），定期由调度器运行并写入健康报告。

### 3.3 混合检索引擎 —— 来自 Hermes + Legion.md

Legion.md §3.3.4 要求三路融合检索。**架构约束**：全部基于 SQLite，不引入 Neo4j 等外部图数据库——WikiLink 图（§3.2b）已用 SQLite 邻接表实现，维护零额外运维成本。

```rust
pub struct HybridRetriever {
    vector_store: Arc<VectorStore>,      // sqlite-vss（内嵌，无额外服务）
    fts_engine: Arc<KnowledgeFtsEngine>, // SQLite FTS5（§3.2b 实现）
    link_graph: Arc<WikiLinkGraph>,      // SQLite 邻接表（§3.2b 实现，替代 Neo4j）
}

impl HybridRetriever {
    pub async fn retrieve(&self, query: &RetrievalQuery) -> Vec<KnowledgeChunk> {
        // 三路并行检索（全部走 SQLite，无跨进程调用）
        let (vector_results, fts_results, graph_results) = tokio::join!(
            self.vector_store.search(&query.embedding, TopK(20)),
            self.fts_engine.search(&query.text, TopK(20)),
            // WikiLink 图遍历：从实体节点出发做 BFS（深度 2 跳）
            async { self.link_graph.bfs_neighbors(&query.entities, Depth(2)) },
        );

        // 融合排序（RRF: Reciprocal Rank Fusion）
        self.fuse_and_rerank(vector_results, fts_results, graph_results, &query)
    }
}
```

### 3.4 渐进式知识加载 —— 来自 Hermes

Hermes 的技能渐进式加载直接应用于 Wiki 知识检索：

```rust
// 来自 Hermes 的渐进式技能加载模式
pub struct KnowledgeLoader {
    metadata_cache: HashMap<String, KnowledgeMetadata>,  // 轻量元数据缓存
}

impl KnowledgeLoader {
    // 阶段 1: 只加载元数据索引（低成本，Agent 初始化时执行）
    pub async fn load_metadata_index(&self, domain: &str) -> Vec<KnowledgeMetadata> {
        // 元数据：触发条件 + 简短描述（不含完整内容）
        self.fetch_metadata(domain).await
    }

    // 阶段 2: Agent 需要时按需加载完整内容
    pub async fn load_full_content(&self, chunk_id: &str) -> KnowledgeChunk {
        self.fetch_with_cache(chunk_id).await
    }
}
```

### 3.5 知识治理管道

完整实现 Legion.md §3.3.5 的治理机制：

```rust
pub struct KnowledgeGovernance {
    version_manager: VersionManager,
    conflict_detector: ConflictDetector,
    review_workflow: ReviewWorkflow,
    quality_scorer: QualityScorer,
}

// AI 产出的知识必须经过审核
pub enum KnowledgeStatus {
    PendingReview,      // AI 生成，等待审核
    UnderReview,        // 审核中
    Approved,           // 已批准，进入正式库
    Conflicting,        // 与已有知识冲突，待裁决
    Archived,           // 已归档（低价值 / 过期）
    Rejected,           // 已拒绝
}

impl KnowledgeGovernance {
    pub async fn submit_ai_knowledge(
        &self,
        content: &str,
        agent_id: &AgentId,
    ) -> KnowledgeEntry {
        // AI 生成知识 → 默认 PendingReview 状态
        let entry = KnowledgeEntry {
            status: KnowledgeStatus::PendingReview,
            created_by: KnowledgeSource::Agent(agent_id.clone()),
            ..Default::default()
        };

        // 冲突检测：新知识与已有知识是否矛盾
        if let Some(conflict) = self.conflict_detector.detect(&entry).await {
            entry.status = KnowledgeStatus::Conflicting;
            self.escalate_conflict(conflict).await;
        }

        entry
    }
}
```

`[可治理]` 所有 AI 产出的知识默认进入待审核队列，只有经过人类审核或高权限 Agent 交叉验证后才进入正式库。

<!-- @end-section -->

---

<!-- @section: adapter-design -->
## 四、Adapter 层设计

### 4.1 设计目标回顾（来自 Legion.md §4.4）

Adapter 层的关键原则是**控制平面与执行平面分离**：Legion 是控制平面，不直接运行 Agent；各 Adapter 是执行平面，通过标准接口与控制平面通信。

### 4.2 六类 Adapter 实现策略

| Adapter 类型 | 实现参考 | 关键设计点 |
|------------|---------|----------|
| **LLM API Adapter** | DeepSeek-TUI `ModelProvider` + Hermes `ProviderTransport` | 统一 SSE 流管道 + NormalizedResponse |
| **CLI Agent Adapter** | Claw Code 进程管理 + Hermes `Process Adapter` | 子进程生命周期 + 标准输出解析 |
| **MCP Adapter** | DeepSeek-TUI MCP crate + Hermes MCP 集成 | JSON-RPC stdio/HTTP + 工具代理 |
| **HTTP Webhook Adapter** | Hermes 平台适配器模式 | BasePlatformAdapter + 重试队列 |
| **Process Adapter** | Claw Code bash 执行 + 权限检查 | bash_validation + 沙盒策略 |
| **Composite Adapter** | 来自 Legion.md 设计 | 多 Adapter 组合 + 统一 RunResult |

### 4.3 标准 Adapter 接口

```rust
// Legion.md §4.4.3 的标准三模块接口
pub trait ServerAdapter: Send + Sync {
    // 必须实现：执行 Agent 心跳
    async fn execute(&self, ctx: ExecutionContext) -> RunResult;
    // 必须实现：环境健康检查
    async fn diagnose(&self) -> EnvironmentDiag;
    // 必须实现：解析执行成本
    fn parse_usage(&self, output: &RunOutput) -> CostData;
}

// CLI Agent Adapter 示例（参考 Claw Code 进程管理）
pub struct CliAgentAdapter {
    cli_path: PathBuf,
    permission_enforcer: Arc<PermissionEnforcer>,  // 来自 Claw Code
    sandbox: SandboxMode,                          // 来自 DeepSeek-TUI
    process_manager: Arc<ProcessManager>,
}

impl ServerAdapter for CliAgentAdapter {
    async fn execute(&self, ctx: ExecutionContext) -> RunResult {
        // 权限检查（来自 Claw Code permission_enforcer）
        self.permission_enforcer.check_before_spawn(&ctx)?;

        // 在沙盒中启动进程（来自 DeepSeek-TUI ExternalSandbox）
        let child = self.process_manager.spawn_sandboxed(
            &self.cli_path,
            &self.build_args(&ctx),
            self.sandbox,
        ).await?;

        // 流式读取输出（来自 DeepSeek-TUI SSE 管道思想）
        self.stream_output(child).await
    }
}
```

### 4.4 平台适配器（多入口支持）

参考 Hermes 的 19 平台适配器架构，Legion 支持多种接入方式：

```rust
// 来自 Hermes BasePlatformAdapter 模式
pub trait PlatformAdapter: Send + Sync {
    async fn on_message(&self, event: MessageEvent) -> SendResult;
    async fn send_message(&self, target: &DeliveryTarget, content: &Content) -> bool;
    fn platform_hint(&self) -> &str;
}

// 统一投递目标语法（来自 Hermes）
pub enum DeliveryTarget {
    Origin,                                    // 回复到消息来源
    Specific { platform: String, id: String }, // "telegram:123456"
    Thread { platform: String, id: String, thread: String }, // "telegram:123456:thread_id"
    Broadcast { scope: BroadcastScope },       // 广播全体/组
}
```

### 4.5 技能安全扫描器 —— 来自 Mission Control

Legion 的技能系统（`legion-skills`，对应 Crate 层 Layer 1）在接受外部技能时必须经过安全扫描，防止恶意技能被 Agent 加载执行。

#### 4.5.1 安装前静态扫描

```rust
pub struct SkillSecurityScanner {
    rules: Vec<SecurityRule>,
}

pub enum SecurityLevel { Critical, Warning, Info }

pub struct SecurityRule {
    pub id:      &'static str,
    pub level:   SecurityLevel,
    pub pattern: Regex,        // 或自定义检测逻辑
    pub message: &'static str,
}

pub struct ScanResult {
    pub status:   SecurityStatus,  // Safe / Warning / Critical
    pub findings: Vec<ScanFinding>,
}

impl SkillSecurityScanner {
    /// 13 条规则（来自 Mission Control skill-registry.ts）
    pub fn scan(&self, skill_content: &str) -> ScanResult {
        let mut findings = vec![];

        for rule in &self.rules {
            if rule.matches(skill_content) {
                findings.push(ScanFinding {
                    rule_id: rule.id,
                    level: rule.level,
                    message: rule.message.to_string(),
                });
            }
        }

        let status = if findings.iter().any(|f| matches!(f.level, SecurityLevel::Critical)) {
            SecurityStatus::Critical   // 阻断安装
        } else if findings.iter().any(|f| matches!(f.level, SecurityLevel::Warning)) {
            SecurityStatus::Warning    // 允许安装，标记可见
        } else {
            SecurityStatus::Safe
        };

        ScanResult { status, findings }
    }
}
```

**13 条安全规则**（来自 Mission Control 实战积累）：

| 规则 ID | 级别 | 检测目标 |
|---------|------|---------|
| `prompt-injection-system` | Critical | "ignore previous instructions" 类提示注入 |
| `prompt-injection-role` | Critical | 角色操控 / 安全绕过指令 |
| `shell-exec-dangerous` | Critical | `rm -rf` / `curl\|bash` / `eval()` 组合 |
| `data-exfiltration` | Critical | 将数据发送到外部的指令 |
| `path-traversal` | Critical | `../` 路径穿越 |
| `ssrf-internal-network` | Critical | 访问 localhost / 内网 IP |
| `ssrf-metadata-endpoint` | Critical | AWS/GCP/Azure 元数据端点（169.254.x.x）|
| `credential-harvesting` | Warning | 硬编码 API key / secret / password |
| `obfuscated-content` | Warning | base64 / 十六进制编码内容 |
| `hidden-instructions` | Warning | HTML 注释中的可疑指令 |
| `excessive-permissions` | Warning | sudo / chmod 777 等高危权限操作 |
| `network-fetch` | Info | 包含外部 HTTP URL 引用 |
| `suspicious-imports` | Info | Legion 扩展：导入已知恶意库 |

#### 4.5.2 技能安装流程

```rust
pub struct SkillInstaller {
    scanner: SkillSecurityScanner,
    registry_client: RegistryClient,  // ClawdHub / skills.sh
    db: Arc<Db>,
}

impl SkillInstaller {
    pub async fn install_from_registry(
        &self,
        skill_id: &str,
        workspace_id: WorkspaceId,
    ) -> Result<InstalledSkill, InstallError> {
        // Step 1: 拉取内容
        let content = self.registry_client.fetch(skill_id).await?;

        // Step 2: 内容完整性校验（SHA-256，ClawdHub 支持）
        self.registry_client.verify_hash(skill_id, &content).await?;

        // Step 3: 安全扫描
        let scan = self.scanner.scan(&content);
        match scan.status {
            SecurityStatus::Critical => {
                // 阻断安装，写入安全事件日志
                self.log_security_event(SecurityEventType::MaliciousSkillBlocked, skill_id);
                return Err(InstallError::SecurityBlocked(scan.findings));
            }
            SecurityStatus::Warning => {
                // 允许安装，但标记风险
                tracing::warn!("技能 {} 包含 Warning 级安全问题: {:?}", skill_id, scan.findings);
            }
            SecurityStatus::Safe => {}
        }

        // Step 4: 写入磁盘（workspace/skills/{name}/SKILL.md）
        let path = self.write_to_disk(skill_id, &content, workspace_id)?;

        // Step 5: 更新数据库（skills 表，security_status 标记）
        let skill = self.db.upsert_skill(&SkillRecord {
            name: skill_id.to_string(),
            path,
            security_status: scan.status,
            content_hash: sha256(&content),
            registry_slug: Some(skill_id.to_string()),
            workspace_id,
        })?;

        Ok(skill)
    }
}
```

**磁盘↔DB 双向同步策略**（disk wins）：

```
磁盘有、DB 无 → INSERT
磁盘 SHA-256 变化 → UPDATE（重新扫描安全）
DB 有、磁盘消失 → DELETE（仅限无 registry_slug 的本地技能；注册表安装的保留）
```

`[可治理]` Critical 安全问题阻断安装并写入安全事件，管理员可查看被拦截的技能和原因。

`[可控风险]` 安全扫描规则版本化（可更新规则库）；扫描在安装时和每次磁盘同步时运行，确保已安装技能的安全状态始终最新。

<!-- @end-section -->

---

<!-- @section: org-comm-design -->
## 五、组织架构与通讯机制设计

### 5.1 四级组织数据模型

直接落地 Legion.md §4.1.1 的组织架构，结合 DeepSeek-TUI 的 SQLite Schema 设计：

```sql
-- 公司级（多公司隔离）
CREATE TABLE companies (
    id          TEXT PRIMARY KEY,
    name        TEXT NOT NULL,
    global_budget_cny REAL NOT NULL,
    governance_rules TEXT,       -- JSON
    status      TEXT DEFAULT 'active'
);

-- 部门级
CREATE TABLE departments (
    id          TEXT PRIMARY KEY,
    company_id  TEXT NOT NULL REFERENCES companies(id),
    name        TEXT NOT NULL,
    monthly_budget_cny REAL,
    data_permissions TEXT         -- JSON
);

-- 团队级（四级层次的第三级：公司→部门→团队→Agent）
CREATE TABLE teams (
    id            TEXT PRIMARY KEY,
    department_id TEXT NOT NULL REFERENCES departments(id),
    name          TEXT NOT NULL,
    lead_agent_id TEXT REFERENCES agents(id),  -- 团队负责人 Agent（循环引用，延迟约束）
    monthly_budget_cny REAL,
    status        TEXT DEFAULT 'active'
);

-- Agent（四级：公司/部门/项目组/Agent）
CREATE TABLE agents (
    id          TEXT PRIMARY KEY,
    company_id  TEXT NOT NULL,
    department_id TEXT REFERENCES departments(id),
    team_id     TEXT REFERENCES teams(id),
    reports_to  TEXT REFERENCES agents(id),  -- 严格汇报链
    role        TEXT NOT NULL,
    persona     TEXT NOT NULL,               -- JSON
    model_policy TEXT NOT NULL,              -- JSON（MaaS 配置）
    evolution   TEXT NOT NULL,               -- JSON（学习配置）
    budget      TEXT NOT NULL,               -- JSON（预算配置）
    status      TEXT DEFAULT 'active'
);

-- 原子任务锁（防止重复执行，来自 Legion.md 通讯机制）
CREATE TABLE task_locks (
    task_id     TEXT PRIMARY KEY,
    agent_id    TEXT NOT NULL,
    locked_at   INTEGER NOT NULL,
    expires_at  INTEGER NOT NULL
);
```

### 5.2 通讯协议实现

Legion.md §4.1.2 的核心协议流程：

**Crate 归属**：`legion-core`（Layer 3）。AgentCoordinator 是**任务编排协调层**，只负责任务的生命周期管理（锁定、路由决策、审计链、状态流转）——不包含 LLM 调用循环本身，后者由 §2.4 的 `AgentRuntime` 负责。

两者职责不重叠：
- `AgentCoordinator` 持有 `maas_client` 仅用于**路由决策**（选哪个模型），然后把决策结果（`RoutingDecision`）传给 `AgentRuntime`
- `AgentRuntime` 持有 `maas_client` 仅用于**实际调用**（stream()），不做路由

```rust
// legion-core（Layer 3）：任务编排协调层
// 职责边界：任务生命周期管理（锁、路由决策、审计、状态流转）
// 不包含 LLM 调用循环（见 §2.4 AgentRuntime）
pub struct AgentCoordinator {
    task_scheduler: Arc<TaskScheduler>,
    task_lock: Arc<AtomicTaskLock>,          // 原子锁，防重复执行
    intelligent_router: Arc<IntelligentRouter>, // 路由决策（§1.7），不调用 LLM
    agent_runtime: Arc<Mutex<AgentRuntime>>, // 执行层（Mutex：run_task 需要 &mut self 维护 cycle_count）
    audit_log: Arc<ImmutableAuditLog>,       // 不可篡改审计链
}

impl AgentCoordinator {
    pub async fn execute_heartbeat(&self, agent: &Agent) -> HeartbeatResult {
        // 步骤1: 身份确认
        self.verify_identity(agent).await?;

        // 步骤2: 获取待办任务（inbox 状态任务）
        let tasks = self.task_scheduler.fetch_pending(agent.id()).await?;

        for task in tasks {
            // 步骤3: 原子锁定任务（防止重复执行）
            let lock = self.task_lock.try_acquire(&task.id, agent.id()).await?;
            if lock.is_none() { continue; }  // 已被其他实例锁定

            // 步骤4: 路由决策（由 IntelligentRouter 完成，不调用 LLM）
            // AgentCoordinator 只做决策，AgentRuntime 做实际调用
            let routing = self.intelligent_router.route(&RoutingContext {
                agent: Some(agent.def.clone()),
                task: Some(task.clone()),
                budget: agent.budget_ctx(),
                ..Default::default()
            }).await?;

            // 步骤5: 执行工作（交给 AgentRuntime 负责 LLM 循环）
            // Mutex 获取：run_task 需要 &mut self（ContextCompressor.cycle_count 跨迭代持久化）
            let result = self.agent_runtime.lock().await.run_task(task.clone(), routing).await;

            // 步骤6: 更新任务状态（in_progress → quality_review）
            self.task_scheduler.advance_status(&task.id, &result).await;

            // 步骤7: 写入审计链（不可篡改）
            self.audit_log.append(AuditEntry::from_task_result(agent, &task, &result)).await;

            // 步骤8: 释放任务锁
            lock.release().await;
        }
    }
}
```

`[可观测]` 每次心跳的完整链路（身份→锁定→路由决策→执行→状态流转）写入不可篡改审计链。

`[可治理]` 通讯权限遵循组织架构，越级通讯或私自广播被架构层拦截。

`[可控风险]` 原子任务锁杜绝重复执行；通讯频率限制防止消息风暴。

<!-- @end-section -->

---

<!-- @section: workflow-dsl -->
## 六、工作流 DSL 执行引擎

### 6.1 8 种原语的执行引擎

Legion.md §4.3.2 定义的 8 种流程原语在执行引擎中的实现：

```rust
pub enum WorkflowStep {
    Sequence { id: String, agent_role: String, action: String },
    Parallel {
        id: String,
        branches: Vec<WorkflowStep>,
        /// 单分支超时（None = 无限等待，生产环境应明确设置）
        timeout_secs: Option<u64>,
        /// 部分失败策略（默认 FailFast）
        on_partial_failure: ParallelFailurePolicy,
    },
    Branch { id: String, condition: String, true_branch: Box<WorkflowStep>, false_branch: Option<Box<WorkflowStep>> },
    Loop { id: String, body: Box<WorkflowStep>, exit_condition: String, max_iterations: u32 },
    Join { id: String, wait_for: Vec<String> },
    ApprovalGate { id: String, approver: ApproverSpec, condition: Option<String> },
    EventWait { id: String, event_type: String, timeout_secs: Option<u64> },
    ErrorHandler { id: String, body: Box<WorkflowStep>, on_failure: FailureAction },
}

/// 并行分支的部分失败语义
pub enum ParallelFailurePolicy {
    /// 任意分支失败立即取消其余分支（默认，适合强依赖场景）
    FailFast,
    /// 等待所有分支完成，收集成功结果；失败分支记录到 errors（适合独立任务批量执行）
    CollectAll,
    /// 至少 n 个分支成功即视为整体成功（适合冗余执行场景）
    Quorum { min_success: usize },
}

pub struct WorkflowExecutor {
    agent_pool: Arc<AgentPool>,
    approval_service: Arc<ApprovalService>,
    event_bus: Arc<EventBus>,
    audit_log: Arc<ImmutableAuditLog>,
}

impl WorkflowExecutor {
    pub async fn execute(&self, step: &WorkflowStep, ctx: &WorkflowContext) -> StepResult {
        match step {
            WorkflowStep::Parallel { branches, timeout_secs, on_partial_failure, .. } => {
                // 并行执行所有分支，带超时控制
                let branch_futures = branches.iter().map(|b| {
                    let fut = self.execute(b, ctx);
                    async move {
                        match timeout_secs {
                            Some(secs) => tokio::time::timeout(
                                Duration::from_secs(*secs), fut
                            ).await.unwrap_or(Err(StepError::Timeout)),
                            None => fut.await,
                        }
                    }
                });

                let results = futures::future::join_all(branch_futures).await;

                // 依据部分失败策略处理结果
                match on_partial_failure {
                    ParallelFailurePolicy::FailFast => {
                        // 有任意失败 → 整体失败
                        let first_err = results.iter().find(|r| r.is_err());
                        if let Some(Err(e)) = first_err {
                            return StepResult::Failed(e.clone());
                        }
                        StepResult::parallel_results(results)
                    }
                    ParallelFailurePolicy::CollectAll => {
                        // 收集所有成功结果，失败分支记录 warning
                        StepResult::partial_results(results)
                    }
                    ParallelFailurePolicy::Quorum { min_success } => {
                        let success_count = results.iter().filter(|r| r.is_ok()).count();
                        if success_count >= *min_success {
                            StepResult::quorum_reached(results, *min_success)
                        } else {
                            StepResult::quorum_failed(results, *min_success)
                        }
                    }
                }
            }

            WorkflowStep::ApprovalGate { approver, condition, id } => {
                // 检查是否满足触发条件
                if let Some(cond) = condition {
                    if !ctx.evaluate(cond) {
                        return StepResult::Skipped;
                    }
                }
                // 暂停流程，等待人类决策（来自 Legion.md 7 大审批门控）
                let ticket = self.approval_service.create_ticket(id, approver, ctx).await;
                self.approval_service.wait_for_decision(ticket).await?;
                StepResult::Approved
            }

            WorkflowStep::ErrorHandler { body, on_failure, .. } => {
                match self.execute(body, ctx).await {
                    Ok(r) => Ok(r),
                    Err(e) => match on_failure {
                        FailureAction::Retry { max, back_to } => self.retry(body, ctx, *max).await,
                        FailureAction::Escalate => self.escalate_to_human(&e, ctx).await,
                        FailureAction::Fallback { step } => self.execute(step, ctx).await,
                    }
                }
            }
            // ... 其他原语
        }
    }
}
```

### 6.2 七大审批门控

完整实现 Legion.md §4.3.4 的治理节点：

```rust
pub enum ApprovalTrigger {
    Recruitment,        // Agent 请求创建下属
    BudgetOverrun,      // 预计成本超阈值
    StrategyReview,     // 管理层方案产出
    ModelUpgrade,       // 请求使用更高等级模型
    AnomalyEscalation,  // 连续失败或异常行为（含 HardLoop）
    QualityGate,        // 交付物不达标
    WikiKnowledge,      // Agent 写入知识库
}
```

#### 6.2.1 HardLoop → AnomalyEscalation 完整响应链

`HardLoop`（收敛比 > 10，见 §2.12）是最严重的运行时异常之一，表明 Agent 陷入工具调用死循环，若不干预将持续消耗预算直至任务超时。完整响应链如下：

**触发方**：`AgentRuntime.run_task()` 内循环（§2.4）—— 每次工具执行完成后立即检测，不依赖后台调度器；`eval_drift_detection` 后台任务（§2.13）仅负责 Layer 4 漂移定期扫描，不承担 HardLoop 实时检测职责。

```
AgentRuntime 每次 execute_tools() 后检测 ratio > 10（HardLoop）
  │  返回 TaskResult::AnomalyDetected → AgentCoordinator 接收并启动响应链
  │
  ▼ 1. AgentCoordinator 发送强制中断信号给 TaskScheduler
       TaskScheduler.force_interrupt(task_id)
  │
  ▼ 2. TaskScheduler 原子更新任务状态
       tasks.status = Suspended（原子 CAS，防竞态）
       tasks.suspended_at = now()
       tasks.suspend_reason = "HardLoop: ratio={ratio:.1}"
       tasks.suspend_snapshot = AgentContext 快照（用于恢复上下文）
  │
  ▼ 3. AgentCoordinator 确认执行已停止
       AgentRuntime 已通过返回 TaskResult::AnomalyDetected 退出循环（无需额外 cancel）
       若有并发子任务仍在运行：CancellationToken::cancel()，最多等待 tool_timeout_secs
  │
  ▼ 4. 创建审批工单
       ApprovalService.create_ticket(
           trigger: AnomalyEscalation,
           task_id, agent_id,
           evidence: { ratio, tool_call_log: 最近 50 次调用 },
           approver: 任务创建者 / 管理员
       )
  │
  ▼ 5. SSE 广播前端通知（管理员立即知晓）
       EventBus.broadcast("task.suspended", { task_id, reason })
```

**审批决策处理**：

```rust
pub async fn handle_anomaly_decision(
    &self,
    ticket_id: TicketId,
    decision: ApprovalDecision,
) -> Result<()> {
    match decision {
        ApprovalDecision::Approved { note } => {
            // 恢复执行：从 suspend_snapshot 重建上下文
            // 可选：注入人类指导（降低循环风险）
            let snapshot = self.task_store.get_suspend_snapshot(ticket_id).await?;
            let mut ctx = AgentContext::restore_from_snapshot(snapshot);
            if let Some(guidance) = note {
                ctx.inject_human_guidance(guidance);  // 将人类说明注入系统提示末尾
            }
            // 重置收敛比计数器（防止立即再触发）
            ctx.reset_loop_counter();
            // 原子更新状态 Suspended → InProgress
            self.task_store.update_status(ticket_id, TaskStatus::InProgress).await?;
            // 重新提交给调度器执行
            self.task_scheduler.requeue(ticket_id, ctx).await?;
        }
        ApprovalDecision::Denied { reason } => {
            // 拒绝恢复：任务终止为 Failed
            self.task_store.update_status_with_reason(
                ticket_id,
                TaskStatus::Failed,
                &format!("Suspended task denied by approver: {reason}"),
            ).await?;
            // 通知 Agent 任务已终止
            self.event_bus.broadcast("task.failed", { ticket_id, reason }).await;
        }
    }
    Ok(())
}
```

**关键设计约束**：

| 约束 | 实现方式 |
|------|---------|
| Suspended 只由 TaskScheduler 写入 | `tasks.status = Suspended` 仅在 `force_interrupt()` 内执行，API 层拒绝直接写 |
| 审批必须有人类参与 | `ApprovalService` 不提供自动批准接口（不可编程调用 approve） |
| 恢复后防二次触发 | `reset_loop_counter()` 重置 `mcp_call_log` 窗口计数，宽限 5 分钟 |
| 上下文完整性 | `suspend_snapshot` 序列化整个 `AgentContext`（含消息历史压缩版本） |
| 审计可追溯 | `ImmutableAuditLog` 记录：触发时间、ratio 值、工具调用证据、审批人、决策 |

`[可控风险]` HardLoop 响应链是 Legion 中**唯一一个人类强制介入点**（其他门控可配置为半自动），确保失控 Agent 不会在无人知晓的情况下持续消耗资源。

<!-- @end-section -->

---

<!-- @section: three-principles-matrix -->
## 七、三原则落地矩阵

### 7.1 每个设计决策的三原则标注

| 设计组件 | 来源 | `[可观测]` | `[可治理]` | `[可控风险]` |
|---------|------|----------|----------|------------|
| MaaS 传输层 NormalizedResponse | Hermes | 路由溯源字段 model_used / provider_id | 传输层可替换 | 统一错误分类学 |
| SSE 流管道 | DeepSeek-TUI | 流事件类型安全，无 any | — | 8MB 背压控制 |
| 连接健康状态机 | DeepSeek-TUI | 状态变更实时可见 | 管理员可手动触发探针 | 三态防崩溃，自动恢复 |
| 错误分类 + 恢复 | Hermes | 分类结果记录日志 | 恢复动作可配置覆盖 | 熔断 + 自动降级 |
| 认知内核优先级编排 | Legion 原创 | 上下文组装过程可追溯 | 安全红线优先级硬编码 | P0 安全红线不可覆盖 |
| 循环检查点 | DeepSeek-TUI | 摘要内容可查 | 管理员可调整 cycle_length | 防上下文窗口耗尽 |
| 批准策略 × 沙盒 | DeepSeek-TUI | 每次工具调用记录批准决策 | RBAC 角色覆盖策略 | SandboxMode 物理隔离 |
| 工具看门狗 | Hermes | before/after 调用记录 | 可配置检测规则 | 精确重复失败检测 |
| 权限执行器 | Claw Code | 每次权限检查记录 | 权限边界可配置 | 越权自动拦截 |
| 三层信号提取 | Evolver | 信号来源可追溯 | 信号规则可调整 | 修复循环检测 |
| Gene/Capsule/Event | Evolver | SHA-256 内容寻址 | Gene 可审核/删除 | 爆炸半径 + 金丝雀 |
| 固化流程 | Evolver | EvolutionEvent 不可变链 | 验证命令可配置 | 自动 Git 回滚 |
| AVOID 线索（α）+ Gene 六元组 | Evolver + 论文 | 每个 Gene 来源轨迹可追溯（σ/ι/Δ）| 管理员可注入/删除 AVOID 线索；审核 Gene 草稿 | 单焦点原则；token 预算 ≤ 230；α 字段强制非空 |
| 蒸馏算子 ψ + GEP 六阶段周期 | Evolver + 论文 | 六阶段状态快照完整记录（scan→solidify）| 管理员在 validate 前人工门控；动态调整 Gene 注入上限 | 修复驱动触发（非定时）；失败模式检测为主 |
| 上下文压缩 | Hermes | 压缩历史可查 | 压缩策略可配置 | 防上下文爆炸 |
| Crate 分层架构 | DeepSeek-TUI | 依赖关系清晰 | 层间接口标准化 | 循环依赖编译拒绝 |
| 不可变审计日志 | Evolver | 每个操作有审计记录 | 仅 INSERT 权限 | 无法回溯修改 |
| 原子任务锁 | Legion.md | 锁定状态可见 | 管理员可强制释放 | 防重复执行 |
| FTS5 + WAL SQLite | Hermes + MC | 全文检索支持；BM25 标题权重 5x | 版本控制 + 回滚 | WAL 并发安全 |
| 知识治理流程 | Legion.md | 知识状态流转透明 | AI 知识必须人工审核 | 冲突检测 + 权限分级 |
| WikiLink 图谱 | Mission Control | gap-detect/consolidate 可查 | 知识缺口定期扫描告警 | 孤立文件检测防知识腐烂 |
| Aegis 质量门控 | Mission Control | 每轮审核结果可查；SSE 广播 | done 终态强制需要 Aegis 批准 | 3 轮上限防无限循环消耗预算 |
| 七状态任务机 | Mission Control | 状态转换历史可追溯 | 原子 claim 防竞态；硬约束拦截绕过 | 僵尸任务 5 次恢复上限防无限；Suspended 仅 TaskScheduler 写入 |
| Trust Score | Mission Control | 每个 Agent 的安全行为历史可查 | 低信任分自动降级派遣 | 冷却期防刷分；严重事件阻断高风险任务 |
| 四层 Eval | Mission Control | 收敛比 / 漂移 / 完成率 8 周趋势 | 循环检测触发强制中断 | Layer 2 实时循环检测，不等任务结束 |
| 后台调度器 | Mission Control | 任务运行状态 / 上次结果可查 | 动态启用：管理 UI 实时切换 | 防重入 + 分散延迟防启动雪崩 |
| 技能安全扫描 | Mission Control | 扫描结果和被拦截技能可审计 | Critical 级阻断安装，管理员可查原因 | 13 条规则覆盖注入/SSRF/路径穿越 |

### 7.2 "避免的坑"总结

从五个项目的教训中提炼，Legion 应避免：

1. **AIAgent 上帝类（来自 Hermes）**：Hermes 的 `run_agent.py` 达 14,123 行。Legion 从设计之初就按 Crate 分层，`legion-core` 是最复杂的 crate，但职责边界清晰：只负责 Agent 主循环编排，不包含具体工具实现。

2. **TUI crate 膨胀（来自 DeepSeek-TUI）**：DeepSeek-TUI 的 `tui` crate 有 230+ 文件，导致编译时间极长。Legion 的前端层不共用一个 crate，按功能域拆分：`legion-app-web`、`legion-app-cli`、`legion-app-mobile`。

3. **parse_sse_chunk 参数膨胀（来自 DeepSeek-TUI）**：7 个 `&mut` 参数。Legion 的 `SseParser` 封装为有状态对象（见 §1.3）。

4. **JSONL 查询性能（来自 Evolver）**：Evolver 的 JSONL 文件随事件增长查询性能下降。Legion 使用 SQLite + FTS5，同时保留 JSONL 格式作为不可变审计日志导出格式。

5. **提示缓存破坏（来自 Hermes）**：对话中期修改系统提示/工具集会破坏 Anthropic 提示缓存，显著增加成本。Legion 的认知内核在 `system_prompt_locked` 后冻结系统提示。

6. **RelayInfo / Session 上帝对象（通用教训）**：按职责拆分上下文结构体：`RequestContext`、`ProviderContext`、`BillingContext`、`StreamContext`，而非一个不断膨胀的 `ExecutionContext`。

7. **核心逻辑混淆（来自 Evolver）**：Evolver 核心 2.9MB 混淆，阻碍社区贡献和安全审计。Legion 完全开源，核心引擎代码可读。

8. **任务外键用字符串（来自 Mission Control）**：MC 的 `tasks.assigned_to` 存储 Agent 名称字符串而非 ID，Agent 改名后所有历史任务出现悬空引用，无法使用 DB 外键约束。Legion 使用不可变 UUID 标识 Agent，名称只用于显示。

9. **两套定时系统职责混淆（来自 Mission Control）**：MC 的 Node.js 调度器（内部运维）和 OpenClaw Cron（业务定时）职责不清晰，UI 混合展示令用户困惑。Legion 明确区分：`BackgroundScheduler`（对用户不可见的系统运维）和 `WorkflowCron`（用户可配置的业务定时任务）。

10. **MCP 工具数量膨胀（来自 Mission Control）**：MC 的 35 个 MCP 工具中存在功能重叠（`mc_agent_costs` 与 `mc_costs_by_agent` 等）。Legion 遵循工具正交原则：每个工具做一件事，通过参数覆盖变体，目标控制在 20 个以内。

11. **任务不能绕过质量门（来自 Mission Control）**：任务可以被代码直接写入 done 是个设计漏洞。Legion 在 **API 层 + 状态机层** 双重硬约束：未通过 Aegis 审核的任务 PUT done 返回 403。

<!-- @end-section -->

---

<!-- @section: implementation-roadmap -->
## 八、实现优先级路线图

### Phase 1：MaaS 基础（最高优先级）

1. `legion-protocol` — 数据结构定义（无依赖，先行）
2. `legion-config` — 配置加载
3. `legion-model` — SQLite Schema（参考 DeepSeek-TUI 4 表结构 + MC 七状态任务机）
4. `legion-maas` — 传输层 + 路由 + 配额（参考 Hermes NormalizedResponse）
5. SSE 流管道（参考 DeepSeek-TUI，封装为 `SseParser` 状态对象）

### Phase 2：Agent 运行时 + 质量管控

6. `legion-execpolicy` — 批准策略 × 沙盒（直接参考 DeepSeek-TUI）
7. `legion-tools` — 工具注册（参考 Hermes 自注册模式）
8. `legion-core` — Agent 主循环（参考 Hermes + Claw Code，含工具看门狗）
9. `legion-memory` — 三层记忆（工作/情景/能力）
10. Aegis 质量门控（参考 MC，§2.10）—— **必须与主循环同期完成**
11. `BackgroundScheduler`（参考 MC，§2.13）—— 防重入 + 动态启用
12. 七状态任务机（参考 MC，§2.10.1）—— API 层 + 状态机层双重保护

### Phase 3：安全体系 + 技能 Hub

13. Trust Score 信任评分（参考 MC，§2.11）—— 事件驱动实时重算
14. 四层 Eval 框架（参考 MC，§2.12）—— 行为健康监控
15. 技能安全扫描器（参考 MC，§4.5）—— 13 条规则 + Critical/Warning 分级
16. `legion-skills` — 技能加载 + 磁盘↔DB 同步（disk wins）

### Phase 4：学习引擎与进化

17. `legion-learning` — 三层信号提取（参考 Evolver `signals.js`，§2.7.1）
18. Gene 六元组资产模型（m/u/π/α/c/v，arXiv:2604.15097）—— **α 字段强制非空，token 预算 ≤ 230**
19. Capsule 六元组 + EvolutionEvent 九元组（含 Ed25519 immutable_seal，§2.7.2）
20. 蒸馏算子 ψ（`ψ(s)` 单次轨迹 + `ψ(ℋ)` 历史归纳，§2.7.6）
21. GEP 六阶段形式化周期（scan→signal→intent→mutate→validate→solidify，§2.7.5）
22. 固化流程（参考 Evolver `solidify.js` + Ed25519 签名，§2.7.3）
23. `legion-evolution` — 进化评估器（5 维度），读取 Phase 3 四层 Eval 数据作为输入（§2.9）

### Phase 5：LLM Wiki 与全平台集成

21. `legion-wiki` — 知识管道 + FTS5 BM25 + WikiLink 图谱（参考 MC §3.2b）
22. 知识治理（版本/溯源/审核/冲突裁决）
23. 知识健康诊断 8 维度 + gap-detect（参考 MC）
24. `legion-gateway` — REST API + WebSocket + SSE 事件流
25. 前端 Web 应用（组织架构可视化 + 工作流画布 + 安全审计面板）

<!-- @end-section -->

---

<!-- @section: related -->
## 相关文档

- [[Legion|Legion V3.0 项目方案]]
- [[../analysis/claude/01-overview|Claw Code 项目架构总览]]
- [[../analysis/deepseek-tui/01-overview|DeepSeek-TUI 项目架构总览]]
- [[../analysis/deepseek-tui/07-insights|DeepSeek-TUI 设计洞察与 Legion 参考]]
- [[../analysis/evolver/01-overview|Evolver 项目架构总览]]
- [[../analysis/evolver/06-evolver-insights|Evolver 洞察与 Legion 参考]]
- [[../analysis/hermes/01-overview|Hermes Agent 项目架构总览]]
- [[../analysis/hermes/06-hermes-insights|Hermes 洞察与 Legion 参考]]
- [[../analysis/mission-control/index|Mission Control 分析索引]]
- [[../analysis/mission-control/03-orchestration-scheduler|Mission Control 编排引擎与调度器]]
- [[../analysis/mission-control/05-security-eval-framework|Mission Control 安全框架与 Agent 评估]]
- [[../analysis/mission-control/07-insights|Mission Control 设计洞察与 Legion 参考]]

<!-- @end-section -->
