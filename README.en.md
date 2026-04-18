# ida-mcp-next

> Languages: **English** · [简体中文](README.md)

> The next-generation MCP server that lets AI assistants drive IDA Pro for reverse engineering directly.

[![Python](https://img.shields.io/badge/python-3.11%2B-blue)](https://www.python.org/)
[![IDA Pro](https://img.shields.io/badge/IDA%20Pro-9.2%2B-orange)](https://hex-rays.com/ida-pro)
[![MCP](https://img.shields.io/badge/MCP-2025--06--18-green)](https://modelcontextprotocol.io/)
[![License](https://img.shields.io/badge/license-MIT-lightgrey)](#license)

Inspired by [ida-pro-mcp](https://github.com/mrexodia/ida-pro-mcp), this project focuses on **IDA Pro 9.2+ headless mode** (built on the official `idalib`). It exposes IDA's analysis capabilities to MCP clients such as Claude Desktop, Kilo, and Cline through the MCP stdio protocol — no GUI required, perfect for automation and multi-agent concurrency.

---

## Table of Contents

- [What it can do (example prompts)](#what-it-can-do-example-prompts)
- [Features](#features)
- [Requirements](#requirements)
- [Installation](#installation)
- [Quick Start](#quick-start)
- [Configuring AI Clients](#configuring-ai-clients)
- [Command-Line Arguments](#command-line-arguments)
- [Environment Variables](#environment-variables)
- [Multi-Agent Concurrency (Isolated Contexts)](#multi-agent-concurrency-isolated-contexts)
- [Supported Binary Formats](#supported-binary-formats)
- [Tool List](#tool-list)
- [Configuration File](#configuration-file)
- [Troubleshooting](#troubleshooting)
- [Development](#development)
- [Acknowledgements](#acknowledgements)
- [License](#license)

> For the full capability matrix and status semantics, see [docs/能力矩阵.md](docs/能力矩阵.md) (currently Chinese-only).

---

## What it can do (example prompts)

Once the server is running and connected to your AI client, you can drive the entire reverse engineering workflow with natural language. For example:

```text
Analyze D:/samples/target.exe and tell me where the entry point is
and which suspicious APIs it calls.
```

```text
Decompile sub_401000 and rename v3 and v5 to something meaningful.
```

```text
Find every string containing "http://" or "https://" and list the
functions that reference them.
```

```text
Scan the entire D:/work/unity_game/ directory, pick the executables
worth reversing, and give me a decompilation overview.
```

```text
Set a breakpoint at 0x401234, start debugging, and once it hits,
report the values of RAX/RCX/RDX.
```

> Write operations (rename / patch / Python execution) require `--unsafe`. Debugger-related tools require `--debugger`.

---

## Features

- **Pure command-line operation** — built on the official `idalib`, no IDA GUI required, ideal for automation and CI/CD
- **Native MCP stdio** — compatible with all standard MCP clients (Claude Desktop, Kilo, Cline, ...). Protocol version `2025-06-18`
- **Multi-session support** — open and switch between multiple binaries inside a single process
- **Multi-agent concurrency** — `--isolated-contexts` keeps each agent's sessions separated, no cross-talk
- **Tiered safety** — write operations, debugger, and Python execution are off by default and must be enabled explicitly
- **Bulk directory analysis** — `analyze_directory` automatically detects PE/ELF/Mach-O and ranks candidates
- **Managed assemblies** — `.NET` / Mono / Unity assemblies are decompiled to C# via `ilspycmd`
- **Runtime isolation** — PDBs and symbol caches are written into `.runtime/`, the repository root stays clean

---

## Requirements

- **Python** 3.11+
- **IDA Pro** 9.2+ (must include `idalib`; the Free / community editions do not ship it)
- **[uv](https://docs.astral.sh/uv/)** package manager
- **Optional**: `ilspycmd` for decompiling .NET assemblies (`dotnet tool install -g ilspycmd`)

---

## Installation

### 1. Set the IDA environment variable

```powershell
# Windows
$env:IDADIR = "C:\Program Files\IDA Professional 9.2"

# Linux / macOS
export IDADIR="/opt/ida-9.2"
```

### 2. Install dependencies

```powershell
uv sync
```

### 3. Verify the installation

```powershell
uv run ida-mcp-next --help
```

---

## Quick Start

```powershell
# Start the server (waits for an MCP client to connect)
uv run ida-mcp-next

# Start the server and immediately open a sample
uv run ida-mcp-next path/to/sample.exe

# Enable write operations + Python execution (high-risk; trusted environments only)
uv run ida-mcp-next --unsafe

# Enable debugger features
uv run ida-mcp-next --debugger

# Multi-agent concurrency mode
uv run ida-mcp-next --isolated-contexts
```

---

## Configuring AI Clients

### Claude Desktop

Configuration file location:

- **Windows**: `%APPDATA%\Claude\claude_desktop_config.json`
- **macOS**: `~/Library/Application Support/Claude/claude_desktop_config.json`

**Minimal configuration**:

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

**Full configuration** (auto-load a sample at startup, enable write ops, enable debugger):

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

> **Security warning**: `--unsafe` enables byte patching and `evaluate_python` (arbitrary code execution); `--debugger` attaches a real debugger to the target process. Only enable them in trusted environments and against samples you own.

### Kilo / Cline (TOML)

```toml
[mcpServers.ida-mcp-next]
command = "uv"
args = ["run", "--directory", "D:/work/ida-mcp-next", "--no-sync", "ida-mcp-next"]

[mcpServers.ida-mcp-next.env]
IDADIR = "C:\\Program Files\\IDA Professional 9.2"
```

---

## Command-Line Arguments

| Argument              | Description                                                          |
| --------------------- | -------------------------------------------------------------------- |
| `<input_path>`        | Optional positional argument: a binary to open at startup            |
| `--config <path>`     | Configuration file path, defaults to `setting.toml`                  |
| `--unsafe`            | Enable write operations, symbol renaming, Python execution, etc.    |
| `--debugger`          | Enable debugger-related tools                                        |
| `--isolated-contexts` | Enable multi-agent isolated-context mode                             |
| `--profile <path>`    | Path to a tool-allowlist profile file                                |

---

## Environment Variables

| Variable                           | Required | Description                                                              |
| ---------------------------------- | :------: | ------------------------------------------------------------------------ |
| `IDADIR`                           |    ✓     | IDA Pro install directory; must point at an installation that has `idalib` |
| `IDA_MCP_NEXT_TEST_BINARY`         |          | Override the default ELF fixture used by `tests/integration`             |
| `IDA_MCP_NEXT_REAL_NATIVE_BINARY`  |          | Override the path to the real native sample used by regression tests     |
| `IDA_MCP_NEXT_REAL_MANAGED_BINARY` |          | Override the path to the real managed (.NET / Mono) sample               |
| `IDA_MCP_NEXT_ILSPYCMD`            |          | Custom `ilspycmd` executable path; falls back to `PATH` when unset       |

---

## Multi-Agent Concurrency (Isolated Contexts)

The server supports two context modes:

- **Shared default context** (default): single-agent / serial analysis; all calls share one session pool
- **Isolated context mode** (`--isolated-contexts`): every agent / workflow uses a stable `context_id` to keep its sessions invisible to other agents

### How to enable

```powershell
# Command line
uv run ida-mcp-next --isolated-contexts
```

Or in `setting.toml`:

```toml
[feature_gates]
isolated_contexts = true
```

### Usage rules

Once enabled, every session-related tool **must explicitly include `context_id`**:

```text
open_binary(path="a.exe", context_id="agent-1")
list_binaries(context_id="agent-1")        # only sees agent-1's own sessions
switch_binary(binary_id="...", context_id="agent-1")
```

If `context_id` is missing the server **fails fast** with an error — it never silently falls back to the shared context.

> Full isolation rules and resource-URI behavior are documented in [docs/能力矩阵.md#上下文与会话隔离矩阵](docs/能力矩阵.md) (Chinese).

---

## Supported Binary Formats

| Category           | Format                       | Decompiler back-end | Notes                |
| ------------------ | ---------------------------- | ------------------- | -------------------- |
| Native executable  | PE (`.exe`/`.dll`/`.sys`)    | IDA Hex-Rays        | Windows binaries     |
| Native executable  | ELF (`.elf`/`.so`)           | IDA Hex-Rays        | Linux / Android      |
| Native executable  | Mach-O                       | IDA Hex-Rays        | macOS / iOS          |
| Native executable  | Raw `.bin` and similar       | IDA Hex-Rays        | Embedded / firmware  |
| Managed assembly   | .NET / Mono / Unity (`.dll`) | `ilspycmd` (C#)     | Requires `ilspycmd`  |

`analyze_directory` auto-detects the formats above by file magic and skips text / resource files.

---

## Tool List

### Session management

| Tool                | Description                                                       |
| ------------------- | ----------------------------------------------------------------- |
| `open_binary`       | Open a binary file and create a session                           |
| `close_binary`      | Close a specific session                                          |
| `list_binaries`     | List currently visible sessions                                   |
| `switch_binary`     | Switch the active session                                         |
| `save_binary`       | Save the IDB database                                             |
| `analyze_directory` | Bulk-scan a directory, detect analyzable binaries, rank them      |

### Information queries

| Tool                   | Description                                                                |
| ---------------------- | -------------------------------------------------------------------------- |
| `survey_binary`        | File overview: architecture, segments, entry point                         |
| `list_functions`       | List all functions                                                         |
| `get_function`         | Get the details of a single function                                       |
| `decompile_function`   | Decompile a function (pseudocode for native, C# for .NET)                  |
| `disassemble_function` | Disassemble a function                                                     |
| `list_imports`         | Import table                                                               |
| `list_globals`         | Global variables                                                           |
| `list_strings`         | String list                                                                |
| `get_xrefs_to`         | Cross-references                                                           |
| `read_struct`          | Structure definitions                                                      |

### Search

| Tool           | Description           |
| -------------- | --------------------- |
| `find_strings` | Search strings        |
| `find_bytes`   | Search byte sequences |
| `search_regex` | Regex search          |

### Write operations (require `--unsafe`, high-risk)

| Tool              | Description                                                                |
| ----------------- | -------------------------------------------------------------------------- |
| `rename_symbols`  | Rename functions / variables                                               |
| `set_comments`    | Add comments                                                               |
| `patch_bytes`     | Modify bytes                                                               |
| `declare_types`   | Declare data types                                                         |
| `evaluate_python` | **Arbitrary Python code execution** — equivalent to IDA command-line privileges |

### Debugger (requires `--debugger`)

| Tool                | Description       |
| ------------------- | ----------------- |
| `debug_start`       | Start debugging   |
| `debug_step_into`   | Step into         |
| `debug_step_over`   | Step over         |
| `debug_continue`    | Continue          |
| `debug_registers`   | Inspect registers |
| `debug_read_memory` | Read memory       |

> The full schema for every tool is available at runtime via the MCP resources `ida://docs/tools` and `ida://docs/tool/{name}`.

---

## Configuration File

`setting.toml` lives at the repository root:

```toml
[logging]
level = "INFO"
directory = "logs"

[server]
protocol_version = "2025-06-18"
server_name = "ida-mcp-next"
server_version = "0.2.0"
default_input_path = ""          # Sample to auto-open at startup (CLI argument takes priority)

[feature_gates]
allow_unsafe = false             # Equivalent to --unsafe
allow_debugger = false           # Equivalent to --debugger
isolated_contexts = false        # Equivalent to --isolated-contexts

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

Runtime side effects (PDB / symbol caches, placeholder files) are all written into `.runtime/` inside the project, so the repository root is no longer polluted. `.runtime/` is added to `.gitignore` by default.

---

## Troubleshooting

<details>
<summary><b>Startup error: cannot find idalib / IDADIR not set</b></summary>

Verify that:

1. `IDADIR` points to your IDA Pro install directory (no trailing slash)
2. The directory contains `idalib/python/idapro` (Free / community editions do not ship `idalib`)
3. After setting it in PowerShell you may need to restart the terminal: `$env:IDADIR = "C:\Program Files\IDA Professional 9.2"`

</details>

<details>
<summary><b>Failed to decompile a .NET assembly: ilspycmd not found</b></summary>

Install ilspycmd:

```powershell
dotnet tool install -g ilspycmd
```

Or specify an explicit path:

```powershell
$env:IDA_MCP_NEXT_ILSPYCMD = "C:\Tools\ilspycmd.exe"
```

</details>

<details>
<summary><b>Calling a tool returns "unsafe operations are disabled"</b></summary>

That tool falls into the write / high-risk category. Pass `--unsafe` at startup, or set `feature_gates.allow_unsafe = true` in `setting.toml`.

</details>

<details>
<summary><b>Multi-agent mode reports "context_id is required"</b></summary>

Once `--isolated-contexts` is on, every session-related call must include `context_id` explicitly. See the [Multi-Agent Concurrency](#multi-agent-concurrency-isolated-contexts) section for details.

</details>

<details>
<summary><b>The `.runtime/` directory is using too much disk space</b></summary>

`.runtime/symbol-cache/` stores PDB and symbol caches and is safe to wipe:

```powershell
Remove-Item -Recurse -Force .runtime/symbol-cache
```

</details>

---

## Development

```powershell
# Type check
uv run basedpyright

# Run unit tests
uv run python -m unittest discover -s tests/unit

# Run integration tests (requires IDA + a real sample)
uv run ida-mcp-next-test path/to/sample.exe
```

---

## Acknowledgements

This project is inspired by [ida-pro-mcp](https://github.com/mrexodia/ida-pro-mcp). Many thanks to the original author for the pioneering work.

---

## License

MIT
