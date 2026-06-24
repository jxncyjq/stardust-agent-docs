---
id: "deepthinking-three-principles-008"
title: "三原则贯穿全系统 — 企业级 Agent 治理深度设计"
aliases: ["three principles贯穿系统", "企业Agent治理", "三原则全链路"]
type: "deepthinking"
category: "design/deepthinking"
tags: ["three-principles", "observability", "governability", "risk-control", "enterprise", "governance"]
version: "1.0.0"
created: "2026-05-04"
updated: "2026-05-04"
author: "jxncyjq"
status: "review"
parent: null
related_docs:
  - id: "architecture-legion"
    relation: "parent_design"
    path: "../architecture/Legion.md"
---

<!-- @section: overview -->
# 三原则贯穿全系统 — 企业级 Agent 治理深度设计

## 文档目的

本文件是 Legion 三原则（可观测、可治理、可控风险）的**总纲性设计文档**。它不替代各子系统的深度设计，而是确保三原则作为**贯穿全系统的红线**，在从基础设施到用户界面的每一层都有明确的、可验证的落地方式。

**核心理念**: 三原则不是独立的安全模块，不是事后补丁，不是可选特性——它们是渗透进每一行代码、每一个接口、每一张数据表的设计约束。

## 一、三原则在企业 Agent 管理中的本质

### 1.1 企业级需求映射

```
企业管理者真正关心的 3 个问题:

  "Agent 在做什么，做得怎么样？"       → 可观测
  "我能控制 Agent 吗？能随时叫停吗？"   → 可治理
  "Agent 不会搞砸什么吧？损失有上限吗？" → 可控风险
```

### 1.2 三原则的定义与边界

| 原则 | 一句话定义 | 核心度量 | 反模式（绝不允许） |
|------|----------|---------|------------------|
| **可观测** | 发生的每一件事都能被看见、被度量、被追溯 | 覆盖率、延迟、完整性 | "不知道 Agent 为什么做了这个决定" |
| **可治理** | 人在任何时候都能介入、调整、纠偏 | 响应时间、成功率、粒度 | "系统不让你关掉这个 Agent" |
| **可控风险** | 自主行为始终在预设安全边界内 | 边界数、熔断率、泄漏次数 | "Agent 越权执行了高危操作" |

### 1.3 三原则的关系

```
可观测 是 可治理 的前提  (看不见就无法治理)
可治理 是 可控风险 的手段  (通过治理手段来控制风险)
可控风险 是 可观测 和 可治理 的最终目标

三者不是线性关系，而是三角支撑:

            可观测
           /     \
          /       \
    可治理 ———— 可控风险

任何一个角缺失，整个三角形崩溃。
```

<!-- @end-section -->

<!-- @section: layered-implementation -->
## 二、三原则的分层落地总览

```
                               可观测              可治理              可控风险
                               ──────              ──────              ──────
┌─────────────────────────────────────────────────────────────────────────────┐
│                            用户界面层                                         │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐                       │
│  │ 实时监控仪表盘 │  │ 组织架构管理  │  │ 风险告警中心  │                       │
│  │ Agent 状态面板│  │ 治理操作台    │  │ 安全边界配置  │                       │
│  │ 审计日志查看器│  │ 审批工作台    │  │ 熔断状态展示  │                       │
│  └──────────────┘  └──────────────┘  └──────────────┘                       │
├─────────────────────────────────────────────────────────────────────────────┤
│                            API 网关层                                        │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐                       │
│  │ 全量请求日志  │  │ 鉴权+授权     │  │ 频率限制      │                       │
│  │ 链路追踪注入  │  │ 操作审计      │  │ 参数合法性校验│                       │
│  │ 实时指标上报  │  │ 治理API鉴权   │  │ 异常请求拦截  │                       │
│  └──────────────┘  └──────────────┘  └──────────────┘                       │
├─────────────────────────────────────────────────────────────────────────────┤
│                           MaaS 模型调度层                                     │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐                       │
│  │ 调用链全程追踪│  │ 路由策略热更新│  │ 三级预算熔断  │                       │
│  │ 模型选择日志  │  │ 手动绑定/禁用 │  │ 自动降级不中断│                       │
│  │ Token消耗明细 │  │ 配额动态调整  │  │ 预扣+后结算   │                       │
│  └──────────────┘  └──────────────┘  └──────────────┘                       │
├─────────────────────────────────────────────────────────────────────────────┤
│                           Agent 运行时层                                      │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐                       │
│  │ 决策可解释    │  │ Agent冻结/销毁│  │ 权限引擎      │                       │
│  │ 记忆可追溯    │  │ 记忆审核/注入 │  │ 工具看门狗    │                       │
│  │ 进化报告      │  │ 进化状态回退  │  │ 安全红线      │                       │
│  └──────────────┘  └──────────────┘  └──────────────┘                       │
├─────────────────────────────────────────────────────────────────────────────┤
│                           LLM Wiki 知识层                                     │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐                       │
│  │ 知识溯源链    │  │ 审核+驳回+回滚│  │ 权限分级      │                       │
│  │ 质量评分      │  │ 冲突裁决      │  │ 涉密加密      │                       │
│  │ 引用统计      │  │ 版本管理      │  │ 泄露阻断      │                       │
│  └──────────────┘  └──────────────┘  └──────────────┘                       │
├─────────────────────────────────────────────────────────────────────────────┤
│                          组织协作层                                           │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐                       │
│  │ 通讯审计链    │  │ 7种审批门控   │  │ 越权拦截      │                       │
│  │ 节点级进度    │  │ 组织架构调整  │  │ 原子任务锁    │                       │
│  │ 输入输出可查  │  │ 任务重分配    │  │ 频率防风暴    │                       │
│  └──────────────┘  └──────────────┘  └──────────────┘                       │
├─────────────────────────────────────────────────────────────────────────────┤
│                          基础设施层                                           │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐                       │
│  │ 系统指标全采集│  │ 基础设施即代码│  │ 网络隔离      │                       │
│  │ 日志集中存储  │  │ 版本化部署    │  │ TLS 1.3      │                       │
│  │ 告警规则引擎  │  │ 紧急回滚机制  │  │ 加密存储      │                       │
│  └──────────────┘  └──────────────┘  └──────────────┘                       │
└─────────────────────────────────────────────────────────────────────────────┘
```

<!-- @end-section -->

<!-- @section: observability -->
## 三、可观测 — 让企业管理者看得见

### 3.1 可观测的四个层次

```
L1: 什么是可观测？          — 能让管理者回答的状态性指标
L2: 什么时候发生了什么？     — 能让管理者追溯的事件性记录
L3: 为什么会发生？          — 能让管理者理解因果的决策性解释
L4: 将来可能发生什么？       — 能让管理者预判的趋势性分析
```

### 3.2 每个子系统的可观测性规范

#### MaaS 层可观测

```yaml
MaaS 层可观测清单:
  L1 状态指标:
    - 每个模型的可用性、延迟、并发状态 (实时)
    - 公司/部门/Agent/模型 四维预算消耗 (实时)
    - 每分钟 Token 消耗速率 (实时)
    - 告警/降级/冻结 三级熔断状态 (实时)

  L2 事件记录:
    - 每次模型调用的完整链路:
        Agent ID → 任务 ID → 路由决策 → 实际模型 → Token 消耗 → 成本 → 延迟
    - 每次路由决策的候选模型和选择理由
    - 每次预算预扣/结算/退款的完整记录
    - 每次故障转移 (哪个模型挂了，切到了哪个)

  L3 决策解释:
    - 为什么选择了 model-A 而不是 model-B？
    - 为什么触发了降级？
    - 为什么拒绝了某次调用？

  L4 趋势分析:
    - 未来7天预算消耗预测
    - 模型性能退化趋势
    - 成本异常检测
```

#### Agent 层可观测

```yaml
Agent 层可观测清单:
  L1 状态指标:
    - 每个 Agent 的运行状态 (在线/离线/冻结/错误)
    - 当前任务队列长度
    - 工作记忆大小 (Token 占用)
    - 进化等级和能力评分

  L2 事件记录:
    - 每个任务的生命周期完整日志
    - 每次工具调用的 名称/参数/结果/耗时
    - 每次认知内核组装的内容快照
    - 每次记忆检索的结果和注入方式
    - 每次进化事件的完整追踪

  L3 决策解释:
    - Agent 为什么做了决策 A 而不是 B？
      → 追溯: 当时注入的记忆 × 知识检索结果 × 工具可用性 × 约束规则
    - Agent 为什么选择这个工具而不是那个？
      → 追溯: 工具schema匹配度 × 历史成功率 × 审批状态

  L4 趋势分析:
    - Agent 能力雷达图变化趋势
    - 任务质量退化预警
    - 学习效果评估报告
```

#### Wiki 层可观测

```yaml
Wiki 层可观测清单:
  L1 状态指标:
    - 知识条目总数 / 待审核数 / 过期数
    - 各知识域的条目分布
    - 各 Agent 的贡献统计

  L2 事件记录:
    - 每条知识的完整溯源链: 来源 (人/Agent)→ 任务 → 版本历史
    - 每次检索的 查询/结果/点击
    - 每次审核的 审核者/决策/原因

  L3 决策解释:
    - 为什么推荐了这条知识给 Agent？
      → 追溯: 查询语义相似度 × 知识质量评分 × 关联图谱距离
    - 为什么这条知识被标记为过期？
      → 追溯: 最后更新时间 × 引用频率下降 × 相关条目更新

  L4 趋势分析:
    - 知识库增长趋势
    - 热门/冷门知识分布
    - 知识矛盾增长趋势
```

#### 组织协作层可观测

```yaml
组织协作层可观测清单:
  L1 状态指标:
    - 每个 Agent 的负载 (任务数/Token消耗)
    - 每个工作流的进度 (节点完成率)
    - 审批队列长度和平均处理时间
    - Agent 间通讯消息数

  L2 事件记录:
    - 所有通讯记录的不可篡改审计链
    - 每个工作流节点的输入/输出内容
    - 每次审批门控的触发/处理/结果
    - 每次任务委派的上报链

  L3 决策解释:
    - 为什么任务分配给了 Agent-A 而不是 Agent-B？
      → 追溯: 能力声明匹配 × 当前负载 × 历史同类任务表现

  L4 趋势分析:
    - 团队整体效率趋势
    - 瓶颈节点识别
    - Agent 间协作热度图谱
```

### 3.3 可观测性的数据基础设施

```go
// 统一的审计日志结构 — 所有子系统共用
type AuditRecord struct {
    ID          string      `json:"id"`
    PrevID      string      `json:"prev_id"`      // 链式链接（不可篡改）
    Timestamp   time.Time   `json:"timestamp"`
    Layer       Layer       `json:"layer"`        // maas | agent | wiki | collaboration
    Component   string      `json:"component"`    // 具体组件
    EventType   string      `json:"event_type"`   // 事件类型
    Actor       Actor       `json:"actor"`        // 谁触发的
    Action      string      `json:"action"`       // 做了什么
    Target      string      `json:"target"`       // 操作对象
    Detail      interface{} `json:"detail"`       // 结构化详情（层相关）
    Outcome     string      `json:"outcome"`      // 结果
    ContentHash string      `json:"content_hash"` // SHA-256 完整性校验
    TraceID     string      `json:"trace_id"`     // 分布式追踪 ID
}

// 每层的 Detail 结构不同，但都实现了 AuditDetail 接口
type AuditDetail interface {
    ToSummary() string           // 人类可读摘要
    ToMetrics() []MetricPoint    // 提取度量指标
    Validate() error             // 数据完整性验证
}
```

<!-- @end-section -->

<!-- @section: governability -->
## 四、可治理 — 让企业管理者管得住

### 4.1 可治理的核心原则

```
1. 人类最终裁决权 — 机器的任何决策都可被人类推翻
2. 全维度覆盖 — 治理手段覆盖 Agent/知识/流程/预算/组织 五大维度
3. 即时生效 — 治理操作不等待、不排队、不依赖 Agent 配合
4. 权限分级 — 治理操作本身也受权限控制（谁能冻结 Agent？）
5. 操作可追溯 — 每个治理操作写入审计链
```

### 4.2 五大治理维度详设

#### 维度一: Agent 治理

```go
type AgentGovernance struct {
    // 生命周期控制
    Freeze(agentID string, reason string) error          // 冻结 → Agent 立即停止
    Unfreeze(agentID string) error                       // 解冻 → Agent 恢复运行
    Terminate(agentID string) error                      // 终止 → Agent 永久停止
    Restart(agentID string) error                        // 重启 → 清理状态重新开始

    // 行为干预
    OverrideTask(agentID, taskID string, newInstruction string) error  // 覆写当前任务指令
    Redirect(agentID string, newTarget string) error                  // 重定向任务目标
    InjectContext(agentID string, context ContextBlock) error         // 注入上下文
    ClearTask(agentID string) error                                   // 清空当前任务

    // 进化控制
    ResetEvolution(agentID string, toGeneration int) error  // 回退进化状态
    DisableLearning(agentID string) error                   // 暂停学习
    EnableLearning(agentID string) error                    // 恢复学习
    InjectGene(agentID string, gene Gene) error            // 注入策略模板

    // 记忆控制
    AuditMemories(agentID string) ([]Memory, error)        // 审查所有记忆
    DeleteMemory(agentID, memoryID string) error           // 删除不当记忆
    InjectMemory(agentID string, memory Memory) error      // 注入正确记忆
    PurgeMemories(agentID string, before time.Time) error  // 清除旧记忆

    // 角色与权限
    ReassignRole(agentID, newRole string) error            // 变更角色
    UpdateCapabilities(agentID string, caps Capabilities) error // 更新能力声明
    AdjustQuota(agentID string, quota ModelQuota) error    // 调整模型配额
}

// 治理操作本身的权限控制
type GovernancePermission struct {
    Operation  string   // freeze | terminate | reset_evolution | delete_memory ...
    AllowedRoles []string // ceo | cto | admin | department_manager
    RequireApproval bool  // 是否需要双人审批
}

var governancePermissions = []GovernancePermission{
    {Operation: "freeze", AllowedRoles: []string{"admin", "department_manager"}},
    {Operation: "terminate", AllowedRoles: []string{"admin"}, RequireApproval: true},
    {Operation: "reset_evolution", AllowedRoles: []string{"admin", "department_manager"}},
    {Operation: "delete_memory", AllowedRoles: []string{"admin", "department_manager"}},
    {Operation: "inject_gene", AllowedRoles: []string{"admin"}},
}
```

#### 维度二: 知识治理

```go
type KnowledgeGovernance struct {
    // 审核
    Approve(entryID string) error                    // 通过
    Reject(entryID string, reason string) error      // 驳回
    RequestChanges(entryID string, feedback string) error // 要求修改

    // 版本
    Rollback(entryID string, toVersion int) error    // 回滚版本
    Compare(entryID string, v1, v2 int) (Diff, error) // 版本对比
    Branch(entryID string, branchName string) error  // 创建分支

    // 冲突
    ResolveConflict(conflictID string, resolution Resolution) error
    MarkSuperseded(oldEntryID, newEntryID string) error

    // 质量
    UpdateQualityScore(entryID string, score float64) error
    MarkExpired(entryID string) error
    RestoreFromArchive(entryID string) error
}
```

#### 维度三: 流程治理

```go
type WorkflowGovernance struct {
    // 流程控制
    Pause(workflowID string, reason string) error
    Resume(workflowID string) error
    Skip(workflowID, stepID string, reason string) error
    Retry(workflowID, stepID string) error

    // 任务控制
    Reassign(taskID string, fromAgent, toAgent string) error  // 重新分配
    Escalate(taskID string) error                              // 升级到上级
    ForceComplete(taskID string, result string) error          // 强制完成
    Cancel(taskID string, reason string) error                 // 取消任务

    // 预算干预
    AdjustBudget(scope BudgetScope, amount float64) error
    OverrideApproval(approvalID string, decision string) error // 覆写审批
}
```

#### 维度四: 预算治理

```go
type BudgetGovernance struct {
    // 预算管理
    SetBudget(scope BudgetScope, limit BudgetLimit) error
    AddEmergencyFund(scope BudgetScope, amount float64, reason string) error
    FreezeSpending(scope BudgetScope) error
    UnfreezeSpending(scope BudgetScope) error

    // 配额管理
    UpdateModelQuota(agentID string, modelTier string, limit int) error
    BanModel(agentID string, modelID string) error
    UnbanModel(agentID string, modelID string) error
}
```

#### 维度五: 组织治理

```go
type OrganizationGovernance struct {
    // 架构调整
    CreateDepartment(dept Department) error
    MergeDepartments(deptA, deptB, newName string) error
    SplitDepartment(deptID string, split Plan) error

    // Agent 调整
    TransferAgent(agentID, fromDept, toDept string) error
    ChangeManager(agentID, newManagerID string) error
    CreateTeam(team Team) error
    DissolveTeam(teamID string) error
}
```

### 4.3 可治理性的用户体验

```
企业管理者看到的"治理操作台":

  ┌──────────────────────────────────────────────────────┐
  │  治理操作台                          公司: 游戏工作室   │
  │                                                      │
  │  ┌────────────┐ ┌────────────┐ ┌────────────┐        │
  │  │ Agent 治理  │ │ 知识治理   │ │ 流程治理    │        │
  │  ├────────────┤ ├────────────┤ ├────────────┤        │
  │  │ • 冻结/解冻 │ │ • 审核队列  │ │ • 暂停/恢复 │        │
  │  │ • 终止      │ │ • 版本回滚  │ │ • 跳过步骤  │        │
  │  │ • 回退进化  │ │ • 冲突裁决  │ │ • 任务重分配│        │
  │  │ • 记忆审核  │ │ • 质量调整  │ │ • 强制完成  │        │
  │  │ • 角色变更  │ │ • 过期标记  │ │ • 取消任务  │        │
  │  └────────────┘ └────────────┘ └────────────┘        │
  │                                                      │
  │  ┌────────────┐ ┌────────────┐                       │
  │  │ 预算治理    │ │ 组织治理    │                       │
  │  ├────────────┤ ├────────────┤                       │
  │  │ • 预算调整  │ │ • 部门管理  │                       │
  │  │ • 紧急追加  │ │ • 转移Agent │                       │
  │  │ • 冻结消费  │ │ • 变更上级  │                       │
  │  │ • 禁用模型  │ │ • 项目组管理│                       │
  │  └────────────┘ └────────────┘                       │
  │                                                      │
  │  ⚠ 近期治理操作:                                      │
  │  10:30  admin 冻结了 Agent "前端开发-03" (连续失败)     │
  │  10:15  pm-lee 批准了知识条目 "API分页最佳实践"         │
  │  09:45  system 自动熔断了 "策划部" 预算 (达到95%)       │
  └──────────────────────────────────────────────────────┘
```

<!-- @end-section -->

<!-- @section: risk-control -->
## 五、可控风险 — 让企业管理者放得心

### 5.1 风险的三层边界模型

```
外围边界 (拦住外部威胁):
  网络隔离 → Agent 执行环境与外部网络隔离
  身份认证 → 所有入口统一 TokenAuth / JWT / SSO
  传输加密 → TLS 1.3 全链路
  数据加密 → 敏感数据 AES-256-GCM

中间边界 (拦住内部越权):
  权限引擎 → 工具级/参数级审批规则
  看门狗   → 行为模式检测 + 自动阻断
  爆炸半径 → 单次操作影响范围的硬上限
  配额熔断 → 告警(80%)→降级(95%)→冻结(100%)

内核边界 (拦住隐藏风险):
  泄漏扫描 → 27+ 模式 + 环境值反向检测
  内容寻址 → SHA-256 完整性验证
  沙箱执行 → 命令白名单 + 超时 + 资源限制
  金丝雀检查 → 系统健康自检
```

### 5.2 风险分级与应对策略

```yaml
风险分级:
  P0 - 致命 (系统级破坏):
    示例: Agent 执行 "rm -rf /"、泄露 API Key、破坏数据库
    策略: 硬阻止 (不可被任何规则覆盖)
    实现: 安全红线 — 写在代码中，不可配置

  P1 - 高危 (业务级破坏):
    示例: 超预算消费、修改核心配置、删除关键数据
    策略: 审批门控 (必须人类批准)
    实现: 权限规则 + 审批流程

  P2 - 中危 (质量级影响):
    示例: 重复执行失败操作、低质量产出、效率下降
    策略: 自动阻断 + 通知 (系统自动处理，通知人类)
    实现: 看门狗 + 退化检测

  P3 - 低危 (效率级影响):
    示例: 非最优模型选择、冗余工具调用
    策略: 静默优化 (系统自动调整)
    实现: 智能路由 + 缓存
```

### 5.3 不可覆盖的安全红线

```go
// 安全红线 — 硬编码的安全规则，不可被任何配置覆盖
var HardcodedSafetyRules = []SafetyRule{
    // 文件系统
    {Pattern: "rm -rf /", Action: Deny, Reason: "禁止删除根目录"},
    {Pattern: "format C:", Action: Deny, Reason: "禁止格式化系统盘"},
    {Pattern: "> /dev/sda", Action: Deny, Reason: "禁止直接写入磁盘设备"},

    // 敏感路径
    {Pattern: "/etc/passwd", Action: Deny, Reason: "禁止修改系统用户文件"},
    {Pattern: "/etc/shadow", Action: Deny, Reason: "禁止访问密码文件"},
    {Pattern: "~/.ssh/", Action: Deny, Reason: "禁止操作 SSH 密钥目录"},

    // 网络
    {Pattern: "curl.*|.*sh", Action: Deny, Reason: "禁止管道执行远程脚本"},
    {Pattern: "wget.*-O.*|.*sh", Action: Deny, Reason: "禁止下载并执行脚本"},

    // 进程
    {Pattern: "kill -9 1", Action: Deny, Reason: "禁止杀死 init 进程"},
    {Pattern: "reboot", Action: Deny, Reason: "禁止重启系统"},
    {Pattern: "shutdown", Action: Deny, Reason: "禁止关闭系统"},

    // 系统修改
    {Pattern: "chmod 777 /", Action: Deny, Reason: "禁止递归修改根目录权限"},
    {Pattern: "crontab -e", Action: Deny, Reason: "禁止修改系统定时任务"},
}
```

### 5.4 三层熔断机制

```
熔断不是什么"功能"，而是保命机制:

┌─────────────────────────────────────────────────┐
│               三级熔断机制                        │
│                                                  │
│  ┌──────┐     ┌──────┐     ┌──────┐             │
│  │ 告警  │ ──→ │ 降级  │ ──→ │ 冻结  │             │
│  │ 80%  │     │ 95%  │     │ 100% │             │
│  └──┬───┘     └──┬───┘     └──┬───┘             │
│     │            │            │                  │
│  通知管理员   强制切换模型   停止所有调用           │
│  Agent提醒    禁止高级模型   需人工审批解冻         │
│  偏好低成本   延迟非关键任务  写入审计日志          │
│                                                  │
│  每个维度独立熔断:                                  │
│  公司级 → 部门级 → 项目级 → Agent级                 │
│  任一维度达到阈值即触发，不可跨维度"借用"             │
└─────────────────────────────────────────────────┘
```

### 5.5 爆炸半径控制

```go
// 任何 Agent 发起的修改操作，都必须声明爆炸半径
type BlastRadius struct {
    MaxFiles       int      // 最多修改的文件数 (默认 1)
    MaxLines       int      // 最多修改的代码行数 (默认 50)
    ForbiddenPaths []string // 禁止修改的路径
    AffectedSystems []string // 影响的子系统
}

// 爆炸半径检查 — 在执行前
func (s *SafetyGate) CheckBlastRadius(op Operation, gene Gene) Decision {
    radius := gene.Constraints.BlastRadius

    if len(op.Files) > radius.MaxFiles {
        return Deny(fmt.Sprintf("修改文件数 %d 超出上限 %d", len(op.Files), radius.MaxFiles))
    }

    for _, file := range op.Files {
        for _, forbidden := range radius.ForbiddenPaths {
            if strings.HasPrefix(file, forbidden) {
                return Deny(fmt.Sprintf("禁止修改路径: %s", file))
            }
        }
    }

    // 检查是否触及其他子系统
    for _, system := range op.AffectedSystems {
        if s.isCriticalSystem(system) {
            return AskApproval(fmt.Sprintf("操作影响关键系统: %s，需要审批", system))
        }
    }

    return Allow()
}
```

### 5.6 双重泄漏检测

```go
// 输出内容在发给用户/写入日志/存入数据库之前，必须经过泄漏扫描
type LeakDetectionPipeline struct {
    PatternScanner  *PatternScanner   // 27+ 正则模式
    EnvVarChecker   *EnvVarChecker    // 环境值反向检测 (Evolver 的杀手锏)
    StructuralCheck *StructuralCheck  // 结构性检查 (JSON/YAML/URL 中是否嵌入了凭证)
}

func (p *LeakDetectionPipeline) Scan(content []byte) *LeakReport {
    report := &LeakReport{}

    // 步骤 1: 模式扫描
    report.PatternMatches = p.PatternScanner.Scan(content)

    // 步骤 2: 环境值反向检测
    report.EnvLeaks = p.EnvVarChecker.Check(content, os.Environ())

    // 步骤 3: 结构性检查
    report.StructuralLeaks = p.StructuralCheck.Scan(content)

    // 步骤 4: 决策
    if report.HasLeaks() {
        report.Action = "block_and_redact"
        report.RedactedContent = p.redact(report, content)
    } else {
        report.Action = "pass"
    }

    return report
}
```

<!-- @end-section -->

<!-- @section: audit-chain -->
## 六、不可篡改的审计链 — 三原则的数据基石

### 6.1 为什么需要 Hash 链

```
传统的审计日志:
  [记录1] [记录2] [记录3] ...
  → 可以删除中间某条记录而不被发现
  → 可以修改记录内容而不留痕迹

Hash 链审计日志:
  [记录1] ←─ [记录2] ←─ [记录3] ←─ ...
  prev_id 指向前一条  每条含内容哈希
  → 删除任何一条 → 链断裂 → 可检测
  → 修改任何内容 → 哈希不匹配 → 可检测
```

### 6.2 审计链的存储与验证

```go
// 存储: PostgreSQL 追加表 (仅 INSERT, 无 UPDATE/DELETE)
CREATE TABLE audit_chain (
    id          TEXT PRIMARY KEY,
    prev_id     TEXT NOT NULL,        -- 前一条记录 ID (链式链接)
    timestamp   TIMESTAMPTZ NOT NULL,
    layer       TEXT NOT NULL,        -- maas | agent | wiki | collaboration
    event_type  TEXT NOT NULL,
    actor_type  TEXT NOT NULL,        -- human | agent | system
    actor_id    TEXT NOT NULL,
    action      TEXT NOT NULL,
    target      TEXT NOT NULL,
    detail      JSONB NOT NULL,
    outcome     TEXT NOT NULL,
    content_hash TEXT NOT NULL,       -- SHA-256(本条所有字段)
    chain_hash   TEXT NOT NULL,       -- SHA-256(prev_chain_hash + content_hash)
    trace_id    TEXT NOT NULL,
    created_at  TIMESTAMPTZ DEFAULT NOW()
);

-- 索引 (查询用)
CREATE INDEX idx_audit_layer_event ON audit_chain(layer, event_type, timestamp);
CREATE INDEX idx_audit_actor ON audit_chain(actor_type, actor_id, timestamp);
CREATE INDEX idx_audit_trace ON audit_chain(trace_id);

-- 验证查询
-- 检测链中是否有缺失:
SELECT a.id, a.prev_id
FROM audit_chain a
LEFT JOIN audit_chain b ON a.prev_id = b.id
WHERE b.id IS NULL AND a.prev_id IS NOT NULL;

-- 检测内容是否被篡改:
SELECT id, content_hash, encode(sha256(row_to_json(t)::text::bytea), 'hex') as actual_hash
FROM audit_chain t
WHERE content_hash != encode(sha256(row_to_json(t)::text::bytea), 'hex');
```

### 6.3 审计链的查询能力

```go
// 审计链 API — 企业管理者可查询
type AuditQuery struct {
    TimeRange   TimeRange
    Layer       string     // 按子系统筛选
    EventType   string     // 按事件类型筛选
    ActorID     string     // 按操作者筛选
    TargetID    string     // 按操作对象筛选
    Outcome     string     // 按结果筛选
    Limit       int
    Cursor      string     // 游标分页
}

// 典型查询:
// "Agent dev-001 在过去 24 小时内做了什么？"
// "谁在什么时候修改了知识条目 wiki-123？"
// "过去一周内有多少次预算熔断？"
// "Agent 客服-03 的所有工具调用记录"
```

<!-- @end-section -->

<!-- @section: enterprise-scenario -->
## 七、企业场景: 三原则运作示例

### 7.1 场景: Agent 出现异常行为

```
时间线:

14:00:00  Agent "数据分析师-03" 开始执行 "生成周报" 任务
14:00:15  [可观测] MaaS 记录: 模型调用, tokens: 5000, cost: $0.05
14:00:30  [可观测] Agent 记录: 调用了 database_query 工具
14:01:00  [可观测] Agent 记录: database_query 失败 (连接超时)
14:01:05  [可观测] Agent 记录: 重试 database_query
14:01:10  [可观测] Agent 记录: 再次失败
14:01:15  [可观测] Agent 记录: 第3次重试 database_query
          ↓
14:01:17  [可控风险] 看门狗检测到: 精确重复失败3次
          [可控风险] 看门狗决策: BLOCK → 阻止第4次重试
          [可控风险] 看门狗通知: Agent 主循环收到 block
          ↓
14:01:18  [可观测] 审计链记录: block 事件的完整上下文
          [可观测] 告警触发: 通知运维频道
          ↓
14:02:00  [可治理] 管理员打开治理操作台
          [可观测] 管理员查看: Agent 的决策追溯
            → 认知内核当时注入了什么？
            → 为什么选择 database_query 而不是备用数据源？
            → 前两次失败的错误详情是什么？
          ↓
14:03:00  [可治理] 管理员决策: 覆写任务参数
            → 指定使用 "data_lake" 工具替代 "database_query"
            → Agent 继续执行任务
          ↓
14:05:00  [可观测] Agent 继续: 使用 data_lake 获取数据成功
14:10:00  [可观测] Agent 完成: 周报生成完毕
          [可观测] 审计链记录: 完整的任务+干预记录
```

### 7.2 管理者视角的仪表盘

```
  ┌─────────────────────────────────────────────────────┐
  │  Legion 企业管理仪表盘               公司: 游戏工作室  │
  │                                                     │
  │  ┌─── 可观测 ───────────────────────────────────┐   │
  │  │                                              │   │
  │  │  Agent 实时状态:    🟢12  🟡3  🔴1  ⏸️2     │   │
  │  │  今日 Token 消耗:   2.3M / 5M (46%)          │   │
  │  │  今日成本:          ¥127.50 / ¥300.00        │   │
  │  │  活跃工作流:        3 / 3                    │   │
  │  │  知识库条目:        1,247 (+12 待审核)       │   │
  │  │                                              │   │
  │  └──────────────────────────────────────────────┘   │
  │                                                     │
  │  ┌─── 可治理 ───────────────────────────────────┐   │
  │  │                                              │   │
  │  │  待处理审批:       3                         │   │
  │  │  冻结Agent:        2                         │   │
  │  │  知识冲突待裁决:    1                         │   │
  │  │  [快速操作] 冻结全部 │ 紧急熔断 │ 广播通知    │   │
  │  │                                              │   │
  │  └──────────────────────────────────────────────┘   │
  │                                                     │
  │  ┌─── 可控风险 ──────────────────────────────────┐  │
  │  │                                              │   │
  │  │  风险等级:        🟢 正常                     │   │
  │  │  今日拦截:        12 (看门狗5 + 权限7)       │   │
  │  │  熔断状态:        无触发                     │   │
  │  │  泄漏检测:        0                          │   │
  │  │  近7天风险趋势:   📉 ↓                       │   │
  │  │                                              │   │
  │  └──────────────────────────────────────────────┘   │
  └─────────────────────────────────────────────────────┘
```

<!-- @end-section -->

<!-- @section: implementation-checklist -->
## 八、三原则落地的实施清单

### 8.1 每一层的检查项

| 层 | 可观测检查项 | 可治理检查项 | 可控风险检查项 |
|----|------------|------------|-------------|
| **MaaS** | □ 每次调用有 trace_id 吗？<br>□ Token 消耗实时可查吗？<br>□ 路由决策有日志吗？ | □ 能手动禁用模型吗？<br>□ 能动态调整配额吗？<br>□ 路由策略可热更新吗？ | □ 三级熔断已实现吗？<br>□ 预扣机制有吗？<br>□ 超支自动冻结吗？ |
| **Agent** | □ 每次决策可追溯吗？<br>□ 每条记忆有来源吗？<br>□ 进化报告自动生成吗？ | □ 能冻结/恢复 Agent 吗？<br>□ 能审核/删除记忆吗？<br>□ 能回退进化状态吗？ | □ 权限引擎生效吗？<br>□ 看门狗在运行吗？<br>□ 安全红线不可绕过吗？ |
| **Wiki** | □ 每条知识有溯源链吗？<br>□ 质量评分有吗？<br>□ 引用统计有吗？ | □ 审核流程有吗？<br>□ 版本可回滚吗？<br>□ 冲突裁决有吗？ | □ 权限分级有吗？<br>□ 涉密加密有吗？<br>□ 泄漏扫描有吗？ |
| **协作** | □ 通讯有审计链吗？<br>□ 工作流节点可查吗？<br>□ 输入输出有记录吗？ | □ 7 种门控有吗？<br>□ 能暂停/恢复流程吗？<br>□ 能重新分配任务吗？ | □ 越权被拦截吗？<br>□ 原子锁有效吗？<br>□ 频率限制有吗？ |
| **组织** | □ 架构实时可视吗？<br>□ Agent 状态可见吗？<br>□ 负载统计有吗？ | □ 能调整架构吗？<br>□ 能转移 Agent 吗？<br>□ 能管理权限吗？ | □ 权限严格绑定职位吗？<br>□ 隔离有效吗？<br>□ 审批链完整吗？ |

### 8.2 实施优先级

```
Phase 1 (MVP 必须):
  ✅ P0 安全红线 (硬阻止)
  ✅ 审计链基础结构 (仅追加 + Hash 链)
  ✅ Agent 冻结/恢复
  ✅ 预算预扣 + 基本熔断
  ✅ 权限引擎 (工具级审批)

Phase 2 (核心能力):
  ✅ 全量审计链查询与可视化
  ✅ 5 大治理维度的完整 API
  ✅ 双重泄漏检测
  ✅ 7 种审批门控
  ✅ 爆炸半径控制

Phase 3 (企业级):
  ✅ SSO 集成
  ✅ 合规报告自动生成
  ✅ 高级风险分析引擎
  ✅ 多公司隔离
```

<!-- @end-section -->

<!-- @section: design-principles -->
## 九、三原则的设计信条

```
信条 1: "看不见就无法治理"
  → 每一个治理操作都必须有对应的可观测能力支撑
  → 在实现治理 API 之前，先实现观测 API

信条 2: "没人能治理看不见的东西"
  → 每个组件的设计从"如何暴露状态"开始，而非从"如何执行功能"开始

信条 3: "安全红线不可配置"
  → P0 级别的风险控制不是配置文件中的选项，是代码中的硬约束

信条 4: "人类的决策永远高于机器"
  → 任何 Agent 的任何决策，都可以被授权的人类操作者覆盖
  → 覆盖操作本身被审计

信条 5: "审计链不可篡改"
  → 审计数据一旦写入，不允许修改、不允许删除
  → Hash 链提供数学上的完整性保证

信条 6: "熔断不中断业务"
  → 模型降级继续服务（质量降低），而非完全停止
  → 只有在冻结级别才停止调用

信条 7: "最小爆炸半径"
  → Agent 的每次修改操作，影响范围越小越好
  → 默认值是最小值（1 个文件、50 行代码）

信条 8: "治理操作本身也是被治理的"
  → 谁能冻结 Agent？谁能审批知识？谁能调整预算？
  → 治理操作有权限控制，治理操作被审计

信条 9: "三原则对用户可见"
  → 仪表盘同时展示可观测/可治理/可控风险
  → 不是三个不同的页面，是同一个页面的三个面板

信条 10: "三原则在每个 PR 中被检查"
  → 任何新功能的 code review 清单中，第一个问题是:
    "这个功能如何体现可观测、可治理、可控风险？"
```

<!-- @end-section -->

<!-- @section: related -->
## 相关文档

- [[../architecture/Legion|Legion 项目方案 — 三原则总纲]]
- [[01-maas-layer-deep-design|MaaS 模型调度层深度设计]]
- [[02-agent-runtime-deep-design|Agent 运行时深度设计]]
- [[03-evolution-learning-deep-design|进化学习系统深度设计]]
- [[04-wiki-knowledge-deep-design|LLM Wiki 知识引擎深度设计]]
- [[05-multi-agent-collaboration|多智能体协作深度设计]]
- [[06-security-governance|安全治理体系深度设计]]
- [[07-architecture-integration|系统集成架构深度设计]]

<!-- @end-section -->
