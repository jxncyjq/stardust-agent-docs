---
id: "reference-legion-agent-plugins-001"
title: "Legion Agent WASM 插件参考手册"
aliases: ["插件手册", "plugin manual", "agent plugins", "WASM 插件", "插件开发"]
type: "reference"
category: "agents/reference"
tags: ["agent", "plugin", "wasm", "wazero", "abi", "signing", "capability", "cli"]
version: "1.2.0"
created: "2026-08-28"
updated: "2026-08-28"
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
| 0 | `abi.OpManifest` | 忽略 | 自述 JSON：`{"name":…,"version":…,"provides":[工具名…]}` |
| 1 | `abi.OpCallTool` | `{"call_id":…,"tool":…,"arguments":{…}}` | `domain.ToolResult`：`{"call_id":…,"success":bool,"output":…}` 或 `{"success":false,"error":…}` |
| 其它 | — | — | **不要 trap**，返回一个可读的小 JSON 错误体 |

**交叉校验**：激活时宿主拿 op 0 的 `provides` 与部署侧声称该插件提供的工具集比对，`provides` 少一个就拒绝挂载。工具失败要用 `{"success":false,"error":…}` 表达，不要 trap——trap 会让整个模块死掉。

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
| `requires` | []string | 本插件通过 `call_tool` 调用的**别的插件的工具名**；不得为空串、不得重复、不得写自己贡献的工具 |

`requires` 与 `capabilities` 不同类：能力在加载期检查，缺了直接拒载；`requires` 未满足（提供方不在）是**可恢复的 suspended 态**，提供方回来即可恢复。

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
{ "keys": [ { "id": "demo-key", "algorithm": "ed25519", "public_key": "<base64>" } ] }
```

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
        "allowed_paths": []
      },
      "tools": [ { "name": "hello_echo", "risk_level": "low", "sensitive": false } ],
      "config": { "任意": "交给 config_get 的原文" }
    }
  ]
}
```

`tools[]` 是部署侧的**接受**清单：`name` 必须是插件声明过的工具；`risk_level` / `sensitive` 只能把插件自己的声明**收紧**，不能放松（`sensitive` 是指针，缺省 = 用插件的声明）。

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
| `agent plugins grant <name>` | 授权一条已登记的 entry | `--capabilities`（必须**恰好**等于插件声明的集合）、`--allowed-hosts` / `--allowed-paths`（可取声明集合的子集）、`--config` |
| `agent plugins deny <name>` | 撤销授权（保留 `grant` 键 → `disabled`） | `--config` |
| `agent plugins keygen` | 生成 Ed25519 密钥对 | `--key-id`、`--private-key`（不覆盖已存在文件） |
| `agent plugins sign <dir>` | 对包目录的 `plugin.json` 签名 | `--private-key` |

**七个子命令分两组，除了名词以外不共享任何东西**：

- `status` / `reload` 是**这个进程**的视图——两者都读 serve 装配的那个 loader。**跨进程没有视图**：在一个只跑 CLI 的进程里执行，会明确报告「本进程没有 loader」，而不是回一个像「没有插件」的空答案。
- `keygen` / `sign` / `install` / `grant` / `deny` 不碰任何 loader、不启动服务。`keygen` 与 `sign` 连配置都不读——它们**生产**验签所消费的东西，与验签一起发布是刻意的：一个能验签却造不出签名的部署，剩下的唯一选择就是把验签关掉，而那正是验签要防的事。`install` / `grant` / `deny` 读插件配置（复用与 serve 相同的缓存、下载上限、来源策略与信任集）并写清单，但它们做的都是**磁盘上的登记或决定，不是启动**：任何一个都要等 `reload` 才会到达运行中的进程。

`install --grant` 必须**恰好**列出插件声明的能力全集（部分授权写出的 entry 永远加载不了）；插件声明了非空 `allowed_hosts` / `allowed_paths` 时，`--grant` 直接拒绝 `http` / `fs`——主机与路径白名单是 `grant` 命令的活。

`install` 从不重载运行中的服务，装完记得 `reload`。

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

`GET /v1/plugins` 的一行（`PluginView`）字段：`name` `version` `state` `detail?` `tools[]` `declared_capabilities/hosts/paths` `declared_unresolved` `declared_unresolved_reason?` `declared_error?` `granted_capabilities/hosts/paths`。

`declared_unresolved_reason` 的三个取值决定界面给不给「取回声明」：

| reason | 含义 | 可取回？ |
|---|---|---|
| `not_cached` | 远程包还没下载 | ✅ 唯一可取回的 |
| `no_cache_configured` | 部署没配 `plugins.cache` | ❌ 改配置 + 重启 serve |
| `load_failed` | 包在但加载不了（坏 wasm、坏签名、缺文件） | ❌ 再取回读到的是同样的坏字节 |

### 7.3 GUI

设置 → 插件：每行一条 entry，未解析的行给次要样式的「取回声明」，「授权」在声明可见之前一直禁用——**不能对看不见的清单点同意**。取回成功后常驻「已取回并缓存该插件包（未授权，可随时撤销）」，并把授权对话框切换到刚取回的声明。取回 / 授权 / 收敛进行中，Esc、标题栏 X、点背景、切 tab 四条路径全部被拦——一个按下去必然无效的取消按钮就是在骗人。

GUI 与 CLI 走**同一批校验函数**（`internal/plugin/consent`），不是第二条各说各话的授权路径。

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

`reload` 有一条要记住的限制：**信任集在 serve 装配时冻结**，运行中换不了。收紧了 keyring 再 `reload`，得到的是**新清单 + 旧信任集**；策略变更必须重启 serve。

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
| `guest exports no linear memory` | 没导出 `memory`（构建目标不对，或不是 cdylib） |
| 装完了跑不起来 | 正常：`install` 不授权，去 `grant` |
| `grant` 报缓存未命中且没配 cache | `plugins.cache` 缺失，远程包无处落盘 |
| HTTP 调用返回 `DENIED` | 目标主机不在 `allowed_hosts`，或重定向跳出了白名单 |
| 响应/文件内容被截断 | 命中 1 MiB 上限，看 `truncated` 字段 |
| `plugins reload` 说没有 loader | 在 CLI 进程里跑的；loader 属于 `agent serve` 那个进程 |
| 状态一直 `pending` | 收敛没跑：还在等任务边界，或另一个 apply 正在进行 |
| `detail` 里出现 `health: N consecutive faults` | 插件连续失败到阈值被**自动卸载**；看 `category` 判断是 timeout/trap/abi，修好包后 `reload` |
| 出现 `plugin/unload_leaked` 事件 | 卸载时仍有在途调用没收敛完；事件里的 `inflight` 是留下的调用数 |
| 422 之后那条一直是 `load_failed` | 不可信的包留在缓存里了；仓内暂无 eviction API，需手工清 `cache/sha256/<digest>` |

<!-- @section: related -->
## 相关文档

- [[design-legion-plugin-system-001|Legion 插件系统设计方案（借鉴 Cordis）]] — 架构取舍、生命周期内核、后续路线
- [[reference-legion-agent-cli-001|Legion Agent CLI 命令速查]] — `agent plugins` 之外的全部命令
- [[reference-legion-agent-http-service-001|Legion Agent HTTP 服务]] — `/v1/plugins` 之外的端点与鉴权
- [[reference-legion-agent-auth-001|Legion Agent 鉴权与授权参考]] — RBAC 动作与角色，`read_plugin` / `write_plugin` 的来处
- [[reference-legion-agent-tools-001|Legion Agent 工具能力]] — 内置工具与插件贡献工具如何同处一份清单
