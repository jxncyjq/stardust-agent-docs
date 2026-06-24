---
id: "design-multi-agent-001"
title: "Multi-Agent 架构设计"
type: "design"
category: "agents/specs"
tags: ["multi-agent", "design", "architecture"]
version: "1.0.0"
created: "2026-06-24"
updated: "2026-06-24"
author: "jxncyjq"
status: "published"
parent: null
children: []
related_docs:
  - id: "spec-multi-agent-implementation-clarification-2026-05-18"
    relation: "related_to"
    path: "./2026-05-18-multi-agent-implementation-clarification.md"
---

# Multi-Agent 架构设计

**日期**：2026-05-18  
**状态**：已修订（v2）  
**范围**：Legion Agent — 扩展 Workflow Engine 支持 per-agent 配置

---

## 背景与目标

Legion Agent 的 Workflow Engine 已支持 `parallel`/`sequence`/`agent_task` 等节点，可在单进程内并发调度多个 task。但目前所有 task 共享同一个 `Runtime`（同一 SOUL/MEMORY/ContextPrefix/ToolRoot），无法让不同 task 使用不同 agent 的配置（人格、记忆、模型、workspace）。

目标：通过扩展 `agentregistry` + `Coordinator`，使 `agent_task` 节点的 `agent_id` 字段能路由到独立的 agent 配置，实现真正的多 agent 角色分工。

---

## 现有调用链分析

```
Workflow Engine.executeAgentTask
  → scheduler.Add(task{AgentID: "researcher"})   // AgentID 来自 TaskSpec

BackgroundScheduler.RunOnce → Coordinator.Heartbeat
  → scheduler.Next(ctx, c.agent.ID)              // ⚠️ 此处覆写 task.AgentID！
  → c.runtime.RunTask(ctx, c.agent, task)         // 单一固定 Runtime，无 per-agent 切换
```

**两个关键 bug**：
1. `Scheduler.Next()` 将 `task.AgentID` 覆写为 coordinator 的 agentID，丢失 Workflow 的路由意图
2. `Coordinator` 持有单一 `*Runtime`，无法为不同 agent 使用不同 ContextPrefix / MaasClient / ToolRoot

---

## 配置结构

### 主配置 `agent.json` 新增 `agents` 字段

```json
{
  "agents": {
    "researcher": "agents/researcher.json",
    "writer":     "agents/writer.json"
  }
}
```

- key 对应 `TaskSpec.agent_id`（Workflow 节点中指定的 agent 名）
- value 是子 agent 配置文件路径（相对于主配置文件目录）
- 主 agent 本身不需要在此注册，它的配置来自主 `agent.json`

### 子 agent 配置文件（如 `agents/researcher.json`）

```json
{
  "id": "researcher",
  "role": "researcher",
  "maas_profile": "dev",
  "context_files": {
    "root": ".",
    "soul_path": "configs/personas/researcher/SOUL.md",
    "memory_path": "configs/personas/researcher/MEMORY.md",
    "tools_path": "configs/personas/researcher/TOOLS.md",
    "agents_path": "AGENTS.md",
    "user_path": "configs/persona/USER.md",
    "max_file_chars": 20000
  },
  "workspace": {
    "docs_root": "docs/research",
    "memory_root": "memory/researcher"
  },
  "skills": {
    "install_root": "skills/researcher"
  }
}
```

**继承规则**：`maas.base_url` / `maas.api_key`、`storage`、`server`、`runtime.max_tool_rounds` 继承自主配置。`maas_profile` 指向主配置 `maas.profiles` 中的 key。

---

## 代码修改点

```
internal/
  config/
    config.go           // Config 新增 Agents map[string]string（修改）

  agentregistry/        // 新建包
    registry.go         // AgentRegistry: Load / Get / Names
    config.go           // AgentConfig struct

  task/
    scheduler.go        // Next() 修改：不覆写已有 AgentID（修改）

  runtime/
    coordinator.go      // Coordinator 新增 AgentRegistry 字段，Heartbeat 按 AgentID 路由（修改）

  cli/
    command.go          // serve 命令启动时加载 AgentRegistry，注入 Coordinator（修改）
```

---

## AgentRegistry

```go
// AgentConfig 子 agent 配置
type AgentConfig struct {
    ID           string                     `json:"id"`
    Role         string                     `json:"role"`
    MaasProfile  string                     `json:"maas_profile"`
    ContextFiles config.ContextFilesConfig  `json:"context_files"`
    Workspace    config.WorkspaceConfig     `json:"workspace"`
    Skills       config.SkillsConfig        `json:"skills"`
}

// Registry 管理已注册的 agent 配置
type Registry struct {
    agents  map[string]AgentConfig
    rootCfg config.Config
}

func Load(ctx context.Context, rootCfg config.Config, configDir string) (*Registry, error)
func (r *Registry) Get(name string) (AgentConfig, bool)
func (r *Registry) Names() []string
```

`Load` 读取 `rootCfg.Agents` 映射，以 `configDir`（主配置文件所在目录）为基准解析相对路径，逐一加载子 agent json。

---

## Scheduler.Next() 修改

**当前行为**（第 53 行）：
```go
task.AgentID = agentID  // 覆写，丢失 Workflow 设置的 AgentID
```

**修改后**：只在 `task.AgentID` 为空时才赋值：
```go
if task.AgentID == "" {
    task.AgentID = agentID
}
```

---

## Coordinator 修改

`CoordinatorConfig` 新增可选字段：

```go
type CoordinatorConfig struct {
    // ...existing fields...
    AgentRegistry *agentregistry.Registry  // 可选；nil 时行为与现在相同
    MaasFactory   func(profile string) port.MaasInferenceClient  // 按 profile 创建 client
}
```

`Heartbeat()` 执行逻辑修改：

```
1. scheduler.Next → taskToRun（AgentID 已被 Workflow 设置，不再被覆写）

2. if c.agentRegistry != nil && taskToRun.AgentID != "" {
       agentCfg, ok = c.agentRegistry.Get(taskToRun.AgentID)
   }

3. if ok:
       contextPrefix = buildContextPrefix(agentCfg, rootCfg)
       maasClient    = c.maasFactory(agentCfg.MaasProfile)
       toolRoot      = agentCfg.ContextFiles.Root
       agentDomain   = domain.Agent{ID: agentCfg.ID, Role: agentCfg.Role}
       rt = runtime.NewRuntime(Config{Maas: maasClient, ContextPrefix: contextPrefix, ToolRoot: toolRoot, ...})
   else:
       rt = c.runtime  // fallback：使用默认 Runtime

4. run, err = rt.RunTask(ctx, agentDomain, taskToRun)
```

---

## 数据流

```
POST /v1/workflows
  {agent_task: {agent_id: "researcher", input: "调研 X"}}
  {agent_task: {agent_id: "writer",     input: "写报告"}}

  → Workflow Engine.executeAgentTask
      → scheduler.Add(task{AgentID: "researcher", Input: "调研 X"})
      → scheduler.Add(task{AgentID: "writer",     Input: "写报告"})

  → Coordinator.Heartbeat (定时触发)
      → scheduler.Next → task{AgentID: "researcher"} (AgentID 不被覆写)
      → agentRegistry.Get("researcher") → AgentConfig
      → buildContextPrefix(researcherCfg)   // researcher SOUL/MEMORY
      → runtime.NewRuntime(researcherMaas, researcherToolRoot)
      → runtime.RunTask → result

  → 下一次 Heartbeat
      → scheduler.Next → task{AgentID: "writer"}
      → agentRegistry.Get("writer") → AgentConfig
      → runtime.NewRuntime(writerMaas, writerToolRoot)
      → runtime.RunTask → result
```

**并行**：`parallel` 节点已通过 goroutine 并发调度多个 task，每个 task 各自对应不同 agent 配置，天然支持。

---

## 错误处理

| 场景 | 处理 |
|------|------|
| `agent_id` 未在注册表中 | fallback 使用默认 Runtime，记录 warn 日志 |
| 子 agent 配置文件不存在 | `Load` 时返回错误，`serve` 启动失败 |
| 子 agent 执行失败 | task 状态 → `failed`，workflow 节点标记 failed |
| `AgentRegistry` 为 nil | 行为与现在完全相同（向后兼容） |

---

## 本阶段范围

**包含：**
- `config.Config` 新增 `Agents map[string]string`
- `agentregistry` 包（Load / Get / Names）
- `Scheduler.Next()` 修改（不覆写已设置的 AgentID）
- `Coordinator` 新增 `AgentRegistry` + `MaasFactory` + per-agent Runtime 构建
- `cli/command.go` 的 `serve` 命令注入 AgentRegistry
- 示例配置文件（agents/researcher.json / agents/writer.json）

**不包含（后期扩展）：**
- TUI 的 `delegate_task` 工具（单独需求）
- 进程隔离 / HTTP 模式
- 子 agent 的 sub-subagent

---

## 测试策略

- `agentregistry`: 正常加载 / 文件缺失 / key 不存在 fallback
- `task/scheduler`: `Next()` 不覆写已设置的 AgentID
- `runtime/coordinator`: mock AgentRegistry，验证 per-agent Runtime 选择
- `cli/command.go`: serve 命令集成测试，验证 AgentRegistry 注入
