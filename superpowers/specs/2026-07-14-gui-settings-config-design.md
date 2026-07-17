---
title: legionAgentGUI 设置菜单 — Agent 配置可视化
type: design-spec
status: draft
created: 2026-07-14
scope: legion/legionAgentGUI
related:
  - "[[legion-gui-wails-gotchas]]"
tags: [gui, config, settings, wails, react]
---

# legionAgentGUI 设置菜单 — Agent 配置可视化设计

## 1. 目标与背景

给 legionAgentGUI（Wails v2 桌面应用）增加一个**设置界面**，把 `legionAgent/agent.json`
配置文件做**结构化可视化编辑**：用户在图形表单里改模型、密钥、会话、主题等参数并保存，
免去手动编辑 JSON。

@section 背景约束

- 配置文件是 `a.cfgPath` 指向的 `agent.json`（GUI 启动时 `resolveConfigPath` 解析得到）。
- 权威 schema 是 Go 的 `config.Config` 结构体（`internal/config/config.go`），共 **14 段**：
  `maas` / `agents` / `storage` / `server` / `service` / `runtime` / `tui` / `session` /
  `skills` / `context_files` / `workspace` / `tasks` / `web` / `evolution`。
- serve 配置在 `serve.BuildService` 时**一次性** `config.Load` 缓存，改配置必须**重建 service**。
- 前后端通信走 **Wails Go 绑定**（不走浏览器 fetch），避开随机端口本地服务的 CORS。
- **Fail-Loud 铁律**（见 `legionAgentGUI/CLAUDE.md`）：配置读写全程禁止兜底，出错必报必记。

## 2. 架构总览

新增一条 config 读写通道（Go 绑定）+ 一个前端设置模态。数据流：

```
GetConfig ──► 表单渲染 ──► 用户编辑 ──► SaveConfig
                                          │
              ┌───────────────────────────┘
              ▼
   严格校验 ──► 写临时文件 ──► config.Load(temp) 验证
              │（任一步失败：报错，不动原文件）
              ▼
   备份 agent.json.bak ──► 原子替换 ──► 重启内嵌 serve
                                          │
              serve:status 事件 ──────────┘
              ▼
        前端 serveStore 接新端口 ──► 成功 toast
```

## 3. 组件设计

### 3.1 Go 绑定 `app_config.go`

新增文件，方法挂在现有 `App` 上（复用 `a.cfgPath`、`a.ctx`、`a.serve`）。

**`GetConfig() (string, error)`**
读 `a.cfgPath` 原始 JSON 字节并返回字符串。
- `a.cfgPath == ""` → 返回 error（无配置路径，不该发生）。
- 读盘失败 → `fmt.Errorf("read config %q: %w", path, err)`。
- 返回**磁盘原文**（不经 `config.Load`），保证「所见即文件内容」，不掺入 env 覆盖/默认值。

**`GetConfigPath() (string, error)`**
返回 `a.cfgPath`，供 UI 顶部展示当前配置文件绝对路径。空则报错。

**`SaveConfig(raw string) error`**
1. **写临时文件**：把 `raw` 写到同目录临时文件（`agent.json.tmp-*`）。
2. **权威校验**：调 `serve.ValidateConfig(ctx, tempPath)`（见下方模块边界说明）验证能被
   `config.Load` 加载（捕获 JSON 语法错、类型不匹配）；失败 → 删临时文件 + 返回 error，**不动原文件**。
3. **备份**：把当前 `agent.json` 拷到 `agent.json.bak`（覆盖旧备份）。
4. **原子替换**：`os.Rename(temp, a.cfgPath)`。
5. **重启 serve**：调 `a.serve.Restart(a.ctx, a.cfgPath)`（见 3.2）。
   重启失败 → emit `serve:error` 事件 + 返回 error，提示 `.bak` 可手动恢复。
- 全程失败点用 `%w` 包装、带路径上下文；**不返回 nil 假装成功**。

> **模块边界**：legionAgentGUI 是**独立 Go module**，无法 import `internal/config`（Go internal 规则）。
> 校验须经 legionAgent 侧**公开桥函数** `serve.ValidateConfig(ctx, path) error`（新增，薄封装
> `internal/config.Load`），与现有 `serve.BuildService` 桥同理。因前端从**强类型草稿**序列化 JSON、
> 不会掺入未知键，故不需要 `DisallowUnknownFields`；`config.Load` 已能捕获语法/类型错，作权威校验足够。

> **注意（Fail-Loud 诚实）**：GUI 的 `ServeManager.Start` 强制 `Addr: "127.0.0.1:0"`
> 覆盖 `server.listen_addr`，所以在 GUI 里改 `listen_addr` **不生效**。该字段按 3.3 归入
> 只读展示，并标注「GUI 内嵌 serve 固定用随机端口，此值不生效」。

### 3.2 `ServeManager.Restart(appCtx, path)`

在 `serve_manager.go` 增加方法：
- `Start` 为每次运行创建 `done chan struct{}`，goroutine 退出时 `defer close(done)`（在其最后一次 `serve:status` emit 之后）。
- `Restart`：先捕获 `prev := m.done` → `Stop()`（cancel 现有 ctx）→ `select` 等 `<-prev`（5s 超时，超时 fail-loud 返回 error）→ `Start(appCtx, path)` 重建 service（新随机端口）。
- **为何用完成信号而非轮询 `running`**：旧 goroutine 停止后仍会补发 `serve:status {running:false}`，该 emit 与 `running` 原子非同步；轮询会在旧「false」发出前就 `Start` 发新「true」，导致前端误判断开。等 `done` 关闭保证旧「false」先于新「true」。
- 复用现成 `serve:status` 事件——前端 `serveStore` 已监听，自动接新端口。

### 3.3 字段分区与可编辑性

**常用区（默认展开）**

| 段 | 字段 |
|----|------|
| maas | `base_url`、`api_key`(密钥)、`default_profile`、`profiles`(增删改) |
| agents | 子 Agent 映射（名称 ↔ 子配置文件路径），增 / 删 / 改名 / 改路径 |
| runtime | `demo_response`、`max_tool_rounds`、`lazy_tools` |
| session | `enabled`、`default_recent_turns`、`max_turn_chars`、`restore_latest_on_tui_start`、`cache_enabled`、`cache_max_entries` |
| tui | `show_prompt`、`show_thinking`、`color_profile`、`theme`(8 色) |
| context_files | `enabled`、`max_file_chars` |

**高级区（折叠）— 可编辑**

| 段 | 可编辑字段 |
|----|-----------|
| server | `admin_token`(密钥)、`public_health_enabled`、`request_id_header` |
| service | `background_interval` |
| tasks | `max_index_lines`、`max_task_lines`、`max_message_chars`、`active_statuses`、`done_statuses` |
| skills | `registry_url` |
| web | `enabled`、`allow_private_hosts`、`timeout_seconds`、`max_response_kb`、`allowlist` |
| evolution | `degradation_threshold`、`degradation_window_days`、`degradation_scan_minutes` |

**高级区 — 只读展示（危险路径类，改错会挂 serve）**

凡「文件系统路径 / 目录 / 驱动 / 监听地址」一律只读灰显：
- `storage.driver`、`storage.path`
- `server.listen_addr`（附「不生效」标注）
- `context_files.root` / `soul_path` / `tools_path` / `user_path` / `memory_path` / `agents_path`(废弃) / `config_root`(废弃)
- `workspace.docs_root`、`workspace.memory_root`
- `tasks.index_path`、`tasks.root`、`tasks.archive_root`
- `skills.install_root`
> 只读字段仍从磁盘读出并显示，保存时**原样写回**（不丢失），只是 UI 禁止编辑。

> **修订（2026-07-15）**：`agents`（子 Agent 映射）**不属于**只读危险路径，已移入常用区、可编辑。
> 理由：它是用户自己维护的内容映射（名称 = workflow 的 `task.agent_id`，值 = 子配置文件路径），
> 增删子 Agent 是正常配置动作，与 `storage.path`/`tasks.root` 这类改错即挂 serve 的系统路径性质不同。
> 由专用 `AgentsEditor` 渲染（增 / 删 / 改名 / 改路径）。

### 3.4 前端组件

- **`types/config.ts`**：镜像 14 段 schema 的 TS 接口（含只读/密钥标注元数据）。
- **`stores/configStore.ts`**（zustand，同现有 store 风格）：
  `load()`(调 GetConfig+GetConfigPath)、`draft` 状态、`dirty` 跟踪、`save()`(调 SaveConfig)、
  错误态。
- **`components/settings/SettingsModal.tsx`**：模态覆盖层，常用区展开 / 高级区折叠。
- **字段控件**（`components/settings/fields/`）：
  - `ToggleField`（bool）、`NumberField`（int/float）、`TextField`（string）
  - `SecretField`：打码显示 + 👁 reveal 切换（用于 `api_key`、各 profile `api_key`、`admin_token`）
  - `ColorField`：色值输入 + 色块预览（theme 8 色）
  - `ProfilesEditor`：`maas.profiles` map 的增 / 删 / 改（每项 model/base_url/api_key/prompt_cache）
  - `StringListField`：`tasks.active_statuses`/`done_statuses`、`web.allowlist` 数组
  - `ReadonlyField`：只读灰显 + 说明
- 每字段帮助文本取自 `agent.complete.example.json` 各段 `_comment`。
- **入口**：Sidebar **底部**齿轮图标 → 打开设置模态。

### 3.5 保存前守卫

点「保存」时先调 `ListTasks`，若存在进行中任务（active status）→ 弹确认框
（提示「重启内嵌 serve 会中断进行中任务」），用户确认后才继续 SaveConfig。

### 3.6 子 Agent 配置编辑（2026-07-15 增补）

确认子 Agent 的文件名后，可用表单编辑该 Agent 的**配置文件内容**（不只是名称→路径映射）。

**权威 schema**：`internal/agentregistry.AgentConfig` —— `id` / `role` / `maas_profile`（须为主配置
`maas.profiles` 中的 profile 名）/ `context_files` / `workspace` / `skills`。这些路径类字段**可编辑**：
它们是该 Agent 自己的产出目录与 persona 选择，非「改错即挂 serve」的系统路径。

**新增桥**：`serve.ValidateAgentConfig(ctx, path)` —— 薄封装新抽出的
`agentregistry.LoadAgentFile`（Load 亦复用之，单一解析入口），保证 UI 校验与启动期解析完全一致。

**新增绑定**：
- `GetAgentConfig(rel) → {exists, content}` —— 相对主配置目录解析。**文件不存在是合法状态**
  （刚添加的 Agent），以 `exists=false` 表达而非报错；其余读失败仍报错。
- `SaveAll(mainRaw, agentFiles)` —— 取代 `SaveConfig`。**分阶段提交**：全部候选先写临时文件并逐个
  权威校验，**全部通过才**逐个备份（沿用原权限）+ 原子替换，最后**只重启一次** serve。任一不合法 →
  什么都不写。新文件按需建目录、无备份。

**路径逃逸防护**：`agents` 里的路径由用户编辑，`resolveAgentPath` 强制解析结果落在主配置目录子树内，
`..` 或越界绝对路径一律拒绝——否则设置界面即可读写磁盘任意文件。

**前端**：`AGENT_SECTIONS` 镜像上述 schema；`AgentsEditor` 每行「配置 ›」钻入二级页（`AgentConfigPage`，
带返回）；草稿存在 `configStore.agentDrafts`，与主配置**共用一个「保存并重启」**。新 Agent 的文件不存在时
按 `agentTemplate` 播种（id/role 取名称，`maas_profile` 取主配置 `default_profile`，产出目录按名称分命名空间），
baseline 记为 `''` 故必然 dirty，保存时创建。

## 4. 错误处理（Fail-Loud 落地）

- 前端：字段类型强转失败 → 阻止保存、标红该字段，不静默丢弃。
- Go `SaveConfig`：strict unmarshal / Load 校验 / 备份 / 替换 / 重启 任一失败都返回带上下文 error；
  **校验未过绝不写盘**；重启失败响亮报错并保留 `.bak`。
- serve 重启失败 → `serve:error` 事件让 ConnectionBadge 显示断开原因，不留「神秘未连接」。

## 5. 测试策略

**Go — `app_config_test.go`**
- `GetConfig` 往返：写测试 config → 读回字节一致。
- `SaveConfig` happy path：合法 JSON → 文件被替换 + `.bak` 生成 + `config.Load` 能加载新文件。
- `SaveConfig` 拒绝非法：坏 JSON / 未知字段 / 类型错 → 返回 error 且**原文件不变、无 `.bak` 误写**。
- 只读字段原样保留：save 一份含路径字段的 config，回读路径不变。
- serve 重启：e2e 门控（沿用 `LEGION_E2E` 模式，避开 Wails）验证 `Restart` 后新端口可用。

**前端 — vitest（沿用 `slashCommands.test.ts` 风格）**
- `configStore` 的 load → draft → dirty → save 状态流。
- SecretField 打码/reveal 切换。
- 只读字段禁编辑断言。

## 6. 非目标（YAGNI）

- 不做多配置文件切换 / 配置模板市场。
- 不做 `listen_addr` 真实生效（GUI 固定随机端口）。
- 不做路径字段的目录选择器（只读，无需）。
- 不做配置版本历史 / 多份备份（只留最近一份 `.bak`）。

## 7. 待实现清单（概要）

0. legionAgent `serve` 包：`ValidateConfig(ctx, path)` 公开桥函数。
1. `serve_manager.go`：`Restart` 方法。
2. `app_config.go`：`GetConfig` / `GetConfigPath` / `SaveConfig`。
3. Go 单元 + e2e 测试。
4. 前端 `types/config.ts` / `stores/configStore.ts`。
5. 前端 `SettingsModal` + `fields/*` 控件。
6. Sidebar 底部齿轮入口接线。
7. 保存前 active-task 守卫。
8. 前端 vitest。
9. `wails generate module` 重新生成 TS 绑定。
