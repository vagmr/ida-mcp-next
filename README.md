# ida-mcp-next

> Languages: **简体中文** · [English](README.en.md)

> 让 AI 助手直接操控 IDA Pro 进行逆向分析的下一代 MCP 服务器。

[![Python](https://img.shields.io/badge/python-3.11%2B-blue)](https://www.python.org/)
[![IDA Pro](https://img.shields.io/badge/IDA%20Pro-9.2%2B-orange)](https://hex-rays.com/ida-pro)
[![MCP](https://img.shields.io/badge/MCP-2025--06--18-green)](https://modelcontextprotocol.io/)
[![License](https://img.shields.io/badge/license-MIT-lightgrey)](#license)

本项目灵感来自 [ida-pro-mcp](https://github.com/mrexodia/ida-pro-mcp)，专注于 **IDA Pro 9.2+ headless 模式**（基于官方 `idalib`），通过 MCP stdio 协议让 Claude、Kilo、Cline 等 AI 客户端能直接驱动 IDA 完成逆向分析任务，无需启动 GUI、适合自动化与多 agent 并发场景。

---

## 目录

- [它能干什么（示例 Prompt）](#它能干什么示例-prompt)
- [特性](#特性)
- [环境要求](#环境要求)
- [安装](#安装)
- [快速开始](#快速开始)
- [配置 AI 客户端](#配置-ai-客户端)
- [命令行参数](#命令行参数)
- [环境变量](#环境变量)
- [多 agent 并发（隔离上下文）](#多-agent-并发隔离上下文)
- [支持的二进制格式](#支持的二进制格式)
- [工具列表](#工具列表)
- [配置文件](#配置文件)
- [故障排查](#故障排查)
- [开发](#开发)
- [致谢](#致谢)
- [License](#license)

> 详细能力边界与状态语义，请参阅 [docs/能力矩阵.md](docs/能力矩阵.md)。

---

## 它能干什么（示例 Prompt）

服务启动并接入 AI 客户端后，可以直接用自然语言驱动整个逆向流程，例如：

```text
分析 D:/samples/target.exe，告诉我入口点在哪、调用了哪些可疑 API。
```

```text
反编译 sub_401000 函数，帮我把里面的 v3、v5 重命名成有意义的名字。
```

```text
搜索所有包含 "http://" 或 "https://" 的字符串，并列出引用了它们的函数。
```

```text
分析 D:/work/unity_game/ 整个目录，挑出最值得逆向的可执行文件并给出反编译概览。
```

```text
在 0x401234 处下断点，启动调试，等命中后把 RAX/RCX/RDX 的值告诉我。
```

> 写操作（重命名/打补丁/执行 Python）需要 `--unsafe` 启用；调试相关工具需要 `--debugger` 启用。

---

## 特性

- **纯命令行运行** — 基于官方 `idalib`，无需启动 IDA GUI，适合自动化与 CI/CD
- **原生 MCP stdio** — 兼容 Claude Desktop、Kilo、Cline 等所有标准 MCP 客户端，协议版本 `2025-06-18`
- **多会话支持** — 单进程内可同时打开多个二进制文件，自由切换
- **多 agent 并发** — 通过 `--isolated-contexts` 隔离不同 agent 的会话，避免串台
- **安全分级** — 写操作 / 调试器 / Python 执行默认关闭，需显式启用
- **目录批量分析** — `analyze_directory` 自动识别 PE/ELF/Mach-O 并打分排序
- **托管程序集** — `.NET` / Mono / Unity 程序集走 `ilspycmd` 反编译为 C#
- **运行时隔离** — PDB / 符号缓存统一写入 `.runtime/`，不污染仓库根目录

---

## 环境要求

- **Python** 3.11+
- **IDA Pro** 9.2+（必须包含 `idalib`，社区版/Free 版不支持）
- **[uv](https://docs.astral.sh/uv/)** 包管理器
- **可选**：`ilspycmd`（用于反编译 .NET 程序集，`dotnet tool install -g ilspycmd`）

---

## 安装

### 1. 设置 IDA 环境变量

```powershell
# Windows
$env:IDADIR = "C:\Program Files\IDA Professional 9.2"

# Linux / macOS
export IDADIR="/opt/ida-9.2"
```

### 2. 安装依赖

```powershell
uv sync
```

### 3. 验证安装

```powershell
uv run ida-mcp-next --help
```

---

## 快速开始

```powershell
# 启动服务（等待 MCP 客户端连接）
uv run ida-mcp-next

# 启动时直接打开样本
uv run ida-mcp-next path/to/sample.exe

# 启用写操作 + Python 执行（高危，仅可信环境使用）
uv run ida-mcp-next --unsafe

# 启用调试器功能
uv run ida-mcp-next --debugger

# 多 agent 并发模式
uv run ida-mcp-next --isolated-contexts
```

---

## 配置 AI 客户端

### Claude Desktop

配置文件位置：

- **Windows**：`%APPDATA%\Claude\claude_desktop_config.json`
- **macOS**：`~/Library/Application Support/Claude/claude_desktop_config.json`

**基础配置**（最小可用）：

```json
{
  "mcpServers": {
    "ida-mcp-next": {
      "command": "uv",
      "args": [
        "run",
        "--directory",
        "D:/work/ida-mcp-next",
        "--no-sync",
        "ida-mcp-next"
      ],
      "env": {
        "IDADIR": "C:\\Program Files\\IDA Professional 9.2"
      }
    }
  }
}
```

**完整配置**（启动时自动加载样本 + 启用写操作 + 启用调试器）：

```json
{
  "mcpServers": {
    "ida-mcp-next": {
      "command": "uv",
      "args": [
        "run",
        "--directory",
        "D:/work/ida-mcp-next",
        "--no-sync",
        "ida-mcp-next",
        "D:/samples/target.exe",
        "--unsafe",
        "--debugger"
      ],
      "env": {
        "IDADIR": "C:\\Program Files\\IDA Professional 9.2"
      }
    }
  }
}
```

> **安全提醒**：`--unsafe` 会开放字节修改与 `evaluate_python` 任意代码执行；`--debugger` 会启动真实调试器附加到目标进程。请仅在受信任环境对自己的样本启用。

### Kilo / Cline (TOML)

```toml
[mcpServers.ida-mcp-next]
command = "uv"
args = ["run", "--directory", "D:/work/ida-mcp-next", "--no-sync", "ida-mcp-next"]

[mcpServers.ida-mcp-next.env]
IDADIR = "C:\\Program Files\\IDA Professional 9.2"
```

---

## 命令行参数

| 参数                  | 说明                                          |
| --------------------- | --------------------------------------------- |
| `<input_path>`        | 启动时自动打开的二进制文件（可选位置参数）    |
| `--config <path>`     | 配置文件路径，默认 `setting.toml`             |
| `--unsafe`            | 启用写操作、符号重命名、Python 执行等高危工具 |
| `--debugger`          | 启用调试器相关工具                            |
| `--isolated-contexts` | 启用多 agent 隔离上下文模式                   |
| `--profile <path>`    | 工具白名单 profile 文件路径                   |

---

## 环境变量

| 变量                               | 必需 | 说明                                            |
| ---------------------------------- | :--: | ----------------------------------------------- |
| `IDADIR`                           |  ✓   | IDA Pro 安装目录，必须指向包含 `idalib` 的版本  |
| `IDA_MCP_NEXT_TEST_BINARY`         |      | 覆盖 `tests/integration` 默认 ELF fixture 路径  |
| `IDA_MCP_NEXT_REAL_NATIVE_BINARY`  |      | 覆盖回归测试中真实 native 样本路径              |
| `IDA_MCP_NEXT_REAL_MANAGED_BINARY` |      | 覆盖回归测试中真实托管样本（.NET/Mono）路径     |
| `IDA_MCP_NEXT_ILSPYCMD`            |      | 自定义 `ilspycmd` 可执行路径，未设置时走 `PATH` |

---

## 多 agent 并发（隔离上下文）

服务支持两种上下文模式：

- **共享默认上下文**（默认）：单 agent / 串行分析场景，所有调用共享同一会话池
- **隔离上下文模式**（`--isolated-contexts`）：每个 agent / workflow 通过稳定的 `context_id` 隔离自己的会话，互不可见

### 启用方式

```powershell
# 命令行
uv run ida-mcp-next --isolated-contexts
```

或在 `setting.toml` 中：

```toml
[feature_gates]
isolated_contexts = true
```

### 使用约束

开启后所有会话相关工具都**必须显式提供 `context_id`**：

```text
open_binary(path="a.exe", context_id="agent-1")
list_binaries(context_id="agent-1")        # 只看得到 agent-1 自己的会话
switch_binary(binary_id="...", context_id="agent-1")
```

缺失 `context_id` 时服务会 **fail-fast** 返回错误，不会回退到共享上下文。

> 完整的隔离规则与资源 URI 行为请参阅 [docs/能力矩阵.md#上下文与会话隔离矩阵](docs/能力矩阵.md)。

---

## 支持的二进制格式

| 类别       | 格式                         | 反编译后端      | 说明                 |
| ---------- | ---------------------------- | --------------- | -------------------- |
| 原生可执行 | PE (`.exe`/`.dll`/`.sys`)    | IDA Hex-Rays    | Windows 二进制       |
| 原生可执行 | ELF (`.elf`/`.so`)           | IDA Hex-Rays    | Linux/Android 二进制 |
| 原生可执行 | Mach-O                       | IDA Hex-Rays    | macOS/iOS 二进制     |
| 原生可执行 | `.bin` 等裸二进制            | IDA Hex-Rays    | 嵌入式 / 固件        |
| 托管程序集 | .NET / Mono / Unity (`.dll`) | `ilspycmd` (C#) | 需 `ilspycmd` 可用   |

`analyze_directory` 工具会自动按文件魔数识别上述格式，跳过文本/资源文件。

---

## 工具列表

### 会话管理

| 工具                | 说明                                       |
| ------------------- | ------------------------------------------ |
| `open_binary`       | 打开二进制文件并创建会话                   |
| `close_binary`      | 关闭指定会话                               |
| `list_binaries`     | 列出当前可见的会话                         |
| `switch_binary`     | 切换当前活动会话                           |
| `save_binary`       | 保存 IDB 数据库                            |
| `analyze_directory` | 批量扫描目录、识别可分析的二进制并打分排序 |

### 信息查询

| 工具                   | 说明                                          |
| ---------------------- | --------------------------------------------- |
| `survey_binary`        | 文件概览：架构、段、入口点                    |
| `list_functions`       | 列出所有函数                                  |
| `get_function`         | 获取单个函数详情                              |
| `decompile_function`   | 反编译函数（native 返回伪代码，.NET 返回 C#） |
| `disassemble_function` | 反汇编函数                                    |
| `list_imports`         | 导入表                                        |
| `list_globals`         | 全局变量                                      |
| `list_strings`         | 字符串列表                                    |
| `get_xrefs_to`         | 交叉引用                                      |
| `read_struct`          | 结构体定义                                    |

### 搜索功能

| 工具           | 说明         |
| -------------- | ------------ |
| `find_strings` | 搜索字符串   |
| `find_bytes`   | 搜索字节序列 |
| `search_regex` | 正则搜索     |

### 写操作（需 `--unsafe`，高危）

| 工具              | 说明                                          |
| ----------------- | --------------------------------------------- |
| `rename_symbols`  | 重命名函数/变量                               |
| `set_comments`    | 添加注释                                      |
| `patch_bytes`     | 修改字节                                      |
| `declare_types`   | 定义数据类型                                  |
| `evaluate_python` | **任意 Python 代码执行**，等同 IDA 命令行权限 |

### 调试器（需 `--debugger`）

| 工具                | 说明       |
| ------------------- | ---------- |
| `debug_start`       | 启动调试   |
| `debug_step_into`   | 单步进入   |
| `debug_step_over`   | 单步跳过   |
| `debug_continue`    | 继续执行   |
| `debug_registers`   | 查看寄存器 |
| `debug_read_memory` | 读取内存   |

> 工具完整 schema 可在运行时通过 MCP 资源 `ida://docs/tools` 与 `ida://docs/tool/{name}` 查询。

---

## 配置文件

项目根目录的 `setting.toml`：

```toml
[logging]
level = "INFO"
directory = "logs"

[server]
protocol_version = "2025-06-18"
server_name = "ida-mcp-next"
server_version = "0.2.0"
default_input_path = ""          # 启动时自动打开的样本（命令行参数优先）

[feature_gates]
allow_unsafe = false             # 等同 --unsafe
allow_debugger = false           # 等同 --debugger
isolated_contexts = false        # 等同 --isolated-contexts

[runtime_workspace]
directory = ".runtime"
symbol_cache_directory = ".runtime/symbol-cache"

[limits]
default_page_size = 100
max_page_size = 1000
max_search_hits = 1000
max_callgraph_depth = 4

[directory_analysis]
recursive = true
max_candidates = 20
max_deep_analysis = 5
prefer_entry_binary = true
prefer_user_code = true
scoring_profile = "default"
include_extensions = [".exe", ".dll", ".sys", ".elf", ".so", ".bin"]
exclude_patterns = ["*.i64", "*.idb", "*.nam", "*.til"]
```

运行时副作用（PDB/符号缓存、占位文件）会统一写入项目内的 `.runtime/`，不会再散落到仓库根目录；`.runtime/` 已默认加入 `.gitignore`。

---

## 故障排查

<details>
<summary><b>启动报错：找不到 idalib / IDADIR 未设置</b></summary>

确认：

1. `IDADIR` 环境变量已正确指向 IDA Pro 安装目录（不要带尾部斜杠）
2. 该目录下存在 `idalib/python/idapro` 子目录（Free 版/社区版没有 `idalib`）
3. PowerShell 环境下设置后需重启终端：`$env:IDADIR = "C:\Program Files\IDA Professional 9.2"`

</details>

<details>
<summary><b>反编译 .NET 程序集失败：ilspycmd 未找到</b></summary>

安装 ilspycmd：

```powershell
dotnet tool install -g ilspycmd
```

或显式指定路径：

```powershell
$env:IDA_MCP_NEXT_ILSPYCMD = "C:\Tools\ilspycmd.exe"
```

</details>

<details>
<summary><b>调用工具返回 "unsafe operations are disabled"</b></summary>

该工具属于写操作 / 高危类别，需要在启动时加 `--unsafe`，或在 `setting.toml` 中设置 `feature_gates.allow_unsafe = true`。

</details>

<details>
<summary><b>多 agent 模式下报错 "context_id is required"</b></summary>

开启 `--isolated-contexts` 后，所有会话相关调用必须显式传入 `context_id`，详见 [多 agent 并发](#多-agent-并发隔离上下文) 章节。

</details>

<details>
<summary><b>`.runtime/` 目录占用空间过大</b></summary>

`.runtime/symbol-cache/` 存放 PDB 与符号缓存，可以安全清空：

```powershell
Remove-Item -Recurse -Force .runtime/symbol-cache
```

</details>

---

## 开发

```powershell
# 类型检查
uv run basedpyright

# 运行单元测试
uv run python -m unittest discover -s tests/unit

# 运行集成测试（需要 IDA + 真实样本）
uv run ida-mcp-next-test path/to/sample.exe
```

---

## 致谢

本项目受到 [ida-pro-mcp](https://github.com/mrexodia/ida-pro-mcp) 的启发，感谢作者的开拓性工作。

---

## License

MIT
