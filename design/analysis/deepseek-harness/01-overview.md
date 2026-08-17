---
id: "analysis-dsh-overview-001"
title: "DeepSeek Harness 项目总览"
aliases: ["dsh overview", "DeepSeek Harness 概览", "deepseek-harness overview"]
type: "analysis"
category: "design/analysis/deepseek-harness"
tags: ["deepseek-harness", "dsh", "cordis", "agent", "typescript", "monorepo"]
version: "1.0.0"
created: "2026-08-15"
updated: "2026-08-15"
author: "jxncyjq"
status: "review"
parent: "analysis-dsh-index"
children: []
related_docs:
  - id: "analysis-dsh-index"
    relation: "related_to"
    path: "./index.md"
  - id: "analysis-dsh-architecture-002"
    relation: "related_to"
    path: "./02-architecture.md"
  - id: "analysis-dsh-plugin-system-003"
    relation: "related_to"
    path: "./03-plugin-system.md"
---

# DeepSeek Harness 项目总览

<!-- @section: overview -->
## 概述

DeepSeek Harness（下称 `dsh`）是 DeepSeek AI 开源的 agent harness（智能体运行框架），MIT 协议，处于 _开发者预览_ 阶段。

核心主张一句话：**一切皆插件（everything is a plugin）**。模型适配器、工具注册表、会话日志、甚至 agent 主循环本身都是挂在同一棵 Cordis 上下文树上的插件，全部可由配置替换。仓库根 `AGENTS.md` 明确写着「没有特权内核可打补丁」——扩展 dsh 的方式是在旁边挂一个插件，而不是改核心。

分析快照：
- 版本 `0.1.0-rc.5`
- HEAD `47f9438`（2026-08-13）
- 代码位置 `f:\source\github\deepseek-harness`

<!-- @end-section -->

<!-- @section: tech-stack -->
## 技术栈

| 维度 | 选型 |
|------|------|
| 语言 | TypeScript，全仓 `strict: true` + `noImplicitAny`，残留 `any` 必须写明为何无法收窄 |
| 模块制式 | 全 ESM（`"type": "module"`），跨包用包名导入，包内相对导入必须带 `.ts` |
| 运行时 | Node `^22.19.0 \|\| >=24.0.0` |
| 包管理 | pnpm 11.7.0 workspaces |
| 插件框架 | [Cordis](https://github.com/cordiverse/cordis)（vendored，非依赖） |
| 构建 | `tsc -b` 出 `lib/types`，`tsdown` 打运行时包 |
| 校验 | zod / schemastery（配置 schema）、Typert（RPC 类型图生成） |
| 测试 | vitest（unit / e2e / snapshot / web / stress 多套 config） |
| 静态检查 | oxlint、knip、jscpd（跨文件克隆检测）、publint、一批自研 `scripts/verify-*` 门禁 |
| 前端 | React + Vite（`apps/web`），Wails 式桌面壳无，纯浏览器 |
| 原生扩展 | `native/landlock-run`（Linux Landlock 沙箱 runner，Node addon） |
| Python | `python/sdk` + `sdk-runtime`（外挂 SDK 与打包运行时） |
| 文档 | 中英双语 + VitePress 站点（`website/`），大量文档由脚本生成并有新鲜度门禁 |

<!-- @end-section -->

<!-- @section: repo-layout -->
## 仓库布局

```
vendor/       vendored Cordis 源码（cordis / loader / include / group / hmr / logger / schemastery / timer / cosmokit）
packages/     219 个工作区包，路径 packages/<group>/<pkg>/，npm 名统一 @deepseek-ai/dsh-<pkg>
apps/         cli（dsh 命令行与 Host 启动器）、web（浏览器前端）
examples/     可直接跑的 cordis.yml 叶子（acp-agent / headless-agent / jsonrpc-agent / mcp-memory / web-cordis / web-schedule）
native/       Landlock 原生 runner
python/       Python SDK
docs/         架构、子系统、生成的目录（tool-catalog / config-catalog / persistence-catalog / module-graph）
scripts/      仓库门禁与生成器
.agents/      Agent 工作流与 Agent Notes（决策记录）
website/      文档站
```

219 个包按「组（group）」组织，组 README 拥有该组的「包 ↔ ctx key」映射表。组的划分不是按技术分层，而是**按能力（capability）分层**：

| 组 | 职责 |
|---|---|
| `core/` | 产品 API 主干：session、system-prompt、tools、agent、agent-loop、scope |
| `llm/` | 模型能力：抽象 seam + DeepSeek/pi-ai/replay 适配器 + token 计量 |
| `shell/` `subprocess/` `terminal/` `sandbox/` | 执行面：bash/pwsh 执行器、进程树、持久 PTY、进程封禁 |
| `fs/` `lsp/` `web/` `skill/` `todo/` `plan/` `goal/` `schedule/` | 各类模型可见能力 |
| `subagent/` `workflow/` `jobs/` | 委派与后台编排 |
| `session/` `session-query/` `storage/` `spill/` `attachment/` | 数据面：持久化、检索、KV、溢出、附件 |
| `bundle/` `preset/` `boot/` | 组合面：profile 补丁层、per-session agent 组合、启动胶水 |
| `api/` `typert/` `host/` `client/` | Web GUI 的 BFF、RPC 类型图、HTTP 宿主、浏览器插件 |
| `acp/` `sdk/` `hooks/` | 对外协议：Agent Client Protocol、JSON-RPC SDK、Claude Code/Codex hook 桥 |
| `extensions/` | agent 自我修改：运行时检视与动态挂载模型编写的插件 |
| `interaction/` `credentials/` `settings/` `identity/` `feedback/` `workspace/` | 人机协作与配置面 |
| `guard/` `runtime-diagnostics/` `test-support/` `util/` | 守护、不变式、测试设施、零依赖工具 |

<!-- @end-section -->

<!-- @section: entrypoints -->
## 运行入口

```bash
npx @deepseek-ai/dsh web     # 启动 Web UI，默认 http://127.0.0.1:3080
```

源码运行：

```bash
pnpm install && pnpm run build && pnpm dsh web
```

三类形态由 profile 决定（见 [[analysis-dsh-architecture-002]]）：

| 形态 | 组成 | 用途 |
|---|---|---|
| `web` | `dsh-base` + `dsh-web-app` | 浏览器 GUI，带 Host/HTTP/RPC 层 |
| `headless` | `dsh-base` + `dsh-headless` | 一次性任务执行，无 server |
| ACP / JSON-RPC | `examples/*` 叶子 | 自动化集成（编辑器/外部编排器） |

<!-- @end-section -->

<!-- @section: engineering -->
## 工程约束（值得注意的几条）

这些约束是 dsh 架构能撑住 219 个包的真实原因，而不是文档口号：

1. **注册即效果**：任何贡献（prompt 段、工具、适配器、监听器）都走 `ctx.effect()` / `ctx.on()`，返回 disposer，插件卸载时自动回卷。因此 HMR 天然成立。
2. **模型可见 ⟺ 已落日志**：凡是能进入模型请求的内容，必须能从 session log 重建；新增模型可见输入必须先新增 session 事件。有运行时不变式（`ctx.invariants`）断言这一点。
3. **禁止硬编码可调参数**：随部署变化的选择必须是可从 `cordis.yml` 改的 `Config` 字段；`DEFAULT_*` 常量不算可配置。协议常量、外部规范、安全不变式则固定。
4. **配置错误必须响亮失败**：能自包含判断的在加载期失败，否则在最早可解析点失败，绝不静默跳过缺失引用。
5. **跨边界不透明 id 必须 branded**（`Branded<B>`），禁止裸 `string`。
6. **同进程类型化边界信任 TypeScript**：不为静态接口已保证的值补运行时校验；只在 parser/config、队列、模型/工具 JSON、持久化、worker、进程、wire 这些真实边界做校验。
7. **非平凡改动必须在同一 PR 附 Agent Note**（`.agents/notes/`），已归档 note 冻结不可改。
8. **测试描述行为而非正确性**：行为过时就连同测试一起改，并在 PR 解释原因。
9. **CI 覆盖率门禁是 `test:coverage`，`packages/*/*/src` 逐文件 100%**。

命令面（根 `package.json` scripts）：`test` / `test:coverage` / `test:e2e`（需 `DEEPSEEK_API_KEY`）/ `test:snapshot`（无 key 回放）/ `typecheck` / `lint` / `duplication` / `build` / `hygiene` / `doc-sync` / `website:build`。

<!-- @end-section -->

## 相关文档

- [[analysis-dsh-index|DeepSeek Harness 分析索引]] — 本系列入口
- [[analysis-dsh-architecture-002|系统架构]] — Cordis 微内核、profile/bundle、turn/step 生命周期
- [[analysis-dsh-plugin-system-003|插件系统]] — 插件形态、capability seam、scope、动态自修改
- [[analysis-dsh-capabilities-004|核心能力]] — 工具管线与各能力子系统
- [[analysis-dsh-insights-005|评估与借鉴]] — 对 Legion 的可移植点
