---
id: "analysis-pi-tools-004"
title: "pi-agent 工具系统与工具装载机制"
aliases: ["pi tools", "AgentTool", "ToolDefinition", "pi 工具注册"]
type: "design"
category: "design/analysis/pi-agent"
tags: ["pi-agent", "analysis", "tools", "tool-registry", "typebox"]
version: "1.0.0"
created: "2026-08-15"
updated: "2026-08-15"
author: "jxncyjq"
status: "review"
parent: "analysis-pi-index"
children: []
related_docs:
  - id: "analysis-pi-architecture-002"
    relation: "depends_on"
    path: "./02-architecture.md"
  - id: "analysis-pi-extensions-006"
    relation: "references"
    path: "./06-extensions-mcp.md"
---

# pi-agent 工具系统与工具装载机制

<!-- @section: two-types -->
## 两个类型：AgentTool 与 ToolDefinition

pi 的工具有两副面孔，分层清晰：

```ts
// packages/agent —— 内核契约，只关心「能被模型调用」
interface AgentTool<TParameters extends TSchema = TSchema, TDetails = any> extends Tool<TParameters> {
  label: string;                                  // UI 显示名
  prepareArguments?: (args: unknown) => Static<TParameters>;   // 校验前的兼容垫片
  execute(toolCallId, params, signal?, onUpdate?): Promise<AgentToolResult<TDetails>>;
  executionMode?: "sequential" | "parallel";      // 单工具级覆盖
}

// packages/coding-agent —— 产品契约，多了「提示词贡献 + 渲染 + 扩展上下文」
interface ToolDefinition<TParams, TDetails, TState> {
  name; label; description; parameters;
  promptSnippet?: string;         // 进 system prompt「Available tools」的一行
  promptGuidelines?: string[];    // 该工具激活时追加到「Guidelines」的条目
  constrainedSampling?: false | ConstrainedSamplingConfig;
  renderShell?: "default" | "self";
  prepareArguments?; executionMode?;
  execute(toolCallId, params, signal, onUpdate, ctx: ExtensionContext): Promise<...>;
  renderCall?(args, theme, context): Component;      // TUI 自定义渲染
  renderResult?(result, options, theme, context): Component;
}
```

`ToolDefinition → AgentTool` 的降维只有 20 行（`tools/tool-definition-wrapper.ts`）：把 `ctx` 通过闭包工厂注入 `execute`，丢掉渲染与提示词字段。反向也提供 `createToolDefinitionFromAgentTool()`，让 SDK 传进来的裸 `AgentTool` 也能进同一个 registry。

**要点**：内核不知道 `promptSnippet` 和 `renderCall` 的存在。工具的「模型可见面」「UI 可见面」「提示词可见面」被拆成三份，各自在正确的层被消费。

参数 schema 用 **TypeBox**（`typebox` 包，非 `@sinclair/typebox`，但 loader 两个名字都做了别名），校验走 `pi-ai` 的 `validateToolArguments`。
<!-- @end-section -->

<!-- @section: builtin -->
## 7 个内置工具

`core/tools/index.ts` 定义 `ToolName = "read" | "bash" | "edit" | "write" | "grep" | "find" | "ls"`。

| 工具 | 要点 |
|------|------|
| `read` | 文本 + 图片（jpg/png/gif/webp/bmp，作为附件送入）。文本截断到 **2000 行或 50KB**（先到先算），提示模型用 `offset/limit` 续读。可选自动缩图（`autoResizeImages`） |
| `bash` | `spawn` + 可配置 shell/前缀，无默认超时（可传 `timeout` 秒，上限 2^31-1 ms），进程树 kill、detached 子进程追踪。**`BashOperations` 是可插拔接口**——扩展可把执行重定向到 SSH/容器 |
| `edit` | 基于精确字符串替换，配 `edit-diff.ts`（500 行）做 diff 呈现 |
| `write` | 写文件，与 `edit` 共用 `file-mutation-queue.ts` 串行化同文件写 |
| `grep` | 外部 **ripgrep**，缺失时 `ensureTool("rg")` 自动从 GitHub Release 下载 |
| `find` | 外部 **fd**，同上自动下载 |
| `ls` | 目录列举 |

**默认只激活 4 个**：`read, bash, edit, write`（`agent-session.ts` 的 `defaultActiveToolNames`）。`grep/find/ls` 存在于注册表但默认不激活——因为默认阵容里有 `bash`，system prompt 会自动补一条 guideline「用 bash 做 ls/rg/find」。CLI 用 `--tools read,grep,find,ls` 可切换到无 bash 的只读阵容。

`tools-manager.ts` 的自动下载逻辑（GitHub latest release → 按平台/架构选 asset → tar/zip 解压 → 放 `~/.pi/agent/bin/`）带 `PI_OFFLINE` 开关和 Termux 特判（Bionic libc 不兼容，提示 `pkg install`）。**每个内置工具都先找系统 PATH，找不到才下载。**
<!-- @end-section -->

<!-- @section: registry -->
## 工具注册表的装配：`_refreshToolRegistry()`

这是整个工具装载机制的心脏（`agent-session.ts:2463`）。一次刷新做四件事：

```
① 收集来源（后者覆盖前者）
   内置 ToolDefinition (createAllToolDefinitions)
   → 扩展注册工具 (extensionRunner.getAllRegisteredTools())
   → SDK customTools（sourceInfo 标 <sdk:name>）

② 过滤：isAllowedTool(name) = (无 allowlist 或在 allowlist 内) && 不在 excludelist 内
   对应 CLI 的 --tools/-t（allowlist）与 --exclude-tools（excludelist）

③ 派生三张表
   _toolDefinitions      name → { definition, sourceInfo }   // 供 /tools UI 与 pi.getAllTools()
   _toolPromptSnippets   name → 一行摘要                      // 供 system prompt「Available tools」
   _toolPromptGuidelines name → 条目数组                      // 供 system prompt「Guidelines」

④ 包装成 AgentTool 存入 _toolRegistry，然后决定「哪些是 active」
```

**active 名单的推导规则**（第 ④ 步）比看起来微妙：

- 显式传了 `activeToolNames` → 用它（再过 allow/exclude）
- 有 allowlist → allowlist 里所有已注册工具全部激活
- `includeAllExtensionTools` → 扩展工具全部激活
- 否则 → **保留上次的 active 集合，并自动激活「本次新出现在注册表里」的工具**

最后一条是热注册的关键：扩展在运行中调 `pi.registerTool()` → `runtime.refreshTools()` → `_refreshToolRegistry()` → 新工具因「不在 previousRegistryNames 里」而被自动激活，模型下一轮就能调用，**无需 `/reload`**。
<!-- @end-section -->

<!-- @section: added-tools -->
## 动态工具与会话回放：addedToolNames

`extensions/wrapper.ts` 里 40 行的 `wrapRegisteredTool` 干了一件不显眼但关键的事：

```ts
const activeBefore = runner.getActiveTools();
const result = await execute(...);
const activeAfter = runner.getActiveTools();
// 若执行期间没有工具被移除，则把新增的工具名挂到结果上
return { ...result, addedToolNames: [...新增的名字] };
```

`addedToolNames` 随 `ToolResultMessage` 落进会话文件。于是**回放历史会话时，能还原「在这条工具结果之后，模型多了哪些工具」**。

这解决的是一个真实难题：工具集不是会话级常量，而是随时间变化的；如果不记录变化点，回放/分支/压缩后的上下文就会和当初的工具集对不上。Legion 若允许运行期动态授权工具，必须处理同样的问题。
<!-- @end-section -->

<!-- @section: prompt -->
## 工具如何影响 system prompt

`core/system-prompt.ts` 的三条规则很值得抄：

1. **不给 `promptSnippet` 的工具不出现在「Available tools」列表里**（`visibleTools = tools.filter(name => !!toolSnippets?.[name])`）。默认 prompt 补一句「除上述工具外，你可能还有项目相关的自定义工具」。即：**工具描述已在 tools 数组里，system prompt 里的清单只是导航，不是重复。**
2. **guidelines 按激活工具条件生成并去重**：只有当 `bash` 激活而 `grep/find/ls` 都未激活时，才注入「用 bash 做文件操作」。扩展工具的 `promptGuidelines` 追加进同一列表。文档特别警告：guidelines 是**扁平追加、不带工具名前缀**，所以每条必须自己写清是哪个工具（写 `Use my_tool when...`，别写 `Use this tool when...`）。
3. **技能区块只在 `read` 工具可用时才注入**——因为技能的加载方式就是「让模型 read 那个文件」。没有 read 就不该宣传技能。见 [[analysis-pi-skills-005]]。
<!-- @end-section -->

<!-- @section: interception -->
## 工具调用的拦截链

一次工具调用完整穿过的关卡：

```
模型发出 toolCall
  → prepareToolCall: 查注册表（找不到 → 合成错误结果）
      → tool.prepareArguments?()      兼容垫片
      → validateToolArguments()       TypeBox 校验
      → config.beforeToolCall()  ── AgentSession 转发 ──> runner.emitToolCall()
                                       扩展可 mutate event.input（原地改参数，不再校验）
                                       扩展可 { block: true, reason, terminate }
  → tool.execute(id, args, signal, onUpdate)
      onUpdate 流式局部结果 → tool_execution_update 事件
  → config.afterToolCall()   ── AgentSession ──> runner.emitToolResult()
                                       扩展可替换 content/details/isError/usage
      → normalizeToolResultImages()   扩展注入的图片也统一走缩放/格式归一
  → tool_execution_end 事件
  → ToolResultMessage（源序）
```

三个设计判断：

- **拦截逻辑放在 AgentSession 的 hook 上，不放在工具包装器里**（wrapper.ts 的注释专门解释了这次重构）。于是拦截对内置工具、扩展工具、SDK 工具**一视同仁**。
- 扩展改参数的方式是**原地 mutate `event.input`**，而且明确「不再重新校验」——快，但把责任推给扩展作者。规范版 harness 的 `before_tool` 改为返回 `args` 并**重新校验**，是个正确的收紧。
- 扩展抛异常时，`beforeToolCall` 把异常**继续抛出**（fail-closed，拦掉执行），注释写着 `Extension failed, blocking execution`。
<!-- @end-section -->

<!-- @section: truncation -->
## 输出治理

- `truncate.ts`：`DEFAULT_MAX_LINES = 2000`、`DEFAULT_MAX_BYTES = 50KB`、`GREP_MAX_LINE_LENGTH = 500`，提供 `truncateHead/truncateTail/truncateLine`。
- `output-accumulator.ts`（222 行）：bash 流式输出的累积与截断。
- `BashToolDetails.fullOutputPath`：**被截断的完整输出落盘**，结果里给出路径，模型可以自己去读。
- `render-utils.ts` / `visual-truncate.ts`：TUI 侧的视觉截断与展开（`getToolsExpanded()`）与模型侧截断分离。

「给模型截断版 + 落盘完整版 + 告诉它文件在哪」是比单纯截断更好的模式，Legion 的工具截断治理可以直接对齐。
<!-- @end-section -->

## 相关文档

- [[analysis-pi-architecture-002|运行时架构与主循环]]
- [[analysis-pi-extensions-006|扩展系统与 No-MCP 立场]]
- [[analysis-pi-skills-005|技能系统]]
- [[analysis-pi-harness-003|Harness 分层与下一代规范]]
