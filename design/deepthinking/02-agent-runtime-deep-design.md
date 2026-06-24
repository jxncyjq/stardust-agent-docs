---
id: "deepthinking-agent-runtime-002"
title: "Agent 运行时深度设计"
aliases: ["agent runtime deep design", "Agent运行时设计", "cognitive core design"]
type: "deepthinking"
category: "design/deepthinking"
tags: ["agent", "runtime", "cognitive-core", "transport", "memory", "deep-design"]
version: "1.0.0"
created: "2026-05-04"
updated: "2026-05-04"
author: "jxncyjq"
status: "review"
parent: null
related_docs:
  - id: "analysis-hermes-runtime-002"
    relation: "reference"
    path: "../analysis/hermes/02-agent-runtime.md"
  - id: "analysis-hermes-tools-003"
    relation: "reference"
    path: "../analysis/hermes/03-tools-skills-plugins.md"
  - id: "analysis-claude-overview-001"
    relation: "reference"
    path: "../analysis/claude/01-overview.md"
  - id: "analysis-evolver-gep-002"
    relation: "reference"
    path: "../analysis/evolver/02-gep-protocol.md"
  - id: "architecture-legion"
    relation: "parent_design"
    path: "../architecture/Legion.md"
---

<!-- @section: overview -->
# Agent 运行时深度设计

## 核心命题

> 如何融合 hermes 的执行广度 + claw-code 的安全深度 + evolver 的学习能力？

答案：**认知内核 + Transport 抽象 + 经验记忆 + 工具系统**，四层架构同时汲取三个参考项目的精华。

## 一、从参考项目中提炼的 Agent 运行时核心架构

### 1.1 三个项目的角色分工

```
hermes-agent  → Agent 应该"如何工作" (执行范式)
claw-code     → Agent 应该"如何安全地工作" (安全范式)
evolver       → Agent 应该"如何越工作越好" (进化范式)

Legion Agent = hermes 的执行 + claw-code 的安全 + evolver 的进化
```

### 1.2 核心架构

```
┌─────────────────────────────────────────────────────────┐
│                   Legion Agent 运行时                     │
│                                                          │
│  ┌──────────────────────────────────────────────────┐   │
│  │              认知内核 (Cognitive Core)             │   │
│  │  上下文组装 · 冲突解决 · 优先级管理                  │   │
│  └────────────────────┬─────────────────────────────┘   │
│                       │                                  │
│  ┌────────────────────┴─────────────────────────────┐   │
│  │              Agent 主循环 (Main Loop)              │   │
│  │  预算控制 · 中断支持 · 错误恢复 · 状态持久化        │   │
│  └──┬──────────┬──────────┬──────────┬──────────────┘   │
│     │          │          │          │                   │
│  ┌──┴──┐  ┌───┴───┐  ┌───┴───┐  ┌───┴────┐             │
│  │记忆  │  │ 工具  │  │ 学习  │  │ 安全   │             │
│  │系统  │  │ 系统  │  │ 引擎  │  │ 护栏   │             │
│  └─────┘  └──────┘  └──────┘  └────────┘             │
│                                                          │
│  ┌──────────────────────────────────────────────────┐   │
│  │           MaaS Transport (MaaS 层提供的接口)        │   │
│  │  模型调用 → NormalizedResponse → 成本/延迟         │   │
│  └──────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────┘
```

<!-- @end-section -->

<!-- @section: cognitive-core -->
## 二、认知内核 — Agent 的"大脑"

### 2.1 认知内核 = 上下文组装引擎

hermes-agent 的 `prompt_builder.py`（56KB）是认知内核的最佳参考。但它的问题是**把所有内容平铺拼接**——身份、平台信息、技能索引、上下文文件、执行指南全塞在一起，缺乏优先级和冲突解决。

Legion 的认知内核必须解决这个问题：

```
执行上下文 = f(角色人设, 任务指令, 目标传导链,
               相关经验, 挂载技能, 组织上下文, 约束规则)

组装过程 (有优先级, 有冲突解决):

优先级 1 (最高): 安全红线
  → 永远不能被任务指令覆盖
  → 示例: "绝对不能执行 rm -rf /"

优先级 2: 组织约束
  → 预算限制、权限边界、部门规则
  → 示例: "单次任务费用不超过 50 CNY"

优先级 3: 任务指令
  → 当前要完成的具体任务
  → 示例: "为用户登录模块编写单元测试"

优先级 4: 经验记忆
  → 过去类似任务的处理经验
  → 示例: "上次这类需求使用了表驱动测试，效果好"

优先级 5: 辅助信息
  → 技能索引、知识库引用、协作者信息
  → 这些信息支持但不直接约束行为
```

### 2.2 冲突解决机制

当不同优先级的内容发生冲突时：

```go
type ConflictResolver struct {
    Rules []ResolutionRule
}

func (r *ConflictResolver) Resolve(instructions []Instruction) []Instruction {
    // 1. 按优先级排序
    sort.Slice(instructions, priorityDesc)

    // 2. 高优先级覆盖低优先级
    resolved := deduplicateByPriority(instructions)

    // 3. 记录冲突（用于审计）
    for _, conflict := range detectConflicts(instructions) {
        auditLog.Record(conflict)
    }

    return resolved
}
```

### 2.3 上下文大小管理

hermes 的 ContextCompressor 提供了三重策略：

| 策略 | 触发条件 | 方式 | 成本 |
|------|---------|------|------|
| 裁剪旧工具输出 | > 1 回合的工具结果 | 替换为占位符 | 零 |
| 保护首尾 | 始终 | 前 3 条 + 后 6 条保留 | 零 |
| 辅助模型摘要 | 上下文将超限 | 便宜模型摘要中间回合 | 低 |
| 迭代更新摘要 | 已有摘要 | 更新而非替换 | 低 |

**Legion 应直接采用此策略**，并将摘要存储为 Agent 的情景记忆。

<!-- @end-section -->

<!-- @section: main-loop -->
## 三、Agent 主循环 — 融合三个项目的精华

### 3.1 主循环设计

```
while (iteration < maxIterations && budget.Remaining() > 0):
    │
    ├── 1. 中断检查 (借鉴 hermes)
    │      if interrupted: break
    │
    ├── 2. 认知内核组装上下文
    │      context = cognitiveCore.Assemble(agent, task, memories)
    │
    ├── 3. MaaS 模型调用 (经由 MaaS Transport)
    │      response = maasTransport.ChatCompletion(ctx, context)
    │      // MaaS 层已完成:
    │      //   - 智能路由 (选择最优模型)
    │      //   - 预算检查 (预扣费用)
    │      //   - 响应规范化 (统一 NormalizedResponse)
    │
    ├── 4. 安全检查 (借鉴 claw-code 的权限系统)
    │      if response.ToolCalls != nil:
    │          for call in response.ToolCalls:
    │              decision = permissionEngine.Authorize(call)
    │              if decision == "deny":
    │                  toolCalls = append(toolCalls, denied(call))
    │              elif decision == "ask":
    │                  toolCalls = append(toolCalls, pendingApproval(call))
    │
    ├── 5. 工具调用执行 (借鉴 hermes 的并发执行)
    │      if len(toolCalls) > 0:
    │          if shouldParallelize(toolCalls):
    │              results = executeConcurrently(toolCalls)
    │          else:
    │              results = executeSequentially(toolCalls)
    │          messages = append(messages, results)
    │
    ├── 6. 经验提取 (借鉴 evolver 的 3 层信号)
    │      signals = signalDetector.Extract(response, toolCallResults)
    │      if len(signals) > 0:
    │          learnEngine.Process(signals)
    │
    ├── 7. 上下文压缩
    │      if contextCompressor.ShouldCompress(messages):
    │          messages = contextCompressor.Compress(messages)
    │
    └── 8. 状态持久化
           sessionDB.Save(messages, usage, cost)
```

### 3.2 与三个参考项目的差异

| 环节 | hermes | claw-code | evolver | **Legion** |
|------|--------|-----------|---------|-----------|
| 模型调用 | 直接调 LLM API | 直接调 Anthropic API | 不调 LLM | **通过 MaaS 层** |
| 响应格式 | NormalizedResponse | Anthropic 原生 | N/A | **MaaS 层规范化** |
| 安全控制 | 工具看门狗 | 完整权限系统 | 固化安全网 | **权限+看门狗+验证** |
| 经验学习 | /memory save 手动 | 无 | GEP 自动 | **自动信号提取** |
| 上下文压缩 | ContextCompressor | 会话压缩 | PROMPT_MAX_CHARS | **分层压缩** |
| 故障恢复 | 凭证池+退避 | 重试 | Git 回滚 | **多级恢复** |

<!-- @end-section -->

<!-- @section: memory-system -->
## 四、经验记忆系统 — 三层记忆架构

### 4.1 记忆分层

借鉴认知科学的三层记忆模型：

```
工作记忆 (Working Memory)
  ├── 生命周期: 单次任务
  ├── 内容: 当前对话上下文、中间状态、临时变量
  ├── 容量: 模型上下文窗口
  ├── 维护: Agent 主循环自动管理
  └── 参考: hermes 的消息历史

情景记忆 (Episodic Memory)
  ├── 生命周期: 跨任务持久化
  ├── 内容: 历史任务的关键决策、成功案例、失败原因、反馈
  ├── 容量: SQLite/PostgreSQL 持久化
  ├── 维护: 任务结束时自动提取 + FTS5 全文索引
  └── 参考: hermes 的 SessionDB + evolver 的 Capsule

能力记忆 (Semantic Memory)
  ├── 生命周期: 长期沉淀
  ├── 内容: 抽象化的工作方法、最佳实践、领域知识
  ├── 容量: LLM Wiki 知识图谱
  ├── 维护: 学习引擎定期蒸馏
  └── 参考: evolver 的 Gene + hermes 的技能
```

### 4.2 记忆生命周期

```
工作任务执行
  │
  ├── 工作记忆 (实时读写)
  │     └── 任务结束 → 过期清理
  │
  ├── 情景记忆提取 (任务结束时)
  │     ├── 关键决策点识别
  │     ├── 成功/失败模式标注
  │     ├── 反馈关联
  │     └── Capsule 创建 (执行证明)
  │
  └── 能力记忆蒸馏 (定期)
        ├── 多条情景记忆聚类
        ├── 抽象为可复用方法
        ├── 存入 LLM Wiki
        └── Gene 生成 (策略模板)
```

### 4.3 记忆注入到认知内核

```
认知内核组装时:
  ├── 1. 根据任务类型检索相关情景记忆 (FTS5 + 语义搜索)
  ├── 2. 根据任务领域检索相关能力记忆 (LLM Wiki 混合检索)
  ├── 3. 优先级排序: 完全匹配 > 相似任务 > 同领域 > 通用
  └── 4. 注入为"相关经验记忆"区块
```

<!-- @end-section -->

<!-- @section: tool-system -->
## 五、工具系统 — 自注册 + 看门狗 + 权限

### 5.1 工具注册模式 (借鉴 hermes)

```go
// tools/code_execution.go
package tools

import "legion/agent/tool"

func init() {
    tool.Registry.Register(tool.Entry{
        Name:     "execute_code",
        Toolset:  "development",
        Category: "execution",
        Schema: tool.Schema{
            Name:        "execute_code",
            Description: "在隔离环境中执行代码",
            Parameters: map[string]interface{}{
                "type": "object",
                "properties": map[string]interface{}{
                    "language": map[string]interface{}{
                        "type": "string",
                        "enum": []string{"python", "go", "javascript", "bash"},
                    },
                    "code": map[string]interface{}{
                        "type": "string",
                    },
                },
                "required": []string{"language", "code"},
            },
        },
        Handler:        executeCodeHandler,
        RequiresApproval: true,            // 借鉴 claw-code 权限
        MaxResultSize:  100_000,            // 借鉴 hermes 结果裁剪
        MaxExecutionMs: 30_000,             // 超时保护
        AllowedInYolo:  false,              // 自动模式下仍需审批
    })
}
```

### 5.2 工具执行流程

```
Agent 决定调用工具 execute_code(...)
  │
  ├── 1. 看门狗检查 (借鉴 hermes)
  │      guardrail.BeforeCall(name, args)
  │      → allow / warn / block / halt
  │
  ├── 2. 权限检查 (借鉴 claw-code)
  │      permission.Authorize(tool, args, agent)
  │      → allow / ask_user / deny
  │
  ├── 3. 参数验证
  │      validator.Validate(args, tool.Schema.Parameters)
  │
  ├── 4. 沙箱执行
  │      sandbox.Execute(tool.Handler, args, timeout)
  │
  ├── 5. 结果处理
  │      result = truncate(result, tool.MaxResultSize)
  │
  └── 6. 看门狗更新
         guardrail.AfterCall(name, result, err)
         → 更新失败计数、检测重复模式
```

### 5.3 工具集组合 (借鉴 hermes 的 includes)

```go
var Toolsets = map[string]Toolset{
    "web":       {Tools: []string{"web_search", "web_extract"}},
    "terminal":  {Tools: []string{"terminal", "code_execution"}},
    "file":      {Tools: []string{"file_read", "file_write", "file_search"}},
    "research":  {Includes: []string{"web", "terminal", "file"}},
    "development": {Includes: []string{"terminal", "file", "web"}},
    "full_stack":  {Tools: []string{...all...}},
}
```

<!-- @end-section -->

<!-- @section: error-recovery -->
## 六、错误分类学与自动恢复

### 6.1 借鉴 hermes 的 FailoverReason

```go
type ErrorCategory int

const (
    ErrAuth          ErrorCategory = iota  // → 刷新凭证
    ErrRateLimit                            // → 退避重试
    ErrContextOverflow                      // → 压缩上下文
    ErrModelNotFound                        // → 回退到备用模型
    ErrTimeout                              // → 抖动退避
    ErrBilling                              // → 通知管理员
    ErrSafetyBlock                          // → 拒绝执行
    ErrToolFailure                          // → 看门狗处理
)

type ClassifiedError struct {
    Category       ErrorCategory
    Original       error
    Retryable      bool
    ShouldCompress bool
    ShouldFallback bool
    RecoveryHint   string
}
```

### 6.2 恢复策略链

```
API 调用失败
  │
  ├── 401/403 → 凭证失效
  │     └── → 自动刷新 → 重试 (最多 2 次)
  │
  ├── 429 → 速率限制
  │     └── → 指数退避 (1s, 2s, 4s, 8s) → 重试
  │           └── 仍失败 → 切换备用模型
  │
  ├── 上下文溢出
  │     └── → 触发上下文压缩 → 重试
  │
  ├── 超时
  │     └── → 抖动退避 → 重试
  │           └── 仍失败 → 切换低延迟模型
  │
  └── 模型不可用
        └── → 按 fallback_chain 降级
              └── 所有候选不可用 → 返回错误 + 通知
```

<!-- @end-section -->

<!-- @section: design-decisions -->
## 七、关键设计决策

| 决策 | 选择 | 理由 |
|------|------|------|
| 语言 | Go | stardust 基础库 + 性能 + 类型安全 |
| 模型调用 | 全经 MaaS 层 | Agent 完全不知道模型细节 |
| 上下文组装 | 优先级排序 + 冲突解决 | hermes 的 prompt_builder 太平铺 |
| 工具注册 | 自注册 + AST 发现 | hermes 验证过的成熟模式 |
| 工具执行 | ThreadPoolExecutor → goroutine | Go 原生并发 |
| 记忆模型 | 三层 (工作/情景/能力) | 认知科学验证 + evolver 资产化 |
| 错误恢复 | 分类学 + 恢复提示 | hermes + evolver 的结合 |
| 安全机制 | 权限 + 看门狗 + 验证 | claw-code + hermes + evolver |

### 与 hermes-agent 的关键差异

1. **AIAgent 不膨胀**: hermes 的 `run_agent.py` 有 14,000 行。Legion 从第一天就模块化拆分为 cognitive_core、main_loop、memory、tools、error_recovery 等独立包
2. **模型调用解耦**: hermes 的 Agent 直接调 LLM API，Legion 的 Agent 只调 MaaS 层接口
3. **进化能力内建**: hermes 完全没有进化机制，Legion 的 Agent 原生支持学习循环

### 不做什么

1. **不在 Agent 循环中处理模型选择** — 那是 MaaS 层的职责
2. **不把所有逻辑塞进一个文件** — hermes 的 14KLOC 教训
3. **不用 Python 写核心循环** — Go 的性能和并发优势
4. **不混淆核心逻辑** — evolver 的教训，Legion 完全开源

<!-- @end-section -->

<!-- @section: related -->
## 相关文档

- [[../analysis/hermes/02-agent-runtime|hermes Agent 运行时分析]]
- [[../analysis/hermes/03-tools-skills-plugins|hermes 工具系统分析]]
- [[../analysis/claude/01-overview|claw-code 架构总览]]
- [[../analysis/evolver/02-gep-protocol|Evolver GEP 协议分析]]
- [[01-maas-layer-deep-design|MaaS 模型调度层深度设计]]
- [[03-evolution-learning-deep-design|进化学习系统深度设计]]
- [[05-multi-agent-collaboration|多智能体协作深度设计]]

<!-- @end-section -->
