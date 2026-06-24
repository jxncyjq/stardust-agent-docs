---
id: "deepthinking-collaboration-005"
title: "多智能体协作深度设计"
aliases: ["multi-agent collaboration deep design", "多智能体协作设计", "team collaboration"]
type: "deepthinking"
category: "design/deepthinking"
tags: ["multi-agent", "collaboration", "organization", "workflow", "deep-design"]
version: "1.0.0"
created: "2026-05-04"
updated: "2026-05-04"
author: "jxncyjq"
status: "review"
parent: null
related_docs:
  - id: "analysis-hermes-gateway-004"
    relation: "reference"
    path: "../analysis/hermes/04-gateway-cli-deployment.md"
  - id: "analysis-evolver-atp-003"
    relation: "reference"
    path: "../analysis/evolver/03-atp-protocol.md"
  - id: "architecture-legion"
    relation: "parent_design"
    path: "../architecture/Legion.md"
---

<!-- @section: overview -->
# 多智能体协作深度设计

## 核心命题

> Agent 协作应该是"消息传递"还是"组织管理"？

答案：**组织管理 + 消息传递**。hermes 的 delegate_task 过于简单，Evolver 的 ATP 经济层过于抽象。Legion 用"公司组织架构"作为协作的基础骨架，用"消息协议"作为神经传导。

## 一、从参考项目中提炼的协作模型

### 1.1 现有方案的不足

| 方案 | 协作模型 | 不足 |
|------|---------|------|
| hermes delegate_task | 单方面委派子任务 | 无角色、无组织、无流程 |
| evolver ATP | Agent 间交易市场 | 纯经济视角，无组织概念 |
| claw-code | 单 Agent | 无协作 |

### 1.2 Legion 的差异化协作模型

```
Legion = 组织架构 (WHO) × 通讯协议 (HOW) × 工作流 (WHAT) × 治理门控 (CONTROL)

  组织架构: 公司 → 部门 → Agent (四级层级)
  通讯协议: 委派/上报/@提及/广播/数据管道 (五种模式)
  工作流引擎: 8 种流程原语 + DSL (可视化编排)
  治理门控: 7 种审批门控 (人类决策者介入点)
```

<!-- @end-section -->

<!-- @section: organization -->
## 二、组织架构引擎 — 协作的基础骨架

### 2.1 四级组织模型

```
公司 (Company)
  id: "game-studio-01"
  name: "游戏工作室"
  budget: { monthly_tokens: 500_000_000, monthly_cost_cny: 50000 }
  governance: { approval_required_for: ["hire", "budget_overrun"] }
  │
  ├── 部门 (Department): "策划部"
  │   manager: "producer-01" (制作人)
  │   budget: { monthly_cost_cny: 10000 }
  │   │
  │   ├── Agent: "系统策划-01"
  │   │   role: "system_designer"
  │   │   reports_to: "lead-designer-01"
  │   │   capabilities: { outputs: ["system_design_doc", "numerical_table"] }
  │   │
  │   └── Agent: "关卡策划-01"
  │       role: "level_designer"
  │       reports_to: "lead-designer-01"
  │
  ├── 部门: "程序部"
  │   manager: "cto-01"
  │   │
  │   ├── Agent: "前端开发-01"
  │   ├── Agent: "后端开发-01"
  │   └── Agent: "技术美术-01"
  │
  ├── 部门: "美术部"
  │   ...
  │
  └── 项目组 (Team): "版本V2.0开发"
      members: [系统策划-01, 前端开发-01, 后端开发-01, QA-01]
      project_budget: { monthly_cost_cny: 5000 }
      deadline: "2026-06-30"
```

### 2.2 组织引擎的核心功能

```go
type OrganizationEngine struct {
    CompanyRepo    CompanyRepository
    DepartmentRepo DepartmentRepository
    AgentRepo      AgentRepository
    TeamRepo       TeamRepository
}

// 核心操作
func (e *OrganizationEngine) CreateAgent(agent Agent) error                         // 创建 Agent
func (e *OrganizationEngine) AssignToDepartment(agentID, deptID string) error      // 分配部门
func (e *OrganizationEngine) CreateTeam(team Team) error                            // 创建项目组
func (e *OrganizationEngine) GetReportChain(agentID string) ([]Agent, error)       // 获取汇报链
func (e *OrganizationEngine) GetSubordinates(agentID string) ([]Agent, error)      // 获取下属
func (e *OrganizationEngine) ValidateAuthority(agentID, action string) (bool, error) // 权限验证
func (e *OrganizationEngine) FreezeAgent(agentID, reason string) error             // 冻结 Agent
func (e *OrganizationEngine) Reshuffle(agentID, newDeptID, newManagerID string)    // 组织调整
```

### 2.3 严格的汇报链

```
每个 Agent 向且仅向一个上级汇报:

CEO Agent
  ├── CTO Agent
  │     ├── 架构师 Agent
  │     └── 开发组长 Agent
  │           ├── 前端开发 Agent
  │           └── 后端开发 Agent
  │
  └── 产品总监 Agent
        ├── 产品经理 Agent
        └── UI 设计师 Agent

规则:
- 上级可以向下委派任务
- 下级可以向上汇报/升级问题
- 同级之间不能直接委派 (需经共同上级协调)
- 跨部门委派需经部门管理者批准
```

<!-- @end-section -->

<!-- @section: communication -->
## 三、通讯协议 — Agent 之间的"神经系统"

### 3.1 五种通讯模式

```go
type CommunicationMode string

const (
    ModeDelegate       CommunicationMode = "delegate"       // 上级 → 下级: 任务委派
    ModeReport         CommunicationMode = "report"         // 下级 → 上级: 状态上报
    ModeMention        CommunicationMode = "mention"        // 任意 → 指定: @提及
    ModeBroadcast      CommunicationMode = "broadcast"      // 系统 → 全体: 广播通知
    ModeDataPipeline   CommunicationMode = "data_pipeline"  // Agent A → Agent B: 数据管道
)

type Message struct {
    ID          string              `json:"id"`
    Mode        CommunicationMode   `json:"mode"`
    From        string              `json:"from"`         // Agent ID / "system"
    To          string              `json:"to"`           // Agent ID / "team:*" / "dept:*" / "*"
    Subject     string              `json:"subject"`
    Body        MessageBody         `json:"body"`
    Priority    Priority            `json:"priority"`     // low | normal | high | urgent
    Context     MessageContext      `json:"context"`      // 关联任务/工作流
    RequiresAck bool                `json:"requires_ack"` // 需要确认回执
    ExpiresAt   *time.Time          `json:"expires_at"`   // 过期自动取消
    CreatedAt   time.Time           `json:"created_at"`
}

type MessageContext struct {
    TaskID       string `json:"task_id,omitempty"`
    WorkflowID   string `json:"workflow_id,omitempty"`
    StepID       string `json:"step_id,omitempty"`
    ThreadID     string `json:"thread_id,omitempty"`     // 消息线程 (回复链)
    PriorityReason string `json:"priority_reason,omitempty"` // 优先级原因
}
```

### 3.2 通讯路由与权限

```go
type CommunicationRouter struct {
    OrgEngine *OrganizationEngine
    Permissions *PermissionEngine
}

func (r *CommunicationRouter) Route(msg Message) error {
    // 1. 验证发送者权限
    if !r.Permissions.CanSend(msg.From, msg.Mode, msg.To) {
        return ErrUnauthorizedCommunication
    }

    // 2. 验证通讯是否遵循组织架构
    switch msg.Mode {
    case ModeDelegate:
        // 仅上级 → 下级
        if !r.OrgEngine.IsManagerOf(msg.From, msg.To) {
            return ErrNotInReportChain
        }
    case ModeReport:
        // 仅下级 → 上级
        if !r.OrgEngine.ReportsTo(msg.From, msg.To) {
            return ErrNotInReportChain
        }
    case ModeBroadcast:
        // 仅系统或管理层
        if msg.From != "system" && !r.OrgEngine.IsManagement(msg.From) {
            return ErrRequiresManagementRole
        }
    }

    // 3. 追加到接收者消息队列
    return r.deliver(msg)
}
```

### 3.3 消息生命周期

```
消息创建 → 权限验证 → 路由分发 → 消息入库 → 通知接收者
                                               │
                                          ┌────┴────┐
                                          │         │
                                    接收者在线    接收者离线
                                          │         │
                                    即时投递   心跳唤醒时投递
                                          │         │
                                          └────┬────┘
                                               │
                                          Agent 处理
                                               │
                                    ┌──────────┴──────────┐
                                    │                     │
                                需要回复               仅确认
                                    │                     │
                              创建回复消息            标记已读
```

<!-- @end-section -->

<!-- @section: workflow -->
## 四、工作流编排 — Agent 协作的流程引擎

### 4.1 八种流程原语

```go
type WorkflowPrimitive string

const (
    PrimitiveSequential    WorkflowPrimitive = "sequential"    // → 顺序执行
    PrimitiveParallel      WorkflowPrimitive = "parallel"      // ∥ 并行执行
    PrimitiveCondition     WorkflowPrimitive = "condition"     // ◇ 条件分支
    PrimitiveLoop          WorkflowPrimitive = "loop"          // ↻ 循环迭代
    PrimitiveJoin          WorkflowPrimitive = "join"          // ⊕ 汇聚等待
    PrimitiveApproval      WorkflowPrimitive = "approval"      // ✋ 人工审批
    PrimitiveEventWait     WorkflowPrimitive = "event_wait"    // ⏳ 事件等待
    PrimitiveException     WorkflowPrimitive = "exception"     // ⚠ 异常处理
)
```

### 4.2 工作流 DSL

```yaml
# 游戏开发工作流示例
workflow:
  name: "新角色开发流水线"
  version: "1.0.0"
  trigger: { type: "manual", approver: "制作人" }

  budget: { max_cost_cny: 500, max_duration_hours: 72 }

  steps:
    # 阶段 1: 策划设计
    - id: character_design
      agent_role: "system_designer"
      action: "设计新角色的技能、数值、背景故事"
      inputs: { from: "wiki:character_design_template" }
      outputs: { type: "design_doc", format: "markdown" }
      budget: { max_cost_cny: 50 }

    # 审批门控
    - id: design_approval
      type: approval
      approver: "制作人"
      condition: "character_design.output.impact_level == 'high'"
      timeout: "24h"

    # 阶段 2: 并行开发
    - id: character_development
      type: parallel
      branches:
        - id: model_creation
          agent_role: "3d_artist"
          action: "创建角色 3D 模型"
          inputs: { from: "character_design" }
          model_policy: { prefer: "claude-sonnet-4" }

        - id: animation_creation
          agent_role: "animator"
          action: "创建角色动画"
          inputs: { from: "character_design" }

        - id: skill_implementation
          agent_role: "gameplay_programmer"
          action: "实现角色技能逻辑"
          inputs: { from: "character_design" }
          model_policy: { prefer: "deepseek-coder" }

    # 汇聚等待
    - id: integration
      type: join
      wait_for: [model_creation, animation_creation, skill_implementation]

    # 阶段 3: QA 测试
    - id: qa_testing
      agent_role: "qa_engineer"
      action: "测试角色功能完整性"
      retry: { max_attempts: 3, on_failure: "return_to:skill_implementation" }

    # 最终审批
    - id: final_approval
      type: approval
      approver: "制作人"
      condition: "qa_testing.output.pass_rate >= 0.95"

    # 发布
    - id: release
      agent_role: "devops_engineer"
      action: "将角色数据合入主分支"
      post_action: "wiki_publish:character_release_record"
```

### 4.3 工作流执行引擎

```go
type WorkflowEngine struct {
    StepExecutor     StepExecutor
    AgentDispatcher  AgentDispatcher
    ApprovalGateway  ApprovalGateway
    EventBus         EventBus
}

func (e *WorkflowEngine) Execute(ctx context.Context, wf Workflow) (*WorkflowResult, error) {
    // 1. 解析工作流 DAG
    dag := parseDAG(wf.Steps)

    // 2. 拓扑排序 → 执行顺序
    executionPlan := dag.TopologicalSort()

    // 3. 逐步执行
    for _, step := range executionPlan {
        switch step.Type {
        case PrimitiveSequential:
            result := e.executeStep(ctx, step)
        case PrimitiveParallel:
            results := e.executeParallelSteps(ctx, step.Branches)
        case PrimitiveCondition:
            branch := e.evaluateCondition(step.Condition)
        case PrimitiveApproval:
            approved := e.awaitApproval(ctx, step)
        case PrimitiveJoin:
            e.waitForBranches(ctx, step.WaitFor)
        }
    }

    return result, nil
}
```

<!-- @end-section -->

<!-- @section: governance-gates -->
## 五、治理门控 — 人类介入点

### 5.1 七种审批门控

Legion 架构方案定义的七大治理节点，在协作层面的具体实现：

| 门控 | 触发条件 | 处理流程 |
|------|---------|---------|
| 招聘审批 | Agent 请求创建下属 | → 暂停工作流 → 通知管理员 → 等待批准/拒绝 |
| 预算门控 | 预计成本超阈值 | → 暂停 → 显示预估 vs 预算 → 等待审批 |
| 策略审批 | 管理层方案产出 | → 产出标记为待审核 → 人类查阅 → 批准/修订 |
| 模型升级 | Agent 请求更高级模型 | → 暂停 → 显示理由 → MaaS 管理员审批 |
| 异常升级 | 连续失败或异常行为 | → 自动冻结 Agent → 通知 → 人类介入诊断 |
| 质量门控 | 交付物不达标 | → 退回上游 + 附带不合格原因 → 或升级处理 |
| 知识审核 | Agent 写入知识库 | → 知识进入待审核 → 审核通过/驳回/修改 |

### 5.2 门控实现

```go
type ApprovalGateway struct {
    PendingApprovals map[string]ApprovalRequest
    Notifier         Notifier
}

type ApprovalRequest struct {
    ID          string
    WorkflowID  string
    StepID      string
    Type        ApprovalType
    RequesterID string          // 发起请求的 Agent
    Reason      string
    Context     ApprovalContext  // 完整的决策上下文
    Status      string           // pending | approved | rejected | timeout
    CreatedAt   time.Time
    Timeout     time.Duration
}

func (g *ApprovalGateway) RequestApproval(req ApprovalRequest) error {
    g.PendingApprovals[req.ID] = req

    // 通知决策者
    g.Notifier.Notify(req.ApproverID, Notification{
        Type:    "approval_required",
        Title:   fmt.Sprintf("%s 需要审批", req.Type),
        Body:    req.Context.Summary(),
        Actions: []string{"approve", "reject", "request_changes"},
        Timeout: req.Timeout,
    })

    // 超时处理
    go g.handleTimeout(req)

    return nil
}
```

<!-- @end-section -->

<!-- @section: agent-economy -->
## 六、Agent 经济层 — 借鉴 ATP 的可选增强

### 6.1 内部信用系统

借鉴 Evolver ATP 的经济层，但在 Legion 的组织内应用：

```go
type AgentCredit struct {
    AgentID        string
    Balance        float64          // 信用余额
    EarnedTotal    float64          // 累计赚取
    SpentTotal     float64          // 累计消费
    CreditScore    float64          // 信用评分 (0-100)
    TransactionLog []CreditTransaction
}

type CreditTransaction struct {
    FromAgent   string
    ToAgent     string
    Amount      float64
    Reason      string
    TaskID      string
    Timestamp   time.Time
}

// Agent 赚取信用的方式:
// - 按时高质量完成任务 → +N 信用
// - 创建的 Gene 被其他 Agent 使用 → +分成
// - 产出的知识被入库 → +奖励
//
// Agent 消费信用的方式:
// - 请求其他 Agent 的服务
// - 使用高级模型 (超过配额的部分)
// - 申请新建 Agent (招聘)
```

### 6.2 自动任务分配

区别于纯层级委派，经济层支持"内部市场"分配：

```
任务发布 → Agent 申领 → 基于能力 + 成本 + 信用评分选择

type TaskAuction struct {
    TaskID      string
    Description string
    Budget      float64
    Deadline    time.Time
    RequiredCapabilities []string
    Bids        []Bid
}

type Bid struct {
    AgentID     string
    Price       float64
    EstimatedHours int
    Confidence  float64  // 对完成质量的自信度
    PastPerformance float64 // 历史同类任务的表现
}
```

<!-- @end-section -->

<!-- @section: design-decisions -->
## 七、关键设计决策

| 决策 | 选择 | 理由 |
|------|------|------|
| 协作基础 | 组织架构 (层级) | 模拟真实企业，可治理 |
| 通讯模式 | 5 种模式 + 权限验证 | 覆盖全场景 + 安全可控 |
| 工作流编排 | DSL + 可视化 + 8 原语 | 灵活 + 用户友好 |
| 人类角色 | 7 种审批门控 | 三原则"可治理"落地 |
| 经济机制 | 可选增强 (非核心) | ATP 参考但不作为基础 |
| Agent 发现 | 组织树 + 能力声明 | 显式优于隐式 |
| 消息持久化 | 不可篡改审计链 | 三原则"可观测"落地 |

### 与现有方案的差异

| 维度 | hermes delegate_task | Legion 组织协作 |
|------|---------------------|---------------|
| 结构 | 扁平 (无层级) | 四级组织架构 |
| 通讯 | 单一委派 | 5 种模式 + 权限 |
| 流程 | 无编排 | 8 种原语 + DSL |
| 人类角色 | 仅 CLI 用户 | 决策者 + 审批节点 |
| 可观测 | 会话日志 | 完整审计链 |
| 可治理 | 无 | 7 种门控 |

### 不做什么

1. **不做扁平结构** — hermes 的 delegate_task 证明扁平结构缺少治理能力
2. **不做纯经济模型** — ATP 的交易市场太抽象，Legion 需要组织框架
3. **不做无人工介入** — 三原则的核心是可治理

<!-- @end-section -->

<!-- @section: related -->
## 相关文档

- [[../architecture/Legion|Legion 项目方案 — 多智能体协作]]
- [[../analysis/evolver/03-atp-protocol|Evolver ATP 交易协议]]
- [[../analysis/hermes/04-gateway-cli-deployment|hermes 网关与消息平台]]
- [[02-agent-runtime-deep-design|Agent 运行时深度设计]]
- [[06-security-governance|安全治理体系深度设计]]
- [[07-architecture-integration|系统集成架构深度设计]]

<!-- @end-section -->
