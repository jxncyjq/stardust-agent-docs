---
id: "analysis-pi-insights-007"
title: "pi-agent 洞察与 Legion 参考"
aliases: ["pi insights", "pi 对 Legion 的启示", "pi vs legion"]
type: "design"
category: "design/analysis/pi-agent"
tags: ["pi-agent", "analysis", "insights", "legion", "design-reference"]
version: "1.0.0"
created: "2026-08-15"
updated: "2026-08-15"
author: "jxncyjq"
status: "review"
parent: "analysis-pi-index"
children: []
related_docs:
  - id: "analysis-pi-harness-003"
    relation: "depends_on"
    path: "./03-harness.md"
  - id: "analysis-pi-extensions-006"
    relation: "depends_on"
    path: "./06-extensions-mcp.md"
---

# pi-agent 洞察与 Legion 参考

<!-- @section: patterns -->
## 一、可直接提炼的设计模式

### 1. 内核不感知能力来源

`packages/agent` 只认识 `AgentTool`、`AgentMessage`、`StreamFn`；内置工具、扩展工具、SDK 工具在它眼里完全一样。所有「这个工具从哪来」的信息压进 `sourceInfo`，只服务于 UI 与诊断。

**对 Legion**：工具注册表应当只吃一种接口，来源（内置 / 配置 / 远端 / 插件）作为元数据挂在旁边，而不是分叉执行路径。Legion 现有的 per-agent 工具授权（`toolauth.gateable` 登记 + `disabled_tools` 勾选）已经是这个方向，但要守住一条纪律：**新增工具必须同时登记到能力目录**，否则授权面与执行面会漂移——pi 用 `promptSnippet` 缺失即不出现在清单里，把这类漂移变成「静默降级」而不是「静默越权」。

### 2. 工具的三副面孔要拆开

模型可见（name/description/schema）、UI 可见（label/renderCall/renderResult）、提示词可见（promptSnippet/promptGuidelines）是三组独立字段，在三个不同层被消费。

**对 Legion**：如果工具描述、前端展示名、系统提示词里的说明目前挤在同一个结构体里，拆开的收益是——**换 UI 不动模型契约，改提示词不动工具实现**。

### 3. 「给模型截断版 + 落盘完整版 + 告知路径」

`BashToolDetails.fullOutputPath`：超限输出写文件，结果里返回路径，模型可自行 `read` 续读；`read` 工具的描述里直接写明「输出截断到 2000 行/50KB，用 offset 续读，需要全文就一直续到完」。

**对 Legion**：与已交付的截断/分页治理方向一致，可补齐的是**把「怎么拿到剩下的」写进工具 description 本身**，而不是只写在 footer 里——模型读 description 的概率远高于读上一次的 footer。

### 4. 并行执行 + 确定性记录

preflight 串行 → 执行并发 → `tool_execution_end` 按完成序 → **工具结果消息回到源序**。

**对 Legion**：任何并行工具执行都必须保证会话记录顺序确定，否则回放、压缩、审计三处全部不可复现。这条排序规则可以原样搬。

### 5. `stopReason == "length"` 全批拒绝

被输出上限截断的助手消息里，工具参数可能**解析成功且校验通过但内容残缺**。pi 的处理是整批返回错误让模型重发。

**对 Legion**：这是一条几乎必然会踩、但很难自己想到的规则。建议直接加进工具执行前置检查。

### 6. `addedToolNames`：把工具集变更点记进会话

工具集不是会话级常量。执行结果里声明「此后新增了哪些工具」，回放时才能还原当时的工具视图。

**对 Legion**：per-agent 动态授权 + 会话恢复的组合下，这个字段是必需品，否则「历史会话重放」与「当时实际可用工具」会对不上。

### 7. 陈旧句柄的教学式失败

`invalidate()` + `assertActive()` + 一段解释「你为什么拿到这个错、正确写法是什么」的长错误信息 + 自动退订事件。

**对 Legion**：会话替换/重载/热更新场景下，把「悬空句柄」做成自解释的运行时错误，比写在文档里有效一个数量级。这与 Legion 已有的 fail-loud 纪律同向。

### 8. 规范先行、可删除的实现

`packages/agent/docs/harness.md` 2941 行规范，明确写着「`src/harness/**` 及其全部测试可直接删除」「**已有测试是证据，不是权威**」，并把交付切成 16 个带验收测试清单的 slice、分成两条互不阻塞的轨道。

**对 Legion**：大型重构（例如记忆分层、会话持久化重做）值得先写这种级别的规范，尤其是**每个 slice 必须测「正常路径 + 每个新状态 + 每个崩溃边界 + 每个竞态的两种顺序」**这条验收标准。
<!-- @end-section -->

<!-- @section: durability -->
## 二、最有价值的一块：可恢复执行模型

Legion 已经做过任务状态落盘（字段级写穿、先落盘后改内存）。pi 的下一代 harness 规范把这件事推到了更彻底的位置，几条可直接借鉴的公理：

| 规范条款 | 对 Legion 的意义 |
|----------|------------------|
| **三仓：不可变 entries / 可变 registers / append-only 用量账本** | 可变状态被隔离到一处，恢复不需要重放历史 |
| **每份 payload 只存在于一个地方** | 杜绝「内存一份、DB 一份、JSON 里还有一份」的漂移 |
| **配置与编排永不进入会话树**（删光 `op.*` 后仍是合法会话） | 会话数据与运行时状态解耦，导出/迁移/审计都干净 |
| **operation 状态无 `finished` 成员**，结束即删除状态、结果落 `lastResult` | 用类型消灭「完成了但状态还在」的中间态 |
| **`settings` 在操作被接受时原子快照** | 运行中改配置不撕裂当前运行 |
| **`AgentTool.replay?: "never" \| "safe"`** | 「工具是否可重放」是工具自己的声明，不是恢复逻辑里的 if-else |
| **`SuspendedOperation.missing: { tools, models }`** | 恢复失败要能区分「代码问题」与「当时那个工具/模型现在没了」 |
| **overflow 恢复只用一次（`overflowRecoveryUsed`）** | 防「溢出→压缩→再溢出」死循环，与 Legion 的 per-tool 熔断同构 |
| **hook 的持久输出在继续执行前先提交** | 「返回了」≠「持久了」；提交前崩溃必须能安全重跑 |
| **显式承认「精确一次」是非目标** | `before_run_end` 可能重复触发，需要幂等的处理器自带持久标记 |

最后一条尤其重要：**与其假装 exactly-once，不如写清楚哪里 at-least-once 并要求幂等。**
<!-- @end-section -->

<!-- @section: mcp -->
## 三、No-MCP 这件事该怎么看

pi 的论证链：MCP 的价值在跨进程/跨语言/跨厂商互操作；如果扩展点本来就同语言同进程，这层协议只剩序列化与生命周期开销。于是用两条本地通路取代：

- **Skills**：能力 = 一个目录（`SKILL.md` 描述 + 脚本/资源）。模型靠描述决定是否 `read` 全文——**渐进式披露**，上下文常驻成本只有一句话。
- **Extensions**：能力 = 进程内 TypeScript，可注册工具、拦 11+ 类事件、换 provider、改 UI。

**代价必须一起记账**：

1. **生态不通**。别人写的 MCP server 用不了，除非有人写桥接扩展。pi 用「兼容其他 harness 的技能目录」（`~/.claude/skills`、`~/.codex/skills` 直接配进 `settings.skills`）部分对冲——技能目录成了事实上的跨 harness 交换格式。
2. **安全模型退化**。MCP server 是独立进程、可独立限权；pi 扩展在宿主进程内跑任意代码，安全模型只剩「读代码 + 容器」。
3. **同语言绑定**。扩展必须是 TS/JS。pi 的回答是「那就写 CLI 工具 + 技能」——语言无关的能力通过**进程边界 + 文本协议（bash）**接入，而不是 RPC 协议。

**对 Legion 的判断建议**（Legion 是 Go，天然没有「同语言进程内插件」这条便宜路）：

- **技能这一层应当无条件学**：`SKILL.md` + 描述注入 + 按需 read，是纯提示词工程，零协议成本，且能直接复用 Anthropic/社区技能仓库。Legion 已有 AGENTS.md 分层加载，技能是它的正交补充——AGENTS.md 是「常驻规则」，技能是「按需工作流」。
- **扩展这一层不宜照抄**：Go 没有等价的运行时加载 TS 的能力，硬做 plugin（`.so`）在跨平台与版本兼容上代价极高。Legion 的等价物应该是 **① 内置工具（Go 实现，走 gateable 授权）+ ② 子进程 CLI 工具（技能里描述用法）+ ③ MCP 客户端（用来消费外部生态）**。
- **也就是说：pi 的 No-MCP 结论不可移植，但它 No-MCP 的理由可移植**——能力接入的关键不是协议，而是「描述如何进上下文」和「执行如何被授权与记录」。Legion 若已有/将有 MCP 客户端，也应当让 MCP 工具走**与内置工具完全相同**的注册表、授权、截断、审计路径（见第一节第 1 条）。
<!-- @end-section -->

<!-- @section: caution -->
## 四、不建议照搬的部分

| 项 | 原因 |
|----|------|
| 扩展进程内全权限、无沙箱 | pi 的用户是单机开发者；Legion 面向服务端/多租户，同样的模型等于把宿主拱手让人 |
| 「同一语义两份实现」的迁移期 | 技能加载器在内核与产品层各有一份，**父目录名校验规则已经分歧**。迁移期允许并存，但必须有收敛期限与差异清单 |
| 3344 行的 `agent-session.ts` | 编排层什么都管（工具/扩展/模型/压缩/导出/命令/UI 绑定）。pi 自己用下一代 harness 在拆它，Legion 不必先走一遍这条弯路 |
| 拒绝内建 TODO / plan mode 等 | 这是「面向能自己写扩展的用户」的产品选择。Legion 的使用者若不写代码，「一切靠扩展」等于「一切都没有」 |
| 工具参数原地 mutate 且不重新校验 | 现役实现的 `tool_call` 允许扩展改 `event.input` 后直接执行。pi 自己在规范版收紧为「返回新参数 + 重新校验」——直接学收紧后的版本 |
<!-- @end-section -->

<!-- @section: actions -->
## 五、给 Legion 的候选动作项

按「收益/成本」排序，均为建议，未评估实施：

1. **技能层（高收益，低成本）**：实现 `SKILL.md` 发现 + 描述注入 + `/skill:name` 显式调用；发现规则、frontmatter 校验、collision 先到者胜、`disable-model-invocation`、项目级技能受信任门控，可直接对齐 pi。
2. **工具截断的自描述（低成本）**：把「如何续读」写进工具 description；超限输出落盘并返回路径。
3. **`length` 截断全批拒绝（低成本）**：工具执行前置检查加一条。
4. **工具集变更点入会话（中成本）**：等价于 `addedToolNames`，动态授权场景的回放正确性依赖它。
5. **可恢复执行模型（高成本，高价值）**：先落「三仓 + 操作状态整体覆写 + 终结事务原子清理」的最小版本；`replay: never|safe` 作为工具契约的一部分同期引入。
6. **并行工具执行的顺序契约（中成本）**：若/当 Legion 开并行工具，直接采用 preflight 串行 / 执行并发 / 结果回源序。
7. **陈旧句柄治理（低成本）**：会话替换后旧句柄一律 fail-loud 且给出教学式错误。
<!-- @end-section -->

## 相关文档

- [[analysis-pi-overview-001|项目架构总览]]
- [[analysis-pi-harness-003|Harness 分层与下一代规范]]
- [[analysis-pi-tools-004|工具系统与装载机制]]
- [[analysis-pi-skills-005|技能系统]]
- [[analysis-pi-extensions-006|扩展系统与 No-MCP 立场]]
