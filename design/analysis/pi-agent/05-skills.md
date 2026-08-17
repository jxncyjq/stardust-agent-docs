---
id: "analysis-pi-skills-005"
title: "pi-agent 技能系统与渐进式披露"
aliases: ["pi skills", "SKILL.md", "Agent Skills 标准", "pi 技能加载"]
type: "design"
category: "design/analysis/pi-agent"
tags: ["pi-agent", "analysis", "skills", "progressive-disclosure", "prompt"]
version: "1.0.0"
created: "2026-08-15"
updated: "2026-08-15"
author: "jxncyjq"
status: "review"
parent: "analysis-pi-index"
children: []
related_docs:
  - id: "analysis-pi-tools-004"
    relation: "related_to"
    path: "./04-tools.md"
  - id: "analysis-pi-extensions-006"
    relation: "related_to"
    path: "./06-extensions-mcp.md"
---

# pi-agent 技能系统与渐进式披露

<!-- @section: model -->
## 核心模型：只注入描述，按需读全文

pi 实现 [Agent Skills 标准](https://agentskills.io/specification)。一个技能 = 一个目录 + 一个 `SKILL.md`（YAML frontmatter + 正文），目录里其余内容完全自由（脚本、参考文档、模板）。

运行链路（`docs/skills.md` 原文四步）：

1. 启动时扫描技能位置，抽取 `name` / `description` / `location`
2. system prompt 注入 XML 区块（只有元数据，**没有正文**）
3. 任务匹配时，模型用 `read` 工具加载完整 `SKILL.md`
4. 模型按指示执行，相对路径按技能目录解析

```xml
<available_skills>
  <skill>
    <name>brave-search</name>
    <description>Web search and content extraction via Brave Search API...</description>
    <location>/home/u/.pi/agent/skills/brave-search/SKILL.md</location>
  </skill>
</available_skills>
```

注入文案里有一句关键指令：*「当技能文件引用相对路径时，按技能目录（SKILL.md 的父目录）解析，并在工具命令中使用该绝对路径」*——这是让「技能自带脚本」真正可用的那一行。

**这就是渐进式披露（progressive disclosure）**：上下文里常驻的只是一句描述（≤1024 字符），全文按需加载。代价是「模型不一定真的会去 read」，文档诚实地写着：*models don't always do this; use prompting or `/skill:name` to force it*。
<!-- @end-section -->

<!-- @section: discovery -->
## 发现与加载

### 位置（安全分级）

| 层级 | 路径 | 说明 |
|------|------|------|
| 全局 | `~/.pi/agent/skills/`、`~/.agents/skills/` | 无条件加载 |
| 项目 | `.pi/skills/`、`.agents/skills/`（cwd 起向上直到 git 根/文件系统根） | **仅在项目被信任后加载** |
| 包 | pi package 的 `skills/` 目录或 `package.json` 的 `pi.skills` | npm/git 分发 |
| 设置 | `settings.json` 的 `skills` 数组 | 文件或目录 |
| CLI | `--skill <path>`（可重复，**即使 `--no-skills` 也仍加载**） | 显式指定优先 |

项目级技能受 **project trust** 门控是必须的：`docs/skills.md` 顶部就挂着安全警告——技能可以指使模型做任何事，并可能携带模型会去执行的代码。

### 发现规则（`core/skills.ts:loadSkillsFromDirInternal`）

- 目录中**存在 `SKILL.md` 就把该目录当作技能根，立即返回、不再递归**
- 否则递归子目录继续找 `SKILL.md`
- 仅在根层（`includeRootFiles`）把直接的 `.md` 文件当作单文件技能（`~/.agents/skills/` 与项目 `.agents/skills/` 例外，忽略根 `.md`——为了与其他 harness 的目录约定兼容）
- 跳过 `.` 开头目录与 `node_modules`
- 尊重 `.gitignore` / `.ignore` / `.fdignore`（模式按相对前缀改写后喂给 `ignore` 库）
- 跟随符号链接（`statSync` 判断真实类型），坏链跳过

### 去重与冲突

- 用 `canonicalizePath()` 解析真实路径去重：**同一文件经不同符号链接被扫到，静默跳过**
- 同名不同文件 → 产出 `collision` 诊断，**先到者胜**
- 描述缺失 → **不加载**（唯一硬失败）；名字非法/超长/描述超长 → 只是 warning，照常加载

pi 有意放宽了标准里「name 必须等于父目录名」的要求，理由写在文档里：*该要求对被多个 harness 共享的技能目录不友好*。——这是「兼容 Claude Code / Codex 技能目录」这一现实目标压过规范洁癖的地方。
<!-- @end-section -->

<!-- @section: frontmatter -->
## Frontmatter 与校验

| 字段 | 必需 | 说明 |
|------|------|------|
| `name` | 是 | ≤64 字符，小写 a-z / 0-9 / 连字符，不得首尾连字符、不得连续连字符 |
| `description` | 是 | ≤1024 字符，**决定模型何时加载它**，要求具体 |
| `license` / `compatibility` / `metadata` | 否 | 标准字段，pi 不解释 |
| `allowed-tools` | 否 | 预批准工具列表（实验性） |
| `disable-model-invocation` | 否 | `true` 则**不进 system prompt**，只能由用户 `/skill:name` 显式调用 |

`disable-model-invocation` 是很实用的一档：把「危险/昂贵/需要人工确认」的技能从模型自主决策里摘出去，但保留人工入口。`formatSkillsForPrompt()` 里直接 `filter(s => !s.disableModelInvocation)`。

未知字段一律忽略——为跨 harness 共享留出空间。
<!-- @end-section -->

<!-- @section: invocation -->
## 显式调用：`/skill:name`

技能同时注册为 slash 命令（`getCommands()` 里 `name: skill:${skill.name}`、`source: "skill"`）。展开逻辑在 `agent-session.ts:_expandSkillCommand`：

```ts
/skill:pdf-tools extract
  ↓ 读取 SKILL.md，剥掉 frontmatter
<skill name="pdf-tools" location="/abs/path/SKILL.md">
References are relative to /abs/path.

<正文>
</skill>

extract            ← 参数原样追加在区块之后
```

三个细节：

- 展开结果是**用户消息**，不是 system prompt 注入——技能内容进入对话历史，会被压缩、会被分支继承。
- 未知技能名 **原样透传**（不报错），让它变成普通文本。
- `agent-session.ts` 另有 `parseSkillBlock()` 反向解析这个 XML 形状，用于 UI 渲染成「技能调用消息」组件（`skill-invocation-message.ts`）而不是一大坨文本。
- 该功能可由 `settings.json` 的 `enableSkillCommands` 开关关闭。

`packages/agent`（内核层）也有一份等价实现：`harness/skills.ts` 的 `formatSkillInvocation(skill, additionalInstructions?)`，以及 `AgentLane.skill(name, additionalInstructions?)`——**技能调用在下一代 harness 里是一等操作**。
<!-- @end-section -->

<!-- @section: two-loaders -->
## 两份技能加载器：一份纯粹，一份务实

| | `packages/agent/src/harness/skills.ts` (375 行) | `packages/coding-agent/src/core/skills.ts` (488 行) |
|---|---|---|
| I/O | 通过抽象 `ExecutionEnv`（`fileInfo/listDir/readTextFile/joinPath/canonicalPath`） | 直接 `node:fs` 同步调用 |
| 错误 | `Result<T, E>` 显式返回，不抛 | try/catch + 诊断数组 |
| 产物 | `Skill { name, description, content, filePath, disableModelInvocation }`（**含正文**） | `Skill { name, description, filePath, baseDir, sourceInfo, disableModelInvocation }`（**不含正文**，按需读） |
| 溯源 | 由调用方通过 `loadSourcedSkills(inputs, mapSkill)` 泛型附加 | 内建 `SourceInfo`（user/project/path/包名） |
| 名称校验 | **要求 name 与父目录同名**，否则 warning | 不做该校验 |

内核版把「文件系统」抽象成接口，是为了让 harness 能跑在非 Node 环境（浏览器、沙箱、远程 FS）。产品版为了启动速度选择「只读元数据、正文延迟到模型 read 时才由 read 工具读」——**这恰好也是渐进式披露在实现层的体现：连 pi 自己都不预读技能正文。**

两份实现并存说明了迁移正在进行中（见 [[analysis-pi-harness-003]]），但也提醒：**同一语义两份实现，校验规则已经出现分歧**（父目录名校验），这是迁移期的真实成本。
<!-- @end-section -->

<!-- @section: vs-templates -->
## 与 Prompt Templates 的分工

pi 还有第三种「文本能力」：prompt templates（`~/.pi/agent/prompts/`、`.pi/prompts/`、包的 `prompts/`）。

| | Skills | Prompt Templates |
|---|--------|------------------|
| 谁触发 | 模型自主（读描述后决定）或用户 `/skill:name` | **只能**用户 `/name args` |
| 进上下文 | 描述常驻 + 正文按需 | 展开后作为用户消息 |
| 结构 | 目录 + `SKILL.md` + 附属资源 | 单个 `.md` |
| 典型用途 | 「怎么做某类任务」的完整工作流 + 脚本 | 「常用提示词」的快捷方式 |

两者都走同一条 `ResourceLoader` → `getCommands()` 聚合，在 TUI 自动补全里并列出现。
<!-- @end-section -->

<!-- @section: skills-as-mcp -->
## 技能是 pi 的 MCP 替代品

README 的原话：

> **No MCP.** Build CLI tools with READMEs (see Skills), or build an extension that adds MCP support.

这句话把技能的定位讲清楚了：**MCP 解决的「让模型知道有个外部能力、知道怎么调」，pi 用「一个装了 CLI 脚本的目录 + 一份 README」解决。** 官方技能仓库（`anthropics/skills`、`badlogic/pi-skills`）提供 PDF/docx/xlsx 处理、Web 搜索、浏览器自动化、Google API、转写等——这些在别的 harness 里正是典型的 MCP server。

代价与收益的对比留到 [[analysis-pi-extensions-006]] 与 [[analysis-pi-insights-007]]。
<!-- @end-section -->

## 相关文档

- [[analysis-pi-tools-004|工具系统与装载机制]]
- [[analysis-pi-extensions-006|扩展系统与 No-MCP 立场]]
- [[analysis-pi-harness-003|Harness 分层与下一代规范]]
- [[analysis-pi-insights-007|对 Legion 的启示]]
