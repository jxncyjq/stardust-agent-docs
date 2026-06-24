---
id: "spec-agent-cognitive-core-000"
title: "CognitiveCore 组件规范"
aliases: ["CognitiveCore规范", "认知内核", "cognitive-core-spec"]
type: "spec"
category: "design/architecture/agent_components"
tags: ["component-spec", "agent-engine", "context", "assembly", "legion"]
version: "0.1.0"
created: "2026-05-09"
updated: "2026-05-09"
author: "jxncyjq"
status: "draft"
parent: "spec-agent-component-registry-000"

component_id: "A00"
layer: "L0"
depends_on: []
optional_deps:
  - "A40"   # MemoryProvider — 缺失时 P5 相关经验记忆返回空，其余 6 要素正常工作
  - "A30"   # SkillSystem — 缺失时 P6 挂载技能返回空列表
conflicts_with: []
required_by:
  - "A01"   # AgentRuntime 持有 CognitiveCore 所有权
assembly_profiles:
  - minimal
  - standard
  - enterprise
---

<!-- @section: overview -->
# CognitiveCore 组件规范

## 1. 组件定位

`CognitiveCore` 是 Agent 的**上下文组装核心**，负责将 7 要素（角色人设、任务指令、目标传导链、经验记忆、技能、组织上下文、约束规则）按优先级编排为一个结构化执行上下文，供 LLM 调用使用。

它不执行 LLM 调用（由 `AgentRuntime` 负责），也不管理任务状态（由 `AgentCoordinator` 负责）。职责边界：**从输入要素生成系统提示和消息历史初始状态**。

```
AgentRuntime
    │
    │ 持有所有权
    ▼
CognitiveCore
    │
    ├── MemoryProvider(A40)   → P5 相关经验记忆（TopK=5）
    ├── SkillSystem(A30)      → P6 挂载技能
    └── ContextCompressor(A03)→ 上下文压缩（压缩窗口触发时）
```

**读者**：实现 Agent 认知内核的工程师、需要理解上下文组装机制的架构师。

<!-- @end-section -->

<!-- @section: interface -->
---

## 2. 接口定义

```go
// CognitiveCore 是 Agent 上下文组装核心。
// 每个 AgentRuntime 实例持有一个 CognitiveCore 的所有权（非 Arc 共享）。
// 并发安全：CognitiveCore 内部含 AtomicBool，部分方法需要 &mut self。
type CognitiveCore interface {
    // AssembleContext 按 7 要素优先级编排执行上下文。
    // 首次调用后设置 system_prompt_locked=true，后续调用直接使用缓存的 P0/P1。
    // 返回 AssembledContext，包含系统提示、消息历史初始状态和可用工具列表。
    AssembleContext(req *TaskRequest) (*AssembledContext, error)

    // ForceCheckpoint 委托给内部 ContextCompressor 强制执行循环检查点。
    // 由 AgentRuntime 在 SoftLoop 触发时调用（忽略 80% 阈值，立即压缩）。
    // 需要 &mut self，因为会修改 ContextCompressor 的消息历史。
    ForceCheckpoint(history *MessageHistory) (*CheckpointResult, error)

    // SystemPromptHash 返回当前系统提示的哈希值（用于 Anthropic prompt cache 命中检测）。
    // 在 system_prompt_locked=true 后值不再变化。
    SystemPromptHash() uint64

    // IsLocked 返回系统提示是否已锁定（首次 AssembleContext 后为 true）。
    IsLocked() bool
}
```

<!-- @end-section -->

<!-- @section: data-types -->
---

## 3. 数据类型定义

### 3.1 输入

```go
// TaskRequest 是 AgentCoordinator 传给 AgentRuntime 的任务请求。
// 在整个任务执行期间不可变（由 AgentCoordinator 在调用前填充完毕）。
type TaskRequest struct {
    Task        *Task          // 任务实体（含 description, type, priority）
    OrgContext  *OrgContext    // 组织上下文（团队位置、协作者）
    GoalChain   []string       // 目标传导链（为什么做，从上级 Agent 或用户传递）
    AgentDef    *AgentDef      // Agent 定义（角色人设、约束规则、能力声明）
}

// AgentDef 是 Agent 的静态定义，构造时注入，对话期间不可变。
type AgentDef struct {
    ID          string
    Role        AgentRole      // 角色枚举（ceo / architect / backend_developer 等）
    Persona     string         // 角色人设（自然语言描述）
    Constraints []string       // 安全红线约束规则（P0，不可覆盖）
    ModelPolicy ModelPolicy    // 模型策略（max_model_tier 等）
    MaxCycleLength int         // CognitiveCore 传给 ContextCompressor 的最大检查点轮次
}

// OrgContext 是 Agent 在组织架构中的位置上下文。
type OrgContext struct {
    TeamID      string
    TeamName    string
    ReportsTo   string         // 上级 Agent ID
    Peers       []string       // 同级 Agent ID 列表
    Collaborators []string     // 当前任务协作者
}
```

### 3.2 输出

```go
// AssembledContext 是 AssembleContext() 的输出，直接传给 LLM 调用。
type AssembledContext struct {
    SystemPrompt  string        // 组装后的系统提示（P0+P1+P2+P3 拼接）
    InitMessages  []Message     // 初始消息列表（P4 任务指令 + P5 经验记忆注入）
    AvailableTools []ToolSpec   // P6：本次任务可用的工具规格列表
    PromptHash    uint64        // 系统提示哈希，用于 Anthropic 提示缓存命中检测
}

// CheckpointResult 是强制检查点的输出。
type CheckpointResult struct {
    Summary     string    // 本轮执行摘要（由轻量模型生成）
    CompressedAt time.Time
    TokensBefore int
    TokensAfter  int
}
```

### 3.3 七要素优先级

```go
// ContextPriority 定义 7 要素的编排优先级（P0 最高，P6 最低）。
type ContextPriority int

const (
    P0_SecurityConstraints ContextPriority = iota // 安全红线（永远最高，不可覆盖）
    P1_PersonaRole                                // 角色人设（锁定后不重建）
    P2_OrgContext                                 // 组织上下文（团队位置、协作者）
    P3_GoalChain                                  // 目标传导链（为什么做）
    P4_TaskInstruction                            // 当前任务指令
    P5_EpisodicMemory                             // 相关经验记忆（TopK=5 检索）
    P6_MountedSkills                              // 挂载技能（工具规格）
)
```

<!-- @end-section -->

<!-- @section: behavior -->
---

## 4. 行为规范

### 4.1 AssembleContext 执行逻辑

```
首次调用（system_prompt_locked = false）：

  Step 1. 从 AgentDef.Constraints 提取安全红线 → P0 片段
  Step 2. 从 AgentDef.Persona 提取角色人设 → P1 片段
  Step 3. 从 req.OrgContext 提取组织上下文 → P2 片段
  Step 4. 从 req.GoalChain 提取目标传导链 → P3 片段
  Step 5. 拼接 P0+P1+P2+P3 为系统提示（system_prompt）
  Step 6. 计算 system_prompt_hash（FNV-64 或 xxHash）
  Step 7. 设置 system_prompt_locked = true（原子写，AtomicBool::store(true, Release)）

  Step 8. 构造初始消息列表：
    a. 将 req.Task.Description 作为首条 user 消息（P4）
    b. 调用 MemoryProvider.prefetch(task) → 取 TopK=5 相关情景记忆注入（P5）
  Step 9. 调用 SkillSystem.load_for_task(task) → 生成可用工具规格列表（P6）

  返回 AssembledContext

后续调用（system_prompt_locked = true）：
  → P0/P1 直接从缓存读取，不重建
  → P2/P3/P4 仍使用新 req 中的值（组织上下文和任务可变）
  → P5/P6 重新检索（记忆和技能可动态更新）
```

**P0 不可覆盖保证**：安全红线（P0）在系统提示中总是排列最前，LLM 优先处理；同时框架层不提供任何参数允许跳过 P0 的拼接，纯结构保证（非依赖 LLM 理解）。

### 4.2 提示缓存稳定性保证（system_prompt_locked）

- `system_prompt_locked` 使用 `AtomicBool`，保证通过 `&self` 即可写入（无需 `&mut self`）
- 首次 `AssembleContext()` 完成后设为 `true`，对话期间不再重建 P0/P1
- 目的：防止系统提示哈希在对话中途变动，破坏 Anthropic 提示缓存（prompt cache break 会导致成本飙升）
- **注意**：任务之间（不同 run_task() 调用）系统提示允许不同；锁定仅在单次任务生命周期内有效

### 4.3 ForceCheckpoint 委托机制

```
ForceCheckpoint(history) 调用时：
  → 委托给内部 ContextCompressor.force_checkpoint(history)
  → 强制执行第 4 层（CycleCheckpoint）：
    a. 用轻量模型生成本轮执行摘要（cycle_briefing）
    b. 将 history 替换为 [摘要消息]，重置 cycle_count
    c. 返回 CheckpointResult
  → 调用方（AgentRuntime）使用返回的压缩后 history 继续循环
```

> `ForceCheckpoint` 需要 `&mut self` 是因为修改了 `ContextCompressor.message_history`。`CognitiveCore` 的其他方法（`AssembleContext`、`SystemPromptHash`、`IsLocked`）均为 `&self`。

### 4.4 冲突解决规则

当 7 要素内容存在冲突时（如 P4 任务指令要求做某操作，P0 安全红线禁止）：
- **高优先级片段始终生效**：P0 > P1 > P2 > P3 > P4 > P5 > P6
- 冲突不阻断组装（不抛错），记录 `ContextConflict` 警告日志
- 管理员可通过 `agent_def.constraints` 审阅冲突的来源

<!-- @end-section -->

<!-- @section: errors -->
---

## 5. 错误定义

```go
var (
    // ErrAssembleContextFailed 上下文组装失败（如 MemoryProvider 超时但非可选降级）。
    ErrAssembleContextFailed = errors.New("cognitive_core: context assembly failed")

    // ErrSystemPromptTooLarge 系统提示超过模型上下文窗口的 30%。
    // CognitiveCore 不直接截断，而是返回此错误让 AgentRuntime 决策。
    ErrSystemPromptTooLarge = errors.New("cognitive_core: system prompt exceeds 30% context limit")
)
```

<!-- @end-section -->

<!-- @section: observability -->
---

## 6. 可观测性

| 事件 | 日志级别 | 说明 |
|------|---------|------|
| 首次 AssembleContext 完成 | INFO | 记录 prompt_hash、system_prompt 字节数、各 P 要素来源 |
| system_prompt_locked 设置 | DEBUG | 记录锁定时间戳 |
| P5 记忆检索结果 | DEBUG | 记录检索到的情景记忆数量、相似度分数 |
| P6 技能加载结果 | DEBUG | 记录加载的技能列表和来源（SkillSource） |
| ContextConflict 发现 | WARN | 记录冲突的 P 要素对、冲突内容摘要 |
| ForceCheckpoint 执行 | INFO | 记录 tokens_before / tokens_after、摘要片段 |

<!-- @end-section -->
