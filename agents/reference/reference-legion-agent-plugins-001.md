---
id: "reference-legion-agent-plugins-001"
title: "Legion Agent WASM 插件参考手册"
aliases: ["插件手册", "plugin manual", "agent plugins", "WASM 插件", "插件开发"]
type: "reference"
category: "agents/reference"
tags: ["agent", "plugin", "wasm", "wazero", "abi", "signing", "capability", "cli"]
version: "1.8.0"
created: "2026-08-28"
updated: "2026-08-29"
author: "jxncyjq"
status: "published"
parent: "reference-legion-agent-user-manual-001"
children: []
related_docs:
  - id: "design-legion-plugin-system-001"
    relation: "implements"
    path: "../../design/architecture/legion-plugin-system.md"
  - id: "reference-legion-agent-cli-001"
    relation: "related_to"
    path: "./reference-legion-agent-cli-001.md"
  - id: "reference-legion-agent-http-service-001"
    relation: "related_to"
    path: "./reference-legion-agent-http-service-001.md"
  - id: "reference-legion-agent-auth-001"
    relation: "depends_on"
    path: "./reference-legion-agent-auth-001.md"
---

# Legion Agent WASM 插件参考手册

面向三类人：**写插件的人**（第三、四章）、**发插件的人**（第五章）、**装插件的运维**（第六到八章）。第二章是一条能在十分钟内跑通的最小 MVP，从零写到插件真的挂载并被模型调用。

设计背景与取舍见 [[design-legion-plugin-system-001|Legion 插件系统设计方案]]；本手册只讲**当前代码实际做的事**。

<!-- @section: overview -->
## 一、心智模型：装 ≠ 授权 ≠ 挂载

插件从磁盘走到模型手里要过四道，任何一道没过，前面几道都不会替它兜底：

| 阶段 | 动作 | 谁在做 | 结果 |
|---|---|---|---|
| 1 取回 | 下载 + 校验 digest + 解包进缓存 | `plugins install` / `POST .../resolve` | 包在 `cache/sha256/<digest>/` |
| 2 登记 | 往 `plugins.json` 追加一条 entry | `plugins install` | **零授权**：`enabled:false` 且**没有** `grant` 段 |
| 3 授权 | 写入 `grant`（能力 / 主机 / 路径），置 `enabled:true` | `plugins grant` / `POST .../grant` / GUI | 磁盘上的**决定** |
| 4 挂载 | 验签 → 校验 wasm digest → 实例化 → 交叉校验 → 注册工具 | serve 的 loader 在**任务边界**收敛 | 工具进入模型可用清单 |

三条不变量，改代码时不要破：

- **`install` 不等于授权。** 不带 `--grant` 装完的 entry 永远跑不起来，直到有人显式 `grant`。
- **磁盘上「从没人决定过」和「决定了拒绝」是两种形状**，判据是 `grant` 这个键在不在，见 §6.2 的三态编码表。
- **能力不是运行期开关，是链接期事实。** 未授权的能力对应的 host 函数根本不注册，guest 一旦 import 它就**实例化失败**，而不是调用时返回 DENIED。

<!-- @section: mvp -->
## 二、最小 MVP（从零到挂载）

以下每一步都在 Windows + Git Bash 下**照本文原样跑过一遍**（2026-08-28），终点是 `GET /v1/plugins` 返回 `state:"loaded"`。前缀 `agent` 指 `go build ./cmd/agent` 产出的二进制（源码运行时是 `go run ./cmd/agent --`）。

> 这一节的成品在 server 仓 `plugin_example/`：同样的 guest、构建与签名脚本、
> 以及一组用真实 wazero 宿主跑的测试（`go test ./plugin_example/...`）。想直接
> 拿来改，从那里开始比从这里抄快。

### 2.1 准备目录

```bash
mkdir -p mvp/pkg mvp/plugins mvp/cache mvp/src
cd mvp
```

### 2.2 写一个最小 guest（Rust）

`rustup target add wasm32-wasip1` 之后：

`Cargo.toml`：

```toml
[package]
name = "hello-plugin"
version = "0.1.0"
edition = "2021"

[lib]
crate-type = ["cdylib"]

[profile.release]
opt-level = "z"
lto = true
strip = true
panic = "abort"
```

`src/lib.rs`（无第三方依赖，可离线构建）：

```rust
use std::alloc::{alloc, dealloc, Layout};

// op 0 的自述。provides 必须覆盖部署侧声称这个插件提供的每一个工具，
// 否则激活期的交叉校验会拒绝挂载。
const MANIFEST: &[u8] =
    br#"{"name":"hello-plugin","version":"0.1.0","provides":["hello_echo"]}"#;

#[no_mangle]
pub extern "C" fn _initialize() {}

#[no_mangle]
pub extern "C" fn plugin_alloc(size: i32) -> i32 {
    if size <= 0 { return 0; }
    unsafe { alloc(Layout::from_size_align(size as usize, 1).unwrap()) as i32 }
}

#[no_mangle]
pub extern "C" fn plugin_free(ptr: i32, size: i32) {
    if ptr <= 0 || size <= 0 { return; }
    unsafe { dealloc(ptr as *mut u8, Layout::from_size_align(size as usize, 1).unwrap()) }
}

fn write_out(b: &[u8]) -> i64 {
    let ptr = plugin_alloc(b.len() as i32);
    if ptr == 0 { return 0; }
    unsafe { std::ptr::copy_nonoverlapping(b.as_ptr(), ptr as *mut u8, b.len()) };
    ((ptr as i64) << 32) | (b.len() as i64)   // 高 32 位是指针，低 32 位是长度
}

#[no_mangle]
pub extern "C" fn plugin_invoke(op: i32, _ptr: i32, _size: i32) -> i64 {
    match op {
        0 => write_out(MANIFEST),
        // op 1：一次工具调用。真实插件应解析入参 JSON；这里返回固定结果。
        1 => write_out(br#"{"success":true,"output":"hello from wasm"}"#),
        _ => write_out(br#"{"error":"unsupported op"}"#),
    }
}
```

构建并放进包目录：

```bash
cargo build --release --target wasm32-wasip1
cp target/wasm32-wasip1/release/hello_plugin.wasm pkg/plugin.wasm
```

### 2.3 写 `plugin.json`

`sha256` 字段是 **plugin.wasm 的摘要**（64 位十六进制，不带 `sha256:` 前缀）：

```bash
sha256sum pkg/plugin.wasm     # 取这一串填进下面
```

`pkg/plugin.json`：

```json
{
  "name": "hello-plugin",
  "version": "0.1.0",
  "abi": 1,
  "sha256": "<plugin.wasm 的 64 位十六进制摘要>",
  "capabilities": ["log"],
  "limits": { "timeout_ms": 5000, "max_memory_pages": 64, "max_instances": 2 },
  "tools": [
    {
      "name": "hello_echo",
      "description": "Say hello from a wasm plugin",
      "group": "demo",
      "risk_level": "low",
      "input_schema": { "type": "object" },
      "timeout_ms": 3000,
      "sensitive": false
    }
  ]
}
```

### 2.4 签名

```bash
agent plugins keygen --key-id demo-key --private-key demo.key
# 把命令打印出的公钥条目粘进 keyring.json 的 keys 数组
agent plugins sign pkg --private-key demo.key      # 生成 pkg/plugin.sig
```

`keyring.json`：

```json
{ "keys": [ { "id": "demo-key", "algorithm": "ed25519", "public_key": "<keygen 打印的 base64>" } ] }
```

### 2.5 打包发布

包**必须恰好三个文件、平铺、无目录**：

```bash
cd pkg && tar -czf ../src/hello.tar.gz plugin.json plugin.wasm plugin.sig && cd ..
sha256sum src/hello.tar.gz     # 这是 --digest 要用的那一串（tarball 摘要，与 plugin.json 里的不是同一个）
```

把 `src/` 挂到任意静态 HTTP 服务上（生产用 HTTPS）。

### 2.6 配置 `agent.json`

```json
{
  "plugins": {
    "manifest": "plugins.json",
    "root": "plugins",
    "cache": "cache",
    "keyring": "keyring.json",
    "require_signature": true,
    "allow_insecure_sources": true
  }
}
```

> `allow_insecure_sources` 只为本地 `http://` 调试打开；它**只放开 scheme**，digest 校验与验签一条不减。相对路径按**进程工作目录**解析，不是配置文件所在目录。

先建一个空清单：`echo '{ "plugins": [] }' > plugins.json`。

### 2.7 安装（登记，不授权）

```bash
agent plugins install "http://127.0.0.1:18099/hello.tar.gz" \
  --digest "sha256:<tarball 摘要>" --config agent.json
```

输出会明说：**已登记但未授权**。此刻 `plugins.json` 里是 `enabled:false` 且没有 `grant` 段。

### 2.8 授权

```bash
agent plugins grant hello-plugin --capabilities log --config agent.json
```

`--capabilities` 必须**恰好**是插件自己在 `plugin.json` 里声明的那一套：多给一个（插件没要过的）算配置错误，少给一个同样拒绝——**部分授权写出的 entry 永远加载不了**，与其写下再让它失败，不如当场拒绝。能力这一维**不能收窄**，要么全给、要么 `deny`。

主机与路径两维**可以收窄**：`--allowed-hosts` / `--allowed-paths` 取插件声明集合的子集即可，但不得出现插件没声明过的项。

### 2.9 起服务并确认挂载

授权只写到磁盘，要有 serve 才会挂载：

```bash
agent serve --config agent.json
curl -s http://127.0.0.1:18110/v1/plugins
```

```json
{"plugins":[{"name":"hello-plugin","version":"0.1.0","state":"loaded",
  "tools":["hello_echo"],"declared_capabilities":["log"],
  "declared_unresolved":false,"granted_capabilities":["log"]}]}
```

`state:"loaded"` 且 `tools` 里有 `hello_echo`，即 MVP 完成——模型的工具清单里已经有它了。

> **别用 `agent plugins status` 当外部检查手段。** `status` 与 `reload` 是**本进程视图**：它们读的是 serve 装配的那个 loader，跨进程没有视图。在一个新起的 CLI 进程里跑，得到的是 `no plugin loader in this process: plugins are assembled by 'agent serve'` ——它明说自己看不到，而不是回一个像「没有插件」的空答案。运行中的部署，外部检查走 `GET /v1/plugins` 或 GUI 面板。

已经在跑的 serve，改完清单后用 `agent plugins reload`（同样只在该进程内）收敛；`install` / `grant` / `deny` 都只动磁盘，不碰运行中的服务。

<!-- @section: authoring -->
## 三、编写插件

宿主是 [wazero](https://wazero.io)，目标 `wasm32-wasip1`，guest 是 **WASI reactor**（导出 `_initialize`，没有 `_start`）。

### 3.0 先用 SDK

**手写 ABI 只在你要写第三种语言时才需要。** 两个官方 SDK 各自承担四个导出、op 分发、内存管理、指针打包、JSON，以及 op 0 的自述（`provides` 从注册的工具推导，所以自述与分发表不可能对不上——那个对不上会让激活期的交叉校验拒绝挂载）。

| | Rust `sdk/rust/legion-plugin` | Go `pkg/legionplugin` |
|---|---|---|
| 产物体积 | ~49 KB | ~3.3 MB |
| 内存下限 | 4 MiB（64 页） | 32 MiB（512 页） |
| 工具链 | `rustup target add wasm32-wasip1` | 只要 Go 1.24+ |
| 能力开关 | Cargo feature | build tag |

体积或内存敏感、或一个部署要挂很多插件，用 Rust；团队只有 Go 就用 Go——3.3 MB 是标准 Go 运行时的代价，不是 SDK 的。

**Rust**：

```rust
use legion_plugin::{declare_plugin, log_info, ToolCall, ToolResult};

declare_plugin!(
    name = "legion-hello",
    version = "0.1.0",
    tools = [("hello_echo", hello_echo)]
);

fn hello_echo(call: &ToolCall) -> ToolResult {
    match call.argument("name") {
        Some(name) => {
            log_info(&format!("hello_echo called with name={name}"));
            ToolResult::ok(format!("hello, {name}!"))
        }
        None => ToolResult::fail("missing required argument: name"),
    }
}
```

构建：`cargo build --release --target wasm32-wasip1`。

**Go**：

```go
package main

import "github.com/stardust/legion-agent/pkg/legionplugin"

// init 而不是 main：插件是 WASI reactor，宿主实例化后直接调导出，
// main 永远不会被调用。
func init() {
	legionplugin.Serve("legion-hello-go", "0.1.0", legionplugin.Tool{
		Name: "hello_echo",
		Handler: func(call legionplugin.ToolCall) legionplugin.ToolResult {
			name := call.Argument("name")
			if name == "" {
				return legionplugin.Fail("missing required argument: name")
			}
			legionplugin.LogInfo("hello_echo called with name=" + name)
			return legionplugin.OK("hello, " + name + "!")
		},
	})
}

func main() {}
```

构建：`GOOS=wasip1 GOARCH=wasm go build -buildmode=c-shared -o plugin.wasm .`。

两侧的能力开关（Rust 的 feature / Go 的 build tag）必须与 `plugin.json` 的 `capabilities` 以及部署侧 `grant --capabilities` **三处一致**——理由见 §1 的第三条不变量。

Go 侧还有一件 Rust 没有的事：`plugin_alloc` 返回后宿主手上只有一个整数地址，Go 的 GC 看不见它，所以 SDK 自己按住每个缓冲区到 `plugin_free`。这是 SDK 内部的事，作者不需要管，但**自己手写 Go guest 的人必须自己做**，否则宿主会写进已被回收的内存。

下面 §3.1–§3.3 是**底层合同**：用 SDK 时不需要读，写第三种语言的 SDK 时才需要。

### 3.1 guest 必须导出的五样

| 导出 | 签名 | 说明 |
|---|---|---|
| `memory` | — | 线性内存导出，缺了直接拒绝实例化 |
| `plugin_alloc` | `(size i32) -> i32` | 宿主写入参、宿主写返回体都走它；返回 0 视为分配失败 |
| `plugin_free` | `(ptr i32, size i32)` | 宿主在调用结束后归还入参内存 |
| `plugin_invoke` | `(op i32, ptr i32, len i32) -> i64` | 唯一入口。返回值**打包**：高 32 位 = 指针，低 32 位 = 长度 |
| `_initialize` | `()` | 宿主用 `WithStartFunctions("_initialize")` 实例化 |

入参为空时宿主**不调用** `plugin_alloc`，直接以 `ptr=0, len=0` 进入 `plugin_invoke`。

### 3.2 op 表

| op | 常量 | 入参 | 返回 |
|---|---|---|---|
| 0 | `abi.OpManifest` | 忽略 | 自述 JSON：`{"name":…,"version":…,"provides":[工具名…],"extensions":[扩展点名…]}` |
| 1 | `abi.OpCallTool` | `{"call_id":…,"tool":…,"arguments":{…}}` | `domain.ToolResult`：`{"call_id":…,"success":bool,"output":…}` 或 `{"success":false,"error":…}` |
| 2 | `abi.OpObserveToolResult` | `{"call_id":…,"tool":…,"arguments":{…},"success":bool,"output":…,"error":…}` | **被丢弃**。返回一个格式正确的小文档即可（SDK 返回 `{}`）。见 §3.4 |
| 3 | `abi.OpDecideToolCall` | `{"call_id":…,"tool":…,"arguments":{…}}`（还没跑，所以没有结果） | `{"decision":"allow"\|"deny"\|"ask","reason":…}`。**答不出来 = 拒绝**，见 §3.4 |
| 4 | `abi.OpPromptSegment` | 空 | `{"text":…}`。**激活时问一次**，答案在插件挂着期间一直用，见 §3.4 |
| 其它 | — | — | **不要 trap**，返回一个可读的小 JSON 错误体 |

**交叉校验**：激活时宿主拿 op 0 的自述与部署侧的说法对两次——`provides` 少一个部署声称的工具就拒绝挂载；`extensions` 少一个部署**授权**了的扩展点也拒绝挂载（见 §3.4）。工具失败要用 `{"success":false,"error":…}` 表达，不要 trap——trap 会让整个模块死掉。

### 3.3 host 函数（模块名 `legion`）

七个函数，各自由一项能力把门。**未授权的能力不注册对应函数**，guest 一旦 import 就实例化失败——所以不要 import 用不到的函数。

| 能力 | 函数 | 签名 | 请求 / 响应 |
|---|---|---|---|
| `log` | `log` | `(level i32, ptr i32, len i32)` | 无返回值。level：0=debug 1=info 2=warn 3=error；未知 level 按 error 记并标注 |
| `config` | `config_get` | `() -> i64` | 返回部署侧 `plugins.json` 里该 entry 的 `config` 原文 JSON |
| `kv` | `kv_get` | `(kp i32, kl i32) -> i64` | 键是原始字节；返回 `{"found":bool,"value":string}`。宿主按插件**命名空间**限定键，读不到别人的 |
| `kv` | `kv_put` | `(kp, kl, vp, vl i32) -> i64` | 返回 `{"ok":bool}` |
| `http` | `http_request` | `(ptr i32, len i32) -> i64` | 请求 `{"method","url","headers"?,"body"?}`；响应 `{"status","headers"?,"body","truncated"?}` |
| `fs` | `read_file` | `(ptr i32, len i32) -> i64` | 请求 `{"path"}`；响应 `{"path","content","truncated"?}` |
| `tool` | `call_tool` | `(ptr i32, len i32) -> i64` | 请求 `{"call_id"?,"tool","arguments"?}`；响应是 `domain.ToolResult` |

失败统一是错误信封 `{"code":…,"message":…}`：

| code | 含义 |
|---|---|
| `DENIED` | 越过授权边界（主机不在 `allowed_hosts`、路径不在 `allowed_paths`、scheme 非 http/https），同时发出 `plugin/call_failed{category=denied}` 事件 |
| `INVALID_REQUEST` | 请求体不合法（缺 `method`、URL 解析不了等） |
| `HOST_ERROR` | 宿主侧执行失败 |

硬边界：

- HTTP 响应体与文件内容各截断到 **1 MiB**，截断时 `truncated:true` 明说，不会把半截当完整给你。
- HTTP 重定向**每一跳都重新校验** `allowed_hosts`，跳出白名单同样 `DENIED`。
- `call_tool` 单链深度上限 **3**；它还与模型共用同一份 per-task 工具预算，插件调工具会消耗任务额度。

### 3.4 扩展点（extensions）

**能力**是插件调宿主的方向，**扩展点**是宿主调插件的方向。两者的执行方式一样：未授权 = **不存在的接线**，不是运行期的一个 `if`。未授权的能力表现为 host 模块里没有那个函数（实例化就失败），未授权的扩展点表现为宿主**根本没注册**这个观察者，op 2 一次也不会到达 guest。

两个扩展点：

| 扩展点 | 时机 | 能改什么 |
|---|---|---|
| `observe` | 一次工具调用**答完之后** | 什么也改不了 |
| `decide` | 一次工具调用**派发之前** | 只能**收紧**：否决，或要求人工审批 |
| `prompt` | **激活时一次** | 往系统提示词里加一段文字（每次推理都在） |

`observe` 的边界，逐条都是刻意的：

- **它是只读的、单向的。** 观察者没有返回值，宿主读完 op 2 的应答就丢掉。调用方拿到的结果在任何观察者跑之前就已经定了。
- **它看不到没跑起来的调用。** 被权限 / 策略 / 护栏拒掉的调用**从未发生**，通知观察者等于报告一次不存在的执行；handler 返回 Go error 也不通知——那是宿主或工具自身的故障，审计里已经有 `tool_failed`。返回 `success:false` 的调用**会**通知：工具跑了，答了「不行」。
- **它看到的是 agent 跑的任意工具**，不只是本插件自己的——这正是这个 seam 的用途。
- **每次通知 200ms 上限**，跑在调用方的 goroutine 上。超时 / trap 按故障计入本插件的健康度（与工具调用失败同一个计数器，连续失败足够多次会被卸载），并发出 `plugin/call_failed`（`operation` 形如 `observe:<tool>`）。贵的活儿留到自己的下一次工具调用里做。
- **授权是子集语义**：插件声明了 `observe`，部署可以**一个都不授**——那是一个完整的答案（插件照常贡献工具，只是不被咨询），与能力必须**全等**授权不同。


#### `decide`：只能收紧的决策点

宿主在派发任何工具之前问一遍已授权的插件，回答 `allow` / `deny` / `ask`。

- **只能收紧。** 征询发生在宿主自己的 enforcer 与 policy **之后**：宿主已经拒掉的调用根本不会给插件看，所以没有任何位置能把拒绝改成放行。插件回 `allow` 只是「我不反对」，不是授权。
- **取最严，与顺序无关**（`deny` > `ask` > `allow`）。多个插件里任何一个 deny 即拒；一个插件的 ask 不会被另一个插件的 allow 冲淡。命中拒绝就短路——省下的是调用方的预算——代价是错误里只点名第一个反对者。
- **fail-closed：答不出来就是拒绝。** 超时、trap、空体、坏 JSON、未知字段、尾随数据、这个 ABI 不认识的 decision 词，全部拒绝该次调用并计入插件健康度。反过来（fail-open）会让「安全控制」变成「把插件搞崩就能关掉的安全控制」。
- **停摆是有界的。** 一个坏插件确实能拒掉若干次调用，但连续故障到阈值后 G1 会**自动卸载**它，之后没有人再拒绝任何东西。运维遇到批量拒绝时该看的是 `plugin/call_failed`（`operation` 形如 `decide:<tool>`）与该插件的健康度，而不是等。
- **预算 = `min(工具超时/4, 200ms)`。** 征询跑在调用方的 goroutine 上，工具还没开始：声明 300ms 超时的工具不该把其中 200ms 花在被问「能不能跑」上。
- 拒绝会以 `ErrPermissionDenied` 的形式返回，错误文本点名**是哪个插件**、**理由是什么**——一个无法归因的拒绝是运维修不了的拒绝。


##### `ask`：要求人工审批

`ask` 不是拒绝，也不是插件自己控制的等待：宿主在**这一轮的边界**挂起该任务（落 checkpoint、结束本次 run），开一张点名**这个插件**与这条理由的审批票，人批了之后任务从检查点继续。

- **同一条队列。** 和宿主自己的 Sensitive 审批共用一套票据、一个 `/v1/approvals`、一个 SSE 事件、一条恢复路径。票上多两个字段：`requested_by`（`host:sensitive` 或 `plugin:<name>`）与 `reason`（插件自己的话）。
- **不看模式。** 宿主的 Sensitive 审批只在 Manual 模式生效，插件的 ask **在 Auto 模式下同样挂起**。装了守门插件的部署，其意图正是「这几类调用要人看一眼」，按模式忽略它等于把 ask 静默降级成 allow。**代价要认**：一个无人值守的 Auto 任务会停下来等人。
- **没有审批通道 = 拒绝。** 部署里没有接审批设施时，ask 按拒绝处理——没人可问的问题不会自己变成「同意」。
- **派发时还会再问一次。** 决策者在一轮里被问两次：round 边界问「要不要人看」，派发时问「现在能不能跑」。第二次是 fail-closed 的兜底——那时必须已经有一张**批准过**的票，否则拒。因此**决策者必须无副作用**。
- **绕过 runtime 的调用拿不到放行。** 插件自己的 `call_tool`、CLI 直接跑的工具，没有任务上下文也就没有票，ask 在那里一律是拒绝。这不是缺陷：那些路径本来就没有人在旁边看着。

Rust：

```rust
declare_plugin!(
    name = "legion-gatekeeper",
    version = "0.1.0",
    tools = [("status", status)],
    decide = decide_call
);

fn decide_call(request: &ToolDecisionRequest) -> ToolDecision {
    if request.tool == "write_file" && frozen() {
        return ToolDecision::deny("writes are frozen during the incident");
    }
    if request.tool == "deploy" {
        return ToolDecision::ask("deploys are reviewed by a human");
    }
    ToolDecision::allow()
}
```

Go：

```go
legionplugin.Decide(func(req legionplugin.ToolDecisionRequest) legionplugin.ToolDecision {
	if req.Tool == "write_file" && frozen() {
		return legionplugin.Deny("writes are frozen during the incident")
	}
	if req.Tool == "deploy" {
		return legionplugin.Ask("deploys are reviewed by a human")
	}
	return legionplugin.Allow()
})
```


#### `prompt`：往系统提示词里加一段文字

被授予这个扩展点的插件，可以贡献一段文字，进入**每一次**推理的系统提示词。

- **只问一次。** 宿主在激活时问 op 4，答案在插件挂着期间一直用。不是每次构建提示词都问：那会在每个任务的关键路径上多一次 wasm 调用，而且答案可能每次不同——那样这段文字就不能待在稳定前缀里。插件的段落是**部署级**事实，重新挂载才会变。
- **进稳定前缀。** 代价是**挂载/卸载各让前缀缓存失效一次**（低频操作）。不这样做的代价更大：每个任务重发一遍，长期 token 成本更高。**运维请注意**：`plugins reload` 之后第一次推理的 prompt 缓存未命中是**预期行为**，不是 bug。
- **带围栏。** 渲染成这样：

```
--- plugin "legion-jira" (untrusted, provided by a deployment-installed plugin) ---
<插件的文字>
--- end plugin "legion-jira" ---
```

  这是**不可信文本进系统提示词**。模型必须能分辨哪句话来自宿主、哪句来自一个被装上的插件；没有围栏，一个插件写「忽略先前的指令」就和宿主的指令等价。

- **有上限**：单插件 2048 rune、全部合计 8192 rune。超长**截断并留痕**（文本里有标记、日志有 Warn），超总量的那些段落被**拒绝**并在日志里点名。按 rune 不按字节：这保护的是模型上下文，按字节会让中文部署只拿到三分之一。
- **顺序按插件名**，不是挂载顺序：同一份部署每次起来必须逐字节一致，否则前缀缓存白做。
- **答不出来 = 拒绝激活。** 一个被授予 `prompt` 却给不出段落的插件，会让部署以为装上了、而模型从没看见那些指令。空 `text` 则是**合法**回答（这个部署里没话说），不渲染任何围栏。
- 提示词里它是独立的命名块 `plugin_prompt`：提示词涨了 2 KB 时要能回答「哪个插件干的」。

Rust：

```rust
declare_plugin!(
    name = "legion-jira",
    version = "0.1.0",
    tools = [("jira_search", jira_search)],
    prompt = prompt_segment
);

fn prompt_segment() -> String {
    String::from("When citing a Jira issue, link it as https://jira.example.com/browse/KEY.")
}
```

Go：

```go
legionplugin.Prompt(func() string {
	return "When citing a Jira issue, link it as https://jira.example.com/browse/KEY."
})
```

**三处联动**（少一处就不通，且各自的失败点不同）：

| 位置 | 缺了会怎样 |
|---|---|
| guest 里注册观察者（SDK 一行） | 部署授权了却没实现 → **激活期拒绝**，`detail` 点名这个扩展点 |
| `plugin.json` 的 `extensions` | `grant --extensions` 直接拒绝：没声明的东西授不了 |
| `agent plugins grant --extensions observe` | 宿主不注册观察者，op 2 永不到达（**静默且正确**：这就是未授权的含义） |

SDK 里各是一行。Rust：

```rust
declare_plugin!(
    name = "legion-hello",
    version = "0.1.0",
    tools = [("hello_echo", hello_echo)],
    observe = log_observation
);

fn log_observation(o: &ToolObservation) {
    log_info(&format!("observed tool={} success={}", o.tool, o.success));
}
```

Go：

```go
func init() {
	legionplugin.Serve("legion-audit", "0.1.0", legionplugin.Tool{ /* … */ })
	legionplugin.Observe(func(o legionplugin.ToolObservation) {
		legionplugin.LogInfo("saw " + o.Tool)
	})
}
```

两个 SDK 的 op 0 自述里，`extensions` 都是**从注册推导**的，与 `provides` 同源：作者不需要维护第二份清单，也就不可能让它和实际接线对不上。


<!-- @section: services -->
## 三点五、命名服务：按能力依赖，而不是按工具名

插件之间原本只能按**具体工具名**耦合：A 要用 B 的能力，就得在 `plugin.json` 的 `requires` 里写死 B 的工具名。换一个实现 = 改所有消费者。

命名服务是这层间接：

```jsonc
// 提供方
"provides_services": ["issue-tracker"]
// 消费方
"requires_services": ["issue-tracker"]
```

四条规则：

- **无人提供 = 消费者挂起**，不是卸载；提供者到场即恢复。走的是 `requires` 那套依赖收敛，不是第二套机制。
- **一个服务名一个提供者，先到先得。** 第二个声明同名服务的插件**激活失败**，错误点名占用者。既不静默让位（那样「谁在提供它」说不清，而两边都显示 `loaded`），也不静默顶替（装上一个插件就换掉别人的实现，消费者毫无察觉）。
- **「先到」按 `plugins.json` 的声明顺序**，不是挂载完成的时序——后者随机器负载变化，会让同一份部署两次启动的服务归属不同。
- **服务名与工具名是两个命名空间。** 服务可以与某个工具同名而不冲突：模型从不看见服务名，也没有任何调用经由它派发。挂起原因里服务写作 `service:<名>`，一眼能看出缺的是哪一类东西。

排查：`agent plugins status` 与 `GET /v1/plugins` 的行里带该插件**提供**与**需要**的服务；提供者自己也挂着时，消费者的诊断显示为级联（点名那个提供者），而不是「没人提供它」。

> **本期不做**：`call_tool` 还不能写 `service:<名>/<能力>` —— 消费者目前仍按工具名发起调用，服务只决定生命周期（谁在、谁挂起）。名字解析是下一期（D1b）。


<!-- @section: manifest -->
## 四、`plugin.json` 清单规范

解析是 `DisallowUnknownFields`：**多一个字段就拒绝**，任何嵌套层级都一样。

| 字段 | 类型 | 规则 |
|---|---|---|
| `name` | string | 必填 |
| `version` | string | 必填 |
| `abi` | int | 必须是 `1` |
| `sha256` | string | plugin.wasm 的摘要，**恰好 64 位十六进制**，无前缀；加载时与实际字节比对 |
| `capabilities` | []string | 只能取 `log` / `config` / `kv` / `http` / `fs` / `tool` |
| `limits.timeout_ms` | int | 单次调用超时 |
| `limits.max_memory_pages` | uint32 | **不得为 0**（64 KiB 一页） |
| `limits.max_instances` | int | **≥ 1** |
| `network.allowed_hosts` | []string | 需要 `http` 能力时声明可达主机 |
| `filesystem.allowed_paths` | []string | 需要 `fs` 能力时声明可读路径 |
| `tools` | []ToolDecl | **不得为空**——插件的存在理由就是贡献工具 |
| `tools[].name` | string | 必填，且**不得重名** |
| `tools[].group` | string | 必填，没有 group 的工具无法进能力目录 |
| `tools[].timeout_ms` | int | 必须 > 0 |
| `tools[].description` / `input_schema` / `risk_level` / `sensitive` | — | 呈现给模型的元信息 |
| `provides_services` | []string | **可选**。本插件能充当的**能力名**（见 §三点五）。一个名字一个提供者，先到先得；第二个声明者激活失败 |
| `requires_services` | []string | **可选**。本插件需要谁来提供的能力名。无人提供 → 本插件**挂起**（不是卸载），提供者到场即恢复 |
| `requires` | []string | 本插件通过 `call_tool` 调用的**别的插件的工具名**；不得为空串、不得重复、不得写自己贡献的工具 |
| `extensions` | []string | **可选**。本插件**实现**了哪些宿主扩展点，当前是 `observe` / `decide` / `prompt`（见 §3.4）。它是授权的上界：`grant.extensions` 只能取它的子集，而部署授权了、这里却没有（或 guest 实际没注册）会在**激活期**被拒 |
| `config_schema` | object | **可选**。声明本插件期望的部署侧配置形状（JSON Schema 的一个子集，见 §4.1）。声明了就会在**加载期**校验 `plugins.json` 的 `config`，不合则该条 `failed` 且 `detail` 点名字段；不声明则配置原文直传给 guest，与既有行为一致 |

`requires` 与 `capabilities` 不同类：能力在加载期检查，缺了直接拒载；`requires` 未满足（提供方不在）是**可恢复的 suspended 态**，提供方回来即可恢复。

### 4.1 `config_schema`：够用的子集

部署方在 `plugins.json` 的 `config` 里写的东西，宿主原本一个字都不校验——写错一个键名或给错类型，要等到运行时在 guest 里炸，而那是整条链路上报错能力最弱的一层。声明 `config_schema` 就把这件事挪到加载期，并且**点名是哪个字段错在哪**：

```
plugin "legion-jira": config.auth.token: missing required field "token"
plugin "legion-jira": config.hosts[1]: want a string, got a number
plugin "legion-jira": config.retries: want an integer, got 1.5
```

支持的关键字**只有这七个**，其余（`$ref`、`allOf`、`patternProperties`…）在**解析插件包时**就被按名字拒绝——忽略不认识的关键字会让作者以为自己的约束生效了：

| 关键字 | 可出现在 | 语义 |
|---|---|---|
| `type` | 根与每个属性 | `object` / `string` / `number` / `integer` / `boolean` / `array` |
| `properties` | `type: object` | 字段名 → 子 schema |
| `required` | `type: object` | 必填字段名，**每个都必须在 `properties` 里** |
| `additional_properties` | `type: object` | **缺省 `false`**：未声明的字段是错误 |
| `items` | `type: array` | 元素的子 schema |
| `enum` | 标量 | 允许的取值 |
| `description` | 任意 | 只给人看 |

例子：

```json
"config_schema": {
  "type": "object",
  "properties": {
    "endpoint": { "type": "string", "description": "Jira base URL" },
    "retries":  { "type": "integer" },
    "mode":     { "type": "string", "enum": ["fast", "safe"] },
    "hosts":    { "type": "array", "items": { "type": "string" } }
  },
  "required": ["endpoint"]
}
```

四条要知道的规则：

- **`additional_properties` 缺省 false**：一个写错的键名如果被悄悄忽略，看起来和「这个设置没生效」一模一样，运维没有任何线索。要放开就显式写 `true`。
- **嵌套深度上限 5**：校验器是递归的，而它读的文档是随不可信插件一起来的。
- **`integer` 与 `number` 是分开的**：`retries: 1.5` 会被拒绝。
- **不写 `config` 等于写了 `{}`**：整段省略不能绕过 `required`。

不做的事：schema 的默认值填充（那会让 `config_get` 拿到的东西与 `plugins.json` 里写的不一致）、跨版本 schema 迁移、完整 JSON Schema（要引第三方依赖，而这条路上的每个字节都跟着不可信插件进部署）。

<!-- @section: packaging -->
## 五、打包、签名与发布

### 5.1 包布局

一个 `.tar.gz`，**恰好三个文件，平铺，不带目录**：

```
plugin.json      清单
plugin.wasm      模块（其摘要写在 plugin.json 的 sha256 里）
plugin.sig       plugin.json 原始字节的分离签名
```

多出第四个文件即拒绝整包。解包上限：**16 个条目 / 单条目 64 MiB / 解压后总量 128 MiB**；下载侧另有 30 s 超时与 32 MiB 压缩字节上限（可配）。

### 5.2 签名与信任集

```bash
agent plugins keygen --key-id <id> --private-key <file>   # 已存在的私钥文件绝不覆盖
agent plugins sign <包目录> --private-key <file>          # 写出 plugin.sig，写前先自验
```

签名覆盖的是 **`plugin.json` 的磁盘原始字节**，不重新编码——所以签完再手改 `plugin.json` 一个空格，验签就会失败。

`plugin.sig`：

```json
{ "key_id": "demo-key", "algorithm": "ed25519", "signature": "<base64>" }
```

`keyring.json`（部署侧的信任集，由 `plugins.keyring` 指向）：

```json
{
  "keys": [
    { "id": "demo-key", "algorithm": "ed25519", "public_key": "<base64>" },
    { "id": "ops-2026", "algorithm": "ed25519", "public_key": "<base64>" }
  ],
  "revoked": [
    { "key_id": "demo-key", "revoked_at": "2026-08-29T10:00:00Z", "reason": "laptop stolen" }
  ]
}
```

#### 吊销：作废一把泄漏的私钥

`revoked` 是可选的。`key_id` 必填，`revoked_at`（RFC 3339）与 `reason` 可选、只用于把拒绝说清楚。四条语义：

- **吊销压过 `keys`。** 被吊销的钥匙即使公钥还留着也验不过。**保留公钥是刻意的**：它让错误能说「这把钥匙被吊销于 2026-08-29（laptop stolen）」，而不是含混的「未知的 key id」。
- **可以吊销一把已经从 `keys` 里删掉的钥匙**：那条记录正是让旧包的失败解释得清的东西。
- **同一个 id 吊销两次 → 拒绝解析**：哪条算数、拒绝时引用哪个时间戳，没有答案就不猜。
- **所有 key 都被吊销 → 拒绝解析**：那是空信任集，配上强制验签等于拒绝一切插件，在解析期说出来比在每次挂载时说好。

被吊销钥匙签的包**就是不可信的包**：HTTP 侧 422、界面不给重试（重试一万次那把钥匙还是被吊销的）、并且**会被自动清出缓存**。

**吊销什么时候生效——这条必须记住：不是即时。** 信任集在 `agent serve` 装配时冻结，运行中换不了。改完 `keyring.json` 之后：

- `agent plugins reload` 会**拒绝**收敛，并说明本进程仍在按旧信任集运行、需要重启 serve；
- **重启 serve 之后**吊销才生效：那把钥匙签的插件在收敛时验签失败、状态变 `failed` 并点名，缓存里的那份包同时被清掉。

拒绝而不是照做，正是这条守卫的价值：否则运维会以为改完就算数，而进程里那把钥匙还在验包。

**验签失败的四条路径里只有三条算「不可信」**：签名缺失、签名文件不可解析、验不过——这三条 `errors.Is(err, manifest.ErrUntrustedPackage)` 为真，界面上不给重试。第四条（读 `plugin.sig` 时的 I/O 错误）**刻意不算**：磁盘故障是可重试的环境问题，混进信任信号会让真正的安全事件贬值。

### 5.3 发布与来源

| 来源形态 | 例子 | 规则 |
|---|---|---|
| 远程 | `https://host/pkg.tar.gz` | **必须**带 `digest`（`sha256:` + 64 位十六进制） |
| 远程（明文） | `http://host/pkg.tar.gz` | 需 `plugins.allow_insecure_sources: true`；只放开 scheme，digest 与验签照旧 |
| 本地 | `./plugins/hello-plugin` | 相对 `plugins.root` 解析，不得逃逸出 root |
| 其它 scheme | `file://` `ftp://` `ssh://` | **点名拒绝**，不会被当作本地路径悄悄放行 |

<!-- @section: deployment -->
## 六、部署侧配置

### 6.1 `agent.json` 的 `plugins` 段

| 字段 | 默认 | 说明 |
|---|---|---|
| `manifest` | 空 | 部署清单路径。**空 = 完全不启用插件**（合法部署）；配了但读不了/解析不了 → serve 启动失败 |
| `root` | `plugins` | 本地 source 的解析根，也是可读边界 |
| `cache` | **无默认** | 远程包解包落盘处。有远程 entry 却没配 → serve 启动失败（不会替你挑目录） |
| `keyring` | 空 | 信任集路径；要求签名时必须非空 |
| `require_signature` | **nil = 要求** | 指针型：区分「写了 false」与「没写」 |
| `allow_insecure_sources` | **nil = 拒绝** | 同上 |
| `fetch.timeout_ms` | 30000 | 单次下载端到端超时 |
| `fetch.max_bytes` | 33554432 (32 MiB) | 压缩字节硬上限；0 不是「无限」，是配置错误 |
| `limits.timeout_ms` | 10000 | 单次进插件调用的超时，也是插件 HTTP 客户端的超时 |
| `limits.max_memory_pages` | 256 | 部署级天花板；0 = 部署不声明，用插件自己的 |
| `limits.max_instances` | 4 | 同上 |
| `apply_wait_ms` | 60000 | 收敛等待**任务边界**的上限（不是固定等待） |
| `health.max_consecutive_faults` | 5 | **连续**故障（timeout·trap·abi）达到此数即自动卸载该插件；一次成功调用清零；`denied` 与失败的 `ToolResult` 都不计。0 不是「不限」，是配置错误，Load 期即拒绝 |

> 所有相对路径按 **进程工作目录** 解析，与配置文件位置无关：`agent serve --config /etc/agent.json` 从 `/srv` 启动读的是 `/srv/plugins.json`。配置不在工作目录时请用绝对路径。

### 6.2 插件登记清单 `plugins.json`

```json
{
  "plugins": [
    {
      "name": "hello-plugin",
      "source": "https://host/hello.tar.gz",
      "digest": "sha256:<64 hex>",
      "enabled": true,
      "grant": {
        "capabilities": ["log", "http"],
        "allowed_hosts": ["api.example.com"],
        "allowed_paths": [],
        "extensions": ["observe"]
      },
      "tools": [ { "name": "hello_echo", "risk_level": "low", "sensitive": false } ],
      "config": { "任意": "交给 config_get 的原文" }
    }
  ]
}
```

`tools[]` 是部署侧的**接受**清单：`name` 必须是插件声明过的工具；`risk_level` / `sensitive` 只能把插件自己的声明**收紧**，不能放松（`sensitive` 是指针，缺省 = 用插件的声明）。

`grant.extensions` 与 `grant.capabilities` 的规则**不同**，别把它们统一了：能力必须与插件声明**全等**（部分授权写出的 entry 永远加载不了），扩展点可以是**子集**，缺省 = 一个都不授（见 §3.4）。

**三态磁盘编码**（判据是 `grant` 这个**键在不在**，不是 capabilities 空不空——纯计算插件本来就零能力）：

| 磁盘形状 | 含义 | `status` |
|---|---|---|
| `enabled:false`，**完全没有 `grant` 键** | 已登记，从没人决定过 | `unauthorized` |
| `enabled:false`，**`grant` 键仍在**（能力已清空） | 决定过，撤销了 | `disabled` |
| `enabled:true` + 能力齐全 | 已授权 | `loaded` / `failed` / `suspended` |

<!-- @section: install-authorize -->
## 七、安装、授权与取回

### 7.1 CLI

| 命令 | 用途 | 关键参数 |
|---|---|---|
| `agent plugins status` | 逐条报告清单里每个插件到底成了什么 | `--config`；**本进程视图**，见下 |
| `agent plugins reload` | 重读清单并收敛运行中的插件 | `--config`；**本进程视图**，见下 |
| `agent plugins install <url>` | 取回 + 校验 + 解包 + 登记 | `--digest`（必填）、`--grant`（可选，直接授权）、`--config` |
| `agent plugins grant <name>` | 授权一条已登记的 entry | `--capabilities`（必须**恰好**等于插件声明的集合）、`--allowed-hosts` / `--allowed-paths`（可取声明集合的子集）、`--extensions`（可取声明集合的子集，缺省一个都不授；`decide` 授出去的是**否决工具调用**的权力，见 §3.4）、`--config` |
| `agent plugins deny <name>` | 撤销授权（保留 `grant` 键 → `disabled`） | `--config` |
| `agent plugins keygen` | 生成 Ed25519 密钥对 | `--key-id`、`--private-key`（不覆盖已存在文件） |
| `agent plugins sign <dir>` | 对包目录的 `plugin.json` 签名 | `--private-key` |
| `agent plugins cache list` | 列出缓存里每个包：digest、大小、修改时间、是否完整、**是否仍被清单引用** | `--config` |
| `agent plugins cache remove <digest>` | 删一条。仍被引用也删（运维点名了就删），但会打印是谁还指着它 | `--config` |
| `agent plugins cache prune` | 删掉清单不再引用的条目，外加超过 1 小时的 `.unpack-*` 残留 | `--dry-run`、`--max-bytes`、`--config` |

**七个子命令分两组，除了名词以外不共享任何东西**：

- `status` / `reload` 是**这个进程**的视图——两者都读 serve 装配的那个 loader。**跨进程没有视图**：在一个只跑 CLI 的进程里执行，会明确报告「本进程没有 loader」，而不是回一个像「没有插件」的空答案。
- `keygen` / `sign` / `install` / `grant` / `deny` 不碰任何 loader、不启动服务。`keygen` 与 `sign` 连配置都不读——它们**生产**验签所消费的东西，与验签一起发布是刻意的：一个能验签却造不出签名的部署，剩下的唯一选择就是把验签关掉，而那正是验签要防的事。`install` / `grant` / `deny` 读插件配置（复用与 serve 相同的缓存、下载上限、来源策略与信任集）并写清单，但它们做的都是**磁盘上的登记或决定，不是启动**：任何一个都要等 `reload` 才会到达运行中的进程。

`install --grant` 只管能力，**不授任何扩展点**：装的时候把工具接进来、把观察权留到后面单独决定是刻意的默认。要授就装完再跑一次 `agent plugins grant <name> --capabilities … --extensions observe`。

`install --grant` 必须**恰好**列出插件声明的能力全集（部分授权写出的 entry 永远加载不了）；插件声明了非空 `allowed_hosts` / `allowed_paths` 时，`--grant` 直接拒绝 `http` / `fs`——主机与路径白名单是 `grant` 命令的活。

`install` 从不重载运行中的服务，装完记得 `reload`。

### 7.1.1 缓存治理

`agent plugins cache` 的三条语义，写下来是因为它们都是**刻意的取舍**：

- **`prune` 永不删仍被引用的条目**，`--max-bytes` 也不例外。拿部署指着的包去凑磁盘指标，是把「空间不够」变成「下次 reload 挂载失败」。降不到预算就如实报告差多少并非零退出，让运维自己决定是改 `plugins.json` 还是点名删。
- **`--max-bytes` 让 prune 变成增量的**：不带预算时清掉所有未引用条目；带预算时只清到刚好达标为止，**最旧的先走**。今天没被引用的包，可能在有人回滚 `plugins.json` 的下一秒又被引用——热缓存值钱。
- **「最旧」指最后一次写入，不是最近使用。** 这个部署不记录缓存读取（记录它意味着每次命中都要写盘），把它叫 LRU 是撒谎。
- **`--dry-run` 先看再删**：一条会动磁盘的命令，应该能先告诉你它打算删什么。
- **`.unpack-*` 残留只清超过 1 小时的**：正在下载的暂存目录和崩溃遗留的长得一模一样，年龄是唯一能区分它们的东西。

另外，**验签失败的包会被自动移出缓存**，不需要运维动手：那份字节刚刚没通过信任校验，留在部署会读的目录里没有道理。仅限**信任失败**且**远程条目**——包损坏只会让下次重下一份同样的坏包，而本地条目的目录是运维自己的文件树。

### 7.2 HTTP 端点

| 方法 路径 | RBAC | 语义 |
|---|---|---|
| `GET /v1/plugins` | `read_plugin` | 列出每条 entry 的状态、声明能力、已授能力 |
| `POST /v1/plugins/{name}/grant` | `write_plugin` | 授权并触发收敛 |
| `POST /v1/plugins/{name}/deny` | `write_plugin` | 撤销并触发收敛 |
| `POST /v1/plugins/{name}/resolve` | `write_plugin` | **只取回声明**：下载 + 验签 + 解析，到此为止——不碰 `plugins.json`、不写 grant、不触发收敛。按**副作用**归类进 `write_plugin`（发出站请求 + 往缓存写盘），不是按「改没改授权」 |

`grant` / `deny` 的响应有**四种**而不是三种（端点先写盘再收敛，故除写盘失败外磁盘都已改）：

| 情形 | 响应 |
|---|---|
| 写盘失败 | 4xx/5xx，磁盘未动 |
| 收敛完成 | 200 |
| 收敛**没跑**（等超时 / 取消 / 另一个 apply 在跑） | 200 + `pending_convergence:true` |
| 收敛跑了但这一条激活失败 | 200 + `state:"failed"` |

`resolve` 拿到不可信包时返回 **HTTP 422**，界面据此**不给重试**（重试一万次签名还是不对）。

`POST /v1/plugins/{name}/grant` 的请求体：`capabilities` `allowed_hosts` `allowed_paths` `extensions`——四个字段与 CLI 的四个参数一一对应，走**同一批**校验函数（`internal/plugin/consent`），包括「能力全等、扩展点子集」这条区别。

`GET /v1/plugins` 的一行（`PluginView`）字段：`name` `version` `state` `detail?` `tools[]` `declared_capabilities/hosts/paths` `declared_extensions` `declared_unresolved` `declared_unresolved_reason?` `declared_error?` `granted_capabilities/hosts/paths` `granted_extensions`。声明与已授分开报告的理由在扩展点上同样成立：同意对话框拿**声明**画勾选项，拿**已授**画当前状态。

`declared_unresolved_reason` 的三个取值决定界面给不给「取回声明」：

| reason | 含义 | 可取回？ |
|---|---|---|
| `not_cached` | 远程包还没下载 | ✅ 唯一可取回的 |
| `no_cache_configured` | 部署没配 `plugins.cache` | ❌ 改配置 + 重启 serve |
| `load_failed` | 包在但加载不了（坏 wasm、坏签名、缺文件） | ❌ 再取回读到的是同样的坏字节 |

### 7.3 GUI

**扩展点在同意流里逐项勾选，且默认全不勾选**：能力是只读清单（声明不是菜单），主机 / 路径默认全选可缩小，扩展点相反——它不是缩小一次既有授权，而是新增一份宿主本来不给的权力，其中 `decide` 能否决 agent 的工具调用。每一项下面写明授出去的到底是什么。CLI 的 `--extensions` 不写就是不授，界面不能是同一个问题的第二个更松的答案。

设置 → 插件：每行一条 entry，未解析的行给次要样式的「取回声明」，「授权」在声明可见之前一直禁用——**不能对看不见的清单点同意**。取回成功后常驻「已取回并缓存该插件包（未授权，可随时撤销）」，并把授权对话框切换到刚取回的声明。取回 / 授权 / 收敛进行中，Esc、标题栏 X、点背景、切 tab 四条路径全部被拦——一个按下去必然无效的取消按钮就是在骗人。

GUI 与 CLI 走**同一批校验函数**（`internal/plugin/consent`），不是第二条各说各话的授权路径。

**面板会自己刷新。** 六个插件事件（`plugin/loaded` / `unloaded` / `suspended` / `resumed` / `activation_failed` / `unload_leaked`）经 `/v1/events` 的 SSE 到达 GUI，面板订阅后**去抖 300ms 重新拉一次 `GET /v1/plugins`**：收敛完成、健康度自动卸载、依赖满足后恢复，都不必手点刷新。

三件配套的事实：

- **事件只是「有什么变了」的信号**，界面状态一律从 `GET /v1/plugins` 重取——事件的 `message` 是给人看的一行文本，照它打补丁就会把界面绑死在一个没人承诺的字符串格式上；
- **仍然没有轮询**。没有事件就没有请求；断线由 SSE 桥自己重连，重连后面板打开时的那次全量拉取即是补齐；
- 「刷新」按钮**没有取消**：它和自动刷新走同一条 `load()`，需要立刻确认时随时可以点。

<!-- @section: runtime -->
## 八、运行时状态与收敛

`plugins status` 的状态由**清单**与**loader 实况**合并得出，两边缺一都报不全：

| 状态 | 来源 | 含义 |
|---|---|---|
| `unauthorized` | 清单 | 已登记，从没授权过 |
| `disabled` | 清单 | 授权被显式撤销 |
| `pending` | 清单 | 已授权，等待收敛 |
| `loaded` | loader | 已挂载，工具在册 |
| `suspended` | loader | **仍挂载**（同一个 runtime、同一份 guest 状态），但因 `requires` 的提供方缺席而暂时撤下工具；提供方回来即恢复 |
| `failed` | loader | 目标状态是挂载，但什么都没跑起来 |

收敛在**任务边界**发生：`apply_wait_ms` 是等待边界的**上限**而非固定等待，空闲的 serve 上收敛瞬间完成。

### 8.1 运行期健康度

一次进插件的调用失败，按四类归入 `plugin/call_failed` 事件的 `category`：

| category | 含义 | 计入健康度 |
|---|---|---|
| `timeout` | 超过 `limits.timeout_ms` 没答上来 | ✅ |
| `trap` | wasm trap：越界、`unreachable`、除零 | ✅ |
| `abi` | ABI 违约：`plugin_alloc` 返回 0、结果指针越界、返回体解不开 | ✅ |
| `denied` | host 函数按能力/白名单拒绝 | ❌ 插件越界，不是坏了 |

另外两种**根本不是故障**：guest 返回 `{"success":false}`（工具说「我没做成」，是业务答案），以及调用方自己取消（插件没机会答——否则多按几次中断就能卸掉一个健康插件）。

计数是**连续**的：一次答上来就清零。连续故障达到 `health.max_consecutive_faults` 时，该插件被**自动卸载**，状态转 `failed`，`detail` 形如：

```
health: 5 consecutive faults (last: category=trap tool=demo_echo: invoke op=1: guest trap)
```

同时发 `plugin/unloaded` 且 `reason=health`——它是唯一一次没人要求的卸载，扫事件时要能与改清单区分开。

**不会自动重试，也不会自动装回来**：trap 到阈值的插件下一次多半还是 trap，自动重装会把一次看得见的卸载变成一个看不见的循环。修好包之后走 `agent plugins reload`。

卸载时若等不到在途调用收敛（`drain` 超时），发 `plugin/unload_leaked`，带**在途数**与实际等待时长——「还有 3 个调用在里面」和「还有 1 个」的处置完全不同。

`reload` 有一条要记住的限制：**信任集在 serve 装配时冻结**，运行中换不了。收紧了 keyring 再 `reload`，得到的会是**新清单 + 旧信任集**——所以 `reload` 干脆**拒绝**：它比对配置里的签名策略（是否强制、可信 key id、**已吊销 key id**）与本进程正在跑的那一套，不等就报错并要求重启 serve。加一条吊销同样触发这条守卫，见 §5.2。


<!-- @section: reachability -->
## 八点五、插件的工具是怎么到达模型的

一句话：**每个任务的工具注册表继承插件注册表**。

```
per-agent 注册表（policy / 权限 / 护栏 / 审计都是自己的）
        └── parent: 插件注册表（插件的工具 + observe/decide seam）
```

四条由此而来的事实：

- **能看见也能调用。** 插件的工具进目录（模型看得见）也进派发路径（调得动）。
- **在本 agent 的规矩下跑。** 继承的是**解析**，不是策略：插件的工具受本任务的执行策略、护栏与审计约束，不会把插件注册表的规矩带过来。声明 `risk_level: high/critical` 的插件工具照样被执行策略拒掉——和内置工具一个待遇。
- **`reload` 立刻生效。** 这条链接是**引用**不是快照：新挂的插件对已经存在的任务注册表也可见，卸载后立刻消失。
- **同名时内置赢。** 任务注册表自己注册的名字遮蔽继承来的同名工具——反过来会让一个插件悄悄替换 `write_file`。

权限方面：角色白名单是编译期的，而插件的工具名运行期才有，所以权限判定多了一个**动态来源**（「这个名字是不是插件贡献的」）。它不是绕过：排在 per-agent 覆盖之后、与白名单同一条路径，`disabled_tools` 照样能禁掉一个插件工具（插件工具在 gateable 目录里，这是从一开始就立下的规矩）。

**扩展点也因此才真的生效**：`observe` / `decide`（含 `ask`）挂在插件注册表上，而通知与征询本来就沿 parent 链走——所以插件现在看得见、也拦得住 **agent 自己的**工具调用，不只是插件通过 `call_tool` 发起的那些。

<!-- @section: troubleshooting -->
## 九、排错

| 现象 | 多半是 |
|---|---|
| `abi is 0, want 1` | `plugin.json` 漏了 `"abi": 1` |
| `sha256 mismatch` | 改了 wasm 没同步更新 `plugin.json` 的 `sha256` |
| `parse plugin manifest: unknown field` | `plugin.json` 多写了字段（解析器拒绝未知字段） |
| `tools is empty` | 插件不贡献工具就没有加载的理由 |
| 实例化失败，提示缺 import | guest import 了未被授权能力对应的 host 函数 |
| `cross-check manifest: … missing` | op 0 的 `provides` 没覆盖部署声称的工具 |
| `cross-check manifest: the deployment grants … extension` | 授权了扩展点，但 guest 的 op 0 自述里没有它——插件没注册观察者（或用的是不带 `observe` 的旧构建）。要么重新构建，要么把它从这条 entry 的 `grant.extensions` 里去掉 |
| `grant --extensions` 被拒，说插件没声明 | `plugin.json` 的 `extensions` 里没有它。授一个插件没要过的扩展点是配置写错了，不是慷慨 |
| 任务停在「等待审批」，票上写着某个插件 | 那个插件答了 `ask`。在 GUI 的审批卡片（或 `GET /v1/approvals`）里能看到 `requested_by` 与 `reason`；批准后任务从检查点继续。**Auto 模式也会这样停**——那是 `ask` 的语义，不是 bug |
| `plugins reload` 之后第一次推理慢/贵 | **预期**：插件的提示词段在稳定前缀里，挂载或卸载会让前缀缓存失效一次。之后的任务照常命中 |
| 插件挂不上，`detail` 提到 prompt segment | 它被授予了 `prompt` 却答不出段落（trap/空体/坏 JSON）。被授予却给不出文字的插件不会被挂上——否则部署以为装了指令，而模型从没看见 |
| 提示词莫名变长 | 看 `plugin_prompt` 这个命名块：它是插件贡献的全部文字。单插件超 2048 rune 会被截断（文本里有标记、日志有 Warn），合计超 8192 rune 的段落会被拒绝并在日志里点名 |
| 人已经批了，调用还是被拒 | 派发时那次兜底没找到批准的票。两种常见原因：①这次调用没有任务上下文（插件自己的 `call_tool`、CLI 直接跑），那里本来就没人看着；②lazy 协议下票开在了外层 meta 调用上——票必须按**内层**调用开，这是宿主内部的一致性，遇到请报 bug |
| 工具调用批量被拒，错误里点着某个插件 | 那个插件的决策点在拒。看它是**主动拒**（reason 是插件自己的话）还是**答不出来**（reason 里有 timeout/trap/decode，且有 `plugin/call_failed{operation=decide:*}`）。后者是 fail-closed 生效：坏插件会在连续故障到阈值后被自动卸载，也可以直接 `agent plugins deny <name>` 立刻停掉 |
| 授权了 `decide` 但插件没实现 | 与 observe 同：激活期拒绝，`detail` 点名扩展点 |
| 授权了 `observe`，观察者却从没被调用 | 先确认这一条真的 `loaded`；再确认这次调用**跑起来了**——被权限/策略/护栏拒掉的调用从不通知观察者（§3.4）。另外插件自身的 `plugin/call_failed`（`operation` 是 `observe:<tool>`）说明它被调过但失败了 |
| `guest exports no linear memory` | 没导出 `memory`（构建目标不对，或不是 cdylib） |
| 装完了跑不起来 | 正常：`install` 不授权，去 `grant` |
| 某个插件一直 `suspended`，`waiting_on` 里是 `service:xxx` | 没有插件提供这个能力，或提供者自己也挂着（诊断会点名它）。装一个 `provides_services` 含该名字的插件，或先修好提供者 |
| 插件 `failed`，错误说服务已被别人提供 | 一个服务名只有一个提供者，先声明的保留。要换提供者：把占用者从 `plugins.json` 里去掉（或 `deny`），再让新的那个上 |
| 模型看不到插件的工具 | 先确认这条 entry 是 `loaded`；再确认它没被该 agent 的 `disabled_tools` 禁掉。插件工具经**继承**到达每个任务的注册表（§八点五），所以 `loaded` 之后不需要重启 serve |
| 插件工具被拒，错误是 permission denied | 两种：①该工具声明了 `risk_level: high/critical`，执行策略拒了它（与内置工具同一条规则）；②它被 `disabled_tools` 禁了 |
| `grant` 报缓存未命中且没配 cache | `plugins.cache` 缺失，远程包无处落盘 |
| HTTP 调用返回 `DENIED` | 目标主机不在 `allowed_hosts`，或重定向跳出了白名单 |
| 响应/文件内容被截断 | 命中 1 MiB 上限，看 `truncated` 字段 |
| `plugins reload` 说没有 loader | 在 CLI 进程里跑的；loader 属于 `agent serve` 那个进程 |
| 状态一直 `pending` | 收敛没跑：还在等任务边界，或另一个 apply 正在进行 |
| `detail` 里出现 `config-schema` | `plugins.json` 的 `config` 与插件 `config_schema` 声明的形状不符；错误里有字段路径（`config.auth.token` 这种） |
| `parse plugin manifest …: config_schema …` | 插件自己的 `config_schema` 写得不对（用了不支持的关键字、`required` 里的名字没声明、嵌套超过 5 层），是**插件作者**要修的 |
| `detail` 里出现 `health: N consecutive faults` | 插件连续失败到阈值被**自动卸载**；看 `category` 判断是 timeout/trap/abi，修好包后 `reload` |
| 出现 `plugin/unload_leaked` 事件 | 卸载时仍有在途调用没收敛完；事件里的 `inflight` 是留下的调用数 |
| `signed by a revoked key` | 那把签名钥匙已被 `keyring.json` 的 `revoked` 作废。重试没用，重新用有效钥匙签包（或换回来）；那份包会被自动清出缓存 |
| `reload` 报「策略不同 / restart」 | 配置里的签名策略（含吊销）与本进程正在跑的不一致。信任集在 serve 启动时冻结，**重启 serve** 才会应用 |
| 422 之后那条变回「未缓存」 | 正常：验签失败的包会被**立即移出缓存**（那份字节不可信，不该留在部署会读的目录里）。再点「取回声明」会重新下载并再次 422——修包或换签名密钥才是出路 |
| 缓存越来越大 | `agent plugins cache list` 看占用与引用关系；`prune` 清掉清单不再引用的；`prune --max-bytes N` 把总量压到 N 以下（**只动未引用的**，压不下去会如实报告差多少） |
| 缓存里有 `INCOMPLETE` 条目 | 上一次解包被打断留下的半份目录。它既不算命中也占磁盘，`prune` 不动它（它可能属于正在进行的下载），用 `cache remove <digest>` 点名清除 |

<!-- @section: related -->
## 相关文档

- [[design-legion-plugin-system-001|Legion 插件系统设计方案（借鉴 Cordis）]] — 架构取舍、生命周期内核、后续路线
- [[reference-legion-agent-cli-001|Legion Agent CLI 命令速查]] — `agent plugins` 之外的全部命令
- [[reference-legion-agent-http-service-001|Legion Agent HTTP 服务]] — `/v1/plugins` 之外的端点与鉴权
- [[reference-legion-agent-auth-001|Legion Agent 鉴权与授权参考]] — RBAC 动作与角色，`read_plugin` / `write_plugin` 的来处
- [[reference-legion-agent-tools-001|Legion Agent 工具能力]] — 内置工具与插件贡献工具如何同处一份清单
