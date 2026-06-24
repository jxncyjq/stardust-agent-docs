---
id: "deepthinking-security-006"
title: "安全治理体系深度设计"
aliases: ["security governance deep design", "安全治理设计", "三原则落地"]
type: "deepthinking"
category: "design/deepthinking"
tags: ["security", "governance", "observability", "risk-control", "deep-design"]
version: "1.0.0"
created: "2026-05-04"
updated: "2026-05-04"
author: "jxncyjq"
status: "review"
parent: null
related_docs:
  - id: "analysis-claude-overview-001"
    relation: "reference"
    path: "../analysis/claude/01-overview.md"
  - id: "analysis-evolver-datamodels-005"
    relation: "reference"
    path: "../analysis/evolver/05-data-models.md"
  - id: "analysis-maas-insights-007"
    relation: "reference"
    path: "../analysis/maas/07-maas-insights.md"
  - id: "architecture-legion"
    relation: "parent_design"
    path: "../architecture/Legion.md"
---

<!-- @section: overview -->
# 安全治理体系深度设计

## 核心命题

> 安全治理应该是"外挂防火墙"还是"渗透式治理"？

答案：**渗透式治理**。Legion 架构方案定义的"三原则"——可观测、可治理、可控风险——不是独立的安全模块，而是渗透到每一层架构、每一个组件的设计约束。这份文档从四个参考项目中提炼安全最佳实践，为三原则找到具体的技术落地路径。

## 一、四大参考项目的安全体系盘点

### 1.1 各项目安全能力总览

```
claw-code (Rust):
  权限系统 ★★★★★  (工具级/参数级审批, 条件规则, autoAllow)
  会话压缩 ★★★★   (敏感信息裁剪)
  文件安全 ★★★    (工作目录限制)
  审计追踪 ★★     (基础日志)

new-api (Go):
  密钥提取 ★★★★★  (6 协议 TokenAuth)
  泄漏扫描 ★★★★   (环境值反向检测)
  配额管控 ★★★★★  (多维预算 + 熔断)
  消费审计 ★★★★★  (完整的消费日志)

hermes-agent (Python):
  工具看门狗 ★★★★  (allow→warn→block→halt)
  技能安全扫描 ★★★★ (488+ 威胁模式)
  日志脱敏 ★★★★   (95+ API Key 模式)
  文件黑名单 ★★★   (禁止操作关键路径)

evolver (JavaScript):
  固化安全网 ★★★★★ (验证→金丝雀→爆炸半径→回滚)
  泄漏扫描 ★★★★★  (27+ 模式 + 环境值反向检测)
  源文件保护 ★★★★  (shield.js)
  文件锁 ★★★★    (O_EXCL 原子操作)
  加密 ★★★     (AES-256-GCM)
  内容寻址 ★★★★  (SHA-256 完整性)
```

### 1.2 Legion 应该继承的安全模式

| 安全能力 | 来源 | 优先级 |
|---------|------|--------|
| 权限系统 (工具级/参数级审批) | claw-code | P0 |
| 工具看门狗 (行为模式检测) | hermes | P0 |
| 固化安全网 (变更验证+回滚) | evolver | P0 |
| 泄漏扫描 (模式+环境值反检) | evolver + new-api | P0 |
| 内容寻址 (完整性验证) | evolver | P1 |
| 审计日志 (不可篡改追加链) | evolver + new-api | P1 |
| 日志脱敏 (多级脱敏策略) | hermes | P1 |
| 配额管控 (多级熔断) | new-api | P1 |
| 技能安全扫描 (威胁模式) | hermes | P2 |
| 沙箱执行 (命令白名单+超时) | evolver | P2 |

<!-- @end-section -->

<!-- @section: observability -->
## 二、可观测性 — 全链路透明

### 2.1 三层的可观测性

```
Layer 1: 基础设施级 (MaaS 层)
  - 每次模型调用的完整追踪
  - Token 消耗实时仪表盘
  - 模型延迟/可用性监控
  - 预算消耗趋势预警

Layer 2: 应用级 (Agent 层)
  - 每个 Agent 的决策过程可解释
  - 每条记忆可追溯来源
  - 每次进化有审计记录
  - 能力雷达图 + 退化预警

Layer 3: 业务级 (协作层)
  - 工作流节点级进度看板
  - 任务输入/输出完整记录
  - 通讯审计链 (不可篡改)
  - 审批门控的完整日志
```

### 2.2 审计链设计

借鉴 evolver 的 EvolutionEvent 仅追加日志和区块链式的完整性链：

```go
type AuditEntry struct {
    ID          string      `json:"id"`
    PrevID      string      `json:"prev_id"`      // 前一条审计记录 (链式)
    Category    AuditCategory `json:"category"`
    Action      string      `json:"action"`
    Actor       Actor       `json:"actor"`        // Agent / 人类 / 系统
    Target      string      `json:"target"`       // 操作对象
    Detail      json.RawMessage `json:"detail"`   // 结构化详情
    Outcome     string      `json:"outcome"`      // success | failure | pending
    ContentHash string      `json:"content_hash"` // SHA-256 (防篡改)
    Timestamp   time.Time   `json:"timestamp"`
}

type AuditCategory string
const (
    AuditModelCall    AuditCategory = "model_call"     // MaaS 模型调用
    AuditToolCall     AuditCategory = "tool_call"      // 工具执行
    AuditTaskLifecycle AuditCategory = "task_lifecycle" // 任务生命周期
    AuditEvolution    AuditCategory = "evolution"      // 进化操作
    AuditKnowledge    AuditCategory = "knowledge"      // 知识操作
    AuditPermission   AuditCategory = "permission"     // 权限变更
    AuditCommunication AuditCategory = "communication" // 通讯记录
    AuditBudget       AuditCategory = "budget"         // 预算操作
    AuditGovernance   AuditCategory = "governance"     // 治理门控
)

// 审计链完整性验证
func VerifyAuditChain(entries []AuditEntry) bool {
    for i := 1; i < len(entries); i++ {
        // 1. 链式验证
        if entries[i].PrevID != entries[i-1].ID {
            return false
        }
        // 2. 内容完整性验证
        expectedHash := sha256Hash(entries[i])
        if entries[i].ContentHash != expectedHash {
            return false
        }
    }
    return true
}
```

### 2.3 决策溯源

```
"这个 Agent 为什么做出了这个决策？"

可追溯的因果链:
  1. 认知内核组装日志: 当时注入了哪些上下文、记忆、约束
  2. MaaS 路由日志: 用了哪个模型、为什么选择这个模型
  3. LLM 原始响应: (可选保存，用于事后审计)
  4. 工具调用日志: 调用了哪些工具、参数是什么、结果是什么
  5. 记忆检索日志: 当时检索到了哪些相关记忆
```

<!-- @end-section -->

<!-- @section: governability -->
## 三、可治理性 — 人类始终在回路中

### 3.1 治理能力全景

```
治理维度          治理操作
─────────────────────────────────────
Agent 治理        创建/冻结/销毁/恢复/重置进化状态
记忆治理          查看/审核/删除/手动注入/回退
知识治理          审核/批准/驳回/回滚版本/冲突裁决
流程治理          暂停/恢复/跳过/重分配/强制终止
权限治理          角色调整/能力声明修改/模型配额调整
预算治理          调整/冻结/解冻/紧急追加
组织治理          调整架构/重新分配/合并/拆分
```

### 3.2 人类命令接口

```go
type GovernanceAPI struct {
    // Agent 治理
    FreezeAgent(agentID, reason string) error
    ResetEvolution(agentID string, toCheckpoint int) error
    InjectMemory(agentID string, memory MemoryEntry) error
    DeleteMemory(agentID, memoryID string) error

    // 流程治理
    PauseWorkflow(workflowID, reason string) error
    ResumeWorkflow(workflowID string) error
    ReassignTask(taskID, fromAgent, toAgent string) error
    ForceTerminate(taskID, reason string) error

    // 知识治理
    ApproveKnowledge(entryID string) error
    RejectKnowledge(entryID, reason string) error
    RollbackKnowledge(entryID string, toVersion int) error
    ResolveConflict(conflictID string, resolution Resolution) error

    // 权限治理
    UpdateAgentRole(agentID, newRole string) error
    UpdateModelQuota(agentID string, quota ModelQuota) error
    AdjustBudget(scope BudgetScope, adjustment float64) error
}
```

### 3.3 治理操作的最高权限

```
系统约束 (不可覆盖):
  - 安全红线: Agent 绝对不可学习绕过
  - 审计链: 不可删除、不可修改
  - 人类最终裁决权: Agent 的任何决策都可被人类推翻

人类决策者可以:
  ✅ 冻结/销毁任何 Agent
  ✅ 回滚任何进化状态
  ✅ 删除任何 Agent 记忆
  ✅ 驳回任何 AI 产出的知识
  ✅ 强制终止任何工作流

人类决策者不能:
  ❌ 删除审计记录
  ❌ 修改已记录的审计事件
  ❌ 绕过安全红线
```

<!-- @end-section -->

<!-- @section: risk-control -->
## 四、可控风险 — 安全边界硬约束

### 4.1 三层安全边界

```
外围边界 (基础设施级):
  - 网络隔离 (Agent 执行环境 vs 控制平面)
  - API 鉴权 (所有入口统一 TokenAuth)
  - 传输加密 (TLS 1.3)
  - 数据库加密 (敏感字段 AES-256-GCM)

中间边界 (应用级):
  - 权限系统 (工具级/参数级审批)
  - 工具看门狗 (行为模式检测 + 自动阻断)
  - 爆炸半径限制 (单次操作的影响范围硬上限)
  - 预算熔断 (告警→降级→冻结)

内核边界 (数据级):
  - 泄漏扫描 (模式 + 环境值反向检测)
  - 内容寻址 (SHA-256 完整性)
  - 沙箱执行 (命令白名单 + 超时)
  - 金丝雀检查 (系统健康保护)
```

### 4.2 权限引擎 (借鉴 claw-code)

```go
type PermissionEngine struct {
    Rules []PermissionRule
}

type PermissionRule struct {
    ID          string
    ToolName    string            // 匹配的工具
    Conditions  []Condition       // 触发条件
    Action      PermissionAction  // allow | ask | deny
    Priority    int               // 高优先级规则先匹配
}

type Condition struct {
    Field    string      // agent.role | tool.args.* | task.budget
    Operator string      // eq | ne | in | gt | lt | matches
    Value    interface{}
}

// 示例规则:
rules := []PermissionRule{
    {
        ID:       "deny-rm-rf",
        ToolName: "terminal",
        Conditions: []Condition{
            {Field: "tool.args.command", Operator: "matches", Value: "rm -rf /"},
        },
        Action:   PermissionDeny,
        Priority: 100,
    },
    {
        ID:       "approve-high-cost",
        ToolName: "*",
        Conditions: []Condition{
            {Field: "task.estimated_cost", Operator: "gt", Value: 100},
        },
        Action:   PermissionAsk,
        Priority: 50,
    },
    {
        ID:       "allow-dev-tools",
        ToolName: "*",
        Conditions: []Condition{
            {Field: "agent.role", Operator: "in", Value: []string{"developer", "architect"}},
        },
        Action:   PermissionAllow,
        Priority: 10,
    },
}
```

### 4.3 工具看门狗 (借鉴 hermes)

```go
type ToolGuardrail struct {
    FailureCount    map[string]int      // 每个工具的连续失败次数
    SameToolFailures map[string]int     // 同类工具的连续失败
    IdempotentNoProgress map[string]int // 幂等工具无进展次数
    Thresholds      GuardrailThresholds
}

type GuardrailThresholds struct {
    MaxConsecutiveFailures int  // 最大连续失败 → halt
    MaxSameToolFailures    int  // 最大同类失败 → block
    MaxNoProgressAttempts  int  // 最大无进展 → warn
}

func (g *ToolGuardrail) BeforeCall(name string, args map[string]interface{}) Decision {
    // 精确重复失败检测
    if g.FailureCount[name] >= g.Thresholds.MaxConsecutiveFailures {
        return Decision{Action: "halt", Reason: "连续失败次数超限"}
    }

    // 同类工具连续失败
    toolCategory := getToolCategory(name)
    if g.SameToolFailures[toolCategory] >= g.Thresholds.MaxSameToolFailures {
        return Decision{Action: "block", Reason: "同类工具连续失败"}
    }

    // 幂等工具无进展
    if isIdempotent(name) && g.IdempotentNoProgress[name] >= g.Thresholds.MaxNoProgressAttempts {
        return Decision{Action: "warn", Reason: "幂等工具重复调用无进展"}
    }

    return Decision{Action: "allow"}
}
```

### 4.4 泄漏扫描 (借鉴 evolver + new-api)

```go
type LeakScanner struct {
    Patterns []LeakPattern
}

type LeakPattern struct {
    Name    string
    Regex   *regexp.Regexp
    Category string  // api_key | token | password | private_key | db_url
}

type LeakScanResult struct {
    Found    bool
    Matches  []LeakMatch
    EnvLeaks []EnvLeak   // 环境值反向检测
}

func (s *LeakScanner) Scan(content string, envVars map[string]string) LeakScanResult {
    result := LeakScanResult{}

    // 1. 模式扫描 (27+ 正则)
    for _, pattern := range s.Patterns {
        matches := pattern.Regex.FindAllString(content, -1)
        for _, match := range matches {
            result.Matches = append(result.Matches, LeakMatch{
                Pattern: pattern.Name,
                Category: pattern.Category,
                Match: maskSensitive(match),
            })
        }
    }

    // 2. 环境值反向检测 (Evolver 的杀手级特性)
    for key, value := range envVars {
        if len(value) > 8 && strings.Contains(content, value) {
            result.EnvLeaks = append(result.EnvLeaks, EnvLeak{
                EnvKey: key,
                Leaked: true,
            })
        }
    }

    result.Found = len(result.Matches) > 0 || len(result.EnvLeaks) > 0
    return result
}
```

### 4.5 固化安全网 (借鉴 evolver)

固化安全网在进化系统的 [[03-evolution-learning-deep-design|进化学习系统]] 中已有详细设计，此处不再重复。

<!-- @end-section -->

<!-- @section: compliance -->
## 五、合规与数据保护

### 5.1 数据分类

| 数据等级 | 示例 | 存储 | 传输 | 访问控制 |
|---------|------|------|------|---------|
| 公开 | 产品文档、公开 API | 明文 | 明文 | 全体可读 |
| 内部 | 设计文档、项目计划 | 明文 | TLS | 公司内可读 |
| 机密 | 用户数据、财务信息 | AES-256-GCM | TLS | RBAC |
| 绝密 | API Key、密钥 | AES-256-GCM + 独立密钥 | TLS + mTLS | 管理员 + 审计 |

### 5.2 数据生命周期

```
数据创建 → 分类标注 → 访问控制 → 使用追踪 → 定期审计 → 安全销毁

每个阶段的三原则:
  可观测: 谁在何时访问了什么数据
  可治理: 管理员可随时调整权限、撤销访问
  可控风险: 敏感数据加密存储、访问频率异常告警
```

<!-- @end-section -->

<!-- @section: matrix -->
## 六、三原则落地矩阵 (技术实现版)

| 能力域 | 可观测 | 可治理 | 可控风险 |
|--------|--------|--------|---------|
| **MaaS 模型调度** | 调用链追踪 (AuditEntry 链) + Token 消耗仪表盘 + 路由决策日志 | 路由策略实时可调 + 手动绑定/禁用模型 + 配额随时调整 | 告警(80%)→降级(95%)→冻结(100%)三级熔断 |
| **Agent 运行时** | 决策过程可解释 + 记忆可追溯 + 进化报告自动生成 | 记忆审核/删除/注入 + 进化冻结/回退 + Agent 冻结/销毁 | 权限引擎 + 看门狗 + 安全红线不可学习 |
| **LLM Wiki** | 知识溯源 (来源+版本) + 变更历史 + 质量评分 + 引用统计 | AI 知识需审核 + 冲突裁决 + 版本回滚 + 权限控制 | 权限分级 + 涉密加密 + 错误知识传播阻断 |
| **组织架构** | 实时状态可视化 + 健康度监控 + 负载统计 | 随时调整架构 + 冻结/销毁 Agent + 重分配权限 | 越权操作拦截 + 权限严格绑定组织位置 |
| **工作流协作** | 节点级进度看板 + 输入输出可查 + 通讯审计链 | 七大审批门控 + 暂停/恢复/跳过/重分配 | 高风险节点增强监控 + 失败重试上限 + 异常自动冻结 |
| **进化学习** | 学习过程日志 + Gene 表现追踪 + 能力雷达图 | 学习结果审核 + 不当记忆删除 + 进化策略切换 | 固化安全网 (验证+金丝雀+爆炸半径+回滚) |
| **成本控制** | 多维度用量仪表盘 + 趋势预测 + 异常消费告警 | 公司/部门/Agent/模型多维预算 + 手动调整 | 告警→降级→冻结 三级熔断 + 超支自动冻结 |

<!-- @end-section -->

<!-- @section: design-decisions -->
## 七、关键设计决策

| 决策 | 选择 | 理由 |
|------|------|------|
| 安全架构 | 渗透式 (非外挂) | 三原则是贯穿所有层的设计约束 |
| 审计链 | 类区块链 Hash 链 | 不可篡改 + 可验证完整性 |
| 权限模型 | 条件规则引擎 | claw-code 验证过的最灵活方案 |
| 泄漏检测 | 模式 + 环境反检 | evolver + new-api 的双重验证 |
| 沙箱执行 | 命令白名单 + timeout | evolver 验证过的安全模式 |
| 加密 | AES-256-GCM | 业界标准 |
| 数据分类 | 4 级 (公开/内部/机密/绝密) | 最小权限原则 |

### 不做什么

1. **不做事后安全** — 安全是设计约束，不是事后补丁
2. **不做可删除的审计日志** — 审计链不可篡改
3. **不做无需审批的高风险操作** — 所有高风险操作必须有人类在回路中
4. **不做纯依赖正则的安全检测** — 环境值反向检测是必要补充

<!-- @end-section -->

<!-- @section: related -->
## 相关文档

- [[../architecture/Legion|Legion 项目方案 — 三原则]]
- [[../analysis/claude/01-overview|claw-code 权限系统]]
- [[../analysis/evolver/05-data-models|Evolver 安全机制分析]]
- [[../analysis/maas/05-middleware-and-flow|new-api 中间件与安全]]
- [[01-maas-layer-deep-design|MaaS 模型调度层深度设计]]
- [[02-agent-runtime-deep-design|Agent 运行时深度设计]]
- [[05-multi-agent-collaboration|多智能体协作深度设计]]

<!-- @end-section -->
