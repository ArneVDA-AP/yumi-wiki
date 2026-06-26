---
title: "Yumi Wiki"
tags: [yumi, overview]
---

# Yumi Wiki

**Yumi** is a personal, open desktop GUI for the **Claude Code CLI**, built with
**Tauri 2 + React 19**. It spawns your real `claude` binary as a subprocess,
parses its `stream-json` output, and renders text, **thinking blocks**, tool
calls, and results in a fast, OLED-themed chat UI. It is **not** an API wrapper:
the CLI handles auth, so Yumi uses your existing Claude subscription, and your
`CLAUDE.md`, MCP servers, hooks, and skills all work unchanged.

This wiki is the developer reference for how Yumi is put together. Every
load-bearing claim was checked against the actual source.

!!! info "Scope"
    Yumi is a personal project and intentionally **ungated** — the licensing /
    paywall of the original (**Yume**) is deliberately *not* cloned. See
    [Overview](overview.md).

## Read order

New to the codebase? Read these in order (≈25 min):

1. **[Overview](overview.md)** — what Yumi is, its relationship to Yume, scope & philosophy.
2. **[Architecture](architecture.md)** — the process model and the end-to-end data flow.
3. **[IPC Contract](ipc-contract.md)** — the frozen Rust⇄React interface (events, commands, types).
4. **[Stream Pipeline](stream-pipeline.md)** — how `stream-json` becomes chat messages.

Then dive into whichever subsystem you're working on.

## Catalog

### Orientation
| Page | What's in it |
|------|--------------|
| [Overview](overview.md) | What Yumi is, Yume relationship, scope, feature summary, parity tiers |
| [Architecture](architecture.md) | 3-process model, end-to-end data flow, the spawn→stream→parse→reduce→render pipeline |

### Reference
| Page | What's in it |
|------|--------------|
| [Backend (Rust)](backend-rust.md) | Module-by-module tour of `src-tauri/src` |
| [Frontend (React)](frontend-react.md) | `src/lib` + component map |
| [IPC Contract](ipc-contract.md) | `ClaudeEvent` union, Tauri commands, event channels, `ipc.ts` wrappers |
| [Stream Pipeline](stream-pipeline.md) | `stream-json` → `ClaudeEvent` mapping + the `streamParse` reducer |
| [Data & Persistence](data-and-persistence.md) | SQLite schema, analytics, settings, `~/.yumi` layout |

### Subsystems
| Page | What's in it |
|------|--------------|
| [MCP Bash Server](mcp-bash-server.md) | The reimplemented MCP stdio server + live bash tail |
| [Thinking Proxy](thinking-proxy.md) | The localhost proxy that enables interleaved thinking |
| [Yumi Plugin](yumi-plugin.md) | The bundled Claude Code plugin: agents, commands, guard hook |

### Using & building
| Page | What's in it |
|------|--------------|
| [Features & Shortcuts](features-and-shortcuts.md) | Feature catalogue + the full keyboard map |
| [Build, Run & Test](build-run-test.md) | Toolchain, dependencies, commands, artifacts |
| [Reliability & Design](reliability-and-design.md) | The root-caused bugs, their fixes, and key design decisions |
| [Troubleshooting](troubleshooting.md) | Common failure modes and how to resolve them |
| [Glossary](glossary.md) | Terms used throughout the wiki |

## Status at a glance

- **Stack:** Tauri 2 (Rust) + React 19 + TypeScript (Vite), WebView2, `rusqlite` (bundled, WAL).
- **Builds:** `windows-gnu` toolchain, no MSVC needed (`[lib] crate-type = ["rlib"]`).
- **Gates (last green 2026-06-26):** `cargo test` 38/38 · MCP server test 12/12 · `tsc && vite build` 0 errors · `tauri build --debug --no-bundle` → `yumi.exe`.
- **Scope:** personal use; no commercial/licensing surface.

!!! note "Where the code and the working docs live"
    This wiki is its own repository. The **Yumi application** lives in a separate
    repo: [github.com/ArneVDA-AP/yumi](https://github.com/ArneVDA-AP/yumi). The
    honest, per-feature status matrix (`PARITY.md`), the build contract
    (`CONTRACT.md`), and the continuation anchor (`PROGRESS.md`) live there.
    Source paths in this wiki are written relative to the app's project root
    (e.g. `src-tauri/src/claude/spawner.rs`).

---

*Last reviewed against source: 2026-06-26.*
