---
name: sysplant
description: Generate Windows direct-syscall (syswhispers/hell-gate/canterlot-style) injection code in C, C++, NIM or Rust to bypass ntdll.dll userland hooks (EDR/AV), with optional symbol scramble and egg-hunter stubs
category: post-exploitation
tags: [windows, syscall, direct-syscall, hells-gate, canterlot, syswhispers, syswhispers3, edr-evasion, ntdll, process-injection, shellcode, donut, egg-hunter, unhook, anti-av, offensive]
tech_stack: [windows, c, cpp, nim, rust, mingw]
cwe_ids: [CWE-269, CWE-522, CWE-693, CWE-94]
chains_with: [T1055, T1055.001, T1562.001, T1027, T1622, windows-postexploit]
prerequisites: [T1068]
version: "0.5.2"
author: "x42en"
severity_boost:
  windows-postexploit: "SysPlant direct-syscall loader defeats EDR ntdll userland hooks used by winhook stealth_check/unhook_ntdll targets"
---

# SysPlant — Windows Direct-Syscall Code Generator

SysPlant generates **self-contained** source files (C / C++ / NIM / Rust) that resolve and call Windows `Nt*` syscalls **without going through `ntdll.dll` exports** at runtime. This defeats userland EDR/AV hooks on `ntdll.dll` (e.g. hooked `NtWriteVirtualMemory`, `NtCreateThreadEx`) that break normal `LoadLibrary`/`GetProcAddress`-based shellcode loaders.

Use it to produce the syscall plumbing for process injection, shellcode execution, or Donut-style PE loading on a Windows target that has an EDR hooking the ntdll export table.

> **Scope:** generates *code* to be cross-compiled (MinGW / Nim / Rust) and run on Windows. It does not run shellcode itself. Only use on systems you are authorized to test.

## When to reach for this

- `winhook stealth_check` / `unhook_ntdll` report ntdll hooks that will corrupt a normal injection path, and you need a direct-syscall fallback loader.
- You already have shellcode / a PE and need a loader that calls `NtWriteVirtualMemory`, `NtCreateThreadEx`, `NtAllocateVirtualMemory`, `NtOpenProcess`, etc. directly.
- You need to match an existing payload's syscall surface (e.g. Donut's 14 functions).

## Install (generator side)

```bash
# CLI + Python lib
pip install sysplant
# Or with the MCP server extra:
pip install "sysplant[mcp]"
# Verify
sysplant list --help
```

> **Requires `sysplant >= 0.5.2`.** The MCP bridge (`sysplant-mcp`) moved into the package in 0.5.1 and the `mcp[cli]` dependency is pinned to `>=1.0,<2` in 0.5.2 — do not install `mcp 2.x` alongside it: the server uses the v1 `FastMCP` API (`from mcp.server.fastmcp import FastMCP`), which was removed/renamed (`MCPServer`) in mcp 2.x and hard-crashes at import.

## CLI usage

```bash
# C code, x64, Canterlot's Gate iterator, common preset, symbol scramble, output file
sysplant generate -c -x64 -x -o syscalls.c canterlot

# Rust, x64, Hell's Gate, donut preset (the 14 syscalls Donut needs)
sysplant generate -rust -x64 -p donut -o syscalls.rs hell

# NIM (default lang), x64, explicit function list, indirect stub method
sysplant generate -nim -x64 -f "NtWriteVirtualMemory,NtCreateThreadEx,NtAllocateVirtualMemory" -o syscalls.nim canterlot

# Scan a directory to discover which Nt* functions your code references
sysplant list ./example
```

### Key flags

| Flag | Meaning |
|------|---------|
| `-x86` / `-x64` / `-wow` | Target arch (x64 default). `-wow` = WoW64 (32-bit on 64-bit) |
| `-nim` / `-c` / `-rust` | Output language (NIM default) |
| `-p {common,donut,all}` | Preset syscall set — `common` (default, ~31), `donut` (14, for Donut PE loader), `all` (~300+) |
| `-f A,B,C` | Explicit comma-separated `Nt*` functions (overrides `-p`) |
| `-x` | Scramble internal symbol names to evade static signature matching |
| `-o PATH` | Output file path |
| **last positional** | Iterator: `hell`, `halo`, `tartarus`, `freshy`, `syswhispers`, `syswhispers3`, `canterlot` |

### Iterator (syscall-resolution) selection

| Iterator | Strategy | Pick when |
|----------|----------|-----------|
| `canterlot` **(default)** | PE exception directory (`RUNTIME_FUNCTION`) — no ntdll stub reads | Best default; robust against stub hooking |
| `hell` | Read syscall # from first opcodes of ntdll stub | Classic, minimal footprint |
| `halo` | Hell + walk neighbouring stub if hooked | Stub is hooked |
| `tartarus` | Halo + deeper neighbour search | More aggressive hooking |
| `freshy` | Exception directory by `Nt*` name, sort by address | No ntdll bytes needed |
| `syswhispers` | `Zw*` exports sorted by address | SysWhispers2 style |
| `syswhispers3` | SysWhispers2 + static random-jump offset | Need random indirect jumps |

### Caller stub method

| Method | How the call is made | Trade-off |
|--------|----------------------|-----------|
| `direct` (default) | `syscall` instruction inline | Simplest; AV may flag raw syscall |
| `indirect` | Jump into real ntdll stub | Return addr inside ntdll (evades call-stack checks) but stub may be hooked |
| `random` | Jump to a random ntdll syscall stub | Variance; needs 1 stub addr |
| `egg_hunter` | Replace `syscall;ret` with a random 8-byte marker; `SPT_SanitizeSyscalls()` patches it at runtime | Avoids static `0F 05` signatures |

## MCP server (AI-assisted generation)

SysPlant ships an MCP server (`sysplant-mcp` console script) exposing the generator as tools. Prefer this when you want the agent to pick iterator/method/functions automatically:

| Tool | Purpose |
|------|---------|
| `generate_syscalls` | Main tool — returns a complete source file (C/C++/NIM/Rust) for the given language, arch, iterator, method, preset/functions, scramble |
| `scan_ntfunctions` | Scan a file/dir for referenced `Nt*`/`Zw*` functions → drives the function list |
| `list_supported_syscalls` | All ~300+ known `Nt*` functions |
| `list_common_syscalls` | The 31 most common (injection primitives) |
| `list_donut_syscalls` | The 14 Donut PE-loader functions |
| `get_function_prototype` | Full prototype + params for one `Nt*` function |
| `list_iterators` / `list_methods` / `list_languages` | Enumerate options |

```bash
# stdio (default)
sysplant-mcp
# HTTP for web clients
sysplant-mcp --transport streamable-http --host 0.0.0.0 --port 8080
```

## Cross-compile the generated code (Linux → Windows)

Pick the command matching the language you generated:

```bash
# C (MinGW-w64)
x86_64-w64-mingw32-gcc -Wall -s -static -masm=intel -o loader.exe loader.c

# NIM
nim c -d:release -d:danger -d:strip --opt:size -d:mingw --cpu:amd64 loader.nim

# Rust
cargo build --release --target x86_64-pc-windows-gnu
```

## Typical workflow (agent-driven)

1. **Discover the syscall surface** — `sysplant list ./payload_dir` or MCP `scan_ntfunctions(path)` to learn which `Nt*` the injection code needs.
2. **Generate** — MCP `generate_syscalls(language, arch="x64", iterator="canterlot", method="direct", functions=[...], scramble=True)` (or CLI `sysplant generate ...`).
3. **Cross-compile** with the language-appropriate command above.
4. **Deploy** the resulting `.exe`/`.dll` to the Windows target and invoke the generated entry point.
5. If the EDR is hooking ntdll heavily, retry with `iterator` = `halo`/`tartarus` and `method` = `random` or `egg_hunter`, plus `scramble=True`.

## EDR/AV notes

- `scramble=True` randomizes the ~23 internal `SPT_*` symbol names to defeat static signature matching on the generated code.
- `method=egg_hunter` removes the literal `0F 05` (`syscall`) opcode from the image until `SPT_SanitizeSyscalls()` runs — call it **before** any `Nt*` function.
- None of these bypass kernel-level EDR (minifilter, ETW callbacks, kernel callbacks). They only avoid the **userland ntdll export hook** class of detection. Combine with the `windows-postexploit` stealth program (`unhook_ntdll`, `ppid_spoof`) when those apply.

## Limitations

- x86/WoW64 paths are marked WIP upstream — prefer `-x64` for production payloads.
- Generates source only; you are responsible for the host program that calls it and for linking against `kernel32`/`ntdll` where required.
- Preset `all` produces a very large file — prefer `common`, `donut`, or an explicit `-f` list sized to the payload.
