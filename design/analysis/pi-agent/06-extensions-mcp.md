---
id: "analysis-pi-extensions-006"
title: "pi-agent 扩展系统与 No-MCP 立场"
aliases: ["pi extensions", "ExtensionAPI", "pi 插件", "pi no mcp"]
type: "design"
category: "design/analysis/pi-agent"
tags: ["pi-agent", "analysis", "extensions", "plugin", "mcp", "jiti"]
version: "1.0.0"
created: "2026-08-15"
updated: "2026-08-15"
author: "jxncyjq"
status: "review"
parent: "analysis-pi-index"
children: []
related_docs:
  - id: "analysis-pi-tools-004"
    relation: "depends_on"
    path: "./04-tools.md"
  - id: "analysis-pi-skills-005"
    relation: "related_to"
    path: "./05-skills.md"
---

# pi-agent 扩展系统与 No-MCP 立场

<!-- @section: no-mcp -->
## 先说结论：pi 没有 MCP，且是故意的

全仓库 `mcp` 关键字取证（`grep -ril mcp packages/*/src`）：

| 命中 | 性质 |
|------|------|
| `packages/ai/src/auth/oauth/anthropic.ts:37` | OAuth scope 字符串 `user:mcp_servers`，只为通过 Anthropic 授权 |
| `packages/coding-agent/src/utils/tool-result-images.ts:15` | 注释：把「MCP bridges」列为**扩展**可能产出图片的来源之一 |
| `packages/coding-agent/README.md:394,498` | 「扩展能做什么」列表里的 "MCP server integration"；以及 "**No MCP.**" 的哲学声明 |

**没有任何 MCP client / server / transport 实现。** 官方立场：

> **No MCP.** Build CLI tools with READMEs (see Skills), or build an extension that adds MCP support. [Why?](https://mariozechner.at/posts/2025-11-02-what-if-you-dont-need-mcp/)

即：MCP 在 pi 里被降级为「一个扩展可以实现的东西」，而不是内核关心的协议。取代它的是两条本地通路——**Skills**（描述 + CLI 脚本，见 [[analysis-pi-skills-005]]）和 **Extensions**（进程内 TypeScript）。

这个选择的实质是：**MCP 的价值是跨进程 + 跨语言 + 跨厂商互操作；如果你的扩展点本来就是同语言同进程的，那层协议只剩开销。** pi 赌的是后者。代价见 [[analysis-pi-insights-007]]。
<!-- @end-section -->

<!-- @section: shape -->
## 扩展是什么形状

一个扩展 = 一个默认导出工厂函数的 `.ts`/`.js` 模块：

```ts
export default function (pi: ExtensionAPI) {      // 也可以是 async
  pi.registerTool({ name: "deploy", ... });
  pi.registerCommand("stats", { ... });
  pi.on("tool_call", async (event, ctx) => { ... });
}
```

`async` 工厂会被 **await**，启动流程等待它完成——文档举的用例是「启动前先拉远程模型列表再 `pi.registerProvider()`」。

发现位置（`discoverAndLoadExtensions`，一层不递归）：

1. 项目：`<cwd>/.pi/extensions/`
2. 全局：`~/.pi/agent/extensions/`
3. 配置/CLI 显式路径

每个位置内的规则：直接的 `*.ts`/`*.js` 文件 → 加载；子目录有 `index.ts`/`index.js` → 加载；子目录有 `package.json` 且带 `pi.extensions` 清单 → 按清单加载。**复杂包必须用 manifest**，不给深度递归。
<!-- @end-section -->

<!-- @section: loader -->
## 加载器：jiti + 双模块解析策略

`core/extensions/loader.ts` 用 `jiti/static` 在运行时直接 import TypeScript，无需用户预编译。难点在于「扩展 import 的 `@earendil-works/pi-*` 必须解析到**宿主正在用的那份实例**」，否则类型对得上、`instanceof` 对不上。三种运行形态三套策略：

| 运行形态 | 策略 |
|----------|------|
| Bun 单文件二进制 | `virtualModules` —— 把静态 import 进来的 `_bundled*` 模块对象直接喂给 jiti（`tryNative: false`）。注释强调：**这些 import 必须是静态的，Bun 才会把它们打进可执行文件** |
| TypeScript 源码运行（开发） | `virtualModules` + `tsconfigPaths: true`，复用宿主已解析的模块 |
| 已构建的 Node 安装 | `alias` —— 指向 `dist/*.js` 实体路径，workspace 存在时优先用 workspace 路径 |

暴露给扩展的模块白名单：`typebox`（含 `/compile`、`/value`）、`@earendil-works/pi-agent-core`、`pi-tui`、`pi-ai`（根路径解析到 **compat 入口**，是 core 入口的严格超集，为旧扩展保活）、`pi-ai/oauth`、`pi-ai/providers/all`、`pi-coding-agent`。**每个名字都额外注册了旧作用域 `@mariozechner/*` 别名**——项目改名后不破坏已发布扩展。

注释还留了一条重要的架构约束：

> `loader.ts` 的导出**不从 `index.ts` 再导出**，以避免循环依赖——这样扩展才能 `import from "@earendil-works/pi-coding-agent"`。

打包指南（`docs/packages.md`）配套规定：这些核心包在扩展的 `package.json` 里必须列 `peerDependencies: "*"` 且**不得打包**；其他 pi 包则必须 `bundledDependencies`，因为 **pi 用独立模块根加载各个包，不同安装之间不共享模块**。

模块缓存按 `(cwd, generation)` 令牌管理，`clearExtensionCache()` 递增 generation —— `/reload` 时旧缓存整体失效。
<!-- @end-section -->

<!-- @section: api -->
## ExtensionAPI 全貌

`types.ts`（1728 行）定义的能力面，按类别：

**事件订阅** `pi.on(event, handler)` —— 28 个事件，见下节。

**注册类**
- `registerTool(definition)` —— 见 [[analysis-pi-tools-004]]
- `registerCommand(name, { description, handler, getArgumentCompletions? })`
- `registerShortcut(keyId, { handler })`
- `registerFlag(name, { type, default })` + `getFlag(name)` —— **扩展可以给 CLI 加参数**
- `registerMessageRenderer` / `registerEntryRenderer` / `registerMarkdownTransformer`

**动作类**
- `sendMessage`（自定义消息，进 LLM 上下文）/ `appendEntry`（自定义条目，**不进** LLM 上下文，仅持久化+渲染）
- `sendUserMessage(content, { deliverAs: "steer"|"followUp", expandPromptTemplates })`
- `setSessionName` / `getSessionName` / `setLabel`
- `exec(command, args, options)`
- `getActiveTools` / `getAllTools` / `setActiveTools` / `getCommands`
- `setModel` / `getThinkingLevel` / `setThinkingLevel`

**Provider 注册**
- `registerProvider(name, config)` / `registerProvider(provider)` / `unregisterProvider(name)`
- 可提供 `models`、`baseUrl` 覆盖、自定义 `streamSimple`、完整 `oauth`（login/refreshToken/getApiKey）、`refreshModels`
- **扩展可以接一个全新的 LLM 供应商，包括 OAuth 登录流** —— 这是 pi 扩展能力上限的最好证明（示例：`custom-provider-anthropic`、`custom-provider-gitlab-duo`）

**通信** `pi.events`（EventBus，扩展之间发消息）

**UI**（`ctx.ui`，40+ 方法）：select/confirm/input/notify/editor、setStatus/setWidget/setFooter/setHeader/setTitle、`custom()` 挂任意 TUI 组件（可 overlay）、主题读写、编辑器替换（`setEditorComponent`，示例里有 vim 模式）、自动补全叠加、原始终端输入监听。

**会话控制**（仅 `ExtensionCommandContext`，命令处理器内可用）：`newSession` / `fork` / `navigateTree` / `switchSession` / `reload` / `waitForIdle` / `getSystemPromptOptions`。
<!-- @end-section -->

<!-- @section: events -->
## 事件生命周期

`docs/extensions.md` 里的官方时序图（节选精简）：

```
启动:  project_trust → session_start{startup} → resources_discover{startup}

用户输入:
  扩展命令优先匹配（命中则短路）
  → input（可拦截/改写/接管）
  → 技能与模板展开
  → before_agent_start（可注入消息、改 system prompt）
  → agent_start
  ┌── turn 循环 ──────────────────────────────┐
  │  turn_start
  │  context（可改 messages）
  │  before_provider_headers（原地改 headers，null 删除）
  │  before_provider_request（可替换 payload）
  │  after_provider_response（status+headers，消费流之前）
  │  tool_execution_start → tool_call（可 block）
  │    → tool_execution_update → tool_result（可改）→ tool_execution_end
  │  turn_end
  └───────────────────────────────────────────┘
  → agent_end → agent_settled（无重试/压缩/后续排队了）

/new /resume:  session_before_switch（可取消）→ session_shutdown → session_start → resources_discover
/fork:         session_before_fork（可取消）→ session_shutdown → session_start{fork}
/compact:      session_before_compact（可取消或自带结果）→ session_compact
/tree:         session_before_tree（可取消或自带摘要）→ session_tree
退出:          session_shutdown
```

三个值得记的点：

1. **`agent_end` 与 `agent_settled` 分开**。前者是「loop 停了」，后者是「不会再自动重试/压缩/续跑了」。做通知类扩展（`notify.ts`）必须用后者。
2. **`resources_discover` 让扩展反向提供资源路径**：返回 `{ skillPaths, promptPaths, themePaths }`，于是「扩展动态发现技能」成为可能（示例 `dynamic-resources`）。
3. **`project_trust` 只发给 user/global 与 CLI 扩展，且在项目资源加载之前** —— 信任决策不能由被信任对象自己做。
<!-- @end-section -->

<!-- @section: runtime -->
## Runtime 与生命周期安全

`ExtensionRuntime` 是 loader 与 runner 共享的可变状态，设计上有三层保护：

**① 未初始化即抛**。loader 创建的 runtime 里所有动作方法都是 `notInitialized`（抛异常）。只有 `refreshTools` 是 no-op（因为 `registerTool()` 在加载期就合法）。runner 的 `bindCore()` 才换上真实现。→ **扩展在工厂函数里就调 `pi.sendMessage()` 会得到清晰错误，而不是静默失效。**

**② 注册排队**。`registerProvider` 在 bind 之前把注册塞进 `pendingProviderRegistrations`，bind 后改为直接调 ModelRegistry 立即生效。同一个 API 在两个生命周期阶段语义一致。

**③ 陈旧检测**。`invalidate(message)` 把 runtime 标记为 stale，此后所有 API 调用 `assertActive()` 抛出一段**很长的教学式错误**：

> This extension ctx is stale after session replacement or reload. Do not use a captured `pi` or command ctx after `ctx.newSession()`, `ctx.fork()`, `ctx.switchSession()`, or `ctx.reload()`. For newSession/fork/switchSession, move post-replacement work into `withSession`...

同时自动退订该 runtime 持有的所有 EventBus 订阅（`trackEventBusSubscription`）。

**这是把「会话被替换后旧 ctx 悬空」这个必然的踩坑点，变成一条自我解释的运行时错误。** 文档里还专门有一节 "Session replacement lifecycle and footguns"。Legion 做插件/扩展时，这个模式（陈旧句柄 + 教学式报错 + 自动清理订阅）值得整套照搬。
<!-- @end-section -->

<!-- @section: examples -->
## 示例扩展就是能力上限的说明书

`examples/extensions/` 有 70+ 个示例。挑几个说明「pi 认为不该内建、但用扩展 20 分钟能做」的东西：

| 示例 | 替代了别的产品里的什么内建功能 |
|------|-------------------------------|
| `subagent/` | 子 Agent（每次调用**拉起独立 pi 进程**获得隔离上下文，agent 定义是带 frontmatter 的 `.md`：name/description/tools/model/systemPrompt） |
| `plan-mode/` | Plan Mode |
| `permission-gate.ts` / `confirm-destructive.ts` / `protected-paths.ts` | 权限弹窗 |
| `todo.ts` | 内建 TODO |
| `custom-compaction.ts` / `summarize.ts` | 压缩策略 |
| `git-checkpoint.ts` / `auto-commit-on-exit.ts` / `dirty-repo-guard.ts` | 检查点/自动提交 |
| `sandbox/` / `ssh.ts` / `gondolin/` | 沙箱与远程执行（走 `BashOperations` 替换） |
| `custom-provider-*/` | 自定义 LLM 供应商 + OAuth |
| `structured-output.ts` / `kimi-deferred-tools.ts` | 结构化输出、延迟工具 |
| `claude-rules.ts` | 「让 pi 长得像 Claude Code」 |
| `doom-overlay/` `snake.ts` `space-invaders.ts` | ……等待模型时打游戏 |

这个列表本身就是论据：**当扩展点足够全，「内建」就变成了品味问题而不是能力问题。**
<!-- @end-section -->

<!-- @section: security -->
## 安全边界

- 扩展 = **全权限任意代码**，文档三处挂了同一句警告：*Pi packages run with full system access... Review source code before installing third-party packages.*
- 项目级资源（`.pi/extensions`、`.pi/skills`）受 **project trust** 门控；`project_trust` 事件本身只发给已受信来源。
- 没有沙箱、没有权限声明、没有能力清单。安全模型是「**读代码 + 容器**」，这与「拒绝权限弹窗、建议跑容器」的哲学一致。
- 对比 MCP：MCP server 至少是独立进程、可独立限权；pi 扩展在宿主进程内，一个恶意扩展等价于宿主完全沦陷。**这是 No-MCP 路线最实在的代价。**
<!-- @end-section -->

## 相关文档

- [[analysis-pi-tools-004|工具系统与装载机制]]
- [[analysis-pi-skills-005|技能系统]]
- [[analysis-pi-insights-007|对 Legion 的启示]]
- [[analysis-pi-overview-001|项目架构总览]]
