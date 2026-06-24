---
id: "design-agent-runtime-009"
title: "Agent 运行时引擎技术设计"
aliases: ["Agent 技术设计", "运行时引擎", "认知内核", "对话状态机"]
type: "design"
category: "design/analysis/claude"
tags: ["agent", "runtime", "conversation", "cognitive-core", "memory", "compact"]
version: "1.0.0"
created: "2026-05-03"
updated: "2026-05-03"
author: "jxncyjq"
status: "review"
parent: "design-maas-scheduling-008"
children: ["design-tool-skill-010"]
related_docs:
  - id: "analysis-clawcode-rust-002"
    relation: "references"
    path: "./02-rust-crates-analysis.md"
  - id: "analysis-clawcode-flows-005"
    relation: "references"
    path: "./05-architecture-flows.md"
---

<!-- @section: overview -->
# Agent 运行时引擎 — 技术设计

**文档版本**：V1.0
**编写日期**：2026年5月
**文档性质**：技术设计
**参考来源**：claw-code runtime crate（43 模块）架构分析

---

## 目录

- [一、设计依据](#一、设计依据)
- [二、认知内核设计](#二、认知内核设计)
- [三、对话状态机](#三、对话状态机)
- [四、执行循环](#四、执行循环)
- [五、会话压缩机制](#五、会话压缩机制)
- [六、Token 预算管理](#六、token-预算管理)
- [七、经验记忆系统](#七、经验记忆系统)
- [八、三原则落地](#八、三原则落地)

---

## 一、设计依据

### 1.1 参考来源

| 参考模块 | 核心能力 | Legion 映射 |
|----------|----------|-------------|
| `conversation.rs` | 对话循环状态机、API 请求组装、工具调用循环 | Agent 执行循环 |
| `prompt.rs` | 系统提示组装、指令文件发现与加载 | 认知内核上下文组装 |
| `session.rs` | 消息持久化、版本化序列化、文件轮转 | 工作记忆 + 情景记忆 |
| `compact.rs` + `summary_compression.rs` | 自动压缩、摘要生成、多层合并 | 记忆蒸馏 |
| `usage.rs` | Token 计数、费用估算、累计追踪 | 成本管控 |
| `bootstrap.rs` | 12 阶段启动流程（`BootstrapPhase` 枚举） | Agent 启动初始化 |
| `git_context.rs` | 上下文自动收集 | 组织上下文感知 |

### 1.2 设计目标

Agent 引擎是 Legion 的核心运行时，让每个智能体具备：

1. **完整的执行上下文**（认知内核）
2. **稳定的对话循环**（状态机驱动）
3. **自动的记忆管理**（三层记忆 + 压缩蒸馏）
4. **持续的学习进化**（反馈学习 + 反思 + 观察）

---

## 二、认知内核设计

### 2.1 上下文组装

参考 claw-code 的 `prompt.rs` 系统提示组装模式。每个 Agent 心跳启动时，认知内核负责组装完整的执行上下文：

```
执行上下文 = 角色人设
           + 当前任务指令
           + 目标传导链
           + 相关经验记忆
           + 挂载技能（工具）
           + 组织上下文
           + 约束规则
```

### 2.2 上下文分层与优先级

参考 claw-code 的指令文件优先级设计（`MAX_INSTRUCTION_FILE_CHARS = 4000`，`MAX_TOTAL_INSTRUCTION_CHARS = 12000`）：

```yaml
context_layers:
  - layer: 1    # 最高优先级
    name: "安全红线"
    source: "agent.constraints.safety_rules"
    max_chars: 2000
    conflict_resolution: "始终优先，不可被覆盖"

  - layer: 2
    name: "角色核心定义"
    source: "agent.persona"
    max_chars: 3000

  - layer: 3
    name: "当前任务与目标链"
    source: "task.objective + task.goal_chain"
    max_chars: 4000

  - layer: 4
    name: "相关经验记忆"
    source: "memory.episodic.relevant"
    max_chars: 5000

  - layer: 5
    name: "技能声明"
    source: "agent.skills.active"
    max_chars: 2000

  - layer: 6
    name: "组织上下文"
    source: "org.team_structure + org.communication_rules"
    max_chars: 3000

  - layer: 7
    name: "预算与配额信息"
    source: "agent.budget + maas.quota"
    max_chars: 1000

total_max_chars: 20000
```

### 2.3 上下文冲突解决

```rust
struct ContextAssembler {
    layers: Vec<ContextLayer>,
}

impl ContextAssembler {
    fn assemble(&self, agent: &Agent, task: &Task) -> String {
        let mut used_chars = 0;
        let mut output = String::new();

        for layer in &self.layers {
            let content = layer.resolve(agent, task);
            let truncated = self.truncate_to_budget(&content, layer.max_chars, &mut used_chars);

            // 安全红线层不可截断
            if layer.name == "安全红线" && truncated.len() < content.len() {
                panic!("安全红线内容超长，拒绝截断");
            }

            output.push_str(&truncated);
        }

        output
    }

    fn truncate_to_budget(&self, content: &str, max: usize, used: &mut usize) -> String {
        let remaining = 20_000usize.saturating_sub(*used);
        let budget = max.min(remaining);
        if content.len() <= budget {
            *used += content.len();
            content.to_string()
        } else {
            let truncated = content[..budget].to_string();
            *used += budget;
            format!("{truncated}\n... (内容已截断，剩余部分无法注入)")
        }
    }
}
```

---

## 三、对话状态机

### 3.1 状态定义

参考 claw-code `conversation.rs` 的 `ConversationRuntime::run_turn()` 完整状态机模型：

```
  ┌─────────────────────────────────────────────────────────────┐
  │                    Agent 执行状态机                          │
  │                                                             │
  │  Idle (等待触发)                                             │
  │    │                                                        │
  │    ├── 定时心跳 → WakeUp                                    │
  │    ├── 任务委派 → TaskReceived                              │
  │    ├── @提及   → Mentioned                                  │
  │    ├── 管道触发 → PipelineTriggered                         │
  │    └── 手动唤醒 → ManualWakeUp                              │
  │        │                                                    │
  │        ▼                                                    │
  │  Bootstrapping (加载配置/记忆/技能/工具)                      │
  │    │                                                        │
  │    ▼                                                        │
  │  ContextReady (上下文就绪)                                    │
  │    │                                                        │
  │    ▼                                                        │
  │  ┌─────────────┐                                            │
  │  │ Turn Loop    │ ←────────────────────┐                    │
  │  │             │                      │                    │
  │  │ ApiRequesting ──► ProcessingResponse │                    │
  │  │       ▲                │            │                    │
  │  │       │                ▼            │                    │
  │  │       │    [有工具调用] ToolExecuting │                    │
  │  │       │                │            │                    │
  │  │       │    [执行成功]   │            │                    │
  │  │       └────────────────┘            │                    │
  │  │                                     │                    │
  │  │    [无工具调用]                       │                    │
  │  └─────┬───────────────────────────────┘                    │
  │        ▼                                                    │
  │  AutoCompacting (自动压缩检查)                                │
  │    │                                                        │
  │    ├── 触发压缩 → Compacting → CompactionDone                │
  │    └── 不触发  → TurnEnd                                     │
  │        │                                                    │
  │        ▼                                                    │
  │  ExperienceExtracting (提取经验)                              │
  │    │                                                        │
  │    ▼                                                    │
  │  Idle (回归等待)                                             │
  └─────────────────────────────────────────────────────────────┘
```

### 3.2 状态机实现

```rust
enum AgentState {
    Idle,
    Bootstrapping,
    ContextReady,
    TurnLoop {
        sub_state: TurnSubState,
        iteration: u32,
    },
    AutoCompacting,
    ExperienceExtracting,
}

enum TurnSubState {
    ApiRequesting,
    ProcessingResponse,
    ToolExecuting {
        pending_tools: Vec<ToolCall>,
        current_index: usize,
    },
}

const MAX_TURN_ITERATIONS: u32 = 25;  // 防止无限循环

impl AgentRuntime {
    async fn run_turn(&mut self, trigger: Trigger) -> Result<TurnSummary> {
        // 1. 自动压缩预检查
        self.maybe_auto_compact().await?;

        // 2. 组装上下文
        let context = self.cognitive_core.assemble(&self.agent, &self.current_task);

        // 3. 推入用户消息
        self.session.push_user_message(&trigger.into_message());

        // 4. Turn 循环
        let mut iteration = 0;
        loop {
            if iteration >= MAX_TURN_ITERATIONS {
                return Err(AgentError::MaxIterationsExceeded);
            }

            // 4a. API 请求组装
            let request = self.build_api_request(&context);

            // 4b. 流式调用
            let events = self.maas.stream(request).await?;

            // 4c. 构建助手消息
            let (message, tool_calls) = self.build_assistant_message(events);

            // 4d. 记录用量
            self.usage_tracker.record(&message.usage);

            // 4e. 持久化消息
            self.session.push_assistant_message(message);

            // 4f. 无工具调用 → 结束
            if tool_calls.is_empty() {
                break;
            }

            // 4g. 工具执行循环
            for tool in &tool_calls {
                iteration += 1;
                let result = self.execute_tool(tool).await;
                self.session.push_tool_result(result);
            }
        }

        // 5. Turn 后自动压缩
        self.maybe_auto_compact().await;

        // 6. 提取经验
        let experience = self.extract_experience().await;

        // 7. 持久化
        self.session.persist();

        Ok(TurnSummary {
            iterations: iteration,
            usage: self.usage_tracker.latest_turn.clone(),
            experience,
        })
    }
}
```

---

## 四、执行循环

### 4.1 工具调用 Dispatch

参考 claw-code 的两层工具调用链（`conversation.rs` → `tools/lib.rs`）：

```
API 响应 → ToolUse {id, name, input}
  │
  ├── 第 0 层：Pre-Hook 检查
  │   run_pre_tool_use_hook(tool_name, input)
  │   → allowed / denied / modified_input
  │
  ├── 第 1 层：权限判定
  │   permission_policy.authorize_with_context(tool_name, input, context)
  │   → Allow / Deny{reason}
  │
  ├── 第 2 层：工具执行
  │   tool_registry.execute(tool_name, input)
  │   → ToolResult {output, is_error}
  │
  └── 第 3 层：Post-Hook 处理
      run_post_tool_use_hook(tool_name, result)
      → 副作用处理 / 通知
```

### 4.2 异步工具执行

```rust
trait ToolExecutable: Send + Sync {
    fn name(&self) -> &str;
    fn description(&self) -> &str;
    fn input_schema(&self) -> Value;
    async fn execute(&self, input: Value, context: &ExecutionContext) -> ToolResult;
}

struct ExecutionContext {
    agent_id: String,
    task_id: String,
    workspace_root: PathBuf,
    budget_remaining: f64,
    allowed_operations: Vec<OperationKind>,
}
```

---

## 五、会话压缩机制

### 5.1 触发条件

参考 claw-code `compact.rs` 的自动压缩逻辑：

```rust
const DEFAULT_AUTO_COMPACTION_INPUT_TOKENS: u64 = 100_000;

impl AgentRuntime {
    fn should_compact(&self) -> bool {
        // 条件 1：累计 input_tokens 超过阈值
        if self.usage_tracker.cumulative.input_tokens < self.compaction_threshold {
            return false;
        }
        // 条件 2：有足够多的旧消息可压缩
        let compactable = self.session.messages.len() - PRESERVE_RECENT_COUNT;
        compactable > MIN_COMPACTABLE_COUNT
    }
}

const PRESERVE_RECENT_COUNT: usize = 4;     // 保留最近 4 条
const MIN_COMPACTABLE_COUNT: usize = 8;      // 至少 8 条旧消息才压缩
```

### 5.2 压缩流程

参考 claw-code `compact.rs` 的 6 步压缩流程：

```
步骤 1 — 预检查：不需要压缩直接返回

步骤 2 — 计算边界：
  compacted_prefix_len = 上次 compacted summary 起始位置
  keep_from = messages.len() - PRESERVE_RECENT_COUNT

步骤 3 — 保护 ToolUse/ToolResult 配对：
  若保留部分第一条是 ToolResult → 向前查找配对 ToolUse
  → 边界前移到配对 ToolUse 之前
  → 保证不切断 assistant(tool_use) + user(tool_result) 原子对

步骤 4 — 生成摘要：
  从被移除消息提取：
  - 消息统计（user N, assistant M, tool K）
  - 工具名称列表
  - 最近 3 个用户请求摘要
  - 待处理工作（包含 "todo"/"next"/"pending" 的消息）
  - 关键文件引用（.go/.rs/.py/.md 等）
  - 关键时间线（每条消息单行摘要，截断到 160 字符）

步骤 5 — 构建 continuation 消息：
  System: "本会话从之前的对话继续..."
    + formatted_summary
    + "最近的消息已完整保留。"

步骤 6 — 重组 session：
  compacted_messages = vec![continuation_message]
  compacted_messages.extend(preserved)
  session.messages = compacted_messages
  session.record_compaction(summary, removed_count)
```

### 5.3 多层压缩合并

参考 claw-code `compact.rs` 的 `merge_compact_summaries`：

```rust
fn merge_compact_summaries(previous: &CompactionSummary, current: &CompactionSummary) -> String {
    format!(
        "此前压缩的内容：\n{}\n\n新压缩的内容：\n{}",
        previous.summary,
        current.summary,
    )
}
```

### 5.4 摘要文本二次压缩

参考 claw-code `summary_compression.rs` 的优先级预算控制：

| 优先级 | 内容 | 预算分配 |
|--------|------|----------|
| 0 | 核心细节行（"Scope:"、"Current work:"、"Pending work:"） | 不截断 |
| 1 | 段落标题（以冒号结尾） | 优先保留 |
| 2 | 列表项（"- " 开头） | 次优保留 |
| 3 | 其余行 | 低优先 |

- `max_chars`: 默认 1200
- `max_lines`: 默认 24
- 每行截断到 160 字符

---

## 六、Token 预算管理

### 6.1 累计追踪

参考 claw-code 的 `UsageTracker`：

```rust
struct UsageTracker {
    latest_turn: TokenUsage,
    cumulative: TokenUsage,
    turns: u32,
}

impl UsageTracker {
    fn record(&mut self, usage: TokenUsage) {
        self.latest_turn = usage;
        self.cumulative.input_tokens += usage.input_tokens;
        self.cumulative.output_tokens += usage.output_tokens;
        self.cumulative.cache_creation_input_tokens += usage.cache_creation_input_tokens;
        self.cumulative.cache_read_input_tokens += usage.cache_read_input_tokens;
        self.turns += 1;
    }
}
```

### 6.2 模型 Token 上限感知

```rust
fn max_tokens_for_model(model: &str) -> u32 {
    match model {
        m if m.contains("opus") => 200_000,
        m if m.contains("gpt-4") => 128_000,
        m if m.contains("deepseek") => 64_000,
        _ => 32_000,
    }
}
```

### 6.3 指令文件截断

参考 claw-code 的 `prompt.rs` 指令文件大小限制：

```rust
const MAX_INSTRUCTION_FILE_CHARS: usize = 4_000;    // 每个文件
const MAX_TOTAL_INSTRUCTION_CHARS: usize = 12_000;    // 总计
```

---

## 七、经验记忆系统

### 7.1 三层记忆架构

对应 Legion 项目方案中的三层记忆模型：

```rust
// 工作记忆 - 参考 claw-code session.rs 的消息持久化
struct WorkingMemory {
    messages: Vec<ConversationMessage>,
    intermediate_state: BTreeMap<String, Value>,
    session_id: String,
}

// 情景记忆 - 跨任务持久化的关键经验
struct EpisodicMemory {
    entries: Vec<EpisodicEntry>,
}

struct EpisodicEntry {
    id: String,
    task_summary: TaskSummary,
    key_decisions: Vec<Decision>,
    outcome: Outcome,
    lessons_learned: Vec<String>,
    feedback: Option<Feedback>,
    created_at: DateTime,
    retention_days: u32,
}

// 能力记忆 - 长期沉淀的抽象化知识
struct CapabilityMemory {
    patterns: Vec<BehaviorPattern>,
    best_practices: Vec<BestPractice>,
    domain_knowledge: Vec<DomainKnowledge>,
    evolution_level: u32,
}
```

### 7.2 经验提取

参考 claw-code `compact.rs` 的消息提取逻辑：

```rust
impl AgentRuntime {
    async fn extract_experience(&self) -> Experience {
        // 1. 判断任务是否有值得记录的经验
        if self.is_trivial_task() {
            return Experience::None;
        }

        // 2. 提取关键决策点
        let decisions = self.extract_key_decisions();

        // 3. 评估结果
        let outcome = self.evaluate_outcome();

        // 4. 生成经验教训
        let lessons = self.distill_lessons(&decisions, &outcome);

        // 5. 写入情景记忆
        Experience::Significant(EpisodicEntry {
            task_summary: self.summarize_task(),
            key_decisions: decisions,
            outcome,
            lessons_learned: lessons,
            feedback: self.pending_feedback.take(),
        })
    }
}
```

---

## 八、三原则落地

### 8.1 可观测

| 观测点 | 内容 |
|--------|------|
| 决策追溯 | 每个 turn 的工具调用、API 请求、权限判定日志 |
| 执行看板 | 当前状态、迭代次数、Token 消耗实时展示 |
| 压缩日志 | 每次压缩的摘要、移除消息数、触发原因 |
| 记忆审计 | Agent 的情景记忆列表、进化等级变化 |

### 8.2 可治理

| 治理能力 | 实现方式 |
|----------|----------|
| 暂停 Agent | `AgentState` 模型中注入 `Suspended` 状态 |
| 手动注入记忆 | 管理员直接写入 `EpisodicMemory` |
| 删除不当记忆 | 按记忆 ID 或按时间范围删除 |
| 重置进化状态 | 清空 `CapabilityMemory` |
| 覆盖工具允许列表 | 运行时注入 `ToolRestriction` |

### 8.3 可控风险

| 风险 | 控制 |
|------|------|
| 无限循环 | `MAX_TURN_ITERATIONS = 25` |
| 上下文窗口溢出 | 自动压缩 + 飞行前检查 |
| 错误记忆污染 | AI 生成的记忆标记"待审核"，需人工批准 |
| 敏感信息泄露 | 压缩摘要过滤敏感字段（密钥、令牌） |
<!-- @end-section -->

<!-- @section: related -->
## 相关文档

- [[02-rust-crates-analysis|Rust Crate 功能模块分析]] — runtime crate 参考
- [[05-architecture-flows|架构流程分析]] — 对话状态机参考
- [[design-maas-scheduling-008|MaaS 模型调度层技术设计]] — 上游依赖
- [[design-tool-skill-010|工具与技能系统设计]] — 下游依赖
<!-- @end-section -->
