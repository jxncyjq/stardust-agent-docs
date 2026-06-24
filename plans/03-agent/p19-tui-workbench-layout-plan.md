---
id: "plans-agent-p19-tui-workbench-layout"
title: "Agent P19 TUI 工作台布局计划"
type: "plan"
category: "plans/agent"
tags: ["plan", "agent", "tui", "bubble-tea", "deepseek-tui"]
version: "0.1.0"
created: "2026-05-17"
updated: "2026-05-17"
status: "done"
---

# Agent P19 TUI Workbench Layout Plan

## 目标

参考 DeepSeek-TUI 截图，将 `agent tui` 从普通分区输出升级为全屏工作台布局：

- 顶部状态栏展示 Agent 模式、选中角色/profile 和当前模型，不硬编码 provider 或模型名。
- 主工作区左侧承载会话、结果与事件流，空闲时居中展示 Legion Agent TUI 标识。
- 右侧固定 `Plan` 面板展示计划轨道提示，后续承载 update_plan、goal、cycles。
- 底部固定 `Composer` 输入框，显示 `编写任务或使用 /。` 并承载用户输入。
- 最底部状态条展示 `agent · model` 和快捷键，不展示 running/error 状态。
- 默认关闭主区 `AUDIT` 和 `EVENT` 展示，改为通过 Composer 输入 `/audit` 或 `/event` 命令查看审计动作和事件流。
- CLI/TUI 运行日志不写终端，默认追加写入 `logs/agent.log`。
- Composer 输入 `/` 时显示已支持的本地命令，并可用上下方向键选中命令；非 slash 输入时上下方向键浏览上一条和下一条输入历史。
- `RESULT` 视图使用格式化输出，保留多行正文，并分离任务编号与输出内容。
- 初始化空态将 `Legion Agent TUI` 与 `role · model` 在主工作区居中展示，并使用一致的主题色。
- Composer 在任务运行期间显示“大模型通讯/等待输出”工作状态，并暂时锁定输入。
- Result 输出对 Markdown 列表、粗体标记和长行进行终端友好的纯文本格式化。
- Footer 在任务运行期间切换为“工作中”状态条，并展示底部进度条。
- 主工作区采用对话流显示用户输入、thinking 状态和模型回复，替代传统 `RESULT/Output` 面板式输出。
- Footer 进度条由 Bubble Tea tick 驱动，在工作期间产生动效，任务结束后停止刷新。
- 用户输入和 thinking 支持配置开关；thinking 优先展示 MaaS/OpenAI-compatible 响应中的公开 reasoning 字段，没有该字段时回落到 Agent 运行过程摘要。
- 对模型输出中的伪工具调用进行治理；未接入真实工具协议时不再展示 `search_content(...)` 这类裸文本调用。
- 接入最小模型工具调用闭环：OpenAI-compatible `tool_calls` 可由 Runtime 执行内置只读工具并回灌结果。
- 工具输出不在源头截断；运行中通过事件总线把 `tool_result` 流式推送到 TUI，目录枚举等长输出不再等任务结束后一次性显示。
- 主输出区支持鼠标滚轮上下滚动，便于查看超出屏幕高度的长输出。

## 任务清单

| 任务 ID | 优先级 | 组件 | 任务 | 状态 | 验收 |
|---------|--------|------|------|------|------|
| AG-P19-001 | P0 | TUI/Layout | 增加窗口尺寸状态和全屏工作台布局 | `done` | View 包含 header、main、plan、composer、footer 五区 |
| AG-P19-002 | P0 | TUI/Composer | 底部输入区改为固定 Composer 体验 | `done` | 空输入显示 `编写任务或使用 /。`，输入时显示当前 prompt |
| AG-P19-003 | P1 | TUI/Result | 会话区展示结果、事件流和错误状态 | `done` | 运行完成后左侧区域展示 Result/Event/Audit 摘要 |
| AG-P19-004 | P1 | Tests/Docs | 补充布局专项测试并同步任务表 | `done` | TUI 专项测试和全量验证通过 |
| AG-P19-005 | P1 | TUI/Command | 默认隐藏 Audit，新增 `/audit` 本地命令 | `done` | 普通结果视图不显示 AUDIT，输入 `/audit` 后显示审计动作 |
| AG-P19-006 | P1 | TUI/Command | 默认隐藏 Event，新增 `/event` 本地命令 | `done` | 普通结果视图不显示 EVENT，输入 `/event` 后显示事件流 |
| AG-P19-007 | P1 | TUI/Logging | 默认隐藏 Status，运行日志写入文件 | `done` | TUI 不显示 STATUS running/error，CLI 默认日志追加到 `logs/agent.log` |
| AG-P19-008 | P1 | TUI/Composer | Slash 命令提示与历史导航 | `done` | 输入 `/` 显示 `/audit`、`/event`；slash 模式上下方向键选命令，普通模式上下方向键浏览历史输入 |
| AG-P19-009 | P1 | TUI/Result | 格式化 Result 输出 | `done` | Result 视图分离 Task 与 Output，保留多行正文 |
| AG-P19-010 | P1 | TUI/Display | 模型显示配置化 | `done` | Header/Footer 使用选中 MaaS profile 的模型名，不硬编码 DeepSeek 字符串 |
| AG-P19-011 | P1 | TUI/Display | 初始化品牌区居中与颜色统一 | `done` | 空态主工作区将 `Legion Agent TUI` 和 `role · model` 居中显示，并使用一致主题色 |
| AG-P19-012 | P1 | TUI/Composer | 运行期间 Composer 工作状态 | `done` | 与大模型通讯时显示等待输出状态，并忽略输入编辑/历史/命令选择 |
| AG-P19-013 | P1 | TUI/Result | Markdown 结果格式化 | `done` | 编号/无序列表不保留孤立 marker，粗体标记转纯文本，长行按主面板宽度挂起缩进 |
| AG-P19-014 | P1 | TUI/Footer | 运行期间底部进度条 | `done` | Footer 显示 `工作中 ...` 和横向进度条，空闲快捷键仅在非运行态显示 |
| AG-P19-015 | P1 | TUI/Main | 对话流交互模式 | `done` | 主工作区显示用户 prompt、thinking 状态和带 `●` 前缀的模型回复 |
| AG-P19-016 | P1 | TUI/Footer | 动态工作进度条 | `done` | 运行期间 tick 推进进度条游标，任务结束后停止继续 tick |
| AG-P19-017 | P1 | TUI/Config | Prompt 与 thinking 展示配置化 | `done` | `tui.show_prompt` 与 `tui.show_thinking` 控制主会话区是否显示输入问题和 thinking；thinking 优先使用模型服务返回的公开 reasoning |
| AG-P19-018 | P1 | TUI/Quality | 伪工具调用输出治理 | `done` | 默认工具策略禁止伪工具文本输出，模型输出中出现 `search_content(...)` 等调用时替换为明确能力边界说明 |
| AG-P19-019 | P0 | A01/A20/C70 | 模型驱动工具调用闭环 | `done` | MaaS/OpenAI-compatible `tool_calls` 可解析为 `ToolCall`，Runtime 执行 `search_content/read_file` 后二次调用模型生成最终答案 |
| AG-P19-020 | P0 | TUI/A20/X00 | 工具输出不截断与流式展示 | `done` | `list_files/read_file/search_content` 不截断源输出，Runtime 发布完整 `tool_result`，TUI 运行中实时接收并展示 |
| AG-P19-021 | P1 | TUI/Output | 输出区鼠标滚轮滚动 | `done` | TUI 启用 mouse cell motion；鼠标滚轮在主输出区内可上下滚动长输出 |

## 验证命令

```powershell
go test ./internal/tui -run TestInteractiveModel -count=1
go test ./internal/tui ./internal/cli -count=1
go test ./...
go vet ./...
go build -o NUL ./cmd
```

## 使用方式

```powershell
go run ./cmd -- tui --config .\agent.json
```
